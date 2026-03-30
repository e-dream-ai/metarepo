# Studio Page MVP Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a `/studio` page on the existing frontend that enables combinatorial AI video creation: generate images (Qwen) → define actions → batch generate videos (Wan I2V) → review results → uprez winners.

**Architecture:** New React page with tab-based navigation (Images, Actions, Generate, Results). All state lives in a Zustand store (`useStudioStore`). Uses existing backend APIs (`POST /v1/dream`, playlist endpoints) — no backend changes needed. Real-time progress via existing Socket.IO `/remote-control` namespace.

**Tech Stack:** React, TypeScript, Zustand, styled-components, React Query, Socket.IO (existing), Vite

---

## Conventions (from codebase exploration)

These patterns MUST be followed — they match the existing codebase:

| Pattern | Convention | Example File |
|---------|-----------|-------------|
| **Styling** | styled-components + styled-system | `frontend/src/components/shared/button/button.styled.tsx` |
| **State** | Zustand stores in `src/stores/` | `frontend/src/stores/playback.store.ts` |
| **API hooks** | React Query mutations/queries in `src/api/<feature>/` | `frontend/src/api/dream/mutation/useCreateDream.ts` |
| **Routing** | React Router v7 in `src/routes/router.tsx` | Route constants in `src/constants/routes.constants.ts` |
| **Types** | `src/types/*.types.ts` | `frontend/src/types/dream.types.ts` |
| **Imports** | `@/` alias for `src/` | `import { Dream } from "@/types/dream.types"` |
| **i18n** | `useTranslation()` for all user-facing strings | `const { t } = useTranslation()` |
| **Forms** | react-hook-form + yup schemas in `src/schemas/` | `frontend/src/schemas/create-dream.schema.ts` |
| **Permissions** | `usePermission()` + `<Restricted>` wrapper | Creator role required |
| **Pages** | `src/components/pages/<feature>/` | `frontend/src/components/pages/create/` |

## API Contracts (existing endpoints used)

### Dream Creation (for all job types)
```
POST /v1/dream
Body: {
  name: string,
  prompt: JSON.stringify({ infinidream_algorithm: "qwen-image"|"wan-i2v"|"uprez", ...params }),
  description?: string  // Used to store BATCH_IDENTIFIER
}
Response: { success: true, data: { dream: Dream } }
```

### Qwen Image Params
```json
{ "infinidream_algorithm": "qwen-image", "prompt": "text", "size": "1280*720", "seed": -1 }
```

### Wan I2V Params
```json
{ "infinidream_algorithm": "wan-i2v", "prompt": "text", "image": "source-dream-uuid", "duration": 5, "num_inference_steps": 30, "guidance": 5, "size": "1280*720" }
```

### Uprez Params
```json
{ "infinidream_algorithm": "uprez", "video_uuid": "dream-uuid", "upscale_factor": 2, "interpolation_factor": 2 }
```

### Playlist Operations
```
POST /v1/playlist                          → Create playlist
GET  /v1/playlist?userUUID=X&take=N&skip=M → List user playlists
GET  /v1/playlist/{uuid}/items?take=N&skip=M → Get items (paginated)
PUT  /v1/playlist/{uuid}/add-item          → Add dream to playlist
```

### Dream Status Polling
```
GET /v1/dream/{uuid} → { data: { dream: { status, thumbnail, video, ... } } }
```

### Socket.IO Progress (existing)
```
Namespace: /remote-control
Event: job:progress → { dream_uuid, status, progress (0-100), countdown_ms, preview_frame? }
Join room: socket.emit('join_dream_room', dreamUuid)
Leave room: socket.emit('leave_dream_room', dreamUuid)
```

### Preview Frame
```
GET /v1/dream/{uuid}/preview → { data: { preview_frame: "base64 JPEG" | null } }
```

---

## Task 1: Types and Zustand Store

**Files:**
- Create: `frontend/src/types/studio.types.ts`
- Create: `frontend/src/stores/studio.store.ts`

**Step 1: Create Studio types**

```typescript
// frontend/src/types/studio.types.ts

export type StudioTab = "images" | "actions" | "generate" | "results";

export interface StudioImage {
  uuid: string;          // Dream UUID
  url: string;           // Thumbnail/image URL
  name: string;
  seed?: number;
  status: "queue" | "processing" | "processed" | "failed";
  progress?: number;     // 0-100 during processing
  selected: boolean;     // Starred for animation
  previewFrame?: string; // Base64 JPEG during generation
}

export interface StudioAction {
  id: string;
  prompt: string;
  enabled: boolean;
}

export interface StudioJob {
  imageId: string;       // StudioImage uuid
  actionId: string;      // StudioAction id
  dreamUuid: string;     // Created dream UUID
  status: "queue" | "processing" | "processed" | "failed";
  progress?: number;     // 0-100
  previewFrame?: string; // Base64 JPEG
  selectedForUprez: boolean;
}

export interface WanI2VParams {
  duration: number;      // 3, 5, 7, 10
  numInferenceSteps: number; // 20, 25, 30, 40
  guidance: number;      // 3.0 - 7.0
}

export interface QwenParams {
  seedCount: number;     // 1, 4, 8, 12, 16, 24
  size: string;          // "1280*720", "1024*1024", "720*1280", "512*512"
}

// Combination key for deduplication (matches engines/scripts pattern)
export const createComboKey = (imageUuid: string, actionPrompt: string): string => {
  // Simple hash - matches run_wan_i2v_batch.py BATCH_IDENTIFIER pattern
  const hash = Array.from(actionPrompt).reduce(
    (h, c) => ((h << 5) - h + c.charCodeAt(0)) | 0, 0
  ).toString(16).slice(-8);
  return `${imageUuid}:${hash}`;
};
```

**Step 2: Create Zustand store**

```typescript
// frontend/src/stores/studio.store.ts
import { create } from "zustand";
import type {
  StudioTab,
  StudioImage,
  StudioAction,
  StudioJob,
  QwenParams,
  WanI2VParams,
} from "@/types/studio.types";

interface StudioState {
  // Navigation
  activeTab: StudioTab;
  setActiveTab: (tab: StudioTab) => void;

  // Images
  imagePrompt: string;
  setImagePrompt: (prompt: string) => void;
  qwenParams: QwenParams;
  setQwenParams: (params: Partial<QwenParams>) => void;
  images: StudioImage[];
  addImage: (image: StudioImage) => void;
  updateImage: (uuid: string, updates: Partial<StudioImage>) => void;
  toggleImageSelected: (uuid: string) => void;
  removeImage: (uuid: string) => void;

  // Actions
  actions: StudioAction[];
  addAction: (action: StudioAction) => void;
  updateAction: (id: string, updates: Partial<StudioAction>) => void;
  removeAction: (id: string) => void;
  toggleActionEnabled: (id: string) => void;
  loadPresetPack: (actions: StudioAction[]) => void;

  // Generation
  wanParams: WanI2VParams;
  setWanParams: (params: Partial<WanI2VParams>) => void;
  outputPlaylistId: string | null;
  setOutputPlaylistId: (id: string | null) => void;

  // Combination exclusions (unchecked combos in Generate tab)
  excludedCombos: Set<string>; // Set of "imageUuid:actionId"
  toggleComboExcluded: (key: string) => void;

  // Jobs/Results
  jobs: StudioJob[];
  addJob: (job: StudioJob) => void;
  updateJob: (dreamUuid: string, updates: Partial<StudioJob>) => void;
  toggleJobUprez: (dreamUuid: string) => void;

  // Tab badge
  newCompletedCount: number;
  incrementNewCompleted: () => void;
  clearNewCompleted: () => void;

  // Reset
  resetSession: () => void;
}

const DEFAULT_QWEN_PARAMS: QwenParams = { seedCount: 8, size: "1280*720" };
const DEFAULT_WAN_PARAMS: WanI2VParams = { duration: 5, numInferenceSteps: 30, guidance: 5.0 };

export const useStudioStore = create<StudioState>((set, get) => ({
  // Navigation
  activeTab: "images",
  setActiveTab: (tab) => {
    if (tab === "results") set({ newCompletedCount: 0 });
    set({ activeTab: tab });
  },

  // Images
  imagePrompt: "",
  setImagePrompt: (prompt) => set({ imagePrompt: prompt }),
  qwenParams: DEFAULT_QWEN_PARAMS,
  setQwenParams: (params) => set((s) => ({ qwenParams: { ...s.qwenParams, ...params } })),
  images: [],
  addImage: (image) => set((s) => ({ images: [...s.images, image] })),
  updateImage: (uuid, updates) =>
    set((s) => ({
      images: s.images.map((img) => (img.uuid === uuid ? { ...img, ...updates } : img)),
    })),
  toggleImageSelected: (uuid) =>
    set((s) => ({
      images: s.images.map((img) =>
        img.uuid === uuid ? { ...img, selected: !img.selected } : img
      ),
    })),
  removeImage: (uuid) => set((s) => ({ images: s.images.filter((img) => img.uuid !== uuid) })),

  // Actions
  actions: [],
  addAction: (action) => set((s) => ({ actions: [...s.actions, action] })),
  updateAction: (id, updates) =>
    set((s) => ({
      actions: s.actions.map((a) => (a.id === id ? { ...a, ...updates } : a)),
    })),
  removeAction: (id) => set((s) => ({ actions: s.actions.filter((a) => a.id !== id) })),
  toggleActionEnabled: (id) =>
    set((s) => ({
      actions: s.actions.map((a) => (a.id === id ? { ...a, enabled: !a.enabled } : a)),
    })),
  loadPresetPack: (newActions) => set((s) => ({ actions: [...s.actions, ...newActions] })),

  // Generation
  wanParams: DEFAULT_WAN_PARAMS,
  setWanParams: (params) => set((s) => ({ wanParams: { ...s.wanParams, ...params } })),
  outputPlaylistId: null,
  setOutputPlaylistId: (id) => set({ outputPlaylistId: id }),

  // Combination exclusions
  excludedCombos: new Set(),
  toggleComboExcluded: (key) =>
    set((s) => {
      const next = new Set(s.excludedCombos);
      if (next.has(key)) next.delete(key);
      else next.add(key);
      return { excludedCombos: next };
    }),

  // Jobs
  jobs: [],
  addJob: (job) => set((s) => ({ jobs: [...s.jobs, job] })),
  updateJob: (dreamUuid, updates) =>
    set((s) => ({
      jobs: s.jobs.map((j) => (j.dreamUuid === dreamUuid ? { ...j, ...updates } : j)),
    })),
  toggleJobUprez: (dreamUuid) =>
    set((s) => ({
      jobs: s.jobs.map((j) =>
        j.dreamUuid === dreamUuid ? { ...j, selectedForUprez: !j.selectedForUprez } : j
      ),
    })),

  // Tab badge
  newCompletedCount: 0,
  incrementNewCompleted: () => set((s) => ({ newCompletedCount: s.newCompletedCount + 1 })),
  clearNewCompleted: () => set({ newCompletedCount: 0 }),

  // Reset
  resetSession: () =>
    set({
      activeTab: "images",
      imagePrompt: "",
      qwenParams: DEFAULT_QWEN_PARAMS,
      images: [],
      actions: [],
      wanParams: DEFAULT_WAN_PARAMS,
      outputPlaylistId: null,
      excludedCombos: new Set(),
      jobs: [],
      newCompletedCount: 0,
    }),
}));
```

