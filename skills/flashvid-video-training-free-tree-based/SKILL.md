---
name: "flashvid-video-training-free-tree-based"
description: "Accelerate Video Large Language Models (VLLMs) by compressing visual tokens using FlashVID's training-free spatiotemporal token merging. Applies Attention and Diversity-based Token Selection (ADTS) followed by Tree-based Spatiotemporal Token Merging (TSTM) to eliminate redundant video tokens without retraining. Trigger phrases: 'speed up video LLM inference', 'reduce video tokens for VLLMs', 'FlashVID token merging', 'compress video tokens without training', 'plug-and-play VLLM acceleration', 'extend video frame budget'"
---

FlashVID is a training-free inference acceleration framework for Video Large Language Models (VLLMs) that compresses visual tokens by jointly exploiting spatial and temporal redundancy. Unlike methods that compress space and time independently, FlashVID first selects the most representative tokens per frame via attention-weighted diversity optimization (ADTS), then merges remaining redundant tokens across frames using tree-structured spatiotemporal matching (TSTM). This enables retaining only 10% of visual tokens while preserving 99%+ of original model performance on LLaVA-OneVision, LLaVA-Video, Qwen2.5-VL, and Qwen3-VL -- with zero retraining.

## When to Use

- When the user wants to speed up inference of a Video LLM (LLaVA-OneVision, LLaVA-Video, Qwen2.5-VL, Qwen3-VL) on long or multi-frame video inputs
- When the user needs to process more video frames within a fixed GPU memory or compute budget (e.g., going from 32 to 320 frames)
- When the user asks to reduce visual token count in a VLLM pipeline without finetuning
- When the user is building a video understanding application and hits OOM or latency limits due to high token counts
- When the user wants a plug-and-play acceleration module that can wrap an existing VLLM model
- When the user asks about spatiotemporal token compression, token pruning, or token merging for video models

## Key Technique

**The core insight** is that spatial and temporal redundancy in video are not independent. A moving object occupies different spatial positions across frames, so compressing each frame spatially in isolation misses cross-frame redundancy, and compressing temporally without spatial awareness duplicates similar features. FlashVID addresses both jointly through a two-stage pipeline.

**Stage 1 -- ADTS (Attention and Diversity-based Token Selection):** For each frame, ADTS scores every visual token by combining three signals: (a) attention importance -- how much attention each token receives from other tokens in the vision encoder, (b) event relevance -- cosine similarity between each token and the frame-level average embedding, and (c) spatial diversity -- solving a Calibrated Max-Min Diversity Problem (MMDP) to ensure the selected token subset maximally covers the feature space. The parameter `alpha` (default 0.7) controls what fraction of the final retained tokens come from ADTS vs. TSTM. Tokens selected by ADTS form the base representation.

**Stage 2 -- TSTM (Tree-based Spatiotemporal Token Merging):** For tokens not selected by ADTS, TSTM builds tree structures across frames. It computes cosine similarity between every non-selected token in frame f and every non-selected token in frame f+1. If a token's maximum cross-frame similarity exceeds `temporal_threshold` (default 0.8), they are linked into a parent-child tree node. This produces forest-like structures spanning multiple frames where each tree captures a single visual concept moving through time. Each tree is then aggregated (mean-pooled) into one merged token. The merged tokens are concatenated with ADTS-selected tokens, producing the final compressed token set that enters the LLM.

## Step-by-Step Workflow

1. **Install FlashVID** from the official repository using `uv sync` or pip, cloning from `https://github.com/Fanziyang-v/FlashVID`.

2. **Wrap the target VLLM** with the `flashvid()` function, which monkey-patches the model's forward pass to insert token compression between the vision encoder output and the LLM input:
   ```python
   from flashvid import flashvid
   model = flashvid(
       model,
       retention_ratio=0.1,   # keep 10% of visual tokens
       alpha=0.7,             # 70% from ADTS, 30% from TSTM
       temporal_threshold=0.8  # similarity cutoff for tree merging
   )
   ```

3. **Choose the retention ratio** based on the quality-speed tradeoff. At `retention_ratio=0.1`, expect ~99% accuracy retention with ~10x token reduction. Use 0.2-0.3 for minimal quality loss; use 0.05-0.1 for maximum speedup on long videos.

