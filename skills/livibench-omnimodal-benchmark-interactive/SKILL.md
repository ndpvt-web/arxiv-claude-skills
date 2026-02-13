---
name: "livibench-omnimodal-benchmark-interactive"
description: "Build omnimodal benchmarks and evaluation pipelines for interactive video understanding (livestreams, real-time comments, multi-speaker audio). Applies LiViBench's multi-agent annotation workflow, seed-question-driven QA generation, and Video-to-Comment Retrieval (VCR) module design. Trigger phrases: 'build a livestream video benchmark', 'evaluate multimodal models on interactive video', 'create annotation pipeline for video QA', 'design a multi-agent video description system', 'implement comment retrieval for video understanding', 'build an omnimodal evaluation dataset'."
---

# LiViBench: Omnimodal Benchmark Construction for Interactive Video Understanding

This skill enables Claude to design and implement evaluation benchmarks, annotation pipelines, and retrieval-augmented systems for interactive multimodal video understanding. It codifies the methodology from LiViBench (AAAI 2026), which introduced a semi-automatic, multi-agent annotation workflow for constructing high-quality video QA benchmarks across visual, audio, speech, and real-time comment modalities. The core techniques -- multi-agent video description, seed-question-driven annotation, and embedding-based comment retrieval -- generalize to any project requiring structured evaluation of multimodal or interactive content.

## When to Use

- When building an evaluation benchmark for video, livestream, or multimodal content understanding
- When designing a semi-automatic annotation pipeline that combines LLM generation with human review
- When implementing a multi-agent system where specialized models describe different aspects of the same content
- When constructing a QA dataset from video or multimedia sources using seed-question templates
- When building a retrieval module to select relevant real-time comments (danmaku/barrage) or chat messages for a given video segment
- When evaluating MLLMs on interactive or livestream video tasks across multiple modalities
- When designing a two-stage instruction-tuning curriculum for domain-specific video understanding

## Key Technique

**Multi-Agent Video Description.** Rather than relying on a single model to describe a video comprehensively, LiViBench assigns four specialized roles to different MLLMs: (1) a scene analysis expert focusing on visual style, composition, and background; (2) a detail expert covering appearance, talent actions, and interactive behavior; (3) a logic/event expert handling temporal sequences, causality, and spatial reasoning; (4) a knowledge expert for cultural references, trends, and media symbols. Each agent receives role-specific instructions and contributes a complementary description. This ensemble approach reduces single-model bias and produces richer, more complete annotations than any individual model achieves alone.

**Seed-Question-Driven Annotation.** Instead of freeform question generation (which produces inconsistent difficulty and coverage), the method defines a library of task-specific seed question templates organized into five categories: coarse-grained perception (4 tasks), fine-grained perception (6 tasks), knowledge-based reasoning (3 tasks), general reasoning (4 tasks), and livestream-specific challenges (7 tasks). Models generate candidate questions guided by these seeds, and human annotators filter, refine, and verify at every stage. This produces 24 distinct task types with controlled quality and balanced coverage.

