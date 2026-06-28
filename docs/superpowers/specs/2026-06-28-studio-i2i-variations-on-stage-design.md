# Studio Variations + Prompt Expansion on stage, with fal i2i (FLUX.1 Kontext)

**Date:** 2026-06-28
**Status:** Draft
**Scope:** Cross-repo — `frontend` + `backend` + `worker`
**Supersedes (re-scopes):** `docs/superpowers/specs/2026-06-12-api-key-config-i2i-variations-design.md`
**Source branch (frontend):** `feat/api-key-config-i2i` (the existing ~40-commit PR)
**Target base:** `origin/stage` (all three repos)
**New branches:** `feat/studio-i2i-variations` in each of frontend / backend / worker

## Context

The `feat/api-key-config-i2i` PR was built on a snapshot of `stage` (frontend merge-base `7160fcf`). Competing work has since landed on `stage` that overlaps the PR, and — crucially — the PR's i2i was built on an architecture stage rejected. Two things are true:

1. **Three frontend collisions** force a re-apply rather than a rebase (below).
2. **The PR's i2i is unreachable on stage.** The PR routed i2i through a per-user `UserApiEndpoint` (user pastes their own fal/OpenAI endpoint+key). Stage replaced that with `provider-key` (a stored fal key) + a curated `MODEL_CATALOG`. Stage's catalog has **no image-to-image model** — its image models are text-to-image (`flux-schnell`), and its "i2i" entries are image-to-*video* (Kling). So delivering i2i now means **adding a fal image-to-image model end-to-end**.

### Frontend collisions

