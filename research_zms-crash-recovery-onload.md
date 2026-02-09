# zms Crash Recovery via `img.onload` (8b9d523ff)

## Summary

When zms (the ZoneMinder streaming server) dies mid-playback, the browser
UI silently breaks: the `zmsBroke` flag is set, and subsequent seek/play
commands are swallowed. This patch adds automatic stream restart with
reliable readiness detection via `img.onload`, consolidated into a shared
`restartZmsStream()` helper.

## Background: How zms Streaming Works

The event playback page (`event.js`) uses an `<img id="evtStream">` tag
whose `src` points to the zms CGI binary. zms returns a
`multipart/x-mixed-replace` response — the original Netscape "server push"
mechanism — where each MIME part is a JPEG frame. The browser renders each
part in succession, producing a live video feed.

Alongside the MJPEG image stream, a separate AJAX control channel polls
zms at regular intervals (`setInterval(streamQuery, streamTimeout)`). Each
poll sends a `CMD_QUERY` to `ajax/stream.php`, which forwards the command
over a Unix datagram socket (`zms-{connkey}s.sock`) and reads back a
status struct containing progress, rate, scale, etc. The response flows
through `getCmdResponse()` which updates the UI (progress bar, time
display, play/pause state).

The `connkey` is a 6-digit integer embedded in both the `<img>` URL and
the AJAX requests. It identifies a specific zms instance. The zms process
creates a lock file (`zms-{connkey}.lock`) and a socket file
(`zms-{connkey}s.sock`) on startup. `ajax/stream.php` waits up to ~1
second for the socket file to appear before giving up.

## The Problem

zms can exit unexpectedly for several reasons:

- SIGPIPE from a broken HTTP connection (mitigated by our SIGPIPE-ignore
  patch, but still possible in edge cases)
- Out-of-memory kill
- Segfault from the vector bounds issues (fixed in 011defdad, but present
  in stock ZoneMinder)
- Webserver timeout or process limit reclamation

When zms dies:

1. The `<img>` stream stops updating (last frame frozen on screen)
2. The next `CMD_QUERY` poll fails — `ajax/stream.php` can't find
   the socket file, returns an error JSON
3. `checkStreamForErrors()` returns true in `getCmdResponse()`
4. `zmsBroke` is set to `true` (line 441-442)
5. All subsequent `streamReq()` calls through the polling loop hit the
   same error, keeping `zmsBroke = true`
6. User clicks the progress bar to seek → `streamSeek()` calls
   `streamReq({command: CMD_SEEK})` → AJAX goes to `stream.php` →
   socket doesn't exist → error → nothing happens
7. User clicks Play → `streamReq({command: CMD_PLAY})` → same failure

The UI appears completely stuck. The only recovery was a full page reload.

### Pre-existing partial fix in `playClicked()`

The codebase already had restart logic in `playClicked()` (lines 558-566
before this patch) that checked `zmsBroke` and reset the `<img>` src. But
this logic:

- Was only in `playClicked()`, not in `streamSeek()` — seeking while
  zmsBroke still silently failed
- Set `zmsBroke = false` optimistically before zms was actually ready
- Was inlined, duplicating the restart mechanism

## The Solution

### Approach: `img.onload` as Readiness Signal

When you set `img.src` to a new zms URL, the browser makes a fresh HTTP
request to the CGI, spawning a new zms process with the same `connkey`.
The new process opens its command socket, loads event data, and begins
streaming JPEG frames.

The question is: **how do you know when the new zms is ready to accept
commands?**

Two approaches were evaluated:

#### Option A: Poll-based (`pendingSeek` in `getCmdResponse`)

This was the original implementation (now superseded). On restart, store
the desired seek offset in a `pendingSeek` variable. Set `zmsBroke = false`
immediately. When the next `CMD_QUERY` poll succeeds and
`getCmdResponse()` runs, check for `pendingSeek` and execute it.

Problems:

1. **Premature `zmsBroke = false`**: Setting the flag before zms is ready
   means the polling loop's `CMD_QUERY` calls hit `stream.php` while the
   socket doesn't exist yet. Each call waits up to 1 second
   (`usleep(1000)` x 1000 in stream.php lines 91-98), then returns an
   error. `getCmdResponse` sets `zmsBroke = true` again. This creates a
   toggle cycle until zms actually starts.

2. **Latency**: The seek doesn't execute until the next successful poll.
   With `ZM_WEB_REFRESH_STATUS` at its default, this adds up to several
   seconds of delay between the user's click and the seek actually
   happening.

3. **Double-restart race**: If the user seeks again during the restart
   window (while `zmsBroke` has been toggled back to `true` by a failed
   poll), a second `img.src` reset fires, spawning yet another zms. The
   lock file serializes them, but it's wasteful.

