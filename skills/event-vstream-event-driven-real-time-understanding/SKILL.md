---
name: "event-vstream-event-driven-real-time-understanding"
description: "Build event-driven video stream processing pipelines that detect meaningful state transitions instead of processing every frame. Use when asked to: 'build a real-time video understanding system', 'detect events in a video stream', 'process long video with memory', 'reduce redundant frame processing', 'stream video to LLM efficiently', 'build an event-aware video pipeline'."
---

# Event-VStream: Event-Driven Real-Time Video Stream Understanding

This skill enables Claude to design and implement event-driven video stream processing systems based on the Event-VStream framework. Instead of processing every frame at fixed intervals (which wastes compute on redundant content and forgets past context), Event-VStream detects semantically meaningful state transitions by fusing motion, semantic, and predictive cues, then triggers language generation only at those boundaries. A persistent memory bank consolidates event embeddings for long-horizon reasoning. This approach achieves competitive accuracy while maintaining sub-100ms latency across multi-hour streams.

## When to Use

- When the user asks to build a real-time video understanding or captioning system that must handle long streams without degrading
- When the user wants to reduce compute costs by skipping redundant frames in a video processing pipeline
- When building a system that needs to detect "something happened" moments in continuous video (surveillance, dashcam, egocentric, sports)
- When the user needs an event memory system that persists across hours of video without running out of context
- When implementing streaming video-to-text pipelines that should speak only when something meaningful changes
- When the user wants to segment continuous video into discrete semantic events for indexing or retrieval
- When adapting a vision-language model for real-time streaming instead of batch processing

## Key Technique

**Event-driven processing replaces fixed-interval decoding.** Traditional streaming video-LLM systems sample frames at a constant rate (e.g., every 0.5s) and decode text continuously. This produces repetitive outputs during static scenes and misses fast transitions. Event-VStream flips this: it monitors a lightweight boundary score fusing three complementary signals, and only invokes the expensive language model when a genuine state transition is detected.

**Three-signal boundary detection.** The boundary score `E_t` combines: (1) *semantic drift* -- cosine distance between the current frame embedding and a running event average, catching content changes; (2) *motion cue* -- normalized optical flow or frame-difference energy, which empirically precedes semantic drift by ~2 seconds and acts as an early warning; (3) *prediction error* -- the L2 error of a lightweight 3-layer MLP that predicts the next frame embedding from the previous one, catching unexpected transitions. An adaptive threshold `tau_t` tightens during high-motion segments (to avoid false triggers from continuous motion) and relaxes during stable scenes. Ablation shows all three signals are essential: removing motion drops win rate from 68% to 12%, removing semantics drops it to 38%, and removing prediction drops it to 47%.

**Persistent event memory with merge-or-append.** When a boundary fires, frames within the detected segment are aggregated into an event embedding using Gaussian-weighted pooling (emphasizing frames near the boundary). This embedding is either merged into the most recent memory slot (if cosine similarity exceeds a redundancy threshold `gamma_mem`) or appended as a new entry. This keeps the memory bank compact -- semantically similar consecutive events consolidate rather than accumulating. At generation time, relevant past events are retrieved by similarity to the current event, giving the language model long-horizon context without growing the token budget.

## Step-by-Step Workflow

