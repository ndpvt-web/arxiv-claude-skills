---
name: "reinforced-attention-learning"
description: "Implement Reinforced Attention Learning (RAL) for multimodal LLMs — optimize attention distributions via policy gradients instead of output tokens. Use when: 'optimize attention for multimodal model', 'implement RAL post-training', 'attention-based RL for vision-language', 'build attention distillation pipeline', 'improve MLLM grounding with attention RL', 'apply policy gradients to attention maps'."
---

# Reinforced Attention Learning (RAL)

This skill enables Claude to implement Reinforced Attention Learning, a post-training framework that applies policy gradients directly to internal attention distributions in Multimodal LLMs rather than optimizing output token sequences. RAL treats each token's causal attention map as a latent policy, computes sequence-level advantages from rule-based rewards, and backpropagates through Jensen-Shannon Divergence to reshape where the model attends. This produces stronger visual grounding and cross-modal alignment than verbose chain-of-thought RL approaches like GRPO, particularly on perception-heavy image and video benchmarks.

## When to Use

- When the user wants to post-train a multimodal LLM (e.g., Qwen-VL, LLaVA) with RL but finds that GRPO or verbose reasoning yields minimal perception gains
- When building a training pipeline that optimizes attention allocation across visual tokens rather than generating longer rationales
- When implementing on-policy attention distillation from a larger teacher MLLM to a smaller student
- When the user needs to improve spatial reasoning, chart understanding, or video QA performance without increasing inference cost
- When debugging why RL post-training degrades multimodal performance — RAL addresses the root cause (poor information allocation vs. verbose token generation)
- When the user asks to extract, visualize, or optimize attention maps from the final transformer layer of a vision-language model

## Key Technique

**Core Insight: Optimize Where to Attend, Not What to Generate.** Standard RL post-training (GRPO) optimizes the probability of output token sequences. For multimodal tasks, this often produces verbose "thinking" chains that fail to improve — or actively hurt — perceptual grounding. RAL instead treats the causal attention distribution at each generated token position as a policy to optimize. The attention weights from the final transformer layer, averaged across heads, define a probability distribution over all preceding positions. Policy gradients reshape these distributions so the model allocates more attention to task-relevant visual and textual regions.

**Advantage-Weighted Divergence.** For each rollout, RAL computes a sequence-level advantage using GRPO-style group-relative normalization across G=8 samples per input. The attention loss is `L_AttnRL = E_t[A_t * JSD(p_theta^t || p_old^t)]`, where JSD is Jensen-Shannon Divergence between current and reference attention distributions. Positive advantage minimizes divergence from successful attention patterns; negative advantage pushes the model away from failing patterns. Gradients flow through the softmax Jacobian of the attention logits, providing per-token granularity that avoids vanishing gradients on long sequences.

