---
name: "omnirag-agent-agentic-omnimodal-reasoning"
description: "Build agentic multimodal RAG pipelines that answer questions over long audio-video content under resource constraints. Uses retrieval banks, a plan-retrieve-merge agent loop, and reinforcement learning to optimize tool use. Triggers: 'build a video QA pipeline', 'multimodal retrieval agent', 'answer questions from a long video', 'audio-video search agent', 'agentic RAG for multimedia', 'low-resource video understanding'"
---

# OmniRAG-Agent: Agentic Omnimodal Reasoning for Long Audio-Video QA

This skill teaches Claude to design and implement agentic retrieval-augmented generation (RAG) systems for answering questions over long multimodal content (video, audio, images, text). The core technique from the OmniRAG-Agent paper replaces expensive dense encoding of entire media streams with a three-stage pipeline: (1) build indexed retrieval banks from sampled frames and ASR-transcribed audio segments, (2) run a multi-turn agent loop that plans what evidence is needed, calls retrieval tools, and merges results, and (3) optionally apply Group Relative Policy Optimization (GRPO) to jointly improve tool-calling validity and answer correctness.

## When to Use

- When the user asks to build a system that answers natural-language questions about long videos or audio recordings
- When designing a multimodal RAG pipeline that must retrieve specific frames or audio clips rather than process the entire stream
- When implementing an agentic loop with Thought-Action-Observation reasoning over multimedia content
- When the user needs to handle video QA under compute/token budget constraints (low-resource settings)
- When building a tool-calling agent that selectively queries image and audio retrieval banks
- When applying reinforcement learning (GRPO) to improve an agent's retrieval strategy and answer quality together
- When the user wants to implement CLIP-based cross-modal retrieval with FAISS indexing for video frames or audio transcripts

## Key Technique

**Retrieval Bank Construction.** Instead of encoding an entire long video into a single dense representation (which is expensive and loses fine-grained detail), OmniRAG-Agent samples frames at fixed intervals (every delta seconds) into an image bank and runs ASR on the audio track to produce timestamped transcript segments for an audio bank. Frames are encoded with CLIP's image encoder; audio transcripts and queries are encoded with CLIP's text encoder. Both banks are indexed with FAISS for fast nearest-neighbor lookup. This converts the problem from "understand a 2-hour video" to "search a database of short, localized evidence snippets."

**Multi-Turn Agent Loop.** The agent follows a Thought-Action-Observation cycle for up to T turns. At each turn it generates: (1) a planning trace z_t describing what evidence is still needed, (2) a natural-language retrieval query q_t targeting either the image or audio bank, and (3) a continuation decision c_t. Retrieved evidence E_t is appended to a running history, which is compressed via temporal summarization to stay within context limits. The agent keeps retrieving until it has enough evidence to answer, then emits a final answer. This iterative approach handles complex queries that require correlating visual and auditory information from different parts of the video.

**GRPO for Joint Optimization.** A gated reward function combines format correctness (did the agent produce valid tool calls?) with answer correctness (did it get the right answer?). Format reward R_fmt must exceed 0.5 before performance reward R_perf is counted, ensuring the model first learns to use tools properly before optimizing for accuracy. This is trained with clipped policy gradients and a KL penalty against a reference policy, following the Group Relative Policy Optimization framework.

## Step-by-Step Workflow

1. **Sample and index video frames.** Extract frames from the input video at a fixed interval (e.g., every 2 seconds). Encode each frame with a CLIP image encoder (e.g., `openai/clip-vit-large-patch14`). Store embeddings in a FAISS index alongside metadata: `{segment_id, timestamp_start, timestamp_end, frame_path}`.

2. **Build the audio bank via ASR.** Run an ASR model (e.g., Whisper) on the video's audio track to produce timestamped transcript segments. Encode each transcript with the CLIP text encoder. Index these embeddings in a second FAISS index with metadata: `{segment_id, timestamp_start, timestamp_end, transcript_text, audio_path}`.

3. **Temporally downsample the raw video.** Create a compressed version of the video (lower resolution, reduced frame rate) as a coarse context signal X_tilde. This gives the agent a rough overview without consuming the full token budget.

