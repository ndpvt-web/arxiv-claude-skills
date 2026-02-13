---
name: "phostream-benchmarking-real-world-streaming"
description: "Build streaming multimodal benchmarks and evaluate omnimodal assistants on continuous audio-visual input with temporal reasoning. Use when: 'benchmark a streaming model', 'evaluate real-time video QA', 'test mobile assistant temporal reasoning', 'build a streaming evaluation pipeline', 'measure when-to-respond accuracy', 'create open-ended video QA benchmark'."
---

# PhoStream: Streaming Multimodal Benchmark & Evaluation Pipelines

This skill enables Claude to design, build, and apply streaming evaluation pipelines for multimodal AI assistants that must process continuous audio-visual input and decide *when* to respond, not just *what* to say. Based on the PhoStream benchmark methodology, it covers constructing temporally-annotated QA datasets from video streams, simulating realistic online inference with sliding context windows, and scoring open-ended responses with LLM-as-a-Judge rubrics calibrated for temporal correctness.

## When to Use

- When the user wants to **benchmark a multimodal model** on streaming video/audio comprehension rather than static image or short-clip QA
- When building an **evaluation harness for a mobile or real-time assistant** that must handle on-screen (app tutorials, UI navigation) and off-screen (vlogs, egocentric video) scenarios
- When the user needs to **measure temporal reasoning**: can a model answer backward (retrospective), instant (present-moment), and forward (proactive/deferred) questions correctly?
- When designing an **Automated Generative Pipeline** to produce timestamped open-ended QA pairs from long-form videos at scale
- When implementing an **Online Inference Pipeline** that streams frames to a model at fixed intervals with a sliding context window
- When the user asks to **diagnose early-response bias** — models answering before the required evidence appears in the stream
- When evaluating open-ended multimodal responses with a **structured LLM-as-a-Judge rubric** (0–5 scale mapped to 0–100)

## Key Technique

### Temporal Task Taxonomy

PhoStream's core insight is decomposing streaming QA into three temporal categories that expose fundamentally different model capabilities:

1. **Backward tasks** — retrospective questions about events that already occurred relative to the query timestamp. These test memory and summarization over the stream history.
2. **Instant tasks** — questions answerable from the current 1–2 second window. These test real-time perception.
3. **Forward tasks** — questions issued *before* the answer evidence appears. The model must output "Silent" until a **Timestamp Proactive** (the earliest moment the question becomes answerable), then respond within a 2-second window.

The critical finding is **temporal asymmetry**: state-of-the-art models score 80+ on Backward/Instant but collapse to ~16 on Forward tasks, with early-response rates exceeding 79%. This means models are overconfident — they guess answers before the visual/audio cues arrive. Any streaming benchmark that ignores Forward tasks will dramatically overestimate real-world assistant quality.

### Online Inference Pipeline

Rather than feeding entire videos at once, PhoStream simulates streaming by sending 1-second frame updates to the model within a **60-second sliding context window** (video) while retaining full dialogue history. The model must autonomously decide to respond or emit "Silent" at each timestep. This design directly tests the temporal decision boundary — a capability invisible in offline benchmarks.

### Automated Generative Pipeline with Human Verification

Scaling QA generation uses a multi-stage pipeline: (1) re-encode video to H.265 at 2 FPS, (2) segment into <30MB chunks, (3) generate scene summaries and step-wise scripts via a strong MLLM, (4) produce QA candidates from prompt templates with timestamp annotations, (5) automated cutoff verification to prevent future-information leakage, (6) two rounds of human expert review. This yields high-density annotations (9.6 questions/video) with verified temporal grounding.

## Step-by-Step Workflow

### Building a Streaming Benchmark

1. **Curate source videos across distinct scenarios.** Define on-screen scenarios (e.g., app tutorials, screen recordings) and off-screen scenarios (e.g., vlogs, egocentric footage). Aim for diverse capabilities: action recognition, audio-visual integration, OCR, causal reasoning, UI navigation, social/emotional recognition.