**Step 3: Verify types compile**

Run: `cd frontend && pnpm run type-check`
Expected: No errors related to studio types/store

**Step 4: Commit**

```bash
git add frontend/src/types/studio.types.ts frontend/src/stores/studio.store.ts
git commit -m "feat(studio): add types and Zustand store for Studio page"
```

---

## Task 2: Route, Page Shell, and Tab Navigation

**Files:**
- Modify: `frontend/src/constants/routes.constants.ts` — add STUDIO route
- Modify: `frontend/src/routes/router.tsx` — register route
- Create: `frontend/src/components/pages/studio/studio.page.tsx`
- Create: `frontend/src/components/pages/studio/studio.page.styled.tsx`
- Create: `frontend/src/components/pages/studio/components/studio-tabs.tsx`
- Create: `frontend/src/components/pages/studio/components/studio-tabs.styled.tsx`

**Step 1: Add route constant**

In `frontend/src/constants/routes.constants.ts`, add:
```typescript
STUDIO: "/studio",
```
to the `ROUTES` object.

**Step 2: Create styled components for Studio page**

```typescript
// frontend/src/components/pages/studio/studio.page.styled.tsx
import styled from "styled-components";

export const StudioContainer = styled.div`
  max-width: 1200px;
  margin: 0 auto;
  padding: 1.5rem;
  min-height: calc(100vh - 80px);
`;

export const StudioHeader = styled.div`
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 1.5rem;
`;

export const StudioTitle = styled.h1`
  font-size: 1.5rem;
  font-weight: 600;
  color: ${(props) => props.theme.textPrimaryColor};
`;
```

**Step 3: Create tab navigation component**

```typescript
// frontend/src/components/pages/studio/components/studio-tabs.styled.tsx
import styled, { css } from "styled-components";

export const TabBar = styled.div`
  display: flex;
  gap: 0;
  border-bottom: 1px solid ${(props) => props.theme.colorBackgroundQuaternary};
  margin-bottom: 1.5rem;
`;

export const Tab = styled.button<{ $active: boolean; $badge?: number }>`
  padding: 0.75rem 1.25rem;
  background: none;
  border: none;
  border-bottom: 2px solid transparent;
  color: ${(props) => props.theme.textBodyColor};
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  position: relative;
  transition: color 0.15s, border-color 0.15s;

  ${(props) =>
    props.$active &&
    css`
      color: ${props.theme.textPrimaryColor};
      border-bottom-color: ${props.theme.colorPrimary};
    `}

  &:hover {
    color: ${(props) => props.theme.textPrimaryColor};
  }

  ${(props) =>
    props.$badge &&
    props.$badge > 0 &&
    css`
      &::after {
        content: "${props.$badge}";
        position: absolute;
        top: 0.25rem;
        right: 0.25rem;
        background: ${props.theme.colorPrimary};
        color: white;
        font-size: 0.625rem;
        border-radius: 50%;
        width: 1.125rem;
        height: 1.125rem;
        display: flex;
        align-items: center;
        justify-content: center;
      }
    `}
`;
```

```typescript
// frontend/src/components/pages/studio/components/studio-tabs.tsx
import React from "react";
import { useStudioStore } from "@/stores/studio.store";
import type { StudioTab } from "@/types/studio.types";
import { TabBar, Tab } from "./studio-tabs.styled";

const TABS: { key: StudioTab; label: string }[] = [
  { key: "images", label: "Images" },
  { key: "actions", label: "Actions" },
  { key: "generate", label: "Generate" },
  { key: "results", label: "Results" },
];

export const StudioTabs: React.FC = () => {
  const activeTab = useStudioStore((s) => s.activeTab);
  const setActiveTab = useStudioStore((s) => s.setActiveTab);
  const newCompletedCount = useStudioStore((s) => s.newCompletedCount);

  return (
    <TabBar>
      {TABS.map((tab) => (
        <Tab
          key={tab.key}
          $active={activeTab === tab.key}
          $badge={tab.key === "results" && activeTab !== "results" ? newCompletedCount : undefined}
          onClick={() => setActiveTab(tab.key)}
        >
          {tab.label}
        </Tab>
      ))}
    </TabBar>
  );
};
```

**Step 4: Create Studio page shell**

```typescript
// frontend/src/components/pages/studio/studio.page.tsx
import React from "react";
import { useStudioStore } from "@/stores/studio.store";
import { StudioTabs } from "./components/studio-tabs";
import { StudioContainer, StudioHeader, StudioTitle } from "./studio.page.styled";

export const StudioPage: React.FC = () => {
  const activeTab = useStudioStore((s) => s.activeTab);

  return (
    <StudioContainer>
      <StudioHeader>
        <StudioTitle>Studio</StudioTitle>
      </StudioHeader>
      <StudioTabs />
      {activeTab === "images" && <div>Images tab (coming soon)</div>}
      {activeTab === "actions" && <div>Actions tab (coming soon)</div>}
      {activeTab === "generate" && <div>Generate tab (coming soon)</div>}
      {activeTab === "results" && <div>Results tab (coming soon)</div>}
    </StudioContainer>
  );
};
```

**Step 5: Register route in router**

In `frontend/src/routes/router.tsx`, add the Studio route inside the protected routes for `CREATOR_GROUP`:

```typescript
{
  path: ROUTES.STUDIO,
  element: (
    <ProtectedRoute allowedRoles={[CREATOR_GROUP, ADMIN_GROUP]}>
      <StudioPage />
    </ProtectedRoute>
  ),
},
```

Import `StudioPage` from `@/components/pages/studio/studio.page`.

**Step 6: Verify**

Run: `cd frontend && pnpm run type-check`
Expected: No type errors

Run: `cd frontend && pnpm run dev`
Expected: Navigate to `/studio`, see "Studio" header with 4 tabs. Clicking tabs switches content.

**Step 7: Commit**

```bash
git add frontend/src/constants/routes.constants.ts frontend/src/routes/router.tsx \
  frontend/src/components/pages/studio/
git commit -m "feat(studio): add Studio page shell with tab navigation"
```

---

## Task 3: API Hook — Create Dream from Prompt

The existing `useCreateDream` uses file upload (multipart). The Studio needs a simpler mutation that creates a dream with just a prompt JSON (no file upload) — this is how the Python SDK's `create_dream_from_prompt` works.

**Files:**
- Create: `frontend/src/api/dream/mutation/useCreateDreamFromPrompt.ts`

**Step 1: Create the mutation hook**

```typescript
// frontend/src/api/dream/mutation/useCreateDreamFromPrompt.ts
import { useMutation } from "@tanstack/react-query";
import { axiosClient } from "@/client/axios.client";
import { getRequestHeaders, ContentType } from "@/client/headers";
import type { Dream } from "@/types/dream.types";

interface CreateDreamFromPromptParams {
  name: string;
  prompt: string; // JSON.stringify'd algorithm params
  description?: string;
}

interface CreateDreamResponse {
  success: boolean;
  data: { dream: Dream };
}

const createDreamFromPrompt = async (
  params: CreateDreamFromPromptParams
): Promise<CreateDreamResponse> => {
  const { data } = await axiosClient.post<CreateDreamResponse>("/v1/dream", params, {
    headers: getRequestHeaders({ contentType: ContentType.json }),
  });
  return data;
};

export const useCreateDreamFromPrompt = () => {
  return useMutation<CreateDreamResponse, Error, CreateDreamFromPromptParams>({
    mutationFn: createDreamFromPrompt,
  });
};
```

**Step 2: Verify types compile**

Run: `cd frontend && pnpm run type-check`
Expected: No errors

**Step 3: Commit**

```bash
git add frontend/src/api/dream/mutation/useCreateDreamFromPrompt.ts
git commit -m "feat(studio): add useCreateDreamFromPrompt API hook"
```

---

