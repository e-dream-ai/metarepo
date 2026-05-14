# Phase 1: Transition Generation — Design Spec

**Origin:** Studio roadmap spec (`docs/superpowers/specs/2026-05-12-studio-roadmap-design.md`), Phase 1 section. Refined via brainstorming session 2026-05-14.

**Deck reference:** `docs/presentations/2026-05-13-jay-studio-roadmap-v2.html` — slides 6 (full mockup), 7 (preset progressive reveal), 8 (per-transition editor).

**Depends on:** Phase 0 (Keyframe Strip) — `feat/phase-0-keyframe-strip` branch, PR #612.

**Goal:** Layer transition generation on top of the Phase 0 keyframe strip. Users configure global or per-transition settings, generate video segments between keyframe pairs, track progress inline, preview results, and uprez.

---

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Preset system | Reuse existing `StudioAction` presets + `buildVideoAlgoParams` | Consistent with batch mode, already model-scoped. `buildVideoAlgoParams` handles `wan-i2v` → `wan-i2v-lora` auto-upgrade and all payload construction |
| Per-transition editor placement | Panel below strip | Keeps strip layout stable, more room for controls |
| Results display | Inline on gaps | Spatial — gap becomes a living element showing transition state |
| Client-side preview | Inline player + lightbox expand | Small preview in context, click to expand. Matches existing `LightboxOverlay` pattern |
| "Generate All" with overrides | Respect per-transition overrides | Global settings fill gaps, overrides always win |
| Regeneration | Same panel, "Regenerate" button | Re-submits same params; worker generates new random seed. Phase 4 adds multi-variation comparison |

---

## Data Model

### Extended Flow Types

```typescript
// VideoModel and LoRAConfig imported from studio.types.ts (already defined there)
// import type { VideoModel, LoRAConfig } from "@/types/studio.types";

interface FlowTransition {
  fromKeyframeId: string;          // FlowKeyframe.id
  toKeyframeId: string;            // FlowKeyframe.id

  // Per-transition overrides (undefined = use global)
  presetOverride?: string;         // StudioAction preset ID
  promptOverride?: string;
  durationOverride?: number;       // seconds
  modelOverride?: VideoModel;
  loraOverride?: LoRAConfig[];     // from studio.types.ts

  // Generation state
  dreamUuid?: string;              // created dream UUID
  status: TransitionStatus;
  progress?: number;               // 0-100 during processing

  // Uprez state
  uprezDreamUuid?: string;
  uprezStatus?: TransitionStatus;
  uprezProgress?: number;
}

type TransitionStatus = "idle" | "queue" | "processing" | "processed" | "failed";
```

### Extended Flow Store State

```typescript
interface FlowStoreState {
  // Phase 0 (existing)
  keyframes: FlowKeyframe[];
  loop: boolean;

  // Phase 1 — global transition settings
  globalPresetId: string;          // StudioAction ID (default: "")
  globalPrompt: string;            // default: ""
  globalDuration: number;          // default: 5
  globalModel: VideoModel;         // default: "wan-i2v" (more presets available than ltx-i2v)
  globalNumInferenceSteps: number; // default: 30 (matches batch mode default)
  globalGuidance: number;          // default: 5.0 (matches batch mode default)

  // Phase 1 — transitions (derived from keyframe pairs)
  transitions: FlowTransition[];

  // Phase 1 — UI state
  selectedTransitionIndex: number | null;  // which gap is selected (null = global mode)
  settingsExpanded: boolean;               // collapsed vs expanded settings (preserved when switching between global/per-transition modes)

  // Phase 1 — actions
  setGlobalPreset: (id: string) => void;
  setGlobalPrompt: (prompt: string) => void;
  setGlobalDuration: (duration: number) => void;
  setGlobalModel: (model: VideoModel) => void;
  setGlobalNumInferenceSteps: (steps: number) => void;
  setGlobalGuidance: (guidance: number) => void;
  setTransitionOverride: (index: number, overrides: Partial<FlowTransition>) => void;
  clearTransitionOverride: (index: number) => void;
  selectTransition: (index: number | null) => void;
  setSettingsExpanded: (expanded: boolean) => void;
  updateTransitionStatus: (index: number, status: TransitionStatus, progress?: number) => void;
  setTransitionDream: (index: number, dreamUuid: string) => void;
  setTransitionUprez: (index: number, uprezDreamUuid: string) => void;
  recomputeTransitions: () => void;  // called after keyframe add/remove/reorder
}
```