2. **Re-encode and normalize video streams.** Transcode all videos to a consistent codec (H.265/HEVC) at a canonical frame rate (2 FPS for annotation, preserving originals for timestamp mapping). Segment long videos into chunks under 30MB for API compatibility using a tool like MP4Box.

3. **Generate scene-level metadata.** Feed each video segment to a strong multimodal model to produce: (a) a scene summary describing key events, (b) a step-wise script with per-second descriptions of visual and audio content, (c) entity/object inventories per scene.

4. **Produce temporally-annotated QA candidates.** Using the scene metadata, generate open-ended QA pairs with three required fields per pair:
   - `question`: natural-language question
   - `timestamp_query`: the simulated moment the question is asked
   - `timestamp_proactive`: (Forward tasks only) the earliest moment the answer becomes available
   - `temporal_type`: one of `backward`, `instant`, `forward`
   - `capability`: which of the 10 capability categories it tests
   - `reference_answer`: gold-standard answer text

5. **Run automated cutoff verification.** For each QA pair, confirm that the reference answer cannot be derived from video content *before* the designated timestamp. Filter any pair where the model could answer correctly using only pre-timestamp frames. In the PhoStream pipeline, this step removed ~3.6% of candidates.

6. **Conduct human expert review in two rounds.** Round 1: remove ambiguous, unanswerable, or culturally inappropriate items. Round 2: verify timestamp accuracy and revise temporal labels. Target retention rate: ~92% of automated candidates.

7. **Implement the Online Inference Pipeline.** Build a streaming simulator that:
   - Sends frame updates every 1 second to the model under evaluation
   - Maintains a 60-second sliding window of video context
   - Preserves full dialogue/query history
   - Issues each question at its `timestamp_query`
   - Accepts either a substantive response or a "Silent" token at each timestep
   - For Forward tasks, opens a 2-second response window starting at `timestamp_proactive`

8. **Define the LLM-as-a-Judge scoring rubric.** Use a 0–5 integer scale:
   - **5**: Fully accurate and complete
   - **4**: Correct core reasoning, may omit non-essential details
   - **3**: Partially relevant, weakens core causal link
   - **2**: Tangential or speculative
   - **1**: Factually incorrect
   - **0**: No attempt, off-topic, or placeholder response
   Multiply by 20 for 0–100 reporting. Filter placeholder responses (e.g., "got it", "noted", "understood") before scoring — maintain a list of 40+ standardized non-informative patterns.

9. **Compute temporal diagnostic metrics.** Beyond raw accuracy, calculate:
   - **Early Response Rate (ERR)**: percentage of Forward tasks where the model responds before `timestamp_proactive`
   - **Silent Rate**: percentage of questions where the model never responds
   - **Per-temporal-type scores**: separate Backward, Instant, Forward averages to expose temporal asymmetry

10. **Generate the benchmark report.** Present results as a matrix of (model x temporal_type x capability), highlighting the Backward–Forward gap as the primary diagnostic of streaming readiness. Include ERR as a first-class metric alongside accuracy.

## Concrete Examples

**Example 1: Evaluating a video-language model on streaming screen recordings**

