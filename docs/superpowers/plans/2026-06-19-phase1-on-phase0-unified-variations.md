# Phase-1 on Phase-0: Unified Variations — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebase phase-1 (`feat/api-key-config-i2i`) onto phase-0 (`feat/phase-0-variations-sessions-expansion`), then reconcile the two variation systems into one — phase-1's `i2iCandidate` below-row display driven by phase-0's Variation Settings (presets/seed/expansion/source-prompt) and BYO i2i endpoints — so future phase-0 tweaks flow forward.

**Architecture:** Rebase-first (phase-0 is the base; it already has the Variation Settings panel, presets, controllable seed, prompt expansion, and sessions). Re-apply phase-1's i2i-specific deltas (BYO endpoints, `i2iCandidate` model, below-row, Accept/Discard) during conflict resolution. Then a small set of concrete edits unifies the two: drop phase-0's `variation-grid` popover path, route the per-keyframe Vary through the Variation Settings, un-gate Vary, and add BYO endpoints to the model dropdown.

**Tech Stack:** React 18 + TS, Zustand, styled-components, Vite, vitest, ESLint/Prettier. Worktree: `/Users/maxcarlsonold/edream/frontend/.worktrees/feat-api-key-config` (branch `feat/api-key-config-i2i`).

## Global Constraints

- Package manager: **pnpm**. Quality gates (must pass before push): `pnpm run type-check`, `pnpm run prettier:check`, `pnpm run lint`, `pnpm run build`.
- Errors → **Bugsnag**, never `console.error` (per project anti-rationalization rules).
- Read volatile store data via `getState()` inside callbacks; only subscribe to stable action refs.
- `feat/api-key-config-i2i` is the **open PR #655** branch — the rebase requires a **force-push**, which rewrites shared history. **Confirm with the user before force-pushing** (Task C2).
- Keep phase-1's deltas isolated/minimal so future phase-0→phase-1 rebases stay clean.

---

## Phase A — Rebase phase-1 onto phase-0

### Task A1: Prep + start the rebase

**Files:** none (git only). Worktree: `frontend/.worktrees/feat-api-key-config`.

- [ ] **Step 1: Verify clean tree + up-to-date refs**

```bash
cd /Users/maxcarlsonold/edream/frontend/.worktrees/feat-api-key-config
git status --short            # expect empty
git fetch origin
git log --oneline -1 origin/feat/phase-0-variations-sessions-expansion   # base tip (>= 88076aa)
```
Expected: clean tree; phase-0 tip is the latest (`88076aa` or newer).

- [ ] **Step 2: Create a safety backup branch**

```bash
git branch backup/api-key-config-pre-rebase-2026-06-19
```
Expected: backup branch created (recovery point if the rebase goes wrong).

- [ ] **Step 3: Start the rebase onto phase-0**

```bash
git rebase origin/feat/phase-0-variations-sessions-expansion
```
Expected: rebase begins; stops at the first conflicting commit (the i2i integration commits). Conflicting files will include: `flow-builder.tsx`, `images-tab.tsx`, `keyframe-card.tsx`, `keyframe-card.styled.tsx`, `keyframe-strip.tsx`, `stores/flow.store.ts`, `stores/studio.store.ts`, `types/flow.types.ts`, `transition-settings-panel.tsx`.

- [ ] **Step 4: (no commit — rebase in progress; resolution is Task A2)**

### Task A2: Resolve conflicts (phase-0 base + re-apply phase-1 i2i deltas)

