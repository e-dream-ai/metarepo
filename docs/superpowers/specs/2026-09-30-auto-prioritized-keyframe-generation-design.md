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
  1. in-flight guard  — GET /v1/dream?status=queue,processing (the autogen account's own);
                        skip run if count > MAX_INFLIGHT
  2. template         — GET /v1/dream/:TEMPLATE_DREAM_UUID → base prompt (algo params)
  3. existing pairs   — GET playlist :PLAYLIST_UUID items → each item's prompt →
                        set of (source_dream_uuid,end_source_uuid) already made  [dedup + connectivity]
  4. candidate pool   — GET /v1/feed?mediaType=image (paged) → image dreams w/ playlistItems,
                        status, created_at (feed does NOT return votes or prompt);
                        keep only status == processed (feed has no server-side status filter)
  4b. votes           — GET /v1/dream/:uuid per candidate → upvotes/downvotes (nightly: OK)
  5. feature stand-in — GET /v1/feed/ranked → featured playlists' items → per-image feature
                        score by ORDER (numeric featureRank is stripped; list order is kept)
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
| In-flight guard | `GET /v1/dream?status=queue,processing` (own dreams) | **exists** |
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

## Selection — weighted lottery (pure)

For the candidate images `d`:

1. `popularity(d)` = weighted sum of `votes`, `playlistAdds`, `recency`, `feature`, each
   min-max scaled to [0,1] within the run, **plus a small floor** so every image has a
   chance.
2. `connected(d)` = how many times `d` already appears as a start or end in the output
   playlist's pairs.
3. `w(d) = popularity(d) × (1 + k · connected(d))`.
4. For each of `N` draws: pick a start and an end with probability ∝ `w`. **Redraw** if
   `start == end` or the `(start,end)` pair is already in the playlist.

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
`high_noise_loras[0]`.)

## Dedup & in-flight (no extra storage)

- **Dedup = the output playlist.** The set of `(source,end)` pairs already present (from step
  3) is the dedup set; a drawn pair in that set is redrawn. Delete a dream from the playlist
  to allow its pair to regenerate (manual redo). No cooldown/score-rise in v1.
- **In-flight guard.** Count the `autogen` account's own `queue`/`processing` dreams via
  `GET /v1/dream?status=queue,processing`; skip the run if over `MAX_INFLIGHT`.

## Config

`BACKEND_API`, `API_KEY`, `TEMPLATE_DREAM_UUID`, `PLAYLIST_UUID`, `DRY_RUN` (default `true`),
`JOBS_PER_RUN`, `CONNECTIVITY_K`, signal weights (`W_VOTES`/`W_PLAYLIST`/`W_RECENCY`/`W_FEATURE`),
`HALF_LIFE_DAYS`, `MAX_INFLIGHT`. (Removed from v4: `DATABASE_URL_RO`, `REDIS_URL`,
`CURATION_USER_UUID`, template project/transition keys.) `API_KEY` is the `autogen` account's
key, emitted by the provisioning script (below); `PLAYLIST_UUID` is the output playlist for
the env. Entry requires `BACKEND_API`, `API_KEY`, `TEMPLATE_DREAM_UUID`, `PLAYLIST_UUID` or it
exits non-zero without creating anything.

## Provisioning the `autogen` account (per environment)

A **provisioning script** (run once per env — dev/stage/prod) creates the dedicated `autogen`
user as a non-admin **creator** (so `POST /v1/dream` is permitted), issues its **API key**, and
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
3. **featureRank stand-in coverage** may be thin (featured playlists are mostly videos) —
   measure in the dry run; keep `W_FEATURE` low or drop it.
4. **API cost:** the pool is paged from `/v1/feed?mediaType=image` (adds + status inline), then
   **one `GET /v1/dream/:uuid` per candidate** for votes. Fine nightly, but bound the pool
   (cap pages / pre-filter by adds+recency before fetching votes) for a large catalog; a
   future ranked-signals endpoint would remove the per-image fetch.
5. **`edream_sdk` extensions** must not change behavior for existing SDK consumers (engines,
   video, electric-sheep-engine).

## Deferred / future

- **Watch-through signal** — client [issue #691](https://github.com/e-dream-ai/client/issues/691):
  report how far a dream was watched + skipped-vs-finished. Strongest quality signal; once
  captured and exposed via the API, add a `plays` term to `popularity(d)`.
- Backend **ranked-signals endpoint** that returns the ranked candidate pool in one call
  (optimization over paging the feed).
- Re-generation policy (cooldown/score-rise) if manual playlist-removal proves insufficient.
- Promoting `autogen`'s `edream_sdk` additions into the shared SDK for other consumers.
