# Studio Review Fixes Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Fix all 16 issues identified in the studio feature code review — covering Socket.IO status mapping, Wan I2V params, results grid visibility, uprez retry, type safety, and UX consistency.

**Architecture:** All changes are frontend-only, touching the studio Zustand store, types, hooks, and tab components. No backend changes needed. The fixes follow the existing patterns (Zustand persist, Socket.IO subscription, axiosClient calls).

**Tech Stack:** React, TypeScript, Zustand, Socket.IO, TanStack Query

---

### Task 1: Add Socket.IO Status Mapping Helper

**Files:**
- Create: `frontend/src/components/pages/studio/hooks/mapSocketStatus.ts`

**Step 1: Create the status mapping utility**

```typescript
// frontend/src/components/pages/studio/hooks/mapSocketStatus.ts
import type { StudioImage } from "@/types/studio.types";

/**
 * Maps Socket.IO job:progress status strings (UPPERCASE) to
 * StudioImage/StudioJob status strings (lowercase).
 * Returns undefined if the status is unrecognized (caller should skip update).
 */
export const mapSocketStatus = (
  raw?: string,
): StudioImage["status"] | undefined => {
  if (raw === "COMPLETED") return "processed";
  if (raw === "IN_PROGRESS") return "processing";
  if (raw === "IN_QUEUE") return "queue";
  if (raw === "FAILED") return "failed";
  return undefined;
};
```

**Step 2: Commit**

```bash
git add frontend/src/components/pages/studio/hooks/mapSocketStatus.ts
git commit -m "feat(studio): add Socket.IO status mapping helper"
```

---

### Task 2: Fix Socket.IO Progress Handler — Status Mapping + Stable Listener

**Files:**
- Modify: `frontend/src/components/pages/studio/hooks/useStudioJobProgress.ts`

This is the most critical fix. Two changes:
1. Use `mapSocketStatus` for all status transitions (not just COMPLETED)
2. Split the `JOB_PROGRESS_EVENT` listener into a separate `useEffect` depending only on `[socket]`, consulting refs for data. Room management stays in its own effect tracking `pendingUuids`.

**Step 1: Rewrite useStudioJobProgress.ts**

Replace the entire file content with:

