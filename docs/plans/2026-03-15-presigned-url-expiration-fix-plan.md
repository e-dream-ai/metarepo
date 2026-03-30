# Presigned URL Expiration Fix — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Eliminate broken thumbnail images in Studio by replacing persisted presigned URLs with a stable backend redirect endpoint and a reusable `<PresignedImage>` component.

**Architecture:** New `GET /v1/dream/:uuid/thumbnail` endpoint returns a 302 redirect to a freshly presigned R2 URL. Frontend `<PresignedImage>` component constructs stable URLs using dream UUIDs. Studio components switch from storing/displaying presigned URLs to using this component.

**Tech Stack:** TypeScript, Express, React, Zustand, R2 presigning via `PresignServiceClient`

**Design doc:** `docs/plans/2026-03-15-presigned-url-expiration-fix-design.md`

---

### Task 1: Backend — Add thumbnail redirect endpoint

**Files:**
- Modify: `backend/src/controllers/dream.controller.ts` (add handler at end of file, before closing export)
- Modify: `backend/src/routes/v1/dream.routes.ts` (add route before `export default dreamRouter`)

**Step 1: Write the failing test**

Create test in `backend/src/__tests__/dream.get.test.ts`. Follow the existing `createReqRes` + mock pattern in that file.

```typescript
describe("handleGetDreamThumbnail", () => {
  beforeEach(() => jest.resetModules());

  it("returns 302 redirect with presigned URL when dream has thumbnail", async () => {
    const mockPresignedUrl = "https://r2.example.com/signed-thumb?token=abc";

    jest.mock("clients/presign.client", () => ({
      __esModule: true,
      presignClient: {
        generatePresignedUrl: jest.fn().mockResolvedValue(mockPresignedUrl),
      },
    }));

    const mockDream = {
      uuid: "dream-1",
      thumbnail: "user1/dream-1/thumbnails/dream-1.jpg",
      user: { id: 1 },
      hidden: false,
    };

    jest.mock("shared/datasource", () => ({
      __esModule: true,
      default: {
        getRepository: () => ({
          findOne: jest.fn().mockResolvedValue(mockDream),
        }),
      },
    }));

    // Mock other utils (follow existing pattern in file)
    jest.mock("utils/dream.util", () => ({
      __esModule: true,
      findDreamPlaylistItems: jest.fn().mockResolvedValue([]),
      getDreamSelectedColumns: () => ({}),
    }));
    jest.mock("utils/transform.util", () => ({
      __esModule: true,
      transformDreamWithSignedUrls: jest.fn().mockImplementation(<T>(d: T) => d),
      transformDreamsWithSignedUrls: jest.fn().mockResolvedValue([]),
    }));
    jest.mock("utils/request.util", () => ({
      __esModule: true,
      handleNotFound: jest.fn(),
    }));

    const { handleGetDreamThumbnail } = await import(
      "controllers/dream.controller"
    );

    const { req, res } = createReqRes();
    req.params = { dreamUUID: "dream-1" };

    const redirect = jest.fn();
    const set = jest.fn();
    (res as any).redirect = redirect;
    (res as any).set = set;

    await handleGetDreamThumbnail(req, res);

    expect(set).toHaveBeenCalledWith(
      "Cache-Control",
      "private, max-age=1200",
    );
    expect(redirect).toHaveBeenCalledWith(302, mockPresignedUrl);
  });

  it("returns 404 when dream has no thumbnail", async () => {
    const mockDream = {
      uuid: "dream-1",
      thumbnail: null,
      user: { id: 1 },
      hidden: false,
    };

    jest.mock("shared/datasource", () => ({
      __esModule: true,
      default: {
        getRepository: () => ({
          findOne: jest.fn().mockResolvedValue(mockDream),
        }),
      },
    }));
    jest.mock("clients/presign.client", () => ({
      __esModule: true,
      presignClient: { generatePresignedUrl: jest.fn() },
    }));
    jest.mock("utils/dream.util", () => ({
      __esModule: true,
      findDreamPlaylistItems: jest.fn().mockResolvedValue([]),
      getDreamSelectedColumns: () => ({}),
    }));
    jest.mock("utils/transform.util", () => ({
      __esModule: true,
      transformDreamWithSignedUrls: jest.fn().mockImplementation(<T>(d: T) => d),
      transformDreamsWithSignedUrls: jest.fn().mockResolvedValue([]),
    }));
    jest.mock("utils/request.util", () => ({
      __esModule: true,
      handleNotFound: jest.fn(),
    }));

    const { handleGetDreamThumbnail } = await import(
      "controllers/dream.controller"
    );

    const { req, res, status, json } = createReqRes();
    req.params = { dreamUUID: "dream-1" };

    await handleGetDreamThumbnail(req, res);

    expect(status).toHaveBeenCalledWith(404);
  });
});
```