4. **Initialize the agent state.** Compose the initial prompt from: the user's question Q, the downsampled video context X_tilde, and a system template instructing the agent to use `RetrieveIMG(query)` and `RetrieveAUD(query)` tools in a Thought-Action-Observation format.

5. **Run the agent loop (up to T turns).** At each turn, the LLM generates a thought (what evidence is needed), selects a tool and query, and receives retrieved results. Append the evidence to the interaction history. Apply temporal summarization to compress the running history and prevent context overflow.

6. **Aggregate frame-level hits into segments.** When image retrieval returns multiple frames from the same video segment, group them by `segment_id` to reduce redundancy and improve temporal coverage. Return the top-K segments rather than top-K individual frames.

7. **Merge evidence and generate the answer.** Once the agent decides to stop (continuation flag = stop, or max turns reached), pass the full accumulated evidence history to the LLM to produce the final answer.

8. **Define the gated reward function (for training).** Compute R_fmt as the fraction of required XML/tool-call format tags matched (capped at 1.0). Compute R_perf as exact-match against ground truth. Gate: if R_fmt < 0.5, ignore R_perf entirely (reward = -1 + R_fmt). Otherwise reward = -1 + R_fmt + R_perf.

9. **Train with GRPO.** Sample N trajectories per question. Compute standardized advantages across the group. Update the policy with clipped surrogate objective and KL divergence penalty against the reference (pre-training) policy.

10. **Evaluate and tune retrieval budget.** During training use K=3 retrieved items per query; at inference increase to K=5 for better coverage. Cap max turns at 20. Adjust the frame sampling interval delta based on video length and available compute.

## Concrete Examples

**Example 1: Building a Video QA System with Frame and Audio Retrieval**

User: "I have a collection of lecture videos (1-2 hours each). I want to build a system where users can ask questions and get answers grounded in specific video moments."

Approach:
1. Extract frames every 3 seconds using OpenCV; encode with CLIP and store in FAISS
2. Run Whisper on each video's audio; index ASR segments with CLIP text embeddings
3. Implement the agent loop with two tools:

```python
import faiss
import numpy as np
from transformers import CLIPModel, CLIPProcessor
import whisper

class OmniRAGAgent:
    def __init__(self, clip_model_name="openai/clip-vit-large-patch14",
                 frame_interval=3.0, max_turns=20, k_img=5, k_aud=5):
        self.clip = CLIPModel.from_pretrained(clip_model_name)
        self.processor = CLIPProcessor.from_pretrained(clip_model_name)
        self.whisper = whisper.load_model("base")
        self.frame_interval = frame_interval
        self.max_turns = max_turns
        self.k_img = k_img
        self.k_aud = k_aud
        self.img_index = None  # FAISS index
        self.aud_index = None  # FAISS index
        self.img_metadata = []
        self.aud_metadata = []

    def build_image_bank(self, video_path):
        """Sample frames at fixed intervals, encode with CLIP, index in FAISS."""
        frames = sample_frames(video_path, self.frame_interval)
        embeddings = []
        for i, (timestamp, frame) in enumerate(frames):
            inputs = self.processor(images=frame, return_tensors="pt")
            emb = self.clip.get_image_features(**inputs).detach().numpy()
            emb = emb / np.linalg.norm(emb)
            embeddings.append(emb)
            self.img_metadata.append({
                "segment_id": i, "timestamp": timestamp,
                "frame_path": f"frames/{i:06d}.jpg"
            })
        emb_matrix = np.vstack(embeddings).astype("float32")
        self.img_index = faiss.IndexFlatIP(emb_matrix.shape[1])
        self.img_index.add(emb_matrix)

    def build_audio_bank(self, video_path):
        """Run ASR, encode transcripts with CLIP text encoder, index in FAISS."""
        result = self.whisper.transcribe(video_path)
        embeddings = []
        for seg in result["segments"]:
            inputs = self.processor(text=seg["text"], return_tensors="pt",
                                    padding=True, truncation=True)
            emb = self.clip.get_text_features(**inputs).detach().numpy()
            emb = emb / np.linalg.norm(emb)
            embeddings.append(emb)
            self.aud_metadata.append({
                "segment_id": seg["id"],
                "start": seg["start"], "end": seg["end"],
                "transcript": seg["text"]
            })
        emb_matrix = np.vstack(embeddings).astype("float32")
        self.aud_index = faiss.IndexFlatIP(emb_matrix.shape[1])
        self.aud_index.add(emb_matrix)

    def retrieve_img(self, query, k=None):
        """Retrieve top-K image segments matching the query."""
        k = k or self.k_img
        inputs = self.processor(text=query, return_tensors="pt",
                                padding=True, truncation=True)
        q_emb = self.clip.get_text_features(**inputs).detach().numpy()
        q_emb = q_emb / np.linalg.norm(q_emb)
        scores, indices = self.img_index.search(q_emb.astype("float32"), k)
        return [
            {**self.img_metadata[i], "score": float(scores[0][j])}
            for j, i in enumerate(indices[0])
        ]

    def retrieve_aud(self, query, k=None):
        """Retrieve top-K audio segments matching the query."""
        k = k or self.k_aud
        inputs = self.processor(text=query, return_tensors="pt",
                                padding=True, truncation=True)
        q_emb = self.clip.get_text_features(**inputs).detach().numpy()
        q_emb = q_emb / np.linalg.norm(q_emb)
        scores, indices = self.aud_index.search(q_emb.astype("float32"), k)
        return [
            {**self.aud_metadata[i], "score": float(scores[0][j])}
            for j, i in enumerate(indices[0])
        ]
```

