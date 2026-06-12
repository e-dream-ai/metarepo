# API Key Config & i2i Variations — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let users bring their own API keys for Flux (via FAL) and OpenAI image models, use them to generate keyframe images and i2i variations in the studio.

**Architecture:** New `UserApiEndpoint` entity in backend with encrypted key storage. Backend decrypts key and passes it in BullMQ job data to worker. Worker has two adapter services (OpenAI, FAL) that call external APIs, download results, upload to R2. Frontend adds endpoint CRUD on account page with preset templates, extends studio model pickers with user endpoints, and wires i2i variations into the flow builder.

**Tech Stack:** TypeScript/Express/TypeORM/BullMQ (backend), TypeScript/BullMQ/Axios (worker), React 18/TypeScript/Zustand/React Query/styled-components (frontend)

**Spec:** `docs/superpowers/specs/2026-06-12-api-key-config-i2i-variations-design.md`

**Base branch:** `stage` (in each repo — assumes Phase 0 variations/sessions has landed)

**Repos:** Changes span three repos, all symlinked from metarepo:
- `backend` → `/Users/maxcarlsonold/edream/backend`
- `worker` → `/Users/maxcarlsonold/edream/worker`
- `frontend` → `/Users/maxcarlsonold/edream/frontend`

---

## File Structure

### Backend (7 new, 3 modified)

| File | Responsibility | New/Edit |
|------|---------------|----------|
| `src/entities/UserApiEndpoint.entity.ts` | TypeORM entity for user API endpoints | New |
| `src/migration/XXXX-AddUserApiEndpoints.ts` | Database migration | New |
| `src/services/user-api-endpoint.service.ts` | CRUD, encryption, test-on-save | New |
| `src/controllers/user-api-endpoint.controller.ts` | Route handlers | New |
| `src/routes/v1/user-api-endpoint.routes.ts` | Express routes | New |
| `src/services/endpoint-tester.service.ts` | OpenAI + FAL connection test logic | New |
| `src/types/user-api-endpoint.types.ts` | Request/response types | New |
| `src/routes/v1/router.ts` | Register new routes | Edit |
| `src/utils/dream.util.ts` | Handle userEndpointUuid in dream processing | Edit |
| `src/utils/prompt.util.ts` | Add user-endpoint queue routing | Edit |

### Worker (3 new, 1 modified)

| File | Responsibility | New/Edit |
|------|---------------|----------|
| `src/services/openai-handler.service.ts` | OpenAI-compatible API adapter | New |
| `src/services/fal-handler.service.ts` | FAL API adapter | New |
| `src/services/user-endpoint-handler.service.ts` | Router: delegates to OpenAI or FAL adapter | New |
| `src/index.ts` | Register user-endpoint queue + worker | Edit |

### Frontend (12 new, 6 modified)

| File | Responsibility | New/Edit |
|------|---------------|----------|
| `src/types/user-api-endpoint.types.ts` | TypeScript types | New |
| `src/constants/endpoint-presets.ts` | Preset template definitions | New |
| `src/api/user-api-endpoints/useUserApiEndpoints.ts` | Query hook — list endpoints | New |
| `src/api/user-api-endpoints/useCreateUserApiEndpoint.ts` | Mutation hook — create | New |
| `src/api/user-api-endpoints/useUpdateUserApiEndpoint.ts` | Mutation hook — update | New |
| `src/api/user-api-endpoints/useDeleteUserApiEndpoint.ts` | Mutation hook — delete | New |
| `src/api/user-api-endpoints/useTestUserApiEndpoint.ts` | Mutation hook — test existing | New |
| `src/components/pages/profile/api-endpoints-section.tsx` | Account page endpoint list | New |
| `src/components/pages/profile/api-endpoints-section.styled.tsx` | Styles | New |
| `src/components/pages/profile/add-endpoint-modal.tsx` | Preset picker + key form | New |
| `src/components/pages/profile/add-endpoint-modal.styled.tsx` | Modal styles | New |
| `src/components/shared/profile-card/profile-card.tsx` | Render ApiEndpointsSection | Edit |
| `src/components/pages/studio/components/images-tab.tsx` | Add user endpoints to model picker | Edit |
| `src/components/pages/studio/components/transition-settings-panel.tsx` | Add user endpoints to model picker | Edit |
| `src/components/pages/studio/components/keyframe-card.tsx` | Add "Vary (i2i)" button | Edit |
| `src/components/pages/studio/components/flow-builder.tsx` | Handle i2i dream creation | Edit |
| `src/stores/flow.store.ts` | Add i2iEndpointUuid state | Edit |

---

### Task 1: Backend — Entity and Migration

**Files:**
- Create: `backend/src/entities/UserApiEndpoint.entity.ts`
- Create: `backend/src/types/user-api-endpoint.types.ts`

- [ ] **Step 1: Create the types file**

```typescript
// backend/src/types/user-api-endpoint.types.ts

export type EndpointProviderType = "openai" | "fal";

export interface EndpointCapabilities {
  textToImage: boolean;
  imageToImage: boolean;
  sizes: string[];
}

export interface CreateUserApiEndpointRequest {
  name: string;
  providerType: EndpointProviderType;
  presetId: string;
  endpointUrl: string;
  apiKey: string; // plaintext — encrypted before storage
  modelId: string;
  capabilities: EndpointCapabilities;
}

export interface UpdateUserApiEndpointRequest {
  name?: string;
  endpointUrl?: string;
  apiKey?: string; // plaintext — only present if user is changing the key
  modelId?: string;
  capabilities?: EndpointCapabilities;
}

export interface UserApiEndpointResponse {
  uuid: string;
  name: string;
  providerType: EndpointProviderType;
  presetId: string;
  endpointUrl: string;
  apiKeyLastFour: string;
  modelId: string;
  capabilities: EndpointCapabilities;
  createdAt: Date;
  updatedAt: Date;
}
```

- [ ] **Step 2: Create the entity**

```typescript
// backend/src/entities/UserApiEndpoint.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  ManyToOne,
  JoinColumn,
  CreateDateColumn,
  UpdateDateColumn,
  Index,
  Generated,
} from "typeorm";
import { User } from "./User.entity";
import type {
  EndpointProviderType,
  EndpointCapabilities,
} from "../types/user-api-endpoint.types";

@Entity("user_api_endpoint")
export class UserApiEndpoint {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ type: "varchar", unique: true })
  @Generated("uuid")
  @Index()
  uuid: string;

  @Column()
  @Index()
  userId: number;

  @ManyToOne(() => User)
  @JoinColumn({ name: "userId" })
  user: User;

  @Column()
  name: string;

  @Column()
  providerType: EndpointProviderType;

  @Column()
  presetId: string;

  @Column()
  endpointUrl: string;

  @Column()
  apiKeyEncrypted: string;

  @Column()
  apiKeyIv: string;

  @Column({ length: 4 })
  apiKeyLastFour: string;

  @Column()
  modelId: string;

  @Column({ type: "jsonb" })
  capabilities: EndpointCapabilities;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

- [ ] **Step 3: Generate the migration**

Run: `cd backend && pnpm run migration:generate AddUserApiEndpoints`

This auto-generates the migration from the entity diff. Verify the generated file creates the `user_api_endpoint` table with all columns and the indexes on `uuid` and `userId`.

- [ ] **Step 4: Run the migration**

Run: `cd backend && pnpm run migration:run`
Expected: Migration completes successfully, `user_api_endpoint` table created.

- [ ] **Step 5: Commit**

```bash
cd backend
git add src/entities/UserApiEndpoint.entity.ts src/types/user-api-endpoint.types.ts src/migration/*AddUserApiEndpoints*
git commit -m "feat: add UserApiEndpoint entity and migration"
```

---

### Task 2: Backend — Endpoint Tester Service

**Files:**
- Create: `backend/src/services/endpoint-tester.service.ts`

**Context:** This service tests connectivity to external endpoints by making a minimal API call. Used during create (auto-test-on-save) and for the re-test route.

- [ ] **Step 1: Create the tester service**

```typescript
// backend/src/services/endpoint-tester.service.ts
import axios from "axios";
import type { EndpointProviderType } from "../types/user-api-endpoint.types";
import { APP_LOGGER } from "../utils/logger.util";

interface TestResult {
  success: boolean;
  error?: string;
}

/**
 * Test connectivity to an external API endpoint.
 * Makes a minimal request to verify the API key is valid.
 */
export async function testEndpointConnection(
  providerType: EndpointProviderType,
  endpointUrl: string,
  apiKey: string,
  modelId: string,
): Promise<TestResult> {
  try {
    if (providerType === "openai") {
      return await testOpenAiEndpoint(endpointUrl, apiKey, modelId);
    } else if (providerType === "fal") {
      return await testFalEndpoint(endpointUrl, apiKey);
    }
    return { success: false, error: `Unknown provider type: ${providerType}` };
  } catch (err: unknown) {
    const message =
      err instanceof Error ? err.message : "Unknown error";
    APP_LOGGER.error(`Endpoint test failed: ${message}`);
    return { success: false, error: message };
  }
}

async function testOpenAiEndpoint(
  endpointUrl: string,
  apiKey: string,
  modelId: string,
): Promise<TestResult> {
  try {
    // Use the models endpoint to validate the key without generating an image
    const response = await axios.get(`${endpointUrl}/models`, {
      headers: { Authorization: `Bearer ${apiKey}` },
      timeout: 15000,
    });
    if (response.status === 200) {
      return { success: true };
    }
    return { success: false, error: `Unexpected status: ${response.status}` };
  } catch (err: unknown) {
    if (axios.isAxiosError(err)) {
      if (err.response?.status === 401) {
        return { success: false, error: "Invalid API key" };
      }
      if (err.response?.status === 403) {
        return { success: false, error: "API key lacks required permissions" };
      }
      return {
        success: false,
        error: `Connection failed: ${err.response?.status ?? err.message}`,
      };
    }
    throw err;
  }
}

