---
name: qnx-porting
description: "ALWAYS read this first for any QNX 8.0 porting or packaging task. This is the router skill: it identifies what kind of task you are facing and points to the focused skill that answers it. Holds the universal rules that apply to all QNX porting work. Covers native Alpine aports porting (build on the QNX target with abuild), patch creation, platform-level QNX facts, packaging workflow, and port reporting. Read this, then cascade into the specific skill named for your task."
---

# QNX Porting (Router)

This is the entry point for QNX 8.0 porting work. Read it first, identify the task, then load the specific skill it points you to. Do not try to hold every detail here; this skill orients and delegates. The detailed skills are the source of truth for their areas.

## Universal rules (apply to every QNX porting task)

These hold regardless of which sub-skill you are in:

1. Claims must be proven with command output before acting. Do not assert platform behaviour, dependency state, or build results as fact without having run the command that shows it. When you lack the information, say so and get the evidence rather than filling the gap with plausible reasoning.

   **This applies to facts read in these skills, too.** A skill is a strong prior, not evidence. Anything here describing the current environment — a variable being empty, a package being broken, a tool being absent — was true of one image at one time and is exactly the kind of fact that goes stale or was over-generalised from a single observation. Where a skill states such a fact and it is load-bearing for what you are about to do, spend the one command to confirm it. A skill claim that `$CBUILD`/`$CHOST` were empty once went unverified and put a hardcoded architecture triple into a package declaring `arch="all"`; one `echo` would have caught it. Verifying costs seconds, and when a skill turns out to be wrong, correcting it under rule 6 is the most valuable thing a session can leave behind.

2. Never create a patch from an untested change. Test in the unpacked source tree with native build tools first. The patch is written only after the change is confirmed working. (Full workflow: aports-patch-creation.)

3. Never push, and never make the commits a human owns. Git is a tool you need, not a forbidden one: run it freely for inspection (`git status`, `log`, `diff`, `show`), and use it inside `src/` to generate patches — that is the required patch workflow (see aports-patch-creation). Never `git push`; never commit to the aports tree or any repo a human reviews; never run history-altering or destructive commands (`reset --hard`, `checkout --`, `rebase`, `clean`). Separately, do not run `git update-index` on the user's behalf: skip-worktree is their choice about their own checkout, not a build step. Prepare the change and hand off; the human commits, branches, and pushes.

4. Build native aports packages on the QNX target with abuild. Do not cross-compile the package from a Linux host using qcc/q++ and toolchain files. If a task or an old document assumes a host-side cross-compile (qcc/q++, toolchain-files, build-files), that is the legacy cross-compile path and does not apply to native aports work. (The QNX SDP may still be used on the host to launch a QEMU target; that is fine and is separate from building the package.)

5. Record new confirmed facts back into the right skill the moment they are proven, so the next session starts ahead of this one.

6. Capture friction as you go. Anything that slows a session down, breaks, or could be done faster becomes a skill or TARGET.md update the moment it is proven, not deferred. Route each learning to where the next session will look for it: a platform fact goes in qnx-platform-facts; a build or packaging technique goes in alpine-qnx-porting or qnx-apk-packaging; a patch lesson goes in aports-patch-creation; a connection or target quirk goes in TARGET.md. The test: if a future session would otherwise rediscover this the hard way, record it now.

## How to use these skills

Load this router first, then the skill for your task. Beyond that:

**Read the whole skill you load, not just the part matching your current error.** The rules that decide the *shape* of your output usually sit near the top, before the troubleshooting material you came for. A skill skimmed for one error message will hand you a working result in the wrong form.

**One artifact can be governed by more than one skill.** A patch's content and mechanics are `aports-patch-creation`; the same patch is checked again in `qnx-apk-packaging`'s review-reduction pass. Where two skills speak to the same artifact, apply the rule in the skill you are *doing the work in* — the later skill is a check that it was followed, not the place to discover it. Reaching a rule only at the checking stage means rework, because by then the artifact is written, checksummed and validated.

**Read the validation gate and the review-reduction pass before you think you are done, not after.** They are the definition of finished. Meeting them by accident is luck; meeting them by design is the job.

**If you needed a rule earlier than the place it appears, that is friction — fix it.** Move it, or cross-reference it from the skill where the work actually happens, under rule 6. A correct rule filed at the wrong stage of the workflow fails exactly like a missing one, and the session that just tripped over it is the one best placed to say where it belonged.

## Task router

Find the row that matches what you are doing and load that skill.

**Adapting an Alpine APKBUILD for QNX** (dependency renames, main to core / community to extra, pkgrel reset, maintainer headers, build-system specifics like autotools or CMake or Meson): load `alpine-qnx-porting`.

**Creating, editing, or debugging a patch** (any .patch file, any "Hunk FAILED", any abuild patch-apply error, patch format and naming, and how the hunk itself should be written — platform conditionals and where the change belongs in the source): load `aports-patch-creation`. This is mandatory before producing any patch content.

**Taking a port from working build to PR-ready** (version baseline, APKBUILD metadata cleanup, local repo resolution, subpackage inspection, running tests, the review-reduction pass, the validation gate): load `qnx-apk-packaging`.

**Writing up a finished or blocked port** (the after-action REPORT.md: what was done, changes made, problems hit and resolved, validation results, open items): load `qnx-port-reporting`.

**Adding or extending a skill** (recording a newly proven fact, splitting an oversized skill, creating a skill for a new area): load `skill-authoring`.

**A platform-level problem** (a missing Linux syscall, a link error for sockets or regex, a stack-size segfault, a header difference, a toolchain quirk): load `qnx-platform-facts`.

## Skill map

```
qnx-porting (this router)
├── qnx-platform-facts        platform truths, build-method-independent
├── alpine-qnx-porting        APKBUILD adaptation (deps, build systems, conventions)
├── qnx-apk-packaging         end-to-end port-to-PR workflow + validation gate
├── qnx-port-reporting        after-action REPORT.md per port
├── aports-patch-creation     patch workflow and format gate
└── skill-authoring           how to add or extend a skill in this repo
```

## Per-port notes

Each non-trivial port should carry its own folder under `projects/apks/<pkgname>/` with a `PROJECT-INDEX.md` and a `README.md` recording: what was changed relative to upstream, why, what challenges were hit, and what remains open. Keep per-port specifics here, not in the shared skills, so the shared skills stay general.

## Git and PR conventions

The human owns the commit and PR record (rule 3); the agent may inspect it but does not write it. Use standard fork-based PRs: one branch per change on your personal fork, dependency packages submitted before consumer packages. Commit subjects follow the pattern `<repo>/<pkgname>: new aport` for new packages and `<repo>/<pkgname>: enable/fix build on QNX` for existing ones.
