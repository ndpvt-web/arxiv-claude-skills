---
name: "efficient-estimation-kernel-surrogate"
description: "Build kernel surrogate models to attribute how individual training tasks influence a target task's performance, capturing nonlinear interactions (synergy, antagonism) that linear methods miss. Use when the user says: 'Which training tasks help or hurt this target?', 'Run task attribution on my multi-task model', 'Build a kernel surrogate for task selection', 'Estimate task interactions without retraining', 'Select the best demonstration subset for ICL', 'Quantify task synergy and antagonism'."
---

# Kernel Surrogate Models for Task Attribution

This skill enables Claude to implement kernel-based surrogate models that quantify how each training task in a multi-task system influences performance on a target task. Rather than expensive leave-one-out retraining, the method builds a kernel ridge regression surrogate over binary task-subset vectors, estimated via a gradient-based procedure that requires only a single pretrained checkpoint. It captures second-order task interactions -- synergy, antagonism, XOR effects -- that linear surrogates and influence functions miss, achieving 25% higher correlation with leave-one-out ground truth and 40% better downstream task selection.

## When to Use

- When the user has a multi-task learning pipeline and wants to know which training tasks improve or degrade performance on a specific target task.
- When selecting the best subset of demonstration examples for in-context learning (ICL) with large language models.
- When the user asks to quantify synergy or antagonism between training tasks without retraining the model repeatedly.
- When building a task selection policy for multi-objective reinforcement learning where some objectives conflict.
- When the user needs a data valuation or task importance ranking that goes beyond first-order (linear) approximations.
- When influence function estimates are inaccurate due to strong nonlinear interactions in the training mixture.

## Key Technique

**The core problem:** Given K training tasks and a target task, estimate the function F(s) that maps any binary subset vector s in {0,1}^K to the model's performance on the target when trained on that subset. Direct evaluation requires 2^K retraining runs, which is infeasible.

**Linear surrogates** fit F(s) ~ alpha + beta^T s via least squares on a sample of (subset, performance) pairs. The coefficient beta_k gives task k's attribution score. However, this misses interactions: two tasks might individually hurt performance but jointly help (synergy), or individually help but jointly hurt (antagonism). The paper proves that linear surrogates converge to influence function scores when the loss Hessian is small, explaining why both methods share the same blind spot.

**Kernel surrogates** replace the linear model with kernel ridge regression: g(s) = sum_i theta_i * k(s^(i), s), using an RBF kernel k(s^a, s^b) = exp(-gamma * ||s^a - s^b||^2). This captures all pairwise and higher-order task interactions implicitly. The key computational trick is a gradient-based estimation procedure (GradEx): instead of retraining for each subset, take a first-order Taylor expansion of the model around the pretrained weights W0, then solve a lightweight optimization in compressed gradient space using random projections. This achieves <2% relative error versus actual retraining, at a fraction of the cost.

## Step-by-Step Workflow

1. **Define the task structure.** Enumerate all K training tasks T_1...T_K and the target task T_test. Each task is a dataset of (input, label) pairs. Represent any subset as a binary vector s in {0,1}^K.

2. **Compute base gradients.** Run a forward and backward pass on each task T_k using the pretrained model weights W0. Store per-task gradient vectors nabla_W L_k(W0). Apply Gaussian random projection P (dimension k << d, chosen via Johnson-Lindenstrauss bounds) to compress each gradient to a manageable size.

3. **Sample task subsets.** Draw m subset vectors s^(1)...s^(m) from a Bernoulli distribution with parameter p (typically p = 0.5 for uniform coverage; adjust based on K). Each s^(i) is a random binary mask over the K tasks.

4. **Estimate performance per subset via GradEx.** For each subset s^(i), solve the first-order approximate optimization: minimize the weighted loss over selected tasks in the compressed gradient space. Compute the predicted test performance F_hat(s^(i)) = loss(f_W0(x) + <nabla f_W0(x), Z*>, y) on the target task.

5. **Construct the kernel matrix.** Build the m x m matrix K where K_ij = exp(-gamma * ||s^(i) - s^(j)||^2). Select gamma and regularization lambda via cross-validation on the m surrogate samples.

6. **Solve kernel ridge regression.** Compute theta = (K + lambda * I_m)^{-1} * y, where y is the vector of estimated performances F_hat(s^(i)). The surrogate model is now g(s) = sum_i theta_i * k(s^(i), s).

