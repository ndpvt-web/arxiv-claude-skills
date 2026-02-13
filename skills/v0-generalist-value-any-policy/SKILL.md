---
name: "v0-generalist-value-any-policy"
description: "Implement V0-style generalist value estimation that profiles any LLM policy from behavioral history rather than parameter fitting. Use for intelligent sampling budget allocation, LLM routing, and RL training optimization. Trigger phrases: 'allocate sampling budget across prompts', 'route queries to the cheapest capable model', 'estimate model success probability before running', 'optimize GRPO rollout budget', 'build an LLM router with performance prediction', 'profile model capability from history'"
---

# V0: Generalist Value Estimation for Any Policy at State Zero

This skill enables Claude to implement the V0 approach to value estimation — predicting whether a given LLM will succeed on a given prompt *before running it*, using only behavioral history (past prompt-outcome pairs) as context. Unlike traditional value models that require synchronous retraining as the policy evolves, V0 treats policy capability as explicit context input, enabling zero-gradient adaptation to unseen models. This is directly applicable to building LLM routers, optimizing RL sampling budgets, and any system that needs to predict model performance cheaply.

## When to Use

- When the user wants to build an LLM router that dispatches queries to the cheapest model capable of handling them
- When implementing GRPO or similar RL training and needing to allocate rollout sampling budgets efficiently across prompts
- When the user asks to predict whether a model will succeed on a task before actually running inference
- When building evaluation pipelines that need to profile model capabilities from a sample of past results
- When the user wants to avoid wasting compute on prompts that are either trivially easy or impossibly hard for the current model
- When designing a system that ranks or selects among multiple LLM candidates per query based on cost-performance tradeoffs

## Key Technique

**The core insight:** Instead of training a value model whose parameters encode a specific policy's capability (requiring retraining whenever the policy changes), V0 conditions value predictions on a *context window of historical query-performance pairs*. Given a set C_pi of (instruction, binary_outcome) pairs from a policy, V0 predicts P(success | new_query, C_pi) in a single forward pass. This is formalized as a Posterior Predictive Distribution: P(r|x, C_pi) = integral of P(r|x, M) * P(M|C_pi) dM — the model implicitly infers what kind of policy produced the history, then predicts accordingly.

**The shortcut problem and debiasing:** A naive approach degenerates into a heuristic that merely judges overall policy strength from context (e.g., "this model gets 80% right, so predict 0.8 for everything") while ignoring query-specific reasoning. The paper proves this via mutual information decomposition: I(Y; X, C) = I(Y; C) + I(Y; X|C), where the first term dominates. The fix is a *pairwise ranking loss* that compares scores of a success vs. a failure *within the same context*, algebraically canceling the context-dependent bias: L_rank = -E[log sigma(s(x_success, C) - s(x_failure, C))]. The final objective blends ranking loss (weight alpha=0.25) with soft cross-entropy for calibration.

**Architecture:** V0 uses a frozen embedding model (semantic backbone) to encode queries and context, a Residual Query Adapter that projects entangled embeddings into K independent feature channels via multi-head attention with learnable static queries + dynamic offsets, and a TabPFN inference head that performs Bayesian in-context learning in one forward pass — treating historical pairs as tabular observations to construct decision boundaries without gradients.

## Step-by-Step Workflow

1. **Collect behavioral history for each candidate model.** Run a sample of N prompts (50-200 is sufficient) through each LLM and record binary outcomes (pass/fail based on a verifier or judge). Store as structured pairs: `[(prompt_text, 0_or_1), ...]`.

2. **Encode prompts into semantic embeddings.** Use a sentence embedding model (e.g., Qwen3-Embedding, all-MiniLM, or OpenAI embeddings) to convert each prompt into a fixed-dimensional vector. Cache these embeddings for reuse.

3. **Build the context representation.** For a given policy, sample a context window C of K historical (embedding, outcome) pairs. This context *is* the policy profile — no model weights needed.

4. **Implement the Residual Query Adapter.** Project the concatenated query embedding and context embeddings through a multi-head cross-attention layer with L learnable static query vectors plus dynamic residual offsets. This disentangles semantic features into K structured channels suitable for tabular classification.

5. **Apply a tabular in-context learner as the prediction head.** Use TabPFN (or a similar in-context tabular model) to predict P(success | query, context) in a single forward pass. The context pairs serve as the "training set" and the new query is the "test point."

