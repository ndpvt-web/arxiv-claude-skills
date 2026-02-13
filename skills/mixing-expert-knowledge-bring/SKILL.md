---
name: "mixing-expert-knowledge-bring"
description: "Integrate domain expert knowledge into LLM fine-tuning pipelines using mixed cold-start SFT and reinforcement learning. Use when: 'mix expert knowledge with reasoning data', 'fine-tune LLM for specialized domain', 'build domain expert from general model', 'combine structured expert data with CoT', 'transfer reasoning ability to new domain', 'GRPO training with domain reward function'."
---

# Mixing Expert Knowledge: Bridging General LLM Reasoning to Specialized Domains

This skill enables Claude to design and implement training pipelines that transfer general LLM reasoning capabilities into specialized domains by mixing structured expert knowledge with Chain-of-Thought reasoning data. Based on the LoGos methodology (NeurIPS 2025), it provides a concrete two-phase recipe: (1) a mixed cold-start SFT phase blending heuristic-rule-templated domain data with general long-CoT data, followed by (2) Group Relative Policy Optimization (GRPO) reinforcement learning with a domain-specific piecewise reward function. The approach is domain-agnostic and applies wherever structured professional knowledge sources exist alongside an external validation system.

## When to Use

- When the user wants to fine-tune an LLM to perform well in a specialized domain (medicine, law, finance, game-playing, scientific analysis) while preserving general reasoning
- When building a training data pipeline that converts structured expert databases (APIs, engines, annotated corpora) into SFT-ready JSONL with heuristic reasoning templates
- When designing a reward function for GRPO/RL that grades domain-specific outputs on a ranked piecewise scale
- When deciding data mixing ratios between domain-specific expertise and general reasoning corpora for cold-start SFT
- When the user asks how to prevent catastrophic forgetting of general capabilities while specializing a model
- When constructing an evaluation benchmark for domain-specific LLM performance with structured position/problem sampling

## Key Technique

**The core insight:** Neither general reasoning data alone nor domain expertise alone is sufficient for professional-level specialized performance. General CoT data teaches the model *how to reason* (analyze, consider alternatives, summarize), while structured domain data teaches *what to reason about*. Mixing them in a cold-start SFT phase creates a foundation that reinforcement learning can then elevate to expert level. Without the mixed cold start, RL hits a ceiling below beginner proficiency (~50% accuracy); with it, RL pushes to professional level (88%+).

**The heuristic template is critical.** Raw domain data (e.g., "position X, best move Y") produces poor results. Instead, expert knowledge must be restructured into a multi-step reasoning template: (1) identify the current state, (2) analyze multiple candidate actions with variations, (3) summarize and select the optimal action, (4) produce structured output with confidence/quality scores. This templating is what allows the model to generalize CoT reasoning patterns from math/coding into the new domain.

**GRPO with piecewise rewards refines judgment.** The RL phase uses Group Relative Policy Optimization, sampling multiple responses per query and computing group-relative advantages. The reward function assigns tiered scores: maximum reward for rank-1 actions with accurate quality predictions, moderate penalties for rank-2-3 actions, steeper penalties for rank-4-10 actions, and minimal baseline rewards for out-of-candidate but format-correct responses. This graduated structure teaches nuanced quality discrimination rather than binary right/wrong.

## Step-by-Step Workflow

1. **Identify the domain knowledge source.** Locate a structured expert system, annotated database, or engine that can provide ranked candidate actions with quality scores for domain-specific problems (analogous to KataGo providing top-10 moves with win rates for Go positions).

2. **Sample positions/problems uniformly.** Extract a large-scale set of domain states (target 1M-10M for SFT, 1K for RL). Sample uniformly across difficulty levels and problem types to avoid distribution bias.

3. **Annotate with the expert system.** For each sampled state, run the domain expert to generate ranked candidate actions (top-N) with quality/confidence scores. Store as structured records: `{state, candidates: [{action, rank, score}], metadata}`.

4. **Design the heuristic reasoning template.** Create a 4-step template that mirrors natural reasoning:
   - **State identification:** "The current [domain context] shows [state description]..."
   - **Candidate analysis:** "Considering candidate [action]: [variation analysis with implications]..."
   - **Summary and selection:** "Weighing all candidates, [action] is optimal because..."
   - **Structured output:** Produce the action in a parseable format with predicted quality score.

5. **Apply the template to generate SFT data.** Convert each annotated record into a training example by filling the heuristic template. Output JSONL with `{prompt, completion}` pairs where completions follow the full reasoning chain. Target 100K-10M examples.

