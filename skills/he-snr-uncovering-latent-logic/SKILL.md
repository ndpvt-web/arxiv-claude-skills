---
name: "he-snr-uncovering-latent-logic"
description: "Evaluate and optimize LLM training data quality for software engineering tasks using the HE-SNR (High-Entropy Signal-to-Noise Ratio) metric. Analyzes token-level entropy distributions to predict downstream SWE-bench performance far more reliably than perplexity. Trigger phrases: 'compute HE-SNR', 'evaluate training data quality', 'entropy analysis of model outputs', 'filter SWE training data', 'measure model reasoning capability', 'mid-training evaluation metric'."
---

# HE-SNR: Entropy-Based Training Data Quality and Model Capability Assessment

This skill enables Claude to apply the HE-SNR (High-Entropy Signal-to-Noise Ratio) metric from the paper "Uncovering Latent Logic via Entropy for Guiding Mid-Training on SWE-BENCH." HE-SNR measures how well a language model resolves ambiguous coding decisions — not by checking whether it always picks the single best token, but by examining whether its uncertainty concentrates into small, structured candidate sets ("reasonable hesitation") rather than diffuse noise. This provides a robust predictor of software engineering capability that avoids the failure modes of standard perplexity, especially under long-context scaling.

## When to Use

- When the user wants to evaluate whether a training dataset will improve an LLM's software engineering capability
- When building a data filtering pipeline for mid-training or continued pre-training on code
- When the user asks to compare model checkpoints during training without running full SWE-bench evaluations
- When analyzing token-level prediction patterns to diagnose where a model struggles with code reasoning
- When the user needs to detect the "Long-Context Tax" — where perplexity degrades after context window extension but real capability improves
- When curating SWE-bench-style trajectory data and deciding which tokens carry genuine signal versus formatting noise
- When the user asks "is my model actually getting better at coding?" during a training run

## Key Technique

### The Problem with Perplexity

Standard perplexity (PPL) averages log-loss across all tokens equally. In software engineering contexts, this is misleading for two reasons. First, most tokens in code trajectories are low-entropy boilerplate — imports, XML tags, whitespace, markdown formatting — that contribute noise to the metric. Second, when context windows are extended (e.g., 32K to 128K) via RoPE scaling, attention patterns temporarily lose sharpness. PPL spikes even though the model's downstream SWE-bench performance may actually improve. This "Long-Context Tax" makes PPL an unreliable compass for mid-training.

### The Entropy Compression Hypothesis

HE-SNR is grounded in a fundamental reframing: intelligence is not just top-1 compression (always predicting the single right token) but the capacity to structure uncertainty into "Entropy-Compressed States" at natural boundaries. When a model encounters a genuinely ambiguous coding decision — which API to call, which variable name to use, which error-handling pattern to apply — a capable model narrows its uncertainty to exactly k plausible candidates with roughly uniform probability among them, producing entropy near ln(k). The key boundaries are ln(2) ~ 0.69 (binary decisions), ln(3) ~ 1.10 (three-way choices), and ln(4) ~ 1.39 (four-way choices). A model that compresses its hesitation to these structured states demonstrates genuine reasoning about code; one that spreads probability diffusely across many tokens does not.

### The HE-SNR Metric

HE-SNR focuses exclusively on "High-Entropy Decision Tokens" — the subset of tokens where the model genuinely hesitates (entropy above a threshold epsilon) AND the correct answer is among the model's top-10 candidates. The formula is: `HE-SNR = (1/|H|) * sum_{t in H} p(x_t) / H_top10(x_t)`, where `p(x_t)` is the probability assigned to the correct token (signal) and `H_top10(x_t)` is the top-10 entropy (noise). The threshold epsilon is set at `(ln(3) + ln(4)) / 2 ~ 1.24`, separating structured low-order hesitation from higher-entropy decisions. This metric shows strict linear correlation with SWE-bench Pass@1 across model scales and context windows, unlike PPL.

## Step-by-Step Workflow

