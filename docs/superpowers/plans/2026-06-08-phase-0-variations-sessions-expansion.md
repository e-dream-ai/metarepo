# Phase 0: Variations, Sessions & Prompt Expansion — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add named sessions (save/load/switch), an inline variation grid (seed re-roll + prompt expansion), and `{A|B|C}` prompt expansion syntax to the studio — all frontend-only, no backend changes.

**Architecture:** Sessions wrap the existing flow + batch Zustand stores in a named-slot localStorage system with auto-save. The variation grid is an inline component below keyframes/transitions in the flow builder, fed by seed re-rolls and prompt expansion. The prompt expansion parser is a pure function that computes cross-products of `{A|B|C}` groups.

**Tech Stack:** React 18, TypeScript, Zustand (persist middleware), styled-components v6, Vitest

**Spec:** `docs/superpowers/specs/2026-06-08-variations-sessions-expansion-design.md`

**Base branch:** `main` (in the `frontend` repo at `/Users/maxcarlsonold/edream/frontend`)

---

## File Structure

| File | Responsibility | New/Edit |
|------|---------------|----------|
| `src/components/pages/studio/utils/expand-prompt.ts` | Pure function: parse `{A\|B\|C}` syntax, return expanded string array | New |
| `src/components/pages/studio/utils/__tests__/expand-prompt.test.ts` | Unit tests for prompt parser | New |
| `src/types/flow.types.ts` | Add `VariationCandidate` type, extend `FlowKeyframe` and `FlowTransition` | Edit |
| `src/types/session.types.ts` | `StudioSession` interface | New |
| `src/stores/session.store.ts` | Session CRUD, auto-save, switch, localStorage persistence | New |
| `src/stores/session.store.test.ts` | Session store unit tests | New |
| `src/stores/flow.store.ts` | Export `FlowStoreState` type, add variation actions, persist v4 migration | Edit |
| `src/stores/studio.store.ts` | Export `StudioStoreState` type for session serialization | Edit |
| `src/components/pages/studio/components/variation-grid.tsx` | Inline 2x2/3x3/4x4 grid of variation candidates | New |
| `src/components/pages/studio/components/variation-grid.styled.tsx` | Grid styles (FLOW theme, intentionally not reusing batch ResultCell — different theme systems) | New |
| `src/components/pages/studio/hooks/useSessionAutoSave.ts` | Debounced auto-save via Zustand subscribe (not React deps) | New |
| `src/components/pages/studio/components/prompt-expansion-badge.tsx` | Small badge showing "N variants" next to prompt fields | New |
| `src/components/pages/studio/components/session-switcher.tsx` | Dropdown for session list in studio header | New |
| `src/components/pages/studio/components/session-switcher.styled.tsx` | Session switcher styles | New |
| `src/components/pages/studio/studio.page.tsx` | Mount session switcher in header | Edit |
| `src/components/pages/studio/components/keyframe-card.tsx` | Add "Variations" button | Edit |
| `src/components/pages/studio/components/transition-gap.tsx` | Add "Variations" option for completed transitions | Edit |
| `src/components/pages/studio/components/flow-builder.tsx` | Wire variation generation, prompt expansion into generate flow | Edit |
| `src/components/pages/studio/components/transition-settings-panel.tsx` | Show expansion badge on prompt field | Edit |
| `src/components/pages/studio/components/generate-tab.tsx` | Expand action prompts, update combination count display | Edit |

---

### Task 1: Prompt Expansion Parser

**Files:**
- Create: `frontend/src/components/pages/studio/utils/expand-prompt.ts`
- Create: `frontend/src/components/pages/studio/utils/__tests__/expand-prompt.test.ts`

- [ ] **Step 1: Write the failing tests**

```typescript
// frontend/src/components/pages/studio/utils/__tests__/expand-prompt.test.ts
import { describe, it, expect } from "vitest";
import { expandPrompt, countExpansions } from "../expand-prompt";

describe("expandPrompt", () => {
  it("returns single-element array for plain text", () => {
    expect(expandPrompt("hello world")).toEqual(["hello world"]);
  });

  it("expands a single {A|B|C} group", () => {
    expect(expandPrompt("{fire|water|earth} elemental")).toEqual([
      "fire elemental",
      "water elemental",
      "earth elemental",
    ]);
  });

  it("computes cross-product of two groups", () => {
    const result = expandPrompt("{fire|water} {dragon|phoenix}");
    expect(result).toEqual([
      "fire dragon",
      "fire phoenix",
      "water dragon",
      "water phoenix",
    ]);
  });

  it("handles mixed static and dynamic parts", () => {
    expect(expandPrompt("a {B|C} d")).toEqual(["a B d", "a C d"]);
  });

  it("handles escaped braces", () => {
    expect(expandPrompt("\\{not expanded\\}")).toEqual(["{not expanded}"]);
  });

  it("handles single option in braces (no pipe)", () => {
    expect(expandPrompt("{solo}")).toEqual(["solo"]);
  });

  it("handles empty string", () => {
    expect(expandPrompt("")).toEqual([""]);
  });

  it("trims whitespace in options", () => {
    expect(expandPrompt("{ fire | water }")).toEqual(["fire", "water"]);
  });

  it("handles three groups cross-product", () => {
    const result = expandPrompt("{a|b} {x|y} {1|2}");
    expect(result).toHaveLength(8);
    expect(result).toContain("a x 1");
    expect(result).toContain("b y 2");
  });

  it("caps at 16 expansions", () => {
    // 3 × 3 × 3 = 27 > 16
    const result = expandPrompt("{a|b|c} {x|y|z} {1|2|3}");
    expect(result).toHaveLength(16);
  });
});

describe("countExpansions", () => {
  it("returns 1 for plain text", () => {
    expect(countExpansions("no expansions here")).toBe(1);
  });

  it("returns option count for single group", () => {
    expect(countExpansions("{a|b|c}")).toBe(3);
  });

  it("returns cross-product count for multiple groups", () => {
    expect(countExpansions("{a|b} {x|y|z}")).toBe(6);
  });

  it("caps count at 16", () => {
    // 3 × 3 × 3 = 27 → capped at 16
    expect(countExpansions("{a|b|c} {x|y|z} {1|2|3}")).toBe(16);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx vitest run src/components/pages/studio/utils/__tests__/expand-prompt.test.ts 2>&1 | tail -20`
Expected: FAIL — module not found

- [ ] **Step 3: Implement the parser**

```typescript
// frontend/src/components/pages/studio/utils/expand-prompt.ts

interface ParsedSegment {
  type: "literal" | "group";
  value: string; // for literal
  options: string[]; // for group
}

function parse(template: string): ParsedSegment[] {
  const segments: ParsedSegment[] = [];
  let current = "";
  let i = 0;

  while (i < template.length) {
    // Escaped brace
    if (template[i] === "\\" && (template[i + 1] === "{" || template[i + 1] === "}")) {
      current += template[i + 1];
      i += 2;
      continue;
    }

    // Start of group
    if (template[i] === "{") {
      if (current) {
        segments.push({ type: "literal", value: current, options: [] });
        current = "";
      }
      const closeIdx = template.indexOf("}", i);
      if (closeIdx === -1) {
        // Unmatched brace — treat as literal
        current += template[i];
        i++;
        continue;
      }
      const inner = template.slice(i + 1, closeIdx);
      const options = inner.split("|").map((s) => s.trim());
      segments.push({ type: "group", value: "", options });
      i = closeIdx + 1;
      continue;
    }

    current += template[i];
    i++;
  }

  if (current) {
    segments.push({ type: "literal", value: current, options: [] });
  }

  return segments;
}

function crossProduct(segments: ParsedSegment[]): string[] {
  let results = [""];

  for (const seg of segments) {
    if (seg.type === "literal") {
      results = results.map((r) => r + seg.value);
    } else {
      const next: string[] = [];
      for (const r of results) {
        for (const opt of seg.options) {
          next.push(r + opt);
        }
      }
      results = next;
    }
  }

  return results;
}

const MAX_EXPANSIONS = 16;

/**
 * Expand a prompt template containing `{A|B|C}` groups into all combinations.
 * Returns a single-element array if no expansion syntax is present.
 * Capped at 16 expansions to prevent runaway generation.
 */
export function expandPrompt(template: string): string[] {
  if (!template) return [template];
  const segments = parse(template);
  const results = crossProduct(segments);
  return results.slice(0, MAX_EXPANSIONS);
}

/**
 * Count how many expansions a template would produce without generating them.
 * Useful for showing "N variants" badge.
 */
export function countExpansions(template: string): number {
  if (!template) return 1;
  const segments = parse(template);
  let count = 1;
  for (const seg of segments) {
    if (seg.type === "group") {
      count *= seg.options.length;
    }
  }
  return Math.min(count, MAX_EXPANSIONS);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx vitest run src/components/pages/studio/utils/__tests__/expand-prompt.test.ts 2>&1 | tail -20`
