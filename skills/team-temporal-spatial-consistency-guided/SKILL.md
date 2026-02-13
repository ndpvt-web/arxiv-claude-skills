---
name: "team-temporal-spatial-consistency-guided"
description: "Accelerate MoE diffusion language model inference using TEAM's temporal-spatial consistency framework. Implements expert caching, hot/cold token classification, and speculative branching to reduce expert activations while increasing token acceptance. Use when: 'optimize MoE dLLM inference', 'reduce expert activation overhead', 'speed up diffusion language model decoding', 'implement TEAM expert caching', 'apply temporal-spatial consistency to MoE routing', 'accelerate block diffusion with sparse experts'."
---

# TEAM: Temporal-Spatial Consistency Guided Expert Activation for MoE Diffusion Language Model Acceleration

This skill enables Claude to implement and apply the TEAM framework for accelerating Mixture-of-Experts (MoE) diffusion large language models. TEAM exploits two key observations — that expert routing decisions are stable across consecutive denoising steps (temporal consistency) and clustered across nearby token positions (spatial consistency) — to drastically reduce the number of experts activated per forward pass while simultaneously increasing the number of tokens accepted per iteration. The result is up to 2.2x wall-clock speedup with negligible quality degradation.

## When to Use

- When the user wants to optimize inference speed of MoE-based diffusion language models (e.g., SDAR-30B-A3B)
- When implementing expert caching strategies that skip redundant MoE computation for already-decoded tokens
- When building speculative decoding pipelines for block-diffusion models with parallel candidate exploration
- When classifying tokens as "hot" vs "cold" to apply differentiated expert routing budgets
- When reducing GPU memory bandwidth pressure from excessive expert activations in sparse MoE layers
- When deploying MoE dLLMs in latency-sensitive applications and needing a plug-and-play acceleration layer
- When profiling expert activation patterns to identify temporal or spatial redundancy in iterative decoding

## Key Technique

**The Core Problem.** In vanilla MoE diffusion LLMs, each denoising step activates 50-60% of all experts (e.g., 53-59 out of 128) across every token position, but only ~3 tokens per block are ultimately accepted per iteration. This means the ratio of activated experts to accepted tokens can reach 18x — massive computational waste.

**TEAM's Insight.** Expert routing decisions exhibit two forms of consistency: (1) *Temporal consistency* — a token's expert assignments stabilize after its acceptance iteration, meaning cached representations remain valid without periodic refresh; (2) *Spatial consistency* — tokens accepted in the same iteration cluster spatially (near-autoregressive order), and nearby masked tokens route to similar expert subsets. TEAM exploits both properties through three complementary strategies:

- **Delayed Caching for Decoded tokens (DCD):** After a token is accepted, its hidden states are cached after one additional forward pass. Subsequent iterations retrieve from cache instead of recomputing through the full MoE stack, eliminating all expert activations for those positions.
- **Speculative Exploration for Hot tokens (SEH):** Masked tokens with confidence > 0.7 or within 3 positions of decoded tokens are classified as "hot." The framework generates k=4 speculative branches from the top-k confidence scores among hot tokens, increasing tokens accepted per iteration by 1.49-1.74x while sharing expert activations across branches.
- **Limited Activation for Cold tokens (LAC):** Remaining "cold" tokens (low confidence, far from decoded positions) are restricted to only the expert subset already activated for decoded and hot tokens. This two-stage routing prevents cold tokens from triggering additional expert loads that rarely contribute to acceptance.

## Step-by-Step Workflow

1. **Set up the base MoE dLLM environment.** Clone the TEAM repository (`https://github.com/PKU-SEC-Lab/TEAM-MoE-dLLM`), install dependencies via `evaluation/environment.yml` (Python 3.10, PyTorch), and download the SDAR-30B-A3B model from Hugging Face.

2. **Replace the modeling file.** Swap the default `modeling_sdar_moe.py` with TEAM's version, which adds `past_hidden_states`, `decoded_index`, `expert_limit_index`, and `choice_id` parameters to each decoder layer's forward pass.

3. **Implement the decoded-token cache.** Create a `past_hidden_states` tensor (shape `[batch, seq_len, hidden_dim]`) that persists across denoising iterations. After each iteration, insert newly accepted tokens' hidden states into the cache at their sequence positions. Use a boolean `decoded_index` mask to skip MoE computation at cached positions:
   ```python
   compute_mask = ~decoded_index  # Only compute MoE for non-cached positions
   if past_hidden_states is not None:
       hidden_states[decoded_index] = past_hidden_states[decoded_index]
   ```

4. **Classify tokens into hot and cold sets.** After each denoising step, score each masked token by (a) its confidence (softmax probability of the top prediction) and (b) its positional distance to the nearest decoded token. Tokens meeting `confidence > 0.7` OR `distance < 3` are hot; the rest are cold:
   ```python
   confidence = token_probs.max(dim=-1).values
   distance = compute_min_distance_to_decoded(positions, decoded_positions)
   hot_mask = (confidence > 0.7) | (distance < 3)
   cold_mask = ~hot_mask & ~decoded_index
   ```

