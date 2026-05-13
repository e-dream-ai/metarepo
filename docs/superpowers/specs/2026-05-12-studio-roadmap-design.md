# Studio Roadmap

Studio-focused roadmap for the infinidream.ai studio product. Incorporates feedback from Jay Gidwitz (2026-05-07 and 2026-05-13 meetings) and builds on the completed model swaps project and existing keyframe infrastructure.

**Origin:** Jay Gidwitz feedback sessions (2026-05-07, 2026-05-13). Jay is a professional AI video creator who wants a polished, Pika-style studio experience with image morphing, segment re-rendering, image upload, and simplified UX that hides technical complexity.

**Reference:** Pika "Pikaframes" UI — horizontal keyframe strip with per-transition prompts, clean dark theme, progressive disclosure. Neural Frames — prompt presets, AI prompt enhancement, motion presets.

---

## Baseline (Complete)

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

## Phase 0: Keyframe Strip

The foundational UI surface for the flow builder. Gets the keyframe arrangement experience working with no generation — users can add, remove, reorder, and manage keyframes in a horizontal strip.

### Core Concept

User arranges 2-6 keyframe images in a horizontal strip. This phase builds the strip itself and the mode toggle. Generation comes in Phase 1.

### Aspect Ratio Constraint

MVP assumes uniform aspect ratio across all keyframes in a flow. No crop/resize logic. Mixed aspect ratio handling (letterboxing, smart crop, user choice) deferred to a future phase.

### UI — Empty State and Keyframe Strip

```
+---------------------------------------------------------------------------+
|  STUDIO                                    [Flow] [Batch (Advanced)]      |
+---------------------------------------------------------------------------+
|                                                                            |
|  +- KEYFRAMES --------------------------------------------------------+   |
|  |                                                                     |   |
|  |  [ + Generate ]   [ + Upload ]   [ + From Playlist ]               |   |
|  |                                                                     |   |
|  +---------------------------------------------------------------------+   |
|                                                                            |
+----------------------------------------------------------------------------+
```

With keyframes added:

```
+---------------------------------------------------------------------------+
|  STUDIO                                    [Flow] [Batch (Advanced)]      |
+---------------------------------------------------------------------------+
|                                                                            |
|  +- KEYFRAMES --------------------------------------------------------+   |
|  |                                                                     |   |
|  |  +--------+   ------   +--------+   ------   +--------+            |   |
|  |  |        |            |        |            |        |            |   |
|  |  | [img1] |            | [img2] |            | [img3] |            |   |
|  |  |        |            |        |            |        |            |   |
|  |  +--------+            +--------+            +--------+            |   |
|  |     drag to reorder                                                 |   |
|  |                                                             [ ] Loop|   |
|  |  [+ Generate]  [+ Upload]  [+ From Playlist]                       |   |
|  |                                                                     |   |
|  +---------------------------------------------------------------------+   |
|                                                                            |
+----------------------------------------------------------------------------+
```

### Adding Keyframes

Three methods, all available from the keyframe strip:

| Method | How |
|--------|-----|
| **Generate** | Opens inline image gen (Z Image Turbo / Qwen — uses existing studio image gen) |
| **Upload** | Native file picker or drag & drop anywhere in studio (jpg, png, webp) |
| **From Playlist** | Opens existing `add-from-playlist-modal` to import keyframes |

Uploaded images passed as-is to video models (no crop/resize — LTX and Wan handle arbitrary input).

### Keyframe Management

- **Delete:** Hover over a keyframe to reveal an X button. Click to remove. Loop keyframe cannot be deleted (remove by unchecking Loop).
- **Reorder:** Drag keyframes to new positions in the strip. Loop keyframe is excluded from reorder — always stays at the end.

### Drag & Drop

Drag and drop is supported in two ways:

1. **Reorder keyframes** — drag a keyframe to a new position in the strip
2. **Upload images** — drag image files from desktop onto the studio (anywhere on the page). Drop zone visual feedback (border highlight or overlay). Creates keyframe via existing multipart API.

### Loop Mode

Checkbox in the keyframe strip area: `[ ] Loop`

When enabled, a copy of the first keyframe auto-appends as the final position in the strip:

