---
name: "omni-rrm-advancing-omni-reward"
description: "Build rubric-grounded reward models and preference evaluation pipelines for multimodal AI outputs. Use when asked to 'evaluate model outputs with rubrics', 'build a preference dataset', 'score multimodal responses', 'create a reward model pipeline', 'judge which AI response is better', or 'rank model completions across dimensions'."
---

# Omni-RRM: Rubric-Grounded Multimodal Reward Modeling

This skill enables Claude to design and implement rubric-grounded preference evaluation systems for multimodal AI outputs (text, image, video, audio). Rather than producing opaque scalar scores, the approach from Omni-RRM generates structured, multi-dimension preference judgments with per-dimension justifications. Claude can apply this to build automated preference annotation pipelines, construct quality evaluation rubrics, implement Best-of-N candidate selection, and create training data for reward models -- all without requiring human-labeled preferences.

## When to Use

- When the user needs to evaluate or compare two or more AI-generated responses and determine which is better, with justification
- When building an automated preference dataset from model outputs of varying quality (e.g., contrasting a strong model against a weak model)
- When designing a scoring rubric for multimodal content (image captions, video descriptions, audio transcriptions, text completions)
- When implementing Best-of-N selection to pick the highest-quality candidate from multiple model generations
- When the user wants structured reward signals (not just a single number) for RLHF or DPO training pipelines
- When building a reconcile-and-filter pipeline where multiple judges annotate preferences independently, then disagreements are resolved programmatically

## Key Technique

**Rubric-Grounded Preference Synthesis.** The core insight of Omni-RRM is that reward modeling improves dramatically when you replace opaque scalar scores with structured rubric judgments across fixed dimensions. The system evaluates every response pair on five universal criteria -- fluency, relevance, accuracy, reasoning quality, and safety -- plus modality-specific grounding (visual fidelity for images, temporal consistency for video, content fidelity for audio). Each dimension gets a textual comparative justification, not just a number. This forces the evaluator to reason about *why* one response is better, which produces more reliable and interpretable preference labels.

**Automated Pipeline via Capability-Gap Pairing.** To generate training data without human annotators, the pipeline pairs a strong model with a weak model on identical prompts. The strong model's output is the likely-preferred candidate; the weak model's is the likely-rejected one. But labels are *not* assigned at this stage. Instead, two heterogeneous teacher models independently produce rubric-grounded judgments on each pair. Their outputs are reconciled: full agreement retains the pair with averaged scores; partial agreement merges justifications; verdict conflicts cause the pair to be discarded (with a small sample audited). Rule-based filters then remove duplicates, malformed outputs, score-verdict inconsistencies, and low-quality pairs. This yields a high-quality preference dataset (~41K examples in the paper) with no human labeling.

**Two-Stage Training with GRPO Refinement.** The reward model is trained in two stages. Stage 1 uses supervised fine-tuning (SFT with LoRA) to teach the model to produce the structured rubric-grounded JSON output format. Stage 2 applies Group Relative Policy Optimization (GRPO) on difficult, low-margin pairs where both responses are close in quality. The GRPO reward combines three weighted signals: preference accuracy (0.5), rubric quality (0.3), and format compliance (0.2). This sharpening step is critical -- it teaches the model to discriminate in ambiguous cases where naive judges fail.

## Step-by-Step Workflow

1. **Define rubric dimensions for your domain.** Start with the five universal criteria (fluency, relevance, accuracy, reasoning, safety) and add modality-specific dimensions. For image tasks, add visual grounding and spatial accuracy. For video, add temporal consistency and motion coherence. For audio, add content fidelity and speaker clarity. Each dimension needs a one-sentence definition.

2. **Design the structured output schema.** Define a JSON schema with these mandatory fields: `score_A` (integer 0-10), `score_B` (integer 0-10), `better` (enum: "A", "B", "equal"), `rubric` (object with one text justification per dimension), and `final_verdict` (explicit preference token). This schema is used for both annotation and model output.

3. **Generate candidate response pairs via capability-gap contrasting.** Select a strong model and a weak model for your domain. Feed identical prompts to both. Collect `(prompt, response_A, response_B)` triples. Do NOT assign preference labels at this stage -- that is the job of the teacher judges.

4. **Annotate pairs with multiple independent teacher judges.** Use at least two heterogeneous judge models (e.g., GPT-4o and Claude) to independently evaluate each pair against the rubric. Each judge outputs the full JSON schema: scores, verdict, and per-dimension justifications. Using heterogeneous judges reduces systematic bias.

