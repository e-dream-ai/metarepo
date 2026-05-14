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

### UUID and Image Rendering

Generated images have **dream UUIDs** — `PresignedImage` fetches thumbnails via `/v1/dream/{uuid}/thumbnail`. Uploaded images have **keyframe UUIDs**, which `PresignedImage` cannot resolve.

**Solution:** Uploaded `StudioImage` entries store the direct image URL in the `url` field (returned by `GET /v1/keyframe/{uuid}`). The images grid conditionally renders:
- If `img.url` starts with `http` → plain `<img src={img.url}>` (uploaded images, already a CDN URL)
- Otherwise → `<PresignedImage dreamUuid={img.uuid}>` (generated images, need presigned fetch)

This matches how "Add from Playlist" already stores `dream.thumbnail` in the `url` field — playlist imports just happen to also have valid dream UUIDs.

### Video Generation Compatibility

Flow mode already passes keyframe UUIDs to `buildVideoAlgoParams` as the `imageUuid` parameter, and the backend resolves them correctly for wan-i2v and wan-i2v-lora (the `image` field accepts keyframe UUIDs).

For ltx-i2v, the `source_dream_uuid` field specifically expects a dream UUID. **Uploaded images will not work with ltx-i2v.** This is acceptable for Phase 2 — the UI should disable or hide uploaded images when ltx-i2v is the selected model, with a tooltip explaining why. This matches the existing pattern where LoRA-enabled presets restrict available durations.

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

Handles the full 5-step sequence. Uses `getRequestHeaders` for auth. Throws on failure. Caller handles error display and store updates.

### Images Tab Changes

**File:** `frontend/src/components/pages/studio/components/images-tab.tsx`

1. Add "Upload" button next to "Generate Images" in the generate section header
2. Add hidden `<input type="file" accept="image/jpeg,image/png,image/webp" multiple />`
3. Wire `useFileDropUpload` hook to the images tab container for drag-and-drop
4. On files received (drop or button):
   - For each file, add a placeholder `StudioImage` with `status: "processing"` and a local blob URL for immediate visual feedback
   - Call `uploadKeyframeImage(file)`
   - On success, update the image with `{ uuid: keyframeUuid, url: imageUrl, status: "processed" }`
   - On failure, update with `status: "failed"`
5. Drag-over visual: gold border highlight on the images tab container (same `$dragOver` pattern as `FlowContainer`)
6. Conditional rendering: use direct `<img>` for uploaded images, `PresignedImage` for generated ones

### Studio-Wide Drag-and-Drop

The roadmap specifies drag-and-drop should work regardless of active tab. Wire `useFileDropUpload` at the `studio.page.tsx` level, wrapping the entire studio container. When files are dropped:
- If in batch mode: add to images (same as dropping on Images tab)
- If in flow mode: add as keyframes (existing behavior in flow-builder)

The images-tab-level drop handler is kept as well for the visual highlight, but the studio-level handler ensures drops work from any tab.

### Flow Builder Refactor

**File:** `frontend/src/components/pages/studio/components/flow-builder.tsx`

Replace the inline upload sequence with a call to the shared `uploadKeyframeImage` utility. No behavior change — just extraction.

## What Doesn't Change

- No backend changes
- No new types (uses existing `StudioImage`)
- No changes to studio store (uses existing `addImage`, `updateImage`)
- No changes to `useFileDropUpload` hook (already generic)
- Accepted formats: jpg, png, webp (same as flow mode)
- Images passed as-is to video models (no crop/resize)

## Known Limitations

- Uploaded images cannot be used with ltx-i2v (needs dream UUID for `source_dream_uuid`). UI greys them out when ltx-i2v is selected.
- No file size limit enforced client-side. Large files will upload slowly but won't break.
- No duplicate detection — dropping the same file twice creates two keyframes.

## File Summary

| File | Change |
|------|--------|
| `studio/utils/upload-keyframe-image.ts` | **New** — shared multipart upload utility |
| `studio/components/images-tab.tsx` | **Modify** — add upload button, drag-and-drop, upload handler, conditional rendering |
| `studio/components/images-tab.styled.tsx` | **Modify** — add drag-over style to container |
| `studio/components/flow-builder.tsx` | **Modify** — extract inline upload to shared utility |
| `studio/studio.page.tsx` | **Modify** — add studio-wide drag-and-drop handler |