## Task 4: Images Tab — Generation Form

**Files:**
- Create: `frontend/src/components/pages/studio/components/images-tab.tsx`
- Create: `frontend/src/components/pages/studio/components/images-tab.styled.tsx`

**Step 1: Create styled components**

```typescript
// frontend/src/components/pages/studio/components/images-tab.styled.tsx
import styled from "styled-components";

export const GenerateSection = styled.div`
  border: 1px solid ${(props) => props.theme.colorBackgroundQuaternary};
  border-radius: 8px;
  padding: 1.25rem;
  margin-bottom: 1.5rem;
`;

export const SectionTitle = styled.h3`
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: ${(props) => props.theme.textBodyColor};
  margin-bottom: 1rem;
`;

export const PromptTextarea = styled.textarea`
  width: 100%;
  min-height: 80px;
  padding: 0.75rem;
  border: 1px solid ${(props) => props.theme.colorBackgroundQuaternary};
  border-radius: 6px;
  background: ${(props) => props.theme.colorBackgroundSecondary || "transparent"};
  color: ${(props) => props.theme.textPrimaryColor};
  font-family: inherit;
  font-size: 0.875rem;
  resize: vertical;
  margin-bottom: 1rem;

  &:focus {
    outline: none;
    border-color: ${(props) => props.theme.colorPrimary};
  }
`;

export const FormRow = styled.div`
  display: flex;
  align-items: center;
  gap: 1rem;
  flex-wrap: wrap;
`;

export const FormField = styled.div`
  display: flex;
  align-items: center;
  gap: 0.5rem;
`;

export const FieldLabel = styled.label`
  font-size: 0.8125rem;
  color: ${(props) => props.theme.textBodyColor};
`;

export const StyledSelect = styled.select`
  padding: 0.5rem 0.75rem;
  border: 1px solid ${(props) => props.theme.colorBackgroundQuaternary};
  border-radius: 6px;
  background: ${(props) => props.theme.colorBackgroundSecondary || "transparent"};
  color: ${(props) => props.theme.textPrimaryColor};
  font-size: 0.8125rem;
  cursor: pointer;
`;

export const GenerateButton = styled.button`
  padding: 0.625rem 1.25rem;
  background: ${(props) => props.theme.colorPrimary};
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  margin-left: auto;

  &:hover {
    filter: brightness(120%);
  }

  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
`;

export const ImageGrid = styled.div`
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  gap: 0.75rem;
`;

export const ImageCard = styled.div<{ $selected?: boolean }>`
  position: relative;
  border-radius: 8px;
  overflow: hidden;
  border: 2px solid ${(props) => (props.$selected ? props.theme.colorPrimary : "transparent")};
  cursor: pointer;
  aspect-ratio: 16 / 9;
  background: ${(props) => props.theme.colorBackgroundQuaternary};
`;

export const ImageThumbnail = styled.img`
  width: 100%;
  height: 100%;
  object-fit: cover;
`;

export const StarBadge = styled.button<{ $active?: boolean }>`
  position: absolute;
  top: 0.375rem;
  right: 0.375rem;
  background: rgba(0, 0, 0, 0.6);
  border: none;
  border-radius: 50%;
  width: 1.75rem;
  height: 1.75rem;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 0.875rem;
  color: ${(props) => (props.$active ? "#f5c542" : "#888")};
`;

export const ImageStatus = styled.div`
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 0.375rem 0.5rem;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  font-size: 0.6875rem;
  text-align: center;
`;

export const BottomRow = styled.div`
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 1rem;
`;

export const SelectionCount = styled.span`
  font-size: 0.8125rem;
  color: ${(props) => props.theme.textBodyColor};
`;

export const NavButton = styled.button`
  padding: 0.625rem 1.25rem;
  background: ${(props) => props.theme.colorPrimary};
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;

  &:hover {
    filter: brightness(120%);
  }
`;
```

**Step 2: Create Images tab component**

```typescript
// frontend/src/components/pages/studio/components/images-tab.tsx
import React, { useCallback } from "react";
import { v4 as uuidv4 } from "uuid";
import { useStudioStore } from "@/stores/studio.store";
import { useCreateDreamFromPrompt } from "@/api/dream/mutation/useCreateDreamFromPrompt";
import type { StudioImage } from "@/types/studio.types";
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
  BottomRow,
  SelectionCount,
  NavButton,
} from "./images-tab.styled";

const SEED_OPTIONS = [1, 4, 8, 12, 16, 24];
const SIZE_OPTIONS = ["1280*720", "1024*1024", "720*1280", "512*512"];

export const ImagesTab: React.FC = () => {
  const imagePrompt = useStudioStore((s) => s.imagePrompt);
  const setImagePrompt = useStudioStore((s) => s.setImagePrompt);
  const qwenParams = useStudioStore((s) => s.qwenParams);
  const setQwenParams = useStudioStore((s) => s.setQwenParams);
  const images = useStudioStore((s) => s.images);
  const addImage = useStudioStore((s) => s.addImage);
  const toggleImageSelected = useStudioStore((s) => s.toggleImageSelected);
  const setActiveTab = useStudioStore((s) => s.setActiveTab);
  const createDream = useCreateDreamFromPrompt();

  const selectedCount = images.filter((img) => img.selected).length;

  const handleGenerate = useCallback(() => {
    if (!imagePrompt.trim()) return;

    for (let i = 0; i < qwenParams.seedCount; i++) {
      const algoParams = {
        infinidream_algorithm: "qwen-image",
        prompt: imagePrompt,
        size: qwenParams.size,
        seed: -1, // Random seed for each
      };

      const tempId = uuidv4(); // Temporary ID until dream UUID is returned

      createDream.mutate(
        {
          name: `Qwen Image ${images.length + i + 1}`,
          prompt: JSON.stringify(algoParams),
          description: "Studio generated image",
        },
        {
          onSuccess: (response) => {
            const dream = response.data.dream;
            addImage({
              uuid: dream.uuid,
              url: dream.thumbnail || "",
              name: dream.name,
              status: dream.status as StudioImage["status"],
              selected: false,
            });
          },
        }
      );
    }
  }, [imagePrompt, qwenParams, images.length, createDream, addImage]);

  return (
    <>
      <GenerateSection>
        <SectionTitle>Generate New Images</SectionTitle>
        <PromptTextarea
          placeholder="Describe the image you want to generate..."
          value={imagePrompt}
          onChange={(e) => setImagePrompt(e.target.value)}
        />
        <FormRow>
          <FormField>
            <FieldLabel>Seeds:</FieldLabel>
            <StyledSelect
              value={qwenParams.seedCount}
              onChange={(e) => setQwenParams({ seedCount: Number(e.target.value) })}
            >
              {SEED_OPTIONS.map((n) => (
                <option key={n} value={n}>{n}</option>
              ))}
            </StyledSelect>
          </FormField>
          <FormField>
            <FieldLabel>Size:</FieldLabel>
            <StyledSelect
              value={qwenParams.size}
              onChange={(e) => setQwenParams({ size: e.target.value })}
            >
              {SIZE_OPTIONS.map((s) => (
                <option key={s} value={s}>{s.replace("*", "x")}</option>
              ))}
            </StyledSelect>
          </FormField>
          <GenerateButton
            onClick={handleGenerate}
            disabled={!imagePrompt.trim() || createDream.isLoading}
          >
            {createDream.isLoading ? "Generating..." : "Generate Images"}
          </GenerateButton>
        </FormRow>
      </GenerateSection>

      <GenerateSection>
        <SectionTitle>Image Library</SectionTitle>
        {images.length === 0 ? (
          <p style={{ color: "#888", fontSize: "0.875rem" }}>
            No images yet. Generate some above or add from a playlist.
          </p>
        ) : (
          <ImageGrid>
            {images.map((img) => (
              <ImageCard key={img.uuid} $selected={img.selected}>
                {img.status === "processed" && img.url ? (
                  <ImageThumbnail src={img.url} alt={img.name} />
                ) : (
                  <ImageStatus>
                    {img.status === "queue" && "Queued..."}
                    {img.status === "processing" && `${img.progress ?? 0}%`}
                    {img.status === "failed" && "Failed"}
                  </ImageStatus>
                )}
                <StarBadge
                  $active={img.selected}
                  onClick={(e) => {
                    e.stopPropagation();
                    toggleImageSelected(img.uuid);
                  }}
                >
                  {img.selected ? "\u2605" : "\u2606"}
                </StarBadge>
              </ImageCard>
            ))}
          </ImageGrid>
        )}
        <BottomRow>
          <SelectionCount>
            {selectedCount} selected for animation
          </SelectionCount>
          <NavButton onClick={() => setActiveTab("actions")}>
            Continue to Actions &rarr;
          </NavButton>
        </BottomRow>
      </GenerateSection>
    </>
  );
};
```

**Step 3: Wire into Studio page**

In `frontend/src/components/pages/studio/studio.page.tsx`, replace the placeholder:
```typescript
{activeTab === "images" && <ImagesTab />}
```

**Step 4: Verify**

Run: `cd frontend && pnpm run type-check`
Expected: No errors

Run: `cd frontend && pnpm run dev`
Expected: Images tab shows prompt textarea, seed/size selectors, generate button. Image library section shows empty state.

**Step 5: Commit**

```bash
git add frontend/src/components/pages/studio/components/images-tab.tsx \
  frontend/src/components/pages/studio/components/images-tab.styled.tsx \
  frontend/src/components/pages/studio/studio.page.tsx
git commit -m "feat(studio): add Images tab with Qwen generation form and image grid"
```

---

## Task 5: Socket.IO Integration for Image Progress Polling

