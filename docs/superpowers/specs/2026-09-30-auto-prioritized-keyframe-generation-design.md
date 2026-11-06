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
>   on **fal**, priced `baseUsd 0.35 + usdPerSecond 0.07 × duration`
>   (`models.constants.ts`) → ~$0.70 at the pinned 5 s, $1.05 at 10 s. Add a **per-run cost
>   estimate (derived from the template's `duration`) + a hard spending cap**.
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
> - **Content safety** (Risk 2 resolved): inputs are **SFW**. Important: the feed's nsfw filter
>   is driven by the **authenticated account's own `user.nsfw` profile flag**
>   (`feed.controller.ts` passes `user?.nsfw`), *not* a client request option — so the hand
>   setup **must set the `autogen` account's profile `nsfw = false`** (same for playlist-item
>   reads). This is on the provisioning checklist, not something the engine toggles.
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
  4. pool         — GET /v1/feed?type=dream&mediaType=image (paged to exhaustion ≤ MAX_FEED_PAGES;
                    mediaType is REQUIRED — type=dream alone defaults to VIDEO) → images w/
                    playlistItems, status, created_at (feed omits votes+prompt; SFW only);
                    keep status==processed; drop autogen-owned (self-exclusion)
  5. feature      — (optional, W_FEATURE>0) GET /v1/feed/ranked → walk ranked playlists' items →
                    per-image feature score by ORDER (featureRank value stripped; order kept)
  6. popularity() [pure]  v1 = playlistCount + recency (+ feature); scaled 0–1 + floor
  7. draw_pairs() [pure]  w(d)=pop(d)*(1+k*min(connected,CAP)); skip self/dup/already/usage-cap/
                    aspect-incompatible
  8. cost guard   — cost/video derived from template duration (0.35+0.07*dur); n = min(JOBS_PER_RUN,
                    floor(MAX_SPEND_PER_RUN / cost)); 0 if cost<=0 or cap<=0; log estimate
  9. for each of n pairs (unless DRY_RUN):
        prompt = build_kling_prompt(template_prompt, start, end)   [pure]
        POST /v1/dream (as autogen) → dreamUuid ; add it to PLAYLIST_UUID *right away*
        (incremental, not one batch at end — closes the between-runs crash hole; if the add
        itself throws just after create, catch-and-log the uuid for manual reconcile — create
        and add aren't atomic)
 10. log summary (pool / drawn / est_cost / created / skipped)
```

## Integration edges (public API only; `Authorization: Api-Key <key>` via `edream_sdk`)

| Capability | Endpoint | Status |
|------------|----------|--------|
| Identify self | `GET /auth/user` (`edream_sdk.get_logged_user()`) | **exists** |
| In-flight guard | `GET /v1/dream?userUUID=<self>&status=queue` + `&status=processing` | **exists** |
| Template prompt | `GET /v1/dream/:TEMPLATE_DREAM_UUID` | **exists** |
| Existing pairs (state) | `GET /v1/playlist/:PLAYLIST_UUID/items` — items include each dream's `prompt` | **exists** |
| Candidate pool | `GET /v1/feed?type=dream&mediaType=image` (paged; **`mediaType` REQUIRED** — `type=dream` alone defaults to VIDEO) — `playlistItems`,`status`,`created_at` (no votes/prompt); SFW only; filter `status==processed` client-side | **exists** (needs `media_type` arg added to the SDK's `get_feed`) |
| Feature stand-in | `GET /v1/feed/ranked` — featured playlists in `featureRank` order (value stripped, order kept) | **exists** |
| Create generation | `POST /v1/dream` | **exists** (needs `CREATOR_GROUP` — the hand-made `autogen` account is a creator) |
| Add to playlist | `edream_sdk.add_items_to_playlist` → `POST /v1/playlist/:uuid/items` (atomic, all-or-none; ≤500) | **exists in SDK** — add *freshly-created* uuids only, so the all-or-none contract never trips (no per-item fallback needed) |

**No backend changes for v1** — all existing. New endpoints may come later (Deferred) but must be
**general-purpose** (e.g. "dreams derived from uuid X"), with matching `edream_sdk` methods.

## Signals & popularity

**Images have essentially no votes of their own — only videos get voted on.** The *intended*
signal is therefore: an image's popularity = the engagement (votes + playlist-adds) of the
**videos derived from it** (videos whose prompt lists it as `source_dream_uuid`/`end_source_uuid`).
The public feed does **not** return `prompt`, so tracing those videos per image is not cheap today
(and some links exist only via `startKeyframe`/`endKeyframe`). Authorship, confirmed: **flow +
batch video gen** write `source_dream_uuid`/`end_source_uuid` as frame-derivation
(`build-video-algo-params.ts`, used by `useFlowGeneration`/`useBatchSubmit`); but **image gen
also writes `source_dream_uuid` as a *style reference*** (`build-image-algo-params.ts`) — the
field is overloaded, so a future trace must disambiguate video-derivation from image style-ref
(and also fold in the `startKeyframe`/`endKeyframe` FK links).

- **v1 (no trace available):** `popularity(d)` = `w_playlist · playlistCount(d)` + `w_recency ·
  recency(d)` (+ optional `w_feature · featureScore(d)`), each min-max scaled to [0,1] across the
  run + a floor. `playlistItems` and `created_at` come straight from the feed — **no per-image
  calls**. **Honest framing:** `playlistItems` is *current playlist memberships*, not historical
  adds, and most user stills sit in zero playlists — so in practice v1 ranking is
  **recency-dominated** (≈ recent images + connectivity + floor randomness), not true
  "engagement." Treat the engagement framing as aspirational until the Phase C derived-video
  signal lands; this is fine for a library-growth experiment but say so plainly.
- **Later:** add a `w_votes` term fed by a general-purpose backend endpoint that returns dreams
  derived from a given uuid (Deferred / Phase C), so image popularity reflects its videos'
  engagement without client-side prompt scraping.

### Candidate visibility

The main feed is a **global public gallery for any user** (not caller-scoped):
`buildFeedConditions()` (`backend/src/utils/feed.util.ts`) OR's the caller's own `hidden:true`
dreams with **all `hidden:false` dreams, no owner filter** — so `autogen` sees every user's
non-hidden images. Narrowing: `hidden:true` (others' → excluded); **SFW only** — the feed filters
nsfw by the **caller account's `user.nsfw` profile flag** (not a request param), so the `autogen`
account's profile must be `nsfw=false` (content-safety control, Risk 2; on the setup checklist);
`status==processed` (client-side; the feed applies no status filter but returns `status`); a
`FeedItem` row must exist (dry-run pool size exposes gaps); and **self-exclusion** (drop
`user.uuid == self`). Reported images are **not** excluded.

## Selection — weighted lottery (pure)

1. `popularity(d)` — as above (0–1 scaled + floor).
2. `connected(d)` — times `d` appears as a start/end in the output playlist's pairs, **saturated**
   `min(connected(d), CONNECTED_CAP)` (or `log1p`) so weight can't compound without bound.
3. `w(d) = popularity(d) × (1 + k · saturated_connected(d))`.
4. For each of `N` draws: pick start + end ∝ `w`; **redraw** on `start==end`, a pair already in the
   playlist, a pair whose two images are **aspect-ratio incompatible** (see below), or an image
   that has already hit its **per-run usage cap** `USAGE_CAP` (no single image in more than
   `USAGE_CAP` pairs this run). If no fresh valid pair within a bounded `max_attempts` (pool of 1,
   everything made), **stop and return fewer than `N` (possibly 0)** — never loop. Floor applied
   **before** normalization (no divide-by-zero).

**Rich-get-richer guard.** `w = pop · (1 + k·connected)` is self-reinforcing — without limits a
run can spend its whole budget wiring up the same ~5 images. The saturating `connected` + the
per-run `USAGE_CAP` bound concentration. A `draw_pairs` test asserts **no single image exceeds a
set share of the N pairs**.

`k` biases toward already-used images so videos link into a connected graph; floor + randomness
give long-tail images a chance. Tunables: `JOBS_PER_RUN` (`N`), `CONNECTIVITY_K` (`k`),
`CONNECTED_CAP`, `USAGE_CAP`, weights, `HALF_LIFE_DAYS`. Draw takes an injected `random.Random`
(seeded) for deterministic tests.

**Aspect-ratio compatibility.** The worker **rejects a pair whose start/end images normalize to
different sizes** (`worker/src/.../fal-handlers.ts`), so arbitrary pairing would produce many
failed renders. The feed does **not** return image dimensions, so v1 can't bucket by aspect ratio
without extra data. **v1 stance:** accept-and-log mismatch failures (confirm whether the worker
rejects *before* submitting to fal — if so there's no fal $, just a failed dream that stays in the
output playlist and so won't be retried). **Fix (Deferred):** expose image dimensions (a feed
field or a per-candidate fetch) and pair only within an aspect bucket.

## Prompt building (pure)

`build_kling_prompt(template_prompt, start_uuid, end_uuid)`: deep-copy the Kling template dream's
`prompt`, force `infinidream_algorithm: "kling-25-i2v"`, set `source_dream_uuid = start` and
`end_source_uuid = end`, leave the rest (prompt text, `duration`, `cfg_scale`, etc.) intact — the
worker maps the two source uuids to Kling's start/end frames. Accepts a dict or JSON-string prompt.
**Startup validation:** the template dream's prompt must already be `kling-25-i2v` **and carry a
`duration` of 5 or 10** (the worker rejects other durations — `fal-handlers.ts`); else fail-fast.
The created dream's `name` is deterministic, e.g. `autogen <start8>→<end8>`.

## Dedup & in-flight (no extra storage)

- **Dedup = the output playlist.** `(source,end)` pairs already present are the dedup set; a drawn
  pair in it is redrawn. Delete a dream from the playlist → its pair can regenerate (manual redo).
  When reading items, **skip any whose prompt lacks both source/end uuids** (hand-added/non-i2v).
  **Page the playlist to exhaustion** — the default page is tiny (`PAGINATION.TAKE=10`, max 5000),
  so pass an explicit large `take` and loop; a single default call would silently miss older pairs
  as the playlist grows and re-generate them.
- **In-flight guard.** No single "my dreams by status" endpoint (`/my-dreams` has no status
  filter; `GET /v1/dream` takes a `userUUID` param + single status). Count with two calls
  (`…&status=queue`, `…&status=processing`) using the uuid from `get_logged_user()`; skip the run
  if the sum exceeds `MAX_INFLIGHT`.

## Cost estimate & spending cap

Kling (fal) is priced `baseUsd 0.35 + usdPerSecond 0.07 × duration` (`models.constants.ts`), so
cost is deterministic per run once the template's `duration` is known. **Derive** the per-video
cost at startup from the template dream's `duration` (`0.35 + 0.07 × duration`) rather than
trusting a stale constant — `COST_PER_VIDEO` is an optional override only. Each run caps
`n = min(JOBS_PER_RUN, floor(MAX_SPEND_PER_RUN / cost_per_video))` (and `0` if `cost_per_video <=
0`), computes `estimated_spend = n × cost_per_video`, and `DRY_RUN` logs the estimate + creates
nothing. Client-side guard; the backend may also enforce its own credit limits on `autogen`.

## Config

`BACKEND_API`, `API_KEY`, `TEMPLATE_DREAM_UUID` (a Kling dream), `PLAYLIST_UUID`, `DRY_RUN`
(default `true`), `JOBS_PER_RUN`, `CONNECTIVITY_K`, `CONNECTED_CAP`, `USAGE_CAP`, weights
(`W_PLAYLIST`/`W_RECENCY`/`W_FEATURE`), `HALF_LIFE_DAYS`, `MAX_INFLIGHT`, `MAX_SPEND_PER_RUN`,
`COST_PER_VIDEO` (optional override — otherwise derived from the template's `duration`),
`MAX_FEED_PAGES`.
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
2. **Content safety** — resolved: inputs are **SFW**, enforced by the feed's nsfw filter which
   keys off the **`autogen` account's own `user.nsfw=false` profile flag** (set it in the hand
   setup — not a request option the engine controls).
3. **featureRank coverage** — thin (featured content is mostly videos); keep `W_FEATURE` low or 0.
   *Extension:* allow **any dream, including videos, as a start frame** by extracting a frame
   (thumbnail/filmstrip) — a promising general feature beyond autogen (Deferred).
4. **API cost / efficiency** — fine to start; if it gets bad, add general-purpose backend
   query/search endpoints + `edream_sdk` methods (≈ Phase C) — not planned in detail now.
5. **`edream_sdk` compatibility** — pre-1.0; additive changes are fine; keep existing consumers
   working unless something was genuinely wrong.
6. **Kling cost** — real; bounded by `MAX_SPEND_PER_RUN` + `DRY_RUN`. LTX would be cheaper but only
   branches from a single start frame (no two-image connection) — see Deferred.
7. **Aspect-ratio pairing failures** (Important) — the lottery pairs arbitrary images; the worker
   rejects mismatched start/end sizes, so expect a real render-failure rate. v1 accepts-and-logs;
   **confirm the worker rejects *before* fal submission** (no $ if so). Fix = pair within aspect
   buckets once dimensions are available (Deferred).
8. **Backend credit backstop** — **REQUIRED setup step, not "confirm later".** The client-side
   cap only guards a hand-run CLI; one fat-fingered `MAX_SPEND_PER_RUN` spends real fal dollars
   with nothing behind it. Before the first `DRY_RUN=false`, set a sane server-side
   `dailyQuotaUsd`/credit limit on the `autogen` account (columns exist on `User`) as the hard
   ceiling. On the hand-setup checklist.
9. **Create→render failure** — a dream that *creates* fine but later *fails rendering* is already
   in the output playlist (we add on create), so its pair won't retry. Intended (don't re-attempt
   a bad pair), but note it; its fal cost depends on where the failure occurred.
10. **Pool ceiling + shrinking universe** — `MAX_FEED_PAGES × pageSize` caps the pool to the
    newest N images (feed is `created_at DESC`), and recency dominates scoring, so runs keep
    drawing from the same recent window; with dedup the reachable universe shrinks each run. Log
    both "feed-cap hit" and "pool exhausted vs dedup" so the operator sees when there's nothing new
    to pair.

## Deferred / future

- **Derived-video engagement signal + endpoint.** A general-purpose backend endpoint "dreams
  derived from uuid X" (by `source_dream_uuid`/`end_source_uuid`, and/or `startKeyframe`/
  `endKeyframe`) + an `edream_sdk` method, so image popularity reflects its videos' votes/adds
  without client-side prompt scraping. The endpoint must disambiguate frame-derivation (flow +
  batch video) from image gen's style-ref use of `source_dream_uuid`, and include the
  `startKeyframe`/`endKeyframe` FK links.
- **Any dream (incl. video) as a start frame** via frame extraction (thumbnail/filmstrip) — widens
  the candidate pool and the featureRank coverage; useful platform-wide.
- **LTX branches alongside Kling connections.** LTX is cheaper but branches off a single start
  frame only (no first+last). Keep room for multiple generation patterns; don't design now.
- **Cloud runtime + scheduling** (Heroku app / Scheduler, or elsewhere) — decided after v1.
- **Server-side scoring / efficient query endpoints** (≈ Phase C) if API cost becomes a concern.
- Re-generation policy (cooldown/score-rise) if manual playlist-removal proves insufficient.
- Watch-through signal — client [issue #691](https://github.com/e-dream-ai/client/issues/691).
