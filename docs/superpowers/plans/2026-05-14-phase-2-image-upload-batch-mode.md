# Phase 2: Image Upload (Batch Mode) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add image upload (button + drag-and-drop) to batch mode's Images tab, extract the shared upload utility from flow mode, fix the keyframe UUID bug in video generation, and enable studio-wide drag-and-drop.

**Architecture:** Extract the inline multipart upload from `flow-builder.tsx` into a shared `uploadKeyframeImage` utility. Wire it into `images-tab.tsx` with an Upload button and `useFileDropUpload` hook. Fix `buildVideoAlgoParams` callers to pass image URLs (not keyframe UUIDs) since the worker resolves UUIDs via the dream endpoint which doesn't handle keyframes. Add studio-wide drop handler at `studio.page.tsx`.

**Tech Stack:** React 18, TypeScript, Zustand, styled-components, Axios, Vitest

**Spec:** `docs/superpowers/specs/2026-05-14-phase-2-image-upload-batch-mode-design.md`

**Base branch:** `feat/phase-0-keyframe-strip` (in the `frontend` repo at `/Users/maxcarlsonold/edream/frontend`)

---

### Task 1: Extract Shared Upload Utility

**Files:**
- Create: `frontend/src/components/pages/studio/utils/upload-keyframe-image.ts`
- Create: `frontend/src/components/pages/studio/utils/__tests__/upload-keyframe-image.test.ts`

- [ ] **Step 1: Write the test**

```typescript
// frontend/src/components/pages/studio/utils/__tests__/upload-keyframe-image.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";

// Mock axiosClient before importing the module
vi.mock("@/client/axios.client", () => ({
  axiosClient: {
    post: vi.fn(),
    get: vi.fn(),
  },
}));

vi.mock("@/constants/auth.constants", () => ({
  getRequestHeaders: vi.fn(() => ({ Authorization: "Bearer test" })),
  ContentType: { json: "application/json" },
}));

const { axiosClient } = await import("@/client/axios.client");
const { uploadKeyframeImage } = await import("../upload-keyframe-image");

const mockPost = vi.mocked(axiosClient.post);
const mockGet = vi.mocked(axiosClient.get);

describe("uploadKeyframeImage", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    globalThis.fetch = vi.fn();
  });

  it("returns keyframeUuid, imageUrl, and name on success", async () => {
    // Step 1: POST /v1/keyframe
    mockPost.mockResolvedValueOnce({
      data: { data: { keyframe: { uuid: "kf-123" } } },
    });
    // Step 2: POST /v1/keyframe/kf-123/image/init
    mockPost.mockResolvedValueOnce({
      data: { data: { uploadId: "up-1", urls: ["https://s3.example.com/part1"] } },
    });
    // Step 4: POST /v1/keyframe/kf-123/image/complete
    mockPost.mockResolvedValueOnce({ data: {} });
    // Step 5: GET /v1/keyframe/kf-123
    mockGet.mockResolvedValueOnce({
      data: {
        data: {
          keyframe: {
            uuid: "kf-123",
            image: "https://cdn.example.com/kf-123.jpg",
            name: "photo",
          },
        },
      },
    });

    // Step 3: PUT chunk to presigned URL
    vi.mocked(globalThis.fetch).mockResolvedValueOnce(
      new Response(null, {
        status: 200,
        headers: { ETag: '"abc123"' },
      }),
    );

    const file = new File(["pixels"], "photo.jpg", { type: "image/jpeg" });
    const result = await uploadKeyframeImage(file);

    expect(result).toEqual({
      keyframeUuid: "kf-123",
      imageUrl: "https://cdn.example.com/kf-123.jpg",
      name: "photo",
    });

    // Verify API calls
    expect(mockPost).toHaveBeenCalledTimes(3);
    expect(mockGet).toHaveBeenCalledTimes(1);
    expect(mockPost.mock.calls[0][0]).toBe("/v1/keyframe");
    expect(mockPost.mock.calls[1][0]).toBe("/v1/keyframe/kf-123/image/init");
    expect(mockPost.mock.calls[2][0]).toBe("/v1/keyframe/kf-123/image/complete");
    expect(mockGet.mock.calls[0][0]).toBe("/v1/keyframe/kf-123");
  });

  it("throws on API failure", async () => {
    mockPost.mockRejectedValueOnce(new Error("Network error"));

    const file = new File(["pixels"], "photo.jpg", { type: "image/jpeg" });
    await expect(uploadKeyframeImage(file)).rejects.toThrow("Network error");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm vitest run src/components/pages/studio/utils/__tests__/upload-keyframe-image.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement the shared utility**

```typescript
// frontend/src/components/pages/studio/utils/upload-keyframe-image.ts
import { axiosClient } from "@/client/axios.client";
import { getRequestHeaders, ContentType } from "@/constants/auth.constants";

