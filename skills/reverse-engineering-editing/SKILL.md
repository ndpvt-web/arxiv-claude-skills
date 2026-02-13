---
name: "reverse-engineering-editing"
description: "Audit and defend locate-then-edit model editing methods (ROME, MEMIT, AlphaEdit) against data leakage via spectral analysis of weight updates. Use when: 'audit my model edit for data leakage', 'reverse-engineer what was edited in this model', 'check if edited subjects can be recovered from weight diffs', 'add subspace camouflage defense to model editing', 'analyze ΔW for fingerprints', 'secure my ROME/MEMIT edits against reconstruction attacks'."
---

This skill enables Claude to analyze parameter updates from locate-then-edit model editing methods (ROME, MEMIT, AlphaEdit) to determine whether edited data can be reverse-engineered, and to apply the subspace camouflage defense that prevents such reconstruction. It implements the KSTER (KeySpaceReconstruction-then-EntropyReduction) attack framework from Sun et al. (2026) for red-teaming and the corresponding defense for hardening edits in production.

## When to Use

- When the user wants to audit whether a model edit leaks the edited subject through the weight update matrix ΔW
- When the user asks to reverse-engineer what knowledge was edited in a model by comparing pre- and post-edit weights
- When the user needs to implement or run the KSTER two-stage attack (subject inference + prompt recovery) against ROME, MEMIT, or AlphaEdit edits
- When the user wants to defend model edits against reconstruction by applying subspace camouflage with semantic decoys
- When the user is building a model editing pipeline and needs to verify that edited facts cannot be extracted from published weight diffs
- When the user is doing security research on LLM knowledge editing and needs to understand the spectral fingerprint vulnerability

## Key Technique

