# Visual Creator Workflows Design

Design document for production workflows that leverage existing backend video generation/processing to make content creation easier for AI artists and experimenters.

## Overview

### Target Persona
AI artists and experimenters who want rapid iteration, parameter exploration, and serendipitous discovery. These creators value quick feedback loops, easy parameter tweaking, and the ability to explore many variations before committing to expensive renders.

### Core Concept: Explore → Refine → Render

Three-tier quality ladder enabling fast iteration at low cost:

| Tier | Resolution | Speed | Cost | Purpose |
|------|------------|-------|------|---------|
| **Draft** | 256p | Seconds | ~Free | Generate 10-50 variations, find directions |
| **Preview** | 480p | Minutes | Low | Tune parameters, compare, build sequences |
| **Final** | 1080p+ | Longer | Full | Production output, uprez + concatenate |

Parameters and prompts are preserved across tiers—upgrading from draft to preview to final is one click.

---

## MVP: Combinatorial Workflow for AI Video Creators

### Target Persona: AI Video Specialists

Multimedia creators and AI video specialists who produce music videos, short films, and promotional content. They:
- Have deep experience with visual storytelling (20+ years in some cases)
- Use AI tools like Wan, ComfyUI, LTX2, and image generators as part of their workflow
- Need to iterate quickly through many visual variations
- Value control over the creative process, not just "generate and hope"
- Produce professional output for clients, artists, and personal projects
- Have generated thousands of pieces of content and need **organization above all**
- Work with shot lists and pre-production planning before generation

**Example output:** Music videos, AI commercials, abstract art films, branded content, podcast visualizations.

### Key User Feedback

From conversations with professional AI video creators:

> "The most important thing is organization. I call it workflow, call it whatever you want, but it's organization."

> "As a video editor and producer, I want [the shot list] BEFORE. If I had that in a ComfyUI workflow, that would be the very first thing I do."

> "I've made almost 10,000 pieces of content and what you're introducing is a way to see what I'm doing. The way I'm doing it now is literally one at a time."

> "My biggest gripe is the filename that comes out of it. In ComfyUI, I control the filename."

**Performance reference:** Mid-Journey's image gallery is cited as "the fastest loading gallery I've ever seen on a website" - gallery performance matters for users with thousands of assets.

### The Workflow

The qwen → wan → uprez workflow is the ideal MVP for creator tools. It exercises the core pattern (generate → select → combine → refine) without requiring the full node editor or timeline.

### The Workflow

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  QWEN IMAGES    │     │    WAN I2V      │     │     UPREZ       │
│  N seeds        │────▶│  N × M videos   │────▶│  enhance finals │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                       │
        ▼                       ▼
   select favorites      select favorites
   add more seeds        add more actions