4. Wrap the retrieval tools in an agent loop that calls an LLM with a Thought-Action-Observation prompt template
5. The agent iterates: thinks about what to look for, calls `RetrieveIMG` or `RetrieveAUD`, reads results, decides if more evidence is needed

**Example 2: Implementing the Agent Loop with Thought-Action-Observation**

User: "Show me how to implement the multi-turn agent reasoning loop from the OmniRAG paper."

Approach:

```python
AGENT_TEMPLATE = """You are a multimodal QA agent. Answer the question using
RetrieveIMG(query) and RetrieveAUD(query) tools.

Format each turn as:
<thought>What evidence do I still need?</thought>
<action>ToolName(query text)</action>

When ready to answer:
<thought>I have enough evidence.</thought>
<answer>Your final answer</answer>

Question: {question}
Context: {context_summary}
History: {history}
"""

def run_agent_loop(agent, llm, question, max_turns=20):
    history = []
    summary = "Initial video overview loaded."

    for turn in range(max_turns):
        prompt = AGENT_TEMPLATE.format(
            question=question,
            context_summary=summary,
            history=format_history(history)
        )
        response = llm.generate(prompt)

        # Parse the structured output
        thought = extract_tag(response, "thought")
        action = extract_tag(response, "action")
        answer = extract_tag(response, "answer")

        if answer:
            return {"answer": answer, "evidence": history, "turns": turn + 1}

        # Execute the tool call
        tool_name, query = parse_action(action)
        if tool_name == "RetrieveIMG":
            evidence = agent.retrieve_img(query)
        elif tool_name == "RetrieveAUD":
            evidence = agent.retrieve_aud(query)
        else:
            evidence = {"error": f"Unknown tool: {tool_name}"}

        history.append({
            "turn": turn, "thought": thought,
            "action": action, "observation": evidence
        })

        # Temporal summarization to compress history
        if len(history) > 5:
            summary = summarize_history(llm, history)

    # Max turns reached; force final answer
    return {"answer": llm.generate(f"Based on evidence, answer: {question}\n{history}"),
            "evidence": history, "turns": max_turns}
```

**Example 3: Gated Reward Function for GRPO Training**

User: "How do I implement the reward function that jointly optimizes tool-calling format and answer quality?"

Approach:

```python
import re

REQUIRED_TAGS = ["<thought>", "</thought>", "<action>", "</action>"]

def compute_format_reward(trajectory_text):
    """R_fmt: fraction of required format tags present, capped at 1.0."""
    matches = sum(1 for tag in REQUIRED_TAGS if tag in trajectory_text)
    return min(1.0, 0.5 * matches)

def compute_perf_reward(predicted_answer, ground_truth):
    """R_perf: exact match indicator."""
    return 1.0 if predicted_answer.strip().lower() == ground_truth.strip().lower() else 0.0

def gated_reward(trajectory_text, predicted_answer, ground_truth):
    """Gated composition: format must pass threshold before perf counts."""
    r_fmt = compute_format_reward(trajectory_text)
    r_perf = compute_perf_reward(predicted_answer, ground_truth)
    if r_fmt >= 0.5:
        return -1.0 + r_fmt + r_perf
    else:
        return -1.0 + r_fmt

def compute_grpo_advantages(rewards):
    """Standardized advantages across a group of sampled trajectories."""
    mean_r = np.mean(rewards)
    std_r = np.std(rewards) + 1e-8
    return [(r - mean_r) / std_r for r in rewards]
```

## Best Practices

- **Do:** Sample frames at a consistent interval (2-5 seconds depending on content pace) so the image bank has uniform temporal coverage. Lecture content can use wider intervals; action-heavy content needs tighter sampling.
- **Do:** Aggregate frame-level retrieval hits into segment-level evidence by grouping on `segment_id` before returning to the agent. This reduces redundancy and gives the LLM coherent temporal context.
- **Do:** Apply temporal summarization to the agent's history after 4-5 turns to prevent context overflow. Compress earlier turns into a running summary while keeping the most recent turn in full.
- **Do:** Gate the reward function so format correctness is a prerequisite for answer credit. Without gating, the model may learn to skip tool calls and guess answers directly.
- **Avoid:** Encoding the entire video densely into the LLM context. The whole point of the retrieval bank approach is to avoid this -- the agent should only see sampled frames and retrieved snippets.
- **Avoid:** Using raw audio waveforms as retrieval keys. ASR-to-text followed by text embedding is more practical and enables cross-modal retrieval with the same CLIP text encoder used for queries.

## Error Handling

- **Empty retrieval results:** If a query returns no results above a similarity threshold, the agent should rephrase the query or switch modalities (try audio if image retrieval fails, and vice versa). Add a fallback in the agent template: "If retrieval returns no relevant results, reformulate your query with different keywords."
- **ASR failures on noisy audio:** Whisper may produce low-quality transcripts for music, overlapping speech, or non-speech audio. Log ASR confidence scores per segment and exclude segments below a threshold from the audio bank.
- **Agent stuck in loops:** If the agent issues the same or very similar queries across consecutive turns, detect this (cosine similarity > 0.95 between consecutive query embeddings) and force a continuation decision or inject a prompt nudge.
- **FAISS index out of memory:** For very long videos (10+ hours), use `IndexIVFFlat` with nlist clusters instead of `IndexFlatIP` to reduce memory. Trade exact search for approximate search.
- **Format parsing failures during GRPO:** If the LLM generates malformed tool calls, the format reward R_fmt will be low, gating out the performance reward. This is by design -- the model learns valid formatting before optimizing answers.

## Limitations

- Requires CLIP and ASR models as preprocessing dependencies, adding setup complexity compared to end-to-end approaches.
- Text-based audio retrieval via ASR loses non-speech audio information (music, sound effects, ambient sounds). Queries about non-verbal audio cues will fail.
- GRPO training requires multiple sampled trajectories per question (typically 8-16), which is compute-intensive even with a small base model.
- The fixed-interval frame sampling may miss brief but important visual events that fall between sample points. Adaptive keyframe detection could help but is not part of the core method.
- Performance gains from the agent loop and GRPO are modest (1-3% absolute) over simple RAG on some benchmarks -- the biggest win comes from adding retrieval banks in the first place.
- The system assumes questions can be answered from visual frames and speech transcripts. Content requiring understanding of spatial relationships, 3D scenes, or complex temporal dynamics may need richer representations.

## Reference

**Paper:** [OmniRAG-Agent: Agentic Omnimodal Reasoning for Low-Resource Long Audio-Video Question Answering](https://arxiv.org/abs/2602.03707v2) (Zhu et al., 2026). Focus on Section 3 for the full architecture (bank construction, agent loop, GRPO), Algorithm 1 in the appendix for pseudocode, and Tables 2-4 for ablation results showing the incremental contribution of RAG (+2.7%), agent loop (+0.8%), and RL (+1.2%).