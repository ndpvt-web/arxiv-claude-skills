---
name: "why-attention-patterns-exist"
description: >
  Analyze, classify, and optimize LLM attention heads using the TAPPA framework
  (Temporal Attention Pattern Predictability Analysis). Applies query self-similarity
  metrics to diagnose attention behavior, guide KV cache compression budget allocation,
  and inform structural pruning decisions.
  Trigger phrases: "analyze attention patterns", "classify attention heads",
  "optimize KV cache budget", "diagnose attention heads", "prune attention heads",
  "attention pattern analysis"
---

# TAPPA: Temporal Attention Pattern Predictability Analysis

This skill enables Claude to apply the TAPPA framework from Yang et al. (2026) to analyze, classify, and optimize attention heads in large language models. TAPPA unifies previously fragmented observations about attention patterns -- retrieval heads, sink heads, diagonal/recency traces, homogeneous heads, and vertical structures -- under a single explanatory principle: **query self-similarity along the temporal dimension** (q-similarity). By computing how consistently a head's query vectors evolve across decoding steps, you can predict whether a head exhibits stable, compressible patterns or volatile, information-rich behavior, then use that signal to make principled decisions about KV cache budgets and structural pruning.

## When to Use

- When the user wants to understand why certain attention heads in their model behave differently (sinks, retrieval, recency, etc.)
- When optimizing KV cache memory by allocating per-head token retention budgets
- When performing structural pruning to remove redundant attention heads or layers
- When debugging a model's attention behavior during fine-tuning or inference
- When the user asks "which heads matter most" or "where should I allocate cache budget"
- When building custom attention-aware inference engines that need head-level profiling
- When comparing attention behavior across model checkpoints or architectures

## Key Technique

**The Core Insight: Query Self-Similarity (Q-Similarity)**

Standard attention computes `softmax(QK^T / sqrt(d_k)) V`. TAPPA's insight is to stop analyzing the *output* attention matrix and instead analyze the *input* queries temporally. For each head, compute the cosine similarity between query vectors at consecutive decoding steps t and t+1. Heads where queries change slowly (high q-similarity) produce predictable attention patterns -- these are your sink heads, diagonal heads, and homogeneous heads. Heads where queries shift rapidly (low q-similarity) produce unpredictable, content-dependent patterns -- these are your retrieval heads that actually perform semantic lookup.

**Why This Matters for Optimization**

This distinction directly maps to resource allocation. High q-similarity heads are predictable: their attention targets can be anticipated, so you can aggressively compress their KV cache (fewer tokens retained) without quality loss. Low q-similarity heads are unpredictable: they perform genuine information retrieval, so they need larger cache budgets to preserve the tokens they might attend to. This inversion -- give *more* memory to *less* predictable heads -- is counterintuitive but consistently outperforms uniform allocation baselines. For pruning, the logic flips: high q-similarity heads contribute less unique computation and are safer to remove entirely.

**The Five Pattern Categories**

TAPPA classifies heads into: (1) **Retrieval heads** -- low q-similarity, content-focused attention on semantically relevant tokens; (2) **Sink heads** -- high q-similarity, persistent attention on BOS/special tokens; (3) **Recency/diagonal heads** -- high q-similarity, attention concentrated on recent positions via a sliding window; (4) **Homogeneous heads** -- high q-similarity, near-uniform attention distribution; (5) **Vertical structure heads** -- position-specific, head-independent patterns suggesting layer-level specialization.

## Step-by-Step Workflow

1. **Extract attention weights and query vectors.** Run a calibration sequence through the model with hooks on every attention layer. Capture both the raw query tensors Q (shape: `[num_heads, seq_len, head_dim]`) and the post-softmax attention weight matrices A (shape: `[num_heads, seq_len, seq_len]`).