1. **Set up the frame ingestion loop.** Accept video frames from a stream source (webcam, RTSP, file) at a fixed capture rate (2 FPS is the paper's default). Extract a visual embedding `f_t` for each frame using a pretrained vision encoder (CLIP, SigLIP, or a VideoLLM-Online encoder).

2. **Maintain a running event representation.** Keep an exponential moving average `f_bar` of frame embeddings within the current event segment: `f_bar <- (1 - rho) * f_bar + rho * f_t`. This serves as the "what the current event looks like" anchor.

3. **Compute the three-signal boundary score.** For each frame, calculate:
   - Semantic drift: `(1 - cosine_similarity(f_t, f_bar))`
   - Motion cue: normalized frame-difference energy or optical flow magnitude `m_tilde_t`
   - Prediction error: `c_t = ||MLP(f_{t-1}) - f_t||^2` using a lightweight 3-layer MLP
   - Combined score: `E_t = w_sem * (1 - s_t) + w_mot * m_tilde_t + w_pred * c_t`

4. **Apply the adaptive threshold.** Compute `tau_t = tau_0 * (1 + eta * Var(m_{t-w:t}))` where `tau_0 = 0.96` and `eta = 0.03`. A boundary fires when `sigmoid(E_t) > tau_t`. Enforce a minimum interval `Delta_min` between triggers to coalesce bursty updates, and a maximum interval `Delta_max` to prevent excessive silence.

5. **Aggregate the event embedding on boundary detection.** Collect all frame embeddings since the last boundary. Compute a Gaussian-weighted average: `E_k = sum(w_i * f_i) / sum(w_i)` where `w_i ~ exp(-|t_i - t_boundary| / sigma)`. This emphasizes frames near the transition point.

6. **Update the persistent memory bank.** Apply the merge-or-append rule: if `cosine_similarity(E_k, E_last) > gamma_mem`, merge via `E_last <- (1 - lambda) * E_last + lambda * E_k`. Otherwise, append `E_k` as a new memory slot.

7. **Retrieve relevant context and generate text.** Query the memory bank for the top-K events most similar to `E_k`. Concatenate these retrieved event embeddings with `E_k` as visual context tokens, prepend a system prompt, and invoke the language model to produce a description or answer: `y_k = LM(E_k, Retrieve(Memory, E_k))`.

8. **Between boundaries, remain silent.** Do not invoke the language model. Continue the frame ingestion and scoring loop. This is the key efficiency gain -- the LM is called only when something meaningful happens.

9. **Reset the running average.** After a boundary fires, reset `f_bar` to the current frame embedding to begin tracking the next event segment.

10. **Implement pacing controls for production use.** Add a minimum silence interval (prevents chattering during rapid scene changes) and a maximum silence interval (forces a periodic update even during very long static scenes for liveness).

## Concrete Examples

**Example 1: Building a security camera event narrator**

User: "I want a system that watches a security camera feed and only generates descriptions when something actually happens -- not every frame."

Approach:
1. Ingest RTSP stream at 2 FPS using OpenCV. Extract CLIP embeddings per frame.
2. Implement the three-signal boundary detector with default parameters (`tau_0=0.96`, `eta=0.03`).
3. Train the prediction MLP on 1 hour of typical footage from the same camera (self-supervised: predict next frame embedding from current).
4. On boundary detection, aggregate the event embedding and query an LLM with the prompt: "Describe what just changed in this security camera view."
5. Store events in a SQLite-backed memory bank with cosine-similarity merge (`gamma_mem=0.85`).
6. Expose results via a WebSocket endpoint that pushes event descriptions to a dashboard.

Output:
```python
import numpy as np
from collections import deque

class EventBoundaryDetector:
    def __init__(self, tau_0=0.96, eta=0.03, window=30, rho=0.05,
                 w_sem=0.4, w_mot=0.35, w_pred=0.25,
                 delta_min=2.0, delta_max=30.0):
        self.tau_0 = tau_0
        self.eta = eta
        self.w_sem = w_sem
        self.w_mot = w_mot
        self.w_pred = w_pred
        self.rho = rho
        self.delta_min = delta_min
        self.delta_max = delta_max
        self.f_bar = None  # Running event average
        self.motion_history = deque(maxlen=window)
        self.last_boundary_time = -float('inf')
        self.predictor = None  # Lightweight MLP, set externally

    def cosine_sim(self, a, b):
        return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b) + 1e-8)

    def update(self, f_t, motion_energy, timestamp):
        if self.f_bar is None:
            self.f_bar = f_t.copy()
            self.last_boundary_time = timestamp
            return False

        # Signal 1: Semantic drift
        sem_drift = 1.0 - self.cosine_sim(f_t, self.f_bar)

        # Signal 2: Normalized motion
        self.motion_history.append(motion_energy)
        m_max = max(self.motion_history) if self.motion_history else 1.0
        m_norm = motion_energy / (m_max + 1e-8)

        # Signal 3: Prediction error
        pred_error = 0.0
        if self.predictor is not None:
            pred_error = self.predictor.prediction_error(f_t)

        # Combined boundary score
        E_t = (self.w_sem * sem_drift +
               self.w_mot * m_norm +
               self.w_pred * pred_error)
        p_t = 1.0 / (1.0 + np.exp(-E_t))  # sigmoid

        # Adaptive threshold
        motion_var = np.var(list(self.motion_history)) if len(self.motion_history) > 1 else 0.0
        tau_t = self.tau_0 * (1.0 + self.eta * motion_var)

        # Pacing: enforce min/max intervals
        elapsed = timestamp - self.last_boundary_time
        if elapsed < self.delta_min:
            fire = False
        elif elapsed > self.delta_max:
            fire = True  # Force periodic update
        else:
            fire = p_t > tau_t

        if fire:
            self.f_bar = f_t.copy()  # Reset running average
            self.last_boundary_time = timestamp

        # Update running average (EMA)
        self.f_bar = (1 - self.rho) * self.f_bar + self.rho * f_t
        return fire
```

**Example 2: Long video indexing with event-based retrieval**

User: "I have 4 hours of conference talk recordings. I want to index them so users can search for specific moments by natural language query."

Approach:
1. Process the video offline at 2 FPS with the boundary detector to segment into discrete events.
2. For each event, generate a text description using the LLM with memory context.
3. Store event embeddings, timestamps, and descriptions in a vector database (e.g., ChromaDB).
4. At query time, embed the user's natural language query, retrieve top-K events by cosine similarity, return timestamps and descriptions.

Output:
```python
class EventMemoryBank:
    def __init__(self, gamma_mem=0.85, merge_lambda=0.3):
        self.events = []       # List of {"embedding": ..., "start": ..., "end": ..., "description": ...}
        self.gamma_mem = gamma_mem
        self.merge_lambda = merge_lambda

    def add_event(self, embedding, start_time, end_time):
        if self.events:
            sim = cosine_sim(embedding, self.events[-1]["embedding"])
            if sim > self.gamma_mem:
                # Merge: consolidate similar consecutive events
                last = self.events[-1]
                last["embedding"] = ((1 - self.merge_lambda) * last["embedding"]
                                     + self.merge_lambda * embedding)
                last["end"] = end_time
                return len(self.events) - 1
        # Append as new event
        self.events.append({
            "embedding": embedding,
            "start": start_time,
            "end": end_time,
            "description": None
        })
        return len(self.events) - 1

    def retrieve(self, query_embedding, top_k=5):
        scores = [(i, cosine_sim(query_embedding, e["embedding"]))
                  for i, e in enumerate(self.events)]
        scores.sort(key=lambda x: -x[1])
        return [self.events[i] for i, _ in scores[:top_k]]
```

**Example 3: Adapting an existing video-LLM for streaming**

User: "I have a batch video QA model. How do I convert it to handle real-time streams without running out of memory on long videos?"

Approach:
1. Replace the fixed frame sampler with the event boundary detector. Instead of uniformly sampling N frames from the entire video, process frames online and accumulate only event embeddings.
2. Replace the flat frame-token sequence with the memory bank. At each generation step, the model sees only the current event embedding plus retrieved past events (typically 5-10 tokens total), not all frames.
3. Add the merge-or-append logic so the memory bank stays bounded regardless of video length.
4. Wrap the model in an async loop: ingest frames continuously, fire the decoder only on boundary events.

Output architecture:
```
Frame Stream (2 FPS)
    |
    v
Vision Encoder (CLIP/SigLIP) --> f_t
    |
    v
Boundary Detector (motion + semantic + prediction)
    |
    |-- No boundary: update EMA, continue
    |
    |-- Boundary detected:
            |
            v
        Event Aggregator (Gaussian-weighted pooling)
            |
            v
        Memory Bank (merge-or-append)
            |
            v
        Retrieve top-K past events
            |
            v
        LLM Decoder (current event + retrieved context -> text)
            |
            v
        Output description / answer
```

## Best Practices

- **Do:** Use all three boundary signals (motion, semantic, prediction). Ablations show removing any one causes major degradation -- motion alone catches 12% of events, all three catch 68%.
- **Do:** Train the prediction MLP on domain-specific video. A model trained on indoor surveillance will have different "surprise" patterns than one trained on driving footage. Self-supervised training (predict next embedding from previous) requires no labels.
- **Do:** Tune `gamma_mem` for your use case. Lower values (0.7) create more granular events; higher values (0.9) aggressively merge similar segments. Start at 0.85 and adjust based on event density.
- **Do:** Enforce pacing controls (`Delta_min`, `Delta_max`). Without them, rapid scene changes (e.g., channel surfing) cause decoder flooding, and static scenes (e.g., parking lot at night) produce no output for hours.
- **Avoid:** Processing at more than 2-4 FPS unless your hardware supports it. The boundary detector is lightweight, but the vision encoder is the bottleneck. Higher FPS gives diminishing returns for boundary detection accuracy.
- **Avoid:** Using fixed thresholds without the adaptive component. The variance-based adjustment (`eta * Var(motion)`) prevents false triggers during sustained motion (e.g., a person continuously walking) while staying sensitive during calm periods.
- **Avoid:** Skipping the Gaussian weighting in event aggregation. Uniform averaging dilutes the transition signal -- frames near the boundary carry the most information about what changed.

## Error Handling

- **Boundary detector fires too often (chattering):** Increase `tau_0` or `Delta_min`. Check if the motion signal is noisy -- apply temporal smoothing (e.g., 3-frame moving average) before feeding it to the detector.
- **Boundary detector misses obvious transitions:** Decrease `tau_0` or check that the vision encoder produces meaningfully different embeddings for different scenes. A poorly-trained encoder yields flat similarity scores.
- **Memory bank grows unbounded:** The merge-or-append rule should prevent this, but verify `gamma_mem` is not set too low. Add a hard cap (e.g., 500 events) with FIFO eviction of oldest entries if needed.
- **Prediction MLP produces constant error:** The MLP may have collapsed to predicting the mean embedding. Retrain with a lower learning rate or add dropout. Ensure training data includes diverse transitions.
- **Latency spikes on boundary events:** The LLM decode is the expensive step. Use speculative decoding, quantized models, or limit max output tokens to maintain real-time performance. The paper reports sub-100ms per token with LLaMA-3-8B on RTX 6000 Ada.
- **Out-of-memory on long streams:** Ensure frame embeddings are discarded after event aggregation. Only event-level embeddings should persist in the memory bank. Each event is a single vector, not a sequence of frame tokens.

## Limitations

- **Requires a vision encoder.** The boundary detector operates on frame embeddings, not raw pixels. You need a pretrained vision encoder (CLIP, SigLIP, or similar). The choice of encoder affects what counts as "semantically different."
- **Not suitable for frame-level precision tasks.** Event-VStream operates at event granularity. If you need to identify the exact frame where an action occurs (e.g., frame-accurate video editing), this approach gives only approximate boundaries.
- **Motion signal is camera-dependent.** Optical flow from a static surveillance camera behaves very differently from a handheld or ego-centric camera. The `eta` parameter needs tuning per camera setup.
- **Cold start problem.** The prediction MLP needs domain-specific training data. On completely novel video domains, prediction error provides little signal until the MLP adapts.
- **Text generation quality depends on the base LLM.** Event-VStream is an architecture for *when* to generate, not *what* to generate. The quality of descriptions depends on the underlying vision-language model.
- **Single-stream only.** The paper addresses one video stream at a time. Multi-camera systems require separate detector instances with a shared or federated memory bank (not covered in the paper).

## Reference

**Paper:** [Event-VStream: Event-Driven Real-Time Understanding for Long Video Streams](https://arxiv.org/abs/2601.15655v1) (Guo et al., 2026). Focus on Section 3 (boundary detection formulas and adaptive threshold), Section 4 (memory bank merge-or-append rule), and Table 2 (ablation showing contribution of each signal).