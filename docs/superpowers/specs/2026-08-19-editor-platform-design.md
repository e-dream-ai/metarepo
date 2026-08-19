# Editor Platform — Milestone A: standalone editors, frontend-only WorkOS, editor-config API

Issue: [backend#397](https://github.com/e-dream-ai/backend/issues/397) — "App platform: API keys → Studio separation → OAuth2"

## Context

Studio lives inside the main frontend today (`frontend/src/components/pages/studio/`, 84 files / ~11.5k lines
on `origin/stage`). Two problems follow from that: a bad Studio deploy ships with the whole app, and nobody
outside the team can build or customize an editor.

Issue #397's Phase 2 proposed splitting Studio out and sharing auth via a `wos-session` cookie scoped to
`.alpha.infinidream.ai`. This plan replaces that with a **frontend-only WorkOS** approach: editors are
standalone static SPAs that authenticate in the browser and call the API with a bearer JWT. No shared cookie,
no cookie-domain surgery, no coupling between the main app's session and an editor's.

Intended outcome: `flow` and `batch` ship as two independently deployed GitHub Pages apps, users' named
save-slots move from browser localStorage to the server so they survive a device change, and a third party can
fork an editor and run it against our API.

The cookie-domain change from the issue is **not needed** and is dropped. mystixxx's Cloudflare-Worker router
is also unnecessary — per-editor subdomains cover it (see Origins). Their scoped-API-proxy suggestion is the
right shape for Milestone B's broker and is deferred there, not dropped.

## Constraints discovered (verified, not assumed)

1. **WorkOS redirect URIs and CORS origins are dashboard-only.** There is no API to add them
   ([docs](https://workos.com/docs/sso/redirect-uris)). A forked editor on an arbitrary origin therefore can
   *never* talk to WorkOS directly, which is why self-serve registration cannot be built as one piece and is
   split into Milestone B.
2. **The backend's existing `Bearer` path cannot accept an AuthKit token.** `requireAuth`
   (`backend/src/middlewares/require-auth.middleware.ts:162`) routes `Bearer` to `authenticateWorkOS`, which
   calls `authenticateWithSessionCookie` (`backend/src/utils/workos.util.ts:46`) — that only understands a
   *sealed session* encrypted with `WORKOS_COOKIE_PASSWORD`. AuthKit hands the browser a **JWT** instead.
3. **AuthKit tokens are JWTs verifiable via JWKS** at `https://api.workos.com/sso/jwks/<clientId>`
   ([docs](https://workos.com/docs/authkit/sessions)), and carry `role` / `permissions` / `exp` claims.
   `jwks-rsa` is already a backend dependency (legacy Cognito path).
4. **Client-only AuthKit needs a custom auth domain** or it falls back to `devMode`, which keeps the refresh
   token in localStorage instead of an HTTP-only cookie ([docs](https://workos.com/docs/user-management/client-only)).
   It also cannot run in an iframe.
5. **`*.infinidream.ai` is already CORS-allowed** — `ALLOWED_DOMAIN_PATTERNS`
   (`backend/src/constants/api.constants.ts:53`) matches it, so per-editor subdomains need **zero** backend
   CORS change.
6. **Named save-slots already exist client-side.** `StudioSession { id, name, createdAt, updatedAt, mode,
   flowState, batchState, thumbnail? }` with `MAX_SESSIONS = 20` (`frontend/src/types/session.types.ts`),
   autosaved on a 2s debounce (`useSessionAutoSave.ts`), rename/switch/delete UI in `session-switcher.tsx`.
   Unchanged on `origin/stage`. The config API generalizes this shape rather than inventing one.
7. **Nothing else depends on the Bearer-sealed-session path.** The Python SDK sends `Api-Key`
   (`python-api/src/edream_sdk/client/api_client.py:27`); the desktop client sends no Bearer. It is kept
   working anyway as a fallback.

## Scope

**In (Milestone A):** canonical `flow` + `batch` editor apps, backend bearer-JWT auth, editor-config API,
localStorage→server migration, main-frontend `/studio` → link.

**Forks in Milestone A:** a fork detects it is not on a known origin and asks for a **personal API key**,
which `requireAuth` already accepts. No registration, no new attack surface, works today.

**Out (Milestone B, its own spec):** self-serve editor registration, an auth broker on our domain (one
registered callback, editor identity in `state`), user-visible consent, revocable per-editor grants, DB-driven
CORS. Milestone A keeps `editorId` first-class and the token-verification seam a clean branch point so B slots
in without rework.

**Decided:** auth migration covers editors only — the main frontend keeps its `wos-session` cookie untouched,
and two auth paths coexist. Editors are gated behind auth (login or API key); localStorage is repurposed as an
always-on write-through draft cache, not a logged-out editing mode.

## Architecture

```
flow.infinidream.ai  ─┐                            ┌─ verify JWT via WorkOS JWKS (no API call)
(GH Pages, authkit-js)│  Authorization: Bearer JWT  │
                      ├──→ api.infinidream.ai ──────┤─ /api/v2/editor/:editorId/configs  (new)
batch.infinidream.ai ─┘         (existing)          │
                                                    └─ /api/v1/dream, /playlist, ...  (unchanged)
infinidream.ai (main app) ──→ same API via wos-session cookie  (unchanged)
```

### 1. Editor apps

Two new repos, `e-dream-ai/studio-flow` and `e-dream-ai/studio-batch`: Vite + React, built to GitHub Pages
with a `CNAME` per env — `flow.infinidream.ai`, `flow.stage.infinidream.ai`, `flow.alpha.infinidream.ai`,
same for `batch`.

Own subdomain per editor rather than `e-dream-ai.github.io/studio-flow`, because: `*.infinidream.ai` is
already CORS-allowed (constraint 5); each editor gets its own origin, so one editor's localStorage and tokens
are unreadable by another app sharing `github.io`; and path-hosted SPAs need the GH Pages `404.html` routing
hack.

Copy the studio tree from a **pinned `origin/stage` commit** and record it in each repo's README — studio
churned +1235/−518 lines across 31 files in the last 11 stage commits, so "which commit was this forked from"
will be asked. Carry across: `components/pages/studio/**`, the `flow` / `studio` / `studio-mode` / `session`
stores, `types/`, `constants/`, `client/axios.client.ts` (7 lines), and its 4 `components/shared` deps —
`cost-estimate`, `credit-limit-notice`, `credits-meter`, `presigned-image`, verified to be the full closure
(they import no further shared components).

Per editor per env, one-time in the WorkOS dashboard: redirect URI, CORS allow-list entry, and a custom auth
domain so the refresh token is an HTTP-only cookie rather than `devMode` localStorage (constraint 4).

`axios.client.ts` changes from `withCredentials: true` to an interceptor attaching
`Authorization: Bearer <access_token>` from `authkit-js` (or `Api-Key <key>` in fork mode).

### 2. Backend: bearer-JWT auth

One new branch in `requireAuth`. Add a **pure** `classifyBearerToken(token)` returning `{ kind: "jwt" }` or
`{ kind: "sealed" }` from token structure (three dot-separated segments whose header decodes to JSON with
`alg: "RS256"` and a `kid`). JWT → new verifier; sealed → today's `authenticateWorkOS`, byte-for-byte
unchanged; verification failure on the JWT path falls back to the sealed path so no existing caller can break.

The JWT verifier avoids the cookie path's per-request WorkOS API calls entirely:

1. Verify signature against `https://api.workos.com/sso/jwks/${WORKOS_CLIENT_ID}` using `jwks-rsa`, with the
   key cache on. Check `exp`.
2. Look up the local user by `workOSId` = `sub` (`User.entity.ts:37`), with the same eager relations
   `syncWorkOSUser` uses (`backend/src/utils/user.util.ts:140-154`) — one DB query, zero WorkOS calls.
3. On miss (no local row yet, or a row predating the `workOSId` backfill): `workos.userManagement.getUser(sub)`
   → `syncWorkOSUser(workOSUser)`. Self-healing, and only on first contact.
4. Take `role` from the JWT claim instead of `listOrganizationMemberships`, which the cookie path calls on
   **every** request (`backend/src/utils/workos.util.ts:129-133`). Set the same `res.locals.user` /
   `userRole` / `workosUser` shape so every downstream controller and `checkRoleMiddleware` is untouched.

Nothing about the cookie, `workOSCookieConfig` (`workos.util.ts:25`), `Api-Key`
(`require-auth.middleware.ts:154`), or `handleCustomOrigin` (`backend/src/utils/api.util.ts:8`) changes.

### 3. Editor-config API

New entity `backend/src/entities/EditorConfig.entity.ts`, following the `Keyframe.entity.ts` convention
(numeric `id` PK + `@Generated("uuid")` public `uuid`, `@ManyToOne User`, `@CreateDateColumn` /
`@UpdateDateColumn` / `@DeleteDateColumn`):

| column      | type          | notes                                          |
| ----------- | ------------- | ---------------------------------------------- |
| `id`        | serial pk     | internal                                       |
| `uuid`      | uuid, indexed | public identifier                              |
| `user`      | fk → users    | indexed, cascade on user delete                |
| `editorId`  | varchar(64)   | `"flow"`, `"batch"`, or a fork slug            |
| `name`      | varchar(120)  |                                                |
| `config`    | jsonb         | the editor's own state blob, opaque to us      |
| `version`   | integer       | optimistic concurrency, starts at 1            |
| `thumbnail` | varchar(2048) | nullable; a URL set explicitly by the client   |

Unique `(user, editorId, name)`; index `(user, editorId)`. Generated migration via
`pnpm run migration:generate`.

Routes mounted alongside the existing v2 mounts (`backend/src/routes/v1/router.ts:539`) as
`/api/v2/editor`, with `requireAuth` + a zod schema in `src/schemas/`, controller in
`src/controllers/editor-config.controller.ts`, swagger JSDoc matching `keyframe.routes.ts`:

- `GET /api/v2/editor/:editorId/configs` — metadata only (`uuid`, `name`, `version`, timestamps, `thumbnail`),
  **no `config` bodies**, so the switcher UI stays cheap
- `POST /api/v2/editor/:editorId/configs` — `{ name, config, thumbnail? }`
- `GET|PUT|DELETE /api/v2/editor/:editorId/configs/:uuid`

Rules: `editorId` matches `^[a-z0-9-]{1,64}$` and needs no registration in Milestone A; `config` capped at
256KB (413 over). `thumbnail` is a separate column rather than something we read out of `config`, because the
backend never parses that blob — it mirrors `StudioSession.thumbnail`, which the client already derives from the
first keyframe's `imageUrl` (`frontend/src/stores/session.store.ts:87-105`); data URLs are rejected so list
responses stay small. 20 configs per `(user, editorId)`, mirroring `MAX_SESSIONS` (409 over, with the name of the
oldest so the client can offer to replace it); `PUT` sends the expected `version` and gets **409** on mismatch —
worth the small cost because a 2s autosave in two open tabs otherwise loses work silently. Configs are strictly
private to their owner; every read and write filters on `user`. Sharing is deliberately not designed yet.

### 4. Client: draft cache + sync

`session.store.ts` keeps writing to localStorage on the existing 2s debounce, then syncs: `saveCurrentSession`
(`session.store.ts:125`) also fires a debounced `PUT`, tracking `version` per slot. On 409 the store refetches
and surfaces a "this slot changed elsewhere — keep yours or theirs?" prompt rather than silently clobbering.
Server list is the source of truth on load; local-only slots are pushed up. One code path, authed or not; if
unauthed the sync step is a no-op and the shell shows a login / paste-your-key screen.

### 5. Migrating existing users' sessions — must land before the `/studio` route is removed

`flow.infinidream.ai` is a different origin from `infinidream.ai` and **cannot read** the existing
`studio-sessions` localStorage key. Without this step every user's current save-slots silently vanish on the
move.

While the main frontend still has both the local data and a working cookie session, a one-time migration reads
`SESSIONS_STORAGE_KEY` (`frontend/src/types/session.types.ts`), maps each `StudioSession` to
`POST /api/v2/editor/:mode/configs` (`mode` → `editorId`, `name` → `name`, the mode's state blob → `config`),
marks a `studio-sessions-migrated` flag, and leaves the local data in place as a safety net. Only after this
ships does `/studio` in `frontend/src/routes/router.tsx:35` become a link to the editor subdomain.

## Tasks (TDD, in order, one commit each, in a worktree)

1. `classifyBearerToken` — pure, unit tests first: JWT, sealed session, garbage, empty, JWT-shaped-but-`alg:none`.
2. JWT verifier + `requireAuth` branch. Integration tests over all four token shapes — JWT, sealed, `Api-Key`,
   none — since regressing the cookie path breaks the entire main app.
3. `EditorConfig` entity + migration.
4. Config validation as pure functions — slug, name, size cap, count cap — unit tests first.
5. The five endpoints + controller. Integration tests: ownership isolation (user A cannot read/write/delete
   user B's config by `uuid`), 409 on stale `version`, 413 over size, 409 over count, unique-name conflict.
6. `session.store.ts` sync layer, with the mapper as a tested pure function.
7. One-time migration in the main frontend, with the mapper unit-tested.
8. Scaffold `studio-flow` from the pinned commit; swap `axios.client.ts` to bearer; wire `authkit-js`; GH Pages
   workflow + `CNAME`.
9. Same for `studio-batch`.
10. Replace the main frontend `/studio` route with a link.

Steps 1–7 are backend + main frontend and ship first; the editor repos (8–9) can only be verified once WorkOS
dashboard entries exist, so that setup is a prerequisite, not a code task.

## Verification

- `cd backend && pnpm run test && pnpm run lint:fix && pnpm run build`; `cd frontend && pnpm run type-check && pnpm run lint`.
- **Cookie path unbroken** (the main risk): log into the local frontend against staging per
  `AGENTS.md` → Local Development, and confirm dreams, playlists, and studio all still load.
- **JWT path end-to-end:** grab a real AuthKit access token from a browser session, then
  `curl -H "Authorization: Bearer $JWT" localhost:8080/api/v2/editor/flow/configs` → 200 with `[]`, and
  `curl` the same with a tampered signature → 401.
- **Config CRUD:** create two named configs, list (assert no `config` bodies in the response), update one and
  confirm `version` increments, `PUT` with the stale version → 409, delete one, list again.
- **Cross-user isolation:** with two API keys, confirm user B gets 404 (not 403 — do not leak existence) on
  user A's config `uuid`.
- **Migration:** with real localStorage sessions present, run the one-time migration against local backend and
  confirm each slot appears via the API with its name and state intact, the flag is set, and a second load
  does not duplicate.
- **Editor app:** load `studio-flow` locally, log in through AuthKit, create a flow, confirm the slot appears
  in the API, hard-reload and confirm it rehydrates from the server; then run it with an API key pasted and
  confirm fork mode works.

## Open items

- WorkOS custom auth domain per env — needs dashboard access and DNS; confirm it exists before task 8.
- Whether `flow` and `batch` should ship as two repos or one repo with two build targets. Two is planned
  (independent deploys were the point); one repo is the cheaper reversal if maintaining two proves annoying.
- Sharing/publishing configs, and config schema versioning across editor versions, are both deliberately
  unspecified. The `config` blob is opaque to the backend, so the editor owns its own migrations.

**Next step:** `superpowers:writing-plans` turns this spec into the task-level implementation plan.
