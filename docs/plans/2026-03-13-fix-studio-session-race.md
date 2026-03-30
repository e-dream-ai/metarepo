# Fix Studio Session Race Condition

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Fix the concurrent WorkOS session refresh race condition that causes sign-outs on the studio page.

**Architecture:** Two-pronged fix: (1) Backend adds in-memory deduplication of concurrent session refreshes so only one WorkOS refresh call is made per expired token, (2) Frontend adds 401 response deduplication so multiple concurrent 401s trigger only one logout, plus fixes the error interceptor bug.

**Tech Stack:** TypeScript/Express (backend), React/Axios (frontend)

---

### Task 1: Backend — Deduplicate concurrent session refreshes

**Files:**
- Modify: `backend/src/utils/workos.util.ts`

**What:** Add an in-memory Map that caches in-flight refresh promises keyed by authToken. When multiple concurrent requests try to refresh the same expired token, they all await the same promise instead of each calling WorkOS independently.

**Implementation:**

```typescript
// Add above authenticateWorkOS
const pendingRefreshes = new Map<string, Promise<{ authenticated: boolean; sealedSession?: string }>>();

// Replace refreshWorkOSSession usage inside authenticateWorkOS:
// Instead of:
//   const refreshResult = await refreshWorkOSSession(authToken);
// Use:
//   const refreshResult = await deduplicatedRefresh(authToken);

const deduplicatedRefresh = async (authToken: string) => {
  const existing = pendingRefreshes.get(authToken);
  if (existing) return existing;

  const promise = refreshWorkOSSession(authToken).finally(() => {
    pendingRefreshes.delete(authToken);
  });

  pendingRefreshes.set(authToken, promise);
  return promise;
};
```

### Task 2: Frontend — Fix interceptor error handling and deduplicate 401 logout

**Files:**
- Modify: `frontend/src/hooks/useHttpInterceptors.ts`

**What:**
1. Fix bug on line 48: `return error.response` resolves the promise for non-401 errors (like 403). Should reject.
2. Add deduplication so multiple concurrent 401s only trigger one logout cycle.

**Implementation:**

```typescript
let isLoggingOut = false;

const generateResponseInterceptor = async ({ logout }: ResponseGeneratorProps) => {
  return axiosClient.interceptors.response.use(
    (response) => response,
    async (error) => {
      if (error?.response?.status === 401) {
        if (!isLoggingOut) {
          isLoggingOut = true;
          queryClient.clear();
          await logout();
          router.navigate(ROUTES.SIGNIN);
          isLoggingOut = false;
        }
        return Promise.reject(error);
      }

      return Promise.reject(error);
    },
  );
};
```

### Task 3: Frontend — Serialize batch submissions to avoid concurrent auth requests

**Files:**
- Modify: `frontend/src/components/pages/studio/hooks/useBatchSubmit.ts`

**What:** Change BATCH_SIZE from 5 to 1 to serialize dream creation requests during batch submit. This prevents concurrent requests that can race on session refresh. Once the backend dedup fix is deployed, this can be reverted back to 5.

---
