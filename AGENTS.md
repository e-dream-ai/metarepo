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

## Commands by Repository

### backend

```bash
pnpm run dev              # Development with hot reload
pnpm run build            # Compile TypeScript
pnpm run test             # All tests
pnpm run test:unit        # Unit tests only
pnpm run lint:fix         # Auto-fix lint
pnpm run migration:run    # Run DB migrations
pnpm run migration:generate <name>  # Generate migration
```

### frontend

```bash
pnpm run dev              # Vite dev server (localhost:5173)
pnpm run build            # Production build
pnpm run lint             # ESLint check
pnpm run type-check       # TypeScript validation
```

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

### engines

```bash
pip install -r requirements.txt
python3 scripts/run_wan_i2v_batch.py     # Image-to-video batch
python3 scripts/run_uprez_batch.py       # Video upscaling batch
python3 scripts/run_qwen_image_batch.py  # Image generation batch
```

### landing-page

```bash
pnpm run dev              # Next.js dev server with Turbopack
pnpm run build            # Static export to out/
pnpm run biome:check      # Lint + format (Biome)
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

### Start both servers

```bash
cd backend  && pnpm run dev   # tsx watch, serves on :8080
cd frontend && pnpm run dev   # vite, serves on :5173
```

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

## Key Integrations

- **Auth:** WorkOS
- **Database:** PostgreSQL with TypeORM migrations
- **Job Queue:** Redis + BullMQ (Node) / RQ (Python)
- **GPU:** RunPod serverless with model-specific endpoints (Deforum, AnimateDiff, Wan, Uprez, Qwen)
- **Storage:** Cloudflare R2 (public CDN)
- **Real-time:** Socket.IO with Redis adapter
- **Monitoring:** Bugsnag error tracking

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

## RunPod Endpoints

Worker submits to different RunPod endpoints based on job type:

- `RUNPOD_DEFORUM_ENDPOINT_ID` - Deforum animation
- `RUNPOD_ANIMATEDIFF_ENDPOINT_ID` - AnimateDiff video
- `RUNPOD_UPREZ_ENDPOINT_ID` - Video upscaling
- `RUNPOD_HUNYUAN_ENDPOINT_ID` - Wan T2V/I2V models

## Deployment

| Service             | Platform   | Trigger                       |
| ------------------- | ---------- | ----------------------------- |
| backend             | Heroku     | Push to `stage`/`main`        |
| frontend            | Cloudflare | Push to `stage`/`main`        |
| video               | RunPod     | Docker Hub via GitHub Actions |
| worker              | Heroku     | Manual                        |
| landing-page        | Cloudflare | Static export                 |
| gpu-container-comfy | RunPod     | Docker Hub via GitHub Actions |

## Shared SDK (edream_sdk)

**Quickstart:** https://docs.google.com/document/d/1sXfGgogyrDyaOOxCyG6uvkG1l6uTUE2iNdkqVAa-N0Q

`edream_sdk` (python-api repo) is used by video, engines, and electric-sheep-engine for backend API communication.

```bash
# Install
pip install git+ssh://git@github.com/e-dream-ai/python-api.git

# Or clone and install locally
git clone https://github.com/e-dream-ai/python-api.git
cd python-api && pip install -r requirements.txt
```

### Basic Usage

```python
from edream_sdk.client import create_edream_client
import json

client = create_edream_client("https://api.infinidream.ai/api/v1", api_key)

# Create a dream
dream = client.create_dream_from_prompt({
    "name": "My Dream",
    "prompt": json.dumps({
        "infinidream_algorithm": "wan-i2v",
        "prompt": "slow zoom into crystal cave",
        "image": "uuid-or-url",
        "duration": 5
    })
})

# Poll status
dream = client.get_dream(dream["uuid"])  # status: queue → processing → processed