**Video-to-Comment Retrieval (VCR).** Interactive livestreams generate massive volumes of real-time comments (the paper's dataset contains ~1.45M comments). Naively including all comments overflows context windows and degrades model performance. VCR solves this by: (1) uniformly sampling video frames, (2) encoding frames and comments into a shared embedding space (using CLIP-style encoders), (3) computing frame-comment similarity to retrieve the top-k most relevant comments per frame, and (4) sorting retrieved comments chronologically before concatenation with the query. This selective retrieval consistently outperforms both no-comment and all-comment baselines.

## Step-by-Step Workflow

### A. Building a Multimodal Video Benchmark

1. **Collect and filter source videos.** Score candidate videos on spatiotemporal complexity (scene dynamics, content depth, narrative structure, cognitive load) on a 1-10 scale per dimension. Exclude videos scoring below a threshold (e.g., average < 3) to ensure sufficient complexity for meaningful evaluation.

2. **Design the task taxonomy.** Define 5 category groups with specific task types:
   - *Coarse-grained perception:* video topic, scene description, overall sentiment, content summarization
   - *Fine-grained perception:* attribute recognition, object counting, temporal querying, spatial relationship, action recognition, text/overlay reading
   - *Knowledge-based reasoning:* cultural reference identification, domain expertise, trend recognition
   - *General reasoning:* causal reasoning, counterfactual reasoning, multi-step inference, comparative reasoning
   - *Domain-specific:* (for livestreams: gift acknowledgment, audience interaction, real-time comment understanding, host behavior, multi-person interaction, engagement pattern, comment sentiment)

3. **Generate multi-agent descriptions.** Assign 3-4 specialized models or prompts to describe each video from distinct perspectives (scene composition, fine detail, temporal logic, domain knowledge). Provide each agent with explicit role instructions specifying its focus dimensions. Aggregate descriptions into a comprehensive video profile.

4. **Create seed question templates.** For each task type, write 3-5 template questions that define the expected question structure, difficulty level, and answer format. Examples:
   - Attribute recognition: "What is the [attribute] of the [entity] visible at [timestamp]?"
   - Causal reasoning: "Why does [event A] lead to [event B] in this video?"
   - Comment understanding: "What is the audience's dominant reaction to [event]?"

5. **Generate candidate QA pairs.** Feed multi-agent descriptions and seed templates to an LLM to produce candidate questions, correct answers, and plausible distractors for each video. Generate 3-5 candidates per task type per video.

6. **Human-in-the-loop filtering.** At each stage (video filtering, description review, question curation, answer verification), route outputs through human annotators who remove ambiguous items, correct errors, and ensure quality. Track inter-annotator agreement on a sample to calibrate quality thresholds.

7. **Assemble and validate the benchmark.** Compile the final dataset with metadata (video ID, modality availability, task category, difficulty). Run baseline evaluations across multiple MLLMs to verify task discrimination and establish reference scores.

### B. Implementing Video-to-Comment Retrieval (VCR)

8. **Encode video frames.** Uniformly sample N frames from the video. Pass each through a visual encoder (e.g., CLIP ViT) to produce frame embeddings.

9. **Encode comments.** Tokenize and embed all real-time comments using the corresponding text encoder from the same CLIP model family. Store as a searchable embedding index.

10. **Retrieve and rank.** For each sampled frame, compute cosine similarity against all comment embeddings. Select the top-k comments per frame. Deduplicate across frames, sort chronologically, and concatenate as the comment context provided to the downstream model alongside the user's question.

## Concrete Examples

**Example 1: Building a Livestream QA Benchmark**

```
User: I have 5,000 livestream clips with audio and real-time chat messages.
I want to build an evaluation benchmark for multimodal models. Help me
design the pipeline.

Approach:
1. Score each clip for spatiotemporal complexity using an MLLM:
   - Prompt: "Rate this video on content depth (1-10), scene dynamics (1-10),
     narrative structure (1-10), cognitive load (1-10). Return JSON."
   - Filter out clips with average score < 3.

2. Define the task taxonomy (adapt the 5-group structure above to your domain).
   For gaming livestreams, replace "gift acknowledgment" with
   "gameplay event recognition"; for e-commerce, add "product demonstration QA".

3. For each retained clip, run 4 specialized description prompts:
   - Scene agent: "Describe the visual scene, camera angles, lighting, set design."
   - Detail agent: "Describe all visible text overlays, UI elements, and
     participant actions frame by frame."
   - Logic agent: "Describe the sequence of events, cause-and-effect, and
     temporal transitions."
   - Domain agent: "Identify cultural references, memes, trending topics,
     or domain-specific knowledge shown."

4. Feed aggregated descriptions + seed templates into an LLM to generate
   multiple-choice questions. Each question targets exactly one task type.

5. Route generated QA through human review. Discard items where annotators
   disagree on the correct answer.

Output structure:
{
  "video_id": "clip_0042",
  "modalities": ["video", "audio", "speech_transcript", "chat_messages"],
  "task_category": "fine_grained_perception",
  "task_type": "attribute_recognition",
  "question": "What color is the jersey worn by the host at 00:32?",
  "options": ["A. Red", "B. Blue", "C. White", "D. Green"],
  "answer": "B",
  "difficulty": "easy",
  "requires_modality": ["video"]
}
```

**Example 2: Implementing a Comment Retrieval Module**

```
User: My livestream analysis model is choking on 10,000+ real-time comments
per video. How do I select only the relevant ones?

Approach:
1. Install and load a multilingual CLIP model (e.g., chinese-clip for
   Chinese content, or openai/clip-vit-large-patch14 for English).

2. Sample frames uniformly (e.g., 1 frame per 2 seconds for a 5-min video
   = 150 frames).

3. Encode all frames and all comments into the shared embedding space.

4. For each frame embedding, compute cosine similarity with every comment
   embedding. Retrieve the top-5 comments per frame.

5. Deduplicate, sort by original timestamp, and format:

Code sketch (Python):
```python
import torch
from transformers import CLIPModel, CLIPProcessor

model = CLIPModel.from_pretrained("openai/clip-vit-large-patch14")
processor = CLIPProcessor.from_pretrained("openai/clip-vit-large-patch14")

def retrieve_comments(frames, comments, top_k=5):
    # Encode frames
    img_inputs = processor(images=frames, return_tensors="pt", padding=True)
    img_embeds = model.get_image_features(**img_inputs)
    img_embeds = img_embeds / img_embeds.norm(dim=-1, keepdim=True)

    # Encode comments
    txt_inputs = processor(text=[c["text"] for c in comments],
                           return_tensors="pt", padding=True, truncation=True)
    txt_embeds = model.get_text_features(**txt_inputs)
    txt_embeds = txt_embeds / txt_embeds.norm(dim=-1, keepdim=True)

    # Retrieve top-k per frame
    similarity = img_embeds @ txt_embeds.T  # (num_frames, num_comments)
    topk_indices = similarity.topk(top_k, dim=-1).indices

    # Deduplicate and sort chronologically
    selected = set()
    for frame_indices in topk_indices:
        selected.update(frame_indices.tolist())
    retrieved = sorted(selected, key=lambda i: comments[i]["timestamp"])
    return [comments[i] for i in retrieved]
```

Output: A filtered list of ~200-500 comments (instead of 10,000+) that are
visually relevant to the video content, sorted by timestamp, ready to
concatenate as context for your model.
```

**Example 3: Multi-Agent Video Description System**

```
User: I want to generate rich descriptions of cooking tutorial videos using
multiple models to cover different aspects.

Approach:
1. Define 4 agent roles with explicit instructions:

   SCENE_AGENT_PROMPT = """You are a scene analysis expert. Describe:
   - Kitchen layout, lighting, camera angle
   - Background elements and set design
   - Overall visual style and production quality
   Do NOT describe facial features. Focus on environment and composition."""

   DETAIL_AGENT_PROMPT = """You are a detail analysis expert. Describe:
   - All ingredients visible and their states (chopped, whole, cooked)
   - Kitchen tools and equipment being used
   - Text overlays, recipe cards, or on-screen graphics
   - Hand movements and cooking techniques shown."""

   LOGIC_AGENT_PROMPT = """You are a temporal logic expert. Describe:
   - The step-by-step sequence of cooking actions
   - Cause-and-effect (e.g., "added salt, then tasted and added more")
   - Transitions between preparation stages
   - Any mistakes or corrections made."""

   KNOWLEDGE_AGENT_PROMPT = """You are a culinary knowledge expert. Identify:
   - Cooking techniques by name (julienne, deglaze, temper)
   - Cultural origin of the dish
   - Ingredient substitution possibilities
   - Food safety considerations shown or missing."""

2. Run each prompt against the video (or extracted frames + transcript).
3. Concatenate all 4 descriptions into a unified video profile.
4. Use the profile as context for downstream QA generation.

Output: A structured multi-perspective description document per video,
typically 400-800 words, covering visual, procedural, and knowledge aspects.
```

## Best Practices

- **Do:** Assign each agent a distinct, non-overlapping focus dimension. Overlap between agents produces redundancy without improving coverage.
- **Do:** Include human review checkpoints at every pipeline stage (filtering, description, question generation, answer verification). Fully automated pipelines accumulate errors that compound downstream.
- **Do:** Use seed question templates to control task diversity and difficulty. Without templates, LLMs gravitate toward easy, superficial questions.
- **Do:** When implementing VCR, tune top-k per frame empirically. Too few comments lose signal; too many reintroduce noise. Start with k=5 and validate on a held-out set.
- **Avoid:** Feeding raw, unfiltered real-time comments directly into model context. High-volume comment streams overwhelm context windows and consistently degrade performance compared to selective retrieval.
- **Avoid:** Using a single model for both video description and QA generation. The same model's blind spots propagate into both the context and the questions, creating systematically unanswerable items.
- **Avoid:** Generating distractors randomly. Plausible distractors should be semantically close to the correct answer (same category, similar attributes) to test genuine understanding rather than surface-level elimination.

## Error Handling

| Problem | Cause | Solution |
|---|---|---|
| Low inter-annotator agreement on QA items | Ambiguous question or subjective answer | Rewrite questions with explicit temporal/spatial grounding; add "at timestamp X" or "in the left half of frame" |
| VCR retrieves irrelevant comments | CLIP embedding space misalignment for domain-specific content | Fine-tune the CLIP model on domain comment-frame pairs, or fall back to keyword/timestamp proximity matching |
| Agent descriptions contradict each other | Different models hallucinate conflicting details | Add a reconciliation step: feed all descriptions to a judge model that flags contradictions for human resolution |
| Generated questions are too easy | Seed templates lack difficulty controls | Add difficulty tiers to templates (easy: direct observation, medium: single-step inference, hard: multi-step reasoning across modalities) |
| Comment encoding fails for non-Latin scripts | Tokenizer mismatch | Use a multilingual CLIP variant (e.g., Chinese-CLIP for CJK content, multilingual-CLIP for mixed-language streams) |

## Limitations

- **Modality requirements.** The full pipeline assumes access to video frames, audio, speech transcripts, and timestamped comments. Projects with only video+text will benefit from the multi-agent and seed-question techniques but cannot use VCR.
- **CLIP alignment ceiling.** VCR relies on visual-textual alignment quality. For highly domain-specific content (medical procedures, industrial processes), off-the-shelf CLIP models may retrieve poorly without fine-tuning.
- **Annotation cost.** While semi-automatic, the pipeline still requires human annotators at every stage. For benchmarks exceeding ~5,000 items, plan for significant annotation labor.
- **Language dependency.** Seed question templates and agent prompts must be authored per language. The templates do not automatically transfer across languages.
- **Interactive-only design.** The task taxonomy is optimized for interactive content with audience participation. For passive video (movies, surveillance), the livestream-specific category (7 of 24 tasks) will not apply.

## Reference

**Paper:** [LiViBench: An Omnimodal Benchmark for Interactive Livestream Video Understanding](https://arxiv.org/abs/2601.15016v1) (AAAI 2026). Look for: Section 3 (benchmark construction pipeline), Section 4 (multi-agent description and seed-question method), Section 5 (VCR module architecture), and Table 4 (ablation showing VCR impact across comment density levels).