**Step 2: Run test to verify it fails**

Run: `cd backend && pnpm run test -- --testPathPattern=dream.get.test --verbose`
Expected: FAIL — `handleGetDreamThumbnail` is not exported

**Step 3: Write the controller handler**

Add to `backend/src/controllers/dream.controller.ts` (before the closing `export default`):

```typescript
/**
 * GET /v1/dream/:dreamUUID/thumbnail
 * Returns a 302 redirect to a freshly presigned thumbnail URL.
 */
const handleGetDreamThumbnail = async (req: RequestType, res: ResponseType) => {
  const { dreamUUID } = req.params;
  const user = res.locals.user;

  const dream = await dreamRepository.findOne({
    where: { uuid: dreamUUID },
    relations: { user: true },
    select: { uuid: true, thumbnail: true, hidden: true, user: { id: true } },
  });

  if (!dream) {
    return handleNotFound(req, res);
  }

  const isOwner = dream.user.id === user?.id;
  const isAllowed = canExecuteAction({
    isOwner,
    allowedRoles: [ROLES.ADMIN_GROUP],
    userRole: user?.role?.name,
  });

  if (dream.hidden && !isAllowed) {
    return handleNotFound(req, res);
  }

  if (!dream.thumbnail) {
    return res
      .status(httpStatus.NOT_FOUND)
      .json(jsonResponse({ success: false, message: "No thumbnail available" }));
  }

  const presignedUrl = await presignClient.generatePresignedUrl(dream.thumbnail);
  res.set("Cache-Control", "private, max-age=1200");
  return res.redirect(302, presignedUrl);
};
```

Make sure `presignClient` is imported at the top of the file:
```typescript
import { presignClient } from "clients/presign.client";
```

Add `handleGetDreamThumbnail` to the exported `dreamController` object.

**Step 4: Add the route**

Add to `backend/src/routes/v1/dream.routes.ts` (before `export default dreamRouter`):

```typescript
dreamRouter.get(
  "/:dreamUUID/thumbnail",
  requireAuth,
  checkRoleMiddleware([
    ROLES.USER_GROUP,
    ROLES.CREATOR_GROUP,
    ROLES.ADMIN_GROUP,
  ]),
  dreamController.handleGetDreamThumbnail,
);
```

**Important:** This route must be placed BEFORE any wildcard or catch-all routes that match `/:dreamUUID/:something`. Check existing route order — it should go near the other `/:dreamUUID/...` routes (around line 198 where `/:dreamUUID/vote` is defined).

**Step 5: Run tests to verify they pass**

Run: `cd backend && pnpm run test -- --testPathPattern=dream.get.test --verbose`
Expected: PASS

**Step 6: Run full backend test suite**

Run: `cd backend && pnpm run test`
Expected: All existing tests still pass

**Step 7: Commit**

```bash
git add backend/src/controllers/dream.controller.ts backend/src/routes/v1/dream.routes.ts backend/src/__tests__/dream.get.test.ts
git commit -m "feat(backend): add GET /v1/dream/:uuid/thumbnail redirect endpoint"
```

