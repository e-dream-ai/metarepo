# Z Image Turbo Integration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Z Image Turbo as an image generation option in the studio, using RunPod's public `/runsync` endpoint at $0.005/image.

**Architecture:** New `zit-image` algorithm routes through a dedicated BullMQ queue to RunPod's public sync endpoint. Frontend gets a model picker in the images tab. Z Image Turbo images are tracked via `StudioImage` (same as Qwen). Store is refactored from `qwenParams` to `imageGenParams` with a `model` field.

**Tech Stack:** TypeScript, Express, BullMQ, React 18, Zustand 5 (with persist), Vitest

**Spec:** `docs/superpowers/specs/2026-03-30-model-swaps-vae-preview-design.md`

**Verification commands:**
```bash
# Frontend
cd /Users/maxcarlsonold/edream/frontend
pnpm run test          # Vitest — all tests must pass
pnpm run type-check    # tsc --noEmit — 0 errors
pnpm run lint          # ESLint — 0 errors

# Worker
cd /Users/maxcarlsonold/edream/worker
npm run build          # Compile TypeScript — 0 errors
```

---

## File Map

### Worker (`worker/`)

| File | Action | Responsibility |
|------|--------|---------------|
| `src/shared/env.ts` | No change | Z Image Turbo uses existing `RUNPOD_API_KEY` against public endpoint |
| `src/config/runpod.config.ts` | Modify | Add `zitImage` endpoint using `PublicEndpointService('z-image-turbo')` |
| `src/workers/job-handlers.ts` | Modify | Add `handleZitImageJob` handler |
| `src/workers/job-handlers.ts` | Modify | Add `ZitImageParams` interface |
| `src/services/cli.service.ts` | Modify | Add `zit-image` case to `inferQueueName`, add `zitimage` queue |
| `src/index.ts` | Modify | Register `zitimage` queue and worker |

### Frontend (`frontend/`)

| File | Action | Responsibility |
|------|--------|---------------|
| `src/types/studio.types.ts` | Modify | Rename `QwenParams` → `ImageGenParams` with `model` field |
| `src/stores/studio.store.ts` | Modify | Rename `qwenParams` → `imageGenParams`, add migration v3 |
| `src/stores/studio.store.test.ts` | Modify | Add migration test, update existing tests |
| `src/components/pages/studio/components/images-tab.tsx` | Modify | Add model picker, dynamic size options, dynamic dream name |
| `src/components/pages/studio/constants/size-options.ts` | Create | Size options per model |

---

### Task 1: Add Z Image Turbo Endpoint to Worker Config

**Files:**
- Modify: `worker/src/config/runpod.config.ts`

- [ ] **Step 1: Add zitImage endpoint**

In `worker/src/config/runpod.config.ts`, add the new endpoint to the `endpoints` object:

```typescript
export const endpoints = {
  animatediff: runpod.endpoint(env.RUNPOD_ANIMATEDIFF_ENDPOINT_ID),
  hunyuan: runpod.endpoint(env.RUNPOD_HUNYUAN_ENDPOINT_ID),
  deforum: runpod.endpoint(env.RUNPOD_DEFORUM_ENDPOINT_ID),
  uprez: runpod.endpoint(env.RUNPOD_UPREZ_ENDPOINT_ID),
  videoingest: runpod.endpoint(env.RUNPOD_VIDEOINGEST_ENDPOINT_ID),
  wanT2V: new PublicEndpointService('wan-2-2-t2v-720'),
  wanI2V: new PublicEndpointService('wan-2-2-i2v-720'),
  wanI2VLora: new PublicEndpointService('wan-2-2-t2v-720-lora'),
  qwenImage: new PublicEndpointService('qwen-image-t2i'),
  zitImage: new PublicEndpointService('z-image-turbo'),
} as const;
```

- [ ] **Step 2: Build worker to verify no type errors**

Run: `cd /Users/maxcarlsonold/edream/worker && npm run build`

Expected: Compiles with 0 errors.

- [ ] **Step 3: Commit**

```bash
cd /Users/maxcarlsonold/edream/worker
git add src/config/runpod.config.ts
git commit -m "feat: add Z Image Turbo public endpoint to runpod config"
```

---

### Task 2: Add Z Image Turbo Job Handler

