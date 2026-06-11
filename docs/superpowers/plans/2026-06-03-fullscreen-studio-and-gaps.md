# Fullscreen Studio + Save to Playlist Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make Studio a fullscreen, chromeless experience (no header/footer) and wire up the "Save to Playlist" button.

**Architecture:** Move the `/studio` route out of `RootElement` into a new top-level route with its own `StudioLayout` wrapper. Add a `SaveToPlaylistModal` to the flow builder action bar using existing playlist API hooks.

**Tech Stack:** React, React Router, styled-components, Zustand, React Query, lucide-react

**Spec:** `docs/superpowers/specs/2026-06-03-fullscreen-studio-and-gaps-design.md`

---

## File Structure

| File | Responsibility | New/Edit |
|------|---------------|----------|
| `src/components/pages/studio/studio.layout.tsx` | Chromeless layout wrapper with providers + GA tracking | New |
| `src/routes/router.tsx` | Move studio to top-level route, add error boundary | Edit |
| `src/components/pages/studio/studio.page.tsx` | Add back arrow to header, remove redundant title | Edit |
| `src/components/pages/studio/studio.page.styled.tsx` | Full viewport container, back button styles | Edit |
| `src/components/pages/studio/components/save-to-playlist-modal.tsx` | Modal: create new or select existing playlist, bulk-add transitions | New |
| `src/components/pages/studio/components/save-to-playlist-modal.styled.tsx` | Modal styles following existing modal pattern | New |
| `src/components/pages/studio/components/flow-action-bar.tsx` | Replace "Coming soon" toast with modal trigger | Edit |

---

### Task 1: Create StudioLayout wrapper

**Files:**
- Create: `src/components/pages/studio/studio.layout.tsx`

- [ ] **Step 1: Create the layout component**

```tsx
// src/components/pages/studio/studio.layout.tsx
import { useEffect } from "react";
import ReactGA from "react-ga4";
import Providers, { withProviders } from "@/providers/providers";
import { StudioPage } from "./studio.page";

const StudioLayout = () => {
  useEffect(() => {
    ReactGA.send({
      hitType: "pageview",
      page: window.location.pathname + window.location.search,
    });
  }, []);

  return <StudioPage />;
};

export const StudioLayoutWithProviders =
  withProviders(...Providers)(StudioLayout);
```

- [ ] **Step 2: Verify file compiles**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx tsc --noEmit --pretty 2>&1 | head -30`
Expected: No errors related to `studio.layout.tsx`

- [ ] **Step 3: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/studio.layout.tsx
git commit -m "feat(studio): add chromeless StudioLayout wrapper"
```

---

### Task 2: Move studio route to top-level in router

**Files:**
- Modify: `src/routes/router.tsx`

- [ ] **Step 1: Add import for StudioLayoutWithProviders**

At the top of `router.tsx`, replace the `StudioPage` import:

Replace:
```tsx
import { StudioPage } from "@/components/pages/studio/studio.page";
```

With:
```tsx
import { StudioLayoutWithProviders } from "@/components/pages/studio/studio.layout";
```

- [ ] **Step 2: Remove studio from RootElement children**

Delete this child route block from the `children` array:

```tsx
      {
        path: ROUTES.STUDIO,
        element: (
          <ProtectedRoute
            allowedRoles={[ROLES.CREATOR_GROUP, ROLES.ADMIN_GROUP]}
          >
            <StudioPage />
          </ProtectedRoute>
        ),
      },
```

- [ ] **Step 3: Add studio as a top-level route**

Add a new entry to the `createBrowserRouter` array, after the `RootElement` route object (as a sibling, not a child):

```tsx
  {
    path: ROUTES.STUDIO,
    element: (
      <ProtectedRoute allowedRoles={[ROLES.CREATOR_GROUP, ROLES.ADMIN_GROUP]}>
        <StudioLayoutWithProviders />
      </ProtectedRoute>
    ),
    errorElement: <NotFoundPageWithProviders />,
  },
```

The final router array should look like:

```tsx
export const router = createBrowserRouter([
  {
    path: ROUTES.ROOT,
    element: <RootElementWithProviders />,
    errorElement: <NotFoundPageWithProviders />,
    children: [
      // ... all existing children EXCEPT studio ...
    ],
  },
  {
    path: ROUTES.STUDIO,
    element: (
      <ProtectedRoute allowedRoles={[ROLES.CREATOR_GROUP, ROLES.ADMIN_GROUP]}>
        <StudioLayoutWithProviders />
      </ProtectedRoute>
    ),
    errorElement: <NotFoundPageWithProviders />,
  },
]);
```

- [ ] **Step 4: Verify compilation**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx tsc --noEmit --pretty 2>&1 | head -30`
Expected: No errors

- [ ] **Step 5: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/routes/router.tsx
git commit -m "feat(studio): move studio to top-level chromeless route"
```

---

### Task 3: Update StudioPage header and container for fullscreen

**Files:**
- Modify: `src/components/pages/studio/studio.page.styled.tsx`
- Modify: `src/components/pages/studio/studio.page.tsx`

- [ ] **Step 1: Update styled components for fullscreen + back button**

Replace the entire contents of `studio.page.styled.tsx`:

```tsx
import styled from "styled-components";
import { FLOW } from "@/constants/flow-theme.constants";

export const StudioContainer = styled.div<{ $dragOver?: boolean }>`
  height: 100vh;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  background: ${FLOW.bg};
  transition: outline-color 0.2s;
  outline: 2px solid transparent;
  outline-offset: -2px;

  ${(props) =>
    props.$dragOver &&
    `
    outline-color: ${FLOW.accent};
  `}
`;

export const StudioHeader = styled.div`
  display: flex;
  align-items: center;
  padding: 12px 20px;
  border-bottom: 1px solid ${FLOW.border};
  gap: 16px;
  flex-shrink: 0;
`;

export const BackButton = styled.button`
  display: flex;
  align-items: center;
  justify-content: center;
  width: 32px;
  height: 32px;
  border-radius: ${FLOW.radiusSm};
  border: 1px solid ${FLOW.border};
  background: transparent;
  color: ${FLOW.textDim};
  cursor: pointer;
  transition: all 0.2s;
  flex-shrink: 0;

  &:hover {
    border-color: ${FLOW.borderHover};
    color: ${FLOW.text};
  }
`;

export const StudioTitle = styled.h1`
  font-size: 1.125rem;
  font-weight: 600;
  color: ${FLOW.text};
  font-family: ${FLOW.fontFamily};
`;

export const HeaderSpacer = styled.div`
  flex: 1;
`;

export const NewSessionButton = styled.button`
  background: transparent;
  border: 1px solid ${FLOW.border};
  color: ${FLOW.textDim};
  padding: 0.375rem 0.75rem;
  border-radius: 6px;
  font-size: 0.8125rem;
  font-family: ${FLOW.fontFamily};
  cursor: pointer;
  transition: all 0.2s;

  &:hover {
    border-color: ${FLOW.borderHover};
    color: ${FLOW.text};
  }
`;

export const StudioBody = styled.div`
  flex: 1;
  overflow-y: auto;
  max-width: 1200px;
  width: 100%;
  margin: 0 auto;
  padding: 1.5rem;
`;

export const ModeToggle = styled.div`
  display: flex;
  background: ${FLOW.bg};
  border: 1px solid ${FLOW.border};
  border-radius: 8px;
  padding: 3px;
  gap: 2px;
`;

export const ModeButton = styled.button<{ $active: boolean }>`
  padding: 6px 16px;
  border-radius: 6px;
  font-size: 13px;
  font-family: ${FLOW.fontFamily};
  color: ${(props) => (props.$active ? FLOW.text : FLOW.textMuted)};
  background: ${(props) => (props.$active ? FLOW.bgElevated : "transparent")};
  border: none;
  cursor: pointer;
  transition: all 0.2s;

  &:hover {
    color: ${FLOW.text};
  }
`;
```

- [ ] **Step 2: Update StudioPage to use back arrow and new layout**

Replace the entire contents of `studio.page.tsx`:

```tsx
import React, { lazy, Suspense, useCallback } from "react";
import { useNavigate } from "react-router-dom";
import { ArrowLeft } from "lucide-react";
import { v4 as uuidv4 } from "uuid";
import Bugsnag from "@bugsnag/js";
import { useStudioStore } from "@/stores/studio.store";
import { useStudioModeStore } from "@/stores/studio-mode.store";
import { useFlowStore } from "@/stores/flow.store";
import { ROUTES } from "@/constants/routes.constants";
import { StudioTabs } from "./components/studio-tabs";
import { useStudioJobProgress } from "./hooks/useStudioJobProgress";
import { useFileDropUpload } from "./hooks/useFileDropUpload";
import { useUploadImageDream } from "@/api/dream/mutation/useUploadImageDream";
import {
  StudioContainer,
  StudioHeader,
  StudioTitle,
  BackButton,
  HeaderSpacer,
  NewSessionButton,
  StudioBody,
  ModeToggle,
  ModeButton,
} from "./studio.page.styled";

const ImagesTab = lazy(() =>
  import("./components/images-tab").then((m) => ({ default: m.ImagesTab })),
);
const ActionsTab = lazy(() =>
  import("./components/actions-tab").then((m) => ({ default: m.ActionsTab })),
);
const GenerateTab = lazy(() =>
  import("./components/generate-tab").then((m) => ({ default: m.GenerateTab })),
);
const ResultsTab = lazy(() =>
  import("./components/results-tab").then((m) => ({ default: m.ResultsTab })),
);
const FlowBuilder = lazy(() =>
  import("./components/flow-builder").then((m) => ({
    default: m.FlowBuilder,
  })),
);

export const StudioPage: React.FC = () => {
  const navigate = useNavigate();
  const mode = useStudioModeStore((s) => s.mode);
  const setMode = useStudioModeStore((s) => s.setMode);

  const activeTab = useStudioStore((s) => s.activeTab);
  const resetSession = useStudioStore((s) => s.resetSession);
  const hasContent = useStudioStore(
    (s) => s.images.length > 0 || s.actions.length > 0 || s.jobs.length > 0,
  );
  useStudioJobProgress();

  const addImage = useStudioStore((s) => s.addImage);
  const updateImage = useStudioStore((s) => s.updateImage);
  const addKeyframe = useFlowStore((s) => s.addKeyframe);
  const uploadDream = useUploadImageDream();

  const handleBack = useCallback(() => {
    if (window.history.length > 1) {
      navigate(-1);
    } else {
      navigate(ROUTES.REMOTE_CONTROL);
    }
  }, [navigate]);

  const handleStudioDrop = useCallback(
    async (files: File[]) => {
      const currentMode = useStudioModeStore.getState().mode;

      for (const file of files) {
        if (currentMode === "batch") {
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
            const result = await uploadDream.mutateAsync({ file });
            updateImage(placeholderUuid, {
              uuid: result.dreamUuid,
              url: result.imageUrl,
              status: "processed",
              name: result.name,
            });
          } catch (err) {
            Bugsnag.notify(err as Error);
            updateImage(placeholderUuid, { status: "failed" });
          } finally {
            URL.revokeObjectURL(blobUrl);
          }
        } else {
          try {
            const result = await uploadDream.mutateAsync({ file });
            addKeyframe({
              id: uuidv4(),
              dreamUuid: result.dreamUuid,
              imageUrl: result.imageUrl,
              name: result.name,
            });
          } catch (err) {
            Bugsnag.notify(err as Error);
          }
        }
      }
    },
    [addImage, updateImage, addKeyframe, uploadDream],
  );

  const { isDragOver, dropHandlers } = useFileDropUpload({
    accept: ["image/jpeg", "image/png", "image/webp"],
    onFiles: handleStudioDrop,
  });

  const handleNewSession = () => {
    if (
      !window.confirm(
        "Start a new session? This will clear all current images, actions, and results.",
      )
    )
      return;
    resetSession();
  };

  return (
    <StudioContainer $dragOver={isDragOver} {...dropHandlers}>
      <StudioHeader>
        <BackButton onClick={handleBack} aria-label="Go back">
          <ArrowLeft size={16} />
        </BackButton>
        <StudioTitle>Studio</StudioTitle>
        <ModeToggle>
          <ModeButton $active={mode === "flow"} onClick={() => setMode("flow")}>
            Flow
          </ModeButton>
          <ModeButton
            $active={mode === "batch"}
            onClick={() => setMode("batch")}
          >
            Batch (Advanced)
          </ModeButton>
        </ModeToggle>
        <HeaderSpacer />
        {mode === "batch" && hasContent && (
          <NewSessionButton onClick={handleNewSession}>
            New Session
          </NewSessionButton>
        )}
      </StudioHeader>

      <StudioBody>
        <Suspense fallback={null}>
          {mode === "flow" && <FlowBuilder />}
          {mode === "batch" && (
            <>
              <StudioTabs />
              {activeTab === "images" && <ImagesTab />}
              {activeTab === "actions" && <ActionsTab />}
              {activeTab === "generate" && <GenerateTab />}
              {activeTab === "results" && <ResultsTab />}
            </>
          )}
        </Suspense>
      </StudioBody>
    </StudioContainer>
  );
};
```

