# Studio Review Round 2 — Fixes Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Fix the 11 critical and high-severity issues found by the competing code review agents.

**Architecture:** All changes are frontend-only (React/TypeScript/Zustand/Socket.IO). Pure-function and store fixes get TDD. Hook and component fixes get type-check + lint verification. Commits after each task.

**Tech Stack:** React 18, TypeScript, Zustand 5 (with persist), Socket.IO, Vitest, Vite 5

**Verification commands:**
```bash
cd /Users/maxcarlsonold/edream/frontend
pnpm run test          # Vitest — all tests must pass
pnpm run type-check    # tsc --noEmit — 0 errors
pnpm run lint          # ESLint — 0 errors
```

---

## Issue Map

| Task | Issue # | Severity | Description |
|------|---------|----------|-------------|
| 1 | 1 | Critical | `setActiveTab` double `set()` race condition |
| 2 | 2 | Critical | `isGenerating` not in `try/finally` — button stuck |
| 3 | 3 | Critical | Poll + socket double-counting `newCompletedCount` |
| 4 | 4 | Critical | Uprez `selectedForUprez` not cleared — duplicate submissions |
| 5 | 6 | High | Null playlist UUID guard in `useBatchSubmit` |
| 6 | 7 | High | Socket room leave-rejoin churn |
| 7 | 9 | High | Image URL not updated on socket COMPLETED |
| 8 | 10 | High | "Already submitted" count includes uprez jobs |
| 9 | 11 | High | `uprezCount` filter mismatch in results-tab |
| 10 | 8 | High | Playlist items capped at 100 |

---

### Task 1: Atomic `setActiveTab` in Zustand Store

**Problem:** `setActiveTab` calls `set()` twice — first to clear `newCompletedCount`, then to set `activeTab`. Between calls, subscribers re-render with stale state, and a concurrent `incrementNewCompleted` could be wiped.

**Files:**
- Modify: `src/stores/studio.store.ts:68-71`
- Modify: `src/stores/studio.store.test.ts`

**Step 1: Update existing test to verify atomicity**

In `src/stores/studio.store.test.ts`, the test `"clears when switching to results tab"` already passes. Add a new test that verifies both fields update atomically:

```typescript
it("atomically sets activeTab and clears newCompletedCount", () => {
  useStudioStore.getState().incrementNewCompleted();
  useStudioStore.getState().incrementNewCompleted();

  // Subscribe and capture intermediate states
  const snapshots: Array<{ activeTab: string; count: number }> = [];
  const unsub = useStudioStore.subscribe((state) => {
    snapshots.push({
      activeTab: state.activeTab,
      count: state.newCompletedCount,
    });
  });

  useStudioStore.getState().setActiveTab("results");
  unsub();

  // Should be exactly ONE state update, not two
  expect(snapshots).toHaveLength(1);
  expect(snapshots[0]).toEqual({ activeTab: "results", count: 0 });
});
```

**Step 2: Run test to verify it fails**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run test`

Expected: FAIL — snapshots will have length 2 (two separate `set()` calls).

**Step 3: Fix `setActiveTab` to single atomic `set()`**

In `src/stores/studio.store.ts`, replace lines 68-71:

```typescript
// BEFORE:
setActiveTab: (tab: StudioTab) => {
  if (tab === "results") set({ newCompletedCount: 0 });
  set({ activeTab: tab });
},

// AFTER:
setActiveTab: (tab: StudioTab) =>
  set({
    activeTab: tab,
    ...(tab === "results" ? { newCompletedCount: 0 } : {}),
  }),
```

**Step 4: Run tests to verify green**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run test`

Expected: All tests pass.

**Step 5: Run type-check and lint**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run type-check && pnpm run lint`

Expected: 0 errors.

**Step 6: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/stores/studio.store.ts src/stores/studio.store.test.ts
git commit -m "fix: atomic setActiveTab to prevent race condition between badge clear and tab switch"
```

---

### Task 2: Wrap `isGenerating` in `try/finally` in ImagesTab