Images are generated async. We need to poll dream status or subscribe to Socket.IO events to update the image grid as Qwen jobs complete.

**Files:**
- Create: `frontend/src/components/pages/studio/hooks/useStudioJobProgress.ts`

**Step 1: Create progress polling hook**

This hook subscribes to Socket.IO `job:progress` events and also polls dream status for images/jobs that are pending.

```typescript
// frontend/src/components/pages/studio/hooks/useStudioJobProgress.ts
import { useEffect, useRef, useCallback } from "react";
import { useSocket } from "@/hooks/useSocket";
import { axiosClient } from "@/client/axios.client";
import { useStudioStore } from "@/stores/studio.store";

const POLL_INTERVAL_MS = 10_000; // 10 seconds

export const useStudioJobProgress = () => {
  const { socket } = useSocket();
  const images = useStudioStore((s) => s.images);
  const updateImage = useStudioStore((s) => s.updateImage);
  const jobs = useStudioStore((s) => s.jobs);
  const updateJob = useStudioStore((s) => s.updateJob);
  const activeTab = useStudioStore((s) => s.activeTab);
  const incrementNewCompleted = useStudioStore((s) => s.incrementNewCompleted);
  const pollRef = useRef<ReturnType<typeof setInterval> | null>(null);

  // Subscribe to Socket.IO rooms for all pending dreams
  useEffect(() => {
    if (!socket) return;

    const pendingImageUuids = images
      .filter((img) => img.status === "queue" || img.status === "processing")
      .map((img) => img.uuid);

    const pendingJobUuids = jobs
      .filter((j) => j.status === "queue" || j.status === "processing")
      .map((j) => j.dreamUuid);

    const allUuids = [...pendingImageUuids, ...pendingJobUuids];

    allUuids.forEach((uuid) => {
      socket.emit("join_dream_room", uuid);
    });

    const handleProgress = (data: {
      dream_uuid: string;
      status?: string;
      progress?: number;
      preview_frame?: string;
    }) => {
      const { dream_uuid, progress, preview_frame } = data;

      // Check if this is an image
      const image = images.find((img) => img.uuid === dream_uuid);
      if (image) {
        updateImage(dream_uuid, {
          progress,
          previewFrame: preview_frame,
          status: data.status === "COMPLETED" ? "processed" : image.status,
        });
      }

      // Check if this is a job
      const job = jobs.find((j) => j.dreamUuid === dream_uuid);
      if (job) {
        const wasNotCompleted = job.status !== "processed";
        const isNowCompleted = data.status === "COMPLETED";

        updateJob(dream_uuid, {
          progress,
          previewFrame: preview_frame,
          status: isNowCompleted ? "processed" : job.status,
        });

        if (wasNotCompleted && isNowCompleted && activeTab !== "results") {
          incrementNewCompleted();
        }
      }
    };

    socket.on("job:progress", handleProgress);

    return () => {
      socket.off("job:progress", handleProgress);
      allUuids.forEach((uuid) => {
        socket.emit("leave_dream_room", uuid);
      });
    };
  }, [socket, images, jobs, updateImage, updateJob, activeTab, incrementNewCompleted]);

  // Polling fallback: check status of pending items periodically
  const pollPending = useCallback(async () => {
    const pendingImages = images.filter(
      (img) => img.status === "queue" || img.status === "processing"
    );
    const pendingJobs = jobs.filter(
      (j) => j.status === "queue" || j.status === "processing"
    );

    for (const img of pendingImages) {
      try {
        const { data } = await axiosClient.get(`/v1/dream/${img.uuid}`);
        const dream = data.data.dream;
        updateImage(img.uuid, {
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

        updateJob(job.dreamUuid, {
          status: dream.status,
        });

        if (wasNotCompleted && isNowCompleted && activeTab !== "results") {
          incrementNewCompleted();
        }
      } catch {
        // Silently skip failed polls
      }
    }
  }, [images, jobs, updateImage, updateJob, activeTab, incrementNewCompleted]);

  useEffect(() => {
    const hasPending =
      images.some((img) => img.status === "queue" || img.status === "processing") ||
      jobs.some((j) => j.status === "queue" || j.status === "processing");

    if (hasPending) {
      pollRef.current = setInterval(pollPending, POLL_INTERVAL_MS);
      return () => {
        if (pollRef.current) clearInterval(pollRef.current);
      };
    }
  }, [images, jobs, pollPending]);
};
```

**Step 2: Wire hook into Studio page**

In `studio.page.tsx`, add:
```typescript
import { useStudioJobProgress } from "./hooks/useStudioJobProgress";

// Inside component:
useStudioJobProgress();
```

**Step 3: Verify**

Run: `cd frontend && pnpm run type-check`
Expected: No errors

**Step 4: Commit**

```bash
git add frontend/src/components/pages/studio/hooks/useStudioJobProgress.ts \
  frontend/src/components/pages/studio/studio.page.tsx
git commit -m "feat(studio): add Socket.IO progress tracking and polling for Studio jobs"
```

---

## Task 6: Actions Tab

**Files:**
- Create: `frontend/src/components/pages/studio/components/actions-tab.tsx`
- Create: `frontend/src/components/pages/studio/components/actions-tab.styled.tsx`
- Create: `frontend/src/components/pages/studio/constants/action-presets.ts`

**Step 1: Create preset packs**

```typescript
// frontend/src/components/pages/studio/constants/action-presets.ts
import type { StudioAction } from "@/types/studio.types";
import { v4 as uuidv4 } from "uuid";

export interface PresetPack {
  name: string;
  actions: Omit<StudioAction, "id">[];
}

export const ACTION_PRESETS: PresetPack[] = [
  {
    name: "Camera Basics",
    actions: [
      { prompt: "slow zoom in, camera gently pushing forward", enabled: true },
      { prompt: "slow zoom out, camera pulling back to reveal", enabled: true },
      { prompt: "pan left to right, smooth motion", enabled: true },
      { prompt: "pan right to left, smooth motion", enabled: true },
      { prompt: "pan upward, revealing sky", enabled: true },
      { prompt: "pan downward, descending", enabled: true },
      { prompt: "push in, dramatic approach", enabled: true },
      { prompt: "pull out, widening perspective", enabled: true },
    ],
  },
  {
    name: "Cinematic",
    actions: [
      { prompt: "dolly forward, smooth cinematic approach", enabled: true },
      { prompt: "orbit around subject, 180 degrees, smooth motion", enabled: true },
      { prompt: "crane up, rising above the scene", enabled: true },
      { prompt: "tracking shot, following motion left to right", enabled: true },
      { prompt: "rack focus, shifting depth of field", enabled: true },
    ],
  },
  {
    name: "Organic",
    actions: [
      { prompt: "gentle breathing motion, subtle life", enabled: true },
      { prompt: "subtle sway, natural wind movement", enabled: true },
      { prompt: "floating drift, weightless motion", enabled: true },
      { prompt: "heartbeat pulse, rhythmic expansion", enabled: true },
    ],
  },
  {
    name: "Abstract",
    actions: [
      { prompt: "morph and flow, organic transformation", enabled: true },
      { prompt: "color shift, gradual hue rotation", enabled: true },
      { prompt: "kaleidoscope spin, symmetrical rotation", enabled: true },
      { prompt: "fractal zoom, infinite recursive detail", enabled: true },
    ],
  },
];

export const createActionsFromPreset = (preset: PresetPack): StudioAction[] =>
  preset.actions.map((a) => ({ ...a, id: uuidv4() }));
```

**Step 2: Create styled components**

```typescript
// frontend/src/components/pages/studio/components/actions-tab.styled.tsx
import styled from "styled-components";

export const ActionList = styled.div`
  display: flex;
  flex-direction: column;
  gap: 0;
  border: 1px solid ${(props) => props.theme.colorBackgroundQuaternary};
  border-radius: 8px;
  overflow: hidden;
  margin-bottom: 1rem;
`;

export const ActionRow = styled.div`
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.625rem 0.75rem;
  border-bottom: 1px solid ${(props) => props.theme.colorBackgroundQuaternary};

  &:last-child {
    border-bottom: none;
  }
`;

export const ActionCheckbox = styled.input.attrs({ type: "checkbox" })`
  width: 1.125rem;
  height: 1.125rem;
  cursor: pointer;
  flex-shrink: 0;
`;

export const ActionInput = styled.input`
  flex: 1;
  padding: 0.5rem;
  border: 1px solid transparent;
  border-radius: 4px;
  background: transparent;
  color: ${(props) => props.theme.textPrimaryColor};
  font-size: 0.875rem;

  &:focus {
    outline: none;
    border-color: ${(props) => props.theme.colorPrimary};
    background: ${(props) => props.theme.colorBackgroundSecondary || "transparent"};
  }
`;

export const DeleteButton = styled.button`
  background: none;
  border: none;
  color: ${(props) => props.theme.textBodyColor};
  cursor: pointer;
  padding: 0.25rem;
  font-size: 1rem;
  opacity: 0.6;

  &:hover {
    opacity: 1;
    color: ${(props) => props.theme.colorDanger || "#e55"};
  }
`;

export const SummaryBox = styled.div`
  border: 1px solid ${(props) => props.theme.colorBackgroundQuaternary};
  border-radius: 8px;
  padding: 1rem;
  text-align: center;
  font-size: 0.875rem;
  color: ${(props) => props.theme.textBodyColor};
  margin-top: 1rem;
`;

export const SummaryHighlight = styled.span`
  color: ${(props) => props.theme.textPrimaryColor};
  font-weight: 600;
`;
```

**Step 3: Create Actions tab component**