```
User: I have a multimodal model that's supposed to help users navigate phone
settings. I want to benchmark it on streaming screen recordings. How should
I set up the evaluation?

Approach:
1. Collect 50+ screen recording videos of settings navigation (Wi-Fi setup,
   Bluetooth pairing, accessibility configuration), each 5-15 minutes long.
2. Re-encode to H.265 at 2 FPS. Segment into <30MB chunks.
3. Generate scene scripts via a reference MLLM, capturing each UI transition,
   tap, scroll, and text displayed.
4. Produce QA pairs across temporal types:
   - Backward: "What setting did the user change before opening Bluetooth?"
     (timestamp_query: 4:30, answer derivable from 2:15 tap event)
   - Instant: "What toggle is currently highlighted on screen?"
     (timestamp_query: 6:12, answer visible in current frame)
   - Forward: "Will the user successfully connect to the Wi-Fi network?"
     (timestamp_query: 1:00, timestamp_proactive: 3:45 when connection
     confirmation appears)
5. Run online inference: stream frames at 1/sec with 60s sliding window.
   Record response timing and content.
6. Score with LLM-as-a-Judge. Compute per-type scores and ERR.

Output (example report):
| Temporal Type | Avg Score (0-100) | Early Response Rate |
|---------------|-------------------|---------------------|
| Backward      | 78.4              | N/A                 |
| Instant       | 82.1              | N/A                 |
| Forward       | 21.6              | 83.2%               |

Diagnosis: Model answers Forward questions prematurely 83% of the time,
guessing outcomes before UI confirmation appears. Recommend adding a
"wait for evidence" calibration step or confidence thresholding.
```

**Example 2: Building an automated QA generation pipeline for egocentric video**

```
User: I have 200 egocentric videos from wearable cameras. I need to generate
streaming QA pairs automatically. Can you help me build the pipeline?

Approach:
1. Preprocess videos:
   - ffmpeg -i input.mp4 -c:v hevc_nvenc -r 2 -segment_time 120
     -f segment chunk_%03d.mp4
2. For each chunk, call a multimodal API to generate structured metadata:
   {
     "scene_summary": "User walks through grocery store...",
     "timestamped_events": [
       {"t": 12, "event": "picks up cereal box, reads label"},
       {"t": 34, "event": "compares prices on shelf tags"},
       ...
     ]
   }
3. Apply QA generation templates per capability:
   - Action & Activity: "What is the person doing at [t]?"
   - Fine-grained Visual: "What brand is on the item they're holding?"
   - Audio-Visual Integration: "What did the cashier say when scanning?"
   - Forward/Proactive: "What will the person pick up next?"
     (set timestamp_query before the reach event, timestamp_proactive at
     the moment the item is visibly grasped)
4. Run cutoff verification: for each QA pair, query the reference model
   with only frames up to timestamp_query. If it answers correctly,
   the question leaks future info — discard it.
5. Output as JSONL:
   {"video_id": "ego_042", "question": "What brand of cereal...",
    "timestamp_query": 45.0, "timestamp_proactive": null,
    "temporal_type": "instant", "capability": "fine_grained_visual",
    "reference_answer": "Cheerios, visible on the yellow box label"}

Expected yield: ~8-10 QA pairs per video after filtering, ~1,600-2,000 total.
```

**Example 3: Diagnosing early-response bias in a production assistant**

```
User: Our voice assistant answers too quickly during video calls — it
interrupts before the other person finishes their point. How can I
measure and fix this?

Approach:
1. Frame this as a Forward-task temporal reasoning problem. The assistant
   must learn to emit "Silent" until sufficient evidence accumulates.
2. Construct a targeted Forward-task evaluation set:
   - Record 50 video call segments where a speaker makes a multi-sentence
     point (evidence arrives over 5-15 seconds)
   - Create questions like "What solution does the speaker propose?"
     with timestamp_query at sentence start, timestamp_proactive at
     the sentence where the proposal is fully stated
3. Run the online inference pipeline against your assistant:
   - Stream audio+video at 1-second intervals
   - Log response timing at each interval
4. Measure Early Response Rate and score quality of early vs. on-time answers.
5. Mitigation strategies based on findings:
   - If ERR > 70%: Add a confidence threshold — only respond when
     internal confidence exceeds a calibrated cutoff
   - If early responses are low-quality: Fine-tune with "Silent" token
     supervision on Forward-task training data
   - If early responses are high-quality but premature: Adjust response
     latency with a minimum-evidence-window parameter

Output:
ERR before fix: 84.3% (model interrupts 84% of the time)
ERR after confidence thresholding (0.7): 31.2%
Average response quality (0-100): 68.4 → 79.1 (on-time answers are better)
```

