---
name: "llms-as-high-dimensional-nonlinear"
description: "Analyze, debug, and design LLM systems using the mathematical framework of high-dimensional nonlinear autoregressive models with attention. Apply equation-level reasoning to training losses, alignment objectives (RLHF/DPO/RSFT/RLVR), inference behavior (hallucination, sycophancy, ICL, CoT), and architectural decisions. Trigger phrases: 'analyze my training loss', 'debug alignment objective', 'why is my model hallucinating', 'compare DPO vs RLHF', 'explain attention mathematically', 'design a reward model'"
---

# LLMs as High-Dimensional Nonlinear Autoregressive Models

This skill equips Claude to reason about LLM training, alignment, and inference through the lens of Krishnamurthy's autoregressive framework, where transformers are formalized as Delta-order nonlinear autoregressive models with bilinear-softmax-linear attention. Instead of treating LLM components as disconnected architectural blocks, this framework unifies pretraining, alignment (RLHF, DPO, RSFT, RLVR), and generation under a single probabilistic model — enabling precise diagnosis of training bugs, principled comparison of alignment methods, mathematical analysis of failure modes like hallucination and sycophancy, and informed architectural decisions grounded in equations rather than intuition.

## When to Use

- When a user asks to debug or analyze a training loss function for a transformer model (e.g., "my cross-entropy loss plateaus", "loss spikes during fine-tuning")
- When comparing alignment methods: RLHF vs DPO vs RSFT vs RLVR for a specific use case
- When diagnosing hallucination in a deployed model and the user wants a principled explanation or mitigation strategy
- When implementing or reviewing autoregressive generation code (temperature sampling, KV caching, top-k/top-p)
- When a user asks about sycophancy, reward hacking, or alignment artifacts and wants the mathematical root cause
- When designing a reward model or preference dataset and needing to understand what the objective actually optimizes
- When implementing RAG and wanting to understand how retrieval mathematically constrains generation
- When reviewing or writing nanoGPT/nanochat-style training loops and needing correctness checks against the formal model

## Key Technique

**The Autoregressive Core.** An LLM generates token `x_{t+1}` by sampling from `pi_t = softmax((W_out * f_{theta,t}(x_{t-Delta+1:t}) + b_out) / tau)`, where `f_{theta,t}` is the transformer feature map applied to the last Delta tokens (the context window), `W_out` is the unembedding matrix, and `tau` is temperature. The feature map is constructed by stacking L layers, each applying the bilinear-softmax-linear attention operation: compute attention scores via bilinear form `<M*h_t, N*h_s>` (query-key dot product), normalize with softmax across positions, then linearly aggregate value vectors `L*h_s`. This decomposition is critical because it reveals that expressiveness comes from the *composition* of these operations across layers — not from any single layer in isolation.

**Alignment as KL-Regularized Reward Maximization.** All alignment methods share a common structure: maximize expected reward while staying close to a reference policy. RLHF solves `argmax E[R_phi(y) - beta * KL(pi_theta || pi_ref)]` with a learned reward model. DPO eliminates the reward model entirely by deriving a closed-form loss from Bradley-Terry preferences: `L_DPO = -log sigma(beta * [log(pi_theta(y+|x)/pi_theta(y-|x)) - log(pi_ref(y+|x)/pi_ref(y-|x))])`. RSFT approximates the same target distribution `pi*(y|x) proportional to pi_ref(y|x) * exp(R(y)/tau)` via generate-score-select-finetune. RLVR replaces the learned reward model with a deterministic verifier (compiler, test suite), eliminating reward hacking. Understanding these as points on a spectrum — from fully learned rewards (RLHF) to fully verifiable rewards (RLVR) — guides method selection.

**Failure Modes from First Principles.** Hallucination arises when approximately neutral Jacobian dynamics in residual streams fail to correct semantic deviations, causing generation to drift into coherent but factually incorrect continuations. The hallucination rate `Hall(theta) = E[1 - V(c(y))]` can be reduced by RAG because retrieved context constrains the generation trajectory toward evidence-supported regions. Sycophancy is an alignment artifact: preference models that reward agreement signals regardless of veracity cause `theta*` to overweight user-affirming completions. These are not vague explanations — they are consequences of the optimization objectives.

## Step-by-Step Workflow

1. **Identify the component under analysis.** Determine whether the user's question concerns pretraining (loss function, data), alignment (RLHF/DPO/RSFT/RLVR objective), or inference (generation, hallucination, caching). Map it to the corresponding equation in the framework.

2. **Write out the relevant objective explicitly.** For training: `L(theta) = -sum log pi_hat_t(x_{t+1})`. For DPO: the Bradley-Terry preference loss. For RLHF: the KL-regularized reward objective. For generation: the autoregressive sampling equation. Making the equation concrete prevents hand-waving.

3. **Trace the data flow through the model.** Token embeddings `W_emb[x]` -> L layers of bilinear-softmax-linear attention with residual connections -> unembedding `W_out * h + b_out` -> softmax with temperature. Identify where the user's issue likely originates (embedding, attention, output head, temperature).