```

**Key characteristics:**
- Non-linear: add images or actions at any time, not strictly sequential
- Combinatorial: 3 images × 3 actions = 9 videos
- Iterative: review/select at each stage before proceeding

### MVP UI: Studio Page (`/studio`)

New page on existing infinidream.ai frontend, using existing backend APIs. The page has a tab-based layout allowing the creator to focus on one stage at a time while maintaining context.

---

#### Screen 1: Images Tab

Generate source images with Qwen, or import from existing playlists.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  STUDIO                              [Images] [Actions] [Generate] [Results]│
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─ GENERATE NEW IMAGES ────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  Prompt:                                                              │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │  │
│  │  │ a mystical forest creature in watercolor style, soft lighting,  │ │  │
│  │  │ detailed fur texture, enchanted forest background               │ │  │
│  │  └─────────────────────────────────────────────────────────────────┘ │  │
│  │                                                                       │  │
│  │  Seeds: [8 ▼]    Size: [1280x720 ▼]    [Generate Images]            │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─ IMAGE LIBRARY ──────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐  │  │
│  │  │   ★    │ │        │ │   ★    │ │        │ │   ★    │ │   ⟳    │  │  │
│  │  │ [img1] │ │ [img2] │ │ [img3] │ │ [img4] │ │ [img5] │ │ 34%... │  │  │
│  │  │seed:42 │ │seed:43 │ │seed:44 │ │seed:45 │ │seed:46 │ │seed:47 │  │  │
│  │  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘ └────────┘  │  │
│  │                                                                       │  │
│  │  ┌────────┐ ┌────────┐                                               │  │
│  │  │   ·    │ │   ·    │  ← pending jobs                               │  │
│  │  │queued  │ │queued  │                                               │  │
│  │  └────────┘ └────────┘                                               │  │
│  │                                                                       │  │
│  │  3 selected for animation                 [+ Add from Playlist]      │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  [Continue to Actions →]                                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Element descriptions:**

| Element | What it does |
|---------|--------------|
| `Prompt` textarea | Multi-line text for Qwen image generation. Supports detailed descriptions. |
| `Seeds: [8 ▼]` | How many images to generate (1, 4, 8, 12, 16, 24). Each gets unique random seed. |
| `Size: [1280x720 ▼]` | Image dimensions. Options: 1280×720, 1024×1024, 720×1280, 512×512. |
| `[Generate Images]` | Queues N Qwen jobs. New images appear in library below as they complete. Can click multiple times to generate more batches. |
| Image cards | Show thumbnail, seed number, status. Click thumbnail to expand. Click ★ to select for I2V. |
| `★` (star) | Toggles image selection. Selected images (yellow star) will be used in video generation. |
| `⟳ 34%` | Image currently generating. Shows progress percentage. |
| `· queued` | Job waiting in queue, not yet started. |
| `3 selected` | Live count of starred images. |
| `[+ Add from Playlist]` | Opens playlist browser modal (see below). |
| `[Continue to Actions →]` | Navigates to Actions tab. Also accessible via tab bar. |

**Add from Playlist Modal:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  ADD IMAGES FROM PLAYLIST                                        [×]       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Select playlist: [My Creature Studies ▼]                                  │
│                                                                             │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐        │
│  │   ☑    │ │   ☑    │ │   ☐    │ │   ☐    │ │   ☑    │ │   ☐    │        │
│  │ [img]  │ │ [img]  │ │ [img]  │ │ [img]  │ │ [img]  │ │ [img]  │        │
│  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘ └────────┘        │
│                                                                             │
│  3 images selected                                                          │
│                                                                             │
│  [Cancel]                                                [Add Selected]    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

Allows importing images from any of the creator's existing playlists. Useful for:
- Reusing images from previous sessions
- Mixing hand-crafted images with generated ones
- Building on previous work

---

#### Screen 2: Actions Tab

Define motion/action prompts that will be applied to each selected image.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  STUDIO                              [Images] [Actions] [Generate] [Results]│
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─ ACTION PROMPTS ─────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  These prompts describe camera motion or transformations. Each       │  │
│  │  selected image will be animated with each enabled action.           │  │
│  │                                                                       │  │
│  │  ┌───┬───────────────────────────────────────────────────────┬─────┐ │  │
│  │  │ ☑ │ slow zoom in, camera gently pushing forward           │ [×] │ │  │
│  │  ├───┼───────────────────────────────────────────────────────┼─────┤ │  │
│  │  │ ☑ │ pan left to right, revealing the scene gradually      │ [×] │ │  │
│  │  ├───┼───────────────────────────────────────────────────────┼─────┤ │  │
│  │  │ ☑ │ orbit around subject, 180 degrees, smooth motion      │ [×] │ │  │
│  │  ├───┼───────────────────────────────────────────────────────┼─────┤ │  │
│  │  │ ☐ │ float upward, ascending into clouds slowly            │ [×] │ │  │
│  │  ├───┼───────────────────────────────────────────────────────┼─────┤ │  │
│  │  │ ☐ │ gentle breathing motion, subtle life                  │ [×] │ │  │
│  │  └───┴───────────────────────────────────────────────────────┴─────┘ │  │
│  │                                                                       │  │
│  │  [+ Add Action]                              [Load Preset Pack ▼]    │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─ SUMMARY ────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  3 images selected  ×  3 actions enabled  =  9 videos                │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  [← Back to Images]                              [Continue to Generate →]  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Element descriptions:**

| Element | What it does |
|---------|--------------|
| `☑` checkbox | Enables/disables action for current batch. Unchecked actions are saved but not used. |
| Action text field | Editable inline. Click to edit, Enter to save. Describes motion/transformation. |
| `[×]` delete button | Removes action from list entirely. |
| `[+ Add Action]` | Adds new empty row at bottom. Focus moves to text field for immediate typing. |
| `[Load Preset Pack ▼]` | Dropdown with pre-made action sets (see below). Appends to existing list. |
| Summary | Live calculation showing total videos that will be generated. |

**Camera Motion Presets** (inspired by LTX2's built-in LoRAs):

| Preset Pack | Actions Included |
|-------------|------------------|
| **Camera Basics** | zoom in, zoom out, pan left, pan right, pan up, pan down, push in, pull out |
| **Cinematic** | dolly forward, orbit 180°, crane up, tracking shot, rack focus |
| **Organic** | gentle breathing, subtle sway, floating drift, heartbeat pulse |
| **Abstract** | morph and flow, color shift, kaleidoscope spin, fractal zoom |

**Behind the curtain: Actions → LoRAs**

The user-facing "action" concept maps to LoRAs on the backend. Users don't need to know about LoRAs - they just pick actions:

| User sees | Backend uses |
|-----------|--------------|
| "zoom in" | `wan22_14b_i2v_zoom_in.safetensors` |
| "orbit 180°" | `wan22_14b_i2v_orbit_high_noise.safetensors` + `_low_noise.safetensors` |
| "pan left" | `wan22_14b_i2v_pan_left.safetensors` |
| Custom text | Passed as prompt, no LoRA (or combined with LoRA) |

This abstraction:
- Hides technical complexity from creators
- Allows mixing presets (LoRAs) with custom prompts
- Can evolve to support new LoRAs without UI changes
- "LoRAs seem to steer the scene more than the prompt" - presets are more reliable than text-only

**Good action prompts** (for custom/text-based actions):
- Describe camera movement: "slow zoom", "pan left", "orbit", "dolly in"
- Add style/feel: "smooth motion", "gentle", "dramatic", "ethereal"
- Be specific: "180 degrees", "gradually", "subtle"
- Combine motions: "slow zoom while panning left"

---

#### Screen 3: Generate Tab

Preview all combinations and launch the batch.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  STUDIO                              [Images] [Actions] [Generate] [Results]│
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─ COMBINATION PREVIEW ────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  Review all image × action combinations before generating.           │  │
│  │  Uncheck any you want to skip.                                        │  │
│  │                                                                       │  │
│  │            │ zoom in       │ pan left      │ orbit 180°    │         │  │
│  │  ──────────┼───────────────┼───────────────┼───────────────┤         │  │
│  │  creature1 │ ☑ [thumb]     │ ☑ [thumb]     │ ☑ [thumb]     │         │  │
│  │  seed:42   │               │               │               │         │  │
│  │  ──────────┼───────────────┼───────────────┼───────────────┤         │  │
│  │  creature3 │ ☑ [thumb]     │ ☐ [thumb]     │ ☑ [thumb]     │         │  │
│  │  seed:44   │               │ (skip)        │               │         │  │
│  │  ──────────┼───────────────┼───────────────┼───────────────┤         │  │
│  │  creature5 │ ☑ [thumb]     │ ☑ [thumb]     │ ☑ [thumb]     │         │  │
│  │  seed:46   │               │               │               │         │  │
│  │                                                                       │  │
│  │  8 of 9 combinations selected                                         │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─ OUTPUT SETTINGS ────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  Duration: [5 seconds ▼]     Steps: [30 ▼]     Guidance: [5.0 ▼]    │  │
│  │                                                                       │  │
│  │  Output playlist: [Creature Study Batch 02 ▼]         [+ Create New]    │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  [← Back to Actions]                                    [Generate All →]   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Element descriptions:**

| Element | What it does |
|---------|--------------|
| Combination grid | Shows every image × action pair. Click checkbox to include/exclude specific combos. |
| `☐ (skip)` | Unchecked combinations won't be generated. Useful when certain pairings don't make sense. |
| `8 of 9 selected` | Live count of combinations that will be generated. |
| `Duration: [5s ▼]` | Video length for all generated clips: 3s, 5s, 7s, 10s. |
| `Steps: [30 ▼]` | Inference steps (quality vs speed): 20, 25, 30, 40. Higher = better but slower. |
| `Guidance: [5.0 ▼]` | How closely to follow prompt: 3.0 (loose) to 7.0 (strict). |
| `Output playlist` | Where results will be saved. Select existing or create new. |
| `[+ Create New]` | Creates playlist with auto-name like "Studio 2026-02-13 14:30". |
| `[Generate All →]` | **Main action button.** Queues all checked combinations as Wan I2V jobs. Navigates to Results tab. |

---

#### Screen 4: Results Tab

Monitor progress and review completed videos.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  STUDIO                              [Images] [Actions] [Generate] [Results]│
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─ BATCH PROGRESS ─────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  Batch: "Creature Study Batch 02"              5 of 8 complete           │  │
│  │  ████████████████████░░░░░░░░  62%         ~4 min remaining          │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─ RESULTS MATRIX ─────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │            │ zoom in       │ pan left      │ orbit 180°    │         │  │
│  │  ──────────┼───────────────┼───────────────┼───────────────┤         │  │
│  │            │    ★          │               │    ★          │         │  │
│  │  creature1 │ [✓ thumb]     │ [✓ thumb]     │ [⟳ 67%  ]     │         │  │
│  │            │  click→view   │  click→view   │  rendering    │         │  │
│  │  ──────────┼───────────────┼───────────────┼───────────────┤         │  │
│  │            │               │               │               │         │  │
│  │  creature3 │ [✓ thumb]     │    skipped    │ [⟳ 23%  ]     │         │  │
│  │            │  click→view   │               │  rendering    │         │  │
│  │  ──────────┼───────────────┼───────────────┼───────────────┤         │  │
│  │            │    ★          │               │               │         │  │
│  │  creature5 │ [✓ thumb]     │ [· queued]    │ [· queued]    │         │  │
│  │            │  click→view   │  waiting      │  waiting      │         │  │
│  │                                                                       │  │
│  │  ★ = marked for uprez                                                │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌─ ACTIONS ────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  [Uprez Selected (3)]    [Retry Failed]    [View Playlist]          │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Element descriptions:**

| Element | What it does |
|---------|--------------|
| Progress bar | Overall batch progress. Shows count and percentage. |
| `~4 min remaining` | Estimated time based on average job duration. |
| Results matrix | Rows = images, Columns = actions. Each cell shows job status and thumbnail when complete. |
| Cell: `[✓ thumb]` | Completed successfully. Shows video thumbnail. Click to open in new tab (infinidream.ai/dream/:uuid). |
| Cell: `[⟳ 67%]` | Currently rendering. Shows progress percentage. Updates in real-time via Socket.IO. |
| Cell: `[· queued]` | Waiting in queue. Will start when GPU becomes available. |
| Cell: `skipped` | Combination was unchecked in Generate tab. Grayed out. |
| `★` star on cell | Click to mark video for uprez. Yellow star appears. Can star while still rendering. |
| `[Uprez Selected (3)]` | Queues uprez jobs for all starred videos. Count shows how many selected. Opens uprez options modal. |
| `[Retry Failed]` | Re-queues any failed jobs with same parameters. Only visible if failures exist. |
| `[View Playlist]` | Opens output playlist in new tab. All results are automatically added there. |

**Uprez Options Modal** (Phase 2):

Different upscalers work better for different content types:

```
┌─────────────────────────────────────────────────────────────────┐
│  UPREZ OPTIONS                                       [×]        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Upscaler:                                                      │
│  ◉ Auto-detect (AI analyzes content)                           │
│  ○ Realistic (photos, live action)                             │
│  ○ Animation (cartoons, anime, illustrated)                    │
│  ○ Abstract (patterns, textures, AI art)                       │
│                                                                 │
│  Scale: [2x ▼]    Interpolation: [2x ▼]                        │
│                                                                 │
│  [Cancel]                                    [Uprez 3 Videos]  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Cell click behavior:**
- Completed: Opens dream view page in new tab
- Rendering: Shows larger preview with live progress
- Queued: Shows position in queue

