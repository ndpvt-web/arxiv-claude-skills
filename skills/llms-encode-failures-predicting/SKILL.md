---
name: "llms-encode-failures-predicting"
description: "Build probe-based routing systems that predict LLM success before generation and route queries to cost-optimal models. Use when: 'predict which problems the model will fail on', 'route queries to cheaper models', 'reduce inference cost with model routing', 'build a difficulty-aware LLM router', 'train probes on model activations', 'optimize multi-model inference cost'."
---

# Predicting LLM Success from Pre-Generation Activations

This skill enables Claude to design and implement **probe-based LLM routing systems** that predict whether a model will succeed on a given input *before generation begins*, then route queries across a pool of models to minimize cost while preserving accuracy. The core insight from the paper: linear probes trained on a model's internal activations at the last token position before generation can predict that model's success rate on math and coding tasks, and this signal can drive routing decisions that cut inference cost by up to 70%.

## When to Use

- When the user wants to **reduce inference costs** by routing easy queries to smaller/cheaper models and hard queries to larger ones
- When building a **multi-model serving system** that needs to decide which model handles which request
- When the user asks to **predict model difficulty** or estimate whether an LLM will solve a problem correctly
- When designing **cascade inference pipelines** where a cheap model attempts first and a stronger model is called only when needed
- When the user wants to **benchmark model-specific difficulty** as opposed to human-perceived difficulty
- When implementing **confidence-based routing** without relying on output-based heuristics like token probabilities or chain-of-thought length

## Key Technique

**Linear probes on pre-generation activations.** Before an LLM generates any output, its residual stream at the final input token already encodes a signal about whether it will succeed. The method extracts hidden-state activations from the residual stream (before layer normalization) at the last token of the input prompt. A linear probe (ridge regression for continuous success rates, L2-regularized logistic regression for binary pass/fail) is trained on these activations to predict the model's success probability. This substantially outperforms surface features like input length or TF-IDF vectors, demonstrating that models encode a genuine internal representation of task difficulty.

**Model-specific difficulty diverges from human difficulty.** Using the E2H-AMC benchmark (which provides both human and model performance on identical problems), the paper shows that what a model finds hard is fundamentally different from what humans find hard -- and this gap *widens* with extended reasoning (chain-of-thought). Models allocate more computation to problems humans find difficult, even when the model reliably solves them. This means output length is a poor proxy for model uncertainty, while pre-generation probes capture true model-specific difficulty.

**Routing via probe predictions.** Two routing strategies leverage these probes: (1) **Cascade routing**: a cheap model attempts the query; if its probe-predicted success probability falls below threshold tau, the query escalates to a stronger model. (2) **Utility-based routing**: for each query, select the model maximizing `p_hat(x) - lambda * cost`, where lambda controls the accuracy-cost tradeoff. Sweeping lambda traces a Pareto frontier. On MATH, utility routing matched the best single model's accuracy at 70% lower cost.

## Step-by-Step Workflow

1. **Define the model pool.** Select 2-4 models spanning a cost-capability range (e.g., a 7B math specialist, a 20B general model, and a 70B+ frontier model). Record each model's per-token or per-query cost.

2. **Generate calibration data.** For each model in the pool, run it on a representative sample of your task domain (500-2000 problems). For each problem, sample K=50 responses at temperature 0.7 and compute the empirical success rate: `s = (1/K) * sum(correct_k)`. This gives you ground-truth difficulty labels per model per problem.

3. **Extract pre-generation activations.** For each problem, run a forward pass through each model with just the input prompt (no generation). Extract the hidden state vector from the residual stream at the final input token position, before the final layer norm. Store these as your probe training features.

4. **Train linear probes per model.** For each model, fit a ridge regression probe (for continuous success rate prediction) or L2-regularized logistic regression (for binary pass/fail). Use 80/20 train/validation split. Grid-search regularization strength alpha over `{1e-3, 1e-2, ..., 1e4}`. Apply Platt scaling for probability calibration on the validation set.

