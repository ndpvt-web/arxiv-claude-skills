---
name: "flashvid-video-training-free-tree-based"
description: "Accelerate Video Large Language Models (VLLMs) using FlashVID's training-free spatiotemporal token compression. Applies Attention and Diversity-based Token Selection (ADTS) plus Tree-based Spatiotemporal Token Merging (TSTM) to reduce visual tokens by up to 90% with minimal accuracy loss. Use when: 'speed up video LLM inference', 'reduce video token count', 'compress visual tokens for video understanding', 'integrate FlashVID with Qwen2.5-VL or LLaVA', 'process longer videos within memory budget', 'plug-and-play video token merging'."
---

# FlashVID: Training-Free Spatiotemporal Token Compression for Video LLMs

This skill enables Claude to help users integrate and configure FlashVID, a training-free inference acceleration framework for Video Large Language Models. FlashVID compresses visual tokens by first selecting the most informative tokens per frame using attention and diversity scoring (ADTS), then merging temporally redundant tokens across frames via hierarchical tree structures (TSTM). The method retains only 10% of visual tokens while preserving 99.1% of accuracy on LLaVA-OneVision, and can extend video frame input by 10x on Qwen2.5-VL within the same compute budget.

## When to Use

- When the user wants to speed up video LLM inference without retraining the model
- When processing long videos that exceed GPU memory or cause slow prefilling
- When integrating FlashVID into a pipeline using LLaVA-OneVision, LLaVA-Video, Qwen2.5-VL, or Qwen3-VL
- When the user asks how to reduce visual token count for video understanding tasks
- When the user wants to increase the number of input frames within a fixed computational budget
- When implementing spatiotemporal token merging or token pruning for video transformers
- When benchmarking video LLM efficiency on VideoMME, EgoSchema, MVBench, LongVideoBench, or MLVU

## Key Technique

**The core insight:** Existing VLLM compression methods treat spatial and temporal redundancy independently, missing the fact that correlated visual features shift in position, scale, and orientation across frames. FlashVID addresses this with a two-stage pipeline applied *after* the vision encoder but *before* the LLM backbone (the "Before-LLM Compression" paradigm), requiring zero training or fine-tuning.

**Stage 1 — ADTS (Attention and Diversity-based Token Selection):** For each frame, ADTS computes three signals: (1) *[CLS] attention scores* derived from the vision encoder's attention matrices (or averaged received-attention weights when no explicit [CLS] token exists), identifying visually salient tokens; (2) *diversity scores* from a frame-wise cosine distance matrix `D = 1 - cos(E, E)`, ensuring selected tokens are not redundant with each other; (3) *event relevance calibration* via global average-pooled frame embeddings, emphasizing tokens relevant to the overall video narrative. These three signals are combined to solve a calibrated Max-Min Diversity Problem (MMDP), selecting a compact set of `r * N` tokens per frame (where `r` is the retention ratio). The parameter `alpha` controls the balance between ADTS-selected tokens and TSTM-merged tokens.

**Stage 2 — TSTM (Tree-based Spatiotemporal Token Merging):** The remaining unselected tokens are organized into hierarchical trees spanning consecutive frames. Cosine similarity is computed between each token and its counterpart in the adjacent frame: `S = cos(E_f, E_{f+1})`. If similarity exceeds a temporal threshold `T_tau`, tokens are linked parent-to-child across frames, forming trees. All tokens within each tree are aggregated via mean pooling into a single representative. This captures fine-grained temporal continuity (e.g., a moving object tracked across frames) while discarding truly redundant duplicates.

## Step-by-Step Workflow

1. **Install FlashVID** — Clone the repository and install dependencies using `uv`:
   ```bash
   git clone https://github.com/Fanziyang-v/FlashVID.git
   cd FlashVID && uv sync
   ```

2. **Load the base Video LLM** — Instantiate one of the supported models (LLaVA-OneVision, LLaVA-Video, Qwen2.5-VL, or Qwen3-VL) with its standard loading procedure (transformers, model weights, processor).

