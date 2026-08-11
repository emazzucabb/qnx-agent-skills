# Runbook — for the human running the exercise

The exercise is three short prompts — can you see the skills, connect to my target and
show me the aports tree, port `<package>` and write the report. Everything the operator
needs around them is here.

This file does not contain the answer, but it does spell out the pass criteria and name
the skill a correct run updates, so keep it out of the agent's way during a run you intend
to judge — hand over a copy of the repo without it, rather than relying on an instruction
not to read it.

Neither your prompts nor any worked answers are tracked (see `.gitignore`). Keep a local
`PROMPTS.md` if it helps you stay consistent between runs, and if you keep a `SOLUTIONS.md`
store it outside the working tree so it cannot be copied into a run by accident.

---

## Before the first run

- A QNX 8.0 target reachable over SSH, with an `aports` tree checked out on it.
  `TARGET.md` explains how to get one if you have none.
- `sshpass` on the host if the target uses password auth.
- Your connection details to hand. Prompt 1 asks for them in chat rather than reading
  them from `TARGET.md`, which ships with placeholders and no credentials because it is
  a tracked file.
- Open the agent in the repo root. The skills are linked into `.claude/skills`,
  `.codex/skills` and `.agents/skills`; if your client only reads a global directory,
  run `./setup.sh` once.

If you let the agent save your details into `TARGET.md`, keep them out of commits:

```bash
git update-index --skip-worktree TARGET.md
```

---

## What a good Prompt 1 looks like

- Seven skills listed: `qnx-porting`, `qnx-platform-facts`, `alpine-qnx-porting`,
  `qnx-apk-packaging`, `qnx-port-reporting`, `aports-patch-creation`, `skill-authoring`
- Real command output — `QNX <host> 8.0.0 ... x86pc QNX` — not a description of it
- The tree path and a git remote pointing at your fork
- Elevation proven with an actual `apk add --simulate` that returns OK, and either the
  passwordless-apk route or a wrapper at mode **700**
- The orphaned-virtuals count reported, nothing deleted
- `abuild clean && abuild -K unpack prepare` succeeding
- An offer to save your details, and no attempt to save them unasked

**Warning signs**

- Success claimed without command output. The universal rules require evidence; a
  readiness check that skips it is already off the rails.
- A wrapper left at mode 755 — that is the root password readable by every local user.
- Writing to `TARGET.md` without asking, committing anything, or starting to port.
- A non-zero orphaned-virtuals count that you then ignore: run the reset before the real
  run, or `builddeps failed` may bite mid-demo for reasons unrelated to the port.

## What a good Prompt 2 looks like

1. Reproduces the failure with `abuild -r` and shows the real compiler error
2. Diagnoses against the actual source and system headers, not from memory
3. Installs the make-dependencies and proves the fix in `src/` with the package's own
   native build system — **not** with `abuild`
4. Only then creates the patch, using git so the `a/`/`b/` format is right by construction
5. Registers it in `source=`, runs `abuild checksum`, confirms it applies with no
   `Hunk FAILED` and no `.rej`
6. Runs `abuild -r -c -K` — clean build, tests passing, expected APKs
7. Writes `REPORT.md` into this repo on the host
8. Records the durable platform fact into `qnx-platform-facts`

**The run is not continuous.** `aports-patch-creation` tells the agent to ask before
creating a patch and before running `abuild -r`, and Prompt 2 asks it to show each error
before fixing it. Expect three or four stops where it waits for you. That is the skills
working, not the agent stalling.

**Timing.** Three builds of roughly a minute each — measured on an x86_64 QEMU target:
reproduce ~52s, native `./configure && make` ~15s, full `abuild -r -c -K` ~57s. Total
wall time depends on how long the agent spends reasoning and how quickly you answer its
pauses, and has not yet been measured for a genuinely cold run. Time your first one and
record the real number here.

## Prompt 3

Step 8 above is the closing beat. Prompt 3 surfaces it: the agent shows the diff of what
it wrote back into the skills, and says what the next port gets for free. "It just made
the next port easier" is a stronger ending than a clean build log.

---

## Resetting between runs

A completed run leaves changes in **two** places. Reset both, or the next run starts with
the answer in hand.

### 1. The package on the target

Do not hardcode the patch filename — the agent picks its own name under the `NNNN-*.patch`
convention, so a literal `rm` may silently do nothing.

The exercise starts with the package directory **deleted from the worktree** — `git status`
in the aports tree should show the package's files as ` D` before a run. Reset therefore
means deleting the directory the agent created, not restoring it from git:

```bash
cd <aports>
rm -rf extra/libassuan          # the whole directory the run created
git status --short              # must show ' D' on the package files again, not ' M'
```

Do **not** use `git checkout -- extra/libassuan/APKBUILD` here. That restores the committed
reference port into the worktree, which un-deletes the answer and destroys the premise of
the next run. It is the right command only if your baseline has the package tracked and
merely modified; confirm which baseline you have with `git status` before reaching for it.

Clear orphaned makedepends virtuals left by interrupted builds — `qnx-apk-packaging`
names these as a cause of `builddeps failed`:

```bash
apk info | grep makedepends
sudo apk del $(apk info | grep makedepends)    # if any; not while a build is running
```

**Then check that the purge did not take the toolchain with it.** Those virtuals own real
build dependencies, and deleting them cascades: on this image it removed 138 packages and
took `make` along, which makes the *next* run fail with `make: not found` — an environment
failure that looks nothing like a porting problem and wastes a take.

```bash
command -v make cc patch tar pkgconf     # all must resolve
sudo apk add build-base                  # restores make and the compiler if any are gone
```

### 2. This repo on the host

A correct run writes `projects/apks/libassuan/REPORT.md` and records the platform fact
into `skills/qnx-platform-facts/SKILL.md`, possibly touching `TARGET.md` too.

```bash
git status                                   # see what the run changed
git checkout -- skills/ TARGET.md            # only if you want to re-derive the fact
rm -rf projects/apks/libassuan/
```

Keep the recorded fact if you are using the repo for real work — that is the system
working as designed. Revert it only when re-running the exercise.

### 3. Move the spoilers out of reach

Worked answers are not tracked here, so there is normally nothing to move. Check anyway —
a spoiler that rides along in a folder copy is invisible until someone reviews the
recording:

```bash
ls SOLUTIONS.md 2>/dev/null                  # must not exist in the copy you hand over
ls projects/apks/<pkgname>/                  # a previous run's REPORT.md is also an answer
grep -rn "<pkgname>" skills/                 # skills must not name the exercise package
```

Also move any directory on the target holding a finished copy of the same patch, and
remember `RUNBOOK.md` itself names the skill a correct run updates.

### 4. Confirm the starting state is genuinely broken

```bash
cd <aports>/extra/libassuan
SUDO_APK=/tmp/sudo-apk abuild -r 2>&1 | grep "error:" | head -3
```

The `SUDO_APK` prefix matters: bare `abuild -r` uninstalls makedepends when it finishes,
so the next run cannot reinstall them without a tty and dies at `builddeps failed` before
compiling anything — you would see no compiler error at all and think the reset failed.

---

## Ground rules the agent is held to

From `AGENTS.md`. Worth knowing so you can tell a good run from a bad one.

1. **Claims are proven with command output.** No asserting a build works without the
   command that shows it.
2. **No patch from an untested change.** The fix is validated in the unpacked source
   first; the patch is written afterward.
3. **Never push, and never write the commits a human owns.** Git as a tool is fine —
   read-only inspection, and the throwaway repo in `src/` that generates the patch.
4. **Native builds on the target.** Built on QNX with `abuild`, not cross-compiled.
