# Phase 0: Keyframe Strip — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the keyframe strip UI surface for the flow builder — add, delete, reorder, upload, and loop keyframes in a horizontal strip. No generation yet (Phase 1).

**Architecture:** New Zustand store (`flow.store.ts`) manages flow state independently from the batch store. A mode toggle (`Flow / Batch`) on the studio page switches between the new flow builder and the existing 4-tab batch workflow. The keyframe strip uses `@dnd-kit/sortable` for drag-and-drop reorder and the existing keyframe CRUD API for persistence.

**Tech Stack:** React 18, TypeScript (strict), Zustand (persisted), styled-components, @dnd-kit/core + @dnd-kit/sortable, existing keyframe API hooks, Vitest.

**Spec:** `docs/superpowers/specs/2026-05-12-studio-roadmap-design.md` — Phase 0 section.

**Working directory:** `frontend/` (all paths relative to `frontend/src/` unless noted)

---

### Task 1: Flow Types

**Files:**
- Create: `src/types/flow.types.ts`

- [ ] **Step 1: Create the flow types file**

```typescript
// src/types/flow.types.ts

export type StudioMode = "flow" | "batch";

export interface FlowKeyframe {
  id: string; // local UUID for drag/drop identity
  keyframeUuid: string; // backend Keyframe.uuid
  imageUrl: string; // presigned URL for display
  name: string; // display name
  isLoopKeyframe?: boolean; // true for auto-generated loop frame
}

export interface FlowState {
  keyframes: FlowKeyframe[]; // ordered list (excludes loop keyframe — derived)
  loop: boolean;
}
```

- [ ] **Step 2: Verify types compile**

Run: `pnpm run type-check`
Expected: PASS — no type errors

- [ ] **Step 3: Commit**

```bash
git add src/types/flow.types.ts
git commit -m "feat(flow): add FlowKeyframe and FlowState types"
```

---

### Task 1b: Flow Design Tokens

**Files:**
- Create: `src/constants/flow-theme.constants.ts`

The flow builder introduces a new design language matching the studio roadmap deck (slides 4, 6, 8, 10). These tokens are used by all flow builder styled components. They intentionally differ from the existing app theme (`PrimaryTheme`) — the flow builder is a new UI surface with its own visual identity (gold accent, refined backgrounds, elevated contrast).

- [ ] **Step 1: Create the flow design tokens file**

```typescript
// src/constants/flow-theme.constants.ts

/**
 * Flow Builder design tokens.
 * Matches the studio roadmap deck (2026-05-13-jay-studio-roadmap-v2.html).
 * Used by all flow-builder styled components.
 */
export const FLOW = {
  // Backgrounds (darkest → lightest)
  bg: "#0c0c0e",
  bgCard: "#161619",
  bgElevated: "#1e1e22",
  bgInput: "#26262b",

  // Borders
  border: "#2a2a30",
  borderHover: "#3a3a42",

  // Text
  text: "#e8e6e3",
  textDim: "#8a8890",
  textMuted: "#5a5860",

  // Accent (gold)
  accent: "#d4a853",
  accentDim: "rgba(212, 168, 83, 0.15)",
  accentGlow: "rgba(212, 168, 83, 0.08)",

  // Status
  success: "#4ade80",
  processing: "#60a5fa",
  queued: "#6b7280",

  // Radii
  radius: "12px",
  radiusSm: "8px",

  // Typography
  fontFamily: "'DM Sans', sans-serif",
} as const;
```

- [ ] **Step 2: Verify types compile**

Run: `pnpm run type-check`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add src/constants/flow-theme.constants.ts
git commit -m "feat(flow): add flow builder design tokens matching deck"
```

---

### Task 2: Flow Store

**Files:**
- Create: `src/stores/flow.store.ts`
- Create: `src/stores/flow.store.test.ts`

- [ ] **Step 1: Write failing tests for the flow store**

```typescript
// src/stores/flow.store.test.ts
import { describe, it, expect, beforeEach, beforeAll } from "vitest";

// Polyfill localStorage for Node environment (must run before store import)
beforeAll(() => {
  const store: Record<string, string> = {};
  globalThis.localStorage = {
    getItem: (key: string) => store[key] ?? null,
    setItem: (key: string, value: string) => {
      store[key] = value;
    },
    removeItem: (key: string) => {
      delete store[key];
    },
    clear: () => {
      Object.keys(store).forEach((k) => delete store[k]);
    },
    get length() {
      return Object.keys(store).length;
    },
    key: (index: number) => Object.keys(store)[index] ?? null,
  };
});

// Dynamic import after localStorage is set up (persist middleware needs it)
const { useFlowStore } = await import("./flow.store");

// Reset store between tests
beforeEach(() => {
  useFlowStore.getState().resetFlow();
});

