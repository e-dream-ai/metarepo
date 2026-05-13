# Model Swaps & VAE Preview Design

Design spec for replacing studio generation models with faster, cheaper, higher-quality alternatives and adding early VAE preview for video generation.

**Origin:** Feedback from Jef (AI video creator, 2026-03-17 meeting). Jef recommended Z Image Turbo, LTX 2.3, and Nvidia Super Resolution as drop-in improvements. VAE preview was described as "almost mandatory" for creative workflows.

**Scope:** New model integrations (B) and VAE preview (C) only. UX fixes (A), sessions/projects (D), and NSFW handling (E) are separate efforts.

**Relation to existing docs:**
- Extends `docs/plans/2026-01-30-visual-creator-workflows-design.md` (studio architecture, combinatorial workflow)
- Studio page MVP: `docs/plans/2026-02-16-studio-page-mvp.md`
- Previous fixes: `docs/plans/2026-02-24-studio-review-round2-fixes.md`

---

## Goals

1. **Faster generation** — Z Image Turbo: ~5s per image (vs Qwen ~15-30s). LTX 2.3: <2 min for 10s 1080p video.
2. **Lower cost** — Z Image Turbo: $0.005/image via RunPod public endpoint. LTX 2.3: smaller model, faster renders = less GPU time.
3. **Better quality** — Both models produce noticeably better output per Jef's testing.
4. **Early feedback** — VAE preview gives creators a ~240px preview in ~5 seconds, before committing to a full render.
5. **Keep existing models** — Qwen, Wan I2V, and current Uprez stay available as part of the toolbox.

---

## Architecture Overview

```
Studio UI (model picker)
    │
    ├─ zit-image ──→ Worker ──→ RunPod Public API (/v2/z-image-turbo/runsync)
    │                              $0.005/image, ~5s, no container needed
    │
    ├─ ltx-i2v ───→ Worker ──→ RunPod Serverless (custom endpoint)
    │                              e-dream-ai/gpu-container-ltx, ComfyUI + TAESD
    │
    ├─ nvidia-uprez → Worker ──→ RunPod Serverless (custom endpoint)
    │                              e-dream-ai/gpu-container-nvidia-vsr
    │                              Uses Nvidia RTX Nodes (VFX SDK)
    │                              Target GPU: L40S or A40
    │
    ├─ qwen-image ─→ (existing, unchanged)
    ├─ wan-i2v ────→ (existing, unchanged)
    └─ uprez ──────→ (existing, unchanged)
```

---

## 1. Z Image Turbo Integration

### Why

- 6B parameter model, ~11GB
- ~5 second renders
- Quality rivals commercial APIs
- RunPod already hosts a public endpoint — no container to build or maintain

### RunPod Public Endpoint

- **URL:** `https://api.runpod.ai/v2/z-image-turbo/runsync`
- **Auth:** Bearer token (existing RunPod API key)
- **Cost:** $0.005 per image
- **Response:** Returns image URL (expires 7 days)
- **Sizes:** 512x512, 768x768, 1024x1024, 1280x1280, 1024x768, 768x1024, 1280x720, 720x1280

### Worker Changes

New case in endpoint routing for `infinidream_algorithm: "zit-image"`:

1. Call RunPod public API directly (not a custom endpoint)
2. Use `/runsync` — synchronous, returns result immediately
3. Download result image from RunPod's temporary URL
4. **Verify R2 upload succeeded** before marking dream as `processed` (RunPod URL expires in 7 days — if download/upload fails silently, the dream would have a dead URL)
5. Update dream status to `processed`

**Sync vs async path:** This is different from existing algorithms which use RunPod's async `/run` + status polling. Key implications:

- **No progress events.** Z Image Turbo jobs emit no intermediate `job:progress` via Socket.IO. Jobs jump directly from `queue` to `processed` (or `failed`). This is acceptable given the ~5s render time — the frontend already handles this gracefully (shows "Queued..." then the final image).
- **Timeout.** The runsync call must have a 30-second timeout. If exceeded, mark the dream as `failed`.
- **Worker concurrency.** Z Image Turbo jobs should use their own BullMQ queue or concurrency pool to avoid blocking slower video generation jobs. A blocked runsync call should not hold up LTX/Wan renders.
- **Batch parallelism.** The frontend creates separate dreams per seed via `Promise.all` (same as Qwen), so each becomes its own BullMQ job. With 8 seeds at ~5s each, all 8 run in parallel across worker slots — not serially. This is the intended flow.

