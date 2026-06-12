# Phase 1: API Key Config & i2i Variations

**Date:** 2026-06-12
**Status:** Draft
**Scope:** Backend + Worker + Frontend
**Parent spec:** `docs/superpowers/specs/2026-06-08-variations-sessions-expansion-design.md` (Phase 1)
**Base branch:** `stage` (assumes Phase 0 variations/sessions has landed)

## Context

Jay wants Flux ("the French open-source MidJourney competitor") and MidJourney-style i2i variations. Jef wants cheaper self-hosted alternatives. A user-endpoint system lets power users bring their own API keys for external image models — zero infrastructure cost to us, immediate access to Flux and OpenAI image generation.

This is Phase 1 of the variations/sessions/expansion spec. Phase 0 (sessions, seed re-rolls, prompt expansion) is shipped.

---

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Provider types | OpenAI + FAL | Covers gpt-image-1 and Flux (via FAL). Two adapters, not a generic system. |
| API call location | Worker | Same job lifecycle as RunPod — queue, progress, completion, R2 upload. Socket.IO progress works unchanged. |
| Key encryption | Reuse existing `CIPHER_KEY` | AES-256-CBC via `crypto.util.ts`. One secret to manage. |
| Key delivery to worker | Backend decrypts, passes in BullMQ job data | No new DB connections in worker. Key is ephemeral in Redis (job TTL). |
| Config UI location | Account page | Endpoint CRUD on profile page. "Add endpoint..." link in studio model picker. |
| Endpoint setup UX | Preset templates | "Pick a service, paste your key, save." Presets: Flux Schnell, Flux Pro, gpt-image-1, Custom. |
| Save behavior | Auto-tests on save | Save calls the provider to verify the key. Fails with inline error if bad. No separate test button on the add form. Existing endpoints get a "Test" button for re-verification. |
| Model picker style | Flat list with labels | User endpoints appended to existing model list, tagged "your key". "Add endpoint..." link at bottom. |
| Image integration | Keyframe source only | User endpoint images become flow keyframes. No batch mode integration in Phase 1. |
| i2i endpoint selection | Session-level default | `i2iEndpointUuid` in flow store. User picks once, all i2i variations use it until changed. |

---

## Data Model

### UserApiEndpoint Entity (Backend)

```typescript
@Entity()
class UserApiEndpoint {
  @PrimaryGeneratedColumn("uuid")
  uuid: string;

  @Column()
  @Index()
  userId: string;

  @ManyToOne(() => User)
  @JoinColumn({ name: "userId" })
  user: User;

  @Column()
  name: string;                    // "My Flux Schnell"

  @Column()
  providerType: "openai" | "fal";

  @Column()
  presetId: string;                // "flux-schnell" | "flux-pro" | "openai-gpt-image-1" | "custom"

  @Column()
  endpointUrl: string;             // full URL

  @Column()
  apiKeyEncrypted: string;         // AES-256-CBC encrypted (content)

  @Column()
  apiKeyIv: string;                // initialization vector

  @Column()
  apiKeyLastFour: string;          // "...x7Bf" for display

  @Column()
  modelId: string;                 // model identifier sent to provider

  @Column({ type: "jsonb" })
  capabilities: {
    textToImage: boolean;
    imageToImage: boolean;
    sizes: string[];
  };

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

### Endpoint Presets (Frontend Constants)

Presets are frontend-only constants. Backend stores whatever the frontend sends — adding new presets is a frontend-only change.

```typescript
interface EndpointPreset {
  id: string;
  name: string;
  providerType: "openai" | "fal";
  endpointUrl: string;
  modelId: string;
  capabilities: { textToImage: boolean; imageToImage: boolean; sizes: string[] };
  description: string;
  keyUrl: string;             // link to provider's key management page
}