4. **Global state coupling**: `pendingSeek` is a module-level variable
   that creates an implicit contract between `streamSeek()` and
   `getCmdResponse()` — two functions with no obvious relationship.

#### Option B: `img.onload` (chosen)

For `multipart/x-mixed-replace` streams, browsers fire `img.onload` when
the first MIME part (JPEG frame) is fully decoded. This is the most
reliable signal that zms is alive: if it delivered a frame, it has
necessarily opened its command socket (socket setup happens in
`openComms()` before the streaming loop begins in `runStream()`).

Advantages:

1. **No premature state change**: `zmsBroke` stays `true` until `onload`
   fires. The polling loop's failed `CMD_QUERY` calls harmlessly set
   `zmsBroke = true` (already true). No toggle cycle.

2. **Minimal latency**: The seek fires the instant the first frame arrives,
   not on the next poll interval. Typically faster by the duration of one
   `streamTimeout` cycle.

3. **No global state**: The seek offset is captured in the `onload`
   closure. No `pendingSeek` variable needed.

4. **One-shot handler**: `img.onload = null` after firing prevents
   interference with normal MJPEG streaming (where browsers may fire
   `onload` per frame).

### Implementation

```javascript
function restartZmsStream(onReady) {
  const img = document.getElementById('evtStream');
  const url = new URL(img.src);
  url.searchParams.set('scale', currentScale);
  img.src = '';                    // kill old connection
  img.onload = function() {       // set handler before new src
    img.onload = null;             // one-shot
    zmsBroke = false;              // now safe
    if (onReady) onReady();        // e.g. streamSeek(offset)
  };
  img.src = url.href;              // triggers new zms spawn
}
```

The ordering matters:

1. **Save URL before clearing** — `new URL(img.src)` captures the current
   stream URL including connkey, event ID, auth, etc.
2. **Clear src first** — `img.src = ''` aborts the dead connection. No
   `onload` handler is set yet, so any error event from the empty src is
   harmless.
3. **Set onload before new src** — Ensures the handler is in place before
   the browser starts the new request. Even if the first frame arrived
   synchronously (impossible for a network request, but defensive), the
   handler would catch it.
4. **Set new src** — Browser initiates the CGI request, zms starts up.

### Call Sites

**`streamSeek(offset)`** when `zmsBroke`:
```javascript
restartZmsStream(function() { streamSeek(offset); });
```
The closure captures `offset`. When `onload` fires, `zmsBroke` is cleared
and `streamSeek` is re-invoked — this time taking the `else` branch and
sending `CMD_SEEK` normally.

**`playClicked()`** when `zmsBroke`:
```javascript
restartZmsStream();
```
No callback needed. zms starts in play mode by default (it begins
streaming frames immediately). The `streamPlay()` call at the end of
`playClicked()` updates the UI buttons.

## Edge Cases

### User seeks multiple times during restart

If the user clicks the progress bar again before `onload` fires:

- `streamSeek(offset2)` is called, `zmsBroke` is still `true`
- `restartZmsStream` is called again with a new closure
- `img.onload` is overwritten with the new callback (offset2)
- `img.src` is reset again — potentially killing the in-progress restart

The user's most recent seek intention wins. The lock file mechanism
(`zms-{connkey}.lock` with `flock(LOCK_EX)`) prevents multiple zms
instances from running concurrently on the same connkey.

### `onload` never fires

If zms fails to start entirely (e.g., missing event files, permission
error), `onload` never fires and `zmsBroke` stays `true`. The user remains
stuck, same as before this patch. An `img.onerror` handler could be added
to surface this, but that's a separate concern.

### Polling races

Between `img.src` reset and `onload`, the `setInterval(streamQuery)`
polling loop continues. These `CMD_QUERY` calls may:

- Fail (socket not found) → `getCmdResponse` sets `zmsBroke = true`
  (already true, no effect)
- Succeed (zms started between polls) → `getCmdResponse` sets
  `zmsBroke = false`, then `onload` fires and sets it `false` again
  (idempotent)

Both cases are safe. The `onload` callback executes in the same JS event
loop tick as the handler fires, so no AJAX response can interleave between
`zmsBroke = false` and `streamSeek(offset)`.

### Browser MJPEG `onload` behavior

For `multipart/x-mixed-replace` responses:

- **Chrome**: Fires `onload` for each MIME part (each frame)
- **Firefox**: Same behavior
- **Safari**: Same behavior

The `img.onload = null` after the first fire prevents repeated callback
invocations during normal streaming.

## Files Changed

- `web/skins/classic/views/js/event.js` — +24 / -9 lines

## Related Commits

- `011defdad` — harden vector bounds (prevents some zms crashes)
- `328d5c2a9` — sendFrame inside mutex (prevents race condition crashes)
- `0ca3ed315` — keepalive timer reset (prevents timeout-induced SIGPIPE)
- `4b098fb76` — binary search seek (built on top of this commit)