async function testFalEndpoint(
  endpointUrl: string,
  apiKey: string,
): Promise<TestResult> {
  try {
    // FAL: submit a minimal request — it will queue but validates the key
    const response = await axios.post(
      endpointUrl,
      { prompt: "test", image_size: "square", num_images: 1 },
      {
        headers: { Authorization: `Key ${apiKey}` },
        timeout: 15000,
      },
    );
    // 200 (sync) or 201/202 (async queued) both mean the key works
    if (response.status >= 200 && response.status < 300) {
      return { success: true };
    }
    return { success: false, error: `Unexpected status: ${response.status}` };
  } catch (err: unknown) {
    if (axios.isAxiosError(err)) {
      if (err.response?.status === 401) {
        return { success: false, error: "Invalid API key" };
      }
      if (err.response?.status === 403) {
        return { success: false, error: "API key lacks required permissions" };
      }
      // FAL returns 422 for bad params but that still means auth passed
      if (err.response?.status === 422) {
        return { success: true };
      }
      return {
        success: false,
        error: `Connection failed: ${err.response?.status ?? err.message}`,
      };
    }
    throw err;
  }
}
```

- [ ] **Step 2: Commit**

```bash
cd backend
git add src/services/endpoint-tester.service.ts
git commit -m "feat: add endpoint tester service for OpenAI and FAL"
```

---

### Task 3: Backend — CRUD Service

**Files:**
- Create: `backend/src/services/user-api-endpoint.service.ts`

**Context:** Handles create/read/update/delete for user API endpoints. Encrypts API keys before storage, decrypts for test and job dispatch. Uses `crypto.util.ts` (AES-256-CBC with `CIPHER_KEY`).

- [ ] **Step 1: Create the service**

```typescript
// backend/src/services/user-api-endpoint.service.ts
import appDataSource from "database/app-data-source";
import { UserApiEndpoint } from "../entities/UserApiEndpoint.entity";
import { encrypt, decrypt } from "../utils/crypto.util";
import { testEndpointConnection } from "./endpoint-tester.service";
import type {
  CreateUserApiEndpointRequest,
  UpdateUserApiEndpointRequest,
  UserApiEndpointResponse,
} from "../types/user-api-endpoint.types";

function lastFour(apiKey: string): string {
  return apiKey.slice(-4);
}

function toResponse(entity: UserApiEndpoint): UserApiEndpointResponse {
  return {
    uuid: entity.uuid,
    name: entity.name,
    providerType: entity.providerType,
    presetId: entity.presetId,
    endpointUrl: entity.endpointUrl,
    apiKeyLastFour: entity.apiKeyLastFour,
    modelId: entity.modelId,
    capabilities: entity.capabilities,
    createdAt: entity.createdAt,
    updatedAt: entity.updatedAt,
  };
}

export class UserApiEndpointService {
  async list(userId: number): Promise<UserApiEndpointResponse[]> {
    const repo = appDataSource.getRepository(UserApiEndpoint);
    const endpoints = await repo.find({
      where: { userId },
      order: { createdAt: "ASC" },
    });
    return endpoints.map(toResponse);
  }

  async getByUuid(
    uuid: string,
    userId: number,
  ): Promise<UserApiEndpoint | null> {
    const repo = appDataSource.getRepository(UserApiEndpoint);
    return repo.findOne({ where: { uuid, userId } });
  }

  async create(
    userId: number,
    data: CreateUserApiEndpointRequest,
  ): Promise<{ endpoint?: UserApiEndpointResponse; error?: string }> {
    // Test connection before saving
    const testResult = await testEndpointConnection(
      data.providerType,
      data.endpointUrl,
      data.apiKey,
      data.modelId,
    );
    if (!testResult.success) {
      return { error: testResult.error ?? "Connection test failed" };
    }

    const { iv, content } = encrypt(data.apiKey);
    const repo = appDataSource.getRepository(UserApiEndpoint);
    const entity = repo.create({
      userId,
      name: data.name,
      providerType: data.providerType,
      presetId: data.presetId,
      endpointUrl: data.endpointUrl,
      apiKeyEncrypted: content,
      apiKeyIv: iv,
      apiKeyLastFour: lastFour(data.apiKey),
      modelId: data.modelId,
      capabilities: data.capabilities,
    });
    const saved = await repo.save(entity);
    return { endpoint: toResponse(saved) };
  }

  async update(
    uuid: string,
    userId: number,
    data: UpdateUserApiEndpointRequest,
  ): Promise<{ endpoint?: UserApiEndpointResponse; error?: string }> {
    const repo = appDataSource.getRepository(UserApiEndpoint);
    const entity = await repo.findOne({ where: { uuid, userId } });
    if (!entity) return { error: "Endpoint not found" };

    // If key is changing, test with the new key
    if (data.apiKey) {
      const testResult = await testEndpointConnection(
        entity.providerType,
        data.endpointUrl ?? entity.endpointUrl,
        data.apiKey,
        data.modelId ?? entity.modelId,
      );
      if (!testResult.success) {
        return { error: testResult.error ?? "Connection test failed" };
      }
      const { iv, content } = encrypt(data.apiKey);
      entity.apiKeyEncrypted = content;
      entity.apiKeyIv = iv;
      entity.apiKeyLastFour = maskKey(data.apiKey);
    }

    if (data.name !== undefined) entity.name = data.name;
    if (data.endpointUrl !== undefined) entity.endpointUrl = data.endpointUrl;
    if (data.modelId !== undefined) entity.modelId = data.modelId;
    if (data.capabilities !== undefined)
      entity.capabilities = data.capabilities;

    const saved = await repo.save(entity);
    return { endpoint: toResponse(saved) };
  }

  async delete(uuid: string, userId: number): Promise<boolean> {
    const repo = appDataSource.getRepository(UserApiEndpoint);
    const entity = await repo.findOne({ where: { uuid, userId } });
    if (!entity) return false;
    await repo.remove(entity);
    return true;
  }

  async test(
    uuid: string,
    userId: number,
  ): Promise<{ success: boolean; error?: string }> {
    const repo = appDataSource.getRepository(UserApiEndpoint);
    const entity = await repo.findOne({ where: { uuid, userId } });
    if (!entity) return { success: false, error: "Endpoint not found" };
    const apiKey = decrypt({
      iv: entity.apiKeyIv,
      content: entity.apiKeyEncrypted,
    });
    return testEndpointConnection(
      entity.providerType,
      entity.endpointUrl,
      apiKey,
      entity.modelId,
    );
  }

  /**
   * Decrypt the API key for a user endpoint.
   * Used by dream creation to pass decrypted key in BullMQ job data.
   */
  async decryptKey(
    uuid: string,
    userId: number,
  ): Promise<{
    apiKey: string;
    endpointUrl: string;
    providerType: string;
    modelId: string;
  } | null> {
    const repo = appDataSource.getRepository(UserApiEndpoint);
    const entity = await repo.findOne({ where: { uuid, userId } });
    if (!entity) return null;
    const apiKey = decrypt({
      iv: entity.apiKeyIv,
      content: entity.apiKeyEncrypted,
    });
    return {
      apiKey,
      endpointUrl: entity.endpointUrl,
      providerType: entity.providerType,
      modelId: entity.modelId,
    };
  }
}

export const userApiEndpointService = new UserApiEndpointService();
```

- [ ] **Step 2: Commit**

```bash
cd backend
git add src/services/user-api-endpoint.service.ts
git commit -m "feat: add UserApiEndpoint CRUD service with encryption and test-on-save"
```

---

### Task 4: Backend — Controller and Routes

**Files:**
- Create: `backend/src/controllers/user-api-endpoint.controller.ts`
- Create: `backend/src/routes/v1/user-api-endpoint.routes.ts`
- Modify: `backend/src/routes/v1/router.ts`

- [ ] **Step 1: Create the controller**

```typescript
// backend/src/controllers/user-api-endpoint.controller.ts
import httpStatus from "http-status";
import { userApiEndpointService } from "../services/user-api-endpoint.service";
import { jsonResponse } from "utils/responses.util";
import type { RequestType, ResponseType } from "types/express.types";
import type {
  CreateUserApiEndpointRequest,
  UpdateUserApiEndpointRequest,
} from "../types/user-api-endpoint.types";

export const handleListEndpoints = async (
  req: RequestType,
  res: ResponseType,
) => {
  try {
    const userId = res.locals.user.id;
    const endpoints = await userApiEndpointService.list(userId);
    return res
      .status(httpStatus.OK)
      .json(jsonResponse({ endpoints }, true, "Endpoints retrieved"));
  } catch (err) {
    return res
      .status(httpStatus.INTERNAL_SERVER_ERROR)
      .json(jsonResponse(null, false, "Failed to list endpoints"));
  }
};

export const handleCreateEndpoint = async (
  req: RequestType<CreateUserApiEndpointRequest>,
  res: ResponseType,
) => {
  try {
    const userId = res.locals.user.id;
    const result = await userApiEndpointService.create(userId, req.body);
    if (result.error) {
      return res
        .status(httpStatus.UNPROCESSABLE_ENTITY)
        .json(jsonResponse(null, false, result.error));
    }
    return res
      .status(httpStatus.CREATED)
      .json(jsonResponse({ endpoint: result.endpoint }, true, "Endpoint created"));
  } catch (err) {
    return res
      .status(httpStatus.INTERNAL_SERVER_ERROR)
      .json(jsonResponse(null, false, "Failed to create endpoint"));
  }
};

export const handleUpdateEndpoint = async (
  req: RequestType<UpdateUserApiEndpointRequest, unknown, { uuid: string }>,
  res: ResponseType,
) => {
  try {
    const userId = res.locals.user.id;
    const { uuid } = req.params;
    const result = await userApiEndpointService.update(uuid, userId, req.body);
    if (result.error) {
      const status = result.error === "Endpoint not found"
        ? httpStatus.NOT_FOUND
        : httpStatus.UNPROCESSABLE_ENTITY;
      return res
        .status(status)
        .json(jsonResponse(null, false, result.error));
    }
    return res
      .status(httpStatus.OK)
      .json(jsonResponse({ endpoint: result.endpoint }, true, "Endpoint updated"));
  } catch (err) {
    return res
      .status(httpStatus.INTERNAL_SERVER_ERROR)
      .json(jsonResponse(null, false, "Failed to update endpoint"));
  }
};

export const handleDeleteEndpoint = async (
  req: RequestType<unknown, unknown, { uuid: string }>,
  res: ResponseType,
) => {
  try {
    const userId = res.locals.user.id;
    const { uuid } = req.params;
    const deleted = await userApiEndpointService.delete(uuid, userId);
    if (!deleted) {
      return res
        .status(httpStatus.NOT_FOUND)
        .json(jsonResponse(null, false, "Endpoint not found"));
    }
    return res
      .status(httpStatus.OK)
      .json(jsonResponse(null, true, "Endpoint deleted"));
  } catch (err) {
    return res
      .status(httpStatus.INTERNAL_SERVER_ERROR)
      .json(jsonResponse(null, false, "Failed to delete endpoint"));
  }
};

export const handleTestEndpoint = async (
  req: RequestType<unknown, unknown, { uuid: string }>,
  res: ResponseType,
) => {
  try {
    const userId = res.locals.user.id;
    const { uuid } = req.params;
    const result = await userApiEndpointService.test(uuid, userId);
    if (!result.success) {
      return res
        .status(httpStatus.UNPROCESSABLE_ENTITY)
        .json(jsonResponse(null, false, result.error ?? "Test failed"));
    }
    return res
      .status(httpStatus.OK)
      .json(jsonResponse(null, true, "Connection successful"));
  } catch (err) {
    return res
      .status(httpStatus.INTERNAL_SERVER_ERROR)
      .json(jsonResponse(null, false, "Failed to test endpoint"));
  }
};
```

- [ ] **Step 2: Create the routes file**

```typescript
// backend/src/routes/v1/user-api-endpoint.routes.ts
import { Router } from "express";
import { requireAuth } from "../../middlewares/require-auth.middleware";
import {
  handleListEndpoints,
  handleCreateEndpoint,
  handleUpdateEndpoint,
  handleDeleteEndpoint,
  handleTestEndpoint,
} from "../../controllers/user-api-endpoint.controller";

