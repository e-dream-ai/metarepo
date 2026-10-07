# Auto-Prioritized Keyframe Video Generation — Design

**Date:** 2026-09-30 (revised 2026-10-05, v3 after two review rounds)
**Status:** Design — reviewed twice against `origin/stage`; ready for implementation planning
**Repos touched:** `backend` (all new code lives here)

> **Revision note (v3).** Follow-up review confirmed the v2 fixes against real code
> and added: queue **after** commit (never inside the txn — avoids orphan jobs);
> template settings read from `transitions[key].settings` with the default key =
> first transition in **frame order**; `high_noise_loras` sent as the whole array
> with a model-fit check; helper sets `name`/`mediaType`/`user`; v1 weights bias to
> `featureRank`+recency (stills are thinner-signal); and ltx-i2v (RunPod, no pricing)
> incurs no credit charge, making the system-account credit question moot.

> **Revision note (v2).** A spec review against real `origin/stage` code found three
> load-bearing assumptions in v1 were false. This revision fixes them:
> 1. **Unit of relevance is the still *image Dream*, not the `Keyframe` entity.**
>    Flow `Keyframe` rows carry no image (`Keyframe.image` nullable; issue #716) and
>    cannot drive generation. Image Dreams (`mediaType = "image"`) carry engagement
>    signals *directly* and their UUID is exactly what i2v consumes.
> 2. **No keyframe FK linking is needed.** LTX i2v is driven by `source_dream_uuid`
>    (start) + `end_source_uuid` (end) in the prompt — both image-Dream UUIDs. The
>    `startKeyframe`/`endKeyframe` FKs are after-the-fact metadata and irrelevant here.
> 3. **Flow-wide globals no longer exist.** Generation settings live per-transition
>    (`TransitionSettings`); the "flow app template" is a *designated transition* in a
>    saved flow `EditorProject`.

> **Terminology.** In this document a "keyframe" means **a still image Dream used as
> an i2v source or target** — the thing a creator drops into a flow. It is a `Dream`
> row with `mediaType = "image"`, not the `Keyframe` entity.

## Purpose

Grow the platform's video library unattended. A nightly job ranks still image
Dreams by engagement, pairs the most relevant ones (hub-and-spoke), and enqueues
LTX i2v generation between them — no human clicks "Generate." Popular images spawn
many varied videos by being paired with several partners.

## Intent (agreed brief)

- **Outcome:** auto-fill the content library, running unattended.
- **Unit of relevance:** still **image Dream** (`mediaType = "image"`), scored by
  its *direct* engagement signals.
- **Primitive:** LTX i2v between two image Dreams — `source_dream_uuid` (start) +
  `end_source_uuid` (end) in an `ltx-i2v` prompt. Reuses the existing queue/worker
  via `processDreamRequest`; no new generation path, no keyframe linking.
- **Generation template:** a designated **transition inside a flow `EditorProject`**
  (a saved flow). The cron reads that transition's `TransitionSettings` and applies
  them, pinned to `model = "ltx-i2v"`. This is what "use a flow app template" means.
- **Cadence:** scheduled top-N batch (nightly, UTC), with hard caps.
- **Signals (v1):** upvotes/downvotes, playlist adds, featureRank, recency — all
  first-class `Dream` columns/relations. **Usage-as-i2v-source and plays/playthroughs
  deferred** (not cleanly queryable / not in DB).
- **Pairing:** hub-and-spoke — top images are hubs, each paired with several
  partners so a popular image produces many variations.
- **Dedup:** cooldown + score-rise override, keyed on `(start, end, template)`, with
  a unique constraint for idempotency.
- **Ownership:** all auto-generated videos belong to a single **system/curation
  account**. Visible immediately.

## Signals — all direct on `Dream` (verified on `origin/stage`)

`backend/src/entities/Dream.entity.ts`:

- `mediaType: DreamMediaType` (`"video" | "image"`) — line 81. Candidate pool =
  image Dreams.
- `upvotes` / `downvotes` — lines 149/152.
- `playlistItems: PlaylistItem[]` (OneToMany) — line 157. Playlist-add count.
- `featureRank` — line 133 (admin curation).
- `reports: Report[]` (OneToMany) — line 237 (safety gate).
- `nsfw` / `hidden` flags; `created_at` / `updated_at` (recency).
- `prompt` is a nullable `json` column (line 222) — note: i2v source refs live
  *inside* prompt JSON, which is why "usage as i2v source" is not cheaply queryable
  and is deferred.

## Architecture

A standalone script, run nightly by **Heroku Scheduler**, mirroring the existing
`cron:default-playlist` pattern (`backend/package.json`,
`backend/src/script/default-playlist.ts`). The backend has no cron/repeatable-job
system; this is the established pattern and avoids the double-fire a `setInterval`
in the web dyno would cause.

New modules, isolated by responsibility:

| Module | Kind | Responsibility |
|--------|------|----------------|
| `src/script/auto-generate.ts` | entrypoint | `cron:auto-generate` — wire deps, run once, exit |
| `src/services/auto-generation.service.ts` | impure shell | query signals → call pure fns → filter via dedup log → create Dreams (in txns) |
| `src/utils/keyframe-scoring.ts` | **pure** | `scoreImageDreams(signals[], weights, now) → Scored[]` (normalize + weight) |
| `src/utils/candidate-pairing.ts` | **pure** | `buildCandidates(scored[], config) → Candidate[]` (hub-and-spoke) |
| `src/utils/auto-generation-dedup.ts` | **pure** | `shouldGenerate(candidate, priorLog, config, now) → boolean` |
| `src/utils/auto-gen-prompt.ts` | **pure** | `buildLtxI2vPrompt(templateSettings, startUuid, endUuid) → promptJson` |
| `createAutoGeneratedDream()` | shared | server-side Dream create + queue (no keyframes) |

Data flow:

```
Heroku Scheduler (nightly, UTC)
  → cron:auto-generate
    → auto-generation.service.run()
      → queue-pressure guard (skip if too many auto jobs in flight)
      → load template transition (EditorProject.state.transitions[key]) → TransitionSettings
      → load candidate image Dreams + signals (SQL, safety-filtered)
      → scoreImageDreams()   [pure]  normalize per-run, then weight
      → buildCandidates()    [pure]  hub-and-spoke
      → shouldGenerate()     [pure]  cooldown + score-rise, vs auto_generation_log
      → take top MAX_JOBS_PER_RUN
      → for each:
          txn { createAutoGeneratedDream(systemUser, ltx-i2v prompt)
                + insert auto_generation_log row } ── commit
          then (AFTER commit): processDreamRequest(dream) → ltxi2v queue
                               on queue failure: mark Dream failed
      → log run summary (scored / paired / generated / skipped-dedup / skipped-guard)
```

## Component: scoring (pure)

```ts
interface ImageDreamSignals {
  uuid: string;
  netVotes: number;        // max(0, upvotes − downvotes)   (clamp ≥ 0)
  playlistCount: number;   // |playlistItems|
  featureRank: number;     // Dream.featureRank
  updatedAt: Date;
  ownerUuid: string;       // for diversity picks in pairing
}

// Per-run normalization: each term min-max scaled to [0,1] across the candidate
// set, THEN weighted. Makes weights interpretable and prevents one raw unit
// (e.g. votes) from dominating counts.
score(d) = w_votes    * norm(netVotes)
         + w_playlist * norm(playlistCount)
         + w_feature  * norm(featureRank)
         + w_recency  * recencyDecay(updatedAt, now, HALF_LIFE_DAYS)   // already 0..1
```

- Weights and `HALF_LIFE_DAYS` are config constants.
- `netVotes` clamped at 0 so downvoted content can't cancel other signals into
  negatives.
- **Cold-start policy (explicit):** a brand-new image with zero votes/playlists/
  featureRank scores only on recency. It will not out-rank engaged content and may
  never surface — acceptable for v1 (we want *proven* content). Documented, not a bug.
- **Thin-signal reality (v1 weighting):** stills *are* first-class public content —
  they are votable/addable and have their own public feed filter (`STILLS`,
  `frontend/src/constants/feed.constants.ts`) — so the signal pool is real. But
  image Dreams likely accrue far fewer votes than videos, so for many of them
  `netVotes`/`playlistCount` ≈ 0 and ranking leans on `featureRank` + recency.
  Default weights for v1 should therefore **bias toward `w_feature` and
  `w_recency`**, revisited once stills-engagement volume is known.
- `now` is injected (no `Date.now()` inside) for deterministic tests.

### Safety gate (applied in the candidate query, before scoring)

Exclude image Dreams that are `nsfw`, `hidden`, have any `Report`, are not
`processed`, or whose owner's account is inactive. Note: `User` has **no**
disabled/banned column — the only inactive-account concept is soft-deletion
(`deleted_at`; semantics per `utils/account-status.util.ts: isAccountDeleted`).
In the `createQueryBuilder` candidate query the soft-delete predicate must be
written **explicitly** on the joined `User` alias (`<user>.deleted_at IS NULL`) —
TypeORM does not auto-apply it to joined relations. Because output is system-owned
and public immediately, this gate is the primary content-safety control — pinned
here, not deferred. (Residual risk: an un-flagged-but-objectionable source image. See
Risks.)

## Component: pairing (pure) — hub-and-spoke

```ts
interface PairingConfig {
  hubsPerRun: number;          // e.g. 10
  spokesPerHub: number;        // e.g. 3
  diversityRatio: number;      // fraction of spokes that are diversity picks, e.g. 1/3
  bothDirections: boolean;     // also emit spoke→hub? default false for v1
}

interface Candidate {
  startDreamUuid: string;      // hub  → source_dream_uuid
  endDreamUuid: string;        // spoke → end_source_uuid
  pairScore: number;           // score(start) + score(end), both normalized
}
```

- Hubs = top `hubsPerRun` image Dreams by score.
- Each hub's spokes = `(1 − diversityRatio)` **quality picks** (next highest
  scorers not equal to the hub) + `diversityRatio` **diversity picks**, defined
  deterministically: *lowest-scored candidate whose `ownerUuid` differs from the
  hub's, tie-break by `uuid`*. Deterministic so the pure function is testable.