**Problem:** If any promise in `handleGenerate` rejects (and the `.catch` doesn't swallow it), or if `Promise.all` throws, `setIsGenerating(false)` never fires. The generate button stays disabled forever.

**Files:**
- Modify: `src/components/pages/studio/components/images-tab.tsx:45-86`

**Step 1: Fix `handleGenerate` to use try/finally**

In `src/components/pages/studio/components/images-tab.tsx`, replace the `handleGenerate` callback (lines 45-86):

```typescript
const handleGenerate = useCallback(async () => {
  if (!imagePrompt.trim()) return;
  setIsGenerating(true);

  try {
    const baseSeed = Math.floor(Math.random() * 99_000) + 1;

    const promises = Array.from({ length: qwenParams.seedCount }, (_, i) => {
      const seed = baseSeed + i;
      const algoParams = {
        infinidream_algorithm: "qwen-image",
        prompt: imagePrompt,
        size: qwenParams.size,
        seed,
      };

      return axiosClient
        .post("/v1/dream", {
          name: `Qwen Image ${images.length + i + 1}`,
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
            size: qwenParams.size,
            status: (dream.status as StudioImage["status"]) || "queue",
            selected: false,
          });
        })
        .catch((err) => {
          console.error("Failed to create image:", err);
        });
    });

    await Promise.all(promises);
  } finally {
    setIsGenerating(false);
  }
}, [imagePrompt, qwenParams, images.length, addImage, setIsGenerating]);
```

Note: Also add `setIsGenerating` to the dependency array since we're using it inside the callback.

**Step 2: Run verification**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run test && pnpm run type-check && pnpm run lint`

Expected: All pass.

**Step 3: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/components/images-tab.tsx
git commit -m "fix: wrap isGenerating in try/finally to prevent stuck button on error"
```

---

### Task 3: Deduplicate Badge Counting in `useStudioJobProgress`

**Problem:** Both the socket listener and the polling fallback call `incrementNewCompleted()` for the same job. If the socket fires COMPLETED and the poll reads the API before the ref updates, the badge double-counts.

**Files:**
- Modify: `src/components/pages/studio/hooks/useStudioJobProgress.ts`

**Step 1: Add a `completedFlaggedUuids` ref and guard both paths**

In `src/components/pages/studio/hooks/useStudioJobProgress.ts`:

1. After the existing refs (line ~40), add:
```typescript
// Track which UUIDs have already triggered the badge increment
const completedFlaggedUuids = useRef(new Set<string>());
```

2. In the socket handler (inside `handleProgress`, around line 95-101), change the badge logic:
```typescript
// BEFORE:
if (
  wasNotCompleted &&
  isNowCompleted &&
  activeTabRef.current !== "results"
) {
  incrementNewCompletedRef.current();
}

// AFTER:
if (
  wasNotCompleted &&
  isNowCompleted &&
  !completedFlaggedUuids.current.has(dream_uuid) &&
  activeTabRef.current !== "results"
) {
  completedFlaggedUuids.current.add(dream_uuid);
  incrementNewCompletedRef.current();
}
```

3. In the polling fallback (around line 167-172), apply the same guard:
```typescript
// BEFORE:
if (
  wasNotCompleted &&
  isNowCompleted &&
  activeTabRef.current !== "results"
) {
  incrementNewCompletedRef.current();
}

// AFTER:
if (
  wasNotCompleted &&
  isNowCompleted &&
  !completedFlaggedUuids.current.has(job.dreamUuid) &&
  activeTabRef.current !== "results"
) {
  completedFlaggedUuids.current.add(job.dreamUuid);
  incrementNewCompletedRef.current();
}
```

**Step 2: Run verification**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run test && pnpm run type-check && pnpm run lint`

Expected: All pass.

**Step 3: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/hooks/useStudioJobProgress.ts
git commit -m "fix: deduplicate badge counting between socket and polling paths"
```

---

### Task 4: Clear `selectedForUprez` After Uprez Submission + Fix `uprezCount`

**Problem:** After uprezzing, `selectedForUprez` stays `true` on source jobs. The user can click "Uprez Selected" again and create duplicates. Also, `uprezCount` doesn't filter by `jobType !== "uprez"` or `status === "processed"`, mismatching the actual submission filter.

**Files:**
- Modify: `src/stores/studio.store.ts` (add `clearSelectedForUprez` action)
- Modify: `src/stores/studio.store.test.ts` (test new action)
- Modify: `src/components/pages/studio/components/results-tab.tsx:84,110-162`

**Step 1: Write failing test for `clearSelectedForUprez`**

In `src/stores/studio.store.test.ts`, add a new describe block:

```typescript
describe("clearSelectedForUprez", () => {
  it("clears selectedForUprez on all non-uprez processed jobs", () => {
    useStudioStore.getState().addJob({
      imageId: "img1",
      actionId: "act1",
      dreamUuid: "dream1",
      jobType: "wan-i2v",
      status: "processed",
      selectedForUprez: true,
    });
    useStudioStore.getState().addJob({
      imageId: "img1",
      actionId: "act2",
      dreamUuid: "dream2",
      jobType: "wan-i2v",
      status: "processing",
      selectedForUprez: true,
    });

    useStudioStore.getState().clearSelectedForUprez();

    const jobs = useStudioStore.getState().jobs;
    // Processed job should be cleared
    expect(jobs[0].selectedForUprez).toBe(false);
    // Non-processed job should be unchanged
    expect(jobs[1].selectedForUprez).toBe(true);
  });
});
```

**Step 2: Run test to verify it fails**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run test`

Expected: FAIL — `clearSelectedForUprez is not a function`

**Step 3: Add `clearSelectedForUprez` to store**

In `src/stores/studio.store.ts`:

1. Add to the type definition (after `toggleJobUprez` around line 48):
```typescript
clearSelectedForUprez: () => void;
```

2. Add the implementation (after `toggleJobUprez` implementation, around line 161):
```typescript
clearSelectedForUprez: () =>
  set((s) => ({
    jobs: s.jobs.map((j) =>
      j.selectedForUprez && j.status === "processed"
        ? { ...j, selectedForUprez: false }
        : j,
    ),
  })),
```

**Step 4: Run tests to verify green**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run test`

Expected: All pass.

**Step 5: Fix `uprezCount` and call `clearSelectedForUprez` in results-tab**

In `src/components/pages/studio/components/results-tab.tsx`:

1. Add store selector for new action:
```typescript
const clearSelectedForUprez = useStudioStore((s) => s.clearSelectedForUprez);
```

2. Fix `uprezCount` (line 84) to match the submission filter:
```typescript
// BEFORE:
const uprezCount = jobs.filter((j) => j.selectedForUprez).length;

// AFTER:
const uprezCount = jobs.filter(
  (j) => j.selectedForUprez && j.status === "processed" && j.jobType !== "uprez",
).length;
```

3. In `handleUprezSelected`, after the `for` loop completes (before `finally`), clear the flags:
```typescript
// After the for loop, before finally:
clearSelectedForUprez();
```

Full updated `handleUprezSelected`:
```typescript
const handleUprezSelected = useCallback(async () => {
  const toUprez = jobs.filter(
    (j) =>
      j.selectedForUprez &&
      j.status === "processed" &&
      j.jobType !== "uprez",
  );
  if (toUprez.length === 0) return;

  setIsUprezzing(true);
  try {
    for (const job of toUprez) {
      // ... existing loop body unchanged ...
    }
    clearSelectedForUprez();
  } finally {
    setIsUprezzing(false);
  }
}, [jobs, createDream, addJob, outputPlaylistId, clearSelectedForUprez]);
```

**Step 6: Run full verification**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run test && pnpm run type-check && pnpm run lint`

Expected: All pass.

**Step 7: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/stores/studio.store.ts src/stores/studio.store.test.ts src/components/pages/studio/components/results-tab.tsx
git commit -m "fix: clear selectedForUprez after uprez submission to prevent duplicates"
```

---

### Task 5: Guard Null Playlist UUID in `useBatchSubmit`

**Problem:** If playlist creation succeeds but the API returns `undefined` for `uuid`, the code proceeds to call `/v1/playlist/undefined/add-item` for every combo. No guard exists.

**Files:**
- Modify: `src/components/pages/studio/hooks/useBatchSubmit.ts:53-63`

**Step 1: Add null guard after playlist creation**

In `src/components/pages/studio/hooks/useBatchSubmit.ts`, replace lines 53-63:

```typescript
// BEFORE:
let playlistId = outputPlaylistId;
if (!playlistId) {
  const now = new Date();
  const name = `Studio ${now
    .toISOString()
    .slice(0, 16)
    .replace("T", " ")}`;
  const { data } = await axiosClient.post("/v1/playlist", { name });
  playlistId = data.data.playlist.uuid;
  setOutputPlaylistId(playlistId);
}

// AFTER:
let playlistId = outputPlaylistId;
if (!playlistId) {
  const now = new Date();
  const name = `Studio ${now
    .toISOString()
    .slice(0, 16)
    .replace("T", " ")}`;
  const { data } = await axiosClient.post("/v1/playlist", { name });
  playlistId = data.data?.playlist?.uuid;
  if (!playlistId) {
    console.error("Playlist creation returned no UUID");
    return;
  }
  setOutputPlaylistId(playlistId);
}
```

**Step 2: Run verification**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run test && pnpm run type-check && pnpm run lint`

Expected: All pass.

**Step 3: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/hooks/useBatchSubmit.ts
git commit -m "fix: guard against null playlist UUID from API in batch submit"
```

---

### Task 6: Diff-Based Socket Room Management

**Problem:** Every time `pendingUuids` changes (e.g., new job added or status update), the cleanup effect fires `leave_dream_room` for ALL pending UUIDs, then re-joins them all. This creates a brief window where progress events are missed.

**Files:**
- Modify: `src/components/pages/studio/hooks/useStudioJobProgress.ts:111-130`

**Step 1: Replace effect 2 with diff-based room management**

In `src/components/pages/studio/hooks/useStudioJobProgress.ts`, replace the room management effect (lines 111-130):

```typescript
// --- Effect 2: Join/leave socket rooms for pending UUIDs (diff-based) ---
const joinedRoomsRef = useRef(new Set<string>());

useEffect(() => {
  if (!socket) return;

  const currentSet = new Set(pendingUuids);

  // Join rooms we haven't joined yet
  const toJoin = pendingUuids.filter(
    (uuid) => !joinedRoomsRef.current.has(uuid),
  );
  toJoin.forEach((uuid) => {
    socket.emit(JOIN_DREAM_ROOM_EVENT, uuid);
    joinedRoomsRef.current.add(uuid);
  });

  // Leave rooms no longer needed
  for (const uuid of joinedRoomsRef.current) {
    if (!currentSet.has(uuid)) {
      socket.emit(LEAVE_DREAM_ROOM_EVENT, uuid);
      joinedRoomsRef.current.delete(uuid);
    }
  }

  // On reconnect, rejoin all current rooms
  const handleReconnect = () => {
    joinedRoomsRef.current.forEach((uuid) => {
      socket.emit(JOIN_DREAM_ROOM_EVENT, uuid);
    });
  };
  socket.on("connect", handleReconnect);

  return () => {
    socket.off("connect", handleReconnect);
  };
}, [socket, pendingUuids]);
```

Note: The cleanup NO LONGER leaves all rooms. Rooms are left incrementally when UUIDs drop out of `pendingUuids`. We only need to clean up the "connect" listener.

Add a separate unmount cleanup to leave all rooms when the hook unmounts:

```typescript
// --- Cleanup: leave all rooms on unmount ---
useEffect(() => {
  return () => {
    if (!socket) return;
    joinedRoomsRef.current.forEach((uuid) => {
      socket.emit(LEAVE_DREAM_ROOM_EVENT, uuid);
    });
    joinedRoomsRef.current.clear();
  };
  // eslint-disable-next-line react-hooks/exhaustive-deps
}, []);
```

Wait — `socket` needs to be stable for this to work. Since `socket` comes from `useSocket()` which returns a stable reference, this is fine. But to be safe, use a ref for socket in the unmount cleanup. Actually, we already have the `socket` in scope. The simpler approach is to use a ref:

```typescript
const socketRef = useRef(socket);
useEffect(() => {
  socketRef.current = socket;
}, [socket]);
```

Then the unmount cleanup:
```typescript
useEffect(() => {
  return () => {
    const s = socketRef.current;
    if (!s) return;
    joinedRoomsRef.current.forEach((uuid) => {
      s.emit(LEAVE_DREAM_ROOM_EVENT, uuid);
    });
    joinedRoomsRef.current.clear();
  };
}, []);
```

**Step 2: Run verification**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run test && pnpm run type-check && pnpm run lint`

Expected: All pass.

**Step 3: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/hooks/useStudioJobProgress.ts
git commit -m "fix: diff-based socket room management to prevent leave-rejoin churn"
```

---

### Task 7: Fetch Image URL on Socket COMPLETED

**Problem:** When a Qwen image job completes via Socket.IO, `status` and `previewFrame` are updated but `url` stays empty. The image thumbnail is blank until the next 10-second poll cycle.

**Files:**
- Modify: `src/components/pages/studio/hooks/useStudioJobProgress.ts:75-82`

**Step 1: Add API fetch for completed images in socket handler**

In the `handleProgress` function, after updating the image (around line 82), add:

```typescript
const image = imagesRef.current.find((img) => img.uuid === dream_uuid);
if (image) {
  updateImageRef.current(dream_uuid, {
    progress,
    previewFrame: preview_frame,
    ...(mappedStatus ? { status: mappedStatus } : {}),
  });

  // Fetch final thumbnail URL when image completes
  if (mappedStatus === "processed") {
    axiosClient
      .get(`/v1/dream/${dream_uuid}`)
      .then(({ data }) => {
        const dream = data.data?.dream;
        if (dream?.thumbnail) {
          updateImageRef.current(dream_uuid, {
            url: dream.thumbnail,
          });
        }
      })
      .catch(() => {
        // Polling fallback will handle this
      });
  }
}
```

**Step 2: Run verification**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run test && pnpm run type-check && pnpm run lint`

Expected: All pass.

**Step 3: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/hooks/useStudioJobProgress.ts
git commit -m "fix: fetch image thumbnail URL on socket COMPLETED instead of waiting for poll"
```

---

### Task 8: Filter Uprez Jobs from "Already Submitted" Count

**Problem:** `generate-tab.tsx` shows `jobs.length` in the "already submitted" message, which includes uprez jobs. This inflates the count and confuses users.

**Files:**
- Modify: `src/components/pages/studio/components/generate-tab.tsx:171`

**Step 1: Filter uprez jobs from the count**

In `src/components/pages/studio/components/generate-tab.tsx`, replace line 171:

```typescript
// BEFORE:
{jobs.length > 0 && ` (${jobs.length} already submitted)`}

// AFTER:
{(() => {
  const wanJobCount = jobs.filter((j) => j.jobType !== "uprez").length;
  return wanJobCount > 0 ? ` (${wanJobCount} already submitted)` : null;
})()}
```

Actually, cleaner approach — add a derived value at the top of the component (near the other `useMemo`/derived values):

```typescript
const wanJobCount = jobs.filter((j) => j.jobType !== "uprez").length;
```

Then in JSX:
```typescript
{wanJobCount > 0 && ` (${wanJobCount} already submitted)`}
```

Note: This requires importing `StudioJob` type or relying on inference. Since `jobs` is already typed from the store, `j.jobType` is already typed. No extra imports needed.

**Step 2: Run verification**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run test && pnpm run type-check && pnpm run lint`

Expected: All pass.

**Step 3: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/components/generate-tab.tsx
git commit -m "fix: exclude uprez jobs from 'already submitted' count in generate tab"
```

---

### Task 9: Fix `uprezCount` Filter in ResultsTab

**Note:** This is already handled in Task 4 step 5. This task is intentionally empty — the fix is part of Task 4. Included in the issue map for traceability.

---

### Task 10: Increase Playlist Items Limit

**Problem:** `add-from-playlist-modal.tsx` fetches only 100 playlist items. Power users with large playlists see truncated lists with no indication.

**Files:**
- Modify: `src/components/pages/studio/components/add-from-playlist-modal.tsx:65`

**Step 1: Increase limit from 100 to 500**

In `src/components/pages/studio/components/add-from-playlist-modal.tsx`, replace line 65:

```typescript
// BEFORE:
.get(`/v1/playlist/${selectedPlaylistId}/items?take=100&skip=0`)

// AFTER:
.get(`/v1/playlist/${selectedPlaylistId}/items?take=500&skip=0`)
```

**Step 2: Run verification**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run test && pnpm run type-check && pnpm run lint`

Expected: All pass.

**Step 3: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/components/add-from-playlist-modal.tsx
git commit -m "fix: increase playlist items fetch limit from 100 to 500"
```

---

## Summary of All File Changes

| File | Tasks | Changes |
|------|-------|---------|
| `src/stores/studio.store.ts` | 1, 4 | Atomic `setActiveTab`, add `clearSelectedForUprez` |
| `src/stores/studio.store.test.ts` | 1, 4 | Tests for atomicity and clearSelectedForUprez |
| `src/components/pages/studio/components/images-tab.tsx` | 2 | try/finally around isGenerating |
| `src/components/pages/studio/hooks/useStudioJobProgress.ts` | 3, 6, 7 | Badge dedup, diff-based rooms, fetch URL on COMPLETED |
| `src/components/pages/studio/hooks/useBatchSubmit.ts` | 5 | Null playlist UUID guard |
| `src/components/pages/studio/components/results-tab.tsx` | 4 | clearSelectedForUprez call, uprezCount filter fix |
| `src/components/pages/studio/components/generate-tab.tsx` | 8 | Filter uprez from "already submitted" count |
| `src/components/pages/studio/components/add-from-playlist-modal.tsx` | 10 | Bump items limit to 500 |

## Known Deferred Issues (Medium Severity)

These were identified by reviewers but deferred from this plan:

- **Silent `.catch(() => {})` everywhere** — no user error feedback. Needs error toast/banner system.
- **No i18n** — all strings hardcoded English. Project convention violation but MVP scope.
- **Playlist create not debounced** — rapid clicks create duplicates. Minor UX issue.
- **Completed job thumbnails lost on reload** — `previewFrame` stripped on persist, no final URL stored on job. Needs job-level URL field.
- **`createComboKey` vs `BATCH_IDENTIFIER` format mismatch** — frontend dedup uses `action.id`, BATCH_IDENTIFIER uses prompt hash. Works within a session; breaks after reset+reimport. Acceptable for MVP.
