# fal i2i (FLUX.1 Kontext) Cross-Repo — Implementation Plan (Part 2)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a fal **image-to-image** model (FLUX.1 Kontext) end-to-end so the studio `i2i` variation method (hidden in Part 1) produces a real re-imaged result, billed per image, using the platform-or-personal fal key resolved server-side.

**Architecture:** Worker gains a Kontext image-to-image submit path (passes `image_url`); backend adds the model to `MODEL_CATALOG`, a `perImage` pricing kind, and algorithm registration; frontend re-enables the i2i method and sends `{ infinidream_algorithm: 'flux-kontext-i2i', prompt, source_dream_uuid, seed }`. No new provider plumbing (reuses the existing fal client + key-resolver).

**Tech Stack:** TypeScript across `worker` (BullMQ, `@fal-ai/client`), `backend` (Express/TypeORM), `frontend` (React/Zustand). Tests: each repo's existing runner.

**Spec:** `docs/superpowers/specs/2026-06-28-studio-i2i-variations-on-stage-design.md` (Part 2 + Resolved).

**Prerequisite:** Part 1 plan merged (the i2i method UI + `VariationCandidate.method:"i2i"` exist; method currently hidden).

## Global Constraints

- Package managers: pnpm (frontend/backend), npm (worker — matches its scripts).
- Branch per repo: `feat/studio-i2i-variations` (worker, backend) off each repo's `origin/stage`; frontend continues the Part-1 branch.
- **Verified fal Kontext contract** (do not re-derive): endpoint `fal-ai/flux-pro/kontext`; input required `prompt`, `image_url`; optional `seed`, `guidance_scale` (3.5), `num_images` (1), `aspect_ratio`. **No `image_size`/width/height.** Output `images[]` of `{url,height,width,content_type}` + `seed`.
- **Pricing:** `$0.04`/image → new `perImage` `ModelPricing` kind. `perImage` models carry **no** `imageSizes` constraint.
- **Source-image field is `source_dream_uuid`** end-to-end (frontend payload key === worker read key; `dream.util` forwards `...promptJson` verbatim — do NOT edit `dream.util`).
- **mediaType routing is backend-authoritative** via `isImageGenerationAlgorithm` — frontend does not send `mediaType`.
- Four pricing files must stay in sync: backend `model.types.ts` + `cost.util.ts`; frontend `src/types/model.types.ts` + `src/utils/model-cost.util.ts`.

---

## Worker (`/Users/maxcarlsonold/edream/worker`)

### Task 1: Register the Kontext model + image input field

**Files:**
- Modify: `src/providers/provider.types.ts`
- Modify: `src/config/models.config.ts`

**Interfaces — Produces:** `NormalizedImageInput.imageUrl?: string`; `ImageModelConfig.inputImage?: boolean`; `WORKER_MODELS['flux-kontext-i2i']`.

- [ ] **Step 1: Branch**

```bash
cd /Users/maxcarlsonold/edream/worker
git fetch origin --prune && git switch -c feat/studio-i2i-variations origin/stage
```

- [ ] **Step 2: Add `imageUrl` to `NormalizedImageInput`**

In `src/providers/provider.types.ts`, add to `NormalizedImageInput`:
```ts
imageUrl?: string; // source image for image-to-image models (e.g. Kontext)
```

- [ ] **Step 3: Add `inputImage` flag + the Kontext model**

In `src/config/models.config.ts`, add `inputImage?: boolean;` to `ImageModelConfig`, and add to `WORKER_MODELS`:
```ts
'flux-kontext-i2i': {
  id: 'flux-kontext-i2i',
  provider: 'fal',
  mediaType: 'image',
  endpoint: 'fal-ai/flux-pro/kontext',
  inputImage: true,
},
```

- [ ] **Step 4: Type-check + commit**

Run: `npm run build`
Expected: compiles.
```bash
git add src/providers/provider.types.ts src/config/models.config.ts
git commit -m "feat(worker): register flux-kontext-i2i model + image input field"
```

---

### Task 2: `buildKontextInput` + route it in `submitImage`

**Files:**
- Modify: `src/providers/fal.provider.ts`

> **Worker has no test runner.** `package.json`'s `test` is the stub `echo … && exit 1`; there are no jest/vitest deps and zero test files. So the usual failing-test-first cycle is not available here. **Primary verification = `npm run build` (tsc) + the Task 9 staging smoke** (which asserts the real fal request body). `buildKontextInput` is a pure function — if you want unit coverage, do the *optional* runner setup below; otherwise skip the test steps. Do **not** rely on `npm test`.
>
> **Optional runner setup** (only if adding worker unit tests): `npm i -D vitest`, set `"test": "vitest run"` in `package.json`, then the assertions below run via `npm test -- fal.provider`.