```typescript
// frontend/src/components/pages/studio/components/actions-tab.tsx
import React from "react";
import { v4 as uuidv4 } from "uuid";
import { useStudioStore } from "@/stores/studio.store";
import { ACTION_PRESETS, createActionsFromPreset } from "../constants/action-presets";
import {
  GenerateSection,
  SectionTitle,
  FormRow,
  StyledSelect,
  NavButton,
  BottomRow,
} from "./images-tab.styled";
import {
  ActionList,
  ActionRow,
  ActionCheckbox,
  ActionInput,
  DeleteButton,
  SummaryBox,
  SummaryHighlight,
} from "./actions-tab.styled";

export const ActionsTab: React.FC = () => {
  const actions = useStudioStore((s) => s.actions);
  const addAction = useStudioStore((s) => s.addAction);
  const updateAction = useStudioStore((s) => s.updateAction);
  const removeAction = useStudioStore((s) => s.removeAction);
  const toggleActionEnabled = useStudioStore((s) => s.toggleActionEnabled);
  const loadPresetPack = useStudioStore((s) => s.loadPresetPack);
  const images = useStudioStore((s) => s.images);
  const setActiveTab = useStudioStore((s) => s.setActiveTab);

  const selectedImageCount = images.filter((img) => img.selected).length;
  const enabledActionCount = actions.filter((a) => a.enabled).length;
  const totalVideos = selectedImageCount * enabledActionCount;

  const handleAddAction = () => {
    addAction({ id: uuidv4(), prompt: "", enabled: true });
  };

  const handleLoadPreset = (e: React.ChangeEvent<HTMLSelectElement>) => {
    const presetName = e.target.value;
    if (!presetName) return;
    const preset = ACTION_PRESETS.find((p) => p.name === presetName);
    if (preset) {
      loadPresetPack(createActionsFromPreset(preset));
    }
    e.target.value = ""; // Reset select
  };

  return (
    <>
      <GenerateSection>
        <SectionTitle>Action Prompts</SectionTitle>
        <p style={{ fontSize: "0.8125rem", color: "#888", marginBottom: "1rem" }}>
          These prompts describe camera motion or transformations. Each selected image
          will be animated with each enabled action.
        </p>

        {actions.length > 0 && (
          <ActionList>
            {actions.map((action) => (
              <ActionRow key={action.id}>
                <ActionCheckbox
                  checked={action.enabled}
                  onChange={() => toggleActionEnabled(action.id)}
                />
                <ActionInput
                  value={action.prompt}
                  placeholder="Describe motion or transformation..."
                  onChange={(e) => updateAction(action.id, { prompt: e.target.value })}
                />
                <DeleteButton onClick={() => removeAction(action.id)}>&times;</DeleteButton>
              </ActionRow>
            ))}
          </ActionList>
        )}

        <FormRow>
          <NavButton onClick={handleAddAction}>+ Add Action</NavButton>
          <StyledSelect onChange={handleLoadPreset} defaultValue="">
            <option value="" disabled>
              Load Preset Pack...
            </option>
            {ACTION_PRESETS.map((preset) => (
              <option key={preset.name} value={preset.name}>
                {preset.name}
              </option>
            ))}
          </StyledSelect>
        </FormRow>
      </GenerateSection>

      <SummaryBox>
        <SummaryHighlight>{selectedImageCount}</SummaryHighlight> images selected &times;{" "}
        <SummaryHighlight>{enabledActionCount}</SummaryHighlight> actions enabled ={" "}
        <SummaryHighlight>{totalVideos}</SummaryHighlight> videos
      </SummaryBox>

      <BottomRow>
        <NavButton onClick={() => setActiveTab("images")}>&larr; Back to Images</NavButton>
        <NavButton onClick={() => setActiveTab("generate")}>Continue to Generate &rarr;</NavButton>
      </BottomRow>
    </>
  );
};
```

**Step 4: Wire into Studio page**

In `studio.page.tsx`:
```typescript
{activeTab === "actions" && <ActionsTab />}
```

**Step 5: Verify**

Run: `cd frontend && pnpm run type-check`
Expected: No errors

Run: `cd frontend && pnpm run dev`
Expected: Actions tab shows action list, add button, preset dropdown, and summary calculation.

**Step 6: Commit**

```bash
git add frontend/src/components/pages/studio/components/actions-tab.tsx \
  frontend/src/components/pages/studio/components/actions-tab.styled.tsx \
  frontend/src/components/pages/studio/constants/action-presets.ts \
  frontend/src/components/pages/studio/studio.page.tsx
git commit -m "feat(studio): add Actions tab with action CRUD and preset packs"
```

---

## Task 7: Generate Tab — Combination Preview and Batch Submission

**Files:**
- Create: `frontend/src/components/pages/studio/components/generate-tab.tsx`
- Create: `frontend/src/components/pages/studio/components/generate-tab.styled.tsx`
- Create: `frontend/src/components/pages/studio/hooks/useBatchSubmit.ts`

**Step 1: Create styled components**

```typescript
// frontend/src/components/pages/studio/components/generate-tab.styled.tsx
import styled from "styled-components";

export const CombinationGrid = styled.div`
  overflow-x: auto;
  margin-bottom: 1.5rem;
`;

export const GridTable = styled.table`
  width: 100%;
  border-collapse: collapse;
  font-size: 0.8125rem;
`;

export const GridHeader = styled.th`
  padding: 0.5rem;
  text-align: center;
  font-weight: 500;
  color: ${(props) => props.theme.textBodyColor};
  border-bottom: 1px solid ${(props) => props.theme.colorBackgroundQuaternary};
  max-width: 150px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
`;

export const GridRowHeader = styled.td`
  padding: 0.5rem;
  font-weight: 500;
  color: ${(props) => props.theme.textBodyColor};
  border-right: 1px solid ${(props) => props.theme.colorBackgroundQuaternary};
  min-width: 100px;
`;

export const GridCell = styled.td<{ $excluded?: boolean }>`
  padding: 0.5rem;
  text-align: center;
  border-bottom: 1px solid ${(props) => props.theme.colorBackgroundQuaternary};
  opacity: ${(props) => (props.$excluded ? 0.4 : 1)};
  cursor: pointer;

  &:hover {
    background: ${(props) => props.theme.colorBackgroundQuaternary};
  }
`;

export const CellThumb = styled.img`
  width: 60px;
  height: 34px;
  object-fit: cover;
  border-radius: 4px;
`;

export const CellCheckbox = styled.input.attrs({ type: "checkbox" })`
  margin-top: 0.25rem;
`;

export const SettingsGrid = styled.div`
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 1rem;
`;

export const PlaylistRow = styled.div`
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-top: 1rem;
`;
```

**Step 2: Create batch submit hook**

```typescript
// frontend/src/components/pages/studio/hooks/useBatchSubmit.ts
import { useCallback, useState } from "react";
import { useStudioStore } from "@/stores/studio.store";
import { useCreateDreamFromPrompt } from "@/api/dream/mutation/useCreateDreamFromPrompt";
import { axiosClient } from "@/client/axios.client";
import { createComboKey } from "@/types/studio.types";

export const useBatchSubmit = () => {
  const images = useStudioStore((s) => s.images);
  const actions = useStudioStore((s) => s.actions);
  const wanParams = useStudioStore((s) => s.wanParams);
  const excludedCombos = useStudioStore((s) => s.excludedCombos);
  const outputPlaylistId = useStudioStore((s) => s.outputPlaylistId);
  const setOutputPlaylistId = useStudioStore((s) => s.setOutputPlaylistId);
  const addJob = useStudioStore((s) => s.addJob);
  const setActiveTab = useStudioStore((s) => s.setActiveTab);
  const jobs = useStudioStore((s) => s.jobs);

  const createDream = useCreateDreamFromPrompt();
  const [isSubmitting, setIsSubmitting] = useState(false);

  const getSelectedCombinations = useCallback(() => {
    const selectedImages = images.filter((img) => img.selected && img.status === "processed");
    const enabledActions = actions.filter((a) => a.enabled && a.prompt.trim());

    const combos: Array<{ image: typeof selectedImages[0]; action: typeof enabledActions[0] }> = [];

    for (const image of selectedImages) {
      for (const action of enabledActions) {
        const comboKey = `${image.uuid}:${action.id}`;
        if (!excludedCombos.has(comboKey)) {
          // Also skip if already submitted
          const existingJob = jobs.find(
            (j) => j.imageId === image.uuid && j.actionId === action.id
          );
          if (!existingJob) {
            combos.push({ image, action });
          }
        }
      }
    }

    return combos;
  }, [images, actions, excludedCombos, jobs]);

  const submit = useCallback(async () => {
    setIsSubmitting(true);

    try {
      // Ensure output playlist exists
      let playlistId = outputPlaylistId;
      if (!playlistId) {
        const now = new Date();
        const name = `Studio ${now.toISOString().slice(0, 16).replace("T", " ")}`;
        const { data } = await axiosClient.post("/v1/playlist", { name });
        playlistId = data.data.playlist.uuid;
        setOutputPlaylistId(playlistId);
      }

      const combos = getSelectedCombinations();

      for (const { image, action } of combos) {
        const batchIdentifier = createComboKey(image.uuid, action.prompt);
        const algoParams = {
          infinidream_algorithm: "wan-i2v",
          prompt: action.prompt,
          image: image.uuid,
          size: "1280*720",
          duration: wanParams.duration,
          num_inference_steps: wanParams.numInferenceSteps,
          guidance: wanParams.guidance,
        };

        try {
          const response = await createDream.mutateAsync({
            name: `${image.name} - ${action.prompt.slice(0, 40)}`,
            prompt: JSON.stringify(algoParams),
            description: `Studio batch. BATCH_IDENTIFIER:${batchIdentifier}`,
          });

          const dream = response.data.dream;

          addJob({
            imageId: image.uuid,
            actionId: action.id,
            dreamUuid: dream.uuid,
            status: dream.status as "queue" | "processing" | "processed" | "failed",
            selectedForUprez: false,
          });

          // Add to output playlist
          await axiosClient.put(`/v1/playlist/${playlistId}/add-item`, {
            type: "dream",
            uuid: dream.uuid,
          });
        } catch (err) {
          console.error("Failed to create dream for combo:", batchIdentifier, err);
        }
      }

      setActiveTab("results");
    } finally {
      setIsSubmitting(false);
    }
  }, [
    outputPlaylistId,
    setOutputPlaylistId,
    getSelectedCombinations,
    wanParams,
    createDream,
    addJob,
    setActiveTab,
  ]);

  return { submit, isSubmitting, getSelectedCombinations };
};
```

