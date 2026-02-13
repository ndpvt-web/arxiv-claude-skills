---
name: "stealthrl-reinforcement-learning-paraphrase"
description: >
  Build RL-based adversarial paraphrase pipelines to stress-test AI-text detectors.
  Implements the StealthRL framework: GRPO training with LoRA adapters against
  multi-detector ensembles, composite reward functions balancing evasion and semantic
  preservation, and transferability evaluation protocols.
  Trigger phrases: "red-team AI text detectors", "adversarial paraphrase attack",
  "stress-test text detection", "RL paraphrase evasion", "detector robustness evaluation",
  "build a StealthRL pipeline"
---

# StealthRL: RL-Based Adversarial Paraphrase Attacks for Detector Robustness Evaluation

This skill enables Claude to implement the StealthRL framework -- a reinforcement learning
pipeline that trains a paraphrase policy to stress-test AI-text detectors under realistic
adversarial conditions. The core technique uses Group Relative Policy Optimization (GRPO)
with LoRA adapters on a base language model, optimizing a composite reward that balances
detector evasion against semantic preservation. This is a defensive security tool: it
exposes robustness gaps in detection systems so they can be hardened, not bypassed for
malicious purposes.

## When to Use

- When the user asks to **evaluate the robustness** of an AI-text detection system against adversarial paraphrasing
- When building a **red-team evaluation pipeline** for text classifiers (AI-generated vs. human-written)
- When implementing **GRPO-based RL fine-tuning** with LoRA adapters for any text rewriting objective
- When designing a **multi-objective reward function** that combines classifier evasion with semantic similarity
- When the user needs to **reproduce or extend StealthRL** attack settings (M0-M5) against detector families
- When assessing **transferability of adversarial attacks** across unseen detector architectures
- When building an **evaluation harness** with bootstrap confidence intervals and LLM-based Likert scoring

## Key Technique

**StealthRL** trains a paraphrase policy using Group Relative Policy Optimization (GRPO), a
variant of policy gradient methods that samples a group of candidate outputs per input and
computes advantages via group-relative normalization: `A_g = (R_g - R_mean) / sigma_R`. This
avoids training a separate value network. The policy is a Qwen3-4B model fine-tuned with
LoRA adapters (rank 32, alpha 32, dropout 0.05), keeping the base model frozen and training
only ~0.5% of parameters. A KL penalty (coefficient 0.05) against the frozen reference
policy prevents mode collapse and preserves fluency.

The composite reward function is `R(x, y) = alpha * R_det(y) + beta * R_sem(x, y)` where
alpha=1.0 and beta=0.1. The detector reward `R_det` is a weighted combination of normalized
scores from multiple detectors (e.g., w1=0.6 for RoBERTa classifier, w2=0.4 for
FastDetectGPT). The semantic reward `R_sem` is the cosine similarity between E5 embeddings
of the original text x and the paraphrase y. This asymmetric weighting (10:1 evasion vs.
semantics) aggressively optimizes for evasion while maintaining a floor on meaning
preservation.

The critical finding is **transferability**: attacks trained against RoBERTa and FastDetectGPT
transfer to the held-out Binoculars detector (achieving 0.001 TPR@1%FPR), revealing shared
architectural vulnerabilities across detector families rather than detector-specific
overfitting. This makes StealthRL a principled adversarial evaluation protocol -- it exposes
systemic weaknesses, not just per-detector blind spots.

## Step-by-Step Workflow

1. **Prepare the training corpus**: Collect 5,000-10,000 AI-generated text samples (e.g.,
   from the MAGE dataset or by prompting a target LLM). Store as JSONL with fields
   `{"id": str, "text": str, "source_model": str}`. Split 80/20 for train/eval.

2. **Set up the detector ensemble**: Implement scoring interfaces for each detector family.
   For RoBERTa: load `openai-community/roberta-large-openai-detector` and extract the
   P(AI) logit. For FastDetectGPT: compute conditional probability curvature using a
   scoring model (e.g., `gpt-neo-2.7B`). For Binoculars: pair `gpt2-medium`/`gpt2-large`
   and compute the cross-entropy-to-perplexity ratio. Normalize all scores to [0, 1].

3. **Configure the composite reward function**: Implement the weighted reward
   `R = 1.0 * R_det + 0.1 * R_sem`. For R_det, weight the training-visible detectors
   (e.g., 0.6 RoBERTa + 0.4 FastDetectGPT). For R_sem, compute cosine similarity of E5
   embeddings (`intfloat/e5-base-v2`). Hold out one detector family (e.g., Binoculars) to
   test transferability.

4. **Initialize the LoRA-adapted policy model**: Load the base model (Qwen3-4B-Instruct)
   and attach LoRA adapters with rank=32, alpha=32, dropout=0.05 targeting attention
   projection layers (q_proj, k_proj, v_proj, o_proj). Freeze the base model. Create a
   frozen copy as the reference policy for KL computation.