- [ ] **Step 3: Verify compilation**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx tsc --noEmit --pretty 2>&1 | head -30`
Expected: No errors

- [ ] **Step 4: Manual smoke test**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run dev`

Verify:
1. Navigate to `/studio` — should render fullscreen with no header/footer
2. Back arrow in top-left navigates to previous page
3. Flow/Batch toggle works
4. Mode toggle is centered in the header bar
5. Content scrolls within the body area, header stays fixed

- [ ] **Step 5: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/studio.page.tsx src/components/pages/studio/studio.page.styled.tsx
git commit -m "feat(studio): fullscreen layout with back arrow and sticky header"
```

---

### Task 4: Create SaveToPlaylistModal styled components

**Files:**
- Create: `src/components/pages/studio/components/save-to-playlist-modal.styled.tsx`

- [ ] **Step 1: Create modal styles**

These reuse the same pattern as `add-from-playlist-modal.styled.tsx` but with FLOW theme tokens for consistency.

```tsx
// src/components/pages/studio/components/save-to-playlist-modal.styled.tsx
import styled from "styled-components";
import { FLOW } from "@/constants/flow-theme.constants";

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
  background: ${FLOW.bgCard};
  border: 1px solid ${FLOW.border};
  border-radius: 12px;
  width: 90%;
  max-width: 480px;
  max-height: 80vh;
  display: flex;
  flex-direction: column;
`;

export const ModalHeader = styled.div`
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem 1.25rem;
  border-bottom: 1px solid ${FLOW.border};
`;

export const ModalTitle = styled.h3`
  font-size: 1rem;
  font-weight: 600;
  color: ${FLOW.text};
  font-family: ${FLOW.fontFamily};
`;

export const CloseButton = styled.button`
  background: none;
  border: none;
  color: ${FLOW.textDim};
  font-size: 1.25rem;
  cursor: pointer;
  transition: color 0.2s;

  &:hover {
    color: ${FLOW.text};
  }
`;

export const ModalBody = styled.div`
  padding: 1.25rem;
  overflow-y: auto;
  flex: 1;
`;

export const ModeToggleRow = styled.div`
  display: flex;
  background: ${FLOW.bg};
  border: 1px solid ${FLOW.border};
  border-radius: 8px;
  padding: 3px;
  gap: 2px;
  margin-bottom: 1rem;
`;

export const ModeTab = styled.button<{ $active: boolean }>`
  flex: 1;
  padding: 6px 12px;
  border-radius: 6px;
  font-size: 13px;
  font-family: ${FLOW.fontFamily};
  color: ${(p) => (p.$active ? FLOW.text : FLOW.textMuted)};
  background: ${(p) => (p.$active ? FLOW.bgElevated : "transparent")};
  border: none;
  cursor: pointer;
  transition: all 0.2s;

  &:hover {
    color: ${FLOW.text};
  }
`;

export const NameInput = styled.input`
  width: 100%;
  background: ${FLOW.bgInput};
  border: 1px solid ${FLOW.border};
  border-radius: 6px;
  padding: 9px 12px;
  color: ${FLOW.text};
  font-size: 14px;
  font-family: ${FLOW.fontFamily};
  outline: none;
  transition: border-color 0.2s;

  &:focus {
    border-color: ${FLOW.accent};
  }
