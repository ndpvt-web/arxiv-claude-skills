---
name: "winflora-incentivizing-client-adaptive-aggregation"
description: "Implement privacy-heterogeneous federated LoRA fine-tuning with noise-aware incentive aggregation (WinFLoRA). Use when: 'set up federated LoRA with differential privacy', 'aggregate LoRA adapters from clients with different privacy levels', 'implement noise-aware weighted aggregation for federated learning', 'build incentive mechanism for federated fine-tuning', 'estimate client noise from LoRA updates', 'federated LLM fine-tuning with privacy heterogeneity'."
---

# WinFLoRA: Privacy-Heterogeneous Federated LoRA with Noise-Aware Incentive Aggregation

This skill enables Claude to implement WinFLoRA, a federated LoRA fine-tuning system where clients inject varying levels of differential privacy (DP) noise into their LoRA adapters. The core innovation is a three-part mechanism: (1) **LOO-PCA noise estimation** that infers each client's noise level from their uploaded LoRA matrices without requiring trust or disclosure, (2) **Noise-aware Weight Allocation (NWA)** that assigns higher aggregation weights to lower-noise clients, and (3) **Individual Noise Adaptation (INA)** where clients use a UCB bandit strategy to balance privacy protection against downstream performance. Together, these align individual incentives with global model quality — clients who contribute cleaner updates get more influence over the global model.

## When to Use

- When the user wants to **fine-tune an LLM across multiple clients** (e.g., hospitals, enterprises) that have different privacy requirements and cannot share raw data.
- When implementing **federated LoRA** and clients inject different amounts of DP noise, causing naive averaging (FedAvg) to degrade the global model.
- When the user needs to **estimate noise levels from uploaded model updates** without clients self-reporting their privacy budgets.
- When building an **incentive-compatible aggregation** mechanism where contributing higher-quality (lower-noise) updates is rewarded with more influence on the shared model.
- When the user asks to implement **differential privacy in federated learning** and wants the aggregation to be robust to heterogeneous noise scales.
- When deploying federated fine-tuning for **web-service LLMs** (chatbots, search, recommendation) where each tenant has distinct privacy policies.

## Key Technique

**The Problem:** In federated LoRA, each client fine-tunes low-rank adapters (A, B matrices) locally and uploads them. When clients add different amounts of DP noise (some add heavy noise for strong privacy, others add little), naive aggregation treats all updates equally. This lets high-noise clients free-ride on low-noise contributors while dragging down global accuracy. There is no incentive for clients to contribute clean updates.

**The Solution — LOO-PCA Noise Estimation:** WinFLoRA estimates each client's noise without requiring disclosure. For each client i, the server builds a "public subspace" from all *other* clients' LoRA B matrices using PCA (via SVD). It projects client i's update onto this subspace — the component that aligns with the shared signal. The residual (the part orthogonal to the subspace) is attributed to noise. The estimated noise variance is `σ̂ᵢ² = ‖rᵢ‖² / max(d_B - K, 1)`, where rᵢ is the residual and K is the subspace rank. This leave-one-out design prevents a client's own noise from contaminating its own estimate.

**Aggregation as Incentive + Client Adaptation:** Aggregation weights are set inversely proportional to estimated noise: `wᵢ = (1/(σ̂ᵢ + τ)) / Σⱼ(1/(σ̂ⱼ + τ))`, where τ ≈ 1e-8 stabilizes division. Clients who add less noise get more weight — hence more influence on the global model and better downstream accuracy on their tasks. On the client side, each client runs a UCB multi-armed bandit over a discrete set of noise levels (e.g., {0, 0.1, 0.5, 1.0}) to maximize a utility function that balances task performance against privacy benefit. Over rounds, clients converge to noise levels that optimally trade off their individual privacy needs against model quality.

## Step-by-Step Workflow

1. **Initialize LoRA adapters on a base LLM.** Choose rank r (typically 8-16), target modules (query/value projections), and initialize A with Kaiming and B with zeros per standard LoRA. Distribute the same initial adapters to all N clients.

2. **Configure the client privacy action space.** Define a discrete set of noise scales Σ = {0, 0.1, 0.5, 1.0} (or finer granularity). Each client starts with a uniform prior over these actions and initializes UCB parameters: empirical utility μ̂ₖ = 0 for each action k, visit counts nₖ = 0, exploration parameter κ (e.g., 1.0), and EMA decay β (e.g., 0.3).

3. **Run local LoRA fine-tuning on each client.** Each client trains on its local (non-IID) data for a fixed number of local steps, producing updated ΔAᵢ, ΔBᵢ matrices.

4. **Inject calibrated DP noise into adapters before upload.** Each client selects a noise action σᵢ via UCB: `Iₖ = μ̂ₖ + κ·sqrt(2·ln(t) / max(1, nₖ))`, picks argmax, then adds Gaussian noise ξ ~ N(0, σᵢ²·I) independently to both A and B matrices. Upload the noised (Aᵢ + ξₐ, Bᵢ + ξᵦ) to the server.

