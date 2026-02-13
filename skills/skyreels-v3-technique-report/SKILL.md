---
name: skyreels-v3-technique-report
description: >
  Generate videos from reference images, extend existing videos, and create talking avatars
  using SkyReels-V3's unified multimodal diffusion Transformer pipeline. Covers setup,
  prompt crafting, inference configuration, and multi-GPU optimization.
  Trigger phrases: "generate video from images", "extend this video", "create talking avatar",
  "SkyReels video generation", "image to video with identity preservation",
  "audio-driven avatar video"
---

# SkyReels-V3 Video Generation Pipeline

This skill enables Claude to help users set up and run SkyReels-V3, a state-of-the-art conditional video generation system built on diffusion Transformers with multimodal in-context learning. SkyReels-V3 unifies three paradigms in a single architecture: generating videos from 1-4 reference images while preserving subject identity, extending existing videos with seamless shot continuation or cinematic transitions, and producing minute-length talking avatar videos driven by audio input. Claude can guide users through installation, model selection, prompt engineering, inference scripting, memory optimization, and integration into larger production pipelines.

## When to Use

- When the user wants to generate a video from one or more reference images while keeping the subject's appearance consistent (face, clothing, style).
- When the user needs to extend an existing video clip — either continuing the same shot or switching to a new cinematic angle (cut-in, cut-out, reverse shot, multi-angle, cut-away).
- When the user wants to create a talking avatar video from a portrait image and an audio file (speech, narration).
- When the user asks how to set up SkyReels-V3 locally, optimize VRAM usage, or run multi-GPU inference.
- When the user needs to write a batch processing script that generates multiple videos with different prompts or reference inputs.
- When the user wants to build a web API or pipeline wrapper around SkyReels-V3 inference.

## Key Technique

SkyReels-V3 is built on a 14B/19B parameter diffusion Transformer (DiT) that treats all conditioning signals — reference images, prior video frames, audio features — as in-context tokens alongside the noisy video latents. Unlike adapter-based approaches that bolt reference encoders onto a frozen backbone, SkyReels-V3 trains a single unified model that jointly attends to multimodal context during the denoising process. This means the same architecture handles image-to-video, video extension, and audio-to-video without task-specific heads or fine-tuning.

The data pipeline is critical to quality. For reference-to-video training, SkyReels-V3 uses cross-frame pairing (selecting temporally diverse but semantically consistent frame pairs from video clips), image editing (extracting subjects and completing backgrounds to create non-trivial reference-target pairs), and semantic rewriting (regenerating captions to avoid copy-paste shortcuts). This three-stage augmentation forces the model to learn genuine identity preservation rather than pixel copying. During training, an image-video hybrid strategy mixes static image data with video data, and multi-resolution joint optimization trains across aspect ratios (1:1, 3:4, 4:3, 16:9, 9:16) so the model generalizes to any output format.

For video extension, the model uses a shot-switching detector trained on film data to classify transition types, combined with multi-segment positional encoding that maintains temporal coherence across concatenated clips. For talking avatars, a first-and-last frame insertion pattern anchors the generation: key frames are established first, then intermediate motion is filled in, ensuring lip sync and consistent appearance over minute-length outputs.

## Step-by-Step Workflow

### 1. Install SkyReels-V3 and verify dependencies

```bash
git clone https://github.com/SkyworkAI/SkyReels-V3
cd SkyReels-V3
pip install -r requirements.txt
# Requires Python 3.12+, CUDA 12.8+, PyTorch with CUDA support
python -c "import torch; print(torch.cuda.is_available())"
```

### 2. Select the correct model checkpoint for the task

| Task | Model | Size | HuggingFace ID |
|------|-------|------|-----------------|
| Reference images to video | SkyReels-V3-R2V-14B | 14B | `SkyworkAI/SkyReels-V3-R2V-14B` |
| Video extension | SkyReels-V3-V2V-14B | 14B | `SkyworkAI/SkyReels-V3-V2V-14B` |
| Talking avatar | SkyReels-V3-A2V-19B | 19B | `SkyworkAI/SkyReels-V3-A2V-19B` |

Models auto-download from HuggingFace on first run. To use a local path, pass `--model_id /path/to/model`.

### 3. Prepare reference inputs according to the task type

- **Reference-to-video**: Provide 1-4 high-quality images (PNG/JPG). Images should clearly show the subject with minimal occlusion. Diverse angles improve identity preservation.
- **Video extension**: Provide the source video (MP4). For shot-switching, decide the transition type.
- **Talking avatar**: Provide one front-facing portrait image and an audio file (MP3/WAV, max 200 seconds).

### 4. Write an effective text prompt

The prompt describes the desired video content. Structure it as: **subject description + action + scene/environment + cinematic style**. For shot-switching, prefix with the transition tag in brackets.