```typescript
import { useEffect, useRef, useMemo, useCallback } from "react";
import useSocket from "@/hooks/useSocket";
import { axiosClient } from "@/client/axios.client";
import { useStudioStore } from "@/stores/studio.store";
import {
  JOB_PROGRESS_EVENT,
  JOIN_DREAM_ROOM_EVENT,
  LEAVE_DREAM_ROOM_EVENT,
} from "@/constants/remote-control.constants";
import { mapSocketStatus } from "./mapSocketStatus";

const POLL_INTERVAL_MS = 10_000;

export const useStudioJobProgress = () => {
  const { socket } = useSocket();
  const images = useStudioStore((s) => s.images);
  const updateImage = useStudioStore((s) => s.updateImage);
  const jobs = useStudioStore((s) => s.jobs);
  const updateJob = useStudioStore((s) => s.updateJob);
  const activeTab = useStudioStore((s) => s.activeTab);
  const incrementNewCompleted = useStudioStore((s) => s.incrementNewCompleted);

  // Refs for live data accessed inside socket/poll handlers
  const imagesRef = useRef(images);
  const jobsRef = useRef(jobs);
  const activeTabRef = useRef(activeTab);
  useEffect(() => { imagesRef.current = images; }, [images]);
  useEffect(() => { jobsRef.current = jobs; }, [jobs]);
  useEffect(() => { activeTabRef.current = activeTab; }, [activeTab]);

  // Stable refs for store actions (never change)
  const updateImageRef = useRef(updateImage);
  const updateJobRef = useRef(updateJob);
  const incrementNewCompletedRef = useRef(incrementNewCompleted);
  useEffect(() => { updateImageRef.current = updateImage; }, [updateImage]);
  useEffect(() => { updateJobRef.current = updateJob; }, [updateJob]);
  useEffect(() => { incrementNewCompletedRef.current = incrementNewCompleted; }, [incrementNewCompleted]);

  // Stable set of pending UUIDs that need socket rooms
  const pendingUuids = useMemo(() => {
    const imageUuids = images
      .filter((img) => img.status === "queue" || img.status === "processing")
      .map((img) => img.uuid);
    const jobUuids = jobs
      .filter((j) => j.status === "queue" || j.status === "processing")
      .map((j) => j.dreamUuid);
    return [...imageUuids, ...jobUuids];
  }, [images, jobs]);

  // --- Effect 1: Register JOB_PROGRESS_EVENT listener ONCE per socket ---
  useEffect(() => {
    if (!socket) return;

    const handleProgress = (data: {
      dream_uuid: string;
      status?: string;
      progress?: number;
      preview_frame?: string;
    }) => {
      const { dream_uuid, progress, preview_frame } = data;
      const mappedStatus = mapSocketStatus(data.status);

      const image = imagesRef.current.find((img) => img.uuid === dream_uuid);
      if (image) {
        updateImageRef.current(dream_uuid, {
          progress,
          previewFrame: preview_frame,
          ...(mappedStatus ? { status: mappedStatus } : {}),
        });
      }

      const job = jobsRef.current.find((j) => j.dreamUuid === dream_uuid);
      if (job) {
        const wasNotCompleted = job.status !== "processed";
        const isNowCompleted = mappedStatus === "processed";

        updateJobRef.current(dream_uuid, {
          progress,
          previewFrame: preview_frame,
          ...(mappedStatus ? { status: mappedStatus } : {}),
        });

        if (wasNotCompleted && isNowCompleted && activeTabRef.current !== "results") {
          incrementNewCompletedRef.current();
        }
      }
    };

    socket.on(JOB_PROGRESS_EVENT, handleProgress);
    return () => {
      socket.off(JOB_PROGRESS_EVENT, handleProgress);
    };
  }, [socket]);

  // --- Effect 2: Join/leave socket rooms for pending UUIDs ---
  useEffect(() => {
    if (!socket || pendingUuids.length === 0) return;

    const joinRooms = () => {
      pendingUuids.forEach((uuid) => {
        socket.emit(JOIN_DREAM_ROOM_EVENT, uuid);
      });
    };

    if (socket.connected) joinRooms();
    socket.on("connect", joinRooms);

    return () => {
      socket.off("connect", joinRooms);
      pendingUuids.forEach((uuid) => {
        socket.emit(LEAVE_DREAM_ROOM_EVENT, uuid);
      });
    };
  }, [socket, pendingUuids]);

  // --- Effect 3: Polling fallback ---
  const pollPendingRef = useRef<() => Promise<void>>();
  useEffect(() => {
    pollPendingRef.current = async () => {
      const pendingImages = imagesRef.current.filter(
        (img) => img.status === "queue" || img.status === "processing",
      );
      const pendingJobs = jobsRef.current.filter(
        (j) => j.status === "queue" || j.status === "processing",
      );

      for (const img of pendingImages) {
        try {
          const { data } = await axiosClient.get(`/v1/dream/${img.uuid}`);
          const dream = data.data.dream;
          updateImageRef.current(img.uuid, {
            status: dream.status,
            url: dream.thumbnail || dream.video || img.url,
          });
        } catch {
          // Silently skip failed polls
        }
      }

      for (const job of pendingJobs) {
        try {
          const { data } = await axiosClient.get(`/v1/dream/${job.dreamUuid}`);
          const dream = data.data.dream;
          const wasNotCompleted = job.status !== "processed";
          const isNowCompleted = dream.status === "processed";

          updateJobRef.current(job.dreamUuid, {
            status: dream.status,
          });

          if (wasNotCompleted && isNowCompleted && activeTabRef.current !== "results") {
            incrementNewCompletedRef.current();
          }
        } catch {
          // Silently skip failed polls
        }
      }
    };
  }, []);

  const hasPending = pendingUuids.length > 0;

  useEffect(() => {
    if (!hasPending) return;
    const interval = setInterval(() => {
      pollPendingRef.current?.();
    }, POLL_INTERVAL_MS);
    return () => clearInterval(interval);
  }, [hasPending]);
};
```

**Step 2: Commit**

```bash
git add frontend/src/components/pages/studio/hooks/useStudioJobProgress.ts
git commit -m "fix(studio): map all Socket.IO statuses and stabilize event listener"
```

---

### Task 3: Add `size` to StudioImage + `jobType` to StudioJob + Fix Hash

**Files:**
- Modify: `frontend/src/types/studio.types.ts`

**Step 1: Update types and fix hash function**

Replace the entire file:

```typescript
export type StudioTab = "images" | "actions" | "generate" | "results";

export interface StudioImage {
  uuid: string;
  url: string;
  name: string;
  seed?: number;
  size?: string;
  status: "queue" | "processing" | "processed" | "failed";
  progress?: number;
  selected: boolean;
  previewFrame?: string;
}

export interface LoRAConfig {
  path: string;
  scale: number;
}

export interface StudioAction {
  id: string;
  prompt: string;
  enabled: boolean;
  highNoiseLoras?: LoRAConfig[];
  lowNoiseLoras?: LoRAConfig[];
}

export type StudioJobType = "wan-i2v" | "uprez";

export interface StudioJob {
  imageId: string;
  actionId: string;
  dreamUuid: string;
  jobType: StudioJobType;
  status: "queue" | "processing" | "processed" | "failed";
  progress?: number;
  previewFrame?: string;
  selectedForUprez: boolean;
  startedAt?: number;
  completedAt?: number;
}

export interface WanI2VParams {
  duration: number;
  numInferenceSteps: number;
  guidance: number;
}

export interface QwenParams {
  seedCount: number;
  size: string;
}

export const createComboKey = (
  imageUuid: string,
  actionPrompt: string,
): string => {
  const hash = Math.abs(
    Array.from(actionPrompt).reduce(
      (h, c) => ((h << 5) - h + c.charCodeAt(0)) | 0,
      0,
    ),
  )
    .toString(16)
    .padStart(8, "0")
    .slice(-8);
  return `${imageUuid}:${hash}`;
};
```

