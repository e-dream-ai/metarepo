# Auto-Prioritized Keyframe Video Generation — Design

**Date:** 2026-09-30 (revised 2026-10-08, v5 — API-only `autogen-engine`, per Spot review)
**Status:** Design — adopting PR #1 review feedback
**Home:** new **`autogen-engine`** repo (**Python**), reusing `edream_sdk` + the engines-style
CLI/batch infra. Design doc lives in metarepo until the repo exists.

> **Revision note (v5) — API-only, Python, à la `engines`/`electric-sheep-engine`.** Spot's
> PR #1 review replaced the direct-DB/Redis design with one that runs as an **ordinary user
> account through the public API**. Changes from v4:
> - **Repo/stack:** `autogen-engine`, **Python**, reusing **`edream_sdk`** (python-api) and
>   the engines-style batch-script/CLI infra. Talks **only to the backend HTTP API** — no
>   direct DB, no admin role, no Redis.
> - **Auth:** the **`autogen`** account's API key. `edream_sdk` already authenticates with the
>   backend's `HeaderAPIKeyStrategy` (`Authorization: Api-Key <key>`, prefix `"Api-Key "` —
>   `backend/src/clients/passport.client.ts`), so no bespoke client is needed; extend
>   `edream_sdk` for any endpoint it doesn't yet wrap (feed `mediaType`, playlist items,
>   ranked feed, `my-dreams` status).
> - **Account = `autogen`, provisioned per env.** A dedicated **`autogen`** user (non-admin
>   **creator** — `POST /v1/dream` requires `CREATOR_GROUP`) owns everything it generates and
>   supplies the API key. A **provisioning script** creates this user + issues its API key in
>   each environment (dev/stage/prod); no route change needed. See Plan.
> - **Template = a dream, not a flow project.** `TEMPLATE_DREAM_UUID` → `GET /v1/dream/:uuid`
>   returns its `prompt`; copy it, force `infinidream_algorithm:"ltx-i2v"`, set new
>   `source_dream_uuid`/`end_source_uuid`, `POST /v1/dream`. Drop `EditorProject`/transition-key.
> - **State lives in the output playlist, not Redis.** `PLAYLIST_UUID`; every generated
>   dream is added to it; at run start read the playlist's items' prompts — the
>   `(source_dream_uuid, end_source_uuid)` pairs there are both the **dedup** set and the
>   **connectivity** signal. Remove a video from the playlist = manual "redo". No Redis,
>   no TTLs; cooldown/score-rise dropped for v1.
> - **Selection: weighted lottery** (see §Selection) instead of hub-and-spoke.
> - **Signals via API only;** `featureRank` is stripped for non-admins — approximate via
>   `/v1/feed/ranked` position (low weight; coverage may be thin).
> - **Input filter:** all processed image dreams from any source; nsfw/hidden come from
>   the feed as the calling user sees it; **do not** exclude reported images.
> - **This plan also builds the initial backend APIs** `autogen-engine` needs (any
>   feed/dream/playlist capability not already exposed to a regular user) — see Plan.
> - **Deferred signal:** client [issue #691](https://github.com/e-dream-ai/client/issues/691)
>   (report end-of-playback: how far through + skipped vs. finished) replaces the vaguer
>   "% played-through."

> **Terminology.** A "keyframe" here = **a still image Dream used as an i2v source/target**
> (`mediaType = "image"`), not the `Keyframe` entity.

## Purpose

Grow the library unattended. A nightly job ranks still image dreams by engagement, draws
pairs via a weighted lottery biased toward already-connected images, and creates LTX i2v
videos between them through the public API — owned by the `autogen` account, added to one
output playlist, visible immediately.

## Architecture

`autogen-engine` is a standalone **Python** service (reusing `edream_sdk` + the engines-style
batch infra) on its **own Heroku app**, run by **Heroku Scheduler** on a one-off dyno nightly
(UTC). It holds **no persistent state of its own** — all state is the backend's (the output
playlist and the dreams themselves). It is a thin client over a handful of API endpoints plus
the pure selection logic. It runs as the **`autogen`** user via that account's API key.

