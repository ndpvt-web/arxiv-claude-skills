---
name: "videothinker-building-agentic-videollms"
description: "Build agentic video understanding systems with LLM-guided tool reasoning. Implements the VideoThinker pattern: confidence-gated two-stage reasoning where an LLM dynamically invokes temporal retrieval, spatial zoom, and temporal zoom tools to answer questions about long videos. Use when: 'build a video QA agent', 'analyze long videos with tool use', 'implement agentic video understanding', 'create a VideoLLM pipeline', 'add temporal retrieval to video analysis', 'build confidence-gated video reasoning'."
---

# VideoThinker: Agentic Video Understanding with LLM-Guided Tool Reasoning

This skill enables Claude to help build agentic Video LLM systems that dynamically explore long videos using structured tool calls -- temporal retrieval, temporal zoom, and spatial zoom -- rather than relying on static uniform frame sampling. The core pattern from the VideoThinker paper is a two-stage confidence-gated architecture: first attempt direct reasoning on sampled frames, then fall back to multi-step tool-augmented exploration when confidence is low. The synthetic training pipeline (caption-space reasoning grounded back to video) provides a reusable blueprint for generating tool-use training data without requiring pre-existing long-video comprehension.

## When to Use

- When the user wants to build a video QA system that handles long-form videos (10+ minutes) where uniform frame sampling loses critical information
- When implementing an agentic pipeline where an LLM decides which video segments to examine, zoom into, or retrieve based on a query
- When designing a confidence-gated inference system that escalates from cheap direct reasoning to expensive tool-augmented reasoning
- When generating synthetic tool-use training data by reasoning over text captions then grounding trajectories back to multimodal inputs
- When adding temporal retrieval (clip search, subtitle search) or temporal/spatial zoom capabilities to an existing VideoLLM
- When building a two-stage VideoLLM that adaptively allocates compute based on question difficulty

## Key Technique

**The Circular Dependency Problem and Caption-Space Solution.** Building an agentic video model requires training data with tool-use trajectories (e.g., "retrieve clip at 12:30, zoom into frames 12:30-12:40, answer"). But generating such trajectories requires a model that already understands long videos -- a chicken-and-egg problem. VideoThinker breaks this cycle by converting videos into dense text captions, then having a powerful text-only LLM agent (Qwen3-235B) reason over captions while invoking tools. The tool calls reference timestamps and queries just as they would against real video, but the LLM only sees text. These caption-space trajectories are then grounded back to video by replacing `CaptionZoom` outputs with actual video frames at the referenced timestamps, producing interleaved video-and-tool-reasoning training samples.

**Two-Stage Confidence-Gated Inference.** At inference time, VideoThinker first attempts direct reasoning: sample 32 frames uniformly (short videos) or retrieve top-k clips (long videos), generate an answer, and compute a token-level confidence score. If confidence falls below a threshold (default 0.7), the system triggers Stage 2: multi-step tool-augmented reasoning where the model dynamically calls retrieval and zoom tools, accumulating up to 64 total frames across both stages. This design avoids wasting compute on easy questions while enabling deep exploration for hard ones.

**The Tool Suite.** Six tools cover three categories: (1) **Temporal Retrieval** -- `ClipRetrieval` (semantic video search via LanguageBind embeddings over 10-second segments), `SubtitleRetrieval` (transcript search via Whisper + embedding similarity), `SubtitleSummary` (query-focused transcript summarization); (2) **Temporal Zoom** -- `FrameZoom` (extract 8 resampled frames from a time interval), `SubtitleZoom` (get subtitles within an interval), `CaptionZoom` (FrameZoom + generate natural language caption); (3) **Spatial Zoom** is reserved for region-of-interest cropping. Each tool has typed parameters (`video_path`, `query`, `interval`, `topk`) making them straightforward to implement as Python functions.

## Step-by-Step Workflow

### Building the Agentic Video QA Pipeline

1. **Define the tool interface as typed Python functions.** Create a `VideoToolkit` class with methods for each tool: `clip_retrieval(video_path, query, topk=3)`, `subtitle_retrieval(video_path, query, topk=5)`, `subtitle_summary(video_path, query)`, `frame_zoom(video_path, start_sec, end_sec, num_frames=8)`, `subtitle_zoom(video_path, start_sec, end_sec)`, and `caption_zoom(video_path, start_sec, end_sec)`. Each returns a structured result with timestamps and content.

2. **Implement the video indexing backend.** Segment videos into 10-second clips. Encode each clip using a video-language encoder (e.g., LanguageBind-Video) to produce embeddings. Store embeddings in a vector index (FAISS or similar). Transcribe audio via Whisper and index subtitle segments with timestamps.

3. **Build the confidence scoring function.** After the model generates an answer sequence, compute confidence as the geometric mean of token probabilities: `gamma = exp(mean(log(p(token_t | context))))`. This requires access to logprobs from the model's forward pass.