5. **Implement GRPO training loop**: For each training batch, sample G=8 candidate
   paraphrases per input at temperature=1.0, top_p=0.9, max_tokens=512. Score all
   candidates with the composite reward. Compute group-normalized advantages. Update the
   policy using clipped policy gradient with the KL penalty (lambda=0.05). Train for 3
   epochs with learning rate 2.8e-4 and effective batch size 16.

6. **Define attack settings for controlled evaluation**: Implement the M0-M5 hierarchy:
   M0 = original AI text (no attack), M1 = standard LLM paraphrase (no detector feedback),
   M2 = full StealthRL RL-trained policy, M3 = detector-guided candidate selection
   (inference-time reranking), M4 = single-detector RL (ablation), M5 = character-level
   Unicode substitution baseline (SilverSpeak). This lets you isolate which component drives
   evasion.

7. **Run evaluation at security-relevant operating points**: Compute TPR@1%FPR (the true
   positive rate at 1% false positive rate threshold) for each attack-detector pair. Also
   compute full AUROC. Use 500-iteration bootstrap resampling (seed=42) for 95% confidence
   intervals. Report per-detector and mean metrics.

8. **Conduct LLM-based quality evaluation**: Sample 200 paraphrased texts per attack method.
   Score each on two Likert axes (1-5): linguistic quality (fluency, grammar, naturalness)
   and semantic similarity (meaning preservation). Use a blind evaluation setup -- the judge
   LLM sees only source and paraphrase, no method labels.

9. **Analyze detector score distributions**: Plot histograms of detector scores for human
   text, original AI text, and each attack variant. Identify where the attack distribution
   overlaps with the human distribution -- this explains *why* evasion succeeds (the
   paraphraser shifts the AI score distribution into the human-like region).

10. **Assess transferability and report**: Evaluate the trained policy against the held-out
    detector. If TPR@1%FPR drops substantially on the held-out detector, the attack is
    exploiting detector-specific artifacts. If it transfers (as StealthRL demonstrates),
    this reveals shared architectural vulnerabilities that need systemic mitigation.

## Concrete Examples

**Example 1: Setting up the composite reward function**

User: "I want to build a StealthRL-style reward function for red-teaming a RoBERTa-based
AI text detector while preserving semantic meaning."

Approach:
1. Load the RoBERTa detector and E5 embedding model
2. Implement the weighted composite reward
3. Normalize detector scores to [0, 1] where 0 = "human-like"

Output:
```python
import torch
from transformers import AutoModelForSequenceClassification, AutoTokenizer, AutoModel
import torch.nn.functional as F

class StealthRLReward:
    def __init__(self, detector_weight=1.0, semantic_weight=0.1):
        self.alpha = detector_weight
        self.beta = semantic_weight

        # RoBERTa AI-text detector
        self.det_tokenizer = AutoTokenizer.from_pretrained(
            "openai-community/roberta-large-openai-detector"
        )
        self.det_model = AutoModelForSequenceClassification.from_pretrained(
            "openai-community/roberta-large-openai-detector"
        )
        self.det_model.eval()

        # E5 embedding model for semantic similarity
        self.emb_tokenizer = AutoTokenizer.from_pretrained("intfloat/e5-base-v2")
        self.emb_model = AutoModel.from_pretrained("intfloat/e5-base-v2")
        self.emb_model.eval()

    def _detector_score(self, text: str) -> float:
        """Return P(AI) in [0, 1]. Lower = more human-like = better for attacker."""
        inputs = self.det_tokenizer(text, return_tensors="pt", truncation=True, max_length=512)
        with torch.no_grad():
            logits = self.det_model(**inputs).logits
        probs = F.softmax(logits, dim=-1)
        p_ai = probs[0, 1].item()  # index 1 = AI class
        return p_ai

    def _semantic_similarity(self, original: str, paraphrase: str) -> float:
        """Cosine similarity of E5 embeddings."""
        texts = [f"passage: {original}", f"passage: {paraphrase}"]
        inputs = self.emb_tokenizer(texts, return_tensors="pt", padding=True, truncation=True)
        with torch.no_grad():
            outputs = self.emb_model(**inputs)
        embeddings = outputs.last_hidden_state[:, 0, :]  # CLS pooling
        embeddings = F.normalize(embeddings, p=2, dim=1)
        return (embeddings[0] @ embeddings[1]).item()

    def compute(self, original: str, paraphrase: str) -> dict:
        p_ai = self._detector_score(paraphrase)
        r_det = 1.0 - p_ai  # reward = how human-like the paraphrase appears
        r_sem = self._semantic_similarity(original, paraphrase)
        r_total = self.alpha * r_det + self.beta * r_sem
        return {"r_total": r_total, "r_det": r_det, "r_sem": r_sem, "p_ai": p_ai}
```

