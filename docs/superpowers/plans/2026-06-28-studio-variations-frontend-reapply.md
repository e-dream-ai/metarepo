# Studio Variations Frontend Re-apply — Implementation Plan (Part 1)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Re-apply the studio **variations (prompt-expansion variations, with a controllable seed)** and the **prompt-expansion badge** UX onto `origin/stage`, dropping the PR's user-settings and session systems (stage owns both). Note: the only variation method the PR actually *emits* is `"expansion"` (the `"seed"` member of `VariationCandidate.method` is a type-level value with no producer; `variationSeed`/`DEFAULT_VARIATION_SEED` is a stable seed *value* layered into expansion batches, not a separate method). The `i2i` method ships **unreachable** here; it is enabled in Part 2.

**Architecture:** Fresh branch off `origin/stage`. Verbatim-copy the genuinely-new files from the PR branch `feat/api-key-config-i2i`; additively graft variation types/actions/UI-state onto stage's `flow.store`/`studio.store`/`flow.types`; reconcile the handful of files stage also rewrote; strip all BYO-endpoint reads (replaced server-side in Part 2).

**Tech Stack:** React 18, Vite, TypeScript, Zustand, React Query, vitest, eslint.

**Spec:** `docs/superpowers/specs/2026-06-28-studio-i2i-variations-on-stage-design.md` (Part 1 + Frontend collisions).

## Global Constraints

- Package manager: **pnpm** (never npm/yarn).
- Repo: `/Users/maxcarlsonold/edream/frontend`. Source-of-truth for copies: branch `feat/api-key-config-i2i`. Base: `origin/stage`.
- Verify commands: `pnpm run type-check` (`tsc --noEmit`), `pnpm run lint` (`eslint . --ext .js,.ts`), `pnpm test` (`vitest run`).
- **`pnpm run lint` does NOT cover `.tsx`** (its `--ext` excludes `.tsx`). For wiring tasks, **`pnpm run type-check` is the authoritative gate** for `.tsx` correctness; treat a clean `type-check` as the pass signal, not lint.
- **BYO removal is "until grep is clean":** when stripping `useUserApiEndpoints`/`userEndpointUuid`/`i2iEndpoint*`/`UserApiEndpoint`/`selectedEndpoint` from a file, remove **every** resulting dangling reference (state, `useMemo`/`useCallback` deps, guards, JSX) until that file's `grep -nE "useUserApiEndpoints|UserApiEndpoint|userEndpointUuid|i2iEndpoint|selectedEndpoint"` returns nothing. The line numbers cited in tasks are starting points, not the complete set.
- **Errors → Bugsnag**, never `console.error` (project rule).
- **Never persist unsettled records**: `partialize` must exclude `i2iCandidate` keyframes and any record without a backend UUID.
- **No `userEndpointUuid`, no `useUserApiEndpoints`, no `endpoint-presets`** anywhere — that feature is dropped.
- Read volatile store data via `getState()` inside callbacks; only subscribe to stable action refs (avoids re-render storms).
- The `i2i` variation **method** is hidden in this plan (no built-in image path until Part 2).

---

### Task 0: Branch setup

**Files:** none (git only).

- [ ] **Step 1: Fetch and branch off stage**

```bash
cd /Users/maxcarlsonold/edream/frontend
git fetch origin --prune
git switch -c feat/studio-i2i-variations origin/stage
```

- [ ] **Step 2: Verify clean baseline**

Run: `pnpm install && pnpm run type-check && pnpm test`
Expected: install succeeds; type-check clean; tests PASS (stage baseline green).

- [ ] **Step 3: Commit marker (empty)**

```bash
git commit --allow-empty -m "chore: start studio variations re-apply on stage"
```

---

### Task 1: Graft variation types into `flow.types.ts`

Stage never touched `flow.types.ts`, so this is purely additive.

**Files:**
- Modify: `src/types/flow.types.ts`
- Test: `src/types/__tests__/flow.types.test.ts` (port variation cases from PR)

**Interfaces — Produces:** `VariationCandidate`, and the new optional fields on `FlowKeyframe` / `FlowTransition` consumed by Tasks 3–11.

- [ ] **Step 1: Apply the PR's additive type hunks**

Add to `FlowKeyframe`: `i2iCandidate?: boolean`, `i2iParentId?: string`, `i2iStatus?: "queue"|"processing"|"processed"|"failed"`, `variations?: VariationCandidate[]`, `activeVariationId?: string`.
Add to `FlowTransition`: `variations?: VariationCandidate[]`, `activeVariationId?: string`.
Add the new interface:

```ts
export interface VariationCandidate {
  id: string;
  method: "seed" | "expansion" | "i2i";
  prompt?: string;
  seed?: number;
  dreamUuid?: string;
  imageUrl?: string;
  status: TransitionStatus;
  progress?: number;
}
```

Reference the exact source: `git show feat/api-key-config-i2i:src/types/flow.types.ts`.

- [ ] **Step 2: Port the variation type tests**

Copy only the variation-related cases from the PR's `flow.types.test.ts` (skip any referencing session/endpoint types).
Run: `git show feat/api-key-config-i2i:src/types/__tests__/flow.types.test.ts` and lift the `VariationCandidate` / `i2iCandidate` cases.

- [ ] **Step 3: Type-check + test**

Run: `pnpm run type-check && pnpm test src/types`
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add src/types/flow.types.ts src/types/__tests__/flow.types.test.ts
git commit -m "feat(studio): add variation types to flow.types"
```

---

### Task 2: Copy verbatim util modules (`expand-prompt`, `variation-status`) + tests

These files don't exist on stage and have no dependency on dropped code — copy verbatim.

**Files:**
- Create: `src/components/pages/studio/utils/expand-prompt.ts` (+ `__tests__/expand-prompt.test.ts`)
- Create: `src/components/pages/studio/utils/variation-status.ts` (+ `__tests__/variation-status.test.ts`)

**Interfaces — Produces:** the `expand-prompt` and `variation-status` exports consumed by Tasks 3, 6, 11.

- [ ] **Step 1: Copy the four files verbatim from the PR branch**

```bash
for f in \
  src/components/pages/studio/utils/expand-prompt.ts \
  src/components/pages/studio/utils/__tests__/expand-prompt.test.ts \
  src/components/pages/studio/utils/variation-status.ts \
  src/components/pages/studio/utils/__tests__/variation-status.test.ts; do
  git show "feat/api-key-config-i2i:$f" > "$f"
done
```

- [ ] **Step 2: Run the ported tests (must pass as-is)**

Run: `pnpm test src/components/pages/studio/utils`
Expected: PASS (these are self-contained unit tests).

- [ ] **Step 3: Commit**

```bash
git add src/components/pages/studio/utils
git commit -m "feat(studio): add expand-prompt + variation-status utils"
```

---

### Task 3: Graft variation + i2i-candidate actions into `flow.store.ts`

Stage's `flow.store` has zero variation refs; the action blocks are additive. The reconcile + partialize edits extend stage's existing code.

**Files:**
- Modify: `src/stores/flow.store.ts`
- Test: `src/stores/flow.store.test.ts` (port variation cases)

**Interfaces — Produces (consumed by Tasks 6–11):**
`addI2iCandidates(parentId, candidates)`, `acceptI2iCandidate(id)`, `discardI2iCandidate(id)`, `addKeyframeVariations(keyframeId, candidates)`, `selectKeyframeVariation(keyframeId, variationId)`, `clearKeyframeVariations(keyframeId)`, `updateKeyframeVariation(keyframeId, variationId, patch)`, `addTransitionVariations(transitionIndex, candidates)`, `selectTransitionVariation(transitionIndex, variationId)`, `clearTransitionVariations(transitionIndex)`, `updateTransitionVariation(transitionIndex, variationId, patch)`.

- [ ] **Step 1: Write failing tests for the additive actions**

Port from `git show feat/api-key-config-i2i:src/stores/flow.store.test.ts` only the cases for the actions above plus: "i2i candidates are excluded from derived transitions", "accept promotes a candidate", "discard removes a candidate", "selectKeyframeVariation only applies when status==processed". Drop any case referencing `i2iEndpointUuid`/`setI2iEndpoint` or sessions.

- [ ] **Step 2: Run tests — verify they fail**

Run: `pnpm test src/stores/flow.store.test.ts`
Expected: FAIL (actions undefined).

- [ ] **Step 3: Graft the action blocks**

Copy the action implementations verbatim from `git show feat/api-key-config-i2i:src/stores/flow.store.ts` for every name in the Produces list. **Omit** `i2iEndpointUuid` / `setI2iEndpoint` (BYO state — dropped). Ensure the timeline-derivation helper filters `!kf.i2iCandidate` (reconcile with stage's helper name — stage's derivation lives near the top of the file).

- [ ] **Step 4: Extend stage's `reconcileStaleTransitions`**

Inside stage's existing `reconcileStaleTransitions` action (`flow.store.ts:339`), add the PR's `reconcileVariations` helper and map it over keyframe + transition `variations`. Both rehydration entry points (`onRehydrateStorage:393`, `session.store.loadSession:201`) already call this action, so no other call sites change. Use the PR's **exact** body (PR `flow.store.ts:424–463`) — note it **fails** in-flight variations that have **no `dreamUuid`** (no recoverable backend job) and **keeps** those with a `dreamUuid` (the polling hook re-attaches). A no-op here ships permanent ghost "processing" spinners.

```ts
const reconcileVariations = (vars?: VariationCandidate[]) =>
  vars?.map((v) =>
    (v.status === "queue" || v.status === "processing") && !v.dreamUuid
      ? { ...v, status: "failed" as const, progress: undefined }
      : v,
  );
