# Auto-Prioritized Keyframe Video Generation — Design

**Date:** 2026-09-30 (revised 2026-10-08, v5 — API-only `autogen-engine`, per Spot review)
**Status:** Design — adopting PR #1 review feedback
**Home:** new **`autogen-engine`** repo (Node/TypeScript). Design doc lives in metarepo until the repo exists.

> **Revision note (v5) — API-only, à la `electric-sheep-engine`.** Spot's PR #1 review
> replaced the direct-DB/Redis design with one that runs as an **ordinary user account
> through the public API**. Changes from v4:
> - **Repo:** `autogen-engine` (Node/TS). Talks **only to the backend HTTP API** — no
>   direct DB, no admin role, no Redis.
> - **Auth:** regular (non-admin) **curation user** API key, header
>   **`Authorization: Api-Key <key>`** (v4 wrongly said `Api-Key: <key>`). The backend's
>   `HeaderAPIKeyStrategy` uses prefix `"Api-Key "` (`backend/src/clients/passport.client.ts`);
>   `edream_sdk` and `worker/src/marketing-cli.ts` already send exactly this from
>   Node — but there is **no JS/TS SDK**, so `autogen-engine` ships a small typed client.
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
videos between them through the public API — owned by the curation account, added to one
output playlist, visible immediately.

## Architecture

`autogen-engine` is a standalone Node/TS service on its **own Heroku app**, run by
**Heroku Scheduler** on a one-off dyno nightly (UTC). It holds **no persistent state of its
own** — all state is the backend's (the output playlist and the dreams themselves). It is a
thin client over a handful of API endpoints plus the pure selection logic.

```
Heroku Scheduler (nightly, UTC) → one-off dyno: node dist/index.js → run():
  1. in-flight guard  — GET /v1/dream/my-dreams?status=queue,processing (as curation user);
                        skip run if count > MAX_INFLIGHT
  2. template         — GET /v1/dream/:TEMPLATE_DREAM_UUID → base prompt (algo params)
  3. existing pairs   — GET playlist :PLAYLIST_UUID items → each item's prompt →
                        set of (source_dream_uuid,end_source_uuid) already made  [dedup + connectivity]
  4. candidate pool   — GET /v1/feed?mediaType=image (paged) → image dreams + playlistItems;
                        votes per image via GET /v1/dream/:uuid (nightly, one call each is fine)
  5. feature stand-in — GET /v1/feed/ranked → featured playlists' items → per-image feature score
  6. popularity()     [pure]  votes + playlist-adds + recency + feature, scaled 0–1 + floor
  7. drawPairs()      [pure]  weighted lottery, N draws, bias by connectivity k, skip self/dup
  8. for each drawn pair (unless DRY_RUN):
        prompt = buildLtxI2vPrompt(templatePrompt, start, end)   [pure]
        POST /v1/dream (Authorization: Api-Key)  → dreamUuid
        add dreamUuid to PLAYLIST_UUID
  9. log run summary (pool / drawn / created / skipped)
  → exit 0
```

## Integration edges (all public API; `Authorization: Api-Key <key>`)

| Capability | Endpoint (intended) | Notes |
|------------|--------------------|-------|
| In-flight guard | `GET /v1/dream/my-dreams?status=queue,processing` | count as the curation user |
| Template prompt | `GET /v1/dream/:TEMPLATE_DREAM_UUID` | copy `prompt`, force ltx-i2v |
| Existing pairs (state) | read `PLAYLIST_UUID` items + their prompts | dedup + connectivity |
| Candidate pool + signals | `GET /v1/feed?mediaType=image` (paged) | playlistItems inline; votes per-dream |
| Votes per image | `GET /v1/dream/:uuid` | one call/image, nightly — acceptable |
| Feature stand-in | `GET /v1/feed/ranked` | featured playlists → position → score |
| Create generation | `POST /v1/dream` | owner = curation account |
| Add to playlist | add-item endpoint | every generated dream |

The exact exists-vs-build status of each endpoint is settled in the Plan; capabilities not
already exposed to a regular user are **built as initial backend APIs** as part of this work.

## Signals (what a regular user can see)