| Area | Stage's version | PR's version | Resolution |
|------|-----------------|--------------|------------|
| **User API keys** | `provider-key` — stored fal key, server-side; frontend sees only status | `user-api-endpoints` — per-endpoint records | **Drop PR's; adopt stage's** |
| **Sessions** | `keyframes-sessions` (#659) — own `session.store`/`session-switcher`/`session.types`/`useSessionAutoSave` | parallel implementation of same files | **Drop PR's; adopt stage's** |
| **Studio internals** | reworked `transition-settings-panel`, `useFlowJobProgress`, `flow.store` (kept `persist`), model-catalog UI | PR also rewrote these | **Adopt stage's; graft only variation/expansion** |

A 40-commit rebase would force hand-resolution of hundreds of conflict lines in the settings + session code we discard. Instead: **branch fresh off `origin/stage` and re-apply only the genuinely-unique features** — variations + prompt expansion — onto stage's current code.

## Decisions (locked with user)

| Decision | Choice |
|----------|--------|
| Branch strategy | Fresh `feat/studio-i2i-variations` per repo off `origin/stage`; old PR branches kept as backup; new PRs |
| Settings feature | Drop PR's `user-api-endpoints`; adopt stage's `provider-key` |
| Sessions | Adopt stage's session system; drop PR's |
| Variation methods shipping | `seed`, `expansion` (no backend dep) **and** `i2i` (via new fal model) |
| i2i provider | **fal only**, FLUX.1 Kontext. OpenAI i2i deferred to a follow-up (stage is fal-only end-to-end; OpenAI = a whole second provider) |
| Key delivery | Server-side. Frontend never holds the key; `dream.util`'s `resolveProviderKeyDecision` picks personal-vs-global fal key. No `userEndpointUuid` anywhere. |

---

# Part 1 — Frontend re-apply (variations + prompt expansion)

## Keep set (re-apply onto stage)

**Verbatim-copyable new files** (absent on stage):
- `studio/components/variation-grid.tsx` + `.styled.tsx`
- `studio/components/variation-lightbox.tsx` + `.styled.tsx`
- `studio/components/variation-settings-panel.tsx` *(strip BYO endpoint reads)*
- `studio/components/prompt-expansion-badge.tsx`
- `studio/constants/variation-presets.ts`
- `studio/utils/variation-status.ts` + `__tests__/variation-status.test.ts`
- `studio/utils/expand-prompt.ts` + `__tests__/expand-prompt.test.ts`

**Additive type grafts** — `src/types/flow.types.ts` (stage never touched it; applies clean):
- `FlowKeyframe`: `i2iCandidate?`, `i2iParentId?`, `i2iStatus?`, `variations?`, `activeVariationId?`
- `FlowTransition`: `variations?`, `activeVariationId?`
- new `VariationCandidate` interface

**Store grafts — `src/stores/flow.store.ts`** (stage has zero variation refs; the `set()` blocks are additive):
- i2i candidate staging: `addI2iCandidates`, `acceptI2iCandidate`, `discardI2iCandidate`
- keyframe variations: `addKeyframeVariations`, `selectKeyframeVariation`, `clearKeyframeVariations`, `updateKeyframeVariation`
- transition variations: `addTransitionVariations`, `selectTransitionVariation`, `clearTransitionVariations`, `updateTransitionVariation`
- timeline-derivation helper filters out `i2iCandidate` keyframes so candidates never produce transitions/generation — **reconcile with stage's derivation code**.
- **Drop** `i2iEndpointUuid` / `setI2iEndpoint` (BYO state — replaced by server-side provider-key).

**Store grafts — `src/stores/studio.store.ts`** (PR adds variation UI state; stage has the file):
- add `variationPresetId`, `variationCustomPrompt`, `variationSeed` (+ setters), `DEFAULT_VARIATION_SEED`.
- persist the preset/prompt/seed choices via stage's `studioPartialize` (not transient UI).

**Wiring grafts into stage's versions** (KEEP hunks only; discard session/endpoint hunks):
- `flow-builder.tsx`, `images-tab.tsx`, `keyframe-card.tsx(.styled)`, `keyframe-strip.tsx(.styled)`, `generate-tab.tsx`, `actions-tab.tsx`, `transition-gap.tsx`, `transition-settings-panel.tsx`, `studio.page.tsx(.styled)` — variation grid/lightbox mounts + prompt-expansion.
- Hooks `useFlowGeneration.ts`, `useFlowJobProgress.ts`, `useBatchSubmit.ts` — variation progress tracking + candidate result swap-in.

## Variation rehydration — extend stage's existing action

The PR's variation reconciliation lives inside a `reconcileStaleTransitions` action (`reconcileVariations` helper over keyframe + transition `variations`). Stage **already has** `reconcileStaleTransitions` (`flow.store.ts:339`) and calls it from **both** rehydration entry points — `flow.store` `onRehydrateStorage` (`:393`) and stage's `session.store.loadSession` (`session.store.ts:201`). So this is an **additive graft into stage's existing action**, not a relocation — both paths are covered automatically. Stage kept `persist`/`flowPartialize`/`onRehydrateStorage` (it only extracted `flowPartialize` into an exported function). Lower risk than a relocation; still verify the session-switch path manually.

**Partialize graft:** add `&& !kf.i2iCandidate` to stage's `flowPartialize` keyframe filter so staging candidates aren't persisted (matches the project's "don't persist unsettled keyframes" rule).

## BYO strip → provider-key

Remove every `useUserApiEndpoints` / `UserApiEndpoint` / `endpoint-presets` read and the endpoint `<select>` UI from `variation-settings-panel.tsx`, `flow-builder.tsx`, `images-tab.tsx`. The i2i variation method no longer references an endpoint; it selects the fal i2i model id and passes the source image (Part 2). Provider key is resolved server-side.

## Drop set (do not carry over)

- `src/api/user-api-endpoints/*` (5 hooks)
- `components/pages/profile/add-endpoint-modal*.tsx`, `api-endpoints-section*.tsx`
- `components/shared/profile-card/profile-card.tsx` (PR's edit — keep stage's provider-key version)
- `src/constants/endpoint-presets.ts`, `src/types/user-api-endpoint.types.ts`
- PR's `src/stores/session.store.ts`, `studio/components/session-switcher.tsx(.styled)`, `src/types/session.types.ts`, `studio/hooks/useSessionAutoSave.ts`
- PR's `src/main.tsx` / `src/App.tsx` edits **except** any route/provider variations genuinely need (the +107 `main.tsx` is mostly endpoint/query wiring — re-add to stage's only what variations require, if anything).

---

# Part 2 — fal i2i (FLUX.1 Kontext), end-to-end

Goal: a fal **image-to-image** model the variation i2i method can target. Reuses stage's fal client, key-resolver, and credit system.

### Worker (`worker` repo)

1. **`src/providers/provider.types.ts`** — add `imageUrl?: string` to `NormalizedImageInput`.
2. **`src/config/models.config.ts`** — add an `ImageModelConfig` entry, e.g.
   `'flux-kontext-i2i': { id, provider: 'fal', mediaType: 'image', endpoint: 'fal-ai/flux-pro/kontext' }`.
   *Confirm the exact fal endpoint slug during implementation* (Kontext Pro vs dev).
3. **`src/providers/fal.provider.ts`** — `buildFluxInput` (or a Kontext-specific builder) passes `image_url: input.imageUrl` and `prompt` for the Kontext endpoint. FLUX Kontext takes `{ prompt, image_url }` and returns `images[]` — `pollImage` already extracts `images[].url`, so the poll path is unchanged.
4. **`src/workers/fal-handlers.ts`** — the image handler currently builds `NormalizedImageInput` with no image. For an i2i model, read the source image from job data (mirror the video handler: `source_dream_uuid: image` → `processImageForEndpoint(image, jobId)`) and set `input.imageUrl`. Branch on whether the model is i2i (e.g. presence of the image field / a config flag).

### Backend (`backend` repo)

1. **`src/constants/models.constants.ts`** — add the catalog entry: `{ id: 'flux-kontext-i2i', label: 'FLUX.1 Kontext', provider: PROVIDERS.FAL, mediaType: IMAGE, constraints, pricing }`.
2. **Pricing** — Kontext is priced **per image**, but `ModelPricing` only has `perMegapixel` / `perSecond`. Add a `perImage` kind (or map to `perMegapixel` at the model's fixed size). *Decide during implementation; flag for `provider-credit.service`.*
3. **`SupportedAlgorithm`** (`utils/prompt.util`) — add `'flux-kontext-i2i'`.
4. **`src/utils/dream.util.ts`** — fal routing already keys on `model.provider === FAL` and resolves the provider key via `resolveProviderKeyDecision`. Ensure the i2i image dream passes the **source image** into the worker job data (the field the fal image handler reads). The i2i variation dream's prompt carries the source image (URL or source dream uuid); thread it through.

### Frontend (`frontend` repo)

1. The variation **i2i** method builds a payload selecting the fal i2i model and the source image:
   `{ infinidream_algorithm: 'flux-kontext-i2i', prompt, source_dream_uuid|image: <parent keyframe image>, seed }`.
   No `userEndpointUuid`. Provider key resolves server-side.
2. The source image is the parent keyframe's image (already in flow store). Use the field name the backend/worker expect (align with `dream.util` ↔ fal-handlers — likely `source_dream_uuid` or `image`).
3. i2i method availability: no endpoint gating. It works whenever a fal key (personal or global) is resolvable. Optionally surface "uses your fal key if set, else platform key." Confirm credit/availability behavior with `resolveProviderKeyDecision` + admin-credits.

## Conflict-risk ranking (frontend)

1. **`flow.store.ts`** — extend `reconcileStaleTransitions` + `flowPartialize`. Variation actions additive.
2. **`useFlowJobProgress.ts`** — stage rewrote it heavily; re-thread variation progress + candidate swap-in. (PR did **not** touch `useStudioJobProgress.ts`.)
3. **`transition-settings-panel.tsx`** — stage +148 vs PR +23; re-apply only variation/expansion controls.
4. **`images-tab.tsx`** — stage +68 vs PR +171; reconcile after BYO strip.

## Testing

- Frontend unit tests verbatim: `variation-status.test.ts`, `expand-prompt.test.ts`, plus variation cases from `flow.store.test.ts` / `flow.types.test.ts`. Skip PR's session-store tests (stage owns sessions).
- Worker: unit-test the Kontext input builder (passes `image_url`) and the i2i handler image plumbing.
- Backend: catalog + pricing + routing unit coverage for the new model.
- `pnpm run type-check` + `pnpm run lint` (frontend), build/tests (backend, worker).
- Manual on staging: configure a fal key (or rely on global), run an i2i variation from a keyframe, confirm a *distinct* re-imaged result (not the source), accept/discard candidates, confirm candidates never enter the timeline/generation, verify prompt-expansion badge, verify variation status survives a session switch.

## Out of scope

- **OpenAI i2i** (gpt-image-1) — follow-up; requires a second worker provider + backend `ProviderName`/validator/pricing + frontend OpenAI key UI.
- Batch-mode i2i beyond the variation flow.
- Any change to stage's session system.

## Confirm during implementation

- Exact fal FLUX.1 Kontext endpoint slug + its input contract (`image_url`, optional `guidance_scale`/`strength`).
- Pricing model for Kontext (`perImage` vs fixed-size `perMegapixel`) and its credit handling.
- The exact source-image field name the backend↔worker agree on for fal i2i (`source_dream_uuid` vs `image`).
