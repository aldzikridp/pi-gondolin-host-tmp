# PLANS.md — Move guest scratch dirs off RAM (host-disk mounts)

Target: `pi/packages/coding-agent/examples/extensions/gondolin/index.ts` (example
extension, `@earendil-works/gondolin` 0.12.0, pi 0.85.1).

## Goal

The stock Gondolin guest init mounts tmpfs (RAM) on `/tmp`, `/var/tmp`,
`/var/cache`, `/var/log`, and `/root` before `sandboxd` starts
(`host/src/alpine/init-scripts.ts` in gondolin). Guest tools like npm/pip dump
caches and build scratch into `/tmp` and `$HOME` (`/root`), so large builds eat
guest RAM.

Make `/tmp`, `/var/tmp`, `/var/cache`, and `/root` host-disk backed for the pi
extension session, without building a custom image.

## Design

- Create one host scratch root per VM session with `fs.mkdtempSync()`
  (`mktemp -d` semantics: 6 random chars, atomic exclusive create, mode 0700).
- One subdirectory per guest mount, chmod'd to match the mode the guest has
  today (measured: 1777 for the tmp dirs, 700 for `/root`; `mkdir` mode is
  umask-filtered, so chmod explicitly after creation).
- Add four `RealFSProvider` entries to `VM.create({ vfs: { mounts } })`.
  Gondolin's init mounts tmpfs first and then bind-mounts every
  `sandboxfs.bind=` entry over it, so the VFS mount wins. The hidden tmpfs
  receives no writes and therefore consumes no RAM.
- Everything else stays VM-local: only these four paths write through to host.
- Cleanup: close the VM first, then `rmSync(scratchRoot)` on session shutdown;
  also delete the scratch root if `VM.create` throws (so a retry via
  `ensureVm()` cannot leak directories).

Not in scope: `/var/log` (stays tmpfs; can be added later with the same
pattern), custom-image alternative (`rootfsInitExtra` + `rootfs.sizeMb`).

## Changes — `index.ts` only, no new dependencies

### 1. Header comment (top of file)

The doc comment currently says "other guest filesystem changes are isolated to
the VM." Update it to say the workspace **and the scratch dirs** (`/tmp`,
`/var/tmp`, `/var/cache`, `/root`) write through to the host.

### 2. Imports

```ts
import fs from "node:fs";
import os from "node:os";
import path from "node:path";
```

### 3. Constants (next to `GUEST_WORKSPACE`)

```ts
const SCRATCH_MOUNTS: Record<string, { subdir: string; mode: number }> = {
	"/tmp": { subdir: "tmp", mode: 0o1777 },
	"/var/tmp": { subdir: "var-tmp", mode: 0o1777 },
	"/var/cache": { subdir: "var-cache", mode: 0o1777 },
	"/root": { subdir: "root", mode: 0o700 },
};
```

### 4. Module state + helpers (inside `export default function (pi)`)

```ts
let scratchRoot: string | undefined;

function removeScratchMounts(): void {
	if (!scratchRoot) return;
	fs.rmSync(scratchRoot, { recursive: true, force: true });
	scratchRoot = undefined;
}

function createScratchMounts(): Record<string, RealFSProvider> {
	const root = fs.mkdtempSync(path.join(os.tmpdir(), "gondolin-scratch-"));
	const mounts: Record<string, RealFSProvider> = {};
	try {
		for (const [guestPath, { subdir, mode }] of Object.entries(SCRATCH_MOUNTS)) {
			const hostPath = path.join(root, subdir);
			fs.mkdirSync(hostPath, { recursive: true });
			fs.chmodSync(hostPath, mode);
			mounts[guestPath] = new RealFSProvider(hostPath);
		}
	} catch (error) {
		fs.rmSync(root, { recursive: true, force: true });
		throw error;
	}
	scratchRoot = root;
	return mounts;
}
```

### 5. `startVm()` — wire into `VM.create`