**Files:**
- Modify: `worker/src/workers/job-handlers.ts`

- [ ] **Step 1: Add ZitImageParams interface**

Near the other param interfaces at the top of `job-handlers.ts`, add:

```typescript
interface ZitImageParams {
  prompt: string;
  size?: string;
  seed?: number;
  output_format?: string;
  enable_safety_checker?: boolean;
}
```

- [ ] **Step 2: Add handleZitImageJob function**

After `handleQwenImageJob`, add the new handler. This uses `runSync` instead of `run` + `handleStatus` because the public endpoint returns results synchronously in ~5 seconds:

```typescript
export async function handleZitImageJob(job: Job): Promise<any> {
  const {
    prompt,
    size,
    seed = -1,
    output_format = 'png',
    enable_safety_checker = true,
    dream_uuid,
    auto_upload = true,
  } = job.data as ZitImageParams & { dream_uuid?: string; auto_upload?: boolean };

  if (!prompt || typeof prompt !== 'string') {
    throw new Error('prompt is required and must be a string');
  }

  const input: Record<string, unknown> = {
    prompt,
    seed,
    output_format,
    enable_safety_checker,
  };

  if (size) {
    if (typeof size !== 'string') {
      throw new Error('size must be a string in format "W*H", e.g., "1024*1024"');
    }
    input.size = size;
  }

  // Z Image Turbo uses runsync — returns result directly, no polling needed
  // 30-second timeout to avoid blocking the worker on slow responses
  const timeoutMs = 30_000;
  const result = await Promise.race([
    endpoints.zitImage.runSync(input),
    new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error('Z Image Turbo timed out after 30s')), timeoutMs)
    ),
  ]);

  if (result.status === 'FAILED') {
    throw new Error(result.error || 'Z Image Turbo generation failed');
  }

  const imageUrl = result.output?.result;
  if (!imageUrl) {
    throw new Error('Z Image Turbo returned no image URL');
  }

  // Download from RunPod's temporary URL and upload to R2
  let r2Url: string | undefined;
  if (dream_uuid && auto_upload !== false) {
    try {
      await videoServiceClient.uploadGeneratedImage(dream_uuid, imageUrl, result.output?.generation_time);
      r2Url = imageUrl;
    } catch (error: any) {
      console.error(`Failed to upload Z Image Turbo result for dream ${dream_uuid}:`, error.message || error);
    }
  }

  return {
    ...result.output,
    r2_url: r2Url,
    runpod_status: result.status,
  };
}
```

- [ ] **Step 3: Add the import for endpoints if not already imported**