**Tab notifications:**

The tab bar shows completion badges so the creator can work on other tabs while renders complete:

```
[Images] [Actions] [Generate] [Results (3 new ✓)]
```

- Badge appears when jobs complete while viewing another tab
- Shows count of newly completed since last viewed
- Clears when Results tab is opened
- Optional: Browser notification when batch completes (if enabled)

---

#### Non-linear Workflow Support

The key insight from this workflow: **it's not strictly sequential**. At any point:

| From | You can | What happens |
|------|---------|--------------|
| Results tab | Go back to Images | Add more images, they appear as new rows |
| Results tab | Go back to Actions | Add more actions, they appear as new columns |
| Any tab | Generate more | Only new combinations are queued (existing ones skipped) |

**Example flow:**
1. Generate 4 images, select 2, define 3 actions → 6 videos
2. Review results, 2 are great, 4 are meh
3. Go back to Images, generate 4 more, select 1 new favorite
4. Go to Generate → now shows 3 images × 3 actions, but 6 already done
5. Click Generate → only 3 new videos queued (the new image × 3 actions)

This is powered by `BATCH_IDENTIFIER` tracking - each image+action combo gets a unique hash stored in the dream description, so the system knows what's already been generated.

---

### Typical Creator Session

1. **Start**: Open `/studio`, fresh session
2. **Image prompt**: "bioluminescent deep sea creature, intricate scales, dark oceanic background, volumetric lighting"
3. **Generate 8 images**: Click `[Generate Images]` with seeds=8
4. **Wait ~30s**: Watch images appear as Qwen jobs complete
5. **Select 3 favorites**: Click ★ on the best ones
6. **Define actions**: Switch to Actions tab, add:
   - "slow zoom into the eye, mysterious atmosphere"
   - "creature slowly rotates, revealing bioluminescent patterns"
   - "gentle floating motion, particles drifting in current"
7. **Review combos**: Switch to Generate tab, see 3×3=9 grid, uncheck 1 that doesn't work
8. **Generate**: Click `[Generate All →]`, watch Results tab
9. **Monitor**: 8 videos rendering over ~10 minutes, progress updating live
10. **Review**: Click completed videos to watch, ★ the 3 best
11. **Iterate**: Back to Images, generate 4 more seeds, select 1 new good one
12. **Generate again**: Only 3 new videos queued (new image × 3 actions)
13. **Uprez**: Click `[Uprez Selected (4)]` to enhance winners
14. **Done**: All results in playlist, share or continue

### Implementation Plan

**Phase 1: Core UI (frontend only, uses existing APIs)**

