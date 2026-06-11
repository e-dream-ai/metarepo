# Variations, Sessions & Prompt Expansion

**Date:** 2026-06-08
**Status:** Draft
**Scope:** Frontend (Phase 0), Backend (Phase 1 — API keys, Phase 2 — session persistence)

## Context

Phases 0–2 of the studio roadmap are shipped (keyframe strip, transition generation, image upload, fullscreen layout, save to playlist). This spec covers the next wave of features, drawn from Jay Gidwitz's May 7 and May 13 feedback sessions and Jef's February 13 and March 17 sessions.

**Jay's #1 ask (May 7, 00:14:21):** "The most important part of it for at least this process is that with MidJourney you can do an image and then do variations on that image. So it would need that ability because otherwise the loops become very obvious."

**Jay on seed re-rolls vs variations (00:15:14):** "I found the best results haven't been choosing images from the same prompt, but from doing like the variations... I wonder if they're like starting with the image and then diffusing it a bit and then going from there."

**Jef's #1 ask (Feb 13):** "The most important thing is organization. I call it a workflow, call whatever you want, but it's organization."

**Jef on prompt generation (Feb 13):** "What if we had an auto prompt generator?"

---

## Phased Delivery

| Phase | What | Backend? | Dependencies |
|-------|------|----------|-------------|
| 0 | Sessions (localStorage), seed re-roll grid, prompt expansion | No | None |
| 1 | API key config, i2i variations via user keys, model picker integration | Yes | None (parallel with Phase 0) |
| 2 | Session persistence (backend entity, cross-device sync) | Yes | Phase 0 sessions exist |

Phases 0 and 1 are independent and can be worked in parallel. Phase 2 depends on Phase 0.

---

## Phase 0: Sessions, Variations & Prompt Expansion (Frontend Only)

### 1. Sessions

#### Problem

The studio currently has a single localStorage blob per mode (`flow-session` key for flow, `studio-session` key for batch). Starting a new project overwrites the old one. Jef described wanting to "click new project and then I'm in, and then everything that I do while I'm in that project is in there."

#### Design

A session is a named snapshot of an entire studio workspace — both flow and batch state.

```typescript
interface StudioSession {
  id: string;                    // UUID
  name: string;                  // user-editable, default: "Session — YYYY-MM-DD HH:mm"
  createdAt: string;             // ISO date
  updatedAt: string;             // ISO date
  mode: "flow" | "batch";       // which mode was active
  flowState: FlowStoreState;    // full flow store snapshot
  batchState: StudioStoreState; // full batch store snapshot
  thumbnail?: string;           // data URL of first keyframe/image (for session list)
}
```

**Storage:** localStorage key `studio-sessions` holds a `StudioSession[]`. Each session is a complete snapshot. Current active session tracked via `studio-active-session-id` key.

**Session limit:** Cap at 20 sessions in localStorage to avoid storage limits (~5MB). Show warning when approaching limit. Oldest sessions can be deleted.

#### UX

**Session switcher** in the studio header (between back arrow and mode toggle):

- Dropdown shows session list: name, date, mode badge (Flow/Batch), thumbnail
- "New Session" creates a blank session, auto-saves the current one first
- "Rename" inline edit on session name
- "Delete" with confirmation
- "Duplicate" copies current session

**Auto-save:** Current session auto-saves to localStorage on every store change (debounced, 2s). No explicit "Save" button needed — it just works like a Google Doc.

**Session reset:** The existing "New Session" button in batch mode header and `FlowReset` component should create a new session (not just clear state).

#### Interaction with Existing Stores

The flow store (`flow-session` key) and studio store (`studio-session` key) continue to hold the *active* session's state. The session manager reads/writes these keys when switching sessions:

1. **Switch session:** Serialize current flow + batch stores → save to `studio-sessions[activeId]` → load target session's `flowState`/`batchState` into the stores → update `studio-active-session-id`
2. **Auto-save:** Debounced write of current store state into `studio-sessions[activeId]`
3. **New session:** Auto-save current → create blank entry in `studio-sessions` → reset both stores

No migration needed for existing users — their current localStorage state becomes "Session 1" on first load.

#### Files Changed