6. **Curate general CoT reasoning data.** Collect or source long chain-of-thought examples from math, coding, and logical reasoning datasets (e.g., from open-source reasoning corpora). These provide the reasoning scaffolding the model will transfer to the new domain.

7. **Run mixed cold-start SFT.** Fine-tune the base model on the blended dataset (domain + general CoT) for 1 epoch of each. Use cosine annealing (4e-5 to 4e-6), max sequence length 32,768 tokens. Monitor both domain accuracy and general benchmark scores to confirm no degradation.

8. **Design the piecewise reward function for GRPO.** Implement a reward function that:
   - Awards maximum score (coefficient ~0.8) for rank-1 actions with accurate quality predictions
   - Applies moderate penalty (coefficient ~0.6) for rank-2-3 actions
   - Applies steep penalty (coefficient ~0.4) for rank-4-10 actions
   - Gives minimal baseline reward for format-correct but out-of-candidate responses
   - Gives zero reward for malformed outputs

9. **Run GRPO reinforcement learning.** Train with batch size 64, 16 rollouts per data point, max response length 8,192 tokens, KL coefficient 5e-4 (low, to accommodate large distribution shift from pretraining). Use ~1K curated RL examples covering diverse difficulty levels.

10. **Evaluate on a held-out domain benchmark.** Construct a standardized evaluation set (1K+ problems) with known expert rankings. Measure top-1 accuracy, top-3 accuracy, legal/valid action rate, and quality score prediction error. Compare against domain baselines and general LLM baselines.

## Concrete Examples

**Example 1: Building a Medical Diagnosis Reasoning Model**

```
User: I want to fine-tune Qwen2.5-7B to reason about differential
diagnosis from patient symptoms, using our structured medical
knowledge base that ranks diagnoses by likelihood.

Approach:
1. Extract 2M patient-symptom-diagnosis records from the knowledge base,
   each with ranked differential diagnoses and confidence scores.

2. Design the heuristic template:
   - State: "Patient presents with [symptoms]. Relevant history: [history]."
   - Analysis: "Considering [diagnosis-1]: Symptom [X] is consistent
     because [reasoning]. However, [Y] argues against it..."
   - Summary: "The most likely diagnosis is [diagnosis] with confidence
     [score] because [key differentiating factors]."
   - Output: {"diagnosis": "...", "confidence": 0.85, "differentials": [...]}

3. Generate 2M templated SFT examples in JSONL format.

4. Source 500K general medical/scientific CoT examples from open datasets.

5. Mixed SFT: 1 epoch domain + 1 epoch general CoT, cosine anneal 4e-5→4e-6.

6. GRPO with piecewise reward:
   - Rank-1 diagnosis correct + confidence within 5%: reward 1.0
   - Rank-2-3 diagnosis: reward 0.6 with confidence penalty
   - Rank-4+: reward 0.3
   - Invalid/malformed: reward 0.0

7. Evaluate on held-out 1K cases against physician panel rankings.

Output (model response after training):
"Patient presents with acute onset chest pain, dyspnea, and unilateral
leg swelling. Considering pulmonary embolism: the triad of chest pain,
dyspnea, and leg swelling is classic for PE, especially given sudden
onset. DVT in the leg is the likely embolic source. Considering
pneumothorax: chest pain and dyspnea fit, but leg swelling is
unexplained. Considering MI: chest pain and dyspnea fit, but
unilateral leg swelling is atypical. The most likely diagnosis is
pulmonary embolism with confidence 0.88."
```

**Example 2: Legal Case Outcome Prediction**

```
User: We have a database of 500K legal cases with judge annotations
ranking likely outcomes. How do I structure the training pipeline?

Approach:
1. Sample 500K cases. For each, extract: case facts, statute references,
   ranked outcomes with judicial confidence scores from the annotation DB.

2. Heuristic template:
   - State: "Case involves [facts]. Applicable statutes: [refs]."
   - Analysis: "Under [statute-1], outcome [A] is supported by [precedent].
     Counter-argument: [opposing interpretation]..."
   - Summary: "Most probable outcome is [outcome] (confidence [score])
     based on [decisive factors]."
   - Output: {"outcome": "...", "confidence": 0.72, "key_statutes": [...]}

3. Mix with 200K general legal/logical reasoning CoT examples.

4. SFT for 1 epoch on combined data, then GRPO with:
   - Rank-1 outcome + accurate confidence: reward 0.8 * accuracy_weight
   - Rank-2-3 outcomes: reward 0.6 * (1 - rank_deviation_penalty)
   - Rank-4+: reward 0.4 * format_correctness

5. Evaluate: top-1 accuracy, calibration of confidence scores,
   legal validity of cited statutes.
```

