---
name: "decouple-searching-training-scaling"
description: "Optimize LLM pre-training data mixtures using DeMix — a model-merging approach that decouples mixture search from training cost. Train domain-specific component models once, then evaluate unlimited data ratio combinations via weighted parameter averaging instead of retraining. Triggers: 'optimize data mixture', 'find best pre-training ratio', 'DeMix data mixing', 'model merging for data selection', 'scale data mixture search', 'pre-training data ratio optimization'"
---

# DeMix: Decouple Searching from Training for Optimal Data Mixtures

This skill enables Claude to guide users through the DeMix framework for discovering optimal pre-training data mixtures without prohibitive search costs. Instead of training a new proxy model for every candidate mixture ratio, DeMix trains one component model per data domain, then constructs proxy models via weighted linear merging of their parameters. This decouples the search budget from the training budget — evaluating 100 candidate mixtures costs only 100 cheap merge-and-evaluate cycles, not 100 full training runs. The technique applies to any scenario where you need to balance multiple data sources (general text, math, code, multilingual) in LLM pre-training.

## When to Use

- When a user needs to determine the optimal ratio of general, math, code, or other domain data for LLM pre-training
- When building a data mixing pipeline and wants to evaluate many candidate ratios cheaply
- When the user asks how to balance domain-specific competence (math, code) against general language ability
- When scaling up data mixture search beyond what RegMix or CLIMB can afford at proxy scale
- When the user wants to reproduce or extend DeMix results using the open-source DeMix Corpora (22T tokens)
- When designing a component-model training strategy for model merging experiments
- When the user asks about using model merging as a proxy for training-from-scratch on mixed data

## Key Technique

**The core insight:** When multiple models are initialized from the same base checkpoint and fine-tuned on different domains, their parameter deltas remain small (roughly 10% of parameter magnitude). Under this condition, linearly averaging the parameters with weights `[alpha_1, ..., alpha_N]` closely approximates the model you would get by training from scratch on a mixture with those same proportions. This linear approximation holds because the weight deltas from separate domains are nearly orthogonal in high-dimensional parameter space.

**The DeMix pipeline has four phases.** First, train a shared base model on general-purpose data. Second, train N component models — each initialized from the base model and trained on a 50/50 mix of general data and one domain-specific dataset (math, code, etc.). Third, for each candidate mixture ratio `alpha`, construct a proxy model by computing `M_proxy = sum(alpha_i * Theta_i)` where `Theta_i` are the component model parameters. Evaluate this proxy on a benchmark suite (cost: ~0.3 GPU-hours on an H800, essentially free compared to training). Fourth, use a LightGBM regression model to predict which ratios will score highest, iteratively sampling and refining across 3 rounds (64, 32, 16 candidates) before selecting the final mixture as the average of the top candidates.

**Why this beats alternatives:** RegMix and CLIMB require training a separate small proxy model (2B-8B tokens) for every candidate ratio. Evaluating 96 ratios at 8B tokens each costs 768B-1344B tokens of compute. DeMix trains 7 component models at 30B tokens each (210B total) then evaluates 112+ proxies via merging at negligible cost — achieving equal or better rank correlation (Spearman rho = 0.81) at 3-6x lower total cost.

## Step-by-Step Workflow

1. **Identify candidate data domains.** Enumerate the distinct data sources you want to mix (e.g., general web text, mathematics, Python code, multilingual text). Each domain becomes one component. DeMix tested with 6 candidate domains plus a general base.

2. **Train the shared base model.** Train a single model from scratch on your general-purpose corpus. The paper uses 50B tokens on general data with Qwen3-1.7B architecture (batch size 512, sequence length 8192, learning rate 3e-4, cosine schedule with 20% minimum).

3. **Train component models.** For each of the N domains, initialize from the base model checkpoint and continue training on a 50/50 mix of general data and the domain-specific data. Train each component for 30B tokens. All component models share identical hyperparameters — only the data differs.

4. **Sample candidate mixture ratios.** Generate candidate weight vectors `[alpha_1, ..., alpha_N]` uniformly from the probability simplex (all weights non-negative, summing to 1). Start with 64 candidates in the first iteration. Use `iterative_sample/sample.py` from the DeMix repo.