| File | Change |
|------|--------|
| `src/stores/session.store.ts` | **New** — session list, active session, save/load/switch/delete actions |
| `src/components/pages/studio/components/session-switcher.tsx` | **New** — dropdown UI in studio header |
| `src/components/pages/studio/components/session-switcher.styled.tsx` | **New** — styles |
| `src/components/pages/studio/studio.page.tsx` | Add session switcher to header |
| `src/stores/flow.store.ts` | Export state type for serialization |
| `src/stores/studio.store.ts` | Export state type for serialization, handle `excludedCombos` Set serialization |

---

### 2. Inline Variation Grid

#### Problem

Jay wants MidJourney-style variations — generate multiple candidates, pick the best. Currently, flow mode generates one result per transition and one image per keyframe. Batch mode has a results grid but it's a separate tab, not inline.

#### Design

An inline grid that appears below a keyframe or transition in the flow builder, showing N variation candidates. The grid is the same component regardless of how variations were generated (seed re-roll, prompt expansion, or i2i).

#### Variation Methods

Three methods, all producing results that display in the same inline grid:

| Method | How | Available |
|--------|-----|-----------|
| **Seed re-roll** | Same prompt, N different random seeds | Day 1 (all models) |
| **Prompt expansion** | `{A\|B\|C}` syntax expands to N prompts | Day 1 (all models) |
| **i2i** | Feed existing image to img2img model | Phase 1 (Flux, OpenAI, MidJourney via API keys) |

#### Keyframe Variations

User clicks a keyframe and chooses "Variations" (context menu or button). Options:

- **Re-roll (N seeds):** Generate 4/9/16 images with the same prompt but different seeds
- **Expand prompt:** If prompt contains `{A|B|C}`, expand and generate each variant
- **Vary (i2i):** Feed this image to an img2img model for similar-but-different results (Phase 1 only, grayed out until API key configured for a model that supports i2i)

Results appear in an inline grid below the keyframe:

```
+--------+   ------   +--------+   ------   +--------+
| [img2] |            | [img3] |            | [img4] |
|   ▼    |            |        |            |        |
+--------+            +--------+            +--------+
+--------------------------------------------------+
|  [v1 ✓]  [v2]  [v3]  [v4]                       |
|  click to swap           [+ More]  [Close]       |
+--------------------------------------------------+
```

- Grid layout: 2x2 (4), 3x3 (9), or 4x4 (16) based on count
- Each cell shows image thumbnail + generation status (queued/processing/done/failed)
- Click a variation to swap it into the keyframe slot (original preserved internally)
- "[+ More]" generates another batch
- "[Close]" collapses the grid

#### Transition Variations

Same pattern. User clicks a completed transition gap and chooses "Variations":

- **Re-roll:** N seeds with same settings
- **Expand prompt:** If transition prompt contains `{A|B|C}`

Results appear inline between the two keyframes:

```
+--------+                              +--------+
|        |   +--------------------+     |        |
| [img1] |   | [v1 ✓] [v2] [v3] |     | [img2] |
|        |   | click to swap     |     |        |
+--------+   +--------------------+    +--------+
```

- Click to swap winner into the transition's `dreamUuid`
- Previous result preserved (can revert)

#### Data Model

Extend `FlowKeyframe` and `FlowTransition` with variation tracking:

```typescript
interface VariationCandidate {
  id: string;              // local UUID
  method: "seed" | "expansion" | "i2i";
  prompt?: string;         // if expanded, the specific prompt used
  seed?: number;           // if seed re-roll
  dreamUuid?: string;      // backend dream UUID (for images and transitions)
  imageUrl?: string;       // for keyframe image variations
  status: TransitionStatus;
  progress?: number;
}

// Add to FlowKeyframe:
interface FlowKeyframe {
  // ...existing fields...
  variations?: VariationCandidate[];
  activeVariationId?: string;  // which variation is currently displayed
}

// Add to FlowTransition:
interface FlowTransition {
  // ...existing fields...
  variations?: VariationCandidate[];
  activeVariationId?: string;
}
```

Variations are stored in the flow store and persist with sessions. The `activeVariationId` tracks which candidate is currently "live" in the flow.

#### Batch Mode

The variation grid is a **flow mode** feature. Batch mode already has its own variation mechanisms:

- **Seed re-rolls:** The existing `seedCount` parameter in image generation already produces N images per prompt. No change needed.
- **Prompt expansion:** Expands action prompts into additional columns in the results matrix. This integrates naturally with the existing combinatorial grid — more actions = more columns.

No inline variation grid is needed for batch mode.

#### Reuse from Batch Mode