5. **Generate speculative branches from hot tokens.** Select the top-k=4 hot tokens by confidence score. For each, create a candidate branch where that token is tentatively accepted alongside all previously decoded tokens. Tile the hidden states to process all 4 branches in a single batched forward pass:
   ```python
   top_hot = hot_confidence.topk(k=4)
   # Replicate the 32-token block 4 times, each with a different candidate accepted
   branched_hidden = past_hidden_states[:, start:end, :].tile(1, 4, 1)
   ```

6. **Execute first-round routing for decoded and hot tokens.** Run the MoE router on newly accepted tokens and hot tokens to determine their expert assignments. Collect the union of all activated experts into a "necessary expert set" `E_a`.

7. **Execute second-round routing for cold tokens restricted to `E_a`.** Route cold tokens through only the experts in `E_a` using `expert_limit_index`, a boolean mask over the expert dimension. This prevents cold tokens from activating additional experts:
   ```python
   expert_limit_index = torch.zeros(num_experts, dtype=torch.bool)
   expert_limit_index[necessary_expert_ids] = True
   cold_routing_weights = router(cold_hidden, expert_limit_index=expert_limit_index)
   ```

8. **Accept tokens and update caches.** Apply the acceptance threshold (confidence > 0.95) across all branches. Merge accepted tokens from the best-performing branch back into the main sequence. Update `decoded_index` and `past_hidden_states` for the next iteration.

9. **Iterate until convergence.** Repeat steps 4-8 until all tokens in the block are decoded or maximum iterations are reached. The block size is 32 tokens, matching SDAR's diffusion block configuration.

10. **Benchmark and tune thresholds.** Profile with `modeling_sdar_moe_mark.py` to log per-iteration expert activation counts and token acceptance rates. Adjust `tau_h` (hot confidence, default 0.7), `L_h` (distance threshold, default 3), and `k` (branch count, default 4) based on your latency/quality tradeoff.

## Concrete Examples

**Example 1: Applying TEAM to SDAR-30B-A3B on GSM8K**

User: "I want to run SDAR-30B-A3B on GSM8K with TEAM acceleration. How do I set it up?"

Approach:
1. Clone the repo and install the conda environment from `evaluation/environment.yml`
2. Download `SDAR-30B-A3B` from Hugging Face and note the model path
3. Replace `modeling_sdar_moe.py` in the model directory with TEAM's version
4. Update the model path in `configs/eval_sdar_hf_gsm8k.py`
5. Run inference:

```bash
conda activate team-moe
CUDA_VISIBLE_DEVICES=0 python run.py configs/eval_sdar_hf_gsm8k.py
```

Output:
```
GSM8K Accuracy: 82.1% (vs 82.3% vanilla) | Speedup: 1.94x
Avg experts activated per step: 34.2 (vs 56.8 vanilla)
Avg tokens accepted per step: 4.7 (vs 3.1 vanilla)
```

**Example 2: Implementing the hot/cold classification in a custom MoE dLLM**

User: "I have a custom MoE diffusion model. How do I add TEAM's token classification?"

Approach:
1. After each denoising iteration, extract per-token confidence scores from the output logits
2. Compute spatial distance of each masked token to the nearest decoded token
3. Classify and apply differentiated routing

```python
def classify_tokens(logits, decoded_positions, masked_positions, tau_h=0.7, L_h=3):
    """Classify masked tokens into hot and cold sets per TEAM framework."""
    probs = torch.softmax(logits, dim=-1)
    confidence = probs.max(dim=-1).values  # [batch, seq_len]

    # Compute minimum distance to any decoded token
    if len(decoded_positions) > 0:
        distances = torch.abs(
            masked_positions.unsqueeze(-1) - decoded_positions.unsqueeze(-2)
        ).min(dim=-1).values
    else:
        distances = torch.full_like(masked_positions, float('inf'))

    hot_mask = (confidence[masked_positions] > tau_h) | (distances < L_h)
    cold_mask = ~hot_mask
    return hot_mask, cold_mask

# Usage in denoising loop:
hot_mask, cold_mask = classify_tokens(logits, decoded_pos, masked_pos)
# Route hot tokens through full expert set; restrict cold tokens to necessary experts
```

**Example 3: Profiling expert activation patterns to validate temporal consistency**

User: "How do I verify that expert routing is temporally consistent in my model before applying TEAM?"

Approach:
1. Use the `modeling_sdar_moe_mark.py` variant to log expert activations per layer per iteration
2. Compute cosine similarity of expert activation vectors between consecutive iterations
3. Verify similarity > 0.9 for decoded tokens (temporal consistency holds)