2. **Compute per-head q-similarity scores.** For each head h in each layer l, compute the mean pairwise cosine similarity between consecutive query vectors: `q_sim(l, h) = mean(cos_sim(q_t, q_{t+1}))` for t in [1, T-1]. This yields a single scalar per head summarizing temporal query stability.

3. **Classify heads into pattern categories.** Apply threshold-based or clustering-based classification using q-similarity scores combined with attention entropy: high q-sim + low entropy = sink or recency head; high q-sim + high entropy = homogeneous head; low q-sim + variable entropy = retrieval head. Visualize the attention matrices of representative heads from each cluster to confirm.

4. **Rank heads by q-similarity.** Sort all heads across all layers by their q-similarity score. This ranking is the foundation for both cache budget allocation and pruning priority.

5. **Allocate KV cache budgets (for inference optimization).** Given a total token budget B across all heads, assign per-head budgets inversely proportional to q-similarity: `budget(l, h) = B * (1 - q_sim(l, h)) / sum(1 - q_sim)`. Heads with low q-similarity (retrieval heads) get more tokens; heads with high q-similarity (sink/recency heads) get fewer.

6. **Configure head-specific retention policies.** For each pattern type, apply the appropriate eviction strategy within its allocated budget: sink heads retain only the first K tokens; recency heads use a sliding window of size W; retrieval heads use a full recent-window plus importance-scored historical tokens; homogeneous heads can use aggressive random sampling.

7. **Validate on downstream benchmarks.** Run the model with the new per-head cache budgets on evaluation tasks (perplexity, downstream accuracy). Compare against uniform-budget baselines and methods like H2O, SnapKV, or PyramidKV.

8. **For pruning: remove high q-similarity heads first.** Mark heads above a q-similarity threshold as pruning candidates. Remove them layer by layer, evaluating after each removal to catch any unexpected quality drops. High q-similarity heads are the safest to prune because their behavior is predictable and often redundant.

9. **Iterate and profile across inputs.** Q-similarity scores can vary by input distribution. Run the profiling on a representative calibration set (512-2048 tokens from target domain) to get robust estimates before deploying.

## Concrete Examples

**Example 1: Profiling attention heads in Llama-3.1-8B**

User: "I want to understand which attention heads in my Llama-3.1-8B model are doing useful retrieval vs. just sinking to BOS tokens."

Approach:
1. Load the model and register forward hooks on all 32 layers x 32 heads.
2. Run 5-10 calibration sequences (each ~512 tokens) from the target domain.
3. Compute q-similarity for all 1024 heads.
4. Classify and report.

Output:
```python
import torch
import torch.nn.functional as F

def compute_q_similarity(model, input_ids, device="cuda"):
    """Compute per-head query self-similarity across decoding steps."""
    q_sims = {}  # (layer, head) -> similarity score
    queries = {}  # (layer, head) -> list of query vectors

    # Register hooks to capture queries
    hooks = []
    for layer_idx, layer in enumerate(model.model.layers):
        def hook_fn(module, input, output, layer_idx=layer_idx):
            # Extract Q from attention input (model-specific)
            hidden = input[0]
            q = module.q_proj(hidden)  # [batch, seq, num_heads * head_dim]
            batch, seq, _ = q.shape
            head_dim = module.head_dim
            num_heads = module.num_heads
            q = q.view(batch, seq, num_heads, head_dim)
            for h in range(num_heads):
                key = (layer_idx, h)
                if key not in queries:
                    queries[key] = []
                # Store per-position query vectors
                queries[key].append(q[0, :, h, :].detach().cpu())
        hooks.append(layer.self_attn.register_forward_hook(hook_fn))

    # Forward pass
    with torch.no_grad():
        model(input_ids.to(device))

    for hook in hooks:
        hook.remove()

    # Compute q-similarity per head
    for (layer_idx, head_idx), q_list in queries.items():
        q_cat = torch.stack(q_list, dim=0)  # [num_samples, seq, head_dim]
        q_flat = q_cat.view(-1, q_cat.shape[-1])  # flatten samples
        # Consecutive cosine similarity
        cos_sims = F.cosine_similarity(q_flat[:-1], q_flat[1:], dim=-1)
        q_sims[(layer_idx, head_idx)] = cos_sims.mean().item()

    return q_sims

# After computing:
# Sort heads by q-similarity
sorted_heads = sorted(q_sims.items(), key=lambda x: x[1])

print("=== LOW Q-SIM (Retrieval Heads - keep large cache) ===")
for (l, h), sim in sorted_heads[:10]:
    print(f"  Layer {l:2d}, Head {h:2d}: q_sim = {sim:.4f}")

print("\n=== HIGH Q-SIM (Sink/Recency Heads - safe to compress) ===")
for (l, h), sim in sorted_heads[-10:]:
    print(f"  Layer {l:2d}, Head {h:2d}: q_sim = {sim:.4f}")
```