The batch mode results grid (`ResultCell`, `ResultThumb`, `ResultCellStatus` styled components in `results-tab.styled.tsx`) provides the visual building blocks. The variation grid uses the FLOW theme constants (dark, custom-themed) which is the target design system for the entire studio — batch mode should be migrated to FLOW theme constants over time. The variation grid draws from the same visual patterns as the batch results grid but uses FLOW theme tokens directly.

#### Files Changed

| File | Change |
|------|--------|
| `src/types/flow.types.ts` | Add `VariationCandidate`, extend `FlowKeyframe` and `FlowTransition` |
| `src/stores/flow.store.ts` | Add variation actions: `generateVariations`, `selectVariation`, `addMoreVariations`, `clearVariations` |
| `src/components/pages/studio/components/variation-grid.tsx` | **New** — inline grid component |
| `src/components/pages/studio/components/variation-grid.styled.tsx` | **New** — grid styles, reusing ResultCell/ResultThumb patterns |
| `src/components/pages/studio/components/keyframe-card.tsx` | Add "Variations" button, render variation grid when active |
| `src/components/pages/studio/components/transition-gap.tsx` | Add "Variations" option for completed transitions |
| `src/components/pages/studio/components/flow-builder.tsx` | Wire up variation generation (create dreams with different seeds/prompts) |

---

### 3. Prompt Expansion

#### Problem

Jef asked for an "auto prompt generator." The original creator workflows doc (Jan 30) spec'd a `{A|B|C}` expansion syntax for batch mode. Extending this to flow mode enables prompt-driven variations — a different approach than seed re-rolls that produces more diverse results.

#### Syntax

```
{A|B|C}       → expands to 3 variants, one per option
{A|B} {X|Y}   → cross-product: 4 variants (A+X, A+Y, B+X, B+Y)
[optional]     → 2 variants: with and without the bracketed text
```

Nesting is not supported in v1. Literal braces can be escaped with backslash. The `[optional]` syntax is deferred to v2 — brackets are too common in normal prompt text to be safe delimiters without false positives.

#### Parser

A pure function that takes a prompt string and returns an array of expanded strings:

```typescript
function expandPrompt(template: string): string[] {
  // Parse {A|B|C} groups and [optional] groups
  // Compute cross-product of all groups
  // Return array of expanded strings
}
```

- If no expansion syntax is present, returns `[template]` (single element)
- Preview: show "N variants" badge next to the prompt field when expansion syntax is detected

#### Where It Works

**Flow mode (global prompt + per-transition overrides):**
- User writes `{zoom in|orbit left|push through}` as the transition prompt
- "Generate All" expands to 3 variants per transition
- Results feed the inline variation grid — user picks the best for each transition

**Batch mode (action prompts):**
- User writes `{fire|water|earth} elemental [dancing]` as an action prompt
- Generate tab shows expanded combination count
- Each expansion becomes a separate column in the results matrix

**Keyframe generation (image prompts):**
- User writes `{crystal cave|nebula|forest} landscape` when generating keyframes
- Generates N images, one per expansion
- Results show in the variation grid for that keyframe

#### Interaction with Seed Re-rolls

Prompt expansion and seed re-rolls are independent axes:

- Expansion only → N variants (one seed each)
- Seeds only → N variants (one prompt each)
- Both → N×M variants (every prompt × every seed). Capped at 16 to prevent runaway generation.

#### Files Changed

| File | Change |
|------|--------|
| `src/components/pages/studio/utils/expand-prompt.ts` | **New** — parser + expander |
| `src/components/pages/studio/utils/__tests__/expand-prompt.test.ts` | **New** — parser tests |
| `src/components/pages/studio/components/prompt-expansion-badge.tsx` | **New** — "N variants" indicator next to prompt fields |
| `src/components/pages/studio/components/transition-settings-panel.tsx` | Show expansion badge, wire expansion into generation |
| `src/components/pages/studio/components/flow-builder.tsx` | Expand prompts before submitting generation jobs |
| `src/components/pages/studio/components/generate-tab.tsx` | Expand action prompts, update combination count |
| `src/components/pages/studio/components/actions-tab.tsx` | Show expansion badge on actions with `{...}` syntax |

---

## Phase 1: API Key Config & i2i Variations (Backend + Frontend)

### 1. API Key Configuration

#### Problem

Jay recommended Flux ("the French open-source MidJourney competitor") and wanted to bring his own model endpoints. Jef said "get away from using APIs" — meaning use self-hosted or cheaper alternatives. A generic API key config lets power users plug in any OpenAI-compatible endpoint immediately.