## Best Practices

**Do:**
- Always separate evaluation by temporal type (Backward/Instant/Forward). Aggregate scores hide the most critical failure mode (premature responding).
- Include Early Response Rate as a first-class metric alongside accuracy. A model that scores 80 overall but has 90% ERR on Forward tasks is not streaming-ready.
- Use a sliding context window (60 seconds) rather than feeding entire videos. This simulates real memory constraints and tests the model's ability to work with limited history.
- Filter placeholder/non-informative responses before scoring. Maintain a comprehensive list of filler patterns across languages.
- Verify temporal grounding with automated cutoff checks before human review — it catches ~3-5% of leaked QA pairs that humans may miss.

**Avoid:**
- Do not use multiple-choice formats for streaming benchmarks. They mask temporal reasoning failures because models can guess from options without evidence.
- Do not evaluate Forward tasks at `timestamp_query` — always evaluate at `timestamp_proactive` with a response window. Scoring at query time rewards premature answers.
- Do not assume high Backward/Instant scores imply streaming competence. The PhoStream finding shows these scores are uncorrelated with Forward performance.
- Do not skip the human verification rounds. Automated pipelines produce ~8% unsuitable candidates (ambiguous timestamps, culturally problematic content, unanswerable questions).

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Model never responds ("Silent" on everything) | Overly conservative response threshold | Lower confidence threshold; verify the model receives the query in its context window |
| QA pairs answerable before `timestamp_query` | Future-information leakage in generation | Re-run cutoff verification; tighten the automated filter to check 5 seconds before timestamp, not just at timestamp |
| LLM-as-a-Judge scores inconsistent across runs | Stochastic judge model | Use temperature 0 for judging; average across 3 judge calls; report inter-judge agreement |
| Video segmentation breaks mid-sentence/action | Fixed-size chunking ignores scene boundaries | Use scene-change detection (e.g., PySceneDetect) to find natural breakpoints before segmenting |
| Early Response Rate near 100% for all models | Questions may be trivially predictable | Review Forward questions — ensure they require specific visual/audio evidence, not common-sense guesses |
| Sliding window drops critical context | Important event occurred > 60 seconds ago | For Backward tasks spanning long ranges, extend the window or maintain a compressed event summary buffer |

## Limitations

- **Forward-task evaluation requires precise timestamp annotation.** If `timestamp_proactive` is off by even a few seconds, ERR measurements become unreliable. This is the most labor-intensive part of benchmark construction.
- **LLM-as-a-Judge has known biases** — it tends to favor verbose answers and may not penalize hallucinated details sufficiently. Cross-validate with human scoring on a subset.
- **The 60-second sliding window is a design choice, not a universal standard.** Real mobile assistants may have different memory architectures. Benchmark results may not transfer to models with longer or shorter effective context.
- **Audio evaluation is underrepresented** in current capability categories. Models strong on visual tasks may appear better overall even if audio comprehension is weak.
- **This methodology benchmarks comprehension timing, not generation latency.** A model may score well on temporal reasoning but still be too slow for real-time deployment due to inference speed.
- **Forward tasks are inherently harder to construct at scale** because they require identifying moments where a question can be asked before evidence appears — many natural questions don't have this structure.

## Reference

**Paper:** [PhoStream: Benchmarking Real-World Streaming for Omnimodal Assistants in Mobile Scenarios](https://arxiv.org/abs/2601.22575v1) (Lu et al., 2026)

**Key takeaway:** The Backward–Forward performance gap (80+ vs. ~16 on a 0–100 scale) and the Early Response Rate metric are the paper's most actionable contributions. When building or evaluating any streaming multimodal system, use Forward-task ERR as your primary diagnostic — it reveals whether your model knows *when* to speak, not just *what* to say.

**Code:** https://github.com/Lucky-Lance/PhoStream