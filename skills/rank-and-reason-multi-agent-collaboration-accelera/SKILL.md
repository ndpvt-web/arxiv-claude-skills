---
name: "rank-and-reason-multi-agent-collaboration-accelera"
description: |
  Multi-agent Rank-and-Reason framework for zero-shot protein mutation prediction and selection.
  Implements VenusRAR's two-stage pipeline: (1) context-aware multi-model ensemble ranking with
  dynamic weight calibration, and (2) physics-informed expert panel auditing via chain-of-thought
  reasoning against geometric, structural, and biophysical constraints.

  Use this skill when the user says:
  - "Predict beneficial protein mutations" or "rank protein variants"
  - "Build a multi-agent ensemble for mutation scoring"
  - "Audit protein mutation candidates with structural constraints"
  - "Implement a rank-then-reason pipeline for protein engineering"
  - "Select mutations for wet-lab validation"
  - "Zero-shot protein fitness prediction"
---

# Rank-and-Reason: Multi-Agent Protein Mutation Prediction

This skill enables Claude to implement the VenusRAR two-stage agentic framework for zero-shot protein mutation prediction. The core idea: instead of trusting any single protein language model (PLM), orchestrate multiple specialized PLMs through a context-aware ensemble (Rank-Stage), then filter candidates through a panel of LLM-based expert agents that apply chain-of-thought reasoning against biophysical constraints (Reason-Stage). This approach achieved a Spearman correlation of 0.551 on ProteinGym and improved Top-5 Hit Rate by up to 367% over static ensemble baselines.

## When to Use

- When the user needs to predict which protein mutations are likely beneficial without experimental data (zero-shot)
- When building a pipeline that aggregates scores from multiple protein language models (ESM2, SaProt, GEMME, etc.)
- When the user wants to go beyond raw PLM scores by applying structural and biophysical validation
- When implementing multi-agent collaboration where agents have distinct expert roles (statistician, structural biologist, experimentalist)
- When selecting a small set of mutation candidates for expensive wet-lab experiments and maximizing expected hit rate
- When the user asks to reproduce or adapt the VenusRAR pipeline from the paper

## Key Technique

**The Correlation-Precision Gap.** Standard PLM ensembles optimize global Spearman correlation (how well the full ranking matches ground truth), but protein engineers care about *precision at the top*: are the top-N candidates actually beneficial? VenusRAR addresses this gap with two stages that separately handle ranking fidelity and top-candidate quality.

**Rank-Stage: Context-Aware Ensemble.** Two agents collaborate: a *Computational Expert* runs each PLM scorer (sequence-based: ESM2, ProGen3; structure-based: ProSST, ESM-IF1, SaProt; MSA-based: GEMME, VenusREM), and a *Virtual Biologist* dynamically calibrates model weights based on three signals: (1) *objective-prior calibration* — matching model inductive biases to the engineering goal (activity vs. stability vs. binding), (2) *structural calibration* — attenuating structure-based model weights where predicted structure confidence (pLDDT) is low (<50), and (3) *evolutionary calibration* — downweighting MSA-based models when the alignment is shallow. The final score is a weighted sum across all models and modalities.

**Reason-Stage: Expert Panel Auditing.** Three LLM-based agents form an expert panel that audits the top candidates via chain-of-thought reasoning. The *Statistical Auditor* constructs a high-recall candidate pool (top-200 from ensemble + top-200 from each individual model) and checks for ranking inconsistencies. The *Structural Biologist* evaluates geometric feasibility — rotamer configurations, backbone stability, pLDDT-conditional trust (preferring evolutionary consensus at low-confidence residues). The *Experimental Expert* flags developability risks: expression problems via relative solvent accessibility (RSA), net charge imbalance, and a biophysical exclusion set of reactive/disruptive residues. Candidates pass only if all three validators approve; rejected candidates are replaced by the next-best from the pool.

## Step-by-Step Workflow

1. **Parse inputs and define the engineering objective.** Collect the wild-type protein sequence, 3D structure (PDB/mmCIF or AlphaFold prediction), and MSA. Define the fitness objective explicitly: activity enhancement, thermostability, binding affinity, or expression yield. This objective drives weight calibration downstream.

2. **Run each PLM scorer independently.** Execute scoring across three modalities — sequence (ESM2, ProGen3), structure (ProSST-2048, ESM-IF1, SaProt-AF650M), and MSA (GEMME, VenusREM). Store raw per-variant scores in a unified table keyed by mutation identifier (e.g., `A42G`, `K156E`).