# Playlist management
playlist = client.create_playlist({"name": "Batch"})
client.add_item_to_playlist(playlist["uuid"], type="dream", item_uuid=dream["uuid"])
```

### Supported Algorithms

| Algorithm    | `infinidream_algorithm` | Key Params                                               |
| ------------ | ----------------------- | -------------------------------------------------------- |
| Qwen Image   | `qwen-image`            | `prompt`, `size`, `seed`                                 |
| Wan T2V      | `wan-t2v`               | `prompt`, `duration`, `size`                             |
| Wan I2V      | `wan-i2v`               | `prompt`, `image`, `duration`                            |
| Wan I2V LoRA | `wan-i2v-lora`          | `prompt`, `image`, `high_noise_loras`, `low_noise_loras` |
| Deforum      | `deforum`               | `0` (prompt), `max_frames`, `width`, `height`            |
| AnimateDiff  | `animatediff`           | `prompts`, `frame_count`, `steps`                        |
| Uprez        | `uprez`                 | `video_uuid`, `upscale_factor`, `interpolation_factor`   |

### Test Script

```bash
cd python-api
cp .env.example .env  # Add API_KEY from infinidream.ai/my-profile
python tests/gen.py --algo deforum
python tests/gen.py --algo qwen-image
python tests/gen.py --algo wan-i2v
```

## Package Managers

- **Node repos:** Use `pnpm` (not npm/yarn)
- **Python repos:** Use `pip` with virtualenv/pyenv

## Design Documents

- `docs/plans/2026-01-30-visual-creator-workflows-design.md` - Creator workflows, preview system, batch processing

## GPU Container CD Pipeline

Applies to all `gpu-container-*` repos (e.g. `gpu-container-deforum`, `gpu-container-ltx`).

### How it works

1. **GitHub Action** (`.github/workflows/build-and-push.yml`) triggers on push to `main` or manual dispatch.
2. It builds the Docker image for `linux/amd64`, tags it with `<timestamp>-<short-sha>` and `latest`, and pushes both to **GHCR** (`ghcr.io/e-dream-ai/<repo>:<version>`).
3. The image is now available at `ghcr.io/e-dream-ai/<repo>:latest` (and the versioned tag).

### Updating the RunPod endpoint (currently manual)

After a new image is pushed to GHCR, the RunPod serverless endpoint must be told to use it:

1. Go to [RunPod Console → Serverless](https://www.runpod.io/console/serverless).
2. Open the endpoint for this container.
3. Click **New Release**.
4. Paste the new GHCR image URL (e.g. `ghcr.io/e-dream-ai/gpu-container-ltx:20260530123456-abc1234`).
5. Save — RunPod pulls the image and makes it live for new jobs.

### How to automate the RunPod update

RunPod exposes a GraphQL API at `https://api.runpod.io/graphql`. Our endpoints are deployed directly from GHCR (no separate template), so the image is updated via `saveEndpoint` using the endpoint ID.

**Required secrets/variables:**

- `RUNPOD_API_KEY` — repo secret (or shared org-level secret). Get it from RunPod Console → Settings → API Keys.
- `RUNPOD_<NAME>_ENDPOINT_ID` — repo variable (Settings → Variables). The endpoint ID is visible in the URL when you open an endpoint in the RunPod console.

```yaml
- name: Update RunPod endpoint image
  env:
      RUNPOD_API_KEY: ${{ secrets.RUNPOD_API_KEY }}
      RUNPOD_ENDPOINT_ID: ${{ vars.RUNPOD_LTX_ENDPOINT_ID }}
      IMAGE_TAG: ${{ steps.meta.outputs.image_tag }}
  run: |
      curl -s -X POST "https://api.runpod.io/graphql?api_key=${RUNPOD_API_KEY}" \
        -H "Content-Type: application/json" \
        -d "{\"query\": \"mutation { saveEndpoint(input: { id: \\\"${RUNPOD_ENDPOINT_ID}\\\", imageName: \\\"${IMAGE_TAG}\\\" }) { id imageName } }\"}"
```

Each GPU container repo needs its own endpoint ID variable. `RUNPOD_API_KEY` can be a shared org-level secret.