`;

export const PlaylistList = styled.div`
  display: flex;
  flex-direction: column;
  gap: 4px;
  max-height: 280px;
  overflow-y: auto;
`;

export const PlaylistItem = styled.button<{ $selected: boolean }>`
  display: block;
  width: 100%;
  text-align: left;
  background: ${(p) => (p.$selected ? FLOW.accentDim : "transparent")};
  border: 1px solid ${(p) => (p.$selected ? FLOW.accent : FLOW.border)};
  border-radius: 6px;
  padding: 10px 12px;
  color: ${(p) => (p.$selected ? FLOW.accent : FLOW.textDim)};
  font-size: 13px;
  font-family: ${FLOW.fontFamily};
  cursor: pointer;
  transition: all 0.15s;

  &:hover {
    border-color: ${FLOW.borderHover};
    color: ${FLOW.text};
  }
`;

export const Summary = styled.p`
  font-size: 12px;
  color: ${FLOW.textMuted};
  margin-top: 1rem;
`;

export const ModalFooter = styled.div`
  display: flex;
  justify-content: flex-end;
  align-items: center;
  gap: 10px;
  padding: 1rem 1.25rem;
  border-top: 1px solid ${FLOW.border};
`;

export const CancelButton = styled.button`
  background: transparent;
  border: 1px solid ${FLOW.border};
  color: ${FLOW.textDim};
  padding: 8px 16px;
  border-radius: 6px;
  font-size: 13px;
  font-family: ${FLOW.fontFamily};
  cursor: pointer;
  transition: all 0.2s;

  &:hover {
    border-color: ${FLOW.borderHover};
    color: ${FLOW.text};
  }
`;

export const SaveButton = styled.button`
  background: ${FLOW.accent};
  color: ${FLOW.bg};
  border: none;
  border-radius: 6px;
  font-size: 13px;
  font-weight: 600;
  font-family: ${FLOW.fontFamily};
  padding: 8px 20px;
  cursor: pointer;
  transition: all 0.2s;

  &:hover:not(:disabled) {
    background: #e0b45e;
  }

  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
`;
```

- [ ] **Step 2: Verify compilation**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx tsc --noEmit --pretty 2>&1 | head -20`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/components/save-to-playlist-modal.styled.tsx
git commit -m "feat(flow): add SaveToPlaylistModal styled components"
```

---

### Task 5: Create SaveToPlaylistModal component

**Files:**
- Create: `src/components/pages/studio/components/save-to-playlist-modal.tsx`

- [ ] **Step 1: Create the modal component**