#### Backend

New `UserApiEndpoint` entity:

```typescript
@Entity()
class UserApiEndpoint {
  @PrimaryGeneratedColumn("uuid")
  uuid: string;

  @Column()
  userId: string;          // owner

  @Column()
  name: string;            // "My Flux", "OpenAI Production"

  @Column()
  type: "openai-compatible"; // extensible later

  @Column()
  endpointUrl: string;     // "https://api.openai.com/v1"

  @Column()
  apiKeyEncrypted: string; // AES-256-GCM encrypted

  @Column()
  modelId: string;         // "gpt-image-1", "flux-schnell", etc.

  @Column({ type: "jsonb", nullable: true })
  capabilities: {
    textToImage?: boolean;
    imageToImage?: boolean;  // enables i2i variations
    sizes?: string[];        // supported output sizes
  };

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

**API endpoints:**

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/v1/user/api-endpoints` | List user's configured endpoints |
| `POST` | `/v1/user/api-endpoints` | Create new endpoint |
| `PUT` | `/v1/user/api-endpoints/:uuid` | Update endpoint |
| `DELETE` | `/v1/user/api-endpoints/:uuid` | Delete endpoint |
| `POST` | `/v1/user/api-endpoints/:uuid/test` | Test connection (sends a minimal request) |

**Encryption:**
- AES-256-GCM with `API_KEY_ENCRYPTION_SECRET` env var
- API responses return only last 4 characters of key (`...Bf2`)
- Dual-secret rotation: `API_KEY_ENCRYPTION_SECRET_PREVIOUS` for decryption during rollover

**Worker integration:**

When a dream is created with a user endpoint's model, the worker:
1. Dream prompt includes `userEndpointUuid` field identifying which endpoint to use
2. Worker looks up the `UserApiEndpoint` by UUID, verifies it belongs to the dream's owner
3. Decrypts the API key
4. Calls the endpoint using OpenAI-compatible format (POST to `{endpointUrl}/images/generations` or `{endpointUrl}/images/edits` for i2i)
5. Downloads result, uploads to R2, updates dream status

**Backend dream creation:** The `POST /v1/dream` endpoint accepts an optional `userEndpointUuid` in the prompt JSON. Backend validates the endpoint exists and belongs to the user before queuing the job.

#### Frontend

**Account settings page** — new "API Keys" section:

- List configured endpoints (name, model, masked key, edit/delete)
- "Add Endpoint" form: name, endpoint URL, API key, model ID, capabilities checkboxes
- "Test Connection" button validates before saving

**Model picker integration:**

User endpoints appear in studio model dropdowns alongside platform models:

```
Image Model:
  Z Image Turbo (platform)
  Qwen Image (platform)
  ─────────────────────
  My Flux (your key)      ← from UserApiEndpoint
  OpenAI gpt-image-1 (your key)
```

#### Files Changed (Backend)

| File | Change |
|------|--------|
| `src/entity/user-api-endpoint.entity.ts` | **New** — entity |
| `src/migration/XXXX-add-user-api-endpoints.ts` | **New** — migration |
| `src/routes/user-api-endpoint.routes.ts` | **New** — CRUD + test routes |
| `src/services/user-api-endpoint.service.ts` | **New** — encryption/decryption, CRUD |
| `src/services/encryption.service.ts` | **New** — AES-256-GCM encrypt/decrypt |
| `src/utils/prompt.util.ts` | Recognize user endpoint model names |

#### Files Changed (Worker)

| File | Change |
|------|--------|
| `src/services/user-endpoint-handler.service.ts` | **New** — generic OpenAI-compatible handler |
| `src/services/job-handler.service.ts` | Route to user endpoint handler when dream uses a user model |

#### Files Changed (Frontend)

| File | Change |
|------|--------|
| `src/api/user-api-endpoints/` | **New** — React Query hooks for CRUD + test |
| `src/components/pages/account/api-keys-section.tsx` | **New** — settings UI |
| `src/components/pages/studio/constants/size-options.ts` | Support dynamic sizes from endpoint capabilities |
| Model picker components | Add user endpoints to dropdown |

### 2. i2i Variations

When a user has configured an endpoint with `capabilities.imageToImage: true`, the "Vary (i2i)" option becomes available in the variation grid.