**Example 2: KV cache budget allocation for inference**

User: "I need to reduce my KV cache memory by 60% without killing quality. How should I allocate budgets per head?"

Approach:
1. Profile the model's q-similarity scores on a calibration set.
2. Allocate the remaining 40% budget inversely proportional to q-similarity.
3. Configure per-head eviction policies.

Output:
```python
import numpy as np

def allocate_kv_budgets(q_sims, total_budget, num_layers, num_heads):
    """Allocate per-head KV cache token budgets using TAPPA q-similarity."""
    # Compute inverse q-similarity weights
    weights = {}
    for (l, h), sim in q_sims.items():
        weights[(l, h)] = 1.0 - sim  # Lower similarity -> higher weight

    total_weight = sum(weights.values())

    budgets = {}
    for (l, h), w in weights.items():
        raw_budget = total_budget * (w / total_weight)
        # Enforce minimum of 4 tokens (BOS + minimal context)
        budgets[(l, h)] = max(4, int(raw_budget))

    return budgets

def assign_eviction_policy(q_sims, budgets, sim_threshold_high=0.85):
    """Assign eviction strategy per head based on pattern type."""
    policies = {}
    for (l, h), sim in q_sims.items():
        b = budgets[(l, h)]
        if sim > sim_threshold_high:
            # Sink or recency head: sliding window + first token
            policies[(l, h)] = {"type": "sliding_window", "window": b - 1,
                                "keep_first": 1}
        else:
            # Retrieval head: importance-scored full history
            policies[(l, h)] = {"type": "importance_scored", "budget": b,
                                "score_fn": "cumulative_attention"}
    return policies

# Usage:
# total_budget = 0.4 * seq_len * num_layers * num_heads
# budgets = allocate_kv_budgets(q_sims, total_budget, 32, 32)
# policies = assign_eviction_policy(q_sims, budgets)
```

**Example 3: Identifying pruning candidates**

User: "Which heads can I safely prune from my Qwen2.5 model for faster inference?"

Approach:
1. Compute q-similarity across all heads.
2. Rank heads by q-similarity descending (highest = most redundant).
3. Prune iteratively, validating perplexity after each batch removal.

Output:
```python
def rank_pruning_candidates(q_sims, top_k=50):
    """Rank heads by pruning safety (highest q-sim = safest to remove)."""
    ranked = sorted(q_sims.items(), key=lambda x: x[1], reverse=True)
    print(f"Top {top_k} pruning candidates (highest q-similarity):\n")
    print(f"{'Layer':>6} {'Head':>6} {'Q-Sim':>8} {'Pattern':>14}")
    print("-" * 38)
    for (l, h), sim in ranked[:top_k]:
        pattern = ("sink/homog" if sim > 0.90 else
                   "recency" if sim > 0.80 else "mixed")
        print(f"{l:>6} {h:>6} {sim:>8.4f} {pattern:>14}")
    return [(l, h) for (l, h), _ in ranked[:top_k]]

# Iterative pruning loop:
# for batch in chunks(candidates, batch_size=10):
#     prune_heads(model, batch)
#     ppl = evaluate_perplexity(model, eval_data)
#     if ppl > threshold:
#         restore_heads(model, batch)
#         break
```