3. **Compute data quality signals for weight calibration.** Extract per-residue pLDDT from the structure prediction, MSA depth (number of effective sequences Neff), and model description metadata. These three signals feed the Virtual Biologist's calibration logic.

4. **Calibrate ensemble weights dynamically.** Implement the Virtual Biologist agent's three calibration rules:
   - *Objective-prior*: Boost models whose inductive bias aligns with the goal (e.g., upweight structure models for stability, MSA models for conserved function).
   - *Structural confidence*: For residues with pLDDT < 50, attenuate structure-based model weights toward zero.
   - *Evolutionary depth*: For MSA depth < threshold (typically Neff < 30), reduce MSA-based model weights.
   Compute calibrated weights `w_{j,t}` and produce the ensemble score: `S_rank(x) = sum(w * F(x))`.

5. **Construct the high-recall candidate pool.** Take the top-200 variants by calibrated ensemble score, PLUS the top-200 from each individual model. This yields a pool of up to `(num_models + 1) * 200` candidates, recovering variants that a single model ranks highly but the ensemble underestimates.

6. **Implement the Statistical Auditor agent.** This agent manages the candidate pool, detects ranking inconsistencies (variants ranked very differently across models), and enforces positional diversity (avoid clustering all picks at a single residue position).

7. **Implement the Structural Biologist agent.** For each candidate in descending ensemble-score order, evaluate: (a) Is the mutant rotamer geometrically feasible? (b) Is the backbone locally stable (no steric clashes)? (c) If pLDDT_i < 50 at the mutation site, defer to evolutionary consensus from MSA models rather than structure-based scores. Accept or reject with chain-of-thought justification.

8. **Implement the Experimental Expert agent.** For each candidate passing structural review, check: (a) RSA — is the residue buried or exposed, and does the mutation respect the hydrophobic/hydrophilic environment? (b) Net charge — does the mutation introduce a problematic charge at the surface? (c) Biophysical exclusion — reject mutations to/from cysteine (disulfide disruption), proline in helices (kink introduction), or glycine at tight turns (flexibility loss). Accept or reject with justification.

9. **Run the iterative accept/replace loop.** Walk candidates in descending score order. A candidate is accepted only if ALL three validators approve. On rejection, advance to the next candidate from the pool. Continue until the experimental budget N is filled (e.g., N=10, 20, or 30 variants).

10. **Output the final ranked selection with audit trails.** Produce a table of N selected mutations, each annotated with: ensemble score, per-model ranks, and the chain-of-thought reasoning from each expert agent explaining why it passed. This audit trail is critical for downstream experimental interpretation.

## Concrete Examples

**Example 1: Building a Rank-Stage Ensemble Scorer**

User: "I have ESM2, SaProt, and GEMME scores for 500 mutations of my enzyme. Help me build a calibrated ensemble."

Approach:
1. Load the three score vectors and normalize each to zero-mean, unit-variance (z-scores) for comparability.
2. Assess data quality: check pLDDT per residue from the AlphaFold structure, count MSA depth.
3. Assign initial weights: 1/3 each. Then calibrate:
   - If the goal is catalytic activity (functional), boost GEMME weight by 1.5x (evolutionary conservation captures active-site constraints).
   - For residues where pLDDT < 50, set SaProt weight to 0.1x (low-confidence structure).
   - If MSA has Neff < 30, set GEMME weight to 0.5x (shallow evolutionary signal).
4. Renormalize weights to sum to 1.0, compute weighted sum.

Output:
```python
import numpy as np
from scipy.stats import zscore

scores = {
    "ESM2": zscore(esm2_raw),
    "SaProt": zscore(saprot_raw),
    "GEMME": zscore(gemme_raw),
}

weights = {"ESM2": 1.0, "SaProt": 1.0, "GEMME": 1.5}  # objective-prior: boost MSA for activity

# Structural calibration: attenuate SaProt at low-confidence positions
for i, mut in enumerate(mutations):
    pos = int(mut[1:-1])  # e.g., "A42G" -> 42
    if plddt[pos] < 50:
        scores["SaProt"][i] *= 0.1

# Evolutionary calibration: attenuate GEMME if MSA is shallow
if neff < 30:
    weights["GEMME"] *= 0.5

# Normalize weights and compute ensemble
w_total = sum(weights.values())
ensemble = sum(weights[m] / w_total * scores[m] for m in scores)
top_200 = np.argsort(ensemble)[::-1][:200]
```