4. **Check dimensional consistency.** Vocabulary size V, hidden dimension d, context length Delta, number of layers L. Verify that matrix dimensions match: `W_emb` is V x d, `W_out` is V x d, attention matrices M, N, L are d x d_head. Mismatches here are a common source of bugs.

5. **Analyze the gradient or optimization landscape.** For training issues: check if the cross-entropy gradient is well-behaved (vanishing/exploding through L attention layers). For alignment: verify that the KL penalty beta is appropriate — too small allows reward hacking, too large prevents learning.

6. **Diagnose failure modes using the mathematical indicators.** For hallucination: check `Hall(theta)` — is retrieval augmentation reducing it? For sycophancy: is the reward model correlating with agreement rather than correctness? For mode collapse in RLHF: is KL divergence from reference growing unbounded?

7. **Recommend the appropriate alignment method.** If verifiable rewards exist (code, math) -> RLVR. If preference data is available and you want simplicity -> DPO. If you need fine-grained reward shaping -> RLHF. If you want a simple baseline -> RSFT with high N.

8. **Validate with code.** Write or review the implementation against the formal equations. The nanoGPT pattern for generation is the reference: crop context to block_size, forward pass to get logits, divide by temperature, softmax, multinomial sample, concatenate.

9. **Verify KV cache correctness for inference.** Cache state `S_t = union over L layers of {(k_{l,s}, v_{l,s})}` for all positions s. Each new token only computes its own query and attends over cached keys/values. Confirm O(t) per-token complexity rather than O(t^2).

10. **Communicate findings in terms of the framework.** Frame explanations using the autoregressive model vocabulary: "the generation trajectory drifts because...", "the alignment objective is maximizing...", "the attention scores concentrate on...". This provides actionable precision.

## Concrete Examples

**Example 1: Debugging a DPO Training Run**

User: "My DPO loss is oscillating and the model outputs are getting worse. What's going wrong?"

Approach:
1. Write the DPO loss: `L = -log sigma(beta * [log(pi_theta(y+|x)/pi_theta(y-|x)) - log(pi_ref(y+|x)/pi_ref(y-|x))])`
2. Check beta value — oscillation often means beta is too large, making the sigmoid argument swing between extremes
3. Verify that `pi_ref` is frozen and not being updated (a common bug)
4. Check if preferred/rejected pairs have sufficient quality gap — if `pi_ref` already assigns similar probability to both, the log-ratio difference is near zero and gradients are noisy
5. Inspect whether the model is overfitting to surface patterns in preference data

Output:
```
Diagnosis: Your beta=0.5 is too aggressive for your preference margin.

The DPO objective computes:
  delta = beta * (log_ratio_policy - log_ratio_ref)

When beta is large and preference pairs are close in quality,
delta oscillates between large positive and negative values,
causing unstable sigmoid gradients.

Recommendations:
1. Reduce beta to 0.1 and monitor KL(pi_theta || pi_ref)
2. Filter preference pairs where |R(y+) - R(y-)| < threshold
3. Verify pi_ref is frozen: ensure ref_model.eval() and no_grad()

Reference code check:
  # Correct DPO loss
  ref_logratios = ref_logprobs_chosen - ref_logprobs_rejected  # frozen
  policy_logratios = policy_logprobs_chosen - policy_logprobs_rejected
  loss = -F.logsigmoid(beta * (policy_logratios - ref_logratios)).mean()
```

**Example 2: Analyzing Hallucination in a Deployed Model**

User: "Our model confidently generates false claims about recent events. How do we fix this mathematically?"

Approach:
1. Frame using the hallucination indicator: `H(y) = 1 - V(c(y))`, where V is a verification function
2. Explain the autoregressive drift mechanism: once a false token enters the context, subsequent tokens condition on it, and approximately neutral Jacobian dynamics fail to self-correct
3. Quantify expected hallucination rate: `Hall(theta) = E_{x,y}[H(y)]`
4. Show how RAG reduces this: `Hall_RAG(theta) << Hall(theta)` by prepending retrieved evidence `[z_1:...:z_m]` to the context

Output:
```
Root cause: The autoregressive generation equation
  x_{t+1} ~ softmax(W_out * f_theta(x_{t-Delta+1:t}) / tau)
conditions each new token on ALL previous tokens including any
hallucinated ones. The residual stream Jacobian is approximately
neutral — it propagates errors without correcting them.

Mitigation strategy (ordered by effectiveness):

1. RAG grounding: Prepend retrieved evidence to context
   x'_{1:t'} = [z_1:...:z_m : x_{1:t}]
   This constrains the generation trajectory toward evidence-
   supported regions, measurably reducing Hall(theta).

2. Lower temperature: Reduce tau from 1.0 to 0.3-0.7.
   Lower tau sharpens softmax, concentrating probability mass
   on high-likelihood (typically factual) continuations.

3. RLVR alignment: If claims are verifiable, train with
   deterministic verification rewards instead of human preferences.
   This eliminates reward model errors that RLHF is susceptible to.

4. Self-consistency decoding: Sample N completions, extract claims,
   retain only claims verified across majority of samples.
```

**Example 3: Choosing Between Alignment Methods**

