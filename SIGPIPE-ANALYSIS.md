# SIGPIPE handling analysis for zms

This document records the analysis behind commit 8ea4bc15a (ignored SIGPIPE
in zms) and its subsequent revert 5ccc5c118.  The branch
`research_sigpipe-zms-analysis` preserves both the original change and this
write-up for future reference.

---

## What the commit does

Adds two lines to `src/zms.cpp`:

```cpp
#include <csignal>                       // line 27
std::signal(SIGPIPE, SIG_IGN);           // line 283, in main() after zmSetDefaultDieHandler()
```

With `SIG_IGN` installed, any write to a broken stdout pipe returns -1 with
`errno == EPIPE` instead of killing the process via signal 13.

---

## zms process model

zms is a **per-request CGI binary**, not a daemon.  The web server (Apache
or nginx) spawns one zms process per browser stream request.  The process
writes an HTTP multipart response to stdout, which the web server proxies
back to the browser.  When `runStream()` returns (or the process exits),
the connection is done.

Each zms process holds:
- one MySQL connection (via `dbconn` global)
- one or two threads (main + optional command_processor)
- 1-5 MB of memory (event_data, frame vector, image buffers)

Normal shutdown path:
`runStream()` returns → `exit_zm(0)` → `Image::Deinitialise()`,
`dbQueue.stop()`, `zmDbClose()`, `logTerm()` → `exit(0)`.

---

## How disconnects work without SIG_IGN (current behavior)

When the browser closes the connection (navigation, page refresh, stream
restart), the next `fputs`/`fprintf`/`fflush` to stdout delivers **SIGPIPE**.
The default handler terminates the process immediately—no destructors run, no
`exit_zm()` cleanup, no `zmDbClose()`.

The MySQL client library detects the abandoned connection server-side via
`wait_timeout` (default 28800s / 8h) and cleans it up.  In practice, the
connection slot is recovered well before any limit is hit for typical
ZoneMinder installations.

This is the **standard Unix behavior** for CGI processes and is relied on by
the web server to reap finished streams.

---

## How disconnects work with SIG_IGN (the reverted change)

With SIGPIPE ignored, writes to stdout return -1/EPIPE.  The shutdown
sequence depends on which code path detects the error:

| Write site | Return checked? | Propagation |
|---|---|---|
| `send_buffer()` (L1272-1284) | Yes — fprintf + fwrite | returns `false` → `sendFrame()` returns `false` → `zm_terminate = true` |
| `send_file()` (L1229-1270) | Yes — fprintf + zm_sendfile | returns `false` → same chain |
| `sendTextFrame()` in zm_stream.cpp (L251-309) | Partially — fputs/fprintf/fwrite checked; final fputs+fflush unchecked | returns `false` if early write fails; may return `true` with broken pipe if only the trailing writes fail |
| `sendFrame()` headers (L879, 941, 946, 952, 957) | **No** — fprintf/fputs unchecked | EPIPE silently ignored; continues to send_buffer/send_file which catches it |
| `sendFrame()` trailer (L971-972) | **No** — fputs + fflush unchecked | EPIPE silently ignored; returns `true`; caught on next iteration |
| `runStream()` content-type header (L983) | **No** | EPIPE ignored; first sendFrame catches it |
| MPEG path `vid_stream->EncodeFrame()` (L871-874) | **No** — writes go through ffmpeg | EPIPE may or may not propagate depending on ffmpeg's internal handling |

### Detection latency with SIG_IGN

- **Actively streaming**: < 100 ms.  `send_buffer()`/`send_file()` detect
  EPIPE on the next frame's Content-Length write.
- **Paused at event end**: up to **5 seconds** (`MAX_STREAM_DELAY` at
  `zm_stream.h:47`).  The process sits in `sleep_for(delta)` until the
  keepalive timer fires, then the next `sendFrame()` detects EPIPE.
- **Between events** (MODE_ALL): ~1 second.  `sendTextFrame()` fires at
  1-second intervals (L1050) and partially detects EPIPE.

### Missing error checks in EventStream

The `EventStream::runStream()` main loop (L1007-1213) has **no
`feof(stdout)` / `ferror(stdout)` check**.  Compare with
`MonitorStream::runStream()` which checks both at the top of every iteration
(zm_monitorstream.cpp L559-567):

```cpp
// MonitorStream has this; EventStream does not
while (!zm_terminate) {
  if (feof(stdout)) {
    Debug(2, "feof stdout");
    zm_terminate = true;
    break;
  } else if (ferror(stdout)) {
    Debug(2, "ferror stdout");
    zm_terminate = true;
    break;
  }
  // ...
}
```