7. **Extract attribution scores.** For each task k, compute the leave-one-out attribution: I_k = g([1]_K) - g([1]_K - e_k), where e_k is the k-th standard basis vector. Rank tasks by |I_k| for importance, sign for direction (positive = helpful, negative = harmful).

8. **Detect pairwise interactions.** Compute interaction scores: I_{j,k} = g(e_j + e_k) - g(e_j) - g(e_k) + g(0). Positive values indicate synergy; negative values indicate antagonism.

9. **Apply to downstream selection.** Use the surrogate to solve the subset selection problem: find s* = argmax_s g(s) subject to constraints (e.g., budget on number of tasks). For small K, enumerate; for large K, use greedy forward selection guided by the surrogate.

10. **Validate with spot checks.** Retrain on 2-3 top-ranked and bottom-ranked subsets to confirm the surrogate predictions track actual performance within the expected <2% error margin.

## Concrete Examples

**Example 1: Selecting ICL demonstrations for a classification task**

User: "I have 20 candidate demonstration examples for in-context learning on SST-2 sentiment classification. Which subset of demos gives the best accuracy?"

Approach:
1. Define K=20 tasks, each being a single demonstration example. Target task is SST-2 test accuracy.
2. Load the pretrained LLM and compute per-example gradients on the few-shot prompt.
3. Sample m=200 random subsets (binary masks over the 20 demos).
4. For each subset, use GradEx to estimate test loss without retraining.
5. Fit an RBF kernel surrogate over the 200 (subset, loss) pairs.
6. Use greedy forward selection on the surrogate to find the best 5-demo subset.

Output:
```python
import numpy as np
from sklearn.kernel_ridge import KernelRidge
from sklearn.model_selection import GridSearchCV

# Step 3: Sample subsets
K, m = 20, 200
subsets = np.random.binomial(1, 0.5, size=(m, K))

# Step 4: Estimate performance per subset (GradEx outputs)
# performances[i] = estimated test loss for subset subsets[i]
performances = gradex_estimate(model, subsets, target_data)

# Step 5-6: Fit kernel surrogate
param_grid = {"alpha": [0.01, 0.1, 1.0], "gamma": [0.01, 0.1, 1.0]}
krr = GridSearchCV(
    KernelRidge(kernel="rbf"), param_grid, cv=5, scoring="neg_mean_squared_error"
)
krr.fit(subsets, performances)

# Step 7: Attribution scores
full = np.ones(K)
attributions = np.array([
    krr.predict(full.reshape(1, -1))[0]
    - krr.predict((full - np.eye(K)[k]).reshape(1, -1))[0]
    for k in range(K)
])

# Step 9: Greedy forward selection for best 5
selected = []
for _ in range(5):
    best_k, best_val = -1, float("inf")
    for k in range(K):
        if k in selected:
            continue
        candidate = np.zeros(K)
        candidate[selected + [k]] = 1
        val = krr.predict(candidate.reshape(1, -1))[0]
        if val < best_val:
            best_k, best_val = k, val
    selected.append(best_k)

print(f"Selected demos: {selected}")
print(f"Predicted test loss: {best_val:.4f}")
# Attributions ranked:
for k in np.argsort(attributions):
    print(f"  Demo {k}: attribution = {attributions[k]:+.4f}")
```

**Example 2: Diagnosing task conflicts in multi-task training**

User: "My model trains on translation, summarization, QA, and code generation simultaneously. Performance on QA dropped after adding code generation. Can you quantify the interaction?"

Approach:
1. Define K=4 tasks. Target = QA test loss.
2. Compute compressed per-task gradients at the current checkpoint.
3. Sample m=100 subsets (enough for K=4 since 2^4=16 is small; oversample for robustness).
4. Run GradEx for each subset to estimate QA loss.
5. Fit kernel surrogate and extract pairwise interaction matrix.

Output:
```
Task Attribution (LOO scores, lower = more helpful):
  Translation:    -0.032 (helpful)
  Summarization:  -0.018 (helpful)
  QA:             -0.041 (helpful, self-contribution)
  Code Gen:       +0.027 (harmful)

Pairwise Interactions (negative = synergy, positive = antagonism):
  Translation x Summarization:  -0.012 (synergistic)
  Translation x Code Gen:       +0.005 (mildly antagonistic)
  Summarization x Code Gen:     +0.003 (mildly antagonistic)
  QA x Code Gen:                +0.045 (strongly antagonistic)

Recommendation: Code generation antagonizes QA (interaction = +0.045).
Consider training with reduced code gen weight or staged training.
```

**Example 3: Task importance ranking for curriculum learning**

