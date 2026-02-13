---
name: "frost-filtering-reasoning-outliers"
description: "Implement FROST (Filtering Reasoning Outliers with Attention) to prune unnecessary reasoning steps from LLM chain-of-thought outputs using attention-based scoring and Softmax1. Use when: 'optimize reasoning chain', 'reduce token usage in CoT', 'prune reasoning steps', 'filter reasoning outliers', 'implement attention-based reasoning pruning', 'make chain-of-thought more efficient'."
---

# FROST: Filtering Reasoning Outliers with Attention for Efficient Reasoning

FROST enables Claude to help users implement attention-aware reasoning pruning systems that identify and remove unnecessary steps from chain-of-thought (CoT) outputs. The core technique replaces standard softmax with Softmax1 (adding a +1 constant to the denominator), uses sentence-level attention aggregation to score each reasoning step's contribution to the final answer, and suppresses low-contribution steps. This yields dramatically shorter reasoning traces (up to 70% token reduction) while improving accuracy by eliminating noise from redundant verification, self-checking loops, and repetitive restatements.

## When to Use

- When the user wants to reduce token costs of reasoning models (e.g., o1-style, Phi-4-Reasoning, DeepSeek-R1) by pruning verbose CoT outputs
- When building a post-processing pipeline that filters generated reasoning traces before presenting them to users
- When implementing custom attention mechanisms that suppress outlier activations in transformer models
- When fine-tuning a reasoning model with LoRA to produce more concise, higher-quality reasoning chains
- When the user asks to detect which reasoning steps actually matter in a multi-step chain-of-thought
- When implementing Softmax1 or attention sink mechanisms in a custom transformer architecture
- When analyzing attention distributions to diagnose reasoning quality issues (high kurtosis, extreme infinity norms)

## Key Technique

**Reasoning outliers** are sentences in a chain-of-thought trace that receive low attention weight and contribute negligibly to the final answer token. These typically manifest as redundant re-derivations, unnecessary self-checks ("Let me verify..."), circular restatements, or tangential explorations. FROST identifies them by measuring each sentence's aggregated attention flow toward the answer token (or the `</think>` sink token), then suppresses sentences below a contribution threshold.

The mechanism has two parts. First, **Softmax1** replaces standard softmax with `exp(x_i) / (sum_j exp(x_j) + 1)`. The extra +1 in the denominator acts as a learnable baseline: if no token is strongly relevant, attention mass flows to the implicit "null" option rather than being forced onto irrelevant tokens. This contracts the tail of the attention distribution (kurtosis drops ~91%) while preserving the ranking of genuinely important tokens. Unlike Sparsemax or Entmax which sharpen bidirectionally (risking loss of critical signals), Softmax1 performs selective tail contraction only.

Second, **sentence-level aggregation** pools token-level attention scores into sentence scores using a monotone operator (sum, mean, logsumexp, or max). The theoretical guarantee is that if every token in sentence A has higher attention than every token in sentence B, then sentence A's aggregated score exceeds B's. Sentences with aggregated attention probability below threshold epsilon contribute at most O(epsilon) perturbation to final logits across all L transformer layers, making them safe to prune without degrading output quality.

## Step-by-Step Workflow

1. **Segment the reasoning trace into sentences.** Split the model's chain-of-thought output at sentence boundaries (periods, newlines, or step markers like "Step 1:", "Therefore,"). Each segment becomes a scoring unit. Preserve the question prefix (Q) and final answer (A) as anchors that are never pruned.

2. **Extract token-level attention weights.** For each token `t_i` in the reasoning trace, extract `a_{i,A}` -- the attention weight from `t_i` to the final answer token (or `</think>` token if using a think-tag model). Use the last attention layer or average across the final K layers for stability.

3. **Aggregate to sentence-level scores.** For each sentence `S_j` containing tokens `{t_1, ..., t_n}`, compute the sentence score: `W_j = sum(a_{i,A} for t_i in S_j)`. Alternatives: use `mean()` for length-normalized scoring or `logsumexp()` for smooth approximation of max.

4. **Normalize sentence scores.** Apply Softmax1 to the sentence score vector: `alpha_j = exp(W_j) / (sum_k exp(W_k) + 1)`. The +1 term ensures that if all sentences are equally irrelevant, no single sentence is artificially boosted.

5. **Prune low-attention sentences.** Remove sentences where `alpha_j <= epsilon`. Start with `epsilon = 0.01` (1% of total attention mass) and tune based on your accuracy-brevity tradeoff. Alternatively, keep the top-K sentences by score if you want a fixed compression ratio.

6. **Reconstruct the filtered reasoning trace.** Concatenate the surviving sentences in their original order, preserving logical flow. Prepend the original question and append the original final answer.

7. **Validate reasoning coherence.** Check that the filtered trace still contains the key logical transitions needed to reach the answer. If a pruned sentence contained a variable definition or equation referenced later, restore it.

