---
name: "vista-scene-aware-optimization-streaming"
description: |
  Implement Vista-style scene-aware streaming video processing pipelines with dynamic segmentation,
  hierarchical compression, and selective recall. Use when building video QA systems, streaming video
  analysis, real-time surveillance analytics, or any application that processes continuous video and
  must answer queries against an ever-growing visual context.

  Trigger phrases:
  - "streaming video question answering"
  - "scene-aware video segmentation and compression"
  - "build a video stream processing pipeline with memory management"
  - "real-time video QA with GPU/CPU memory offloading"
  - "long-form video understanding with selective recall"
  - "process continuous video frames and answer queries efficiently"
---

# Vista: Scene-Aware Optimization for Streaming Video QA

This skill enables Claude to design and implement **streaming video processing pipelines** that use Vista's three-stage architecture: scene-aware segmentation (dynamically grouping frames into coherent scenes), scene-aware compression (reducing each scene to a compact token via temporal-spatial pooling with L2-norm weighting), and scene-aware recall (retrieving only relevant scenes at query time via dot-product scoring). The approach solves the core tension in streaming video QA — maintaining full context without unbounded memory growth — by keeping compressed scene tokens on GPU, offloading full-resolution frames to CPU, and selectively recalling only the top-k relevant scenes when a question arrives.

## When to Use

- When building a streaming video QA system that must answer arbitrary post-hoc queries over hours of continuous video
- When designing a surveillance or monitoring pipeline that needs to index and retrieve relevant scenes from a live feed
- When implementing a video processing backend that must manage GPU memory efficiently while retaining the ability to recall full-resolution frames on demand
- When adding scene boundary detection to a video pipeline (e.g., for automatic chaptering, highlight extraction, or content indexing)
- When refactoring a naive "store all frames" video system into a tiered memory architecture with compressed GPU cache and CPU-offloaded originals
- When integrating scene-level retrieval into an existing vision-language model pipeline (LLaVA, InternVL, Qwen-VL, etc.)

## Key Technique

Vista's insight is that streaming video naturally organizes into **scenes** — temporally contiguous segments with visual coherence — and that these scenes are the right unit for compression, storage, and retrieval. Unlike fixed-window approaches that either lose context (small windows) or overflow memory (large windows), Vista adapts its segmentation to the actual visual content.

**Scene boundary detection** uses a dual-condition check: a new scene begins when the current frame's similarity to both the scene's anchor frame (first frame) AND the immediately preceding frame drops below a threshold `τ` (optimal: 0.8). This dual condition prevents false boundaries from momentary occlusions while catching genuine scene transitions. A temporal overlap of 1 frame between consecutive scenes smooths boundary effects.

**Compression** follows a temporal-then-spatial strategy: (1) average-pool across time for each spatial patch independently, (2) reshape into a 2D spatial grid, (3) weight patches by their L2 norms (high-norm patches carry more visual information), (4) aggregate via spatial average pooling into a single compact scene token. This token lives on GPU for fast retrieval. The original full-resolution features are offloaded to CPU. At query time, **scene-aware recall** encodes the text query, computes dot-product similarity against all scene tokens, selects the top-k (k=3) most relevant scenes, fetches their full-resolution frames from CPU, and combines them with recent frames from a local sliding window to form the model input.

## Step-by-Step Workflow

1. **Set up the frame ingestion pipeline.** Accept frames from a video stream (live camera, RTSP, file decode). Extract visual features using a frozen vision encoder (e.g., SigLIP, CLIP ViT). Store each frame's feature tensor with its timestamp.

2. **Implement scene boundary detection.** Maintain the current scene's anchor frame (first frame) and previous frame. For each new frame, compute cosine similarity against both. If both similarities fall below threshold `τ = 0.8`, trigger a scene boundary. Start a new scene with the current frame as anchor. Apply 1-frame overlap (include the boundary frame in both scenes).