**How it works:**
1. User clicks keyframe → Variations → Vary (i2i)
2. Frontend sends existing image + prompt to the i2i-capable endpoint
3. Endpoint returns a modified version of the image
4. Result appears in the variation grid alongside seed/expansion variants

**API format (OpenAI-compatible i2i):**
```
POST {endpointUrl}/images/edits
{
  "image": <base64 or URL>,
  "prompt": "crystal cave with blue lighting",
  "n": 4,
  "size": "1024x1024"
}
```

For endpoints that support batch generation (`n > 1`), a single API call can produce multiple variations. The grid shows all results.

---

## Phase 2: Server-Side Session Persistence

### Problem

Phase 0 sessions live in localStorage — device-local, lost on browser clear, no cross-device sync. Power users like Jef want durable project organization.

### Design

Migrate sessions from localStorage to a backend entity. The frontend session manager becomes a sync layer:

- Auto-save debounced writes go to the API instead of localStorage
- Session list fetched from backend on page load
- localStorage acts as offline cache (write-through)

### Backend Entity

```typescript
@Entity()
class StudioSession {
  @PrimaryGeneratedColumn("uuid")
  uuid: string;

  @Column()
  userId: string;

  @Column()
  name: string;

  @Column({ type: "jsonb" })
  flowState: object;

  @Column({ type: "jsonb" })
  batchState: object;

  @Column()
  mode: string;

  @Column({ nullable: true })
  thumbnailUrl: string;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

**API endpoints:**

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/v1/studio/sessions` | List user's sessions |
| `GET` | `/v1/studio/sessions/:uuid` | Load session |
| `POST` | `/v1/studio/sessions` | Create session |
| `PUT` | `/v1/studio/sessions/:uuid` | Update (auto-save) |
| `DELETE` | `/v1/studio/sessions/:uuid` | Delete session |

**Migration path:** On first backend session load, if user has localStorage sessions but no backend sessions, migrate them up automatically.

---

## Anti-Rationalization Table

| Rationalization | Reality |
|----------------|---------|
| "Seed re-rolls are enough for variations." | Jay explicitly said seeds aren't enough (00:15:14). But they're the fast path for day 1. i2i via API keys is the real goal. |
| "Prompt expansion is only useful in batch mode." | In flow mode, `{zoom in\|orbit\|drift}` gives you 3 transition variants per gap. That IS the variation feature driven by prompts. |
| "Sessions can wait — just clear and start over." | Jef's #1 ask. Without sessions, every new project destroys the last one. |
| "We should host Flux/OpenAI ourselves before shipping API keys." | API keys let power users (Jay, Jef) bring their own endpoints immediately. Zero infrastructure cost to us. Hosting comes later if demand warrants. |
| "i2i variations need a special UI." | They use the same inline grid as seed re-rolls and prompt expansion. The variation method is metadata, not a different component. |
| "20 sessions in localStorage is too few." | 20 sessions × ~50KB each = ~1MB. Phase 2 moves to backend with no limit. |

---

## Roadmap Summary

| Phase | What | Effort | Key Deliverable |
|-------|------|--------|-----------------|
| 0 | Sessions + Variations + Prompt Expansion | Frontend only | Named sessions (localStorage), inline 4/9/16 variation grid (seed re-roll + prompt expansion), `{A\|B\|C}` syntax in flow + batch |
| 1 | API Key Config + i2i | Backend + Frontend | Encrypted endpoint storage, OpenAI-compatible handler, i2i variations via user keys, model picker integration |
| 2 | Server-Side Sessions | Backend + Frontend | Migrate localStorage sessions to backend entity, cross-device sync, offline cache |

---

## References

- Jay Gidwitz feedback (2026-05-07): `docs/transcripts/2026-05-07-jay-gidwitz-studio-feedback.md`
- Jay Gidwitz follow-up (2026-05-13): Presentation decks in `docs/presentations/`
- Jef feedback session 1 (2026-02-13): `docs/transcripts/2026-02-13-jef-studio-feedback-1.md`
- Jef feedback session 2 (2026-03-17): `docs/transcripts/2026-03-17-jef-studio-feedback-2.md`
- Original creator workflows design: `docs/plans/2026-01-30-visual-creator-workflows-design.md`
- Studio roadmap spec: `docs/superpowers/specs/2026-05-12-studio-roadmap-design.md`
- Prompt expansion syntax: `docs/plans/2026-01-30-visual-creator-workflows-design.md` (Use Case 5)
