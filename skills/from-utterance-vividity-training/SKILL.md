---
name: "from-utterance-vividity-training"
description: "Train expressive subtitle translation LLMs using Adaptive Local Preference Optimization (ALPO) — a segment-level preference alignment method that surpasses standard DPO for multi-line, domain-specific translation. Use when: 'build an expressive subtitle translator', 'train translation model with preference optimization', 'implement segment-level DPO for translation', 'align translation LLM for vividness', 'create subtitle parallel corpus pipeline', 'fine-tune LLM for liberal translation style'."
---

# Adaptive Local Preference Optimization for Expressive Translation

This skill enables Claude to implement ALPO (Adaptive Local Preference Optimization), a two-stage training pipeline for building domain-customized translation LLMs that produce expressive, vivid output — particularly for visual media subtitles. Unlike standard DPO which optimizes over entire outputs, ALPO applies process-supervised preference alignment at the individual segment (subtitle line) level, with adaptive weighting and dynamic temperature scaling. The method achieves state-of-the-art vividness scores that exceed both GPT-4o and human translators on subtitle benchmarks (ICLR 2026).

## When to Use

- When the user wants to train a translation LLM that prioritizes expressiveness and natural style over word-for-word fidelity
- When building a preference optimization pipeline for multi-line structured outputs (subtitles, dialogues, multi-turn text)
- When implementing segment-level or process-supervised DPO variants instead of outcome-level alignment
- When constructing preference pairs from LLM-as-judge scoring for translation quality
- When the user needs to fine-tune an open-source LLM (LLaMA, Qwen, GLM) for subtitle or creative translation
- When designing an evaluation framework that scores accuracy, naturalness, and vividness independently
- When building a multilingual subtitle parallel corpus from aligned video sources

## Key Technique

**The Problem with Standard DPO for Translation.** Direct Preference Optimization treats an entire generated response as a single unit — one "chosen" output vs. one "rejected" output. For subtitle translation, a single input contains many lines (often 10-50+), each requiring independent quality judgments. A translation might nail 9 out of 10 lines but fail on one. Outcome-level DPO cannot express this — it either rewards or penalizes the whole block, creating noisy gradients that wash out fine-grained signal.

**ALPO's Core Innovation.** Adaptive Local Preference Optimization decomposes preference alignment to the segment level. For each subtitle line `s_i`, ALPO independently samples candidate translations, scores them via an LLM evaluator, selects chosen/rejected pairs, and applies a modified Bradley-Terry loss with two critical adaptations: (1) an **adaptive gating function** that filters out low-diversity segments where preference signal is weak (fewer than 3 unique candidates or score variance below threshold), and (2) a **dynamic temperature** `β_i` that scales the loss by the normalized reward gap for each segment, so lines with clear quality differences get stronger gradient signal. The loss is:

```
L_alpo = -E[ Σ w(s_i) · log σ( β_i·log(π_θ(t_i^c|p_i)/π_ref(t_i^c|p_i)) - β_i·log(π_θ(t_i^r|p_i)/π_ref(t_i^r|p_i)) ) ]
```

**Exposure Bias Mitigation.** Since each line's translation is conditioned on preceding lines, ALPO uses a scheduled prefix mixing strategy: during training, the prefix for line `i` is the chosen translation with probability `λ` (ramping from 0.2 to 0.6) or a sampled alternative with probability `1-λ`. This prevents the model from only learning to continue from "perfect" prefixes and improves robustness at inference time.

## Step-by-Step Workflow

### Stage 1: Data Preparation and SFT

1. **Construct the parallel corpus.** Collect subtitle pairs across target language directions (e.g., en-zh, en-de, ko-zh). Align multi-line subtitle blocks using timestamp matching. Reserve ~80% for SFT training and ~20% for ALPO alignment. Deduplicate and filter segments shorter than 3 tokens.

2. **Run supervised fine-tuning (SFT).** Fine-tune the base LLM on the parallel corpus using standard cross-entropy loss. The input format should present source subtitles as a block with line breaks, instructing the model to produce an expressive, natural translation. Save this checkpoint as both the starting point for ALPO and the reference model `π_ref`.