### Transition Derivation

Transitions are derived from adjacent keyframe pairs. When keyframes change (add, remove, reorder), transitions are recomputed:

1. Build new transition list from adjacent pairs: `[kf[0]→kf[1], kf[1]→kf[2], ...]`
2. For each new pair, check if an existing transition matches the same `fromKeyframeId + toKeyframeId`
3. If match found: preserve overrides, dreamUuid, status (carry forward existing state)
4. If no match: create new transition with `status: "idle"` and no overrides

When loop is enabled, the loop transition pair uses `fromKeyframeId: keyframes[last].id` and `toKeyframeId: keyframes[0].id` — NOT `"__loop__"`. The `__loop__` synthetic keyframe only exists for visual rendering in `keyframesWithLoop()`; transitions must reference real keyframe IDs so they can be matched on reload and so `keyframeUuid` can be resolved for generation. `recomputeTransitions()` must use `keyframesWithLoop()` to discover pairs but store real IDs on the resulting transitions.

### Persistence & Hydration

Transitions are persisted in localStorage (they carry `dreamUuid`, overrides, and status that must survive page refresh). However, stale state must be handled on reload:

- **`onRehydrateStorage` callback**: after store hydration, call `recomputeTransitions()` to reconcile with current keyframes
- **Stale status reconciliation**: any transition with `status: "processing"` or `status: "queue"` on reload gets reset to `status: "failed"` (the Socket.IO connection was lost). The user can retry via "Regenerate".
- **Alternative**: instead of resetting to "failed", re-subscribe to Socket.IO rooms for any in-flight `dreamUuid`s and poll their current status. More complex but recovers gracefully if the job actually completed while the tab was closed. **Recommended for v1: reset to "failed"** — simpler, and the user can regenerate.

---

## UI Components

### 1. Enhanced Transition Gap

The gap between keyframes evolves through 6 visual states:

| State | Visual | Interaction |
|-------|--------|-------------|
| **Empty** | Dashed line (`#2a2a30`) | Click → open settings panel |
| **Configured** | Gold dashed line (`#d4a853`) + duration label | Click → open settings panel. A gap is "configured" when it has any per-transition override set (presetOverride, promptOverride, durationOverride, modelOverride, or loraOverride). |
| **Queued** | Mini thumbnail (48×34px) with "queued" text | Non-interactive |
| **Processing** | Mini thumbnail with percentage + progress bar | Non-interactive |
| **Complete** | Mini thumbnail with ✓ checkmark, green duration | Click → open preview / settings |
| **Failed** | Mini thumbnail with "!" icon, red border | Click → open settings panel (retry) |

Gap width expands from 64px (empty/configured) to 80px (queued/processing/complete/failed) to accommodate the thumbnail.

Transitions between states use `fadeSlideUp` animation (0.4s ease).

### 2. Transition Settings Panel

Appears below the keyframe strip. Two modes:

**Global mode** (no specific gap selected):
- Shown when 2+ keyframes exist and no gap is selected
- Controls apply to all transitions without overrides

**Per-transition mode** (specific gap selected):
- Header shows "Editing: nebula → crystal" with the pair names
- Controls pre-filled from override values (or global defaults if no override)
- Changes stored as overrides on that specific transition

#### Collapsed (default)

```
+- TRANSITION SETTINGS -----------------------------------------------+
|                                                                      |
|  Preset: [Slow zoom ▾]    Duration: [5s ▾]    [Generate All]       |
|                                                     ▾ Customize     |
+----------------------------------------------------------------------+
```

Fields:
- **Preset** — `<select>` populated from `StudioAction` presets, filtered by current model
- **Duration** — `<select>` with model-dependent options (LTX: 5/10/15/20s, Wan: 5/8/10s, Wan+LoRA: 5/8s). Uses existing `getAllowedDurationsForActions` logic.
- **Generate All** / **Generate** button (gold accent)
- **▾ Customize** link to expand

#### Expanded

```
+- TRANSITION SETTINGS -----------------------------------------------+
|                                                                      |
|  Preset: [Slow zoom ▾]                                  [▴ Collapse]|
|                                                                      |
|  Prompt: [slow zoom into scene, cinematic motion            ]       |
|  Duration: [5s ▾]   Model: [LTX 2.3 ▾]   LoRA: [Zoom In ▾]      |
|                                                                      |
|  ▸ Advanced (steps, guidance, CFG...)                                |
|                                                                      |
|  [Generate All]                                                      |
|                                                                      |
+----------------------------------------------------------------------+
```