**Example 3: Reward Function Implementation**

```python
# Piecewise GRPO reward function (domain-agnostic template)
def domain_reward(prediction, ground_truth_candidates, alpha=0.1, beta=10):
    """
    prediction: {"action": str, "quality_score": float}
    ground_truth_candidates: [{"action": str, "rank": int, "score": float}, ...]
    """
    pred_action = prediction["action"]
    pred_score = prediction["quality_score"]

    # Find if prediction matches any candidate
    matched = None
    for candidate in ground_truth_candidates:
        if candidate["action"] == pred_action:
            matched = candidate
            break

    if matched is None:
        # Format-correct but out-of-candidate: minimal reward
        return 0.1 if is_valid_format(prediction) else 0.0

    rank = matched["rank"]
    true_score = matched["score"]
    score_accuracy = 1.0 - min(abs(pred_score - true_score) * beta, 1.0)

    if rank == 1:
        return 0.8 + 0.2 * score_accuracy  # Range: [0.8, 1.0]
    elif rank <= 3:
        return 0.6 * (1.0 - alpha * (rank - 1)) * score_accuracy
    elif rank <= 10:
        return 0.4 * (1.0 - alpha * (rank - 1)) * score_accuracy
    else:
        return 0.1
```

## Best Practices

- **Do:** Structure domain data with multi-step heuristic reasoning templates, not raw input-output pairs. The template is the single most impactful design decision -- ablations show it doubles accuracy ceilings.
- **Do:** Mix general CoT data (math, coding, logic) with domain data during SFT at roughly 1:1 epoch ratio. This transfers reasoning scaffolding without degrading domain learning.
- **Do:** Use a low KL coefficient (5e-4) in GRPO to allow large distribution shifts from the pretrained model, since the domain may be far from pretraining distribution.
- **Do:** Design reward functions with graduated tiers (not binary correct/incorrect). This teaches the model to discriminate quality among plausible candidates.
- **Avoid:** Training on more than 1 epoch of domain data during cold start -- diminishing returns set in and risk overfitting to template surface patterns.
- **Avoid:** Skipping the mixed cold-start phase and jumping straight to RL. Without the reasoning foundation from SFT, GRPO hits a hard performance ceiling below beginner level regardless of training duration.

## Error Handling

- **Expert system unavailable or slow:** Pre-compute all annotations in batch before starting the pipeline. Cache results in JSONL. Never call the expert system during training.
- **Quality score distribution mismatch:** If the expert system's confidence scores are poorly calibrated (e.g., always 0.9+), normalize them relative to the candidate set before computing rewards. Use rank-relative scoring instead of absolute scores.
- **Model generates invalid domain actions:** Add a format-checking validator to the reward function that returns 0.0 for unparseable outputs. Track the invalid-action rate during RL; if it exceeds 20%, reduce learning rate or add more format-focused SFT examples.
- **General capability degradation during SFT:** Monitor MATH/GPQA/HumanEval scores after each SFT checkpoint. If degradation exceeds 2%, reduce domain data ratio or add more general CoT data.
- **RL reward hacking:** If the model converges to always predicting rank-2-3 safe candidates instead of rank-1, increase the reward gap between tier 1 and tier 2 (e.g., coefficients 0.9 vs 0.5 instead of 0.8 vs 0.6).

## Limitations

- Requires a structured expert system or annotated database that can provide ranked candidates with quality scores. Domains without such a system (e.g., creative writing, open-ended design) cannot use this approach directly.
- The heuristic template must be designed by domain experts -- this is a manual, domain-specific step that cannot be fully automated. Poor template design limits the performance ceiling.
- Computationally expensive: the full pipeline requires 32-64 A800 GPUs for 7B-32B models. Smaller-scale experiments are possible but results may not transfer.
- Autoregressive LLM architecture may constrain efficiency for domains requiring real-time search or simulation (the model reasons sequentially, not in parallel like MCTS).
- The approach works best when the domain has clear right/wrong answers with gradable quality. Domains with subjective or multi-valid-answer outputs need modified reward functions.

## Reference

**Paper:** [Mixing Expert Knowledge: Bring Human Thoughts Back To the Game of Go](https://arxiv.org/abs/2601.16447v1) (NeurIPS 2025)
**Key takeaway:** Look at Section 3 (Methodology) for the heuristic template design and Section 4 (Experiments) for ablation studies proving that the mixed cold-start + GRPO combination is strictly necessary -- neither component alone reaches professional-level performance.
**Code/Data:** [github.com/Entarochuan/LoGos](https://github.com/Entarochuan/LoGos)