6. **Train with the composite debiased objective.** Combine pairwise ranking loss (alpha=0.25) with soft cross-entropy (1-alpha=0.75). For each training batch, sample pairs of (success_query, failure_query) from the same context to compute the ranking term. This prevents shortcut learning.

7. **For budget allocation:** Given predicted success probabilities {p_i} for a batch of prompts, allocate sampling budget B_i to maximize utility: `Utility(B_i, p_i) = B_i * (1 - p_i) * [1 - (1 - p_i)^(B_i - 1)]`. Skip prompts where p_i is near 0 (impossible) or near 1 (trivial). Concentrate rollouts on the uncertain middle range.

8. **For LLM routing:** Score each candidate model m on query x as `r_beta = beta * P(success | x, C_m) + (1 - beta) * (1 - normalized_cost_m)`, where beta controls the performance-cost tradeoff. Route to the model with the highest r_beta.

9. **Validate with Intra-Context AUC.** For each context, compute AUC on held-out queries from the same policy. A well-trained V0 should achieve >0.85 Intra-Context AUC, confirming it discriminates query difficulty rather than just estimating average policy strength.

10. **Deploy as a lightweight pre-filter.** At inference time, updating V0 for a new or fine-tuned model requires only collecting ~100 behavioral samples and passing them as context — no retraining, no gradient updates, no GPU hours.

## Concrete Examples

**Example 1: Building an LLM Router**

```
User: I have 4 models (GPT-4o, Claude Sonnet, Llama-70B, Llama-8B) with different
costs. Build a router that sends each query to the cheapest model that can handle it.

Approach:
1. Sample 150 diverse prompts and run each through all 4 models.
   Record pass/fail via an automated judge. Result:
     gpt4o_history = [("Solve x^2=4", 1), ("Prove Riemann...", 0), ...]
     sonnet_history = [("Solve x^2=4", 1), ("Prove Riemann...", 0), ...]
     llama70b_history = [("Solve x^2=4", 1), ("Prove Riemann...", 0), ...]
     llama8b_history = [("Solve x^2=4", 0), ("Prove Riemann...", 0), ...]

2. Embed all prompts with a sentence transformer.
3. For each incoming query, predict P(success | query, C_model) for all 4 models.
4. Compute routing score with beta=0.6 and normalized costs:
     costs = {"gpt4o": 1.0, "sonnet": 0.7, "llama70b": 0.3, "llama8b": 0.05}
     for model in models:
         score = 0.6 * p_success[model] + 0.4 * (1 - costs[model])
5. Route to argmax(score).

Output:
  Query: "What is the capital of France?"
  -> llama8b (p=0.98, cost=0.05, score=0.97)

  Query: "Write a formal proof that sqrt(2) is irrational"
  -> sonnet (p=0.91, cost=0.7, score=0.67)

  Query: "Translate this legal contract to Mandarin with correct terminology"
  -> gpt4o (p=0.85, cost=1.0, score=0.51)
  Savings: ~60% cost reduction vs always using GPT-4o, <2% quality loss.
```

**Example 2: Optimizing GRPO Sampling Budget**

```
User: I'm training a math model with GRPO. Some prompts are trivially easy and some
are impossible — I'm wasting rollouts. Help me allocate budget intelligently.

Approach:
1. After each training epoch, collect 200 (prompt, pass/fail) samples from the
   current policy checkpoint as context C_current.
2. For the next batch of 1000 training prompts, predict p_i = P(success | x_i, C_current).
3. Compute optimal budget allocation per prompt:
     base_budget = 8  # default rollouts per prompt
     for each prompt i:
         if p_i > 0.95:    budget_i = 1   # trivial, one sample suffices
         elif p_i < 0.05:  budget_i = 1   # impossible, don't waste compute
         else:              budget_i = round(base_budget * 4 * p_i * (1 - p_i))
                            # peaks at p=0.5 (maximum uncertainty)
4. Redistribute saved budget to uncertain prompts (0.2 < p < 0.8).

Output:
  Before: 1000 prompts x 8 rollouts = 8000 total samples
  After:  1000 prompts, variable budget = 4200 total samples
  - 312 trivial prompts (p>0.95): 312 samples (saved 2184)
  - 89 impossible prompts (p<0.05): 89 samples (saved 623)
  - 599 informative prompts: 3799 samples (redistributed savings)
  Result: ~2% higher reward with 47% fewer rollouts.
```