Expected: All tests PASS

- [ ] **Step 5: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/utils/expand-prompt.ts src/components/pages/studio/utils/__tests__/expand-prompt.test.ts
git commit -m "feat(studio): add prompt expansion parser with {A|B|C} syntax"
```

---

### Task 2: VariationCandidate Type + Flow Store Extensions

**Files:**
- Modify: `frontend/src/types/flow.types.ts`
- Modify: `frontend/src/stores/flow.store.ts`

- [ ] **Step 1: Write the failing test**

```typescript
// frontend/src/types/__tests__/flow.types.test.ts
// Add to existing test file or create new:
import { describe, it, expect } from "vitest";
import type { VariationCandidate, FlowKeyframe } from "@/types/flow.types";

describe("VariationCandidate type", () => {
  it("accepts a seed variation", () => {
    const v: VariationCandidate = {
      id: "v1",
      method: "seed",
      seed: 42,
      dreamUuid: "dream-123",
      imageUrl: "https://example.com/img.jpg",
      status: "processed",
    };
    expect(v.method).toBe("seed");
  });

  it("accepts an expansion variation", () => {
    const v: VariationCandidate = {
      id: "v2",
      method: "expansion",
      prompt: "fire elemental",
      status: "queue",
    };
    expect(v.prompt).toBe("fire elemental");
  });

  it("accepts i2i variation", () => {
    const v: VariationCandidate = {
      id: "v3",
      method: "i2i",
      status: "processing",
      progress: 45,
    };
    expect(v.method).toBe("i2i");
  });

  it("FlowKeyframe accepts variations array", () => {
    const kf: FlowKeyframe = {
      id: "kf-1",
      dreamUuid: "dream-settled", // must have dreamUuid to survive partialize
      imageUrl: "https://example.com/img.jpg",
      name: "test",
      variations: [
        { id: "v1", method: "seed", seed: 1, status: "processed", imageUrl: "url1" },
        { id: "v2", method: "seed", seed: 2, status: "queue" },
      ],
      activeVariationId: "v1",
    };
    expect(kf.variations).toHaveLength(2);
    expect(kf.activeVariationId).toBe("v1");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx vitest run src/types/__tests__/flow.types.test.ts 2>&1 | tail -20`
Expected: FAIL — `VariationCandidate` not exported

- [ ] **Step 3: Add VariationCandidate type to flow.types.ts**

Add at the end of `frontend/src/types/flow.types.ts`:

```typescript
export interface VariationCandidate {
  id: string; // local UUID
  method: "seed" | "expansion" | "i2i";
  prompt?: string; // the specific prompt used (if expansion)
  seed?: number; // the seed used (if seed re-roll)
  dreamUuid?: string; // backend dream UUID
  imageUrl?: string; // resolved image URL (for keyframe variations)
  status: TransitionStatus;
  progress?: number; // 0-100
}
```

Add `variations` and `activeVariationId` fields to `FlowKeyframe`:

```typescript
export interface FlowKeyframe {
  // ...existing fields...

  // Variation tracking (Phase 3)
  variations?: VariationCandidate[];
  activeVariationId?: string; // which variation is currently displayed
}
```

Add the same fields to `FlowTransition`:

```typescript
export interface FlowTransition {
  // ...existing fields (after uprezProgress)...

  // Variation tracking (Phase 3)
  variations?: VariationCandidate[];
  activeVariationId?: string;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx vitest run src/types/__tests__/flow.types.test.ts 2>&1 | tail -20`
Expected: PASS

- [ ] **Step 5: Add variation actions to flow store**

In `frontend/src/stores/flow.store.ts`, add to the `FlowStoreState` type (after `reconcileStaleTransitions`):

```typescript
  // Variation actions
  addKeyframeVariations: (keyframeId: string, candidates: VariationCandidate[]) => void;
  selectKeyframeVariation: (keyframeId: string, variationId: string) => void;
  clearKeyframeVariations: (keyframeId: string) => void;
  updateKeyframeVariation: (keyframeId: string, variationId: string, patch: Partial<VariationCandidate>) => void;
  addTransitionVariations: (transitionIndex: number, candidates: VariationCandidate[]) => void;
  selectTransitionVariation: (transitionIndex: number, variationId: string) => void;
  clearTransitionVariations: (transitionIndex: number) => void;
  updateTransitionVariation: (transitionIndex: number, variationId: string, patch: Partial<VariationCandidate>) => void;
```

Add the implementations in the store's `set` callback:

```typescript
      addKeyframeVariations: (keyframeId, candidates) =>
        set((s) => ({
          keyframes: s.keyframes.map((kf) =>
            kf.id === keyframeId
              ? { ...kf, variations: [...(kf.variations || []), ...candidates] }
              : kf,
          ),
        })),

      selectKeyframeVariation: (keyframeId, variationId) =>
        set((s) => {
          const kf = s.keyframes.find((k) => k.id === keyframeId);
          const variation = kf?.variations?.find((v) => v.id === variationId);
          if (!kf || !variation || variation.status !== "processed") return s;
          return {
            keyframes: s.keyframes.map((k) =>
              k.id === keyframeId
                ? {
                    ...k,
                    activeVariationId: variationId,
                    imageUrl: variation.imageUrl || k.imageUrl,
                    dreamUuid: variation.dreamUuid || k.dreamUuid,
                  }
                : k,
            ),
          };
        }),

      clearKeyframeVariations: (keyframeId) =>
        set((s) => ({
          keyframes: s.keyframes.map((kf) =>
            kf.id === keyframeId
              ? { ...kf, variations: undefined, activeVariationId: undefined }
              : kf,
          ),
        })),

      updateKeyframeVariation: (keyframeId, variationId, patch) =>
        set((s) => ({
          keyframes: s.keyframes.map((kf) =>
            kf.id === keyframeId
              ? {
                  ...kf,
                  variations: kf.variations?.map((v) =>
                    v.id === variationId ? { ...v, ...patch } : v,
                  ),
                }
              : kf,
          ),
        })),

      addTransitionVariations: (index, candidates) =>
        set((s) => ({
          transitions: s.transitions.map((t, i) =>
            i === index
              ? { ...t, variations: [...(t.variations || []), ...candidates] }
              : t,
          ),
        })),

      selectTransitionVariation: (index, variationId) =>
        set((s) => {
          const transition = s.transitions[index];
          const variation = transition?.variations?.find((v) => v.id === variationId);
          if (!transition || !variation || variation.status !== "processed") return s;
          return {
            transitions: s.transitions.map((t, i) =>
              i === index
                ? {
                    ...t,
                    activeVariationId: variationId,
                    dreamUuid: variation.dreamUuid || t.dreamUuid,
                    status: "processed" as const,
                  }
                : t,
            ),
          };
        }),

      clearTransitionVariations: (index) =>
        set((s) => ({
          transitions: s.transitions.map((t, i) =>
            i === index
              ? { ...t, variations: undefined, activeVariationId: undefined }
              : t,
          ),
        })),

      updateTransitionVariation: (index, variationId, patch) =>
        set((s) => ({
          transitions: s.transitions.map((t, i) =>
            i === index
              ? {
                  ...t,
                  variations: t.variations?.map((v) =>
                    v.id === variationId ? { ...v, ...patch } : v,
                  ),
                }
              : t,
          ),
        })),
```

- [ ] **Step 6: Export FlowStoreState type**

At the bottom of the `FlowStoreState` type definition (currently it's a private `type`), change it to:

```typescript
export type FlowStoreState = {
  // ... all existing fields ...
};
```

- [ ] **Step 7: Update persist version to 4 and update `partialize` to include variation fields**

Update the persist config's `version` to `4`. No data migration needed — new fields default to `undefined`.

```typescript
      version: 4,
```

**Critical:** Update the `partialize` function to include `variations` and `activeVariationId` on keyframes. The existing `partialize` strips keyframes to a whitelist of fields (look for the `keyframes: state.keyframes.map(...)` block). Add the two new fields to the mapped object:

```typescript
      partialize: (state) => ({
        // ...existing fields...
        keyframes: state.keyframes
          .filter((kf) => kf.keyframeUuid || kf.dreamUuid) // existing filter
          .map((kf) => ({
            id: kf.id,
            keyframeUuid: kf.keyframeUuid,
            dreamUuid: kf.dreamUuid,
            imageUrl: kf.imageUrl,
            name: kf.name,
            isLoopKeyframe: kf.isLoopKeyframe,
            // NEW: persist variation state for settled keyframes
            variations: kf.variations?.filter((v) => v.status !== "queue" && v.status !== "processing"),
            activeVariationId: kf.activeVariationId,
          })),
        // Filter in-flight variation candidates from transitions too (same as keyframes above).
        // Without this, stale "processing" candidates survive reload and get stuck permanently.
        transitions: state.transitions.map((t) => ({
          ...t,
          variations: t.variations?.filter(
            (v) => v.status !== "queue" && v.status !== "processing",
          ),
        })),
        // ...rest of existing partialize...
      }),
```

Note: Only persist variation candidates with settled status (`processed` or `failed`). In-flight candidates (`queue`, `processing`) are stripped — they'll be stale on reload.

- [ ] **Step 8: Verify compilation**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx tsc --noEmit --pretty 2>&1 | head -30`
Expected: No errors related to flow.types.ts or flow.store.ts

- [ ] **Step 9: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/types/flow.types.ts src/stores/flow.store.ts src/types/__tests__/flow.types.test.ts
git commit -m "feat(studio): add VariationCandidate type and flow store variation actions"
```

---

### Task 3: Session Types and Store

**Files:**
- Create: `frontend/src/types/session.types.ts`
- Create: `frontend/src/stores/session.store.ts`
- Create: `frontend/src/stores/session.store.test.ts`
- Modify: `frontend/src/stores/studio.store.ts`

- [ ] **Step 1: Write the session types**

```typescript
// frontend/src/types/session.types.ts
import type { StudioMode } from "@/types/flow.types";

export interface StudioSession {
  id: string;
  name: string;
  createdAt: string; // ISO date
  updatedAt: string; // ISO date
  mode: StudioMode; // which mode was active when last saved
  flowState: Record<string, unknown>; // serialized FlowStoreState
  batchState: Record<string, unknown>; // serialized StudioStoreState
  thumbnail?: string; // data URL of first keyframe/image
}

export const MAX_SESSIONS = 20;
export const SESSIONS_STORAGE_KEY = "studio-sessions";
export const ACTIVE_SESSION_KEY = "studio-active-session-id";
```

- [ ] **Step 2: Write the failing test**

```typescript
// frontend/src/stores/session.store.test.ts
import { describe, it, expect, beforeEach, vi } from "vitest";

// Mock localStorage
const localStorageMock = (() => {
  let store: Record<string, string> = {};
  return {
    getItem: (key: string) => store[key] ?? null,
    setItem: (key: string, value: string) => { store[key] = value; },
    removeItem: (key: string) => { delete store[key]; },
    clear: () => { store = {}; },
  };
})();
Object.defineProperty(globalThis, "localStorage", { value: localStorageMock });

import { useSessionStore } from "./session.store";
import { SESSIONS_STORAGE_KEY, ACTIVE_SESSION_KEY } from "@/types/session.types";

describe("session store", () => {
  beforeEach(() => {
    localStorage.clear();
    useSessionStore.setState({
      sessions: [],
      activeSessionId: null,
    });
  });

  it("creates a new session", () => {
    const { createSession } = useSessionStore.getState();
    createSession("My Flow");
    const { sessions, activeSessionId } = useSessionStore.getState();
    expect(sessions).toHaveLength(1);
    expect(sessions[0].name).toBe("My Flow");
    expect(activeSessionId).toBe(sessions[0].id);
  });

  it("switches between sessions", () => {
    const { createSession } = useSessionStore.getState();
    createSession("Session A");
    createSession("Session B");
    const { sessions } = useSessionStore.getState();
    expect(sessions).toHaveLength(2);

    const { switchSession } = useSessionStore.getState();
    switchSession(sessions[0].id);
    expect(useSessionStore.getState().activeSessionId).toBe(sessions[0].id);
  });

  it("renames a session", () => {
    const { createSession } = useSessionStore.getState();
    createSession("Old Name");
    const { sessions, renameSession } = useSessionStore.getState();
    renameSession(sessions[0].id, "New Name");
    expect(useSessionStore.getState().sessions[0].name).toBe("New Name");
  });

  it("deletes a session", () => {
    const { createSession } = useSessionStore.getState();
    createSession("To Delete");
    createSession("Keep");
    const { sessions, deleteSession } = useSessionStore.getState();
    deleteSession(sessions[0].id);
    expect(useSessionStore.getState().sessions).toHaveLength(1);
    expect(useSessionStore.getState().sessions[0].name).toBe("Keep");
  });

  it("duplicates a session", () => {
    const { createSession } = useSessionStore.getState();
    createSession("Original");
    const { sessions, duplicateSession } = useSessionStore.getState();
    duplicateSession(sessions[0].id);
    const updated = useSessionStore.getState().sessions;
    expect(updated).toHaveLength(2);
    expect(updated[1].name).toBe("Original (copy)");
  });

  it("caps at MAX_SESSIONS", () => {
    const { createSession } = useSessionStore.getState();
    for (let i = 0; i < 22; i++) {
      createSession(`Session ${i}`);
    }
    expect(useSessionStore.getState().sessions.length).toBeLessThanOrEqual(20);
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx vitest run src/stores/session.store.test.ts 2>&1 | tail -20`
Expected: FAIL — module not found

- [ ] **Step 4: Implement the session store**

```typescript
// frontend/src/stores/session.store.ts
import { create } from "zustand";
import { v4 as uuidv4 } from "uuid";
import type { StudioSession } from "@/types/session.types";
import {
  MAX_SESSIONS,
  SESSIONS_STORAGE_KEY,
  ACTIVE_SESSION_KEY,
} from "@/types/session.types";
import { useFlowStore } from "@/stores/flow.store";
import { useStudioStore } from "@/stores/studio.store";
import { useStudioModeStore } from "@/stores/studio-mode.store";

type SessionStoreState = {
  sessions: StudioSession[];
  activeSessionId: string | null;

  createSession: (name?: string) => void;
  switchSession: (id: string) => void;
  renameSession: (id: string, name: string) => void;
  deleteSession: (id: string) => void;
  duplicateSession: (id: string) => void;
  saveCurrentSession: () => void;
  loadFromStorage: () => void;
};

function defaultSessionName(): string {
  const now = new Date();
  const date = now.toLocaleString(undefined, {
    year: "numeric",
    month: "2-digit",
    day: "2-digit",
    hour: "2-digit",
    minute: "2-digit",
  });
  return `Session — ${date}`;
}

function captureCurrentState(): {
  flowState: Record<string, unknown>;
  batchState: Record<string, unknown>;
  mode: "flow" | "batch";
  thumbnail?: string;
} {
  const flowRaw = useFlowStore.getState();
  // Strip function values — only serialize data fields
  const flowState = JSON.parse(
    JSON.stringify(flowRaw, (key, value) =>
      typeof value === "function" ? undefined : value,
    ),
  ) as Record<string, unknown>;

  const batchRaw = useStudioStore.getState();
  // CRITICAL: Convert excludedCombos Set to Array before serialization.
  // JSON.stringify silently converts Set to {} — breaking .has()/.add() on restore.
  const batchState = JSON.parse(
    JSON.stringify(batchRaw, (key, value) => {
      if (typeof value === "function") return undefined;
      if (value instanceof Set) return { __type: "Set", values: [...value] };
      return value;
    }),
  ) as Record<string, unknown>;

  const mode = useStudioModeStore.getState().mode;

  // Extract thumbnail from first keyframe or first image (reuse already-read state)
  let thumbnail: string | undefined;
  if (flowRaw.keyframes.length > 0 && flowRaw.keyframes[0].imageUrl) {
    thumbnail = flowRaw.keyframes[0].imageUrl;
  } else if (batchRaw.images.length > 0 && batchRaw.images[0].url) {
    thumbnail = batchRaw.images[0].url;
  }

  return { flowState, batchState, mode, thumbnail };
}

function persistToStorage(sessions: StudioSession[], activeId: string | null) {
  try {
    localStorage.setItem(SESSIONS_STORAGE_KEY, JSON.stringify(sessions));
    if (activeId) {
      localStorage.setItem(ACTIVE_SESSION_KEY, activeId);
    } else {
      localStorage.removeItem(ACTIVE_SESSION_KEY);
    }
  } catch {
    // localStorage full — silently fail, sessions still in memory
  }
}

// Initialize from localStorage synchronously so activeSessionId is set
// before any auto-save effect runs. Without this, auto-save would no-op
// because activeSessionId would be null until SessionSwitcher mounts.
function loadInitialState(): { sessions: StudioSession[]; activeSessionId: string | null } {
  try {
    const raw = localStorage.getItem(SESSIONS_STORAGE_KEY);
    const activeId = localStorage.getItem(ACTIVE_SESSION_KEY);
    if (raw) {
      return { sessions: JSON.parse(raw), activeSessionId: activeId };
    }
  } catch {
    // Corrupted storage — start fresh
  }
  return { sessions: [], activeSessionId: null };
}

const initialState = loadInitialState();

export const useSessionStore = create<SessionStoreState>()((set, get) => ({
  sessions: initialState.sessions,
  activeSessionId: initialState.activeSessionId,

  createSession: (name) => {
    // Auto-save current session first
    const { activeSessionId, sessions } = get();
    let updatedSessions = [...sessions];

    if (activeSessionId) {
      const current = captureCurrentState();
      const now = new Date().toISOString();
      updatedSessions = updatedSessions.map((s) =>
        s.id === activeSessionId
          ? { ...s, ...current, updatedAt: now }
          : s,
      );
    }

    // Create new session
    const now = new Date().toISOString();
    const newSession: StudioSession = {
      id: uuidv4(),
      name: name || defaultSessionName(),
      createdAt: now,
      updatedAt: now,
      mode: useStudioModeStore.getState().mode,
      flowState: {},
      batchState: {},
    };

    // Enforce cap — drop oldest if over limit
    updatedSessions.push(newSession);
    if (updatedSessions.length > MAX_SESSIONS) {
      updatedSessions = updatedSessions.slice(updatedSessions.length - MAX_SESSIONS);
    }

    // Reset stores
    useFlowStore.getState().resetFlow();
    useStudioStore.getState().resetSession();

    set({ sessions: updatedSessions, activeSessionId: newSession.id });
    persistToStorage(updatedSessions, newSession.id);
  },

  switchSession: (id) => {
    // Save current session first
    get().saveCurrentSession();

    const session = get().sessions.find((s) => s.id === id);
    if (!session) return;

    // Load target session's state into stores
    if (Object.keys(session.flowState).length > 0) {
      useFlowStore.setState(session.flowState as Partial<ReturnType<typeof useFlowStore.getState>>);
      // CRITICAL: reconcile stale in-flight transitions (queue/processing → failed)
      // and recompute transition array from keyframes. This mirrors onRehydrateStorage
      // behavior that normally runs at app boot.
      useFlowStore.getState().reconcileStaleTransitions();
      useFlowStore.getState().recomputeTransitions();
    } else {
      useFlowStore.getState().resetFlow();
    }

    if (Object.keys(session.batchState).length > 0) {
      // CRITICAL: Deserialize excludedCombos from Array back to Set.
      // captureCurrentState serializes Set as { __type: "Set", values: [...] }.
      const batchState = { ...session.batchState } as Record<string, unknown>;
      const excludedRaw = batchState.excludedCombos as
        | { __type: string; values: string[] }
        | undefined;
      if (excludedRaw && excludedRaw.__type === "Set") {
        batchState.excludedCombos = new Set(excludedRaw.values);
      } else {
        batchState.excludedCombos = new Set<string>();
      }
      useStudioStore.setState(batchState as Partial<ReturnType<typeof useStudioStore.getState>>);
    } else {
      useStudioStore.getState().resetSession();
    }

    useStudioModeStore.getState().setMode(session.mode);

    set({ activeSessionId: id });
    persistToStorage(get().sessions, id);
  },

  renameSession: (id, name) =>
    set((s) => {
      const sessions = s.sessions.map((sess) =>
        sess.id === id ? { ...sess, name } : sess,
      );
      persistToStorage(sessions, s.activeSessionId);
      return { sessions };
    }),

  deleteSession: (id) => {
    const { activeSessionId: currentActive, sessions: currentSessions } = get();
    const sessions = currentSessions.filter((sess) => sess.id !== id);
    const wasActive = currentActive === id;
    const newActiveId = wasActive
      ? sessions.length > 0
        ? sessions[sessions.length - 1].id
        : null
      : currentActive;

    // IMPORTANT: Only update the sessions list here — do NOT change activeSessionId yet.
    // switchSession reads activeSessionId to know which session to save current state INTO.
    // If we changed it before calling switchSession, the deleted session's state would
    // overwrite the replacement session.
    set({ sessions });
    persistToStorage(sessions, currentActive);

    if (wasActive && newActiveId) {
      // switchSession will: save current state (to currentActive — correct, it's about
      // to be removed anyway), load replacement, then set activeSessionId = newActiveId.
      get().switchSession(newActiveId);
    } else if (wasActive) {
      // No sessions left — reset everything
      set({ activeSessionId: null });
      persistToStorage(sessions, null);
      useFlowStore.getState().resetFlow();
      useStudioStore.getState().resetSession();
    }
  },

  duplicateSession: (id) =>
    set((s) => {
      const source = s.sessions.find((sess) => sess.id === id);
      if (!source) return s;
      const now = new Date().toISOString();
      const copy: StudioSession = {
        ...structuredClone(source),
        id: uuidv4(),
        name: `${source.name} (copy)`,
        createdAt: now,
        updatedAt: now,
      };
      let sessions = [...s.sessions, copy];
      if (sessions.length > MAX_SESSIONS) {
        sessions = sessions.slice(sessions.length - MAX_SESSIONS);
      }
      persistToStorage(sessions, s.activeSessionId);
      return { sessions };
    }),

  saveCurrentSession: () => {
    const { activeSessionId, sessions } = get();
    if (!activeSessionId) return;
    const current = captureCurrentState();
    const now = new Date().toISOString();
    const updatedSessions = sessions.map((s) =>
      s.id === activeSessionId
        ? { ...s, ...current, updatedAt: now }
        : s,
    );
    set({ sessions: updatedSessions });
    persistToStorage(updatedSessions, activeSessionId);
  },

  loadFromStorage: () => {
    try {
      const raw = localStorage.getItem(SESSIONS_STORAGE_KEY);
      const activeId = localStorage.getItem(ACTIVE_SESSION_KEY);
      if (raw) {
        const sessions = JSON.parse(raw) as StudioSession[];
        set({ sessions, activeSessionId: activeId });
      } else {
        // First load — create a session from current state
        const current = captureCurrentState();
        const hasContent =
          useFlowStore.getState().keyframes.length > 0 ||
          useStudioStore.getState().images.length > 0;
        if (hasContent) {
          const now = new Date().toISOString();
          const session: StudioSession = {
            id: uuidv4(),
            name: "Session 1",
            createdAt: now,
            updatedAt: now,
            ...current,
          };
          set({ sessions: [session], activeSessionId: session.id });
          persistToStorage([session], session.id);
        }
      }
    } catch {
      // Corrupted storage — start fresh
    }
  },
}));
```

- [ ] **Step 5: Export StudioStoreState type from studio.store.ts**

In `frontend/src/stores/studio.store.ts`, change the private `type StudioState` to:

```typescript
export type StudioStoreState = {
  // ... all existing fields unchanged ...
};
```

And update the `create<StudioState>()` call to `create<StudioStoreState>()`.

- [ ] **Step 6: Run tests**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx vitest run src/stores/session.store.test.ts 2>&1 | tail -30`
Expected: All tests PASS

- [ ] **Step 7: Verify compilation**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx tsc --noEmit --pretty 2>&1 | head -30`
Expected: No errors

- [ ] **Step 8: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/types/session.types.ts src/stores/session.store.ts src/stores/session.store.test.ts src/stores/studio.store.ts
git commit -m "feat(studio): add session store with localStorage persistence"
```

---

### Task 4: Session Switcher Component

**Files:**
- Create: `frontend/src/components/pages/studio/components/session-switcher.tsx`
- Create: `frontend/src/components/pages/studio/components/session-switcher.styled.tsx`
- Modify: `frontend/src/components/pages/studio/studio.page.tsx`

- [ ] **Step 1: Create styled components**

```typescript
// frontend/src/components/pages/studio/components/session-switcher.styled.tsx
import styled from "styled-components";
import { FLOW } from "@/constants/flow-theme.constants";

export const SwitcherContainer = styled.div`
  position: relative;
`;

export const SwitcherButton = styled.button`
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 6px 12px;
  background: ${FLOW.bgElevated};
  border: 1px solid ${FLOW.border};
  border-radius: 8px;
  color: ${FLOW.text};
  font-size: 13px;
  font-family: inherit;
  cursor: pointer;
  transition: border-color 0.2s;
  max-width: 200px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;

  &:hover {
    border-color: ${FLOW.borderHover};
  }
`;

export const Dropdown = styled.div`
  position: absolute;
  top: calc(100% + 4px);
  left: 0;
  z-index: 100;
  min-width: 280px;
  max-height: 400px;
  overflow-y: auto;
  background: ${FLOW.bgCard};
  border: 1px solid ${FLOW.border};
  border-radius: 12px;
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.4);
  padding: 8px;
`;

export const SessionItem = styled.div<{ $active?: boolean }>`
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 10px;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.15s;
  background: ${(p) => (p.$active ? FLOW.accentDim : "transparent")};

  &:hover {
    background: ${FLOW.bgElevated};
  }
`;

export const SessionThumb = styled.div`
  width: 36px;
  height: 24px;
  border-radius: 4px;
  overflow: hidden;
  background: ${FLOW.bgInput};
  flex-shrink: 0;

  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }
`;

export const SessionInfo = styled.div`
  flex: 1;
  min-width: 0;
`;

export const SessionName = styled.div`
  font-size: 13px;
  color: ${FLOW.text};
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
`;

export const SessionMeta = styled.div`
  font-size: 11px;
  color: ${FLOW.textMuted};
`;

export const ModeBadge = styled.span`
  font-size: 10px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.08em;
  padding: 2px 6px;
  border-radius: 4px;
  background: ${FLOW.bgInput};
  color: ${FLOW.textDim};
`;

export const DropdownActions = styled.div`
  border-top: 1px solid ${FLOW.border};
  margin-top: 4px;
  padding-top: 4px;
`;

export const ActionButton = styled.button`
  display: flex;
  align-items: center;
  gap: 8px;
  width: 100%;
  padding: 8px 10px;
  background: none;
  border: none;
  border-radius: 8px;
  color: ${FLOW.textDim};
  font-size: 13px;
  font-family: inherit;
  cursor: pointer;
  transition: background 0.15s;

  &:hover {
    background: ${FLOW.bgElevated};
    color: ${FLOW.text};
  }
`;
```

- [ ] **Step 2: Create the session switcher component**

```typescript
// frontend/src/components/pages/studio/components/session-switcher.tsx
import React, { useCallback, useEffect, useRef, useState } from "react";
import { ChevronDown, Plus, Copy, Trash2 } from "lucide-react";
import { useSessionStore } from "@/stores/session.store";
import {
  SwitcherContainer,
  SwitcherButton,
  Dropdown,
  SessionItem,
  SessionThumb,
  SessionInfo,
  SessionName,
  SessionMeta,
  ModeBadge,
  DropdownActions,
  ActionButton,
} from "./session-switcher.styled";

export const SessionSwitcher: React.FC = () => {
  const sessions = useSessionStore((s) => s.sessions);
  const activeSessionId = useSessionStore((s) => s.activeSessionId);
  const createSession = useSessionStore((s) => s.createSession);
  const switchSession = useSessionStore((s) => s.switchSession);
  const deleteSession = useSessionStore((s) => s.deleteSession);
  const duplicateSession = useSessionStore((s) => s.duplicateSession);
  const renameSession = useSessionStore((s) => s.renameSession);
  const loadFromStorage = useSessionStore((s) => s.loadFromStorage);

  const [open, setOpen] = useState(false);
  const [editingId, setEditingId] = useState<string | null>(null);
  const [editName, setEditName] = useState("");
  const containerRef = useRef<HTMLDivElement>(null);

  // Load sessions on mount
  useEffect(() => {
    loadFromStorage();
  }, [loadFromStorage]);

  // Close dropdown on outside click
  useEffect(() => {
    if (!open) return;
    const handler = (e: MouseEvent) => {
      if (containerRef.current && !containerRef.current.contains(e.target as Node)) {
        setOpen(false);
      }
    };
    document.addEventListener("mousedown", handler);
    return () => document.removeEventListener("mousedown", handler);
  }, [open]);

  const activeSession = sessions.find((s) => s.id === activeSessionId);

  const handleNew = useCallback(() => {
    createSession();
    setOpen(false);
  }, [createSession]);

  const handleSwitch = useCallback(
    (id: string) => {
      if (id !== activeSessionId) {
        switchSession(id);
      }
      setOpen(false);
    },
    [activeSessionId, switchSession],
  );

  const handleRenameStart = useCallback((id: string, currentName: string) => {
    setEditingId(id);
    setEditName(currentName);
  }, []);

  const handleRenameCommit = useCallback(() => {
    if (editingId && editName.trim()) {
      renameSession(editingId, editName.trim());
    }
    setEditingId(null);
  }, [editingId, editName, renameSession]);

  const formatDate = (iso: string) => {
    const d = new Date(iso);
    return d.toLocaleDateString(undefined, { month: "short", day: "numeric" });
  };

  return (
    <SwitcherContainer ref={containerRef}>
      <SwitcherButton onClick={() => setOpen((o) => !o)}>
        {activeSession?.name || "No Session"}
        <ChevronDown size={14} />
      </SwitcherButton>

      {open && (
        <Dropdown>
          {sessions.map((session) => (
            <SessionItem
              key={session.id}
              $active={session.id === activeSessionId}
              onClick={() => handleSwitch(session.id)}
            >
              <SessionThumb>
                {session.thumbnail && <img src={session.thumbnail} alt="" />}
              </SessionThumb>
              <SessionInfo>
                {editingId === session.id ? (
                  <input
                    autoFocus
                    value={editName}
                    onChange={(e) => setEditName(e.target.value)}
                    onBlur={handleRenameCommit}
                    onKeyDown={(e) => {
                      if (e.key === "Enter") handleRenameCommit();
                      if (e.key === "Escape") setEditingId(null);
                    }}
                    onClick={(e) => e.stopPropagation()}
                    style={{
                      background: "transparent",
                      border: "none",
                      color: "inherit",
                      font: "inherit",
                      width: "100%",
                      outline: "none",
                    }}
                  />
                ) : (
                  <SessionName
                    onDoubleClick={(e) => {
                      e.stopPropagation();
                      handleRenameStart(session.id, session.name);
                    }}
                  >
                    {session.name}
                  </SessionName>
                )}
                <SessionMeta>
                  {formatDate(session.updatedAt)}{" "}
                  <ModeBadge>{session.mode}</ModeBadge>
                </SessionMeta>
              </SessionInfo>
              <Copy
                size={14}
                style={{ opacity: 0.4, cursor: "pointer", flexShrink: 0 }}
                onClick={(e) => {
                  e.stopPropagation();
                  duplicateSession(session.id);
                }}
              />
              {sessions.length > 1 && (
                <Trash2
                  size={14}
                  style={{ opacity: 0.4, cursor: "pointer", flexShrink: 0 }}
                  onClick={(e) => {
                    e.stopPropagation();
                    deleteSession(session.id);
                  }}
                />
              )}
            </SessionItem>
          ))}
          <DropdownActions>
            <ActionButton onClick={handleNew}>
              <Plus size={14} /> New Session
            </ActionButton>
          </DropdownActions>
        </Dropdown>
      )}
    </SwitcherContainer>
  );
};
```

- [ ] **Step 3: Mount in studio header**

In `frontend/src/components/pages/studio/studio.page.tsx`, add the import:

```typescript
import { SessionSwitcher } from "./components/session-switcher";
```

In the JSX, add `<SessionSwitcher />` in the `StudioHeader` between `BackButton` and `StudioTitle`:

Find the header section and add SessionSwitcher after the BackButton:

```tsx
<StudioHeader>
  <BackButton onClick={handleBack}>
    <ArrowLeft size={18} />
  </BackButton>
  <SessionSwitcher />
  <StudioTitle>Studio</StudioTitle>
  {/* ... rest of header ... */}
</StudioHeader>
```

- [ ] **Step 4: Add auto-save hook**

Create a dedicated hook that subscribes to both stores via Zustand's `subscribe` API (not React useEffect deps, which can't track arbitrary store mutations):

```typescript
// frontend/src/components/pages/studio/hooks/useSessionAutoSave.ts
import { useEffect } from "react";
import { useFlowStore } from "@/stores/flow.store";
import { useStudioStore } from "@/stores/studio.store";
import { useSessionStore } from "@/stores/session.store";

export function useSessionAutoSave() {
  useEffect(() => {
    let timer: ReturnType<typeof setTimeout> | null = null;

    const debouncedSave = () => {
      if (timer) clearTimeout(timer);
      timer = setTimeout(() => {
        useSessionStore.getState().saveCurrentSession();
      }, 2000);
    };

    // Subscribe to both stores — fires on ANY state change
    const unsubFlow = useFlowStore.subscribe(debouncedSave);
    const unsubStudio = useStudioStore.subscribe(debouncedSave);

    return () => {
      unsubFlow();
      unsubStudio();
      if (timer) clearTimeout(timer);
    };
  }, []);
}
```

In `studio.page.tsx`, import and call the hook inside `StudioPage`:

```typescript
import { useSessionAutoSave } from "./hooks/useSessionAutoSave";

// Inside StudioPage component, after other hooks:
useSessionAutoSave();
```

- [ ] **Step 5: Verify compilation**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx tsc --noEmit --pretty 2>&1 | head -30`
Expected: No errors

- [ ] **Step 6: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/components/session-switcher.tsx src/components/pages/studio/components/session-switcher.styled.tsx src/components/pages/studio/studio.page.tsx
git commit -m "feat(studio): add session switcher to studio header with auto-save"
```

---

### Task 5: Variation Grid Component

**Files:**
- Create: `frontend/src/components/pages/studio/components/variation-grid.tsx`
- Create: `frontend/src/components/pages/studio/components/variation-grid.styled.tsx`

- [ ] **Step 1: Create styled components**

```typescript
// frontend/src/components/pages/studio/components/variation-grid.styled.tsx
import styled from "styled-components";
import { FLOW } from "@/constants/flow-theme.constants";

export const GridContainer = styled.div`
  padding: 12px 0;
  animation: fadeSlideUp 0.3s ease;

  @keyframes fadeSlideUp {
    from { opacity: 0; transform: translateY(8px); }
    to { opacity: 1; transform: translateY(0); }
  }
`;

export const Grid = styled.div<{ $columns: number }>`
  display: grid;
  grid-template-columns: repeat(${(p) => p.$columns}, 1fr);
  gap: 8px;
  max-width: 480px;
`;

export const GridCell = styled.div<{ $active?: boolean; $status?: string }>`
  position: relative;
  border-radius: 8px;
  overflow: hidden;
  border: 2px solid ${(p) => (p.$active ? FLOW.accent : FLOW.border)};
  cursor: ${(p) => (p.$status === "processed" ? "pointer" : "default")};
  transition: border-color 0.2s, transform 0.2s;
  aspect-ratio: 16 / 9;

  &:hover {
    border-color: ${(p) =>
      p.$status === "processed" ? FLOW.accent : FLOW.borderHover};
    ${(p) => p.$status === "processed" && "transform: translateY(-1px);"}
  }
`;

export const GridCellImg = styled.img`
  width: 100%;
  height: 100%;
  object-fit: cover;
`;

export const GridCellOverlay = styled.div`
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 11px;
  font-weight: 500;
`;

export const QueuedOverlay = styled(GridCellOverlay)`
  background: ${FLOW.bgElevated};
  color: ${FLOW.textMuted};
`;

export const ProcessingOverlay = styled(GridCellOverlay)`
  background: rgba(0, 0, 0, 0.5);
  color: ${FLOW.processing};
`;

export const FailedOverlay = styled(GridCellOverlay)`
  background: rgba(0, 0, 0, 0.5);
  color: ${FLOW.error};
`;

export const ActiveBadge = styled.div`
  position: absolute;
  top: 4px;
  right: 4px;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: ${FLOW.accent};
  color: ${FLOW.bg};
  font-size: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
`;

export const GridActions = styled.div`
  display: flex;
  gap: 8px;
  margin-top: 8px;
`;

export const GridActionButton = styled.button`
  padding: 6px 12px;
  background: ${FLOW.bgElevated};
  border: 1px solid ${FLOW.border};
  border-radius: 6px;
  color: ${FLOW.textDim};
  font-size: 12px;
  font-family: inherit;
  cursor: pointer;
  transition: all 0.2s;

  &:hover {
    border-color: ${FLOW.borderHover};
    color: ${FLOW.text};
  }
`;
```

- [ ] **Step 2: Create the variation grid component**

```typescript
// frontend/src/components/pages/studio/components/variation-grid.tsx
import React from "react";
import { Check } from "lucide-react";
import type { VariationCandidate } from "@/types/flow.types";
import {
  GridContainer,
  Grid,
  GridCell,
  GridCellImg,
  QueuedOverlay,
  ProcessingOverlay,
  FailedOverlay,
  ActiveBadge,
  GridActions,
  GridActionButton,
} from "./variation-grid.styled";

interface VariationGridProps {
  variations: VariationCandidate[];
  activeVariationId?: string;
  onSelect: (variationId: string) => void;
  onGenerateMore?: () => void;
  onClose: () => void;
}

function gridColumns(count: number): number {
  if (count <= 4) return 2;
  if (count <= 9) return 3;
  return 4;
}

export const VariationGrid: React.FC<VariationGridProps> = ({
  variations,
  activeVariationId,
  onSelect,
  onGenerateMore,
  onClose,
}) => {
  if (variations.length === 0) return null;

  return (
    <GridContainer>
      <Grid $columns={gridColumns(variations.length)}>
        {variations.map((v) => (
          <GridCell
            key={v.id}
            $active={v.id === activeVariationId}
            $status={v.status}
            onClick={() => {
              if (v.status === "processed") onSelect(v.id);
            }}
          >
            {v.status === "processed" && v.imageUrl && (
              <GridCellImg src={v.imageUrl} alt={v.prompt || "variation"} />
            )}
            {v.status === "queue" && <QueuedOverlay>Queued</QueuedOverlay>}
            {v.status === "processing" && (
              <ProcessingOverlay>
                {v.progress != null ? `${v.progress}%` : "..."}
              </ProcessingOverlay>
            )}
            {v.status === "failed" && <FailedOverlay>Failed</FailedOverlay>}
            {v.id === activeVariationId && v.status === "processed" && (
              <ActiveBadge>
                <Check size={10} />
              </ActiveBadge>
            )}
          </GridCell>
        ))}
      </Grid>
      <GridActions>
        {onGenerateMore && (
          <GridActionButton onClick={onGenerateMore}>+ More</GridActionButton>
        )}
        <GridActionButton onClick={onClose}>Close</GridActionButton>
      </GridActions>
    </GridContainer>
  );
};
```

- [ ] **Step 3: Verify compilation**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx tsc --noEmit --pretty 2>&1 | head -30`
Expected: No errors

- [ ] **Step 4: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/components/variation-grid.tsx src/components/pages/studio/components/variation-grid.styled.tsx
git commit -m "feat(studio): add inline variation grid component"
```

---

### Task 6: Prompt Expansion Badge Component

**Files:**
- Create: `frontend/src/components/pages/studio/components/prompt-expansion-badge.tsx`

- [ ] **Step 1: Create the badge component**

```typescript
// frontend/src/components/pages/studio/components/prompt-expansion-badge.tsx
import React from "react";
import styled from "styled-components";
import { FLOW } from "@/constants/flow-theme.constants";
import { countExpansions } from "../utils/expand-prompt";

const Badge = styled.span`
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 2px 8px;
  border-radius: 10px;
  background: ${FLOW.accentDim};
  color: ${FLOW.accent};
  font-size: 11px;
  font-weight: 600;
  white-space: nowrap;
`;

interface PromptExpansionBadgeProps {
  prompt: string;
}

export const PromptExpansionBadge: React.FC<PromptExpansionBadgeProps> = ({
  prompt,
}) => {
  const count = countExpansions(prompt);
  if (count <= 1) return null;
  return <Badge>{count} variants</Badge>;
};
```

- [ ] **Step 2: Verify compilation**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx tsc --noEmit --pretty 2>&1 | head -30`
Expected: No errors

- [ ] **Step 3: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/components/prompt-expansion-badge.tsx
git commit -m "feat(studio): add prompt expansion badge component"
```

---

### Task 7: Wire Variations into Keyframe Card

**Files:**
- Modify: `frontend/src/components/pages/studio/components/keyframe-card.tsx`

- [ ] **Step 1: Read the current keyframe-card.tsx**

Run: `cat /Users/maxcarlsonold/edream/frontend/src/components/pages/studio/components/keyframe-card.tsx`

Study the component structure. The card renders an image thumbnail with delete button and drag handle.

- [ ] **Step 2: Add variations button and grid**

Add imports at the top of keyframe-card.tsx:

```typescript
import { Shuffle } from "lucide-react";
import { useFlowStore } from "@/stores/flow.store";
import { VariationGrid } from "./variation-grid";
```

Add a "Variations" icon button to the card (near the delete button area), visible on hover. When clicked, it toggles a `showVariations` local state.

Below the keyframe card (outside the drag wrapper), conditionally render:

```tsx
{showVariations && keyframe.variations && keyframe.variations.length > 0 && (
  <VariationGrid
    variations={keyframe.variations}
    activeVariationId={keyframe.activeVariationId}
    onSelect={(variationId) => selectKeyframeVariation(keyframe.id, variationId)}
    onGenerateMore={() => {/* TODO: Task 8 wires this */}}
    onClose={() => {
      setShowVariations(false);
    }}
  />
)}
```

Add a Shuffle icon button that, when clicked, calls `onRequestVariations(keyframe.id)` (passed as prop from flow-builder). This triggers variation generation in Task 8.

- [ ] **Step 3: Verify compilation**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx tsc --noEmit --pretty 2>&1 | head -30`
Expected: No errors

- [ ] **Step 4: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/components/keyframe-card.tsx
git commit -m "feat(studio): add variations button and grid to keyframe card"
```

---

### Task 8: Wire Variation Generation into Flow Builder

**Files:**
- Modify: `frontend/src/components/pages/studio/components/flow-builder.tsx`
- Modify: `frontend/src/components/pages/studio/components/transition-settings-panel.tsx`
- Modify: `frontend/src/components/pages/studio/components/transition-gap.tsx`

This task wires the actual generation logic — creating dreams with different seeds for variations, updating variation candidates with progress, and integrating prompt expansion into "Generate All".

- [ ] **Step 1: Read the current flow-builder.tsx, transition-settings-panel.tsx, and useFlowGeneration hook**

Run:
```bash
cat /Users/maxcarlsonold/edream/frontend/src/components/pages/studio/components/flow-builder.tsx
cat /Users/maxcarlsonold/edream/frontend/src/components/pages/studio/hooks/useFlowGeneration.ts
```

Study how `generateAll` and `generateOne` work — they create dreams via `axiosClient.post` and update transition status via the flow store.

- [ ] **Step 2: Add keyframe variation generation**

In flow-builder.tsx, add a `handleKeyframeVariations` callback that:

1. Gets the keyframe's current image prompt (from the image gen params or a default)
2. Creates N (4 by default) `VariationCandidate` entries with `status: "queue"`
3. Calls `addKeyframeVariations(keyframeId, candidates)` on the flow store
4. For each candidate, creates a dream via `axiosClient.post` with a random seed
5. On success, updates the candidate via `updateKeyframeVariation` with the dream UUID and image URL

```typescript
const handleKeyframeVariations = useCallback(
  async (keyframeId: string) => {
    const kf = useFlowStore.getState().keyframes.find((k) => k.id === keyframeId);
    if (!kf) return;

    const prompt = useStudioStore.getState().imagePrompt || "abstract art";
    const model = useStudioStore.getState().imageGenParams.model;
    const size = useStudioStore.getState().imageGenParams.size;
    const count = 4;

    // Create candidate placeholders
    const candidates: VariationCandidate[] = Array.from({ length: count }, () => ({
      id: uuidv4(),
      method: "seed" as const,
      seed: Math.floor(Math.random() * 999999),
      status: "queue" as const,
    }));

    useFlowStore.getState().addKeyframeVariations(keyframeId, candidates);

    // Fire all generation requests in parallel (AGENTS.md: never use sequential for-loop)
    await Promise.all(
      candidates.map(async (candidate) => {
        try {
          useFlowStore.getState().updateKeyframeVariation(keyframeId, candidate.id, {
            status: "processing",
          });
          const promptPayload = JSON.stringify({
            infinidream_algorithm: model,
            prompt,
            size,
            seed: candidate.seed,
          });
          const { data } = await axiosClient.post(
            "/v1/dream",
            { name: `Variation ${candidate.seed}`, prompt: promptPayload },
            { headers: getRequestHeaders() },
          );
          const dreamUuid = data?.data?.dream?.uuid;
          if (dreamUuid) {
            useFlowStore.getState().updateKeyframeVariation(keyframeId, candidate.id, {
              dreamUuid,
              status: "queue",
            });
            // Poll for completion (reuse existing polling pattern from useStudioJobProgress)
          }
        } catch (err) {
          Bugsnag.notify(err instanceof Error ? err : new Error(String(err)));
          useFlowStore.getState().updateKeyframeVariation(keyframeId, candidate.id, {
            status: "failed",
          });
        }
      }),
    );
  },
  [],
);
```

Pass `handleKeyframeVariations` down to `KeyframeStrip` → `KeyframeCard` as the `onRequestVariations` prop.

- [ ] **Step 3: Add prompt expansion to transition generation**

In the `useFlowGeneration` hook (or inline in flow-builder.tsx where `generateAll` is defined), before submitting jobs, expand the prompt:

```typescript
import { expandPrompt } from "../utils/expand-prompt";

// Inside generateAll or generateOne:
const prompt = resolvedSettings.prompt;
const expanded = expandPrompt(prompt);

if (expanded.length > 1) {
  // Create variation candidates for each expansion
  const candidates: VariationCandidate[] = expanded.map((expandedPrompt) => ({
    id: uuidv4(),
    method: "expansion" as const,
    prompt: expandedPrompt,
    status: "queue" as const,
  }));
  useFlowStore.getState().addTransitionVariations(transitionIndex, candidates);

  // Generate each variant
  for (const candidate of candidates) {
    // Create dream with candidate.prompt instead of the template prompt
    // ... same pattern as existing generateOne but with candidate.prompt
  }
} else {
  // Single prompt — generate normally (existing behavior)
}
```

- [ ] **Step 4: Add expansion badge to transition settings panel**

In `transition-settings-panel.tsx`, import and render the badge next to the prompt textarea:

```typescript
import { PromptExpansionBadge } from "./prompt-expansion-badge";

// Next to the prompt label:
<PromptExpansionBadge prompt={prompt} />
```

- [ ] **Step 5: Add variations option to transition gap**

In `transition-gap.tsx`, for transitions with `status === "processed"`, add a "Variations" option (similar to the existing click-to-edit behavior). When clicked, it opens a variation grid inline.

- [ ] **Step 6: Verify compilation**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx tsc --noEmit --pretty 2>&1 | head -30`
Expected: No errors

- [ ] **Step 7: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/components/flow-builder.tsx src/components/pages/studio/components/transition-settings-panel.tsx src/components/pages/studio/components/transition-gap.tsx src/components/pages/studio/hooks/useFlowGeneration.ts
git commit -m "feat(studio): wire variation generation and prompt expansion into flow builder"
```

---

### Task 9: Prompt Expansion in Batch Mode

**Files:**
- Modify: `frontend/src/components/pages/studio/components/generate-tab.tsx`
- Modify: `frontend/src/components/pages/studio/components/actions-tab.tsx`

- [ ] **Step 1: Read current generate-tab.tsx and actions-tab.tsx**

Run:
```bash
cat /Users/maxcarlsonold/edream/frontend/src/components/pages/studio/components/generate-tab.tsx
cat /Users/maxcarlsonold/edream/frontend/src/components/pages/studio/components/actions-tab.tsx
```

Study how combinations are counted and how actions feed into the generation grid.

- [ ] **Step 2: Add expansion badge to actions tab**

In `actions-tab.tsx`, next to each action's prompt text, render the expansion badge:

```typescript
import { PromptExpansionBadge } from "./prompt-expansion-badge";

// In the action row, after the prompt display:
<PromptExpansionBadge prompt={action.prompt} />
```

- [ ] **Step 3: Expand actions in generate tab**

In `generate-tab.tsx`, where the combination count is computed, expand actions first:

```typescript
import { expandPrompt } from "../utils/expand-prompt";

// When computing combinations:
const expandedActions = enabledActions.flatMap((action) => {
  const expanded = expandPrompt(action.prompt);
  if (expanded.length <= 1) return [action];
  return expanded.map((prompt, i) => ({
    ...action,
    id: `${action.id}__exp${i}`,
    prompt,
  }));
});

// Use expandedActions instead of enabledActions for:
// - Combination count display
// - Grid column headers
// - Job submission
```

Update the combination count display to show expanded count:

```typescript
const totalCombos = selectedImages.length * expandedActions.length;
// Display: "24 combinations (3 images × 8 actions, 2 expanded)"
```

- [ ] **Step 4: Verify compilation**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx tsc --noEmit --pretty 2>&1 | head -30`
Expected: No errors

- [ ] **Step 5: Commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add src/components/pages/studio/components/generate-tab.tsx src/components/pages/studio/components/actions-tab.tsx
git commit -m "feat(studio): add prompt expansion to batch mode actions and generate tab"
```

---

### Task 10: Final Integration Test

- [ ] **Step 1: Run all tests**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx vitest run 2>&1 | tail -30`
Expected: All tests pass

- [ ] **Step 2: Run type check**

Run: `cd /Users/maxcarlsonold/edream/frontend && npx tsc --noEmit --pretty 2>&1 | tail -20`
Expected: No type errors

- [ ] **Step 3: Run lint**

Run: `cd /Users/maxcarlsonold/edream/frontend && pnpm run lint 2>&1 | tail -20`
Expected: No lint errors (or only pre-existing ones)

- [ ] **Step 4: Manual smoke test**

Start dev server: `cd /Users/maxcarlsonold/edream/frontend && pnpm run dev`

Test:
1. **Sessions:** Open studio → see session switcher in header → create new session → switch back → verify state restored
2. **Variations:** Add 2+ keyframes → click Shuffle icon on a keyframe → verify 4 images generate in inline grid → click one to swap
3. **Prompt expansion:** Type `{zoom in|orbit|drift}` in transition prompt → see "3 variants" badge → generate → verify 3 results in variation grid
4. **Batch expansion:** Go to batch mode → add action with `{fire|water} dragon` → go to generate tab → verify 2 expanded actions in grid

- [ ] **Step 5: Final commit**

```bash
cd /Users/maxcarlsonold/edream/frontend
git add -A
git commit -m "feat(studio): Phase 0 complete — sessions, variations grid, prompt expansion"
```
