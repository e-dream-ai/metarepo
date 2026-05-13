# Studio Roadmap

Studio-focused roadmap for the infinidream.ai studio product. Incorporates feedback from Jay Gidwitz (2026-05-07 meeting) and builds on the completed model swaps project and existing keyframe infrastructure.

**Origin:** Jay Gidwitz feedback session (2026-05-07). Jay is a professional AI video creator who wants a polished, Pika-style studio experience with image morphing, segment re-rendering, image upload, and simplified UX that hides technical complexity.

**Reference:** Pika "Pikaframes" UI — horizontal keyframe strip with per-transition prompts, clean dark theme, progressive disclosure.

---

## Phase 0: Baseline (Complete)

All model swaps and keyframe infrastructure have landed on stage/main.

### Model Swaps (Shipped)

| Deliverable | Status | Repo |
|---|---|---|
| Z Image Turbo (worker + backend + frontend) | Merged | worker, backend, frontend#587 |
| LTX 2.3 I2V (handler, queue, ComfyUI workflow, gpu-container) | Merged | worker#31, backend#412, gpu-container-ltx |
| Nvidia VSR upscaling (handler, queue, gpu-container) | Merged | worker#31, backend#412, gpu-container-nvidia-vsr |
| VAE preview (TAESD in gpu-container-ltx) | Merged | gpu-container-ltx |
| Frontend model pickers (image, video, uprez) | Merged | frontend#587 |
| Store refactors (imageGenParams v3, videoGenParams v4 migrations) | Merged | frontend#587 |
| Action preset model scoping (Wan vs LTX filtering) | Merged | frontend#587 |
| LTX Camera action presets (Lightricks LoRAs) | Merged | frontend#587 |

### Keyframe Data Model (Shipped)

| Deliverable | Status |
|---|---|
| Keyframe entity (UUID, name, image, owner, soft delete) | In production |
| Keyframe CRUD API (`/api/v1/keyframe/`) | In production |
| Multipart image upload for keyframes | In production |
| Playlist-keyframe junction with ordering | In production |
| Dream start/end keyframe references | In production |
| Frontend: create, view/edit, playlist keyframes tab | In production |
| Keyframe selector component in dream editor | In production |

---

## Phase 1: Flow Builder

The studio's new default experience. A Pika-style keyframe arrangement UI that replaces the combinatorial workflow as the primary mode. The existing batch workflow (Images, Actions, Generate, Results) moves behind an "Advanced / Batch Mode" toggle.

### Core Concept

User arranges 2-6 keyframe images in a horizontal strip. The system generates video transitions between each adjacent pair. One global prompt governs all transitions, with optional per-transition overrides for prompt and duration.

### UI Design: Progressive Reveal

The interface starts minimal and reveals sections as they become relevant:

**Stage 1 — Empty state:** Only the keyframe strip with add buttons.

```
┌─────────────────────────────────────────────────────────────────────────┐
│  STUDIO                                    [Flow] [Batch (Advanced)]   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─ KEYFRAMES ──────────────────────────────────────────────────────┐  │
│  │                                                                   │  │
│  │  [ + Generate ]   [ + Upload ]   [ + From Playlist ]             │  │
│  │                                                                   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Stage 2 — 2+ keyframes added:** Transition settings appear.

```
┌─────────────────────────────────────────────────────────────────────────┐
│  STUDIO                                    [Flow] [Batch (Advanced)]   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─ KEYFRAMES ──────────────────────────────────────────────────────┐  │
│  │                                                                   │  │
│  │  ┌────────┐   ╌╌╌╌╌╌   ┌────────┐   ╌╌╌╌╌╌   ┌────────┐       │  │
│  │  │        │   5s      │        │   5s      │        │       │  │
│  │  │ [img1] │           │ [img2] │           │ [img3] │       │  │
│  │  │        │           │        │           │        │       │  │
│  │  └────────┘           └────────┘           └────────┘       │  │
│  │     drag to reorder                                          │  │
│  │                                                               │  │
│  │  [+ Generate]  [+ Upload]  [+ From Playlist]                 │  │
│  │                                                               │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                       │
│  ┌─ TRANSITION SETTINGS ────────────────────────────────────────┐    │
│  │                                                               │    │
│  │  Global prompt: [smooth transition, cinematic motion    ]    │    │
│  │                                                               │    │
│  │  Duration: [5s ▼]  Model: [LTX 2.3 ▼]  [Generate All]       │    │
│  │                                                               │    │
│  └───────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Stage 3 — Generation started:** Results section fades in.

