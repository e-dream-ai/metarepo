---
name: gpu-container-deploy
description: How gpu-container-* images get built to GHCR and rolled out to RunPod serverless endpoints (manual New Release, or automated via the RunPod GraphQL saveEndpoint mutation), plus which RunPod endpoint env vars the worker uses.
---

# GPU Container CD Pipeline

Applies to all `gpu-container-*` repos (e.g. `gpu-container-deforum`, `gpu-container-ltx`).

Each repo's `.github/workflows/build-and-push.yml` builds on push to `main` (or manual dispatch) and pushes `ghcr.io/e-dream-ai/<repo>:<timestamp>-<short-sha>` plus `:latest`.

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

## RunPod Endpoints

Worker submits to different RunPod endpoints based on job type:

- `RUNPOD_DEFORUM_ENDPOINT_ID` - Deforum animation
- `RUNPOD_ANIMATEDIFF_ENDPOINT_ID` - AnimateDiff video
- `RUNPOD_UPREZ_ENDPOINT_ID` - Video upscaling
- `RUNPOD_HUNYUAN_ENDPOINT_ID` - Wan T2V/I2V models