export const userApiEndpointRouter = Router();

userApiEndpointRouter.get("/", requireAuth, handleListEndpoints);
userApiEndpointRouter.post("/", requireAuth, handleCreateEndpoint);
userApiEndpointRouter.put("/:uuid", requireAuth, handleUpdateEndpoint);
userApiEndpointRouter.delete("/:uuid", requireAuth, handleDeleteEndpoint);
userApiEndpointRouter.post("/:uuid/test", requireAuth, handleTestEndpoint);
```

- [ ] **Step 3: Register routes in router.ts**

In `backend/src/routes/v1/router.ts`, add:

```typescript
// Add import at top
import { userApiEndpointRouter } from "./user-api-endpoint.routes";

// Add route registration alongside existing routes (near the dream router registration)
app.use("/api/v1/user/api-endpoints", userApiEndpointRouter);
```

- [ ] **Step 4: Verify build**

Run: `cd backend && pnpm run build`
Expected: No compilation errors.

- [ ] **Step 5: Commit**

```bash
cd backend
git add src/controllers/user-api-endpoint.controller.ts src/routes/v1/user-api-endpoint.routes.ts src/routes/v1/router.ts
git commit -m "feat: add UserApiEndpoint CRUD routes and controller"
```

---

### Task 5: Backend — Dream Creation Queue Change

**Files:**
- Modify: `backend/src/utils/dream.util.ts`
- Modify: `backend/src/utils/prompt.util.ts`

**Context:** When a dream's prompt JSON contains `userEndpointUuid`, the backend looks up the endpoint, decrypts the key, and passes it in the BullMQ job data. The job routes to the `"user-endpoint"` queue.

- [ ] **Step 1: Add user-endpoint queue routing to prompt.util.ts**

In `backend/src/utils/prompt.util.ts`, add a helper to detect user endpoint jobs:

```typescript
// Add after the existing ALGORITHM_TO_QUEUE_MAP

export const isUserEndpointJob = (promptJson: Record<string, unknown>): boolean => {
  return typeof promptJson.userEndpointUuid === "string" &&
    promptJson.userEndpointUuid.length > 0;
};

export const USER_ENDPOINT_QUEUE = "user-endpoint";
```

- [ ] **Step 2: Modify dream.util.ts to handle user endpoint jobs**

In `backend/src/utils/dream.util.ts`, in the `processDreamRequest` function, add user endpoint handling **before** calling `parsePromptJson`. The reason: `parsePromptJson` returns `null` when `infinidream_algorithm` is missing, and user endpoint prompts intentionally omit that field. We must intercept user endpoint jobs from the raw prompt string before the algorithm-based routing.

```typescript
// Add import at top
import { userApiEndpointService } from "../services/user-api-endpoint.service";
import { USER_ENDPOINT_QUEUE } from "./prompt.util";

// Inside processDreamRequest, BEFORE the existing parsePromptJson call, add:

// Check for user endpoint job before algorithm-based routing.
// parsePromptJson returns null when infinidream_algorithm is missing,
// so we must intercept user endpoint jobs from the raw prompt first.
if (dream.prompt) {
  let rawPrompt: Record<string, unknown> | null = null;
  try {
    rawPrompt = typeof dream.prompt === "string"
      ? JSON.parse(dream.prompt)
      : dream.prompt;
  } catch {
    // Not JSON — fall through to existing logic
  }

  if (rawPrompt?.userEndpointUuid) {
    const endpointUuid = rawPrompt.userEndpointUuid as string;
    const userId = dream.user.id;
    const endpointData = await userApiEndpointService.decryptKey(
      endpointUuid,
      userId,
    );

    if (!endpointData) {
      APP_LOGGER.error(
        `User endpoint not found: ${endpointUuid} for user ${userId}`,
      );
      return;
    }

    const jobData = {
      dream_uuid: dream.uuid,
      userEndpointDecryptedKey: endpointData.apiKey,
      userEndpointUrl: endpointData.endpointUrl,
      userEndpointProvider: endpointData.providerType,
      userEndpointModelId: endpointData.modelId,
      prompt: rawPrompt.prompt as string,
      image: rawPrompt.image as string | undefined,
      size: rawPrompt.size as string | undefined,
      n: rawPrompt.n as number | undefined,
    };

    await queueWorkerJob(USER_ENDPOINT_QUEUE, jobData);
    return;
  }
}

// Existing parsePromptJson + algorithm routing continues below...
```

- [ ] **Step 3: Verify build**

Run: `cd backend && pnpm run build`
Expected: No compilation errors.

- [ ] **Step 4: Commit**

```bash
cd backend
git add src/utils/dream.util.ts src/utils/prompt.util.ts
git commit -m "feat: route user endpoint dreams to dedicated queue with decrypted key"
```

---

### Task 6: Worker — OpenAI Adapter

**Files:**
- Create: `worker/src/services/openai-handler.service.ts`

**Context:** Calls OpenAI-compatible image APIs (generations and edits), downloads results, uploads to R2. Returns an array of R2 URLs.

- [ ] **Step 1: Create the OpenAI adapter**

```typescript
// worker/src/services/openai-handler.service.ts
import axios from "axios";
import type { Job } from "bullmq";
import { R2UploadService } from "./r2-upload.service";
import { APP_LOGGER } from "../shared/logger";

const r2UploadService = new R2UploadService();

export interface OpenAiResult {
  r2Urls: string[];
  renderDuration: number;
}

export async function handleOpenAiJob(
  job: Job,
  endpointUrl: string,
  apiKey: string,
  modelId: string,
  prompt: string,
  imageUrl?: string,
  size?: string,
  n?: number,
): Promise<OpenAiResult> {
  const startTime = Date.now();
  const count = n ?? 1;
  const resolvedSize = size ?? "1024x1024";

  await job.updateProgress({ status: "IN_PROGRESS", progress: 10 });

  let responseData: Array<{ url?: string; b64_json?: string }>;

  if (imageUrl) {
    // Image-to-image (edits)
    // OpenAI /images/edits requires multipart/form-data, not JSON.
    APP_LOGGER.info(`OpenAI i2i: ${modelId}, n=${count}`);

    // Download source image as buffer
    const imageResponse = await axios.get(imageUrl, {
      responseType: "arraybuffer",
      timeout: 30000,
    });
    const imageBuffer = Buffer.from(imageResponse.data);

    // Build multipart form data
    const FormData = (await import("form-data")).default;
    const form = new FormData();
    form.append("model", modelId);
    form.append("image", imageBuffer, {
      filename: "source.png",
      contentType: "image/png",
    });
    form.append("prompt", prompt);
    form.append("size", resolvedSize);
    form.append("n", String(count));

    const { data } = await axios.post(
      `${endpointUrl}/images/edits`,
      form,
      {
        headers: {
          Authorization: `Bearer ${apiKey}`,
          ...form.getHeaders(),
        },
        timeout: 120000,
      },
    );
    responseData = data.data;
  } else {
    // Text-to-image (generations)
    APP_LOGGER.info(`OpenAI t2i: ${modelId}, n=${count}`);
    const { data } = await axios.post(
      `${endpointUrl}/images/generations`,
      {
        model: modelId,
        prompt,
        size: resolvedSize,
        n: count,
      },
      {
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
        },
        timeout: 120000,
      },
    );
    responseData = data.data;
  }

  await job.updateProgress({ status: "IN_PROGRESS", progress: 60 });

  // Download and upload each result to R2
  const r2Urls: string[] = [];
  for (let i = 0; i < responseData.length; i++) {
    const item = responseData[i];
    let r2Url: string;

    if (item.url) {
      r2Url = await r2UploadService.downloadAndUploadImage(
        item.url,
        `${job.id}-${i}`,
      );
    } else if (item.b64_json) {
      // Write base64 to temp file, upload
      const buffer = Buffer.from(item.b64_json, "base64");
      const fs = await import("fs/promises");
      const os = await import("os");
      const path = await import("path");
      const tmpPath = path.join(os.tmpdir(), `openai-${job.id}-${i}.png`);
      await fs.writeFile(tmpPath, buffer);
      r2Url = await r2UploadService.uploadImageToR2(
        tmpPath,
        `${job.id}-${i}`,
      );
      await fs.unlink(tmpPath);
    } else {
      APP_LOGGER.warn(`OpenAI result ${i} has no url or b64_json`);
      continue;
    }

    r2Urls.push(r2Url);

    const uploadProgress = 60 + Math.round(((i + 1) / responseData.length) * 30);
    await job.updateProgress({ status: "IN_PROGRESS", progress: uploadProgress });
  }

  const renderDuration = Date.now() - startTime;
  return { r2Urls, renderDuration };
}
```

- [ ] **Step 2: Commit**

```bash
cd worker
git add src/services/openai-handler.service.ts
git commit -m "feat: add OpenAI adapter for user endpoint jobs"
```

---

### Task 7: Worker — FAL Adapter

**Files:**
- Create: `worker/src/services/fal-handler.service.ts`

**Context:** Calls FAL API (async with polling), downloads results, uploads to R2.

- [ ] **Step 1: Create the FAL adapter**

```typescript
// worker/src/services/fal-handler.service.ts
import axios from "axios";
import type { Job } from "bullmq";
import { R2UploadService } from "./r2-upload.service";
import { APP_LOGGER } from "../shared/logger";

const r2UploadService = new R2UploadService();

export interface FalResult {
  r2Urls: string[];
  renderDuration: number;
}

const FAL_POLL_INTERVAL_MS = 3000;
const FAL_TIMEOUT_MS = 300000; // 5 minutes

