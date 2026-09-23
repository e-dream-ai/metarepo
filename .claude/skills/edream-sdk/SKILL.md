---
name: edream-sdk
description: edream_sdk (python-api repo) reference - install, supported infinidream_algorithm values and their key params, and the gen.py test script. Use when writing Python that talks to the e-dream backend API or submitting dreams by algorithm.
---

# Shared SDK (edream_sdk)

**Quickstart:** https://docs.google.com/document/d/1sXfGgogyrDyaOOxCyG6uvkG1l6uTUE2iNdkqVAa-N0Q

`edream_sdk` (python-api repo) is used by video, engines, and electric-sheep-engine for backend API communication.

```bash
# Install
pip install git+ssh://git@github.com/e-dream-ai/python-api.git

# Or clone and install locally
git clone https://github.com/e-dream-ai/python-api.git
cd python-api && pip install -r requirements.txt
```

See `python-api/src/edream_sdk/client/edream_client.py` for the client API (`create_edream_client`, `create_dream_from_prompt`, `get_dream`, playlist methods).

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
