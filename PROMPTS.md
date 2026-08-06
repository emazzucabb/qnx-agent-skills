# Getting Started: Prompts for a Full End-to-End Port

This file gives you a copy-pasteable path from a fresh clone of this repo to a
completed, validated QNX port. Run the two prompts in order.

The worked example is `libassuan`, which fails to build on QNX 8.0 for one small,
well-understood reason. Swap in any package once the flow works.

---

## Before you start

You need:

- A running QNX 8.0 target you can reach over SSH, with an `aports` tree checked out
  on it. (See `TARGET.md`; if you have no target yet, that file explains how to get one.)
- `sshpass` on the host if your target uses password auth.
- Your target's connection details to hand: user, host, SSH port, password or key path,
  and the path to your authoritative aports tree.

> **`TARGET.md` ships with placeholders and no credentials, on purpose.** It is tracked
> by git, so anything written into it can be pushed. You supply your connection details
> in the first prompt below instead. Once the agent has verified they work, it will offer
> to save them into your local `TARGET.md` so later sessions do not have to ask — accept
> only if you are comfortable with that file holding them, and keep it out of commits:
>
> ```bash
> git update-index --skip-worktree TARGET.md
> ```

Then open your agent in this directory. The skills are already linked into
`.claude/skills`, `.codex/skills` and `.agents/skills`, so most clients find them with no
setup. If yours only reads a global directory, run `./setup.sh` once.

---

## Prompt 1 — readiness check

Paste this first. It verifies the agent can see the skills and reach the target, and
nothing else.

Fill in the four values at the top, then paste the whole thing.

```text
Before doing any work, run a readiness check and report back. Do not start any porting
task yet.

My QNX target:
- user:     <user>
- host:     <host>          SSH port <port>
- auth:     password <password>        (or: SSH key at <path/to/key>)
- aports:   <path to the authoritative aports tree, if you know it>

1. Read AGENTS.md in this directory and follow its setup section. Confirm you can see
   the skills it points to, and list them by name.
2. Read TARGET.md. Using the connection details above, confirm you can reach the target
   non-interactively. Prove it by running `uname -a` and `whoami` on the target and
   showing me the real output.
3. Confirm the authoritative aports tree exists on the target, and report its path and
   git remote. If several aports trees exist, tell me which one has my fork as its
   remote.
4. TARGET.md ships with placeholders. Once the connection is verified, ask me whether
   you should write these values into my local TARGET.md so future sessions do not have
   to ask. Do not write them unless I say yes, and do not run any git commands.

Report what worked and anything you could not verify. Then stop and wait.
```

**A good result looks like:**

- Seven skills listed: `qnx-porting`, `qnx-platform-facts`, `alpine-qnx-porting`,
  `qnx-apk-packaging`, `qnx-port-reporting`, `aports-patch-creation`, `skill-authoring`
- Real target output, e.g. `QNX <host> 8.0.0 ... x86pc QNX` — actual command output,
  not a description of what it would show
- The tree path and a git remote pointing at your aports fork
- An offer to save the connection details to your local `TARGET.md`, and no attempt to
  save them without asking

**Warning signs:**

- It reports success without showing command output. The universal rules require claims
  to be backed by real output; a readiness check that skips that is already off the rails.
- It writes to `TARGET.md` without asking, or runs any git command.
- It starts porting something. This prompt ends with "stop and wait" for a reason.

---

## Prompt 2 — the port

Once the readiness check is clean, paste this. Replace `extra/libassuan` with your
package.

```text
extra/libassuan in the aports tree on the target fails to build on QNX 8.0. Port it.

Work only from the skills in this repository - load the qnx-porting router first and let
it route you. Do not use any other skills, and do not read anything under projects/apks/
or any previous port's notes; derive everything yourself.

All work happens natively on the target with abuild. Expect to hit a build wall that
needs a patch - follow aports-patch-creation exactly, and never create a patch from a
change you have not tested first.

When the port passes the validation gate, write the after-action REPORT.md per
qnx-port-reporting under projects/apks/libassuan/.

Do not run any git commands - I handle all git operations. Before you fix each problem,
show me the error and how you diagnosed it.
```

**What a correct run does**, roughly 10-15 minutes with three builds of about a minute
each:

1. Reproduces the failure with `abuild -r` and shows the real compiler error
2. Diagnoses it against the actual source and system headers, not from memory
3. Installs the make-dependencies and proves the fix in `src/` with the package's own
   native build system — **not** with `abuild`
4. Only then creates the patch, using git so the `a/`/`b/` header format is correct by
   construction
5. Registers it in `source=`, runs `abuild checksum`, and confirms it applies with no
   `Hunk FAILED` and no `.rej`
6. Runs `abuild -r -c -K`, gets a clean build with tests passing and the expected APKs
7. Writes `REPORT.md`

For `libassuan` specifically, the answer is a three-line `__QNX__`-gated
`#include <sys/select.h>` in `src/assuan-socket.c`. QNX does not pull that header in
through `<sys/types.h>` the way glibc does, so `fd_set`, `struct timeval` and the `FD_*`
macros are all undeclared without it.

---

## Re-running

The agent leaves the package modified and the patch in place. To run the exercise again
on the same package, restore it to its pre-port state:

```bash
cd <aports>/extra/libassuan
rm -f 0001-qnx-include-sys-select.patch
sed -i '/0001-qnx-include-sys-select.patch/d' APKBUILD
abuild checksum
abuild clean && rm -rf src pkg
```

Confirm the starting state is genuinely broken before you begin:

```bash
abuild -r 2>&1 | grep "error:" | head -3
# expect: use of undeclared identifier 'fd_set'
```

If you are demonstrating this to an audience, also move any directory containing a
finished version of the same patch out of the agent's reach first — otherwise it may
find the answer instead of deriving it.

---

## Ground rules the agent is held to

These come from `AGENTS.md` and apply to every run. Worth knowing so you can tell a good
run from a bad one:

1. **Claims are proven with command output.** No asserting that something builds, or that
   a dependency exists, without having run the command that shows it.
2. **No patch from an untested change.** The fix is validated in the unpacked source
   first; the patch is written afterward.
3. **The human runs all git operations.** The agent edits and builds. It never commits
   and never pushes.
4. **Native builds on the target.** Packages are built on QNX with `abuild`, not
   cross-compiled from the host.