```tsx
// src/components/pages/studio/components/save-to-playlist-modal.tsx
import React, { useState, useCallback } from "react";
import { toast } from "react-toastify";
import { Loader2 } from "lucide-react";
import Bugsnag from "@bugsnag/js";
import { useFlowStore } from "@/stores/flow.store";
import { useShallow } from "zustand/react/shallow";
import { useCreatePlaylist } from "@/api/playlist/mutation/useCreatePlaylist";
import { useAddPlaylistItem } from "@/api/playlist/mutation/useAddPlaylistItem";
import { useUserPlaylists } from "../hooks/useUserPlaylists";
import { ROUTES } from "@/constants/routes.constants";
import {
  ModalOverlay,
  ModalContent,
  ModalHeader,
  ModalTitle,
  CloseButton,
  ModalBody,
  ModeToggleRow,
  ModeTab,
  NameInput,
  PlaylistList,
  PlaylistItem,
  Summary,
  ModalFooter,
  CancelButton,
  SaveButton,
} from "./save-to-playlist-modal.styled";

interface Props {
  onClose: () => void;
}

export const SaveToPlaylistModal: React.FC<Props> = ({ onClose }) => {
  const { transitions } = useFlowStore(
    useShallow((s) => ({ transitions: s.transitions })),
  );

  const completedTransitions = transitions.filter(
    (t) => t.status === "processed" && t.dreamUuid,
  );

  const [mode, setMode] = useState<"new" | "existing">("new");
  const [playlistName, setPlaylistName] = useState(
    `Studio Flow — ${new Date().toISOString().slice(0, 10)}`,
  );
  const [selectedPlaylistId, setSelectedPlaylistId] = useState("");
  const [isSaving, setIsSaving] = useState(false);
  const [progress, setProgress] = useState({ current: 0, total: 0 });

  const { playlists, addPlaylistToCache } = useUserPlaylists();
  const createPlaylist = useCreatePlaylist();
  const addPlaylistItem = useAddPlaylistItem();

  const canSave =
    completedTransitions.length > 0 &&
    (mode === "new" ? playlistName.trim().length > 0 : selectedPlaylistId !== "");

  const handleSave = useCallback(async () => {
    if (!canSave) return;
    setIsSaving(true);
    const total = completedTransitions.length;
    setProgress({ current: 0, total });

    try {
      let playlistUUID: string;
      let finalName: string;

      if (mode === "new") {
        const result = await createPlaylist.mutateAsync({
          name: playlistName.trim(),
        });
        const playlist = result.data?.playlist;
        if (!playlist) throw new Error("No playlist in response");
        playlistUUID = playlist.uuid;
        finalName = playlist.name;
        addPlaylistToCache({ uuid: playlist.uuid, name: playlist.name });
      } else {
        playlistUUID = selectedPlaylistId;
        finalName =
          playlists.find((p) => p.uuid === selectedPlaylistId)?.name ??
          "playlist";
      }

      for (let i = 0; i < completedTransitions.length; i++) {
        setProgress({ current: i + 1, total });
        await addPlaylistItem.mutateAsync({
          playlistUUID,
          values: {
            type: "dream",
            uuid: completedTransitions[i].dreamUuid!,
          },
        });
      }

      toast.success(
        <span>
          Saved {total} transition{total !== 1 ? "s" : ""} to{" "}
          <a
            href={`${ROUTES.VIEW_PLAYLIST}/${playlistUUID}`}
            style={{ color: "inherit", textDecoration: "underline" }}
          >
            {finalName}
          </a>
        </span>,
      );
      onClose();
    } catch (err) {
      Bugsnag.notify(err as Error);
      toast.error("Failed to save — please try again.");
    } finally {
      setIsSaving(false);
    }
  }, [
    canSave,
    mode,
    playlistName,
    selectedPlaylistId,
    completedTransitions,
    createPlaylist,
    addPlaylistItem,
    addPlaylistToCache,
    playlists,
    onClose,
  ]);

  return (
    <ModalOverlay onClick={onClose}>
      <ModalContent onClick={(e) => e.stopPropagation()}>
        <ModalHeader>
          <ModalTitle>Save to Playlist</ModalTitle>
          <CloseButton onClick={onClose}>&times;</CloseButton>
        </ModalHeader>

        <ModalBody>
          <ModeToggleRow>
            <ModeTab $active={mode === "new"} onClick={() => setMode("new")}>
              New Playlist
            </ModeTab>
            <ModeTab
              $active={mode === "existing"}
              onClick={() => setMode("existing")}
            >
              Existing Playlist
            </ModeTab>
          </ModeToggleRow>

          {mode === "new" ? (
            <NameInput
              value={playlistName}
              onChange={(e) => setPlaylistName(e.target.value)}
              placeholder="Playlist name"
              autoFocus
            />
          ) : (
            <PlaylistList>
              {playlists.length === 0 && (
                <Summary>No playlists found. Create a new one instead.</Summary>
              )}
              {playlists.map((pl) => (
                <PlaylistItem
                  key={pl.uuid}
                  $selected={selectedPlaylistId === pl.uuid}
                  onClick={() => setSelectedPlaylistId(pl.uuid)}
                >
                  {pl.name}
                </PlaylistItem>
              ))}
            </PlaylistList>
          )}

          <Summary>
            {isSaving
              ? `Adding ${progress.current} of ${progress.total}...`
              : `Adding ${completedTransitions.length} transition${completedTransitions.length !== 1 ? "s" : ""}`}
          </Summary>
        </ModalBody>

        <ModalFooter>
          <CancelButton onClick={onClose} disabled={isSaving}>
            Cancel
          </CancelButton>
          <SaveButton onClick={handleSave} disabled={!canSave || isSaving}>
            {isSaving ? (
              <Loader2
                size={14}
                strokeWidth={2.4}
                style={{ animation: "spin 1.4s linear infinite" }}
              />
            ) : (
              "Save"
            )}
          </SaveButton>
        </ModalFooter>
      </ModalContent>
    </ModalOverlay>
  );
};
```