describe("flow store", () => {
  describe("keyframes", () => {
    it("starts with empty keyframes", () => {
      expect(useFlowStore.getState().keyframes).toEqual([]);
    });

    it("adds a keyframe", () => {
      useFlowStore.getState().addKeyframe({
        id: "kf-1",
        keyframeUuid: "uuid-1",
        imageUrl: "https://example.com/img.jpg",
        name: "nebula",
      });
      expect(useFlowStore.getState().keyframes).toHaveLength(1);
      expect(useFlowStore.getState().keyframes[0].name).toBe("nebula");
    });

    it("removes a keyframe by id", () => {
      const store = useFlowStore.getState();
      store.addKeyframe({
        id: "kf-1",
        keyframeUuid: "uuid-1",
        imageUrl: "https://example.com/1.jpg",
        name: "nebula",
      });
      store.addKeyframe({
        id: "kf-2",
        keyframeUuid: "uuid-2",
        imageUrl: "https://example.com/2.jpg",
        name: "crystal",
      });
      useFlowStore.getState().removeKeyframe("kf-1");
      const kfs = useFlowStore.getState().keyframes;
      expect(kfs).toHaveLength(1);
      expect(kfs[0].id).toBe("kf-2");
    });

    it("does not remove loop keyframes", () => {
      const store = useFlowStore.getState();
      store.addKeyframe({
        id: "kf-1",
        keyframeUuid: "uuid-1",
        imageUrl: "https://example.com/1.jpg",
        name: "nebula",
        isLoopKeyframe: true,
      });
      useFlowStore.getState().removeKeyframe("kf-1");
      expect(useFlowStore.getState().keyframes).toHaveLength(1);
    });

    it("reorders keyframes", () => {
      const store = useFlowStore.getState();
      store.addKeyframe({
        id: "kf-1",
        keyframeUuid: "uuid-1",
        imageUrl: "https://example.com/1.jpg",
        name: "first",
      });
      store.addKeyframe({
        id: "kf-2",
        keyframeUuid: "uuid-2",
        imageUrl: "https://example.com/2.jpg",
        name: "second",
      });
      store.addKeyframe({
        id: "kf-3",
        keyframeUuid: "uuid-3",
        imageUrl: "https://example.com/3.jpg",
        name: "third",
      });
      // Move kf-3 to position 0
      useFlowStore.getState().reorderKeyframes(["kf-3", "kf-1", "kf-2"]);
      const ids = useFlowStore.getState().keyframes.map((k) => k.id);
      expect(ids).toEqual(["kf-3", "kf-1", "kf-2"]);
    });
  });

  describe("loop", () => {
    it("starts with loop disabled", () => {
      expect(useFlowStore.getState().loop).toBe(false);
    });

    it("toggles loop", () => {
      useFlowStore.getState().setLoop(true);
      expect(useFlowStore.getState().loop).toBe(true);
    });
  });

  describe("derived: keyframesWithLoop", () => {
    it("returns keyframes as-is when loop is off", () => {
      const store = useFlowStore.getState();
      store.addKeyframe({
        id: "kf-1",
        keyframeUuid: "uuid-1",
        imageUrl: "https://example.com/1.jpg",
        name: "nebula",
      });
      store.addKeyframe({
        id: "kf-2",
        keyframeUuid: "uuid-2",
        imageUrl: "https://example.com/2.jpg",
        name: "crystal",
      });
      const result = useFlowStore.getState().keyframesWithLoop();
      expect(result).toHaveLength(2);
      expect(result.every((k) => !k.isLoopKeyframe)).toBe(true);
    });

    it("appends loop keyframe mirroring first when loop is on", () => {
      const store = useFlowStore.getState();
      store.addKeyframe({
        id: "kf-1",
        keyframeUuid: "uuid-1",
        imageUrl: "https://example.com/1.jpg",
        name: "nebula",
      });
      store.addKeyframe({
        id: "kf-2",
        keyframeUuid: "uuid-2",
        imageUrl: "https://example.com/2.jpg",
        name: "crystal",
      });
      store.setLoop(true);
      const result = useFlowStore.getState().keyframesWithLoop();
      expect(result).toHaveLength(3);
      expect(result[2].isLoopKeyframe).toBe(true);
      expect(result[2].keyframeUuid).toBe("uuid-1");
      expect(result[2].imageUrl).toBe("https://example.com/1.jpg");
      expect(result[2].name).toBe("nebula");
    });

    it("returns empty when no keyframes even with loop on", () => {
      useFlowStore.getState().setLoop(true);
      expect(useFlowStore.getState().keyframesWithLoop()).toEqual([]);
    });

    it("returns single keyframe without loop frame when only one keyframe", () => {
      const store = useFlowStore.getState();
      store.addKeyframe({
        id: "kf-1",
        keyframeUuid: "uuid-1",
        imageUrl: "https://example.com/1.jpg",
        name: "nebula",
      });
      store.setLoop(true);
      const result = useFlowStore.getState().keyframesWithLoop();
      // Loop needs at least 2 keyframes to make sense (otherwise loop frame = first frame with nothing between)
      expect(result).toHaveLength(1);
    });
  });

  describe("resetFlow", () => {
    it("resets to initial state", () => {
      const store = useFlowStore.getState();
      store.addKeyframe({
        id: "kf-1",
        keyframeUuid: "uuid-1",
        imageUrl: "https://example.com/1.jpg",
        name: "nebula",
      });
      store.setLoop(true);
      store.resetFlow();
      expect(useFlowStore.getState().keyframes).toEqual([]);
      expect(useFlowStore.getState().loop).toBe(false);
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm run test -- src/stores/flow.store.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement the flow store**

```typescript
// src/stores/flow.store.ts
import { create } from "zustand";
import { persist } from "zustand/middleware";
import type { FlowKeyframe } from "@/types/flow.types";

const LOOP_KEYFRAME_ID = "__loop__";

type FlowStoreState = {
  keyframes: FlowKeyframe[];
  loop: boolean;

  addKeyframe: (keyframe: FlowKeyframe) => void;
  removeKeyframe: (id: string) => void;
  reorderKeyframes: (orderedIds: string[]) => void;
  setLoop: (loop: boolean) => void;
  keyframesWithLoop: () => FlowKeyframe[];
  resetFlow: () => void;
};

const initialState = {
  keyframes: [] as FlowKeyframe[],
  loop: false,
};

export const useFlowStore = create<FlowStoreState>()(
  persist(
    (set, get) => ({
      ...initialState,

      addKeyframe: (keyframe) =>
        set((state) => ({
          keyframes: [...state.keyframes, keyframe],
        })),

      removeKeyframe: (id) =>
        set((state) => ({
          keyframes: state.keyframes.filter(
            (kf) => kf.id !== id || kf.isLoopKeyframe,
          ),
        })),

      reorderKeyframes: (orderedIds) =>
        set((state) => {
          const byId = new Map(state.keyframes.map((kf) => [kf.id, kf]));
          const reordered = orderedIds
            .map((id) => byId.get(id))
            .filter(Boolean) as FlowKeyframe[];
          return { keyframes: reordered };
        }),

      setLoop: (loop) => set({ loop }),

      keyframesWithLoop: () => {
        const { keyframes, loop } = get();
        if (!loop || keyframes.length < 2) return [...keyframes];
        const first = keyframes[0];
        const loopKf: FlowKeyframe = {
          id: LOOP_KEYFRAME_ID,
          keyframeUuid: first.keyframeUuid,
          imageUrl: first.imageUrl,
          name: first.name,
          isLoopKeyframe: true,
        };
        return [...keyframes, loopKf];
      },

      resetFlow: () => set(initialState),
    }),
    {
      name: "flow-session",
      version: 1,
    },
  ),
);
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm run test -- src/stores/flow.store.test.ts`
Expected: All 9 tests PASS

- [ ] **Step 5: Verify types compile**

Run: `pnpm run type-check`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add src/stores/flow.store.ts src/stores/flow.store.test.ts
git commit -m "feat(flow): add flow store with keyframe CRUD, reorder, loop"
```

---

### Task 3: Install dnd-kit

**Files:**
- Modify: `package.json`

- [ ] **Step 1: Install @dnd-kit packages**

Run: `pnpm add @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities`

- [ ] **Step 2: Verify install and types compile**

Run: `pnpm run type-check`
Expected: PASS

- [ ] **Step 3: Commit**

```bash
git add package.json pnpm-lock.yaml
git commit -m "chore: add @dnd-kit/core, @dnd-kit/sortable, @dnd-kit/utilities"
```

---

### Task 4: Studio Mode Toggle

**Files:**
- Create: `src/stores/studio-mode.store.ts`
- Modify: `src/components/pages/studio/studio.page.tsx`
- Modify: `src/components/pages/studio/studio.page.styled.tsx`

- [ ] **Step 1: Create the studio mode store**

A tiny store for the mode toggle. Separate from both flow and batch stores so they don't depend on each other.

```typescript
// src/stores/studio-mode.store.ts
import { create } from "zustand";
import { persist } from "zustand/middleware";
import type { StudioMode } from "@/types/flow.types";

type StudioModeState = {
  mode: StudioMode;
  setMode: (mode: StudioMode) => void;
};

export const useStudioModeStore = create<StudioModeState>()(
  persist(
    (set) => ({
      mode: "flow" as StudioMode,
      setMode: (mode) => set({ mode }),
    }),
    { name: "studio-mode", version: 1 },
  ),
);
```

- [ ] **Step 2: Add mode toggle styled components**

Append to `studio.page.styled.tsx`:

```typescript
// Add to existing studio.page.styled.tsx
import { FLOW } from "@/constants/flow-theme.constants";

export const ModeToggle = styled.div`
  display: flex;
  background: ${FLOW.bg};
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

- [ ] **Step 3: Update StudioPage to show mode toggle and conditionally render**

```typescript
// src/components/pages/studio/studio.page.tsx
import React, { lazy, Suspense } from "react";
import { useStudioStore } from "@/stores/studio.store";
import { useStudioModeStore } from "@/stores/studio-mode.store";
import { StudioTabs } from "./components/studio-tabs";
import { useStudioJobProgress } from "./hooks/useStudioJobProgress";
import {
  StudioContainer,
  StudioHeader,
  StudioTitle,
  NewSessionButton,
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
  import("./components/flow-builder").then((m) => ({ default: m.FlowBuilder })),
);

export const StudioPage: React.FC = () => {
  const mode = useStudioModeStore((s) => s.mode);
  const setMode = useStudioModeStore((s) => s.setMode);

  const activeTab = useStudioStore((s) => s.activeTab);
  const resetSession = useStudioStore((s) => s.resetSession);
  const hasContent = useStudioStore(
    (s) => s.images.length > 0 || s.actions.length > 0 || s.jobs.length > 0,
  );
  useStudioJobProgress();

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
    <StudioContainer>
      <StudioHeader>
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
        {mode === "batch" && hasContent && (
          <NewSessionButton onClick={handleNewSession}>
            New Session
          </NewSessionButton>
        )}
      </StudioHeader>

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
    </StudioContainer>
  );
};
```

- [ ] **Step 4: Create a placeholder FlowBuilder component**

```typescript
// src/components/pages/studio/components/flow-builder.tsx
import React from "react";
import styled from "styled-components";
import { FLOW } from "@/constants/flow-theme.constants";

const Container = styled.div`
  background: ${FLOW.bgCard};
  border: 1px solid ${FLOW.border};
  border-radius: 16px;
  text-align: center;
  padding: 4rem 2rem;
  color: ${FLOW.textDim};
  font-family: ${FLOW.fontFamily};
`;

export const FlowBuilder: React.FC = () => {
  return <Container>Flow Builder — coming soon</Container>;
};
```

- [ ] **Step 5: Verify it compiles and renders**

Run: `pnpm run type-check && pnpm run build`
Expected: PASS — no type or build errors

- [ ] **Step 6: Commit**

```bash
git add src/stores/studio-mode.store.ts src/components/pages/studio/studio.page.tsx src/components/pages/studio/studio.page.styled.tsx src/components/pages/studio/components/flow-builder.tsx
git commit -m "feat(flow): add Flow/Batch mode toggle to studio page"
```

---

### Task 5: Keyframe Card Component

**Files:**
- Create: `src/components/pages/studio/components/keyframe-card.tsx`
- Create: `src/components/pages/studio/components/keyframe-card.styled.tsx`

- [ ] **Step 1: Create styled components for keyframe card**

```typescript
// src/components/pages/studio/components/keyframe-card.styled.tsx
import styled, { css } from "styled-components";
import { FLOW } from "@/constants/flow-theme.constants";

export const CardWrapper = styled.div<{ $loop?: boolean; $isDragging?: boolean }>`
  flex-shrink: 0;
  width: 140px;
  height: 100px;
  border-radius: ${FLOW.radiusSm};
  overflow: hidden;
  border: 2px solid ${FLOW.border};
  position: relative;
  cursor: grab;
  transition: border-color 0.2s, opacity 0.2s, transform 0.2s;

  ${(props) =>
    props.$loop &&
    css`
      opacity: 0.5;
      cursor: default;
      border-style: dashed;
    `}

  ${(props) =>
    props.$isDragging &&
    css`
      opacity: 0.4;
      transform: scale(0.95);
    `}

  &:hover {
    border-color: ${(props) => (props.$loop ? FLOW.border : FLOW.borderHover)};
  }
`;

export const CardImage = styled.img`
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
`;

export const CardPlaceholder = styled.div`
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 2px;
  background: linear-gradient(135deg, #1a1520, #1d1825);
  font-size: 11px;
  color: ${FLOW.textMuted};
`;

export const CardLabel = styled.div`
  position: absolute;
  bottom: 6px;
  left: 8px;
  font-size: 10px;
  font-weight: 600;
  color: #fff;
  background: rgba(0, 0, 0, 0.6);
  padding: 2px 6px;
  border-radius: 4px;
  backdrop-filter: blur(4px);
`;

export const LoopBadge = styled.span`
  font-size: 9px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  color: ${FLOW.accent};
  opacity: 0.7;
`;

export const DeleteButton = styled.button`
  position: absolute;
  top: 4px;
  right: 4px;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: rgba(0, 0, 0, 0.7);
  color: ${FLOW.textMuted};
  font-size: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  border: none;
  cursor: pointer;
  opacity: 0;
  transition: opacity 0.2s;

  ${CardWrapper}:hover & {
    opacity: 1;
  }
`;
```

- [ ] **Step 2: Create the KeyframeCard component**

```typescript
// src/components/pages/studio/components/keyframe-card.tsx
import React from "react";
import { useSortable } from "@dnd-kit/sortable";
import { CSS } from "@dnd-kit/utilities";
import type { FlowKeyframe } from "@/types/flow.types";
import {
  CardWrapper,
  CardImage,
  CardPlaceholder,
  CardLabel,
  LoopBadge,
  DeleteButton,
} from "./keyframe-card.styled";

interface Props {
  keyframe: FlowKeyframe;
  index: number;
  onDelete?: (id: string) => void;
}

export const KeyframeCard: React.FC<Props> = ({ keyframe, index, onDelete }) => {
  const isLoop = keyframe.isLoopKeyframe ?? false;

  const {
    attributes,
    listeners,
    setNodeRef,
    transform,
    transition,
    isDragging,
  } = useSortable({
    id: keyframe.id,
    disabled: isLoop,
  });

  const style = {
    transform: CSS.Transform.toString(transform),
    transition,
  };

  return (
    <CardWrapper
      ref={setNodeRef}
      style={style}
      $loop={isLoop}
      $isDragging={isDragging}
      {...(isLoop ? {} : { ...attributes, ...listeners })}
    >
      {keyframe.imageUrl ? (
        <CardImage src={keyframe.imageUrl} alt={keyframe.name} />
      ) : (
        <CardPlaceholder>{keyframe.name}</CardPlaceholder>
      )}

      <CardLabel>
        {isLoop ? (
          <>
            {keyframe.name} <LoopBadge>Loop</LoopBadge>
          </>
        ) : (
          `${index + 1}`
        )}
      </CardLabel>

      {!isLoop && onDelete && (
        <DeleteButton
          onClick={(e) => {
            e.stopPropagation();
            onDelete(keyframe.id);
          }}
        >
          &times;
        </DeleteButton>
      )}
    </CardWrapper>
  );
};
```

- [ ] **Step 3: Verify types compile**

Run: `pnpm run type-check`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add src/components/pages/studio/components/keyframe-card.tsx src/components/pages/studio/components/keyframe-card.styled.tsx
git commit -m "feat(flow): add KeyframeCard component with drag, delete, loop support"
```

---

### Task 6: Keyframe Strip & Flow Builder Shell

**Files:**
- Create: `src/components/pages/studio/components/keyframe-strip.tsx`
- Create: `src/components/pages/studio/components/keyframe-strip.styled.tsx`
- Modify: `src/components/pages/studio/components/flow-builder.tsx`

- [ ] **Step 1: Create keyframe strip styled components**

```typescript
// src/components/pages/studio/components/keyframe-strip.styled.tsx
import styled from "styled-components";
import { FLOW } from "@/constants/flow-theme.constants";

export const StripSection = styled.div`
  padding: 28px;
`;

export const SectionLabel = styled.div`
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.12em;
  color: ${FLOW.textMuted};
  margin-bottom: 20px;
  font-family: ${FLOW.fontFamily};
`;

export const StripContainer = styled.div`
  display: flex;
  align-items: center;
  gap: 0;
  overflow-x: auto;
  padding-bottom: 8px;
`;

export const TransitionGap = styled.div`
  flex-shrink: 0;
  width: 64px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 4px;
`;

export const GapLine = styled.div`
  width: 32px;
  border-top: 2px dashed ${FLOW.border};
`;

export const StripControls = styled.div`
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 20px;
`;

export const AddButtons = styled.div`
  display: flex;
  gap: 10px;
`;

export const AddButton = styled.button`
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 14px;
  background: ${FLOW.bgElevated};
  border: 1px solid ${FLOW.border};
  border-radius: ${FLOW.radiusSm};
  color: ${FLOW.textDim};
  font-size: 13px;
  cursor: pointer;
  font-family: ${FLOW.fontFamily};
  transition: all 0.2s;

  &:hover {
    border-color: ${FLOW.borderHover};
    color: ${FLOW.text};
  }
`;

export const AddButtonPlus = styled.span`
  font-size: 15px;
  color: ${FLOW.accent};
`;

export const LoopToggle = styled.label`
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 13px;
  color: ${FLOW.textDim};
  cursor: pointer;
  user-select: none;
  font-family: ${FLOW.fontFamily};
`;

export const LoopCheckbox = styled.input`
  accent-color: ${FLOW.accent};
  cursor: pointer;
`;

export const EmptyState = styled.div`
  text-align: center;
  padding: 48px 16px;
  color: ${FLOW.textDim};
  font-size: 14px;
`;
```

- [ ] **Step 2: Create the KeyframeStrip component**

```typescript
// src/components/pages/studio/components/keyframe-strip.tsx
import React, { useCallback, useState } from "react";
import {
  DndContext,
  closestCenter,
  PointerSensor,
  useSensor,
  useSensors,
  type DragEndEvent,
} from "@dnd-kit/core";
import {
  SortableContext,
  horizontalListSortingStrategy,
} from "@dnd-kit/sortable";
import { useFlowStore } from "@/stores/flow.store";
import { KeyframeCard } from "./keyframe-card";
import {
  StripSection,
  SectionLabel,
  StripContainer,
  TransitionGap,
  GapLine,
  StripControls,
  AddButtons,
  AddButton,
  AddButtonPlus,
  LoopToggle,
  LoopCheckbox,
  EmptyState,
} from "./keyframe-strip.styled";

interface Props {
  onAddGenerate: () => void;
  onAddUpload: () => void;
  onAddFromPlaylist: () => void;
}

export const KeyframeStrip: React.FC<Props> = ({
  onAddGenerate,
  onAddUpload,
  onAddFromPlaylist,
}) => {
  const removeKeyframe = useFlowStore((s) => s.removeKeyframe);
  const reorderKeyframes = useFlowStore((s) => s.reorderKeyframes);
  const setLoop = useFlowStore((s) => s.setLoop);
  const rawKeyframes = useFlowStore((s) => s.keyframes);

  // Reactive selector — subscribes to state changes and recomputes derived list
  const displayKeyframes = useFlowStore((s) => s.keyframesWithLoop());
  // Read loop separately for the checkbox (also ensures re-render on toggle)
  const loop = useFlowStore((s) => s.loop);

  const sensors = useSensors(
    useSensor(PointerSensor, { activationConstraint: { distance: 5 } }),
  );

  const handleDragEnd = useCallback(
    (event: DragEndEvent) => {
      const { active, over } = event;
      if (!over || active.id === over.id) return;

      const oldIndex = rawKeyframes.findIndex((kf) => kf.id === active.id);
      const newIndex = rawKeyframes.findIndex((kf) => kf.id === over.id);
      if (oldIndex === -1 || newIndex === -1) return;

      const newOrder = [...rawKeyframes];
      const [moved] = newOrder.splice(oldIndex, 1);
      newOrder.splice(newIndex, 0, moved);
      reorderKeyframes(newOrder.map((kf) => kf.id));
    },
    [rawKeyframes, reorderKeyframes],
  );

  // Build items with gaps interleaved
  const stripItems: React.ReactNode[] = [];
  const sortableIds = rawKeyframes.map((kf) => kf.id);

  displayKeyframes.forEach((kf, i) => {
    if (i > 0) {
      stripItems.push(
        <TransitionGap key={`gap-${i}`}>
          <GapLine />
        </TransitionGap>,
      );
    }
    stripItems.push(
      <KeyframeCard
        key={kf.id}
        keyframe={kf}
        index={i}
        onDelete={removeKeyframe}
      />,
    );
  });

  return (
    <StripSection>
      <SectionLabel>Keyframes</SectionLabel>

      {displayKeyframes.length === 0 ? (
        <EmptyState>
          Add keyframes to get started. Generate, upload, or import from a
          playlist.
        </EmptyState>
      ) : (
        <DndContext
          sensors={sensors}
          collisionDetection={closestCenter}
          onDragEnd={handleDragEnd}
        >
          <SortableContext
            items={sortableIds}
            strategy={horizontalListSortingStrategy}
          >
            <StripContainer>{stripItems}</StripContainer>
          </SortableContext>
        </DndContext>
      )}

      <StripControls>
        <AddButtons>
          <AddButton onClick={onAddGenerate}>
            <AddButtonPlus>+</AddButtonPlus> Generate
          </AddButton>
          <AddButton onClick={onAddUpload}>
            <AddButtonPlus>+</AddButtonPlus> Upload
          </AddButton>
          <AddButton onClick={onAddFromPlaylist}>
            <AddButtonPlus>+</AddButtonPlus> From Playlist
          </AddButton>
        </AddButtons>

        {rawKeyframes.length >= 2 && (
          <LoopToggle>
            <LoopCheckbox
              type="checkbox"
              checked={loop}
              onChange={(e) => setLoop(e.target.checked)}
            />
            Loop
          </LoopToggle>
        )}
      </StripControls>
    </StripSection>
  );
};
```

- [ ] **Step 3: Update FlowBuilder to render the strip with add handlers**

```typescript
// src/components/pages/studio/components/flow-builder.tsx
import React, { useState, useCallback, useRef } from "react";
import { v4 as uuidv4 } from "uuid";
import styled from "styled-components";
import { useFlowStore } from "@/stores/flow.store";
import { axiosClient } from "@/client/axios.client";
import { ContentType, getRequestHeaders } from "@/constants/auth.constants";
import { FLOW } from "@/constants/flow-theme.constants";
import type { FlowKeyframe } from "@/types/flow.types";
import { KeyframeStrip } from "./keyframe-strip";
import { AddKeyframesFromPlaylistModal } from "./add-keyframes-from-playlist-modal";

const FlowContainer = styled.div`
  background: ${FLOW.bgCard};
  border: 1px solid ${FLOW.border};
  border-radius: 16px;
  overflow: hidden;
`;

export const FlowBuilder: React.FC = () => {
  const addKeyframe = useFlowStore((s) => s.addKeyframe);
  const [showPlaylistModal, setShowPlaylistModal] = useState(false);
  const fileInputRef = useRef<HTMLInputElement>(null);

  const handleAddGenerate = useCallback(() => {
    // TODO: Phase 1 will add inline image gen. For now, stub a placeholder.
    // This will be replaced with an inline image gen panel that uses
    // the same Z Image Turbo / Qwen flow as ImagesTab.
    alert("Inline image generation coming in Phase 1");
  }, []);

  const handleAddUpload = useCallback(() => {
    fileInputRef.current?.click();
  }, []);

  const uploadFiles = useCallback(
    async (files: File[]) => {
      const headers = getRequestHeaders({ contentType: ContentType.json });

      for (const file of files) {
        try {
          const extension = file.name.split(".").pop() ?? "jpg";

          // Create keyframe via backend (needs JSON content-type for body parsing)
          const createRes = await axiosClient.post(
            "/v1/keyframe",
            { name: file.name.replace(/\.[^.]+$/, "") },
            { headers },
          );
          const keyframeUuid = createRes.data.data.keyframe.uuid;

          // Init multipart upload (backend expects { extension }, not fileName/fileType)
          const initRes = await axiosClient.post(
            `/v1/keyframe/${keyframeUuid}/image/init`,
            { extension },
            { headers },
          );
          // Response has `urls` (not presignedUrls) and `uploadId`
          const { uploadId, urls } = initRes.data.data;

          // Upload parts
          const parts: { ETag: string; PartNumber: number }[] = [];
          const chunkSize = 5 * 1024 * 1024; // 5MB
          for (let i = 0; i < urls.length; i++) {
            const start = i * chunkSize;
            const end = Math.min(start + chunkSize, file.size);
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

          // Complete upload (backend expects { extension, parts, uploadId })
          await axiosClient.post(
            `/v1/keyframe/${keyframeUuid}/image/complete`,
            { extension, parts, uploadId },
            { headers },
          );

          // Fetch the created keyframe to get the image URL
          const kfRes = await axiosClient.get(`/v1/keyframe/${keyframeUuid}`);
          const kfData = kfRes.data.data.keyframe;

          addKeyframe({
            id: uuidv4(),
            keyframeUuid: kfData.uuid,
            imageUrl: kfData.image,
            name: kfData.name,
          });
        } catch (err) {
          console.error("Failed to upload keyframe image:", err);
        }
      }
    },
    [addKeyframe],
  );

  const handleFileSelected = useCallback(
    async (e: React.ChangeEvent<HTMLInputElement>) => {
      const files = e.target.files;
      if (!files) return;
      await uploadFiles(Array.from(files));
      if (fileInputRef.current) fileInputRef.current.value = "";
    },
    [uploadFiles],
  );

  const handleAddFromPlaylist = useCallback(() => {
    setShowPlaylistModal(true);
  }, []);

  return (
    <FlowContainer>
      <KeyframeStrip
        onAddGenerate={handleAddGenerate}
        onAddUpload={handleAddUpload}
        onAddFromPlaylist={handleAddFromPlaylist}
      />

      <input
        ref={fileInputRef}
        type="file"
        accept="image/jpeg,image/png,image/webp"
        multiple
        style={{ display: "none" }}
        onChange={handleFileSelected}
      />

      {showPlaylistModal && (
        <AddKeyframesFromPlaylistModal onClose={() => setShowPlaylistModal(false)} />
      )}
    </FlowContainer>
  );
};
```

> **Note:** The `AddFromPlaylistModal` currently adds items to the batch store (`useStudioStore`). In Task 7 we'll create a flow-specific version that adds keyframes to the flow store instead.

- [ ] **Step 4: Verify types compile and build succeeds**

Run: `pnpm run type-check && pnpm run build`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add src/components/pages/studio/components/keyframe-strip.tsx src/components/pages/studio/components/keyframe-strip.styled.tsx src/components/pages/studio/components/flow-builder.tsx
git commit -m "feat(flow): add KeyframeStrip with drag-and-drop reorder and loop toggle"
```

---

### Task 7: Add-from-Playlist for Flow Mode

**Files:**
- Create: `src/components/pages/studio/components/add-keyframes-from-playlist-modal.tsx`

The existing `AddFromPlaylistModal` is tightly coupled to the batch store (`useStudioStore.addImage`). Rather than modifying it and risking batch mode regressions, create a flow-specific variant that adds `FlowKeyframe` entries.

- [ ] **Step 0: Verify the playlist keyframes endpoint exists**

The modal fetches `/v1/playlist/:uuid/keyframes`. Verify this endpoint exists in the backend by checking:
```bash
cd ../backend && grep -r "keyframes" src/routes/ src/controllers/ --include="*.ts" | grep -i playlist
```
The existing `usePlaylistKeyframes` hook in `frontend/src/api/playlist/query/usePlaylistKeyframes.ts` already calls this endpoint and returns `{ keyframes: PlaylistKeyframe[], totalCount }`, confirming it exists. If the endpoint is not found, fall back to using the `/v1/playlist/:uuid/items` endpoint and filter for keyframe-type items.

- [ ] **Step 1: Create the flow-specific playlist modal**

```typescript
// src/components/pages/studio/components/add-keyframes-from-playlist-modal.tsx
import React, { useEffect, useState, useCallback } from "react";
import { v4 as uuidv4 } from "uuid";
import { axiosClient } from "@/client/axios.client";
import { ContentType, getRequestHeaders } from "@/constants/auth.constants";
import { useFlowStore } from "@/stores/flow.store";
import { useUserPlaylists } from "../hooks/useUserPlaylists";
import type { PlaylistKeyframe } from "@/types/playlist.types";
import { StyledSelect, NavButton, SecondaryNavButton, ImageThumbnail } from "./images-tab.styled";
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

export const AddKeyframesFromPlaylistModal: React.FC<Props> = ({ onClose }) => {
  const addKeyframe = useFlowStore((s) => s.addKeyframe);
  const existingKeyframes = useFlowStore((s) => s.keyframes);
  const { playlists } = useUserPlaylists();
  const [selectedPlaylistId, setSelectedPlaylistId] = useState("");
  const [playlistKeyframes, setPlaylistKeyframes] = useState<PlaylistKeyframe[]>([]);
  const [selectedUuids, setSelectedUuids] = useState<Set<string>>(new Set());
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!selectedPlaylistId) {
      setPlaylistKeyframes([]);
      return;
    }
    setLoading(true);
    axiosClient
      .get(`/v1/playlist/${selectedPlaylistId}/keyframes`, {
        headers: getRequestHeaders({ contentType: ContentType.json }),
      })
      .then((res) => {
        // Response shape: { data: { keyframes: PlaylistKeyframe[] } }
        // Each PlaylistKeyframe has nested .keyframe with uuid/name/image
        const items: PlaylistKeyframe[] = res.data.data?.keyframes ?? [];
        setPlaylistKeyframes(items);
      })
      .catch(() => setPlaylistKeyframes([]))
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
    const existingUuids = new Set(existingKeyframes.map((kf) => kf.keyframeUuid));
    for (const item of playlistKeyframes) {
      const kf = item.keyframe;
      if (!kf) continue;
      if (selectedUuids.has(kf.uuid) && !existingUuids.has(kf.uuid)) {
        addKeyframe({
          id: uuidv4(),
          keyframeUuid: kf.uuid,
          imageUrl: kf.image,
          name: kf.name,
        });
      }
    }
    onClose();
  }, [playlistKeyframes, selectedUuids, existingKeyframes, addKeyframe, onClose]);

  // Filter to items that have a valid keyframe
  const validItems = playlistKeyframes.filter((pk) => pk.keyframe);

  return (
    <ModalOverlay onClick={onClose}>
      <ModalContent onClick={(e) => e.stopPropagation()}>
        <ModalHeader>
          <ModalTitle>Add Keyframes from Playlist</ModalTitle>
          <CloseButton onClick={onClose}>&times;</CloseButton>
        </ModalHeader>
        <ModalBody>
          <StyledSelect
            value={selectedPlaylistId}
            onChange={(e) => setSelectedPlaylistId(e.target.value)}
          >
            <option value="">Select a playlist...</option>
            {playlists.map((pl) => (
              <option key={pl.uuid} value={pl.uuid}>
                {pl.name}
              </option>
            ))}
          </StyledSelect>

          {loading && <p style={{ color: "#999", marginTop: "1rem" }}>Loading...</p>}

          {validItems.length > 0 && (
            <ImageSelectGrid>
              {validItems.map((item) => (
                <ImageSelectCard
                  key={item.keyframe!.uuid}
                  $selected={selectedUuids.has(item.keyframe!.uuid)}
                  onClick={() => toggleSelected(item.keyframe!.uuid)}
                >
                  {/* ImageThumbnail is styled.img — use it directly, not as a wrapper */}
                  <ImageThumbnail
                    src={item.keyframe!.image}
                    alt={item.keyframe!.name}
                  />
                </ImageSelectCard>
              ))}
            </ImageSelectGrid>
          )}
        </ModalBody>
        <ModalFooter>
          <SecondaryNavButton onClick={onClose}>Cancel</SecondaryNavButton>
          <NavButton onClick={handleAdd} disabled={selectedUuids.size === 0}>
            Add {selectedUuids.size > 0 ? `(${selectedUuids.size})` : ""}
          </NavButton>
        </ModalFooter>
      </ModalContent>
    </ModalOverlay>
  );
};
```

- [ ] **Step 2: Verify FlowBuilder already uses the new modal**

The Task 6 FlowBuilder code already imports and renders `AddKeyframesFromPlaylistModal`. No changes needed here — just verify the import is correct:

```typescript
// flow-builder.tsx should already have:
import { AddKeyframesFromPlaylistModal } from "./add-keyframes-from-playlist-modal";
```

- [ ] **Step 3: Verify types compile**

Run: `pnpm run type-check`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add src/components/pages/studio/components/add-keyframes-from-playlist-modal.tsx src/components/pages/studio/components/flow-builder.tsx
git commit -m "feat(flow): add flow-specific Add from Playlist modal"
```

---

### Task 8: Drag-and-Drop File Upload

**Files:**
- Modify: `src/components/pages/studio/components/flow-builder.tsx`
- Create: `src/components/pages/studio/hooks/useFileDropUpload.ts`

- [ ] **Step 1: Create the drag-and-drop upload hook**

```typescript
// src/components/pages/studio/hooks/useFileDropUpload.ts
import { useState, useCallback, type DragEvent } from "react";

interface UseFileDropUploadOptions {
  accept: string[];
  onFiles: (files: File[]) => void;
}

export const useFileDropUpload = ({ accept, onFiles }: UseFileDropUploadOptions) => {
  const [isDragOver, setIsDragOver] = useState(false);

  const handleDragOver = useCallback((e: DragEvent) => {
    e.preventDefault();
    e.stopPropagation();
    setIsDragOver(true);
  }, []);

  const handleDragLeave = useCallback((e: DragEvent) => {
    e.preventDefault();
    e.stopPropagation();
    setIsDragOver(false);
  }, []);

  const handleDrop = useCallback(
    (e: DragEvent) => {
      e.preventDefault();
      e.stopPropagation();
      setIsDragOver(false);

      const files = Array.from(e.dataTransfer.files).filter((f) =>
        accept.some((type) => f.type === type),
      );
      if (files.length > 0) onFiles(files);
    },
    [accept, onFiles],
  );

  return {
    isDragOver,
    dropHandlers: {
      onDragOver: handleDragOver,
      onDragLeave: handleDragLeave,
      onDrop: handleDrop,
    },
  };
};
```

- [ ] **Step 2: Add drop zone styling to FlowBuilder**

In `flow-builder.tsx`, add the drop zone wrapper and hook usage:

In `flow-builder.tsx`, make these changes (the `uploadFiles` function already exists from Task 6 — we just add the drop hook and update the container):

```typescript
// Add import at the top:
import { useFileDropUpload } from "../hooks/useFileDropUpload";

// Replace the FlowContainer styled component:
const FlowContainer = styled.div<{ $dragOver?: boolean }>`
  background: ${FLOW.bgCard};
  border: 1px solid ${FLOW.border};
  border-radius: 16px;
  overflow: hidden;
  position: relative;
  min-height: 200px;
  transition: border-color 0.2s, background-color 0.2s;

  ${(props) =>
    props.$dragOver &&
    `
    border-color: ${FLOW.accent};
    background-color: ${FLOW.accentDim};
  `}
`;

// Inside the component, add the drop hook (uploadFiles already exists from Task 6):
const { isDragOver, dropHandlers } = useFileDropUpload({
  accept: ["image/jpeg", "image/png", "image/webp"],
  onFiles: uploadFiles,
});

// Update the JSX return to pass drop handlers:
return (
  <FlowContainer $dragOver={isDragOver} {...dropHandlers}>
    {/* ... existing content unchanged ... */}
  </FlowContainer>
);
```

- [ ] **Step 3: Verify types compile**

Run: `pnpm run type-check`
Expected: PASS

- [ ] **Step 4: Commit**

```bash
git add src/components/pages/studio/hooks/useFileDropUpload.ts src/components/pages/studio/components/flow-builder.tsx
git commit -m "feat(flow): add drag-and-drop file upload to flow builder"
```

---

### Task 9: Quality Gates & Final Verification

**Files:**
- All files from Tasks 1-8

- [ ] **Step 1: Run all tests**

Run: `pnpm run test`
Expected: All tests PASS (including new flow store tests)

- [ ] **Step 2: Run type check**

Run: `pnpm run type-check`
Expected: PASS

- [ ] **Step 3: Run linter**

Run: `pnpm run lint`
Expected: PASS (or fix any lint issues)

- [ ] **Step 4: Run prettier check**

Run: `pnpm run prettier:check`
Expected: PASS (or run `pnpm run prettier` to fix)

- [ ] **Step 5: Run build**

Run: `pnpm run build`
Expected: PASS — production build succeeds

- [ ] **Step 6: Manual smoke test**

Run: `pnpm run dev` and verify in browser:

1. Studio page shows `[Flow] [Batch (Advanced)]` toggle in header
2. Default mode is "Flow" — shows keyframe strip with empty state message
3. Click "Batch (Advanced)" — switches to existing 4-tab workflow
4. Click "Flow" — switches back to flow builder
5. Click "+ Upload" — file picker opens, select an image, keyframe appears in strip
6. Add 2+ keyframes — loop checkbox appears
7. Check "Loop" — dimmed loop keyframe appears at end, mirrors first keyframe
8. Drag a keyframe to reorder — strip reorders, loop keyframe updates
9. Hover a keyframe — delete X appears; click to remove
10. Drag an image file from desktop onto the flow builder — border highlights, keyframe added on drop
11. Refresh page — flow state persists (localStorage)

- [ ] **Step 7: Commit any fixes from smoke test**

```bash
git add -A
git commit -m "fix(flow): address issues found during smoke test"
```

---

## File Map Summary

| File | Action | Purpose |
|------|--------|---------|
| `src/types/flow.types.ts` | Create | FlowKeyframe, FlowState, StudioMode types |
| `src/constants/flow-theme.constants.ts` | Create | Flow builder design tokens (colors, radii, typography from deck) |
| `src/stores/flow.store.ts` | Create | Zustand store for flow state (keyframes, loop) |
| `src/stores/flow.store.test.ts` | Create | Store unit tests |
| `src/stores/studio-mode.store.ts` | Create | Flow/Batch mode toggle state |
| `src/components/pages/studio/studio.page.tsx` | Modify | Add mode toggle, conditionally render Flow/Batch |
| `src/components/pages/studio/studio.page.styled.tsx` | Modify | Add ModeToggle, ModeButton styled components |
| `src/components/pages/studio/components/flow-builder.tsx` | Create | Flow builder shell with upload handling |
| `src/components/pages/studio/components/keyframe-strip.tsx` | Create | Keyframe strip with DnD, loop, add buttons |
| `src/components/pages/studio/components/keyframe-strip.styled.tsx` | Create | Strip styled components |
| `src/components/pages/studio/components/keyframe-card.tsx` | Create | Individual keyframe card (sortable, delete, loop) |
| `src/components/pages/studio/components/keyframe-card.styled.tsx` | Create | Card styled components |
| `src/components/pages/studio/components/add-keyframes-from-playlist-modal.tsx` | Create | Flow-specific add-from-playlist modal |
| `src/components/pages/studio/hooks/useFileDropUpload.ts` | Create | Drag-and-drop file upload hook |