---

### Task 2: Frontend — Create `<PresignedImage>` component and `getPresignedUrl` utility

**Files:**
- Create: `frontend/src/components/shared/presigned-image/presigned-image.tsx`
- Create: `frontend/src/components/shared/presigned-image/index.ts`

**Step 1: Create the component**

`frontend/src/components/shared/presigned-image/presigned-image.tsx`:

```typescript
import React from "react";
import { URL } from "@/constants/api.constants";

export const getPresignedUrl = (
  dreamUuid: string,
  mediaType: "thumbnail" | "video" = "thumbnail",
): string => `${URL}/v1/dream/${dreamUuid}/${mediaType}`;

interface PresignedImageProps
  extends React.ImgHTMLAttributes<HTMLImageElement> {
  dreamUuid: string;
  mediaType?: "thumbnail" | "video";
}

export const PresignedImage = React.forwardRef<
  HTMLImageElement,
  PresignedImageProps
>(({ dreamUuid, mediaType = "thumbnail", ...imgProps }, ref) => (
  <img ref={ref} src={getPresignedUrl(dreamUuid, mediaType)} {...imgProps} />
));

PresignedImage.displayName = "PresignedImage";
```

`frontend/src/components/shared/presigned-image/index.ts`:

```typescript
export { PresignedImage, getPresignedUrl } from "./presigned-image";
```

**Step 2: Verify it compiles**

Run: `cd frontend && pnpm run type-check`
Expected: No new errors

**Step 3: Commit**

```bash
git add frontend/src/components/shared/presigned-image/
git commit -m "feat(frontend): add reusable PresignedImage component and getPresignedUrl utility"
```

---

### Task 3: Frontend — Update `images-tab.tsx` to use `<PresignedImage>`

**Files:**
- Modify: `frontend/src/components/pages/studio/components/images-tab.tsx`

**Step 1: Update imports**

Add at top of file:
```typescript
import { PresignedImage, getPresignedUrl } from "@/components/shared/presigned-image";
```

**Step 2: Replace image thumbnail rendering (around line 162)**

Change:
```typescript
<ImageThumbnail
  src={img.url}
  alt={img.name}
  onClick={() => setExpandedImageUrl(img.url)}
  style={{ cursor: "zoom-in" }}
/>
```

To:
```typescript
<ImageThumbnail
  as={PresignedImage}
  dreamUuid={img.uuid}
  alt={img.name}
  onClick={() => setExpandedImageUuid(img.uuid)}
  style={{ cursor: "zoom-in" }}
/>
```

Note: If `ImageThumbnail` is a styled `img`, using `as={PresignedImage}` will forward styled props. If styled-components `as` doesn't work cleanly here, wrap instead:
```typescript
<PresignedImage
  dreamUuid={img.uuid}
  alt={img.name}
  onClick={() => setExpandedImageUuid(img.uuid)}
  style={{ cursor: "zoom-in" }}
  className={imageThumbnailClassName}
/>
```

Check how `ImageThumbnail` is defined — if it's `styled.img`, prefer the `as` approach.

**Step 3: Update lightbox state from URL to UUID**

Change `expandedImageUrl` state to `expandedImageUuid`:
```typescript
// Before:
const [expandedImageUrl, setExpandedImageUrl] = useState<string | null>(null);

// After:
const [expandedImageUuid, setExpandedImageUuid] = useState<string | null>(null);
```

Update the lightbox render:
```typescript
// Before:
{expandedImageUrl && (
  <LightboxOverlay onClick={() => setExpandedImageUrl(null)}>
    <LightboxImage src={expandedImageUrl} alt="Expanded" />
  </LightboxOverlay>
)}

// After:
{expandedImageUuid && (
  <LightboxOverlay onClick={() => setExpandedImageUuid(null)}>
    <LightboxImage src={getPresignedUrl(expandedImageUuid)} alt="Expanded" />
  </LightboxOverlay>
)}
```