4. **Implement Stage 1: Direct Reasoning.** For videos under 10 minutes, uniformly sample 32 frames. For longer videos, call `clip_retrieval` with the user query to get the top-k most relevant clips, then sample frames from those clips. Pass frames + question to the VideoLLM. Compute the confidence score on the generated answer.

5. **Implement Stage 2: Tool-Augmented Reasoning (triggered when confidence < threshold).** Format available tools as a system prompt with function signatures and descriptions. Let the LLM generate a reasoning chain interleaved with tool calls. Parse each tool call, execute it against the video backend, and inject results (frames or text) back into the context. Cap total frames at 64 across both stages.

6. **Build the tool-call parser and executor loop.** Implement a `run_agent_loop(model, video, question, tools, max_steps=5)` function that: (a) generates text until a tool call tag is emitted, (b) parses the tool name and arguments, (c) executes the tool, (d) appends the result to context, (e) repeats until the model emits a final answer or hits max steps.

7. **Wire up the two-stage gating logic.** Create the orchestrator: run Stage 1, check confidence against threshold tau (default 0.7), if below threshold run Stage 2, return the final answer. Log which stage produced the answer for monitoring.

### Generating Synthetic Tool-Use Training Data

8. **Generate dense captions for each video.** Run a VideoLLM (e.g., Qwen2.5-VL-7B) on the full video to produce segment-level captions with timestamps. Store as `{video_id: [{start, end, caption}, ...]}`.

9. **Generate tool-use trajectories in caption space.** For each QA pair, prompt a strong text LLM with: the question, the available tools (where `CaptionZoom` returns text captions instead of frames), and instruct it to reason step-by-step, calling tools as needed. Sample 5 trajectories at temperature 0.7. Filter to keep only trajectories whose final answer matches ground truth.

10. **Ground trajectories back to video.** For each `CaptionZoom` call in a trajectory, replace the text caption output with the actual video frames from that time interval (encoded as `<video>` tokens or frame paths). This produces interleaved video-text-tool training samples ready for supervised fine-tuning.

## Concrete Examples

**Example 1: Building a Long-Video QA Agent**

User: "I want to build a system that can answer questions about hour-long lecture videos. It should be able to find relevant segments and zoom in."

Approach:
1. Index the lecture video: segment into 10-second clips, encode with LanguageBind, transcribe with Whisper
2. Implement the tool suite as Python functions wrapping the index
3. Build the two-stage pipeline:

```python
class VideoThinkerAgent:
    def __init__(self, model, toolkit, tau=0.7, max_frames=64):
        self.model = model
        self.toolkit = toolkit
        self.tau = tau
        self.max_frames = max_frames

    def answer(self, video_path: str, question: str) -> str:
        # Stage 1: Direct reasoning
        clips = self.toolkit.clip_retrieval(video_path, question, topk=3)
        frames = []
        for clip in clips:
            frames.extend(self.toolkit.frame_zoom(
                video_path, clip["start"], clip["end"], num_frames=8
            ))
        response, confidence = self.model.generate_with_confidence(
            frames=frames[:32], question=question
        )
        if confidence >= self.tau:
            return response

        # Stage 2: Tool-augmented reasoning
        return self.run_tool_loop(video_path, question, frames_used=len(frames))

    def run_tool_loop(self, video_path, question, frames_used, max_steps=5):
        context = self.build_tool_prompt(question)
        for step in range(max_steps):
            output = self.model.generate(context, stop_at_tool_call=True)
            if output.has_final_answer:
                return output.answer
            tool_name, args = self.parse_tool_call(output.text)
            result = self.toolkit.execute(tool_name, video_path=video_path, **args)
            if isinstance(result, list) and all(is_frame(r) for r in result):
                frames_used += len(result)
                if frames_used > self.max_frames:
                    result = result[:self.max_frames - frames_used]
            context.append({"role": "tool", "content": result})
        return self.model.generate(context + [{"role": "user", "content": "Now give your final answer."}]).text
```

**Example 2: Generating Synthetic Training Data via Caption-Space Reasoning**

User: "I have 10k video QA pairs but no tool-use annotations. How do I create training data for an agentic VideoLLM?"

Approach:
1. Caption every video segment with a VideoLLM
2. Have a strong text LLM reason in caption space with tool calls
3. Ground back to video

```python
def generate_synthetic_trajectories(video_id, question, answer_gt, captions, llm):
    """Generate tool-use trajectories in caption space, then ground to video."""
    tool_prompt = f"""You are analyzing a video to answer: {question}
Available tools:
- CaptionZoom(start_sec, end_sec): Returns caption of video segment
- SubtitleZoom(start_sec, end_sec): Returns subtitles in interval
- ClipRetrieval(query, topk): Returns timestamps of relevant clips

Video duration: {captions[-1]['end']}s
Think step by step. Call tools to gather evidence, then answer."""

    trajectories = []
    for _ in range(5):  # sample 5 diverse trajectories
        traj = llm.generate(tool_prompt, temperature=0.7,
                            tool_executor=CaptionSpaceExecutor(captions))
        if traj.final_answer == answer_gt:
            trajectories.append(traj)

    if not trajectories:
        trajectories = [random.choice(all_5)]  # fallback

    # Ground: replace caption text with actual frames
    for traj in trajectories:
        for step in traj.tool_calls:
            if step.tool == "CaptionZoom":
                step.output = FrameTokens(video_id, step.args["start"], step.args["end"])
    return trajectories
```