```
Heroku Scheduler (nightly, UTC) → one-off dyno: python -m autogen_engine → run():
  0. jitter           — sleep(random(0, STARTUP_JITTER_SECONDS)); all calls go through a
                        shared rate limiter (MAX_RPS, sequential) + backoff-on-429/5xx
  1. in-flight guard  — GET /v1/dream?userUUID=<AUTOGEN_USER_UUID>&status=queue, then
                        &status=processing (status is a single enum → two calls); sum counts;
                        skip run if sum > MAX_INFLIGHT
  2. template         — GET /v1/dream/:TEMPLATE_DREAM_UUID → base prompt (algo params)
  3. existing pairs   — GET playlist :PLAYLIST_UUID items → each item's prompt →
                        set of (source_dream_uuid,end_source_uuid) already made  [dedup + connectivity]
  4. candidate pool   — GET /v1/feed?type=dream&mediaType=image (paged, ≤ MAX_FEED_PAGES) →
                        image dreams w/ playlistItems, status, created_at (feed omits votes +
                        prompt); keep only status == processed. (Ordered created_at DESC — new
                        inserts can shift offsets between pages; tolerable for a capped sweep.)
  4a. shortlist       — rank by cheap feed signals (playlist-adds + recency), take top
                        SHORTLIST_SIZE  [bounds the vote fetches below]
  4b. votes           — GET /v1/dream/:uuid for the SHORTLIST only → upvotes/downvotes
  5. feature stand-in — GET /v1/feed/ranked (playlists) → walk each ranked playlist's items
                        (extra GET /:uuid/items calls) → per-image score by ORDER (numeric
                        featureRank stripped; order kept). Optional: set W_FEATURE=0 to skip
                        this fan-out entirely (coverage is thin — featured content is mostly
                        videos)
  6. popularity()     [pure]  votes + playlist-adds + recency + feature, scaled 0–1 + floor
  7. drawPairs()      [pure]  weighted lottery, N draws, bias by connectivity k, skip self/dup
  8. for each drawn pair (unless DRY_RUN):
        prompt = buildLtxI2vPrompt(templatePrompt, start, end)   [pure]
        POST /v1/dream (as autogen, Authorization: Api-Key)  → dreamUuid
        add dreamUuid to PLAYLIST_UUID (batch add at end of run)
  9. log run summary (pool / drawn / created / skipped)
  → exit 0
```

## Integration edges (all public API; `Authorization: Api-Key <key>`)

| Capability | Endpoint | Status |
|------------|----------|--------|
| In-flight guard | `GET /v1/dream?userUUID=<AUTOGEN_USER_UUID>&status=queue` + `&status=processing` (single-status enum → two calls; `/my-dreams` has no status filter) | **exists** |
| Template prompt | `GET /v1/dream/:TEMPLATE_DREAM_UUID` | **exists** |
| Existing pairs (state) | `GET /v1/playlist/:PLAYLIST_UUID/items` — items `leftJoinAndSelect` the full dream incl. `prompt` | **exists** (no follow-up calls) |
| Candidate pool | `GET /v1/feed?mediaType=image` (paged) — returns `playlistItems`, `status`, `created_at`/`updated_at`, `user` (**not** votes, **not** prompt); filter to `status == processed` client-side | **exists** |
| Votes per image | `GET /v1/dream/:uuid` — `upvotes`/`downvotes` | **exists** (one call/candidate, nightly) |
| Feature stand-in | `GET /v1/feed/ranked` — featured playlists in `featureRank` order (numeric value stripped; order kept) | **exists** |
| Create generation | `POST /v1/dream` | **exists, but requires `CREATOR_GROUP`** → `autogen` is a creator account |
| Add to playlist | `POST /v1/playlist/:uuid/items` (atomic batch ≤500) | **exists** |

