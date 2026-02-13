---
name: "reward-free-alignment-conflicting-objectives"
description: "Implement multi-objective LLM alignment using RACO (Reward-free Alignment for Conflicting Objectives) — a method that resolves gradient conflicts between competing objectives via clipped conflict-averse gradient descent without requiring reward models. Use when: 'align model with multiple conflicting goals', 'multi-objective DPO training', 'balance helpfulness and safety in alignment', 'Pareto-optimal LLM fine-tuning', 'resolve gradient conflicts in RLHF', 'RACO alignment implementation'."
---

# Reward-free Alignment for Conflicting Objectives (RACO)

This skill enables Claude to help users implement and debug multi-objective LLM alignment pipelines using RACO, a framework that directly optimizes pairwise preference data across multiple conflicting objectives (e.g., helpfulness vs. safety, conciseness vs. quality) without training separate reward models. RACO resolves gradient conflicts between objectives using a clipped variant of conflict-averse gradient descent, guaranteeing convergence to Pareto-critical points that respect user-specified objective weights.

## When to Use

- When the user wants to fine-tune an LLM on two or more preference datasets that represent conflicting goals (e.g., helpfulness + harmlessness, quality + brevity)
- When weighted DPO produces unstable training or one objective dominates the other
- When the user asks how to find Pareto-optimal trade-offs between competing alignment objectives
- When implementing multi-objective alignment without training explicit reward models
- When the user has per-objective pairwise preference data (chosen/rejected pairs per goal) and wants a single training run
- When debugging gradient conflict issues during multi-objective fine-tuning
- When comparing RACO against baselines like Rewarded Soups, RiC, MODPO, or weighted DPO

## Key Technique

**The core problem:** Standard weighted DPO sums per-objective losses into a single scalar loss. When objectives conflict — their gradients point in opposing directions — the weighted sum can produce update directions that worsen one or more objectives. This happens frequently: in Reddit summarization, ~60% of examples have fully conflicting quality vs. conciseness gradients.

**RACO's solution:** Instead of naively summing gradients, RACO (1) computes each objective's gradient independently from its own DPO loss, (2) detects conflicts by solving a quadratic program that finds the minimum-norm correction to the weighted gradient such that the corrected direction reduces all objectives simultaneously, and (3) clips the correction coefficients so no objective gets upweighted beyond its user-specified importance. The clipping step is critical — without it, the conflict resolution can overcorrect and distort the user's intended trade-off by amplifying low-weight objectives.

**Why it works:** The clipped correction provably converges to points that are both critical for the weighted loss and Pareto-critical across all objectives. In the two-objective case, clipping strictly improves convergence rate compared to unclipped CAGrad. Practically, this means the user specifies weights like w=[0.7, 0.3] for "70% quality, 30% safety" and RACO finds the best achievable point respecting that preference — not an arbitrary point on the Pareto frontier.

## Step-by-Step Workflow

1. **Prepare per-objective preference datasets.** For each objective i (e.g., helpfulness, harmlessness), collect or split data into (prompt, chosen_response, rejected_response) triples where chosen/rejected are judged solely on that objective. Store each objective's data separately.

2. **Define objective weights on the simplex.** Choose weights w = [w_1, ..., w_m] with sum = 1 reflecting the desired trade-off. For example, w=[0.6, 0.4] means 60% emphasis on objective 1. These weights are inputs, not learned.

3. **Initialize the policy and reference model.** Load the base/instruct model as both the trainable policy `pi_theta` and frozen reference `pi_ref`. Use the same reference throughout training.

4. **Compute per-objective DPO gradients.** For each mini-batch, compute separate gradients g_i = nabla L_i(theta) where each L_i is the standard DPO loss for objective i:
   ```
   L_i(theta) = -E[log sigma(beta * (log pi_theta(y_i+|x)/pi_ref(y_i+|x)
                                     - log pi_theta(y_i-|x)/pi_ref(y_i-|x)))]
   ```

5. **Form the weighted gradient.** Compute g_0 = sum(w_i * g_i) — this is what standard weighted DPO would use as its update.

6. **Solve the conflict-averse correction.** Find coefficients p = [p_1, ..., p_m] minimizing the worst-case inner product max_i <g_i, d> subject to ||d - g_0|| <= c * ||g_0||, where c is the correction radius (typically c=0.4 for instruct models, c=0.7-0.8 for base models). This has a closed-form solution via KKT conditions.

7. **Apply clipping.** Set p_tilde_i = min(p_i, w_i) for each objective. This prevents any objective from receiving more gradient weight than the user specified, avoiding overcorrection.

8. **Compute the final update direction.** G_0 = sum(p_tilde_i * g_i). Update theta <- theta - eta * G_0.

9. **Iterate with standard training schedule.** Use learning rates in 1e-6 to 5e-5 range, batch sizes of 32-128 preference pairs per objective, and train for 1-3 epochs. Monitor per-objective metrics separately.