8. **For fine-tuning (optional): Replace softmax with Softmax1 in attention layers.** Modify the model's attention computation to use `softmax1(x) = exp(x) / (sum exp(x) + 1)` instead of standard softmax. Apply LoRA adapters (rank=8, alpha=16) and fine-tune for ~5000 steps on reasoning data with lr=1e-5.

9. **Measure outlier metrics.** Compute the maximum infinity norm (`max absolute activation`) and kurtosis of the activation distribution across layers. Lower values indicate successful outlier suppression. Target: infinity norm reduction >10%, kurtosis reduction >50%.

10. **Iterate threshold selection.** Evaluate on a held-out set: measure accuracy and token count at several epsilon values (0.005, 0.01, 0.02, 0.05). Select the epsilon that maximizes accuracy while minimizing token usage.

## Concrete Examples

**Example 1: Post-processing a verbose math reasoning trace**

User: "I have a reasoning model that generates 2000-token CoT traces for simple algebra. Help me implement FROST-style filtering to cut the fluff."

Approach:
1. Parse the model's output, splitting at sentence boundaries
2. Run the same prompt through the model with `output_attentions=True`
3. Extract attention weights from the last layer, focusing on attention to the final answer token
4. Aggregate per-sentence and apply Softmax1 normalization
5. Filter sentences below threshold

```python
import torch
import torch.nn.functional as F

def softmax1(x: torch.Tensor, dim: int = -1) -> torch.Tensor:
    """Softmax with +1 denominator -- the core FROST activation."""
    e_x = torch.exp(x - x.max(dim=dim, keepdim=True).values)
    return e_x / (e_x.sum(dim=dim, keepdim=True) + 1.0)

def frost_filter(
    text: str,
    attention_weights: torch.Tensor,  # shape: (num_tokens, num_tokens)
    tokenizer,
    epsilon: float = 0.01,
) -> str:
    """Filter reasoning trace using FROST attention scoring."""
    # Split into sentences
    sentences = segment_into_sentences(text)

    # Map sentences to token spans
    token_spans = map_sentences_to_token_spans(sentences, tokenizer)

    # Get attention to final answer token (last non-padding token)
    answer_idx = attention_weights.shape[-1] - 1
    attn_to_answer = attention_weights[:, answer_idx]  # (num_tokens,)

    # Aggregate per sentence using sum
    sentence_scores = []
    for start, end in token_spans:
        score = attn_to_answer[start:end].sum().item()
        sentence_scores.append(score)

    # Apply Softmax1 normalization
    scores_tensor = torch.tensor(sentence_scores)
    alpha = softmax1(scores_tensor, dim=0)

    # Keep sentences above threshold (always keep first and last)
    filtered = [sentences[0]]  # Always keep question/setup
    for i in range(1, len(sentences) - 1):
        if alpha[i].item() > epsilon:
            filtered.append(sentences[i])
    filtered.append(sentences[-1])  # Always keep final answer

    return " ".join(filtered)
```

Output: A trace reduced from 2000 tokens to ~600 tokens, retaining the critical derivation steps and dropping redundant verifications like "Let me double-check by substituting back..." and "We can confirm that...".

**Example 2: Replacing softmax with Softmax1 in a HuggingFace model**

User: "I want to fine-tune Qwen2.5-Math with FROST's Softmax1 attention. Show me how to patch the attention layer."

Approach:
1. Identify the attention class in the model
2. Replace the softmax call with Softmax1
3. Add LoRA adapters for parameter-efficient fine-tuning

```python
import torch
from torch import nn
from transformers import AutoModelForCausalLM
from peft import get_peft_model, LoraConfig

def patch_attention_with_softmax1(model):
    """Replace standard softmax with Softmax1 in all attention layers."""
    for name, module in model.named_modules():
        if hasattr(module, 'attn_dropout') and hasattr(module, 'num_heads'):
            # Store reference to original forward
            original_forward = module.forward

            def make_patched_forward(orig_fn):
                def patched_forward(*args, **kwargs):
                    # Intercept by temporarily replacing F.softmax
                    original_softmax = F.softmax
                    F.softmax = lambda x, dim=-1, **kw: softmax1(x, dim=dim)
                    try:
                        result = orig_fn(*args, **kwargs)
                    finally:
                        F.softmax = original_softmax
                    return result
                return patched_forward

            module.forward = make_patched_forward(original_forward)
    return model

# Usage
model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-Math-1.5B")
model = patch_attention_with_softmax1(model)

lora_config = LoraConfig(
    r=8, lora_alpha=16,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
    lora_dropout=0.05,
    task_type="CAUSAL_LM",
)
model = get_peft_model(model, lora_config)
# Fine-tune with standard SFT: lr=1e-5, 5000 steps, batch_size=8
```

**Example 3: Diagnosing reasoning quality via attention outlier metrics**

User: "My fine-tuned model's CoT outputs are getting worse. How can I use FROST's metrics to diagnose the problem?"

Approach:
1. Collect activations across all layers during inference
2. Compute infinity norm and kurtosis per layer
3. Compare against baseline to identify outlier explosion