**Exists-vs-build (from the `origin/stage` API survey):** every capability above is already
served to a regular/creator user — so the backend work is **not** new query endpoints. The
only backend/ops items are: (a) a **provisioning script** that creates the `autogen` creator
account + its API key per env (which is also what satisfies the `CREATOR_GROUP` requirement
on `POST /v1/dream`), and (b) extending **`edream_sdk`** to wrap the endpoints it doesn't yet
cover (feed `mediaType`, playlist items, ranked feed, dream-status filter). A purpose-built
ranked-signals endpoint is deferred (the feed already carries the signals).

## Signals (what a regular user can see)

| Signal | Source |
|--------|--------|
| playlist adds | feed returns each dream's `playlistItems` (count the array) |
| upvotes / downvotes | per-image `GET /v1/dream/:uuid` (feed does **not** include votes) |
| recency | dream `created_at`/`updated_at` (in the feed) |
| featureRank (stand-in) | position in `/v1/feed/ranked` featured playlists → `1/(1+position)`; **0** if in none; keep weight low (coverage likely thin, mostly videos) |

Reported images are **not** excluded. nsfw/hidden exclusion is whatever the feed returns for
the calling user. Plays/watch-through deferred (client #691).

### Candidate visibility — how `autogen` sees other users' dreams

The main feed is a **global public gallery for any (non-admin) user**, not scoped to the
caller. In `backend/src/utils/feed.util.ts`, `buildFeedConditions()` OR's two dream
conditions for a non-admin: (1) `hidden: true` **AND `user = caller`** (your own unlisted
items), and (2) `hidden: false` **with no `user` filter** — i.e. *everyone's* non-hidden
dreams. So `autogen`, as an ordinary account, sees all other users' non-hidden image dreams;
it need not own anything. Narrowing:

- **`hidden: true`** on another user's dream → excluded (unlisted stays out). ✓
- **`nsfw`** → excluded unless the `nsfw` option is set (leave it off). ✓
- **`status`** → the feed applies **no** server-side status filter, so `autogen` must keep
  only `status == processed` client-side (only finished stills are renderable i2v sources).
- **`FeedItem` requirement** → a dream surfaces only if it has a `FeedItem` row (the feed
  joins `FeedItem → dreamItem`). Any processed public image lacking one is invisible to the
  pool — the dry run's reported pool size will expose any such gap.
- **Self-exclusion** → `autogen` drops candidates owned by its own account
  (`user.uuid == AUTOGEN_USER_UUID`). Its output is video (never an image candidate), so this
  is defensive — a guard against self-referential selection / feedback loops, not a load-bearing
  filter.

## Selection — weighted lottery (pure)

For the candidate images `d`:

1. `popularity(d)` = weighted sum of `votes`, `playlistAdds`, `recency`, `feature`, each
   min-max scaled to [0,1] within the run, **plus a small floor** so every image has a
   chance.
2. `connected(d)` = how many times `d` already appears as a start or end in the output
   playlist's pairs.
3. `w(d) = popularity(d) × (1 + k · connected(d))`.
4. For each of `N` draws: pick a start and an end with probability ∝ `w`. **Redraw** if
   `start == end` or the `(start,end)` pair is already in the playlist. If no fresh valid pair
   is found within a bounded `max_attempts` (pool of 1, or everything already made), **stop and
   return fewer than `N` (possibly 0)** — never loop. The floor is applied **before** the 0–1
   normalization, so an all-zero-signal set still yields positive weights (no divide-by-zero).

`k` biases toward images already used, so videos link into a connected graph; the floor +
randomness give long-tail images a chance (this replaces the v4 "lowest-scored
different-owner" diversity pick). Tunables: `N` (`JOBS_PER_RUN`), `k` (`CONNECTIVITY_K`),
signal weights, `HALF_LIFE_DAYS`.

**Purity/determinism:** the draw takes an **injected `random.Random`** (seeded) so tests are
deterministic; production seeds from the clock at run start.

## Prompt building (pure)

`buildLtxI2vPrompt(templatePrompt, startUuid, endUuid)`: start from the template dream's
`prompt` object, force `infinidream_algorithm: "ltx-i2v"`, set `source_dream_uuid = start`
and `end_source_uuid = end`, and leave the template's other params (prompt text, duration,
guidance, loras, seed) intact. (The worker reads guidance as `guidance` and uses
`high_noise_loras[0]`.) **Startup validation:** the template dream's prompt must already be
ltx-i2v-shaped — if its `infinidream_algorithm` isn't `ltx-i2v`, fail-fast rather than
force-set it onto non-ltx params (which would enqueue jobs the worker can't run).

The created dream's **`name`** is derived deterministically, e.g. `autogen <start8>→<end8>`
(first 8 chars of each source uuid), so output is identifiable in the `autogen` account.

## Dedup & in-flight (no extra storage)

- **Dedup = the output playlist.** The set of `(source,end)` pairs already present (from step
  3) is the dedup set; a drawn pair in that set is redrawn. Delete a dream from the playlist
  to allow its pair to regenerate (manual redo). No cooldown/score-rise in v1. **When reading
  playlist items, skip any whose prompt lacks both `source_dream_uuid` and `end_source_uuid`**
  (a hand-added or non-i2v item) rather than erroring.
- **In-flight guard.** There is no single "my dreams by status" endpoint: `/my-dreams` is
  caller-scoped but has no status filter, and `GET /v1/dream` filters by a `userUUID` param
  with a **single** status enum. So count the `autogen` account's non-terminal dreams with two
  calls — `GET /v1/dream?userUUID=<AUTOGEN_USER_UUID>&status=queue` and `…&status=processing` —
  and sum. `AUTOGEN_USER_UUID` is emitted by the provisioning script. Skip the run if the sum
  exceeds `MAX_INFLIGHT`.

## Being a good API citizen (v1 load management)

Until Phase C moves scoring server-side, `autogen-engine` is a nightly client making many
reads + a burst of writes against the shared API server. v1 keeps that gentle:

- **Startup jitter.** Heroku Scheduler fires on fixed boundaries (:00), so every scheduled job
  across the fleet would hit at once. At run start, `sleep(random(0, STARTUP_JITTER_SECONDS))`
  (default up to 300s) before the first call, so this run lands off the minute/hour.
- **Client-side rate limit.** All API calls go through one pacer — a token bucket / min-interval
  (`MAX_RPS`, default low, e.g. 2–4 req/s) so we never fan out bursts. Calls are **sequential**
  (concurrency 1); no parallel floods.
- **Two-stage candidate narrowing (also the big call-count win).** Page the feed for the cheap
  signals it *does* return (playlist-adds + recency), rank those, take the top `SHORTLIST_SIZE`
  (e.g. 200), and fetch per-image **votes only for the shortlist** — so vote calls are bounded
  by `SHORTLIST_SIZE`, not catalog size. Also cap feed paging at `MAX_FEED_PAGES`.
- **Spread the writes.** `POST /v1/dream` also enqueues a GPU job, so creates are paced by the
  same limiter (not bursted), and the `MAX_INFLIGHT` guard caps how much work is outstanding.
- **Backoff + jitter on errors.** On `429`/`5xx`, retry with exponential backoff + jitter,
  honoring `Retry-After`; give up after `RETRY_MAX` and log. Never hot-loop.

**Why not BullMQ here?** A broker is overkill for one nightly batch — and BullMQ is Node, while
`autogen-engine` is Python. In-process pacing + jitter + backoff gives the politeness without
new infra. A delayed/scheduled queue (RQ/Celery, or driving it from the Node worker) only earns
its keep if we later want to **drip generation across the day** instead of one nightly run — a
future option, noted, not v1.

## Config

`BACKEND_API`, `API_KEY`, `AUTOGEN_USER_UUID`, `TEMPLATE_DREAM_UUID`, `PLAYLIST_UUID`,
`DRY_RUN` (default `true`), `JOBS_PER_RUN`, `CONNECTIVITY_K`, signal weights
(`W_VOTES`/`W_PLAYLIST`/`W_RECENCY`/`W_FEATURE`), `HALF_LIFE_DAYS`, `MAX_INFLIGHT`, and the
politeness knobs `STARTUP_JITTER_SECONDS` (300), `MAX_RPS` (3), `SHORTLIST_SIZE` (200),
`MAX_FEED_PAGES` (20), `RETRY_MAX` (4). (Removed from v4: `DATABASE_URL_RO`, `REDIS_URL`,
`CURATION_USER_UUID`, template project/transition keys.) `API_KEY` + `AUTOGEN_USER_UUID` are the
`autogen` account's key and user uuid, both emitted by the provisioning script (below);
`PLAYLIST_UUID` is the output playlist for the env. Entry requires `BACKEND_API`, `API_KEY`,
`AUTOGEN_USER_UUID`, `TEMPLATE_DREAM_UUID`, `PLAYLIST_UUID` or it exits non-zero without creating
anything; and it **fail-fast exits** if `TEMPLATE_DREAM_UUID` or `PLAYLIST_UUID` 404s, or if the
template dream's prompt is not ltx-i2v-shaped (see Prompt building).

## Provisioning the `autogen` account (per environment)

A **provisioning script** (run once per env — dev/stage/prod) creates the dedicated `autogen`
user as a non-admin **creator** (so `POST /v1/dream` is permitted), issues its **API key**,
records its **user uuid** (→ `AUTOGEN_USER_UUID`, needed for the in-flight guard), and
ensures an **output playlist** exists; it prints the `API_KEY` + `PLAYLIST_UUID` to configure
`autogen-engine`. It lives where user + API-key + role creation is possible (a backend
`src/script/*` admin script, following the `cron:default-playlist` script pattern). This is the
only backend-side work — no new query endpoints are needed (the API survey confirmed the rest
exists for a creator account).

## Deployment & rollout

- **Heroku app `autogen-engine`**, no web dyno; **Heroku Scheduler** runs `python -m autogen_engine`
  nightly (UTC). One-off dyno; no persistent local state (ephemeral FS is fine).
- **Rollout (config flips):** (1) **dry-run** (default) — score/draw/log, create nothing;
  (2) **capped live** — `DRY_RUN=false`, `JOBS_PER_RUN=2`, eyeball the `autogen` account's
  output; (3) **full** — raise `JOBS_PER_RUN`, leave on schedule.

## Testing (pytest)

- **Pure** (`popularity`, `drawPairs`, `build_ltx_i2v_prompt`): unit tests — scaling + floor,
  connectivity weighting, redraw on self/dup/already-made, `N` honored, deterministic with a
  seeded `random.Random`; prompt forces ltx-i2v + sets source/end and preserves template params.
- **`run()` orchestration:** unit-tested with a fake `edream_sdk` client — in-flight guard
  skip, dry-run creates nothing, dedup against playlist, caps, template-fetch fail-fast, every
  created dream added to the playlist.
- **`edream_sdk` extensions:** test any newly-wrapped endpoints (feed `mediaType`, playlist
  items, ranked feed, dream-status filter) with a mocked HTTP layer (correct
  `Authorization: Api-Key` header, request/response shapes).
- **Provisioning script:** idempotency (re-run finds the existing `autogen` user, doesn't
  duplicate), following the backend repo's script/controller-test conventions.

## Risks / open decisions

1. **`autogen` account provisioning** (the per-env script) + its API key; the output
   `PLAYLIST_UUID` and the `TEMPLATE_DREAM_UUID` (an existing ltx-i2v dream) must be
   created/recorded per env.
2. **Content-safety sign-off**: `autogen`-owned, instantly-public video from other users'
   images. (Reported images are *not* filtered per review — confirm that's intended.)
2b. **Discovery pollution** (DECISION): should `autogen`'s content appear in the global feed /
   ranked-trending / search, or be excluded from those surfaces (visible only via its playlist +
   direct link)? High output volume means fully-public would likely drown human creators. See
   Backend API improvements #5; leans toward excluding `autogen` from discovery.
3. **featureRank stand-in coverage** may be thin (featured playlists are mostly videos) —
   measure in the dry run; keep `W_FEATURE` low or drop it.
4. **API cost:** the pool is paged from `/v1/feed?mediaType=image` (adds + status inline), then
   **one `GET /v1/dream/:uuid` per candidate** for votes. Fine nightly, but bound the pool
   (cap pages / pre-filter by adds+recency before fetching votes) for a large catalog; a
   future ranked-signals endpoint would remove the per-image fetch.
5. **`edream_sdk` extensions** must not change behavior for existing SDK consumers (engines,
   video, electric-sheep-engine).

## Backend API improvements (follow-on, not required for v1)

v1 composes existing endpoints, which is enough nightly but has rough edges. These are
backend changes to make *later* — each is an API-server task, decided with the backend team:

1. **Votes are omitted from the feed — likely on purpose.** `getFeedDreamItemSelectedColumns`
   returns no `upvotes`/`downvotes`; raw counts may be withheld deliberately (anti-gaming,
   or they're surfaced differently in-product). v1 reads them via per-dream `GET /v1/dream/:uuid`
   (which does expose them) — but **do not widen the feed to return votes** without confirming
   that's acceptable. The better path is to keep votes server-side and never ship raw counts to
   clients at all (see #2).
2. **A dedicated candidate/signals endpoint.** A regular-user `GET /v1/autogen/candidates`
   (or `GET /v1/dream/ranked-stills`) that returns processed, non-hidden image dreams already
   scored/ordered server-side — folding in votes, playlist-adds, recency, and featureRank in
   **one query**, with `status = processed` enforced server-side. Removes: the per-image vote
   fetch, client-side status filtering, deep feed pagination, and the need to expose raw votes.
3. **`FeedItem` coverage.** Confirm every processed public image has a `FeedItem` row (else it is
   invisible to the pool); if not, either backfill or have the candidate endpoint query dreams
   directly rather than through `FeedItem`.
4. **Batch vote lookup** as a cheaper interim than #2 (e.g. `POST /v1/dream/votes` with a list of
   uuids) if the full endpoint is more than we want now.
5. **Keep `autogen` out of discovery surfaces.** Because output is public + instantly visible,
   the `autogen` account's high volume would otherwise dominate the **global feed, ranked/
   trending, and search** that humans see, and skew engagement signals. Options (backend, a
   product call — see Risks): exclude the `autogen` userId from the default/ranked feed + search
   (treat it as a system/curation account, reachable only via its playlist + direct link); or
   mark its dreams so clients can filter; or keep output `hidden` until curated (conflicts with
   "visible immediately"). `autogen` does **not** vote, so it adds no vote-skew directly; the
   issue is its *content* appearing in everyone's discovery.

These feed directly into the **server-side scoring phase** (plan Phase C): once the backend can
rank candidates, the heavy read/compute moves off `autogen-engine`.

## Deferred / future

- **Server-side scoring (plan Phase C).** Move `popularity`/selection into the API server so
  `autogen-engine` stops paging the feed and fetching per-image votes — far fewer round-trips,
  no raw-vote exposure, one SQL pass instead of N calls. `autogen-engine` becomes a thin
  trigger (ask the backend for ranked pairs, then create + add to playlist), or the backend owns
  the whole loop. Gets its own spec/plan.
- **Watch-through signal** — client [issue #691](https://github.com/e-dream-ai/client/issues/691):
  report how far a dream was watched + skipped-vs-finished. Strongest quality signal; once
  captured and exposed, add a `plays` term to `popularity(d)`.
- Re-generation policy (cooldown/score-rise) if manual playlist-removal proves insufficient.
- Promoting `autogen`'s `edream_sdk` additions into the shared SDK for other consumers.
