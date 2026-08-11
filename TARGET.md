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

- Authoritative aports tree: `/var/home/qnx/aports`
  An image often carries several `aports*` trees (scratch copies, old experiments).
  Only one is authoritative; use this one path for all PR-bound work and do not read
  from the others. **This is a path, not a credential — keep it filled in.** Leaving it
  blank costs every session a round-trip with the operator, and the discovery sweep
  below cannot resolve the ambiguity on its own (see the warning there).
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

**The sweep produces candidates, not an answer.** "The tree whose remote is your fork" is
not discriminating in practice: on this image all four `aports*` trees share the same fork
as their remote and differ only by branch, so the heuristic selects everything. Treat the
sweep as a way to enumerate what exists, then have the operator name the authoritative one
**once**, and record it above so no later session has to ask again. If the path above is
already filled in, trust it and do not re-run this sweep to second-guess it.

## The on-target loop

1. SSH to the target.
2. `cd` into the package directory in the authoritative tree.
3. Edit APKBUILD / source / patches in place on the target.
4. Iterate with `abuild -K` (keeps `src/`); test changes with the native build system in the unpacked tree. Never `abuild -r` while iterating (it wipes `src/`).
5. Run the validation gate before reporting complete (see `AGENTS.md` and `qnx-apk-packaging`).
6. The human owns the commit and PR record. The agent may inspect the tree with read-only
   git and use git inside `src/` to generate patches, but never commits to the aports tree
   and never pushes.

## Non-interactive abuild dependency install

`abuild -r` installs makedepends through `$SUDO_APK`, which cannot gain root without a tty and fails with `builddeps failed`.

**Preferred fix — grant passwordless apk, so no password is stored anywhere.** Confirm the real path first; the sudoers rule must name the actual binary or it will parse, install, and silently never match:

```sh
command -v apk                      # /usr/bin/apk on the QSTI x86_64 image
echo 'qnx ALL=(ALL) NOPASSWD: /usr/bin/apk' | sudo tee /etc/sudoers.d/10-apk-nopasswd
sudo chmod 440 /etc/sudoers.d/10-apk-nopasswd
sudo visudo -c                      # must report no errors
```

`abuild -r` then works with no wrapper and no secret on disk. Add a new drop-in rather than editing an existing one; images often already ship something like `/etc/sudoers.d/00-qnx`.

**Fallback, only where sudoers cannot be edited** — a wrapper holding the dev password. Create it private: `chmod +x` alone leaves it world-readable (755) in a world-readable directory, which exposes the password that gets root to every local user.

```sh
(umask 077; cat > /tmp/sudo-apk <<'WRAPPER'
#!/bin/sh
printf '%s\n' <password> | sudo -S apk "$@"
WRAPPER
)
chmod 700 /tmp/sudo-apk
ls -la /tmp/sudo-apk                # confirm -rwx------

cd <authoritative-tree>/extra/<pkg>
SUDO_APK=/tmp/sudo-apk abuild -r
```

Note that `abuild -r` **uninstalls** makedepends when it finishes, so the second run in any sequence needs elevation again. A bare `abuild -r` that worked once will fail at `builddeps failed` the next time.

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
- Orphaned `.makedepends-*` virtual packages accumulate across interrupted builds and
  are a documented cause of `builddeps failed` (see `qnx-apk-packaging`). Check with
  `apk info | grep makedepends` and clear with
  `sudo apk del $(apk info | grep makedepends)`.
- Sudoers permissions are wrong out of the box (confirmed 2026-08-06). `sudo visudo -c`
  reports `bad permissions, should be mode 0440` for both `/etc/sudoers` (ships 0640)
  and `/etc/sudoers.d/00-qnx` (ships 0664, i.e. group-writable). sudo tolerates this
  today, but some builds refuse to run at all on bad sudoers permissions, and a
  group-writable drop-in is a local privilege concern. Fix once `visudo -c` otherwise
  passes: `sudo chmod 440 /etc/sudoers /etc/sudoers.d/00-qnx`.
- **The target clock drifts behind and silently breaks every HTTPS fetch** (confirmed
  2026-08-06). The QEMU guest clock does not track the host across suspend/resume: it read
  six days behind (`Jul 31` against a host `Aug 6`). Certificates issued inside that window
  have a `notBefore` in the target's future, so TLS is correctly rejected and every fetch
  dies before transferring a byte:

  ```
  curl: (60) SSL certificate problem: certificate is not yet valid or the system clock is incorrect
  ```

  This blocks `abuild` itself, not just manual `curl` - abuild pulls source tarballs over
  HTTPS, so a port cannot even start. The error names the clock, but it reads like a CA or
  network problem and is easy to chase in the wrong direction. Check it first on any fresh
  session, and note the clock can be *behind* even when `date` looks superficially sane:

  ```sh
  # on the host - a difference of more than a few minutes will break TLS
  diff <(date -u) <(sshpass -e ssh -p 2227 ... qnx@localhost 'date -u')
  ```

  Fix by setting it forward from the host's UTC time (forward-only, so no build-timestamp
  hazard):

  ```sh
  NOW=$(date -u +%m%d%H%M%Y.%S)
  printf '%s\n' <password> | sudo -S date -u $NOW
  ```

  There is no NTP client running on this image, so it will drift again.
- **The passwordless-apk drop-in grants `apk` and nothing else** (confirmed 2026-08-06).
  `sudo -n apk ...` works, but any other elevated command still needs the password route
  (`printf '%s\n' <password> | sudo -S ...`) - `sudo -n date` returns
  `sudo: a password is required`. That is the rule working as intended, not a broken
  sudoers file. Also note `/etc/sudoers.d/` is not readable by `qnx`, so the drop-in cannot
  be inspected directly; prove elevation functionally with
  `sudo -n apk add --simulate <installed-pkg>` instead.
- **Empty runtime packages are not unique to `libssh2` above.** Several packages on this
  image are installed, report a version, and ship no files at all, so a dependency can look
  satisfied while the library is absent from the filesystem. Check any runtime dependency
  you are about to rely on rather than trusting the install state:

  ```sh
  apk info -L <pkg>            # no file list = empty package
  ls /usr/lib/lib<name>.so*    # confirm the library actually exists
  ```
- **`apk info -e <pkg>` prints nothing on this image, installed or not** (apk-tools
  `3.0.0_rc5`, confirmed 2026-08-10). Any grep- or output-based test therefore reports
  *every* package as missing, which reads as a broken dependency and is not. Only the exit
  status carries the answer:

  ```sh
  apk info -e busybox   >/dev/null 2>&1; echo $?   # 0 = installed
  apk info -e nosuchpkg >/dev/null 2>&1; echo $?   # 1 = not installed
  apk list -I | grep '^<pkg>-'                     # use this when you want to see it
  ```
- **Where passwordless apk actually lives varies, so check before adding a drop-in**
  (confirmed 2026-08-10). On this image `/etc/sudoers.d/` is **empty** — there is no
  `00-qnx` — and the NOPASSWD rule is in `/etc/sudoers` itself. Following the drop-in
  recipe blindly adds a redundant rule. One command tells you what is really in force:

  ```sh
  sudo -n -l          # expect a line like: (ALL) NOPASSWD: /usr/bin/apk
  ```
- **The sudoers-permissions remediation above is operator work, not agent work.**
  Passwordless sudo covers `/usr/bin/apk` **only**, so `sudo visudo -c`, `sudo chmod` and
  even reading `/etc/sudoers` all fail with `sudo: a password is required`. An agent cannot
  verify or fix it; do not spend commands trying. Report it and move on.
- Any repaired system files: none so far on this image.