const ENDPOINT_PRESETS: EndpointPreset[] = [
  {
    id: "flux-schnell",
    name: "FAL — Flux Schnell",
    providerType: "fal",
    endpointUrl: "https://fal.run/fal-ai/flux/schnell",
    modelId: "fal-ai/flux/schnell",
    capabilities: { textToImage: true, imageToImage: true, sizes: ["1024x1024", "1280x720", "720x1280"] },
    description: "Fast image generation & i2i · ~$0.003/image",
    keyUrl: "https://fal.ai/dashboard/keys",
  },
  {
    id: "flux-pro",
    name: "FAL — Flux Pro",
    providerType: "fal",
    endpointUrl: "https://fal.run/fal-ai/flux-pro",
    modelId: "fal-ai/flux-pro",
    capabilities: { textToImage: true, imageToImage: true, sizes: ["1024x1024", "1280x720", "720x1280"] },
    description: "Higher quality, slower · ~$0.05/image",
    keyUrl: "https://fal.ai/dashboard/keys",
  },
  {
    id: "openai-gpt-image-1",
    name: "OpenAI — gpt-image-1",
    providerType: "openai",
    endpointUrl: "https://api.openai.com/v1",
    modelId: "gpt-image-1",
    capabilities: { textToImage: true, imageToImage: true, sizes: ["1024x1024", "1536x1024", "1024x1536"] },
    description: "High quality image generation & editing · ~$0.02–0.19/image",
    keyUrl: "https://platform.openai.com/api-keys",
  },
  // "custom" is not a preset — it's the absence of one, with all fields manual
];
```

---

## Backend API

All routes behind `requireAuth`. Authorization: every operation checks `endpoint.userId === req.user.id`.

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/v1/user/api-endpoints` | List user's endpoints (key masked to last 4) |
| `POST` | `/v1/user/api-endpoints` | Create — tests connection first, returns 422 if bad key |
| `PUT` | `/v1/user/api-endpoints/:uuid` | Update — re-tests if key changed |
| `DELETE` | `/v1/user/api-endpoints/:uuid` | Delete |
| `POST` | `/v1/user/api-endpoints/:uuid/test` | Re-test existing endpoint |

### Test-on-Save Logic

Performed in the service layer during create and update (when key changes):

- **OpenAI:** `POST {endpointUrl}/images/generations` with `{ model, prompt: "test", size: "256x256", n: 1 }`. Verify 200 response.
- **FAL:** `POST {endpointUrl}` with `{ prompt: "test", image_size: "square" }` + `Authorization: Key {apiKey}`. Verify 200 or accepted status.

If test fails, return HTTP 422 with the provider's error message. Frontend shows it inline below the key input.

### Dream Creation Change

When `POST /v1/dream` receives a prompt JSON containing `userEndpointUuid`:

1. Look up `UserApiEndpoint` by UUID, verify it belongs to `req.user`
2. Decrypt the API key via `crypto.util.decrypt()`
3. Determine queue: `"user-endpoint"` (new queue, single queue for all user endpoint jobs)
4. Include in BullMQ job data:
   ```typescript
   {
     dream_uuid: string,
     userEndpointDecryptedKey: string,
     userEndpointUrl: string,
     userEndpointProvider: "openai" | "fal",
     userEndpointModelId: string,
     prompt: string,
     image?: string,        // source image URL for i2i
     size?: string,
     n?: number,            // batch count for variations
   }
   ```
5. Route to queue `"user-endpoint"` instead of the algorithm-specific queue

### Files Changed (Backend)

| File | Change |
|------|--------|
| `src/entities/UserApiEndpoint.entity.ts` | **New** — entity |
| `src/migration/XXXX-add-user-api-endpoints.ts` | **New** — migration |
| `src/routes/user-api-endpoint.routes.ts` | **New** — CRUD + test routes |
| `src/controllers/user-api-endpoint.controller.ts` | **New** — route handlers |
| `src/services/user-api-endpoint.service.ts` | **New** — CRUD, encrypt/decrypt, test-on-save |
| `src/services/dream.service.ts` (or dream controller) | Edit — handle `userEndpointUuid` in dream creation, decrypt key, add to job payload |
| `src/utils/prompt.util.ts` | Edit — recognize user endpoint jobs, route to `"user-endpoint"` queue |

---

## Worker Handler

### New Queue

`"user-endpoint"` — single queue for all user endpoint jobs. Handler inspects `userEndpointProvider` in job data and delegates to the appropriate adapter.

### OpenAI Adapter (`openai-handler.service.ts`)

**Text-to-image:**
```
POST {endpointUrl}/images/generations
Authorization: Bearer {apiKey}
Body: { model, prompt, size, n }
Response: { data: [{ url | b64_json }] }
```

**Image-to-image:**
```
POST {endpointUrl}/images/edits
Authorization: Bearer {apiKey}
Body: { model, image (base64 or URL), prompt, size, n }
Response: { data: [{ url | b64_json }] }
```

