# Phase 2: Image Upload (Batch Mode) — Design Spec

**Date:** 2026-05-14
**Status:** Approved
**Depends on:** Phase 0 (keyframe strip with upload) merged into `feat/phase-0-keyframe-strip`

---

## Goal

Add image upload (button + drag-and-drop) to the batch mode Images tab. Users can upload their own images as source material for video generation, alongside AI-generated images.

## Architecture

### Upload Flow

Reuse the same multipart keyframe upload sequence already working in flow mode:

1. `POST /v1/keyframe` — create keyframe entity, get UUID
2. `POST /v1/keyframe/{uuid}/image/init` — get `uploadId` + presigned URLs
3. `PUT` each 5MB chunk to presigned URLs
4. `POST /v1/keyframe/{uuid}/image/complete` — finalize with parts array
5. `GET /v1/keyframe/{uuid}` — fetch finalized keyframe with image URL

The resulting keyframe UUID and image URL populate a `StudioImage` with `status: "processed"` (ready immediately, same as playlist imports).

### Shared Upload Utility

The multipart upload sequence is currently inlined in `flow-builder.tsx`. Extract it into a shared utility so both flow mode and batch mode call the same function:

**New file:** `frontend/src/components/pages/studio/utils/upload-keyframe-image.ts`

```typescript
interface UploadResult {
  keyframeUuid: string;
  imageUrl: string;
  name: string;
}

export async function uploadKeyframeImage(file: File): Promise<UploadResult>
```

Handles the full 5-step sequence. Throws on failure. Caller handles error display and store updates.

### Images Tab Changes

**File:** `frontend/src/components/pages/studio/components/images-tab.tsx`

1. Add "Upload" button next to "Generate Images" in the generate section header
2. Add hidden `<input type="file" accept="image/jpeg,image/png,image/webp" multiple />`
3. Wire `useFileDropUpload` hook to the tab container for drag-and-drop
4. On files received (drop or button):
   - For each file, call `uploadKeyframeImage(file)`
   - On success, call `addImage({ uuid: keyframeUuid, url: imageUrl, name, status: "processed", selected: false })`
   - On failure, `console.error` (matches existing error handling pattern)
5. Drag-over visual: gold border highlight on the images tab container (same `$dragOver` pattern as `FlowContainer`)

### Flow Builder Refactor

**File:** `frontend/src/components/pages/studio/components/flow-builder.tsx`

Replace the inline upload sequence with a call to the shared `uploadKeyframeImage` utility. No behavior change — just extraction.

## What Doesn't Change

- No backend changes
- No new types (uses existing `StudioImage`)
- No changes to studio store
- No changes to `useFileDropUpload` hook (already generic)
- No changes to generation, actions, or results tabs
- Accepted formats: jpg, png, webp (same as flow mode)
- Images passed as-is to video models (no crop/resize)

## File Summary

| File | Change |
|------|--------|
| `studio/utils/upload-keyframe-image.ts` | **New** — shared multipart upload utility |
| `studio/components/images-tab.tsx` | **Modify** — add upload button, drag-and-drop, upload handler |
| `studio/components/images-tab.styled.tsx` | **Modify** — add drag-over style to container |
| `studio/components/flow-builder.tsx` | **Modify** — extract inline upload to shared utility |