**Example 2: Implementing the Reason-Stage Expert Panel**

User: "I have ranked mutations. Help me build the expert panel to filter the top candidates."

Approach:
1. Define the three validator functions with chain-of-thought output.
2. Construct the high-recall pool from ensemble top-200 + per-model top-200.
3. Iterate through candidates, applying all three validators.
4. On rejection, log the reason and advance to the next candidate.

Output:
```python
def statistical_audit(mutation, ensemble_rank, per_model_ranks, selected_positions):
    """Statistical Auditor: check ranking consistency and positional diversity."""
    reasoning = []
    rank_spread = max(per_model_ranks.values()) - min(per_model_ranks.values())
    if rank_spread > 400:
        reasoning.append(f"WARN: High rank spread ({rank_spread}) across models — inconsistent signal.")

    pos = int(mutation[1:-1])
    if selected_positions.count(pos) >= 2:
        reasoning.append(f"REJECT: Position {pos} already has 2 selections — enforcing diversity.")
        return False, reasoning

    reasoning.append("PASS: Consistent ranking, position not over-represented.")
    return True, reasoning

def structural_review(mutation, plddt_score, rotamer_feasible, msa_consensus):
    """Structural Biologist: geometric and confidence checks."""
    reasoning = []
    pos = int(mutation[1:-1])

    if not rotamer_feasible:
        reasoning.append(f"REJECT: Mutant rotamer at position {pos} creates steric clash.")
        return False, reasoning

    if plddt_score < 50:
        target_aa = mutation[-1]
        if target_aa not in msa_consensus.get(pos, set()):
            reasoning.append(f"REJECT: Low pLDDT ({plddt_score:.0f}) and {target_aa} absent from MSA consensus.")
            return False, reasoning
        reasoning.append(f"NOTE: Low pLDDT but mutation matches MSA consensus — accepting.")

    reasoning.append("PASS: Rotamer feasible, structural confidence adequate.")
    return True, reasoning

def experimental_review(mutation, rsa, net_charge_delta, wt_aa, mut_aa):
    """Experimental Expert: developability and biophysical checks."""
    reasoning = []
    exclusion_pairs = {("C", "any"), ("any", "C"), ("P", "helix"), ("G", "tight_turn")}

    if mut_aa == "C" or wt_aa == "C":
        reasoning.append("REJECT: Cysteine mutation risks disulfide disruption.")
        return False, reasoning

    if rsa < 0.2 and mut_aa in "DEKRH":
        reasoning.append(f"REJECT: Charged residue {mut_aa} in buried position (RSA={rsa:.2f}).")
        return False, reasoning

    if abs(net_charge_delta) > 2:
        reasoning.append(f"REJECT: Net charge shift of {net_charge_delta} risks solubility issues.")
        return False, reasoning

    reasoning.append("PASS: No expression or solubility red flags.")
    return True, reasoning

# Iterative accept/replace loop
selected = []
audit_log = []
for candidate in sorted_pool:
    if len(selected) >= budget_N:
        break
    pass_stat, r1 = statistical_audit(candidate, ...)
    if not pass_stat:
        audit_log.append((candidate, "stat", r1)); continue
    pass_struct, r2 = structural_review(candidate, ...)
    if not pass_struct:
        audit_log.append((candidate, "struct", r2)); continue
    pass_exp, r3 = experimental_review(candidate, ...)
    if not pass_exp:
        audit_log.append((candidate, "exp", r3)); continue
    selected.append(candidate)
    audit_log.append((candidate, "ACCEPTED", r1 + r2 + r3))
```

**Example 3: Adapting the Pattern to a Non-Protein Domain**

User: "I want to use this rank-then-reason approach to select the best ML model configurations from a hyperparameter sweep."

Approach:
1. **Rank-Stage analog**: Ensemble multiple evaluation metrics (accuracy, F1, latency, memory) with context-aware weighting. If deployment target is edge device, upweight latency/memory metrics. If the test set is small, attenuate accuracy weight (low statistical confidence, analogous to low pLDDT).
2. **Reason-Stage analog**: Define an expert panel:
   - *Statistical Auditor*: Check for metric inconsistencies (high accuracy but low F1 suggests class imbalance issue).
   - *Systems Expert*: Verify the config is deployable (memory fits target, latency within SLA).
   - *Robustness Expert*: Check cross-validation variance — reject configs with high variance even if mean is good.