3. **Compress completed scenes into scene tokens.** When a scene closes, take its N frame features (each with spatial patches). Apply temporal average pooling across the N frames per spatial position. Reshape into a 2D grid. Compute L2 norms per patch as importance weights. Apply weighted spatial average pooling within a sliding window (window size `a = 2`). Final average pool into a single scene token vector. Store this token on GPU.

4. **Offload full-resolution frames to CPU.** Move the original per-frame feature tensors for the completed scene from GPU to CPU memory (using `tensor.cpu()` or equivalent). Maintain an index mapping scene IDs to their CPU tensor locations.

5. **Maintain a local sliding window.** Keep the most recent M frames (uncompressed) on GPU as a "local context window." This ensures the model always has access to the immediate visual context without requiring recall.

6. **Encode incoming queries.** When a user query arrives, encode it with the language model's text encoder to produce a query embedding vector `q`.

7. **Score and retrieve relevant scenes.** Compute dot-product similarity `α_i = q · T_i` between the query embedding and every scene token. Select the top-k scenes (k=3) by score. Fetch their full-resolution frame features from CPU back to GPU.

8. **Construct the model input.** Concatenate: (a) full-resolution frames from recalled scenes, (b) recent frames from the local sliding window, (c) the tokenized query. Feed this into the vision-language model for answer generation.

9. **Enforce capacity limits.** Cap maximum scene length at `m = 8` frames to prevent unbounded growth in static scenes. If a scene exceeds this, force a boundary. Monitor total scene token count and apply eviction (oldest-first) if GPU memory pressure exceeds a threshold.

10. **Return the answer and update state.** Deliver the model's response. Continue ingesting frames. The recalled scenes return to their compressed state (scene tokens stay on GPU, full frames go back to CPU).

## Concrete Examples

**Example 1: Building a streaming video QA backend**

```
User: I need a Python service that processes a live video stream and answers
questions about what happened. It should handle hours of video without running
out of GPU memory.

Approach:
1. Create a FrameIngester class that decodes frames from an RTSP stream and
   extracts features via a CLIP ViT encoder.
2. Implement SceneManager with:
   - add_frame(feature): checks dual-condition boundary (anchor sim < 0.8
     AND adjacent sim < 0.8), groups frames into scenes.
   - compress_scene(scene): temporal avg pool → spatial L2-weighted pool →
     single token. Stores token on GPU, offloads frames to CPU.
3. Implement SceneRecaller with:
   - recall(query_embedding, k=3): dot-product against all scene tokens,
     returns top-k scene frame features fetched from CPU.
4. Wire into a FastAPI endpoint: POST /query accepts {"question": "..."}.
   Encodes query, recalls scenes, builds input, runs VLM inference.

Output (key class):
```python
class SceneManager:
    def __init__(self, tau=0.8, max_scene_len=8, window_size=2):
        self.tau = tau
        self.max_scene_len = max_scene_len
        self.window_size = window_size
        self.scenes = []           # list of compressed scene tokens (GPU)
        self.cpu_store = {}        # scene_id -> full-res frames (CPU)
        self.current_scene = []
        self.anchor_feature = None
        self.prev_feature = None

    def add_frame(self, frame_feature: torch.Tensor):
        if self.anchor_feature is None:
            self.anchor_feature = frame_feature
            self.prev_feature = frame_feature
            self.current_scene.append(frame_feature)
            return

        sim_anchor = F.cosine_similarity(
            frame_feature.flatten().unsqueeze(0),
            self.anchor_feature.flatten().unsqueeze(0)
        ).item()
        sim_adj = F.cosine_similarity(
            frame_feature.flatten().unsqueeze(0),
            self.prev_feature.flatten().unsqueeze(0)
        ).item()

        if (sim_anchor < self.tau and sim_adj < self.tau) \
                or len(self.current_scene) >= self.max_scene_len:
            self._close_scene()
            # Overlap: include boundary frame in new scene
            self.current_scene = [frame_feature]
            self.anchor_feature = frame_feature
        else:
            self.current_scene.append(frame_feature)

        self.prev_feature = frame_feature

    def _close_scene(self):
        scene_id = len(self.scenes)
        frames = torch.stack(self.current_scene)  # (N, C, H, W) or (N, P, D)
        token = self._compress(frames)
        self.scenes.append(token)                  # GPU
        self.cpu_store[scene_id] = frames.cpu()    # offload to CPU

    def _compress(self, frames: torch.Tensor) -> torch.Tensor:
        # Temporal pooling: average across time per patch
        temporal = frames.mean(dim=0)  # (P, D) or (C, H, W)
        # L2-norm importance weighting
        norms = temporal.norm(dim=-1, keepdim=True)
        weighted = temporal * (norms / (norms.sum() + 1e-8))
        # Spatial aggregation
        token = weighted.mean(dim=0)  # (D,)
        return token