5. **Merge and evaluate proxies.** For each candidate ratio, compute the weighted average of component model parameters: `M_proxy = sum(alpha_i * Theta_i)`. Save the merged checkpoint and evaluate on your benchmark suite (ARC-E, HellaSwag, PIQA, WinoGrande, SIQA for general; HumanEval, MBPP for code; GSM8K, MATH for math). Use `model_merge/merge_models.sh` and OpenCompass for evaluation.

6. **Compute ranking scores.** For each proxy, compute a macro-averaged rank across all benchmarks. This single score captures multi-domain performance in one number, avoiding the trap of optimizing for only one domain.

7. **Train the predictor model.** Fit a LightGBM regressor (learning rate 0.02, 300 iterations) mapping mixture weights to ranking scores. Use `iterative_sample/train_predictor.py`.

8. **Iterate the search.** Sample a new batch of candidate ratios (32 in round 2, 16 in round 3). Use the predictor to pre-screen candidates, keeping only the most promising ones for actual merge-and-evaluate. Retrain the predictor after each round with the expanded dataset.

9. **Select the final mixture.** Average the top-performing mixture ratios (paper uses top 128 across all rounds) to obtain the final data proportions. This averaging reduces noise from individual evaluations.

10. **Train the final model.** Use the discovered mixture ratios to combine your data sources and train the production model at full scale. Validate on held-out benchmarks to confirm the proxy predictions transfer.

## Concrete Examples

**Example 1: Setting up a DeMix pipeline for a 1.7B parameter model**

User: "I have general web data, math data (from OpenWebMath), and Python code data. I want to find the best mixture for pre-training a 1.7B model. How do I set up DeMix?"

Approach:
1. Organize data into three directories: `data/general/`, `data/math/`, `data/code/`
2. Train base model on `data/general/` for 50B tokens
3. Train 3 component models, each initialized from the base:
   - `component_general`: 50% general + 50% general (control)
   - `component_math`: 50% general + 50% math
   - `component_code`: 50% general + 50% code
4. Sample 64 weight triples `(alpha_general, alpha_math, alpha_code)` from the 2-simplex
5. For each triple, merge component checkpoints and evaluate

Output — merge configuration YAML for one candidate:
```yaml
# merge_config_sample_001.yaml
merge_method: linear
models:
  - model: checkpoints/component_general
    parameters:
      weight: 0.60
  - model: checkpoints/component_math
    parameters:
      weight: 0.25
  - model: checkpoints/component_code
    parameters:
      weight: 0.15
dtype: bfloat16
```

Run: `mergekit-yaml merge_config_sample_001.yaml merged_models/sample_001/`

Then evaluate: `opencompass --models merged_models/sample_001/ --datasets arc_e hellaswag piqa humaneval mbpp gsm8k math_bench`

**Example 2: Iterative refinement with the LightGBM predictor**

User: "I've evaluated 64 merged proxies. How do I use those results to find better mixtures?"

Approach:
1. Collect results into a CSV with columns: `alpha_general, alpha_math, alpha_code, rank_score`
2. Train LightGBM predictor on the 64 data points
3. Sample 10,000 new candidate ratios from the simplex
4. Use predictor to score all 10,000; select top 32 for actual evaluation
5. Merge and evaluate those 32; add results to training set
6. Repeat once more with 16 candidates

```python
import numpy as np
import lightgbm as lgb

# Load round 1 results
data = np.loadtxt("round1_results.csv", delimiter=",", skiprows=1)
X = data[:, :3]  # alpha weights
y = data[:, 3]   # rank scores (lower is better)

# Train predictor
train_data = lgb.Dataset(X, label=y)
params = {"learning_rate": 0.02, "num_iterations": 300, "objective": "regression"}
predictor = lgb.train(params, train_data)

# Sample new candidates from simplex
def sample_simplex(n, dim):
    x = np.random.exponential(1, (n, dim))
    return x / x.sum(axis=1, keepdims=True)

candidates = sample_simplex(10000, 3)
scores = predictor.predict(candidates)

# Select top 32 candidates for actual evaluation
top_indices = np.argsort(scores)[:32]
round2_candidates = candidates[top_indices]
np.savetxt("round2_candidates.csv", round2_candidates, delimiter=",",
           header="alpha_general,alpha_math,alpha_code", comments="")
```

**Example 3: Validating proxy quality with rank correlation**

User: "How do I know if the merged proxies actually predict real training outcomes?"