5. **Estimate per-client noise via LOO-PCA on the server.** For each client i: (a) stack all other clients' flattened B matrices into X⁻ⁱ, (b) center the data, (c) compute SVD, (d) select top-K singular vectors as the public subspace basis, (e) project client i's centered vector onto this subspace, (f) compute residual rᵢ and estimate `σ̂ᵢ² = ‖rᵢ‖² / max(d_B - K, 1)`.

6. **Compute noise-aware aggregation weights.** Calculate raw scores `sᵢ = 1/(σ̂ᵢ + τ)` with τ = 1e-8, then normalize: `wᵢ = sᵢ / Σⱼ sⱼ`. These weights replace the uniform 1/N used in naive FedAvg.

7. **Aggregate into the global LoRA update.** Use block-stacking aggregation: `ΔW_global = Σᵢ wᵢ · Bᵢ · Aᵢ`. Apply this to the frozen base model weights for inference.

8. **Distribute the global model back to clients and update bandit state.** Each client evaluates local accuracy on a validation split, computes utility `Uᵢ = G(accuracy) + γᵢ · (σᵢ/σ_max)`, and updates the UCB empirical estimate: `μ̂ₖ ← (1-β)·μ̂ₖ + β·Uᵢ` for the action taken. Increment visit count nₖ.

9. **Repeat for T communication rounds** (typically 15-20 rounds suffice). Monitor global accuracy, average client utility, and noise distribution convergence across rounds.

10. **Evaluate and deploy.** After training, assess the global model on a held-out test set. Report per-client utilities and noise choices to verify incentive alignment — low-γ clients should converge to low noise, high-γ clients to higher noise.

## Concrete Examples

**Example 1: Federated text classification across organizations**

User: "I have 10 organizations that each have private text data. I want to fine-tune TinyLlama for news classification without sharing data. Some orgs need strong privacy (ε ≈ 3), others don't care. Set up WinFLoRA."

Approach:
1. Load TinyLlama base model, attach LoRA adapters (rank=8) to attention layers.
2. Partition AGNews dataset across 10 clients using Dirichlet(α=0.3) for realistic non-IID splits.
3. Define noise action space Σ = {0, 0.1, 0.5, 1.0} and assign privacy preference γᵢ ~ N(0.5, 0.01).
4. Each round: clients train locally for 1 epoch, select noise via UCB, upload noised adapters.
5. Server runs LOO-PCA, computes inverse-noise weights, aggregates.
6. After 20 rounds, evaluate.

Output:
```
Round 20 Results:
  Global Accuracy: 78.4% (vs. 51.5% with uniform FedAvg under same noise)
  Avg Client Utility: 0.72 (vs. 0.28 baseline)
  Client Noise Distribution:
    Low-privacy clients (γ < 0.4):  converged to σ = 0.0-0.1
    High-privacy clients (γ > 0.6): converged to σ = 0.5-1.0
  Aggregation weights range: [0.02, 0.18] (low-noise clients weighted ~9x more)
```

**Example 2: Implementing the LOO-PCA noise estimator in Python**

User: "Write the noise estimation function that takes a list of LoRA B matrices from clients and returns per-client noise estimates."

Approach:
1. Flatten each client's B matrix into a vector.
2. For each client, build the leave-one-out matrix from remaining clients.
3. Center, compute SVD, project, measure residual.

Output:
```python
import numpy as np

def estimate_client_noise(B_matrices: list[np.ndarray], K: int = None) -> np.ndarray:
    """Estimate per-client noise via Leave-One-Out PCA.

    Args:
        B_matrices: List of N LoRA B matrices, each shape (d_out, r).
        K: Number of principal components for public subspace.
            If None, uses min(N-2, r).

    Returns:
        Array of shape (N,) with estimated noise std per client.
    """
    N = len(B_matrices)
    X = np.array([B.flatten() for B in B_matrices])  # (N, d_B)
    d_B = X.shape[1]

    if K is None:
        K = min(N - 2, B_matrices[0].shape[1])

    noise_estimates = np.zeros(N)
    for i in range(N):
        # Leave-one-out: all clients except i
        X_minus_i = np.delete(X, i, axis=0)  # (N-1, d_B)
        mean_minus_i = X_minus_i.mean(axis=0)

        # Center
        X_centered = X_minus_i - mean_minus_i
        x_i_centered = X[i] - mean_minus_i

        # SVD for public subspace
        U, S, Vt = np.linalg.svd(X_centered, full_matrices=False)
        P_basis = Vt[:K]  # (K, d_B)

        # Project client i onto public subspace
        projection = P_basis.T @ (P_basis @ x_i_centered)
        residual = x_i_centered - projection

        # Noise estimate
        noise_var = np.dot(residual, residual) / max(d_B - K, 1)
        noise_estimates[i] = np.sqrt(max(noise_var, 0))

    return noise_estimates
```

**Example 3: Adding noise-aware aggregation to an existing federated LoRA loop**

