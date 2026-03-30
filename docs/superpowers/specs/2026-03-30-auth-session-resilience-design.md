# Auth Session Resilience Design

**Date:** 2026-03-30
**Status:** Draft
**Scope:** Backend fixes #1-3; notes on client follow-ups #4-5

## Problem

Users are forced to re-authenticate more often than necessary. The root cause is a set of related issues in how the backend handles WorkOS session refresh, socket authentication, and logout.

### Prior fixes (context)

- **Frontend 401 race** (`frontend@700ecd8`): Deduplicated concurrent 401 logout cycles with `isLoggingOut` flag
- **Error interceptor** (`frontend@700ecd8`): Non-401 errors now properly reject
- **Transient auth 503** (`backend@f00429e`): WorkOS/network blips return 503 instead of clearing cookie
- **Studio workaround** (`frontend@700ecd8`): `BATCH_SIZE=1` serializes requests to avoid refresh race

These fixed the studio sign-out problem. The issues below affect all pages and both the web frontend and native C++ client.

## Architecture Overview

All clients authenticate via WorkOS sealed sessions stored in an `httpOnly` cookie (`wos-session`):

```
Frontend (React)  ──cookie──→  Backend (Express)  ──→  WorkOS API
C++ Client        ──cookie header──→  Backend     ──→  WorkOS API
```

**HTTP auth flow** (`require-auth.middleware.ts`):
1. `requireAuth` reads `wos-session` from cookie (or `Bearer` from header)
2. `workOSAuth` calls `authenticateWorkOS()` which validates, then refreshes if expired
3. On refresh, new sealed session is set via `Set-Cookie` response header

**Socket auth flow** (`socket.middleware.ts`):
1. `socketCookieParserMiddleware` parses cookies from handshake headers
2. `socketAuthMiddleware` (deprecated Cognito) — passes through if no `query.token`
3. `socketWorkOSAuth` reads `socket.cookies["wos-session"]`, validates only (no refresh)

## Issues

### #1: Backend concurrent refresh race (HIGH)

**Location:** `backend/src/utils/workos.util.ts:53-97` — `authenticateWorkOS()`

**Cause:** When N concurrent requests arrive with an expired sealed session, each independently calls `refreshWorkOSSession()`. WorkOS uses rotating refresh tokens — the first refresh consumes the old token and returns a new sealed session. Requests 2-N attempt to refresh the same consumed token and fail, returning null → 401 → frontend logout.

**Impact:** Any page that fires parallel API calls on load (most pages). The `BATCH_SIZE=1` workaround only covers the studio page.

**Reproduction:** Open any page with 2+ concurrent API calls after the session has expired (WorkOS session TTL is typically short, ~1hr access token). The first request refreshes successfully; subsequent requests get 401.

### #2: Socket middleware rejects refreshable sessions (MEDIUM)

**Location:** `backend/src/middlewares/socket.middleware.ts:91-121` — `socketWorkOSAuth()`

**Cause:** `socketWorkOSAuth` calls `authenticateAndGetWorkOSSession()` which only validates — it does not attempt refresh. The HTTP middleware (`workOSAuth`) calls `authenticateWorkOS()` which validates then refreshes. When a client connects with an expired-but-refreshable session, the socket connection is rejected.

**Impact:** Both web frontend and C++ client. The web frontend works around this by catching the `connect_error`, calling `authenticateUser()` (which hits the HTTP `/authenticate` endpoint, refreshing the cookie), then creating a new socket. The C++ client has no such workaround — socket reconnections with stale sessions fail silently.

**Relationship to #1:** The socket refresh fix must use the same Redis dedup as #1 to avoid consuming the refresh token and breaking concurrent HTTP requests.

### #3: V2 logout requires auth — catch-22 on expired sessions (LOW)

**Location:** `backend/src/routes/v2/auth.routes.ts:218`

```typescript
authRouter.get("/logout", requireAuth, authController.logout);
```

**Cause:** `requireAuth` validates the WorkOS session before the logout handler runs. If the session is expired, the middleware returns 401 (and clears the cookie via `handleWorkOSAuthFailure`). The frontend's 401 interceptor then triggers a client-side logout, showing an error toast even though the session was actually cleaned up.

**Impact:** Confusing UX — user clicks logout with an expired session, sees "error signing out" toast, but is actually logged out. Functionally correct but misleading.

## Design

### Fix #1: Redis-based refresh deduplication

Add a Redis lock to `authenticateWorkOS()` so only one refresh runs per expired session. Concurrent requests wait for the result.

**Algorithm:**

```
authenticateWorkOS(authToken):
  session = authenticateAndGetWorkOSSession(authToken)
  if session: return { session }

  # Session expired, need refresh
  lockKey = "wos-refresh-lock:" + sha256(authToken)
  resultKey = "wos-refresh-result:" + sha256(authToken)

  # Try to acquire lock
  acquired = redis.SET(lockKey, "1", NX, EX 10)

  if acquired:
    try:
      refreshResult = refreshWorkOSSession(authToken)
      if refreshResult.authenticated && refreshResult.sealedSession:
        # Cache result for concurrent waiters
        redis.SET(resultKey, refreshResult.sealedSession, EX 30)
        session = authenticateAndGetWorkOSSession(refreshResult.sealedSession)
        return session ? { session, sealedSession: refreshResult.sealedSession } : null
      else:
        # Refresh failed — cache a failure sentinel so waiters don't also try
        redis.SET(resultKey, "__FAILED__", EX 5)
        return null
    finally:
      redis.DEL(lockKey)

  else:
    # Another request is refreshing — wait for result
    sealedSession = pollRedis(resultKey, timeout=5s, interval=100ms)
    if sealedSession && sealedSession != "__FAILED__":
      session = authenticateAndGetWorkOSSession(sealedSession)
      return session ? { session, sealedSession } : null
    else:
      return null
```