Changes:
- Added `size?: string` to `StudioImage`
- Added `StudioJobType` type and `jobType: StudioJobType` to `StudioJob`
- Fixed `createComboKey` hash: wrap in `Math.abs()` and `padStart` to avoid negative hex and ensure 8 chars

**Step 2: Commit**

```bash
git add frontend/src/types/studio.types.ts
git commit -m "feat(studio): add size to StudioImage, jobType to StudioJob, fix hash"
```

---

### Task 4: Update Zustand Store — isGenerating, newCompletedCount, partialize

**Files:**
- Modify: `frontend/src/stores/studio.store.ts`

**Step 1: Update the store**

Replace the entire file:

```typescript
import { create } from "zustand";
import { persist } from "zustand/middleware";
import type {
  StudioTab,
  StudioImage,
  StudioAction,
  StudioJob,
  QwenParams,
  WanI2VParams,
} from "@/types/studio.types";

type StudioState = {
  activeTab: StudioTab;
  setActiveTab: (tab: StudioTab) => void;

  imagePrompt: string;
  setImagePrompt: (prompt: string) => void;
  qwenParams: QwenParams;
  setQwenParams: (params: Partial<QwenParams>) => void;
  images: StudioImage[];
  addImage: (image: StudioImage) => void;
  updateImage: (uuid: string, updates: Partial<StudioImage>) => void;
  toggleImageSelected: (uuid: string) => void;
  removeImage: (uuid: string) => void;

  isGenerating: boolean;
  setIsGenerating: (v: boolean) => void;

  actions: StudioAction[];
  addAction: (action: StudioAction) => void;
  updateAction: (id: string, updates: Partial<StudioAction>) => void;
  removeAction: (id: string) => void;
  toggleActionEnabled: (id: string) => void;
  loadPresetPack: (actions: StudioAction[]) => void;

  wanParams: WanI2VParams;
  setWanParams: (params: Partial<WanI2VParams>) => void;
  outputPlaylistId: string | null;
  setOutputPlaylistId: (id: string | null) => void;

  excludedCombos: Set<string>;
  toggleComboExcluded: (key: string) => void;

  jobs: StudioJob[];
  addJob: (job: StudioJob) => void;
  updateJob: (dreamUuid: string, updates: Partial<StudioJob>) => void;
  removeJob: (dreamUuid: string) => void;
  toggleJobUprez: (dreamUuid: string) => void;

  newCompletedCount: number;
  incrementNewCompleted: () => void;
  clearNewCompleted: () => void;

  resetSession: () => void;
};

const DEFAULT_QWEN_PARAMS: QwenParams = { seedCount: 8, size: "1280*720" };
const DEFAULT_WAN_PARAMS: WanI2VParams = {
  duration: 5,
  numInferenceSteps: 30,
  guidance: 5.0,
};

export const useStudioStore = create<StudioState>()(
  persist(
    (set) => ({
      activeTab: "images" as StudioTab,
      setActiveTab: (tab: StudioTab) => {
        if (tab === "results") set({ newCompletedCount: 0 });
        set({ activeTab: tab });
      },

      imagePrompt: "",
      setImagePrompt: (prompt: string) => set({ imagePrompt: prompt }),
      qwenParams: DEFAULT_QWEN_PARAMS,
      setQwenParams: (params: Partial<QwenParams>) =>
        set((s) => ({ qwenParams: { ...s.qwenParams, ...params } })),
      images: [] as StudioImage[],
      addImage: (image: StudioImage) =>
        set((s) => ({ images: [...s.images, image] })),
      updateImage: (uuid: string, updates: Partial<StudioImage>) =>
        set((s) => ({
          images: s.images.map((img) =>
            img.uuid === uuid ? { ...img, ...updates } : img,
          ),
        })),
      toggleImageSelected: (uuid: string) =>
        set((s) => ({
          images: s.images.map((img) =>
            img.uuid === uuid ? { ...img, selected: !img.selected } : img,
          ),
        })),
      removeImage: (uuid: string) =>
        set((s) => ({ images: s.images.filter((img) => img.uuid !== uuid) })),

      isGenerating: false,
      setIsGenerating: (v: boolean) => set({ isGenerating: v }),

      actions: [] as StudioAction[],
      addAction: (action: StudioAction) =>
        set((s) => ({ actions: [...s.actions, action] })),
      updateAction: (id: string, updates: Partial<StudioAction>) =>
        set((s) => ({
          actions: s.actions.map((a) =>
            a.id === id ? { ...a, ...updates } : a,
          ),
        })),
      removeAction: (id: string) =>
        set((s) => ({ actions: s.actions.filter((a) => a.id !== id) })),
      toggleActionEnabled: (id: string) =>
        set((s) => ({
          actions: s.actions.map((a) =>
            a.id === id ? { ...a, enabled: !a.enabled } : a,
          ),
        })),
      loadPresetPack: (newActions: StudioAction[]) =>
        set((s) => ({ actions: [...s.actions, ...newActions] })),

      wanParams: DEFAULT_WAN_PARAMS,
      setWanParams: (params: Partial<WanI2VParams>) =>
        set((s) => ({ wanParams: { ...s.wanParams, ...params } })),
      outputPlaylistId: null,
      setOutputPlaylistId: (id: string | null) => set({ outputPlaylistId: id }),

      excludedCombos: new Set<string>(),
      toggleComboExcluded: (key: string) =>
        set((s) => {
          const next = new Set(s.excludedCombos);
          if (next.has(key)) next.delete(key);
          else next.add(key);
          return { excludedCombos: next };
        }),

      jobs: [] as StudioJob[],
      addJob: (job: StudioJob) => set((s) => ({ jobs: [...s.jobs, job] })),
      updateJob: (dreamUuid: string, updates: Partial<StudioJob>) =>
        set((s) => ({
          jobs: s.jobs.map((j) => {
            if (j.dreamUuid !== dreamUuid) return j;
            const merged = { ...j, ...updates };
            if (updates.status === "processing" && !j.startedAt) {
              merged.startedAt = Date.now();
            }
            if (updates.status === "processed" && !j.completedAt) {
              merged.completedAt = Date.now();
            }
            return merged;
          }),
        })),
      removeJob: (dreamUuid: string) =>
        set((s) => ({
          jobs: s.jobs.filter((j) => j.dreamUuid !== dreamUuid),
        })),
      toggleJobUprez: (dreamUuid: string) =>
        set((s) => ({
          jobs: s.jobs.map((j) =>
            j.dreamUuid === dreamUuid
              ? { ...j, selectedForUprez: !j.selectedForUprez }
              : j,
          ),
        })),

      newCompletedCount: 0,
      incrementNewCompleted: () =>
        set((s) => ({ newCompletedCount: s.newCompletedCount + 1 })),
      clearNewCompleted: () => set({ newCompletedCount: 0 }),

      resetSession: () =>
        set({
          activeTab: "images" as StudioTab,
          imagePrompt: "",
          qwenParams: DEFAULT_QWEN_PARAMS,
          images: [],
          actions: [],
          wanParams: DEFAULT_WAN_PARAMS,
          outputPlaylistId: null,
          excludedCombos: new Set<string>(),
          jobs: [],
          newCompletedCount: 0,
          isGenerating: false,
        }),
    }),
    {
      name: "studio-session",
      version: 2,
      partialize: (state) => ({
        activeTab: state.activeTab,
        imagePrompt: state.imagePrompt,
        qwenParams: state.qwenParams,
        images: state.images.map((img) => ({
          ...img,
          previewFrame: undefined,
        })),
        actions: state.actions,
        wanParams: state.wanParams,
        outputPlaylistId: state.outputPlaylistId,
        excludedCombos: [...state.excludedCombos],
        jobs: state.jobs.map((j) => ({ ...j, previewFrame: undefined })),
        // Intentionally excluded: newCompletedCount, isGenerating
      }),
      storage: {
        getItem: (name) => {
          const raw = localStorage.getItem(name);
          if (!raw) return null;
          const parsed = JSON.parse(raw);
          if (parsed?.state?.excludedCombos) {
            parsed.state.excludedCombos = new Set(parsed.state.excludedCombos);
          }
          return parsed;
        },
        setItem: (name, value) => {
          const serializable = {
            ...value,
            state: {
              ...value.state,
              excludedCombos: Array.isArray(value.state.excludedCombos)
                ? value.state.excludedCombos
                : value.state.excludedCombos
                  ? [...value.state.excludedCombos]
                  : [],
            },
          };
          localStorage.setItem(name, JSON.stringify(serializable));
        },
        removeItem: (name) => localStorage.removeItem(name),
      },
      migrate: (persisted: any, version: number) => {
        if (version < 2) {
          // Add jobType to any persisted jobs missing it
          if (persisted.jobs) {
            persisted.jobs = persisted.jobs.map((j: any) => ({
              ...j,
              jobType: j.jobType ?? (j.actionId?.startsWith("uprez-") ? "uprez" : "wan-i2v"),
            }));
          }
        }
        return persisted;
      },
    },
  ),
);

export default useStudioStore;
```