### FAL Adapter (`fal-handler.service.ts`)

**Submit:**
```
POST {endpointUrl}
Authorization: Key {apiKey}
Body: { prompt, image_url?, image_size, num_images }
Response: { request_id, status } or synchronous { images: [{ url }] }
```

**Poll (if async):**
```
GET {statusUrl}
Authorization: Key {apiKey}
Response: { status: "IN_QUEUE" | "IN_PROGRESS" | "COMPLETED", progress?, images? }
```

FAL progress maps to `job.updateProgress()` for Socket.IO delivery.

### Shared Post-Processing (Both Adapters)

1. Download result image(s) to buffer
2. Upload to R2 via existing `uploadToR2` utility
3. Create image-type dream record with R2 URL
4. Update dream status to `processed`
5. If multiple results (n > 1 for variations), each image becomes a separate dream. The frontend tracks the set via `VariationCandidate[]` on the keyframe/transition — no backend parent-child linking needed.

### Error Handling

- Provider auth errors (401) → dream status `failed`, error in dream metadata ("Invalid API key"). No Bugsnag (user's key issue).
- Rate limits (429) → dream status `failed`, error "Rate limited — try again later". No Bugsnag.
- Provider server errors (500+) → dream status `failed`, Bugsnag alert.
- Network errors → dream status `failed`, Bugsnag alert.

### Files Changed (Worker)

| File | Change |
|------|--------|
| `src/services/openai-handler.service.ts` | **New** — OpenAI-compatible API adapter |
| `src/services/fal-handler.service.ts` | **New** — FAL API adapter |
| `src/workers/job-handlers.ts` | Edit — add `handleUserEndpointJob` handler, route by provider |
| `src/config/queues.ts` (or equivalent) | Edit — register `"user-endpoint"` queue |

---

## Frontend — Account Page

### React Query Hooks

New directory: `src/api/user-api-endpoints/`

| Hook | Purpose |
|------|---------|
| `useUserApiEndpoints()` | GET list, query key `"userApiEndpoints"` |
| `useCreateUserApiEndpoint()` | POST, invalidates list on success |
| `useUpdateUserApiEndpoint()` | PUT, invalidates list |
| `useDeleteUserApiEndpoint()` | DELETE, invalidates list |
| `useTestUserApiEndpoint()` | POST test for existing endpoints |

### Account Page Section

New component `api-endpoints-section.tsx` rendered below existing `ApiKeyCard` on profile page.

**Empty state:** Dashed border box, "No endpoints configured", "Add Endpoint" button.

**Populated state:** List of endpoint cards showing:
- Provider icon letter (F/O) with colored background
- Name and model
- Connection status indicator
- Masked key ("...x7Bf")
- Capability badges (t2i, i2i)
- Test / Edit / Delete buttons

### Add Endpoint Modal

Two-step flow inside a modal:

**Step 1 — Choose service:** List of preset cards (Flux Schnell, Flux Pro, gpt-image-1, Custom). Each shows name, description, price estimate, capabilities.

**Step 2 — Enter key:** Pre-filled name (editable) + API key password input + "Get a key" link to provider. Save button calls create mutation. Error shown inline if test fails.

**Custom preset:** Step 2 shows additional fields — endpoint URL, model ID, provider type dropdown (OpenAI/FAL), capability checkboxes.

**Edit:** Reuses Step 2 form, pre-filled. Key field shows "••••••x7Bf" placeholder — only replaced if user types a new key.

### Files Changed (Frontend — Account)

| File | Change |
|------|--------|
| `src/api/user-api-endpoints/useUserApiEndpoints.ts` | **New** — query hook |
| `src/api/user-api-endpoints/useCreateUserApiEndpoint.ts` | **New** — mutation |
| `src/api/user-api-endpoints/useUpdateUserApiEndpoint.ts` | **New** — mutation |
| `src/api/user-api-endpoints/useDeleteUserApiEndpoint.ts` | **New** — mutation |
| `src/api/user-api-endpoints/useTestUserApiEndpoint.ts` | **New** — mutation |
| `src/components/pages/profile/api-endpoints-section.tsx` | **New** — list + empty state |
| `src/components/pages/profile/api-endpoints-section.styled.tsx` | **New** — styles |
| `src/components/pages/profile/add-endpoint-modal.tsx` | **New** — preset picker + key form |
| `src/components/pages/profile/add-endpoint-modal.styled.tsx` | **New** — modal styles |
| `src/components/pages/profile/profile.page.tsx` | Edit — render `ApiEndpointsSection` |
| `src/constants/endpoint-presets.ts` | **New** — preset template definitions |
| `src/types/user-api-endpoint.types.ts` | **New** — TypeScript types |

---

## Frontend — Studio Integration

### Model Picker Changes

Images tab model dropdown and flow builder transition settings panel model dropdown: append user endpoints after platform models.

```
Z Image Turbo
Qwen Image
Flux Schnell · your key
gpt-image-1 · your key
──────────────
+ Add endpoint...
```

"Add endpoint..." opens the account page in a new tab.

User endpoints fetched via `useUserApiEndpoints()`, cached by React Query. Only endpoints with `capabilities.textToImage` appear in image model pickers.

### Flow Store Additions

```typescript
// New state (not persisted — session-level, resets on switch)
i2iEndpointUuid: string | null;
setI2iEndpoint: (uuid: string | null) => void;
```

### i2i Variation Flow

1. User clicks keyframe → Variations → "Vary (i2i)" button
2. Button is grayed out if no i2i-capable endpoints are configured
3. If `i2iEndpointUuid` is null, small inline picker appears to set the session default
4. Creates a dream via `POST /v1/dream` with `userEndpointUuid` + source image URL
5. Backend decrypts key, queues to `"user-endpoint"` queue
6. Worker calls provider, uploads result to R2
7. Progress tracked via existing `useFlowJobProgress` — Socket.IO + polling, unchanged
8. Result appears in variation grid as `VariationCandidate` with `method: "i2i"`

### Image Generation Flow

1. Images tab: user selects a user endpoint model, enters prompt, clicks generate
2. `POST /v1/dream` with `userEndpointUuid` and prompt (no source image)
3. Result is an image-type dream — same output format as Qwen/ZIT
4. User adds result to flow as a keyframe via existing "My Images" picker

### Files Changed (Frontend — Studio)

| File | Change |
|------|--------|
| `src/components/pages/studio/components/images-tab.tsx` | Edit — append user endpoints to model dropdown |
| `src/components/pages/studio/components/transition-settings-panel.tsx` | Edit — append user endpoints to model dropdown (if applicable) |
| `src/components/pages/studio/components/keyframe-card.tsx` | Edit — add "Vary (i2i)" option to variations menu |
| `src/components/pages/studio/components/flow-builder.tsx` | Edit — handle i2i variation dream creation |
| `src/stores/flow.store.ts` | Edit — add `i2iEndpointUuid` + setter |

---

## Anti-Rationalization Table

| Rationalization | Reality |
|----------------|---------|
| "We should build a generic adapter system for any provider." | Two adapters (OpenAI + FAL) cover the actual use cases. A generic system adds weeks for hypothetical providers. Add adapters when real users need them. |
| "The worker should look up the endpoint itself." | Backend already has DB access and crypto utils. Passing the decrypted key in job data avoids new DB connections in the worker. Redis is already trusted with session tokens. |
| "Users should configure capabilities manually for all presets." | Presets pre-fill everything. Users just paste a key. Manual config is the Custom escape hatch, not the default path. |
| "We need auto-detection of endpoint capabilities." | Auto-detect is fragile and adds API surface. Preset templates solve 95% of use cases. Custom is manual fallback. |
| "i2i should use the endpoint that created the keyframe." | Breaks for uploaded images (most keyframes). Session-level default is simpler and handles all sources. |
| "Test connection should be a separate explicit step." | Save-tests-then-saves is one click. Separate test button adds friction — users will skip it, save a bad key, and wonder why generation fails. |
| "User endpoints should work in batch mode too." | Batch mode adds scope without adding user value. Keyframe source in flow mode is Jay's actual ask. Batch integration is additive later. |

---

## Out of Scope

- Batch mode integration for user endpoints (Phase 1 is keyframe source only)
- Video generation via user endpoints (image-only in Phase 1)
- MidJourney integration (no stable API)
- Endpoint sharing between users
- Usage tracking / billing for user endpoint calls
- Admin dashboard for user endpoint management