If the sync call fails or times out, the worker marks the dream as `failed` — same as existing error handling. RunPod does not charge for failed generations.

### Backend Changes

- New algorithm name `zit-image` recognized in dream creation
- No new env var needed — uses existing `RUNPOD_API_KEY` against the public endpoint
- Dream prompt format:

```json
{
  "infinidream_algorithm": "zit-image",
  "prompt": "bioluminescent deep sea creature",
  "size": "1280*720",
  "seed": 42
}
```

### Frontend Changes

- Images tab: model picker dropdown (Z Image Turbo / Qwen Image)
- Store: `imageGenParams.model` field (`"zit-image" | "qwen-image"`)
- When generating, pass selected model as `infinidream_algorithm`
- **Size options per model:** Show only sizes supported by the selected model. Shared sizes (1280x720, 1024x1024, 720x1280, 512x512) cover both. Z Image Turbo also supports 768x768, 1280x1280, 1024x768, 768x1024. Show the full Z Image Turbo list when it's selected; show the current Qwen list when Qwen is selected.
- **Dream name:** Currently hardcoded as `"Qwen Image ${n}"` in `images-tab.tsx:88`. Change to use the selected model name: `"Z Image ${n}"` or `"Qwen Image ${n}"`.

**Important:** Z Image Turbo jobs are tracked via `StudioImage` (the `images` array in the store), NOT `StudioJob` (the `jobs` array). This is the same pattern as Qwen. `StudioJobType` does not include `"zit-image"` — that type is for image generation, not video/uprez jobs.

---

## 2. LTX 2.3 Integration

### Why

- 50fps, 1920x1080, 10-second videos in under 2 minutes
- Can render up to 20-second videos
- Better quality than Wan I2V per Jef's testing
- LoRAs are essential — LTX has no movement without them, but even a low-strength LoRA brings it to life

### Repository: [`e-dream-ai/gpu-container-ltx`](https://github.com/e-dream-ai/gpu-container-ltx)

Custom ComfyUI container for RunPod serverless, following the existing `gpu-container` pattern:

- Base: `runpod/worker-comfyui` Docker image
- Models: LTX 2.3 checkpoint + ~5 supporting models (exact list TBD — waiting on Jef)
- Workflow: ComfyUI workflow JSON with TAESD preview node (see VAE Preview section)
- LoRAs: Baked into container or downloaded at runtime
- Deploy: GitHub Actions → Docker Hub → RunPod serverless endpoint

### Worker Changes

New case in endpoint routing for `infinidream_algorithm: "ltx-i2v"`:

- Route to `RUNPOD_LTX_ENDPOINT_ID` (new env var)
- Async flow: `/run` → poll status → relay progress via Socket.IO (same as Wan)
- Progress updates include `preview_frame` (VAE preview arrives early)

### Backend Changes

- New env var: `RUNPOD_LTX_ENDPOINT_ID`
- New algorithm name `ltx-i2v` recognized in dream creation
- Dream prompt format:

```json
{
  "infinidream_algorithm": "ltx-i2v",
  "prompt": "slow zoom into crystal cave",
  "image": "uuid-or-url",
  "duration": 10,
  "high_noise_loras": [{ "path": "lora-url", "scale": 0.8 }],
  "low_noise_loras": []
}
```

### LoRA Handling

Same pattern as existing Wan I2V LoRAs:

- `high_noise_loras` / `low_noise_loras` arrays in prompt JSON
- Action presets map user-facing motion names to LTX-specific LoRA URLs
- Frontend passes LoRAs transparently — just different model paths
- LTX-specific action presets will be added once Jef provides LoRA details

**Important (from Jef):** LTX produces static video without LoRAs. Even a "stable camera" LoRA at 20% strength activates the model. The studio should hint at this when LTX is selected and no actions have LoRAs.

### Duration Options

LTX supports longer videos than Wan. Duration options must become model-aware:

| Model | Duration Options | With LoRAs |
|-------|-----------------|------------|
| Wan I2V | 5, 8, 10 seconds | 5, 8 seconds (LoRA constraint) |
| LTX 2.3 | 5, 10, 15, 20 seconds | TBD (depends on Jef's LoRA details) |

The existing `getAllowedDurationsForActions()` in `duration-options.ts` must accept a `model` parameter and return the correct set. The `clampDurationToAllowed()` function is model-agnostic and stays unchanged.

### Frontend Changes

- Generate tab: model picker dropdown (LTX 2.3 / Wan I2V)
- Store: `videoGenParams.model` field (`"ltx-i2v" | "wan-i2v"`)
- Conditional defaults: LTX may have different step/guidance ranges than Wan
- Hint text when LTX selected with no LoRA actions: "LTX works best with motion presets"
- `StudioJobType` expands: `"wan-i2v" | "ltx-i2v" | "uprez" | "nvidia-uprez"`

### Action Preset Model Scoping

Action presets currently contain Wan-specific LoRA URLs. Applying Wan LoRAs to an LTX render would fail or produce garbage. Presets must be model-scoped:

```typescript
export interface PresetPack {
  name: string;
  model: "wan-i2v" | "ltx-i2v" | "all";  // NEW: which video model this preset is for
  actions: Omit<StudioAction, "id">[];
}
```

- When the video model picker is set to LTX, only show presets with `model: "ltx-i2v"` or `model: "all"`
- When set to Wan, only show presets with `model: "wan-i2v"` or `model: "all"`
- Text-only presets (no LoRAs) use `model: "all"` — they work with any model
- Existing Wan LoRA presets get `model: "wan-i2v"`
- LTX LoRA presets will be added with `model: "ltx-i2v"` once Jef provides details

### Hardcoded Algorithm References

The following locations hardcode `"wan-i2v"` or `"wan-i2v-lora"` and must read from `videoGenParams.model` instead:

| File | Lines | What's Hardcoded |
|------|-------|-----------------|
| `hooks/useBatchSubmit.ts` | 92, 103 | `infinidream_algorithm: "wan-i2v-lora"` / `"wan-i2v"` |
| `hooks/useBatchSubmit.ts` | 125 | `jobType: "wan-i2v"` |
| `components/results-tab.tsx` | 271, 282 | `infinidream_algorithm: "wan-i2v-lora"` / `"wan-i2v"` (retry) |
| `components/results-tab.tsx` | 305 | `jobType: "wan-i2v"` (retry) |
| `components/images-tab.tsx` | 76 | `infinidream_algorithm: "qwen-image"` |

For `useBatchSubmit.ts`: read `videoGenParams.model` from the store. If model is `"ltx-i2v"`, use `"ltx-i2v"` as the algorithm (LTX LoRA handling is in the prompt params, not a separate algorithm name). If model is `"wan-i2v"`, keep existing `"wan-i2v"` / `"wan-i2v-lora"` split.

For `images-tab.tsx`: read `imageGenParams.model` from the store.

For `results-tab.tsx` retry: read the `jobType` from the existing job being retried — don't hardcode.

---

## 3. Nvidia Super Resolution Integration

### Why

- Uses Nvidia RTX Nodes for ComfyUI (`Comfy-Org/Nvidia_RTX_Nodes_ComfyUI`)
- RTX Video Super Resolution node — upscales images and video using Nvidia VFX SDK
- 4K upscaling 30x faster than popular alternatives, fraction of the VRAM cost
- Quality settings: LOW, MEDIUM, HIGH, ULTRA

### GPU Compatibility (Confirmed)

The ComfyUI node uses the **Nvidia VFX SDK**, which supports datacenter GPUs — NOT limited to consumer RTX cards like the driver-level VSR feature.

**VFX SDK supported GPUs:** A40, L40, L4, A30, B200, A2, H100, A10, T4, B100, A100, B40

**RunPod serverless availability:**

| GPU | On RunPod? | VFX SDK? | Recommended? |
|-----|-----------|----------|-------------|
| L40S | Yes | Yes | **Yes — best value for video workloads** |
| A40 | Yes | Yes | **Yes — good fallback** |
| RTX 4090 | Yes | Yes | Works but consumer card |
| A100 | Yes | Yes | Overkill for upscaling |
| H100 | Yes | Yes | Overkill for upscaling |

**Recommendation:** Target **L40S** as primary, **A40** as fallback for the serverless endpoint.

### Status: Partially Blocked

- GPU compatibility: **confirmed** (VFX SDK works on RunPod datacenter GPUs)
- ComfyUI node: **identified** — `Comfy-Org/Nvidia_RTX_Nodes_ComfyUI`, install via ComfyUI Manager
- Workflow details: **waiting on Jef** for his specific workflow and quality settings
- Can proceed with basic container setup using the known node

### Planned Approach

- Repository: [`e-dream-ai/gpu-container-nvidia-vsr`](https://github.com/e-dream-ai/gpu-container-nvidia-vsr)
- New env var: `RUNPOD_NVIDIA_SR_ENDPOINT_ID`
- New algorithm name: `nvidia-uprez`
- Worker routing: same async pattern as existing uprez
- Frontend: uprez model picker in results tab (Nvidia Super Resolution / Current Uprez)
- Target GPU config for endpoint: L40S primary, A40 fallback

### Dream Prompt Format (tentative)

```json
{
  "infinidream_algorithm": "nvidia-uprez",
  "video_uuid": "source-dream-uuid",
  "upscale_factor": 2,
  "quality": "HIGH"
}
```

---

## 4. VAE Preview

### Why

A lightweight VAE decoder (TAESD) produces a ~240px preview image in ~5 seconds before the full diffusion render begins. Jef called this "almost mandatory" — it lets creators see if a generation is heading in the right direction before waiting minutes for the full result.

### How It Works

The VAE preview is entirely a GPU-container workflow concern. The rest of the stack already supports it:

| Layer | What Happens | Changes Needed |
|-------|-------------|----------------|
| `gpu-container-ltx` | TAESD node in ComfyUI workflow decodes initial noise into ~240px preview. Emits as `preview_frame` in RunPod status update. | New (workflow design) |
| Worker | Receives `preview_frame` from RunPod status, stores in Redis (`job:preview:{dreamUUID}`, 3hr TTL), broadcasts via Socket.IO | None — already does this |
| Backend | Serves preview via `GET /v1/dream/{uuid}/preview`, broadcasts `job:progress` with `preview_frame` | None — already does this |
| Frontend | `useStudioJobProgress` receives `preview_frame`, `results-tab` renders it in grid cell (line 413-416) | None — already does this |

**Confirmed by code review:** `useStudioJobProgress` hook already passes `preview_frame` to both `updateImage` and `updateJob`, and results-tab already renders `job.previewFrame` as a base64 image. Zero frontend/backend/worker changes needed.

### Model Applicability

| Model | VAE Preview? | Reason |
|-------|-------------|--------|
| LTX 2.3 | Yes | Primary use case. TAESD preview in ~5s before full render. |
| Z Image Turbo | No | Full image returns in ~5s anyway. No progress events (runsync). |
| Wan I2V | Already has it | Sends progress frames mid-render. |
| Qwen Image | No | Fast enough without preview. |
| Nvidia Super Resolution | No | Upscaling is deterministic — no need to preview. |

### Frontend Behavior

- Results grid cell shows blurry VAE preview as soon as it arrives (~5s after job starts)
- Real render progress frames replace it naturally as they come in
- No new UI elements, no confirmation gate — just better loading state

**Note:** VAE preview is part of the LTX container workflow (implementation step 2), not a separate step. Listed separately for clarity but ships with LTX 2.3.

---

## 5. Frontend Model Selection UI

### Images Tab

```
Prompt: [.................................]
Images: [8 v]   Size: [1280x720 v]   Model: [Z Image Turbo v]   [Generate Images]
                                              |-- Z Image Turbo
                                              '-- Qwen Image
```

Size options change based on selected model.

### Generate Tab (output settings)

```
Model: [LTX 2.3 v]   Duration: [10 seconds v]   Steps: [30 v]   Guidance: [5.0 v]
       |-- LTX 2.3
       '-- Wan I2V
Output playlist: [Batch 02 v]   [+ Create New]
```

Duration options change based on selected model. Action preset dropdown filtered by model.

### Results Tab

Uprez button offers model choice:
```
[Uprez Selected (3) v]
 |-- Nvidia Super Resolution
 '-- Current Uprez
```

### Store Changes

Generalize params to include model selection:

```typescript
// Current
qwenParams: QwenParams;     // { seedCount, size }
wanParams: WanI2VParams;    // { duration, numInferenceSteps, guidance }

// New
imageGenParams: ImageGenParams;  // { model: "zit-image" | "qwen-image", seedCount, size }
videoGenParams: VideoGenParams;  // { model: "ltx-i2v" | "wan-i2v", duration, numInferenceSteps, guidance }
uprezModel: "nvidia-uprez" | "uprez";
```

Model selection persists in the studio session (survives reload).

When LTX is selected and no actions have LoRAs, show hint: "LTX works best with motion presets."

### Store Migration

Renaming `qwenParams` → `imageGenParams` and `wanParams` → `videoGenParams` requires a Zustand persist migration:

1. **Bump version** from 2 to 3
2. **Add migration** in the `migrate` function:
   ```typescript
   if (version < 3) {
     // Rename qwenParams → imageGenParams, add default model
     if (state.qwenParams) {
       state.imageGenParams = { ...state.qwenParams, model: "qwen-image" };
       delete state.qwenParams;
     }
     // Rename wanParams → videoGenParams, add default model
     if (state.wanParams) {
       state.videoGenParams = { ...state.wanParams, model: "wan-i2v" };
       delete state.wanParams;
     }
     // Add uprezModel default
     state.uprezModel = "uprez";
   }
   ```
3. **Update `partialize`** to persist `imageGenParams`, `videoGenParams`, `uprezModel` instead of `qwenParams`, `wanParams`
4. **Update `resetSession`** to use new field names and defaults

### Model Picker Locking

When jobs are in progress, the model picker should be **read-only** for that stage. Rationale: switching from Wan to LTX mid-session would create jobs with mixed models sharing the same results grid, with incompatible LoRA presets applied.

- Image model picker: locked once images are generating (unlocked after all images resolve)
- Video model picker: locked once video jobs are submitted (unlocked on `resetSession`)
- Alternative: store model choice per-job via `jobType` (already exists). The model picker only affects new submissions. This is cleaner but means the results grid can contain mixed-model jobs. Acceptable for power users.

**Decision:** Store per-job via `jobType`. The picker affects new submissions only. Existing `jobType` field already distinguishes `"wan-i2v"` from `"ltx-i2v"`.

---

## 6. New Environment Variables

| Variable | Service | Purpose |
|----------|---------|---------|
| `RUNPOD_LTX_ENDPOINT_ID` | Worker | Custom LTX 2.3 serverless endpoint |
| `RUNPOD_NVIDIA_SR_ENDPOINT_ID` | Worker | Custom Nvidia Super Resolution endpoint |

Z Image Turbo uses the public endpoint URL directly — no endpoint ID needed.

---

## 7. New Repositories

| Repo | Status | Contents |
|------|--------|----------|
| [`e-dream-ai/gpu-container-ltx`](https://github.com/e-dream-ai/gpu-container-ltx) | **Pushed to main** | Dockerfile (CUDA 12.4, ComfyUI, LTX 2.3 FP8 ~22GB, Gemma 3 text encoder, distilled LoRA, spatial upscaler, 7 camera LoRAs, TAESD preview models), CI (container-builder runner), `--preview-method taesd` enabled |
| [`e-dream-ai/gpu-container-nvidia-vsr`](https://github.com/e-dream-ai/gpu-container-nvidia-vsr) | **Pushed to main** | Dockerfile (CUDA 12.4, ComfyUI, Nvidia RTX Nodes, VideoHelperSuite), CI, test inputs for image + video upscaling, target L40S/A40 |

Both follow the existing `gpu-container` pattern: Dockerfile → GitHub Actions → Docker Hub → RunPod serverless. CI uses `container-builder` (self-hosted runner) for release builds due to large image sizes (~40GB for LTX).

---

## Implementation Status (updated 2026-03-31)

### Done

| Deliverable | PR / Repo | Status |
|---|---|---|
| **Z Image Turbo — worker** | Patrick merged to `worker/stage` | Shipped |
| **Z Image Turbo — backend** | Patrick merged to `backend/stage` | Shipped |
| **Z Image Turbo — frontend** | [frontend#587](https://github.com/e-dream-ai/frontend/pull/587) | PR open |
| **Frontend: imageGenParams refactor** (qwenParams → imageGenParams, v3 migration) | frontend#587 | PR open |
| **Frontend: videoGenParams refactor** (wanParams → videoGenParams, v4 migration) | frontend#587 | PR open |
| **Frontend: Video model picker** (LTX 2.3 / Wan I2V) + model-aware durations | frontend#587 | PR open |
| **Frontend: Action preset model scoping** | frontend#587 | PR open |
| **Frontend: Hardcoded algorithm fixes** (useBatchSubmit, results-tab retry) | frontend#587 | PR open |
| **Frontend: LTX hint text** ("LTX works best with motion presets") | frontend#587 | PR open |
| **Frontend: Uprez model picker** (Nvidia SR / Current Uprez) | frontend#587 | PR open |
| **Worker: LTX I2V handler + queue** | [worker#31](https://github.com/e-dream-ai/worker/pull/31) | PR open |
| **Worker: Nvidia VSR handler + queue** | worker#31 | PR open |
| **Backend: ltx-i2v + nvidia-uprez routing** | [backend#412](https://github.com/e-dream-ai/backend/pull/412) | PR open |
| **gpu-container-ltx** | [e-dream-ai/gpu-container-ltx](https://github.com/e-dream-ai/gpu-container-ltx) | Pushed to main |
| **gpu-container-nvidia-vsr** | [e-dream-ai/gpu-container-nvidia-vsr](https://github.com/e-dream-ai/gpu-container-nvidia-vsr) | Pushed to main |
| **Design spec + workflows reference** | This file + `model-workflows-reference.md` | Done |

### Blocked on Jef

| Item | Waiting On | Impact |
|------|-----------|--------|
| LTX 2.3 optimized ComfyUI workflow | Jef's minimal workflow | Have official template as placeholder in container |
| LTX LoRA recommendations + strengths | Jef's tested LoRAs | Have 7 Lightricks camera LoRAs as defaults |
| LTX duration constraints with LoRAs | Jef's tested params | Currently returns full [5, 10, 15, 20] for LTX |
| Nvidia VSR quality/workflow settings | Jef's preferred quality | Defaults to quality: "HIGH" |

### Deploy Steps (after merge)

1. Set `RUNPOD_LTX_ENDPOINT_ID` in worker Heroku config (after creating RunPod serverless endpoint from `gpu-container-ltx` Docker image)
2. Set `RUNPOD_NVIDIA_SR_ENDPOINT_ID` in worker Heroku config (after creating RunPod serverless endpoint from `gpu-container-nvidia-vsr` Docker image, targeting L40S/A40)
3. Merge backend#412, worker#31, frontend#587 to stage

---

## Testing

| Test | How |
|------|-----|
| Z Image Turbo generates images | Studio → select Z Image Turbo → generate → images appear in ~5s |
| Z Image Turbo no progress events | Images show "Queued..." then jump to final image (no percentage) |
| Z Image Turbo R2 upload | Verify image URL works after RunPod's 7-day temporary URL expires |
| Z Image Turbo timeout | Simulate slow response → dream marked failed after 30s |
| Z Image Turbo concurrency | Generate 24 images → verify they don't block video generation jobs |
| Z Image Turbo size options | Switch models → size dropdown updates to model-specific options |
| Z Image Turbo dream name | Generated dreams named "Z Image 1" not "Qwen Image 1" |
| LTX 2.3 generates video | Studio → select LTX 2.3 → generate → video renders with progress |
| VAE preview appears early | LTX job shows blurry preview in results grid within ~5s of job start |
| LoRAs produce movement | LTX job with motion preset LoRA shows camera motion in output |
| LTX without LoRAs hint | Select LTX, add no-LoRA actions → hint text appears |
| LTX duration options | Select LTX → duration dropdown shows 5, 10, 15, 20s options |
| Preset filtering | Select LTX → only LTX-compatible presets shown. Select Wan → only Wan presets shown |
| Batch submit uses correct algo | Select LTX → submit → dreams created with `infinidream_algorithm: "ltx-i2v"` |
| Retry uses job's original algo | Retry a Wan job → uses `wan-i2v`. Retry an LTX job → uses `ltx-i2v` |
| Model selection persists | Select Z Image Turbo → reload page → still selected |
| Store migration | Clear localStorage, set version 2 data → reload → migrated to version 3 with defaults |
| Old models still work | Switch to Qwen/Wan → generate → works as before |
| Mixed model session | Generate some Wan jobs, switch to LTX, generate more → both types in results grid |
| Nvidia uprez works | Select completed video → uprez with Nvidia SR → enhanced video returned |
| Nvidia uprez GPU | Verify endpoint runs on L40S/A40, not overkill H100 |