// within the existing set(): add  ...(t.variations && { variations: reconcileVariations(t.variations) })
// to the transition map, and map keyframes:  kf.variations ? { ...kf, variations: reconcileVariations(kf.variations) } : kf
```

- [ ] **Step 5: Port the full partialize variation graft**

Stage's exported `flowPartialize` (`flow.store.ts:150`) currently strips `variations`. Port the PR's partialize variation handling (PR `flow.store.ts:613–645`) onto it — three edits:
1. Keyframe `.filter(...)`: add `&& !kf.i2iCandidate` (staging candidates never persist).
2. Keyframe `.map(...)`: add `variations: kf.variations?.filter((v) => v.status !== "queue" && v.status !== "processing")` and `activeVariationId: kf.activeVariationId` (persist settled variations, not in-flight).
3. `transitions` map: `transitions: state.transitions.map((t) => ({ ...t, variations: t.variations?.filter((v) => v.status !== "queue" && v.status !== "processing") }))`.

Without (2)/(3), keyframe variations never survive a raw reload and in-flight transition variations get persisted then failed. (The T13 session-switch smoke routes through `session.store` full objects and would NOT catch this — verify a raw page reload too.)

- [ ] **Step 6: Run tests + type-check**

Run: `pnpm run type-check && pnpm test src/stores/flow.store.test.ts`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/stores/flow.store.ts src/stores/flow.store.test.ts
git commit -m "feat(studio): graft variation actions + reconcile/partialize into flow.store"
```

---

### Task 4: Graft variation UI state into `studio.store.ts`

**Files:**
- Modify: `src/stores/studio.store.ts`

**Interfaces — Produces (consumed by Task 6):** `variationPresetId` (defaults to `DEFAULT_VARIATION_PRESET_ID` from `variation-presets.ts`), `variationCustomPrompt`, `variationSeed` (defaults to `DEFAULT_VARIATION_SEED`), and their setters.

- [ ] **Step 1: Graft the state + setters**

Copy the variation state fields, setters, `DEFAULT_VARIATION_SEED`, and the `variationPresetId` default (`DEFAULT_VARIATION_PRESET_ID`, imported from `../constants/variation-presets` added in Task 5) from `git show feat/api-key-config-i2i:src/stores/studio.store.ts`.

- [ ] **Step 2: Persist the choices in `studioPartialize`**

Add `variationPresetId`, `variationCustomPrompt`, `variationSeed` to stage's `studioPartialize` (persist user choices, not transient UI).

- [ ] **Step 3: Type-check**

Run: `pnpm run type-check`
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add src/stores/studio.store.ts
git commit -m "feat(studio): add variation settings state to studio.store"
```

---

### Task 5: Copy verbatim presentational components

No dependency on dropped code (they consume Task 1 types + Task 3/4 store).

**Files (create, copy verbatim from the PR branch):**
- `src/components/pages/studio/components/variation-grid.tsx` + `.styled.tsx`
- `src/components/pages/studio/components/variation-lightbox.tsx` + `.styled.tsx`
- `src/components/pages/studio/components/prompt-expansion-badge.tsx`
- `src/components/pages/studio/constants/variation-presets.ts`

- [ ] **Step 1: Copy the files**

```bash
for f in \
  src/components/pages/studio/components/variation-grid.tsx \
  src/components/pages/studio/components/variation-grid.styled.tsx \
  src/components/pages/studio/components/variation-lightbox.tsx \
  src/components/pages/studio/components/variation-lightbox.styled.tsx \
  src/components/pages/studio/components/prompt-expansion-badge.tsx \
  src/components/pages/studio/constants/variation-presets.ts; do
  git show "feat/api-key-config-i2i:$f" > "$f"