Locate-then-edit methods (ROME, MEMIT, AlphaEdit) modify specific FFN weight matrices to inject or update factual knowledge. They compute a weight update ΔW such that `(W + ΔW) * k* = v*`, where `k*` is the key vector (subject's hidden state at the target layer) and `v*` is the desired output value. The critical vulnerability is that **the row space of ΔW encodes a recoverable fingerprint of every edited subject**. For ROME, `ΔW = r* k*^T C^{-1} / (k*^T C^{-1} k*)` is literally rank-one with `k*` determining the row space. For MEMIT, `ΔW = R K^T (C + K K^T)^{-1}` has row space spanning the column space of K, which stacks all subject key vectors.

**Stage 1 -- KeySpace Reconstruction:** Compute `M = ΔW * C` (where C is the key covariance matrix, estimable from a calibration corpus). Apply SVD to get `M = U Σ V^T`. The top-N right singular vectors `V_N` span the subject subspace. Score each candidate subject `s_i` by projecting its key vector: `ρ = ||V_N^T k_i|| / ||k_i||`. Subjects with high projection scores were edited. On Llama3-8B with MEMIT (N=100), this achieves 0.99 recall.

**Stage 2 -- Entropy Reduction:** For each recovered subject `s_i`, score candidate relation prompts `r_j` using: `Score(r_j | s_i) = [H(s_i, r_j; θ) - H(s_i, r_j; θ')] / [H(s_i, r_j; θ') + ε]`, where H is the output entropy and θ/θ' are pre/post-edit models. Edited prompts show sharp entropy drops (the model becomes more confident after editing), yielding top-20 recall of 0.94.

**Defense -- Subspace Camouflage:** Inject decoy key vectors to pollute the row space: `K_tilde = K + α * (||K|| / ||K_decoy||) * K_decoy`. Recompute the update as `ΔW_defense = ΔW K (K_tilde^T C^{-1} K)^{-1} K_tilde^T C^{-1}`. This preserves `ΔW * K = ΔW_defense * K` (edit utility intact) while burying the true subject fingerprint under decoy directions. At α=5, subject rank degrades from ~51 to ~394 while edit efficacy stays at 0.96.

## Step-by-Step Workflow

### Attack (Red-Team Audit)

1. **Extract the weight diff.** Obtain ΔW by subtracting pre-edit weights from post-edit weights at the target FFN layer (typically the shallowest edited layer). For ROME this is one layer; for MEMIT it spans a range -- use the shallowest.

2. **Estimate the key covariance matrix C.** Run a calibration corpus (e.g., 100K tokens from Wikipedia) through the model, collecting hidden states at the target layer's FFN input. Compute `C = (1/n) Σ k_i k_i^T`. Cache this -- it is model-specific, not edit-specific.

3. **Compute the fingerprint matrix M.** Multiply: `M = ΔW @ C`. This transforms the update into a space where subject key vectors are directly recoverable.

4. **Run SVD on M.** Decompose `M = U Σ V^T`. Extract the top-N right singular vectors `V_N` (where N is the suspected number of edits; if unknown, use the singular value gap to estimate rank).

5. **Build a candidate subject vocabulary.** Tokenize a large entity list (e.g., Wikidata entities, CounterFact subjects) and compute their key vectors `k_i` by running each through the model to the target layer.

6. **Score and rank candidates.** For each candidate, compute `ρ_i = ||V_N^T k_i|| / ||k_i||`. Sort descending. The top-N candidates are the recovered edited subjects.

7. **Recover prompts via entropy reduction.** For each recovered subject `s_i`, enumerate candidate relation templates (e.g., "The capital of [X] is", "The CEO of [X] is"). Feed `(s_i, r_j)` through both θ and θ'. Compute the entropy score. Rank by score descending -- the highest-scoring template is the recovered edit prompt.

8. **Report findings.** Output the recovered (subject, relation, target) triples with confidence scores. Flag any edits where sensitive data (PII, proprietary facts) was recovered.

### Defense (Hardening)

9. **Generate semantic decoys.** Sample decoy subjects from the same domain as the real edits (e.g., if editing country capitals, sample other country names). Compute their key vectors to form `K_decoy`.

10. **Apply subspace camouflage.** Compute `K_tilde = K + α * (||K||_2 / ||K_decoy||_2) * K_decoy` with α in [3, 10]. Recompute the defended update: `ΔW_defense = ΔW @ K @ inv(K_tilde^T @ C_inv @ K) @ K_tilde^T @ C_inv`. Replace the original ΔW with ΔW_defense before publishing.

## Concrete Examples

**Example 1: Auditing a MEMIT batch edit on Llama3-8B**

User: "I applied MEMIT to edit 50 facts in Llama3-8B. Can an attacker figure out what I edited from the weight diff?"

Approach:
1. Load the pre-edit and post-edit model checkpoints
2. Extract ΔW at the shallowest edited MLP layer (e.g., layer 4's `mlp.down_proj`)
3. Estimate C from 100K Wikipedia tokens at that layer
4. Compute M = ΔW @ C, run SVD, extract top-50 right singular vectors
5. Score CounterFact entity key vectors against V_50
6. Report which subjects are recoverable

Output:
```
Subject Recovery Audit (MEMIT, Llama3-8B, layer 4)
===================================================
Edits analyzed: 50
Subjects recovered (top-50 recall): 48/50 (96%)

Top recovered subjects with projection scores:
  1. "Eiffel Tower"    ρ = 0.94
  2. "Barack Obama"    ρ = 0.91
  3. "Tesla Motors"    ρ = 0.89
  ...
  48. "Lake Baikal"    ρ = 0.52

RISK: HIGH -- Nearly all edited subjects are recoverable from the
published weight diff via spectral analysis. Apply subspace camouflage
before releasing these weights.
```

**Example 2: Defending edits with subspace camouflage**

User: "Secure my ROME edit so the subject can't be recovered from the weight update."

Approach:
1. Identify the edited subject key vector K and the current ΔW
2. Sample 20 decoy entities from the same semantic domain
3. Compute decoy key vectors K_decoy
4. Apply camouflage with α=5
5. Verify edit still works, then verify subject is no longer top-ranked

Output:
```python
import torch

def apply_subspace_camouflage(delta_W, K, K_decoy, C_inv, alpha=5.0):
    """Replace ΔW with a defended version that preserves edit utility
    but buries the subject fingerprint under decoy directions."""
    # Normalize and blend
    scale = alpha * torch.norm(K) / torch.norm(K_decoy)
    K_tilde = K + scale * K_decoy  # (d_in, N + N_decoy)

    # Recompute defended update preserving ΔW @ K = ΔW_def @ K
    A = K_tilde.T @ C_inv @ K          # (N+N_d, N)
    B = K_tilde.T @ C_inv              # (N+N_d, d_in)
    delta_W_def = delta_W @ K @ torch.linalg.solve(A, B)
    return delta_W_def

# Post-defense verification:
# Subject rank before camouflage: 1  (trivially recoverable)
# Subject rank after camouflage:  394 (buried in noise)
# Edit efficacy (P(new_target | subject)): 0.97 -> 0.96 (preserved)
```

**Example 3: Estimating number of edits from a weight diff**

User: "I have a weight diff but don't know how many facts were edited. Can you estimate?"

Approach:
1. Compute M = ΔW @ C
2. Run SVD and plot singular values
3. Look for the spectral gap (sharp drop) to estimate rank = number of edits

Output:
```
Singular Value Analysis of ΔW @ C
===================================
σ_1  = 12.41
σ_2  = 11.87
σ_3  = 10.93
...
σ_10 = 8.21
σ_11 = 0.34  <-- spectral gap
σ_12 = 0.29

Estimated number of edits: 10
(Gap ratio σ_10/σ_11 = 24.1, threshold > 5.0 indicates clear rank boundary)
```

## Best Practices

**Do:**
- Always estimate C from a calibration corpus that matches the model's pretraining distribution (Wikipedia works well). A poorly estimated C degrades subject recovery accuracy significantly.
- Use the shallowest edited layer for analysis -- it contains the strongest subject fingerprint because deeper layers mix more contextual information.
- When applying subspace camouflage, select decoy entities from the same semantic domain as the real edits to make the decoys statistically indistinguishable from true subjects.
- Validate that camouflaged edits preserve utility by checking edit efficacy, paraphrase generality, and neighborhood specificity after applying the defense.

**Avoid:**
- Do not assume the attack fails on AlphaEdit just because it uses null-space projection -- the row space fingerprint persists (AlphaEdit recall is lower but still exploitable at ~0.70 for N=100).
- Do not set camouflage α too high (>10) without testing -- excessive noise degrades edit specificity even though it improves defense.
- Do not skip the entropy reduction stage when auditing -- subject recovery alone does not reveal *what* was changed, only *who* was edited. The full (subject, relation, target) triple requires both stages.
- Do not use random token sequences as decoys -- they are trivially separable from real subject vectors via clustering, defeating the defense.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| SVD returns no clear spectral gap | Too few edits or edits are nearly colinear | Lower the gap ratio threshold or use cross-validation with known edit counts |
| Subject recall is low despite correct ΔW | Wrong layer selected or C estimated from mismatched distribution | Verify layer index matches the editing config; re-estimate C from a larger/better-matched corpus |
| Camouflage breaks edit efficacy | α too large or K_decoy too close to K | Reduce α or sample decoys from adjacent (not identical) semantic categories |
| Entropy scores are flat across all relations | Pre/post models too similar at that prompt | Ensure θ and θ' genuinely differ; check that the edit was applied to the correct layer |
| Out-of-memory during C estimation | Covariance matrix too large for GPU | Use batch accumulation: `C += k_batch @ k_batch.T` over chunks; or use diagonal approximation for initial screening |

## Limitations

- **Requires weight-level access.** The attack needs the raw ΔW (or both pre/post checkpoints). It does not apply to black-box API-only models.
- **Covariance estimation dependency.** Subject recovery quality degrades if the attacker cannot estimate C well (e.g., unknown pretraining data distribution). Defense auditors with full model access do not face this limitation.
- **Prompt recovery is open-vocabulary constrained.** Stage 2 scores candidate relation templates, so it requires a pre-built template set covering plausible edit types. Truly novel relation patterns may be missed.
- **Scales with edit count.** For very large batch edits (N > 1000), SVD becomes expensive and the subject subspace may be harder to separate from the residual spectrum.
- **Defense utility tradeoff.** Subspace camouflage at high α (>7) begins to degrade neighborhood specificity -- edits start affecting semantically adjacent but unedited facts.
- **Limited to locate-then-edit methods.** Does not apply to fine-tuning-based editing, in-context editing, or methods that do not produce a localized ΔW.

## Reference

Sun, Z., Luo, M., Wang, Y., Chen, Z., & He, T. (2026). *Reverse-Engineering Model Editing on Language Models.* arXiv:2602.10134v1. [https://arxiv.org/abs/2602.10134v1](https://arxiv.org/abs/2602.10134v1)

Key sections: Theorem E.13 (subject recovery guarantee), Algorithm 1 (KSTER attack pipeline), Section 5 (subspace camouflage formulation), Tables 1-3 (attack success rates across GPT-J, Llama3-8B, Qwen2.5-7B).