```

**Example 2: Adding scene-aware retrieval to an existing video index**

```
User: I have a video archive with pre-extracted CLIP features per frame.
I want to add scene-level indexing so I can search by natural language queries.

Approach:
1. Run scene segmentation offline: iterate frames sequentially, apply the
   dual-condition boundary detector (tau=0.8) to partition into scenes.
2. Compress each scene into a single token using temporal-spatial pooling.
3. Build a scene token matrix (N_scenes x D). Store alongside a metadata
   index mapping scene_id -> (start_frame, end_frame, timestamps).
4. At query time: encode query with CLIP text encoder, compute dot products
   against all scene tokens, return top-k scene metadata with frame ranges.

Output:
```python
def build_scene_index(frame_features: List[torch.Tensor], tau=0.8, max_len=8):
    """Partition pre-extracted frame features into scenes and compress."""
    scenes = []
    current, anchor_idx = [0], 0

    for i in range(1, len(frame_features)):
        sim_anchor = F.cosine_similarity(
            frame_features[i].unsqueeze(0),
            frame_features[anchor_idx].unsqueeze(0)
        ).item()
        sim_adj = F.cosine_similarity(
            frame_features[i].unsqueeze(0),
            frame_features[i - 1].unsqueeze(0)
        ).item()

        if (sim_anchor < tau and sim_adj < tau) or len(current) >= max_len:
            scenes.append((current[0], current[-1], current))
            current = [i]
            anchor_idx = i
        else:
            current.append(i)

    if current:
        scenes.append((current[0], current[-1], current))

    tokens = []
    for start, end, indices in scenes:
        feats = torch.stack([frame_features[j] for j in indices])
        token = feats.mean(dim=0)  # temporal pool
        norm = token.norm(dim=-1, keepdim=True)
        token = token * (norm / (norm.sum() + 1e-8))
        tokens.append(token.mean(dim=0))

    return torch.stack(tokens), scenes  # (N, D), scene metadata

def query_scenes(query_text, scene_tokens, clip_model, k=3):
    q = clip_model.encode_text(query_text)  # (D,)
    scores = (scene_tokens @ q.unsqueeze(-1)).squeeze(-1)
    top_k = scores.topk(k).indices.tolist()
    return top_k
```

**Example 3: Real-time meeting summarization with scene-aware chunking**

```
User: I'm building a meeting recorder that segments a video call into
topic-based scenes and lets users ask "what was discussed about X?"

Approach:
1. Capture screen frames at 1 FPS. Extract features with a vision encoder.
2. Apply Vista segmentation (tau=0.8) — scene boundaries naturally align
   with topic changes (new slides, speaker switches, shared screen changes).
3. Compress each scene. Additionally store a text summary of each scene
   (generated by the VLM at scene-close time) as metadata.
4. On query: encode question, score against scene tokens, retrieve top-3
   scenes, pass full frames + text summaries to VLM for a grounded answer.

Output structure:
  SceneStore:
    scene_0: {token: Tensor(D), frames_cpu: Tensor(N,C,H,W),
              summary: "Team discussed Q3 revenue targets", t_start: 0:00, t_end: 2:15}
    scene_1: {token: Tensor(D), frames_cpu: Tensor(N,C,H,W),
              summary: "Demo of new dashboard feature", t_start: 2:15, t_end: 5:40}
    ...

  Query: "What was the revenue target?"
  → Recall scene_0 (score: 0.87), scene_3 (score: 0.72), scene_7 (score: 0.65)
  → Fetch full frames for those scenes → VLM generates grounded answer
```