done
```

- [ ] **Step 2: Type-check (imports resolve against Tasks 1/3/4)**

Run: `pnpm run type-check`
Expected: PASS (these only import flow types + store actions now present). If `variation-lightbox` imports a react-query video hook, confirm it exists on stage; fix the import path if stage renamed it.

- [ ] **Step 3: Commit**

```bash
git add src/components/pages/studio/components/variation-grid.* \
        src/components/pages/studio/components/variation-lightbox.* \
        src/components/pages/studio/components/prompt-expansion-badge.tsx \
        src/components/pages/studio/constants/variation-presets.ts
git commit -m "feat(studio): add variation grid/lightbox + prompt-expansion badge + presets"
```

---

### Task 6: Add `variation-settings-panel.tsx` with BYO stripped

**Files:**
- Create: `src/components/pages/studio/components/variation-settings-panel.tsx`

**Interfaces — Consumes:** Task 2 utils, Task 4 studio.store state, Task 3 store actions. **Produces:** the settings panel mounted in Task 10.

- [ ] **Step 1: Copy the PR file, then strip BYO**

Start from `git show feat/api-key-config-i2i:src/components/pages/studio/components/variation-settings-panel.tsx`. Then remove every BYO element:

- Delete `import { useUserApiEndpoints } from "@/api/user-api-endpoints/useUserApiEndpoints";` (line 4).
- Delete the `useUserApiEndpoints()` call and `i2iEndpoints` (lines ~56–57).
- Delete the `i2iEndpointUuid` / `setI2iEndpoint` store reads (lines ~60–61).
- In the model `<select>`: set `value={model}` (drop the `endpoint:` prefix branch) and simplify `onChange` to just set the built-in model (delete the `if (value.startsWith("endpoint:"))` block and the `setI2iEndpoint(...)` calls).
- Delete the `<optgroup label="My i2i endpoints">` block (lines ~109–115).

The panel keeps: presets dropdown, custom-prompt textarea, seed control, expansion preview. It no longer references endpoints.

- [ ] **Step 2: Type-check + lint**

Run: `pnpm run type-check && pnpm run lint -- src/components/pages/studio/components/variation-settings-panel.tsx`
Expected: PASS, and `grep -n "useUserApiEndpoints\|i2iEndpoint\|endpoint:" src/components/pages/studio/components/variation-settings-panel.tsx` returns nothing.

- [ ] **Step 3: Commit**

```bash
git add src/components/pages/studio/components/variation-settings-panel.tsx
git commit -m "feat(studio): variation settings panel (BYO endpoint reads stripped)"
```

---

### Task 7: Wire keyframe-card + keyframe-strip (variation grid, exclude candidates)

**Files:**
- Modify: `src/components/pages/studio/components/keyframe-card.tsx` + `.styled.tsx`
- Modify: `src/components/pages/studio/components/keyframe-strip.tsx` + `.styled.tsx`

- [ ] **Step 1: Apply the PR's keyframe-card variation hunks**

Diff source: `git diff 7160fcf feat/api-key-config-i2i -- src/components/pages/studio/components/keyframe-card.tsx src/components/pages/studio/components/keyframe-card.styled.tsx`. Apply the variation-grid mount + shuffle/Vary button hunks onto stage's current `keyframe-card`. Resolve conflicts by keeping stage's structure and inserting the variation UI.

- [ ] **Step 2: Apply the keyframe-strip candidate-exclusion hunks**

Diff source: `git diff 7160fcf feat/api-key-config-i2i -- src/components/pages/studio/components/keyframe-strip.tsx src/components/pages/studio/components/keyframe-strip.styled.tsx`. Apply the hunks that exclude `i2iCandidate` keyframes from gap-mapping/sorting.

- [ ] **Step 3: Type-check + lint + test**

Run: `pnpm run type-check && pnpm run lint && pnpm test src/components/pages/studio`
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add src/components/pages/studio/components/keyframe-card.* src/components/pages/studio/components/keyframe-strip.*
git commit -m "feat(studio): wire variation grid into keyframe card; exclude candidates from strip"
```

---

### Task 8: Wire `flow-builder.tsx` (variation triggers; i2i hidden)

**Files:**
- Modify: `src/components/pages/studio/components/flow-builder.tsx`

- [ ] **Step 1: Apply variation-generation hunks, drop BYO**