```python
import torch
from scipy.stats import kurtosis
import numpy as np

def compute_frost_diagnostics(model, input_ids):
    """Compute FROST attention outlier metrics for diagnosis."""
    hooks, activations = [], {}

    def make_hook(name):
        def hook_fn(module, input, output):
            activations[name] = output[0].detach().cpu()
        return hook_fn

    # Register hooks on all attention output projections
    for name, module in model.named_modules():
        if name.endswith('.o_proj'):
            hooks.append(module.register_forward_hook(make_hook(name)))

    with torch.no_grad():
        model(input_ids)

    for h in hooks:
        h.remove()

    # Compute metrics
    max_inf_norm = 0.0
    kurtosis_values = []

    for name, act in activations.items():
        flat = act.float().numpy().flatten()
        layer_inf_norm = np.max(np.abs(flat))
        layer_kurtosis = kurtosis(flat, fisher=True)

        max_inf_norm = max(max_inf_norm, layer_inf_norm)
        kurtosis_values.append(layer_kurtosis)

    avg_kurtosis = np.mean(kurtosis_values)

    return {
        "max_infinity_norm": max_inf_norm,
        "avg_kurtosis": avg_kurtosis,
        "per_layer_kurtosis": dict(zip(activations.keys(), kurtosis_values)),
    }

# Interpretation:
# - max_infinity_norm > 35: likely has attention outliers degrading reasoning
# - avg_kurtosis > 100: heavy-tailed activations; consider Softmax1 adaptation
# - Healthy FROST model targets: inf_norm < 30, kurtosis < 25
```

## Best Practices

- **Do** always preserve the first sentence (problem statement) and last sentence (final answer) when pruning -- these are anchors, not candidates for removal.
- **Do** use sum aggregation as the default pooling operator for sentence scores; it naturally favors longer, more substantive reasoning steps over short filler sentences.
- **Do** subtract the max logit before computing Softmax1 for numerical stability: `exp(x - max(x)) / (sum exp(x - max(x)) + 1)`.
- **Do** validate that pruned traces remain logically coherent -- check for dangling variable references or broken equation chains after filtering.
- **Avoid** applying hard pruning during training; Softmax1 handles suppression implicitly through gradient flow. Hard cutoffs during training break gradient propagation.
- **Avoid** using bidirectional sharpening functions like Sparsemax for this task -- they can inadvertently zero out critical reasoning steps along with outliers. Softmax1's asymmetric tail contraction is safer.
- **Avoid** pruning on models that produce very short CoT traces (<200 tokens). FROST is designed for verbose reasoning models where redundancy is the bottleneck.

## Error Handling

- **NaN in Softmax1 computation**: If input logits contain extreme values (>1e4), the exp() call overflows. Always apply max-subtraction normalization before Softmax1. If NaNs persist after normalization, clamp inputs to [-100, 100].
- **All sentences pruned**: If epsilon is set too high, all intermediate sentences may fall below threshold. Implement a fallback: if fewer than 2 reasoning sentences survive, halve epsilon and re-filter.
- **Token-sentence alignment errors**: Tokenizer byte-pair encoding may split sentence boundaries mid-token. Use character-level offsets from the tokenizer's `offset_mapping` to align sentences to token spans accurately.
- **Attention shape mismatches with KV-cache**: During autoregressive generation with KV-cache, attention matrices are not square. Extract attention weights from the full prompt pass (without cache) for accurate scoring.
- **Gradient instability during Softmax1 fine-tuning**: The +1 denominator can cause gradient magnitudes to differ from standard softmax. Use gradient clipping (max_norm=1.0) and start with a lower learning rate (1e-5).

## Limitations

- FROST requires access to attention weights, which means it cannot be applied as a black-box filter to closed API models (GPT-4, Claude). It works only when you have model internals access.
- The technique is validated primarily on mathematical reasoning benchmarks (AIME, MATH-500, AMC, Minerva). Effectiveness on free-form reasoning, coding, or multi-hop QA is not empirically established.
- Sentence segmentation quality directly affects filtering quality. Poorly segmented traces (e.g., code blocks, tables, multi-line equations) may lead to inappropriate pruning.
- The method assumes a clear "final answer" token exists as an attention sink. Models without explicit answer delimiters require adaptation of the anchor token selection.
- Fine-tuning with Softmax1 requires ~5000 steps of supervised training. For inference-only pruning (no fine-tuning), the attention scoring is less reliable since the model wasn't trained to concentrate attention appropriately.

## Reference

**Paper**: [FROST: Filtering Reasoning Outliers with Attention for Efficient Reasoning](https://arxiv.org/abs/2601.19001v1) -- Luo et al., 2026. Look for Section 4 (Softmax1 formulation and theoretical guarantees), Section 5 (monotone pooling and deployment-time suppression theorems), and Table 1 (benchmark results across AIME, MATH-500, AMC, Minerva).

**Code**: [github.com/robinzixuan/FROST](https://github.com/robinzixuan/FROST) -- Reference implementation with custom attention modules for Qwen, Phi, and GPT architectures.