**Example 2: Implementing the GRPO training step**

User: "Implement the GRPO update step where I sample a group of candidates and compute
group-relative advantages."

Approach:
1. Sample G candidates per input from the policy
2. Score each with the composite reward
3. Normalize advantages within the group
4. Compute clipped policy gradient with KL penalty

Output:
```python
import torch
import torch.nn.functional as F

def grpo_step(policy, ref_policy, tokenizer, reward_fn, batch, config):
    """
    One GRPO training step.

    config keys: group_size (8), kl_coeff (0.05), clip_eps (0.2), max_tokens (512)
    """
    G = config["group_size"]  # 8
    all_log_probs, all_ref_log_probs, all_advantages = [], [], []

    for sample in batch:
        original_text = sample["text"]
        prompt = f"Paraphrase the following text while preserving its meaning:\n\n{original_text}\n\nParaphrase:"
        input_ids = tokenizer.encode(prompt, return_tensors="pt").to(policy.device)

        # Sample G candidates from current policy
        candidates = []
        candidate_log_probs = []
        for _ in range(G):
            with torch.no_grad():
                output = policy.generate(
                    input_ids, max_new_tokens=config["max_tokens"],
                    temperature=1.0, top_p=0.9, do_sample=True,
                    return_dict_in_generate=True, output_scores=True
                )
            gen_ids = output.sequences[0, input_ids.shape[1]:]
            candidates.append(tokenizer.decode(gen_ids, skip_special_tokens=True))
            # Collect per-token log probs for policy gradient
            candidate_log_probs.append(compute_log_prob(policy, input_ids, gen_ids))

        # Score all candidates
        rewards = torch.tensor([
            reward_fn.compute(original_text, c)["r_total"] for c in candidates
        ])

        # Group-relative advantage normalization
        mean_r, std_r = rewards.mean(), rewards.std() + 1e-8
        advantages = (rewards - mean_r) / std_r

        # Compute reference policy log probs for KL penalty
        for i, c in enumerate(candidates):
            gen_ids = tokenizer.encode(c, return_tensors="pt").to(policy.device)
            ref_lp = compute_log_prob(ref_policy, input_ids, gen_ids)
            all_ref_log_probs.append(ref_lp)
            all_log_probs.append(candidate_log_probs[i])
            all_advantages.append(advantages[i])

    # Policy gradient with KL penalty
    log_probs = torch.stack(all_log_probs)
    ref_log_probs = torch.stack(all_ref_log_probs)
    advantages = torch.stack(all_advantages)

    kl_divergence = (log_probs - ref_log_probs).mean()
    policy_loss = -(log_probs * advantages).mean()
    total_loss = policy_loss + config["kl_coeff"] * kl_divergence

    return total_loss, {"policy_loss": policy_loss.item(), "kl": kl_divergence.item()}
```

**Example 3: Evaluation harness with bootstrap confidence intervals**

User: "Build an evaluation script that computes TPR@1%FPR and AUROC with bootstrap CIs
for my detector against adversarial paraphrases."

Approach:
1. Score both human and paraphrased texts with the detector
2. Compute ROC curve and extract TPR at the 1% FPR operating point
3. Bootstrap resample 500 times for 95% confidence intervals

Output:
```python
import numpy as np
from sklearn.metrics import roc_curve, roc_auc_score

def tpr_at_fpr(y_true, y_scores, target_fpr=0.01):
    """Compute TPR at a specific FPR threshold."""
    fprs, tprs, _ = roc_curve(y_true, y_scores)
    # Interpolate to find TPR at exactly target_fpr
    idx = np.searchsorted(fprs, target_fpr, side="right") - 1
    if idx < 0:
        return 0.0
    return tprs[idx]

def evaluate_with_bootstrap(human_scores, ai_scores, n_bootstrap=500, seed=42):
    """
    Evaluate detector performance with bootstrap 95% CIs.

    human_scores: detector P(AI) scores for human text (should be low)
    ai_scores: detector P(AI) scores for AI/paraphrased text (should be high if detected)
    """
    rng = np.random.RandomState(seed)
    y_true = np.concatenate([np.zeros(len(human_scores)), np.ones(len(ai_scores))])
    y_scores = np.concatenate([human_scores, ai_scores])

    # Point estimates
    auroc = roc_auc_score(y_true, y_scores)
    tpr_1fpr = tpr_at_fpr(y_true, y_scores, target_fpr=0.01)

    # Bootstrap
    auroc_boots, tpr_boots = [], []
    n = len(y_true)
    for _ in range(n_bootstrap):
        idx = rng.choice(n, size=n, replace=True)
        try:
            auroc_boots.append(roc_auc_score(y_true[idx], y_scores[idx]))
            tpr_boots.append(tpr_at_fpr(y_true[idx], y_scores[idx]))
        except ValueError:
            continue  # skip if only one class in bootstrap sample

    return {
        "auroc": auroc,
        "auroc_ci95": (np.percentile(auroc_boots, 2.5), np.percentile(auroc_boots, 97.5)),
        "tpr_at_1fpr": tpr_1fpr,
        "tpr_ci95": (np.percentile(tpr_boots, 2.5), np.percentile(tpr_boots, 97.5)),
    }

# Usage:
# results = evaluate_with_bootstrap(human_scores, attacked_scores)
# print(f"AUROC: {results['auroc']:.3f} ({results['auroc_ci95'][0]:.3f}-{results['auroc_ci95'][1]:.3f})")
# print(f"TPR@1%FPR: {results['tpr_at_1fpr']:.4f} ({results['tpr_ci95'][0]:.4f}-{results['tpr_ci95'][1]:.4f})")
```

