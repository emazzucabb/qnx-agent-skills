# QNX Agent Skills

This repository turns any skills-capable coding agent into a QNX 8.0 development companion. It carries a set of skills (QNX platform knowledge, Alpine aports porting, packaging, patch creation, port reporting) plus the rules that keep the work sound.

If you are an AI agent reading this file, follow the setup below, then work from the skills.

## Agent setup

**1. Make the skills discoverable.** The canonical skill set lives in `skills/`. Symlinks are already committed for the common project-level locations, so if you are running from this directory you should find the skills at whichever of these your client uses:

```
.claude/skills/     .codex/skills/     .agents/skills/
```

All three point at `skills/`. There is one copy on disk; nothing needs duplicating.

If your client only reads a global directory (`~/.claude/skills`, `~/.codex/skills`, `~/.agents/skills`), run `./setup.sh` once. It links `skills/` into the global location for the client you name. If your client uses a location none of the above cover, create the symlink yourself and open a PR adding it here so the next person gets it for free.

**2. Read `TARGET.md`.** It holds how to reach the QNX target: connection, authentication, the authoritative aports tree path, and any facts specific to that image.

`TARGET.md` ships with placeholders and **no credentials**, by design — it is a tracked file, so anything written into it can end up pushed. When you find placeholders:

1. Ask the user for the missing values (user, host, port, password or key path, tree path). Do not guess them, and do not proceed to anything that touches the target until you have them.
2. Verify the connection works before doing anything else.
3. Then **offer** to record the values in the user's local `TARGET.md` so future sessions do not have to ask again. Only write them if the user says yes.
4. If you do write them, tell the user plainly that `TARGET.md` is tracked by git, that a password committed to a shared or public repo is exposed permanently, and that `git update-index --skip-worktree TARGET.md` keeps their filled-in copy out of commits.

Never commit the change yourself, and never run `git update-index` on the user's behalf — rule 3 below applies here as everywhere.

**3. Load the `qnx-porting` skill.** It is the router. It holds the task-to-skill mapping and repeats these rules in the form your client will apply them.

## Universal rules

These hold for every task, in every skill.

1. **Prove claims with command output before acting.** Do not assert platform behaviour, dependency state, or build results as fact without having run the command that shows it. When you lack the information, say so and go get it rather than filling the gap with plausible reasoning.

2. **Never create a patch from an untested change.** Test in the unpacked source tree with native build tools first. The patch is written only after the change is confirmed working.

3. **Never push, and never make the commits a human owns.** Git itself is not off limits — it is a tool you need. Run it freely for inspection (`git status`, `log`, `diff`, `show`), and use it inside `src/` to generate patches, which is the required patch workflow (see `aports-patch-creation`). What you never do: `git push`; commit to the aports tree or any repo a human reviews; or run history-altering or destructive commands (`reset --hard`, `checkout --`, `rebase`, `clean`) that could discard work. Separately, do not run `git update-index` on the user's behalf: skip-worktree is their choice about their own checkout, not a build step. Prepare the change and hand off — the human commits, branches, and pushes.

4. **Native builds on the target.** Packages are built on the QNX target with `abuild`, compiled by the target's own toolchain, not cross-compiled from the host with `qcc`/`q++` and toolchain files. (The QNX SDP may be used on the host to launch a QEMU target; that is separate from building the package.) If a task or an old document assumes a host-side cross-compile, that is the legacy path and does not apply here.

5. **Record confirmed facts back into the right skill** the moment they are proven, so the next session starts ahead of this one.

6. **Capture friction as you go.** Anything that slows a session down, breaks, or could be done faster becomes a skill or `TARGET.md` update the moment it is proven, not deferred. Route each learning to where the next session will look for it: a platform fact to `qnx-platform-facts`, a build or packaging technique to `alpine-qnx-porting` or `qnx-apk-packaging`, a patch lesson to `aports-patch-creation`, a target or connection quirk to `TARGET.md`. The test: if a future session would otherwise rediscover this the hard way, record it now. When adding or editing a skill, follow `skill-authoring`.

## Skill map

```
qnx-porting (router, read first)
├── qnx-platform-facts        QNX platform truths (libc gaps, stack, macros, toolchain)
├── alpine-qnx-porting        APKBUILD adaptation, per build system
├── qnx-apk-packaging         end-to-end port-to-PR workflow and validation gate
├── qnx-port-reporting        the after-action REPORT.md written per port
├── aports-patch-creation     patch workflow and format gate
└── skill-authoring           how to add or extend a skill in this repo
```

## Per-port notes

This repository lives on the **host**, not on the target. Notes, reports and skill updates are written here; source, APKBUILDs, patches and builds live on the target. Keep that split straight: a `REPORT.md` written into the target's filesystem is lost work.

Each non-trivial port gets a folder under `projects/apks/<pkgname>/` holding a `PROJECT-INDEX.md` (start here), a project README carrying the running changelog, and a `REPORT.md` (the after-action write-up for a human reviewer). Copy `projects/PROJECT-INDEX-template.md` when starting a new one. Keep per-package specifics there so the shared skills stay general.

## Validation gate

Before reporting any port complete. Run these **on the target**, in the package directory of the authoritative aports tree:

```bash
abuild clean && abuild -K unpack prepare  # patches APPLY here, no Hunk FAILED, no .rej
abuild -r -c -K                        # builds, tests pass, expected APKs produced
find pkg -name '*.so*' | sort          # subpackage split correct, nothing orphaned
git status                             # read-only check: only intended files modified
```