3. Iterate top candidates, accept only those passing all three checks.

Output:
```
Rank-Stage: weighted_score = 0.3*accuracy + 0.1*f1 + 0.35*latency_inv + 0.25*memory_inv
  (weights shifted toward latency for edge deployment objective)

Reason-Stage Audit:
  Config A (score=0.91): ACCEPTED — metrics consistent, 45ms latency within 50ms SLA, CV std=0.02
  Config B (score=0.89): REJECTED by Robustness Expert — CV std=0.15 indicates instability
  Config C (score=0.87): REJECTED by Systems Expert — 2.1GB exceeds 2GB device memory
  Config D (score=0.85): ACCEPTED — all checks pass
```

## Best Practices

- **Do:** Normalize scores to z-scores before ensembling. Raw PLM scores are on incomparable scales — ESM2 outputs log-likelihoods while GEMME outputs evolutionary couplings. Z-score normalization makes weighted combination meaningful.
- **Do:** Build the candidate pool from BOTH the ensemble top-N and each individual model's top-N. This high-recall pooling is critical — the paper shows it recovers beneficial variants that ensemble averaging buries.
- **Do:** Include chain-of-thought reasoning in every accept/reject decision. The audit trail is not just for logging — it forces the reasoning agent to systematically check each constraint rather than making snap judgments.
- **Do:** Calibrate weights per-residue, not globally. A structure model may be trustworthy for the core but unreliable for a disordered loop — pLDDT-based attenuation handles this.
- **Avoid:** Using a single global weighting scheme regardless of data quality. The paper's key insight is that weight calibration is *conditioned on context* (structure confidence, MSA depth, engineering objective).
- **Avoid:** Skipping the Reason-Stage when the experimental budget is small. The smaller the budget N, the more precision matters and the larger the gain from expert auditing. At N=5, the paper shows up to 367% improvement.

## Error Handling

- **Missing structure data**: If no 3D structure is available, set all structure-model weights to zero and rely on sequence + MSA modalities. Flag this in the audit trail.
- **Shallow or missing MSA**: If Neff < 10, set MSA-model weights to near-zero. If no MSA exists, use only sequence and structure models.
- **Candidate pool exhaustion**: If the pool is exhausted before filling the budget N, relax the strictest constraint (typically the Experimental Expert's exclusion set) and re-run with a logged warning.
- **All models agree on a bad candidate**: If a candidate ranks top-10 in ALL models but fails structural review, investigate whether the structure data is incorrect rather than reflexively rejecting — model consensus is a strong signal.
- **Conflicting expert verdicts with poor justification**: If chain-of-thought reasoning is shallow or contradictory, increase the reasoning model's temperature or switch to a stronger reasoning model (the paper found that chain-of-thought capability is a prerequisite, not a luxury).

## Limitations

- The framework requires access to multiple PLM scoring tools (ESM2, SaProt, GEMME, etc.). If only one model is available, the Rank-Stage degenerates to a single-model scorer and the calibration logic provides no benefit.
- Reason-Stage quality depends heavily on the LLM's chain-of-thought capability. The paper shows standard chat models underperform reasoning-augmented models significantly — use models with strong reasoning (e.g., o1-class or DeepSeek-Reasoner).
- Structural constraints assume a reliable 3D structure. For intrinsically disordered proteins or regions with pLDDT < 30 everywhere, the Structural Biologist agent's checks become unreliable.
- The biophysical exclusion rules (cysteine, proline, glycine) are heuristics, not universal laws. Some beneficial mutations violate these rules. The framework trades recall for precision — acceptable when experimental budgets are small, but overly conservative for large-scale screens.
- Wet-lab validation showed 46.7% positive rate — impressive but not infallible. The framework maximizes *expected* fitness under budget constraints, not guaranteed outcomes.

## Reference

[Rank-and-Reason: Multi-Agent Collaboration Accelerates Zero-Shot Protein Mutation Prediction](https://arxiv.org/abs/2602.00197v2) — Tan et al., 2026. Focus on Sections 3-4 for the Rank-Stage ensemble calibration formulas and Reason-Stage expert panel protocol, and Appendices C-D for the actual agent system prompts.