## Best Practices

- **Do** hold out at least one detector family from training to measure transferability -- this distinguishes systemic vulnerabilities from detector-specific overfitting.
- **Do** use the 1% FPR operating point for evaluation, not just AUROC. In security contexts, the false positive rate constraint is what determines real-world deployability.
- **Do** weight evasion much higher than semantics in the reward (10:1 ratio). The KL penalty against the reference policy already provides a fluency floor, so the explicit semantic reward primarily prevents catastrophic meaning drift.
- **Do** plot detector score distributions (not just aggregate metrics) to diagnose *why* evasion succeeds -- the paraphraser shifts the AI text distribution to overlap with human text scores.
- **Avoid** training against a single detector. Single-detector RL (the M4 setting) overfits to that detector's decision boundary and transfers poorly.
- **Avoid** using this framework to evade detection for dishonest purposes. StealthRL is designed as an adversarial evaluation protocol to *improve* detectors by exposing their weaknesses.

## Error Handling

- **OOM during GRPO sampling**: Group size 8 with 512-token generations consumes significant VRAM. Reduce group size to 4 or use gradient accumulation. Alternatively, score candidates in mini-batches rather than all at once.
- **Reward collapse (all candidates score similarly)**: If the standard deviation of group rewards approaches zero, advantages become numerically unstable. Add epsilon=1e-8 to the denominator during normalization. Also check that detector scores are actually varying -- a saturated detector returns constant scores.
- **KL divergence explosion**: If KL grows rapidly, the policy is diverging from the reference. Increase the KL coefficient from 0.05 to 0.1-0.2, or reduce the learning rate. Monitor KL per step and halt training if it exceeds 10.0.
- **Low semantic similarity after training**: If R_sem drops below 0.7, increase beta from 0.1 to 0.2-0.3. Alternatively, add a hard threshold that assigns zero reward to paraphrases with cosine similarity below 0.6.
- **Bootstrap CI computation fails**: Ensure at least 100 samples per class (human and AI). With very small test sets, bootstrap samples may contain only one class, causing `roc_auc_score` to error. Catch and skip degenerate samples.

## Limitations

- **Compute requirements**: GRPO with G=8 candidates per sample multiplies inference cost by 8x during training. A full StealthRL training run requires a GPU with at least 24GB VRAM (A100 recommended for the 4B parameter model).
- **Detector API dependency**: The reward function requires running detector inference at every training step. Detectors behind rate-limited APIs cannot be used directly -- you need local model weights.
- **Not a general-purpose paraphraser**: The trained policy is specifically optimized to evade the training detector ensemble. It may produce unnatural phrasings that are detectable by humans or by detectors using very different architectures (e.g., watermark-based detection).
- **Evaluation assumes access to human text**: Computing TPR@FPR and AUROC requires a parallel corpus of human-written text. The quality of evaluation depends on this human text being representative of the target domain.
- **Ethical boundary**: This technique is for defensive research -- auditing and improving detector robustness. It should not be applied to bypass detection for academic dishonesty, fraud, or deception.

## Reference

**Paper**: [StealthRL: Reinforcement Learning Paraphrase Attacks for Multi-Detector Evasion of AI-Text Detectors](https://arxiv.org/abs/2602.08934v1) (Ranganath & Ramesh, 2026)

Key sections to study: Section 3 for the GRPO reward formulation and LoRA configuration, Section 4 for the M0-M5 attack taxonomy, and Section 5.3 for the transferability analysis showing shared architectural vulnerabilities across detector families.

**Code**: [github.com/suraj-ranganath/StealthRL](https://github.com/suraj-ranganath/StealthRL) -- MIT licensed. Use `configs/stealthrl_small.yaml` as a starting configuration and `scripts/run_stealthbench.py` for the full evaluation pipeline.