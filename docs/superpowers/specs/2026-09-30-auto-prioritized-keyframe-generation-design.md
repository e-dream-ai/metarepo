# Auto-Prioritized Keyframe Video Generation — Design

**Date:** 2026-09-30 (revised 2026-10-22, v6 — Kling, manual CLI, per Spot's 2nd review)
**Status:** Design — adopting PR #1 review round 2
**Home:** new **`autogen-engine`** repo (**Python**), reusing `edream_sdk` + the engines-style
CLI/batch infra. Design doc lives in metarepo until the repo exists.

> **Revision note (v6) — Spot's 2nd review.** v5 was close; this applies his follow-up:
> - **Generate with Kling, not LTX.** Only Kling handles first+last frames properly (why the
>   flow app defaults to `kling-25-i2v`, Kling 2.5 Turbo Pro, 5 s). Pin `kling-25-i2v`; make
>   `TEMPLATE_DREAM_UUID` a **Kling** dream. The worker already maps `source_dream_uuid`/
>   `end_source_uuid` to Kling's start/end images, so the prompt shape is unchanged. Kling runs
>   on **fal, billed per video** → add a **per-run cost estimate + spending cap**.
> - **No provisioning script.** Spot creates the `autogen` account, its API key, and the output
>   playlist **by hand per env**. Drop the backend provisioning task and `AUTOGEN_USER_UUID`;
>   look up the key's own user at startup via `edream_sdk.get_logged_user()` (`GET /auth/user`).
> - **Extend `edream_sdk`** (python-api) for new endpoints, **general-purpose** (not
>   autogen-specific), instead of local wrappers. The SDK is pre-1.0; keep existing consumers
>   working, additive changes are fine.
> - **No rate limiter / jitter / retry.** A few hundred sequential calls isn't a load concern; a
>   failed call just fails the run and we re-run. Any future rate-limit/retry belongs in
>   `edream_sdk`, not here.
> - **v1 is a manual laptop CLI.** No Heroku app / Procfile / Scheduler. Run by hand
>   (`python -m autogen_engine`, config from env / `.env`), as often as we like while testing.
>   `DRY_RUN` still useful. Where it runs in the cloud + scheduling is out of scope.
> - **Images have no votes — credit from derived videos.** An image's popularity comes from the
>   engagement (votes, playlist-adds) of the **videos made from it** (those listing it as
>   `source_dream_uuid`/`end_source_uuid`). This needs a tracing step the public API can't do
>   cheaply yet (the feed omits `prompt`), so **v1 approximates** popularity from the image's own
>   playlist-adds + recency; the derived-video signal arrives with a general-purpose backend
>   endpoint (Deferred).
> - **No discovery exclusion** (Risk 2b resolved): `autogen` is treated like any regular user;
>   no hiding, no C0.
> - **Content safety** (Risk 2 resolved): restrict inputs to **SFW** via the feed's nsfw filter
>   (leave `nsfw` off → feed returns `nsfw:false` only).
> - **Backend endpoints:** v1 uses existing ones; new ones may come later but must be
>   **general-purpose** ("dreams derived from X"), not autogen-specific.

> **Terminology.** A "keyframe" here = **a still image Dream used as a start/end frame**
> (`mediaType = "image"`), not the `Keyframe` entity.

## Purpose

Grow the library. A CLI (run by hand in v1) ranks still image dreams by engagement, draws pairs
via a weighted lottery biased toward already-connected images, and creates **Kling i2v** videos
between them through the public API — owned by the `autogen` account, added to one output
playlist that also serves as dedup/connectivity state.

## Intent

- **Home/stack:** `autogen-engine`, **Python**, reusing `edream_sdk` + engines-style CLI infra.
- **Runtime (v1):** a **manual CLI** on a laptop; env / `.env` config; `DRY_RUN` default. Cloud +
  scheduling deferred.
- **Account:** a hand-created `autogen` user; its API key in config. The engine looks up its own
  user via `get_logged_user()` (no `AUTOGEN_USER_UUID`).
- **Generation:** **`kling-25-i2v`** between two image dreams (`source_dream_uuid` start,
  `end_source_uuid` end), created via `POST /v1/dream`; template params copied from a Kling
  `TEMPLATE_DREAM_UUID`.
- **Signals (v1):** playlist-adds + recency (+ optional `featureRank` stand-in). Image votes are
  ~absent; derived-video engagement is the target signal, pending a backend trace (Deferred).
- **Selection:** connectivity-biased weighted lottery (details are mine, per Spot).
- **Dedup/connectivity:** the output playlist (`PLAYLIST_UUID`) — its items' prompts give the
  `(source,end)` pairs already made.
- **Cost:** Kling is billed per video → a per-run estimate + a hard spending cap.

## Architecture

`autogen-engine` is a standalone **Python** CLI (reusing `edream_sdk`). v1 is **run by hand**;
it holds **no persistent state** — all state is the backend's (the output playlist + the dreams).
It talks **only to the public API** as the `autogen` user via that account's API key.

```
python -m autogen_engine   (manual; config from env/.env; DRY_RUN default true)
  0. who am I     — edream_sdk.get_logged_user() (GET /auth/user) → autogen's own user uuid
  1. in-flight    — GET /v1/dream?userUUID=<self>&status=queue, then &status=processing
                    (single-status enum → two calls); sum; skip run if > MAX_INFLIGHT
  2. template     — GET /v1/dream/:TEMPLATE_DREAM_UUID → base prompt; validate it IS kling-25-i2v
  3. existing     — GET /v1/playlist/:PLAYLIST_UUID/items → each item's prompt →
                    set of (source_dream_uuid,end_source_uuid) already made  [dedup + connectivity]
  4. pool         — GET /v1/feed?type=dream&mediaType=image (paged ≤ MAX_FEED_PAGES) → images w/
                    playlistItems, status, created_at (feed omits votes+prompt; SFW only);
                    keep status==processed; drop autogen-owned (self-exclusion)
  5. feature      — (optional, W_FEATURE>0) GET /v1/feed/ranked → walk ranked playlists' items →
                    per-image feature score by ORDER (featureRank value stripped; order kept)
  6. popularity() [pure]  v1 = playlist-adds + recency (+ feature); scaled 0–1 + floor
  7. draw_pairs() [pure]  weighted lottery w(d)=pop(d)*(1+k*connected(d)); skip self/dup/already
  8. cost guard   — n = min(JOBS_PER_RUN, floor(MAX_SPEND_PER_RUN / COST_PER_VIDEO)); log estimate
  9. for each of n pairs (unless DRY_RUN):
        prompt = build_kling_prompt(template_prompt, start, end)   [pure]
        POST /v1/dream (as autogen) → dreamUuid ; add to PLAYLIST_UUID (batch at end)
 10. log summary (pool / drawn / est_cost / created / skipped)
```

## Integration edges (public API only; `Authorization: Api-Key <key>` via `edream_sdk`)

| Capability | Endpoint | Status |
|------------|----------|--------|
| Identify self | `GET /auth/user` (`edream_sdk.get_logged_user()`) | **exists** |
| In-flight guard | `GET /v1/dream?userUUID=<self>&status=queue` + `&status=processing` | **exists** |
| Template prompt | `GET /v1/dream/:TEMPLATE_DREAM_UUID` | **exists** |
| Existing pairs (state) | `GET /v1/playlist/:PLAYLIST_UUID/items` — items include each dream's `prompt` | **exists** |
| Candidate pool | `GET /v1/feed?type=dream&mediaType=image` (paged) — `playlistItems`,`status`,`created_at` (no votes/prompt); SFW only; filter `status==processed` client-side | **exists** |
| Feature stand-in | `GET /v1/feed/ranked` — featured playlists in `featureRank` order (value stripped, order kept) | **exists** |
| Create generation | `POST /v1/dream` | **exists** (needs `CREATOR_GROUP` — the hand-made `autogen` account is a creator) |
| Add to playlist | `POST /v1/playlist/:uuid/items` (atomic batch ≤500) | **exists** |

**No backend changes for v1** — all existing. New endpoints may come later (Deferred) but must be
**general-purpose** (e.g. "dreams derived from uuid X"), with matching `edream_sdk` methods.

## Signals & popularity

**Images have essentially no votes of their own — only videos get voted on.** The *intended*
signal is therefore: an image's popularity = the engagement (votes + playlist-adds) of the
**videos derived from it** (videos whose prompt lists it as `source_dream_uuid`/`end_source_uuid`).
The public feed does **not** return `prompt`, so tracing those videos per image is not cheap today
(and some links exist only via `startKeyframe`/`endKeyframe`). **Confirm which apps write
`source_dream_uuid`** — the flow app does (`build-video-algo-params.ts`); others TBD.

- **v1 (no trace available):** `popularity(d)` = `w_playlist · playlistAdds(d)` + `w_recency ·
  recency(d)` (+ optional `w_feature · featureScore(d)`), each min-max scaled to [0,1] across the
  run + a floor. `playlistAdds` and `created_at` come straight from the feed — **no per-image
  calls**.
- **Later:** add a `w_votes` term fed by a general-purpose backend endpoint that returns dreams
  derived from a given uuid (Deferred / Phase C), so image popularity reflects its videos'
  engagement without client-side prompt scraping.

### Candidate visibility

The main feed is a **global public gallery for any user** (not caller-scoped):
`buildFeedConditions()` (`backend/src/utils/feed.util.ts`) OR's the caller's own `hidden:true`
dreams with **all `hidden:false` dreams, no owner filter** — so `autogen` sees every user's
non-hidden images. Narrowing: `hidden:true` (others' → excluded); **SFW only** (leave the feed's
`nsfw` option off → `nsfw:false` rows only — this is the content-safety control, Risk 2);
`status==processed` (client-side; the feed applies no status filter but returns `status`); a
`FeedItem` row must exist (dry-run pool size exposes gaps); and **self-exclusion** (drop
`user.uuid == self`). Reported images are **not** excluded.

## Selection — weighted lottery (pure)

1. `popularity(d)` — as above (0–1 scaled + floor).
2. `connected(d)` — times `d` appears as a start/end in the output playlist's pairs.
3. `w(d) = popularity(d) × (1 + k · connected(d))`.
4. For each of `N` draws: pick start + end ∝ `w`; **redraw** on `start==end` or a pair already in
   the playlist. If no fresh valid pair within a bounded `max_attempts` (pool of 1, everything
   made), **stop and return fewer than `N` (possibly 0)** — never loop. Floor applied **before**
   normalization (no divide-by-zero on all-zero signals).

`k` biases toward already-used images so videos link into a connected graph; floor + randomness
give long-tail images a chance. Tunables: `JOBS_PER_RUN` (`N`), `CONNECTIVITY_K` (`k`), weights,
`HALF_LIFE_DAYS`. Draw takes an injected `random.Random` (seeded) for deterministic tests.
(Selection/generation details are the implementer's call.)

## Prompt building (pure)

`build_kling_prompt(template_prompt, start_uuid, end_uuid)`: deep-copy the Kling template dream's
`prompt`, force `infinidream_algorithm: "kling-25-i2v"`, set `source_dream_uuid = start` and
`end_source_uuid = end`, leave the rest (prompt text, `duration`, `cfg_scale`, etc.) intact — the
worker maps the two source uuids to Kling's start/end frames. Accepts a dict or JSON-string prompt.
**Startup validation:** the template dream's prompt must already be `kling-25-i2v`; else fail-fast.
The created dream's `name` is deterministic, e.g. `autogen <start8>→<end8>`.

## Dedup & in-flight (no extra storage)

- **Dedup = the output playlist.** `(source,end)` pairs already present are the dedup set; a drawn
  pair in it is redrawn. Delete a dream from the playlist → its pair can regenerate (manual redo).
  When reading items, **skip any whose prompt lacks both source/end uuids** (hand-added/non-i2v).
- **In-flight guard.** No single "my dreams by status" endpoint (`/my-dreams` has no status
  filter; `GET /v1/dream` takes a `userUUID` param + single status). Count with two calls
  (`…&status=queue`, `…&status=processing`) using the uuid from `get_logged_user()`; skip the run
  if the sum exceeds `MAX_INFLIGHT`.

## Cost estimate & spending cap

Kling (fal) bills per video. Config carries a `COST_PER_VIDEO` estimate (USD). Each run computes
`estimated_spend = n × COST_PER_VIDEO` and **caps `n = min(JOBS_PER_RUN, floor(MAX_SPEND_PER_RUN /
COST_PER_VIDEO))`**. `DRY_RUN` logs the estimate and creates nothing. (This is a client-side guard;
the backend may also enforce its own credit limits on the `autogen` account.)

## Config

`BACKEND_API`, `API_KEY`, `TEMPLATE_DREAM_UUID` (a Kling dream), `PLAYLIST_UUID`, `DRY_RUN`
(default `true`), `JOBS_PER_RUN`, `CONNECTIVITY_K`, weights (`W_PLAYLIST`/`W_RECENCY`/`W_FEATURE`),
`HALF_LIFE_DAYS`, `MAX_INFLIGHT`, `COST_PER_VIDEO`, `MAX_SPEND_PER_RUN`, `MAX_FEED_PAGES`.
(Dropped vs v5: `AUTOGEN_USER_UUID` — use `get_logged_user()`; `W_VOTES` — v1 has no image votes;
`STARTUP_JITTER_SECONDS`/`MAX_RPS`/`RETRY_MAX`/`SHORTLIST_SIZE` — no limiter, no per-image vote
fetch.) The `autogen` account + its API key + the output playlist are created **by hand per env**.
Entry requires `BACKEND_API`, `API_KEY`, `TEMPLATE_DREAM_UUID`, `PLAYLIST_UUID`, else it exits
non-zero; also fail-fast if `TEMPLATE_DREAM_UUID`/`PLAYLIST_UUID` 404 or the template isn't
`kling-25-i2v`.

## Running it (v1)

A CLI run by hand: `python -m autogen_engine`, config from the shell env or a `.env` file, invoked
as often as we like while iterating. `DRY_RUN=true` (default) scores/draws/logs the estimated spend
and creates nothing; flip to `DRY_RUN=false` to generate. **No Heroku/Procfile/Scheduler in v1** —
once we're happy with the output, where it runs in the cloud and how it's scheduled is a separate,
later decision.

## Testing (pytest)

- **Pure** (`popularity`, `draw_pairs`, `build_kling_prompt`): scaling + floor, connectivity
  weighting, redraw/exhaustion (returns ≤ N, never loops), deterministic with a seeded
  `random.Random`; prompt forces `kling-25-i2v` + sets source/end + preserves template params.
- **`run()`** with a fake API client: in-flight skip, dry-run creates nothing (but logs the cost
  estimate), dedup vs playlist, self-exclusion + processed-only pool, cost cap bounds `n`,
  fail-fast on template/playlist 404 or non-Kling template, per-pair create failure continues,
  each created dream added to the playlist, playlist items missing source/end skipped.
- **`edream_sdk` additions** (in python-api): the new general-purpose methods, tested with a mocked
  HTTP layer (correct `Authorization: Api-Key`, request/response shapes), keeping existing
  consumers working.

## Risks / open decisions (Spot's answers folded in)

1. **Account provisioning** — resolved: created by hand per env (only two deployments).
2. **Content safety** — resolved: inputs restricted to **SFW** via the feed's nsfw filter.
3. **featureRank coverage** — thin (featured content is mostly videos); keep `W_FEATURE` low or 0.
   *Extension:* allow **any dream, including videos, as a start frame** by extracting a frame
   (thumbnail/filmstrip) — a promising general feature beyond autogen (Deferred).
4. **API cost / efficiency** — fine to start; if it gets bad, add general-purpose backend
   query/search endpoints + `edream_sdk` methods (≈ Phase C) — not planned in detail now.
5. **`edream_sdk` compatibility** — pre-1.0; additive changes are fine; keep existing consumers
   working unless something was genuinely wrong.
6. **Kling cost** — real; bounded by `MAX_SPEND_PER_RUN` + `DRY_RUN`. LTX would be cheaper but only
   branches from a single start frame (no two-image connection) — see Deferred.

## Deferred / future

- **Derived-video engagement signal + endpoint.** A general-purpose backend endpoint "dreams
  derived from uuid X" (by `source_dream_uuid`/`end_source_uuid`, and/or `startKeyframe`/
  `endKeyframe`) + an `edream_sdk` method, so image popularity reflects its videos' votes/adds
  without client-side prompt scraping. Confirm which apps write `source_dream_uuid` first.
- **Any dream (incl. video) as a start frame** via frame extraction (thumbnail/filmstrip) — widens
  the candidate pool and the featureRank coverage; useful platform-wide.
- **LTX branches alongside Kling connections.** LTX is cheaper but branches off a single start
  frame only (no first+last). Keep room for multiple generation patterns; don't design now.
- **Cloud runtime + scheduling** (Heroku app / Scheduler, or elsewhere) — decided after v1.
- **Server-side scoring / efficient query endpoints** (≈ Phase C) if API cost becomes a concern.
- Re-generation policy (cooldown/score-rise) if manual playlist-removal proves insufficient.
- Watch-through signal — client [issue #691](https://github.com/e-dream-ai/client/issues/691).