**Step 4: Verify it compiles**

Run: `cd frontend && pnpm run type-check`
Expected: No new errors

**Step 5: Commit**

```bash
git add frontend/src/components/pages/studio/components/images-tab.tsx
git commit -m "fix(studio): use PresignedImage in images tab to prevent expired thumbnails"
```

---

### Task 4: Frontend — Update `results-tab.tsx` to use `<PresignedImage>`

**Files:**
- Modify: `frontend/src/components/pages/studio/components/results-tab.tsx`

**Step 1: Update imports**

Add:
```typescript
import { PresignedImage } from "@/components/shared/presigned-image";
```

**Step 2: Replace job thumbnail rendering (around line 411-422)**

Change:
```typescript
<ResultThumb>
  {job.previewFrame ? (
    <ResultThumbImg
      src={`data:image/jpeg;base64,${job.previewFrame}`}
      alt="preview"
    />
  ) : job.thumbnailUrl ? (
    <ResultThumbImg
      src={job.thumbnailUrl}
      alt="preview"
    />
  ) : null}
```

To:
```typescript
<ResultThumb>
  {job.previewFrame ? (
    <ResultThumbImg
      src={`data:image/jpeg;base64,${job.previewFrame}`}
      alt="preview"
    />
  ) : job.status === "processed" ? (
    <ResultThumbImg
      as={PresignedImage}
      dreamUuid={job.dreamUuid}
      alt="thumbnail"
    />
  ) : null}
```

Same note as Task 3 re: styled component `as` prop — check if `ResultThumbImg` is `styled.img`.

**Step 3: Verify it compiles**

Run: `cd frontend && pnpm run type-check`
Expected: No new errors

**Step 4: Commit**

```bash
git add frontend/src/components/pages/studio/components/results-tab.tsx
git commit -m "fix(studio): use PresignedImage in results tab for job thumbnails"
```

---

### Task 5: Frontend — Update `generate-tab.tsx` to use `<PresignedImage>`

**Files:**
- Modify: `frontend/src/components/pages/studio/components/generate-tab.tsx`

**Step 1: Update imports**

Add:
```typescript
import { PresignedImage } from "@/components/shared/presigned-image";
```

**Step 2: Replace combo preview thumbnail (around line 136)**

Change:
```typescript
{image.url && <CellThumb src={image.url} alt="" />}
```

To:
```typescript
{image.status === "processed" && (
  <CellThumb as={PresignedImage} dreamUuid={image.uuid} alt="" />
)}
```

**Step 3: Verify it compiles**

Run: `cd frontend && pnpm run type-check`
Expected: No new errors

**Step 4: Commit**

```bash
git add frontend/src/components/pages/studio/components/generate-tab.tsx
git commit -m "fix(studio): use PresignedImage in generate tab combo previews"
```

---

### Task 6: Frontend — Update `add-from-playlist-modal.tsx` to use `<PresignedImage>`

**Files:**
- Modify: `frontend/src/components/pages/studio/components/add-from-playlist-modal.tsx`

**Step 1: Update imports**

Add:
```typescript
import { PresignedImage } from "@/components/shared/presigned-image";
```

**Step 2: Replace playlist item thumbnail (around line 172)**

Change:
```typescript
<ImageThumbnail src={dream.thumbnail} alt={dream.name} />
```

To:
```typescript
<ImageThumbnail as={PresignedImage} dreamUuid={dream.uuid} alt={dream.name} />
```

**Step 3: Verify it compiles**

Run: `cd frontend && pnpm run type-check`
Expected: No new errors

**Step 4: Commit**

```bash
git add frontend/src/components/pages/studio/components/add-from-playlist-modal.tsx
git commit -m "fix(studio): use PresignedImage in playlist import modal"
```

---

### Task 7: Frontend — Remove dead thumbnail-fetching code from `useStudioJobProgress.ts`

**Files:**
- Modify: `frontend/src/components/pages/studio/hooks/useStudioJobProgress.ts`