**Step 3: Create Generate tab component**

```typescript
// frontend/src/components/pages/studio/components/generate-tab.tsx
import React, { useEffect, useState } from "react";
import { useStudioStore } from "@/stores/studio.store";
import { useBatchSubmit } from "../hooks/useBatchSubmit";
import { axiosClient } from "@/client/axios.client";
import { useAuth } from "@/hooks/useAuth";
import {
  GenerateSection,
  SectionTitle,
  FormRow,
  FormField,
  FieldLabel,
  StyledSelect,
  NavButton,
  BottomRow,
} from "./images-tab.styled";
import {
  CombinationGrid,
  GridTable,
  GridHeader,
  GridRowHeader,
  GridCell,
  CellThumb,
  CellCheckbox,
  SettingsGrid,
  PlaylistRow,
} from "./generate-tab.styled";

const DURATION_OPTIONS = [3, 5, 7, 10];
const STEPS_OPTIONS = [20, 25, 30, 40];
const GUIDANCE_OPTIONS = [3.0, 4.0, 5.0, 6.0, 7.0];

export const GenerateTab: React.FC = () => {
  const images = useStudioStore((s) => s.images);
  const actions = useStudioStore((s) => s.actions);
  const wanParams = useStudioStore((s) => s.wanParams);
  const setWanParams = useStudioStore((s) => s.setWanParams);
  const excludedCombos = useStudioStore((s) => s.excludedCombos);
  const toggleComboExcluded = useStudioStore((s) => s.toggleComboExcluded);
  const outputPlaylistId = useStudioStore((s) => s.outputPlaylistId);
  const setOutputPlaylistId = useStudioStore((s) => s.setOutputPlaylistId);
  const setActiveTab = useStudioStore((s) => s.setActiveTab);
  const jobs = useStudioStore((s) => s.jobs);
  const { user } = useAuth();

  const { submit, isSubmitting, getSelectedCombinations } = useBatchSubmit();

  const [playlists, setPlaylists] = useState<Array<{ uuid: string; name: string }>>([]);

  const selectedImages = images.filter((img) => img.selected && img.status === "processed");
  const enabledActions = actions.filter((a) => a.enabled && a.prompt.trim());

  // Total combos minus excluded minus already-submitted
  const newCombos = getSelectedCombinations();
  const totalPossible = selectedImages.length * enabledActions.length;
  const excludedCount = totalPossible - newCombos.length - jobs.length;

  // Fetch user playlists for output selector
  useEffect(() => {
    if (!user?.uuid) return;
    axiosClient
      .get(`/v1/playlist?userUUID=${user.uuid}&take=50&skip=0`)
      .then(({ data }) => {
        setPlaylists(
          data.data.playlists.map((p: { uuid: string; name: string }) => ({
            uuid: p.uuid,
            name: p.name,
          }))
        );
      })
      .catch(() => {});
  }, [user?.uuid]);

  const handleCreatePlaylist = async () => {
    const now = new Date();
    const name = `Studio ${now.toISOString().slice(0, 16).replace("T", " ")}`;
    try {
      const { data } = await axiosClient.post("/v1/playlist", { name });
      const playlist = data.data.playlist;
      setOutputPlaylistId(playlist.uuid);
      setPlaylists((prev) => [{ uuid: playlist.uuid, name: playlist.name }, ...prev]);
    } catch (err) {
      console.error("Failed to create playlist:", err);
    }
  };

  return (
    <>
      <GenerateSection>
        <SectionTitle>Combination Preview</SectionTitle>
        <p style={{ fontSize: "0.8125rem", color: "#888", marginBottom: "1rem" }}>
          Review all image &times; action combinations before generating. Uncheck any you want to skip.
        </p>

        <CombinationGrid>
          <GridTable>
            <thead>
              <tr>
                <GridHeader />
                {enabledActions.map((action) => (
                  <GridHeader key={action.id} title={action.prompt}>
                    {action.prompt.slice(0, 20)}...
                  </GridHeader>
                ))}
              </tr>
            </thead>
            <tbody>
              {selectedImages.map((image) => (
                <tr key={image.uuid}>
                  <GridRowHeader>
                    {image.name}
                  </GridRowHeader>
                  {enabledActions.map((action) => {
                    const comboKey = `${image.uuid}:${action.id}`;
                    const excluded = excludedCombos.has(comboKey);
                    const existingJob = jobs.find(
                      (j) => j.imageId === image.uuid && j.actionId === action.id
                    );

                    return (
                      <GridCell
                        key={comboKey}
                        $excluded={excluded}
                        onClick={() => !existingJob && toggleComboExcluded(comboKey)}
                      >
                        {image.url && <CellThumb src={image.url} alt="" />}
                        <br />
                        {existingJob ? (
                          <span style={{ fontSize: "0.6875rem", color: "#6c6" }}>submitted</span>
                        ) : (
                          <CellCheckbox
                            checked={!excluded}
                            onChange={() => toggleComboExcluded(comboKey)}
                            onClick={(e) => e.stopPropagation()}
                          />
                        )}
                      </GridCell>
                    );
                  })}
                </tr>
              ))}
            </tbody>
          </GridTable>
        </CombinationGrid>

        <p style={{ fontSize: "0.8125rem", color: "#888", textAlign: "center" }}>
          {newCombos.length} of {totalPossible} combinations selected
          {jobs.length > 0 && ` (${jobs.length} already submitted)`}
        </p>
      </GenerateSection>

      <GenerateSection>
        <SectionTitle>Output Settings</SectionTitle>
        <SettingsGrid>
          <FormField>
            <FieldLabel>Duration:</FieldLabel>
            <StyledSelect
              value={wanParams.duration}
              onChange={(e) => setWanParams({ duration: Number(e.target.value) })}
            >
              {DURATION_OPTIONS.map((d) => (
                <option key={d} value={d}>{d} seconds</option>
              ))}
            </StyledSelect>
          </FormField>
          <FormField>
            <FieldLabel>Steps:</FieldLabel>
            <StyledSelect
              value={wanParams.numInferenceSteps}
              onChange={(e) => setWanParams({ numInferenceSteps: Number(e.target.value) })}
            >
              {STEPS_OPTIONS.map((s) => (
                <option key={s} value={s}>{s}</option>
              ))}
            </StyledSelect>
          </FormField>
          <FormField>
            <FieldLabel>Guidance:</FieldLabel>
            <StyledSelect
              value={wanParams.guidance}
              onChange={(e) => setWanParams({ guidance: Number(e.target.value) })}
            >
              {GUIDANCE_OPTIONS.map((g) => (
                <option key={g} value={g}>{g}</option>
              ))}
            </StyledSelect>
          </FormField>
        </SettingsGrid>

        <PlaylistRow>
          <FieldLabel>Output playlist:</FieldLabel>
          <StyledSelect
            value={outputPlaylistId || ""}
            onChange={(e) => setOutputPlaylistId(e.target.value || null)}
          >
            <option value="">Select playlist...</option>
            {playlists.map((p) => (
              <option key={p.uuid} value={p.uuid}>{p.name}</option>
            ))}
          </StyledSelect>
          <NavButton onClick={handleCreatePlaylist}>+ Create New</NavButton>
        </PlaylistRow>
      </GenerateSection>

      <BottomRow>
        <NavButton onClick={() => setActiveTab("actions")}>&larr; Back to Actions</NavButton>
        <NavButton
          onClick={submit}
          disabled={isSubmitting || newCombos.length === 0}
          style={{ background: newCombos.length === 0 ? "#555" : undefined }}
        >
          {isSubmitting ? "Submitting..." : `Generate ${newCombos.length} Videos \u2192`}
        </NavButton>
      </BottomRow>
    </>
  );
};
```

**Step 4: Wire into Studio page**

In `studio.page.tsx`:
```typescript
{activeTab === "generate" && <GenerateTab />}
```

**Step 5: Verify**

Run: `cd frontend && pnpm run type-check`
Expected: No errors

**Step 6: Commit**

```bash
git add frontend/src/components/pages/studio/components/generate-tab.tsx \
  frontend/src/components/pages/studio/components/generate-tab.styled.tsx \
  frontend/src/components/pages/studio/hooks/useBatchSubmit.ts \
  frontend/src/components/pages/studio/studio.page.tsx
git commit -m "feat(studio): add Generate tab with combination grid and batch submission"
```

---

## Task 8: Results Tab — Progress Matrix and Live Updates

**Files:**
- Create: `frontend/src/components/pages/studio/components/results-tab.tsx`
- Create: `frontend/src/components/pages/studio/components/results-tab.styled.tsx`

**Step 1: Create styled components**