Key changes:
- Added `isGenerating` / `setIsGenerating` to store (persists across tab switches)
- `newCompletedCount` excluded from `partialize` (no stale badge on reload)
- `excludedCombos` converted to array in `partialize` (safer serialization)
- `setItem` handles both array and Set for `excludedCombos`
- Bumped `version` to 2 with `migrate` to add `jobType` to old persisted jobs
- `isGenerating` excluded from `partialize` (ephemeral)
- `resetSession` also resets `isGenerating`

**Step 2: Commit**

```bash
git add frontend/src/stores/studio.store.ts
git commit -m "fix(studio): move isGenerating to store, fix serialization, exclude stale badge"
```

---

### Task 5: Update ImagesTab — Use Store isGenerating + Store Image Size

**Files:**
- Modify: `frontend/src/components/pages/studio/components/images-tab.tsx`

**Step 1: Update ImagesTab to use store isGenerating and store image size**

Changes to make:
1. Replace local `useState(false)` for `isGenerating` with store selectors
2. Add `size` from `qwenParams.size` when creating images

In the component, change:
```typescript
// REMOVE:
const [isGenerating, setIsGenerating] = useState(false);

// ADD:
const isGenerating = useStudioStore((s) => s.isGenerating);
const setIsGenerating = useStudioStore((s) => s.setIsGenerating);
```