**Key details:**

- **Lock key**: `wos-refresh-lock:<sha256>` with 10s TTL (deadlock protection)
- **Result key**: `wos-refresh-result:<sha256>` with 30s TTL (enough for concurrent requests to pick up)
- **Failure sentinel**: `__FAILED__` with 5s TTL prevents waiters from retrying a failed refresh
- **Hash the sealed session**: The sealed session is large and sensitive; only the hash is used as a Redis key
- Uses existing Redis connection (already available for BullMQ)
- The `withTimeout` wrapper in `workOSAuth` already handles the case where polling takes too long (10s auth timeout)

**Files changed:**
- `backend/src/utils/workos.util.ts` — modify `authenticateWorkOS()`, add Redis lock/poll helpers

**After deployment:** Revert `BATCH_SIZE=1` in `frontend/src/components/pages/studio/hooks/useBatchSubmit.ts`.

### Fix #2: Socket middleware refresh support

Replace `authenticateAndGetWorkOSSession()` with `authenticateWorkOS()` in `socketWorkOSAuth`. This reuses the same function (with Redis dedup from #1) that the HTTP middleware uses.

```typescript
export const socketWorkOSAuth = async (
  socket: Socket,
  next: (err?: ExtendedError | undefined) => void,
) => {
  const authToken =
    socket.handshake.headers?.authorization?.split("Bearer ")[1] ||
    socket.cookies["wos-session"];

  try {
    const result = await authenticateWorkOS(authToken);

    if (!result) {
      return next(authError);
    }

    // Note: result.sealedSession (if refreshed) cannot be sent back to
    // the client via socket — no HTTP response. The client's cookie stays
    // stale until the next HTTP request refreshes it. This is acceptable.

    const user = await syncWorkOSUser(result.session.user);
    socket.data.user = user;

    if (user) {
      return next();
    } else {
      return next(authError);
    }
  } catch (e) {
    APP_LOGGER.error(e);
    return next(authError);
  }
};
```

**Also:** Remove `socketAuthMiddleware` (deprecated Cognito) from the middleware chain in `middleware.ts`. No client uses Cognito tokens for socket auth — the C++ client on master sends `wos-session` via cookie header. Removing dead code eliminates a potential error path (Cognito JWT validation throwing and rejecting the socket).

**Files changed:**
- `backend/src/middlewares/socket.middleware.ts` — update `socketWorkOSAuth`, remove `socketAuthMiddleware`
- `backend/src/middlewares/middleware.ts` — remove `socketAuthMiddleware` from chain

### Fix #3: Remove auth requirement from logout

Remove `requireAuth` from the V2 logout route. The handler already reads the session directly from the cookie/header and doesn't need the middleware's user context.

```typescript
// before
authRouter.get("/logout", requireAuth, authController.logout);

// after
authRouter.get("/logout", authController.logout);
```

The `logout` handler (`auth.controller.ts:991-1024`) already:
1. Reads the auth token from cookie/header directly
2. Calls `res.clearCookie("wos-session")` unconditionally (line 994, before any fallible operation)
3. Attempts to call the WorkOS logout URL (best-effort)
4. Returns 200 on success; catch block calls `handleWorkosError` on failure

Note: if the sealed session is expired, `loadSealedSession` / `getLogoutUrl` may throw. The cookie is already cleared before this point, so the client-side logout succeeds regardless. However, the response would be an error. To make this robust, the handler should catch errors from WorkOS logout URL calls and still return 200 — the cookie is already cleared, and the WorkOS session will expire naturally. This is a small improvement to make alongside removing `requireAuth`.

This matches V1's approach (`authRouter.post("/logout", authController.handleLogout)` — no `requireAuth`). The endpoint is idempotent and safe without auth — it only invalidates the caller's own session cookie.

**Files changed:**
- `backend/src/routes/v2/auth.routes.ts` — remove `requireAuth` from logout route

## Client-side follow-ups (not in scope)

### #4: C++ client leaks sealed session in URL query params

The C++ client passes the same map as both query params and HTTP headers:
```cpp
query["Cookie"] = string_format("wos-session=%s", sealedSession.c_str());
s_SIOClient.connect(url, query, query);  // (url, query_params, http_extra_headers)
```

The sealed session appears in the URL `?Cookie=wos-session%3D<sealed>`, which server access logs and intermediary proxies may capture.

**Fix:** Use separate maps for query params and headers. Only send `Cookie` as an HTTP header. Send `Edream-Client-Type` and `Edream-Client-Version` as both.

### #5: C++ client socket reconnection uses stale session

When `socket.io-client-cpp` auto-reconnects, it reuses the original query/headers from the initial `connect()` call. If `RefreshSealedSession()` updated the stored session via HTTP, the socket reconnection still uses the old one.

**Fix:** On successful `RefreshSealedSession()`, disconnect and reconnect the socket with the new session. Or disable auto-reconnect and manage reconnection manually (similar to the web frontend's approach in `socket.context.tsx`).

## Testing

- **#1**: Send 5+ concurrent authenticated requests after session expiry. All should succeed (at most one refresh call to WorkOS). Verify via Redis key inspection or WorkOS audit logs.
- **#2**: Let a session expire, then trigger a socket connection (tab focus, page load). Socket should connect without requiring an HTTP request first.
- **#3**: Let a session expire, then click logout. Should succeed with no error toast.
- **Regression**: Verify normal auth flow (login, page load, socket connect, logout) still works. Verify API key auth (Python SDK) is unaffected. Verify transient errors (503) still don't clear cookies.