export interface UploadResult {
  keyframeUuid: string;
  imageUrl: string;
  name: string;
}

const CHUNK_SIZE = 5 * 1024 * 1024; // 5MB

export async function uploadKeyframeImage(file: File): Promise<UploadResult> {
  const headers = getRequestHeaders({ contentType: ContentType.json });
  const extension = file.name.split(".").pop() ?? "jpg";
  const name = file.name.replace(/\.[^.]+$/, "");

  // Step 1: Create keyframe
  const createRes = await axiosClient.post(
    "/v1/keyframe",
    { name },
    { headers },
  );
  const keyframeUuid = createRes.data.data.keyframe.uuid;

  // Step 2: Init multipart upload
  const initRes = await axiosClient.post(
    `/v1/keyframe/${keyframeUuid}/image/init`,
    { extension },
    { headers },
  );
  const { uploadId, urls } = initRes.data.data;

  // Step 3: Upload chunks to presigned URLs
  const parts: { ETag: string; PartNumber: number }[] = [];
  for (let i = 0; i < urls.length; i++) {
    const start = i * CHUNK_SIZE;
    const end = Math.min(start + CHUNK_SIZE, file.size);
    const chunk = file.slice(start, end);
    const uploadRes = await fetch(urls[i], {
      method: "PUT",
      body: chunk,
    });
    parts.push({
      ETag: uploadRes.headers.get("ETag") ?? "",
      PartNumber: i + 1,
    });
  }

  // Step 4: Complete multipart upload
  await axiosClient.post(
    `/v1/keyframe/${keyframeUuid}/image/complete`,
    { extension, parts, uploadId },
    { headers },
  );

  // Step 5: Fetch finalized keyframe
  const kfRes = await axiosClient.get(`/v1/keyframe/${keyframeUuid}`, {
    headers,
  });
  const kfData = kfRes.data.data.keyframe;

  return {
    keyframeUuid: kfData.uuid,
    imageUrl: kfData.image,
    name: kfData.name,
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm vitest run src/components/pages/studio/utils/__tests__/upload-keyframe-image.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/utils/upload-keyframe-image.ts src/components/pages/studio/utils/__tests__/upload-keyframe-image.test.ts
git commit -m "feat(studio): extract shared uploadKeyframeImage utility"
```

---

### Task 2: Refactor FlowBuilder to Use Shared Utility

**Files:**
- Modify: `frontend/src/components/pages/studio/components/flow-builder.tsx`

- [ ] **Step 1: Replace inline upload with shared utility**

In `flow-builder.tsx`, replace the `uploadFiles` callback (lines 69-128) with a call to the shared utility:

```typescript
// Replace the existing import block — add uploadKeyframeImage
import { uploadKeyframeImage } from "@/components/pages/studio/utils/upload-keyframe-image";

// Replace the uploadFiles callback entirely:
const uploadFiles = useCallback(
  async (files: File[]) => {
    for (const file of files) {
      try {
        const result = await uploadKeyframeImage(file);
        addKeyframe({
          id: uuidv4(),
          keyframeUuid: result.keyframeUuid,
          imageUrl: result.imageUrl,
          name: result.name,
        });
      } catch (err) {
        console.error("Failed to upload keyframe image:", err);
      }
    }
  },
  [addKeyframe],
);
```

Also remove the now-unused imports: `axiosClient`, `ContentType`, `getRequestHeaders` (these are no longer used directly in this file since the upload utility handles them).

- [ ] **Step 2: Run type-check**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm type-check`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/components/flow-builder.tsx
git commit -m "refactor(flow): use shared uploadKeyframeImage utility in FlowBuilder"
```

---

### Task 3: Fix Keyframe UUID Bug in Flow Generation

**Files:**
- Modify: `frontend/src/components/pages/studio/hooks/useFlowGeneration.ts`

**Context:** `useFlowGeneration` passes `fromKf.keyframeUuid` to `buildVideoAlgoParams`, which puts it in the `image` field. The worker resolves UUIDs via `GET /dream/{uuid}` — keyframe UUIDs 404 on this endpoint. Fix: pass `fromKf.imageUrl` instead (the worker accepts URLs directly).

- [ ] **Step 1: Fix the imageUuid parameter**

In `useFlowGeneration.ts`, change the `buildVideoAlgoParams` call (line 71-79):

```typescript
// Change this line:
      imageUuid: fromKf.keyframeUuid,
// To:
      imageUuid: fromKf.imageUrl,
```

- [ ] **Step 2: Run type-check**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm type-check`
Expected: No errors

- [ ] **Step 3: Run all tests**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm vitest run`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/hooks/useFlowGeneration.ts
git commit -m "fix(flow): pass imageUrl instead of keyframeUuid to buildVideoAlgoParams

The worker resolves image references via GET /dream/{uuid}, which only
handles dream UUIDs. Keyframe UUIDs 404 on this endpoint. The worker
also accepts direct URLs, so pass the CDN image URL instead."
```

---

### Task 4: Fix Keyframe UUID Bug in Batch Submit

**Files:**
- Modify: `frontend/src/components/pages/studio/hooks/useBatchSubmit.ts`

**Context:** Same bug as Task 3, but in batch mode. `useBatchSubmit` passes `image.uuid` to `buildVideoAlgoParams`. For uploaded images, this is a keyframe UUID which will 404 on the worker's dream endpoint. Fix: pass `image.url` when it's a direct HTTP URL (uploaded images), fall back to `image.uuid` for generated images (dream UUIDs that the worker can resolve).

- [ ] **Step 1: Fix the imageUuid parameter**

In `useBatchSubmit.ts`, change the `buildVideoAlgoParams` call (around line 92-95):

```typescript
// Change this line:
              imageUuid: image.uuid,
// To:
              imageUuid: image.url?.startsWith("http") ? image.url : image.uuid,
```

This passes the CDN image URL for uploaded images (whose `url` field is a direct HTTP URL) and the dream UUID for generated images (whose `url` field is empty or a relative path resolved by `PresignedImage`).

- [ ] **Step 2: Run type-check**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm type-check`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/hooks/useBatchSubmit.ts
git commit -m "fix(studio): pass image URL instead of keyframe UUID in batch submit

Same fix as flow mode — the worker resolves UUIDs via GET /dream/{uuid}
which doesn't handle keyframe UUIDs. Pass the direct CDN URL for uploaded
images so the worker can fetch the image directly."
```

---

### Task 5: Add Upload to Images tab

**Files:**
- Modify: `frontend/src/components/pages/studio/components/images-tab.tsx`
- Modify: `frontend/src/components/pages/studio/components/images-tab.styled.tsx`

- [ ] **Step 1: Add drag-over styled variant to GenerateSection**

In `images-tab.styled.tsx`, add a new styled component for the droppable container:

```typescript
// Add after the existing GenerateSection
export const ImagesTabContainer = styled.div<{ $dragOver?: boolean }>`
  transition: border-color 0.2s;
  border: 2px solid transparent;
  border-radius: 10px;
  padding: 2px;

  ${(props) =>
    props.$dragOver &&
    `
    border-color: ${props.theme.colorPrimary};
  `}
`;

// Add after LightboxImage — same styling but for plain <img> (uploaded images)
export const LightboxUploadedImage = styled.img`
  max-width: 90vw;
  max-height: 90vh;
  object-fit: contain;
  border-radius: 8px;
`;
```

- [ ] **Step 2: Add upload functionality to ImagesTab**

In `images-tab.tsx`, add imports and upload handling:

```typescript
// Update the existing React import to add useRef:
import React, { useCallback, useMemo, useRef, useState } from "react";

// Add new imports:
import { v4 as uuidv4 } from "uuid";
import { useFileDropUpload } from "../hooks/useFileDropUpload";
import { uploadKeyframeImage } from "@/components/pages/studio/utils/upload-keyframe-image";
// Add ImagesTabContainer and LightboxUploadedImage to the styled import:
import { ImagesTabContainer, LightboxUploadedImage } from "./images-tab.styled";

// Inside ImagesTab component, after existing store selectors:
const updateImage = useStudioStore((s) => s.updateImage);
const fileInputRef = useRef<HTMLInputElement>(null);

const handleUploadFiles = useCallback(
  async (files: File[]) => {
    for (const file of files) {
      // Add placeholder with local blob URL for immediate feedback.
      // Keep the placeholder UUID stable — never swap it. The rendering
      // and batch submit logic branch on img.url.startsWith("http") to
      // distinguish uploaded images from generated ones.
      const placeholderUuid = uuidv4();
      const blobUrl = URL.createObjectURL(file);
      addImage({
        uuid: placeholderUuid,
        url: blobUrl,
        name: file.name.replace(/\.[^.]+$/, ""),
        status: "processing",
        selected: false,
      });

      try {
        const result = await uploadKeyframeImage(file);
        updateImage(placeholderUuid, {
          url: result.imageUrl,
          status: "processed",
          name: result.name,
        });
      } catch (err) {
        console.error("Failed to upload image:", err);
        updateImage(placeholderUuid, { status: "failed" });
      } finally {
        URL.revokeObjectURL(blobUrl);
      }
    }
  },
  [addImage, updateImage],
);

const handleFileSelected = useCallback(
  async (e: React.ChangeEvent<HTMLInputElement>) => {
    const files = e.target.files;
    if (!files) return;
    await handleUploadFiles(Array.from(files));
    if (fileInputRef.current) fileInputRef.current.value = "";
  },
  [handleUploadFiles],
);

const { isDragOver, dropHandlers } = useFileDropUpload({
  accept: ["image/jpeg", "image/png", "image/webp"],
  onFiles: handleUploadFiles,
});
```

- [ ] **Step 3: Update the JSX**

Wrap the entire component return in `ImagesTabContainer` with drop handlers. Add Upload button next to Generate. Add hidden file input. Update image rendering to conditionally use `<img>` for uploaded images. Update empty state text.

Replace the entire return block:

```tsx
return (
  <ImagesTabContainer $dragOver={isDragOver} {...dropHandlers}>
    <GenerateSection>
      <SectionTitle>Generate New Images</SectionTitle>
      <PromptTextarea
        placeholder="Describe the image you want to generate..."
        value={imagePrompt}
        onChange={(e) => setImagePrompt(e.target.value)}
      />
      <FormRow>
        <FormField>
          <FieldLabel>Model:</FieldLabel>
          <StyledSelect
            value={imageGenParams.model}
            onChange={(e) => {
              const newModel = e.target.value as ImageModel;
              setImageGenParams({
                model: newModel,
                size: clampSizeToModel(imageGenParams.size, newModel),
              });
            }}
          >
            {IMAGE_MODELS.map((m) => (
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
        <SecondaryNavButton onClick={() => fileInputRef.current?.click()}>
          Upload
        </SecondaryNavButton>
        <GenerateButton
          onClick={handleGenerate}
          disabled={!imagePrompt.trim() || isGenerating}
        >
          {isGenerating ? "Generating..." : "Generate Images"}
        </GenerateButton>
      </FormRow>
    </GenerateSection>

    <GenerateSection>
      <SectionTitle>Image Library</SectionTitle>
      {images.length === 0 ? (
        <EmptyStateText>
          No images yet. Generate some above, upload, or add from a
          playlist.
        </EmptyStateText>
      ) : (
        <ImageGrid>
          {images.map((img) => (
            <ImageCard key={img.uuid} $selected={img.selected}>
              {img.status === "processed" ? (
                img.url.startsWith("http") ? (
                  <ImageThumbnail
                    src={img.url}
                    alt={img.name}
                    onClick={() => setExpandedImageUuid(img.uuid)}
                    style={{ cursor: "zoom-in" }}
                  />
                ) : (
                  <ImageThumbnail
                    as={PresignedImage}
                    dreamUuid={img.uuid}
                    alt={img.name}
                    onClick={() => setExpandedImageUuid(img.uuid)}
                    style={{ cursor: "zoom-in" }}
                  />
                )
              ) : img.status === "processing" && img.url ? (
                <ImageThumbnail
                  src={img.url}
                  alt={img.name}
                  style={{ opacity: 0.5 }}
                />
              ) : (
                <ImageStatus>
                  {img.status === "queue" && "Queued..."}
                  {img.status === "processing" && `${img.progress ?? 0}%`}
                  {img.status === "failed" && "Failed"}
                </ImageStatus>
              )}
              {img.seed != null && <SeedLabel>#{img.seed}</SeedLabel>}
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
        <ButtonRow>
          <SelectionCount>
            {selectedCount} selected for animation
          </SelectionCount>
          {processedImages.length > 0 && (
            <SecondaryNavButton
              onClick={
                allProcessedSelected ? deselectAllImages : selectAllImages
              }
            >
              {allProcessedSelected ? "Deselect All" : "Select All"}
            </SecondaryNavButton>
          )}
        </ButtonRow>
        <ButtonRow>
          <SecondaryNavButton onClick={() => setShowPlaylistModal(true)}>
            + Add from Playlist
          </SecondaryNavButton>
          <NavButton onClick={() => setActiveTab("actions")}>
            Continue to Actions &rarr;
          </NavButton>
        </ButtonRow>
      </BottomRow>
    </GenerateSection>

    <input
      ref={fileInputRef}
      type="file"
      accept="image/jpeg,image/png,image/webp"
      multiple
      style={{ display: "none" }}
      onChange={handleFileSelected}
    />

    {showPlaylistModal && (
      <AddFromPlaylistModal onClose={() => setShowPlaylistModal(false)} />
    )}

    {expandedImageUuid && (
      <LightboxOverlay onClick={() => setExpandedImageUuid(null)}>
        {(() => {
          const img = images.find((i) => i.uuid === expandedImageUuid);
          return img?.url.startsWith("http") ? (
            <LightboxUploadedImage src={img.url} alt="Expanded" />
          ) : (
            <LightboxImage dreamUuid={expandedImageUuid} alt="Expanded" />
          );
        })()}
      </LightboxOverlay>
    )}
  </ImagesTabContainer>
);
```

- [ ] **Step 4: Run type-check**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm type-check`
Expected: No errors

- [ ] **Step 5: Run all tests**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm vitest run`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/components/images-tab.tsx src/components/pages/studio/components/images-tab.styled.tsx
git commit -m "feat(studio): add image upload button and drag-and-drop to batch mode Images tab"
```

---

### Task 6: Add Studio-Wide Drag-and-Drop

**Files:**
- Modify: `frontend/src/components/pages/studio/studio.page.tsx`
- Modify: `frontend/src/components/pages/studio/studio.page.styled.tsx`

**Context:** The roadmap requires drag-and-drop to work regardless of active tab. Wire `useFileDropUpload` at the studio page level. In batch mode, dropped files become `StudioImage` entries. In flow mode, the existing `FlowBuilder` drop handler already handles it.

- [ ] **Step 1: Add drag-over style to StudioContainer**

In `studio.page.styled.tsx`, update `StudioContainer`:

```typescript
export const StudioContainer = styled.div<{ $dragOver?: boolean }>`
  max-width: 1200px;
  margin: 0 auto;
  padding: 1.5rem;
  min-height: calc(100vh - 80px);
  transition: outline-color 0.2s;
  outline: 2px solid transparent;
  outline-offset: -2px;
  border-radius: 12px;

  ${(props) =>
    props.$dragOver &&
    `
    outline-color: ${props.theme.colorPrimary};
  `}
`;
```

- [ ] **Step 2: Wire drop handler in StudioPage**

In `studio.page.tsx`, add the upload handler:

```typescript
// Update the existing React import to add useCallback:
import React, { lazy, Suspense, useCallback } from "react";

// Add new imports:
import { v4 as uuidv4 } from "uuid";
import { useFileDropUpload } from "./hooks/useFileDropUpload";
import { uploadKeyframeImage } from "./utils/upload-keyframe-image";
import { useFlowStore } from "@/stores/flow.store";

// Inside StudioPage component, after existing hooks:
const addImage = useStudioStore((s) => s.addImage);
const updateImage = useStudioStore((s) => s.updateImage);
const addKeyframe = useFlowStore((s) => s.addKeyframe);

const handleStudioDrop = useCallback(
  async (files: File[]) => {
    const currentMode = useStudioModeStore.getState().mode;

    for (const file of files) {
      if (currentMode === "batch") {
        // Same pattern as images-tab: stable placeholder UUID, no swap
        const placeholderUuid = uuidv4();
        const blobUrl = URL.createObjectURL(file);
        addImage({
          uuid: placeholderUuid,
          url: blobUrl,
          name: file.name.replace(/\.[^.]+$/, ""),
          status: "processing",
          selected: false,
        });

        try {
          const result = await uploadKeyframeImage(file);
          updateImage(placeholderUuid, {
            url: result.imageUrl,
            status: "processed",
            name: result.name,
          });
        } catch (err) {
          console.error("Failed to upload image:", err);
          updateImage(placeholderUuid, { status: "failed" });
        } finally {
          URL.revokeObjectURL(blobUrl);
        }
      } else {
        // Flow mode — add as keyframe
        try {
          const result = await uploadKeyframeImage(file);
          addKeyframe({
            id: uuidv4(),
            keyframeUuid: result.keyframeUuid,
            imageUrl: result.imageUrl,
            name: result.name,
          });
        } catch (err) {
          console.error("Failed to upload keyframe:", err);
        }
      }
    }
  },
  [addImage, updateImage, addKeyframe],
);

const { isDragOver, dropHandlers } = useFileDropUpload({
  accept: ["image/jpeg", "image/png", "image/webp"],
  onFiles: handleStudioDrop,
});
```

Update the JSX to apply drop handlers to `StudioContainer`:

```tsx
return (
  <StudioContainer $dragOver={isDragOver} {...dropHandlers}>
    {/* ... rest unchanged ... */}
  </StudioContainer>
);
```

- [ ] **Step 3: Run type-check**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm type-check`
Expected: No errors

- [ ] **Step 4: Run all tests**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm vitest run`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/studio.page.tsx src/components/pages/studio/studio.page.styled.tsx
git commit -m "feat(studio): add studio-wide drag-and-drop for image upload"
```

---

### Task 7: Final Verification

**Files:** All previously created/modified files

- [ ] **Step 1: Run full type-check**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm type-check`
Expected: No errors

- [ ] **Step 2: Run full test suite**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm vitest run`
Expected: ALL PASS

- [ ] **Step 3: Run lint**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm lint`
Expected: No errors

- [ ] **Step 4: Build**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm build`
Expected: Successful build

- [ ] **Step 5: Manual smoke test checklist**

Start dev server: `cd /Users/maxcarlsonold/edream/frontend && pnpm dev`

Verify:
1. Batch mode → Images tab → Upload button appears next to Generate
2. Click Upload → file picker opens, select jpg/png/webp
3. Uploaded image appears in grid with processing state, then processed
4. Uploaded image shows as direct `<img>` (no PresignedImage)
5. Select uploaded image → star badge works
6. Continue to Actions → Generate → video generation works with uploaded image
7. Drag image file onto Images tab → drop highlight, image uploads
8. Drag image file onto Actions/Generate/Results tab → still uploads (studio-wide handler)
9. Switch to Flow mode → drag image file → adds as keyframe
10. Flow mode → upload via button → keyframe added
11. Flow mode → generate transitions → uses imageUrl (not keyframeUuid)
12. Click uploaded image → lightbox shows direct `<img>`
13. Generated image → lightbox shows `PresignedImage` (existing behavior preserved)

---

## File Summary

### New Files (2)
| File | Purpose |
|------|---------|
| `studio/utils/upload-keyframe-image.ts` | Shared multipart upload utility |
| `studio/utils/__tests__/upload-keyframe-image.test.ts` | Upload utility tests |

### Modified Files (6)
| File | Change |
|------|--------|
| `studio/components/flow-builder.tsx` | Use shared upload utility (extract inline code) |
| `studio/hooks/useFlowGeneration.ts` | Pass `imageUrl` instead of `keyframeUuid` (bugfix) |
| `studio/hooks/useBatchSubmit.ts` | Pass image URL instead of keyframe UUID for uploaded images (bugfix) |
| `studio/components/images-tab.tsx` | Upload button, drag-drop, conditional image rendering |
| `studio/components/images-tab.styled.tsx` | `ImagesTabContainer`, `LightboxUploadedImage` |
| `studio/studio.page.tsx` + `.styled.tsx` | Studio-wide drag-and-drop handler |