```
+--------+   ------   +--------+   ------   +--------+   ------   +--------+
|        |            |        |            |        |            | locked  |
| [img1] |            | [img2] |            | [img3] |            | [img1]  |
|        |            |        |            |        |            |  Loop   |
+--------+            +--------+            +--------+            +--------+
                                                                    dimmed
```

- Loop keyframe is **visible but locked** — slightly dimmed, labeled "Loop"
- Cannot be edited, deleted, or dragged
- Unchecking the Loop checkbox removes the loop keyframe
- Loop keyframe updates automatically if the first keyframe is changed or swapped
- Loop keyframe cannot be reordered — always stays at the end
- If keyframes are dragged to reorder, the loop frame still mirrors whichever keyframe is first

### Mode Toggle

Top of studio page: `[Flow]  [Batch (Advanced)]`

- **Flow** (default) — the keyframe flow builder described here
- **Batch (Advanced)** — the existing combinatorial workflow (Images, Actions, Generate, Results tabs)
- Switching modes does not destroy state — each mode's state is independent in the Zustand store

### Data Model (Phase 0 Subset)

Flow state is stored in Zustand (persisted to localStorage), not a new backend entity:

```typescript
interface StudioFlow {
  keyframes: FlowKeyframe[];       // ordered list
  loop: boolean;                   // when true, last keyframe mirrors first
}

interface FlowKeyframe {
  id: string;                      // local ID for drag/drop
  keyframeUuid: string;            // backend keyframe UUID
  imageUrl: string;                // presigned URL for display
  isLoopKeyframe?: boolean;        // true for the auto-generated loop frame
}
```

When `loop` is true, a synthetic `FlowKeyframe` with `isLoopKeyframe: true` is appended, mirroring the first keyframe's `keyframeUuid` and `imageUrl`. This keyframe is excluded from drag/drop reorder and delete operations.

### Interaction with Existing Systems

| Existing System | How Phase 0 Uses It |
|-----------------|----------------------|
| Keyframe CRUD API | Create keyframes (upload, generate) |
| Playlist-keyframe ordering | Import keyframes from playlists |
| Studio image gen | Inline image generation for keyframes |

---

## Phase 1: Transition Generation

Adds the generation pipeline on top of the Phase 0 keyframe strip. Preset-driven transition settings, per-transition generate/regenerate, results tracking, client-side preview, and uprez.

### UI — Progressive Reveal (Continued)

When 2+ keyframes are present, transition settings fade in below the keyframe strip (collapsed by default):

```
|  +- TRANSITION SETTINGS (collapsed) ---------------------------------+    |
|  |                                                                    |    |
|  |  Preset: [Slow zoom v]    Duration: [5s v]    [Generate All]      |    |
|  |                                                                    |    |
|  +--------------------------------------------------------------------+    |
```

Expanded:

```
|  +- TRANSITION SETTINGS ---------------------------------------------+    |
|  |                                                                    |    |
|  |  Preset: [Slow zoom v]                                [Collapse ^] |    |
|  |                                                                    |    |
|  |  Prompt: [slow zoom into scene, cinematic motion  ]               |    |
|  |  Duration: [5s v]   Model: [LTX 2.3 v]   LoRA: [Zoom In v]      |    |
|  |                                                                    |    |
|  |  > Advanced (steps, guidance, CFG...)                              |    |
|  |                                                                    |    |
|  |  [Generate All]                                                    |    |
|  |                                                                    |    |
|  +--------------------------------------------------------------------+    |
```

Generation started — results section fades in:

```
|  +- RESULTS ----------------------------------------------------------+   |
|  |                                                                     |   |
|  |  Transition 1    Transition 2    Transition 3                       |   |
|  |  [OK thumb]      [~ 45%]        [. queued]                          |   |
|  |  img1 -> img2    img2 -> img3   img3 -> img4                        |   |
|  |                                                                     |   |
|  +---------------------------------------------------------------------+   |
```

All transitions complete — export options appear:

```
|  +- RESULTS ----------------------------------------------------------+   |
|  |                                                                     |   |
|  |  Transition 1    Transition 2    Transition 3                       |   |
|  |  [OK thumb]      [OK thumb]     [OK thumb]                          |   |
|  |  img1 -> img2    img2 -> img3   img3 -> img4                        |   |
|  |                                                                     |   |
|  |  [Preview All]   [Uprez All v]   [Save to Playlist]                |   |
|  |                   |-- Nvidia SR                                     |   |
|  |                   '-- Current Uprez                                 |   |
|  |                                                                     |   |
|  +---------------------------------------------------------------------+   |
```

### Transition Settings — Progressive Reveal

The transition settings bar uses its own progressive disclosure:

- **Collapsed (default):** Preset picker, duration, and generate button. Minimal cognitive load for the common case — pick a preset and go.
- **Expanded:** Full controls — prompt text, model, LoRA, and advanced settings. Selecting a preset pre-fills all expanded fields. If you never expand, you just pick a preset + duration and hit generate.
- Presets are populated from existing `StudioAction` presets (already model-scoped via Baseline work). Selecting a preset fills prompt, LoRA, and duration fields as a starting point — user can customize after.

Same progressive pattern applies to per-transition overrides (see below).

### Per-Transition Overrides

Clicking the gap between two keyframes opens an inline editor for that specific transition:

```
+--------+                              +--------+
|        |   +--------------------+     |        |
| [img1] |   | Preset: [Slow v]   |    | [img2] |
|        |   | > Customize        |    |        |
+--------+   | [Generate]    [x]  |    +--------+
             +--------------------+
```

Expanded:

```
+--------+                              +--------+
|        |   +--------------------+     |        |
| [img1] |   | Preset: [Slow v]   |    | [img2] |
|        |   | prompt: [.........]|    |        |
+--------+   | duration: [5s v]   |    +--------+
             | model: [LTX 2.3 v] |
             | [Generate]    [x]  |
             +--------------------+
```

- Override prompt, duration, model, and/or LoRA for just that transition
- Individual **Generate** button to render just this one transition
- Close (x) to collapse back to the minimal gap view

### Per-Transition Regeneration (Segment Re-rendering)

Clicking a completed transition's thumbnail reopens the inline editor with the result visible:

```
+--------+                              +--------+
|        |   +--------------------+     |        |
| [img1] |   | [OK video thumb  ]|     | [img2] |
|        |   | Preset: [Slow v]   |    |        |
+--------+   | > Customize        |    +--------+
             | [Regenerate]  [x]  |
             +--------------------+
```

- Shows current result as playable thumbnail
- Fields pre-filled with what was used
- **Regeneration uses a new random seed** — the user "pulls the lever" for a different result from the same inputs. The seed is not exposed in the UI.
- Edit prompt, duration, or model, then hit **Regenerate** — new job replaces the old result
- Only the selected segment re-renders; all other transitions stay unchanged
- The final transition (e.g., img3 -> Loop) generates like any other, creating a seamless loop

### Generate Behavior

