---
name: "legalone-family-foundation-reliable"
description: "Build domain-specialized LLM training pipelines using the LegalOne three-phase methodology: Plasticity-Adjusted Sampling for domain adaptation, Agentic CoT Distillation for structured reasoning data, and Curriculum RL for progressive skill acquisition. Use when: 'build a domain-adapted model', 'create training pipeline for specialized LLM', 'distill reasoning from expert data', 'curriculum reinforcement learning for LLM', 'domain adaptation without catastrophic forgetting', 'agentic data synthesis pipeline'."
---

This skill enables Claude to design and implement three-phase domain-specialized LLM training pipelines following the LegalOne methodology. The core insight is that reliable domain reasoning requires (1) perplexity-guided data scheduling during continual pre-training to prevent catastrophic forgetting, (2) agentic multi-stage workflows to synthesize structured chain-of-thought data from raw domain texts, and (3) curriculum reinforcement learning that progresses from memorization through understanding to autonomous reasoning. These techniques generalize beyond legal AI to any domain requiring rigorous multi-step reasoning over specialized knowledge.

## When to Use

- When building a continual pre-training pipeline that must acquire domain knowledge without degrading general capabilities
- When the user needs to convert raw domain documents (legal cases, medical records, financial reports) into structured reasoning training data
- When designing a curriculum RL strategy that progressively builds from recall to analysis to generation
- When implementing perplexity-based data filtering and sampling schedules for domain corpora
- When creating an agentic pipeline that decomposes expert workflows into chain-of-thought training trajectories
- When the user asks how to distill reasoning from a large teacher model into structured SFT data with quality control

## Key Technique

**Phase 1 — Plasticity-Adjusted Sampling (PAS):** During continual pre-training (mid-training), data is stratified into perplexity buckets (low/mid/high PPL, computed by a reference model). A plasticity coefficient `alpha(t) = lr(t) / lr_max` controls sampling: during the learning rate re-warmup (alpha near 0), the scheduler concentrates sampling on low-PPL "anchor" data (familiar, stabilizing examples), then relaxes to a target mixture as the learning rate stabilizes (alpha near 1). This prevents the loss spikes and catastrophic forgetting that plague naive domain adaptation. Documents above a PPL threshold (e.g., 30) are filtered entirely as noise.

**Phase 2 — Legal Agentic CoT Distillation (LEAD):** Raw domain texts are converted into structured reasoning data through a four-stage pipeline: (1) decompose source documents into structured modules (facts, reasoning, decisions) via regex/heuristic segmentation, (2) generate diverse questions through structural logic distillation, multi-perspective user simulation, and real-world query alignment, (3) synthesize chain-of-thought answers using a configurable agentic workflow where each node in the reasoning graph (e.g., fact-finding, rule-retrieval, deduction, conclusion) has expert protocols, external knowledge retrieval, and doctrinal templates, (4) refine trajectories by internalizing external references and merging multi-step traces into compact self-contained reasoning, then filter via both heuristic rules and LLM-as-judge scoring across six quality dimensions.

**Phase 3 — Curriculum Reinforcement Learning:** Training progresses through stages of increasing cognitive demand: memorization (exact recall with ROUGE rewards), application (provision identification with soft-match F1), analysis (multi-choice reasoning with accuracy rewards), prediction (outcome forecasting with LLM-as-judge), and generation (document drafting with rubric-based evaluation). Each stage increases max output length (1K to 32K tokens) and reasoning complexity. The algorithm uses a token-level baseline `b2 = sum(Li*Ri) / sum(Li)` instead of the standard sequence-level baseline, which provably reduces gradient variance and eliminates length bias.

## Step-by-Step Workflow

### Phase 1: Domain Adaptation via PAS

1. **Compute reference perplexity for all training documents** using a base model (e.g., the model you are adapting). Partition documents into buckets: low-PPL (PPL <= 5), mid-PPL (5 < PPL <= 15), high-PPL (PPL > 15). Filter out any document with PPL > 30 as noise.

2. **Define the PAS sampling distribution.** Implement the formula:
   ```
   P(bucket_i | t) = w_i * exp(lambda * I(i==low) * (1 - alpha(t))) / Z(t)
   ```
   where `alpha(t) = lr(t) / lr_max`, `lambda = 5`, `w_i` are target equilibrium proportions, and `Z(t)` is the normalizing constant. During re-warmup, this concentrates ~80%+ sampling on low-PPL data; at stable LR, it relaxes to your target mix (e.g., 20% low-PPL).