**On-Policy Attention Distillation.** Beyond self-improvement, RAL introduces a distillation variant: sample trajectories from the student model's own policy, then supervise its attention distributions toward the teacher's attention patterns on those same trajectories. This avoids exposure bias (the student never sees teacher-generated sequences it couldn't produce). The combined objective is `L_total = L_RL + mu*L_GKD + gamma_attn*L_AttnDistill`, unifying token-level RL, output logit distillation, and attention structure transfer.

## Step-by-Step Workflow

1. **Set up the base model and freeze non-language components.** Load the multimodal LLM (e.g., Qwen-2.5-VL-7B). Freeze the visual encoder and multimodal projector. Only the language model backbone receives gradient updates — attention optimization happens within the transformer layers that process both visual and text tokens.

2. **Patch attention to use eager mode and extract last-layer weights.** Replace any flash/SDPA attention with eager attention in the final transformer layer. After each forward pass, capture the raw attention weight tensor `alpha_{t,i}` for each generated token `t` over all preceding positions `i`. Average across all attention heads to get a single distribution per token position.

3. **Compute the causal attention policy distribution.** For each generated token at position `t`, normalize: `p_theta^t(i) = alpha_{t,i} / sum_{j<=t-1} alpha_{t,j}`. This is the "attention policy" — a probability distribution over context positions that RAL will optimize.

4. **Generate rollouts with group sampling.** For each input (image/video + question), sample G=8 completions at temperature 0.9. Store both the generated token sequences and the per-token attention distributions from step 3 for each rollout.

5. **Score rollouts with rule-based rewards.** Apply a two-component reward: accuracy reward `r_acc` (1.0 for correct answer in `<answer>` tags, 0.0 otherwise) and format reward `r_fmt` (1.0 for correct `<think>...</think><answer>...</answer>` structure). Combine as `r_i = 0.9 * r_acc + 0.1 * r_fmt`.

6. **Compute group-relative advantages.** For the G rollouts of each input, compute `A_i = (r_i - mean(r)) / std(r)`. This eliminates the need for a separate critic model and normalizes reward signal across the group.

7. **Compute the attention RL loss.** For each rollout, compute Jensen-Shannon Divergence between the current attention distribution and the reference (from the policy at rollout time): `L_AttnRL = sum_t A_t * JSD(p_theta^t || p_old^t)`. Backpropagate through the softmax Jacobian: `d_p_j / d_e_i = p_i * (delta_{ij} - p_j)`.

8. **Combine with the standard RL objective.** The total loss is `L_total = L_RL + lambda_attn * L_AttnRL`, where `L_RL` is the standard token-level policy gradient loss and `lambda_attn` controls attention regularization strength. Start with `lambda_attn=1.0` and tune from {0.5, 1.0, 5.0}.

9. **(Optional) Add on-policy attention distillation.** If a larger teacher model is available, run the student's generated trajectories through the teacher to extract teacher attention distributions. Add `L_AttnDistill = sum_t JSD(p_student^t || p_teacher^t)` without advantage weighting. The unified loss becomes `L_total = L_RL + mu*L_GKD + gamma_attn*L_AttnDistill`, with `gamma_attn` tuned from {0.05, 0.5, 1.0}.

10. **Train and evaluate.** Train for a single epoch over the RL dataset with learning rate 1e-5, KL coefficient beta=0.04. Evaluate on perception-heavy benchmarks (V*, MME, ChartQA for images; VideoMME, NExTQA, MVBench for video). Expect the largest gains on tasks with high visual density — RAL's advantage widens from +1.6 at 512-token resolution to +6.3 at 2048-token resolution.

## Concrete Examples

**Example 1: Implementing the RAL attention loss in PyTorch**

User: "I have a Qwen-VL model fine-tuned with GRPO but it's not improving on visual grounding tasks. Help me add RAL's attention-based RL loss."

Approach:
1. Patch the model's last transformer layer to return raw attention weights
2. Implement the causal attention policy extraction
3. Implement the JSD-based attention RL loss with advantage weighting
4. Integrate into the existing GRPO training loop

Output:
```python
import torch
import torch.nn.functional as F

def extract_causal_attention_policy(attn_weights, generated_start_idx):
    """
    Extract per-token attention policy from last layer.
    attn_weights: (batch, heads, seq_len, seq_len) from eager attention
    Returns: (batch, gen_len, context_len) normalized attention distributions
    """
    # Average across heads
    avg_attn = attn_weights.mean(dim=1)  # (batch, seq, seq)
    # Extract only generated token positions attending to context
    gen_attn = avg_attn[:, generated_start_idx:, :generated_start_idx]
    # Normalize to form valid probability distributions
    attn_policy = gen_attn / gen_attn.sum(dim=-1, keepdim=True).clamp(min=1e-8)
    return attn_policy

def jsd_loss(p, q):
    """Jensen-Shannon Divergence between attention distributions."""
    m = 0.5 * (p + q)
    kl_pm = F.kl_div(m.log(), p, reduction='none').sum(-1)
    kl_qm = F.kl_div(m.log(), q, reduction='none').sum(-1)
    return 0.5 * (kl_pm + kl_qm)

def ral_attention_loss(current_attn_policy, reference_attn_policy, advantages):
    """
    Reinforced Attention Learning loss.
    current_attn_policy: (batch, gen_len, context_len)
    reference_attn_policy: (batch, gen_len, context_len) — from rollout time
    advantages: (batch,) — sequence-level GRPO advantages
    """
    # Per-token JSD between current and reference attention
    per_token_jsd = jsd_loss(current_attn_policy, reference_attn_policy)  # (batch, gen_len)
    # Average over token positions
    mean_jsd = per_token_jsd.mean(dim=-1)  # (batch,)
    # Weight by advantage: positive advantage -> minimize divergence from good patterns
    loss = (advantages * mean_jsd).mean()
    return loss

# Integration into training loop:
# lambda_attn = 1.0
# total_loss = grpo_token_loss + lambda_attn * ral_attention_loss(
#     current_attn, reference_attn, advantages
# )
```

**Example 2: On-policy attention distillation from a larger teacher**

User: "I want to distill attention patterns from Qwen-2.5-VL-32B into a 7B student. Standard KD on logits isn't helping much on video QA."

Approach:
1. Generate trajectories from the student model (on-policy sampling)
2. Run those same trajectories through the teacher with eager attention
3. Extract attention policies from both models on the student's sequences
4. Compute attention distillation loss without advantage weighting
5. Combine with standard GKD and RL losses

Output:
```python
def on_policy_attention_distillation(
    student_model, teacher_model, input_ids, visual_features,
    generated_start_idx, mu=1.0, gamma_attn=0.5
):
    """
    On-policy attention distillation: student generates,
    both models produce attention, student learns teacher's patterns.
    """
    # Step 1: Student generates trajectories (already done in rollout phase)
    # Step 2: Forward pass through both models on student-generated sequences
    student_out = student_model(
        input_ids=input_ids, pixel_values=visual_features,
        output_attentions=True  # requires eager attention patch
    )
    with torch.no_grad():
        teacher_out = teacher_model(
            input_ids=input_ids, pixel_values=visual_features,
            output_attentions=True
        )

    # Step 3: Extract last-layer attention policies
    student_attn = extract_causal_attention_policy(
        student_out.attentions[-1], generated_start_idx
    )
    teacher_attn = extract_causal_attention_policy(
        teacher_out.attentions[-1], generated_start_idx
    )

    # Step 4: Pure structural imitation — no advantage weighting
    attn_distill_loss = jsd_loss(student_attn, teacher_attn).mean()

    # Step 5: Combine with GKD on output logits
    gkd_loss = compute_gkd_loss(student_out.logits, teacher_out.logits)

    return mu * gkd_loss + gamma_attn * attn_distill_loss
```

**Example 3: RAL-zero — attention optimization without chain-of-thought**

User: "I want to improve my MLLM's video understanding but I can't afford the inference cost of long thinking chains. Can RAL help without verbose reasoning?"

Approach:
1. Configure RAL in "zero" mode — no `<think>` block, direct answer only
2. Apply attention RL loss on the short answer generation sequence
3. The model learns better visual attention patterns without producing reasoning tokens

Output:
```python
# RAL-zero configuration
ral_zero_config = {
    "use_thinking_template": False,     # No <think>...</think> block
    "answer_template": "<answer>{}</answer>",
    "reward_weights": {"accuracy": 1.0, "format": 0.0},  # Only accuracy matters
    "lambda_attn": 1.0,
    "rollouts_per_input": 8,
    "temperature": 0.9,
    "extract_attention_from": "last_layer",
    "average_across_heads": True,
}

# RAL-zero achieves strong results without any reasoning overhead:
# NExTQA: 76.2 (vs 73.1 GRPO, 72.2 base)
# VideoMME: 65.1 (vs 63.8 GRPO)
# This works because attention optimization directly improves
# visual grounding — the model learns WHERE to look, not what to say.
```

## Best Practices

- **Do:** Extract attention from the *last* transformer layer only — earlier layers capture low-level patterns that are less amenable to task-level optimization. Average across all heads to get a stable signal.
- **Do:** Use Jensen-Shannon Divergence (not KL divergence) for the attention loss. JSD is symmetric and bounded, preventing the training instability that asymmetric KL causes on peaked attention distributions.
- **Do:** Start with `lambda_attn=1.0` for the attention loss weight. If training becomes unstable, reduce to 0.5. Values above 5.0 tend to over-regularize the attention and hurt token-level RL learning.
- **Do:** Freeze the visual encoder and multimodal projector. RAL operates on the language model's attention over already-encoded visual tokens — unfreezing the encoder introduces conflicting gradient signals.
- **Avoid:** Applying RAL to pure text-only tasks. The method's advantage comes from optimizing attention allocation across modalities. For text-only reasoning, standard GRPO or token-level RL is more appropriate.
- **Avoid:** Using flash attention or SDPA during RAL training. These kernels don't return attention weight matrices. You must patch the model to use eager attention in the target layer to extract the distributions needed for the loss.

## Error Handling

- **Attention weights are all zeros or NaN:** This usually means flash attention is still active. Verify the last layer is patched to eager attention. Check with `model.config._attn_implementation` or the layer's `forward` method.
- **JSD loss explodes to infinity:** Attention distributions may contain exact zeros. Add epsilon clamping: `attn_policy = attn_policy.clamp(min=1e-8)` before computing JSD. Re-normalize after clamping.
- **No improvement over GRPO baseline:** Check that `lambda_attn` is not too low (effectively ignoring attention loss) or too high (drowning out the token RL signal). Also verify that the visual token count is substantial — RAL gains scale with visual density. Below 512 visual tokens, improvements may be marginal.
- **Out of memory during rollout phase:** Eager attention materializes the full (seq_len, seq_len) attention matrix. For long sequences, extract attention only for the generated token positions, not the full context. Alternatively, checkpoint the attention extraction.
- **Distillation makes student worse:** Ensure trajectories are generated on-policy (from the student). Running teacher-generated trajectories through the student creates exposure bias — the student sees token sequences it would never produce, making the attention targets misleading.

## Limitations

- **Requires eager attention at training time.** Flash attention and SDPA cannot return attention matrices. This increases memory usage proportional to sequence length squared in the target layer, limiting maximum context during training.
- **Only optimizes the last transformer layer.** The paper found this sufficient, but some architectures may distribute visual reasoning across multiple layers. Multi-layer RAL is unexplored.
- **Limited to multimodal tasks.** The core advantage — optimizing cross-modal attention allocation — does not transfer to text-only settings where attention patterns are already well-structured from pretraining.
- **Requires group sampling (G=8).** Each input needs 8 rollouts to compute stable advantages, making RAL ~8x more expensive per training step than supervised fine-tuning. On 8xH100s, expect ~120 hours for 51.2k examples.
- **Reward function must be rule-based and immediate.** RAL inherits GRPO's requirement for a computable reward signal. Tasks without clear correctness criteria (open-ended generation, creative writing) cannot use this framework directly.

## Reference

**Paper:** [Reinforced Attention Learning](https://arxiv.org/abs/2602.04884v1) — Li, Ni, Qu, Miao, Yang (2026). Look for Section 3 (method formulation of attention policies and JSD-based loss), Section 4 (on-policy attention distillation), and Tables 2-3 (benchmark results showing consistent gains over GRPO across image and video tasks).