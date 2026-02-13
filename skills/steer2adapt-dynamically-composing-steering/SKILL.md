---
name: "steer2adapt-dynamically-composing-steering"
description: "Implement the Steer2Adapt framework for adapting LLMs at inference time by dynamically composing steering vectors from a reusable semantic subspace. Use when: 'compose steering vectors for a new task', 'adapt model behavior without fine-tuning', 'build a steering vector library', 'dynamically steer LLM reasoning or safety', 'few-shot behavior adaptation with activation steering', 'combine concept directions for multi-objective LLM control'."
---

# Steer2Adapt: Dynamically Composing Steering Vectors for Inference-Time LLM Adaptation

This skill enables Claude to help users implement the Steer2Adapt framework, which adapts LLMs to new tasks by composing a linear combination of pre-extracted steering vectors rather than training new ones. The core insight is that tasks within a domain (reasoning, safety, etc.) share a small set of underlying behavioral dimensions. By extracting these dimensions as basis vectors into a frozen semantic subspace, new task behaviors emerge through Bayesian optimization of mixture coefficients over just 12 calibration examples -- no gradients, no fine-tuning, pure inference-time intervention.

## When to Use

- When a user wants to steer an LLM's behavior (reasoning style, safety posture, personality) without fine-tuning or prompt engineering
- When adapting a model to a new task that shares behavioral dimensions with previously characterized tasks (e.g., a new reasoning benchmark using existing logic/arithmetic/code concept vectors)
- When building a reusable library of steering vectors that can be mixed-and-matched across tasks
- When the user needs multi-objective LLM control (e.g., simultaneously improving honesty while reducing sycophancy) via composed directions
- When implementing few-shot adaptation where only 6-12 labeled examples are available for calibration
- When the user asks about activation steering, representation engineering, or inference-time model adaptation techniques

## Key Technique

**Semantic Prior Subspace.** Steer2Adapt begins by extracting a set of k steering vectors (typically k=5) using contrastive pairs -- e.g., "honest vs. dishonest" prompts run through the model, with the mean activation difference at target layers yielding a direction vector. These k vectors form a frozen concept dictionary V = [v1, ..., vk] in R^(d x k), where d is the model's hidden dimension. This subspace captures the behavioral dimensions of a domain. For reasoning, the paper uses Big Five personality traits (Openness, Conscientiousness, Extraversion, Agreeableness, Neuroticism). For safety, it uses Fairness, Sycophancy, Refusal, Hallucination, and Lawfulness directions.

**Dynamic Composition via Bayesian Optimization.** To adapt to a new task, the method searches for a coefficient vector alpha in R^k such that the composed steering vector V*alpha, when injected into model activations at layers {8, 10, 12, 14, 16, 18, 20, 22, 24}, maximizes task performance on a small calibration set. The search uses Bayesian optimization with a stability-aware objective: J(alpha) rewards probability gains on initially-incorrect examples while penalizing regressions on initially-correct ones. A penalty hierarchy (lambda_flip > lambda_drop >> Gain) enforces risk-averse optimization -- flipping a correct prediction to incorrect is penalized far more heavily than a confidence drop.

**Why It Works.** Because the search space is k-dimensional (k=5) rather than d-dimensional (d=4096+), Bayesian optimization converges reliably with only 12 calibration examples. The composed vector captures task-specific behavior as a weighted blend of interpretable concept dimensions, making the adaptation transparent and debuggable.

## Step-by-Step Workflow

1. **Select the domain and identify concept dimensions.** Determine whether the target task falls under reasoning, safety, or another domain. List 3-7 behavioral dimensions that plausibly span the domain (e.g., for reasoning: analytical depth, systematic thinking, creativity, precision, persistence).

2. **Extract contrastive steering vectors.** For each concept dimension, create 50-100 contrastive prompt pairs (e.g., "Think step by step carefully" vs. "Answer immediately without thinking"). Run each pair through the target model, collect hidden-state activations at target layers, and compute the mean difference vector: `v_i = mean(h_positive) - mean(h_negative)`.

3. **Construct the frozen semantic subspace.** Stack all k steering vectors into a matrix V = [v1, ..., vk]. Optionally normalize each vector to unit length. This concept dictionary is frozen and reused across all tasks in the domain.

4. **Prepare a calibration set.** Collect 12 labeled examples for the new task -- 6 that the base model answers correctly and 6 it answers incorrectly. This balanced split is critical for the stability-aware objective.

5. **Define the stability-aware objective function.** Implement J(alpha) = sum over incorrect examples of probability gain minus lambda_flip * flip_penalty minus lambda_drop * confidence_drop_penalty. Set lambda_flip > lambda_drop >> 1 to enforce the risk-averse hierarchy.