```typescript
// frontend/src/components/pages/studio/components/results-tab.styled.tsx
import styled, { css } from "styled-components";

export const ProgressBar = styled.div`
  border: 1px solid ${(props) => props.theme.colorBackgroundQuaternary};
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.5rem;
`;

export const ProgressInfo = styled.div`
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.5rem;
  font-size: 0.875rem;
`;

export const ProgressTrack = styled.div`
  height: 8px;
  background: ${(props) => props.theme.colorBackgroundQuaternary};
  border-radius: 4px;
  overflow: hidden;
`;

export const ProgressFill = styled.div<{ $percent: number }>`
  height: 100%;
  width: ${(props) => props.$percent}%;
  background: ${(props) => props.theme.colorPrimary};
  border-radius: 4px;
  transition: width 0.3s ease;
`;

export const ResultCell = styled.td<{ $status?: string }>`
  padding: 0.5rem;
  text-align: center;
  border-bottom: 1px solid ${(props) => props.theme.colorBackgroundQuaternary};
  position: relative;
  min-width: 120px;

  ${(props) =>
    props.$status === "processed" &&
    css`
      cursor: pointer;
      &:hover {
        background: ${props.theme.colorBackgroundQuaternary};
      }
    `}
`;

export const ResultThumb = styled.div`
  position: relative;
  width: 100px;
  height: 56px;
  margin: 0 auto 0.25rem;
  border-radius: 4px;
  overflow: hidden;
  background: ${(props) => props.theme.colorBackgroundQuaternary};
`;

export const ResultThumbImg = styled.img`
  width: 100%;
  height: 100%;
  object-fit: cover;
`;

export const ResultCellStatus = styled.div<{ $color?: string }>`
  font-size: 0.6875rem;
  color: ${(props) => props.$color || props.theme.textBodyColor};
`;

export const UprezStarBadge = styled.button<{ $active?: boolean }>`
  position: absolute;
  top: 2px;
  right: 2px;
  background: rgba(0, 0, 0, 0.6);
  border: none;
  border-radius: 50%;
  width: 1.5rem;
  height: 1.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 0.75rem;
  color: ${(props) => (props.$active ? "#f5c542" : "#888")};
`;

export const ActionBar = styled.div`
  display: flex;
  gap: 0.75rem;
  margin-top: 1rem;
  flex-wrap: wrap;
`;

export const ActionButton = styled.button<{ $variant?: "primary" | "secondary" }>`
  padding: 0.5rem 1rem;
  border-radius: 6px;
  font-size: 0.8125rem;
  font-weight: 500;
  cursor: pointer;
  border: 1px solid ${(props) => props.theme.colorBackgroundQuaternary};

  ${(props) =>
    props.$variant === "primary"
      ? css`
          background: ${props.theme.colorPrimary};
          color: white;
          border-color: ${props.theme.colorPrimary};
        `
      : css`
          background: transparent;
          color: ${props.theme.textPrimaryColor};
        `}

  &:hover {
    filter: brightness(120%);
  }

  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
`;
```

**Step 2: Create Results tab component**

```typescript
// frontend/src/components/pages/studio/components/results-tab.tsx
import React, { useMemo, useCallback, useState } from "react";
import { useStudioStore } from "@/stores/studio.store";
import { useCreateDreamFromPrompt } from "@/api/dream/mutation/useCreateDreamFromPrompt";
import { axiosClient } from "@/client/axios.client";
import {
  GenerateSection,
  SectionTitle,
} from "./images-tab.styled";
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
  const createDream = useCreateDreamFromPrompt();

  const [isUprezzing, setIsUprezzing] = useState(false);

  const selectedImages = images.filter((img) => img.selected);
  const enabledActions = actions.filter((a) => a.enabled && a.prompt.trim());

  const completedCount = jobs.filter((j) => j.status === "processed").length;
  const totalCount = jobs.length;
  const failedCount = jobs.filter((j) => j.status === "failed").length;
  const progressPercent = totalCount > 0 ? Math.round((completedCount / totalCount) * 100) : 0;

  const uprezCount = jobs.filter((j) => j.selectedForUprez).length;

  const getJob = useCallback(
    (imageUuid: string, actionId: string) =>
      jobs.find((j) => j.imageId === imageUuid && j.actionId === actionId),
    [jobs]
  );

  const handleUprezSelected = useCallback(async () => {
    const toUprez = jobs.filter((j) => j.selectedForUprez && j.status === "processed");
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

          const dream = response.data.dream;

          addJob({
            imageId: job.imageId,
            actionId: `uprez-${job.actionId}`,
            dreamUuid: dream.uuid,
            status: dream.status as "queue",
            selectedForUprez: false,
          });

          if (outputPlaylistId) {
            await axiosClient.put(`/v1/playlist/${outputPlaylistId}/add-item`, {
              type: "dream",
              uuid: dream.uuid,
            });
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
    // Re-submit failed jobs with same params (simplified: just log for now)
    const failed = jobs.filter((j) => j.status === "failed");
    console.log("Retry failed jobs:", failed.length);
    // TODO: re-submit with same params from original job description
  }, [jobs]);

  return (
    <>
      <ProgressBar>
        <ProgressInfo>
          <span>
            {completedCount} of {totalCount} complete
          </span>
          <span>{progressPercent}%</span>
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
                  {enabledActions.map((action) => (
                    <GridHeader key={action.id} title={action.prompt}>
                      {action.prompt.slice(0, 20)}...
                    </GridHeader>
                  ))}
                </tr>
              </thead>
              <tbody>
                {selectedImages.map((image) => (
                  <tr key={image.uuid}>
                    <GridRowHeader>{image.name}</GridRowHeader>
                    {enabledActions.map((action) => {
                      const job = getJob(image.uuid, action.id);

                      if (!job) {
                        return (
                          <ResultCell key={`${image.uuid}:${action.id}`}>
                            <ResultCellStatus $color="#555">--</ResultCellStatus>
                          </ResultCell>
                        );
                      }

                      return (
                        <ResultCell
                          key={job.dreamUuid}
                          $status={job.status}
                          onClick={() => {
                            if (job.status === "processed") {
                              window.open(`/dream/${job.dreamUuid}`, "_blank");
                            }
                          }}
                        >
                          <ResultThumb>
                            {job.previewFrame ? (
                              <ResultThumbImg
                                src={`data:image/jpeg;base64,${job.previewFrame}`}
                                alt="preview"
                              />
                            ) : job.status === "processed" ? (
                              <ResultThumbImg
                                src={`${import.meta.env.VITE_BACKEND_URL}/dream/${job.dreamUuid}`}
                                alt="result"
                                onError={(e) => {
                                  (e.target as HTMLImageElement).style.display = "none";
                                }}
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
                            {job.status === "processing" && `${job.progress ?? 0}%`}
                            {job.status === "queue" && "queued"}
                            {job.status === "failed" && "failed"}
                          </ResultCellStatus>
                        </ResultCell>
                      );
                    })}
                  </tr>
                ))}
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
          <ActionButton onClick={handleRetryFailed}>
            Retry Failed ({failedCount})
          </ActionButton>
        )}
        {outputPlaylistId && (
          <ActionButton
            onClick={() => window.open(`/playlist/${outputPlaylistId}`, "_blank")}
          >
            View Playlist
          </ActionButton>
        )}
      </ActionBar>
    </>
  );
};
```

**Step 3: Wire into Studio page**

In `studio.page.tsx`:
```typescript
{activeTab === "results" && <ResultsTab />}
```

**Step 4: Verify**

Run: `cd frontend && pnpm run type-check`
Expected: No errors

**Step 5: Commit**

```bash
git add frontend/src/components/pages/studio/components/results-tab.tsx \
  frontend/src/components/pages/studio/components/results-tab.styled.tsx \
  frontend/src/components/pages/studio/studio.page.tsx
git commit -m "feat(studio): add Results tab with progress matrix, uprez, and live updates"
```

---

## Task 9: Add from Playlist Modal

**Files:**
- Create: `frontend/src/components/pages/studio/components/add-from-playlist-modal.tsx`
- Create: `frontend/src/components/pages/studio/components/add-from-playlist-modal.styled.tsx`

**Step 1: Create styled components**

```typescript
// frontend/src/components/pages/studio/components/add-from-playlist-modal.styled.tsx
import styled from "styled-components";

export const ModalOverlay = styled.div`
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
`;

export const ModalContent = styled.div`
  background: ${(props) => props.theme.colorBackgroundPrimary || "#1a1a1a"};
  border-radius: 12px;
  width: 90%;
  max-width: 700px;
  max-height: 80vh;
  display: flex;
  flex-direction: column;
`;

export const ModalHeader = styled.div`
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem 1.25rem;
  border-bottom: 1px solid ${(props) => props.theme.colorBackgroundQuaternary};
`;

export const ModalTitle = styled.h3`
  font-size: 1rem;
  font-weight: 600;
  color: ${(props) => props.theme.textPrimaryColor};
`;

export const CloseButton = styled.button`
  background: none;
  border: none;
  color: ${(props) => props.theme.textBodyColor};
  font-size: 1.25rem;
  cursor: pointer;
`;

export const ModalBody = styled.div`
  padding: 1.25rem;
  overflow-y: auto;
  flex: 1;
`;

export const ModalFooter = styled.div`
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem 1.25rem;
  border-top: 1px solid ${(props) => props.theme.colorBackgroundQuaternary};
`;

export const ImageSelectGrid = styled.div`
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
  gap: 0.5rem;
  margin-top: 1rem;