| Signal | Source |
|--------|--------|
| playlist adds | feed returns each dream's `playlistItems` |
| upvotes / downvotes | `GET /v1/dream/:uuid` |
| recency | dream `created_at`/`updated_at` |
| featureRank (stand-in) | position in `/v1/feed/ranked` featured playlists → `1/(1+position)`; **0** if in none; keep weight low (coverage likely thin, mostly videos) |

Reported images are **not** excluded. nsfw/hidden exclusion is whatever the feed returns for
the calling user. Plays/watch-through deferred (client #691).

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

**Purity/determinism:** the draw takes an **injected RNG** (seedable) so tests are
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
- **In-flight guard = `my-dreams`.** Count the curation account's `queue`/`processing`
  dreams via the API; skip the run if over `MAX_INFLIGHT`.

## Config

`BACKEND_API`, `API_KEY`, `TEMPLATE_DREAM_UUID`, `PLAYLIST_UUID`, `DRY_RUN` (default `true`),
`JOBS_PER_RUN`, `CONNECTIVITY_K`, signal weights (`W_VOTES`/`W_PLAYLIST`/`W_RECENCY`/`W_FEATURE`),
`HALF_LIFE_DAYS`, `MAX_INFLIGHT`. (Removed from v4: `DATABASE_URL_RO`, `REDIS_URL`,
`CURATION_USER_UUID`, template project/transition keys.) Entry requires `BACKEND_API`,
`API_KEY`, `TEMPLATE_DREAM_UUID`, `PLAYLIST_UUID` or it exits non-zero without creating anything.

## Deployment & rollout

- **Heroku app `autogen-engine`**, no web dyno; **Heroku Scheduler** runs `node dist/index.js`
  nightly (UTC). One-off dyno; no persistent local state (ephemeral FS is fine).
- **Rollout (config flips):** (1) **dry-run** (default) — score/draw/log, create nothing;
  (2) **capped live** — `DRY_RUN=false`, `JOBS_PER_RUN=2`, eyeball the curation account's
  output; (3) **full** — raise `JOBS_PER_RUN`, leave on schedule.

## Testing

- **Pure** (`popularity`, `drawPairs`, `buildLtxI2vPrompt`): unit tests — scaling + floor,
  connectivity weighting, redraw on self/dup/already-made, N honored, deterministic with an
  injected RNG; prompt forces ltx-i2v + sets source/end and preserves template params.
- **`run()` orchestration:** unit-tested with a fake API client — in-flight guard skip, dry-run
  creates nothing, dedup against playlist, caps, template-fetch fail-fast, each created dream
  added to the playlist.
- **API client + any new backend endpoints:** backend endpoint tests follow the repo's
  existing controller-test pattern; the `autogen-engine` client is tested with a mocked HTTP
  layer (correct `Authorization: Api-Key` header, request/response shapes).

## Risks / open decisions

1. **Curation account** (regular user) provisioning + its API key; the output `PLAYLIST_UUID`
   and the `TEMPLATE_DREAM_UUID` (an existing ltx-i2v dream) must be created/recorded.
2. **Content-safety sign-off**: curation-owned, instantly-public video from other users'
   images. (Reported images are *not* filtered per review — confirm that's intended.)
3. **featureRank stand-in coverage** may be thin (featured playlists are mostly videos) —
   measure in the dry run; keep `W_FEATURE` low or drop it.
4. **API cost:** one `GET /v1/dream/:uuid` per candidate image for votes — fine nightly, but
   if the pool is large consider a feed field or a batch endpoint (a Plan decision).
5. **Backend API additions** (step 8 of the Plan) must not regress existing feed/dream
   behavior for other callers.

## Deferred / future

- **Watch-through signal** — client [issue #691](https://github.com/e-dream-ai/client/issues/691):
  report how far a dream was watched + skipped-vs-finished. Strongest quality signal; once
  captured and exposed via the API, add a `plays` term to `popularity(d)`.
- Backend **ranked-signals endpoint** that returns the candidate pool + votes in one call
  (removes the per-image vote fetch).
- Re-generation policy (cooldown/score-rise) if manual playlist-removal proves insufficient.
- A JS/TS SDK (if more Node services appear).