3. **Wrap the model with `flashvid()`** — Call the wrapper function to inject the compression pipeline between the vision encoder and the LLM backbone:
   ```python
   from flashvid import flashvid
   model = flashvid(
       model,
       retention_ratio=0.1,   # keep 10% of tokens via ADTS
       alpha=0.7,             # 70% ADTS-selected, 30% TSTM-merged
       temporal_threshold=0.8, # merge tokens with >0.8 cosine similarity
   )
   ```

4. **Choose the retention ratio based on your accuracy/speed tradeoff** — At `retention_ratio=0.1`, expect ~99% relative accuracy and ~6.3x prefilling speedup. For less aggressive compression, use 0.2-0.3. For maximum speed at some accuracy cost, try 0.05.

5. **Tune `alpha` to balance selection vs. merging** — `alpha=1.0` uses only ADTS (pure selection, ~98.5% accuracy). `alpha=0.0` uses only TSTM (pure merging, ~97.4% accuracy). The optimal `alpha=0.7` combines both for 99.1% accuracy. Adjust if your video domain has unusual temporal dynamics.

6. **Set the temporal threshold `T_tau`** — Higher values (0.9) are conservative and only merge very similar tokens; lower values (0.6) merge more aggressively. Default 0.8 works well across benchmarks. Lower it for static/slow videos; raise it for fast-action content.

7. **Run inference** — Pass video frames through the wrapped model normally. FlashVID intercepts tokens after vision encoding and compresses them transparently:
   ```bash
   python playground/llava_ov_infer.py \
     --video-path video.mp4 \
     --question "Describe the video in detail." \
     --num-frames 32 \
     --enable-flashvid
   ```

8. **Scale to longer videos** — With 90% token reduction, you can increase `--num-frames` by up to 10x within the same memory budget. For example, go from 32 frames to 320 frames on Qwen2.5-VL, which yields an 8.6% relative improvement on long-video benchmarks.

9. **Evaluate on standard benchmarks** — Use the provided `scripts/` and `lmms-eval/` integration to reproduce results on VideoMME, EgoSchema, MVBench, LongVideoBench, and MLVU.

10. **Integrate into custom pipelines** — Since FlashVID is a pure Python wrapper requiring no model weight changes, embed it into any serving framework (vLLM, TGI, custom FastAPI) by wrapping the model object before serving.

## Concrete Examples

**Example 1: Speeding up LLaVA-OneVision inference**

User: "I'm running LLaVA-OneVision for video QA but inference is too slow. Can I compress the visual tokens without retraining?"

Approach:
1. Install FlashVID alongside the existing LLaVA-OneVision setup
2. After loading the model, wrap it: `model = flashvid(model, retention_ratio=0.1, alpha=0.7, temporal_threshold=0.8)`
3. Run inference as usual — the model now processes 90% fewer visual tokens
4. Expect ~6.3x prefilling speedup and ~2.1x time-to-first-token improvement with 99.1% accuracy retention

Output:
```
# Before FlashVID: 32 frames x 729 tokens/frame = 23,328 visual tokens
# After FlashVID:  32 frames -> ~2,333 effective tokens (10% retention)
# Prefilling time: 4.2s -> 0.67s (6.3x speedup)
# VideoMME accuracy: 58.2 -> 57.7 (99.1% retained)
```

**Example 2: Processing 10x more frames on Qwen2.5-VL**

User: "I need Qwen2.5-VL to understand a 10-minute video but I can only fit 32 frames in memory. How do I get more temporal coverage?"

Approach:
1. Wrap Qwen2.5-VL with FlashVID at retention_ratio=0.1
2. Increase frame sampling from 32 to 320 frames — the token count remains equivalent to the original 32-frame budget
3. The model now sees 10x more temporal information, improving long-video comprehension

Output:
```python
from flashvid import flashvid

model = flashvid(model, retention_ratio=0.1, alpha=0.7, temporal_threshold=0.8)

# Same GPU memory as 32 uncompressed frames, but with 320 frames of coverage
response = model.generate(
    video_path="long_video.mp4",
    num_frames=320,
    question="Summarize the key events in this video."
)
# Result: +8.6% relative improvement on long-video benchmarks vs. 32-frame baseline
```

**Example 3: Tuning compression parameters for action-heavy video**

