# Target Connection (QNX 8.0)

All porting work happens **on the QNX target itself**, over SSH. There is no host-side aports tree and no scp round-trip: connect to the target, edit the source/APKBUILD/patches there, and build there with `abuild`. The Linux host only launches the target (if it is a local VM) and runs the SSH session.

The simplest way to get a target is the official Quick Start Target Image (QSTI). See the [QSTI for QEMU guide](https://www.qnx.com/developers/docs/qnxeverywhere/com.qnx.doc.target_images/topic/qsti_qemu/about.html) or the [QSTI for Raspberry Pi guide](https://www.qnx.com/developers/docs/qnxeverywhere/com.qnx.doc.target_images/topic/qsti/intro.html). With QSTI you launch the target with `mkqnximage --run` and get its IP with `mkqnximage --getip`.

If you already have your own QNX 8.0 disk image, the `run.sh` in this repo is a QEMU launcher template you can edit and tune directly (set the image path, RAM, cores, and the SSH port forward). It is an alternative to QSTI for the bring-your-own-image case; the instructions are in the script's header comments.

## Connecting

```bash
ssh <user>@<host>
```

- User: `<user>` (depends on the image; QSTI images use `qnx`)
- Host: `<target-host>` (the device or VM IP; use `mkqnximage --getip` for QSTI)
- Password / key: fill in below. If SSH is forwarded to a non-standard host port
  (a hand-rolled QEMU launcher, for example), add `-p <port>` to every ssh and
  sshpass command in this file.

> Fill these in locally, but think before you commit them. This repo is public.
> A shared dev password in git history is public forever. Prefer key-based auth,
> or keep your filled-in copy out of commits (`git update-index --skip-worktree TARGET.md`).
- Port: standard SSH port `22` by default (the QSTI case, connect straight to the target IP). If your setup forwards SSH to a non-standard host port instead (for example a hand-rolled QEMU launcher forwarding host `2227` to guest `22`), add `-p <port>` to every ssh and sshpass command below.
- Authentication: fill in your method below (password or key)

For non-interactive use (required for Claude Code to work without prompting). Add `-p <port>` after `ssh` if your setup uses a forwarded port:

```bash
sshpass -p <password> ssh \
  -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o LogLevel=ERROR \
  <user>@<host> '<command>'
```

Or, if using key-based auth:

```bash
ssh -i ~/.ssh/<your-key> <user>@<host> '<command>'
```

## About authentication

A local development target often uses a simple shared dev password. That is fine for a throwaway local image. For anything shared or networked, use key-based SSH and proper secrets handling.

For sudo on the target, pipe the password:

```bash
printf '%s\n' <password> | sudo -S <command>
```

## On-target paths

- Authoritative aports tree: `<FILL IN THE PATH TO YOUR AUTHORITATIVE TREE>`
  An image often carries several `aports*` trees (scratch copies, old experiments).
  Only one is authoritative: the clean tree whose git remote points at your aports
  fork. Identify it once with the discovery sweep below and record it here, so no
  session has to guess. Use this one path for all PR-bound work.
- Package output: `/var/home/qnx/packages/<repo>/<arch>` (for example `extra/x86_64`)
- Local repo resolution: add local package output paths before remote repos in
  `/etc/apk/repositories`, then `sudo apk update` (see the `qnx-apk-packaging` skill).

## Discovery sweep (run on a fresh image to populate this file)

Run this on the target to find which aports trees exist and which is authoritative:

```bash
whoami; uname -m
ls -la /var/home/qnx
for d in /var/home/qnx/aports*; do
  echo "== $d =="
  git -C "$d" remote -v 2>/dev/null | head -1
  git -C "$d" rev-parse --abbrev-ref HEAD 2>/dev/null
done
```

Record the tree with the SSH remote pointing at your fork as the authoritative one above.

## The on-target loop

1. SSH to the target.
2. `cd` into the package directory in the authoritative tree.
3. Edit APKBUILD / source / patches in place on the target.
4. Iterate with `abuild -K` (keeps `src/`); test changes with the native build system in the unpacked tree. Never `abuild -r` while iterating (it wipes `src/`).
5. Run the validation gate before reporting complete (see CLAUDE.md and `qnx-apk-packaging`).
6. The human runs all git operations from the target. The agent never commits or pushes.

## Non-interactive abuild dependency install

`abuild -r` installs makedepends through `$SUDO_APK`, which cannot gain root without a tty and fails with `builddeps failed`. Fix: point `SUDO_APK` at a wrapper that pipes the dev password to sudo, then build with it:

```sh
cat > /tmp/sudo-apk <<'WRAPPER'
#!/bin/sh
printf '%s\n' <password> | sudo -S apk "$@"
WRAPPER
chmod +x /tmp/sudo-apk
cd <authoritative-tree>/extra/<pkg>
SUDO_APK=/tmp/sudo-apk abuild -r
```

## Token-efficient remote-build pattern

Redirect the full `abuild` log to a file on the target and pull back only key lines:

```sh
SUDO_APK=/tmp/sudo-apk abuild -r > /tmp/build.log 2>&1; echo "EXIT=$?"
grep -nE '>>>|Hunk FAILED|error:|Build complete|builddeps failed' /tmp/build.log | tail
```

## Image-specific facts (fill in as you discover them)

Record anything specific to your image that bit you once, so it is not rediscovered:

- Architecture: x86_64 QEMU. `uname -m` returns `x86pc`; `uname -a` reports
  `QNX localhost 8.0.0 ... x86pc QNX`. CMake reports `CMAKE_SYSTEM_NAME=QNX` and
  `CMAKE_SYSTEM_PROCESSOR=unknown` (confirmed 2026-08-05 with a standalone probe).
- Known missing busybox applets: `cmp` - fix with `checkdepends="diffutils"`;
  `killall` - no package available.
- Repository issues: check `/etc/apk/repositories` early. Two failure modes are common
  and both print on *every* `apk add` / `apk update`:
  - A pre-release or QA repo returning **HTTP 403 Forbidden** on the index.
  - An internal-only mirror that fails DNS from your network
    (`address family for host not supported`).
  Neither is fatal, but together they make every apk transaction noisy and slow, and
  the noise hides real errors. Comment out the ones that do not resolve for you.
- Broken packages on this image (confirmed 2026-08-05 during the libgit2 port):
  - `libssh2` (runtime) is an **empty package** - `apk info -L libssh2` lists no files,
    and there is no `libssh2.so*` anywhere on the filesystem.
  - `libssh2-dev` ships only headers plus `libssh2.pc`, and that `.pc` advertises
    `-lssh2`, so pkg-config reports success for a library that does not exist.
  - `libssh2-static` does provide `/usr/lib/libssh2.a`, but it was built **without
    `-fPIC`**, so it cannot be linked into any shared object
    (`relocation R_X86_64_PC32 ... can not be used when making a shared object`).
    Net effect: no consumer can enable SSH in a shared library on this image.
  - `libssh2` is not in the aports tree; it exists only in the local
    `/var/home/qnx/packages/extra` repo.
- Any repaired system files: none so far on this image.