| Component | Purpose | Existing API |
|-----------|---------|--------------|
| `StudioPage` | Main container, state management | - |
| `ImageGenerator` | Qwen batch with seed count | `POST /dream` with `qwen-image` |
| `ImageSelector` | Grid with starring, pull from playlist | `GET /playlist/:id/items` |
| `ActionList` | CRUD for action prompts | Local state (or localStorage) |
| `CombinationPreview` | Show N×M grid before submitting | Local computation |
| `BatchSubmitter` | Queue all combinations | `POST /dream` × N×M |
| `ResultsGrid` | Matrix view of jobs, live status | `GET /dream/:id`, Socket.IO |

**Phase 2: Backend optimizations**

| Enhancement | Purpose |
|-------------|---------|
| Batch dream creation endpoint | Single API call for N dreams |
| Batch progress aggregation | One Socket.IO subscription for batch |
| Skip existing combinations | Check `BATCH_IDENTIFIER` server-side |

**Phase 3: Persistence & workflow**

| Enhancement | Purpose |
|-------------|---------|
| Save/load studio sessions | Resume work across sessions |
| Action presets | Reusable action libraries (camera motions, style transforms) |
| Workflow templates | "Music Video I2V Pipeline" as one-click setup |
| Shot list import | Load pre-planned shots from external source |
| Custom naming/metadata | Control over filenames and organization |

**Phase 4: AI-assisted features**