Additional fields when expanded:
- **Prompt** — textarea, pre-filled from preset
- **Model** — `<select>` with LTX 2.3, Wan I2V
- **LoRA** — `<select>` filtered by selected model
- **▸ Advanced** — expandable section with steps, guidance scale, CFG (hidden by default)

Selecting a preset fills prompt and LoRAs as starting values (from `StudioAction`). Duration and model are **independent controls** — `StudioAction` does not carry these fields. Presets are filtered by the currently selected model via `PresetPack.model`. User can customize prompt/LoRAs after selecting.

#### Model / Preset / Duration Reactivity

The settings panel fields are reactive to each other. The existing `buildVideoAlgoParams` and `getAllowedDurationsForActions` functions are the source of truth — the panel calls them, not reimplements them.

**Cascade rules:**

1. **Model changes** →
   - Preset dropdown re-filters to `PresetPack`s matching the new model (via `PresetPack.model`)
   - Duration options update (LTX: 5/10/15/20s, Wan: 5/8/10s)
   - LoRA dropdown re-filters to model-appropriate LoRAs
   - If current preset is invalid for new model → clear preset selection
   - If current duration is invalid for new model → snap to closest valid duration

2. **Preset changes** →
   - Prompt textarea fills from `StudioAction.prompt`
   - LoRAs fill from `StudioAction.highNoiseLoras` / `lowNoiseLoras`
   - If preset has LoRAs (Wan camera presets) → duration clamps to 5/8s only (via `getAllowedDurationsForActions`)
   - If current duration is no longer valid → snap to closest valid duration

3. **No preset selected** (user typed custom prompt) →
   - Duration uses model defaults via `getAllowedDurationsForActions([], model)` — returns full range for the model
   - No LoRAs attached (prompt-only generation)

