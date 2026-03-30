# Fix Presigned URL Expiration in Studio

**Date:** 2026-03-15
**Status:** Approved

## Problem

The Studio persists presigned R2 URLs (30-minute TTL) to localStorage via Zustand. When the user returns after 30 minutes, all thumbnails are broken (403 from R2). New thumbnails work because they were just presigned; older ones have expired.

This affects:
- Qwen-generated images (`StudioImage.url`)
- Playlist-imported images (`StudioImage.url`)
- Completed job thumbnails (`StudioJob.thumbnailUrl`)

## Solution

Replace presigned URL storage with a stable redirect endpoint. The frontend stores only dream UUIDs and resolves thumbnails at render time through a backend 302 redirect.

### Backend: Redirect Endpoint

**`GET /v1/dream/:uuid/thumbnail`**

1. Auth middleware (same as other dream endpoints)
2. Fetch dream's `thumbnail` column (lightweight query)
3. Verify ownership (user owns dream or is admin)
4. If no thumbnail, return 404
5. Presign the R2 key via `presignClient.generatePresignedUrl()`
6. Return `302 Found` with `Location: <presigned-url>`
7. Set `Cache-Control: private, max-age=1200` (20 min, under 30 min TTL)

Added to `backend/src/routes/v1/dream.routes.ts`. Requires auth, no role restriction.

### Frontend: `<PresignedImage>` Component

**`frontend/src/components/shared/presigned-image.tsx`**

```typescript
interface PresignedImageProps extends React.ImgHTMLAttributes<HTMLImageElement> {
  dreamUuid: string;
  mediaType?: "thumbnail" | "video"; // defaults to "thumbnail"
}
```

- Constructs `src` as `${API_BASE_URL}/v1/dream/${dreamUuid}/${mediaType}`
- Passes through all standard `<img>` props
- No internal state, caching, or hooks -- just URL construction

Also exports a `getPresignedUrl(dreamUuid, mediaType?)` utility for cases needing a raw URL string (lightbox, background-image).

### Studio Component Changes

| Component | Before | After |
|-----------|--------|-------|
| `images-tab.tsx` (processed) | `<img src={img.url}>` | `<PresignedImage dreamUuid={img.uuid}>` |
| `images-tab.tsx` (lightbox) | `<img src={expandedImageUrl}>` | `<img src={getPresignedUrl(expandedUuid)}>` |
| `results-tab.tsx` (jobs) | `src={job.previewFrame \|\| job.thumbnailUrl}` | `previewFrame ? <img src={previewFrame}> : <PresignedImage dreamUuid={job.dreamUuid}>` |
| `generate-tab.tsx` (combos) | `<img src={image.url}>` | `<PresignedImage dreamUuid={image.uuid}>` |
| `add-from-playlist-modal.tsx` | `<img src={dream.thumbnail}>` | `<PresignedImage dreamUuid={dream.uuid}>` |

### Code Removed

- `missingThumbnailJobs` effect in `useStudioJobProgress.ts` (~20 lines) -- no longer needed since display doesn't depend on fetching thumbnail URLs
- `thumbnailUrl` update logic from the polling handler -- display uses dreamUuid directly

### What Stays Unchanged

- Polling logic for `status` and `progress` updates
- Socket `job:progress` handler for preview frames during processing
- Store persistence shape (images/jobs arrays still persisted, just `url`/`thumbnailUrl` no longer used for display)
- `previewFrame` display during active processing

## Trade-offs

- **Every processed image render hits the backend** -- one lightweight DB query + one presign call per thumbnail load. Mitigated by `Cache-Control: private, max-age=1200` (browser caches for 20 min).
- **302 targets change** -- browser image caching is less effective than static URLs. Acceptable for Studio's interactive use case.
- **Reusable** -- `<PresignedImage>` can replace presigned URL usage anywhere in the app, preventing this bug class globally.