5. **Reconcile and filter annotations.** Apply three-case reconciliation: (a) if both judges agree on verdict and scores are close, retain with averaged scores; (b) if verdicts agree but justifications differ, retain and merge justifications; (c) if verdicts conflict, discard the pair. Then apply rule-based filters to remove: duplicate/near-duplicate responses, empty or invalid content, score-verdict inconsistencies, malformed rubric fields, pairs where both responses fall below a quality floor, and low semantic-confidence pairs.

6. **Format the training dataset.** Structure each retained example as `(input_prompt, rubric_grounded_judgment_json)` where the judgment includes the full schema. Split into SFT training set (all pairs) and a hard-pairs subset (pairs where score difference <= 2) for the GRPO stage.

7. **Train Stage 1: Supervised Fine-Tuning.** Fine-tune your base model (e.g., Qwen2.5-VL-7B) with LoRA on the full dataset using standard next-token prediction loss on the rubric-grounded JSON output. This teaches the model the output format and basic preference discrimination.

8. **Train Stage 2: GRPO on hard pairs.** Fine-tune further using Group Relative Policy Optimization on the hard-pairs subset. Use a composite reward: preference accuracy (weight 0.5), rubric justification quality (weight 0.3), and format compliance (weight 0.2). This stage sharpens discrimination on ambiguous cases.

9. **Deploy for Best-of-N selection.** At inference time, generate N candidates (typically N=5) from your base generator for each query. Use the trained reward model to perform pairwise preference comparisons among candidates. Return the most-preferred candidate along with its rubric-grounded justification.

10. **Validate with human audit.** Sample 5% of discarded conflict pairs and a random subset of retained pairs for human review. Compare human verdicts against the pipeline's labels to measure annotation reliability and calibrate your filtering thresholds.

## Concrete Examples

**Example 1: Building a preference evaluation rubric for image captioning**

User: "I need to evaluate image captioning model outputs. Help me build a structured evaluation system."

Approach:
1. Define rubric dimensions: fluency (grammatical quality), relevance (caption addresses image content), accuracy (factual correctness of described objects/relations), visual grounding (references correspond to actual image regions), and safety (no harmful content).
2. Create the output schema:

```json
{
  "score_A": 7,
  "score_B": 4,
  "better": "A",
  "rubric": {
    "fluency": "Response A uses natural, well-formed sentences. Response B has a dangling modifier in the second clause.",
    "relevance": "Both responses address the main subject (a dog in a park). Response A also describes background elements visible in the image.",
    "accuracy": "Response A correctly identifies the breed as a golden retriever. Response B incorrectly calls it a labrador.",
    "visual_grounding": "Response A references the red frisbee in the dog's mouth, which is clearly visible. Response B mentions a ball that is not present.",
    "safety": "Neither response contains harmful content."
  },
  "final_verdict": "A"
}
```

3. Pair a strong captioner (e.g., GPT-4V) against a weak one (e.g., BLIP-2) on the same images.
4. Run two independent judge models to produce rubric judgments, reconcile, and filter.

**Example 2: Implementing Best-of-N selection for a video QA system**

User: "My video QA model sometimes gives mediocre answers. Can I use reward modeling to pick the best one?"

Approach:
1. Generate N=5 candidate answers per video-question pair from the base model (temperature sampling).
2. Define video-specific rubric: fluency, relevance, accuracy, temporal_consistency ("Does the answer reflect events in correct temporal order?"), reasoning ("Does the answer demonstrate understanding of causal relationships in the video?").
3. Use a rubric-grounded judge to perform pairwise comparisons among the 5 candidates.
4. Select the candidate that wins the most pairwise comparisons.

```python
import itertools

def best_of_n_select(candidates, judge_fn, prompt, rubric_dims):
    """Select best candidate via pairwise rubric-grounded comparison."""
    wins = {i: 0 for i in range(len(candidates))}

    for i, j in itertools.combinations(range(len(candidates)), 2):
        judgment = judge_fn(
            prompt=prompt,
            response_a=candidates[i],
            response_b=candidates[j],
            rubric_dimensions=rubric_dims
        )
        if judgment["better"] == "A":
            wins[i] += 1
        elif judgment["better"] == "B":
            wins[j] += 1

    best_idx = max(wins, key=wins.get)
    return candidates[best_idx], wins
```

**Example 3: Automated preference dataset construction without human labels**

User: "I want to create a preference dataset for training a reward model, but I can't afford human annotators."

Approach:
1. Select capability-gap model pairs for your domain:
```python
GENERATOR_PAIRS = {
    "text": ("claude-3-opus", "claude-3-haiku"),
    "image_caption": ("gpt-4v", "llava-1.5-7b"),
    "video_qa": ("qwen2.5-vl-72b", "qwen2.5-vl-3b"),
}
```

