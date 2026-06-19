# Phase-1 on Phase-0: unified variation system + branch stacking

- **Date:** 2026-06-19
- **Status:** Design — awaiting review
- **Goal:** Rebase phase-1 (`feat/api-key-config-i2i`) onto phase-0 (`feat/phase-0-variations-sessions-expansion`) so phase-0 tweaks flow forward, by converging the two divergent "variation" systems into one.

## Why

Both branches forked from the same `stage` commit and **independently rework the same studio files** (`flow-builder`, `keyframe-card(.styled)`, `keyframe-strip`, `flow.store`, `studio.store`, `flow.types`) with **different variation systems**:

- **phase-0:** `keyframe.variations[]` sub-array + shuffle `VariationsButton` + `variation-grid` popover; Variation Settings panel (canned presets, controllable seed, `{a|b|c}` expansion, source-prompt inheritance); sessions.
- **phase-1:** `i2iCandidate` staging keyframes rendered in a **"Variations" row below** (`keyframe-strip` `VariationsSection`) with Accept/Discard; BYO i2i endpoints; gated `Vary` button.

A straight rebase conflicts heavily and irreconcilably (two "variations button" components, two data models). Converging them first makes the rebase clean and keeps future phase-0 tweaks mergeable.

## Decisions (approved)

- **Target:** phase-1. **Display:** keep phase-1's below-row (`i2iCandidate` + Accept/Discard) — preferred over `variation-grid`.
- **One unified system** supporting both: canned presets/seed/expansion **and** optional BYO i2i endpoint as the "model".
- **Data model:** keep `i2iCandidate` (reuses phase-1's tested gating, Accept/Discard, `partialize`, below-row rendering); layer phase-0's *generation richness* on top. (Chosen over adopting `keyframe.variations`, which would rework all of that.)

## Design

### Components
- Port phase-0's **Variation Settings panel** into phase-1's Flow view: Model · Variation style · Seed + collapsible Customize (canned presets, `{a|b|c}` expansion, live count). Already built on phase-0; carry over.
- **Unify the Model dropdown:** list built-in image models **and** configured BYO i2i endpoints (`useUserApiEndpoints`, `imageToImage`). Built-in → seed/preset/expansion generation; BYO endpoint → i2i variation via that endpoint.
- Drop phase-0's `variation-grid` popover + shuffle `VariationsButton` (superseded by phase-1's below-row + the panel).

### Generation behavior
- One **Vary** trigger per keyframe → N `i2iCandidate` keyframes in the below row, built from Variation Settings: preset modifiers **or** custom prompts, **source-image-prompt** as base, **stable seed** (constant default, session-scoped) with **+More seed offset** so repeated presses yield new images.
- **Un-gate Vary:** today disabled without an i2i endpoint. After: always available — built-in model by default; i2i only when a BYO endpoint is selected as the model.

### Data flow
Unchanged from phase-1 downstream: candidate dream created → `buildVideoAlgoParams` → backend → worker. i2i path uses the BYO endpoint queue; built-in path uses the normal image/video model path.

### Display / lifecycle
Unchanged from phase-1: candidates in the "Variations" row; Accept promotes to a timeline keyframe; Discard removes; candidates excluded from timeline/transitions/generation/`partialize`.

## Rebase / stacking approach

1. Land the unified-variation changes **on phase-1** first (so its variation code resembles phase-0's foundation).
2. Rebase `feat/api-key-config-i2i` onto `feat/phase-0-variations-sessions-expansion`. Conflict resolution policy:
   - **Shared studio scaffolding** (stores, flow-builder structure, keyframe-card layout, Variation Settings panel): take **phase-0** as the base.
   - **i2i-specific bits** (BYO endpoint integration, `i2iCandidate` gating, Accept/Discard, un-gated Vary): re-apply **phase-1** on top.
3. Force-push the rebased phase-1 (open PR #655) — confirm before doing so.
4. Going forward: tweak phase-0 → rebase phase-1 onto updated phase-0. Keeping the variation code converged minimizes recurring conflicts.

## Out of scope (separate spec)
- Sessions port to phase-1 (#2) — phase-0 already has it; after the rebase phase-1 inherits it, so no separate port may be needed.
- LTX-2.3 camera-LoRA fix — see `docs/plans/2026-06-19-ltx-2.3-camera-lora-not-applied.md`.

## Testing
- Type-check + lint + build (CI gates).
- Unit: variation prompt assembly (preset vs custom, expansion, seed offset); reconcile/rehydration (already covered).
- Manual: Vary with no endpoint (built-in), Vary with BYO i2i endpoint, Accept/Discard, +More uniqueness, session switch with in-flight variations.

## Risks
- Force-pushing the open PR #655 (rewrites shared history) — confirm first.
- Recurring rebase conflicts if the variation code diverges again — mitigated by converging it now and keeping phase-1's deltas minimal/isolated.