- Each `(hub, spoke)` → one `Candidate`. Directed: start = hub.
- `pairScore = score(start) + score(end)`; final list sorted desc.
- Guard against degenerate pairs: skip `start == end`; skip exact duplicate
  unordered pairs within a run unless `bothDirections`.

## Component: the flow-app template (a designated transition)

Generation settings come from a **saved flow**, not an invented format. On
`origin/stage`:

- `EditorProject` (`backend/src/entities/EditorProject.entity.ts`) has
  `editorId ("flow")`, `name`, `state: jsonb`, `schemaVersion`.
- `state` persists as `{ referenceFrames, transitions }`
  (`frontend/.../utils/editor-project-state.ts`), where `transitions` is a **keyed
  object** `Record<"fromId::toId", PersistedTransition>` and the settings live at
  `transition.settings` (`PersistedTransition = Omit<FlowTransition,"progress">`,
  `frontend/src/types/flow.types.ts`). `TransitionSettings = { prompt,
  negativePrompt, duration, model, steps, guidance, seed, highNoiseLoras,
  lowNoiseLoras }`.

Config supplies:
- `AUTO_GEN_TEMPLATE_PROJECT_UUID` — the designated flow `EditorProject`.
- `AUTO_GEN_TEMPLATE_TRANSITION_KEY` — which transition in `state.transitions`
  (the `"fromId::toId"` key). If omitted, the default is the **first transition in
  frame order**, i.e. the key `` `${referenceFrames[0].id}::${referenceFrames[1].id}` ``
  (object key order is not reliable; frame order is how `fromPersistedFlowState`
  reconstructs the flow — `editor-project-state.ts`).