**Example 3: Adding Confidence Gating to an Existing VideoLLM**

User: "My video model works fine on short clips but fails on 30-minute videos. How do I add adaptive reasoning?"

Approach:
1. Add logprob-based confidence scoring to the existing model
2. Implement the two-stage gate

```python
import torch
import math

def compute_confidence(logits: torch.Tensor, token_ids: torch.Tensor) -> float:
    """Geometric mean of token probabilities for the generated answer."""
    log_probs = torch.log_softmax(logits, dim=-1)
    selected = log_probs.gather(-1, token_ids.unsqueeze(-1)).squeeze(-1)
    return math.exp(selected.mean().item())

def confidence_gated_inference(model, video, question, toolkit, tau=0.7):
    # Stage 1: cheap pass with 32 uniformly sampled frames
    frames_s1 = uniform_sample(video, n=32)
    answer, logits, ids = model.generate_with_logits(frames_s1, question)
    conf = compute_confidence(logits, ids)

    if conf >= tau:
        return {"answer": answer, "stage": 1, "confidence": conf}

    # Stage 2: tool-augmented deep dive
    agent = VideoThinkerAgent(model, toolkit, tau=tau)
    answer_s2 = agent.run_tool_loop(video, question, frames_used=32)
    return {"answer": answer_s2, "stage": 2, "confidence": conf}
```

## Best Practices

- **Do** set the confidence threshold tau between 0.7 and 0.8. The paper shows performance degrades at tau=1.0 (always triggers Stage 2, wasting compute) and at tau=0.0 (never triggers, missing hard questions).
- **Do** cap total frames across both stages (64 is a good default). Without a cap, the agent loop can consume unbounded memory and context length.
- **Do** segment videos into fixed-length clips (10 seconds works well) for retrieval indexing. Shorter segments increase retrieval precision but reduce context per clip.
- **Avoid** using only uniform frame sampling for videos longer than 10 minutes. Information loss grows linearly with duration; retrieval-based sampling targets the relevant segments.
- **Avoid** training on failed trajectories. When generating synthetic data, filter to keep only trajectories that reach the correct answer. The paper samples 5 trajectories and retains correct ones, falling back to random selection only when all fail.
- **Do** implement `SubtitleRetrieval` alongside `ClipRetrieval` for videos with speech. Multimodal retrieval (visual + audio transcript) substantially improves temporal localization accuracy.

## Error Handling

- **Tool call parsing failures.** The model may generate malformed tool calls. Implement a regex-based parser with a fallback: if parsing fails, prompt the model to reformat its tool call. After 2 consecutive parse failures, force a direct answer.
- **Empty retrieval results.** `ClipRetrieval` may return no results for vague queries. Fall back to uniform frame sampling when retrieval returns zero hits, and log a warning.
- **Confidence score pathology.** Very short answers (1-2 tokens) produce unreliable confidence scores due to small sample size. Set a minimum answer length (e.g., 5 tokens) before trusting the confidence gate; below that threshold, always trigger Stage 2.
- **Frame budget exhaustion.** If the agent uses all 64 frames before reaching a conclusion, force it to produce a final answer from accumulated context rather than silently failing.
- **Timestamp out of bounds.** Tools may receive timestamps beyond video duration. Clamp all timestamp arguments to `[0, video_duration]` before execution.

## Limitations

- The two-stage architecture adds latency for hard questions. Stage 2 involves multiple sequential tool calls, each requiring model inference. This is unsuitable for real-time applications.
- Caption-space synthetic data generation requires a very strong text LLM (the paper uses a 235B-parameter MoE model). Smaller LLMs produce lower-quality trajectories that degrade training.
- The approach assumes videos can be meaningfully captioned segment-by-segment. For videos with heavy visual-only content (e.g., surveillance, abstract art), caption quality bottlenecks the entire pipeline.
- Spatial zoom (region cropping) is defined but underexplored in the paper. The primary gains come from temporal retrieval and temporal zoom.
- The confidence threshold tau is sensitive to the model and domain. A threshold tuned on one benchmark may not transfer; plan to calibrate on a held-out validation set per deployment.

## Reference

**Paper:** [VideoThinker: Building Agentic VideoLLMs with LLM-Guided Tool Reasoning](https://arxiv.org/abs/2601.15724v1) (Li et al., 2026). Focus on Section 3 (tool definitions and synthetic data pipeline) and Section 4.3 (confidence-gated two-stage inference) for implementation details.