3. **Validate the LLM evaluator.** Before using an LLM as reward model, verify its reliability: sample 200+ translation pairs across language directions, collect human preference judgments, and compute Spearman rank correlation. Target ρ ≥ 0.80. Use a capable evaluator model (e.g., Qwen3-14B or comparable) with a structured prompt scoring accuracy (meaning preservation), naturalness (target-language fluency), and vividness (emotional/atmospheric expressiveness) on a 0-100 scale.

### Stage 2: ALPO Preference Construction and Training

4. **Sample candidate translations per segment.** For each subtitle block `x` in the alignment set, iterate line by line. For line `i`, use the SFT model to sample `k=15` candidate translations, conditioned on the source line plus the prefix of previously chosen translations. Deduplicate candidates and optionally include the human reference translation.

5. **Score candidates with the LLM evaluator.** For each segment's candidate set, call the evaluator LLM to score vividness (the primary optimization target). Record full score vectors for adaptive weight computation.

6. **Select chosen/rejected pairs with margin control.** For each segment, select the chosen translation `t_i^c` randomly from the top-3 scored candidates (randomization prevents mode collapse). Select the rejected translation `t_i^r` as the third-lowest scored candidate — not the absolute worst, to avoid trivially easy contrasts that provide weak learning signal. Set the chosen translation as the prefix for the next segment.

7. **Compute adaptive weights and temperatures.** For each segment `s_i`:
   - **Gating**: Set `w(s_i) = 0` if the candidate set has ≤ 3 unique translations or score variance ≤ 5. Otherwise, set `w(s_i) = δ(s_i)` proportional to candidate diversity after deduplication.
   - **Temperature**: Compute `β_i = (r(s_i, t_i^c) - r(s_i, t_i^r)) / max_gap` where `max_gap` is the maximum reward gap across all segments in the batch.

8. **Train with ALPO loss and prefix mixing.** Apply the segment-level preference loss with the computed weights and temperatures. For each training step, construct the conditioning prefix using the scheduled mixing strategy (`λ` ramps from 0.2 to 0.6 over training). Use `π_ref` from the SFT checkpoint for the KL-divergence regularization terms.

9. **Evaluate on three dimensions.** Score the trained model's outputs using multiple strong LLM judges (e.g., 3 different models) on accuracy (0-100), naturalness (0-100), and vividness (0-100). Compare against SFT baseline, commercial LLMs, and human references. Expect vividness gains of +15-20 points over SFT with maintained or improved accuracy.

10. **Iterate on low-performing directions.** For language pairs with limited training data (e.g., ko-zh, zh-th), ALPO often outperforms larger commercial models because segment-level optimization is more data-efficient. If a direction underperforms, increase `k` (sample count) to improve preference pair quality, or reduce the gating variance threshold.

## Concrete Examples

**Example 1: Implementing ALPO Training Loop in PyTorch**

User: "I want to implement the ALPO loss function for training a subtitle translation model. I have an SFT-finetuned Qwen2.5-7B and scored preference pairs per subtitle line."

Approach:
1. Define the segment-level ALPO loss with adaptive weights and dynamic beta
2. Implement the gating function to filter low-signal segments
3. Add scheduled prefix mixing to the data collator
4. Wire into a standard training loop with the SFT model as frozen reference