User: "I already have a FedAvg loop for LoRA. Show me how to swap in noise-aware weighted aggregation."

Approach:
1. After collecting client LoRA updates, call the noise estimator.
2. Compute inverse-noise weights.
3. Replace uniform averaging with weighted sum.

Output:
```python
def noise_aware_aggregate(client_As, client_Bs, tau=1e-8):
    """Replace FedAvg with WinFLoRA's NWA aggregation.

    Args:
        client_As: List of N LoRA A matrices.
        client_Bs: List of N LoRA B matrices.
        tau: Numerical stability constant.

    Returns:
        delta_W: Aggregated weight update matrix.
        weights: Per-client aggregation weights (for logging).
    """
    # Step 1: Estimate noise from B matrices
    noise_stds = estimate_client_noise(client_Bs)

    # Step 2: Inverse-noise scoring
    scores = 1.0 / (noise_stds + tau)
    weights = scores / scores.sum()

    # Step 3: Weighted aggregation via block product
    delta_W = sum(w * (B @ A) for w, A, B in zip(weights, client_As, client_Bs))

    return delta_W, weights

# In your federated loop, replace:
#   delta_W = sum(B @ A for A, B in zip(As, Bs)) / N
# with:
#   delta_W, weights = noise_aware_aggregate(As, Bs)
```

## Best Practices

- **Do:** Run LOO-PCA on the B matrices specifically, not A. The paper finds B matrices carry more distinguishable signal for noise estimation because A matrices are initialized with Kaiming (higher variance) making noise harder to separate.
- **Do:** Use a discrete, coarse noise action space (4-6 levels). Finer granularity slows UCB convergence without meaningfully improving utility. Start with {0, 0.1, 0.5, 1.0}.
- **Do:** Set the EMA decay β between 0.2-0.5. Lower β gives more stable utility estimates but slower adaptation; higher β reacts faster but oscillates. β = 0.3 is a reliable default.
- **Do:** Log per-client weights each round. If one client consistently gets near-zero weight, investigate — they may be adding too much noise or have corrupted data.
- **Avoid:** Skipping the leave-one-out step. Using all clients (including i) to build the PCA subspace contaminates the noise estimate — client i's own noise inflates the subspace, causing systematic underestimation.
- **Avoid:** Setting τ too large (e.g., > 0.01). This flattens the weight distribution toward uniform, negating the incentive mechanism. Keep τ ≈ 1e-8 unless you have a specific reason to dampen weight disparity.

## Error Handling

- **Too few clients (N < 4):** LOO-PCA becomes unreliable because the leave-one-out subspace is built from only N-2 vectors. Fall back to uniform aggregation or use a minimum-client threshold before activating NWA.
- **Degenerate SVD / rank-deficient matrices:** When clients have very low LoRA rank or identical updates, SVD may produce near-zero singular values. Clip K to the number of non-trivial singular values (e.g., those above 1% of the largest).
- **All clients inject zero noise:** Noise estimates will all be near-zero, making scores blow up before normalization. The τ constant handles this — all weights become approximately uniform, which is correct behavior when there is no noise heterogeneity.
- **UCB exploration causing utility drops in early rounds:** The bandit needs exploration, so some clients will try high-noise actions early. This is expected. If it is unacceptable, warm-start μ̂ₖ with reasonable priors (e.g., set high-noise actions to lower initial estimates).
- **Non-IID data causing noise estimate bias:** Highly non-IID data distributions can make client updates structurally different in ways unrelated to noise. If noise estimates seem uncorrelated with actual noise levels, increase the number of communication rounds or use a data-heterogeneity-aware subspace rank K.

## Limitations

- **Assumes Gaussian DP noise model.** The LOO-PCA noise estimation assumes additive isotropic Gaussian noise. If clients use Laplace noise, local DP mechanisms, or non-standard noise distributions, the estimates will be biased.
- **Requires N ≥ 4 clients.** The leave-one-out PCA needs enough clients to build a meaningful subspace. With very few clients, the public subspace is too low-dimensional for reliable noise separation.
- **Honest-but-curious threat model.** WinFLoRA assumes clients follow the protocol (train locally, add noise as selected). It does not defend against Byzantine adversaries who upload arbitrary malicious updates.
- **Tested primarily on text classification.** Evaluations cover AGNews, DBpedia, and 20Newsgroups with TinyLlama and GPT2-Large. Generalization to generation tasks, vision-language models, or very large models (70B+) is plausible but unvalidated.
- **Communication cost scales with N.** LOO-PCA runs N separate SVD decompositions per round. For large N (hundreds of clients), consider batched or approximate methods.

## Reference

**Paper:** [WinFLoRA: Incentivizing Client-Adaptive Aggregation in Federated LoRA under Privacy Heterogeneity](https://arxiv.org/abs/2602.01126v1) — Kou et al., 2026. Focus on Algorithms 1-3 (LOO-PCA, NWA, INA) in Sections 3-4 and the ablation studies in Section 5 for parameter sensitivity guidance.