3. **Configure the continual pre-training run** with a WSD (Warmup-Stable-Decay) learning rate schedule. Use approximately 2,000 re-warmup steps to peak LR, then a long stable phase, then cool-down. Set a general-to-domain data ratio (e.g., 4:6). Use the PAS scheduler to dynamically adjust batch composition at each step.

4. **Validate stability throughout training** by monitoring validation loss on both domain and general benchmarks. The low-PPL anchor data should act as a numerical damper — if you see loss spikes during warmup, increase `lambda` or extend the re-warmup phase.

### Phase 2: Agentic CoT Data Synthesis (LEAD)

5. **Decompose source documents into structured modules.** For each domain document, segment it into canonical sections (e.g., for legal: Facts, Reasoning, Decision; for medical: History, Diagnosis, Treatment Plan). Use regex-based heuristic segmentation first, then validate with an LLM scorer (discard documents scoring below 3/5 on completeness).

6. **Generate diverse training prompts** using three strategies in parallel:
   - *Structural logic distillation*: Given the document structure, derive questions that require multi-step reasoning across sections
   - *Multi-perspective simulation*: Prompt an LLM to role-play different stakeholders (e.g., patient, doctor, insurer) generating realistic queries
   - *Real-world alignment*: Collect or synthesize questions matching actual user query distributions from domain forums/platforms

7. **Build the agentic CoT synthesis workflow.** Define a directed graph of reasoning stages (e.g., Fact-Finding -> Issue-Identification -> Rule-Retrieval -> Deduction -> Conclusion). At each node:
   - Inject domain expert protocols as system prompts (explicit procedural guardrails)
   - Enable retrieval from an external knowledge base (statutes, guidelines, protocols) with re-ranking
   - Apply doctrinal reasoning templates (e.g., syllogistic reasoning: major premise, minor premise, conclusion)
   Generate the full chain-of-thought by executing this workflow with a strong teacher model.

8. **Refine and internalize the trajectories.** For each generated CoT: (a) rewrite to remove dependence on external retrieval results — paraphrase cited references as internalized knowledge, (b) merge multi-step node outputs into a single coherent end-to-end reasoning trace, preserving correctness while removing redundancy. Then apply two-stage quality filtering:
   - *Heuristic*: Remove truncated, structurally incomplete, or duplicate samples
   - *LLM-as-Judge*: Score each sample 1-10 on reasoning quality, consistency, answer alignment, conciseness, linguistic quality, and overall; discard any sample with overall or any subscore < 7

### Phase 3: Curriculum Reinforcement Learning

9. **Define the curriculum stages with increasing difficulty and context length:**

   | Stage | Task Type | Reward Signal | Max Tokens |
   |-------|-----------|---------------|------------|
   | 1. Memorization | Exact recall / completion | ROUGE score | 1,024 |
   | 2. Application | Identify applicable rules/procedures | Soft-match F1 (ROUGE > 0.5 threshold) | 4,096 |
   | 3. Analysis | Multi-choice reasoning questions | Accuracy | 8,192 |
   | 4. Prediction | Outcome forecasting from facts | LLM-as-judge with domain rubric | 16,384 |
   | 5. Generation | Full document/report drafting | Rubric-based trained evaluator | 32,768 |

10. **Implement token-level baseline for variance reduction.** Replace the standard sequence-level RL baseline `b1 = mean(R_i)` with the length-weighted baseline:
    ```
    b2 = sum(L_i * R_i) / sum(L_i)
    ```
    This eliminates the length bias where long negative samples are over-penalized and long positive samples are over-rewarded. Use DAPO as the base RL algorithm.

## Concrete Examples

**Example 1: Building a Domain-Adapted Medical Reasoning Model**

User: "I want to adapt Llama-3-8B for medical diagnosis reasoning without losing its general capabilities."

Approach:
1. Collect medical corpus (clinical notes, textbooks, guidelines) and compute per-document PPL using Llama-3-8B-base
2. Partition into buckets: low-PPL (< 5), mid-PPL (5-15), high-PPL (15-30), filtered (> 30)
3. Set target mix: 40% general (FineWeb-Edu, Wikipedia), 60% medical
4. Implement PAS scheduler with lambda=5, WSD LR schedule peaking at 3e-4, 2000 re-warmup steps
5. Run continual pre-training for ~100B tokens, validating on both MMLU and MedQA