6. **Run Bayesian optimization over alpha.** Use a Gaussian Process surrogate with Expected Improvement acquisition. Search alpha in R^k (typically k=5), evaluating J(alpha) by injecting V*alpha into model activations at layers {8, 10, 12, 14, 16, 18, 20, 22, 24} and scoring the calibration set.

7. **Compose and inject the final steering vector.** After optimization converges, compute v_combined = V * alpha_optimal. At inference time, add v_combined to hidden states at the target layers: `h' = h + v_combined`.

8. **Validate on held-out examples.** Test the adapted model on examples outside the calibration set. Check that initially-correct predictions remain correct (stability) and that initially-incorrect predictions improve (effectiveness).

9. **Inspect alpha coefficients for interpretability.** Examine which concept dimensions received high vs. low weight. This reveals what behavioral blend the task requires and aids debugging if performance is unexpected.

10. **Iterate or extend the subspace.** If performance is insufficient, consider adding new concept dimensions to V or adjusting the layer injection set. The subspace can grow incrementally without restarting.

## Concrete Examples

**Example 1: Adapting a Model for Improved Code Reasoning**

```
User: I want to improve my Llama-3.1-8B model's performance on MBPP-style
programming tasks using steering vectors, without fine-tuning.

Approach:
1. Define reasoning concept dimensions: Openness (creative problem-solving),
   Conscientiousness (systematic debugging), Extraversion (exploratory search),
   Agreeableness (following instructions precisely), Neuroticism (error sensitivity).

2. Extract 5 steering vectors using contrastive templates:
   - Openness: "Think creatively about multiple solutions" vs.
     "Use only the most obvious approach"
   - Conscientiousness: "Check your work carefully step by step" vs.
     "Give a quick answer without verification"
   (... repeat for all 5 dimensions)

3. Build calibration set: Select 6 MBPP problems the model currently solves
   correctly and 6 it fails on.

4. Run Bayesian optimization over alpha in R^5 with stability-aware objective.
   Typical convergence: ~50 evaluations.

5. Resulting alpha might be: [0.8, 1.2, 0.3, 0.6, -0.2], meaning high weight
   on Conscientiousness (systematic step-by-step), moderate Openness, and
   slightly negative Neuroticism (reducing over-cautious hedging).

Output:
- Composed steering vector injected at layers 8-24
- Code task accuracy improves from ~59% to ~72% (matching paper's results)
- General linguistic competence preserved (BLiMP drop < 3%)
```

**Example 2: Multi-Objective Safety Steering**

```
User: I need my model to be more truthful while maintaining helpfulness.
Can I combine multiple safety steering vectors?

Approach:
1. Extract safety concept vectors: Fairness, Sycophancy (negated for honesty),
   Refusal, Hallucination (negated for truthfulness), Lawfulness.

2. Build calibration set: 6 TruthfulQA questions the model answers correctly,
   6 it gets wrong (gives plausible-sounding but false answers).

3. Run Bayesian optimization. The objective penalizes flipping correct answers
   (lambda_flip=10) more heavily than confidence drops (lambda_drop=2).

4. Inspect resulting alpha: [-0.1, -0.9, 0.2, -1.1, 0.3]
   - Strong negative weight on Sycophancy (reduces people-pleasing)
   - Strong negative weight on Hallucination (reduces confabulation)
   - Mild Refusal and Lawfulness (maintains helpfulness)

Output:
- Truthfulness on TruthfulQA improves to ~84%
- Model still answers helpful questions without excessive refusal
- IMPORTANT: Safety dimensions are partially entangled -- improving honesty
  may slightly reduce fairness scores. Monitor all safety metrics.
```

**Example 3: Implementing the Framework in Python**