## Best Practices

- **Do:** Run q-similarity profiling on a calibration set representative of your deployment domain. Scores shift meaningfully between code, dialogue, and long-document inputs.
- **Do:** Combine q-similarity with attention entropy for more accurate head classification. Q-similarity alone distinguishes predictable from unpredictable, but entropy separates sink heads (low entropy, focused on BOS) from homogeneous heads (high entropy, spread uniformly).
- **Do:** Apply the inverse allocation principle -- give more cache budget to low q-similarity heads and prune high q-similarity heads first. This is the core actionable insight from TAPPA.
- **Do:** Set a minimum per-head budget (at least 4 tokens) even for highly predictable heads. Completely zeroing out a head's cache can cause cascading errors in downstream layers.
- **Avoid:** Computing q-similarity on very short sequences (<128 tokens). The metric needs sufficient temporal context to stabilize.
- **Avoid:** Treating q-similarity scores as static model properties. They vary with input distribution, so always profile on target-domain data before deployment.
- **Avoid:** Pruning or compressing all high q-similarity heads simultaneously without iterative validation. Even redundant-looking heads may have subtle interactions across layers.

## Error Handling

- **Hook registration fails on custom architectures:** Different model families name their attention projections differently (`q_proj`, `query`, `Wq`). Inspect the model's `named_modules()` to find the correct projection layer names before registering hooks.
- **Q-similarity scores are all near 1.0:** This typically means you're computing similarity on the wrong tensor (e.g., post-RoPE keys instead of raw queries) or using a calibration sequence that's too short or too homogeneous. Increase sequence length and use diverse calibration data.
- **Quality degrades unexpectedly after compression:** Some layers contain heads where q-similarity is moderate (~0.5-0.7) and pattern type is ambiguous. These borderline heads need conservative budgets. If quality drops, increase budgets for heads in this mid-range first.
- **OOM during profiling:** Store query vectors in float16 and process one layer at a time rather than hooking all layers simultaneously. Accumulate q-similarity incrementally.
- **RoPE interactions:** The paper shows that Rotary Positional Embeddings interact with q-similarity. When computing raw q-similarity, use pre-RoPE query vectors for cleaner pattern classification. Post-RoPE queries conflate positional and content signals.

## Limitations

- TAPPA's q-similarity metric is designed for autoregressive decoder models with causal attention. It does not directly apply to bidirectional encoders (BERT-style) where all positions attend to all others simultaneously.
- The framework has been validated primarily on Llama-3.1 and Qwen2.5 families. Other architectures (Mamba, RWKV, or models with grouped-query attention) may need adapted profiling logic.
- Q-similarity profiling requires a forward pass with hooks, adding overhead to the initial setup. This is a one-time cost per model-domain pair, not a per-request cost.
- The paper reports average improvements (e.g., +11.34 over EA baseline at budget 512 on Qwen2.5), but gains vary by task. Factual recall and multi-hop reasoning tasks are more sensitive to cache compression than summarization or classification.
- Head classification thresholds (e.g., 0.85 for "high" q-similarity) are empirical. They should be tuned per model family rather than used as universal constants.

## Reference

**Paper:** Yang, Q., Wang, J., Li, X., Bai, Y., & Tong, X. (2026). *Why Attention Patterns Exist: A Unifying Temporal Perspective Analysis.* arXiv:2601.21709v1. https://arxiv.org/abs/2601.21709v1

**Code:** https://github.com/MIRALab-USTC/LLM-TAPPA

**Key takeaway:** Query self-similarity (q-similarity) along the temporal dimension is the single metric that explains why different attention heads exhibit different patterns, and it directly translates into actionable budget allocation for KV cache compression and head pruning priority.