5. **Evaluate probe quality.** Measure Spearman correlation between predicted and actual success rates, and AUROC for binary classification. Expect AUROC in the 0.64-0.78 range depending on task and reasoning budget. Probes should substantially beat the length baseline and TF-IDF baseline.

6. **Implement the router.** Choose a routing strategy:
   - **Cascade**: Set threshold tau. For each incoming query, run the cheapest model's probe. If predicted success >= tau, use that model. Otherwise escalate.
   - **Utility**: For each query, compute `score_i = p_hat_i(x) - lambda * cost_i` for every model. Route to `argmax(score_i)`. Tune lambda on a validation set to hit your target cost-accuracy operating point.

7. **Trace the Pareto frontier.** Sweep lambda (or tau) across its range. Plot accuracy vs. total inference cost. Identify the operating point that meets your budget or accuracy constraint.

8. **Deploy with monitoring.** Log predicted success probabilities alongside actual outcomes. Track probe calibration drift over time. Retrain probes periodically as the task distribution shifts.

9. **Handle edge cases.** For queries where all probes predict low success, route to the strongest model and flag for human review. For queries where all probes predict high success, always use the cheapest model.

## Concrete Examples

**Example 1: Math competition routing system**

User: "I'm serving math problems through an API using three models (7B, 20B, 70B). Most problems are easy but I'm paying frontier prices for everything. Help me build a router."

Approach:
1. Generate calibration data: run each model on 1000 MATH problems, K=50 samples each
2. Extract activations from final input token of each model's residual stream
3. Train three linear probes (one per model) predicting success rate
4. Implement utility router:

```python
import numpy as np

class ProbeRouter:
    def __init__(self, probes, costs, lam=0.5):
        """
        probes: dict mapping model_name -> trained sklearn linear probe
        costs: dict mapping model_name -> cost per query (dollars)
        lam: accuracy-cost tradeoff parameter
        """
        self.probes = probes
        self.costs = costs
        self.lam = lam

    def route(self, activation_dict):
        """
        activation_dict: {model_name: hidden_state_vector} from forward passes
        Returns: name of the model to use for generation
        """
        best_model, best_score = None, -float('inf')
        for name, probe in self.probes.items():
            p_success = probe.predict_proba(
                activation_dict[name].reshape(1, -1)
            )[0, 1]
            score = p_success - self.lam * self.costs[name]
            if score > best_score:
                best_score = score
                best_model = name
        return best_model
```

Output: On MATH, this routes ~60% of queries to the 7B model, ~25% to 20B, and ~15% to 70B, matching the 70B model's accuracy at ~30% of the cost.

**Example 2: Cascade routing for code generation**

User: "I want to try the small model first and only call the expensive model when it's likely to fail. How do I set the threshold?"

Approach:
1. Train a binary probe on the small model's activations predicting pass/fail
2. Implement cascade with threshold search:

```python
from sklearn.metrics import accuracy_score

def find_optimal_threshold(probe, val_activations, val_labels,
                           cost_small, cost_large, budget_ratio=0.5):
    """Find tau that maximizes accuracy within budget."""
    probs = probe.predict_proba(val_activations)[:, 1]
    best_tau, best_acc = 0.5, 0

    for tau in np.arange(0.1, 0.95, 0.05):
        escalate = probs < tau
        # Queries kept by small model use small model's actual correctness
        # Escalated queries use large model's actual correctness
        predicted_correct = np.where(escalate, val_labels_large, val_labels_small)
        cost = (1 - escalate.mean()) * cost_small + escalate.mean() * cost_large

        if cost <= budget_ratio * cost_large * len(probs):
            acc = predicted_correct.mean()
            if acc > best_acc:
                best_acc = acc
                best_tau = tau

    return best_tau

# At inference time:
def cascade_route(query_activation, probe, tau):
    p_success = probe.predict_proba(query_activation.reshape(1, -1))[0, 1]
    if p_success >= tau:
        return "small_model"  # confident it will succeed
    return "large_model"      # escalate
```

Output: With tau=0.65, the cascade calls the large model on only ~35% of queries while maintaining 98% of its overall accuracy.

**Example 3: Diagnosing model-specific difficulty**

