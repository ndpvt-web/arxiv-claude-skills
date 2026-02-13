---
name: "step-35-flash-open"
description: "Build efficient agentic AI systems using sparse MoE routing, hybrid sliding-window/full attention, multi-token prediction, and RL-based self-improvement -- the architecture patterns from Step 3.5 Flash. Use when: 'design an MoE agent architecture', 'optimize multi-round agent latency', 'implement speculative decoding for agents', 'build a scalable RL training pipeline for tool use', 'reduce inference cost for agentic workloads', 'architect a sparse expert system for code agents'."
---

# Step 3.5 Flash: Frontier-Level Agentic Intelligence with Sparse MoE

This skill teaches you to apply the architectural and training patterns from Step 3.5 Flash -- a 196B-parameter sparse Mixture-of-Experts model with only 11B active parameters -- to build, optimize, and deploy efficient agentic AI systems. The core insight: frontier-level agent performance does not require frontier-level compute. By combining sparse expert routing, hybrid attention patterns (3:1 sliding-window/full), multi-token prediction, and a stability-aware RL framework, you can build agents that reason sharply and execute fast across code, math, and tool-use tasks.

## When to Use

- When designing a Mixture-of-Experts architecture for agentic workloads (tool calling, code generation, multi-turn reasoning)
- When optimizing inference latency for multi-round agent interactions that involve heavy context prefilling followed by prolonged decoding
- When implementing speculative decoding via multi-token prediction heads to reduce agent response time
- When building an RL training pipeline that combines verifiable rewards (test execution, proof checking) with preference feedback for open-ended tasks
- When you need to stabilize off-policy RL training for MoE models where expert routing creates distributional shift
- When architecting hybrid attention (sliding-window + full attention) to handle long-context agentic workloads efficiently
- When deploying a high-parameter-count model with low active-parameter inference cost in production

## Key Technique

**Sparse MoE with Agent-Optimized Attention.** Step 3.5 Flash uses 288 routed experts plus 1 shared expert per MoE layer, with top-8 routing (8 of 289 experts activate per token). Of 45 total layers, the first 3 are dense and the remaining 42 are MoE layers. This gives a 196B total parameter budget but only 11B active per forward pass. The routing uses a learned router with loss-free load-balancing plus an EP-level (expert-parallel) balancing loss that minimizes the product of token fractions and routing probabilities at the group level, preventing stragglers across distributed ranks.

**Hybrid 3:1 Sliding-Window/Full Attention (S3F1).** The attention pattern interleaves three Sliding-Window Attention (SWA) layers (window size W=512, 96 query heads) with one Full Attention layer (GQA-8, i.e., 8 key-value heads). A leading full-attention layer precedes the repeating S3F1 blocks. SWA layers use head-wise gated attention (data-dependent) instead of fixed sink tokens, and the augmented query head count (96 vs. the standard 64) compensates for the reduced receptive field. This pattern cuts memory and compute for long-context agent sessions while preserving global information flow every 4th layer.

**MIS-Filtered Policy Optimization (MIS-PO) for Stable Agent RL.** Standard off-policy RL is unstable with MoE models because expert-level routing amplifies distributional shift. MIS-PO replaces continuous importance weighting with discrete distributional filtering at both token and trajectory levels, restricting optimization to samples within a stable trust region. It adds truncation-aware value bootstrapping (adjusting value estimates for trajectories cut short by context limits) and uses routing confidence as a stability proxy. Rewards come from three sources: verifiable signals (test-case execution, mathematical proof verification), LLM-based judgment for open-ended tasks, and agent-specific success/failure signals from tool execution. This enables consistent self-improvement across code, math, and tool use without training collapse.

## Step-by-Step Workflow

### Designing the MoE Agent Architecture