**Step 1: Remove the `missingThumbnailJobs` effect (lines 105-125)**

Delete this entire `useEffect` block:
```typescript
useEffect(() => {
  const state = useStudioStore.getState();
  const missingThumbnailJobs = state.jobs.filter(
    (j) => j.status === "processed" && !j.previewFrame && !j.thumbnailUrl,
  );
  if (missingThumbnailJobs.length === 0) return;

  missingThumbnailJobs.forEach(async (job) => {
    try {
      const { data } = await axiosClient.get(`/v1/dream/${job.dreamUuid}`);
      const dream = data.data.dream;
      if (dream.thumbnail || dream.video) {
        state.updateJob(job.dreamUuid, {
          thumbnailUrl: dream.thumbnail || dream.video,
        });
      }
    } catch {
      // Silently skip
    }
  });
}, [jobs]);
```

**Step 2: Remove `thumbnailUrl` update from polling handler (around line 162-166)**

Change:
```typescript
state.updateJob(job.dreamUuid, {
  status: dream.status,
  ...(dream.thumbnail || dream.video
    ? { thumbnailUrl: dream.thumbnail || dream.video }
    : {}),
});
```

To:
```typescript
state.updateJob(job.dreamUuid, {
  status: dream.status,
});
```

**Step 3: Similarly, remove `thumbnailUrl` update from image polling (around line 146-149)**

Change:
```typescript
state.updateImage(img.uuid, {
  status: dream.status,
  url: dream.thumbnail || dream.video || img.url,
});
```

To:
```typescript
state.updateImage(img.uuid, {
  status: dream.status,
});
```

**Step 4: Remove `thumbnailUrl` from socket handler (around line 64)**

Remove:
```typescript
...(thumbnail || video ? { thumbnailUrl: thumbnail || video } : {}),
```

And remove `thumbnail` and `video` from the destructured data type.

**Step 5: Verify it compiles**

Run: `cd frontend && pnpm run type-check`
Expected: No new errors

**Step 6: Commit**

```bash
git add frontend/src/components/pages/studio/hooks/useStudioJobProgress.ts
git commit -m "fix(studio): remove dead thumbnail URL fetching code

Display now uses PresignedImage with dream UUIDs, so we no longer
need to fetch and store presigned URLs for thumbnails."
```

---

### Task 8: Smoke test on staging

**Step 1: Start local dev environment**

Run: `cd backend && pnpm run dev` (in one terminal)
Run: `cd frontend && pnpm run dev` (in another terminal)

**Step 2: Manual verification checklist**

- [ ] Navigate to Studio, Images tab — existing processed images display thumbnails
- [ ] Generate new Qwen images — thumbnails appear after processing
- [ ] Import images from playlist — thumbnails display correctly
- [ ] Switch to Generate tab — combo grid shows image thumbnails
- [ ] Submit a batch — Results tab shows preview frames during processing, then thumbnails when done
- [ ] Close browser, wait 1 minute, reopen Studio — all thumbnails still load (the key test)
- [ ] Open Network tab — thumbnail requests show 302 → presigned R2 URL
- [ ] Check `Cache-Control: private, max-age=1200` header on 302 response

**Step 3: Commit any fixes discovered during smoke test**

---

### Task 9: Final review and cleanup

**Step 1: Run full frontend checks**

Run: `cd frontend && pnpm run type-check && pnpm run lint`
Expected: Clean

**Step 2: Run full backend tests**

Run: `cd backend && pnpm run test`
Expected: All pass

**Step 3: Check for any remaining references to `img.url` or `job.thumbnailUrl` in Studio components**

Search: `grep -r "img\.url\|thumbnailUrl" frontend/src/components/pages/studio/`

Any remaining references in display code should be converted. References in type definitions and store logic are fine to keep for now.

**Step 4: Final commit if needed, then open PR**

```bash
git push -u origin fix/presigned-url-expiration
```
