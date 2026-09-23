# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

E-dream (infinidream.ai) is a generative AI platform for creating and managing animated dream content. It's a multi-repo architecture with specialized services.

## Architecture

```
Frontend (React) ──→ Backend (Node/Express) ──→ Worker (BullMQ)
                           │                         │
                           ↓                         ↓
                    Video Service ←──────── GPU Container (RunPod)
                    (RunPod/FFmpeg)           (ComfyUI/PyTorch)
                           │
                           ↓
                    Storage (R2)
```

**Data flow:** User creates dream via frontend → backend queues job → worker submits to RunPod → gpu-container-comfy runs ComfyUI workflow → result stored in R2 → video service processes (thumbnails/filmstrips) → frontend displays

## Repositories

| Repo                    | Stack                             | Purpose                                                |
| ----------------------- | --------------------------------- | ------------------------------------------------------ |
| `backend`               | TypeScript/Express/TypeORM/BullMQ | Main API, auth (WorkOS), Socket.IO, job orchestration  |
| `frontend`              | React/Vite/TypeScript/Zustand     | Web UI for dream creation, playback, playlists         |
| `video`                 | Python/FFmpeg (RunPod container)  | Video processing: thumbnails, filmstrips, transcoding  |
| `worker`                | TypeScript/Express/BullMQ         | GPU job coordinator, RunPod submission, Bull Dashboard |
| `gpu-container-comfy`   | Python/Docker/ComfyUI             | Serverless GPU container on RunPod                     |
| `python-api`            | Python                            | edream_sdk - Python client for backend API             |
| `engines`               | Python                            | Batch processing scripts (wan-i2v, uprez, qwen)        |
| `electric-sheep-engine` | Python                            | Legacy Electric Sheep playlist sync                    |
| `landing-page`          | Next.js/React/Tailwind/Biome      | Static website (infinidream.ai)                        |
| `client`                | C++                               | Native macOS desktop app/screensaver                   |

## Keeping Local Clones Fresh

**Fetch before you trust what you read.** Every repo here is a symlink to a
sibling checkout that goes stale silently — nothing warns you that the file you
just opened is months behind what is deployed. Reading a stale clone does not
produce an obvious error; it produces a confident, wrong answer about how the
running system behaves.

This is not hypothetical. `gpu-container-uprez` was once found 60 commits and 11
months behind `origin/main`. The local copy had no progress-reporting code at
all, so the honest conclusion from reading it — "this container never reports
progress" — was exactly backwards. The deployed image reports progress in four
mapped phases.

The `gpu-container-*` repos are the worst offenders: they are deployed from GHCR
to RunPod, so what runs in production is whatever image was last pushed, which
has no connection to what your working tree says. `video` and
`electric-sheep-engine` are edited rarely enough that a checkout can sit
untouched for a year.

For the drift-survey script, use the `check-repo-drift` skill.