```
│  ┌─ RESULTS ────────────────────────────────────────────────────────┐  │
│  │                                                                   │  │
│  │  Transition 1    Transition 2    Transition 3                    │  │
│  │  [✓ thumb]       [⟳ 45%]        [· queued]                      │  │
│  │  img1 → img2     img2 → img3    img3 → img4                     │  │
│  │                                                                   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
```

**Stage 4 — All transitions complete:** Uprez and export options appear.

```
│  ┌─ RESULTS ────────────────────────────────────────────────────────┐  │
│  │                                                                   │  │
│  │  Transition 1    Transition 2    Transition 3                    │  │
│  │  [✓ thumb]       [✓ thumb]       [✓ thumb]                       │  │
│  │  img1 → img2     img2 → img3    img3 → img4                     │  │
│  │                                                                   │  │
│  │  [Preview All]   [Uprez All ▼]   [Save to Playlist]             │  │
│  │                   |-- Nvidia SR                                   │  │
│  │                   '-- Current Uprez                               │  │
│  │                                                                   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
```

### Per-Transition Overrides

Clicking the gap between two keyframes opens an inline editor for that specific transition:

```
┌────────┐                            ┌────────┐
│        │   ┌────────────────────┐   │        │
│ [img1] │   │ prompt: [.........]│   │ [img2] │
│        │   │ duration: [5s ▼]   │   │        │
└────────┘   │ [Generate]    [×]  │   └────────┘
             └────────────────────┘
```

- Override prompt and/or duration for just that transition
- Individual **Generate** button to render just this one transition
- Close (x) to collapse back to the minimal gap view

### Per-Transition Regeneration (Segment Re-rendering)

Clicking a completed transition's thumbnail reopens the inline editor with the result visible:

```
┌────────┐                            ┌────────┐
│        │   ┌────────────────────┐   │        │
│ [img1] │   │ [✓ video thumb   ]│   │ [img2] │
│        │   │ prompt: [.........]│   │        │
└────────┘   │ duration: [5s ▼]   │   └────────┘
             │ [Regenerate]  [×]  │
             └────────────────────┘
```

- Shows current result as playable thumbnail
- Fields pre-filled with what was used
- Edit prompt or duration, hit **Regenerate** — new job replaces the old result
- Only the selected segment re-renders; all other transitions stay unchanged

### Generate Behavior