**Files (conflict resolution per file):**
- `src/types/flow.types.ts` — UNION both: keep phase-0's `variations?: VariationCandidate[]` AND phase-1's `i2iCandidate?: boolean` / `i2iParentId?: string` on `FlowKeyframe`.
- `src/stores/flow.store.ts` — keep phase-0's variation actions + `reconcileStaleTransitions` (variation-aware) AND phase-1's `addI2iCandidates` / `acceptI2iCandidate` / `discardI2iCandidate` / `setI2iEndpoint`. Both sets of actions coexist.
- `src/stores/studio.store.ts` — keep phase-0's `variationPresetId` / `variationCustomPrompt` / `variationSeed` (+ defaults, partialize) AND phase-1's user-endpoint additions.
- `src/components/pages/studio/components/flow-builder.tsx` — keep phase-0 structure (renders `VariationSettingsPanel`); keep phase-1's `handleI2iVariation`, `addI2iCandidates`, `i2iEndpoints`, `setI2iEndpoint`, `variationLightboxIndex`. (Unification of the two handlers is Task B2 — for now keep both compiling.)
- `src/components/pages/studio/components/keyframe-card.tsx` / `.styled.tsx` — keep phase-1's `VaryButton` + `CandidateBadge` (top-left) + `CandidateActions` pill; keep phase-0's `CardLabel`/index logic. Resolve the `VariationsButton` (phase-0 shuffle) vs `VaryButton` (phase-1) collision by keeping **both** exports for now (shuffle removed in Task B1).
- `src/components/pages/studio/components/keyframe-strip.tsx` — keep phase-1's `VariationsSection`/`VariationsRow` (below-row). Note phase-0 removed `FlowReset` here (88076aa) — keep it removed.
- `src/components/pages/studio/components/transition-settings-panel.tsx` — accept phase-0 version (its styled exports are reused by the Variation Settings panel).
- `src/components/pages/studio/components/images-tab.tsx` — accept phase-0 (expansion + pill placement) and re-apply any phase-1-only image-tab additions if present.

- [ ] **Step 1: Resolve each conflicted file per the policy above**

For each file in `git diff --name-only --diff-filter=U`, edit to remove conflict markers, keeping phase-0 base + phase-1 i2i deltas. Then `git add <file>`.

- [ ] **Step 2: Continue the rebase**

```bash
git rebase --continue
```
Repeat Step 1 for each subsequent conflicting commit until the rebase completes.
Expected: "Successfully rebased and updated refs/heads/feat/api-key-config-i2i".

- [ ] **Step 3: Type-check (compile gate after merge)**

```bash
pnpm run type-check
```
Expected: PASS. If it fails, fix the merge (most likely a missing field union in `flow.types.ts` or a store action dropped during resolution).

- [ ] **Step 4: Lint + build**

