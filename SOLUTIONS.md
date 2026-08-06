# Solutions — SPOILERS

> **This file is a spoiler. Remove it from the working directory before any run whose
> point is that the agent derived the fix itself.** An agent that reads this retrieves
> the answer instead of deriving it, and nobody watching a recording can tell the
> difference. `PROMPTS.md` instructs the agent not to read it, but an instruction not
> to read a file is not the same as the file being absent.
>
> It exists for the reader who is learning the workflow and wants to know what "good"
> looks like without running anything.

---

## libassuan 2.5.7

**Symptom** — `abuild -r` fails in `src/assuan-socket.c`:

```
assuan-socket.c:714:3: error: use of undeclared identifier 'fd_set'
assuan-socket.c:715:18: error: variable has incomplete type 'struct timeval'
assuan-socket.c:719:3: error: call to undeclared function 'FD_ZERO'
assuan-socket.c:720:3: error: call to undeclared function 'FD_SET'
assuan-socket.c:788:9: error: call to undeclared function 'select'
```

**Cause** — the file declares `fd_set fds;` and a `struct timeval`, then calls `FD_ZERO`,
`FD_SET` and `select`. Its POSIX include branch brings in `<sys/types.h>`,
`<sys/socket.h>`, `<netinet/in.h>` and `<arpa/inet.h>` and nothing else. On QNX those
symbols live in `/usr/include/sys/select.h`, and neither `<sys/types.h>` nor
`<sys/socket.h>` pulls it in — glibc does, which is why the code builds on Linux.

**Fix** — a `__QNX__`-gated include, so no other platform is affected:

```c
 # include <arpa/inet.h>
+# ifdef __QNX__
+#  include <sys/select.h>
+# endif
 #endif
```

**Result** — `abuild -r -c -K` exits 0 in about a minute, the full upstream suite passes
(`version`, `pipeconnect`, `fdpassing`), and four APKs are produced: `libassuan`,
`-dev`, `-static`, `-doc`.

**Durable fact worth recording** — QNX does not pull `<sys/select.h>` in transitively.
Any Linux source using `fd_set`/`select` behind only `<sys/types.h>` or `<sys/socket.h>`
will hit this. That belongs in `qnx-platform-facts`, and a correct run puts it there.