1. **Define expert topology.** Choose the number of routed experts (e.g., 288) and shared experts (e.g., 1) per MoE layer. Set top-k routing (k=8 is the paper's sweet spot for balancing capacity vs. compute). Make the first few layers dense (3 dense layers) to build shared representations before expert specialization begins.

2. **Implement the learned router with dual-level load balancing.** Use a standard linear router projecting hidden states to expert logits, then apply top-k selection. Add two balancing losses: (a) a per-layer auxiliary loss-free balancing mechanism that adjusts routing without gradient signals, and (b) an EP-level balancing loss that penalizes uneven token distribution across expert-parallel ranks. This prevents both expert collapse and distributed-training stragglers.

3. **Configure the S3F1 hybrid attention pattern.** For every group of 4 transformer layers, make the first 3 use sliding-window attention (W=512) with 96 query heads and the 4th use full GQA attention with 8 KV heads. Prepend a single full-attention layer at the start. In the SWA layers, replace fixed attention sinks with a learned per-head gate that controls how much each head attends to the [BOS] token vs. the local window.

4. **Add Multi-Token Prediction (MTP-3) heads.** Attach three lightweight prediction heads that forecast tokens at positions +1, +2, and +3. Each head consists of one SWA layer plus one dense FFN, adding ~0.81B parameters (~0.41% overhead). During training, only train the MTP-1 head during the main phase; clone it to create MTP-2 and MTP-3, then jointly fine-tune all three in a short final phase. Apply position-dependent loss reweighting (downweight distant predictions) to prevent over-optimizing future tokens at the expense of next-token accuracy.

5. **Set up the training data pipeline.** Compose a pre-training mixture of general knowledge, code (including PRs/issues for real-world code reasoning), tool-use traces, and synthetic reasoning data. Target ~17T tokens. Use progressive batch-size scaling (e.g., 8192 -> 12288 -> 16384) to improve training stability. Extend context to 128k tokens during a mid-training phase using synthetic long-context and agentic multi-turn data.

### Training the Agent with RL

6. **Build the reward infrastructure.** Implement three reward channels: (a) verifiable rewards from code test-case execution and mathematical proof checkers, (b) a learned reward model (GenRM/MetaRM) trained on human preference data for open-ended outputs, and (c) agent-specific binary rewards from tool execution success/failure. Prioritize verifiable signals -- they provide the strongest learning signal with zero label noise.

7. **Run MIS-PO reinforcement learning.** Sample trajectories from the current policy, then filter them using distributional criteria at both the token and trajectory level. Discard samples that fall outside the trust region (where the behavior policy diverges too far from the current policy). For truncated trajectories (agent hit context limit mid-task), apply bootstrapped value estimates rather than treating truncation as terminal. Monitor routing confidence across MoE layers as a stability diagnostic -- sudden drops indicate distributional shift.

8. **Iterate SFT and RL in post-training.** First construct expert models via self-distillation, then run supervised fine-tuning on curated agent traces, then apply MIS-PO RL. This staged approach (distill -> SFT -> RL) prevents catastrophic forgetting while steadily improving agent capabilities.

### Optimizing Inference

9. **Enable speculative decoding with MTP heads.** At inference time, use the three MTP heads to draft candidate tokens in parallel. Verify them against the main model's distribution and accept matching tokens. This yields up to 2-3x throughput improvement for the extended decoding phases typical of agentic workloads (where context is prefilled once then many turns of generation follow).

10. **Deploy with expert-parallel inference.** Distribute experts across GPUs using expert parallelism. The EP-level load balancing trained into the router ensures even utilization across devices at inference time, not just during training. Monitor per-rank token counts and adjust batch composition if skew exceeds 15%.

## Concrete Examples

**Example 1: Designing an MoE Router with Load Balancing**

User: "Implement a top-k MoE router in PyTorch with load balancing for 64 experts, top-4 routing."

Approach:
1. Create a linear projection from hidden_dim to num_experts
2. Apply top-k selection with softmax gating
3. Add auxiliary load-balancing loss (fraction-of-tokens * mean-routing-probability per expert)
4. Add EP-level balancing that groups experts by device rank

Output:
```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class MoERouter(nn.Module):
    def __init__(self, hidden_dim: int, num_experts: int = 64, top_k: int = 4,
                 num_ep_groups: int = 8):
        super().__init__()
        self.gate = nn.Linear(hidden_dim, num_experts, bias=False)
        self.top_k = top_k
        self.num_experts = num_experts
        self.num_ep_groups = num_ep_groups  # number of expert-parallel ranks

    def forward(self, x: torch.Tensor):
        # x: (batch * seq_len, hidden_dim)
        logits = self.gate(x)  # (tokens, num_experts)
        scores = F.softmax(logits, dim=-1)

        topk_scores, topk_indices = scores.topk(self.top_k, dim=-1)
        topk_scores = topk_scores / topk_scores.sum(dim=-1, keepdim=True)  # renormalize

        # Per-expert load balancing loss (loss-free style)
        tokens_per_expert = torch.zeros(self.num_experts, device=x.device)
        tokens_per_expert.scatter_add_(0, topk_indices.reshape(-1),
                                        torch.ones_like(topk_indices.reshape(-1), dtype=torch.float))
        fraction = tokens_per_expert / x.shape[0]
        mean_prob = scores.mean(dim=0)
        aux_loss = (fraction * mean_prob).sum() * self.num_experts

        # EP-level balancing: aggregate by rank group
        experts_per_group = self.num_experts // self.num_ep_groups
        group_fraction = fraction.reshape(self.num_ep_groups, experts_per_group).sum(dim=1)
        group_prob = mean_prob.reshape(self.num_ep_groups, experts_per_group).sum(dim=1)
        ep_loss = (group_fraction * group_prob).sum() * self.num_ep_groups

        total_loss = aux_loss + 0.1 * ep_loss
        return topk_scores, topk_indices, total_loss
```

**Example 2: Implementing S3F1 Hybrid Attention**

User: "Build the interleaved sliding-window/full attention stack for a 24-layer transformer."

Approach:
1. Create a leading full-attention layer
2. For the remaining 23 layers, repeat the S3F1 pattern (3 SWA + 1 Full)
3. SWA layers use window_size=512 with 96 query heads; Full layers use GQA with 8 KV heads

Output:
```python
def build_s3f1_attention_config(num_layers: int = 24, swa_window: int = 512,
                                 swa_q_heads: int = 96, full_kv_heads: int = 8,
                                 full_q_heads: int = 64) -> list[dict]:
    """Generate layer-wise attention configuration following S3F1 pattern."""
    layers = []
    # Leading full-attention layer
    layers.append({
        "type": "full", "q_heads": full_q_heads, "kv_heads": full_kv_heads,
        "window_size": None
    })
    # Remaining layers: repeat [SWA, SWA, SWA, Full]
    for i in range(1, num_layers):
        position_in_block = (i - 1) % 4  # 0,1,2 = SWA; 3 = Full
        if position_in_block < 3:
            layers.append({
                "type": "swa", "q_heads": swa_q_heads, "kv_heads": full_kv_heads,
                "window_size": swa_window, "gated_sink": True
            })
        else:
            layers.append({
                "type": "full", "q_heads": full_q_heads, "kv_heads": full_kv_heads,
                "window_size": None
            })
    return layers

# Usage
config = build_s3f1_attention_config(num_layers=24)
for i, layer in enumerate(config):
    print(f"Layer {i:2d}: {layer['type']:4s} | Q-heads: {layer['q_heads']} | Window: {layer['window_size']}")
```

**Example 3: MIS-PO Trust-Region Filtering for Agent RL**

User: "Implement the trajectory filtering step of MIS-PO for a code agent RL loop."

Approach:
1. Compute importance weights (ratio of current policy prob to behavior policy prob)
2. Filter at token level: discard tokens with importance weight outside [1/c, c]
3. Filter at trajectory level: discard entire trajectories where mean weight diverges
4. Handle truncated trajectories with value bootstrapping

Output:
```python
import torch

def mis_po_filter(
    log_probs_current: torch.Tensor,   # (batch, seq_len) from current policy
    log_probs_behavior: torch.Tensor,   # (batch, seq_len) from rollout policy
    rewards: torch.Tensor,              # (batch,) trajectory-level rewards
    truncated: torch.Tensor,            # (batch,) bool: was trajectory truncated?
    value_estimates: torch.Tensor,      # (batch,) V(s_T) for bootstrapping
    clip_ratio: float = 2.0,
    traj_threshold: float = 1.5,
) -> tuple[torch.Tensor, torch.Tensor, torch.Tensor]:
    """Filter trajectories and tokens for stable MoE off-policy RL."""

    # Token-level importance weights
    importance = (log_probs_current - log_probs_behavior).exp()  # (batch, seq_len)

    # Token-level trust region: keep tokens where 1/c <= w <= c
    token_mask = (importance >= 1.0 / clip_ratio) & (importance <= clip_ratio)

    # Trajectory-level filter: mean importance weight should be close to 1
    mean_importance = importance.mean(dim=1)  # (batch,)
    traj_mask = (mean_importance >= 1.0 / traj_threshold) & (mean_importance <= traj_threshold)

    # Truncation-aware value bootstrapping
    # For truncated trajectories, add discounted value estimate instead of treating as terminal
    gamma = 0.99
    adjusted_rewards = rewards.clone()
    adjusted_rewards[truncated] += gamma * value_estimates[truncated]

    return token_mask, traj_mask, adjusted_rewards

# Usage in RL loop:
# token_mask, traj_mask, adj_rewards = mis_po_filter(...)
# loss = compute_policy_loss(logits[traj_mask][:, token_mask[traj_mask]], adj_rewards[traj_mask])
```

## Best Practices

- **Do** start with dense layers before MoE layers (3 dense layers is a good default). Early layers build shared representations that all experts need; premature specialization hurts.
- **Do** monitor per-rank expert utilization during both training and inference. A >15% skew across EP ranks means your balancing loss coefficient needs tuning.
- **Do** train MTP-1 first, then clone and fine-tune MTP-2/3 in a short final phase. Joint training from scratch destabilizes the main next-token prediction objective.
- **Do** use verifiable rewards (test execution, proof checking) as the primary RL signal. They have zero label noise and scale without human annotation.
- **Avoid** using full attention in every layer for agentic workloads. The S3F1 pattern achieves comparable quality at ~40% of the KV-cache memory cost for 128k contexts.
- **Avoid** continuous importance weighting in MoE RL. The discrete filtering approach (MIS-PO) is essential because expert routing amplifies the variance of importance ratios far beyond what clipping alone can handle.

## Error Handling

- **Expert collapse (one expert gets >50% of tokens):** Increase the EP-level balancing loss coefficient. If collapse persists, add a jitter noise term (uniform random ~0.01) to router logits during training.
- **Training loss spikes:** The paper reports a single transient spike across 17T tokens. If spikes recur, clip activations in MoE FFN intermediate layers and reduce learning rate temporarily. Do not restart from checkpoint unless the spike does not recover within 200 steps.
- **MTP head degradation:** If MTP acceptance rate drops below 40% during speculative decoding, the MTP heads are stale. Re-fine-tune them on recent data while freezing the main model.
- **RL routing instability:** If routing confidence (measured as max expert probability per token averaged across layers) drops by >10% during RL, tighten the MIS-PO trust region (reduce `clip_ratio` from 2.0 to 1.5).
- **Context truncation in agent tasks:** Always bootstrap truncated trajectories with value estimates. Treating truncation as terminal creates a systematic negative bias that teaches the agent to avoid long reasoning chains.

## Limitations

- The S3F1 attention pattern with W=512 struggles on tasks requiring precise token-level recall beyond 512 positions in the SWA layers. Tasks like exact substring matching over very long documents may degrade compared to full attention at every layer.
- Top-k routing with k=8 and 288 experts is optimized for large-scale distributed inference (8+ GPUs). On single-GPU setups, the routing overhead may negate the sparse activation savings. For single-GPU deployment, reduce to ~32 experts with top-2 routing.
- MIS-PO requires generating rollouts from the current policy and filtering them, which means ~30-50% of compute is "wasted" on discarded samples. This is the cost of stability -- if your training budget is very tight, standard PPO with aggressive clipping may be more sample-efficient (though less stable).
- Multi-token prediction heads add inference complexity. If your serving stack does not support speculative decoding verification, the MTP heads provide no latency benefit and only add parameters.
- The RL framework assumes access to verifiable reward signals (test suites, proof checkers). For purely open-ended agent tasks (e.g., creative writing, subjective UI evaluation), you fall back to learned reward models, which are noisier and plateau faster.

## Reference

**Paper:** [Step 3.5 Flash: Open Frontier-Level Intelligence with 11B Active Parameters](https://arxiv.org/abs/2602.10604v1) -- Focus on Section 3 (MoE architecture and S3F1 attention), Section 4 (MTP-3 design), and Section 6 (MIS-PO reinforcement learning framework) for implementation-critical details.