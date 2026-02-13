---
name: "beyond-alignment-expanding-reasoning"
description: "Apply Manifold-Reshaping Policy Optimization (MRPO) to expand LLM reasoning capacity beyond alignment. Implements spectral orthogonal exploration and effective rank regularization for RL-based training pipelines. Use when: 'implement MRPO training', 'expand reasoning with spectral exploration', 'add effective rank regularization to GRPO', 'break out of low-rank bias manifold', 'build RLVR pipeline with geometric interventions', 'improve math reasoning with rank-aware rewards'."
---

# Beyond Alignment: Expanding Reasoning via Manifold-Reshaping Policy Optimization

This skill enables Claude to implement **Manifold-Reshaping Policy Optimization (MRPO)**, a two-stage geometric framework that fundamentally expands the reasoning capacity of LLMs beyond what standard RLVR alignment achieves. Rather than merely surfacing latent capabilities already encoded in pre-training, MRPO restructures the inference space by first projecting the policy into the null space of its bias manifold (via Spectral Orthogonal Exploration), then maintaining high-dimensional reasoning trajectories through effective rank regularization during policy optimization. A 4B-parameter model trained with MRPO outperforms Qwen3-32B on mathematical benchmarks by over 23% on AIME 2024.

## When to Use

- When building an RL fine-tuning pipeline (GRPO, PPO, REINFORCE) for math or code reasoning and you want to push beyond the model's pre-trained capability boundary
- When the user asks to implement spectral orthogonal exploration or null-space projection for LLM training cold-starts
- When adding effective rank regularization to an existing policy optimization loop to prevent reasoning collapse
- When constructing a synthetic dataset of high-rank reasoning traces for supervised fine-tuning (Stage I of MRPO)
- When diagnosing whether a model's RL training is stuck on its bias manifold (low effective rank, no genuine exploration)
- When the user wants to implement rank-aware reward shaping that incentivizes geometrically diverse reasoning paths

## Key Technique

**The Problem: Low-Rank Bias Manifold Confinement.** Standard RL (including GRPO) explores within the subspace already carved out by pre-training. The model's hidden-state trajectories during reasoning live on a low-rank manifold -- the top-k principal components of the Gram matrix of hidden states capture nearly all variance. Entropy-reducing gradient updates during RL further contract this manifold, a phenomenon called *reasoning collapse*. The result: RL "aligns" existing capabilities but does not genuinely expand them.

**MRPO's Two-Stage Solution.** Stage I uses *Spectral Orthogonal Exploration (SOE)* to generate training data that lives in the **null space** of the bias manifold. A weaker "student" model generates candidate continuations; these are projected against the principal components of a stronger "teacher" model's hidden states, and only continuations with high orthogonality score (near 1.0) are retained. This produces ~10,000 correct, high-rank reasoning traces used for 1 epoch of SFT, ejecting the policy initialization away from the bias manifold. Stage II then runs GRPO with a **rank-augmented reward**: the standard correctness reward is multiplied by a normalized effective rank bonus computed over sliding windows of the hidden-state trajectory. The effective rank -- the exponential of the spectral entropy of the normalized singular values -- measures the geometric dimensionality of the reasoning path. This prevents the policy from collapsing back onto the low-rank manifold during optimization.

**Why It Works.** Logistic regression analysis shows effective rank is a statistically significant predictor of correctness (coefficient 0.56, p << 10^-5), while token-level entropy is not. High-dimensional reasoning paths access genuinely new solution strategies unavailable on the original manifold.

## Step-by-Step Workflow

### Stage I: Spectral Orthogonal Exploration (SOE) Cold-Start

1. **Select teacher-student model pair.** Choose a stronger model as "teacher" (e.g., Qwen3-4B-Instruct) and a weaker, architecturally different model as "student" (e.g., Gemma-3-4B). Non-overlapping pre-training manifolds are critical -- using the same model family defeats the purpose.

2. **Collect teacher hidden states at failure points.** For each problem where the teacher produces an incorrect trace, extract hidden states from the reasoning context `x_{<t}`. Form a centered matrix `H` from these states and compute the Gram matrix `G = H * H^T`.