Diff source: `git diff 7160fcf feat/api-key-config-i2i -- src/components/pages/studio/components/flow-builder.tsx`. Apply the variation-grid mount + generation-trigger hunks. **Remove** the `useUserApiEndpoints` import (line 11), the `endpointsData` read (line 55), and the `userEndpointUuid` payload branch (line ~253). Keep only the built-in payload branch for `seed`/`expansion` variations.

- [ ] **Step 2: Verify no BYO residue**

Run: `grep -nE "useUserApiEndpoints|userEndpointUuid|i2iEndpoint" src/components/pages/studio/components/flow-builder.tsx`
Expected: no output.

- [ ] **Step 3: Type-check + lint**

Run: `pnpm run type-check && pnpm run lint`
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add src/components/pages/studio/components/flow-builder.tsx
git commit -m "feat(studio): wire variation generation into flow-builder (BYO dropped)"
```

---

### Task 9: Wire `images-tab.tsx` (remove endpoint select; candidate cards)

**Files:**
- Modify: `src/components/pages/studio/components/images-tab.tsx`

- [ ] **Step 1: Apply candidate-card hunks, strip endpoint `<select>`**

Diff source: `git diff 7160fcf feat/api-key-config-i2i -- src/components/pages/studio/components/images-tab.tsx`. Apply the i2i candidate-card display hunks onto stage's images-tab. **Remove**: `useUserApiEndpoints` import (line 50), `UserApiEndpoint` import (line 51), `endpointsData`/`userEndpoints` (lines 84–106), the endpoint `<select>` (lines ~272–290), and `userEndpointUuid` from the submit payload (line ~158). Reconcile against stage's +68 changes by keeping stage's structure.

- [ ] **Step 2: Verify no BYO residue + type-check**

Run: `grep -nE "useUserApiEndpoints|UserApiEndpoint|userEndpointUuid" src/components/pages/studio/components/images-tab.tsx` (expect none); then `pnpm run type-check && pnpm run lint`.
Expected: no residue; PASS.

- [ ] **Step 3: Commit**

```bash
git add src/components/pages/studio/components/images-tab.tsx
git commit -m "feat(studio): images-tab candidate cards (endpoint select removed)"
```

---

### Task 10: Wire remaining mount points (expansion + variation)

**Files:**
- Modify: `src/components/pages/studio/components/generate-tab.tsx`
- Modify: `src/components/pages/studio/components/actions-tab.tsx`
- Modify: `src/components/pages/studio/components/transition-gap.tsx`
- Modify: `src/components/pages/studio/components/transition-settings-panel.tsx`
- Modify: `src/components/pages/studio/studio.page.tsx` + `studio.page.styled.tsx`

- [ ] **Step 1: Apply expansion + variation-mount hunks file-by-file**

For each file run `git diff 7160fcf feat/api-key-config-i2i -- <file>` and apply only the prompt-expansion and variation-panel/lightbox mount hunks onto stage's version. `transition-settings-panel.tsx` is the heaviest reconcile (stage +148) — keep stage's structure, insert the variation controls + `transition-settings` gate on real (non-candidate) keyframe count.

- [ ] **Step 2: Type-check + lint + full studio tests**

Run: `pnpm run type-check && pnpm run lint && pnpm test src/components/pages/studio`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add src/components/pages/studio/components/generate-tab.tsx \
        src/components/pages/studio/components/actions-tab.tsx \
        src/components/pages/studio/components/transition-gap.tsx \
        src/components/pages/studio/components/transition-settings-panel.tsx \
        src/components/pages/studio/studio.page.tsx \
        src/components/pages/studio/studio.page.styled.tsx
git commit -m "feat(studio): mount variation panel/lightbox + prompt expansion across studio tabs"
```

---

### Task 11: Wire generation hooks (variation progress + candidate swap-in)

**Files:**
- Modify: `src/components/pages/studio/hooks/useFlowGeneration.ts`
- Modify: `src/components/pages/studio/hooks/useFlowJobProgress.ts`
- Modify: `src/components/pages/studio/hooks/useBatchSubmit.ts`

- [ ] **Step 1: Apply variation hunks onto stage's hooks**

`useFlowJobProgress.ts` is the heaviest reconcile (stage rewrote it). For each hook, `git diff 7160fcf feat/api-key-config-i2i -- <file>` and apply only the variation-progress-tracking and candidate-result-swap-in hunks onto stage's current logic. Use Bugsnag (not console.error) for error paths. Use a concurrency-capped worker pool (cap 4) for any "generate all variations" loop.

