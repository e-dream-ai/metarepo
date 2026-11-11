# Client mock server and test harness: design

Tracking issue: [client#733](https://github.com/e-dream-ai/client/issues/733). Date: 2026-10-30.

## Goal

Run the real client (Mac, Windows, Linux) against a local server that we fully control, so that scripts, not a person at the keyboard, can reproduce timing, network and remote-control bugs. Priorities, in order:

1. **Crashes and hangs.** Detect them, and capture evidence automatically: the stack sample, the log and the request log.
2. **Timing stress.** Control latency and bandwidth for every channel (API, socket, media), not only error injection, and load the machine's CPU, so we can push the client's timing and look for pauses.
3. **Stutters and dropped or late frames**, read from the client's heartbeat and judged against a frame budget, not only noted (see "Verdicts for frames").

Out of scope for now: the sign-in flow (magic link, WorkOS, refresh). The harness authenticates with an API key that the server accepts unconditionally.

## Architecture

```
            ┌──────────────────────────── mock server (one Python process, one port) ──────────────────────────────┐
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
- **Home:** the client repo, under `client_generic/tests/mock-server/`. It replaces the old `client_generic/tests/mock-websocket-server`, which predates the Socket.IO protocol.

### Client prerequisites (from #733)

1. Accept a scheme given in `settings.content.server` (`http://…` → plain HTTP, and plain `ws://` in `sio_beast.cpp`), log a warning that the connection is unencrypted, and over plain http send only the env API key, never a stored real login.
2. `INFINIDREAM_DATA_DIR`, so that settings, cache and logs live in a temp directory per run.
3. **A render heartbeat log line (new).** One `info` line from the render thread, every 60 s by default. The harness sets `INFINIDREAM_HEARTBEAT_SECONDS=1` to get 1 Hz, and the 60 s default keeps logs small on users' machines. Each line has: current and next dream, frame index, queue depth, buffering/paused/transition state, and the wall-clock interval since the last tick. This is the most reliable hang signal, and the starting point for Phase 3. See "Hang detection" for why the other signals aren't enough.

The harness can start before (1) and (2) land, by having a developer point a Debug build at the server by hand. It can't run unattended until they do.

## Mock API: what the client actually depends on

From the contract audit (client master @ 5a5b471b, backend `origin/stage` @ eeb94b6):

| Endpoint | Shape the client needs | Notes for the mock |
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

## Machine load: CPU

Network speed is one way to starve the client; a busy machine is another. Users run the client beside builds, browsers and games, and a screensaver often starts while something heavy is still running in the background. CPU load challenges the decoder threads, the render thread's timing and the download threads.

**The load generator** is a separate helper process, `load.py`, started and controlled by the runner (and from chaos schedules):

| Knob | Meaning |
|---|---|
| `threads` | number of busy worker processes (processes, not Python threads, so the GIL doesn't cap them) |
| `duty` | fraction of each 10 ms period each worker spins, 0–1 |
| `nice` | the workers' priority; default the same as the client, since a real competing app gets no handicap |
| `schedule` | piecewise over time, like the network schedule: e.g. `[(0s, 0), (20s, all cores at 90%), (60s, 0)]` |
| `pattern` | `steady`, `bursts` (on/off with a random period from the `load` stream), or `ramp` |

Named presets: `idle`, `moderate` (half the cores at 50%), `heavy` (all cores at 90%), `oversubscribed` (twice the core count at 100%), `bursty` (all cores, random 0.5–5 s bursts).

Details that matter:
- **Measure what was achieved.** The generator logs the load it actually produced each second (from `os.times()` or `getrusage`), and the runner logs the client's own CPU use. Both go into the event timeline beside the heartbeats, so a report can say "late ticks rose when load reached 90%".
- **Apple Silicon has performance and efficiency cores.** The machine's P and E core counts are recorded with the run, and `heavy` is defined as the total number of cores so the client's threads can't simply escape to idle cores.
- **Thermals.** Long heavy runs slow the CPU over time, so a 30-minute run is not the same machine at minute 1 and minute 30. The runner records `pmset -g therm` (Mac) at the start and the end.
- **Inside the client.** `INFINIDREAM_TEST_STALL_MS` / `_EVERY_MS` already stall the render thread directly. A matching decoder-side stall (sleep in the frame-reading thread) would test the `short` and `rebuffers` counters, which outside load and throttling can't target precisely.

Later, the same helper can add memory pressure (allocate and touch N GB), disk contention (heavy writes on the cache volume) and GPU load (a Metal compute or render job). CPU comes first.

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

The client limits how long a slow download can last: `ProgressiveHttpStream` sets `CURLOPT_LOW_SPEED_LIMIT` to 1 KB/s and `CURLOPT_LOW_SPEED_TIME` to 30 s. So a "forever" stall, or throttling below 1 KB/s, becomes a client abort after about 30 s, not a wedged download. Scenarios have to expect that abort, and a hang scenario like #728 has to keep watching well past 30 s, or it tests only the abort path.

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

**Today** (2026-11-11) the tests serve whatever is in the alpha client's cache (`/Users/Shared/infinidream.ai/content/mp4`), choosing the smallest or largest files with at least 900 frames. That got the tests going, but the results depend on what's cached on that machine: the throttle test broke when larger videos arrived. We need to **own the test videos**: known content, known properties, and the same bytes on every machine and in every replay.

**Everything comes from alpha (decision, 2026-11-11).** The client only ever plays alpha's encodings (the video service transcodes every upload to HEVC, `-crf 24`, at most 4K), so codecs, bitrates and keyframe spacing outside that pipeline don't need testing, and no new storage is needed. A controlled set:
- **Is a playlist on alpha**, owned by a test account and kept out of shared and default playlists, so users don't come across it by accident. It holds two kinds of dream:
  - **Numbered dreams** that we generate and upload: each frame shows its number in **large Arabic numerals** for people, and the same number as a **barcode strip** for the harness. Each dream also shows its name and has its own background color, so a wrong dream or a crossfade is obvious to people and to the harness. At least 900 frames each, so a dream can't end by itself during a test, plus a few short ones (1 frame, a few seconds) for edge cases. Variety in resolution and length comes from the generator.
  - **Existing alpha dreams** chosen for the range the pipeline really produces: short and long, low resolution to 4K, small and large files, and the usual frame rates.
- **Is pinned by a manifest committed to git:** each dream's uuid, md5, size, frames, fps and resolution, as alpha serves them.
- **Is fetched once per machine** by `fixtures.py fetch <set>`, through the same client API the client uses (playlist, dream info, `/url`), with a test account's key. Files go to `~/edream-fixtures/<set>/`, and every md5 is checked. After that, tests and replays read only the local copy, so they don't depend on alpha being up or unchanged. If alpha re-processes or deletes a dream, the md5 check fails at fetch time and the set gets a new version.
- **Checks the pipeline on upload.** The numbered dreams' encoded copies must have the frame count and fps we generated. If the pipeline drops frames or changes the rate, the numbers on screen won't match the heartbeat, and the fetch step reports it.

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
- `fixtures.py numbered` generates the numbered dreams (ffmpeg: numerals with `drawtext`, barcode bits with `drawbox`, a color and name per dream) for upload to alpha with `edream_sdk`. Metadata edge cases that need no upload, like a huge `frames` value or `fps` as a number, are served as overrides by the mock server.

**Where fixtures live.** The mp4s stay out of git; the default location is `~/edream-fixtures/`, overridable by env. Each `manifest.json` (uuids and md5s) is committed, so a set can be re-fetched by name. Fixtures can be **overridden** at runtime, e.g. by a scenario that serves dream X with a wrong md5, with no edits to the files.

## Runner: launching, watching, verdicts

A `pytest` plugin plus a small library.

1. **Launch.**
   - Make `tmp/<run-id>/data/`.
   - Write `settings.json` there, with `settings.content.server = http://127.0.0.1:<port>`, logging on, and quota-relevant settings.
   - Start the client binary with `INFINIDREAM_DATA_DIR`, `INFINIDREAM_API_KEY=test` and `INFINIDREAM_HEARTBEAT_SECONDS=1`. Binary paths per OS come from config; on Mac the harness runs `infinidream.app/Contents/MacOS/infinidream` directly, and on Windows it runs the windowed `.exe`, not the `.scr`.
   - The key has to come from the env var: over `http` the client ignores `api_key` and `sealed_session` in `settings.json`. Don't set `settings.generator.nickname`, because magic link is tried before the API key. Do set `settings.app.firsttimesetup` to true: otherwise the Mac client hides the player view and playback never starts.
2. **Wait until ready.** Don't drive a scenario until the mock server has seen the client's `hello` and its socket CONNECT on `/remote-control`. The server sees both itself, so this doesn't depend on parsing client logs. Never becoming ready within a timeout is a failed run, reported as such.
3. **Watch** three sources, merged into one timeline:
   - the process
   - the client log (tailed from the log file, which is flushed on every line; the console output arrives in delayed chunks when redirected to a file)
   - the server event log (requests with bytes, how each transfer ended, and socket messages each way)
4. **Drive** through the remote-control socket. The scenario API looks like this:
   ```python
   s.play_dream(D)
   s.wait_log("DownloadRegistry: attach", dream=D, timeout=30)
   s.shape("media", dream=D, bandwidth="200KB/s")
   s.send("forward")
   s.wait_playing(timeout=20)
   ```
5. **Quit.** A clean quit needs a new client feature: there's no `quit` remote-control event today, and in testing SIGTERM simply killed the Mac app (exit -15), so neither can support the "no `.tmp` files" invariant below. Add a `quit` event that the client honors **only when `INFINIDREAM_DATA_DIR` is set**, so a server can never quit a real user's app. It works the same on all three OSes.
6. **Verdicts:**
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
     - the picture on screen isn't black or frozen during playback, and holds still while paused (see "Reading the screen")
   - **Artifacts:** `artifacts/<run-id>/` holds the client log, server event log, stacks or crash report, the scenario's action log, and the seed.

### Hang detection

There are three layers, because each existing signal misses something:

| Signal | Catches | Misses |
|---|---|---|
| Socket silence: no `ping` for >45 s (the client pings every 30 s) | networking or process wedge | a frozen render thread while networking is fine, which is exactly #728 ("only pings") |
| `state_sync` timecode not advancing between two syncs while `paused=false` | a frozen player | it's slow (30 s), and `state_sync` is mostly sent on commands |
| **Render heartbeat** (client prerequisite 3): no heartbeat for >3 s, or heartbeat frame index not advancing for >N s while not paused or buffering | a wedged render or decode loop, quickly | — |

Buffering that never ends counts as a hang too: the heartbeat says "buffering" for longer than the fault scenario allows.

### Verdicts for frames

Crashes and hangs aren't the only failures: we are also looking for **dropped and late frames**. The 1 Hz heartbeat already reports them (client#748): `late_ticks` (ticks longer than 1.5 display intervals), `clamped_ticks` and `lost_ms` (ticks clamped at 100 ms, and the timeline time that lost), and for each clip `discarded` (frames skipped to catch up), `short` and `rebuffers`.

The harness judges them against a **frame budget**, set by the conditions in force at the time:

| Conditions | Budget |
|---|---|
| Unshaped network, idle machine, steady playback | Zero discards, zero clamped ticks; late ticks only in the first few seconds after launch and around a command. Anything else fails. |
| Network starved below the dream's rate | Rebuffers are expected. Fails if the render loop stalls (heartbeat gap), if a rebuffer outlasts its scenario's limit, or if discards keep growing after the network recovers. |
| CPU load | Measured, not passed or failed at first: for each load level, record late ticks per minute, discards per minute and the longest tick. Once there's a baseline, a later build that does clearly worse fails. |
| Around a command or crossfade | A short allowance per event, set from measurements. |

Every budget failure is reported with the time, the heartbeat lines around it, and what the network and load were doing then, from the merged timeline. Known bugs that inflate the counts (#747: each resume after a rebuffer discards a frame and clamps a tick) are noted so they aren't counted twice, and the budget tightens when they're fixed.

The heartbeat sees what the client meant to draw. To confirm what reached the screen, use frame-number clips and, later, readback inside the client (see "Reading the screen").

## Reading the screen

The heartbeat reports what the client thinks it drew. Reading pixels checks that independently. There are three levels, in the order to build them:

1. **Screenshots about once a second** (working, Mac only). `screencapture -l` on the client's window catches a black or frozen picture. It needs Screen Recording permission for the terminal; without it the checks are skipped, and the test says so. Some dreams are legitimately dark, and crossfades can pass through dark frames, so "black" means near-zero brightness for several seconds. A paused client measured a difference of 0.00 between screenshots and a slow-moving dream about 0.5, so the threshold is 0.2.
2. **Frame numbers in the numbered dreams.** Each frame shows its number in large numerals and as a barcode (see "Fixtures"). The harness reads the barcode, so it knows which frame is showing instead of guessing from the picture; people watching or looking at a saved screenshot read the numerals. Skips, repeats, out-of-order frames and wrong dreams all become visible. These clips are tiny and need no hosting. At screenshot rate this already catches a wrong dream or a stuck clip.
3. **Readback inside the client, on test runs only.** This gives per-frame accuracy, since an outside capture tool can drop frames itself at 30-60 fps. It needs code per renderer (Metal first) and must not slow rendering, or it causes the stutters being measured.

A locked screen stops rendering, so rule that out first. Hosted Mac CI can't run Metal, so these checks need the dedicated Mac mini or Linux under Xvfb.

## Chaos mode

```
chaos.py --set pinkfloyd --seed 1234 --duration 30m --profile flaky --load moderate
```

- A seeded RNG chooses from four pools:
  - actions: next, previous, forward, backward, play_dream(random), play_playlist, pause/unpause, speed, quit+relaunch
  - shaping changes: profile switches, schedule ramps
  - faults
  - machine load: CPU load changes (see "Machine load")
- Hang detection, invariants and frame budgets (see "Verdicts for frames") run continuously.

### Making a failure replayable

A seed reproduces what the harness does, not what the client does: the client's download, decode and render threads run on real time, so the same inputs can take a different path. So the seed is necessary but not enough:

1. **Generate the whole schedule up front.** Before launching the client, the seed generates a list of `(when, action, params)` covering commands, shaping changes, faults and load changes. It's written to `schedule.json` in the run folder. The seed is printed on the first line of output, and on failure the harness prints a copy-paste replay command.
2. **Replay from the file, not the seed.** `chaos.py --replay <run>/schedule.json` runs exactly those actions, even after the generator has changed. `--seed N` alone regenerates the schedule and is only for convenience. Because the schedule is plain JSON, a person (or a shrinker) can delete actions and replay to find what matters.
3. **Isolated random streams.** All harness randomness lives in the Python harness and mock server, which are separate processes from the client, so nothing the client does can change it. Inside the harness, nothing uses the shared `random` module's state. Each stream (`commands`, `shaping`, `media-faults`, `api-faults`, `socket-faults`, `load`, `jitter`) gets its own `random.Random` seeded from a hash of `(seed, stream name)`. Adding a new fault type then adds a stream without shifting the others, so old seeds keep producing the same commands.
4. **The client's own randomness.** The client picks dreams with `rand()` and with `mt19937` generators seeded from `random_device` (`PlaylistManager.cpp`), and picks splash images with `rand()`. A test-only `INFINIDREAM_TEST_SEED`, honored only when `INFINIDREAM_DATA_DIR` is set, gives those choices their own seeded generator. Without it, the client behaves exactly as now. The harness derives this seed from the run seed too (stream `client`).
5. **Pin everything else.** `schedule.json` also records the client build (commit, and Debug or Release), the harness commit, the fixture set with each video's uuid and md5, and the machine (model, OS, core counts). Replay warns on any mismatch. This is why chaos needs controlled videos (see "Fixtures"): a schedule that names videos from someone's local cache can't be replayed elsewhere, or even on the same machine once the cache changes.
6. **Anchor timing to client events where it matters.** "800 ms after the client reports `playing` for dream D" replays closer than "12.4 s after launch", because startup time varies from run to run. The schedule supports both: `at` (seconds since launch) and `after` (an event from the server log or the heartbeat, plus an offset).
7. **Replay many times and report the hit rate.** `--replay ... --repeat 20` gives "crashed 4 of 20". For a race, that's the useful result, both to confirm a bug and later to confirm the fix.
8. **Capture everything on the first hit.** The failure may never reproduce, so every failed run keeps the client log, the server's `events.jsonl`, the crash report or stack sample, the schedule, the client's exit status and the load measurements.
9. **Shrink later.** Delta debugging: remove halves, then smaller chunks, of a failing schedule, replaying each candidate several times, until the smallest schedule that still fails is left. At about a minute per run this is an overnight job, so it comes after the basic replay works.

### Crash detection prerequisite

On master, the Mac client catches **SIGSEGV** in `ESScreensaverView.mm` and calls `exit(0)`. A segfault then leaves no crash report and looks like a clean exit (client#769). Until the client stops catching SIGSEGV, the harness also treats `RECEIVED SIGSEGV` in the log as a crash. A test-run exit by any signal other than the SIGTERM the harness sent is a crash too.

Expected first catches: the #728 class (a seek onto a shared download that dies), the duplicate-download registry gap, rebuffer thrash, the malformed-event crashes, and frame drops under CPU load.

## Phase 3: making every dropped or delayed frame visible

This comes after crashes and hangs. The logging audit found the gaps; this is the minimum client work, roughly in priority order:

1. **Render heartbeat** (also the hang signal): always on, every 60 s by default and 1 Hz under the harness. Each line reports ticks, frames popped, discarded, and repeated, queue depth for the current and next clip, the maximum tick interval, and states.
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
| 0 | (client#735) Client prerequisites: scheme override + plain `ws://`, `INFINIDREAM_DATA_DIR`, render heartbeat | A release build runs against `http://127.0.0.1` with a temp data directory |
| 1 | Mock API, socket and media server (unshaped), starting from the basic server and end-to-end test already in `client_generic/tests/mock-server/`; fixtures via `import-cache` and `snapshot`; runner launch, ready gate, watch, crash and hang verdicts with artifacts; test-only `quit` event | A scripted "play a playlist, next ×5, quit" passes on Mac; a scripted malformed event reproduces the null-dereference crash with a crash report |
| 2 | Shaping (per-channel profiles, shared link, schedules, stalls) and media/API/socket faults; #728 scenario | The #728 scenario runs unattended. If it reproduces, we have stacks; if not, it runs as a regression test. The duplicate-download invariant is checked. Reconnect over plain `ws://` works after a server-side drop; #735 only tested connecting. |
| 2b | Controlled video sets on alpha: generate numbered dreams (numerals + barcode), upload them to a test account's playlist, add chosen existing dreams, `fixtures.py fetch` with md5 checks; tests stop reading the local cache | The e2e and throttle tests pass on a machine with an empty client cache, and a screenshot's barcode matches the heartbeat's frame number |
| 2c | CPU load generator (`load.py`) with presets and schedules; frame budgets as verdicts | A run under `heavy` load produces a per-level report of late ticks and discards; an unloaded run that discards a frame fails |
| 3 | Chaos mode: schedule file, isolated random streams, `--replay` with `--repeat`, client test seed, crash detection that sees swallowed SIGSEGVs | Runs for 30 minutes unattended; a failing run's printed replay command re-runs the same schedule and reports a hit rate |
| 4 | Frame-level logging (Phase 3) and `analyze.py`. The heartbeat counters are done (client#748); still needed are wall-clock rebuffer durations, crossfade stalls, present backpressure, and the report | Stutter report for a throttled or loaded run |
| 4b | Shrinking failing chaos schedules | A failing 30-minute schedule shrinks overnight to a few actions |
| 5 | Windows and Linux runner support; Linux headless CI (Xvfb + lavapipe) | The same scenarios pass on all three |

## Open questions

- **A test account for `snapshot`**, with a refillable quota, on alpha and stage.
- **CI hardware for the Mac** (hardware decode and Metal can't run headless on a hosted runner). Probably a dedicated Mac mini.