```
# Reference-to-video prompt
"A young woman in a red dress walks through a sunlit garden, petals falling around her. Cinematic lighting, shallow depth of field."

# Shot-switching extension prompt
"[REVERSE_SHOT] The interviewer nods and responds with a smile, office background with soft lighting."

# Talking avatar prompt
"A professional woman is giving a keynote speech. She speaks with confidence, subtle hand gestures. Conference stage, warm lighting."
```

### 5. Run inference with the appropriate command

**Reference-to-video (single GPU):**
```bash
python3 generate_video.py --task_type reference_to_video \
  --ref_imgs "subject_front.png,subject_side.png" \
  --prompt "A man in a leather jacket walks down a neon-lit city street at night" \
  --duration 5 --offload --seed 42
```

**Single-shot video extension:**
```bash
python3 generate_video.py --task_type single_shot_extension \
  --input_video scene.mp4 \
  --prompt "The camera continues to follow the subject as they enter the building" \
  --duration 5 --offload
```

**Shot-switching video extension:**
```bash
python3 generate_video.py --task_type shot_switching_extension \
  --input_video dialogue.mp4 \
  --prompt "[CUT_IN] Close-up of the character's face showing surprise" \
  --offload
```

**Talking avatar:**
```bash
python3 generate_video.py --task_type talking_avatar \
  --input_image portrait.jpg \
  --input_audio speech.wav \
  --prompt "A man is recording a podcast, speaking naturally with occasional smiles" \
  --seed 42 --offload
```

### 6. Configure resolution and aspect ratio

Output defaults to 720p at 24fps. Specify aspect ratio to match your target platform:

```bash
# Vertical for mobile/TikTok (9:16)
--resolution 720P --aspect_ratio 9:16

# Square for Instagram
--resolution 720P --aspect_ratio 1:1

# Widescreen for YouTube
--resolution 720P --aspect_ratio 16:9
```

### 7. Optimize for limited VRAM (under 24GB)

```bash
export PYTORCH_CUDA_ALLOC_CONF="expandable_segments:True"
python3 generate_video.py --low_vram --resolution 540P \
  --task_type reference_to_video \
  --ref_imgs "image.png" --prompt "..." --offload
```

This enables FP8 weight-only quantization and block-level offloading. Drop to 480P if OOM persists.

### 8. Scale to multi-GPU for faster inference

```bash
torchrun --nproc_per_node=4 generate_video.py \
  --task_type reference_to_video \
  --ref_imgs "img1.png,img2.png" \
  --prompt "..." --duration 5 --offload --use_usp
```

The `--use_usp` flag enables Unified Sequence Parallelism via xDiT for distributing the DiT across GPUs.

### 9. Build a batch processing script

```python
import subprocess
import json

jobs = [
    {"task": "reference_to_video", "refs": "hero_front.png", "prompt": "A hero runs across rooftops at sunset", "out": "hero_run.mp4"},
    {"task": "talking_avatar", "image": "host.jpg", "audio": "intro.wav", "prompt": "A podcast host introduces the show", "out": "intro.mp4"},
]

for job in jobs:
    cmd = ["python3", "generate_video.py", "--task_type", job["task"], "--offload"]
    if job["task"] == "reference_to_video":
        cmd += ["--ref_imgs", job["refs"], "--prompt", job["prompt"]]
    elif job["task"] == "talking_avatar":
        cmd += ["--input_image", job["image"], "--input_audio", job["audio"], "--prompt", job["prompt"]]
    subprocess.run(cmd, check=True)
    # Rename output to desired filename
    subprocess.run(["mv", "output.mp4", job["out"]])
```

### 10. Evaluate output quality

Check generated videos against these criteria:
- **Identity preservation**: Does the subject match reference images (face, clothing, accessories)?
- **Temporal coherence**: Are there flickering artifacts or sudden appearance changes between frames?
- **Motion naturalness**: Do movements look physically plausible?
- **Prompt adherence**: Does the video match the described scene, action, and style?
- **Audio sync** (talking avatar): Are lip movements aligned with the audio?

## Concrete Examples

**Example 1: E-commerce product video from product photos**

User: "I have 3 photos of a sneaker from different angles. Generate a 5-second video showing it rotating on a clean background."

Approach:
1. Use the `reference_to_video` task with 3 reference images.
2. Craft a prompt emphasizing rotation and clean backdrop.
3. Use 1:1 aspect ratio for product showcase.

```bash
python3 generate_video.py --task_type reference_to_video \
  --ref_imgs "sneaker_front.png,sneaker_side.png,sneaker_back.png" \
  --prompt "A white sneaker rotates slowly on a clean white surface, studio lighting, product showcase style" \
  --duration 5 --aspect_ratio 1:1 --offload --seed 123
```

Output: A 5-second 720p video at 24fps showing the sneaker rotating with consistent appearance across all frames.

---

**Example 2: Extending a short film clip with a reverse shot**

User: "I have a 10-second clip of an actor delivering a line. I want to cut to the listener's reaction."