1. **Curate evaluation trajectories.** Collect 200-500 successful SWE-bench problem-solving trajectories (or equivalent code-reasoning traces). Each trajectory should contain the full context: issue description, codebase exploration, and the patch/solution. Aim for ~10-15M tokens total.

2. **Apply structural data filtering.** For each trajectory, mask out observation/context blocks (tool outputs, file contents shown to the model) — these are given context, not model decisions. Retain only "action" tokens: the model's own code edits, commands, and reasoning steps. Remove XML/markdown tags, code comments, whitespace, newlines, and decorative symbols using AST-level parsing where possible.

3. **Compute token-level top-10 entropy.** For each retained token `x_t`, extract the model's top-10 predicted probabilities `p_1...p_10`, normalize them to sum to 1 as `p_hat_i = p_i / sum(p_j)`, then compute `H_top10(x_t) = -sum(p_hat_i * ln(p_hat_i))`. This captures 99.6%+ of the probability mass in practice.

4. **Identify the High-Entropy Decision Set.** A token enters the decision set H if two conditions hold: (a) `H_top10(x_t) > epsilon` where `epsilon = (ln(3) + ln(4)) / 2 ~ 1.2425`, and (b) the ground-truth token `x_t` appears among the model's top-10 candidates. Tokens failing either condition are excluded — low-entropy tokens are trivial, and tokens where the answer isn't in top-10 are beyond structured hesitation.

5. **Compute HE-SNR.** For each token in H, calculate the ratio `p(x_t) / H_top10(x_t)` — the correct-token probability divided by the local entropy. Average these ratios across all tokens in H. This is the HE-SNR score for the checkpoint.

6. **Track HE-SNR across checkpoints.** Plot HE-SNR at each training checkpoint. A rising HE-SNR means the model is learning to resolve ambiguous code decisions with higher confidence while maintaining structured candidate sets. A falling HE-SNR indicates degradation even if PPL improves.

7. **Analyze entropy distribution histograms.** Beyond the scalar HE-SNR, plot the distribution of `H_top10` values across all action tokens. Look for peaks at the natural boundaries (ln(2), ln(3), ln(4)). A capable model shows sharper peaks at lower orders — the "Shift to ln(3)" phenomenon where probability mass migrates from diffuse high-entropy states to structured three-candidate decisions.

8. **Compare against PPL as a sanity check.** Compute standard PPL on the same filtered token set (HE-PPL). If PPL and HE-SNR diverge — especially after context window extension — trust HE-SNR. The divergence itself is diagnostic of the Long-Context Tax.

9. **Use results to guide data selection.** Training data that improves HE-SNR on the evaluation set is genuinely improving the model's code reasoning. Data that improves PPL but not HE-SNR may only be teaching surface patterns. Prioritize data mixtures that drive HE-SNR upward.

## Concrete Examples

**Example 1: Evaluating a mid-training checkpoint after context extension**

User: We just extended our model from 32K to 128K context using YaRN. Perplexity spiked from 3.2 to 4.1 on our code eval set. Did we break the model?

Approach:
1. Load the 32K and 128K checkpoints
2. Run both against the curated SWE trajectory evaluation set
3. Apply structural filtering to isolate action tokens
4. Compute HE-SNR for both checkpoints