- [ ] **Step 1 (optional): Unit assertions for the builder**

```ts
import { buildKontextInput } from '../fal.provider';

test('buildKontextInput passes image_url + prompt and omits image_size', () => {
  const body = buildKontextInput({ prompt: 'glowing', imageUrl: 'https://r2/x.png', seed: 7 });
  expect(body).toEqual({ prompt: 'glowing', image_url: 'https://r2/x.png', num_images: 1, seed: 7 });
  expect(body).not.toHaveProperty('image_size');
});

test('buildKontextInput omits seed when negative/absent', () => {
  const body = buildKontextInput({ prompt: 'x', imageUrl: 'u', seed: -1 });
  expect(body).not.toHaveProperty('seed');
});
```

- [ ] **Step 2: Implement and export `buildKontextInput`; route it**

In `src/providers/fal.provider.ts`:
```ts
export function buildKontextInput(input: NormalizedImageInput): Record<string, unknown> {
  const body: Record<string, unknown> = {
    prompt: input.prompt,
    image_url: input.imageUrl,
    num_images: input.numImages ?? 1,
  };
  if (typeof input.seed === 'number' && input.seed >= 0) {
    body.seed = input.seed;
  }
  return body;
}
```
In `falImageProvider.submitImage` (signature `(endpoint, input, apiKey)` — note it does **not** receive `modelConfig`), branch on `input.imageUrl`: when present (Kontext path) use `buildKontextInput`, else `buildFluxInput`. The handler in Task 3 only sets `imageUrl` for `inputImage` models, so `input.imageUrl` presence is a safe discriminator (this is why we branch on the input, not on `modelConfig`).

- [ ] **Step 3: Build (authoritative gate)**

Run: `npm run build` (and `npm test -- fal.provider` only if you did the optional runner setup).
Expected: compiles.

- [ ] **Step 4: Commit**

```bash
git add src/providers/fal.provider.ts   # + the optional test file if added
git commit -m "feat(worker): add Kontext image-to-image input builder"
```

---

### Task 3: Thread the source image through `handleFalImageJob`

**Files:**
- Modify: `src/workers/fal-handlers.ts`

**Interfaces — Consumes:** `WORKER_MODELS[...].inputImage`, `processImageForEndpoint` (`job-handlers.ts:643`, signature `(imageInput: string, jobId: string) => Promise<string>`). **Produces:** `input.imageUrl` set for i2i models.

> Same runner caveat as Task 2 — verify via `npm run build` + Task 9 staging smoke (no worker test runner). If you added vitest in Task 2, you may add a handler test mocking `processImageForEndpoint` + `submitImage` and asserting `input.imageUrl === 'https://r2/resolved.png'` for `job.data.source_dream_uuid: 'src-uuid'`.

- [ ] **Step 1: Implement the i2i branch**

In `handleFalImageJob` (`:93`), after resolving `modelConfig`, when `modelConfig.inputImage`:
```ts
const sourceRef = job.data.source_dream_uuid;
if (!sourceRef || typeof sourceRef !== 'string') {
  throw new Error(`Model "${infinidream_algorithm}" requires source_dream_uuid`);
}
const imageUrl = await processImageForEndpoint(sourceRef, String(job.id));
input.imageUrl = imageUrl;
```
Set this on the `NormalizedImageInput` before `submitImage`. Import `processImageForEndpoint` if it isn't already in scope.

- [ ] **Step 2: Build (authoritative gate)**

Run: `npm run build`
Expected: compiles.

- [ ] **Step 3: Commit**

```bash
git add src/workers/fal-handlers.ts   # + the optional test file if added
git commit -m "feat(worker): resolve source image for Kontext i2i jobs"
```

---

## Backend (`/Users/maxcarlsonold/edream/backend`)

### Task 4: Add the `perImage` pricing kind

**Files:**
- Modify: `src/types/model.types.ts`
- Modify: `src/utils/cost.util.ts`
- Test: `src/utils/__tests__/cost.util.test.ts`

**Interfaces — Produces:** `ModelPricing` variant `{ kind: 'perImage'; usdPerImage: number }`; `priceFromPricing` handles it (flat — this codebase always generates qty 1; `JobCostParams` is `{ durationSec?, imageSize? }`, there is **no** `num_images`).

> **Note:** `priceFromPricing` is currently module-private (`cost.util.ts:61`). Add `export` to it so the unit test can import it. (`assertValidParams` stays private.)

- [ ] **Step 1: Branch**

```bash
cd /Users/maxcarlsonold/edream/backend
git fetch origin --prune && git switch -c feat/studio-i2i-variations origin/stage
```

- [ ] **Step 2: Write failing pricing test**