Approach:
1. Use `shot_switching_extension` with the `[REVERSE_SHOT]` transition tag.
2. Describe the reaction shot in the prompt.

```bash
python3 generate_video.py --task_type shot_switching_extension \
  --input_video actor_line.mp4 \
  --prompt "[REVERSE_SHOT] The listener reacts with a slight nod and thoughtful expression, same room with warm interior lighting" \
  --offload
```

Output: The original clip seamlessly transitions to a reverse angle showing the listener, maintaining consistent lighting, color grading, and spatial layout.

---

**Example 3: Generating a 60-second talking avatar for a presentation**

User: "I have a headshot and a 45-second audio recording of a speech. Create a talking head video."

Approach:
1. Use `talking_avatar` task with the portrait and audio.
2. Write a prompt describing the speaker's demeanor and setting.
3. The model handles minute-length generation via key-frame inference.

```bash
python3 generate_video.py --task_type talking_avatar \
  --input_image headshot.jpg \
  --input_audio speech_45s.wav \
  --prompt "A professional man delivers a speech with confident posture, subtle gestures, neutral office background" \
  --seed 42 --offload
```

Output: A 45-second video with synchronized lip movements, natural head motion, and consistent facial identity throughout.

## Best Practices

**Do:**
- Provide multiple reference images from different angles for better identity preservation in reference-to-video generation.
- Write prompts that include subject description, action, environment, and cinematic style — the model responds well to structured, detailed prompts.
- Use `--seed` for reproducibility when iterating on prompt variations.
- Start with `--offload` enabled to avoid OOM errors; disable only if you have confirmed sufficient VRAM.
- For talking avatars, use clean front-facing portraits with neutral expressions for best results.

**Avoid:**
- Do not use low-resolution or heavily occluded reference images — the model needs clear subject features to preserve identity.
- Do not omit the transition tag (e.g., `[CUT_IN]`) when using shot-switching extension; without it the model defaults to single-shot continuation behavior.
- Do not exceed 200 seconds of audio for talking avatar — the model was trained with this upper bound.
- Do not expect real-time generation — a 5-second 720p clip takes significant compute even on high-end GPUs. Plan batch jobs accordingly.
- Do not mix unrelated subjects in reference images expecting the model to compose them — provide images of the same subject for identity preservation.

## Error Handling

| Problem | Likely Cause | Solution |
|---------|-------------|----------|
| `CUDA out of memory` | Model exceeds VRAM | Add `--low_vram --resolution 540P` and set `PYTORCH_CUDA_ALLOC_CONF="expandable_segments:True"` |
| Flickering or identity drift in output | Poor reference images or overly generic prompt | Use higher-quality references, add specific appearance details to the prompt |
| Lip sync mismatch in talking avatar | Audio too fast or noisy | Pre-process audio: normalize volume, remove background noise, ensure clear speech |
| Shot-switching produces a jarring cut | Missing or wrong transition tag | Verify the tag matches one of: `CUT_IN`, `CUT_OUT`, `REVERSE_SHOT`, `MULTI_ANGLE`, `CUT_AWAY` |
| Multi-GPU run crashes | xDiT not installed or incompatible | Reinstall xDiT: `pip install xdit` and ensure all GPUs are visible via `nvidia-smi` |
| Model download stalls | HuggingFace rate limit or network issue | Download manually with `huggingface-cli download SkyworkAI/SkyReels-V3-R2V-14B` and pass `--model_id /local/path` |

## Limitations

- **No text rendering in video**: SkyReels-V3 cannot generate readable text or overlays within the video frames. Add text in post-processing.
- **Single-subject focus**: The identity preservation pipeline is optimized for one primary subject. Multi-subject scenes with distinct identity requirements may produce blending artifacts.
- **Compute requirements**: The 14B and 19B parameter models require significant GPU resources. Minimum practical setup is a single GPU with 16GB+ VRAM (with `--low_vram`), but quality and speed improve substantially with 24GB+ or multi-GPU setups.
- **No interactive editing**: Generated videos cannot be iteratively refined region-by-region. To fix specific frames, regenerate with adjusted prompts or seeds.
- **Audio language sensitivity**: The talking avatar model was primarily trained on Mandarin and English speech. Other languages may produce degraded lip sync.
- **Maximum duration**: Individual generation calls are capped at short clips (typically 5-10 seconds for reference-to-video). Longer outputs require chaining via video extension.

## Reference

**Paper**: [SkyReels-V3 Technique Report](https://arxiv.org/abs/2601.17323v2) — Focus on Section 3 (data processing pipeline for cross-frame pairing and semantic rewriting), Section 4 (training strategy with image-video hybrid and multi-resolution optimization), and Section 5 (shot-switching detector and multi-segment positional encoding for video extension).

**Code**: [github.com/SkyworkAI/SkyReels-V3](https://github.com/SkyworkAI/SkyReels-V3) — Reference `generate_video.py` for all inference modes and CLI arguments.