4. **Set alpha** to control the ADTS/TSTM balance. Higher alpha (e.g., 0.8) allocates more tokens to per-frame diversity selection, which is better for videos with diverse spatial content. Lower alpha (e.g., 0.5) allocates more tokens to cross-frame merging, which is better for videos with heavy temporal redundancy (e.g., static camera, slow motion).

5. **Set temporal_threshold** to control how aggressively tokens are merged across frames. Lower thresholds (e.g., 0.6) merge more aggressively -- good for slow-changing scenes. Higher thresholds (e.g., 0.9) merge only near-identical tokens -- safer for fast-motion content.

6. **Run inference normally** -- the wrapped model accepts the same inputs (video path, question/prompt, num_frames) as the original. The compression is transparent:
   ```python
   output = model.generate(video_tokens, text_prompt, max_new_tokens=512)
   ```

7. **For extending frame budgets**, increase `num_frames` proportionally to the compression ratio. With `retention_ratio=0.1`, you can input ~10x more frames within the same memory budget. For example, go from 32 frames to 320 frames on Qwen2.5-VL.

8. **Evaluate** on standard benchmarks (VideoMME, MVBench, MLVU, LongVideoBench, Video-MME) using the provided evaluation scripts in `scripts/` to verify that quality is preserved at your chosen retention ratio.

9. **Integrate into production pipelines** by wrapping the model once at initialization time. No changes needed to data loading, tokenization, or generation logic. The wrapper intercepts the visual token tensor between encoder and LLM.

10. **Profile memory and latency** before and after wrapping. Measure prefill time (which decreases proportionally to token reduction) and peak GPU memory. Use `torch.cuda.max_memory_allocated()` to quantify savings.

## Concrete Examples

**Example 1: Accelerating LLaVA-OneVision on a long video**

User: "I'm running LLaVA-OneVision on 5-minute videos with 64 frames and inference is too slow. How can I speed it up without retraining?"

Approach:
1. Install FlashVID and import the wrapper
2. Wrap the LLaVA-OneVision model with `flashvid(model, retention_ratio=0.1)`
3. Run inference as normal -- the same 64 frames now produce ~90% fewer tokens entering the LLM

```python
from flashvid import flashvid
from llava.model import LlavaOnevisionForConditionalGeneration

model = LlavaOnevisionForConditionalGeneration.from_pretrained(
    "llava-hf/llava-onevision-qwen2-7b-ov-hf",
    torch_dtype=torch.float16,
    device_map="auto",
)
model = flashvid(model, retention_ratio=0.1, alpha=0.7, temporal_threshold=0.8)

# Inference proceeds identically -- but ~10x fewer visual tokens reach the LLM
output = model.generate(**inputs, max_new_tokens=256)
```

Output: Same quality answers with ~10x faster prefill and significantly lower GPU memory.

**Example 2: Processing 10x more frames on Qwen2.5-VL within a fixed budget**

User: "I have Qwen2.5-VL running on an A100 with 32 frames. I want to cover longer videos for better temporal understanding without upgrading hardware."

Approach:
1. Wrap Qwen2.5-VL with FlashVID at retention_ratio=0.1
2. Increase num_frames from 32 to 320
3. The 320 frames compressed at 10% produce roughly the same token count as the original 32 frames uncompressed

```python
from flashvid import flashvid
from transformers import Qwen2_5_VLForConditionalGeneration

model = Qwen2_5_VLForConditionalGeneration.from_pretrained(
    "Qwen/Qwen2.5-VL-7B-Instruct",
    torch_dtype=torch.bfloat16,
    device_map="auto",
)
model = flashvid(model, retention_ratio=0.1, alpha=0.7, temporal_threshold=0.8)

# Now sample 320 frames instead of 32
video_inputs = process_video("long_video.mp4", num_frames=320)
output = model.generate(**video_inputs, max_new_tokens=512)
```