Place in `src/__tests__/cost.util.test.ts` (repo keeps tests flat in `src/__tests__/`, not co-located).
```ts
import { priceFromPricing } from '../utils/cost.util';
test('perImage pricing is a flat per-image cost', () => {
  expect(priceFromPricing({ kind: 'perImage', usdPerImage: 0.04 }, {})).toBeCloseTo(0.04);
});
```

- [ ] **Step 3: Run — verify fail**

Run: `pnpm test -- cost.util`
Expected: FAIL (`priceFromPricing` not exported / `perImage` unhandled).

- [ ] **Step 4: Implement**

In `src/types/model.types.ts`, extend the union:
```ts
| { kind: 'perImage'; usdPerImage: number }
```
In `src/utils/cost.util.ts`: `export` `priceFromPricing`; handle `perImage` as a flat cost: `case 'perImage': return pricing.usdPerImage;` (no quantity term — `JobCostParams` carries none). Ensure `assertValidParams` does **not** require `constraints.imageSizes` for a `perImage` model.

- [ ] **Step 5: Run — pass + build**

Run: `pnpm test -- cost.util && pnpm run build`
Expected: PASS, compiles.

- [ ] **Step 6: Commit**

```bash
git add src/types/model.types.ts src/utils/cost.util.ts src/__tests__/cost.util.test.ts
git commit -m "feat(backend): add perImage pricing kind"
```

---

### Task 5: Add `flux-kontext-i2i` to `MODEL_CATALOG`

**Files:**
- Modify: `src/constants/models.constants.ts`

- [ ] **Step 1: Add the catalog entry**

```ts
{
  id: 'flux-kontext-i2i',
  label: 'FLUX.1 Kontext',
  provider: PROVIDERS.FAL,
  mediaType: DreamMediaType.IMAGE,
  constraints: {},
  pricing: { kind: 'perImage', usdPerImage: 0.04 },
},
```

- [ ] **Step 2: Build (this also surfaces missing `SupportedAlgorithm`/queue-map entries from Task 6 — do Task 6 next)**

Run: `pnpm run build`
Expected: may fail until Task 6 registers the algorithm id — that's expected; proceed to Task 6, then this compiles.

- [ ] **Step 3: Commit (with Task 6)** — commit together after Task 6 so the build is green.

---

### Task 6: Register the algorithm (queue map + image allowlist)

**Files:**
- Modify: `src/utils/prompt.util.ts`

- [ ] **Step 1: Add to all three places**

- `SUPPORTED_ALGORITHMS` array (`~:13`): add `'flux-kontext-i2i'`. (`SupportedAlgorithm` is the derived type `(typeof SUPPORTED_ALGORITHMS)[number]` — editing the array updates the type and forces the `Record` map entry below.)
- `ALGORITHM_TO_QUEUE_MAP` (`~:27`): add `'flux-kontext-i2i': 'falimage'` (same queue flux-schnell uses — verified).
- `isImageGenerationAlgorithm` allowlist (`~:95`): add `'flux-kontext-i2i'`.

- [ ] **Step 2: Build + relevant tests**

Run: `pnpm run build && pnpm test -- prompt.util`
Expected: PASS (Record-over-keys now complete; catalog from Task 5 compiles).

- [ ] **Step 3: Add a routing test**

In `src/__tests__/prompt.util.test.ts`, add a case asserting `isImageGenerationAlgorithm('flux-kontext-i2i') === true` and `ALGORITHM_TO_QUEUE_MAP['flux-kontext-i2i'] === 'falimage'`.
Run: `pnpm test -- prompt.util`
Expected: PASS.

- [ ] **Step 4: Commit Tasks 5 + 6**

```bash
git add src/constants/models.constants.ts src/utils/prompt.util.ts src/__tests__/prompt.util.test.ts
git commit -m "feat(backend): add flux-kontext-i2i model + algorithm registration"
```

---

## Frontend (`/Users/maxcarlsonold/edream/frontend`, branch `feat/studio-i2i-variations`)

### Task 7: Mirror `perImage` pricing on the frontend

**Files:**
- Modify: `src/types/model.types.ts`
- Modify: `src/utils/model-cost.util.ts`
- Test: `src/utils/__tests__/model-cost.util.test.ts` (if present; else add)

- [ ] **Step 1: Extend the union + cost fn (mirror backend Task 4)**

Add `| { kind: 'perImage'; usdPerImage: number }` to the frontend `ModelPricing` (`src/types/model.types.ts`), and handle it **flat** in `estimateUnitCostUsd` (`src/utils/model-cost.util.ts:34`; its `CostParams` is `{ durationSec?, imageSize? }` — no quantity field): `case 'perImage': return pricing.usdPerImage;`. Keep this byte-for-byte consistent with the backend flat implementation.

- [ ] **Step 2: Test + type-check**