10. **Evaluate on the Pareto frontier.** Run inference and compute per-objective metrics (e.g., win-rate for quality, safety score for harmlessness). Compare against single-objective DPO and weighted DPO to verify improved trade-offs.

## Concrete Examples

**Example 1: Multi-objective summarization (quality vs. conciseness)**

User: "I have two preference datasets for Reddit TL;DR — one ranked by summary quality, another by conciseness. I want to train a Llama-3-8B model that balances both. Help me implement RACO."

Approach:
1. Load both datasets as separate DataLoaders, each yielding (prompt, chosen, rejected) triples
2. Set weights w=[0.6, 0.4] (prioritize quality slightly)
3. On each training step, sample a batch from each DataLoader
4. Compute DPO loss and gradient for each objective independently
5. Apply clipped conflict-averse gradient descent with c=0.4

Output (PyTorch-style pseudocode):
```python
import torch
from torch.nn.functional import logsigmoid

def raco_step(model, ref_model, batch_quality, batch_concise,
              weights=[0.6, 0.4], beta=0.1, c=0.4):
    # Step 4: Compute per-objective DPO losses
    grads = []
    for batch in [batch_quality, batch_concise]:
        logps = get_logprobs(model, batch)      # (chosen_lp, rejected_lp)
        ref_lps = get_logprobs(ref_model, batch)
        logits = beta * ((logps[0] - ref_lps[0]) - (logps[1] - ref_lps[1]))
        loss_i = -logsigmoid(logits).mean()
        loss_i.backward(retain_graph=True)
        g_i = collect_flat_grad(model)  # flatten all param grads
        grads.append(g_i)
        model.zero_grad()

    g = torch.stack(grads)  # shape: [m, num_params]
    w = torch.tensor(weights, device=g.device)

    # Step 5: Weighted gradient
    g0 = (w.unsqueeze(1) * g).sum(dim=0)

    # Step 6: Solve conflict-averse correction (closed-form for m=2)
    # Minimize max_i <g_i, d> s.t. ||d - g0|| <= c*||g0||
    p = solve_cag_qp(g, g0, w, c)

    # Step 7: Clip coefficients
    p_clipped = torch.min(p, w)

    # Step 8: Final update direction
    G0 = (p_clipped.unsqueeze(1) * g).sum(dim=0)
    apply_flat_grad(model, G0)  # set .grad from flat vector
    optimizer.step()

def solve_cag_qp(g, g0, w, c):
    """Closed-form for 2 objectives. For m>2, use scipy.optimize."""
    m = g.shape[0]
    # Gram matrix of gradients
    G = g @ g.T
    # Solve: min_{p in simplex} max_i (G @ p)_i
    # subject to ||sum(p_i g_i) - g0|| <= c * ||g0||
    # For m=2: analytic solution via KKT
    g0_norm = g0.norm()
    if m == 2:
        cos_sim = (g[0] @ g[1]) / (g[0].norm() * g[1].norm() + 1e-8)
        if cos_sim >= 0:  # No conflict
            return w.clone()
        # Conflict exists — interpolate toward equal improvement
        alpha = (G[1,1] - G[0,1]) / (G[0,0] - 2*G[0,1] + G[1,1] + 1e-8)
        alpha = alpha.clamp(0, 1)
        p = torch.tensor([alpha, 1-alpha], device=g.device)
        # Project to radius constraint
        d = (p.unsqueeze(1) * g).sum(0)
        if (d - g0).norm() > c * g0_norm:
            t = c * g0_norm / ((d - g0).norm() + 1e-8)
            p = w + t * (p - w)
        return p
    else:
        # General case: use QP solver
        raise NotImplementedError("Use scipy or cvxpy for m > 2")
```

**Example 2: Safety alignment (helpfulness vs. harmlessness)**

User: "I'm fine-tuning Qwen3-4B for a chatbot that must be both helpful and safe. I have BeaverTails preference data split by harmlessness labels and helpfulness labels. How do I set up RACO training?"

Approach:
1. Split BeaverTails into two preference sets: one judged on harmlessness, one on helpfulness
2. Set weights w=[0.5, 0.5] for equal emphasis, or w=[0.3, 0.7] to prioritize safety
3. Use correction radius c=0.4 for the instruct model
4. Train with RACO, monitoring both safety score and helpfulness win-rate

Output (training config):
```yaml
# raco_safety_config.yaml
model:
  name: Qwen/Qwen3-4B-Instruct
  ref_model: Qwen/Qwen3-4B-Instruct  # frozen copy

objectives:
  - name: helpfulness
    dataset: beavertails_helpful_prefs
    beta: 0.1
    weight: 0.3
  - name: harmlessness
    dataset: beavertails_safety_prefs
    beta: 0.1
    weight: 0.7

raco:
  correction_radius: 0.4     # c parameter; 0.4 for instruct, 0.7 for base
  clip_coefficients: true     # critical: enables the clipping step

training:
  learning_rate: 5e-6
  batch_size_per_objective: 64
  num_epochs: 2
  warmup_ratio: 0.1
  gradient_accumulation_steps: 4

evaluation:
  metrics:
    - harmlessness_score    # per-objective
    - helpfulness_winrate   # per-objective
  checkpoints: [500, 1000, 1500]
```