Approach:
1. Train 20-30 reference models on actual data mixtures (expensive but one-time validation)
2. For the same mixture ratios, construct merged proxies
3. Compute Spearman rank correlation between proxy benchmark scores and reference model scores
4. A correlation of rho >= 0.75 indicates the proxy is reliable

```python
from scipy.stats import spearmanr

# proxy_scores: benchmark scores from merged models
# reference_scores: benchmark scores from actually-trained models
# Both arrays are ordered by the same mixture ratios

rho, p_value = spearmanr(proxy_scores, reference_scores)
print(f"Spearman rho: {rho:.3f} (p={p_value:.4f})")

# Also compute top-quartile correlation (most important region)
top_k = len(reference_scores) // 4
top_indices = np.argsort(reference_scores)[-top_k:]
rho_top, _ = spearmanr(proxy_scores[top_indices], reference_scores[top_indices])
print(f"Top-25% rho: {rho_top:.3f}")

# Capability recovery rate
recovery = np.mean(proxy_scores) / np.mean(reference_scores)
print(f"Capability recovery: {recovery:.3f}")
# Target: rho >= 0.75, recovery >= 0.80
```

## Best Practices

- **Do:** Keep the general-data mixing ratio (beta) at 50% when training component models. The paper ablation shows rho drops from 0.787 to 0.667 when reducing general data to 25%. The general data acts as a regularizer that keeps component models in the same region of parameter space.

- **Do:** Use macro-averaged rank across all benchmarks as your optimization target, not any single benchmark score. Optimizing for one domain (e.g., math) will degrade others.

- **Do:** Average the top-K mixture ratios rather than picking the single best one. This ensemble of ratios is more robust than any individual candidate.

- **Do:** Train component models at sufficient scale (30B+ tokens for a 1.7B model). Undertrained components produce unreliable merge proxies.

- **Avoid:** Using more than ~224 proxy evaluations per search. The paper shows diminishing returns and possible overfitting of the LightGBM predictor beyond this point.

- **Avoid:** Merging models trained from different random initializations. All component models must share the same base checkpoint — the linear approximation requires a shared initialization point to keep parameter deltas small and composable.

## Error Handling

- **Low rank correlation (rho < 0.6):** Component models may be undertrained or the general-data ratio beta is too low. Increase training tokens or raise beta toward 50%. Also verify all components were initialized from the same base checkpoint.

- **Merged model produces degenerate outputs:** Check that merge weights sum to 1.0 and that no weight is negative. Verify dtype consistency (all models must use the same precision, e.g., bfloat16).

- **LightGBM predictor fails to generalize:** Ensure the initial random sample covers the simplex uniformly. If your domain count N is large (>8), increase the first-round sample size proportionally — 64 samples may be too sparse for a high-dimensional simplex.

- **Benchmark scores of proxy far below reference (recovery < 0.7):** The component models may have diverged too far from the base. Reduce the component training duration or increase the general-data proportion during component training.

- **Search finds a "good" ratio but final model underperforms:** The proxy approximation has limits. Validate the top 3-5 discovered ratios by actually training small models on those mixtures before committing to a full-scale run.

## Limitations

- The linear merging approximation holds when parameter deltas are small (~10%). For very long component training runs or very different domains, the approximation degrades and proxy fidelity drops.
- DeMix has been validated at 1.7B parameter scale with 50B training tokens. Transfer to 7B+ or 70B+ models is plausible but unverified by the paper.
- The approach requires training N+1 models (1 base + N components) upfront. If N is large (>10 domains), the one-time training cost may still be significant.
- Benchmark-based evaluation favors domains with available benchmarks. Domains without good evaluation suites (e.g., creative writing, niche languages) cannot be reliably optimized.
- The DeMix Corpora is primarily English and Chinese. Applying the discovered ratios to other language distributions requires rerunning the search.

## Reference

**Paper:** [Decouple Searching from Training: Scaling Data Mixing via Model Merging for Large Language Model Pre-training](https://arxiv.org/abs/2602.00747v1) — Li et al., 2026. Focus on Section 3 (method), Table 2 (proxy consistency), Table 3 (mixture quality), and the ablation studies in Section 4.3 for practical parameter choices.

**Code & Data:** [github.com/Lucius-lsr/DeMix](https://github.com/Lucius-lsr/DeMix) — Contains sampling scripts, merge configuration generators, and evaluation tools. The DeMix Corpora (22T tokens) is hosted on Hugging Face at `lucius1022/DeMix_Corpora`.