2. Generate response pairs on your prompt set (same prompt to both models).
3. Send each pair to two heterogeneous teacher judges with the rubric prompt:

```
Evaluate the following two responses to the given query.
Score each on [0-10] and provide a comparative justification
for each rubric dimension: {dimensions}.
Output valid JSON with fields: score_A, score_B, better, rubric, final_verdict.
```

4. Reconcile:
```python
def reconcile(judge_a, judge_b):
    if judge_a["better"] == judge_b["better"]:
        if abs(judge_a["score_A"] - judge_b["score_A"]) <= 2:
            # Case I: full agreement -- average scores, keep either rubric
            return merge_scores(judge_a, judge_b), "retain"
        else:
            # Case II: verdict agrees, scores diverge -- merge justifications
            return merge_justifications(judge_a, judge_b), "retain"
    else:
        # Case III: verdict conflict -- discard
        return None, "discard"
```

5. Apply rule-based filters (duplicate detection, score-verdict consistency, quality floor).
6. Result: a clean preference dataset with rubric-grounded rationales, ready for SFT training.

## Best Practices

- **Do:** Use heterogeneous judge models (different model families) to reduce systematic annotation bias. Two judges from the same family will share blind spots.
- **Do:** Include a format compliance check in your pipeline. Malformed JSON outputs from judges should be retried once, then discarded. The paper uses format compliance as 20% of the GRPO reward signal.
- **Do:** Calibrate your quality floor threshold empirically. If both responses in a pair score below 3/10, the pair teaches the reward model nothing useful -- discard it.
- **Do:** Weight your GRPO composite reward toward preference accuracy (0.5) over rubric quality (0.3) and format (0.2). Getting the verdict right matters most.
- **Avoid:** Assigning preference labels during candidate generation. The capability gap creates a *likely* ordering, but the actual label must come from the rubric evaluation. Strong models sometimes produce worse responses than weak ones.
- **Avoid:** Using per-dimension numeric scores in the rubric output. The paper found that textual comparative justifications per dimension are more reliable than numeric sub-scores, which tend to cluster and lose discriminative power.

## Error Handling

- **Judge produces invalid JSON:** Retry the judge call once with an explicit format reminder. If it fails again, skip the pair. Track skip rates -- if they exceed 10%, your prompt template needs tightening.
- **Both judges disagree on most pairs (>40% conflict rate):** Your rubric dimensions may be ambiguous. Refine dimension definitions with concrete examples of what scores 2, 5, and 8 look like for each dimension.
- **Score-verdict inconsistency (e.g., score_A > score_B but better="B"):** This indicates the judge is confused. Discard these pairs. If the rate exceeds 15%, add an explicit instruction: "The `better` field must be consistent with the higher score."
- **Low discrimination on hard pairs after SFT:** This is expected -- SFT alone cannot distinguish close-quality pairs. The GRPO stage specifically targets this. If discrimination remains poor after GRPO, increase the hard-pairs subset by relaxing the score-difference threshold (e.g., from <= 2 to <= 3).
- **Reconciliation discards too many pairs (>50%):** Your teacher judges may be too different in capability. Use judges of similar strength but different architecture.

## Limitations

- The rubric-grounded approach adds latency compared to scalar reward models. Each evaluation requires generating a full JSON judgment with justifications, making it unsuitable for on-the-fly token-level reward signals during decoding.
- The five-dimension rubric (fluency, relevance, accuracy, reasoning, safety) is general-purpose. Highly specialized domains (e.g., medical imaging, legal text) may need domain-specific dimensions that require expert design.
- Capability-gap pairing assumes you have access to both a strong and weak model. If your domain only has one model, you must create the capability gap artificially (e.g., via temperature variation or truncated outputs), which produces less natural contrast.
- The reconcile-and-filter pipeline discards conflicted pairs, which may systematically exclude the hardest, most informative examples. The 5% human audit mitigates this but does not eliminate the bias.
- Best-of-N selection with pairwise comparison scales as O(N^2). For large N, consider tournament-style brackets or top-k filtering before full pairwise evaluation.

## Reference

**Paper:** [Omni-RRM: Advancing Omni Reward Modeling via Automatic Rubric-Grounded Preference Synthesis](https://arxiv.org/abs/2602.00846v1) (Kong et al., 2026). Focus on Section 3 (pipeline design), Section 4 (two-stage training), and Appendix A.5 (prompt templates) for implementation details.