**Example 3: Debugging gradient conflicts in existing multi-objective training**

User: "My weighted DPO training keeps oscillating — quality improves then crashes. How can I diagnose and fix this with RACO?"

Approach:
1. Instrument training to log per-objective gradient cosine similarity each step
2. Identify conflict frequency (fraction of steps where cos_sim < 0)
3. Replace weighted sum with RACO's clipped conflict-averse update
4. Compare training curves

Output (diagnostic script):
```python
def diagnose_gradient_conflicts(grads_per_objective):
    """Log conflict statistics during training."""
    m = len(grads_per_objective)
    conflicts = 0
    total_pairs = 0
    for i in range(m):
        for j in range(i+1, m):
            cos = torch.cosine_similarity(
                grads_per_objective[i].unsqueeze(0),
                grads_per_objective[j].unsqueeze(0)
            ).item()
            total_pairs += 1
            if cos < 0:
                conflicts += 1
            print(f"  Obj {i} vs {j}: cos_sim = {cos:.4f}"
                  f" {'CONFLICT' if cos < 0 else 'aligned'}")
    conflict_rate = conflicts / total_pairs
    print(f"Conflict rate: {conflict_rate:.1%}")
    # If conflict_rate > 30%, RACO will significantly help
    return conflict_rate
```

## Best Practices

- **Do:** Compute gradients for each objective independently before combining them. Never sum losses first — that destroys the per-objective gradient information needed for conflict detection.
- **Do:** Start with correction radius c=0.4 for instruction-tuned models and c=0.7-0.8 for base models. Tune c if the Pareto frontier doesn't improve over weighted DPO.
- **Do:** Always enable clipping (p_tilde_i = min(p_i, w_i)). Without clipping, the method can distort user-specified trade-offs, especially at extreme weight ratios like 0.8/0.2.
- **Do:** Monitor per-objective metrics separately during training. A single aggregate metric hides whether one objective is being sacrificed.
- **Avoid:** Using RACO when objectives are naturally aligned (high positive gradient cosine similarity). In that case, weighted DPO works fine and RACO adds unnecessary overhead.
- **Avoid:** Setting all weights equal (w=[0.5, 0.5]) when you have a genuine preference. RACO's clipping is most impactful at asymmetric weights — equal weights underutilize the framework's advantage.

## Error Handling

- **Gradient explosion during conflict resolution:** If the correction QP produces very large coefficients, the radius constraint c may be too large. Reduce c (e.g., from 0.8 to 0.4) or add gradient norm clipping before the RACO step.
- **One objective not improving at all:** Check that the per-objective dataset actually contains meaningful preference signal. Compute win-rate of chosen vs. rejected under the reference model — if near 50%, the preference data may be noisy.
- **QP solver instability for m > 2:** The closed-form solution only works for m=2. For 3+ objectives, use `scipy.optimize.minimize` with the SLSQP method or `cvxpy` for the quadratic program. Add a small regularization term (1e-8 * I) to the Gram matrix.
- **Memory issues from storing per-objective gradients:** For large models, compute gradients sequentially (backward pass per objective) rather than in parallel. Use gradient checkpointing to reduce peak memory.
- **Training divergence at extreme weights (e.g., 0.95/0.05):** The minor objective's gradient signal is very weak. Consider increasing its beta parameter to amplify its preference signal, or use a less extreme weight ratio.

## Limitations

- RACO requires separate preference datasets per objective. If you only have a single preference dataset with mixed criteria, you must first decompose it — which is a non-trivial annotation task.
- The per-step overhead scales with the number of objectives m (one backward pass per objective). For m > 4, this becomes expensive.
- Convergence guarantees assume smooth loss landscapes. In practice, DPO losses can be non-smooth near the decision boundary, so theoretical rates are optimistic.
- RACO finds points on the Pareto frontier corresponding to user-specified weights — it does not map out the entire frontier. To explore the full frontier, run multiple training runs with different weight vectors.
- The method has been validated on models up to ~8B parameters. Scaling behavior to 70B+ models is not established in the paper.

## Reference

- **Paper:** [Reward-free Alignment for Conflicting Objectives](https://arxiv.org/abs/2602.02495v2) (Chen, Li, Chen, Lin, 2026)
- **Key sections to read:** Algorithm 1 for the full RACO procedure, Theorem 3.1 for convergence guarantees with clipping, Section 5 for experimental comparisons against weighted DPO/RiC/Rewarded Soups/MODPO, and Appendix B.1 for the closed-form QP solution.