Output:
```python
import torch
import torch.nn.functional as F

class ALPOLoss(torch.nn.Module):
    def __init__(self, min_candidates=3, min_variance=5.0):
        super().__init__()
        self.min_candidates = min_candidates
        self.min_variance = min_variance

    def compute_gate(self, num_unique_candidates, score_variance):
        """Gating function: filter low-diversity segments."""
        active = (num_unique_candidates > self.min_candidates) & (score_variance > self.min_variance)
        return active.float()

    def forward(self, chosen_logps, rejected_logps, ref_chosen_logps, ref_rejected_logps,
                reward_chosen, reward_rejected, num_unique, score_var):
        """
        All inputs shaped [batch, max_segments], with padding mask.
        reward_chosen/rejected: LLM evaluator scores per segment.
        """
        # Dynamic temperature per segment
        reward_gap = reward_chosen - reward_rejected  # [B, S]
        max_gap = reward_gap.max(dim=-1, keepdim=True).values.clamp(min=1e-6)
        beta = reward_gap / max_gap  # [B, S], normalized to [0, 1]

        # Log-ratio differences (standard DPO formulation, per segment)
        chosen_ratio = chosen_logps - ref_chosen_logps    # [B, S]
        rejected_ratio = rejected_logps - ref_rejected_logps  # [B, S]
        logits = beta * (chosen_ratio - rejected_ratio)

        # Gating weights
        gate = self.compute_gate(num_unique, score_var)  # [B, S]

        # Importance weight proportional to candidate diversity
        importance = num_unique.float() / num_unique.float().max(dim=-1, keepdim=True).values.clamp(min=1)
        weights = gate * importance  # [B, S]

        # Weighted segment-level loss
        segment_losses = -F.logsigmoid(logits)
        loss = (weights * segment_losses).sum() / weights.sum().clamp(min=1e-6)
        return loss
```

**Example 2: Building the Preference Pair Sampling Pipeline**

User: "How do I generate ALPO preference pairs from my SFT model and an LLM evaluator?"

Approach:
1. Iterate through subtitle blocks line-by-line
2. Sample k=15 candidates per line conditioned on the chosen prefix
3. Deduplicate and score with the evaluator
4. Select chosen (random top-3) and rejected (third-lowest) per segment

Output:
```python
import random
from collections import OrderedDict

def sample_alpo_pairs(sft_model, evaluator, subtitle_block, k=15):
    """
    subtitle_block: list of source subtitle lines [s_1, s_2, ..., s_n]
    Returns: list of dicts with chosen/rejected translations and scores per segment.
    """
    segments = []
    prefix_translations = []

    for i, source_line in enumerate(subtitle_block):
        # Build prompt with accumulated prefix
        context = build_translation_prompt(subtitle_block, prefix_translations, current_idx=i)

        # Sample k candidates from SFT model
        candidates = []
        for _ in range(k):
            translation = sft_model.generate(context, temperature=0.9, do_sample=True)
            candidates.append(translation)

        # Deduplicate while preserving order
        unique_candidates = list(OrderedDict.fromkeys(candidates))

        # Score each candidate via LLM evaluator (vividness dimension)
        scores = evaluator.score_translations(
            source=source_line,
            candidates=unique_candidates,
            criteria="vividness"
        )

        # Sort by score
        scored = sorted(zip(unique_candidates, scores), key=lambda x: x[1], reverse=True)

        # Select chosen: random from top-3
        top3 = scored[:min(3, len(scored))]
        chosen_trans, chosen_score = random.choice(top3)

        # Select rejected: third-lowest (not absolute worst)
        if len(scored) >= 3:
            rejected_trans, rejected_score = scored[-3]
        else:
            rejected_trans, rejected_score = scored[-1]

        segments.append({
            "source": source_line,
            "chosen": chosen_trans,
            "rejected": rejected_trans,
            "chosen_score": chosen_score,
            "rejected_score": rejected_score,
            "num_unique": len(unique_candidates),
            "score_variance": compute_variance([s for _, s in scored]),
        })

        # Use chosen as prefix for next segment
        prefix_translations.append(chosen_trans)

    return segments
```

**Example 3: Three-Dimensional Translation Evaluation**

User: "Set up an LLM-as-judge evaluation for my subtitle translations, scoring accuracy, naturalness, and vividness."

Approach:
1. Design a structured evaluation prompt with rubric definitions
2. Call multiple LLM judges for reliability
3. Aggregate scores per dimension