Compare against `@{u}` (the branch's own upstream), not `origin/main`. Several
repos sit on `stage` or a feature branch, where a behind-count against `main` is
meaningless noise. `AHEAD > 0` means unpushed local commits — look before you
pull.

When an answer depends on what is actually deployed, `git fetch` and read
`origin/main` directly (`git show origin/main:path/to/file`, `git grep -n pat
origin/main`) rather than the working tree. That inspects the remote state
without touching a checkout that may hold someone's in-progress work.

## Commands by Repository

### worker

```bash
npm run dev               # Watch mode with nodemon
npm run build             # Compile TypeScript
node dist/prompt.js prompt/deforum-fish.json  # Submit job via CLI
```

Bull Dashboard: http://localhost:3000/admin (user: admin)

### gpu-container-comfy

```bash
docker build -t comfy:dev-base --target base --platform linux/amd64 .
docker-compose up         # Local dev (ComfyUI: 8188, API: 8000)
python -m unittest discover  # Run tests
```

### client (macOS)

```bash
brew install git-lfs && git lfs install
open client_generic/MacBuild/e-dream.xcodeproj
./client_generic/MacBuild/build.py
./client_generic/MacBuild/release.py
```

## Local Development (backend + frontend)

The `.env` files in `backend/` and `frontend/` are pre-configured to point at **staging** services (AWS RDS Postgres at `edream-postgres-db-staging...`, Upstash Redis). No local Postgres/Redis is needed — both dev servers connect directly to staging. Make sure both repos are on the `stage` branch so code matches the data.

### Migrations: pulling one does not mean running one

**Do not run `pnpm run migration:run` locally as a reflex after `git pull`.**

Because `backend/.env` points at the shared **staging** RDS (above), the database
you connect to is the same one Heroku deploys against. Backend deploys on push to
`stage`, so by the time a migration file reaches your working tree, that migration
has almost always already been applied to staging by the deploy. Pulling the file
gives you the *source* for a schema change the DB already has.

Running it anyway is not a no-op you can shrug off: it is a write against shared
staging that every other developer and the deployed stage frontend are also using.

Your `.env` will not do it for you either — `TYPEORM_MIGRATIONS_RUN=false` and
`TYPEORM_SYNCHRONIZE=false`, so the dev server never applies anything on startup.

Check instead of assuming (read-only, ~30s, connects to staging):

```bash
cd backend && pnpm run migration:show   # [X] = applied, [ ] = pending
```

Only run `migration:run` when that shows a genuinely pending `[ ]` — normally just
after *you* generated a migration that has not been deployed yet.

**Corollary for schema questions:** an entity may carry indexes TypeORM refuses to
manage, e.g. `@Index("IDX_USER_EMAIL_LOWER", { synchronize: false })` on
`User.entity.ts` — a functional index on `lower(email)` that TypeORM cannot express.
It exists *only* because a migration created it, and it is invisible to schema-sync
tooling. Read `src/migrations/` for the real schema, not just the entities.

### Start both servers

```bash
cd backend  && pnpm run dev   # tsx watch, serves on :8080
cd frontend && pnpm run dev   # vite, serves on :5173
```

To bring both up and fast-forward whichever is on `stage`/`main` (leaving
feature branches alone), use the `dev-up` skill.

Frontend `.env` sets `VITE_BACKEND_URL=http://localhost:8080/api`, so the local frontend talks to the local backend.

### Healthy startup signals (grep these in logs)

- **backend** — `Worker NNNN: Connected with postgres` then `e-dream.ai api 0.0.1 started on port 8080`. Note: no "listening"/"ready" string — match on `started on port`.
- **frontend** — `VITE vX.Y.Z  ready in NNN ms` and `Local:   http://localhost:5173/`.
- **Sanity check:** `curl -o /dev/null -w '%{http_code}\n' http://localhost:8080/api/v1` → `200`, same for `http://localhost:5173`.

### Repo layout gotcha

Inside `metarepo/`, each repo is a symlink to a sibling directory (e.g. `metarepo/backend → ../backend`). `node_modules` and any `pnpm install` run against the real path (`/Users/spot/e-dream-ai/backend/`), not the metarepo path. Error stack traces will show the real path — that's expected, not a misconfiguration.

### Common failure: `Cannot find module 'bullmq'` (or similar) on backend start

Means backend deps are stale / out of sync with the lockfile. Fix:

```bash
cd backend && pnpm install        # if prompted to wipe node_modules, accept
```

(pnpm sometimes asks to reinstall node_modules from scratch when the store/lockfile version drifted; safe to accept since contents come from the registry.)

## Real-time Progress & Preview

During rendering, progress streams via Socket.IO `/remote-control` namespace:

```
job:progress event → { status, progress (0-100), countdown_ms, preview_frame (base64 JPEG) }
```

| Component             | Location                                                            |
| --------------------- | ------------------------------------------------------------------- |
| Worker captures frame | `worker/src/services/status-handler.service.ts:storePreviewFrame()` |
| Redis storage         | `job:preview:{dreamUUID}` (3hr TTL)                                 |
| Backend endpoint      | `GET /v1/dream/{uuid}/preview`                                      |
| Frontend hook         | `frontend/src/api/dream/mutation/useGetDreamPreview.ts`             |
| Progress broadcaster  | `backend/src/services/job-progress.service.ts`                      |

Preview works for Deforum, Wan, Qwen, Uprez. Not yet implemented for AnimateDiff.

## Deployment

| Service             | Platform   | Trigger                       |
| ------------------- | ---------- | ----------------------------- |
| backend             | Heroku     | Push to `stage`/`main`        |
| frontend            | Cloudflare | Push to `stage`/`main`        |
| video               | RunPod     | Docker Hub via GitHub Actions |
| worker              | Heroku     | Push to `stage`/`main`        |
| landing-page        | Cloudflare | Static export                 |
| gpu-container-comfy | RunPod     | Docker Hub via GitHub Actions |

## Shared SDK (edream_sdk)

`edream_sdk` (python-api repo) is used by video, engines, and electric-sheep-engine for backend API communication. For install, algorithms, and the test script, use the `edream-sdk` skill.

## Frontend gotchas

Anti-rationalization tables for frontend store/upload/hook work live in `.claude/rules/frontend-gotchas.md` (auto-loaded for `frontend/**`; read it when working in `../frontend` via its real path).

## Package Managers

- **Node repos:** Use `pnpm` (not npm/yarn)
- **Python repos:** Use `pip` with virtualenv/pyenv

## Design Documents

- `docs/plans/2026-01-30-visual-creator-workflows-design.md` - Creator workflows, preview system, batch processing

## GPU Container CD Pipeline

Images build to GHCR on push to `main`; the RunPod endpoint must then be pointed at the new image. Use the `gpu-container-deploy` skill for the steps.
