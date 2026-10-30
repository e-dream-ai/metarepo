# Client fake server and test harness: design

Tracking issue: [client#733](https://github.com/e-dream-ai/client/issues/733). Date: 2026-10-30.

## Goal

Run the real client (Mac, Windows, Linux) against a local server that we fully control, so that scripts, not a person at the keyboard, can reproduce timing, network and remote-control bugs. Priorities, in order:

1. **Crashes and hangs.** Detect them, and capture evidence automatically: the stack sample, the log and the request log.
2. **Timing stress.** Control latency and bandwidth for every channel (API, socket, media), not only error injection, so we can push the client's timing and look for pauses.
3. **Stutters and dropped frames**, read from client logs. This needs more client logging (Phase 3).

Out of scope for now: the sign-in flow (magic link, WorkOS, refresh). The harness authenticates with an API key that the server accepts unconditionally.

## Architecture

```
            ┌──────────────────────────── fake server (one Python process, one port) ──────────────────────────────┐
 runner ───►│ /control/*   scenario + fault + profile API, event log                                                │
 (pytest)   │ /api/v1/client/*   hello, playlist, dream, url, quota, dislikes, telemetry   ◄── shaped by "api" profile │
            │ /socket.io/  Engine.IO v4 + Socket.IO /remote-control (hand-rolled)            ◄── "socket" profile     │
            │ /media/<uuid>  mp4 bytes, Range/206                                            ◄── "media" profile +    │
            │                                                                                    per-dream faults      │
            └───────────────────────────────▲────────────────────────────────────────────────────────────────────┘
                                            │ http:// + ws://
 runner ── launches ──► client binary  (INFINIDREAM_DATA_DIR=<tmp>, INFINIDREAM_API_KEY=test)
        ── watches ───► process, <tmp>/Logs/*.log, server event log
        ── on failure ► stack sample, crash report, logs → artifacts/<run-id>/
```

- **Language:** Python 3.11+ with `asyncio` and `aiohttp` (HTTP and WebSocket server). It's pure Python, so it runs on all three OSes and in CI.
- **One port.** The client's server setting is a single `host:port`, and the media URLs returned by `/url` point back at the same server. It binds `127.0.0.1` by default, or `0.0.0.0` in LAN mode, so a Mac-hosted server can drive a Windows or Linux client.
- **Home:** the client repo, under `Tests/fake-server/`. It replaces the old `Tests/mock-websocket-server`, which predates the Socket.IO protocol.

### Client prerequisites (from #733)

1. Accept a scheme given in `settings.content.server` (`http://…` → plain HTTP, and plain `ws://` in `sio_beast.cpp`), log a warning that the connection is unencrypted, and over plain http send only the env API key, never a stored real login.
2. `INFINIDREAM_DATA_DIR`, so that settings, cache and logs live in a temp directory per run.
3. **A 1 Hz render heartbeat log line (new).** One `info` line per second from the render thread: current and next dream, frame index, queue depth, buffering/paused/transition state, and the wall-clock interval since the last tick. This is the most reliable hang signal, and the starting point for Phase 3. See "Hang detection" for why the other signals aren't enough.

The harness can start before (1) and (2) land, by having a developer point a Debug build at the server by hand. It can't run unattended until they do.

## Fake API: what the client actually depends on

From the contract audit (client master @ 5a5b471b, backend `origin/stage` @ eeb94b6):

| Endpoint | Shape the client needs | Notes for the fake |
|---|---|---|
| `GET /client/hello` | `{success, data:{quota:int, dislikesCount:int, quotaExpiresAt?, currentPlaylistUUID?}}` | A missing `quota` or `dislikesCount` → "malformed hello"; 4xx → client signs out; 5xx → 60 s backoff ×3. |
| `GET /client/playlist/{id}` | `data.playlist.{name, artist, nsfw, timestamp, loops, transitionTime, contents:[{uuid, timestamp, start_keyframe?, end_keyframe?}]}` | Replay a recorded fixture. |
| `GET /client/playlist/default` | `data.playlist` is a **bare array** of `{uuid, timestamp}` | |
| `POST /client/dream` `{uuids:[…]}` and `GET /client/dream?uuids=` | `data.dreams[]` | Types matter: `size` and `fps` are **strings**, `frames` is an int, and `status` must be `"processed"`. A wrong type is silently defaulted, which makes a good fault. |
| `GET /client/dream/{id}/url` | `{success, data:{url}}` | Return `http://<self>/media/<uuid>?t=<issued>`. The client treats the URL as opaque. |
| `GET /client/quota` | `data.quota`, `quotaExpiresAt?` | Below 100 MiB the client stops prefetching, so this doubles as the quota fault. |
| `GET /client/user/dislikes` | `data.dislikes: [uuid]` | |
| `POST /client/telemetry` | ignored | Recorded: `md5-mismatch` and `video-invalidated` events are useful assertions. |

Auth: accept anything. The client sends a bare `Api-Key:` header. (Side finding: stage's `requireAuth` expects `Authorization: Api-Key …`, and the socket has no API-key path at all, so the client's env-key path probably doesn't work against the real backend. Worth checking separately.)

Every response gets `Edream-*` request headers logged, and passes through the API shaping profile.

## Socket: hand-rolled Engine.IO v4

The client uses one transport (WebSocket), one namespace (`/remote-control`), text frames only, and a small set of packets (`0` open, `2`/`3` ping/pong, `40` connect, `42` event, `44` error). About 150 lines of Python cover that.

I'm choosing hand-rolled over `python-socketio` because chaos testing needs to break the protocol on purpose:
- delay the namespace-connect ack past the client's 20 s timeout
- starve pings
- send malformed or mistyped events
- drop the socket mid-frame

A compliant library makes those hard or impossible.

**Server → client:**
- `new_remote_control_event {event, uuid?, frameNumber?}` for every command: `play_dream`, `play_playlist`, `next`, `previous`, `forward`, `backward`, `pause`, `like_current_dream`, `set_speed_N`, …
- `quota_update`
- `"Error"` and `"Validation error"`

**Client → server, recorded with timestamps:**
- `ping` and `state_sync`, every 30 s and on most commands. All `state_sync` values are strings.
- `new_remote_control_event` with `playing` / `finished` (including `reason`, `frame`, `frames`, `seconds`) / `like` / `dislike` / `report`
- `goodbye`

**Known client crashes to turn into the first regression tests:**
- `new_remote_control_event` with no `event` (null dereference)
- `frameNumber` that isn't an integer (bad cast)

The real backend validates both, so production risk is low, but the client shouldn't trust the server.

## Network shaping: latency and bandwidth for every channel

This is the core of your comment on #733. Each channel (`api`, `socket`, `media`) has a **profile**, which per-dream and per-request rules can override:

| Knob | Meaning |
|---|---|
| `latency_ms` | delay before the response headers (time to first byte) |
| `jitter_ms` | uniform random addition to the latency, from a seeded RNG |
| `bandwidth_bps` | throughput cap for the body, enforced by a token bucket and written in chunks of ~16 KiB |
| `burst_bytes` | how much can go out at full speed before the cap applies (models TCP slow start or a CDN burst) |
| `schedule` | a piecewise function over time, e.g. `[(0s, 5MB/s), (20s, 150KB/s), (60s, 5MB/s)]`. Models a link that degrades and recovers. |
| `stall` | pause for T s at byte offset X, or at time t. T=∞ stalls forever. |

On top of the per-channel profiles sits a **shared link**: one token bucket that *all* concurrent media transfers draw from. It models a household connection where the prefetcher and the player's own downloads compete. That's exactly what we saw on 2026-10-30: prefetching ran at ~0.8 MB/s while two player downloads were active, and at ~4 MB/s once they stopped. Without a shared link, every connection gets the full rate, which hides the real bug pattern.

Socket shaping applies latency (and optional jitter) per message in each direction. Bandwidth doesn't matter for the socket.

Profiles can be changed at runtime through `/control`, so a scenario can play at 5 MB/s and then drop to 100 KB/s just as it presses `forward`.

Named presets: `lan` (unshaped), `home` (2.5 MB/s shared, 30 ms), `slow` (500 KB/s, 150 ms), `stage-worker` (1.2 MB/s with stalls; the conditions under which #728 was seen), `flaky` (`home` plus random 2–10 s stalls).

## Faults (beyond timing)

All faults can be targeted at one dream, at the Nth request, at Range requests only, or at a percentage of requests.

**Media:**
- HTTP 403 (expired signed URL), 404, 500, 503
- drop the connection at offset X or time T
- truncated body with the correct `Content-Length`
- wrong or missing `Content-Length`
- ignore `Range` and return 200 with the full body
- a bad `Content-Range`
- slow headers
- redirect chains (the client follows up to 5)

**API:**
- 4xx/5xx per endpoint
- a malformed body or wrong field types (`fps` as a number, `size` as a number, `frames` as a string, `status` ≠ processed)
- a wrong `md5`
- a playlist `timestamp` bump, to force a metadata refetch
- `quota` dropping to 0 mid-session

**Socket:**
- disconnect, then refuse to reconnect for T
- delay the CONNECT ack beyond 20 s
- stop sending pings
- the malformed events above
- a burst of N commands within M ms
- duplicate commands

**Process level (driven by the runner):**
- quit or kill the client mid-download
- relaunch on a dirty data directory (leftover `.tmp` files)

## Fixtures

A fixture set is a directory:

```
fixtures/<set>/
  manifest.json          # source server, date, playlist ids, per-dream {uuid, md5, size, source}
  api/playlist_<id>.json # raw /client/playlist response
  api/dreams.json        # raw /client/dream batch response for every dream in the set
  media/<uuid>.mp4
```

**Getting fixtures.** You suggested copying from alpha and stage; this covers that.
- `fixtures.py snapshot --server api-alpha.infinidream.ai --playlist <id> --set pinkfloyd` calls the same client endpoints the real client uses, saves the raw responses, then fetches each `/url` and downloads the mp4 while checking its md5. Recording raw client responses, not going through the SDK or admin API, means the fixtures have exactly the shapes and types the client parses.
  - Caveat: `/url` uses up quota on the real account. Use a test account or an admin with refillable quota.
  - Caveat: auth needs whatever header the real backend accepts (`Authorization: Api-Key`, see above).
- `fixtures.py import-cache <data-dir>` builds a set from a client's existing cache with no network at all. The client already saves raw `{success, data}` responses under `content/json/`, and the mp4s are under `content/mp4/`. The 544 MB currently cached on the dev Mac becomes the first set.
- `fixtures.py synth` generates tiny clips (with `Tests/generate_test_videos.sh`) plus matching metadata, for fast CI runs and for edge cases like 1-frame dreams, an odd fps, or a huge `frames` value.

**Where fixtures live.** The mp4s stay out of git; the default location is `~/edream-fixtures/`, overridable by env. Each `manifest.json` (uuids and md5s) is committed, so a set can be re-fetched by name. Fixtures can be **overridden** at runtime, e.g. by a scenario that serves dream X with a wrong md5, with no edits to the files.

## Runner: launching, watching, verdicts

A `pytest` plugin plus a small library.

1. **Launch.**
   - Make `tmp/<run-id>/data/`.
   - Write `settings.json` there, with `settings.content.server = http://127.0.0.1:<port>`, logging on, and quota-relevant settings.
   - Start the client binary with `INFINIDREAM_DATA_DIR` and `INFINIDREAM_API_KEY=test`. Binary paths per OS come from config; on Mac the harness runs `infinidream.app/Contents/MacOS/infinidream` directly.
2. **Watch** three sources, merged into one timeline:
   - the process
   - the client log (tailed, line-buffered)
   - the server event log (requests with bytes, how each transfer ended, and socket messages each way)
3. **Drive** through the remote-control socket. The scenario API looks like this:
   ```python
   s.play_dream(D)
   s.wait_log("DownloadRegistry: attach", dream=D, timeout=30)
   s.shape("media", dream=D, bandwidth="200KB/s")
   s.send("forward")
   s.wait_playing(timeout=20)
   ```
4. **Verdicts:**
   - **Crash:** the process exits or is killed by a signal.
     - Mac: collect `~/Library/Logs/DiagnosticReports/infinidream*.ips`.
     - Windows: launch under `procdump -e -ma`, or collect WER dumps.
     - Linux: core dump, then `gdb -batch -ex "bt full"`.
   - **Hang:** see the next section. On a hang, sample the stacks (`sample` / `procdump` + `cdb` / `gdb -p`), then kill the client.
   - **Invariant violations**, checked at the end and continuously:
     - no two concurrent downloads of the same dream
     - every cached `.mp4` matches its fixture md5
     - no `.tmp` files survive a clean quit
     - after all faults are cleared, playback resumes within N s
   - **Artifacts:** `artifacts/<run-id>/` holds the client log, server event log, stacks or crash report, the scenario's action log, and the seed.

### Hang detection

There are three layers, because each existing signal misses something:

| Signal | Catches | Misses |
|---|---|---|
| Socket silence: no `ping` for >45 s (the client pings every 30 s) | networking or process wedge | a frozen render thread while networking is fine, which is exactly #728 ("only pings") |
| `state_sync` timecode not advancing between two syncs while `paused=false` | a frozen player | it's slow (30 s), and `state_sync` is mostly sent on commands |
| **Render heartbeat** (client prerequisite 3): no heartbeat for >3 s, or heartbeat frame index not advancing for >N s while not paused or buffering | a wedged render or decode loop, quickly | — |

Buffering that never ends counts as a hang too: the heartbeat says "buffering" for longer than the fault scenario allows.

## Chaos mode

```
chaos.py --set pinkfloyd --seed 1234 --duration 30m --profile flaky
```

- A seeded RNG chooses from three pools:
  - actions: next, previous, forward, backward, play_dream(random), play_playlist, pause/unpause, speed, quit+relaunch
  - shaping changes: profile switches, schedule ramps
  - faults
- Each action is logged with a timestamp. Hang detection and invariants run continuously.
- On failure the harness saves the seed and the action log. `chaos.py --replay <run-id>` re-runs the exact sequence. Shrinking a failing sequence to a minimal one (delta debugging) comes later.

Expected first catches: the #728 class (a seek onto a shared download that dies), the duplicate-download registry gap, rebuffer thrash, and the malformed-event crashes.

## Phase 3: making every dropped or delayed frame visible

This comes after crashes and hangs. The logging audit found the gaps; this is the minimum client work, roughly in priority order:

1. **Render heartbeat** (also the hang signal): 1 Hz, always on. Per second it reports ticks, frames popped, discarded, and repeated, queue depth for the current and next clip, the maximum tick interval, and states.
2. **Wall-clock durations.** The playback timeline freezes during a buffering pause, so today `m_TotalBufferingTime` is about 0 and is never read. Add `steady_clock` durations to:
   - "Buffer refilled" (rebuffer duration)
   - initial buffering
   - "Buffering complete, unpausing" (pause duration, plus which clip caused it)
3. **Frames skipped to catch up.** `CClip::Update` → `DiscardFrames` count. Today it's only a `Debug` line, which the log filter drops.
4. **Repeated frames.** The queue was empty (`CClip::Update` returns false), or the clip is drawing the last valid frame while rebuffering. Count both; there are no counters today.
5. **Late or clamped ticks.** In `BeginFrameUpdate`, flag when the raw delta is more than 1.5× the display interval, and when it's clamped at 0.1 s; the clamp silently loses timeline time.
6. **Crossfade stalls.** `UpdateTransition` silently restarts the transition at three spots while the next clip is preloading or buffering, and the next clip isn't included in `IsAnyClipBuffering`. Log the entry and exit with a duration.
7. **Role tags.** Tag "Buffer too low" lines as current or next clip.
8. **Present backpressure.** Time Metal's semaphore wait, D3D11 `Present`, and Vulkan `vkQueuePresentKHR`.

Then `analyze.py <log>` turns a run into a stutter report: stalls longer than 100 ms with their cause (network rebuffer, decode slow, present wait, crossfade starvation), plus counts of drops and repeats per minute, aligned with the server's shaping timeline. So a run can say, for example, "at 12:03:41 bandwidth dropped to 150 KB/s; 2.1 s later the current clip rebuffered for 3.4 s."

## Milestones

| # | Deliverable | Done when |
|---|---|---|
| 0 | Client prerequisites: scheme override + plain `ws://`, `INFINIDREAM_DATA_DIR`, render heartbeat | A release build runs against `http://127.0.0.1` with a temp data directory |
| 1 | Fake API, socket and media server (unshaped); fixtures via `import-cache` and `snapshot`; runner launch, watch, crash and hang verdicts with artifacts | A scripted "play a playlist, next ×5, quit" passes on Mac; a scripted malformed event reproduces the null-dereference crash with a crash report |
| 2 | Shaping (per-channel profiles, shared link, schedules, stalls) and media/API/socket faults; #728 scenario | The #728 scenario runs unattended. If it reproduces, we have stacks; if not, it runs as a regression test. The duplicate-download invariant is checked. |
| 3 | Chaos mode with seeds and replay | Runs for 30 minutes unattended and produces artifacts on failure |
| 4 | Frame-level logging (Phase 3) and `analyze.py` | Stutter report for a throttled run |
| 5 | Windows and Linux runner support; Linux headless CI (Xvfb + lavapipe) | The same scenarios pass on all three |

## Open questions

- **A test account for `snapshot`**, with a refillable quota, on alpha and stage.
- **Is the heartbeat always on, or behind a flag?** At 1 line/s it's ~86k lines a day. I suggest always on but compact; the log already rotates daily.
- **CI hardware for the Mac** (hardware decode and Metal can't run headless on a hosted runner). Probably a dedicated Mac mini.
