# Studio i2i Variations + Prompt Expansion — Re-apply onto stage

**Date:** 2026-06-28
**Status:** Draft
**Scope:** Frontend only (`frontend` repo)
**Supersedes (re-scopes):** `docs/superpowers/specs/2026-06-12-api-key-config-i2i-variations-design.md`
**Source branch:** `feat/api-key-config-i2i` (the existing ~40-commit PR)
**Target base:** `origin/stage`
**New branch:** `feat/studio-i2i-variations`

## Context

The `feat/api-key-config-i2i` PR was built on a snapshot of `stage` (merge-base `7160fcf`). Since then, competing work landed on `stage` that overlaps the PR in three places:

| Area | What stage shipped | What the PR had | Resolution |
|------|--------------------|-----------------|------------|
| **User API keys** | `provider-key` — per-provider BYO keys (fal.ai/OpenAI), mounted in `profile-card.tsx` (#661) | `user-api-endpoints` — per-endpoint records + own profile section | **Drop PR's entirely** |
| **Sessions** | `keyframes-sessions` (#659) — its own `session.store`, `session-switcher`, `session.types`, `useSessionAutoSave` | a parallel implementation of the same files | **Drop PR's, adopt stage's** |
| **Studio internals** | reworked `transition-settings-panel` (+148), `useFlowJobProgress`, `useStudioJobProgress`, `flow.store` persistence, model catalog UI | the PR also rewrote these | **Adopt stage's, graft only variation/expansion wiring** |

Because much of the 40-commit PR builds the dropped settings feature and a now-superseded session system, a straight `git rebase origin/stage` would force hand-resolution of hundreds of conflict lines in code we would then discard. Instead we **re-apply only the two genuinely-unique sub-features** — i2i variations and prompt expansion — onto stage's current studio code, as a fresh branch.

## Decisions (locked with user)

| Decision | Choice |
|----------|--------|
| Branch strategy | New branch `feat/studio-i2i-variations` off `origin/stage`; old branch kept as backup; fresh PR |
| BYO i2i endpoints | **Strip.** Variation picker offers built-in i2i models only. Remove all `useUserApiEndpoints` / `UserApiEndpoint` / `endpoint-presets` reads and stop sending `userEndpointUuid`. BYO can be re-added later atop stage's `provider-key` model. |
| Sessions | Adopt stage's session system as-is. Drop the PR's session files. |
| Keep set | i2i variations + prompt expansion only. |

## Keep set (re-apply)

**Verbatim-copyable new files** (do not exist on stage, no conflict):
- `components/pages/studio/components/variation-grid.tsx` + `.styled.tsx`
- `components/pages/studio/components/variation-lightbox.tsx` + `.styled.tsx`
- `components/pages/studio/components/variation-settings-panel.tsx` *(strip BYO — see below)*
- `components/pages/studio/components/prompt-expansion-badge.tsx`
- `components/pages/studio/constants/variation-presets.ts`
- `components/pages/studio/utils/variation-status.ts` + `__tests__/variation-status.test.ts`
- `components/pages/studio/utils/expand-prompt.ts` + `__tests__/expand-prompt.test.ts`

**Additive type grafts** — `src/types/flow.types.ts` (stage never touched this file; applies clean):
- `FlowKeyframe`: `i2iCandidate?`, `i2iParentId?`, `i2iStatus?`, `variations?: VariationCandidate[]`, `activeVariationId?`
- `FlowTransition`: `variations?: VariationCandidate[]`, `activeVariationId?`
- new `VariationCandidate` interface (`method: "seed" | "expansion" | "i2i"`, etc.)

**Store action grafts** — `src/stores/flow.store.ts` (stage has zero variation refs; the `set()` action blocks are additive):
- i2i candidate staging: `addI2iCandidates`, `acceptI2iCandidate`, `discardI2iCandidate`
- keyframe variations: `addKeyframeVariations`, `selectKeyframeVariation`, `clearKeyframeVariations`, `updateKeyframeVariation`
- transition variations: `addTransitionVariations`, `selectTransitionVariation`, `clearTransitionVariations`, `updateTransitionVariation`
- timeline-derivation helper must filter out `i2iCandidate` keyframes so candidates never produce transitions/generation. **Reconcile with stage's derivation code** if stage changed that helper.
- **Drop** `i2iEndpointUuid` / `setI2iEndpoint` (BYO state).

**Wiring grafts into stage's versions** (KEEP hunks only; discard session/endpoint hunks):
- `flow-builder.tsx` — variation generation triggers, grid mount; **remove** `useUserApiEndpoints` + `userEndpointUuid` payload.
- `images-tab.tsx` — variation candidate cards; **remove** the endpoint `<select>`, `useUserApiEndpoints`, `selectedEndpoint`, `userEndpointUuid`.
- `keyframe-card.tsx` + `.styled.tsx` — variation grid/shuffle button.
- `keyframe-strip.tsx` + `.styled.tsx` — exclude i2i candidates from gap-mapping/sorting.
- `generate-tab.tsx`, `actions-tab.tsx`, `transition-gap.tsx`, `transition-settings-panel.tsx`, `studio.page.tsx` + `.styled.tsx` — prompt-expansion + variation mount points.
- Hooks `useFlowGeneration.ts`, `useFlowJobProgress.ts`, `useBatchSubmit.ts` — variation progress tracking + candidate result swap-in.

## The one non-mechanical graft: variation rehydration

The PR placed its variation **rehydration-reconciliation** (re-checking in-flight variation status after a session switch / reload) inside `flow.store`'s persist/`partialize` layer. Stage **removed** `flow.store`'s own `partialize` and moved persistence into its session system. So the reconciliation logic (`reconcileVariations` over keyframe + transition `variations`) must be **relocated** to stage's session-rehydration path rather than reapplied to `flow.store`'s persist config. This is the highest-risk integration point and gets its own plan step + manual verification.

## Drop set (do not carry over)

- `src/api/user-api-endpoints/*` (5 hooks)
- `src/components/pages/profile/add-endpoint-modal*.tsx`, `api-endpoints-section*.tsx`
- `src/components/shared/profile-card/profile-card.tsx` (PR's edit — keep stage's provider-key version)
- `src/constants/endpoint-presets.ts`
- `src/types/user-api-endpoint.types.ts`
- PR's `session.store.ts`, `session-switcher.tsx(.styled)`, `session.types.ts`, `useSessionAutoSave.ts`
- PR's `src/main.tsx` / `src/App.tsx` edits **except** any route/provider genuinely required by variations (verify; the +107 `main.tsx` change is mostly endpoint/query wiring — re-add to stage's `main.tsx` only what variations need, if anything).

## BYO strip detail — `variation-settings-panel.tsx`

Remove `import { useUserApiEndpoints }`, the `useUserApiEndpoints()` call, and any endpoint-selection UI/state. The panel keeps presets, seed control, expansion toggle, and source-prompt — driven entirely by built-in models. Same strip applies to `flow-builder.tsx` (line ~253 `userEndpointUuid`) and `images-tab.tsx` (endpoint `<select>`, lines ~84–106, ~145–158, ~272–290).

## Conflict-risk ranking

1. **`flow.store.ts`** — variation rehydration relocation (above). Actions themselves additive.
2. **`useFlowJobProgress.ts` / `useStudioJobProgress.ts`** — stage rewrote heavily; variation progress + candidate swap-in must be re-threaded onto stage's logic.
3. **`transition-settings-panel.tsx`** — stage +148 vs PR +23; re-apply only the variation/expansion controls.
4. **`images-tab.tsx`** — stage +68 vs PR +171; reconcile after BYO strip.
5. Everything else is low-risk additive.

## Testing

- Port the unit tests verbatim: `variation-status.test.ts`, `expand-prompt.test.ts`, plus the variation-relevant cases from `flow.store.test.ts` and `flow.types.test.ts` (skip session-store tests — stage owns sessions).
- `pnpm run type-check` and `pnpm run lint` must pass.
- Manual verification on staging: generate keyframe variations on a built-in i2i model, accept/discard candidates, verify candidates never appear in the timeline/generation, verify prompt expansion badge, verify variation status survives a session switch (the rehydration graft).

## Out of scope

- Re-pointing variations at stage's `provider-key` model (future).
- Any backend/worker change. This is frontend-only.
- Batch-mode BYO endpoint integration.

## Open verification (resolve during implementation)

- **Does i2i variation work on a built-in model without a BYO endpoint?** The variation methods are `seed`, `expansion`, `i2i`. `seed` and `expansion` need no endpoint. If the `i2i` method was *only* reachable via a BYO custom endpoint, stripping BYO degrades the variation feature to seed + expansion until a built-in i2i model is wired. Confirm what the backend exposes before deciding whether the i2i method stays in the picker or is hidden in this pass.