Run: `pnpm run type-check && pnpm test src/utils`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add src/types/model.types.ts src/utils/model-cost.util.ts src/utils/__tests__/model-cost.util.test.ts
git commit -m "feat(frontend): mirror perImage pricing kind"
```

---

### Task 8: Re-enable the i2i method + send the Kontext payload

**Files:**
- Modify: `src/components/pages/studio/components/variation-settings-panel.tsx` (un-hide i2i)
- Modify: `src/components/pages/studio/components/flow-builder.tsx` (i2i payload)
- Modify: `src/components/pages/studio/constants/variation-presets.ts` (if methods enumerated there)

**Interfaces — Consumes:** the API-driven model list now containing `flux-kontext-i2i`; the parent keyframe's source dream UUID (`FlowKeyframe.dreamUuid` — documented "Source image Dream UUID"; **not** `keyframeUuid`, which is the backend Keyframe UUID).

- [ ] **Step 1: Add a UI affordance that triggers an i2i variation**

In Part 1, i2i became *unreachable* (its only entry point was the removed BYO endpoint `<select>`) — so this is **new UI**, not un-hiding. The variation model list is API-driven and now includes `flux-kontext-i2i` (Task 5). Surface it in `variation-settings-panel.tsx`'s model dropdown; when the user selects the Kontext model, the Vary action emits a `VariationCandidate` with `method: "i2i"` (driving the payload in Step 2). Remove the Part-1 deferral comment added in Part-1 Task 12.

- [ ] **Step 2: Build the i2i payload in `flow-builder.tsx`**

For the i2i method, send:
```ts
{
  infinidream_algorithm: 'flux-kontext-i2i',
  prompt,                       // expanded/modified variation prompt
  source_dream_uuid: parentSourceDreamUuid,  // parent keyframe's source dream
  seed: baseSeed,
}
```
No `size`, no `userEndpointUuid`, no `mediaType`. `parentSourceDreamUuid` is the parent keyframe's **`dreamUuid`** (the candidate's parent is reachable via `i2iParentId`; the parent's `dreamUuid` is the source image dream). Do **not** use `keyframeUuid`.

- [ ] **Step 3: Type-check + lint + tests**

Run: `pnpm run type-check && pnpm run lint && pnpm test src/components/pages/studio`
Expected: PASS. Confirm `grep -n userEndpointUuid src/components/pages/studio` returns nothing.

- [ ] **Step 4: Commit**

```bash
git add src/components/pages/studio/components/variation-settings-panel.tsx \
        src/components/pages/studio/components/flow-builder.tsx \
        src/components/pages/studio/constants/variation-presets.ts
git commit -m "feat(frontend): enable i2i variation via flux-kontext (provider-key resolved server-side)"
```

---

### Task 9: End-to-end verification on staging

**Files:** none.

- [ ] **Step 1: Deploy/run all three branches against staging**

Worker + backend on `feat/studio-i2i-variations`; frontend pointed at the staging backend (local-frontend-against-staging setup). Ensure a fal key exists (personal via provider-key card, or the platform global key).

- [ ] **Step 2: Run an i2i variation**

From a keyframe with a real source image, pick the **i2i** method, generate. Confirm:
- The candidate's result is a **distinct re-imaged** image (not the source passthrough).
- Billing reflects `$0.04`/image (check credit/cost path).
- Accept promotes the candidate; discard removes it; candidates never enter the timeline.
- A missing source image surfaces a clear error (Bugsnag), not a silent t2i.

- [ ] **Step 3: Confirm the fal request shape (one-time smoke)**

In worker logs for the i2i job, confirm the fal submit body contains `image_url` and `prompt` (and `seed` when set), and no `image_size`.

- [ ] **Step 4: Open the three PRs**

```bash
# in each of worker, backend (frontend PR already open from Part 1 — push the new commits)
gh pr create --base stage --title "fal i2i (FLUX.1 Kontext) — <repo> side" \
  --body "Part 2 of studio i2i variations. Spec: docs/superpowers/specs/2026-06-28-studio-i2i-variations-on-stage-design.md. Depends on Part 1 frontend."
```

---

## Self-Review notes

- **Spec coverage:** Worker (T1–T3), backend (T4–T6), frontend (T7–T8) cover every Part-2 bullet; `dream.util` correctly untouched (constraint reaffirmed). Pricing fan-out hits all four KEEP-IN-SYNC files (T4 + T7).
- **Field-name consistency:** `source_dream_uuid` used identically in frontend payload (T8), worker read (T3), and forwarded by `dream.util` (unchanged). `imageUrl` (normalized input) ↔ `image_url` (fal body) mapping is explicit in T2/T3.
- **mediaType:** backend allowlist (T6) authoritative; frontend sends no `mediaType` (T8).
- **No placeholders:** TDD steps show real test + impl code for the two novel units (Kontext builder T2, perImage pricing T4); port/registration tasks give exact entries.