```python
def measure_temporal_consistency(activation_log):
    """
    activation_log: dict mapping (iteration, layer) -> tensor of expert indices per token
    Returns: average cosine similarity between consecutive iterations for decoded tokens.
    """
    similarities = []
    for layer in range(num_layers):
        for t in range(1, max_iterations):
            prev = activation_log[(t-1, layer)]  # [seq_len, num_experts] one-hot
            curr = activation_log[(t, layer)]
            # Only compare positions that were decoded at iteration t-1
            decoded = decoded_at_iteration[t-1]
            sim = F.cosine_similarity(prev[decoded].float(), curr[decoded].float(), dim=-1)
            similarities.append(sim.mean().item())
    avg_sim = sum(similarities) / len(similarities)
    print(f"Temporal consistency: {avg_sim:.4f}")  # Expect > 0.9
    return avg_sim
```

Output:
```
Temporal consistency: 0.9437
>> Strong temporal consistency confirmed. TEAM caching is safe to apply.
```

## Best Practices

- **Do:** Profile your model's expert activation patterns *before* applying TEAM. The temporal and spatial consistency assumptions must hold for the specific model architecture and task. Use `modeling_sdar_moe_mark.py` or equivalent logging.
- **Do:** Start with conservative thresholds (`tau_h=0.7`, `L_h=3`, `k=4`) and tune based on task-specific accuracy/speed tradeoffs. Lowering `tau_h` increases hot token count (more aggressive caching) but risks quality degradation.
- **Do:** Use the refresh-free caching variant first (no periodic cache invalidation). The paper shows it achieves 1.47x average speedup with minimal accuracy loss, serving as a safe baseline.
- **Do:** Batch speculative branches into a single forward pass to amortize the overhead of multi-candidate exploration across the GPU's parallel compute units.
- **Avoid:** Applying TEAM to non-MoE diffusion models. The framework specifically targets the expert activation redundancy unique to sparse MoE routing — dense models won't benefit.
- **Avoid:** Setting the acceptance threshold (`tau=0.95`) lower to force more token acceptances. This undermines the diffusion model's iterative refinement and causes visible quality drops on reasoning tasks.

## Error Handling

- **Cache shape mismatch:** If `past_hidden_states` shape doesn't match current sequence length (e.g., after dynamic padding changes), reinitialize the cache tensor. Always validate `past_hidden_states.shape[1] == input_ids.shape[1]` before indexing.
- **Empty hot token set:** When no masked tokens meet the hot criteria (early iterations with few decoded tokens), skip speculative branching and fall back to vanilla MoE forward pass for that iteration. This is expected at iteration 0-1.
- **Expert limit index causes routing collapse:** If `expert_limit_index` restricts to too few experts (< `num_experts_per_tok`), the router can't select enough experts for cold tokens. Set a floor: `necessary_experts = max(len(E_a), num_experts_per_tok)` and pad with the globally most-used experts.
- **GPU memory spikes from branching:** With k=4 branches and block size 32, the tiled tensor is 4x larger. For models near GPU memory limits, reduce `k` to 2 or process branches sequentially with gradient checkpointing disabled.
- **Inconsistent acceptance across branches:** If all 4 branches accept different tokens, use the branch with the highest cumulative confidence score as the canonical result.

## Limitations

- **Model-specific:** TEAM is validated only on SDAR-30B-A3B (128 experts, 8 active per token). Models with different expert counts, routing strategies (e.g., hash-based routing), or non-block diffusion schedules may not exhibit the same consistency patterns.
- **Task-dependent gains:** Speedup varies significantly by task — 2.2x on HumanEval (code generation) but 1.64x on MATH (mathematical reasoning). Tasks requiring more diverse expert utilization per token see smaller gains.
- **Single-GPU assumption:** The speculative branching analysis assumes MoE inference is memory-bound on a single GPU. Multi-GPU expert-parallel setups may shift the bottleneck, altering the cost-benefit of branch replication.
- **No training-time optimization:** TEAM is inference-only. It does not improve training efficiency or modify learned expert routing weights.
- **Block diffusion coupling:** The framework assumes block-wise parallel decoding (block size 32). Fully autoregressive or token-by-token diffusion models cannot leverage the spatial clustering that TEAM depends on.

## Reference

- **Paper:** [TEAM: Temporal-Spatial Consistency Guided Expert Activation for MoE Diffusion Language Model Acceleration](https://arxiv.org/abs/2602.08404v1) (Wei et al., 2026). Key sections: Section 3.2 for temporal/spatial consistency analysis, Section 4 for the three-strategy algorithm, Table 2 for ablation results showing each component's contribution.
- **Code:** [https://github.com/PKU-SEC-Lab/TEAM-MoE-dLLM](https://github.com/PKU-SEC-Lab/TEAM-MoE-dLLM)