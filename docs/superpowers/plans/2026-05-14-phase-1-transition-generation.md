# Phase 1: Transition Generation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Layer transition generation on top of the Phase 0 keyframe strip — users configure global or per-transition settings, generate video segments between keyframe pairs, track progress inline, preview results, and uprez.

**Architecture:** Zustand store extended with transition state + persist v2 migration. New components render below the keyframe strip (settings panel, preview, action bar). Generation uses `axiosClient` directly (not mutation hooks) to support sequential multi-dream creation. Progress tracked via Socket.IO rooms with polling fallback, modeled after `useStudioJobProgress` but reimplemented for the flow store.

**Tech Stack:** React 18, TypeScript, Zustand (persist middleware), styled-components v6, Socket.IO, Axios, Vitest

**Spec:** `docs/superpowers/specs/2026-05-14-phase-1-transition-generation-design.md`

**Base branch:** `feat/phase-0-keyframe-strip` (PR #612 must be merged first, or branch off it)

---

### Task 1: Types — FlowTransition and TransitionStatus

**Files:**
- Modify: `frontend/src/types/flow.types.ts`

- [ ] **Step 1: Write the failing test**

Create a type-only test that imports and uses the new types:

```typescript
// frontend/src/types/__tests__/flow.types.test.ts
import { describe, it, expect } from "vitest";
import type { FlowTransition, TransitionStatus } from "@/types/flow.types";

describe("FlowTransition type", () => {
  it("accepts a valid idle transition", () => {
    const t: FlowTransition = {
      fromKeyframeId: "a",
      toKeyframeId: "b",
      status: "idle",
    };
    expect(t.status).toBe("idle");
  });

  it("accepts a transition with all overrides and generation state", () => {
    const t: FlowTransition = {
      fromKeyframeId: "a",
      toKeyframeId: "b",
      presetOverride: "Camera Basics",
      promptOverride: "zoom in",
      durationOverride: 8,
      modelOverride: "wan-i2v",
      loraOverride: [{ path: "lora.safetensors", scale: 1 }],
      dreamUuid: "dream-123",
      status: "processing",
      progress: 45,
      uprezDreamUuid: "uprez-456",
      uprezStatus: "queue",
      uprezProgress: 0,
    };
    expect(t.progress).toBe(45);
  });

  it("enforces TransitionStatus union", () => {
    const statuses: TransitionStatus[] = [
      "idle",
      "queue",
      "processing",
      "processed",
      "failed",
    ];
    expect(statuses).toHaveLength(5);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd frontend && pnpm vitest run src/types/__tests__/flow.types.test.ts`
Expected: FAIL — `FlowTransition` and `TransitionStatus` not exported from `flow.types`

- [ ] **Step 3: Add types to flow.types.ts**

```typescript
// Add to frontend/src/types/flow.types.ts after FlowKeyframe

import type { VideoModel, LoRAConfig } from "@/types/studio.types";

export type TransitionStatus =
  | "idle"
  | "queue"
  | "processing"
  | "processed"
  | "failed";

export interface FlowTransition {
  fromKeyframeId: string; // FlowKeyframe.id
  toKeyframeId: string; // FlowKeyframe.id

  // Per-transition overrides (undefined = use global)
  presetOverride?: string; // PresetPack name
  promptOverride?: string;
  durationOverride?: number; // seconds
  modelOverride?: VideoModel;
  loraOverride?: LoRAConfig[];

  // Generation state
  dreamUuid?: string;
  status: TransitionStatus;
  progress?: number; // 0-100

  // Uprez state (undefined = not started)
  uprezDreamUuid?: string;
  uprezStatus?: "queue" | "processing" | "processed" | "failed";
  uprezProgress?: number;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd frontend && pnpm vitest run src/types/__tests__/flow.types.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add frontend/src/types/flow.types.ts frontend/src/types/__tests__/flow.types.test.ts
git commit -m "feat(flow): add FlowTransition and TransitionStatus types"
```

---

### Task 2: Store — Phase 1 state, actions, persist migration, and hydration

**Files:**
- Modify: `frontend/src/stores/flow.store.ts`
- Modify: `frontend/src/stores/flow.store.test.ts`

**Context:** The existing store has `keyframes`, `loop`, and Phase 0 actions. We extend it with global transition settings, transitions array, UI state, and Phase 1 actions. Persist version bumps from 1 → 2. `onRehydrateStorage` resets stale processing/queue transitions to "failed".

- [ ] **Step 1: Write failing tests for transition derivation**

Add to `frontend/src/stores/flow.store.test.ts`:

```typescript
import type { FlowTransition } from "@/types/flow.types";

// Helper to create a keyframe
const makeKf = (id: string, name = id) => ({
  id,
  keyframeUuid: `uuid-${id}`,
  imageUrl: `https://cdn.example.com/${id}.jpg`,
  name,
});