- [ ] **Step 2: Verify compilation**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx tsc --noEmit --pretty 2>&1 | head -20`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/components/save-to-playlist-modal.tsx
git commit -m "feat(flow): add SaveToPlaylistModal component"
```

---

### Task 6: Wire SaveToPlaylistModal into FlowActionBar

**Files:**
- Modify: `src/components/pages/studio/components/flow-action-bar.tsx`

- [ ] **Step 1: Add modal state and import**

At the top of `flow-action-bar.tsx`, add the import:

```tsx
import { SaveToPlaylistModal } from "./save-to-playlist-modal";
```

Inside the `FlowActionBar` function, add state after the existing `useState` calls:

```tsx
const [showSaveModal, setShowSaveModal] = useState(false);
```

- [ ] **Step 2: Replace the handleSaveToPlaylist callback**

Replace:
```tsx
  const handleSaveToPlaylist = useCallback(() => {
    toast.info("Coming soon — Save to Playlist will be available in Phase 2");
  }, []);
```

With:
```tsx
  const handleSaveToPlaylist = useCallback(() => {
    setShowSaveModal(true);
  }, []);
```

- [ ] **Step 3: Add modal render before the closing fragment**

After the `</ActionBarContainer>` closing tag (but before the function's `return` closes), wrap the return in a fragment and add the modal:

Replace:
```tsx
  return (
    <ActionBarContainer>
      {/* ... existing content ... */}
    </ActionBarContainer>
  );
```

With:
```tsx
  return (
    <>
      <ActionBarContainer>
        {/* ... existing content ... */}
      </ActionBarContainer>
      {showSaveModal && (
        <SaveToPlaylistModal onClose={() => setShowSaveModal(false)} />
      )}
    </>
  );
```

- [ ] **Step 4: Clean up unused toast import if no other usages remain**

Check if `toast` is still used elsewhere in the file. If not, remove the import:
```tsx
import { toast } from "react-toastify";
```

(Note: `toast` is NOT used elsewhere in this file — the preview toast was already removed in PR #632. Remove the import.)

- [ ] **Step 5: Verify compilation**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx tsc --noEmit --pretty 2>&1 | head -20`
Expected: No errors

- [ ] **Step 6: Manual smoke test**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run dev`

Verify:
1. Generate at least one transition in flow mode
2. "Save to Playlist" button appears in the action bar
3. Clicking it opens the modal with "New Playlist" tab active and pre-filled name
4. Switching to "Existing Playlist" shows the user's playlists
5. Clicking "Save" creates the playlist and adds items with progress feedback
6. Success toast appears with playlist link
7. Modal closes on success

- [ ] **Step 7: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/components/flow-action-bar.tsx
git commit -m "feat(flow): wire SaveToPlaylistModal into action bar"
```

---

### Task 7: Final verification and lint

- [ ] **Step 1: Run type check**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run type-check 2>&1 | tail -10`
Expected: No errors

- [ ] **Step 2: Run lint**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run lint 2>&1 | tail -20`
Expected: No errors (or only pre-existing warnings)

- [ ] **Step 3: Run build**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run build 2>&1 | tail -10`
Expected: Build succeeds

- [ ] **Step 4: Full smoke test**

With dev server running:
1. Navigate to `/studio` directly — fullscreen, no header/footer
2. Click back arrow — navigates to previous page (or `/rc` if no history)
3. Toggle Flow/Batch — both modes render correctly in fullscreen
4. In flow mode: add keyframes, generate transitions, verify Preview All and Uprez All still work
5. Click "Save to Playlist" — modal opens, create new or select existing works
6. Navigate to the saved playlist via the toast link — transitions are there in order

- [ ] **Step 5: Commit any lint fixes if needed**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add -A
git commit -m "fix: lint and type-check cleanup"
```