User: "I'm analyzing sports footage with fast motion. The default FlashVID settings are losing important action details."

Approach:
1. Raise `temporal_threshold` from 0.8 to 0.9 — this makes TSTM more conservative, only merging tokens that are nearly identical across frames, preserving fast-changing regions
2. Increase `retention_ratio` from 0.1 to 0.2 — keep more tokens per frame to capture rapid spatial changes
3. Keep `alpha=0.7` or increase to 0.8 to favor ADTS selection (which preserves attention-salient tokens like moving players)

Output:
```python
model = flashvid(
    model,
    retention_ratio=0.2,    # keep 20% for fast-action content
    alpha=0.8,              # favor attention-based selection
    temporal_threshold=0.9,  # conservative merging for dynamic scenes
)
# Tokens retained: 20% (vs. default 10%)
# Speedup: ~3x (vs. default ~6x) — trading speed for accuracy on dynamic content
```

## Best Practices

- **Do:** Start with the default parameters (`retention_ratio=0.1, alpha=0.7, temporal_threshold=0.8`) and adjust only after measuring accuracy on your specific task
- **Do:** Use lower retention ratios (0.05-0.1) for videos with mostly static backgrounds and slow motion, where most tokens are genuinely redundant
- **Do:** Increase frame count proportionally when reducing retention ratio — the whole point is to trade token density per frame for broader temporal coverage
- **Do:** Validate on a small held-out set before deploying, since compression impact varies by video domain
- **Avoid:** Setting `alpha=0` (TSTM-only) or `alpha=1` (ADTS-only) in production — the combined approach consistently outperforms either alone by 0.6-1.7%
- **Avoid:** Using very low temporal thresholds (<0.5) on action-heavy or scene-transition-rich videos, as this will merge dissimilar tokens and lose critical temporal information

## Error Handling

| Problem | Cause | Fix |
|---|---|---|
| OOM despite FlashVID | retention_ratio too high or too many frames | Lower `retention_ratio` or reduce `--num-frames` |
| Accuracy drops significantly | Overly aggressive compression for the video type | Increase `retention_ratio` to 0.2-0.3; raise `temporal_threshold` |
| Model outputs incoherent descriptions | Temporal merging collapsing distinct events | Raise `temporal_threshold` to 0.9+; increase `alpha` toward 1.0 |
| Unsupported model error | Using a model not in the supported list | FlashVID currently supports LLaVA-OneVision, LLaVA-Video, Qwen2.5-VL, and Qwen3-VL only |
| No speedup observed | Bottleneck is in text generation, not prefilling | FlashVID accelerates prefilling (visual token processing); it does not affect autoregressive decoding speed |

## Limitations

- **Model support is limited** — Only works with LLaVA-OneVision, LLaVA-Video, Qwen2.5-VL, and Qwen3-VL. Other video LLMs require adapting the wrapper to their specific vision encoder interface.
- **Training-free means no task adaptation** — The compression is generic. Task-specific fine-tuning of compression parameters could yield better results but is outside FlashVID's scope.
- **Prefilling-only speedup** — FlashVID compresses visual tokens before the LLM, so it accelerates the prefilling phase. It does not speed up autoregressive text generation, which may be the bottleneck for short prompts.
- **Static threshold sensitivity** — The temporal threshold is a fixed hyperparameter. Videos with mixed dynamics (slow scenes followed by fast action) may benefit from adaptive thresholds, which FlashVID does not yet support.
- **Evaluation gaps** — Results are demonstrated on academic benchmarks. Domain-specific video tasks (medical imaging, surveillance, manufacturing inspection) may need separate validation.

## Reference

**Paper:** [FlashVID: Efficient Video Large Language Models via Training-free Tree-based Spatiotemporal Token Merging](https://arxiv.org/abs/2602.08024v1) (ICLR 2026 Oral)
**Code:** [github.com/Fanziyang-v/FlashVID](https://github.com/Fanziyang-v/FlashVID)
**Key takeaway:** Look at Algorithm 1 for the full ADTS+TSTM pipeline pseudocode, Table 4 for ablation results showing each component's contribution, and Table 5 for the alpha parameter sensitivity analysis.