3. **Extract bias manifold via micro-SVD.** Decompose `G` to obtain principal components `U_parallel` spanning the teacher's bias manifold. The threshold is: select top-k components such that `sum(lambda_1..k) / sum(lambda_1..d) >= 1 - epsilon` (typically epsilon ~ 0.05).

4. **Generate and score student probes.** Have the student generate `M=8` candidate continuation probes `s_j`. Map each to the teacher's latent space as `z_j`. Compute the orthogonality score: `Omega(s_j) = ||(I - U_parallel * U_parallel^T)(z_j - mu)|| / (||z_j - mu|| + epsilon)`. Values near 1.0 indicate null-space residence.

5. **Select, stitch, and verify.** Choose the probe maximizing Omega, stitch it into the teacher's context, and continue generation. Use `n=16` sampling budget per problem. Retain only traces that are both correct (verified symbolically) and have high orthogonality. Target ~10,000 high-rank traces.

6. **Run SFT cold-start.** Fine-tune the base model on the SOE-generated dataset for 1 epoch with standard cross-entropy loss. This initializes the policy outside the bias manifold.

### Stage II: Rank-Aware GRPO Optimization

7. **Compute sliding-window effective rank for each trajectory.** For each generated response, extract hidden states and compute effective rank over windows of size `w=64`: `erank(H) = exp(-sum(p_i * ln(p_i)))` where `p_i = lambda_i / sum(lambda_j)` from the SVD of the windowed hidden-state matrix. Take the minimum rank over the full trajectory.

8. **Construct rank-augmented rewards.** For each sample: `R_total(y) = I(y is correct) * (1.0 + alpha * NormRank(y))` where `alpha=0.5` and `NormRank` normalizes the effective rank across the batch. Incorrect answers receive zero reward regardless of rank.

9. **Run GRPO with rank-aware advantages.** Compute group-relative advantages: `A_hat_i = (R_total(y_i) - R_bar) / sigma_R` with group size `G=8` samples per prompt. Optimize the standard GRPO objective with these modified advantages for 4 epochs. Use learning rate `1e-6`, AdamW optimizer, global batch size 64, BF16 precision, and DeepSpeed ZeRO-3.

10. **Validate effective rank is maintained.** Monitor that the model's average effective rank does not decrease across training. If rank contracts below the SOE cold-start level, increase `alpha` or add explicit rank floor constraints.

## Concrete Examples

**Example 1: Implementing SOE Probe Scoring**

User: "Help me implement the orthogonality scoring function for SOE probes."

Approach:
1. Implement SVD-based bias manifold extraction
2. Implement null-space projection and scoring
3. Add probe selection logic

Output:
```python
import torch
import torch.nn.functional as F

def extract_bias_manifold(hidden_states: torch.Tensor, variance_threshold: float = 0.95):
    """Extract principal components spanning the bias manifold.

    Args:
        hidden_states: (N, D) tensor of centered hidden states from teacher
        variance_threshold: fraction of variance to capture (1 - epsilon)
    Returns:
        U_parallel: (D, k) orthonormal basis of bias manifold
    """
    H = hidden_states - hidden_states.mean(dim=0, keepdim=True)
    U, S, Vt = torch.linalg.svd(H, full_matrices=False)
    explained = torch.cumsum(S ** 2, dim=0) / (S ** 2).sum()
    k = (explained >= variance_threshold).nonzero(as_tuple=True)[0][0].item() + 1
    return Vt[:k].T  # (D, k)

def orthogonality_score(
    probes: torch.Tensor,
    U_parallel: torch.Tensor,
    mu: torch.Tensor,
    eps: float = 1e-8
) -> torch.Tensor:
    """Score probes by null-space residence. Higher = more orthogonal to bias manifold.

    Args:
        probes: (M, D) candidate probe embeddings in teacher latent space
        U_parallel: (D, k) bias manifold basis
        mu: (D,) mean of teacher hidden states
    Returns:
        scores: (M,) orthogonality scores in [0, 1]
    """
    centered = probes - mu.unsqueeze(0)                     # (M, D)
    projection = centered @ U_parallel @ U_parallel.T       # (M, D)
    residual = centered - projection                        # (M, D)
    scores = residual.norm(dim=1) / (centered.norm(dim=1) + eps)
    return scores  # Omega ~ 1.0 means full null-space residence

def select_best_probe(probes, U_parallel, mu):
    scores = orthogonality_score(probes, U_parallel, mu)
    best_idx = scores.argmax().item()
    return best_idx, scores[best_idx].item()
```