`;

export const ImageSelectCard = styled.div<{ $selected?: boolean }>`
  position: relative;
  border-radius: 6px;
  overflow: hidden;
  border: 2px solid ${(props) => (props.$selected ? props.theme.colorPrimary : "transparent")};
  cursor: pointer;
  aspect-ratio: 16 / 9;
  background: ${(props) => props.theme.colorBackgroundQuaternary};
`;
```

**Step 2: Create modal component**

```typescript
// frontend/src/components/pages/studio/components/add-from-playlist-modal.tsx
import React, { useEffect, useState, useCallback } from "react";
import { axiosClient } from "@/client/axios.client";
import { useAuth } from "@/hooks/useAuth";
import { useStudioStore } from "@/stores/studio.store";
import type { StudioImage } from "@/types/studio.types";
import { StyledSelect, NavButton } from "./images-tab.styled";
import { ImageThumbnail } from "./images-tab.styled";
import {
  ModalOverlay,
  ModalContent,
  ModalHeader,
  ModalTitle,
  CloseButton,
  ModalBody,
  ModalFooter,
  ImageSelectGrid,
  ImageSelectCard,
} from "./add-from-playlist-modal.styled";

interface Props {
  onClose: () => void;
}

interface PlaylistItem {
  dreamItem?: {
    uuid: string;
    name: string;
    thumbnail: string;
    mediaType?: string;
  };
}

export const AddFromPlaylistModal: React.FC<Props> = ({ onClose }) => {
  const { user } = useAuth();
  const addImage = useStudioStore((s) => s.addImage);
  const existingUuids = useStudioStore((s) => new Set(s.images.map((img) => img.uuid)));

  const [playlists, setPlaylists] = useState<Array<{ uuid: string; name: string }>>([]);
  const [selectedPlaylistId, setSelectedPlaylistId] = useState("");
  const [items, setItems] = useState<PlaylistItem[]>([]);
  const [selectedUuids, setSelectedUuids] = useState<Set<string>>(new Set());
  const [loading, setLoading] = useState(false);

  // Fetch user playlists
  useEffect(() => {
    if (!user?.uuid) return;
    axiosClient
      .get(`/v1/playlist?userUUID=${user.uuid}&take=50&skip=0`)
      .then(({ data }) => setPlaylists(data.data.playlists))
      .catch(() => {});
  }, [user?.uuid]);

  // Fetch playlist items when playlist selected
  useEffect(() => {
    if (!selectedPlaylistId) {
      setItems([]);
      return;
    }
    setLoading(true);
    axiosClient
      .get(`/v1/playlist/${selectedPlaylistId}/items?take=100&skip=0`)
      .then(({ data }) => {
        // Filter to only image-type dreams (or dreams that have thumbnails)
        setItems(
          data.data.items.filter(
            (item: PlaylistItem) => item.dreamItem?.thumbnail
          )
        );
      })
      .catch(() => {})
      .finally(() => setLoading(false));
  }, [selectedPlaylistId]);

  const toggleSelected = useCallback((uuid: string) => {
    setSelectedUuids((prev) => {
      const next = new Set(prev);
      if (next.has(uuid)) next.delete(uuid);
      else next.add(uuid);
      return next;
    });
  }, []);

  const handleAdd = useCallback(() => {
    items.forEach((item) => {
      const dream = item.dreamItem;
      if (!dream || !selectedUuids.has(dream.uuid)) return;
      if (existingUuids.has(dream.uuid)) return; // Skip duplicates

      const studioImage: StudioImage = {
        uuid: dream.uuid,
        url: dream.thumbnail,
        name: dream.name,
        status: "processed",
        selected: true, // Auto-select imported images
      };
      addImage(studioImage);
    });
    onClose();
  }, [items, selectedUuids, existingUuids, addImage, onClose]);

  return (
    <ModalOverlay onClick={onClose}>
      <ModalContent onClick={(e) => e.stopPropagation()}>
        <ModalHeader>
          <ModalTitle>Add Images from Playlist</ModalTitle>
          <CloseButton onClick={onClose}>&times;</CloseButton>
        </ModalHeader>

        <ModalBody>
          <StyledSelect
            value={selectedPlaylistId}
            onChange={(e) => {
              setSelectedPlaylistId(e.target.value);
              setSelectedUuids(new Set());
            }}
            style={{ width: "100%" }}
          >
            <option value="">Select a playlist...</option>
            {playlists.map((p) => (
              <option key={p.uuid} value={p.uuid}>
                {p.name}
              </option>
            ))}
          </StyledSelect>

          {loading && <p style={{ marginTop: "1rem", color: "#888" }}>Loading...</p>}

          {items.length > 0 && (
            <ImageSelectGrid>
              {items.map((item) => {
                const dream = item.dreamItem;
                if (!dream) return null;
                const isSelected = selectedUuids.has(dream.uuid);
                const alreadyAdded = existingUuids.has(dream.uuid);

                return (
                  <ImageSelectCard
                    key={dream.uuid}
                    $selected={isSelected}
                    onClick={() => !alreadyAdded && toggleSelected(dream.uuid)}
                    style={{ opacity: alreadyAdded ? 0.4 : 1 }}
                  >
                    <ImageThumbnail src={dream.thumbnail} alt={dream.name} />
                  </ImageSelectCard>
                );
              })}
            </ImageSelectGrid>
          )}
        </ModalBody>

        <ModalFooter>
          <span style={{ fontSize: "0.8125rem", color: "#888" }}>
            {selectedUuids.size} images selected
          </span>
          <div style={{ display: "flex", gap: "0.5rem" }}>
            <NavButton onClick={onClose} style={{ background: "transparent", border: "1px solid #555" }}>
              Cancel
            </NavButton>
            <NavButton onClick={handleAdd} disabled={selectedUuids.size === 0}>
              Add Selected
            </NavButton>
          </div>
        </ModalFooter>
      </ModalContent>
    </ModalOverlay>
  );
};
```

**Step 3: Add "Add from Playlist" button to Images tab**

In `images-tab.tsx`, add state and button:

```typescript
const [showPlaylistModal, setShowPlaylistModal] = useState(false);

// In the BottomRow area, add before the nav button:
<NavButton
  onClick={() => setShowPlaylistModal(true)}
  style={{ background: "transparent", border: "1px solid #555" }}
>
  + Add from Playlist
</NavButton>

// At the bottom of the component return:
{showPlaylistModal && <AddFromPlaylistModal onClose={() => setShowPlaylistModal(false)} />}
```

**Step 4: Verify**

Run: `cd frontend && pnpm run type-check`
Expected: No errors

**Step 5: Commit**

```bash
git add frontend/src/components/pages/studio/components/add-from-playlist-modal.tsx \
  frontend/src/components/pages/studio/components/add-from-playlist-modal.styled.tsx \
  frontend/src/components/pages/studio/components/images-tab.tsx
git commit -m "feat(studio): add playlist import modal for Images tab"
```

---

## Task 10: Add Studio Link to Navigation

**Files:**
- Modify: `frontend/src/components/shared/header/` (or equivalent nav component)

**Step 1: Find the navigation component**

Look at the header component in `frontend/src/components/shared/header/`. Add a "Studio" link in the navigation for creator/admin users, pointing to `/studio`.

The exact modification depends on the existing nav structure. Add it adjacent to the "Create" link.

**Step 2: Verify**

Run: `cd frontend && pnpm run dev`
Expected: "Studio" link appears in navigation for logged-in creators.

**Step 3: Commit**

```bash
git add frontend/src/components/shared/header/
git commit -m "feat(studio): add Studio link to navigation header"
```

---

## Task 11: Polish and Integration Testing

**Step 1: End-to-end manual testing**

Test the full workflow in the browser:
1. Navigate to `/studio`
2. Enter a prompt, generate 4 images
3. Wait for images to complete (verify polling/Socket.IO updates grid)
4. Star 2 images
5. Switch to Actions tab, add 2 custom actions
6. Switch to Generate tab, verify 2x2=4 grid, uncheck 1
7. Click Generate — verify 3 jobs submitted
8. Switch to Results tab — verify progress updates
9. Wait for completion, star 1 result
10. Click Uprez Selected — verify uprez job submitted
11. Click View Playlist — verify output playlist in new tab

**Step 2: Fix any issues found during testing**

Address bugs, styling issues, or edge cases.

**Step 3: Run lint and type-check**

```bash
cd frontend && pnpm run lint:fix && pnpm run type-check
```

**Step 4: Final commit**

```bash
git add -A
git commit -m "feat(studio): polish and integration fixes for Studio MVP"
```

---

## Summary

| Task | Description | New Files |
|------|-------------|-----------|
| 1 | Types + Zustand store | `studio.types.ts`, `studio.store.ts` |
| 2 | Route + page shell + tabs | `studio.page.tsx`, `studio-tabs.tsx` + styled |
| 3 | API hook: createDreamFromPrompt | `useCreateDreamFromPrompt.ts` |
| 4 | Images tab: gen form + grid | `images-tab.tsx` + styled |
| 5 | Socket.IO progress hook | `useStudioJobProgress.ts` |
| 6 | Actions tab: CRUD + presets | `actions-tab.tsx` + styled, `action-presets.ts` |
| 7 | Generate tab: combo grid + batch | `generate-tab.tsx` + styled, `useBatchSubmit.ts` |
| 8 | Results tab: progress matrix | `results-tab.tsx` + styled |
| 9 | Add from Playlist modal | `add-from-playlist-modal.tsx` + styled |
| 10 | Navigation link | Modify existing header |
| 11 | Polish + integration test | Fixes only |

**Total new files:** ~18 files (components, styled, hooks, types, constants)
**Backend changes:** None (Phase 1 uses existing APIs)