## Best Practices

- **Do** use the dual-condition boundary check (anchor AND adjacent similarity). Single-condition checks produce too many false positives from momentary occlusions or camera shake.
- **Do** set a maximum scene length cap (m=8 frames is a good default). Static scenes like title cards or idle cameras will otherwise grow unboundedly.
- **Do** maintain a local sliding window of recent uncompressed frames. This ensures the system can always answer questions about what just happened, even before those frames are compressed into a scene.
- **Do** apply L2-norm weighting during spatial compression — patches with higher norms carry more semantic content. Uniform averaging loses too much signal from visually rich regions.
- **Avoid** setting `τ` too low (< 0.6). This merges distinct scenes and degrades retrieval precision. Start at 0.8 and tune downward only if your video has very gradual transitions.
- **Avoid** recalling too many scenes (k > 5). The input context grows linearly with k, and diminishing returns set in quickly. k=3 is the empirically validated sweet spot.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| All frames land in one scene | τ too high or video is truly static | Lower τ or enforce max scene length cap |
| Too many tiny scenes (1-2 frames each) | τ too low or very dynamic video (e.g., action sequences) | Raise τ or set a minimum scene length |
| GPU OOM during recall | Fetching too many full-resolution frames | Reduce k, downsample recalled frames, or use gradient checkpointing |
| Slow scene token scoring | Thousands of accumulated scenes | Batch dot-product computation; consider approximate nearest neighbor (FAISS) for very long streams |
| Poor retrieval quality | Query embedding and scene tokens in different feature spaces | Ensure both use the same encoder backbone; fine-tune with contrastive loss if needed |
| CPU memory exhaustion from offloaded frames | Very long streams (hours+) | Implement disk-based offloading or frame feature quantization (float16/int8) |

## Limitations

- **Scene boundary quality depends on visual similarity metrics.** Gradual transitions (slow fades, continuous camera pans) may not trigger clean boundaries. Audio or subtitle cues are not used.
- **Compression is lossy.** The single-token-per-scene representation discards fine-grained spatial and temporal details. Queries requiring precise object localization within a scene may need the full-resolution recall path.
- **The approach assumes visual coherence equals semantic coherence.** Two visually similar but semantically different scenes (e.g., two different whiteboards) may be merged or confused.
- **Retrieval is limited to scene-level granularity.** Frame-level retrieval within a recalled scene requires additional processing.
- **Not designed for real-time sub-second latency.** The recall → fetch-from-CPU → VLM-inference pipeline adds latency appropriate for near-real-time (seconds) but not instantaneous response.
- **Requires a pre-trained vision encoder.** The system does not train its own features — quality depends on the backbone (CLIP, SigLIP, etc.).

## Reference

**Paper:** [Vista: Scene-Aware Optimization for Streaming Video Question Answering under Post-Hoc Queries](https://arxiv.org/abs/2602.08448v1) (AAAI 2026)
**Key takeaway:** Read Section 3 for the dual-condition scene boundary formula (`B(F_i) = I[S_anchor < τ ∧ S_adj < τ]`), Section 3.2 for the L2-norm weighted compression algorithm, and Section 3.3 for the dot-product recall mechanism with top-k selection. Table 1 shows StreamingBench results; τ=0.8, k=3, m=8 are the validated hyperparameters.