```bash
pnpm run lint && pnpm run build
```
Expected: both PASS. (Unused-symbol lint errors are expected if a phase-0 symbol is now orphaned — note them; they're cleaned in Phase B.)

- [ ] **Step 5: Commit is implicit (rebase already wrote commits).** Do NOT `git commit` — verify with `git log --oneline -12` that phase-1's commits now sit on top of the phase-0 tip.

---

## Phase B — Unify the two variation systems

After the rebase, both systems coexist (phase-0's `keyframe.variations` + `variation-grid` + shuffle `VariationsButton`, and phase-1's `i2iCandidate` + below-row + `VaryButton`). Converge onto: **phase-1's below-row `i2iCandidate` display, driven by phase-0's Variation Settings + BYO endpoints.**

### Task B1: Un-gate the Vary button + remove the shuffle/`variation-grid` path

**Files:**
- Modify: `src/components/pages/studio/components/keyframe-card.tsx`
- Modify: `src/components/pages/studio/components/keyframe-card.styled.tsx`
- Delete: `src/components/pages/studio/components/variation-grid.tsx`, `variation-grid.styled.tsx` (phase-0 popover — superseded)

**Interfaces:**
- Produces: `VaryButton` always-enabled (no `hasI2iEndpoints` gate). `onRequestVariations`/shuffle `VariationsButton` and `VariationGrid` no longer rendered.

- [ ] **Step 1: Remove the `disabled`/`hasI2iEndpoints` gate from `VaryButton`**

In `keyframe-card.tsx`, change the Vary button so it is always actionable:

```tsx
{!isLoop && !isCandidate && !isBusy && onRequestI2iVariation && (
  <VaryButton
    title="Generate variations"
    onClick={(e) => {
      e.stopPropagation();
      onRequestI2iVariation(keyframe);
    }}
  >
    Vary
  </VaryButton>
)}
```
Remove the now-unused `useUserApiEndpoints`/`hasI2iEndpoints` lines from this component.

- [ ] **Step 2: Remove phase-0's shuffle `VariationsButton` + `VariationGrid` usage**

Delete the `VariationsButton`, `showVariations` state, `onRequestVariations`, and `<VariationGrid .../>` JSX from `keyframe-card.tsx` (phase-0's popover path). Remove their imports.

- [ ] **Step 3: Delete the orphaned grid files**

```bash
git rm src/components/pages/studio/components/variation-grid.tsx \
       src/components/pages/studio/components/variation-grid.styled.tsx
```
Also remove the `VariationsButton` styled export from `keyframe-card.styled.tsx`.

- [ ] **Step 4: Type-check + lint**

```bash
pnpm run type-check && pnpm run lint
```
Expected: PASS (no remaining refs to `VariationGrid`/`VariationsButton`/`onRequestVariations`).

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat(studio): un-gate Vary; drop phase-0 variation-grid popover"
```

### Task B2: Drive `handleI2iVariation` from the Variation Settings

**Files:**
- Modify: `src/components/pages/studio/components/flow-builder.tsx` (`handleI2iVariation`)

**Interfaces:**
- Consumes (from phase-0, now present): `useStudioStore` fields `variationPresetId`, `variationCustomPrompt`, `variationSeed`; `getVariationPreset(id)` and `expandPrompt(template)` from `../constants/variation-presets` and `../utils/expand-prompt`.
- Produces: `handleI2iVariation(keyframe)` generates candidates using preset/custom modifiers + source-image prompt + stable seed (+offset), instead of the hardcoded `VARIATION_PROMPTS`.

- [ ] **Step 1: Replace the hardcoded `VARIATION_PROMPTS` block**

In `handleI2iVariation`, read settings at call time and build modifiers (mirrors phase-0's flow-builder logic):

```ts
const { imagePrompt, imageGenParams, variationPresetId, variationCustomPrompt, variationSeed } =
  useStudioStore.getState();

const customModifiers = variationCustomPrompt
  .split("\n").map((l) => l.trim()).filter(Boolean)
  .flatMap((l) => expandPrompt(l));
const modifiers = (
  customModifiers.length > 0 ? customModifiers : getVariationPreset(variationPresetId).modifiers
).slice(0, 8);

const VARIATION_COUNT = modifiers.length;
const existingCount = keyframe.variations?.length ?? 0;   // existing i2i candidates for this parent
const baseSeed = variationSeed + existingCount;
```

- [ ] **Step 2: Layer the source-image prompt as the base**

Fetch the source dream prompt (carry over phase-0's exact block) and build each candidate's prompt as `` `${basePrompt}, ${modifiers[i]}` `` with `seed: baseSeed`. Use `existingCount` to count this keyframe's current candidates (from the store) so `+More` offsets.

- [ ] **Step 3: Add the import**

```ts
import { getVariationPreset } from "../constants/variation-presets";
import { expandPrompt } from "../utils/expand-prompt";
```

- [ ] **Step 4: Type-check + lint**

```bash
pnpm run type-check && pnpm run lint
```
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/components/pages/studio/components/flow-builder.tsx
git commit -m "feat(studio): drive i2i variations from Variation Settings (presets/seed/expansion/source prompt)"
```

### Task B3: Unify the model dropdown (built-in + BYO i2i endpoints)

**Files:**
- Modify: `src/components/pages/studio/components/variation-settings-panel.tsx`
- Modify: `src/components/pages/studio/components/flow-builder.tsx` (route by selected model)

**Interfaces:**
- Consumes: `useUserApiEndpoints()` (phase-1) → endpoints with `capabilities.imageToImage`.
- Produces: the Variation Settings "Model" dropdown lists built-in image models AND BYO i2i endpoints; `handleI2iVariation` routes to the i2i endpoint when a BYO endpoint is selected, else built-in generation.

- [ ] **Step 1: Add BYO endpoints to the Model `<select>`**

In `variation-settings-panel.tsx`, append an `<optgroup label="My i2i endpoints">` built from `useUserApiEndpoints()` filtered by `capabilities.imageToImage`, with option values prefixed (e.g. `endpoint:<uuid>`). On change, if the value starts with `endpoint:` call `setI2iEndpoint(uuid)` (flow store) and leave `imageGenParams.model` as-is; else clear the i2i endpoint and set the built-in model.

- [ ] **Step 2: Route generation in `handleI2iVariation`**

When an i2i endpoint is selected (`useFlowStore.getState().i2iEndpointUuid` set), build the candidate prompt as today but route via the i2i endpoint params (reuse phase-1's existing i2i `algoParams` shape with `userEndpointUuid` + source image URL). Otherwise use the built-in image-model `algoParams`.

- [ ] **Step 3: Type-check + lint**

```bash
pnpm run type-check && pnpm run lint
```
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add src/components/pages/studio/components/variation-settings-panel.tsx \
        src/components/pages/studio/components/flow-builder.tsx
git commit -m "feat(studio): unify variation model dropdown (built-in models + BYO i2i endpoints)"
```

---

## Phase C — Verify + ship

### Task C1: Full gates + manual test matrix

- [ ] **Step 1: Run all CI gates**

```bash
pnpm run type-check && pnpm run prettier:check && pnpm run lint && pnpm run build
```
Expected: all PASS. Fix any failures inline.

- [ ] **Step 2: Manual smoke (against a backend with the i2i feature)**

Verify, in Flow mode:
1. Add a keyframe → **Vary works with NO endpoint** (built-in model) → candidates appear in the below row.
2. Configure a BYO i2i endpoint → select it in the Variation Settings Model dropdown → Vary → candidates generated via i2i.
3. Accept promotes a candidate to a timeline keyframe; Discard removes it.
4. `+More` yields new (non-duplicate) candidates.
5. Switch sessions with in-flight variations → they rehydrate (not marked failed).

- [ ] **Step 3: Run unit tests**

```bash
pnpm run test
```
Expected: PASS (existing flow.store reconcile/rehydration tests + any added).

### Task C2: Force-push the rebased PR branch (CONFIRM FIRST)

- [ ] **Step 1: Confirm with the user** that force-pushing PR #655 (rewrites history) is OK.

- [ ] **Step 2: Force-push with lease**

```bash
git push --force-with-lease origin feat/api-key-config-i2i
```
Expected: PR #655 updated; now based on phase-0.

- [ ] **Step 3: Verify the PR** shows phase-0 as ancestor and CI is green.

---

## Self-Review notes

- **Spec coverage:** rebase (Phase A) ✓; keep `i2iCandidate` + below-row (A2, B1) ✓; port Variation Settings driving i2i (B2) ✓; unify model dropdown + un-gate Vary (B1, B3) ✓; drop `variation-grid` (B1) ✓; conflict policy (A2) ✓; force-push w/ confirm (C2) ✓; sessions inherited via rebase (no separate task) ✓.
- **Interactive caveat:** Phase A conflict hunks cannot be pre-written (rebase output is not deterministic); A2 gives the per-file resolution policy + compile/lint gates to validate. Phase B edits are concrete because they apply to known post-rebase files.
- **Order refinement vs spec:** rebase-first (not "unify-then-rebase") — phase-0 already carries the Variation Settings/presets/seed/sessions, so re-applying phase-1's i2i deltas onto it avoids duplicating phase-0 work.