**Example 2: Adding Effective Rank Regularization to a GRPO Training Loop**

User: "I have a working GRPO training loop. How do I add effective rank regularization?"

Approach:
1. Implement sliding-window effective rank computation
2. Modify the reward function to include rank bonus
3. Integrate into existing GRPO advantage computation

Output:
```python
import torch
import math

def effective_rank(hidden_states: torch.Tensor) -> float:
    """Compute effective rank (exponential of spectral entropy) of hidden states.

    Args:
        hidden_states: (T, D) tensor of hidden states for a sequence window
    Returns:
        erank: scalar effective rank value
    """
    H = hidden_states - hidden_states.mean(dim=0, keepdim=True)
    _, S, _ = torch.linalg.svd(H, full_matrices=False)
    S = S[S > 1e-10]  # filter numerical zeros
    p = S / S.sum()
    spectral_entropy = -(p * p.log()).sum().item()
    return math.exp(spectral_entropy)

def trajectory_min_rank(hidden_states: torch.Tensor, window_size: int = 64) -> float:
    """Compute minimum effective rank over sliding windows of a trajectory."""
    T = hidden_states.shape[0]
    if T <= window_size:
        return effective_rank(hidden_states)
    ranks = []
    for start in range(0, T - window_size + 1, window_size // 2):
        window = hidden_states[start:start + window_size]
        ranks.append(effective_rank(window))
    return min(ranks)

def rank_augmented_rewards(
    correctness: torch.Tensor,      # (B,) bool
    hidden_states_list: list,        # list of (T_i, D) tensors
    alpha: float = 0.5,
    window_size: int = 64
) -> torch.Tensor:
    """Compute R_total = correctness * (1 + alpha * NormRank)."""
    raw_ranks = torch.tensor([
        trajectory_min_rank(hs, window_size) for hs in hidden_states_list
    ])
    # Normalize ranks to [0, 1] within batch
    rank_min, rank_max = raw_ranks.min(), raw_ranks.max()
    if rank_max - rank_min > 1e-8:
        norm_ranks = (raw_ranks - rank_min) / (rank_max - rank_min)
    else:
        norm_ranks = torch.zeros_like(raw_ranks)

    rewards = correctness.float() * (1.0 + alpha * norm_ranks)
    return rewards

# Integration into GRPO advantage computation:
def grpo_advantages(rewards: torch.Tensor, group_size: int = 8) -> torch.Tensor:
    """Compute group-relative advantages from rank-augmented rewards."""
    B = rewards.shape[0]
    advantages = torch.zeros_like(rewards)
    for i in range(0, B, group_size):
        group = rewards[i:i + group_size]
        mean, std = group.mean(), group.std() + 1e-8
        advantages[i:i + group_size] = (group - mean) / std
    return advantages
```

**Example 3: Diagnosing Bias Manifold Confinement**

User: "My RL-trained model isn't improving on hard math problems. How do I check if it's stuck on its bias manifold?"

Approach:
1. Extract hidden states from correct vs incorrect solutions
2. Compute effective rank for both populations
3. Compare against pre-training baseline