In `handleGenerate`, update the `addImage` call to include `size`:
```typescript
addImage({
  uuid: dream.uuid,
  url: dream.thumbnail || "",
  name: dream.name,
  seed,
  size: qwenParams.size,
  status: (dream.status as StudioImage["status"]) || "queue",
  selected: false,
});
```

Update `useCallback` deps — remove `addImage` standalone (it's stable), add `setIsGenerating`.

**Step 2: Commit**

```bash
git add frontend/src/components/pages/studio/components/images-tab.tsx
git commit -m "fix(studio): use store isGenerating, store image size on generation"
```

---

### Task 6: Fix useBatchSubmit — Use Image Size, Add LoRA Params

**Files:**
- Modify: `frontend/src/components/pages/studio/hooks/useBatchSubmit.ts`

**Step 1: Update batch submit to use image size and complete LoRA params**

Changes:
1. Use `image.size || "1280*720"` instead of hardcoded size for wan-i2v
2. Add `num_inference_steps` and `guidance` to wan-i2v-lora path
3. Add `jobType: "wan-i2v"` to addJob call

```typescript
const algoParams = hasLoras
  ? {
      infinidream_algorithm: "wan-i2v-lora" as const,
      prompt: action.prompt,
      image: image.uuid,
      duration: wanParams.duration,
      num_inference_steps: wanParams.numInferenceSteps,
      guidance: wanParams.guidance,
      seed: -1,
      high_noise_loras: action.highNoiseLoras ?? [],
      low_noise_loras: action.lowNoiseLoras ?? [],
    }
  : {
      infinidream_algorithm: "wan-i2v" as const,
      prompt: action.prompt,
      image: image.uuid,
      size: image.size || "1280*720",
      duration: wanParams.duration,
      num_inference_steps: wanParams.numInferenceSteps,
      guidance: wanParams.guidance,
    };
```

And in the addJob call:
```typescript
addJob({
  imageId: image.uuid,
  actionId: action.id,
  dreamUuid: dream.uuid,
  jobType: "wan-i2v",
  status: (dream.status as "queue" | "processing" | "processed" | "failed") || "queue",
  selectedForUprez: false,
});
```

**Step 2: Commit**

```bash
git add frontend/src/components/pages/studio/hooks/useBatchSubmit.ts
git commit -m "fix(studio): use image size for wan-i2v, add missing LoRA params, add jobType"
```

---

### Task 7: Fix ResultsTab — Derive Grid from Jobs, Fix Retry/Uprez, Fix Casts

**Files:**
- Modify: `frontend/src/components/pages/studio/components/results-tab.tsx`

This is the largest single change. Multiple fixes:

1. **Derive grid from jobs** — Build rows/columns from unique imageId/actionId in `jobs`, not from selected/enabled filters
2. **Fix forced `as "queue"` casts** — Use proper status mapping
3. **Fix uprez addJob** — Include `jobType: "uprez"`
4. **Fix retry** — Handle uprez jobs separately (skip them), add BATCH_IDENTIFIER to description, add missing LoRA params, use image size
5. **Import createComboKey**

**Step 1: Rewrite results-tab.tsx**

```typescript
import React, { useCallback, useMemo, useState } from "react";
import { useStudioStore } from "@/stores/studio.store";
import { useCreateDreamFromPrompt } from "@/api/dream/mutation/useCreateDreamFromPrompt";
import { axiosClient } from "@/client/axios.client";
import { createComboKey } from "@/types/studio.types";
import type { StudioJob } from "@/types/studio.types";
import { GenerateSection, SectionTitle } from "./images-tab.styled";
import { GridTable, GridHeader, GridRowHeader } from "./generate-tab.styled";
import {
  ProgressBar,
  ProgressInfo,
  ProgressTrack,
  ProgressFill,
  ResultCell,
  ResultThumb,
  ResultThumbImg,
  ResultCellStatus,
  UprezStarBadge,
  ActionBar,
  ActionButton,
} from "./results-tab.styled";

export const ResultsTab: React.FC = () => {
  const images = useStudioStore((s) => s.images);
  const actions = useStudioStore((s) => s.actions);
  const jobs = useStudioStore((s) => s.jobs);
  const toggleJobUprez = useStudioStore((s) => s.toggleJobUprez);
  const addJob = useStudioStore((s) => s.addJob);
  const outputPlaylistId = useStudioStore((s) => s.outputPlaylistId);
  const setActiveTab = useStudioStore((s) => s.setActiveTab);
  const createDream = useCreateDreamFromPrompt();

  const wanParams = useStudioStore((s) => s.wanParams);
  const removeJob = useStudioStore((s) => s.removeJob);

  const [isUprezzing, setIsUprezzing] = useState(false);
  const [isRetrying, setIsRetrying] = useState(false);

  // Derive grid dimensions from jobs (not from selected/enabled filters)
  const gridImageIds = useMemo(() => {
    const ids = new Set<string>();
    jobs.filter((j) => j.jobType !== "uprez").forEach((j) => ids.add(j.imageId));
    return [...ids];
  }, [jobs]);

  const gridActionIds = useMemo(() => {
    const ids = new Set<string>();
    jobs.filter((j) => j.jobType !== "uprez").forEach((j) => ids.add(j.actionId));
    return [...ids];
  }, [jobs]);

  const gridImages = useMemo(
    () => gridImageIds.map((id) => images.find((img) => img.uuid === id)).filter(Boolean),
    [gridImageIds, images],
  );

  const gridActions = useMemo(
    () => gridActionIds.map((id) => actions.find((a) => a.id === id)).filter(Boolean),
    [gridActionIds, actions],
  );

  const wanJobs = useMemo(() => jobs.filter((j) => j.jobType !== "uprez"), [jobs]);
  const completedCount = wanJobs.filter((j) => j.status === "processed").length;
  const totalCount = wanJobs.length;
  const failedCount = wanJobs.filter((j) => j.status === "failed").length;
  const progressPercent =
    totalCount > 0 ? Math.round((completedCount / totalCount) * 100) : 0;

  const uprezCount = jobs.filter((j) => j.selectedForUprez).length;

  const timeEstimate = useMemo(() => {
    const done = wanJobs.filter((j) => j.startedAt && j.completedAt);
    if (done.length === 0) return null;
    const avgMs =
      done.reduce((sum, j) => sum + (j.completedAt! - j.startedAt!), 0) /
      done.length;
    const remaining = totalCount - completedCount - failedCount;
    if (remaining <= 0) return null;
    const estimateMs = avgMs * remaining;
    const minutes = Math.ceil(estimateMs / 60_000);
    return minutes <= 1 ? "~1 min remaining" : `~${minutes} min remaining`;
  }, [wanJobs, totalCount, completedCount, failedCount]);

  const getJob = useCallback(
    (imageUuid: string, actionId: string) =>
      jobs.find((j) => j.imageId === imageUuid && j.actionId === actionId && j.jobType !== "uprez"),
    [jobs],
  );

  const handleUprezSelected = useCallback(async () => {
    const toUprez = jobs.filter(
      (j) => j.selectedForUprez && j.status === "processed" && j.jobType !== "uprez",
    );
    if (toUprez.length === 0) return;

    setIsUprezzing(true);
    try {
      for (const job of toUprez) {
        const algoParams = {
          infinidream_algorithm: "uprez",
          video_uuid: job.dreamUuid,
          upscale_factor: 2,
          interpolation_factor: 2,
        };

        try {
          const response = await createDream.mutateAsync({
            name: `Uprez - ${job.dreamUuid.slice(0, 8)}`,
            prompt: JSON.stringify(algoParams),
            description: `Uprez of ${job.dreamUuid}`,
          });

          const dream = response.data?.dream;
          if (!dream) continue;

          addJob({
            imageId: job.imageId,
            actionId: `uprez-${job.actionId}`,
            dreamUuid: dream.uuid,
            jobType: "uprez",
            status: (dream.status as StudioJob["status"]) || "queue",
            selectedForUprez: false,
          });

          if (outputPlaylistId) {
            await axiosClient.put(
              `/v1/playlist/${outputPlaylistId}/add-item`,
              { type: "dream", uuid: dream.uuid },
            );
          }
        } catch (err) {
          console.error("Failed to create uprez job:", err);
        }
      }
    } finally {
      setIsUprezzing(false);
    }
  }, [jobs, createDream, addJob, outputPlaylistId]);

  const handleRetryFailed = useCallback(async () => {
    // Only retry wan-i2v jobs (uprez retries not yet supported)
    const failedJobs = jobs.filter((j) => j.status === "failed" && j.jobType !== "uprez");
    if (failedJobs.length === 0) return;

    setIsRetrying(true);
    try {
      for (const job of failedJobs) {
        const image = images.find((img) => img.uuid === job.imageId);
        const action = actions.find((a) => a.id === job.actionId);
        if (!image || !action) continue;

        const hasLoras =
          (action.highNoiseLoras && action.highNoiseLoras.length > 0) ||
          (action.lowNoiseLoras && action.lowNoiseLoras.length > 0);

        const batchIdentifier = createComboKey(image.uuid, action.prompt);

        const algoParams = hasLoras
          ? {
              infinidream_algorithm: "wan-i2v-lora" as const,
              prompt: action.prompt,
              image: image.uuid,
              duration: wanParams.duration,
              num_inference_steps: wanParams.numInferenceSteps,
              guidance: wanParams.guidance,
              seed: -1,
              high_noise_loras: action.highNoiseLoras ?? [],
              low_noise_loras: action.lowNoiseLoras ?? [],
            }
          : {
              infinidream_algorithm: "wan-i2v" as const,
              prompt: action.prompt,
              image: image.uuid,
              size: image.size || "1280*720",
              duration: wanParams.duration,
              num_inference_steps: wanParams.numInferenceSteps,
              guidance: wanParams.guidance,
            };

        try {
          const response = await createDream.mutateAsync({
            name: `${image.name} - ${action.prompt.slice(0, 40)}`,
            prompt: JSON.stringify(algoParams),
            description: `Studio batch retry. BATCH_IDENTIFIER:${batchIdentifier}`,
          });

          const dream = response.data?.dream;
          if (!dream) continue;

          removeJob(job.dreamUuid);
          addJob({
            imageId: job.imageId,
            actionId: job.actionId,
            dreamUuid: dream.uuid,
            jobType: "wan-i2v",
            status: (dream.status as StudioJob["status"]) || "queue",
            selectedForUprez: false,
          });

          if (outputPlaylistId) {
            await axiosClient.put(
              `/v1/playlist/${outputPlaylistId}/add-item`,
              { type: "dream", uuid: dream.uuid },
            );
          }
        } catch (err) {
          console.error("Failed to retry job:", err);
        }
      }
    } finally {
      setIsRetrying(false);
    }
  }, [jobs, images, actions, wanParams, createDream, removeJob, addJob, outputPlaylistId]);

  return (
    <>
      <ProgressBar>
        <ProgressInfo>
          <span>
            {completedCount} of {totalCount} complete
          </span>
          <span>
            {timeEstimate && (
              <span style={{ marginRight: "1rem", color: "#888" }}>
                {timeEstimate}
              </span>
            )}
            {progressPercent}%
          </span>
        </ProgressInfo>
        <ProgressTrack>
          <ProgressFill $percent={progressPercent} />
        </ProgressTrack>
      </ProgressBar>

      <GenerateSection>
        <SectionTitle>Results Matrix</SectionTitle>
        {jobs.length === 0 ? (
          <p style={{ fontSize: "0.875rem", color: "#888" }}>
            No jobs submitted yet. Go to the Generate tab to start a batch.
          </p>
        ) : (
          <div style={{ overflowX: "auto" }}>
            <GridTable>
              <thead>
                <tr>
                  <GridHeader />
                  {gridActions.map((action) =>
                    action ? (
                      <GridHeader key={action.id} title={action.prompt}>
                        {action.prompt.slice(0, 20)}...
                      </GridHeader>
                    ) : null,
                  )}
                </tr>
              </thead>
              <tbody>
                {gridImages.map((image) =>
                  image ? (
                    <tr key={image.uuid}>
                      <GridRowHeader>{image.name}</GridRowHeader>
                      {gridActions.map((action) => {
                        if (!action) return null;
                        const job = getJob(image.uuid, action.id);

                        if (!job) {
                          return (
                            <ResultCell
                              key={`${image.uuid}:${action.id}`}
                            >
                              <ResultCellStatus $color="#555">
                                --
                              </ResultCellStatus>
                            </ResultCell>
                          );
                        }

                        return (
                          <ResultCell
                            key={job.dreamUuid}
                            $status={job.status}
                            onClick={() => {
                              if (job.status === "processed") {
                                window.open(
                                  `/dream/${job.dreamUuid}`,
                                  "_blank",
                                );
                              }
                            }}
                          >
                            <ResultThumb>
                              {job.previewFrame ? (
                                <ResultThumbImg
                                  src={`data:image/jpeg;base64,${job.previewFrame}`}
                                  alt="preview"
                                />
                              ) : null}

                              {job.status === "processed" && (
                                <UprezStarBadge
                                  $active={job.selectedForUprez}
                                  onClick={(e) => {
                                    e.stopPropagation();
                                    toggleJobUprez(job.dreamUuid);
                                  }}
                                >
                                  {job.selectedForUprez ? "\u2605" : "\u2606"}
                                </UprezStarBadge>
                              )}
                            </ResultThumb>

                            <ResultCellStatus
                              $color={
                                job.status === "processed"
                                  ? "#6c6"
                                  : job.status === "failed"
                                    ? "#c66"
                                    : undefined
                              }
                            >
                              {job.status === "processed" && "done"}
                              {job.status === "processing" &&
                                `${job.progress ?? 0}%`}
                              {job.status === "queue" && "queued"}
                              {job.status === "failed" && "failed"}
                            </ResultCellStatus>
                          </ResultCell>
                        );
                      })}
                    </tr>
                  ) : null,
                )}
              </tbody>
            </GridTable>
          </div>
        )}
      </GenerateSection>

      <ActionBar>
        <ActionButton
          $variant="primary"
          disabled={uprezCount === 0 || isUprezzing}
          onClick={handleUprezSelected}
        >
          {isUprezzing ? "Uprezzing..." : `Uprez Selected (${uprezCount})`}
        </ActionButton>
        {failedCount > 0 && (
          <ActionButton
            onClick={handleRetryFailed}
            disabled={isRetrying}
          >
            {isRetrying ? "Retrying..." : `Retry Failed (${failedCount})`}
          </ActionButton>
        )}
        {outputPlaylistId && (
          <ActionButton
            onClick={() =>
              window.open(`/playlist/${outputPlaylistId}`, "_blank")
            }
          >
            View Playlist
          </ActionButton>
        )}
        <ActionButton
          onClick={() => setActiveTab("generate")}
          style={{ marginLeft: "auto" }}
        >
          &larr; Back to Generate
        </ActionButton>
      </ActionBar>
    </>
  );
};
```

**Step 2: Commit**

```bash
git add frontend/src/components/pages/studio/components/results-tab.tsx
git commit -m "fix(studio): derive results grid from jobs, fix retry/uprez, fix status casts"
```

---

### Task 8: Fix GenerateTab — Memoize Combos

**Files:**
- Modify: `frontend/src/components/pages/studio/components/generate-tab.tsx`

**Step 1: Wrap getSelectedCombinations result in useMemo**

Change line 55 from:
```typescript
const newCombos = getSelectedCombinations();
```

To:
```typescript
const newCombos = useMemo(() => getSelectedCombinations(), [getSelectedCombinations]);
```

Also add `useMemo` to the existing imports from React.

**Step 2: Commit**

```bash
git add frontend/src/components/pages/studio/components/generate-tab.tsx
git commit -m "fix(studio): memoize getSelectedCombinations in GenerateTab"
```

---

### Task 9: Fix ActionsTab — Summary Count Consistency

**Files:**
- Modify: `frontend/src/components/pages/studio/components/actions-tab.tsx`

**Step 1: Filter by `img.selected && img.status === "processed"` to match GenerateTab logic**

Change line 36 from:
```typescript
const selectedImageCount = images.filter((img) => img.selected).length;
```

To:
```typescript
const selectedImageCount = images.filter((img) => img.selected && img.status === "processed").length;
```

**Step 2: Commit**

```bash
git add frontend/src/components/pages/studio/components/actions-tab.tsx
git commit -m "fix(studio): actions tab summary matches generate tab filter logic"
```

---

### Task 10: Fix Playlist Modal — Filter by mediaType, Increase Limit

**Files:**
- Modify: `frontend/src/components/pages/studio/components/add-from-playlist-modal.tsx`

**Step 1: Add mediaType filter and increase playlist limit**

Change the playlist fetch (line 53) from `take=50` to `take=200`:
```typescript
.get(`/v1/playlist?userUUID=${user.uuid}&take=200&skip=0`)
```

Change the items filter (lines 67-70) to also check mediaType:
```typescript
setItems(
  data.data.items.filter(
    (item: PlaylistItem) =>
      item.dreamItem?.thumbnail &&
      (!item.dreamItem.mediaType || item.dreamItem.mediaType === "image"),
  ),
);
```

Note: We use `!item.dreamItem.mediaType ||` as a fallback in case older dreams don't have mediaType set.

**Step 2: Commit**

```bash
git add frontend/src/components/pages/studio/components/add-from-playlist-modal.tsx
git commit -m "fix(studio): filter playlist imports by image mediaType, increase limit to 200"
```

---

### Task 11: Fix GenerateTab Playlist Limit

**Files:**
- Modify: `frontend/src/components/pages/studio/components/generate-tab.tsx`

**Step 1: Increase playlist fetch limit from 50 to 200**

Change line 61 from `take=50` to `take=200`:
```typescript
.get(`/v1/playlist?userUUID=${user.uuid}&take=200&skip=0`)
```

**Step 2: Commit**

```bash
git add frontend/src/components/pages/studio/components/generate-tab.tsx
git commit -m "fix(studio): increase playlist fetch limit to 200"
```

---

### Task 12: Run Type Check and Fix Any Remaining Issues

**Step 1: Run type check**

Run: `cd frontend && pnpm run type-check`
Expected: PASS with no errors

**Step 2: If errors, fix them and commit**

---

### Task 13: Final Commit with All Changes

Run `pnpm run type-check` and `pnpm run lint` to verify everything passes, then push.

```bash
git push
```