```python
# Minimal implementation skeleton for Steer2Adapt

import torch
import numpy as np
from sklearn.gaussian_process import GaussianProcessRegressor
from sklearn.gaussian_process.kernels import Matern

class Steer2Adapt:
    def __init__(self, model, target_layers=(8, 10, 12, 14, 16, 18, 20, 22, 24)):
        self.model = model
        self.target_layers = target_layers
        self.concept_dict = []  # List of steering vectors (V)

    def extract_steering_vector(self, positive_prompts, negative_prompts):
        """Extract one concept dimension via contrastive activation difference."""
        pos_acts = self._collect_activations(positive_prompts)  # shape: (n, d)
        neg_acts = self._collect_activations(negative_prompts)  # shape: (n, d)
        direction = pos_acts.mean(dim=0) - neg_acts.mean(dim=0)
        direction = direction / direction.norm()  # unit normalize
        self.concept_dict.append(direction)
        return direction

    def build_subspace(self):
        """Stack concept vectors into frozen matrix V."""
        self.V = torch.stack(self.concept_dict, dim=1)  # shape: (d, k)

    def compose_vector(self, alpha):
        """Compose steering vector: v_combined = V @ alpha."""
        alpha_t = torch.tensor(alpha, dtype=self.V.dtype)
        return self.V @ alpha_t  # shape: (d,)

    def evaluate_alpha(self, alpha, calib_correct, calib_incorrect,
                       lambda_flip=10.0, lambda_drop=2.0):
        """Stability-aware objective J(alpha)."""
        v = self.compose_vector(alpha)
        gain = sum(self._prob_gain(x, v) for x in calib_incorrect)
        flip_penalty = lambda_flip * sum(
            1 for x in calib_correct if self._flipped(x, v))
        drop_penalty = lambda_drop * sum(
            self._conf_drop(x, v) for x in calib_correct)
        return gain - flip_penalty - drop_penalty

    def optimize(self, calib_correct, calib_incorrect, n_iter=50):
        """Bayesian optimization over k-dimensional alpha space."""
        k = self.V.shape[1]
        gp = GaussianProcessRegressor(kernel=Matern(nu=2.5))
        X_observed, y_observed = [], []

        for i in range(n_iter):
            if i < 2 * k:  # initial random exploration
                alpha = np.random.randn(k) * 0.5
            else:
                alpha = self._acquisition_sample(gp, X_observed, k)

            score = self.evaluate_alpha(
                alpha, calib_correct, calib_incorrect)
            X_observed.append(alpha)
            y_observed.append(score)
            gp.fit(X_observed, y_observed)

        best_idx = np.argmax(y_observed)
        return X_observed[best_idx]
```

## Best Practices

- **Do:** Use balanced calibration sets (equal correct/incorrect examples). The stability-aware objective requires both to function properly.
- **Do:** Keep the concept dictionary small (3-7 dimensions). The Bayesian optimization efficiency comes from low-dimensional search; exceeding ~10 dimensions degrades convergence.
- **Do:** Normalize steering vectors to unit length before composing. This prevents one dominant direction from swamping others due to magnitude differences.
- **Do:** Inject steering vectors at multiple middle-to-late layers (8-24 for 32-layer models). Single-layer injection is fragile; distributed injection is more stable.
- **Avoid:** Mixing concept vectors across unrelated domains (e.g., using safety vectors for arithmetic reasoning). The paper shows this causes "substantial performance degradation" and increased variance.
- **Avoid:** Skipping the stability penalty terms. Without lambda_flip >> lambda_drop, optimization freely sacrifices correct predictions for marginal gains on incorrect ones, producing unstable results.

## Error Handling

- **Alpha diverges or produces nonsensical outputs:** Clamp alpha values to a reasonable range (e.g., [-3, 3]) during optimization. Extreme coefficients indicate the subspace doesn't span the target behavior well.
- **All calibration examples already correct:** The objective degrades to pure stability preservation with no gain signal. Add harder examples or use a different evaluation metric.
- **Performance regresses on held-out set despite calibration gains:** The 12-example calibration set may not represent the task distribution. Increase to 20-30 examples or stratify by difficulty.
- **Entangled safety trade-offs:** When improving one safety metric degrades another (e.g., fairness vs. honesty), this reflects entangled representations in the model, not a framework bug. Accept the trade-off or add explicit constraints to the objective for the metric you want to protect.

## Limitations

- **Domain-bound subspaces.** The concept dictionary must semantically relate to the target task. There is no universal subspace that works across all domains.
- **Entangled safety dimensions.** Safety-related steering vectors are not fully disentangled -- improving one objective (e.g., reducing hallucination) may degrade another (e.g., fairness). Multi-objective Pareto optimization is not yet addressed.
- **Fixed layer injection.** The paper uses a fixed set of injection layers. Optimal layers may vary by model architecture and task; the framework does not auto-select them.
- **Moderate general-competence cost.** Steering preserves ~96.6% of linguistic competence on average, but some tasks (sycophancy reduction) cause up to 4.5% BLiMP degradation.
- **Requires model internals access.** This is an activation-level intervention requiring access to hidden states. It cannot be applied to API-only models (e.g., GPT-4, Claude) -- only to open-weight models where you can hook into forward passes.
- **Calibration set sensitivity.** With only 12 examples, outliers in the calibration set can skew the optimized coefficients. Results should always be validated on held-out data.

## Reference

**Paper:** [Steer2Adapt: Dynamically Composing Steering Vectors Elicits Efficient Adaptation of LLMs](https://arxiv.org/abs/2602.07276v1) (Han et al., 2026)

**Key takeaway:** Look at Section 3 for the full algorithm (semantic subspace construction + stability-aware Bayesian optimization), Table 1-2 for quantitative results across reasoning/safety domains, and Figure 6 for the safety entanglement analysis showing trade-offs between dimensions.