```python
import torch
import numpy as np

def compute_he_snr(logits, target_ids, epsilon=1.2425):
    """
    logits: (seq_len, vocab_size) - model output logits
    target_ids: (seq_len,) - ground truth token ids
    epsilon: entropy threshold, (ln3 + ln4) / 2
    """
    # Get top-10 probabilities and indices
    probs = torch.softmax(logits, dim=-1)
    top10_probs, top10_ids = torch.topk(probs, k=10, dim=-1)

    # Normalize top-10 to sum to 1
    top10_normed = top10_probs / top10_probs.sum(dim=-1, keepdim=True)

    # Compute top-10 entropy per token
    h_top10 = -(top10_normed * torch.log(top10_normed)).sum(dim=-1)

    # Check if target is in top-10
    target_in_top10 = (top10_ids == target_ids.unsqueeze(-1)).any(dim=-1)

    # Build high-entropy decision set
    decision_mask = (h_top10 > epsilon) & target_in_top10

    if decision_mask.sum() == 0:
        return 0.0, 0

    # Get probability of correct token at decision points
    target_probs = probs[torch.arange(len(target_ids)), target_ids]

    # Compute HE-SNR: average of p(x_t) / H_top10(x_t) over decision set
    ratios = target_probs[decision_mask] / h_top10[decision_mask]
    he_snr = ratios.mean().item()

    return he_snr, decision_mask.sum().item()
```

Output:
```
32K checkpoint: PPL=3.2, HE-SNR=0.147, |H|=48,203 decision tokens
128K checkpoint: PPL=4.1, HE-SNR=0.153, |H|=51,847 decision tokens

Diagnosis: PPL degraded due to Long-Context Tax (YaRN disrupts attention
patterns temporarily), but HE-SNR IMPROVED. The model is actually better
at resolving ambiguous code decisions with the longer context. The PPL
spike is a false alarm. Continue training.
```

**Example 2: Filtering a training dataset for code reasoning quality**

User: I have 50GB of code-related training data. How do I filter it to maximize SWE capability?

Approach:
1. Apply the structural filtering pipeline to remove noise tokens
2. Score each document by the proportion of high-entropy decision tokens it produces
3. Rank documents by their contribution to HE-SNR

```python
def filter_code_tokens(tokens, token_types):
    """
    Remove noise tokens that don't reflect model reasoning.
    token_types: list of labels from AST/structural analysis
    """
    noise_types = {
        'whitespace', 'newline', 'xml_tag', 'markdown_format',
        'code_comment', 'decorative_symbol', 'observation_context'
    }
    mask = [t not in noise_types for t in token_types]
    return [tok for tok, keep in zip(tokens, mask) if keep]

def score_document_quality(model, document_tokens, epsilon=1.2425):
    """Score a document by its density of meaningful decision tokens."""
    logits = model(document_tokens)
    probs = torch.softmax(logits, dim=-1)
    top10_probs, top10_ids = torch.topk(probs, k=10, dim=-1)
    top10_normed = top10_probs / top10_probs.sum(dim=-1, keepdim=True)
    h_top10 = -(top10_normed * torch.log(top10_normed)).sum(dim=-1)

    target_in_top10 = (top10_ids == document_tokens[1:].unsqueeze(-1)).any(dim=-1)
    decision_density = ((h_top10 > epsilon) & target_in_top10).float().mean()

    return decision_density.item()
```

Output:
```
Scored 12,000 documents. Distribution:
- Top 20% (decision density > 0.15): Complex bug fixes, multi-file refactors
- Middle 40% (0.05-0.15): Standard coding tasks, API usage
- Bottom 40% (< 0.05): Boilerplate, config files, trivial edits

Recommendation: Oversample the top 20% at 3x weight during training.
These documents force the model to make genuinely ambiguous decisions
that build SWE reasoning capability.
```

**Example 3: Diagnosing entropy distribution shifts during training**

User: Our model's HE-SNR plateaued. What's happening at the entropy level?

Approach:
1. Compute entropy histograms at the current and previous checkpoints
2. Look for the "Shift to ln(3)" phenomenon
3. Identify where probability mass is stuck

```python
def entropy_histogram(model, eval_tokens, bins=50):
    """Plot entropy distribution with natural boundary markers."""
    logits = model(eval_tokens)
    probs = torch.softmax(logits, dim=-1)
    top10_probs, _ = torch.topk(probs, k=10, dim=-1)
    top10_normed = top10_probs / top10_probs.sum(dim=-1, keepdim=True)
    h_top10 = -(top10_normed * torch.log(top10_normed)).sum(dim=-1)

    boundaries = {
        'ln(2)': 0.6931,  # Binary decisions
        'ln(3)': 1.0986,  # Three-way choices
        'ln(4)': 1.3863,  # Four-way choices
        'ln(10)': 2.3026, # Diffuse/stochastic
    }
    return h_top10.numpy(), boundaries
```