Output:
```python
EVAL_PROMPT = """You are evaluating a subtitle translation. Score on three dimensions (0-100 each):

**Accuracy**: Does the translation preserve the original meaning? Penalize omissions, additions, or distortions.
**Naturalness**: Does the translation read fluently in the target language? Penalize awkward phrasing, unnatural word order, or translationese.
**Vividness**: Does the translation expressively convey the emotions, tone, and atmosphere of the original? Reward creative word choices, idiomatic expressions, and stylistic flair that enhance the viewing experience.

Source: {source}
Translation: {translation}
Context (surrounding lines): {context}

Respond in JSON: {{"accuracy": <int>, "naturalness": <int>, "vividness": <int>, "reasoning": "<brief>"}}"""

def evaluate_translation(source, translation, context, judges):
    """Score a translation using multiple LLM judges."""
    all_scores = []
    for judge in judges:
        response = judge.generate(EVAL_PROMPT.format(
            source=source, translation=translation, context=context
        ))
        scores = parse_json_scores(response)
        all_scores.append(scores)

    # Average across judges
    return {
        dim: sum(s[dim] for s in all_scores) / len(all_scores)
        for dim in ["accuracy", "naturalness", "vividness"]
    }
```

## Best Practices

- **Do** validate your LLM evaluator against human judgments before using it for preference pair construction. Target Spearman ρ ≥ 0.80. An unreliable evaluator poisons the entire ALPO pipeline.
- **Do** select the rejected translation as the third-lowest rather than the absolute lowest score. Trivially bad rejections provide weak contrastive signal and can cause training instability.
- **Do** randomize the chosen translation from the top-3 rather than always picking the best. This prevents the model from overfitting to a single evaluator mode and improves diversity.
- **Do** ramp the prefix mixing parameter `λ` gradually (0.2 to 0.6) rather than using a fixed value. Starting low exposes the model to diverse prefixes; ending higher stabilizes convergence.
- **Avoid** applying ALPO to segments with fewer than 3 unique candidate translations or very low score variance — these provide near-zero preference signal and add noise.
- **Avoid** using outcome-level DPO as a drop-in replacement when your outputs are multi-segment. The core insight of ALPO is that segment-level decomposition captures fine-grained quality differences that outcome-level methods average away.

## Error Handling

- **Evaluator score collapse**: If the LLM evaluator assigns near-identical scores to all candidates for a segment, the gating function should filter it out (variance threshold). If this happens for >50% of segments, the evaluator prompt needs refinement or the evaluator model is too weak.
- **Candidate deduplication removes too many**: If sampling `k=15` yields fewer than 4 unique candidates, the SFT model's temperature is too low or the segment is trivially short. Increase temperature to 1.0-1.2 for sampling, or skip very short segments (< 3 source tokens).
- **Prefix error propagation**: A bad chosen translation in early segments can cascade into poor candidates for later segments. The scheduled prefix mixing mitigates this, but monitor per-position quality. If late-segment vividness drops significantly, increase the mixing ratio `λ` earlier.
- **Training divergence with high β**: If the dynamic temperature produces very large `β_i` values for segments with extreme reward gaps, gradient magnitude spikes. Clip `β_i` to a maximum of 2.0 as a safety measure.

## Limitations

- ALPO requires an LLM evaluator that reliably scores the target quality dimension (vividness). For domains where quality is harder to evaluate automatically (e.g., poetry, humor), the evaluator bottleneck is real and human validation overhead increases.
- The method is designed for multi-segment structured outputs. For single-sentence translation tasks, ALPO reduces to approximately standard DPO and offers no advantage — use regular DPO instead.
- Sampling `k=15` candidates per segment with LLM evaluation is computationally expensive at data construction time. For a corpus of 10K subtitle blocks with 20 lines each, this means ~3M evaluator calls. Budget GPU time accordingly or use a smaller evaluator model.
- The three-dimensional evaluation framework (accuracy, naturalness, vividness) is tuned for visual media subtitles. Other domains may need different quality dimensions (e.g., technical precision for legal translation).
- Prefix mixing introduces training complexity and slightly slows convergence compared to standard DPO. For time-constrained settings, a fixed `λ=0.4` is a reasonable approximation.

## Reference

**Paper**: [From Utterance to Vividity: Training Expressive Subtitle Translation LLM via Adaptive Local Preference Optimization](https://arxiv.org/abs/2602.01068v1) (ICLR 2026)

Key sections to study: Algorithm 1 (preference pair sampling), Section 3.3 (ALPO loss formulation with adaptive weights and dynamic temperature), Section 3.4 (scheduled prefix mixing), and Table 3 (multidimensional evaluation results showing +17.5 vividness improvement over SFT baseline).