Without this check, EventStream relies entirely on `sendFrame()` return
values to detect a broken pipe.  Since many of sendFrame's own writes are
unchecked, detection depends on the specific code path (raw JPEG via
send_file/send_buffer works; MPEG path may not).

---

## Resource cost of lingering (with SIG_IGN)

If EPIPE detection is delayed:

| Resource | Cost | Duration |
|---|---|---|
| MySQL connection | 1 idle conn (~1 KB server-side) | up to 5 s |
| Threads | 2 (main + command_processor) | up to 5 s |
| Memory | 1-5 MB (event_data + frames vector) | up to 5 s |
| File descriptors | stdin, stdout, stderr, MySQL socket | up to 5 s |

For EventStream the worst case is 5 seconds of lingering.  This is benign
for any reasonable number of concurrent viewers.  MonitorStream would be
worse in theory (unbounded), but it already has the feof/ferror check.

---

## Why SIG_IGN is not needed as a standalone fix

Two other commits in the `event-playback-fixes` series address the symptoms
that motivated the SIGPIPE patch:

1. **Keepalive timer reset** (604648355) — resets `last_frame_sent = {}`
   when pausing at event end, so a keepalive fires immediately rather than
   leaving a 5-second gap during which the connection can silently break.

2. **JS crash recovery** (62a3585df) — the browser-side JS detects a dead
   zms stream (`zmsBroke` handler) and restarts it, so even if SIGPIPE kills
   zms, the user experience recovers automatically.

Together these make the SIGPIPE kill harmless in practice: the browser
restarts the stream within ~1 second and the old zms process is already
gone.

---

## When SIG_IGN becomes valuable

Ignoring SIGPIPE is the correct approach for a **robust streaming server**,
but only if the EPIPE error path is complete.  It would become valuable as
part of a larger effort that:

1. **Adds feof/ferror checks** to EventStream's main loop (matching
   MonitorStream).
2. **Checks return values** on all stdout writes in `sendFrame()` — the
   headers at L879, L941, L946, L952, L957 and the trailer at L971-972.
3. **Audits the MPEG path** to ensure `vid_stream->EncodeFrame()` propagates
   write errors.
4. **Checks sendTextFrame's trailing writes** — the fputs+fflush at
   zm_stream.cpp L305-306 currently ignore EPIPE.

Without these fixes, SIG_IGN creates a half-working error path: some
disconnects are detected cleanly, others cause the process to spin on
failed writes until the keepalive timer catches it.  The current SIGPIPE
termination is more predictable.

---

## Verdict

SIGPIPE termination is the **intended and currently cleanest** exit path for
zms on browser disconnect.  The process dies immediately, the web server
reaps it, and the JS crash-recovery layer handles the browser side.

Installing `SIG_IGN` without completing the error-checking gaps creates an
inconsistent state where some code paths handle EPIPE gracefully and others
silently ignore it.  The fix should be deferred until the stdout write audit
(items 1-4 above) is done, at which point SIG_IGN + EPIPE handling becomes
strictly better than signal termination.

---

## Commit references

| Commit | Description |
|---|---|
| 8ea4bc15a | `fix: ignore SIGPIPE in zms to survive broken HTTP connections` |
| 5ccc5c118 | `Revert "fix: ignore SIGPIPE in zms to survive broken HTTP connections"` |
| 604648355 | `fix: reset keepalive timer on event-end pause to prevent connection drop` |
| 62a3585df | `fix: recover from zms crash by restarting stream on seek` |

## Key source locations (line numbers as of this branch)

| File | Lines | What |
|---|---|---|
| src/zms.cpp | 280-283 | SIG_IGN installation (on this branch) |
| src/zm.cpp | 27-35 | `exit_zm()` — cleanup on normal exit |
| src/zm_eventstream.cpp | 824-975 | `sendFrame()` — stdout writes, partial error checking |
| src/zm_eventstream.cpp | 977-1227 | `runStream()` — main loop, **no feof/ferror** |
| src/zm_eventstream.cpp | 1033-1044 | keepalive timer (MAX_STREAM_DELAY = 5s) |
| src/zm_eventstream.cpp | 744-815 | `checkEventLoaded()` — event-end pause logic |
| src/zm_eventstream.cpp | 1229-1284 | `send_file()` + `send_buffer()` — **has** error checking |
| src/zm_stream.cpp | 251-309 | `sendTextFrame()` — partial error checking |
| src/zm_stream.h | 47 | `MAX_STREAM_DELAY = Seconds(5)` |
| src/zm_monitorstream.cpp | 558-567 | feof/ferror check — **EventStream lacks this** |