- [ ] **Step 2: Type-check + lint + tests**

Run: `pnpm run type-check && pnpm run lint && pnpm test src/components/pages/studio`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add src/components/pages/studio/hooks/useFlowGeneration.ts \
        src/components/pages/studio/hooks/useFlowJobProgress.ts \
        src/components/pages/studio/hooks/useBatchSubmit.ts
git commit -m "feat(studio): wire variation progress + candidate swap-in into generation hooks"
```

---

### Task 12: Confirm the `i2i` method is unreachable (it has no UI without BYO)

The PR only ever reached the i2i route through the BYO endpoint `<select>` (the `endpoint:`-prefixed option in `flow-builder.tsx` and the panel `optgroup`). Tasks 6/8/9 already removed those, so i2i is unreachable as a *side effect* of the BYO strip — there is no separate method picker to edit. This task just verifies that and documents it.

**Files:**
- Modify: `src/components/pages/studio/components/variation-settings-panel.tsx` (add a comment only)

- [ ] **Step 1: Verify no UI produces `method: "i2i"`**

Run: `git grep -n 'method: *"i2i"\|"i2i"' src/components/pages/studio`
Expected: only type-level / switch-arm references (e.g. in `variation-status.ts`), no UI control that *emits* an i2i candidate. The `VariationCandidate.method: "i2i"` type stays intact for Part 2.

- [ ] **Step 2: Document the deferral**

Add a one-line comment near the variation method/model selection in `variation-settings-panel.tsx`: `// i2i method has no built-in image path until Part 2 (fal Kontext); only expansion variations are reachable here.`

- [ ] **Step 3: Verify + commit**

Run: `pnpm run type-check && pnpm test src/components/pages/studio`
Expected: PASS.
```bash
git add src/components/pages/studio/components/variation-settings-panel.tsx
git commit -m "docs(studio): note i2i variation method deferred to Part 2"
```

---

### Task 13: Full verification + PR

**Files:** none (verification + PR).

- [ ] **Step 1: Confirm no dropped-feature residue across the whole tree**

Run:
```bash
grep -rnE "user-api-endpoints|useUserApiEndpoints|UserApiEndpoint|endpoint-presets|userEndpointUuid|i2iEndpointUuid" src/ || echo "CLEAN"
```
Expected: `CLEAN`.

- [ ] **Step 2: Confirm the dropped files were never added**

Run:
```bash
ls src/api/user-api-endpoints 2>/dev/null && echo "LEAK" || echo "OK: no user-api-endpoints"
ls src/components/pages/profile/api-endpoints-section.tsx 2>/dev/null && echo "LEAK" || echo "OK: no profile endpoints section"
```
Expected: both `OK`.

- [ ] **Step 3: Full build + lint + tests**

Run: `pnpm run type-check && pnpm run lint && pnpm test && pnpm run build`
Expected: all PASS / build succeeds.

- [ ] **Step 4: Push + open PR**

```bash
git push -u origin feat/studio-i2i-variations
gh pr create --base stage --title "Studio variations (seed + expansion) + prompt expansion" \
  --body "Re-applies studio variation + prompt-expansion UX onto stage. Drops the PR's user-api-endpoints settings (stage's provider-key supersedes) and the PR's session system (stage's keyframes-sessions supersedes). The i2i variation method is hidden pending Part 2 (fal Kontext). Spec: docs/superpowers/specs/2026-06-28-studio-i2i-variations-on-stage-design.md"
```

- [ ] **Step 5: Manual smoke on staging**

Run the local-frontend-against-staging setup; create keyframes, run an **expansion** variation (one variation per line or `{a|b|c}` syntax) with the seed control set, accept/discard candidates, confirm candidates never enter the timeline, confirm the prompt-expansion badge. Confirm variation state survives **both** a session switch **and** a raw page reload (the latter exercises the Task 3 partialize graft). Confirm no i2i variation can be triggered.

---

## Self-Review notes

- **Spec coverage:** Tasks map to every Part-1 keep-set item (types T1, utils T2, flow.store T3, studio.store T4, components T5/T6, wiring T7–T11) and every drop-set guard (T13). The reconcile-graft + partialize edits (spec's "one non-mechanical graft") are T3 steps 4–5.
- **i2i:** hidden in T12 per the phasing decision; type stays for Part 2.
- **No BYO residue:** enforced by greps in T6/T8/T9/T13.