The cron loads the project **in-process via the repository** (no HTTP), validates
`schemaVersion`, extracts that transition's `settings` (`transitions[key].settings`),
and **pins `model = "ltx-i2v"`** (the default stored model is `kling-25-i2v`, so this override
is required to honor the brief). `buildLtxI2vPrompt` then mirrors
`buildVideoAlgoParams`' ltx-i2v branch:

```
{
  infinidream_algorithm: "ltx-i2v",
  prompt: settings.prompt,
  source_dream_uuid: startDreamUuid,
  end_source_uuid: endDreamUuid,
  duration: settings.duration,
  negative_prompt?: settings.negativePrompt (if non-empty),
  high_noise_loras?: settings.highNoiseLoras (whole array; worker uses [0]) —
                     only if non-empty AND actionLorasFitModel(settings, "ltx-i2v")
                     (drop a LoRA stored for a different model),
  seed?: settings.seed (if finite),
  <guidance param per GUIDANCE_PARAM["ltx-i2v"]>?: settings.guidance,
}
```

Only the transition's *settings* are used; the template's own
referenceFrames/transitions are ignored — the *pairs* come from hub-and-spoke.
Output is therefore shape-identical to a human-driven LTX i2v transition. A
**curated set** of template transitions rotated for variety is a noted future
extension (the dedup key already carries the template ref).

