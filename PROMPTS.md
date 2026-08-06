# Prompts

Three prompts, in order. Run **Prompt 1** now and stop when it says to. Do not begin
Prompt 2 or 3 until I tell you to.

Operator notes — what a good run looks like, warning signs, and how to reset between
runs — are in `RUNBOOK.md`. That file is for the human, not for you.

---

## Prompt 1 — readiness check

```text
Before doing any work, run a readiness check and report back. Do not start any porting
task yet.

1. Read AGENTS.md in this directory and follow its setup section. Confirm you can see
   the skills it points to, and list them by name.

2. Read TARGET.md. It ships with placeholders and no credentials. Ask me for the
   connection details you need (user, host, SSH port, password or key path, and the
   authoritative aports tree path). Do not guess them and do not continue until you
   have them.

3. Confirm you can reach the target non-interactively. Prove it by running `uname -a`
   and `whoami` there and showing me the real output.

4. Confirm the authoritative aports tree exists, and report its path and git remote.
   If several aports trees exist, tell me which one has my fork as its remote.

5. Confirm privilege elevation works, because abuild cannot get root without a tty.
   Prefer the passwordless-apk route in TARGET.md so no password is stored on the
   target. If you use the wrapper fallback instead, show me `ls -la` of it and confirm
   the mode is 700, not 755. Either way, prove it with a real `apk add --simulate` of
   an already-installed package and show the output.

6. Report the count of orphaned makedepends virtuals: `apk info | grep makedepends`.
   Do not delete anything — just tell me the count and name them.

7. Confirm `abuild` is on PATH and that ~/.abuild/abuild.conf has a packager key.

8. In the package directory for the port, run `abuild clean && abuild -K unpack prepare`
   and show me the output. This needs no root and proves fetch, checksum verification
   and patch application all work.

9. Ask me whether to write the connection details into my local TARGET.md so future
   sessions do not have to ask. Do not write them unless I say yes, and do not run
   git commands against this repository.

Report what worked and anything you could not verify. Then stop and wait.
```

---

## Prompt 2 — the port

```text
extra/libassuan in the aports tree on the target fails to build on QNX 8.0. Port it.

Work only from the skills in this repository - load the qnx-porting router first and
let it route you. Read AGENTS.md and skills/ only. Do not read any other file in this
repository: not projects/apks/, not SOLUTIONS.md, not any previous port's notes, not
any file whose name suggests it holds an answer. Derive everything yourself from the
target.

All work happens natively on the target with abuild. Expect to hit a build wall that
needs a patch - follow aports-patch-creation exactly, and never create a patch from a
change you have not tested first.

When the port passes the validation gate, write the after-action REPORT.md per
qnx-port-reporting. It goes in this repository on the host, under
projects/apks/libassuan/ - not on the target, where it would be lost.

On git: use it as a tool, do not use it to write history. Read-only git (status, log,
diff, show) is encouraged, and the throwaway git repo inside src/ that
aports-patch-creation uses to generate the patch is expected. What you never do: commit
to the aports tree or to this repo, push anything, or run reset/checkout/rebase/clean.
I make the commits.

Before you fix each problem, show me the error and how you diagnosed it.
```

---

## Prompt 3 — the payoff

```text
Show me what this run left behind for the next one.

1. Show me the diff of everything you changed in this repository on the host.
2. Point specifically at what you recorded into the skills under the capture-friction
   rule - the durable platform fact, not the port-specific notes - and show that diff
   on its own.
3. In two or three sentences: what does the next port of an unrelated package get for
   free that it would not have had before this run?

Do not commit anything.
```