4. **Duration changes** → no cascading effects (duration doesn't affect other fields)

**Algorithm dispatch (invisible to user):**

The user never sees `wan-i2v-lora` as a model option. When generating, `buildVideoAlgoParams` checks if the resolved action has LoRAs:
- If yes → dispatches as `wan-i2v-lora` (with `high_noise_loras` / `low_noise_loras` in payload)
- If no → dispatches as `wan-i2v` (prompt-only payload)
- LTX always includes a LoRA (defaults to "static camera" if none specified)

**Source image field (model-dependent):**
- Wan models: `image` field (accepts keyframe image URL or UUID)
- LTX: `source_dream_uuid` field (accepts dream/image UUID)

The `useFlowGeneration` hook must map keyframe data to the correct field name per model. `buildVideoAlgoParams` already handles this — pass the keyframe's image reference as the `imageUuid` parameter.

#### Per-transition mode differences

- Header: "Editing: {fromName} → {toName}"
- Button says "Generate" (not "Generate All")
- "Regenerate" if transition is already complete
- "Reset to defaults" link — calls `clearTransitionOverride(index)` to remove all overrides, reverting to global settings
- Close button (×) to deselect and return to global mode

#### Animations

- Panel slides in with `fadeSlideUp` (0.4s ease) on first appearance
- Switching between global and per-transition mode uses a crossfade
- Collapse/expand animates height smoothly

### 3. Inline Video Preview

When all transitions are complete, a small inline video player appears below the strip (above action buttons):

```
+- PREVIEW -----------------------------------------------------------+
|                                                                      |
|  [▶ video player, 16:9 aspect, max-width 480px ]                   |
|  Click to expand                                                     |
|                                                                      |
+----------------------------------------------------------------------+
```

- **Client-side concatenation**: segments played sequentially — on `ended` event, switch `<video>` src to next segment. `MediaSource` API is an option for gapless playback but adds complexity; start with sequential and upgrade if needed
- **Click to expand**: opens a `VideoLightbox` overlay (new component, styled similarly to existing `LightboxOverlay` but wrapping a `<video>` element instead of `PresignedImage`)
- **Partial preview**: if some transitions are complete, preview plays available segments only (with placeholder gaps)

### 4. Action Buttons

Appear below the strip when any transition has results:

```
[Preview All]   [Uprez All ▾]   [Save to Playlist]
```

- **Preview All** — opens inline preview (or lightbox if already playing)
- **Uprez All** — dropdown: Nvidia Super Resolution / Current Uprez. Upscales all completed transitions
- **Save to Playlist** — saves individual segments to a playlist (preserves atomized keyframes)

Buttons use the existing flow design tokens: `bgElevated` background, `border` border, `textDim` text. "Uprez All" uses accent styling (`accentDim` background, `accent` text).

---

## Generation Flow

### Creating a transition dream

Each transition generates one dream via a two-step API sequence (the creation endpoint does not accept keyframe IDs — they're attached via update):

1. **Build payload** via `buildVideoAlgoParams({ model, action, imageUuid, imageSize, duration, numInferenceSteps, guidance })`:
   - Full signature requires: `model`, `action` (prompt + LoRAs), `imageUuid`, `imageSize`, `duration`, `numInferenceSteps`, `guidance`
   - `imageSize`: pass `undefined` — `buildVideoAlgoParams` applies `"1280*720"` as default for plain `wan-i2v` in the frontend; the field is omitted entirely for `wan-i2v-lora` and LTX. Alternatively, resolve from keyframe image dimensions if available.
   - `numInferenceSteps` and `guidance`: from global or per-transition override settings (defaults: 30, 5.0). Only used by Wan models; LTX ignores them (worker controls steps internally).
   - This existing function handles all model-specific dispatch:
     - Wan without LoRAs → `{ infinidream_algorithm: "wan-i2v", prompt, image, duration, num_inference_steps, guidance, ... }`
     - Wan with LoRAs → `{ infinidream_algorithm: "wan-i2v-lora", prompt, image, duration, high_noise_loras, low_noise_loras, ... }`
     - LTX → `{ infinidream_algorithm: "ltx-i2v", prompt, source_dream_uuid, duration, high_noise_loras, ... }`
   - **Image reference resolution**: `imageUuid` must be a dream UUID or a value the worker can resolve. `FlowKeyframe.imageUrl` is an R2 CDN URL, not a UUID. The worker's `resolveImageFromDreamUuid` handles URLs for Wan (`image` field), but LTX's `source_dream_uuid` field expects a UUID. **Implementation must resolve this**: either (a) verify the worker accepts URLs in `source_dream_uuid`, (b) use the keyframe's backend UUID and let the worker resolve the image, or (c) add a `keyframeImageUuid` field to `FlowKeyframe` that stores the image's UUID separately from the display URL. Option (b) is preferred — pass `keyframeUuid` and add a backend endpoint or worker logic to resolve keyframe → image.

2. **Create dream** via `axiosClient.post("/v1/dream", ...)` (NOT via `useCreateDreamFromPrompt` mutation hook — `useFlowGeneration` needs to fire multiple creations in a loop for "Generate All", which is incompatible with calling a `useMutation` hook repeatedly):
   - `name`: auto-generated from keyframe pair (e.g., "nebula → crystal")
   - `prompt`: JSON-stringified output from `buildVideoAlgoParams`

3. **Attach keyframes** via `axiosClient.put("/v1/dream/${uuid}", ...)`:
   - `startKeyframe`: from keyframe's `keyframeUuid`
   - `endKeyframe`: to keyframe's `keyframeUuid`

4. Store `dreamUuid` on the transition, set status to `"queue"`

5. Join Socket.IO room for progress tracking

### "Generate All" logic

```
if no transitions have status === "idle" or "failed" → button is disabled (all done or in-flight)

for each transition:
  if status === "processed" → skip (already done)
  if status === "processing" or "queue" → skip (in flight)
  resolve effective settings (override > global)
  create dream → store UUID → set status "queue"
```

### Progress tracking

New hook (`useFlowJobProgress`) modeled after `useStudioJobProgress` — a full reimplementation targeting `useFlowStore`, not a light wrapper. The batch hook hard-codes `useStudioStore` calls throughout:
- Join `dream_room` for each pending transition's `dreamUuid`
- Listen to `job:progress` Socket.IO events
- Update `progress` (0-100) and `status` on the flow store
- Polling fallback at 10s intervals for any "queue" or "processing" transitions

### Regeneration

1. User clicks a completed gap → settings panel opens in per-transition mode
2. Button says "Regenerate"
3. On click: create new dream with same effective settings. No explicit seed parameter is needed — `wan-i2v` gets a random seed from the worker automatically, and `wan-i2v-lora` passes `seed: -1` (random) via `buildVideoAlgoParams`. Re-submitting the same parameters produces a different result each time.
4. Old `dreamUuid` is replaced — previous result is discarded
5. Status resets to "queue", progress tracking begins

---

## Uprez

After transitions complete:

- **Per-transition**: click a completed gap → settings panel shows "Uprez" button
- **Uprez All**: action button below strip, submits all completed transitions

Uses existing uprez job handlers (`nvidia-uprez` / `uprez`):
1. Create uprez dream referencing the transition's `dreamUuid`
2. Store `uprezDreamUuid` on the transition
3. Track progress via same Socket.IO pattern
4. Gap visual shows uprez status (separate from generation status)

---

## Component Architecture

### New files

| File | Purpose |
|------|---------|
| `components/.../transition-gap.tsx` | Enhanced gap with 6 states, click handling |
| `components/.../transition-gap.styled.tsx` | Gap state styles, animations |
| `components/.../transition-settings-panel.tsx` | Below-strip settings (collapsed/expanded) |
| `components/.../transition-settings-panel.styled.tsx` | Panel styles |
| `components/.../flow-preview.tsx` | Inline video preview + lightbox expand |
| `components/.../flow-preview.styled.tsx` | Preview styles |
| `components/.../flow-action-bar.tsx` | Preview All / Uprez All / Save buttons |
| `components/.../flow-action-bar.styled.tsx` | Action bar styles |
| `hooks/useFlowJobProgress.ts` | Socket.IO progress tracking for flow transitions |
| `hooks/useFlowGeneration.ts` | Dream creation logic for transitions (uses `axiosClient` directly, not mutation hooks) |

### Modified files

| File | Change |
|------|--------|
| `stores/flow.store.ts` | Add transitions, global settings, Phase 1 actions. Bump persist version to 2 with v1→v2 migration: `{ transitions: [], globalPresetId: "", globalPrompt: "", globalDuration: 5, globalModel: "wan-i2v", globalNumInferenceSteps: 30, globalGuidance: 5.0, selectedTransitionIndex: null, settingsExpanded: false }`. Add `onRehydrateStorage` to reconcile stale transitions. |
| `stores/flow.store.test.ts` | Add transition derivation and override tests |
| `types/flow.types.ts` | Add FlowTransition, TransitionStatus (import VideoModel, LoRAConfig from studio.types) |
| `constants/flow-theme.constants.ts` | No changes needed (status colors already defined) |
| `components/.../flow-builder.tsx` | Render settings panel, preview, action bar |
| `components/.../keyframe-strip.tsx` | Use TransitionGap instead of plain gap |

### Existing components reused

- `LightboxOverlay` styling pattern from `images-tab.styled.tsx` — referenced for consistent overlay design (but new `VideoLightbox` created for video)
- `StudioAction` presets + `PresetPack` model scoping — for preset picker population
- `buildVideoAlgoParams` — single function for all model-specific payload construction + algorithm dispatch
- `getAllowedDurationsForActions` — reactive duration constraint logic
- Socket.IO progress pattern from `useStudioJobProgress` — reimplemented as `useFlowJobProgress` targeting flow store
- `axiosClient` — direct API calls for dream creation/update (not mutation hooks, since generation loops over multiple transitions)

---

## Interaction with Existing Systems

| System | How Phase 1 Uses It |
|--------|---------------------|
| Dream creation API | Each transition = one dream |
| Socket.IO `/remote-control` | Real-time progress per transition |
| Uprez job handlers | Post-render upscaling (nvidia-uprez, uprez) |
| StudioAction presets | Populate preset picker, pre-fill settings |
| LightboxOverlay pattern | Styling reference for VideoLightbox overlay |
| Keyframe CRUD API | Start/end keyframe references on dreams |

---

## Scope Boundaries

### In scope (Phase 1)
- Global transition settings (collapsed/expanded)
- Per-transition overrides via below-strip panel
- Generate All / individual Generate
- Inline gap status (6 states)
- Regeneration (re-submit, worker generates new seed)
- Socket.IO progress tracking
- Client-side video preview (inline + lightbox)
- Uprez (per-transition and Uprez All)
- Save to Playlist

### Out of scope (later phases)
- Inline image generation for keyframes (Phase 1 stub remains)
- Image & transition variations / multi-result comparison (Phase 4)
- Server-side FFmpeg export pipeline (Future)
- Mixed aspect ratio handling (Future)
- Bounce looping A→B→C→B→A (Future)
- Advanced LoRA training (Phase 3)
- Third-party API key configuration (Phase 3)