- **Individual Generate/Regenerate** on a transition — render just that one
- **Generate All** — submits all transitions that don't already have a completed result (skips done ones)
- After regenerating a segment, "Generate All" still skips it (it's now completed)

### Video Compilation & Export

**Phase 1 scope — client-side preview:** Concatenate segment video blobs in the browser for instant playback. No API call needed. Users can preview the full flow and share/download the concatenated result directly.

**Deferred — server-side export:** Upscale and concatenate in a single FFmpeg pass via the video service. This avoids double-encoding artifacts at segment joins. Deferred until client-side concat proves insufficient or uprez+compile-in-one-pass demand materializes. See Future Enhancements.

Stage 4 buttons for Phase 1:

- **Preview All** — client-side concat playback
- **Uprez All** — upscale each segment independently (existing uprez handlers)
- **Save to Playlist** — save individual segments to a playlist (preserves atomized keyframes for future branching)

### Uprez

After transitions complete:

- **Uprez All** — upscales every completed transition independently
- Individual transitions can also be uprezed one at a time
- Model picker: Nvidia Super Resolution / Current Uprez (same pattern as batch workflow)
- Progress shows inline via existing Socket.IO tracking
- Uses existing `nvidia-uprez` / `uprez` job handlers

### Data Model (Extended from Phase 0)

Phase 1 extends the Phase 0 Zustand store with transition and generation state:

```typescript
interface StudioFlow {
  keyframes: FlowKeyframe[];       // ordered list (from Phase 0)
  transitions: FlowTransition[];   // derived: one per adjacent keyframe pair
  globalPreset?: string;           // action preset ID
  globalPrompt: string;
  globalDuration: number;
  model: VideoModel;               // "ltx-i2v" | "wan-i2v"
  loop: boolean;                   // from Phase 0
}

interface FlowKeyframe {
  id: string;                      // local ID for drag/drop
  keyframeUuid: string;            // backend keyframe UUID
  imageUrl: string;                // presigned URL for display
  isLoopKeyframe?: boolean;        // true for the auto-generated loop frame
}

interface FlowTransition {
  fromKeyframeId: string;          // FlowKeyframe.id
  toKeyframeId: string;            // FlowKeyframe.id
  presetOverride?: string;         // action preset ID, if set overrides global
  promptOverride?: string;         // if set, overrides globalPrompt
  durationOverride?: number;       // if set, overrides globalDuration
  modelOverride?: VideoModel;      // if set, overrides global model
  dreamUuid?: string;              // generated transition dream
  status: StudioJobStatus;         // existing type: "queue" | "processing" | "processed" | "failed"
  uprezDreamUuid?: string;
}
```

Transitions are recomputed when keyframes are added, removed, or reordered. Existing transition state (overrides, dream results) is preserved when possible by matching on the `fromKeyframeId + toKeyframeId` pair.

Each transition generates a dream with `startKeyframeId` set (existing dream field). The transition's prompt is passed as the dream's prompt.

### Interaction with Existing Systems

| Existing System | How Phase 1 Uses It |
|-----------------|----------------------|
| Dream creation API | Each transition = one dream |
| Socket.IO progress | Real-time render progress per transition |
| Uprez job handlers | Post-render upscaling |
| Action presets | Prompt templates in transition settings |
| Output playlists | Save completed flow to playlist |

---

## Phase 2: Image Upload (Batch Mode)

Drag & drop and upload button support in batch mode's Images tab.

**Note:** Flow mode upload is delivered as part of Phase 0. This phase covers ensuring the same capability works in batch mode.

### Scope

- **Flow mode:** Upload button + drag & drop (delivered in Phase 0)
- **Batch mode Images tab:** Add "Upload Image" button alongside "Generate Images"
- Drag & drop anywhere in studio works regardless of active mode

### Implementation

- Upload creates a keyframe via existing `POST /keyframe` + multipart upload API
- In batch mode, uploaded keyframe is converted to a `StudioImage` entry (same as generated images)
- No new backend work — frontend wiring to existing endpoints
- Accept: jpg, png, webp
- Images passed as-is to video models (no crop/resize)

---

## Phase 3: New Image Models & API Key Configuration

Add Flux (Black Forest Labs) and OpenAI image generation as studio image gen options, plus a generic API key configuration system for user-provided endpoints.

### Flux (Black Forest Labs)

- Closest open-source competitor to MidJourney (per Jay's recommendation)
- Integration pattern TBD — likely RunPod public endpoint (same as Z Image Turbo) or direct API
- New `ImageModel` value: `"flux-image"`
- Worker: new job handler for Flux API
- Frontend: add to model picker dropdown
- Size options per model capabilities
- Supports LoRA training deltas (small files, not full checkpoints) — see Future Enhancements

### OpenAI Image Gen (gpt-image-1)

- Different integration pattern — calls OpenAI API directly, not RunPod
- New `ImageModel` value: `"openai-image"`
- Worker: new job handler hitting OpenAI API
- Frontend: add to model picker dropdown

### For Both

- Frontend: new entries in `ImageModel` type, `SIZE_OPTIONS`, model picker labels
- Worker: new job handler per model
- Backend: recognize new algorithm names in dream creation
- Pricing/rate limiting considerations TBD

### Generic API Key Configuration

Users can configure their own model endpoints in **account settings**. Configured endpoints appear as options in studio model dropdowns.

```
+- ACCOUNT SETTINGS > API Keys ----------------------------------------+
|                                                                        |
|  +- Configured Endpoints ------------------------------------------+  |
|  |                                                                  |  |
|  |  "My OpenAI"     gpt-image-1    ********sk-...Bf2  [edit][x]   |  |
|  |  "Local Flux"    flux-image     ********fl-...a91  [edit][x]   |  |
|  |                                                                  |  |
|  |  [+ Add Endpoint]                                                |  |
|  +------------------------------------------------------------------+  |
|                                                                        |
|  +- Add Endpoint ---------------------------------------------------+  |
|  |  Name:     [My Replicate        ]                                |  |
|  |  Type:     [OpenAI-compatible v ]                                |  |
|  |  Endpoint: [https://api.openai.com/v1  ]                        |  |
|  |  API Key:  [sk-...             ]                                 |  |
|  |  Model ID: [gpt-image-1        ]                                |  |
|  |                                                                  |  |
|  |  [Test Connection]    [Save]                                     |  |
|  +------------------------------------------------------------------+  |
+------------------------------------------------------------------------+
```

- Named endpoints appear as options in studio model dropdowns
- Flux and OpenAI can also be platform-provided (no user key needed) if we host our own keys
- "Test Connection" validates the endpoint before saving

### API Key Security

API keys are sensitive credentials and must be handled with care:

- **Encryption at rest:** AES-256-GCM encryption for all stored API keys
- **Encryption secret:** Stored as env var (`API_KEY_ENCRYPTION_SECRET`), never in DB
- **Key rotation:** Dual-secret window strategy — new secret encrypts on next read/write, old secret kept as `API_KEY_ENCRYPTION_SECRET_PREVIOUS` for decryption during rollover period
- **API responses:** Never expose full key — only last 4 characters shown in UI
- **Access control:** Keys scoped to user — no cross-user access via API or DB queries
- **Backend entity:** New `UserApiEndpoint` entity with encrypted `apiKey` column, plus `name`, `type`, `endpoint`, `modelId`, `userId`

### Research Needed Before Implementation

- Flux: RunPod public endpoint availability, API format, supported sizes, pricing
- OpenAI: gpt-image-1 API format, size options, pricing per image
- Both: latency characteristics (sync vs async pattern)

---

## Phase 4: Image & Transition Variations

MidJourney-style variations — given an existing keyframe or completed transition, generate similar-but-different versions. Adapts the existing results grid from batch mode into a linear inline format for flow mode.

### Concept

Image-to-image generation: feed an existing image to a model and get back visually related but distinct alternatives. This avoids the repetitive artifacts Jay described from re-running the same prompt. Transition variations re-render a segment to produce alternative results.

### UX — Inline Linear Grid

Adapts the existing `ResultCell` / `ResultThumb` / `ResultCellStatus` styled components from batch mode's results tab into a horizontal strip that appears inline in the flow builder.

**Keyframe variations** — click a keyframe, choose "Variations":

```
+--------+   ------   +--------+   ------   +--------+
|        |   5s       |        |   5s       |        |
| [img2] |            | [img3] |            | [img4] |
|        |            |   v    |            |        |
+--------+            +--------+            +--------+
                      +--------------------------------+
                      |  +------+ +------+ +------+    |
                      |  | v1 OK| | v2   | | v3   |   |
                      |  |[thumb]| |[thumb]| |[thumb]|   |
                      |  +------+ +------+ +------+    |
                      |  click to swap in    [+ More]  |
                      +--------------------------------+
```

**Transition variations** — click a completed transition, choose "Variations":

```
+--------+                              +--------+
|        |   +--------------------+     |        |
| [img1] |   | [current result  ] |    | [img2] |
|        |   | +----++----++----+ |    |        |
+--------+   | | v1 || v2 || v3 | |    +--------+
             | +----++----++----+ |
             | click to swap  [+] |
             +--------------------+
```

- Reuses `ResultCell`, `ResultThumb`, `ResultCellStatus` styled components from batch mode
- Same progress tracking (percentage, queued, failed states)
- Each keyframe variation is a separate img2img job (or seed variation for models without native img2img)
- Each transition variation is a regeneration with a different random seed
- Clicking a variation swaps it into the active slot; original preserved (can revert)
- "[+ More]" generates additional variations
- Transition variations re-render only that segment — other segments untouched

### Model Support

| Model | Variation Support |
|-------|-------------------|
| Flux | Native img2img / remix (best candidate for keyframe variations) |
| OpenAI | Image editing/variation endpoints |
| Z Image Turbo | TBD — may support image conditioning |
| Qwen | TBD — may only support prompt-based regeneration with different seeds |

### Dependency

Depends on Phase 3 — the best variation results come from Flux and OpenAI which have native variation support. Existing models may only support prompt-based regeneration with different seeds (which is what we have today).

Implementation details deferred until Phase 3 model integrations clarify which variation approaches work best.

---

## UI Polish (Threaded Through All Phases)

Not a separate phase. Design principles applied to every feature as it ships.

### Design Language (Established in Phase 0)

- **Dark theme, minimal chrome** — Pika-style clean aesthetic
- **Progressive disclosure** — hide complexity until needed (transition settings start collapsed, advanced controls behind expandable section)
- **Generous spacing, refined typography** — no cramped controls
- **Smooth transitions** — fade/slide animations for progressive reveal of sections
- **Consistent component library** — Phase 0 establishes shared styled components, subsequent phases inherit

### Advanced Controls

Technical settings (inference steps, guidance scale, CFG, LoRA strengths) hidden behind an expandable "Advanced" section within the expanded transition settings. Not shown by default. Jay's "brain icon" concept — most users never see these controls, power users can access them.

### Aesthetic Standard

Jay described wanting an "Apple-like" interface — polished enough that the tool feels like part of the creative process rather than fighting against it. Every component shipped should meet this bar. Reference: Pika's Pikaframes UI for tone and density.

### Incremental Adoption

- Phase 0 (keyframe strip) ships with the new design language — this is a new UI surface so there's no legacy to reconcile
- Existing batch workflow tabs get a visual refresh when touched (not a separate effort)
- Shared styled components ensure consistency across flow and batch modes

---

## Future Enhancements (Unscoped)

Items confirmed as valuable but not assigned to a phase. To be prioritized as earlier phases ship.

| Enhancement | Description | Origin |
|---|---|---|
| **AI Prompt Enhancement** | Expand short prompts into detailed ones — send user's brief prompt to an LLM for elaboration (Neural Frames "Pimp my Prompt" pattern) | May 13 meeting, Neural Frames research |
| **Prompt Inspiration ("Suggest")** | Generate 3 prompt + camera/motion suggestions based on selected keyframes and context (Neural Frames "Prompt Sparkle" pattern) | May 13 meeting, Neural Frames research |
| **LoRA Training Deltas** | Users upload small training files (not full checkpoints) to customize models. val.ai/Flux pattern — lightweight fine-tuning without hosting 2GB checkpoint files | Jay, May 13 meeting |
| **Mixed Aspect Ratios** | Handle keyframes with different aspect ratios — crop/resize/letterbox logic | May 13 meeting (deferred) |
| **Atomized Keyframe Sharing** | Keep flow segments as individual keyframes that others can branch from, instead of compiling into one video. Enables cross-user creative branching via shared keyframes. | May 13 meeting (deferred, MVP compiles) |
| **Bounce Looping** | Loop mode that reverses (A->B->C->B->A) in addition to circular (A->B->C->A) | May 13 meeting |
| **Saved Prompt Library** | Store and reuse prompt + parameter combinations across flows | Neural Frames research |
| **Server-Side Export Pipeline** | `POST /v1/flow/export` — upscale + concatenate in a single FFmpeg pass via video service. Avoids double-encoding artifacts at segment joins. Accepts ordered dream UUIDs + uprez model. Skips upscaling if segments already uprezed. | May 13 meeting (deferred, client-side concat sufficient for MVP) |

---

## Prompt Tips Reference

Research from Neural Frames community on writing effective AI video/animation prompts. Useful for future prompt template curation and AI prompt enhancement features.

### Prompt Structure Approaches

- **Scene sketching (Stormcrow):** Broad overview -> subject -> specifics -> finishing touches. Start with camera shot type, then describe who/what is in frame, clothing, colors, and finally the art style or medium.
- **Component breakdown (Scott Haynes):** Subject -> action -> environment -> medium (photograph, illustration, painting) -> art style (research specific styles rather than artist names) -> cinematic elements (angles, shots).
- **Literal with atmosphere (First Syndrome):** Literal descriptions combined with decay/distortion details. Example: "bowl of cereal in mid-century kitchen, bleached film, polaroid aesthetic, filmed in 1976."

### Key Techniques

- **Negative prompts:** Exclude unwanted elements to sharpen focus on desired themes. For monochromatic scenes, counterbalance with vibrant colors in negative prompts.
- **Modular word collections:** Curate reusable lists of patterns, textures, styles, colors, objects, actions, scenarios. Organize by category. Combine elements for prompt generation. Prevents creative blocks.
- **Iterative refinement:** Run short test renders (15-30 seconds) with new models. Adjust mid-render. Swap individual elements while preserving structure for cohesive series.
- **Embrace absurdity:** Nonsensical scenarios often produce interesting results. Some of the best results come from two-word prompts.

### What to Include

- Camera direction (shot types, angles, perspective)
- Specific visual descriptors (bleached film, polaroid, vintage)
- Art medium (photography style, digital illustration, painting technique)
- Color and texture (explicit color choices, surface qualities)
- Stylistic references (specific decades, equipment types)
- Music/audio-driven intensity (when applicable)

Source: [Neural Frames — Writing the Perfect AI Prompts](https://www.neuralframes.com/post/writing-the-perfect-ai-prompts-what-our-users-say)

---

## Roadmap Summary

| Phase | What | Dependencies | Key Deliverable |
|-------|------|--------------|-----------------|
| -- | Baseline | None | Model swaps + keyframe data model (complete) |
| 0 | Keyframe Strip | Baseline | Keyframe strip UI, add/delete/reorder, upload, generate, from-playlist, loop mode, drag & drop, mode toggle |
| 1 | Transition Generation | Phase 0 | Preset-driven transition settings, per-transition generate/regenerate, results tracking, client-side preview + concat, uprez, progressive reveal |
| 2 | Image Upload (Batch) | Phase 0 | Drag & drop + upload in batch mode (flow mode covered by Phase 0) |
| 3 | New Image Models + API Keys | Baseline | Flux + OpenAI image gen, generic API key config (encrypted at rest), user-provided endpoints |
| 4 | Image & Transition Variations | Phase 3 | Inline linear variation grid for keyframes and transitions, adapted from batch results grid |
| -- | UI Polish | -- | Threaded through all phases, design language established in Phase 0 |

Phases 2 and 3 are independent of each other and can be worked in parallel. Phase 4 depends on Phase 3.

---

## References

- Jay Gidwitz feedback meeting notes (2026-05-07): [Google Doc](https://docs.google.com/document/d/1ralgAsdqsHPnjfjz1TPhnGOJlbpzchckWI7i4EK1BcU/edit?tab=t.t832szx2kndc)
- Jay Gidwitz follow-up meeting transcript (2026-05-13): confirmed rerendering strategy, loop mode, flow builder design, prompt templates, API key config, video compilation approach
- Pika Pikaframes UI (reference screenshot from Jay)
- Neural Frames — prompt presets, AI prompt enhancement, motion presets: [neuralframes.com](https://www.neuralframes.com/)
- Neural Frames — Writing the Perfect AI Prompts: [neuralframes.com/post/writing-the-perfect-ai-prompts](https://www.neuralframes.com/post/writing-the-perfect-ai-prompts-what-our-users-say)
- Neural Frames — What is Prompting: [help.neuralframes.com](https://help.neuralframes.com/en/articles/10552726-what-is-prompting)
- Model Swaps & VAE Preview spec: `docs/superpowers/specs/2026-03-30-model-swaps-vae-preview-design.md`
- Model Workflows Reference: `docs/superpowers/specs/2026-03-30-model-workflows-reference.md`
- Visual Creator Workflows Design (original vision): `metarepo/docs/plans/2026-01-30-visual-creator-workflows-design.md`
- Studio Page MVP: `metarepo/docs/plans/2026-02-16-studio-page-mvp.md`
