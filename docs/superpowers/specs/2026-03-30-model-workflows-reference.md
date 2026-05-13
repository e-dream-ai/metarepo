# Model Workflows Reference

Reference doc for ComfyUI workflows, models, and LoRAs needed for the model swaps project. Companion to the [design spec](2026-03-30-model-swaps-vae-preview-design.md).

---

## Z Image Turbo

### Models (3 files, ~11GB total)

| File | Size | Location | Download |
|------|------|----------|----------|
| `z_image_turbo_bf16.safetensors` | ~6GB | `models/diffusion_models/` | [HuggingFace](https://huggingface.co/Comfy-Org/z_image_turbo/resolve/main/split_files/diffusion_models/z_image_turbo_bf16.safetensors) |
| `qwen_3_4b.safetensors` | ~4GB | `models/text_encoders/` | [HuggingFace](https://huggingface.co/Comfy-Org/z_image_turbo/resolve/main/split_files/text_encoders/qwen_3_4b.safetensors) |
| `ae.safetensors` | ~200MB | `models/vae/` | [HuggingFace](https://huggingface.co/Comfy-Org/z_image_turbo/resolve/main/split_files/vae/ae.safetensors) |

### Workflow

[Official template](https://raw.githubusercontent.com/Comfy-Org/workflow_templates/refs/heads/main/templates/image_z_image_turbo.json)

**Node pipeline:**
```
UNETLoader (z_image_turbo_bf16.safetensors)
    → ModelSamplingAuraFlow (shift=3)
        → KSampler (euler, res_multistep, 8 steps, cfg=1)
CLIPLoader (qwen_3_4b.safetensors, lumina2)
    → CLIPTextEncode (prompt)
        → positive conditioning
    → ConditioningZeroOut
        → negative conditioning
VAELoader (ae.safetensors)
EmptySD3LatentImage (width, height)
KSampler → VAEDecode → SaveImage
```

**Key settings:**
- Sampler: `res_multistep`
- Scheduler: `simple`
- Steps: 8
- CFG: 1
- Model sampling: AuraFlow with shift=3
- Supported sizes: 512x512, 768x768, 1024x1024, 1280x1280, 1024x768, 768x1024, 1280x720, 720x1280

**Note:** We use RunPod's public endpoint (`/v2/z-image-turbo/runsync`) instead of running ComfyUI ourselves. This workflow is documented for reference only — the public endpoint handles everything.

---

## LTX 2.3 (updated 2026-04-01 from Jef's workflow)

### Models (from Jef's tested workflow)

**Core:**

| File | Size | Location | Source |
|------|------|----------|--------|
| `ltx-2.3-22b-distilled_transformer_only_fp8_scaled.safetensors` | ~11GB | `models/diffusion_models/` | [Kijai/LTX2.3_comfy](https://huggingface.co/Kijai/LTX2.3_comfy) |
| `gemma_3_12B_it_fpmixed.safetensors` | ~6GB | `models/text_encoders/` | [Comfy-Org/ltx-2](https://huggingface.co/Comfy-Org/ltx-2) |
| `ltx-2.3_text_projection_bf16.safetensors` | ~2.3GB | `models/text_encoders/` | [Kijai/LTX2.3_comfy](https://huggingface.co/Kijai/LTX2.3_comfy) |
| `LTX23_video_vae_bf16.safetensors` | ~1.5GB | `models/vae/` | [Kijai/LTX2.3_comfy](https://huggingface.co/Kijai/LTX2.3_comfy) |
| `LTX23_audio_vae_bf16.safetensors` | ~365MB | `models/vae/` | [Kijai/LTX2.3_comfy](https://huggingface.co/Kijai/LTX2.3_comfy) |

**Upscaler + Preview:**

| File | Purpose | Source |
|------|---------|--------|
| `ltx-2.3-spatial-upscaler-x2-1.0.safetensors` | 2x latent upscaler (Jef uses) | [Lightricks/LTX-2.3](https://huggingface.co/Lightricks/LTX-2.3) |
| `ltx-2.3-spatial-upscaler-x2-1.1.safetensors` | 2x latent upscaler (hotfix for long videos) | [Lightricks/LTX-2.3](https://huggingface.co/Lightricks/LTX-2.3) |
| `taeltx2_3.safetensors` / `taeltx2_3_wide.safetensors` | TAESD fast preview | [madebyollin/taehv](https://github.com/madebyollin/taehv) |

**VRAM requirement:** 32GB+ CUDA-compatible GPU. ~60GB disk for models + cache.

### Jef's I2V Workflow (two-pass with audio)

**Original workflow:** `gpu-container-ltx/workflows/jef-ltx-2.3-i2v.json`

**Custom nodes required:**
- ComfyUI-LTXVideo (LTX-specific nodes)
- ComfyUI-VideoHelperSuite (VHS_VideoCombine)
- ComfyUI-KJNodes (VAELoaderKJ, SimpleCalculatorKJ, ImageResizeKJv2, SetNode/GetNode)
- rgthree-comfy (Power Lora Loader)

**Two-pass pipeline:**

Pass 1 — Low-res (704x512, 121 frames):
- Sampler: `lcm`, 8 steps
- Scheduler: `LTXVScheduler` (max_shift=2.05, min_shift=0.95, reverse=true, base_shift=0.1)
- CFG: 1.0

Pass 2 — Upscaled refinement:
- Sampler: `lcm`, 3 steps
- Scheduler: `ManualSigmas` (0.909375, 0.725, 0.421875, 0.0)
- CFG: 1.0

**Key settings:**
- Text encoder: DualCLIPLoader (Gemma 3 fpmixed + LTX text projection)
- Model: UNETLoader (distilled transformer-only FP8) — NOT CheckpointLoaderSimple
- Output: 24fps, H.264 MP4, CRF 19, with audio
- Default camera LoRA: static @ 0.4 strength via Power Lora Loader
- VAE decode: tiled (tile=512, overlap=64)
- T2V/I2V toggle: `LTXVImgToVideoInplace` bypass flag

**Note:** The distilled LoRA (`ltx-2-19b-distilled-lora`) is BYPASSED in Jef's workflow — it's only needed with the dev model, not the distilled transformer.

### Camera Control LoRAs

Official Lightricks camera LoRAs for LTX-2 (19b model). Partially compatible with LTX 2.3 (22b).

| Movement | Filename | Jef's strength |
|----------|----------|----------------|
| Static | `ltx-2-19b-lora-camera-control-static.safetensors` | **0.4** (active in workflow) |
| Dolly In | `ltx-2-19b-lora-camera-control-dolly-in.safetensors` | TBD |
| Dolly Out | `ltx-2-19b-lora-camera-control-dolly-out.safetensors` | TBD |
| Dolly Left | `ltx-2-19b-lora-camera-control-dolly-left.safetensors` | TBD |
| Dolly Right | `ltx-2-19b-lora-camera-control-dolly-right.safetensors` | TBD |
| Jib Up | `ltx-2-19b-lora-camera-control-jib-up.safetensors` | TBD |
| Jib Down | `ltx-2-19b-lora-camera-control-jib-down.safetensors` | TBD |

**Usage notes:**
- Avoid mixing multiple camera control LoRAs
- Describe the "destination" of the movement in the prompt for best results
- Even static LoRA at low strength activates LTX and produces movement

### IC-LoRA Control Modes (Advanced)

| Mode | Purpose | LoRA File |
|------|---------|-----------|
| Canny | Edge preservation | `ltx-2.3-22b-ic-lora-union-control-ref0.5.safetensors` |
| Depth | Camera + spatial geometry | Same union LoRA, depth mode |
| Pose | Human motion transfer | `ltx-2-19b-ic-lora-pose-control.safetensors` |
| Motion Track | V2V motion transfer | `ltx-2.3-22b-ic-lora-motion-track-control-ref0.5.safetensors` |
| Detailer | Video refinement | `ltx-2-19b-ic-lora-detailer.safetensors` |

---

## Nvidia RTX Video Super Resolution

### Requirements

- **ComfyUI node:** [`Comfy-Org/Nvidia_RTX_Nodes_ComfyUI`](https://github.com/Comfy-Org/Nvidia_RTX_Nodes_ComfyUI)
- **Install:** ComfyUI Manager → search "RTX"
- **No model files needed** — uses Nvidia VFX SDK (installed as pip dependency)
- **GPU:** Any GPU with Nvidia VFX SDK support (A100, H100, L40S, A40, RTX 4090, etc.)

### Image Upscale Workflow

```
LoadImage → RTXVideoSuperResolution → SaveImage
```

**RTXVideoSuperResolution node settings:**
- `mode`: `"scale by multiplier"`
- `scale_multiplier`: `2` (2x upscale)
- `quality`: `"ULTRA"` (options: LOW, MEDIUM, HIGH, ULTRA)

### Video Upscale Workflow

```
LoadVideo → GetVideoComponents (frames, audio, fps)
    → RTXVideoSuperResolution (2x, ULTRA)
        → CreateVideo (frames + audio + fps)
            → SaveVideo
```

**Key:** The video workflow preserves audio sync and frame rate through the upscale.

### Quality Settings

| Setting | Speed | Quality |
|---------|-------|---------|
| LOW | Fastest | Basic |
| MEDIUM | Fast | Good |
| HIGH | Moderate | Very good |
| ULTRA | Slowest | Best |

---

## Waiting on Jef (updated 2026-04-01)

| Item | Status | Impact |
|------|--------|--------|
| ~~LTX 2.3 workflow~~ | **Received 2026-04-01** | Integrated into gpu-container-ltx |
| ~~LTX model list~~ | **Resolved** | Distilled FP8 + Gemma fpmixed + text projection + video/audio VAEs |
| ~~LTX sampler settings~~ | **Resolved** | LCM, 8+3 steps, LTXVScheduler |
| LTX LoRA strengths for other cameras | Asked Jef | Static=0.4 confirmed, others TBD |
| LTX max duration tested | Asked Jef | Default 2s, frontend offers up to 20s |
| LTX audio gotchas with long durations | Asked Jef | Audio generation is new |
| Spatial upscaler v1.0 vs v1.1 | Asked Jef | Shipping both, v1.1 is hotfix for long videos |
| Nvidia VSR workflow/quality | Asked Jef | Still waiting |

---

## Sources

- [Z Image Turbo official workflow](https://docs.comfy.org/tutorials/image/z-image/z-image-turbo)
- [LTX 2.3 official workflows](https://docs.comfy.org/tutorials/video/ltx/ltx-2-3)
- [ComfyUI-LTXVideo custom nodes](https://github.com/Lightricks/ComfyUI-LTXVideo)
- [LTX-2.3 models on HuggingFace](https://huggingface.co/Lightricks/LTX-2.3)
- [LTX LoRA documentation](https://docs.ltx.video/open-source-model/usage-guides/lo-ra)
- [LTX 2.3 camera LoRA compatibility discussion](https://huggingface.co/Lightricks/LTX-2.3/discussions/15)
- [Nvidia RTX Nodes ComfyUI](https://github.com/Comfy-Org/Nvidia_RTX_Nodes_ComfyUI)
- [RunPod Z Image Turbo endpoint docs](https://docs.runpod.io/public-endpoints/models/z-image-turbo)