Output — PAS scheduler implementation:
```python
import math

class PlasticityAdjustedSampler:
    def __init__(self, bucket_weights, lambda_val=5, lr_max=3e-4):
        """
        bucket_weights: dict mapping bucket_name -> target equilibrium weight
            e.g., {"low_ppl": 0.2, "mid_ppl": 0.5, "high_ppl": 0.3}
        """
        self.bucket_weights = bucket_weights
        self.lambda_val = lambda_val
        self.lr_max = lr_max
        self.low_ppl_key = "low_ppl"  # anchor bucket

    def get_sampling_probs(self, current_lr):
        alpha = current_lr / self.lr_max  # plasticity coefficient
        concentration = self.lambda_val * (1.0 - alpha)

        unnormalized = {}
        for bucket, w in self.bucket_weights.items():
            indicator = 1.0 if bucket == self.low_ppl_key else 0.0
            unnormalized[bucket] = w * math.exp(concentration * indicator)

        total = sum(unnormalized.values())
        return {b: v / total for b, v in unnormalized.items()}


# Usage during training loop:
sampler = PlasticityAdjustedSampler(
    bucket_weights={"low_ppl": 0.2, "mid_ppl": 0.5, "high_ppl": 0.3},
    lambda_val=5,
    lr_max=3e-4,
)
# At re-warmup start (lr ~ 0): low_ppl gets ~97% of samples
# At stable phase (lr ~ lr_max): low_ppl gets ~20% (target weight)
```

**Example 2: Agentic CoT Data Synthesis for Financial Analysis**

User: "I have 50K SEC filings. How do I turn them into structured reasoning training data?"

Approach:
1. Segment each filing: extract Risk Factors, Financial Statements, MD&A, Legal Proceedings sections via regex
2. Generate questions: "What are the key financial risks?", "How does revenue trend compare to guidance?", "Is the company likely to face regulatory action?"
3. Build agentic workflow: FactExtraction -> RiskIdentification -> FinancialAnalysis -> RegulatoryAssessment -> Conclusion
4. At each node, inject analyst protocols and retrieve relevant accounting standards (GAAP/IFRS)
5. Refine: internalize cited standards, merge node outputs into single CoT, filter with LLM-as-judge

Output — Agentic workflow definition:
```python
WORKFLOW_STAGES = [
    {
        "name": "fact_extraction",
        "protocol": "Extract all quantitative facts: revenue, expenses, margins, "
                     "YoY changes. Flag any restatements or unusual items.",
        "retrieval": {"source": "accounting_standards_db", "query_from": "document"},
        "template": "Given the financial data: {facts}\nKey metrics: {metrics}",
    },
    {
        "name": "risk_identification",
        "protocol": "Identify material risks using the SEC risk factor taxonomy. "
                     "Classify each as operational, financial, regulatory, or market.",
        "retrieval": {"source": "sec_guidelines_db", "query_from": "risk_factors"},
        "template": "Risk category: {category}\nEvidence: {evidence}\nSeverity: {level}",
    },
    {
        "name": "analysis_synthesis",
        "protocol": "Apply fundamental analysis framework. Compare against sector "
                     "benchmarks. Assess going-concern indicators per ASC 205-40.",
        "retrieval": {"source": "sector_benchmarks_db", "query_from": "financials"},
        "template": "Major premise: {standard}\nMinor premise: {company_data}\n"
                     "Conclusion: {assessment}",
    },
]

# Teacher model executes each stage sequentially, producing per-node CoT.
# Refinement step merges into single trajectory and removes retrieval artifacts.
```

**Example 3: Curriculum RL for Code Review Reasoning**

User: "I want to train a model that reviews code with increasing sophistication — from spotting syntax errors to architectural analysis."

Approach:
1. Define five curriculum stages:
   - Stage 1 (Memorization): Given a language spec rule, reproduce it exactly. Reward: ROUGE.
   - Stage 2 (Application): Given code, identify which style/lint rules apply. Reward: F1 on rule set.
   - Stage 3 (Analysis): Multiple-choice — which of these code snippets has a bug? Reward: accuracy.
   - Stage 4 (Prediction): Given a PR diff, predict review outcome and key issues. Reward: LLM-as-judge.
   - Stage 5 (Generation): Write a full code review with suggestions. Reward: rubric-based evaluator.
2. Increase max output tokens per stage: 512, 2048, 4096, 8192, 16384
3. Use token-level baseline b2 to handle the wide variance in review lengths