User: "We're building a coding assistant. Should we use RLHF, DPO, or something else for alignment?"

Approach:
1. Map the domain to the alignment method spectrum
2. Note that code correctness is verifiable (compilers, test suites)
3. Compare objectives: RLHF learns a proxy reward (risk of reward hacking), DPO needs preference pairs (hard to collect at scale for code), RLVR uses deterministic verification (ideal for code)

Output:
```
For a coding assistant, RLVR is the strongest choice.

Method comparison for your use case:

| Method | Reward Source        | Risk              | Fit for Code |
|--------|---------------------|-------------------|--------------|
| RLHF   | Learned R_phi(y)    | Reward hacking    | Medium       |
| DPO    | Preference pairs    | Data quality      | Medium       |
| RSFT   | Best-of-N sampling  | Ceiling = R_phi   | Medium       |
| RLVR   | Compiler/test suite | None (verifiable) | Excellent    |

RLVR objective:
  Generate K reasoning paths per problem
  Reward = 1.0 if code passes all tests, 0.0 otherwise
  Optimize with PPO/GRPO on these binary rewards

Why RLVR wins here:
- Code correctness is deterministically verifiable
- No learned reward model means no reward hacking
- Binary signal is sparse but noise-free
- Scales with test suite coverage, not human annotation

Combine with RSFT as a warm-start: generate N solutions,
keep those passing tests, SFT on them before RLVR training.
```

## Best Practices

- **Do:** Always write the objective function explicitly before analyzing behavior. Vague reasoning about "the model learns to..." is replaced by "theta* = argmin L(theta) where L = ..." — this catches errors.
- **Do:** Check the KL divergence `KL(pi_theta || pi_ref)` during alignment training. If it grows unbounded, the model is drifting too far from the pretrained distribution, risking mode collapse or degenerate outputs.
- **Do:** Use the bilinear-softmax-linear decomposition when debugging attention. Separate issues into: are queries/keys misaligned (bilinear)? Is attention too diffuse (softmax temperature)? Are values carrying the wrong information (linear)?
- **Do:** Frame RAG as context augmentation `x' = [retrieved : original]` — this makes clear that retrieval quality directly determines the generation distribution, not just "helps" it.
- **Avoid:** Treating alignment methods as black boxes. Every method optimizes a specific objective — if the model misbehaves, the first question is "what is the objective actually rewarding?"
- **Avoid:** Ignoring the autoregressive dependency structure when debugging generation. Each token conditions on all previous tokens. A single bad token early in generation corrupts the entire continuation through the context window.

## Error Handling

- **Training loss is NaN or Inf:** Check for numerical overflow in the softmax computation of attention scores. With large d_head, raw dot products `<Mh, Nh>` can exceed float16 range. Solution: verify that 1/sqrt(d_head) scaling is applied to attention logits.
- **DPO loss collapses to zero:** The model has memorized the preference dataset. The log-ratio `log(pi_theta(y+)/pi_theta(y-))` saturates. Reduce learning rate, increase beta, or add dropout.
- **RLHF reward hacking:** The policy exploits artifacts in the learned reward model (e.g., longer outputs score higher). Monitor the actual task metric alongside R_phi. Consider switching to RLVR if verification is possible.
- **KV cache OOM during inference:** Cache size is O(L * d * t) per sequence. For long contexts, use sliding window attention (effectively reducing Delta) or quantize cached key-value pairs to int8.
- **Generation degenerates into repetition:** Temperature tau is too low, causing the softmax to collapse to argmax. Increase tau or apply repetition penalty (multiplicative discount on logits of recently generated tokens).

## Limitations

- This framework provides the mathematical structure but does not predict emergent capabilities — phenomena like in-context learning and chain-of-thought are described as consequences of the autoregressive mechanism, not predicted from first principles.
- The hallucination analysis identifies the mechanism (autoregressive drift with neutral Jacobian) but does not provide a closed-form solution for elimination — mitigation (RAG, lower temperature, RLVR) reduces but cannot guarantee zero hallucination rate.
- Alignment objectives assume the reward model or verifier is well-specified. In practice, reward models have systematic biases (length, style, sycophancy) that the framework flags but cannot automatically correct.
- The bilinear-softmax-linear decomposition applies to standard scaled dot-product attention. Architectural variants (linear attention, state-space hybrids, mixture-of-experts routing) require modified formulations not covered here.
- Quantitative predictions (e.g., "reducing beta from 0.5 to 0.1 will reduce oscillation") are directionally correct but system-dependent — exact values require empirical tuning for specific model sizes and datasets.

## Reference

**Paper:** Krishnamurthy, V. (2026). "LLMs as High-Dimensional Nonlinear Autoregressive Models with Attention: Training, Alignment and Inference." arXiv:2602.00426v1. https://arxiv.org/abs/2602.00426v1

Look for: The unified autoregressive formulation (Section 2), the bilinear-softmax-linear attention decomposition (Section 3), the side-by-side alignment objectives RLHF/DPO/RSFT/RLVR (Section 4), the hallucination and sycophancy analysis (Section 5), and the nanoGPT/nanochat code listings that ground every equation in runnable PyTorch.