User: "I want to understand what kinds of problems my fine-tuned model struggles with, beyond just looking at error rates."

Approach:
1. Train a linear probe on your model's pre-generation activations
2. Run the probe on your eval set to get predicted success probabilities
3. Analyze the probe's learned weight vector:

```python
# After training probe on hidden states of shape (n_samples, hidden_dim)
probe_weights = probe.coef_[0]  # shape: (hidden_dim,)

# Find which activation dimensions most predict failure
failure_dims = np.argsort(probe_weights)[:20]   # most negative = predict failure
success_dims = np.argsort(probe_weights)[-20:]  # most positive = predict success

# Cluster problems by predicted difficulty
from sklearn.cluster import KMeans
probs = probe.predict_proba(all_activations)[:, 1]
hard_mask = probs < 0.3
easy_mask = probs > 0.8

# Analyze hard vs easy problem characteristics
print(f"Hard problems: {hard_mask.sum()} ({hard_mask.mean():.1%})")
print(f"Easy problems: {easy_mask.sum()} ({easy_mask.mean():.1%})")
```

Output: This reveals model-specific blind spots that don't align with human difficulty rankings -- problems that look easy to humans but the model consistently fails on, and vice versa.

## Best Practices

- **Do** extract activations from the residual stream *before* layer normalization at the final input token. This position captures the model's full encoding of the problem before any generation decisions.
- **Do** use K>=50 samples per problem when computing ground-truth success rates. Fewer samples produce noisy labels that degrade probe training.
- **Do** apply Platt scaling to calibrate probe outputs into true probabilities. Raw logistic regression outputs are often poorly calibrated.
- **Do** train separate probes per model. Difficulty is model-specific -- a probe trained on Model A's activations does not transfer to Model B.
- **Avoid** using output length or chain-of-thought verbosity as a difficulty proxy. The paper shows these correlate with *human* difficulty, not model difficulty, and the correlation inverts under extended reasoning.
- **Avoid** assuming human difficulty benchmarks (like IRT parameters) predict model failure. Model-specific probes consistently outperform human difficulty ratings for predicting model success.

## Error Handling

- **Probe AUROC below 0.60**: Your calibration dataset may be too small or too homogeneous. Increase diversity of problems and ensure a balanced distribution of easy/hard instances.
- **Routing underperforms the best single model**: Lambda (or tau) is miscalibrated. Re-sweep on a held-out validation set. Also verify that probe predictions are well-calibrated via reliability diagrams.
- **Activations unavailable (API-only models)**: This technique requires access to internal hidden states. For black-box APIs, fall back to embedding-based proxies or logprob-based confidence estimation, though these are weaker signals.
- **Distribution shift at deployment**: Probe accuracy degrades when the query distribution drifts from training data. Monitor calibration metrics (Brier score) and retrain probes when performance drops.

## Limitations

- Requires **white-box access** to model internals (hidden states). Cannot be applied to models served only through opaque APIs without activation access.
- Probes are **model-specific and task-specific**. A probe trained for math on Model A does not generalize to coding on Model A, or to math on Model B.
- Extended reasoning (long chain-of-thought) **degrades probe signal** -- AUROC drops from ~0.78 to ~0.64 as reasoning budget increases. The method works best with standard inference, not heavy chain-of-thought.
- The activation extraction forward pass still costs compute. For very small models, the probe overhead may not justify the savings. The technique is most valuable when routing between models with large cost differentials.
- Linear probes capture only **linear** structure in the activation space. Nonlinear probes might recover stronger signals but risk overfitting and lose interpretability.

## Reference

**Paper**: [LLMs Encode Their Failures: Predicting Success from Pre-Generation Activations](https://arxiv.org/abs/2602.09924v1) (Lugoloobi et al., 2026). Look for: Section 4 on probe training methodology, Section 5 on routing algorithms (cascade and utility-based), and Section 6 on the divergence between model and human difficulty under extended reasoning. **Code**: [github.com/KabakaWilliam/llms_know_difficulty](https://github.com/KabakaWilliam/llms_know_difficulty)