Output — Curriculum RL config:
```python
CURRICULUM_STAGES = [
    {
        "name": "rule_memorization",
        "task": "Complete the following lint rule definition",
        "reward": "rouge_l",
        "max_tokens": 512,
        "dataset": "lint_rules_completion",
    },
    {
        "name": "rule_application",
        "task": "List all applicable rules for this code snippet",
        "reward": "soft_match_f1",  # ROUGE > 0.5 threshold per rule
        "max_tokens": 2048,
        "dataset": "code_rule_identification",
    },
    {
        "name": "bug_analysis",
        "task": "Which snippet contains the bug? Explain your reasoning.",
        "reward": "accuracy",
        "max_tokens": 4096,
        "dataset": "bug_detection_mcq",
    },
    {
        "name": "review_prediction",
        "task": "Predict the review outcome for this PR diff",
        "reward": "llm_as_judge",
        "max_tokens": 8192,
        "dataset": "pr_review_prediction",
    },
    {
        "name": "full_review_generation",
        "task": "Write a complete code review with actionable suggestions",
        "reward": "rubric_evaluator",
        "max_tokens": 16384,
        "dataset": "code_review_generation",
    },
]

def token_level_baseline(rewards, lengths):
    """Compute b2 = sum(L_i * R_i) / sum(L_i) for variance reduction."""
    weighted_sum = sum(l * r for l, r in zip(lengths, rewards))
    total_length = sum(lengths)
    return weighted_sum / total_length if total_length > 0 else 0.0
```

## Best Practices

- **Do:** Compute PPL using the actual base model you are adapting, not a different model. The perplexity buckets must reflect what is familiar vs. novel to your specific model.
- **Do:** Start curriculum RL from the easiest stage even if the model already performs well on it. The memorization stage anchors factual knowledge and reduces hallucination in later stages.
- **Do:** Apply both heuristic and LLM-as-judge filtering to synthesized CoT data. Heuristic filtering catches structural defects cheaply; LLM-as-judge catches subtle reasoning errors.
- **Do:** Use syllogistic reasoning templates (major premise, minor premise, conclusion) in the agentic CoT workflow. This enforces logical rigor that free-form generation lacks.
- **Avoid:** Skipping the trajectory internalization step. If your SFT data references external retrieval results that the model won't have at inference time, it learns to hallucinate citations.
- **Avoid:** Using a flat data mixture during the learning rate warmup phase. This is when the model is most vulnerable to catastrophic forgetting — PAS exists specifically to protect this window.

## Error Handling

- **Loss spikes during re-warmup**: Increase `lambda` (e.g., from 5 to 8) to concentrate more heavily on low-PPL anchor data, or extend the re-warmup phase beyond 2000 steps.
- **Quality filtering removes too much data (>60%)**: Your agentic workflow protocols are too loose. Tighten the expert protocols at each node, add retrieval guardrails, or use a stronger teacher model.
- **RL reward hacking at later curriculum stages**: The model may generate verbose outputs to game length-correlated rewards. Switch to the token-level baseline `b2` and verify your rubric evaluator penalizes redundancy.
- **Domain performance improves but general capabilities degrade**: Your general-to-domain data ratio in mid-training is too aggressive. Increase the general data proportion or extend cool-down with a lower learning rate.
- **Agentic CoT synthesis produces inconsistent reasoning**: Ensure each workflow node receives the output of all previous nodes as context. Check that retrieval results are relevant — add a re-ranking stage if precision is below 80%.

## Limitations

- PAS requires computing perplexity for the entire training corpus upfront, which is computationally expensive for very large datasets (100B+ tokens). Approximations via random sampling may be necessary.
- The agentic CoT distillation pipeline depends on a strong teacher model (the paper uses Qwen3-235B). If your teacher model is weak in the target domain, the synthesized data will inherit its errors.
- Curriculum RL stages require carefully designed reward functions per stage. Poorly calibrated rewards (especially LLM-as-judge) can stall training or cause reward hacking.
- The three-phase pipeline is sequential and data-intensive. It is designed for serious domain specialization efforts, not quick fine-tuning experiments.
- The technique is validated primarily on Chinese legal text. Adaptation to other languages and domains requires re-calibrating PPL thresholds, document segmentation heuristics, and doctrinal templates.

## Reference

**Paper:** [LegalOne: A Family of Foundation Models for Reliable Legal Reasoning](https://arxiv.org/abs/2602.00642v2) — Li et al., 2026. Focus on Section 3 (PAS formulation and stability analysis), Section 4 (LEAD four-stage pipeline), and Section 5 (curriculum RL stages and token-level baseline derivation).