describe("Phase 1: transitions", () => {
  beforeEach(() => {
    useFlowStore.getState().resetFlow();
  });

  describe("recomputeTransitions", () => {
    it("creates transitions from adjacent keyframe pairs", () => {
      const store = useFlowStore.getState();
      store.addKeyframe(makeKf("a"));
      store.addKeyframe(makeKf("b"));
      store.addKeyframe(makeKf("c"));
      store.recomputeTransitions();

      const { transitions } = useFlowStore.getState();
      expect(transitions).toHaveLength(2);
      expect(transitions[0].fromKeyframeId).toBe("a");
      expect(transitions[0].toKeyframeId).toBe("b");
      expect(transitions[0].status).toBe("idle");
      expect(transitions[1].fromKeyframeId).toBe("b");
      expect(transitions[1].toKeyframeId).toBe("c");
    });

    it("creates no transitions with fewer than 2 keyframes", () => {
      const store = useFlowStore.getState();
      store.addKeyframe(makeKf("a"));
      store.recomputeTransitions();
      expect(useFlowStore.getState().transitions).toHaveLength(0);
    });

    it("preserves existing transition state when pairs still match", () => {
      const store = useFlowStore.getState();
      store.addKeyframe(makeKf("a"));
      store.addKeyframe(makeKf("b"));
      store.recomputeTransitions();

      // Simulate generation completing
      store.setTransitionDream(0, "dream-abc");
      store.updateTransitionStatus(0, "processed");
      store.setTransitionOverride(0, { promptOverride: "custom prompt" });

      // Recompute should preserve state
      store.recomputeTransitions();
      const { transitions } = useFlowStore.getState();
      expect(transitions[0].dreamUuid).toBe("dream-abc");
      expect(transitions[0].status).toBe("processed");
      expect(transitions[0].promptOverride).toBe("custom prompt");
    });

    it("adds loop transition with real keyframe IDs when loop enabled", () => {
      const store = useFlowStore.getState();
      store.addKeyframe(makeKf("a"));
      store.addKeyframe(makeKf("b"));
      store.addKeyframe(makeKf("c"));
      store.setLoop(true);
      store.recomputeTransitions();

      const { transitions } = useFlowStore.getState();
      expect(transitions).toHaveLength(3);
      // Loop transition: last → first, using real IDs
      expect(transitions[2].fromKeyframeId).toBe("c");
      expect(transitions[2].toKeyframeId).toBe("a");
    });

    it("removes loop transition when loop disabled", () => {
      const store = useFlowStore.getState();
      store.addKeyframe(makeKf("a"));
      store.addKeyframe(makeKf("b"));
      store.addKeyframe(makeKf("c"));
      store.setLoop(true);
      store.recomputeTransitions();
      expect(useFlowStore.getState().transitions).toHaveLength(3);

      store.setLoop(false);
      store.recomputeTransitions();
      expect(useFlowStore.getState().transitions).toHaveLength(2);
    });
  });

  describe("transition overrides", () => {
    it("sets per-transition overrides", () => {
      const store = useFlowStore.getState();
      store.addKeyframe(makeKf("a"));
      store.addKeyframe(makeKf("b"));
      store.recomputeTransitions();
      store.setTransitionOverride(0, {
        presetOverride: "Camera Basics",
        durationOverride: 8,
      });

      const t = useFlowStore.getState().transitions[0];
      expect(t.presetOverride).toBe("Camera Basics");
      expect(t.durationOverride).toBe(8);
    });

    it("clears all overrides on a transition", () => {
      const store = useFlowStore.getState();
      store.addKeyframe(makeKf("a"));
      store.addKeyframe(makeKf("b"));
      store.recomputeTransitions();
      store.setTransitionOverride(0, {
        presetOverride: "Organic",
        promptOverride: "test",
        durationOverride: 10,
        modelOverride: "ltx-i2v",
      });
      store.clearTransitionOverride(0);

      const t = useFlowStore.getState().transitions[0];
      expect(t.presetOverride).toBeUndefined();
      expect(t.promptOverride).toBeUndefined();
      expect(t.durationOverride).toBeUndefined();
      expect(t.modelOverride).toBeUndefined();
      expect(t.loraOverride).toBeUndefined();
    });
  });

  describe("transition status and dream tracking", () => {
    it("updates transition status and progress", () => {
      const store = useFlowStore.getState();
      store.addKeyframe(makeKf("a"));
      store.addKeyframe(makeKf("b"));
      store.recomputeTransitions();
      store.updateTransitionStatus(0, "processing", 50);

      const t = useFlowStore.getState().transitions[0];
      expect(t.status).toBe("processing");
      expect(t.progress).toBe(50);
    });

    it("stores dream UUID on a transition", () => {
      const store = useFlowStore.getState();
      store.addKeyframe(makeKf("a"));
      store.addKeyframe(makeKf("b"));
      store.recomputeTransitions();
      store.setTransitionDream(0, "dream-xyz");
      expect(useFlowStore.getState().transitions[0].dreamUuid).toBe(
        "dream-xyz",
      );
    });

    it("stores uprez dream UUID and updates uprez status", () => {
      const store = useFlowStore.getState();
      store.addKeyframe(makeKf("a"));
      store.addKeyframe(makeKf("b"));
      store.recomputeTransitions();
      store.setTransitionUprez(0, "uprez-789");
      store.updateTransitionUprezStatus(0, "processing", 30);

      const t = useFlowStore.getState().transitions[0];
      expect(t.uprezDreamUuid).toBe("uprez-789");
      expect(t.uprezStatus).toBe("processing");
      expect(t.uprezProgress).toBe(30);
    });
  });

  describe("global settings", () => {
    it("has correct defaults", () => {
      const s = useFlowStore.getState();
      expect(s.globalPresetId).toBe("");
      expect(s.globalPrompt).toBe("");
      expect(s.globalDuration).toBe(5);
      expect(s.globalModel).toBe("wan-i2v");
      expect(s.globalNumInferenceSteps).toBe(30);
      expect(s.globalGuidance).toBe(5.0);
      expect(s.selectedTransitionIndex).toBeNull();
      expect(s.settingsExpanded).toBe(false);
    });

    it("sets global settings individually", () => {
      const store = useFlowStore.getState();
      store.setGlobalPreset("Organic");
      store.setGlobalPrompt("gentle drift");
      store.setGlobalDuration(8);
      store.setGlobalModel("ltx-i2v");
      store.setGlobalNumInferenceSteps(20);
      store.setGlobalGuidance(3.5);

      const s = useFlowStore.getState();
      expect(s.globalPresetId).toBe("Organic");
      expect(s.globalPrompt).toBe("gentle drift");
      expect(s.globalDuration).toBe(8);
      expect(s.globalModel).toBe("ltx-i2v");
      expect(s.globalNumInferenceSteps).toBe(20);
      expect(s.globalGuidance).toBe(3.5);
    });
  });

  describe("UI state", () => {
    it("selects and deselects transitions", () => {
      const store = useFlowStore.getState();
      store.selectTransition(2);
      expect(useFlowStore.getState().selectedTransitionIndex).toBe(2);
      store.selectTransition(null);
      expect(useFlowStore.getState().selectedTransitionIndex).toBeNull();
    });

    it("toggles settings expanded", () => {
      const store = useFlowStore.getState();
      store.setSettingsExpanded(true);
      expect(useFlowStore.getState().settingsExpanded).toBe(true);
      store.setSettingsExpanded(false);
      expect(useFlowStore.getState().settingsExpanded).toBe(false);
    });
  });

  describe("hydration", () => {
    it("resets stale processing/queue transitions to failed on recompute", () => {
      const store = useFlowStore.getState();
      store.addKeyframe(makeKf("a"));
      store.addKeyframe(makeKf("b"));
      store.addKeyframe(makeKf("c"));
      store.recomputeTransitions();

      // Simulate in-flight states (as if persisted mid-generation)
      store.updateTransitionStatus(0, "processing", 50);
      store.updateTransitionStatus(1, "queue");

      // Simulate what onRehydrateStorage does
      store.reconcileStaleTransitions();

      const { transitions } = useFlowStore.getState();
      expect(transitions[0].status).toBe("failed");
      expect(transitions[0].progress).toBeUndefined();
      expect(transitions[1].status).toBe("failed");
    });

    it("does not reset processed or idle transitions", () => {
      const store = useFlowStore.getState();
      store.addKeyframe(makeKf("a"));
      store.addKeyframe(makeKf("b"));
      store.addKeyframe(makeKf("c"));
      store.recomputeTransitions();

      store.updateTransitionStatus(0, "processed");
      // leave transitions[1] as "idle"

      store.reconcileStaleTransitions();

      const { transitions } = useFlowStore.getState();
      expect(transitions[0].status).toBe("processed");
      expect(transitions[1].status).toBe("idle");
    });
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd frontend && pnpm vitest run src/stores/flow.store.test.ts`
Expected: FAIL — Phase 1 properties and methods don't exist on the store

- [ ] **Step 3: Implement store changes**

Modify `frontend/src/stores/flow.store.ts`. The complete store should look like this (showing Phase 1 additions integrated with existing Phase 0 code):

```typescript
import { create } from "zustand";
import { persist } from "zustand/middleware";
import type {
  FlowKeyframe,
  FlowTransition,
  TransitionStatus,
} from "@/types/flow.types";
import type { VideoModel } from "@/types/studio.types";

const LOOP_KEYFRAME_ID = "__loop__";

type FlowStoreState = {
  // Phase 0
  keyframes: FlowKeyframe[];
  loop: boolean;
  addKeyframe: (keyframe: FlowKeyframe) => void;
  removeKeyframe: (id: string) => void;
  reorderKeyframes: (orderedIds: string[]) => void;
  setLoop: (loop: boolean) => void;
  keyframesWithLoop: () => FlowKeyframe[];
  resetFlow: () => void;

  // Phase 1 — global transition settings
  globalPresetId: string;
  globalPrompt: string;
  globalDuration: number;
  globalModel: VideoModel;
  globalNumInferenceSteps: number;
  globalGuidance: number;

  // Phase 1 — transitions
  transitions: FlowTransition[];

  // Phase 1 — UI state
  selectedTransitionIndex: number | null;
  settingsExpanded: boolean;

  // Phase 1 — actions
  setGlobalPreset: (id: string) => void;
  setGlobalPrompt: (prompt: string) => void;
  setGlobalDuration: (duration: number) => void;
  setGlobalModel: (model: VideoModel) => void;
  setGlobalNumInferenceSteps: (steps: number) => void;
  setGlobalGuidance: (guidance: number) => void;
  setTransitionOverride: (
    index: number,
    overrides: Partial<FlowTransition>,
  ) => void;
  clearTransitionOverride: (index: number) => void;
  selectTransition: (index: number | null) => void;
  setSettingsExpanded: (expanded: boolean) => void;
  updateTransitionStatus: (
    index: number,
    status: TransitionStatus,
    progress?: number,
  ) => void;
  setTransitionDream: (index: number, dreamUuid: string) => void;
  setTransitionUprez: (index: number, uprezDreamUuid: string) => void;
  updateTransitionUprezStatus: (
    index: number,
    status: "queue" | "processing" | "processed" | "failed",
    progress?: number,
  ) => void;
  recomputeTransitions: () => void;
  reconcileStaleTransitions: () => void;
};

const PHASE_1_DEFAULTS = {
  globalPresetId: "",
  globalPrompt: "",
  globalDuration: 5,
  globalModel: "wan-i2v" as VideoModel,
  globalNumInferenceSteps: 30,
  globalGuidance: 5.0,
  transitions: [] as FlowTransition[],
  selectedTransitionIndex: null as number | null,
  settingsExpanded: false,
};

/**
 * Build transitions from adjacent keyframe pairs.
 * Preserves existing transition state when pairs still match.
 */
function deriveTransitions(
  keyframesWithLoop: FlowKeyframe[],
  existing: FlowTransition[],
): FlowTransition[] {
  // Filter to real keyframe IDs only (map loop keyframe to first real keyframe)
  const pairs: Array<{ fromId: string; toId: string }> = [];
  for (let i = 0; i < keyframesWithLoop.length - 1; i++) {
    const from = keyframesWithLoop[i];
    const to = keyframesWithLoop[i + 1];
    // Use real keyframe IDs — map __loop__ back to the first keyframe's ID
    const fromId =
      from.id === LOOP_KEYFRAME_ID
        ? keyframesWithLoop[0]?.id ?? from.id
        : from.id;
    const toId =
      to.id === LOOP_KEYFRAME_ID
        ? keyframesWithLoop[0]?.id ?? to.id
        : to.id;
    pairs.push({ fromId, toId });
  }

  // Build index of existing transitions for O(1) lookup
  const existingMap = new Map<string, FlowTransition>();
  for (const t of existing) {
    existingMap.set(`${t.fromKeyframeId}:${t.toKeyframeId}`, t);
  }

  return pairs.map(({ fromId, toId }) => {
    const key = `${fromId}:${toId}`;
    const prev = existingMap.get(key);
    if (prev) return prev;
    return { fromKeyframeId: fromId, toKeyframeId: toId, status: "idle" };
  });
}

export const useFlowStore = create<FlowStoreState>()(
  persist(
    (set, get) => ({
      // Phase 0 state (keep existing implementation exactly as-is)
      keyframes: [],
      loop: false,
      addKeyframe: (keyframe) =>
        set((s) => ({ keyframes: [...s.keyframes, keyframe] })),
      removeKeyframe: (id) =>
        set((s) => ({
          keyframes: s.keyframes.filter(
            (kf) => kf.id !== id && !kf.isLoopKeyframe,
          ),
        })),
      reorderKeyframes: (orderedIds) =>
        set((s) => {
          const map = new Map(s.keyframes.map((kf) => [kf.id, kf]));
          return {
            keyframes: orderedIds
              .map((id) => map.get(id))
              .filter(Boolean) as FlowKeyframe[],
          };
        }),
      setLoop: (loop) => set({ loop }),
      keyframesWithLoop: () => {
        const { keyframes, loop } = get();
        if (!loop || keyframes.length < 2) return keyframes;
        const first = keyframes[0];
        return [
          ...keyframes,
          {
            id: LOOP_KEYFRAME_ID,
            keyframeUuid: first.keyframeUuid,
            imageUrl: first.imageUrl,
            name: first.name,
            isLoopKeyframe: true,
          },
        ];
      },
      resetFlow: () =>
        set({
          keyframes: [],
          loop: false,
          ...PHASE_1_DEFAULTS,
        }),

      // Phase 1 — global settings
      ...PHASE_1_DEFAULTS,
      setGlobalPreset: (id) => set({ globalPresetId: id }),
      setGlobalPrompt: (prompt) => set({ globalPrompt: prompt }),
      setGlobalDuration: (duration) => set({ globalDuration: duration }),
      setGlobalModel: (model) => set({ globalModel: model }),
      setGlobalNumInferenceSteps: (steps) =>
        set({ globalNumInferenceSteps: steps }),
      setGlobalGuidance: (guidance) => set({ globalGuidance: guidance }),

      // Phase 1 — transition actions
      setTransitionOverride: (index, overrides) =>
        set((s) => {
          const transitions = [...s.transitions];
          if (!transitions[index]) return s;
          transitions[index] = { ...transitions[index], ...overrides };
          return { transitions };
        }),

      clearTransitionOverride: (index) =>
        set((s) => {
          const transitions = [...s.transitions];
          if (!transitions[index]) return s;
          const t = transitions[index];
          transitions[index] = {
            fromKeyframeId: t.fromKeyframeId,
            toKeyframeId: t.toKeyframeId,
            status: t.status,
            progress: t.progress,
            dreamUuid: t.dreamUuid,
            uprezDreamUuid: t.uprezDreamUuid,
            uprezStatus: t.uprezStatus,
            uprezProgress: t.uprezProgress,
          };
          return { transitions };
        }),

      selectTransition: (index) => set({ selectedTransitionIndex: index }),
      setSettingsExpanded: (expanded) => set({ settingsExpanded: expanded }),

      updateTransitionStatus: (index, status, progress) =>
        set((s) => {
          const transitions = [...s.transitions];
          if (!transitions[index]) return s;
          transitions[index] = {
            ...transitions[index],
            status,
            progress: progress ?? undefined,
          };
          return { transitions };
        }),

      setTransitionDream: (index, dreamUuid) =>
        set((s) => {
          const transitions = [...s.transitions];
          if (!transitions[index]) return s;
          transitions[index] = { ...transitions[index], dreamUuid };
          return { transitions };
        }),

      setTransitionUprez: (index, uprezDreamUuid) =>
        set((s) => {
          const transitions = [...s.transitions];
          if (!transitions[index]) return s;
          transitions[index] = { ...transitions[index], uprezDreamUuid };
          return { transitions };
        }),

      updateTransitionUprezStatus: (index, status, progress) =>
        set((s) => {
          const transitions = [...s.transitions];
          if (!transitions[index]) return s;
          transitions[index] = {
            ...transitions[index],
            uprezStatus: status,
            uprezProgress: progress ?? undefined,
          };
          return { transitions };
        }),

      recomputeTransitions: () =>
        set((s) => {
          // Call keyframesWithLoop via get() — NOT through a selector
          const allKfs = get().keyframesWithLoop();
          return {
            transitions: deriveTransitions(allKfs, s.transitions),
          };
        }),

      reconcileStaleTransitions: () =>
        set((s) => ({
          transitions: s.transitions.map((t) =>
            t.status === "processing" || t.status === "queue"
              ? { ...t, status: "failed" as const, progress: undefined }
              : t,
          ),
        })),
    }),
    {
      name: "flow-session",
      version: 2,
      migrate: (persisted: unknown, version: number) => {
        const state = persisted as Record<string, unknown>;
        if (version < 2) {
          return {
            ...state,
            ...PHASE_1_DEFAULTS,
          };
        }
        return state;
      },
      onRehydrateStorage: () => (state) => {
        if (state) {
          state.reconcileStaleTransitions();
          state.recomputeTransitions();
        }
      },
      partialize: (state) => ({
        keyframes: state.keyframes,
        loop: state.loop,
        transitions: state.transitions,
        globalPresetId: state.globalPresetId,
        globalPrompt: state.globalPrompt,
        globalDuration: state.globalDuration,
        globalModel: state.globalModel,
        globalNumInferenceSteps: state.globalNumInferenceSteps,
        globalGuidance: state.globalGuidance,
        selectedTransitionIndex: state.selectedTransitionIndex,
        settingsExpanded: state.settingsExpanded,
      }),
    },
  ),
);
```

**Important implementation notes:**
- `deriveTransitions` maps `__loop__` IDs back to real keyframe IDs
- `recomputeTransitions` uses `get().keyframesWithLoop()` — NOT through a selector
- `clearTransitionOverride` preserves generation state (dreamUuid, status, etc.) while removing all override fields
- `resetFlow` now also resets Phase 1 defaults
- `onRehydrateStorage` calls `reconcileStaleTransitions()` then `recomputeTransitions()`

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd frontend && pnpm vitest run src/stores/flow.store.test.ts`
Expected: ALL PASS (both Phase 0 existing tests and Phase 1 new tests)

- [ ] **Step 5: Run type-check**

Run: `cd frontend && pnpm type-check`
Expected: No type errors

- [ ] **Step 6: Commit**

```bash
git add frontend/src/stores/flow.store.ts frontend/src/stores/flow.store.test.ts frontend/src/types/flow.types.ts frontend/src/types/__tests__/flow.types.test.ts
git commit -m "feat(flow): add Phase 1 store state, transition derivation, persist v2 migration"
```

---

### Task 3: Preset Resolution Utility

**Files:**
- Create: `frontend/src/components/pages/studio/utils/resolve-flow-settings.ts`
- Create: `frontend/src/components/pages/studio/utils/__tests__/resolve-flow-settings.test.ts`

**Context:** This utility resolves a `PresetPack` name to a single `StudioAction` and computes effective settings (override > global) for a transition. Used by the settings panel for display and by `useFlowGeneration` for payload construction.

- [ ] **Step 1: Write failing tests**

```typescript
// frontend/src/components/pages/studio/utils/__tests__/resolve-flow-settings.test.ts
import { describe, it, expect } from "vitest";
import {
  resolvePresetAction,
  resolveEffectiveSettings,
} from "../resolve-flow-settings";
import type { FlowTransition } from "@/types/flow.types";

describe("resolvePresetAction", () => {
  it("returns first action from a known preset pack", () => {
    const action = resolvePresetAction("Camera Basics");
    expect(action).toBeDefined();
    expect(action!.prompt).toBeTruthy();
    expect(action!.highNoiseLoras).toBeDefined();
  });

  it("returns undefined for unknown preset name", () => {
    expect(resolvePresetAction("Nonexistent Pack")).toBeUndefined();
  });

  it("returns undefined for empty string", () => {
    expect(resolvePresetAction("")).toBeUndefined();
  });
});

describe("resolveEffectiveSettings", () => {
  const globalSettings = {
    globalPresetId: "Camera Basics",
    globalPrompt: "global prompt",
    globalDuration: 5,
    globalModel: "wan-i2v" as const,
    globalNumInferenceSteps: 30,
    globalGuidance: 5.0,
  };

  it("uses global settings when transition has no overrides", () => {
    const transition: FlowTransition = {
      fromKeyframeId: "a",
      toKeyframeId: "b",
      status: "idle",
    };

    const settings = resolveEffectiveSettings(transition, globalSettings);
    expect(settings.presetId).toBe("Camera Basics");
    expect(settings.prompt).toBe("global prompt");
    expect(settings.duration).toBe(5);
    expect(settings.model).toBe("wan-i2v");
    expect(settings.numInferenceSteps).toBe(30);
    expect(settings.guidance).toBe(5.0);
  });

  it("overrides with per-transition values", () => {
    const transition: FlowTransition = {
      fromKeyframeId: "a",
      toKeyframeId: "b",
      status: "idle",
      presetOverride: "Organic",
      promptOverride: "override prompt",
      durationOverride: 10,
      modelOverride: "ltx-i2v",
    };

    const settings = resolveEffectiveSettings(transition, globalSettings);
    expect(settings.presetId).toBe("Organic");
    expect(settings.prompt).toBe("override prompt");
    expect(settings.duration).toBe(10);
    expect(settings.model).toBe("ltx-i2v");
    // Non-overridden values fall through from global
    expect(settings.numInferenceSteps).toBe(30);
    expect(settings.guidance).toBe(5.0);
  });

  it("builds a bare action when no preset is selected", () => {
    const transition: FlowTransition = {
      fromKeyframeId: "a",
      toKeyframeId: "b",
      status: "idle",
    };
    const settings = resolveEffectiveSettings(transition, {
      ...globalSettings,
      globalPresetId: "",
    });

    expect(settings.action.prompt).toBe("global prompt");
    expect(settings.action.highNoiseLoras).toEqual([]);
    expect(settings.action.lowNoiseLoras).toEqual([]);
  });

  it("resolves action from preset pack using first action", () => {
    const transition: FlowTransition = {
      fromKeyframeId: "a",
      toKeyframeId: "b",
      status: "idle",
    };
    const settings = resolveEffectiveSettings(transition, globalSettings);

    // Action should come from Camera Basics pack, first action
    expect(settings.action).toBeDefined();
    expect(settings.action.prompt).toBeTruthy();
  });

  it("applies prompt override on top of preset action", () => {
    const transition: FlowTransition = {
      fromKeyframeId: "a",
      toKeyframeId: "b",
      status: "idle",
      promptOverride: "my custom prompt",
    };
    const settings = resolveEffectiveSettings(transition, globalSettings);
    // Prompt override replaces preset prompt, but LoRAs from preset are preserved
    expect(settings.action.prompt).toBe("my custom prompt");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd frontend && pnpm vitest run src/components/pages/studio/utils/__tests__/resolve-flow-settings.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement resolve-flow-settings.ts**

```typescript
// frontend/src/components/pages/studio/utils/resolve-flow-settings.ts
import type { StudioAction, VideoModel } from "@/types/studio.types";
import type { FlowTransition } from "@/types/flow.types";
import {
  ACTION_PRESETS,
  createActionsFromPreset,
} from "@/components/pages/studio/constants/action-presets";

/**
 * Resolve a PresetPack name to a single StudioAction (the first action in the pack).
 * Returns undefined if the preset name is empty or not found.
 */
export function resolvePresetAction(
  presetName: string,
): StudioAction | undefined {
  if (!presetName) return undefined;
  const pack = ACTION_PRESETS.find((p) => p.name === presetName);
  if (!pack) return undefined;
  const actions = createActionsFromPreset(pack);
  return actions[0];
}

interface GlobalSettings {
  globalPresetId: string;
  globalPrompt: string;
  globalDuration: number;
  globalModel: VideoModel;
  globalNumInferenceSteps: number;
  globalGuidance: number;
}

interface EffectiveSettings {
  presetId: string;
  prompt: string;
  duration: number;
  model: VideoModel;
  numInferenceSteps: number;
  guidance: number;
  action: Pick<StudioAction, "prompt" | "highNoiseLoras" | "lowNoiseLoras">;
}

/**
 * Compute effective settings for a transition: override > global.
 * Resolves the preset to a concrete StudioAction for buildVideoAlgoParams.
 */
export function resolveEffectiveSettings(
  transition: FlowTransition,
  global: GlobalSettings,
): EffectiveSettings {
  const presetId = transition.presetOverride ?? global.globalPresetId;
  const prompt = transition.promptOverride ?? global.globalPrompt;
  const duration = transition.durationOverride ?? global.globalDuration;
  const model = transition.modelOverride ?? global.globalModel;
  const numInferenceSteps = global.globalNumInferenceSteps;
  const guidance = global.globalGuidance;

  // Resolve preset → StudioAction
  let action: Pick<StudioAction, "prompt" | "highNoiseLoras" | "lowNoiseLoras">;

  if (transition.loraOverride) {
    // Explicit LoRA override — use it directly
    action = {
      prompt,
      highNoiseLoras: transition.loraOverride,
      lowNoiseLoras: [],
    };
  } else {
    const presetAction = resolvePresetAction(presetId);
    if (presetAction) {
      // Use preset's LoRAs, but override prompt if specified
      action = {
        prompt,
        highNoiseLoras: presetAction.highNoiseLoras ?? [],
        lowNoiseLoras: presetAction.lowNoiseLoras ?? [],
      };
    } else {
      // No preset — bare prompt, no LoRAs
      action = { prompt, highNoiseLoras: [], lowNoiseLoras: [] };
    }
  }

  return { presetId, prompt, duration, model, numInferenceSteps, guidance, action };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd frontend && pnpm vitest run src/components/pages/studio/utils/__tests__/resolve-flow-settings.test.ts`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git add frontend/src/components/pages/studio/utils/resolve-flow-settings.ts frontend/src/components/pages/studio/utils/__tests__/resolve-flow-settings.test.ts
git commit -m "feat(flow): add preset resolution and effective settings utility"
```

---

### Task 4: Transition Gap Component

**Files:**
- Create: `frontend/src/components/pages/studio/components/transition-gap.tsx`
- Create: `frontend/src/components/pages/studio/components/transition-gap.styled.tsx`

**Context:** Replaces the existing plain `<TransitionGap>` + `<GapLine>` in `keyframe-strip.tsx`. Renders 6 visual states based on `FlowTransition.status` and whether overrides are set. The existing `TransitionGap` and `GapLine` styled components in `keyframe-strip.styled.tsx` will be superseded by this new component (the old styled components stay in place — `keyframe-strip.tsx` will import from the new file instead).

- [ ] **Step 1: Create styled components**

```typescript
// frontend/src/components/pages/studio/components/transition-gap.styled.tsx
import styled, { css, keyframes } from "styled-components";
import { FLOW } from "@/constants/flow-theme.constants";

const fadeSlideUp = keyframes`
  from {
    opacity: 0;
    transform: translateY(6px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
`;

export const GapContainer = styled.div<{ $expanded: boolean }>`
  flex-shrink: 0;
  width: ${(p) => (p.$expanded ? "80px" : "64px")};
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 4px;
  cursor: pointer;
  transition: width 0.3s ease;
`;

export const GapLine = styled.div<{
  $configured: boolean;
  $failed: boolean;
}>`
  width: 32px;
  border-top: 2px dashed
    ${(p) =>
      p.$failed
        ? "#ef4444"
        : p.$configured
          ? FLOW.accent
          : FLOW.border};
  transition: border-color 0.3s ease;
`;

export const GapThumbnail = styled.div<{ $status: string }>`
  width: 48px;
  height: 34px;
  border-radius: 4px;
  overflow: hidden;
  position: relative;
  background: ${FLOW.bgCard};
  animation: ${fadeSlideUp} 0.4s ease;

  ${(p) =>
    p.$status === "failed" &&
    css`
      border: 1px solid #ef4444;
    `}

  ${(p) =>
    p.$status === "processed" &&
    css`
      border: 1px solid ${FLOW.success};
    `}

  img,
  video {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }
`;

export const GapStatusLabel = styled.span<{ $status: string }>`
  font-size: 9px;
  font-family: ${FLOW.fontFamily};
  color: ${(p) => {
    switch (p.$status) {
      case "processed":
        return FLOW.success;
      case "processing":
        return FLOW.processing;
      case "failed":
        return "#ef4444";
      default:
        return FLOW.textMuted;
    }
  }};
  text-transform: uppercase;
  letter-spacing: 0.5px;
`;

export const GapProgressBar = styled.div`
  width: 40px;
  height: 2px;
  background: ${FLOW.bgInput};
  border-radius: 1px;
  overflow: hidden;
`;

export const GapProgressFill = styled.div<{ $percent: number }>`
  height: 100%;
  width: ${(p) => p.$percent}%;
  background: ${FLOW.processing};
  transition: width 0.3s ease;
`;

export const DurationLabel = styled.span`
  font-size: 10px;
  font-family: ${FLOW.fontFamily};
  color: ${FLOW.accent};
`;

export const GapIcon = styled.span<{ $color: string }>`
  font-size: 14px;
  color: ${(p) => p.$color};
  line-height: 1;
`;
```

- [ ] **Step 2: Create the component**

```typescript
// frontend/src/components/pages/studio/components/transition-gap.tsx
import type { FlowTransition } from "@/types/flow.types";
import { FLOW } from "@/constants/flow-theme.constants";
import {
  GapContainer,
  GapLine,
  GapThumbnail,
  GapStatusLabel,
  GapProgressBar,
  GapProgressFill,
  DurationLabel,
  GapIcon,
} from "./transition-gap.styled";

interface TransitionGapProps {
  transition: FlowTransition;
  effectiveDuration: number;
  onClick: () => void;
}

function hasOverrides(t: FlowTransition): boolean {
  return !!(
    t.presetOverride ||
    t.promptOverride ||
    t.durationOverride !== undefined ||
    t.modelOverride ||
    t.loraOverride
  );
}

export function TransitionGapEnhanced({
  transition,
  effectiveDuration,
  onClick,
}: TransitionGapProps) {
  const { status, progress } = transition;
  const configured = hasOverrides(transition);
  const expanded = status !== "idle" || configured;

  // Empty state — no overrides, no generation
  if (status === "idle" && !configured) {
    return (
      <GapContainer $expanded={false} onClick={onClick}>
        <GapLine $configured={false} $failed={false} />
      </GapContainer>
    );
  }

  // Configured but not yet generated
  if (status === "idle" && configured) {
    return (
      <GapContainer $expanded={false} onClick={onClick}>
        <GapLine $configured $failed={false} />
        <DurationLabel>{effectiveDuration}s</DurationLabel>
      </GapContainer>
    );
  }

  // Queued
  if (status === "queue") {
    return (
      <GapContainer $expanded onClick={onClick}>
        <GapThumbnail $status="queue">
          <GapStatusLabel $status="queue">queued</GapStatusLabel>
        </GapThumbnail>
      </GapContainer>
    );
  }

  // Processing
  if (status === "processing") {
    return (
      <GapContainer $expanded onClick={onClick}>
        <GapThumbnail $status="processing">
          <GapStatusLabel $status="processing">
            {progress ?? 0}%
          </GapStatusLabel>
        </GapThumbnail>
        <GapProgressBar>
          <GapProgressFill $percent={progress ?? 0} />
        </GapProgressBar>
      </GapContainer>
    );
  }

  // Complete
  if (status === "processed") {
    return (
      <GapContainer $expanded onClick={onClick}>
        <GapThumbnail $status="processed">
          <GapIcon $color={FLOW.success}>&#x2713;</GapIcon>
        </GapThumbnail>
        <DurationLabel>{effectiveDuration}s</DurationLabel>
      </GapContainer>
    );
  }

  // Failed
  return (
    <GapContainer $expanded onClick={onClick}>
      <GapThumbnail $status="failed">
        <GapIcon $color="#ef4444">!</GapIcon>
      </GapThumbnail>
      <GapStatusLabel $status="failed">failed</GapStatusLabel>
    </GapContainer>
  );
}
```

- [ ] **Step 3: Run type-check**

Run: `cd frontend && pnpm type-check`
Expected: No errors

- [ ] **Step 4: Commit**

```bash
git add frontend/src/components/pages/studio/components/transition-gap.tsx frontend/src/components/pages/studio/components/transition-gap.styled.tsx
git commit -m "feat(flow): add TransitionGapEnhanced component with 6 visual states"
```

---

### Task 5: Integrate Enhanced Gaps into Keyframe Strip

**Files:**
- Modify: `frontend/src/components/pages/studio/components/keyframe-strip.tsx`

**Context:** Replace the existing plain `<TransitionGap><GapLine /></TransitionGap>` with the new `<TransitionGapEnhanced>` component. The strip now reads transitions from the flow store and computes effective duration per gap.

- [ ] **Step 1: Update keyframe-strip.tsx**

In the `KeyframeStrip` component, update the gap rendering logic:

```typescript
// Add imports at top of keyframe-strip.tsx
import { TransitionGapEnhanced } from "./transition-gap";
import { useFlowStore } from "@/stores/flow.store";
import { useShallow } from "zustand/react/shallow";

// Inside KeyframeStrip component, add store selectors:
const { transitions, selectTransition, globalDuration } = useFlowStore(
  useShallow((s) => ({
    transitions: s.transitions,
    selectTransition: s.selectTransition,
    globalDuration: s.globalDuration,
  })),
);

// Replace the gap rendering in the forEach loop:
// OLD:
//   stripItems.push(
//     <TransitionGap key={`gap-${i}`}>
//       <GapLine />
//     </TransitionGap>,
//   );
//
// NEW:
if (i > 0) {
  const transitionIndex = i - 1;
  const transition = transitions[transitionIndex];
  if (transition) {
    const effectiveDuration =
      transition.durationOverride ?? globalDuration;
    stripItems.push(
      <TransitionGapEnhanced
        key={`gap-${transitionIndex}`}
        transition={transition}
        effectiveDuration={effectiveDuration}
        onClick={() => selectTransition(transitionIndex)}
      />,
    );
  } else {
    // Fallback for transitions not yet computed
    stripItems.push(
      <TransitionGap key={`gap-${i}`}>
        <GapLine />
      </TransitionGap>,
    );
  }
}
```

Note: Keep the old `TransitionGap` and `GapLine` imports from `keyframe-strip.styled.tsx` as fallback. They'll be used when transitions haven't been computed yet (e.g., on first render before `recomputeTransitions` runs).

- [ ] **Step 2: Call recomputeTransitions when keyframes change**

In `flow-builder.tsx`, add an effect that recomputes transitions whenever keyframes or loop changes:

```typescript
// Add to flow-builder.tsx
import { useFlowStore } from "@/stores/flow.store";
import { useShallow } from "zustand/react/shallow";

// Inside FlowBuilder component:
const { keyframes, loop, recomputeTransitions } = useFlowStore(
  useShallow((s) => ({
    keyframes: s.keyframes,
    loop: s.loop,
    recomputeTransitions: s.recomputeTransitions,
  })),
);

// Recompute transitions when keyframes or loop changes
useEffect(() => {
  recomputeTransitions();
}, [keyframes.length, loop, recomputeTransitions]);
```

- [ ] **Step 3: Run type-check and verify**

Run: `cd frontend && pnpm type-check`
Expected: No errors

Run: `cd frontend && pnpm vitest run src/stores/flow.store.test.ts`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git add frontend/src/components/pages/studio/components/keyframe-strip.tsx frontend/src/components/pages/studio/components/flow-builder.tsx
git commit -m "feat(flow): integrate enhanced transition gaps into keyframe strip"
```

---

### Task 6: Transition Settings Panel

**Files:**
- Create: `frontend/src/components/pages/studio/components/transition-settings-panel.tsx`
- Create: `frontend/src/components/pages/studio/components/transition-settings-panel.styled.tsx`

**Context:** Panel below keyframe strip. Two modes: global (no gap selected) and per-transition (gap selected). Collapsed shows preset + duration + generate button. Expanded adds prompt, model, LoRA, advanced. Reactivity follows cascade rules from spec.

- [ ] **Step 1: Create styled components**

```typescript
// frontend/src/components/pages/studio/components/transition-settings-panel.styled.tsx
import styled, { css, keyframes } from "styled-components";
import { FLOW } from "@/constants/flow-theme.constants";

const slideIn = keyframes`
  from {
    opacity: 0;
    transform: translateY(8px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
`;

export const PanelContainer = styled.div`
  background: ${FLOW.bgCard};
  border: 1px solid ${FLOW.border};
  border-radius: ${FLOW.radius};
  padding: 16px 20px;
  margin-top: 12px;
  animation: ${slideIn} 0.4s ease;
`;

export const PanelHeader = styled.div`
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 12px;
`;

export const PanelTitle = styled.span`
  font-family: ${FLOW.fontFamilySerif};
  font-size: 14px;
  color: ${FLOW.text};
  text-transform: uppercase;
  letter-spacing: 1px;
`;

export const PanelSubtitle = styled.span`
  font-family: ${FLOW.fontFamily};
  font-size: 12px;
  color: ${FLOW.textDim};
`;

export const CloseButton = styled.button`
  background: none;
  border: none;
  color: ${FLOW.textDim};
  cursor: pointer;
  font-size: 16px;
  padding: 4px;
  line-height: 1;

  &:hover {
    color: ${FLOW.text};
  }
`;

export const FieldRow = styled.div`
  display: flex;
  align-items: center;
  gap: 12px;
  flex-wrap: wrap;
`;

export const FieldGroup = styled.div`
  display: flex;
  flex-direction: column;
  gap: 4px;
`;

export const FieldLabel = styled.label`
  font-family: ${FLOW.fontFamily};
  font-size: 11px;
  color: ${FLOW.textDim};
  text-transform: uppercase;
  letter-spacing: 0.5px;
`;

export const Select = styled.select`
  background: ${FLOW.bgInput};
  border: 1px solid ${FLOW.border};
  border-radius: ${FLOW.radiusSm};
  color: ${FLOW.text};
  font-family: ${FLOW.fontFamily};
  font-size: 13px;
  padding: 6px 10px;
  cursor: pointer;
  min-width: 140px;

  &:hover {
    border-color: ${FLOW.borderHover};
  }

  &:focus {
    outline: none;
    border-color: ${FLOW.accent};
  }
`;

export const PromptTextarea = styled.textarea`
  background: ${FLOW.bgInput};
  border: 1px solid ${FLOW.border};
  border-radius: ${FLOW.radiusSm};
  color: ${FLOW.text};
  font-family: ${FLOW.fontFamily};
  font-size: 13px;
  padding: 8px 10px;
  width: 100%;
  min-height: 60px;
  resize: vertical;

  &:hover {
    border-color: ${FLOW.borderHover};
  }

  &:focus {
    outline: none;
    border-color: ${FLOW.accent};
  }

  &::placeholder {
    color: ${FLOW.textMuted};
  }
`;

export const GenerateButton = styled.button<{ $disabled?: boolean }>`
  background: ${(p) => (p.$disabled ? FLOW.bgInput : FLOW.accentDim)};
  color: ${(p) => (p.$disabled ? FLOW.textMuted : FLOW.accent)};
  border: 1px solid ${(p) => (p.$disabled ? FLOW.border : FLOW.accent)};
  border-radius: ${FLOW.radiusSm};
  font-family: ${FLOW.fontFamily};
  font-size: 13px;
  font-weight: 500;
  padding: 6px 16px;
  cursor: ${(p) => (p.$disabled ? "not-allowed" : "pointer")};
  transition: all 0.2s ease;
  white-space: nowrap;

  &:hover:not(:disabled) {
    background: ${FLOW.accent};
    color: ${FLOW.bg};
  }
`;

export const ToggleLink = styled.button`
  background: none;
  border: none;
  color: ${FLOW.textDim};
  font-family: ${FLOW.fontFamily};
  font-size: 12px;
  cursor: pointer;
  padding: 0;
  margin-top: 8px;

  &:hover {
    color: ${FLOW.text};
  }
`;

export const ResetLink = styled.button`
  background: none;
  border: none;
  color: ${FLOW.textMuted};
  font-family: ${FLOW.fontFamily};
  font-size: 11px;
  cursor: pointer;
  padding: 0;
  text-decoration: underline;

  &:hover {
    color: ${FLOW.textDim};
  }
`;

export const ExpandedSection = styled.div`
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px solid ${FLOW.border};
`;

export const AdvancedToggle = styled.button`
  background: none;
  border: none;
  color: ${FLOW.textMuted};
  font-family: ${FLOW.fontFamily};
  font-size: 12px;
  cursor: pointer;
  padding: 0;
  text-align: left;

  &:hover {
    color: ${FLOW.textDim};
  }
`;

export const AdvancedFields = styled.div`
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
  padding: 8px 0;
`;

export const NumberInput = styled.input`
  background: ${FLOW.bgInput};
  border: 1px solid ${FLOW.border};
  border-radius: ${FLOW.radiusSm};
  color: ${FLOW.text};
  font-family: ${FLOW.fontFamily};
  font-size: 13px;
  padding: 6px 10px;
  width: 80px;

  &:hover {
    border-color: ${FLOW.borderHover};
  }

  &:focus {
    outline: none;
    border-color: ${FLOW.accent};
  }
`;
```

- [ ] **Step 2: Create the panel component**

```typescript
// frontend/src/components/pages/studio/components/transition-settings-panel.tsx
import { useState, useMemo, useCallback } from "react";
import { useFlowStore } from "@/stores/flow.store";
import { useShallow } from "zustand/react/shallow";
import type { VideoModel } from "@/types/studio.types";
import {
  ACTION_PRESETS,
  createActionsFromPreset,
} from "@/components/pages/studio/constants/action-presets";
import {
  getAllowedDurationsForActions,
  clampDurationToAllowed,
} from "@/components/pages/studio/constants/duration-options";
import { resolvePresetAction } from "@/components/pages/studio/utils/resolve-flow-settings";
import {
  PanelContainer,
  PanelHeader,
  PanelTitle,
  PanelSubtitle,
  CloseButton,
  FieldRow,
  FieldGroup,
  FieldLabel,
  Select,
  PromptTextarea,
  GenerateButton,
  ToggleLink,
  ResetLink,
  ExpandedSection,
  AdvancedToggle,
  AdvancedFields,
  NumberInput,
} from "./transition-settings-panel.styled";

interface TransitionSettingsPanelProps {
  onGenerateAll: () => void;
  onGenerateOne: (index: number) => void;
  isGenerating: boolean;
}

export function TransitionSettingsPanel({
  onGenerateAll,
  onGenerateOne,
  isGenerating,
}: TransitionSettingsPanelProps) {
  const {
    transitions,
    keyframes,
    selectedTransitionIndex,
    settingsExpanded,
    globalPresetId,
    globalPrompt,
    globalDuration,
    globalModel,
    globalNumInferenceSteps,
    globalGuidance,
    setGlobalPreset,
    setGlobalPrompt,
    setGlobalDuration,
    setGlobalModel,
    setGlobalNumInferenceSteps,
    setGlobalGuidance,
    setTransitionOverride,
    clearTransitionOverride,
    selectTransition,
    setSettingsExpanded,
  } = useFlowStore(
    useShallow((s) => ({
      transitions: s.transitions,
      keyframes: s.keyframes,
      selectedTransitionIndex: s.selectedTransitionIndex,
      settingsExpanded: s.settingsExpanded,
      globalPresetId: s.globalPresetId,
      globalPrompt: s.globalPrompt,
      globalDuration: s.globalDuration,
      globalModel: s.globalModel,
      globalNumInferenceSteps: s.globalNumInferenceSteps,
      globalGuidance: s.globalGuidance,
      setGlobalPreset: s.setGlobalPreset,
      setGlobalPrompt: s.setGlobalPrompt,
      setGlobalDuration: s.setGlobalDuration,
      setGlobalModel: s.setGlobalModel,
      setGlobalNumInferenceSteps: s.setGlobalNumInferenceSteps,
      setGlobalGuidance: s.setGlobalGuidance,
      setTransitionOverride: s.setTransitionOverride,
      clearTransitionOverride: s.clearTransitionOverride,
      selectTransition: s.selectTransition,
      setSettingsExpanded: s.setSettingsExpanded,
    })),
  );

  const [advancedOpen, setAdvancedOpen] = useState(false);

  // Per-transition mode?
  const isPerTransition = selectedTransitionIndex !== null;
  const selectedTransition =
    selectedTransitionIndex !== null
      ? transitions[selectedTransitionIndex]
      : null;

  // Effective values (override > global)
  const currentPresetId = selectedTransition?.presetOverride ?? globalPresetId;
  const currentPrompt = selectedTransition?.promptOverride ?? globalPrompt;
  const currentDuration = selectedTransition?.durationOverride ?? globalDuration;
  const currentModel = selectedTransition?.modelOverride ?? globalModel;
  const currentSteps = globalNumInferenceSteps;
  const currentGuidance = globalGuidance;

  // Filter presets by model: pack.model === currentModel || pack.model === "all"
  const filteredPresets = useMemo(
    () =>
      ACTION_PRESETS.filter(
        (p) => p.model === currentModel || p.model === "all",
      ),
    [currentModel],
  );

  // Compute allowed durations
  const allowedDurations = useMemo(() => {
    if (!currentPresetId) {
      return getAllowedDurationsForActions([], currentModel);
    }
    const action = resolvePresetAction(currentPresetId);
    if (!action) return getAllowedDurationsForActions([], currentModel);
    return getAllowedDurationsForActions([action], currentModel);
  }, [currentPresetId, currentModel]);

  // Setter helpers — dispatch to override or global depending on mode
  const setValue = useCallback(
    (field: string, value: unknown) => {
      if (isPerTransition && selectedTransitionIndex !== null) {
        setTransitionOverride(selectedTransitionIndex, {
          [field]: value,
        });
      } else {
        switch (field) {
          case "presetOverride":
            setGlobalPreset(value as string);
            break;
          case "promptOverride":
            setGlobalPrompt(value as string);
            break;
          case "durationOverride":
            setGlobalDuration(value as number);
            break;
          case "modelOverride":
            setGlobalModel(value as VideoModel);
            break;
        }
      }
    },
    [
      isPerTransition,
      selectedTransitionIndex,
      setTransitionOverride,
      setGlobalPreset,
      setGlobalPrompt,
      setGlobalDuration,
      setGlobalModel,
    ],
  );

  const handlePresetChange = useCallback(
    (presetName: string) => {
      setValue("presetOverride", presetName || undefined);

      // Fill prompt from preset
      const action = resolvePresetAction(presetName);
      if (action) {
        setValue("promptOverride", action.prompt);
      }

      // Clamp duration if needed
      const newAllowed = presetName
        ? getAllowedDurationsForActions(
            action ? [action] : [],
            currentModel,
          )
        : getAllowedDurationsForActions([], currentModel);
      const clamped = clampDurationToAllowed(currentDuration, newAllowed);
      if (clamped !== currentDuration) {
        setValue("durationOverride", clamped);
      }
    },
    [setValue, currentModel, currentDuration],
  );

  const handleModelChange = useCallback(
    (model: VideoModel) => {
      setValue("modelOverride", model);

      // Check if current preset is valid for new model
      if (currentPresetId) {
        const pack = ACTION_PRESETS.find((p) => p.name === currentPresetId);
        if (pack && pack.model !== "all" && pack.model !== model) {
          setValue("presetOverride", undefined);
        }
      }

      // Clamp duration to new model's allowed values
      const action = resolvePresetAction(currentPresetId);
      const newAllowed = action
        ? getAllowedDurationsForActions([action], model)
        : getAllowedDurationsForActions([], model);
      const clamped = clampDurationToAllowed(currentDuration, newAllowed);
      if (clamped !== currentDuration) {
        setValue("durationOverride", clamped);
      }
    },
    [setValue, currentPresetId, currentDuration],
  );

  // Generate All disabled?
  const generateAllDisabled =
    isGenerating ||
    transitions.every((t) =>
      ["processed", "queue", "processing"].includes(t.status),
    );

  // Don't show if fewer than 2 keyframes
  if (keyframes.length < 2) return null;

  // Transition header info
  const fromName =
    selectedTransition &&
    keyframes.find((kf) => kf.id === selectedTransition.fromKeyframeId)?.name;
  const toName =
    selectedTransition &&
    keyframes.find((kf) => kf.id === selectedTransition.toKeyframeId)?.name;

  const isComplete = selectedTransition?.status === "processed";

  return (
    <PanelContainer>
      <PanelHeader>
        <div>
          <PanelTitle>Transition Settings</PanelTitle>
          {isPerTransition && fromName && toName && (
            <PanelSubtitle>
              {" "}
              &mdash; Editing: {fromName} &rarr; {toName}
            </PanelSubtitle>
          )}
        </div>
        {isPerTransition && (
          <CloseButton onClick={() => selectTransition(null)}>
            &times;
          </CloseButton>
        )}
      </PanelHeader>

      {/* Collapsed view */}
      <FieldRow>
        <FieldGroup>
          <FieldLabel>Preset</FieldLabel>
          <Select
            value={currentPresetId}
            onChange={(e) => handlePresetChange(e.target.value)}
          >
            <option value="">No preset</option>
            {filteredPresets.map((p) => (
              <option key={p.name} value={p.name}>
                {p.name}
              </option>
            ))}
          </Select>
        </FieldGroup>

        <FieldGroup>
          <FieldLabel>Duration</FieldLabel>
          <Select
            value={currentDuration}
            onChange={(e) =>
              setValue("durationOverride", Number(e.target.value))
            }
          >
            {allowedDurations.map((d) => (
              <option key={d} value={d}>
                {d}s
              </option>
            ))}
          </Select>
        </FieldGroup>

        <GenerateButton
          $disabled={
            isPerTransition
              ? isGenerating
              : generateAllDisabled
          }
          disabled={
            isPerTransition
              ? isGenerating
              : generateAllDisabled
          }
          onClick={() => {
            if (isPerTransition && selectedTransitionIndex !== null) {
              onGenerateOne(selectedTransitionIndex);
            } else {
              onGenerateAll();
            }
          }}
        >
          {isPerTransition
            ? isComplete
              ? "Regenerate"
              : "Generate"
            : "Generate All"}
        </GenerateButton>
      </FieldRow>

      {/* Expand/collapse toggle */}
      {!settingsExpanded ? (
        <ToggleLink onClick={() => setSettingsExpanded(true)}>
          &#9662; Customize
        </ToggleLink>
      ) : (
        <>
          <ToggleLink onClick={() => setSettingsExpanded(false)}>
            &#9652; Collapse
          </ToggleLink>

          <ExpandedSection>
            <FieldGroup>
              <FieldLabel>Prompt</FieldLabel>
              <PromptTextarea
                value={currentPrompt}
                placeholder="Describe the transition motion..."
                onChange={(e) => setValue("promptOverride", e.target.value)}
              />
            </FieldGroup>

            <FieldRow>
              <FieldGroup>
                <FieldLabel>Model</FieldLabel>
                <Select
                  value={currentModel}
                  onChange={(e) =>
                    handleModelChange(e.target.value as VideoModel)
                  }
                >
                  <option value="wan-i2v">Wan I2V</option>
                  <option value="ltx-i2v">LTX 2.3</option>
                </Select>
              </FieldGroup>
            </FieldRow>

            <AdvancedToggle
              onClick={() => setAdvancedOpen(!advancedOpen)}
            >
              {advancedOpen ? "&#9662;" : "&#9656;"} Advanced (steps,
              guidance)
            </AdvancedToggle>

            {advancedOpen && (
              <AdvancedFields>
                <FieldGroup>
                  <FieldLabel>Steps</FieldLabel>
                  <NumberInput
                    type="number"
                    min={1}
                    max={100}
                    value={currentSteps}
                    onChange={(e) =>
                      setGlobalNumInferenceSteps(Number(e.target.value))
                    }
                  />
                </FieldGroup>
                <FieldGroup>
                  <FieldLabel>Guidance</FieldLabel>
                  <NumberInput
                    type="number"
                    min={0}
                    max={20}
                    step={0.5}
                    value={currentGuidance}
                    onChange={(e) =>
                      setGlobalGuidance(Number(e.target.value))
                    }
                  />
                </FieldGroup>
              </AdvancedFields>
            )}
          </ExpandedSection>
        </>
      )}

      {/* Per-transition extras */}
      {isPerTransition && selectedTransitionIndex !== null && (
        <ResetLink onClick={() => clearTransitionOverride(selectedTransitionIndex)}>
          Reset to defaults
        </ResetLink>
      )}
    </PanelContainer>
  );
}
```

- [ ] **Step 3: Run type-check**

Run: `cd frontend && pnpm type-check`
Expected: No errors

- [ ] **Step 4: Commit**

```bash
git add frontend/src/components/pages/studio/components/transition-settings-panel.tsx frontend/src/components/pages/studio/components/transition-settings-panel.styled.tsx
git commit -m "feat(flow): add transition settings panel with collapsed/expanded/per-transition modes"
```

---

### Task 7: useFlowGeneration Hook

**Files:**
- Create: `frontend/src/components/pages/studio/hooks/useFlowGeneration.ts`

**Context:** Creates dreams for transitions using `axiosClient` directly (not mutation hooks). Handles single generation and "Generate All" loop. Resolves effective settings per transition, builds payload via `buildVideoAlgoParams`, creates dream, attaches keyframes.

- [ ] **Step 1: Create the hook**

```typescript
// frontend/src/components/pages/studio/hooks/useFlowGeneration.ts
import { useCallback, useState } from "react";
import { useFlowStore } from "@/stores/flow.store";
import { useShallow } from "zustand/react/shallow";
import { axiosClient } from "@/client/axios.client";
import { getRequestHeaders, ContentType } from "@/constants/auth.constants";
import { buildVideoAlgoParams } from "@/components/pages/studio/utils/build-video-algo-params";
import { resolveEffectiveSettings } from "@/components/pages/studio/utils/resolve-flow-settings";
import type { FlowTransition } from "@/types/flow.types";

export function useFlowGeneration() {
  const [isGenerating, setIsGenerating] = useState(false);

  const {
    transitions,
    keyframes,
    globalPresetId,
    globalPrompt,
    globalDuration,
    globalModel,
    globalNumInferenceSteps,
    globalGuidance,
    setTransitionDream,
    updateTransitionStatus,
  } = useFlowStore(
    useShallow((s) => ({
      transitions: s.transitions,
      keyframes: s.keyframes,
      globalPresetId: s.globalPresetId,
      globalPrompt: s.globalPrompt,
      globalDuration: s.globalDuration,
      globalModel: s.globalModel,
      globalNumInferenceSteps: s.globalNumInferenceSteps,
      globalGuidance: s.globalGuidance,
      setTransitionDream: s.setTransitionDream,
      updateTransitionStatus: s.updateTransitionStatus,
    })),
  );

  const globalSettings = {
    globalPresetId,
    globalPrompt,
    globalDuration,
    globalModel,
    globalNumInferenceSteps,
    globalGuidance,
  };

  const generateTransition = useCallback(
    async (index: number, transition: FlowTransition) => {
      const settings = resolveEffectiveSettings(transition, globalSettings);

      // Find keyframe data for image reference
      const fromKf = keyframes.find(
        (kf) => kf.id === transition.fromKeyframeId,
      );
      if (!fromKf) {
        console.error(
          `Keyframe not found: ${transition.fromKeyframeId}`,
        );
        updateTransitionStatus(index, "failed");
        return;
      }

      // Build algo params
      const algoParams = buildVideoAlgoParams({
        model: settings.model,
        action: settings.action,
        imageUuid: fromKf.keyframeUuid,
        imageSize: undefined,
        duration: settings.duration,
        numInferenceSteps: settings.numInferenceSteps,
        guidance: settings.guidance,
      });

      // Find to-keyframe name for auto-naming
      const toKf = keyframes.find(
        (kf) => kf.id === transition.toKeyframeId,
      );
      const name = `${fromKf.name || "frame"} → ${toKf?.name || "frame"}`;

      try {
        // Step 1: Create dream
        const headers = getRequestHeaders({
          contentType: ContentType.json,
        });
        const { data: createData } = await axiosClient.post(
          "/v1/dream",
          {
            name,
            prompt: JSON.stringify(algoParams),
          },
          { headers },
        );
        const dreamUuid = createData?.data?.dream?.uuid;
        if (!dreamUuid) {
          throw new Error("No dream UUID returned from API");
        }

        // Step 2: Attach keyframes
        await axiosClient.put(
          `/v1/dream/${dreamUuid}`,
          {
            startKeyframe: fromKf.keyframeUuid,
            endKeyframe: toKf?.keyframeUuid,
          },
          { headers },
        );

        // Step 3: Store UUID and set status
        setTransitionDream(index, dreamUuid);
        updateTransitionStatus(index, "queue");
      } catch (error) {
        console.error("Failed to create transition dream:", error);
        updateTransitionStatus(index, "failed");
      }
    },
    [keyframes, globalSettings, setTransitionDream, updateTransitionStatus],
  );

  const generateAll = useCallback(async () => {
    setIsGenerating(true);
    try {
      // Read current transitions from store (fresh read)
      const currentTransitions = useFlowStore.getState().transitions;
      for (let i = 0; i < currentTransitions.length; i++) {
        const t = currentTransitions[i];
        // Skip already done or in-flight
        if (
          t.status === "processed" ||
          t.status === "processing" ||
          t.status === "queue"
        ) {
          continue;
        }
        await generateTransition(i, t);
      }
    } finally {
      setIsGenerating(false);
    }
  }, [generateTransition]);

  const generateOne = useCallback(
    async (index: number) => {
      setIsGenerating(true);
      try {
        const t = useFlowStore.getState().transitions[index];
        if (t) {
          await generateTransition(index, t);
        }
      } finally {
        setIsGenerating(false);
      }
    },
    [generateTransition],
  );

  return { generateAll, generateOne, isGenerating };
}
```

- [ ] **Step 2: Run type-check**

Run: `cd frontend && pnpm type-check`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
git add frontend/src/components/pages/studio/hooks/useFlowGeneration.ts
git commit -m "feat(flow): add useFlowGeneration hook for dream creation"
```

---

### Task 8: useFlowJobProgress Hook

**Files:**
- Create: `frontend/src/components/pages/studio/hooks/useFlowJobProgress.ts`

**Context:** Reimplementation of `useStudioJobProgress` targeting `useFlowStore`. Joins Socket.IO rooms for pending transitions, listens to `job:progress` events, updates store. Polling fallback at 10s intervals. Mounted in `flow-builder.tsx`.

- [ ] **Step 1: Create the hook**

```typescript
// frontend/src/components/pages/studio/hooks/useFlowJobProgress.ts
import { useEffect, useCallback, useMemo } from "react";
import { useFlowStore } from "@/stores/flow.store";
import { useShallow } from "zustand/react/shallow";
import { useSocket } from "@/hooks/useSocket";
import { axiosClient } from "@/client/axios.client";
import { getRequestHeaders, ContentType } from "@/constants/auth.constants";
import {
  JOB_PROGRESS_EVENT,
  JOIN_DREAM_ROOM_EVENT,
  LEAVE_DREAM_ROOM_EVENT,
} from "@/constants/remote-control.constants";

const POLL_INTERVAL_MS = 10_000;

// Maps backend status strings to TransitionStatus
function mapStatus(
  backendStatus: string,
): "queue" | "processing" | "processed" | "failed" | null {
  switch (backendStatus?.toUpperCase?.()) {
    case "IN_QUEUE":
    case "QUEUE":
      return "queue";
    case "IN_PROGRESS":
    case "PROCESSING":
      return "processing";
    case "COMPLETED":
    case "PROCESSED":
      return "processed";
    case "FAILED":
    case "TIMED_OUT":
    case "CANCELLED":
      return "failed";
    default:
      return null;
  }
}

export function useFlowJobProgress() {
  const { socket } = useSocket();

  const { transitions, updateTransitionStatus } = useFlowStore(
    useShallow((s) => ({
      transitions: s.transitions,
      updateTransitionStatus: s.updateTransitionStatus,
    })),
  );

  // Collect pending dream UUIDs with their transition indices
  const pendingEntries = useMemo(() => {
    const entries: Array<{ uuid: string; index: number; isUprez: boolean }> =
      [];
    transitions.forEach((t, i) => {
      if (
        t.dreamUuid &&
        (t.status === "queue" || t.status === "processing")
      ) {
        entries.push({ uuid: t.dreamUuid, index: i, isUprez: false });
      }
      if (
        t.uprezDreamUuid &&
        (t.uprezStatus === "queue" || t.uprezStatus === "processing")
      ) {
        entries.push({
          uuid: t.uprezDreamUuid,
          index: i,
          isUprez: true,
        });
      }
    });
    return entries;
  }, [transitions]);

  const pendingUuids = useMemo(
    () => pendingEntries.map((e) => e.uuid),
    [pendingEntries],
  );

  // Build UUID → { index, isUprez } lookup
  const uuidMap = useMemo(() => {
    const map = new Map<
      string,
      { index: number; isUprez: boolean }
    >();
    for (const e of pendingEntries) {
      map.set(e.uuid, { index: e.index, isUprez: e.isUprez });
    }
    return map;
  }, [pendingEntries]);

  // Handle job:progress events
  const handleProgress = useCallback(
    (data: {
      dreamUuid?: string;
      dream_uuid?: string;
      status?: string;
      progress?: number;
    }) => {
      const uuid = data.dreamUuid || data.dream_uuid;
      if (!uuid) return;

      const entry = uuidMap.get(uuid);
      if (!entry) return;

      const mappedStatus = mapStatus(data.status ?? "");
      if (!mappedStatus) return;

      if (entry.isUprez) {
        useFlowStore
          .getState()
          .updateTransitionUprezStatus(
            entry.index,
            mappedStatus,
            data.progress,
          );
      } else {
        updateTransitionStatus(entry.index, mappedStatus, data.progress);
      }
    },
    [uuidMap, updateTransitionStatus],
  );

  // Register Socket.IO event listener
  useEffect(() => {
    if (!socket) return;
    socket.on(JOB_PROGRESS_EVENT, handleProgress);
    return () => {
      socket.off(JOB_PROGRESS_EVENT, handleProgress);
    };
  }, [socket, handleProgress]);

  // Join/leave Socket.IO rooms for pending UUIDs
  useEffect(() => {
    if (!socket || pendingUuids.length === 0) return;

    const joinRooms = () => {
      pendingUuids.forEach((uuid) =>
        socket.emit(JOIN_DREAM_ROOM_EVENT, uuid),
      );
    };

    joinRooms();
    socket.on("connect", joinRooms);

    return () => {
      socket.off("connect", joinRooms);
      pendingUuids.forEach((uuid) =>
        socket.emit(LEAVE_DREAM_ROOM_EVENT, uuid),
      );
    };
  }, [socket, pendingUuids]);

  // Polling fallback
  useEffect(() => {
    if (pendingUuids.length === 0) return;

    const poll = async () => {
      const headers = getRequestHeaders({
        contentType: ContentType.json,
      });
      for (const entry of pendingEntries) {
        try {
          const { data } = await axiosClient.get(
            `/v1/dream/${entry.uuid}`,
            { headers },
          );
          const dream = data?.data?.dream;
          if (!dream) continue;

          const mappedStatus = mapStatus(dream.status);
          if (!mappedStatus) continue;

          if (entry.isUprez) {
            useFlowStore
              .getState()
              .updateTransitionUprezStatus(entry.index, mappedStatus);
          } else {
            updateTransitionStatus(entry.index, mappedStatus);
          }
        } catch {
          // Polling failure is non-fatal — Socket.IO is primary
        }
      }
    };

    const interval = setInterval(poll, POLL_INTERVAL_MS);
    return () => clearInterval(interval);
  }, [pendingEntries, updateTransitionStatus]);
}
```

- [ ] **Step 2: Run type-check**

Run: `cd frontend && pnpm type-check`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
git add frontend/src/components/pages/studio/hooks/useFlowJobProgress.ts
git commit -m "feat(flow): add useFlowJobProgress hook for Socket.IO progress tracking"
```

---

### Task 9: Wire Everything into FlowBuilder

**Files:**
- Modify: `frontend/src/components/pages/studio/components/flow-builder.tsx`

**Context:** Mount `useFlowJobProgress` and `useFlowGeneration` hooks. Render `TransitionSettingsPanel` below the keyframe strip. This is the integration task that connects all previous tasks.

- [ ] **Step 1: Update flow-builder.tsx**

Add to the `FlowBuilder` component:

```typescript
// Add imports
import { TransitionSettingsPanel } from "./transition-settings-panel";
import { useFlowGeneration } from "@/components/pages/studio/hooks/useFlowGeneration";
import { useFlowJobProgress } from "@/components/pages/studio/hooks/useFlowJobProgress";

// Inside FlowBuilder component body:

// Mount progress tracking
useFlowJobProgress();

// Generation controls
const { generateAll, generateOne, isGenerating } = useFlowGeneration();

// In the JSX return, after <KeyframeStrip> and before the closing wrapper:
<TransitionSettingsPanel
  onGenerateAll={generateAll}
  onGenerateOne={generateOne}
  isGenerating={isGenerating}
/>
```

- [ ] **Step 2: Run type-check**

Run: `cd frontend && pnpm type-check`
Expected: No errors

- [ ] **Step 3: Run all tests**

Run: `cd frontend && pnpm vitest run`
Expected: ALL PASS

- [ ] **Step 4: Commit**

```bash
git add frontend/src/components/pages/studio/components/flow-builder.tsx
git commit -m "feat(flow): wire generation hooks and settings panel into flow builder"
```

---

### Task 10: Video Preview and VideoLightbox

**Files:**
- Create: `frontend/src/components/pages/studio/components/flow-preview.tsx`
- Create: `frontend/src/components/pages/studio/components/flow-preview.styled.tsx`

**Context:** Inline video player below strip that plays completed transition segments sequentially. Click opens a `VideoLightbox` overlay. Partial preview plays available segments only.

- [ ] **Step 1: Create styled components**

```typescript
// frontend/src/components/pages/studio/components/flow-preview.styled.tsx
import styled, { keyframes } from "styled-components";
import { FLOW } from "@/constants/flow-theme.constants";

const fadeIn = keyframes`
  from { opacity: 0; }
  to { opacity: 1; }
`;

export const PreviewContainer = styled.div`
  background: ${FLOW.bgCard};
  border: 1px solid ${FLOW.border};
  border-radius: ${FLOW.radius};
  padding: 16px;
  margin-top: 12px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  animation: ${fadeIn} 0.4s ease;
`;

export const PreviewLabel = styled.span`
  font-family: ${FLOW.fontFamilySerif};
  font-size: 12px;
  color: ${FLOW.textDim};
  text-transform: uppercase;
  letter-spacing: 1px;
`;

export const VideoWrapper = styled.div`
  width: 100%;
  max-width: 480px;
  aspect-ratio: 16 / 9;
  border-radius: ${FLOW.radiusSm};
  overflow: hidden;
  background: ${FLOW.bg};
  cursor: pointer;
  position: relative;

  video {
    width: 100%;
    height: 100%;
    object-fit: contain;
  }
`;

export const ClickHint = styled.span`
  font-family: ${FLOW.fontFamily};
  font-size: 11px;
  color: ${FLOW.textMuted};
`;

export const LightboxOverlay = styled.div`
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.9);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  animation: ${fadeIn} 0.3s ease;
  cursor: pointer;
`;

export const LightboxVideo = styled.div`
  width: 90vw;
  max-width: 960px;
  aspect-ratio: 16 / 9;
  border-radius: ${FLOW.radius};
  overflow: hidden;

  video {
    width: 100%;
    height: 100%;
    object-fit: contain;
  }
`;

export const SegmentIndicator = styled.div`
  display: flex;
  gap: 4px;
  align-items: center;
`;

export const SegmentDot = styled.div<{ $active: boolean }>`
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: ${(p) => (p.$active ? FLOW.accent : FLOW.border)};
  transition: background 0.2s ease;
`;
```

- [ ] **Step 2: Create the component**

```typescript
// frontend/src/components/pages/studio/components/flow-preview.tsx
import { useRef, useState, useCallback, useEffect } from "react";
import { createPortal } from "react-dom";
import { useFlowStore } from "@/stores/flow.store";
import { useShallow } from "zustand/react/shallow";
import {
  PreviewContainer,
  PreviewLabel,
  VideoWrapper,
  ClickHint,
  LightboxOverlay,
  LightboxVideo,
  SegmentIndicator,
  SegmentDot,
} from "./flow-preview.styled";

// TODO: Replace with actual video URL resolution.
// For now, this constructs a URL from dream UUID.
// The actual implementation needs to fetch the dream's video URL from the API
// or construct the R2 CDN URL from the dream's thumbnail/video path.
function getDreamVideoUrl(dreamUuid: string): string {
  // This will be replaced with actual URL resolution in integration
  return `/v1/dream/${dreamUuid}/video`;
}

export function FlowPreview() {
  const transitions = useFlowStore(
    useShallow((s) => s.transitions),
  );

  const completedSegments = transitions
    .filter((t) => t.status === "processed" && t.dreamUuid)
    .map((t) => ({
      dreamUuid: t.dreamUuid!,
      url: getDreamVideoUrl(t.dreamUuid!),
    }));

  const [currentIndex, setCurrentIndex] = useState(0);
  const [lightboxOpen, setLightboxOpen] = useState(false);
  const videoRef = useRef<HTMLVideoElement>(null);
  const lightboxVideoRef = useRef<HTMLVideoElement>(null);

  // Reset index when segments change
  useEffect(() => {
    if (currentIndex >= completedSegments.length) {
      setCurrentIndex(0);
    }
  }, [completedSegments.length, currentIndex]);

  const handleEnded = useCallback(() => {
    if (currentIndex < completedSegments.length - 1) {
      setCurrentIndex((prev) => prev + 1);
    } else {
      setCurrentIndex(0); // Loop back
    }
  }, [currentIndex, completedSegments.length]);

  if (completedSegments.length === 0) return null;

  const currentUrl = completedSegments[currentIndex]?.url;

  return (
    <>
      <PreviewContainer>
        <PreviewLabel>Preview</PreviewLabel>
        <VideoWrapper onClick={() => setLightboxOpen(true)}>
          <video
            ref={videoRef}
            src={currentUrl}
            autoPlay
            muted
            onEnded={handleEnded}
          />
        </VideoWrapper>
        <SegmentIndicator>
          {completedSegments.map((_, i) => (
            <SegmentDot key={i} $active={i === currentIndex} />
          ))}
        </SegmentIndicator>
        <ClickHint>Click to expand</ClickHint>
      </PreviewContainer>

      {lightboxOpen &&
        createPortal(
          <LightboxOverlay onClick={() => setLightboxOpen(false)}>
            <LightboxVideo onClick={(e) => e.stopPropagation()}>
              <video
                ref={lightboxVideoRef}
                src={currentUrl}
                autoPlay
                controls
                onEnded={handleEnded}
              />
            </LightboxVideo>
          </LightboxOverlay>,
          document.body,
        )}
    </>
  );
}
```

- [ ] **Step 3: Run type-check**

Run: `cd frontend && pnpm type-check`
Expected: No errors

- [ ] **Step 4: Commit**

```bash
git add frontend/src/components/pages/studio/components/flow-preview.tsx frontend/src/components/pages/studio/components/flow-preview.styled.tsx
git commit -m "feat(flow): add inline video preview with lightbox overlay"
```

---

### Task 11: Action Bar and Uprez

**Files:**
- Create: `frontend/src/components/pages/studio/components/flow-action-bar.tsx`
- Create: `frontend/src/components/pages/studio/components/flow-action-bar.styled.tsx`

**Context:** Buttons below preview: Preview All, Uprez All (dropdown), Save to Playlist (stub). Uprez creates uprez dreams for all completed transitions.

- [ ] **Step 1: Create styled components**

```typescript
// frontend/src/components/pages/studio/components/flow-action-bar.styled.tsx
import styled from "styled-components";
import { FLOW } from "@/constants/flow-theme.constants";

export const ActionBarContainer = styled.div`
  display: flex;
  gap: 10px;
  margin-top: 12px;
  flex-wrap: wrap;
`;

export const ActionButton = styled.button<{ $accent?: boolean }>`
  background: ${(p) => (p.$accent ? FLOW.accentDim : FLOW.bgElevated)};
  color: ${(p) => (p.$accent ? FLOW.accent : FLOW.textDim)};
  border: 1px solid ${(p) => (p.$accent ? FLOW.accent : FLOW.border)};
  border-radius: ${FLOW.radiusSm};
  font-family: ${FLOW.fontFamily};
  font-size: 13px;
  padding: 8px 16px;
  cursor: pointer;
  transition: all 0.2s ease;

  &:hover {
    background: ${(p) => (p.$accent ? FLOW.accent : FLOW.borderHover)};
    color: ${(p) => (p.$accent ? FLOW.bg : FLOW.text)};
  }

  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
`;

export const UprezDropdown = styled.div`
  position: relative;
  display: inline-block;
`;

export const DropdownMenu = styled.div`
  position: absolute;
  top: 100%;
  left: 0;
  margin-top: 4px;
  background: ${FLOW.bgElevated};
  border: 1px solid ${FLOW.border};
  border-radius: ${FLOW.radiusSm};
  min-width: 200px;
  z-index: 10;
  overflow: hidden;
`;

export const DropdownItem = styled.button`
  display: block;
  width: 100%;
  background: none;
  border: none;
  color: ${FLOW.text};
  font-family: ${FLOW.fontFamily};
  font-size: 13px;
  padding: 10px 16px;
  text-align: left;
  cursor: pointer;

  &:hover {
    background: ${FLOW.bgInput};
  }
`;
```

- [ ] **Step 2: Create the component**

```typescript
// frontend/src/components/pages/studio/components/flow-action-bar.tsx
import { useState, useCallback } from "react";
import { useFlowStore } from "@/stores/flow.store";
import { useShallow } from "zustand/react/shallow";
import { axiosClient } from "@/client/axios.client";
import { getRequestHeaders, ContentType } from "@/constants/auth.constants";
import type { UprezModel } from "@/types/studio.types";
import {
  ActionBarContainer,
  ActionButton,
  UprezDropdown,
  DropdownMenu,
  DropdownItem,
} from "./flow-action-bar.styled";

interface FlowActionBarProps {
  onPreviewAll: () => void;
}

export function FlowActionBar({ onPreviewAll }: FlowActionBarProps) {
  const { transitions, setTransitionUprez, updateTransitionUprezStatus } =
    useFlowStore(
      useShallow((s) => ({
        transitions: s.transitions,
        setTransitionUprez: s.setTransitionUprez,
        updateTransitionUprezStatus: s.updateTransitionUprezStatus,
      })),
    );

  const [uprezDropdownOpen, setUprezDropdownOpen] = useState(false);
  const [isUprezzing, setIsUprezzing] = useState(false);

  const hasResults = transitions.some((t) => t.status === "processed");

  const handleUprezAll = useCallback(
    async (uprezModel: UprezModel) => {
      setUprezDropdownOpen(false);
      setIsUprezzing(true);
      const headers = getRequestHeaders({ contentType: ContentType.json });

      try {
        for (let i = 0; i < transitions.length; i++) {
          const t = transitions[i];
          if (t.status !== "processed" || !t.dreamUuid) continue;
          if (t.uprezStatus === "processed" || t.uprezStatus === "processing")
            continue;

          try {
            const algoParams = {
              infinidream_algorithm: uprezModel,
              video_uuid: t.dreamUuid,
              upscale_factor: 2,
              interpolation_factor: 2,
            };

            const { data } = await axiosClient.post(
              "/v1/dream",
              {
                name: `Uprez: ${t.fromKeyframeId} → ${t.toKeyframeId}`,
                prompt: JSON.stringify(algoParams),
              },
              { headers },
            );

            const uprezUuid = data?.data?.dream?.uuid;
            if (uprezUuid) {
              setTransitionUprez(i, uprezUuid);
              updateTransitionUprezStatus(i, "queue");
            }
          } catch (error) {
            console.error(`Uprez failed for transition ${i}:`, error);
            updateTransitionUprezStatus(i, "failed");
          }
        }
      } finally {
        setIsUprezzing(false);
      }
    },
    [transitions, setTransitionUprez, updateTransitionUprezStatus],
  );

  const handleSaveToPlaylist = useCallback(() => {
    // Phase 1 stub — show toast
    // TODO: Replace with actual toast notification system
    alert("Coming soon — Save to Playlist will be available in Phase 2");
  }, []);

  if (!hasResults) return null;

  return (
    <ActionBarContainer>
      <ActionButton onClick={onPreviewAll}>Preview All</ActionButton>

      <UprezDropdown>
        <ActionButton
          $accent
          disabled={isUprezzing}
          onClick={() => setUprezDropdownOpen(!uprezDropdownOpen)}
        >
          Uprez All &#9662;
        </ActionButton>
        {uprezDropdownOpen && (
          <DropdownMenu>
            <DropdownItem onClick={() => handleUprezAll("nvidia-uprez")}>
              Nvidia Super Resolution
            </DropdownItem>
            <DropdownItem onClick={() => handleUprezAll("uprez")}>
              Standard Uprez
            </DropdownItem>
          </DropdownMenu>
        )}
      </UprezDropdown>

      <ActionButton onClick={handleSaveToPlaylist}>
        Save to Playlist
      </ActionButton>
    </ActionBarContainer>
  );
}
```

- [ ] **Step 3: Wire action bar and preview into flow-builder.tsx**

```typescript
// Add to flow-builder.tsx imports
import { FlowPreview } from "./flow-preview";
import { FlowActionBar } from "./flow-action-bar";

// Add state for preview
const [showPreview, setShowPreview] = useState(false);

// In JSX, after <TransitionSettingsPanel>:
<FlowPreview />
<FlowActionBar onPreviewAll={() => setShowPreview(true)} />
```

- [ ] **Step 4: Run type-check**

Run: `cd frontend && pnpm type-check`
Expected: No errors

- [ ] **Step 5: Run all tests**

Run: `cd frontend && pnpm vitest run`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git add frontend/src/components/pages/studio/components/flow-action-bar.tsx frontend/src/components/pages/studio/components/flow-action-bar.styled.tsx frontend/src/components/pages/studio/components/flow-builder.tsx
git commit -m "feat(flow): add action bar with uprez all and save-to-playlist stub"
```

---

### Task 12: Final Integration and Smoke Test

**Files:**
- All previously created/modified files

**Context:** Verify everything compiles, tests pass, and the feature is usable end-to-end.

- [ ] **Step 1: Run full type-check**

Run: `cd frontend && pnpm type-check`
Expected: No errors

- [ ] **Step 2: Run full test suite**

Run: `cd frontend && pnpm vitest run`
Expected: ALL PASS

- [ ] **Step 3: Run lint**

Run: `cd frontend && pnpm lint`
Expected: No errors (or only pre-existing ones)

- [ ] **Step 4: Build**

Run: `cd frontend && pnpm build`
Expected: Successful build

- [ ] **Step 5: Manual smoke test checklist**

Start dev server: `cd frontend && pnpm dev`

Verify:
1. Add 3+ keyframes — gaps appear between them
2. Gaps show "empty" state (dashed line)
3. Settings panel appears below strip when 2+ keyframes exist
4. Select a preset — prompt fills, duration options adjust
5. Change model — presets re-filter, duration options update
6. Click a gap — panel switches to per-transition mode with header
7. Set per-transition overrides — gap shows "configured" (gold line + duration)
8. Click "Reset to defaults" — overrides clear
9. Click Generate All — dreams created, gaps show "queued"
10. Progress updates via Socket.IO — gaps show percentage + progress bar
11. Completed transitions show checkmark + green duration
12. Preview appears with completed segments
13. Click preview to open lightbox
14. Uprez All creates uprez jobs
15. Save to Playlist shows "Coming soon" toast
16. Refresh page — persisted state loads, stale processing resets to "failed"
17. Toggle loop — loop transition appears and generates correctly

- [ ] **Step 6: Commit any fixes from smoke testing**

```bash
git add -u
git commit -m "fix(flow): address issues found during smoke testing"
```

---

## File Summary

### New Files (13)
| File | Purpose |
|------|---------|
| `frontend/src/types/__tests__/flow.types.test.ts` | Type validation tests |
| `frontend/src/components/pages/studio/utils/resolve-flow-settings.ts` | Preset resolution + effective settings |
| `frontend/src/components/pages/studio/utils/__tests__/resolve-flow-settings.test.ts` | Tests for preset resolution |
| `frontend/src/components/pages/studio/components/transition-gap.tsx` | Enhanced gap with 6 visual states |
| `frontend/src/components/pages/studio/components/transition-gap.styled.tsx` | Gap styles |
| `frontend/src/components/pages/studio/components/transition-settings-panel.tsx` | Settings panel (collapsed/expanded/per-transition) |
| `frontend/src/components/pages/studio/components/transition-settings-panel.styled.tsx` | Panel styles |
| `frontend/src/components/pages/studio/components/flow-preview.tsx` | Inline video preview + lightbox |
| `frontend/src/components/pages/studio/components/flow-preview.styled.tsx` | Preview styles |
| `frontend/src/components/pages/studio/components/flow-action-bar.tsx` | Action buttons (Preview All, Uprez All, Save stub) |
| `frontend/src/components/pages/studio/components/flow-action-bar.styled.tsx` | Action bar styles |
| `frontend/src/components/pages/studio/hooks/useFlowGeneration.ts` | Dream creation logic |
| `frontend/src/components/pages/studio/hooks/useFlowJobProgress.ts` | Socket.IO progress tracking |

### Modified Files (5)
| File | Change |
|------|---------|
| `frontend/src/types/flow.types.ts` | Add FlowTransition, TransitionStatus |
| `frontend/src/stores/flow.store.ts` | Phase 1 state, actions, persist v2, hydration |
| `frontend/src/stores/flow.store.test.ts` | Phase 1 transition tests |
| `frontend/src/components/pages/studio/components/flow-builder.tsx` | Mount hooks, render panel/preview/action bar |
| `frontend/src/components/pages/studio/components/keyframe-strip.tsx` | Use TransitionGapEnhanced |