Verify that `endpoints` and `videoServiceClient` are already imported at the top of `job-handlers.ts` (they should be — they're used by existing handlers).

- [ ] **Step 4: Build worker to verify no type errors**

Run: `cd /Users/maxcarlsonold/edream/worker && npm run build`

Expected: Compiles with 0 errors.

- [ ] **Step 5: Commit**

```bash
cd /Users/maxcarlsonold/edream/worker
git add src/workers/job-handlers.ts
git commit -m "feat: add handleZitImageJob using runsync for Z Image Turbo"
```

---

### Task 3: Register Z Image Turbo Queue and Worker

**Files:**
- Modify: `worker/src/services/cli.service.ts`
- Modify: `worker/src/index.ts`

- [ ] **Step 1: Add zit-image to CLI routing**

In `worker/src/services/cli.service.ts`, add the queue to the `queues` record (around line 37):

```typescript
  private readonly queues: Record<string, QueueConfig> = {
    video: this.createQueueConfig('video'),
    hunyuanvideo: this.createQueueConfig('hunyuanvideo'),
    deforumvideo: this.createQueueConfig('deforumvideo'),
    uprezvideo: this.createQueueConfig('uprezvideo'),
    want2v: this.createQueueConfig('want2v'),
    wani2v: this.createQueueConfig('wani2v'),
    wani2vlora: this.createQueueConfig('wani2vlora'),
    qwenimage: this.createQueueConfig('qwenimage'),
    zitimage: this.createQueueConfig('zitimage'),
  };
```

- [ ] **Step 2: Add zit-image case to inferQueueName**

In the `switch` block in `inferQueueName` (around line 85), add:

```typescript
      case 'zit-image':
        return 'zitimage';
```

- [ ] **Step 3: Update the error message to include zit-image**

Update the error messages in `inferQueueName` to include `zit-image` in the allowed values list:

```typescript
    if (!algorithm) {
      throw new InvalidArgumentError(
        "Missing 'infinidream_algorithm'. Allowed values: animatediff, hunyuan, deforum, uprez, wan-t2v, wan-i2v, wan-i2v-lora, qwen-image, zit-image"
      );
    }
```

And in the `default` case:

```typescript
      default:
        throw new InvalidArgumentError(
          `Unknown 'infinidream_algorithm': ${algorithm}. Allowed values: animatediff, hunyuan, deforum, uprez, wan-t2v, wan-i2v, wan-i2v-lora, qwen-image, zit-image`
        );
```

- [ ] **Step 4: Register queue and worker in index.ts**

In `worker/src/index.ts`:

Add import:
```typescript
import {
  handleImageJob,
  handleVideoJob,
  handleHunyuanVideoJob,
  handleDeforumVideoJob,
  handleUprezVideoJob,
  handleWanT2VJob,
  handleWanI2VJob,
  handleWanI2VLoraJob,
  handleQwenImageJob,
  handleZitImageJob,
  handleVideoIngestJob,
} from './workers/job-handlers.js';
```

Add worker registration (after `qwenimage` line):
```typescript
WorkerFactory.createWorker('zitimage', handleZitImageJob);
```

Add queue (after `qwenImageQueue`):
```typescript
const zitImageQueue = new Queue('zitimage', {
  connection: redisClient,
  streams: {
    events: {
      maxLen: 100,
    },
  },
});
```

Add to BullBoard (in the `queues` array of `createBullBoard`):
```typescript
    new BullMQAdapter(zitImageQueue),
```

- [ ] **Step 5: Build worker to verify no type errors**

Run: `cd /Users/maxcarlsonold/edream/worker && npm run build`

Expected: Compiles with 0 errors.

- [ ] **Step 6: Commit**

```bash
cd /Users/maxcarlsonold/edream/worker
git add src/services/cli.service.ts src/index.ts
git commit -m "feat: register zitimage queue, worker, and CLI routing"
```

---

### Task 4: Refactor Frontend Types — `QwenParams` → `ImageGenParams`

**Files:**
- Modify: `frontend/src/types/studio.types.ts`

- [ ] **Step 1: Add ImageModel type and rename QwenParams**

In `frontend/src/types/studio.types.ts`, replace the `QwenParams` interface:

```typescript
// Before:
export interface QwenParams {
  seedCount: number;
  size: string;
}

// After:
export type ImageModel = 'qwen-image' | 'zit-image';

export interface ImageGenParams {
  model: ImageModel;
  seedCount: number;
  size: string;
}
```

- [ ] **Step 2: Run type-check to see what breaks**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run type-check 2>&1 | head -30`

Expected: Errors in `studio.store.ts` and `images-tab.tsx` referencing `QwenParams`.

- [ ] **Step 3: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/types/studio.types.ts
git commit -m "feat: add ImageModel type, rename QwenParams to ImageGenParams"
```

---

### Task 5: Update Zustand Store — `qwenParams` → `imageGenParams` with Migration

**Files:**
- Modify: `frontend/src/stores/studio.store.ts`
- Modify: `frontend/src/stores/studio.store.test.ts`

- [ ] **Step 1: Write failing test for migration**

In `frontend/src/stores/studio.store.test.ts`, add a test for the v2→v3 migration:

```typescript
describe("migration v2 → v3", () => {
  it("renames qwenParams to imageGenParams with default model", () => {
    // Simulate v2 persisted state
    const v2State = {
      qwenParams: { seedCount: 8, size: "1280*720" },
    };

    // Run migration
    const migrated = useStudioStore.persist?.options?.migrate?.(v2State, 2) as Record<string, unknown>;

    expect(migrated.imageGenParams).toEqual({
      model: "qwen-image",
      seedCount: 8,
      size: "1280*720",
    });
    expect(migrated.qwenParams).toBeUndefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run test`

Expected: FAIL — `imageGenParams` is undefined because migration doesn't exist yet.

- [ ] **Step 3: Update the store**

In `frontend/src/stores/studio.store.ts`:

Update the import:
```typescript
import type {
  StudioTab,
  StudioImage,
  StudioAction,
  StudioJob,
  ImageGenParams,
  WanI2VParams,
} from "@/types/studio.types";
```

Replace all `qwenParams` references in the type definition:
```typescript
// Before:
  qwenParams: QwenParams;
  setQwenParams: (params: Partial<QwenParams>) => void;

// After:
  imageGenParams: ImageGenParams;
  setImageGenParams: (params: Partial<ImageGenParams>) => void;
```

Replace the default:
```typescript
// Before:
const DEFAULT_QWEN_PARAMS: QwenParams = { seedCount: 8, size: "1280*720" };

// After:
const DEFAULT_IMAGE_GEN_PARAMS: ImageGenParams = { model: "qwen-image", seedCount: 8, size: "1280*720" };
```

Replace the store implementation:
```typescript
// Before:
      qwenParams: DEFAULT_QWEN_PARAMS,
      setQwenParams: (params: Partial<QwenParams>) =>
        set((s) => ({ qwenParams: { ...s.qwenParams, ...params } })),

// After:
      imageGenParams: DEFAULT_IMAGE_GEN_PARAMS,
      setImageGenParams: (params: Partial<ImageGenParams>) =>
        set((s) => ({ imageGenParams: { ...s.imageGenParams, ...params } })),
```

Replace in `partialize`:
```typescript
// Before:
        qwenParams: state.qwenParams,

// After:
        imageGenParams: state.imageGenParams,
```

Replace in `resetSession`:
```typescript
// Before:
          qwenParams: DEFAULT_QWEN_PARAMS,

// After:
          imageGenParams: DEFAULT_IMAGE_GEN_PARAMS,
```

Bump version and add migration:
```typescript
      version: 3,
```

In the `migrate` function, after the `version < 2` block:
```typescript
        if (version < 3) {
          // eslint-disable-next-line @typescript-eslint/no-explicit-any
          const qp = state.qwenParams as any;
          if (qp) {
            state.imageGenParams = { model: "qwen-image", ...qp };
            delete state.qwenParams;
          }
        }
```

- [ ] **Step 4: Run tests to verify green**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run test`

Expected: All tests pass.

- [ ] **Step 5: Run type-check**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run type-check`

Expected: Errors in `images-tab.tsx` only (referencing old `qwenParams`/`setQwenParams`). That's fixed in Task 6.

- [ ] **Step 6: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/stores/studio.store.ts src/stores/studio.store.test.ts
git commit -m "feat: rename qwenParams to imageGenParams with model field, add v3 migration"
```

---

### Task 6: Create Size Options Constants

**Files:**
- Create: `frontend/src/components/pages/studio/constants/size-options.ts`

- [ ] **Step 1: Create the file**

```typescript
import type { ImageModel } from "@/types/studio.types";

const QWEN_SIZES = ["1280*720", "1024*1024", "720*1280", "512*512"] as const;

const ZIT_SIZES = [
  "1280*720",
  "1024*1024",
  "720*1280",
  "512*512",
  "768*768",
  "1280*1280",
  "1024*768",
  "768*1024",
] as const;

export const SIZE_OPTIONS: Record<ImageModel, readonly string[]> = {
  "qwen-image": QWEN_SIZES,
  "zit-image": ZIT_SIZES,
};

export const IMAGE_COUNT_OPTIONS = [1, 4, 8, 12, 16, 24] as const;
```

- [ ] **Step 2: Run type-check**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run type-check 2>&1 | head -10`

Expected: Still errors in `images-tab.tsx` (old refs), but no errors in the new file.

- [ ] **Step 3: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/constants/size-options.ts
git commit -m "feat: add per-model size options for image generation"
```

---

### Task 7: Update Images Tab — Model Picker + Dynamic Options

**Files:**
- Modify: `frontend/src/components/pages/studio/components/images-tab.tsx`

- [ ] **Step 1: Update imports and constants**

Replace the top of `images-tab.tsx`:

```typescript
import React, { useCallback, useMemo, useState } from "react";
import { useStudioStore } from "@/stores/studio.store";
import { axiosClient } from "@/client/axios.client";
import type { StudioImage, ImageModel } from "@/types/studio.types";
import {
  SIZE_OPTIONS,
  IMAGE_COUNT_OPTIONS,
} from "../constants/size-options";
import {
  GenerateSection,
  SectionTitle,
  PromptTextarea,
  FormRow,
  FormField,
  FieldLabel,
  StyledSelect,
  GenerateButton,
  ImageGrid,
  ImageCard,
  ImageThumbnail,
  StarBadge,
  ImageStatus,
  SeedLabel,
  BottomRow,
  SelectionCount,
  NavButton,
  SecondaryNavButton,
  EmptyStateText,
  ButtonRow,
  LightboxOverlay,
  LightboxImage,
} from "./images-tab.styled";
import {
  PresignedImage,
  getPresignedUrl,
} from "@/components/shared/presigned-image";
import { AddFromPlaylistModal } from "./add-from-playlist-modal";

const MODEL_LABELS: Record<ImageModel, string> = {
  "zit-image": "Z Image Turbo",
  "qwen-image": "Qwen Image",
};
```

- [ ] **Step 2: Update store selectors**

Replace the store selectors at the top of the component:

```typescript
export const ImagesTab: React.FC = () => {
  const imagePrompt = useStudioStore((s) => s.imagePrompt);
  const setImagePrompt = useStudioStore((s) => s.setImagePrompt);
  const imageGenParams = useStudioStore((s) => s.imageGenParams);
  const setImageGenParams = useStudioStore((s) => s.setImageGenParams);
  const images = useStudioStore((s) => s.images);
  const addImage = useStudioStore((s) => s.addImage);
  const toggleImageSelected = useStudioStore((s) => s.toggleImageSelected);
  const selectAllImages = useStudioStore((s) => s.selectAllImages);
  const deselectAllImages = useStudioStore((s) => s.deselectAllImages);
  const setActiveTab = useStudioStore((s) => s.setActiveTab);

  const isGenerating = useStudioStore((s) => s.isGenerating);
  const setIsGenerating = useStudioStore((s) => s.setIsGenerating);
  const [showPlaylistModal, setShowPlaylistModal] = useState(false);
  const [expandedImageUuid, setExpandedImageUuid] = useState<string | null>(
    null,
  );

  const sizeOptions = SIZE_OPTIONS[imageGenParams.model];
```

- [ ] **Step 3: Update handleGenerate to use imageGenParams**

```typescript
  const handleGenerate = useCallback(async () => {
    if (!imagePrompt.trim()) return;
    setIsGenerating(true);

    const baseSeed = Math.floor(Math.random() * 99_000) + 1;

    const promises = Array.from({ length: imageGenParams.seedCount }, (_, i) => {
      const seed = baseSeed + i;
      const algoParams = {
        infinidream_algorithm: imageGenParams.model,
        prompt: imagePrompt,
        size: imageGenParams.size,
        seed,
      };

      const modelLabel = MODEL_LABELS[imageGenParams.model];

      return axiosClient
        .post("/v1/dream", {
          name: `${modelLabel} ${images.length + i + 1}`,
          prompt: JSON.stringify(algoParams),
          description: "Studio generated image",
        })
        .then(({ data }) => {
          const dream = data.data?.dream;
          if (!dream) return;
          addImage({
            uuid: dream.uuid,
            url: dream.thumbnail || "",
            name: dream.name,
            seed,
            size: imageGenParams.size,
            status: (dream.status as StudioImage["status"]) || "queue",
            selected: false,
          });
        })
        .catch((err) => {
          console.error("Failed to create image:", err);
        });
    });

    await Promise.all(promises);
    setIsGenerating(false);
  }, [imagePrompt, imageGenParams, images.length, addImage, setIsGenerating]);
```

- [ ] **Step 4: Update the JSX form row**

Replace the `<FormRow>` block with model picker added:

```typescript
        <FormRow>
          <FormField>
            <FieldLabel>Model:</FieldLabel>
            <StyledSelect
              value={imageGenParams.model}
              onChange={(e) =>
                setImageGenParams({ model: e.target.value as ImageModel })
              }
            >
              {(Object.keys(MODEL_LABELS) as ImageModel[]).map((m) => (
                <option key={m} value={m}>
                  {MODEL_LABELS[m]}
                </option>
              ))}
            </StyledSelect>
          </FormField>
          <FormField>
            <FieldLabel>Images:</FieldLabel>
            <StyledSelect
              value={imageGenParams.seedCount}
              onChange={(e) =>
                setImageGenParams({ seedCount: Number(e.target.value) })
              }
            >
              {IMAGE_COUNT_OPTIONS.map((n) => (
                <option key={n} value={n}>
                  {n}
                </option>
              ))}
            </StyledSelect>
          </FormField>
          <FormField>
            <FieldLabel>Size:</FieldLabel>
            <StyledSelect
              value={imageGenParams.size}
              onChange={(e) => setImageGenParams({ size: e.target.value })}
            >
              {sizeOptions.map((s) => (
                <option key={s} value={s}>
                  {s.replace("*", "x")}
                </option>
              ))}
            </StyledSelect>
          </FormField>
          <GenerateButton
            onClick={handleGenerate}
            disabled={!imagePrompt.trim() || isGenerating}
          >
            {isGenerating ? "Generating..." : "Generate Images"}
          </GenerateButton>
        </FormRow>
```

- [ ] **Step 5: Run type-check and lint**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run type-check && pnpm run lint`

Expected: 0 errors.

- [ ] **Step 6: Run tests**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run test`

Expected: All tests pass.

- [ ] **Step 7: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/components/images-tab.tsx
git commit -m "feat: add model picker to images tab, support Z Image Turbo and Qwen"
```

---

### Task 8: Verify End-to-End

This task is manual verification — no code changes.

- [ ] **Step 1: Start frontend dev server**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run dev`

- [ ] **Step 2: Open studio page**

Navigate to `http://localhost:5173/studio`

- [ ] **Step 3: Verify model picker**

- Model dropdown shows "Z Image Turbo" and "Qwen Image"
- Switching model changes size options
- "Images:" label (not "Seeds:")
- Image cards show `#42` format (not `seed:42`)

- [ ] **Step 4: Verify selection persists on reload**

- Select Z Image Turbo → reload page → still selected

- [ ] **Step 5: Test generation (if backend is running)**

- Select Z Image Turbo → enter prompt → generate → verify dreams are created with `infinidream_algorithm: "zit-image"` in the prompt JSON

---

## Summary of All Changes

| File | Changes |
|------|---------|
| `worker/src/config/runpod.config.ts` | Add `zitImage` endpoint |
| `worker/src/workers/job-handlers.ts` | Add `ZitImageParams` interface, `handleZitImageJob` function |
| `worker/src/services/cli.service.ts` | Add `zitimage` queue, `zit-image` case, update error messages |
| `worker/src/index.ts` | Register `zitimage` queue, worker, BullBoard adapter |
| `frontend/src/types/studio.types.ts` | Add `ImageModel` type, rename `QwenParams` → `ImageGenParams` |
| `frontend/src/stores/studio.store.ts` | Rename `qwenParams` → `imageGenParams`, add v3 migration |
| `frontend/src/stores/studio.store.test.ts` | Add migration test |
| `frontend/src/components/pages/studio/constants/size-options.ts` | New: per-model size options |
| `frontend/src/components/pages/studio/components/images-tab.tsx` | Model picker, dynamic sizes, dynamic dream name |

## Status (updated 2026-03-31)

**This plan is fully implemented.** All tasks (1-8) are complete.

- Worker: Patrick shipped Z Image Turbo to `worker/stage` (endpoint, handler, queue, CLI routing)
- Backend: Patrick shipped to `backend/stage` (algorithm routing, cancellation)
- Frontend: All in [frontend#587](https://github.com/e-dream-ai/frontend/pull/587) (model picker, sizes, dream naming, store refactor)

## Items from "Blocked" section — now also done

These were listed as blocked on Jef but have since been implemented:

- ~~LTX 2.3 integration~~ → worker#31, backend#412, frontend#587, gpu-container-ltx
- ~~VAE preview~~ → TAESD baked into gpu-container-ltx, `--preview-method taesd` enabled
- ~~Nvidia VSR upscaling~~ → worker#31, backend#412, frontend#587, gpu-container-nvidia-vsr
- ~~videoGenParams store refactoring~~ → frontend#587
- ~~Action preset model scoping~~ → frontend#587
- ~~Hardcoded algorithm fixes~~ → frontend#587

**Still blocked on Jef:** LTX optimized workflow, LoRA presets, duration constraints with LoRAs, Nvidia VSR quality tuning. See design spec for details.