| Enhancement | Purpose |
|-------------|---------|
| Preview thumbnails during render | Show low-quality (20%) preview as job progresses (like LTX2's Preview VAE) |
| Smart upscaler selection | LLM analyzes content → suggests upscaler (realistic vs animation vs illustration) |
| Duplicate/dead detection | LLM filters outputs: remove duplicates, flag smeared/failed generations |
| Auto-prompt suggestions | LLM generates action prompt variations based on image content |
| Professional/Hobbyist modes | Simplified UI for casual users, full controls for power users |

**Note on LLM integration:** Avoid cloud LLMs with heavy content filtering (e.g., Gemini) for analysis - they may refuse to process legitimate creative content. Consider local models (Gemma) or permissive APIs.

**Future model integration:**
- **LTX2** - Extremely fast (50 videos/hour locally), has Preview VAE and 9 camera motion LoRAs built-in. Not yet on RunPod but worth monitoring.
- **Seed Dance** - New model from China with impressive results. Watch for availability.

### Design Considerations

**Mobile-first navigation:**
- Left sidebar navigation (top-to-bottom) translates well to iPhone/iPad
- Tab bar across top for main sections
- Should work from iPad while rendering on desktop
- "I can click it and it'll go like add to playlist or make new playlist like the same thing that YouTube has"

**Gallery performance:**
- Reference: Mid-Journey's gallery loads thousands of images instantly
- Lazy loading, virtualized lists for large libraries
- Thumbnail generation on ingest

**Professional vs Hobbyist modes** (Phase 3):
- Hobbyist: Simplified UI, fewer options, guided workflow
- Professional: Full parameter access, shot list import, batch controls

### Why This Works as MVP

1. **Uses existing APIs** - no backend changes for Phase 1
2. **Exercises core pattern** - generate, select, combine, refine
3. **Real user workflow** - AI video creators are already doing this manually via scripts
4. **Incremental path** - can evolve into full node editor later
5. **Validates preview system** - batch progress grid needs Socket.IO streaming
6. **Organization first** - addresses the #1 pain point for power users

### Technical Notes

**Frontend state structure:**
```typescript
interface StudioState {
  // Images
  imagePrompt: string;
  imageParams: QwenParams;
  images: Array<{ uuid: string; url: string; selected: boolean }>;

  // Actions
  actions: Array<{ id: string; prompt: string; enabled: boolean }>;

  // Generation
  outputPlaylistId: string | null;
  wanParams: WanI2VParams;

  // Results
  jobs: Map<string, { imageId: string; actionId: string; dreamUuid: string; status: string }>;
}
```

**Combination key for deduplication:**
```typescript
const comboKey = (imageUuid: string, actionPrompt: string) =>
  `${imageUuid}:${hashMd5(actionPrompt).slice(0, 8)}`;
```

**Socket.IO subscription for batch:**
```typescript
// Subscribe to all jobs in batch
jobs.forEach(job => {
  socket.emit('join', `DREAM:${job.dreamUuid}`);
});

socket.on('job:progress', (data) => {
  updateJobStatus(data.dreamUuid, data.status, data.progress);
});
```

---

## Use Cases

| Use Case | MVP Phase | Description |
|----------|-----------|-------------|
| Qwen Image Batches | Phase 1 | Generate N images with different seeds |
| Combinatorial I2V | Phase 1 | N images × M actions = N×M videos |
| Seed Exploration | Phase 2 | Draft tier for rapid iteration |
| Prompt Variations | Phase 2 | Expansion syntax `{A|B|C}` |
| Uprez Pipeline | Phase 1 | Enhance selected videos |

### Use Case 1: Qwen Image Batches (MVP Phase 1)

**"Generate starting images"**

Artist enters image prompt, selects seed count (e.g., 8), clicks "Generate Images". System queues 8 Qwen jobs with sequential or random seeds. Grid fills in as each completes. Artist stars favorites for next stage.

```
Prompt: "mystical forest creature, watercolor style, soft lighting"
Seeds: 8
         ↓
┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐
│ ★  │ │    │ │ ★  │ │    │ │ ★  │ │    │ │    │ │    │
│seed│ │seed│ │seed│ │seed│ │seed│ │seed│ │seed│ │seed│
│ 1  │ │ 2  │ │ 3  │ │ 4  │ │ 5  │ │ 6  │ │ 7  │ │ 8  │
└────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘
         ↓
3 images selected for animation
```

**Non-linear**: Can generate more images anytime, add images from existing playlists.

### Use Case 2: Combinatorial I2V (MVP Phase 1)

**"Animate with multiple actions"** — the core combinatorial workflow

Take selected images and a list of action prompts, generate all combinations via Wan I2V.

```
3 selected images    ×    3 action prompts    =    9 videos
     │                         │
     │                         ├── "slow zoom in, camera pushing forward"
     │                         ├── "pan left to right, revealing scene"
     │                         └── "orbit around subject, 180 degrees"
     │
     ├── creature_seed1.png
     ├── creature_seed3.png
     └── creature_seed5.png
```

**Results matrix** shows all combinations with live progress:

```
              action1     action2     action3
creature1     [✓ done]    [✓ done]    [⟳ 67%]
creature3     [✓ done]    [⟳ 23%]    [· queue]
creature5     [· queue]   [· queue]   [· queue]
```

**Non-linear additions**:
- Add more images mid-batch → new row appears
- Add more actions mid-batch → new column appears
- System skips already-generated combinations

### Use Case 3: Uprez Pipeline (MVP Phase 1)

**"Enhance the winners"**

After reviewing I2V results, star the best videos. Click "Uprez Selected" to queue enhancement jobs (2x upscale + frame interpolation).

```
9 videos generated → 3 starred → 3 uprez jobs queued
```

### Use Case 4: Seed Exploration (Phase 2)

**"Find the magic seed"**

Draft tier (256p, fewer steps) for rapid iteration at ~10% cost. Artist explores 20-50 seeds quickly, then promotes winners to full quality.

```
Draft (256p, 15 steps)  →  Preview (480p, 20 steps)  →  Final (1080p, 30 steps)
      ~$0.02                    ~$0.10                      ~$0.50
```

One-click upgrade preserves all parameters, just changes quality tier.

### Use Case 5: Prompt Variations (Phase 2)

**"What if I said it differently?"**

Expansion syntax for prompt variations:

```
{A|B|C}     → generates 3 versions, one with each option
{A|B} {X|Y} → generates 4 versions (A+X, A+Y, B+X, B+Y)
[optional]  → generates 2 versions, with and without
```

Example:
```
Input:  "{fire|water|earth} elemental [dancing] in {forest|void}"
Output: 12 combinations (shown in preview grid before submitting)
```

### Use Case 6: Motion Presets (Phase 2)

**"Quick animation styles"**

Pre-defined action prompts for common camera movements:

```
┌─────────────────────────────────────────────────────────────────┐
│  MOTION PRESETS                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ ☑ Zoom In       ☑ Zoom Out      ☐ Push In      ☐ Pull Out  │ │
│  │ ☑ Pan Left      ☐ Pan Right     ☐ Tilt Up      ☐ Tilt Down │ │
│  │ ☐ Orbit CW      ☐ Orbit CCW     ☐ Float Up     ☐ Dolly     │ │
│  │ ☑ Custom: [gentle sway, breathing motion            ]       │ │
│  └─────────────────────────────────────────────────────────────┘ │
│  4 presets selected + 1 custom = 5 actions                       │
└─────────────────────────────────────────────────────────────────┘
```

Presets expand to full prompts (e.g., "Zoom In" → "slow zoom in, camera gently pushing forward into the scene").

---

## Timeline View

Premiere-style editing for generated content:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  TIMELINE: "nebula journey"                    [▶ Play] [⏸] [⛶ Full]      │
├─────────────────────────────────────────────────────────────────────────────┤
│  PREVIEW                                                                    │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         [video preview]                               │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                              ▼ 00:07.2 / 00:20.0                            │
├─────────────────────────────────────────────────────────────────────────────┤
│  0s      5s       10s      15s      20s                                    │
│  ├───────┼────────┼────────┼────────┤                                      │
│  │       ▼ playhead                                                        │
│  │                                                                         │
│  │ V1 ┌────────┬────────┬──────────┬────────┐                             │
│  │    │clip_42 │clip_47 │crystal_zm│void_dft│                             │
│  │    │  5.0s  │  5.0s  │   5.0s   │  5.0s  │                             │
│  │    └────────┴────────┴──────────┴────────┘                             │
│  │      ◄──►  trim handles                                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Features:**
- Scrub playhead with real-time preview
- Trim handles for in/out points (non-destructive)
- Drag to reorder, click to open inspector
- Double-click opens in node editor

**Preview playback:** Load individual clip URLs for scrubbing; generate concat playlist (HLS/DASH) for continuous play.

---

## Node-Based Pipeline Editor

Visual workflow builder (Rete.js-style) where each node represents a generation or processing step.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PIPELINE: "image to final video"                    [▶ Run] [💾 Save]     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐             │
│   │ Qwen Image   │      │  Wan I2V     │      │   Uprez      │             │
│   │──────────────│      │──────────────│      │──────────────│             │
│   │prompt: "nebu-│      │prompt: "slow │      │scale: 2x     │             │
│   │la crystal"   │      │zoom"         │      │interp: 2x    │             │
│   │size: 1024x   │      │duration: 5s  │      │              │             │
│   │         [img]●──────●[img]    [vid]●──────●[vid]   [vid]●──→ output   │
│   └──────────────┘      └──────────────┘      └──────────────┘             │
│                                                                             │
│   Click node to edit params │ Double-click to open inner view              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Node Types

| Node | Inputs | Outputs | Maps to |
|------|--------|---------|---------|
| **Qwen Image** | prompt, params | image URL | `handleQwenImageJob` |
| **Wan T2V** | prompt, params | video URL | `handleWanT2VJob` |
| **Wan I2V** | image, prompt, params | video URL | `handleWanI2VJob` |
| **Wan I2V LoRA** | image(s), prompt, loras | video URL | `handleWanI2VLoraJob` |
| **Deforum** | prompts, keyframes | video URL | `handleDeforumVideoJob` |
| **AnimateDiff** | prompt, params | video URL | `handleVideoJob` |
| **Uprez** | video, params | video URL | `handleUprezVideoJob` |
| **Concat** | video[] | video URL | FFmpeg concat (new) |
| **Trim** | video, in/out | video URL | FFmpeg trim (new) |
| **Slow-Mo** | video, factor | video URL | Frame interpolation/generation (new) |
| **Speed Ramp** | video, curve | video URL | Variable speed (new) |
| **Loop** | seed, body | outputs[] | Iteration with feedback (new) |

### Node Expansion

Double-click node to see/edit underlying parameters or raw JSON:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  NODE: Wan I2V (expanded)                                      [✕ Close]   │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────┐  ┌─────────────────────────────────────────┐  │
│  │ VISUAL PARAMS           │  │ RAW PROMPT JSON                         │  │
│  │─────────────────────────│  │─────────────────────────────────────────│  │
│  │ Prompt: [slow zoom]     │  │ {                                       │  │
│  │ Duration: [5s ▼]        │  │   "infinidream_algorithm": "wan-i2v",   │  │
│  │ Steps:    [30 ▼]        │  │   "prompt": "slow zoom...",             │  │
│  │ Guidance: [5.0 ━━●━]    │  │   "image": "{{input.image}}",           │  │
│  │ Seed:     [42    ]      │  │   ...                                   │  │
│  └─────────────────────────┘  └─────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

Edit visual params → JSON updates live. Edit JSON → visual params update.

---

## Chained Sequence Pipeline

### Last Frame → Next Input for Visual Continuity

Two levels of pipeline:
1. **Clip Pipeline** - how a single clip is generated (Image → I2V → Uprez)
2. **Sequence Pipeline** - how clips chain together (Clip1.lastFrame → Clip2.input)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  SEQUENCE PIPELINE                                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐       ┌─────────────┐       ┌─────────────┐               │
│  │   CLIP 1    │       │   CLIP 2    │       │   CLIP 3    │               │
│  │"nebula form"│       │"crystals"   │       │"explosion"  │               │
│  │●[in] [out]●├───────┤●[in] [out]●├───────┤●[in] [out]●├──→            │
│  └─────────────┘lastFrm└─────────────┘lastFrm└─────────────┘               │
│                                                                             │
│  Chain modes:                                                              │
│  ◉ Chain last frame  ○ Independent clips  ○ Custom connections            │
└─────────────────────────────────────────────────────────────────────────────┘
```

**How chaining works:**
1. Clip 1 completes
2. Extract last frame via FFmpeg: `ffmpeg -sseof -1 -i clip1.mp4 -frames:v 1 last.png`
3. Upload to R2, get URL
4. Clip 2 receives this URL as its "image" input
5. Repeat...

### Advanced Chaining

- **Keyframe extraction at specific points** (not just last frame)
- **Bidirectional chaining** for I2V with start+end frames (Wan I2V LoRA)
- **Branch and merge** for parallel variations that converge

---

## Hierarchical Pipelines / Nested Sequences

Any sequence can be collapsed into a single node and used in a parent pipeline:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  MASTER PIPELINE: "full film"                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐         │
│  │  ▤ SEQUENCE     │    │  ▤ SEQUENCE     │    │  ▤ SEQUENCE     │         │
│  │  "act 1"        │    │  "act 2"        │    │  "act 3"        │         │
│  │  4 clips, 20s   │    │  6 clips, 35s   │    │  3 clips, 15s   │         │
│  │●[in]     [out]●├────┤●[in]     [out]●├────┤●[in]     [out]●├──→       │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘         │
│                                                                             │
│  Double-click to expand and edit inner structure                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

Hierarchy can go deeper:
```
full film (master)
├── act 1 (sequence)
│   ├── clip 1
│   ├── clip 2
│   ├── scene 1a (nested sequence)
│   │   ├── clip 3
│   │   └── clip 4
│   └── clip 5
├── act 2 (sequence)
└── act 3 (sequence)
```

Breadcrumb navigation: `📁 full film › 📁 act 1 › 📁 scene 1a › 🎬 clip 3`

---

## BullMQ Flows for Hierarchical Execution

Map pipeline hierarchy to BullMQ Flow structure with parent-child job dependencies:

```typescript
const masterFlow = {
  name: 'master',
  queueName: 'pipeline',
  data: { type: 'master', name: 'full film' },
  children: [
    {
      name: 'sequence',
      data: { type: 'sequence', name: 'act 1' },
      children: [
        {
          name: 'clip',
          queueName: 'generation',
          data: { type: 'wan-i2v', prompt: 'nebula forms', image: '{{seedImage}}' },
        },
        {
          name: 'clip',
          queueName: 'generation',
          data: { type: 'wan-i2v', prompt: 'crystals emerge', image: '{{prev.lastFrame}}' },
          opts: { parent: { waitChildrenKey: 'clip-0' } },
        },
        // ... more clips
      ],
    },
    {
      name: 'sequence',
      data: { type: 'sequence', name: 'act 2', image: '{{prev.lastFrame}}' },
      opts: { parent: { waitChildrenKey: 'sequence-0' } },
      children: [/* act 2 clips */],
    },
  ],
};
```

### Flow Operations

| Action | Behavior |
|--------|----------|
| **Pause** | Stops processing new jobs in subtree |
| **Cancel** | Cancels active + waiting jobs in subtree |
| **Retry** | Re-queues failed job with same params |
| **Re-run** | Re-queues entire subtree |
| **Upgrade** | Re-queues at higher quality tier |

### Partial Re-render

Edit clip 3's prompt → only clip 3 and its dependents need re-render. System shows: "Editing clip 3 will invalidate 6 downstream jobs. Re-render?"

---

## Creative Routing Patterns

### Zoom Into/Out of Image

Iterative crop/expand with generation:

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  Source  │────→│  Crop    │────→│  Inpaint │────→│  I2V     │────┐
│  Image   │     │  Center  │     │  Expand  │     │  Morph   │    │
└──────────┘     │  50%     │     │  Edges   │     └──────────┘    │
     ▲           └──────────┘     └──────────┘                     │
     │                         LOOP N TIMES                        │
     └─────────────────────────────────────────────────────────────┘
```

### Feedback Loops

Output feeds back as input for iterative evolution:

**Style Drift:** I2V → Style Transfer → Extract Frame → feed back as input

**Iterative Refinement:** Each pass adds detail at progressively higher quality

**Dream Amplification:** Blend generated output with original at configurable ratio

### Loop Node

First-class loop support with iteration variables:

```
┌─────────────────────────────────────────────────────────────────┐
│  ⟳ LOOP: "style evolution"                                     │
│  Iterations: [8]                                                │
│                                                                 │
│  Loop variables:                                                │
│  {{loop.index}}     = 0, 1, 2, ...                             │
│  {{loop.progress}}  = 0.0 ... 1.0                              │
│  {{loop.prev}}      = previous iteration output                │
│                                                                 │
│  Accumulation: ◉ Collect all  ○ Keep final  ○ Concat           │
└─────────────────────────────────────────────────────────────────┘
```

**Prompt interpolation across loop:**
- Start: "serene forest lake at dawn"
- End: "cosmic nebula with floating crystals"
- System interpolates prompts across iterations

---

## Speed Manipulation & Frame Generation

### Slow-Motion

Two approaches:
1. **Interpolation** - blend between existing frames (fast, can artifact)
2. **AI Generation** - generate new frames via I2V (slower, higher quality)

```
Input: 24fps video, 1 second (24 frames)
Output: 24fps video, 8 seconds (192 frames) - 168 new frames generated

┌────┐ ┌────┐
│ F1 │ │ F2 │ ...  original frames
└──┬─┘ └──┬─┘
   │      │
   ▼      ▼
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ F1 │ g1 │ g2 │ g3 │ g4 │ g5 │ g6 │ g7 │ F2 │ ...  output
└────┴────┴────┴────┴────┴────┴────┴────┘
      └─────── AI generated ───────┘
```

### Speed Ramp / Time Remapping

Keyframed variable speed with curve editor:

```
speed
 4x ┤                    ●────●
    │                   ╱      ╲
 2x ┤                  ╱        ╲
    │                 ╱          ╲
 1x ┤────●───────────●            ●───────────●────
    │
0.25┤                                          ●────●  (slow-mo)
    └────┬────┬────┬────┬────┬────┬────┬────┬────┬────
        0s   1s   2s   3s   4s   5s   6s   7s   8s   9s
```

Segments below 1x speed trigger AI frame generation; segments above 1x use frame dropping.

### Timeline Speed Visualization

Speed changes visible with visual indicators:

```
┌────────────┬══════┬────────────────────┬▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓┐
│  normal    │ fast │      normal        │   slow-mo     │
│  1.0x      │ 4.0x │      1.0x          │   0.25x       │
└────────────┴──────┴────────────────────┴───────────────┘

Legend: ══ sped up (compressed)   ▓▓ slowed down (stretched)
```

### Speed-Matched Transitions

Auto-generate speed ramp when clips have different speeds at their boundaries.

---

## Beat Detection & Audio Sync

### Two Modes

1. **Generation-time sync** - analyze audio first, generate clips matching beat structure
2. **Playback-time sync** - screensaver adjusts playback to match live audio

### Audio Analysis Node

```
┌─────────────────────────────────────────────────────────────────┐
│  🎵 AUDIO ANALYSIS NODE                                         │
│                                                                 │
│  DETECTED                           OUTPUTS                    │
│  BPM: 120.0                         ●[beat_times]              │
│  Time signature: 4/4                ●[measures]                │
│  Duration: 3:42                     ●[energy]                  │
│                                     ●[sections]                │
│                                     ●[bpm]                     │
└─────────────────────────────────────────────────────────────────┘
```

### Beat-Synced Generation

Generate clips with durations matching musical structure:

```
Duration mapping:
○ 1 bar   = 2.0s  (4 beats)
◉ 2 bars  = 4.0s  (8 beats)  ← clip duration
○ 4 bars  = 8.0s  (16 beats)

|1   2   3   4   |1   2   3   4   |1   2   3   4   |
├───────────────┼───────────────┼───────────────┤
│    Clip 1     │    Clip 2     │    Clip 3     │
│    2 bars     │    2 bars     │    2 bars     │
```

### Energy-Driven Prompts

Map audio energy to generation parameters:

| Energy | Prompt Style | Motion | Speed |
|--------|-------------|--------|-------|
| Low (0.0-0.3) | calm, slow, ethereal | slow pan | 0.5x |
| High (0.7-1.0) | intense, explosive, chaotic | fast zoom | 2x |

Interpolate between energy levels using `{{energy}}` variable in prompts.

### Sync Metadata Storage

**Sidecar JSON (recommended):**
```
clip-abc123.mp4
clip-abc123.sync.json
```

```json
{
  "bpm": 120,
  "beats": [0.0, 0.5, 1.0, 1.5, 2.0],
  "downbeats": [0.0, 2.0, 4.0],
  "energy": [0.2, 0.3, 0.5, 0.8, 0.9],
  "syncPoints": [{"t": 0.0, "type": "downbeat"}],
  "stretchLimits": {"min": 0.85, "max": 1.15}
}
```

**Filename encoding (basic info):**
```
clip-abc123_120bpm_4bar_e75.mp4
```

**Database:** `Dream.syncMetadata` JSON column for authoritative data.

### Screensaver Beat Sync

Client-side real-time beat matching:

**Audio input options:**
- System audio loopback
- Microphone (live events)
- Audio file
- Network stream (Spotify metadata, Ableton Link)

**Sync behaviors:**
- Cut to next clip on downbeat
- Stretch/compress clip to hit beats (±15% speed)
- Trigger visual effects on beat (flash, zoom pulse)
- Match clip energy to audio energy

**Beat alignment algorithm:**
```
on_beat_detected(beat_time):
  time_to_next_beat = next_beat - now
  time_remaining = clip.duration - clip.current_time

  if close_enough:
    target_speed = time_remaining / time_to_next_beat
    if 0.85 < target_speed < 1.15:
      clip.playback_speed = target_speed  // imperceptible
    else:
      find_nearest_sync_point_and_transition()

  if is_downbeat and should_transition:
    start_transition_to_next_clip()
```

**Beat-triggered effects:**
- Brightness flash
- Zoom pulse
- Hard cuts vs. fades
- Intensity scaled by energy level

---

## Library & Reuse

Sequences and pipelines as reusable templates:

```
📁 My Templates
├── ▤ "4-clip zoom sequence"
│     [Use Template] [Edit]
├── ▤ "I2V + Uprez pipeline"
│     [Use Template] [Edit]
└── 🎬 "nebula seed 42"
      [Add to Timeline] [Duplicate + Vary]
```

- **Use Template**: Creates instance with same structure
- **Duplicate + Vary**: Creates copy with seed/prompt variations
- **Save as Template**: Any sequence can become reusable

---

## Implementation Notes

### Existing Infrastructure to Leverage

- **BullMQ** - already used for job queues, extend for Flows
- **Socket.IO** - already used for real-time via `/remote-control` namespace
- **Preview system** - `job:preview:{uuid}` Redis storage, `GET /v1/dream/{uuid}/preview` endpoint, `useGetDreamPreview` hook
- **Progress streaming** - `job:progress` Socket.IO event with `preview_frame`, `progress`, `countdown_ms`
- **RunPod endpoints** - Wan T2V, Wan I2V, Uprez, Deforum, Qwen (preview works for all except AnimateDiff)
- **FFmpeg** - already in video service for thumbnails/filmstrips
- **R2 storage** - already used for video/image hosting
- **Redis caching** - `job:progress:{uuid}` for reconnect recovery, 3hr TTL

### Real-Time Progress & Preview System

The backend streams progress via Socket.IO `/remote-control` namespace:

```typescript
// job:progress event payload (JobProgressData interface)
{
  status: 'IN_PROGRESS' | 'IN_QUEUE' | 'COMPLETED' | 'FAILED',
  progress: 45,           // percentage (0-100)
  countdown_ms: 12000,    // estimated time remaining
  preview_frame?: string  // base64 JPEG (latest frame)
}
```

**Preview Infrastructure:**

| Layer | Component | File |
|-------|-----------|------|
| **Frontend** | `useGetDreamPreview(uuid)` | `frontend/src/api/dream/mutation/useGetDreamPreview.ts` |
| **Frontend** | Preview button handler | `frontend/src/components/pages/view-dream/view-dream.page.tsx:600-624` |
| **Backend** | `GET /v1/dream/{uuid}/preview` | `backend/src/controllers/dream.controller.ts:1801-1842` |
| **Backend** | Progress broadcaster | `backend/src/services/job-progress.service.ts` |
| **Worker** | `storePreviewFrame()` | `worker/src/services/status-handler.service.ts:256-267` |
| **Storage** | Redis key | `job:preview:{dreamUUID}` (3hr TTL, base64 JPEG) |

**Data flow:**
```
ComfyUI renders frame → RunPod status includes output.preview_frame
    → Worker polls, extracts frame → storePreviewFrame() to Redis
    → Two consumption paths:
        1. Socket.IO: job:progress event includes preview_frame (real-time)
        2. REST API: GET /preview fetches from Redis (on-demand)
    → Frontend: currently only uses REST via "Preview" button click
```

**Current limitations:**

| Limitation | Impact | Fix Location |
|------------|--------|--------------|
| Manual refresh only | Poor UX during long renders | Frontend: subscribe to `job:progress` |
| Single frame (no history) | Can't scrub through progress | Worker: use Redis list instead of string |
| No AnimateDiff support | Algorithm excluded | GPU container: add preview to workflow |
| Preview lost after completion | Can't see intermediate frames | Worker: persist to S3 or extend TTL |

### Python SDK & Batch Scripts

**Quickstart:** [Infinidream Cloud Generation Quickstart](https://docs.google.com/document/d/1sXfGgogyrDyaOOxCyG6uvkG1l6uTUE2iNdkqVAa-N0Q/edit?tab=t.0)

#### edream_sdk (python-api repo)

```python
from edream_sdk.client import create_edream_client

client = create_edream_client(backend_url, api_key)

# Create dream with any algorithm
dream = client.create_dream_from_prompt({
    "name": "My Dream",
    "description": "...",
    "prompt": json.dumps({
        "infinidream_algorithm": "wan-i2v",  # or: qwen-image, wan-t2v, deforum, animatediff, uprez
        "prompt": "...",
        "image": "uuid-or-url",  # for i2v
        # ... algorithm-specific params
    })
})

# Poll for completion
dream = client.get_dream(dream["uuid"])  # status: queue → processing → processed

# Playlist management
playlist = client.create_playlist({"name": "Batch Output"})
client.add_item_to_playlist(playlist_uuid, type="dream", item_uuid=dream["uuid"])
```

#### engines repo - Batch Scripts

| Script | Purpose | Config |
|--------|---------|--------|
| `run_qwen_image_batch.py` | Generate N images with different seeds | `qwen-image-config.json` |
| `run_wan_i2v_batch.py` | Images × Actions → Videos | `job.json` |
| `run_uprez_batch.py` | Upscale entire playlist | config in script |

**Wan I2V batch config (job.json):**
```json
{
  "image_playlist_uuid": "...",  // source images
  "combos": ["zoom in slowly", "pan left", "orbit around"],  // action prompts
  "prompt": "base prompt applied to all",
  "playlist_uuid": "...",  // output playlist (or create new)
  "duration": 5,
  "num_inference_steps": 30
}
```

This produces `len(images) × len(combos)` videos, skipping already-generated combinations via `BATCH_IDENTIFIER` tracking.

### New Components Needed

1. **Pipeline/Flow API** - create, save, run hierarchical job flows
2. **Timeline component** - React component for video arrangement
3. **Node editor component** - Rete.js or similar for visual pipelines
4. **Frame extraction service** - extract specific frames for chaining
5. **Concat service** - FFmpeg concat for sequence rendering
6. **Beat detection** - Aubio/Essentia integration (server or client)
7. **Sync metadata** - schema and API for beat/energy data

### Preview System Enhancements

Building on existing infrastructure (`job:preview:{uuid}`, Socket.IO `job:progress`):

| Enhancement | Effort | Value |
|-------------|--------|-------|
| Auto-refresh preview in UI | Low | Better UX during long renders |
| Frame history buffer (last N frames) | Medium | Scrub through generation progress |
| Push frames via Socket.IO (vs polling) | Low | Already partially implemented, wire to UI |
| AnimateDiff preview support | Medium | Requires ComfyUI workflow changes |
| Batch preview grid (all seeds at once) | Medium | Critical for seed exploration use case |
| Draft tier (256p) fast preview | High | Requires new RunPod endpoint config |

**Key files to modify:**
- `frontend/src/components/pages/view-dream/view-dream.page.tsx` - add auto-refresh, frame history UI
- `worker/src/services/status-handler.service.ts` - frame history buffer in Redis
- `backend/src/services/job-progress.service.ts` - already broadcasts `preview_frame`, ensure frontend consumes
- `gpu-container/` - AnimateDiff workflow preview frame extraction

### Client (Screensaver) Changes

- Audio capture (CoreAudio/WASAPI)
- Beat detection library (Aubio)
- Variable speed playback
- Beat-triggered effects
- Sync point awareness
- Optional: WebSocket for external beat sources

---

## Summary

This design enables AI artists to:

1. **Explore rapidly** at draft quality (256p, seconds, cheap)
2. **Refine promising directions** at preview quality
3. **Render final output** with uprez and concatenation
4. **Build complex sequences** with visual continuity via frame chaining
5. **Create hierarchical compositions** with nested sequences
6. **Apply creative effects** like zoom loops, feedback, speed ramps
7. **Sync to music** both at generation time and during playback

All powered by existing backend infrastructure (BullMQ, RunPod, FFmpeg, Socket.IO) with targeted extensions for flows, timeline, and node editing.

### MVP Path

**Start with the combinatorial workflow** (qwen → wan → uprez) as a Studio page on the existing site:

1. **Phase 1**: Frontend-only `/studio` page using existing APIs
2. **Phase 2**: Backend batch optimizations
3. **Phase 3**: Persistence, presets, templates
4. **Future**: Evolve into full node editor with timeline

This validates the core pattern (generate → select → combine → refine) with a real user workflow before investing in the full visual pipeline editor.