Output: 8.6% relative improvement in video understanding accuracy (per the paper's benchmarks) because the model sees 10x more temporal context.

**Example 3: Tuning compression for a fast-action sports video**

User: "I'm analyzing basketball game footage with lots of fast motion. Default FlashVID settings drop too many details."

Approach:
1. Increase `temporal_threshold` to 0.9 so only very similar tokens are merged (fast motion means adjacent frames differ more)
2. Increase `retention_ratio` to 0.2 for a less aggressive compression
3. Increase `alpha` to 0.8 to weight ADTS (per-frame diversity) over TSTM (cross-frame merging)

```python
model = flashvid(
    model,
    retention_ratio=0.2,       # keep 20% -- less aggressive
    alpha=0.8,                 # favor per-frame diversity over temporal merging
    temporal_threshold=0.9,    # only merge near-identical cross-frame tokens
)
```

Output: Better preservation of fast-moving player details while still achieving ~5x token reduction.

## Best Practices

**Do:**
- Start with the paper's recommended defaults (`retention_ratio=0.1`, `alpha=0.7`, `temporal_threshold=0.8`) and only adjust if quality degrades on your specific video type.
- Profile both prefill latency and peak memory to quantify the benefit -- the main speedup is in the prefill stage where visual tokens are processed.
- Use the extended frame budget capability to improve accuracy on long videos rather than just for speedup -- more frames with compression often beats fewer frames without it.
- Test on your target benchmark or task after applying FlashVID to verify acceptable quality at your chosen retention ratio.

**Avoid:**
- Do not set `retention_ratio` below 0.05 -- below this level, critical spatial information is lost and accuracy drops sharply.
- Do not apply FlashVID to single-image inputs where there is no temporal redundancy to exploit -- TSTM provides no benefit without multiple frames, and ADTS alone is less effective than image-specific token pruning methods.
- Do not assume the same hyperparameters work for all video types -- fast-action video needs higher temporal_threshold and retention_ratio than static surveillance footage.
- Do not combine FlashVID with other token pruning methods simultaneously without benchmarking, as double compression can cause compounding information loss.

## Error Handling

- **OOM even after FlashVID:** Reduce `retention_ratio` further (e.g., from 0.1 to 0.07) or reduce `num_frames`. The token count scales as `num_frames * tokens_per_frame * retention_ratio`.
- **Quality degradation on specific video types:** Increase `retention_ratio` to 0.2-0.3 and tune `alpha` and `temporal_threshold` for that content type. Static videos benefit from lower temporal_threshold; dynamic videos need higher values.
- **Model not supported:** FlashVID currently supports LLaVA-OneVision, LLaVA-Video, Qwen2.5-VL, and Qwen3-VL. For other VLLMs, you would need to adapt the wrapper to intercept visual tokens at the correct point in the architecture (between vision encoder output and LLM input).
- **Incorrect wrapping:** Ensure `flashvid()` is called after the model is fully loaded and before any `generate()` calls. The wrapper modifies the model in-place.
- **No speedup observed:** The benefit is primarily in the prefill phase. If your bottleneck is generation (decoding), FlashVID won't help. Measure prefill time specifically with `torch.cuda.Event` timers.

## Limitations

- Only works for video inputs with multiple frames -- single-image understanding tasks do not benefit from the spatiotemporal merging.
- Currently supports four model families (LLaVA-OneVision, LLaVA-Video, Qwen2.5-VL, Qwen3-VL). Extending to other VLLMs requires adapting the vision-token interception point.
- The MMDP optimization in ADTS adds a small constant overhead per frame. For very short videos (2-4 frames), this overhead may offset the token reduction benefit.
- Compression is applied uniformly across all frames. Scenes with rapid content changes may lose important transition details that a content-adaptive per-frame retention ratio could preserve.
- The method assumes visual tokens are the dominant cost. If the text prompt is very long relative to the video tokens, the relative speedup diminishes.

## Reference

**Paper:** [FlashVID: Efficient Video Large Language Models via Training-free Tree-based Spatiotemporal Token Merging](https://arxiv.org/abs/2602.08024v1) (ICLR 2026 Oral). Focus on Section 3 for the ADTS and TSTM algorithms, Table 1 for benchmark results across retention ratios, and Table 3 for the frame-extension experiments on Qwen2.5-VL.

**Code:** [https://github.com/Fanziyang-v/FlashVID](https://github.com/Fanziyang-v/FlashVID)