export async function handleFalJob(
  job: Job,
  endpointUrl: string,
  apiKey: string,
  prompt: string,
  imageUrl?: string,
  size?: string,
  n?: number,
): Promise<FalResult> {
  const startTime = Date.now();
  const count = n ?? 1;

  await job.updateProgress({ status: "IN_PROGRESS", progress: 5 });

  // Build FAL request body
  const body: Record<string, unknown> = {
    prompt,
    num_images: count,
  };

  if (size) {
    // FAL uses image_size as an object or string like "landscape_16_9"
    body.image_size = parseFalSize(size);
  }

  if (imageUrl) {
    body.image_url = imageUrl;
  }

  APP_LOGGER.info(
    `FAL ${imageUrl ? "i2i" : "t2i"}: n=${count}, url=${endpointUrl}`,
  );

  // Submit request
  const { data: submitData } = await axios.post(endpointUrl, body, {
    headers: {
      Authorization: `Key ${apiKey}`,
      "Content-Type": "application/json",
    },
    timeout: 30000,
  });

  // Check if response is synchronous (has images directly)
  if (submitData.images) {
    return await processResults(job, submitData.images, startTime);
  }

  // Async — poll for completion
  const requestId = submitData.request_id;
  if (!requestId) {
    throw new Error("FAL returned no request_id and no images");
  }

  // FAL async polling uses queue.fal.run, not fal.run.
  // Extract the app path from endpointUrl (e.g., "fal-ai/flux/schnell" from
  // "https://fal.run/fal-ai/flux/schnell") and build queue URLs.
  const appPath = new URL(endpointUrl).pathname.replace(/^\//, "");
  const statusUrl = `https://queue.fal.run/${appPath}/requests/${requestId}/status`;
  const resultUrl = `https://queue.fal.run/${appPath}/requests/${requestId}`;

  let elapsed = 0;
  while (elapsed < FAL_TIMEOUT_MS) {
    await new Promise((resolve) => setTimeout(resolve, FAL_POLL_INTERVAL_MS));
    elapsed += FAL_POLL_INTERVAL_MS;

    try {
      const { data: statusData } = await axios.get(statusUrl, {
        headers: { Authorization: `Key ${apiKey}` },
        timeout: 10000,
      });

      if (statusData.status === "COMPLETED") {
        // Fetch the full result
        const { data: resultData } = await axios.get(resultUrl, {
          headers: { Authorization: `Key ${apiKey}` },
          timeout: 10000,
        });
        return await processResults(
          job,
          resultData.images ?? resultData.output?.images ?? [],
          startTime,
        );
      }

      if (statusData.status === "FAILED") {
        throw new Error(
          `FAL job failed: ${statusData.error ?? "unknown error"}`,
        );
      }

      // Map queue position or progress to percentage
      const progress = statusData.queue_position != null
        ? Math.min(10 + Math.round((1 / (statusData.queue_position + 1)) * 40), 50)
        : 30;

      await job.updateProgress({ status: "IN_PROGRESS", progress });
    } catch (err) {
      if (axios.isAxiosError(err) && err.response?.status === 404) {
        // Request not found — might still be starting
        continue;
      }
      throw err;
    }
  }

  throw new Error(`FAL job timed out after ${FAL_TIMEOUT_MS}ms`);
}

async function processResults(
  job: Job,
  images: Array<{ url: string }>,
  startTime: number,
): Promise<FalResult> {
  await job.updateProgress({ status: "IN_PROGRESS", progress: 60 });

  const r2Urls: string[] = [];
  for (let i = 0; i < images.length; i++) {
    const img = images[i];
    if (!img.url) continue;

    const r2Url = await r2UploadService.downloadAndUploadImage(
      img.url,
      `${job.id}-${i}`,
    );
    r2Urls.push(r2Url);

    const uploadProgress =
      60 + Math.round(((i + 1) / images.length) * 30);
    await job.updateProgress({ status: "IN_PROGRESS", progress: uploadProgress });
  }

  const renderDuration = Date.now() - startTime;
  return { r2Urls, renderDuration };
}

function parseFalSize(size: string): string {
  // Convert "1024x1024" → FAL format
  // FAL accepts { width, height } or preset strings
  const parts = size.split("x");
  if (parts.length === 2) {
    const w = parseInt(parts[0], 10);
    const h = parseInt(parts[1], 10);
    if (w === h) return "square";
    if (w > h) return "landscape_16_9";
    return "portrait_9_16";
  }
  return size;
}
```

- [ ] **Step 2: Commit**

```bash
cd worker
git add src/services/fal-handler.service.ts
git commit -m "feat: add FAL adapter for user endpoint jobs"
```

---

### Task 8: Worker — Queue Registration and Job Router

**Files:**
- Create: `worker/src/services/user-endpoint-handler.service.ts`
- Modify: `worker/src/index.ts`

**Context:** Register the `user-endpoint` queue, create a handler that routes to OpenAI or FAL based on job data.

- [ ] **Step 1: Create the job router**

```typescript
// worker/src/services/user-endpoint-handler.service.ts
import type { Job } from "bullmq";
import { handleOpenAiJob } from "./openai-handler.service";
import { handleFalJob } from "./fal-handler.service";
import { VideoServiceClient } from "./video-service.client";
import { APP_LOGGER } from "../shared/logger";

const videoServiceClient = new VideoServiceClient();

export async function handleUserEndpointJob(job: Job): Promise<unknown> {
  const {
    dream_uuid,
    userEndpointDecryptedKey,
    userEndpointUrl,
    userEndpointProvider,
    userEndpointModelId,
    prompt,
    image,
    size,
    n,
  } = job.data;

  if (!dream_uuid) throw new Error("Missing dream_uuid");
  if (!userEndpointDecryptedKey) throw new Error("Missing API key in job data");
  if (!userEndpointUrl) throw new Error("Missing endpoint URL in job data");
  if (!userEndpointProvider) throw new Error("Missing provider type in job data");
  if (!prompt) throw new Error("Missing prompt in job data");

  APP_LOGGER.info(
    `User endpoint job: provider=${userEndpointProvider}, dream=${dream_uuid}`,
  );

  try {
    let result: { r2Urls: string[]; renderDuration: number };

    if (userEndpointProvider === "openai") {
      result = await handleOpenAiJob(
        job,
        userEndpointUrl,
        userEndpointDecryptedKey,
        userEndpointModelId,
        prompt,
        image,
        size,
        n,
      );
    } else if (userEndpointProvider === "fal") {
      result = await handleFalJob(
        job,
        userEndpointUrl,
        userEndpointDecryptedKey,
        prompt,
        image,
        size,
        n,
      );
    } else {
      throw new Error(`Unknown provider type: ${userEndpointProvider}`);
    }

    if (result.r2Urls.length === 0) {
      throw new Error("No images returned from provider");
    }

    // Upload the primary result as the dream's image.
    // r2Urls are already presigned R2 URLs (uploaded by the adapter).
    // videoServiceClient.uploadGeneratedImage detects the R2 hostname
    // and skips re-upload, just sets the dream's image path.
    await videoServiceClient.uploadGeneratedImage(
      dream_uuid,
      result.r2Urls[0],
      result.renderDuration,
    );

    // Log if additional images were generated but not attached.
    // Each i2i variation fires a separate dream (n=1 per call),
    // so n>1 in a single job is not expected in normal flow.
    if (result.r2Urls.length > 1) {
      APP_LOGGER.warn(
        `User endpoint job produced ${result.r2Urls.length} images but only the first was attached to dream ${dream_uuid}. Additional images: ${result.r2Urls.slice(1).join(", ")}`,
      );
    }

    await job.updateProgress({
      status: "COMPLETED",
      progress: 100,
      r2_url: result.r2Urls[0],
    });

    return {
      r2_url: result.r2Urls[0],
      render_duration: result.renderDuration,
    };
  } catch (err: unknown) {
    const message = err instanceof Error ? err.message : "Unknown error";
    APP_LOGGER.error(`User endpoint job failed: ${message}`);

    await job.updateProgress({
      status: "FAILED",
      progress: 0,
      error: message,
    });

    throw err;
  }
}
```

- [ ] **Step 2: Register queue in index.ts**

In `worker/src/index.ts`, add the user-endpoint queue alongside the existing queue registrations:

```typescript
// Add import at top
import { handleUserEndpointJob } from "./services/user-endpoint-handler.service.js";

// Add alongside existing WorkerFactory.createWorker calls:
WorkerFactory.createWorker("user-endpoint", handleUserEndpointJob);

// Add queue instance for Bull Board (alongside existing queue instances):
const userEndpointQueue = new Queue("user-endpoint", {
  connection: redisConnection,
});

// Add to Bull Board adapters array:
new BullMQAdapter(userEndpointQueue),
```

- [ ] **Step 3: Verify build**

Run: `cd worker && npm run build`
Expected: No compilation errors.

- [ ] **Step 4: Commit**

```bash
cd worker
git add src/services/user-endpoint-handler.service.ts src/index.ts
git commit -m "feat: register user-endpoint queue with OpenAI/FAL routing"
```

---

### Task 9: Frontend — Types and Preset Constants

**Files:**
- Create: `frontend/src/types/user-api-endpoint.types.ts`
- Create: `frontend/src/constants/endpoint-presets.ts`

- [ ] **Step 1: Create types**

```typescript
// frontend/src/types/user-api-endpoint.types.ts

export type EndpointProviderType = "openai" | "fal";

export interface EndpointCapabilities {
  textToImage: boolean;
  imageToImage: boolean;
  sizes: string[];
}

export interface UserApiEndpoint {
  uuid: string;
  name: string;
  providerType: EndpointProviderType;
  presetId: string;
  endpointUrl: string;
  apiKeyLastFour: string;
  modelId: string;
  capabilities: EndpointCapabilities;
  createdAt: string;
  updatedAt: string;
}

export interface CreateUserApiEndpointParams {
  name: string;
  providerType: EndpointProviderType;
  presetId: string;
  endpointUrl: string;
  apiKey: string;
  modelId: string;
  capabilities: EndpointCapabilities;
}

export interface UpdateUserApiEndpointParams {
  uuid: string;
  name?: string;
  endpointUrl?: string;
  apiKey?: string;
  modelId?: string;
  capabilities?: EndpointCapabilities;
}
```

- [ ] **Step 2: Create preset constants**

```typescript
// frontend/src/constants/endpoint-presets.ts
import type {
  EndpointProviderType,
  EndpointCapabilities,
} from "@/types/user-api-endpoint.types";

export interface EndpointPreset {
  id: string;
  name: string;
  providerType: EndpointProviderType;
  endpointUrl: string;
  modelId: string;
  capabilities: EndpointCapabilities;
  description: string;
  keyUrl: string;
}

export const ENDPOINT_PRESETS: EndpointPreset[] = [
  {
    id: "flux-schnell",
    name: "FAL — Flux Schnell",
    providerType: "fal",
    endpointUrl: "https://fal.run/fal-ai/flux/schnell",
    modelId: "fal-ai/flux/schnell",
    capabilities: {
      textToImage: true,
      imageToImage: true,
      sizes: ["1024x1024", "1280x720", "720x1280"],
    },
    description: "Fast image generation & i2i · ~$0.003/image",
    keyUrl: "https://fal.ai/dashboard/keys",
  },
  {
    id: "flux-pro",
    name: "FAL — Flux Pro",
    providerType: "fal",
    endpointUrl: "https://fal.run/fal-ai/flux-pro",
    modelId: "fal-ai/flux-pro",
    capabilities: {
      textToImage: true,
      imageToImage: true,
      sizes: ["1024x1024", "1280x720", "720x1280"],
    },
    description: "Higher quality, slower · ~$0.05/image",
    keyUrl: "https://fal.ai/dashboard/keys",
  },
  {
    id: "openai-gpt-image-1",
    name: "OpenAI — gpt-image-1",
    providerType: "openai",
    endpointUrl: "https://api.openai.com/v1",
    modelId: "gpt-image-1",
    capabilities: {
      textToImage: true,
      imageToImage: true,
      sizes: ["1024x1024", "1536x1024", "1024x1536"],
    },
    description: "High quality image generation & editing · ~$0.02–0.19/image",
    keyUrl: "https://platform.openai.com/api-keys",
  },
];
```

- [ ] **Step 3: Commit**

```bash
cd frontend
git add src/types/user-api-endpoint.types.ts src/constants/endpoint-presets.ts
git commit -m "feat: add UserApiEndpoint types and preset constants"
```

---

### Task 10: Frontend — React Query Hooks

**Files:**
- Create: `frontend/src/api/user-api-endpoints/useUserApiEndpoints.ts`
- Create: `frontend/src/api/user-api-endpoints/useCreateUserApiEndpoint.ts`
- Create: `frontend/src/api/user-api-endpoints/useUpdateUserApiEndpoint.ts`
- Create: `frontend/src/api/user-api-endpoints/useDeleteUserApiEndpoint.ts`
- Create: `frontend/src/api/user-api-endpoints/useTestUserApiEndpoint.ts`

- [ ] **Step 1: Create query hook**

```typescript
// frontend/src/api/user-api-endpoints/useUserApiEndpoints.ts
import { useQuery } from "@tanstack/react-query";
import { axiosClient } from "@/client/axios.client";
import { ContentType, getRequestHeaders } from "@/constants/auth.constants";
import type { ApiResponse } from "@/types/api.types";
import type { UserApiEndpoint } from "@/types/user-api-endpoint.types";
import useAuth from "@/hooks/useAuth";

export const USER_API_ENDPOINTS_QUERY_KEY = "userApiEndpoints";

const getEndpoints = () => {
  return async () =>
    axiosClient
      .get("/v1/user/api-endpoints", {
        headers: getRequestHeaders({ contentType: ContentType.json }),
      })
      .then((res) => res.data);
};

export const useUserApiEndpoints = () => {
  const { user } = useAuth();
  return useQuery<ApiResponse<{ endpoints: UserApiEndpoint[] }>, Error>(
    [USER_API_ENDPOINTS_QUERY_KEY],
    getEndpoints(),
    { enabled: Boolean(user) },
  );
};
```

- [ ] **Step 2: Create mutation hooks**

```typescript
// frontend/src/api/user-api-endpoints/useCreateUserApiEndpoint.ts
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { axiosClient } from "@/client/axios.client";
import { ContentType, getRequestHeaders } from "@/constants/auth.constants";
import type { ApiResponse } from "@/types/api.types";
import type {
  UserApiEndpoint,
  CreateUserApiEndpointParams,
} from "@/types/user-api-endpoint.types";
import { USER_API_ENDPOINTS_QUERY_KEY } from "./useUserApiEndpoints";

const createEndpoint = async (params: CreateUserApiEndpointParams) => {
  const { data } = await axiosClient.post<
    ApiResponse<{ endpoint: UserApiEndpoint }>
  >("/v1/user/api-endpoints", params, {
    headers: getRequestHeaders({ contentType: ContentType.json }),
  });
  return data;
};

export const useCreateUserApiEndpoint = () => {
  const queryClient = useQueryClient();
  return useMutation<
    ApiResponse<{ endpoint: UserApiEndpoint }>,
    Error,
    CreateUserApiEndpointParams
  >(createEndpoint, {
    mutationKey: ["createUserApiEndpoint"],
    onSuccess: () => {
      queryClient.invalidateQueries([USER_API_ENDPOINTS_QUERY_KEY]);
    },
  });
};
```

```typescript
// frontend/src/api/user-api-endpoints/useUpdateUserApiEndpoint.ts
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { axiosClient } from "@/client/axios.client";
import { ContentType, getRequestHeaders } from "@/constants/auth.constants";
import type { ApiResponse } from "@/types/api.types";
import type {
  UserApiEndpoint,
  UpdateUserApiEndpointParams,
} from "@/types/user-api-endpoint.types";
import { USER_API_ENDPOINTS_QUERY_KEY } from "./useUserApiEndpoints";

const updateEndpoint = async ({ uuid, ...params }: UpdateUserApiEndpointParams) => {
  const { data } = await axiosClient.put<
    ApiResponse<{ endpoint: UserApiEndpoint }>
  >(`/v1/user/api-endpoints/${uuid}`, params, {
    headers: getRequestHeaders({ contentType: ContentType.json }),
  });
  return data;
};

export const useUpdateUserApiEndpoint = () => {
  const queryClient = useQueryClient();
  return useMutation<
    ApiResponse<{ endpoint: UserApiEndpoint }>,
    Error,
    UpdateUserApiEndpointParams
  >(updateEndpoint, {
    mutationKey: ["updateUserApiEndpoint"],
    onSuccess: () => {
      queryClient.invalidateQueries([USER_API_ENDPOINTS_QUERY_KEY]);
    },
  });
};
```

```typescript
// frontend/src/api/user-api-endpoints/useDeleteUserApiEndpoint.ts
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { axiosClient } from "@/client/axios.client";
import { ContentType, getRequestHeaders } from "@/constants/auth.constants";
import type { ApiResponse } from "@/types/api.types";
import { USER_API_ENDPOINTS_QUERY_KEY } from "./useUserApiEndpoints";

const deleteEndpoint = async (uuid: string) => {
  const { data } = await axiosClient.delete<ApiResponse<null>>(
    `/v1/user/api-endpoints/${uuid}`,
    { headers: getRequestHeaders({ contentType: ContentType.json }) },
  );
  return data;
};

export const useDeleteUserApiEndpoint = () => {
  const queryClient = useQueryClient();
  return useMutation<ApiResponse<null>, Error, string>(deleteEndpoint, {
    mutationKey: ["deleteUserApiEndpoint"],
    onSuccess: () => {
      queryClient.invalidateQueries([USER_API_ENDPOINTS_QUERY_KEY]);
    },
  });
};
```

```typescript
// frontend/src/api/user-api-endpoints/useTestUserApiEndpoint.ts
import { useMutation } from "@tanstack/react-query";
import { axiosClient } from "@/client/axios.client";
import { ContentType, getRequestHeaders } from "@/constants/auth.constants";
import type { ApiResponse } from "@/types/api.types";

const testEndpoint = async (uuid: string) => {
  const { data } = await axiosClient.post<ApiResponse<null>>(
    `/v1/user/api-endpoints/${uuid}/test`,
    {},
    { headers: getRequestHeaders({ contentType: ContentType.json }) },
  );
  return data;
};

export const useTestUserApiEndpoint = () => {
  return useMutation<ApiResponse<null>, Error, string>(testEndpoint, {
    mutationKey: ["testUserApiEndpoint"],
  });
};
```

- [ ] **Step 3: Run type-check**

Run: `cd frontend && pnpm type-check`
Expected: No errors.

- [ ] **Step 4: Commit**

```bash
cd frontend
git add src/api/user-api-endpoints/
git commit -m "feat: add React Query hooks for user API endpoint CRUD"
```

---

### Task 11: Frontend — Account Page Endpoints Section

**Files:**
- Create: `frontend/src/components/pages/profile/api-endpoints-section.styled.tsx`
- Create: `frontend/src/components/pages/profile/api-endpoints-section.tsx`
- Create: `frontend/src/components/pages/profile/add-endpoint-modal.styled.tsx`
- Create: `frontend/src/components/pages/profile/add-endpoint-modal.tsx`
- Modify: `frontend/src/components/shared/profile-card/profile-card.tsx`

**Context:** The profile page has a `ProfileCard` component that renders `ApiKeyCard` when `showApiKeyCard` is true. We add an `ApiEndpointsSection` below it. The "Add Endpoint" button opens a modal with the two-step preset flow.

- [ ] **Step 1: Create styled components for the section**

```typescript
// frontend/src/components/pages/profile/api-endpoints-section.styled.tsx
import styled from "styled-components";

export const SectionContainer = styled.div`
  margin-top: 1.5rem;
  padding: 1.5rem;
  background: rgba(0, 0, 0, 0.3);
  border: 1px solid ${(p) => p.theme.borderColor || "#2a2a2a"};
  border-radius: 12px;
`;

export const SectionHeader = styled.div`
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
`;

export const SectionTitle = styled.h3`
  font-size: 1rem;
  font-weight: 600;
  color: ${(p) => p.theme.textPrimaryColor};
  margin: 0;
`;

export const SectionSubtitle = styled.p`
  font-size: 0.75rem;
  color: ${(p) => p.theme.textSecondaryColor || "#888"};
  margin: 0.25rem 0 0;
`;

export const AddButton = styled.button`
  background: ${(p) => p.theme.textAccentColor || "#c9a227"};
  color: #0d0d0d;
  border: none;
  border-radius: 8px;
  padding: 0.5rem 1rem;
  font-size: 0.8rem;
  font-weight: 500;
  cursor: pointer;

  &:hover {
    opacity: 0.9;
  }
`;

export const EmptyState = styled.div`
  border: 1px dashed ${(p) => p.theme.borderColor || "#333"};
  border-radius: 8px;
  padding: 2rem;
  text-align: center;
  color: ${(p) => p.theme.textSecondaryColor || "#666"};
  font-size: 0.85rem;
`;

export const EndpointCard = styled.div`
  border: 1px solid ${(p) => p.theme.borderColor || "#2a2a2a"};
  border-radius: 8px;
  padding: 0.875rem 1rem;
  margin-bottom: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
`;

export const EndpointInfo = styled.div`
  display: flex;
  align-items: center;
  gap: 0.75rem;
`;

export const ProviderIcon = styled.div<{ $color: string }>`
  width: 2rem;
  height: 2rem;
  background: ${(p) => p.$color};
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.875rem;
  font-weight: 600;
  color: ${(p) => p.theme.textPrimaryColor};
`;

export const EndpointName = styled.div`
  font-size: 0.875rem;
  font-weight: 500;
  color: ${(p) => p.theme.textPrimaryColor};
`;

export const EndpointMeta = styled.div`
  font-size: 0.7rem;
  color: ${(p) => p.theme.textSecondaryColor || "#888"};
  margin-top: 0.125rem;
`;

export const StatusDot = styled.span<{ $color: string }>`
  color: ${(p) => p.$color};
`;

export const EndpointActions = styled.div`
  display: flex;
  gap: 0.5rem;
`;

export const ActionBtn = styled.button<{ $danger?: boolean }>`
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid ${(p) => p.theme.borderColor || "#333"};
  border-radius: 6px;
  color: ${(p) => (p.$danger ? "#ef4444" : p.theme.textSecondaryColor || "#aaa")};
  padding: 0.375rem 0.625rem;
  font-size: 0.75rem;
  cursor: pointer;

  &:hover {
    background: rgba(255, 255, 255, 0.1);
  }

  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
`;
```

- [ ] **Step 2: Create the section component**

```typescript
// frontend/src/components/pages/profile/api-endpoints-section.tsx
import { useState, useCallback } from "react";
import { useUserApiEndpoints } from "@/api/user-api-endpoints/useUserApiEndpoints";
import { useDeleteUserApiEndpoint } from "@/api/user-api-endpoints/useDeleteUserApiEndpoint";
import { useTestUserApiEndpoint } from "@/api/user-api-endpoints/useTestUserApiEndpoint";
import { AddEndpointModal } from "./add-endpoint-modal";
import { toast } from "react-toastify";
import type { UserApiEndpoint } from "@/types/user-api-endpoint.types";
import {
  SectionContainer,
  SectionHeader,
  SectionTitle,
  SectionSubtitle,
  AddButton,
  EmptyState,
  EndpointCard,
  EndpointInfo,
  ProviderIcon,
  EndpointName,
  EndpointMeta,
  StatusDot,
  EndpointActions,
  ActionBtn,
} from "./api-endpoints-section.styled";

const PROVIDER_COLORS: Record<string, string> = {
  fal: "#1a1a2e",
  openai: "#1a2e1a",
};

export function ApiEndpointsSection() {
  const { data } = useUserApiEndpoints();
  const deleteMutation = useDeleteUserApiEndpoint();
  const testMutation = useTestUserApiEndpoint();
  const [modalOpen, setModalOpen] = useState(false);
  const [editingEndpoint, setEditingEndpoint] =
    useState<UserApiEndpoint | null>(null);

  const endpoints = data?.data?.endpoints ?? [];

  const handleDelete = useCallback(
    (uuid: string) => {
      if (!confirm("Delete this endpoint?")) return;
      deleteMutation.mutate(uuid, {
        onSuccess: () => toast.success("Endpoint deleted"),
        onError: () => toast.error("Failed to delete endpoint"),
      });
    },
    [deleteMutation],
  );

  const handleTest = useCallback(
    (uuid: string) => {
      testMutation.mutate(uuid, {
        onSuccess: () => toast.success("Connection successful"),
        onError: (err) =>
          toast.error(err.message || "Connection failed"),
      });
    },
    [testMutation],
  );

  const handleEdit = useCallback((endpoint: UserApiEndpoint) => {
    setEditingEndpoint(endpoint);
    setModalOpen(true);
  }, []);

  const handleCloseModal = useCallback(() => {
    setModalOpen(false);
    setEditingEndpoint(null);
  }, []);

  return (
    <SectionContainer>
      <SectionHeader>
        <div>
          <SectionTitle>Your API Endpoints</SectionTitle>
          <SectionSubtitle>
            Bring your own AI models to the studio
          </SectionSubtitle>
        </div>
        <AddButton onClick={() => setModalOpen(true)}>
          + Add Endpoint
        </AddButton>
      </SectionHeader>

      {endpoints.length === 0 ? (
        <EmptyState>
          No endpoints configured.
          <br />
          Add an endpoint to use Flux, OpenAI, or other models in the studio.
        </EmptyState>
      ) : (
        endpoints.map((ep) => (
          <EndpointCard key={ep.uuid}>
            <EndpointInfo>
              <ProviderIcon
                $color={PROVIDER_COLORS[ep.providerType] ?? "#2a2a2a"}
              >
                {ep.providerType === "fal" ? "F" : "O"}
              </ProviderIcon>
              <div>
                <EndpointName>{ep.name}</EndpointName>
                <EndpointMeta>
                  <StatusDot $color="#4ade80">●</StatusDot> Key: ...{ep.apiKeyLastFour}
                  &nbsp;·&nbsp;
                  {[
                    ep.capabilities.textToImage && "t2i",
                    ep.capabilities.imageToImage && "i2i",
                  ]
                    .filter(Boolean)
                    .join(", ")}
                </EndpointMeta>
              </div>
            </EndpointInfo>
            <EndpointActions>
              <ActionBtn
                onClick={() => handleTest(ep.uuid)}
                disabled={testMutation.isLoading}
              >
                Test
              </ActionBtn>
              <ActionBtn onClick={() => handleEdit(ep)}>Edit</ActionBtn>
              <ActionBtn $danger onClick={() => handleDelete(ep.uuid)}>
                Delete
              </ActionBtn>
            </EndpointActions>
          </EndpointCard>
        ))
      )}

      <AddEndpointModal
        isOpen={modalOpen}
        onClose={handleCloseModal}
        editingEndpoint={editingEndpoint}
      />
    </SectionContainer>
  );
}
```

- [ ] **Step 3: Create styled components for the modal**

```typescript
// frontend/src/components/pages/profile/add-endpoint-modal.styled.tsx
import styled from "styled-components";

export const ModalOverlay = styled.div`
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
`;

export const ModalContent = styled.div`
  background: #0d0d0d;
  border: 1px solid #2a2a2a;
  border-radius: 12px;
  width: 480px;
  max-height: 80vh;
  overflow-y: auto;
  padding: 1.5rem;
`;

export const ModalTitle = styled.h3`
  font-size: 1rem;
  color: ${(p) => p.theme.textPrimaryColor};
  margin: 0 0 1.25rem;
`;

export const PresetCard = styled.div<{ $selected?: boolean }>`
  border: 1px solid ${(p) => (p.$selected ? "#c9a227" : "#333")};
  border-radius: 8px;
  padding: 0.875rem 1rem;
  margin-bottom: 0.5rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 0.75rem;

  &:hover {
    border-color: #555;
  }
`;

export const PresetIcon = styled.div<{ $color: string }>`
  width: 2.25rem;
  height: 2.25rem;
  background: ${(p) => p.$color};
  border-radius: 6px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1rem;
  font-weight: 600;
  color: ${(p) => p.theme.textPrimaryColor};
  flex-shrink: 0;
`;

export const PresetInfo = styled.div`
  flex: 1;
`;

export const PresetName = styled.div`
  font-size: 0.875rem;
  font-weight: 500;
  color: ${(p) => p.theme.textPrimaryColor};
`;

export const PresetDesc = styled.div`
  font-size: 0.7rem;
  color: ${(p) => p.theme.textSecondaryColor || "#888"};
  margin-top: 0.125rem;
`;

export const PresetCaps = styled.div`
  font-size: 0.7rem;
  color: #4ade80;
`;

export const FormGroup = styled.div`
  margin-bottom: 1rem;
`;

export const FormLabel = styled.label`
  display: block;
  font-size: 0.7rem;
  color: ${(p) => p.theme.textSecondaryColor || "#888"};
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-bottom: 0.375rem;
`;

export const FormInput = styled.input`
  width: 100%;
  background: #1a1a1a;
  border: 1px solid #333;
  border-radius: 6px;
  color: ${(p) => p.theme.textPrimaryColor};
  padding: 0.625rem 0.75rem;
  font-size: 0.8rem;
  box-sizing: border-box;

  &:focus {
    outline: none;
    border-color: #c9a227;
  }

  &::placeholder {
    color: #555;
  }
`;

export const FormHint = styled.div`
  font-size: 0.7rem;
  color: #666;
  margin-top: 0.375rem;

  a {
    color: #c9a227;
    text-decoration: none;
  }
`;

export const FormError = styled.div`
  font-size: 0.75rem;
  color: #ef4444;
  margin-top: 0.375rem;
`;

export const ButtonRow = styled.div`
  display: flex;
  gap: 0.625rem;
  justify-content: flex-end;
  margin-top: 1.25rem;
`;

export const ModalButton = styled.button<{
  $accent?: boolean;
  $outline?: boolean;
}>`
  background: ${(p) =>
    p.$accent ? "#c9a227" : p.$outline ? "transparent" : "#1a1a1a"};
  color: ${(p) => (p.$accent ? "#0d0d0d" : p.$outline ? "#aaa" : "#aaa")};
  border: 1px solid ${(p) => (p.$accent ? "#c9a227" : "#333")};
  border-radius: 8px;
  padding: 0.5rem 1rem;
  font-size: 0.8rem;
  cursor: pointer;
  font-weight: ${(p) => (p.$accent ? 500 : 400)};

  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
`;

export const CustomFields = styled.div`
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px solid #2a2a2a;
`;

export const FormSelect = styled.select`
  width: 100%;
  background: #1a1a1a;
  border: 1px solid #333;
  border-radius: 6px;
  color: ${(p) => p.theme.textPrimaryColor};
  padding: 0.625rem 0.75rem;
  font-size: 0.8rem;
  box-sizing: border-box;
  cursor: pointer;

  &:focus {
    outline: none;
    border-color: #c9a227;
  }
`;
```

- [ ] **Step 4: Create the modal component**

```typescript
// frontend/src/components/pages/profile/add-endpoint-modal.tsx
import { useState, useCallback, useEffect } from "react";
import { createPortal } from "react-dom";
import { useCreateUserApiEndpoint } from "@/api/user-api-endpoints/useCreateUserApiEndpoint";
import { useUpdateUserApiEndpoint } from "@/api/user-api-endpoints/useUpdateUserApiEndpoint";
import { ENDPOINT_PRESETS } from "@/constants/endpoint-presets";
import { toast } from "react-toastify";
import type { EndpointPreset } from "@/constants/endpoint-presets";
import type {
  UserApiEndpoint,
  EndpointProviderType,
} from "@/types/user-api-endpoint.types";
import {
  ModalOverlay,
  ModalContent,
  ModalTitle,
  PresetCard,
  PresetIcon,
  PresetInfo,
  PresetName,
  PresetDesc,
  PresetCaps,
  FormGroup,
  FormLabel,
  FormInput,
  FormSelect,
  FormHint,
  FormError,
  ButtonRow,
  ModalButton,
  CustomFields,
} from "./add-endpoint-modal.styled";

const PROVIDER_COLORS: Record<string, string> = {
  fal: "#1a1a2e",
  openai: "#1a2e1a",
};

interface AddEndpointModalProps {
  isOpen: boolean;
  onClose: () => void;
  editingEndpoint: UserApiEndpoint | null;
}

export function AddEndpointModal({
  isOpen,
  onClose,
  editingEndpoint,
}: AddEndpointModalProps) {
  const createMutation = useCreateUserApiEndpoint();
  const updateMutation = useUpdateUserApiEndpoint();

  const [step, setStep] = useState<"preset" | "form">(
    editingEndpoint ? "form" : "preset",
  );
  const [selectedPreset, setSelectedPreset] = useState<EndpointPreset | null>(
    null,
  );
  const [isCustom, setIsCustom] = useState(false);

  // Form fields
  const [name, setName] = useState("");
  const [apiKey, setApiKey] = useState("");
  const [endpointUrl, setEndpointUrl] = useState("");
  const [modelId, setModelId] = useState("");
  const [providerType, setProviderType] =
    useState<EndpointProviderType>("openai");
  const [error, setError] = useState("");

  // Reset state when modal opens/closes
  useEffect(() => {
    if (isOpen) {
      if (editingEndpoint) {
        setStep("form");
        setName(editingEndpoint.name);
        setApiKey("");
        setEndpointUrl(editingEndpoint.endpointUrl);
        setModelId(editingEndpoint.modelId);
        setProviderType(editingEndpoint.providerType);
        const preset = ENDPOINT_PRESETS.find(
          (p) => p.id === editingEndpoint.presetId,
        );
        setSelectedPreset(preset ?? null);
        setIsCustom(!preset);
      } else {
        setStep("preset");
        setSelectedPreset(null);
        setIsCustom(false);
        setName("");
        setApiKey("");
        setEndpointUrl("");
        setModelId("");
        setProviderType("openai");
      }
      setError("");
    }
  }, [isOpen, editingEndpoint]);

  const handleSelectPreset = useCallback((preset: EndpointPreset) => {
    setSelectedPreset(preset);
    setIsCustom(false);
    setName(preset.name);
    setEndpointUrl(preset.endpointUrl);
    setModelId(preset.modelId);
    setProviderType(preset.providerType);
    setStep("form");
  }, []);

  const handleSelectCustom = useCallback(() => {
    setSelectedPreset(null);
    setIsCustom(true);
    setName("");
    setEndpointUrl("");
    setModelId("");
    setProviderType("openai");
    setStep("form");
  }, []);

  const handleSave = useCallback(async () => {
    setError("");

    if (!name.trim()) {
      setError("Name is required");
      return;
    }
    if (!editingEndpoint && !apiKey.trim()) {
      setError("API key is required");
      return;
    }
    if (isCustom && !endpointUrl.trim()) {
      setError("Endpoint URL is required");
      return;
    }
    if (isCustom && !modelId.trim()) {
      setError("Model ID is required");
      return;
    }

    if (editingEndpoint) {
      updateMutation.mutate(
        {
          uuid: editingEndpoint.uuid,
          name: name.trim(),
          ...(apiKey.trim() ? { apiKey: apiKey.trim() } : {}),
          ...(isCustom
            ? { endpointUrl: endpointUrl.trim(), modelId: modelId.trim() }
            : {}),
        },
        {
          onSuccess: () => {
            toast.success("Endpoint updated");
            onClose();
          },
          onError: (err) => {
            setError(err.message || "Failed to update endpoint");
          },
        },
      );
    } else {
      const preset = selectedPreset;
      createMutation.mutate(
        {
          name: name.trim(),
          providerType: isCustom ? providerType : preset!.providerType,
          presetId: preset?.id ?? "custom",
          endpointUrl: isCustom ? endpointUrl.trim() : preset!.endpointUrl,
          apiKey: apiKey.trim(),
          modelId: isCustom ? modelId.trim() : preset!.modelId,
          capabilities: preset?.capabilities ?? {
            textToImage: true,
            imageToImage: true,
            sizes: ["1024x1024"],
          },
        },
        {
          onSuccess: () => {
            toast.success("Endpoint added");
            onClose();
          },
          onError: (err) => {
            setError(err.message || "Failed to create endpoint");
          },
        },
      );
    }
  }, [
    name,
    apiKey,
    endpointUrl,
    modelId,
    providerType,
    isCustom,
    selectedPreset,
    editingEndpoint,
    createMutation,
    updateMutation,
    onClose,
  ]);

  if (!isOpen) return null;

  const isSaving = createMutation.isLoading || updateMutation.isLoading;

  return createPortal(
    <ModalOverlay onClick={onClose}>
      <ModalContent onClick={(e) => e.stopPropagation()}>
        {step === "preset" ? (
          <>
            <ModalTitle>Add Endpoint — Choose Service</ModalTitle>
            {ENDPOINT_PRESETS.map((preset) => (
              <PresetCard
                key={preset.id}
                onClick={() => handleSelectPreset(preset)}
              >
                <PresetIcon
                  $color={PROVIDER_COLORS[preset.providerType] ?? "#2a2a2a"}
                >
                  {preset.providerType === "fal" ? "F" : "O"}
                </PresetIcon>
                <PresetInfo>
                  <PresetName>{preset.name}</PresetName>
                  <PresetDesc>{preset.description}</PresetDesc>
                </PresetInfo>
                <PresetCaps>
                  {[
                    preset.capabilities.textToImage && "t2i",
                    preset.capabilities.imageToImage && "i2i",
                  ]
                    .filter(Boolean)
                    .join(", ")}
                </PresetCaps>
              </PresetCard>
            ))}

            <PresetCard onClick={handleSelectCustom} style={{ opacity: 0.7 }}>
              <PresetIcon $color="#2a2a2a">⚙</PresetIcon>
              <PresetInfo>
                <PresetName>Custom OpenAI-Compatible</PresetName>
                <PresetDesc>
                  Any endpoint that speaks OpenAI format
                </PresetDesc>
              </PresetInfo>
              <PresetCaps style={{ color: "#888" }}>manual</PresetCaps>
            </PresetCard>
          </>
        ) : (
          <>
            <ModalTitle>
              {editingEndpoint ? "Edit Endpoint" : "Add Endpoint"}
            </ModalTitle>

            <FormGroup>
              <FormLabel>Display Name</FormLabel>
              <FormInput
                value={name}
                onChange={(e) => setName(e.target.value)}
                placeholder="My Flux Schnell"
              />
            </FormGroup>

            <FormGroup>
              <FormLabel>API Key</FormLabel>
              <FormInput
                type="password"
                value={apiKey}
                onChange={(e) => setApiKey(e.target.value)}
                placeholder={
                  editingEndpoint
                    ? `Current: ••••••••${editingEndpoint.apiKeyLastFour}`
                    : "Paste your API key"
                }
              />
              <FormHint>
                Your key is encrypted at rest and never shared.
                {selectedPreset && (
                  <>
                    {" "}
                    <a
                      href={selectedPreset.keyUrl}
                      target="_blank"
                      rel="noopener noreferrer"
                    >
                      Get a key →
                    </a>
                  </>
                )}
              </FormHint>
            </FormGroup>

            {isCustom && (
              <CustomFields>
                <FormGroup>
                  <FormLabel>Provider Type</FormLabel>
                  <FormSelect
                    value={providerType}
                    onChange={(e) =>
                      setProviderType(
                        e.target.value as EndpointProviderType,
                      )
                    }
                  >
                    <option value="openai">OpenAI-Compatible</option>
                    <option value="fal">FAL</option>
                  </FormSelect>
                </FormGroup>

                <FormGroup>
                  <FormLabel>Endpoint URL</FormLabel>
                  <FormInput
                    value={endpointUrl}
                    onChange={(e) => setEndpointUrl(e.target.value)}
                    placeholder="https://api.example.com/v1"
                  />
                </FormGroup>

                <FormGroup>
                  <FormLabel>Model ID</FormLabel>
                  <FormInput
                    value={modelId}
                    onChange={(e) => setModelId(e.target.value)}
                    placeholder="model-name"
                  />
                </FormGroup>
              </CustomFields>
            )}

            {error && <FormError>{error}</FormError>}

            <ButtonRow>
              {!editingEndpoint && (
                <ModalButton onClick={() => setStep("preset")}>
                  Back
                </ModalButton>
              )}
              <ModalButton $outline onClick={onClose}>
                Cancel
              </ModalButton>
              <ModalButton $accent onClick={handleSave} disabled={isSaving}>
                {isSaving ? "Saving..." : "Save"}
              </ModalButton>
            </ButtonRow>
          </>
        )}
      </ModalContent>
    </ModalOverlay>,
    document.body,
  );
}
```

- [ ] **Step 5: Wire into ProfileCard**

In `frontend/src/components/shared/profile-card/profile-card.tsx`, add the import and render `ApiEndpointsSection` below the existing `ApiKeyCard`:

```typescript
// Add import
import { ApiEndpointsSection } from "@/components/pages/profile/api-endpoints-section";

// In the JSX, after the ApiKeyCard rendering (inside the showApiKeyCard conditional):
{showApiKeyCard && (
  <>
    <ApiKeyCard user={user} />
    <ApiEndpointsSection />
  </>
)}
```

- [ ] **Step 6: Run type-check**

Run: `cd frontend && pnpm type-check`
Expected: No errors.

- [ ] **Step 7: Commit**

```bash
cd frontend
git add src/components/pages/profile/api-endpoints-section.tsx src/components/pages/profile/api-endpoints-section.styled.tsx src/components/pages/profile/add-endpoint-modal.tsx src/components/pages/profile/add-endpoint-modal.styled.tsx src/components/shared/profile-card/profile-card.tsx
git commit -m "feat: add API endpoints section and add-endpoint modal to profile page"
```

---

### Task 12: Frontend — Studio Model Picker and i2i Integration

**Files:**
- Modify: `frontend/src/components/pages/studio/components/images-tab.tsx`
- Modify: `frontend/src/components/pages/studio/components/keyframe-card.tsx`
- Modify: `frontend/src/components/pages/studio/components/flow-builder.tsx`
- Modify: `frontend/src/stores/flow.store.ts`

**Context:** Extend the images tab model dropdown with user endpoints. Add "Vary (i2i)" to keyframe card. Add `i2iEndpointUuid` to flow store.

- [ ] **Step 1: Add i2iEndpointUuid to flow store**

In `frontend/src/stores/flow.store.ts`, add to the store state type and implementation:

```typescript
// Add to FlowStoreState type:
i2iEndpointUuid: string | null;
setI2iEndpoint: (uuid: string | null) => void;

// Add to the store defaults (not persisted):
i2iEndpointUuid: null,
setI2iEndpoint: (uuid) => set({ i2iEndpointUuid: uuid }),

// In resetFlow, add:
i2iEndpointUuid: null,

// Do NOT add i2iEndpointUuid to partialize — it should not persist.
```

- [ ] **Step 2: Extend transition settings panel model picker**

In `frontend/src/components/pages/studio/components/transition-settings-panel.tsx`, add user endpoints to the model dropdown (same pattern as images tab):

```typescript
// Add import
import { useUserApiEndpoints } from "@/api/user-api-endpoints/useUserApiEndpoints";

// Inside the component:
const { data: endpointsData } = useUserApiEndpoints();
const userImageEndpoints = (endpointsData?.data?.endpoints ?? []).filter(
  (ep) => ep.capabilities.textToImage,
);

// In the model <Select>, after the existing model options, append:
{userImageEndpoints.map((ep) => (
  <option key={ep.uuid} value={ep.uuid}>
    {ep.name} · your key
  </option>
))}
```

- [ ] **Step 3: Extend images tab model picker**

In `frontend/src/components/pages/studio/components/images-tab.tsx`:

```typescript
// Add import
import { useUserApiEndpoints } from "@/api/user-api-endpoints/useUserApiEndpoints";

// Inside the component, add:
const { data: endpointsData } = useUserApiEndpoints();
const userEndpoints = (endpointsData?.data?.endpoints ?? []).filter(
  (ep) => ep.capabilities.textToImage,
);

// Replace the model <select> to include user endpoints:
<StyledSelect
  value={imageGenParams.model}
  onChange={(e) => {
    const val = e.target.value;
    // Check if it's a user endpoint UUID
    const userEp = userEndpoints.find((ep) => ep.uuid === val);
    if (userEp) {
      // Store user endpoint UUID in a local state for generation
      setSelectedUserEndpoint(userEp);
      return;
    }
    setSelectedUserEndpoint(null);
    const newModel = val as ImageModel;
    setImageGenParams({
      model: newModel,
      size: clampSizeToModel(imageGenParams.size, newModel),
    });
  }}
>
  {IMAGE_MODELS.map((m) => (
    <option key={m} value={m}>
      {MODEL_LABELS[m]}
    </option>
  ))}
  {userEndpoints.map((ep) => (
    <option key={ep.uuid} value={ep.uuid}>
      {ep.name} · your key
    </option>
  ))}
</StyledSelect>
```

Add local state for tracking user endpoint selection:

```typescript
const [selectedUserEndpoint, setSelectedUserEndpoint] =
  useState<UserApiEndpoint | null>(null);
```

In the `handleGenerate` function, when `selectedUserEndpoint` is set, use the user endpoint:

```typescript
// When selectedUserEndpoint is not null, build prompt with userEndpointUuid
const algoParams = selectedUserEndpoint
  ? {
      userEndpointUuid: selectedUserEndpoint.uuid,
      prompt: imagePrompt,
      size: selectedUserEndpoint.capabilities.sizes[0] ?? "1024x1024",
    }
  : {
      infinidream_algorithm: imageGenParams.model,
      prompt: imagePrompt,
      size: imageGenParams.size,
      seed,
    };
```

- [ ] **Step 3: Add i2i variation to keyframe card**

In `frontend/src/components/pages/studio/components/keyframe-card.tsx`, add a "Vary (i2i)" button to the variations menu:

```typescript
// Add import
import { useUserApiEndpoints } from "@/api/user-api-endpoints/useUserApiEndpoints";
import { useFlowStore } from "@/stores/flow.store";

// Inside the component:
const { data: endpointsData } = useUserApiEndpoints();
const hasI2iEndpoints = (endpointsData?.data?.endpoints ?? []).some(
  (ep) => ep.capabilities.imageToImage,
);
const i2iEndpointUuid = useFlowStore((s) => s.i2iEndpointUuid);

// Add "Vary (i2i)" button alongside existing variation buttons:
<VariationButton
  disabled={!hasI2iEndpoints}
  onClick={() => onRequestI2iVariation?.(keyframe)}
  title={hasI2iEndpoints ? "Generate i2i variations" : "Configure an i2i endpoint in account settings"}
>
  Vary (i2i)
</VariationButton>
```

- [ ] **Step 4: Wire i2i generation in flow-builder.tsx**

In `frontend/src/components/pages/studio/components/flow-builder.tsx`, add the i2i variation handler:

```typescript
// Add import
import { useUserApiEndpoints } from "@/api/user-api-endpoints/useUserApiEndpoints";

// Inside FlowBuilder component:
const i2iEndpointUuid = useFlowStore((s) => s.i2iEndpointUuid);
const setI2iEndpoint = useFlowStore((s) => s.setI2iEndpoint);
const { data: endpointsData } = useUserApiEndpoints();
const i2iEndpoints = (endpointsData?.data?.endpoints ?? []).filter(
  (ep) => ep.capabilities.imageToImage,
);

const handleI2iVariation = useCallback(
  async (keyframe: FlowKeyframe) => {
    let endpointUuid = i2iEndpointUuid;

    // If no default set and multiple i2i endpoints exist, show picker.
    // If only one i2i endpoint, auto-select it.
    if (!endpointUuid) {
      if (i2iEndpoints.length === 0) return;
      if (i2iEndpoints.length === 1) {
        endpointUuid = i2iEndpoints[0].uuid;
        setI2iEndpoint(endpointUuid);
      } else {
        // Multiple endpoints — show picker inline.
        // For now, auto-select first. TODO: replace with inline picker UI.
        endpointUuid = i2iEndpoints[0].uuid;
        setI2iEndpoint(endpointUuid);
      }
    }

    if (!endpointUuid) return;

    const imageUrl = keyframe.imageUrl;
    if (!imageUrl) return;

    const variationCount = 4;
    const candidates: VariationCandidate[] = Array.from(
      { length: variationCount },
      () => ({
        id: crypto.randomUUID(),
        method: "i2i" as const,
        status: "queue" as const,
      }),
    );

    // Add candidates to store
    addKeyframeVariations(keyframe.id, candidates);

    // Fire all dream creation calls concurrently (not sequential).
    // AGENTS.md: "A sequential for-loop is fine for Generate All" → Reality: No.
    const headers = getRequestHeaders({ contentType: ContentType.json });
    const promises = candidates.map((candidate, i) => {
      const algoParams = {
        userEndpointUuid: endpointUuid,
        prompt: `variation of ${keyframe.name || "image"}`,
        image: keyframe.dreamUuid ?? imageUrl,
        n: 1,
      };

      return axiosClient
        .post(
          "/v1/dream",
          {
            name: `i2i variation ${i + 1}`,
            prompt: JSON.stringify(algoParams),
          },
          { headers },
        )
        .then(({ data }) => {
          const dreamUuid = data?.data?.dream?.uuid;
          if (dreamUuid) {
            updateKeyframeVariation(keyframe.id, candidate.id, {
              dreamUuid,
              status: "queue",
            });
          }
        })
        .catch(() => {
          updateKeyframeVariation(keyframe.id, candidate.id, {
            status: "failed",
          });
        });
    });

    await Promise.allSettled(promises);
  },
  [
    i2iEndpointUuid,
    i2iEndpoints,
    setI2iEndpoint,
    addKeyframeVariations,
    updateKeyframeVariation,
  ],
);
```

Pass `onRequestI2iVariation={handleI2iVariation}` to the keyframe card component.

- [ ] **Step 5: Run type-check**

Run: `cd frontend && pnpm type-check`
Expected: No errors.

- [ ] **Step 6: Run tests**

Run: `cd frontend && pnpm vitest run`
Expected: All tests pass.

- [ ] **Step 7: Commit**

```bash
cd frontend
git add src/stores/flow.store.ts src/components/pages/studio/components/images-tab.tsx src/components/pages/studio/components/transition-settings-panel.tsx src/components/pages/studio/components/keyframe-card.tsx src/components/pages/studio/components/flow-builder.tsx
git commit -m "feat: integrate user endpoints into studio model picker and add i2i variations"
```

---

### Task 13: Final Integration and Verification

**Files:** All previously created/modified files.

- [ ] **Step 1: Backend — full build and lint**

Run: `cd backend && pnpm run build && pnpm run lint`
Expected: No errors.

- [ ] **Step 2: Worker — full build**

Run: `cd worker && npm run build`
Expected: No errors.

- [ ] **Step 3: Frontend — type-check, tests, build**

Run: `cd frontend && pnpm type-check && pnpm vitest run && pnpm build`
Expected: All pass, successful build.

- [ ] **Step 4: Manual smoke test checklist**

Start backend, worker, and frontend dev servers.

Verify:
1. Navigate to profile page — "Your API Endpoints" section appears below API Key card
2. Click "Add Endpoint" — modal opens with preset list
3. Select "FAL — Flux Schnell" — step 2 shows name + key form
4. Enter a valid FAL API key, click Save — endpoint created (auto-test passes)
5. Enter an invalid key, click Save — error shown inline ("Invalid API key")
6. Endpoint appears in the list with masked key and capabilities
7. Click "Test" on existing endpoint — success toast
8. Click "Edit" — modal opens with pre-filled values
9. Click "Delete" — confirm dialog, endpoint removed
10. Navigate to studio → Images tab → model dropdown shows "Flux Schnell · your key"
11. Select Flux endpoint, enter prompt, generate — image created via FAL
12. Generated image appears in "My Images"
13. Add image as flow keyframe
14. Click keyframe → Variations → "Vary (i2i)" — i2i variations generated
15. Variation results appear in the grid
16. Click a variation to swap it into the keyframe slot

- [ ] **Step 5: Commit any fixes from smoke testing**

```bash
# In each repo that has fixes:
git add -u
git commit -m "fix: address issues found during smoke testing"
```

---

## File Summary

### New Files (22)

| Repo | File | Purpose |
|------|------|---------|
| backend | `src/entities/UserApiEndpoint.entity.ts` | Entity |
| backend | `src/types/user-api-endpoint.types.ts` | Types |
| backend | `src/migration/XXXX-AddUserApiEndpoints.ts` | Migration |
| backend | `src/services/endpoint-tester.service.ts` | Connection testing |
| backend | `src/services/user-api-endpoint.service.ts` | CRUD + encryption |
| backend | `src/controllers/user-api-endpoint.controller.ts` | Route handlers |
| backend | `src/routes/v1/user-api-endpoint.routes.ts` | Express routes |
| worker | `src/services/openai-handler.service.ts` | OpenAI adapter |
| worker | `src/services/fal-handler.service.ts` | FAL adapter |
| worker | `src/services/user-endpoint-handler.service.ts` | Job router |
| frontend | `src/types/user-api-endpoint.types.ts` | Types |
| frontend | `src/constants/endpoint-presets.ts` | Preset definitions |
| frontend | `src/api/user-api-endpoints/useUserApiEndpoints.ts` | Query hook |
| frontend | `src/api/user-api-endpoints/useCreateUserApiEndpoint.ts` | Mutation |
| frontend | `src/api/user-api-endpoints/useUpdateUserApiEndpoint.ts` | Mutation |
| frontend | `src/api/user-api-endpoints/useDeleteUserApiEndpoint.ts` | Mutation |
| frontend | `src/api/user-api-endpoints/useTestUserApiEndpoint.ts` | Mutation |
| frontend | `src/components/pages/profile/api-endpoints-section.tsx` | Account UI |
| frontend | `src/components/pages/profile/api-endpoints-section.styled.tsx` | Styles |
| frontend | `src/components/pages/profile/add-endpoint-modal.tsx` | Add/Edit modal |
| frontend | `src/components/pages/profile/add-endpoint-modal.styled.tsx` | Modal styles |

### Modified Files (9)

| Repo | File | Change |
|------|------|--------|
| backend | `src/routes/v1/router.ts` | Register endpoint routes |
| backend | `src/utils/dream.util.ts` | Handle userEndpointUuid in dream processing |
| backend | `src/utils/prompt.util.ts` | Add user-endpoint queue detection |
| worker | `src/index.ts` | Register user-endpoint queue + worker |
| frontend | `src/components/shared/profile-card/profile-card.tsx` | Render ApiEndpointsSection |
| frontend | `src/components/pages/studio/components/images-tab.tsx` | User endpoints in model picker |
| frontend | `src/components/pages/studio/components/transition-settings-panel.tsx` | User endpoints in model picker |
| frontend | `src/components/pages/studio/components/keyframe-card.tsx` | Vary (i2i) button |
| frontend | `src/components/pages/studio/components/flow-builder.tsx` | i2i dream creation |
| frontend | `src/stores/flow.store.ts` | i2iEndpointUuid state |