```ts
const created = await VM.create({
	sessionLabel: `pi ${path.basename(localCwd)}`,
	vfs: {
		mounts: {
			[GUEST_WORKSPACE]: new RealFSProvider(localCwd),
			...createScratchMounts(),
		},
	},
}).catch((error) => {
	removeScratchMounts(); // no leak if create fails and ensureVm retries
	throw error;
});
```

### 6. `session_shutdown` — cleanup in `finally`

Replace the `if (!activeVm) return;` early return:

```ts
pi.on("session_shutdown", async (_event, ctx) => {
	const activeVm = vm;
	vm = undefined;
	vmStarting = undefined;
	ctx.ui.setStatus("gondolin", ctx.ui.theme.fg("muted", "Gondolin: stopping"));
	try {
		if (activeVm) await activeVm.close();
	} finally {
		removeScratchMounts();
		ctx.ui.setStatus("gondolin", undefined);
	}
});
```

### 7. `/gondolin` status command

Add a line so the host scratch location is visible when debugging:

```ts
`Scratch (host): ${scratchRoot ?? "(none)"}`,
```

Optional: extend the ready notification to mention that `/tmp`, `/var/tmp`,
`/var/cache`, and `/root` are host-disk backed.

## Behavior after change

- Guest `/proc/mounts` shows the tmpfs's still mounted, each shadowed by a
  `sandboxfs` bind on top (`nosuid,nodev`, inherited). Bind target wins.
- Guest modes stay identical to today: 1777 / 1777 / 1777 / 700.
- `TMPDIR=/tmp`, `XDG_CACHE_HOME=/tmp/.cache`, `UV_CACHE_DIR=/tmp/.cache/uv`,
  `HOME=/root` all keep working unchanged.
- `/root` is a fresh empty home per session — same as the current tmpfs
  behavior, so no image dotfiles become newly visible/hidden.
- Cost: these paths are now FUSE RPCs (slower than tmpfs), and host disk usage
  grows with guest scratch. `/workspace` already has the same properties.
- Isolation: `RealFSProvider` confines access to the scratch dirs (lexical `..`
  check + realpath follow check, symlink escapes rejected, fail-closed). Fresh
  0700 root + atomic mkdtemp; only the host user running pi owns/writes them.
- If pi is SIGKILLed, the scratch root is left under host `os.tmpdir()`; normal
  exit always removes it, OS tmp-cleanup handles the rest.

## Verification (run on a real host)

1. `cd packages/coding-agent/examples/extensions/gondolin && npm install --ignore-scripts`
2. `cd /path/to/project && pi -e .../extensions/gondolin`
3. In the guest (`bash` tool):
   - `mount | grep -E ' /tmp | /root | /var/tmp | /var/cache '` → tmpfs line plus
     a `sandboxfs` line for each path.
   - `stat -c '%a %n' /tmp /var/tmp /var/cache /root` → `1777 1777 1777 700`.
   - `dd if=/dev/zero of=/tmp/probe bs=1M count=64` then `rm /tmp/probe`.
4. On the host: the probe file appears under the mkdtemp root
   (`ls /tmp/gondolin-scratch-*/tmp`), and host `df` shows the space consumed
   while guest `free -m` stays flat.
5. `touch /root/.probe` in guest → appears at `<scratch>/root/.probe` on host.
6. Quit pi → `ls -d /tmp/gondolin-scratch-*` is empty.
7. Regression: read/write/edit/bash/ls/find/grep tools still work against
   `/workspace`; guest `/var/log` remains tmpfs.

## Fallbacks

- FUSE too slow for `/tmp` heavy builds → drop `/tmp` from `SCRATCH_MOUNTS`
  (keep `/root`, `/var/tmp`, `/var/cache`), or set `TMPDIR` to a
  `/workspace/.tmp` directory instead.
- Want `/tmp` on the VM's local disk instead of FUSE → custom image with
  `init.rootfsInitExtra` that `umount`s the tmpfs mounts and `rootfs.sizeMb`
  baked at build time (runtime `rootfs.size` needs `resize2fs`, absent from the
  stock image).