## Component: dedup + re-generation (pure predicate + new table)

New table `auto_generation_log`:

| column | type | note |
|--------|------|------|
| `id` | PK | |
| `start_dream_uuid` | uuid | |
| `end_dream_uuid` | uuid | |
| `template_ref` | varchar | `projectUuid#transitionKey` |
| `pair_score` | float | normalized pair score at generation time |
| `dream_uuid` | uuid | the created Dream |
| `created_at` | timestamp | |

**Unique constraint** on `(start_dream_uuid, end_dream_uuid, template_ref)` for
idempotency; insert the log row **in the same transaction** as the Dream create, so
a crash mid-run cannot leave a Dream with no log row (would otherwise regenerate).

`shouldGenerate(candidate, priorLogForTriple, config, now)`:

- No prior row for `(start, end, template)` → generate.
- Prior row older than `COOLDOWN_DAYS` (default 30) → generate.
- Within cooldown but current `pairScore` ≥ `(1 + REGEN_SCORE_RISE_PCT/100) ×`
  **max** logged `pair_score` for the triple → generate (compare against max, not
  last, so a steadily-rising score can't re-fire every run).
- Otherwise skip.

A popular image paired with *different* partners is never blocked — the intended
"many variations" behavior.

## Component: server-side Dream creation (no keyframes)

New helper `createAutoGeneratedDream(systemUser, promptJson, name)`:

1. Creates a `Dream` with the fields `processDreamRequest` + the schema require:
   `user = systemUser` (or `userId` — `getOwnerId` in `utils/ownership.util.ts`
   accepts either), `status = QUEUE`, `mediaType` (defaults to VIDEO, correct for
   ltx output), `name` (generated, e.g. from the pair/template — `Dream.name` is
   set on every controller create), `autoGenerated = true`, and
   `prompt = JSON.stringify(promptJson)`. **No keyframe FK linking** — i2v sources
   are the two image-Dream UUIDs inside the prompt.
2. **Ordering (avoids orphan jobs):** persist the Dream **and** the
   `auto_generation_log` row inside one DB transaction and **commit first**; only
   *after* commit call `processDreamRequest(dream)` to enqueue. If enqueue fails,
   mark the Dream failed (as the controller already does). Never call
   `processDreamRequest` inside the transaction — it performs an external enqueue
   plus its own `dreamRepository.update`, so a rollback after enqueue would leave a
   worker job pointing at a vanished Dream.
3. `processDreamRequest` routes via the algorithm registry in
   `backend/src/utils/prompt.util.ts` (`ALGORITHM_REGISTRY` — `"ltx-i2v" → { queue:
   "ltxi2v", media: "video" }`). The registry *is* `prompt.util.ts`; there is no
   separate registry module.

Simpler than v1's "extract `createDreamWithKeyframes` from the controller": no
keyframe round-trip, controller untouched. Reuse any existing Dream-construction
helper on stage if one fits; otherwise this is a small new function.

**Credit note:** `ltx-i2v` is a RunPod provider model with no `pricing`
(`models.constants.ts`), so `processDreamRequest`'s credit-reservation branch
(`provider === FAL && pricing`) is skipped — auto-generating ltx-i2v incurs no
credit charge. The system-account "credit bypass" decision is therefore moot for
the chosen model (revisit only if a FAL-priced model is ever templated).

## Ownership, visibility, observability

- **Owner:** a single system/curation account. **None exists on `origin/stage`** —
  creating it is an explicit prerequisite task (migration/seed), with decisions on:
  WorkOS identity backing it, whether it bypasses credit/cost accounting (it should),
  and the license/`ccbyLicense` / proprietary-dream flags to set. Id supplied via
  config (`CURATION_USER_ID`).
- **Visibility:** visible immediately — no hidden/review flag on output. The input
  safety gate is the control.
- **`Dream.autoGenerated`** (new boolean column, default false, indexed): marks
  auto-generated videos for the queue-pressure guard, metrics, and future filtering.
- **Run summary logging:** counts for scored / paired / generated / skipped-dedup /
  skipped-guard, plus the template ref used — so a run is auditable.

## Budget / safety caps (all config)

| Config | Default | Purpose |
|--------|---------|---------|
| `MAX_JOBS_PER_RUN` | 50 | hard ceiling on Dreams created per run |
| `HUBS_PER_RUN` | 10 | top images used as hubs |
| `SPOKES_PER_HUB` | 3 | partners per hub |
| `DIVERSITY_RATIO` | 1/3 | fraction of spokes that are diversity picks |
| `AUTO_GEN_TEMPLATE_PROJECT_UUID` | — | designated flow `EditorProject` |
| `AUTO_GEN_TEMPLATE_TRANSITION_KEY` | first | transition within the template |
| `COOLDOWN_DAYS` | 30 | min days before re-generating the same triple |
| `REGEN_SCORE_RISE_PCT` | 50 | score rise (vs max logged) that overrides cooldown |
| `MAX_INFLIGHT_AUTO_JOBS` | 100 | queue-pressure guard; skip run if exceeded |
| `HALF_LIFE_DAYS` | 30 | recency decay half-life |
| `CURATION_USER_ID` | — | system owner of auto-generated Dreams |
| weights `w_votes/w_playlist/w_feature/w_recency` | tuned | scoring |

**Queue-pressure guard:** before building candidates, count `autoGenerated` Dreams
still in `QUEUE`/`PROCESSING`; if over `MAX_INFLIGHT_AUTO_JOBS`, log and exit so
nightly batches never pile up.

**First-run note:** with an empty `auto_generation_log`, the first run generates up
to `MAX_JOBS_PER_RUN` immediately. "Nightly" is UTC (Heroku Scheduler).

## Failure handling

If a generated Dream fails downstream, its `auto_generation_log` row remains, so the
pair won't retry until cooldown. Acceptable for v1. A future improvement: on gen
failure, delete the log row (or mark it failed) to allow retry.

## Testing strategy

- **`keyframe-scoring.ts` (pure):** per-run normalization; `netVotes` clamp; recency
  decay; weight application; cold-start; deterministic given fixed `now`.
- **`candidate-pairing.ts` (pure):** hub/spoke counts; deterministic diversity rule;
  directedness; `pairScore` ordering; degenerate-pair guards; fewer candidates than
  hubs.
- **`auto-generation-dedup.ts` (pure):** fresh triple generates; within-cooldown
  skips; cooldown expiry generates; score-rise-vs-max override; different-partner
  never blocked.
- **`auto-gen-prompt.ts` (pure):** correct `ltx-i2v` params incl. optional
  negative_prompt / loras / seed / guidance; model pinned to `ltx-i2v`.
- **`auto-generation.service.ts` (integration, seeded DB):** signals in → correct
  Dreams created with the right source/end UUIDs in prompt, `autoGenerated=true`,
  owner=system; caps respected; queue-pressure guard skips; safety gate excludes
  flagged images; log rows written; crash-between-create-and-log can't duplicate
  (unique constraint + txn).

## Risks / open decisions (resolve during planning, on fresh `origin/stage`)

1. **System account** — creation mechanism (migration/seed vs manual), WorkOS
   identity, license flags. Hard prerequisite. (Credit bypass is moot for ltx-i2v —
   see Credit note — but revisit if a FAL-priced model is ever templated.)
2. **Content safety of system-owned, instantly-public output** — the input gate
   excludes nsfw/hidden/reported, but generating public video from *other users'*
   images under a system account is a product/legal call worth explicit sign-off.
3. **`EditorProject.state` schema stability** — validate `schemaVersion`; decide
   behavior if the designated transition is missing or malformed (fail the run vs
   fall back to defaults).
4. **Candidate-query scale** — aggregating playlist counts across all image Dreams;
   ensure indexed, consider a materialized/cached count if slow.
5. **Guidance param mapping** — reuse `GUIDANCE_PARAM["ltx-i2v"]` semantics from the
   frontend so the server-built prompt matches.

## Out of scope (v1) / future extensions

- Plays / playthroughs as a signal (needs a persistent tracking table).
- Usage-as-i2v-source signal (requires querying source refs inside prompt JSON).
- A curated *set* of template transitions rotated for variety.
- Retry of failed auto-generations.
- Threshold-triggered (event-driven) generation.