Output:
```python
def diagnose_manifold_confinement(model, tokenizer, problems, device="cuda"):
    """Check if model reasoning is confined to a low-rank manifold."""
    correct_ranks, incorrect_ranks = [], []

    for problem in problems:
        for _ in range(8):  # sample multiple solutions
            output = model.generate(tokenizer.encode(problem, return_tensors="pt").to(device),
                                     max_new_tokens=4096, output_hidden_states=True)
            # Extract last-layer hidden states across generated tokens
            hs = torch.stack([h[-1][0, -1, :] for h in output.hidden_states[1:]])
            rank = trajectory_min_rank(hs, window_size=64)

            is_correct = verify_answer(output, problem)  # your verification fn
            if is_correct:
                correct_ranks.append(rank)
            else:
                incorrect_ranks.append(rank)

    print(f"Correct solutions  - mean rank: {sum(correct_ranks)/len(correct_ranks):.1f}")
    print(f"Incorrect solutions - mean rank: {sum(incorrect_ranks)/len(incorrect_ranks):.1f}")
    # If both are low (< 10) and similar, the model is manifold-confined.
    # MRPO Stage I (SOE cold-start) is indicated.
```

## Best Practices

- **Do:** Use architecturally different models for teacher-student SOE pairs. Same-family models share manifold structure, reducing null-space diversity.
- **Do:** Verify SOE-generated traces symbolically (not just by LLM self-evaluation). Incorrect traces in the cold-start SFT data will embed errors into the expanded manifold.
- **Do:** Monitor effective rank throughout training -- it should remain at or above the post-SOE level. Plot rank curves alongside loss curves.
- **Do:** Set `alpha=0.5` for the rank bonus as a starting point. Too high risks rewarding incoherent high-rank gibberish; too low fails to prevent collapse.
- **Avoid:** Applying SOE to problems the teacher already solves correctly. SOE targets failure-point hidden states specifically -- correct solutions already live on the accessible manifold.
- **Avoid:** Skipping Stage I and going directly to rank-regularized GRPO. Without the SOE cold-start, the policy initialization is still on the bias manifold and rank regularization alone cannot eject it.

## Error Handling

| Issue | Symptom | Fix |
|-------|---------|-----|
| SOE produces few valid traces | <500 traces after filtering | Lower the orthogonality threshold from 0.8 to 0.6, increase sampling budget beyond n=16, or use a more diverse student model |
| Effective rank collapses during Stage II | Rank drops below pre-SOE baseline within first epoch | Increase alpha (try 0.8-1.0), reduce learning rate, or add an explicit rank floor as a hard constraint |
| Training becomes unstable | Loss spikes, NaN gradients | Reduce learning rate below 1e-6, enable gradient clipping, verify BF16 precision is not causing SVD numerical issues -- use FP32 for rank computation |
| Student probes all score Omega ~ 0 | Student and teacher share the same manifold | Switch to a student from a different model family or pre-training data distribution |
| Rank-augmented rewards all identical | NormRank has no variance in batch | Increase group size G, sample longer reasoning chains, or increase generation temperature |

## Limitations

- **Requires access to hidden states during training.** MRPO cannot be applied as a black-box wrapper around API-only models. You need full model weights and intermediate activations.
- **Computational overhead.** SVD computation on hidden-state windows adds ~15% to iteration time. The SOE stage requires running two models simultaneously.
- **4B-parameter scale validated.** The paper demonstrates results at 4B parameters. Scaling behavior to 70B+ is untested -- the rank regularization strength may need retuning.
- **Math-focused benchmarks.** Results are validated on AIME, MATH-500, and OlympiadBench. Transfer to code reasoning, natural language inference, or open-ended tasks is not established.
- **SOE requires a suitable student-teacher mismatch.** If no sufficiently orthogonal student model is available, the cold-start stage may produce insufficient high-rank traces.
- **Does not replace good pre-training.** MRPO expands the RL-accessible manifold but still requires a capable base model. It cannot conjure capabilities from an undertrained foundation.

## Reference

**Paper:** [Beyond Alignment: Expanding Reasoning Capacity via Manifold-Reshaping Policy Optimization](https://arxiv.org/abs/2602.02545v1) (Wang et al., 2026). Look for: Section 3 (bias manifold formalization), Algorithm 1 (SOE procedure), Section 4 (rank-augmented GRPO objective), Table 1 (benchmark results showing 4B > 32B).

**Key hyperparameters to reproduce:** LR=1e-6, alpha=0.5, G=8, w=64, SOE traces=10K, SFT=1 epoch, GRPO=4 epochs, BF16, DeepSpeed ZeRO-3.