User: "I have 50 math reasoning subtasks for training a transformer. Rank them by importance for the target task (algebra word problems)."

Approach:
1. K=50 tasks, target = algebra word problem accuracy.
2. Compute per-task gradients, project to k=512 dimensions via random Gaussian matrix.
3. Sample m=500 subsets with Bernoulli(p=0.5).
4. GradEx estimation for all 500 subsets (parallelizable across GPUs).
5. Fit kernel surrogate with cross-validated hyperparameters.
6. Extract attributions and rank.

Output:
```
Top 5 most helpful tasks for algebra word problems:
  1. Linear equations      (attribution: -0.089)
  2. Fraction arithmetic   (attribution: -0.071)
  3. Variable substitution (attribution: -0.063)
  4. Word-to-equation      (attribution: -0.058)
  5. Order of operations   (attribution: -0.044)

Bottom 3 (harmful or irrelevant):
  48. Geometry proofs       (attribution: +0.031)
  49. Combinatorics         (attribution: +0.022)
  50. Number theory trivia  (attribution: +0.018)

Curriculum recommendation: prioritize top 15 tasks in early training.
```

## Best Practices

- **Do:** Use random Gaussian projections to compress gradients before constructing the kernel. Full-dimensional gradients are computationally prohibitive; projections to k=256-1024 dimensions preserve pairwise distances via JL guarantees.
- **Do:** Cross-validate both the kernel bandwidth gamma and the regularization lambda. Small gamma makes the kernel too smooth (underfitting interactions); large gamma makes it too local (overfitting to sampled subsets).
- **Do:** Sample more subsets (m) than the number of tasks (K). A rule of thumb is m >= 5K for linear surrogates and m >= 10K for kernel surrogates to reliably capture second-order effects.
- **Do:** Compare kernel surrogate attributions against the simpler linear surrogate as a sanity check. When both agree, interactions are weak and either suffices. When they diverge, the kernel model is capturing important nonlinearities.
- **Avoid:** Applying kernel surrogates when K > 200 tasks without dimensionality reduction on the subset vectors. The kernel matrix is m x m, but subset distances become less informative in very high dimensions.
- **Avoid:** Using GradEx approximation on models that are far from converged. The first-order Taylor expansion assumes proximity to a local minimum; on poorly trained checkpoints, the <2% error guarantee does not hold.

## Error Handling

- **GradEx divergence:** If the first-order optimization diverges for some subsets, reduce the learning rate or increase regularization. Discard subsets with estimated performance values that are extreme outliers (>3 standard deviations from the mean).
- **Ill-conditioned kernel matrix:** If K + lambda*I is near-singular, increase lambda. This happens when many sampled subsets are nearly identical (common with high p values for small K). Reduce p or use stratified sampling.
- **High cross-validation error:** If the kernel surrogate's cross-validated MSE is high relative to the performance variance, the m samples may be insufficient. Increase m, or fall back to a linear surrogate if the task interactions are genuinely weak.
- **Memory limits on gradient storage:** For very large models, the per-task gradients may not fit in memory even after projection. Use gradient checkpointing, compute projections in streaming fashion, or restrict to final-layer gradients (trading accuracy for feasibility).

## Limitations

- The method requires access to model gradients. It does not apply to black-box APIs where only predictions are available.
- For very large K (hundreds of tasks), the quadratic cost of the kernel matrix and the difficulty of subset sampling in high-dimensional binary spaces reduce effectiveness. Linear surrogates may be more practical in this regime.
- GradEx assumes the first-order Taylor expansion is accurate, which requires the model to be near a local optimum. For early-stage or poorly converged models, retraining-based estimation is more reliable.
- The RBF kernel captures smooth interactions well but may miss discrete or threshold-type effects (e.g., a task only helps when 3+ other specific tasks are present). Consider polynomial or custom kernels for such cases.
- Validation still requires some actual retraining on selected subsets to confirm surrogate predictions, though far fewer runs than full leave-one-out.

## Reference

**Paper:** [Efficient Estimation of Kernel Surrogate Models for Task Attribution](https://arxiv.org/abs/2602.03783v1) -- Zhang, Duan, and Zhang (ICLR 2026). Focus on Algorithm 1 (KernelSM estimation), Proposition 3.1 (linear-influence connection), and Section 5 experiments for implementation guidance.

**Code:** [github.com/VirtuosoResearch/Kernel-surrogate-models](https://github.com/VirtuosoResearch/Kernel-surrogate-models)