- **Individual Generate/Regenerate** on a transition — render just that one
- **Generate All** — submits all transitions that don't already have a completed result (skips done ones)
- After regenerating a segment, "Generate All" still skips it (it's now completed)

### Adding Keyframes

Three methods, all available from the keyframe strip:

| Method | How |
|--------|-----|
| **Generate** | Opens inline image gen (Z Image Turbo / Qwen — uses existing studio image gen) |
| **Upload** | Native file picker or drag & drop anywhere in studio (jpg, png, webp) |
| **From Playlist** | Opens existing `add-from-playlist-modal` to import keyframes |

Uploaded images passed as-is to video models (no crop/resize — LTX and Wan handle arbitrary input).

### Drag & Drop

Drag and drop is supported in two ways:

1. **Reorder keyframes** — drag a keyframe to a new position in the strip
2. **Upload images** — drag image files from desktop onto the studio (anywhere on the page). Drop zone visual feedback (border highlight or overlay). Creates keyframe via existing multipart API.

### Uprez

After transitions complete:

- **Uprez All** — upscales every completed transition using selected model
- Individual transitions can also be uprezed independently
- Model picker: Nvidia Super Resolution / Current Uprez (same pattern as batch workflow)
- Progress shows inline via existing Socket.IO tracking
- Uses existing `nvidia-uprez` / `uprez` job handlers

### Data Model

Flow state is stored in Zustand (persisted to localStorage), not a new backend entity:

```typescript
interface StudioFlow {
  keyframes: FlowKeyframe[];       // ordered list
  transitions: FlowTransition[];   // derived: one per adjacent keyframe pair
  globalPrompt: string;
  globalDuration: number;
  model: VideoModel;               // "ltx-i2v" | "wan-i2v"
}

interface FlowKeyframe {
  id: string;                      // local ID for drag/drop
  keyframeUuid: string;            // backend keyframe UUID
  imageUrl: string;                // presigned URL for display
}

interface FlowTransition {
  fromKeyframeId: string;          // FlowKeyframe.id
  toKeyframeId: string;            // FlowKeyframe.id
  promptOverride?: string;         // if set, overrides globalPrompt
  durationOverride?: number;       // if set, overrides globalDuration
  dreamUuid?: string;              // generated transition dream
  status: StudioJobStatus;         // existing type: "queue" | "processing" | "processed" | "failed"
  uprezDreamUuid?: string;
}
```

Transitions are recomputed when keyframes are added, removed, or reordered. Existing transition state (overrides, dream results) is preserved when possible by matching on the `fromKeyframeId + toKeyframeId` pair.

Each transition generates a dream with `startKeyframeId` set (existing dream field). The transition's prompt is passed as the dream's prompt.

### Interaction with Existing Systems

| Existing System | How Flow Builder Uses It |
|-----------------|--------------------------|
| Keyframe CRUD API | Create keyframes (upload, generate) |
| Playlist-keyframe ordering | Import keyframes from playlists |
| Dream creation API | Each transition = one dream |
| Socket.IO progress | Real-time render progress per transition |
| Uprez job handlers | Post-render upscaling |
| Studio image gen | Inline image generation for keyframes |
| Output playlists | Save completed flow to playlist |

### Mode Toggle

Top of studio page: `[Flow]  [Batch (Advanced)]`

- **Flow** (default) — the keyframe flow builder described here
- **Batch (Advanced)** — the existing combinatorial workflow (Images, Actions, Generate, Results tabs)
- Switching modes does not destroy state — each mode's state is independent in the Zustand store

---

## Phase 2: Image Upload to Studio

Drag & drop and upload button support for bringing external images into the studio.

**Note:** This is largely delivered as part of Phase 1's flow builder. This phase covers ensuring the same upload capability works in batch mode's Images tab.

### Scope

- **Flow mode:** Upload button + drag & drop (delivered in Phase 1)
- **Batch mode Images tab:** Add "Upload Image" button alongside "Generate Images"
- Drag & drop anywhere in studio works regardless of active mode

### Implementation

- Upload creates a keyframe via existing `POST /keyframe` + multipart upload API
- In batch mode, uploaded keyframe is converted to a `StudioImage` entry (same as generated images)
- No new backend work — frontend wiring to existing endpoints
- Accept: jpg, png, webp
- Images passed as-is to video models (no crop/resize)

---

## Phase 3: New Image Models (Flux, OpenAI)

Add Flux (Black Forest Labs) and OpenAI image generation as studio image gen options.

### Flux (Black Forest Labs)

- Closest open-source competitor to MidJourney (per Jay's recommendation)
- Integration pattern TBD — likely RunPod public endpoint (same as Z Image Turbo) or direct API
- New `ImageModel` value: `"flux-image"`
- Worker: new job handler for Flux API
- Frontend: add to model picker dropdown
- Size options per model capabilities

### OpenAI Image Gen (gpt-image-1)

- Different integration pattern — calls OpenAI API directly, not RunPod
- New `ImageModel` value: `"openai-image"`
- Worker: new job handler hitting OpenAI API
- New env var: `OPENAI_API_KEY`
- Frontend: add to model picker dropdown

### For Both

- Frontend: new entries in `ImageModel` type, `SIZE_OPTIONS`, model picker labels
- Worker: new job handler per model
- Backend: recognize new algorithm names in dream creation
- Pricing/rate limiting considerations TBD

### Research Needed Before Implementation

- Flux: RunPod public endpoint availability, API format, supported sizes, pricing
- OpenAI: gpt-image-1 API format, size options, pricing per image
- Both: latency characteristics (sync vs async pattern)

---

## Phase 4: Image Variations

MidJourney-style variations — given an existing image, generate similar-but-different versions.

### Concept

Image-to-image generation: feed an existing image to a model and get back visually related but distinct alternatives. This avoids the repetitive artifacts Jay described from re-running the same prompt.

### UX

- Right-click or long-press a keyframe (flow mode) or image (batch mode) to access "Generate Variations"
- Produces 4-8 alternatives in a popover or mini-grid
- Click one to swap it into that slot
- Original image is preserved (can revert)

### Model Support

| Model | Variation Support |
|-------|-------------------|
| Flux | Native img2img / remix (best candidate) |
| OpenAI | Image editing/variation endpoints |
| Z Image Turbo | TBD — may support image conditioning |
| Qwen | TBD — may only support prompt-based regeneration with different seeds |

### Dependency

Depends on Phase 3 — the best variation results come from Flux and OpenAI which have native variation support. Existing models may only support prompt-based regeneration with different seeds (which is what we have today).

Implementation details deferred until Phase 3 model integrations clarify which variation approaches work best.

---

## UI Polish (Threaded Through All Phases)

Not a separate phase. Design principles applied to every feature as it ships.

### Design Language (Established in Phase 1)

- **Dark theme, minimal chrome** — Pika-style clean aesthetic
- **Progressive disclosure** — hide complexity until needed
- **Generous spacing, refined typography** — no cramped controls
- **Smooth transitions** — fade/slide animations for progressive reveal of sections
- **Consistent component library** — Phase 1 establishes shared styled components, subsequent phases inherit

### Advanced Controls

Technical settings (inference steps, guidance scale, CFG, LoRA strengths) hidden behind an expandable "Advanced" section. Not shown by default. Jay's "brain icon" concept — most users never see these controls, power users can access them.

### Aesthetic Standard

Jay described wanting an "Apple-like" interface — polished enough that the tool feels like part of the creative process rather than fighting against it. Every component shipped should meet this bar. Reference: Pika's Pikaframes UI for tone and density.

### Incremental Adoption

- Phase 1 (flow builder) ships with the new design language — this is a new UI surface so there's no legacy to reconcile
- Existing batch workflow tabs get a visual refresh when touched (not a separate effort)
- Shared styled components ensure consistency across flow and batch modes

---

## Roadmap Summary

| Phase | What | Dependencies | Key Deliverable |
|-------|------|--------------|-----------------|
| 0 | Baseline | None | Model swaps + keyframe data model (complete) |
| 1 | Flow Builder | Phase 0 | Pika-style keyframe strip, per-transition generate/regenerate, uprez, progressive reveal |
| 2 | Image Upload (Batch) | Phase 1 | Drag & drop + upload in batch mode (flow mode covered by Phase 1) |
| 3 | New Image Models | Phase 0 | Flux + OpenAI image gen integration |
| 4 | Image Variations | Phase 3 | img2img variations on existing keyframes |
| -- | UI Polish | -- | Threaded through all phases, design language established in Phase 1 |

Phases 2 and 3 are independent of each other and can be worked in parallel. Phase 4 depends on Phase 3.

---

## References

- Jay Gidwitz feedback meeting notes (2026-05-07): [Google Doc](https://docs.google.com/document/d/1ralgAsdqsHPnjfjz1TPhnGOJlbpzchckWI7i4EK1BcU/edit?tab=t.t832szx2kndc)
- Pika Pikaframes UI (reference screenshot from Jay)
- Model Swaps & VAE Preview spec: `docs/superpowers/specs/2026-03-30-model-swaps-vae-preview-design.md`
- Model Workflows Reference: `docs/superpowers/specs/2026-03-30-model-workflows-reference.md`
- Visual Creator Workflows Design (original vision): `metarepo/docs/plans/2026-01-30-visual-creator-workflows-design.md`
- Studio Page MVP: `metarepo/docs/plans/2026-02-16-studio-page-mvp.md`