**Example 3: Profiling a New Fine-Tuned Model**

```
User: I just fine-tuned Llama-8B on coding tasks. How good is it now compared
to before, without running a full benchmark?

Approach:
1. Take the existing V0 system (already trained on diverse model histories).
2. Run 100 randomly sampled coding prompts through the new fine-tuned model.
   Record outcomes: ft_history = [("FizzBuzz in Rust", 1), ("B-tree impl", 0), ...]
3. Also run the same 100 prompts through base Llama-8B for comparison context.
4. Feed both contexts to V0 and predict on a held-out set of 500 coding prompts.

Output:
  Base Llama-8B capability profile:
    Easy coding (loops, strings):     p = 0.82
    Medium (data structures, APIs):   p = 0.41
    Hard (algorithms, system design): p = 0.09

  Fine-tuned Llama-8B capability profile:
    Easy coding (loops, strings):     p = 0.95 (+13%)
    Medium (data structures, APIs):   p = 0.68 (+27%)
    Hard (algorithms, system design): p = 0.22 (+13%)

  Conclusion: Fine-tuning significantly improved medium-difficulty tasks.
  The model is now competitive with Llama-70B base on easy/medium coding.
```

## Best Practices

- **Do:** Use at least 50 behavioral samples per model for context. Fewer leads to noisy capability profiles; 100-200 is the sweet spot for stability.
- **Do:** Always include the pairwise ranking loss (alpha=0.25) during training. Without it, the model collapses to predicting average success rate regardless of query content.
- **Do:** Validate using Intra-Context AUC (not just overall accuracy). This metric specifically measures whether the model discriminates between easy and hard queries *within* a single policy's context.
- **Do:** Cache embeddings aggressively. The embedding step is the most expensive per-query cost; the TabPFN forward pass is near-instant.
- **Avoid:** Using V0 predictions as the sole signal — combine with actual rollout results. V0 is a prior, not an oracle.
- **Avoid:** Training on contexts from only one or two models. V0 generalizes by seeing diverse capability profiles during training. Use histories from at least 5-10 distinct model checkpoints or variants.
- **Avoid:** Context windows larger than 200 pairs. The TabPFN head handles up to ~1000 observations, but diminishing returns set in quickly and latency increases.

## Error Handling

- **Context too small (<20 samples):** V0 predictions will be unreliable. Fall back to uniform budget allocation or a simple heuristic (e.g., prompt length as difficulty proxy). Log a warning.
- **Distribution shift in queries:** If incoming queries are from a very different domain than the training data, Intra-Context AUC will drop. Detect this by monitoring the embedding distance between new queries and the training distribution. Re-collect behavioral samples from the shifted domain.
- **Shortcut collapse during training:** If validation shows high overall accuracy but Intra-Context AUC near 0.5, the model has learned to predict mean success rate only. Increase the ranking loss weight alpha (try 0.4-0.5) and ensure training batches contain diverse policies.
- **TabPFN numerical instability:** With very imbalanced contexts (>95% success or >95% failure), the Bayesian posterior can become degenerate. Add Laplace smoothing by injecting 1-2 synthetic opposite-outcome pairs into the context.

## Limitations

- V0 estimates value only at state zero (the initial prompt). It does not predict intermediate reasoning quality or partial credit — only binary success/failure on the complete task.
- The approach assumes a reliable verifier or judge exists to label outcomes. For open-ended creative tasks without clear pass/fail criteria, V0 cannot be directly applied.
- TabPFN scales to ~1000 context points and ~500 feature dimensions. For extremely large-scale routing across thousands of models, a different inference head may be needed.
- V0 profiles *average* policy behavior. It cannot capture non-deterministic variance (e.g., a model that succeeds 50% of the time on a specific prompt through lucky sampling).
- The technique requires behavioral samples from each new model. Fully zero-shot prediction for a model with no history is not possible — at minimum, ~50 samples are needed.

## Reference

- **Paper:** [V0: A Generalist Value Model for Any Policy at State Zero](https://arxiv.org/abs/2602.03584v1) (Zhang et al., 2026)
- **Key insight to look for:** Section 3.2 on the mutual information decomposition that reveals the shortcut problem, and Section 4.1 on the pairwise ranking debiasing loss. These are the critical contributions that make context-conditioned value estimation actually work rather than collapsing to a trivial baseline predictor.