Output:
```
Checkpoint 5000: Peak at ln(4), broad tail past ln(10)
Checkpoint 8000: Peak shifted to ln(3), tail shrinking
Checkpoint 11000 (current): Same ln(3) peak, tail unchanged

Diagnosis: The model learned to compress 4-way decisions to 3-way
(good progress early on) but has stopped compressing further.
The persistent tail past ln(10) suggests a subset of token patterns
where the model hasn't learned structure. Investigate those tokens —
they likely correspond to specific code patterns missing from
training data.
```

## Best Practices

- **Do:** Always filter out observation/context tokens before computing HE-SNR. Including tool outputs and file contents in the metric dilutes it with tokens the model didn't actually decide on.
- **Do:** Use AST-level parsing when filtering code tokens rather than regex. Regex misses nested structures and can accidentally strip meaningful code that looks like formatting.
- **Do:** Track both HE-SNR and the decision set size |H| separately. A rising HE-SNR with shrinking |H| might mean the model is only improving on easy decisions while avoiding hard ones.
- **Do:** Set epsilon at `(ln(3) + ln(4)) / 2 ~ 1.2425` as the paper validates. This threshold separates structured low-order hesitation from higher-entropy noise.
- **Avoid:** Using raw perplexity as the sole guide for mid-training decisions, especially after context window modifications. PPL is compromised by the Long-Context Tax.
- **Avoid:** Computing HE-SNR on fewer than 200 trajectories. The metric needs sufficient high-entropy decision tokens to be stable — aim for at least 40K tokens in the decision set.

## Error Handling

**Empty decision set (|H| = 0):** If no tokens pass both the entropy threshold and the top-10 membership check, the model may be severely undertrained on the domain. Lower epsilon temporarily to `ln(2) ~ 0.69` to capture binary decision tokens, or verify the evaluation data contains genuine coding decisions rather than just boilerplate.

**HE-SNR and PPL move in opposite directions:** This is expected during context window extension (Long-Context Tax). Trust HE-SNR. If it happens without context changes, investigate whether the training data has shifted to contain more surface-pattern-heavy content that reduces PPL without improving reasoning.

**Entropy histogram shows no peaks at natural boundaries:** The model has not yet learned to compress uncertainty into structured states. This typically occurs very early in training. Continue training and monitor — peaks should emerge as the model acquires domain structure.

**Inconsistent HE-SNR across evaluation runs:** Ensure the data filtering pipeline is deterministic. Non-determinism in AST parsing or token-to-character offset alignment can cause different tokens to be included/excluded across runs.

## Limitations

- HE-SNR is validated specifically for software engineering tasks (SWE-bench). Its correlation with downstream performance in other domains (math, creative writing, general QA) is not established by this paper.
- The metric requires access to model logits (full probability distributions), not just generated text. It cannot be computed via API-only access to closed-source models.
- The curated evaluation set of SWE trajectories requires significant upfront investment to build. Poor trajectory quality will produce misleading HE-SNR values.
- HE-SNR uses top-10 truncation, which covers 99.6% of probability mass in practice but may miss edge cases in models with unusually flat distributions.
- The epsilon threshold of 1.2425 is empirically validated on MoE architectures. Dense models or significantly different architectures may require recalibration.

## Reference

**Paper:** [HE-SNR: Uncovering Latent Logic via Entropy for Guiding Mid-Training on SWE-BENCH](https://arxiv.org/abs/2601.20255v1) (Wang et al., 2026). Key sections: Section 3 for the Entropy Compression Hypothesis formalization, Section 4 for the HE-SNR metric derivation, and Figures 5-6 for the empirical validation showing HE-SNR's linear correlation with SWE-bench Pass@1 where PPL fails.