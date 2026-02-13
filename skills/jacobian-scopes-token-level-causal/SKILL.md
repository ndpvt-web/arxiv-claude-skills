---
name: "jacobian-scopes-token-level-causal"
description: "Implement Jacobian Scope token-level causal attribution for LLM interpretability. Computes gradient-based influence scores showing which input tokens most affect a model's prediction. Three variants: Semantic Scope (target logit sensitivity), Fisher Scope (full distribution shift), Temperature Scope (confidence change). Use when asked to 'attribute predictions to input tokens', 'explain which tokens influence output', 'compute Jacobian attribution scores', 'run token-level causal analysis', 'measure input token influence on LLM output', or 'debug LLM prediction with gradient attribution'."
---

# Jacobian Scopes: Token-Level Causal Attribution in LLMs

This skill enables Claude to implement and apply Jacobian Scopes -- a suite of gradient-based methods that quantify how each input token causally influences an LLM's next-token prediction. By computing the Jacobian matrix of the model's final hidden state with respect to input embeddings, then projecting it through task-specific lenses (target logit, Fisher information, or hidden-state norm), you produce per-token scalar attribution scores in a single backward pass. This gives mechanistic, token-granularity explanations of why a model made a specific prediction.

## When to Use

- When the user wants to understand which tokens in a prompt most influence a specific predicted token (e.g., "why did GPT predict 'Paris' here?")
- When debugging translation or instruction-following failures by attributing errors to specific source tokens
- When investigating bias -- checking whether demographic tokens disproportionately steer predictions
- When analyzing in-context learning to see whether a model attends to demonstration labels vs. formatting tokens
- When comparing attribution methods: the user wants Semantic vs. Fisher vs. Temperature scope side-by-side
- When building an interpretability pipeline that needs efficient, single-backward-pass token attribution
- When the user asks to visualize or rank input token importance for any autoregressive LLM prediction

## Key Technique

**The Jacobian as a causal lens.** For a transformer producing final hidden state `y` from input embeddings `x_1, ..., x_T`, the Jacobian at position `t` is `J_t = dy/dx_t`, a `d_model x d_model` matrix. Under first-order Taylor expansion, a small perturbation `delta_t` to token `t`'s embedding causes `Delta_y ~ J_t * delta_t`. The Jacobian thus encodes the linearized causal effect of every input token on the output. All three scope variants extract different scalar summaries from this same Jacobian.

**Three complementary projections.** Semantic Scope picks a target token (e.g., the predicted word) and computes `||w_target^T J_t||_2` where `w_target` is the unembedding vector for that token -- this measures how much input token `t` shifts the logit for the target. Fisher Scope computes `tr(J_t^T F_u J_t)` where `F_u = W^T(diag(p) - pp^T)W` is the Fisher information in hidden space -- this measures total distributional shift across the entire vocabulary. Temperature Scope computes `||y_hat^T J_t||_2` where `y_hat = y/||y||_2` -- this measures how much token `t` affects the model's overall confidence (effective inverse temperature `beta = ||y||_2`).

**Efficiency.** Semantic and Temperature Scopes require only O(1) backward passes (a single vector-Jacobian product via standard autograd). Fisher Scope requires O(d_model) passes for the exact trace, but Hutchinson's stochastic trace estimator reduces this to a small constant number of passes with controlled variance. All methods piggyback on the same forward pass.

## Step-by-Step Workflow

1. **Load model and tokenizer.** Use a HuggingFace autoregressive model (LLaMA, Pythia/GPTNeoX supported out-of-box). Ensure the model is in eval mode with gradients enabled on the embedding layer.

2. **Tokenize the input prompt.** Record the token IDs and keep a mapping from token index to decoded string for later visualization. Identify the prediction position (typically the last token or a specific position of interest).

3. **Extract the embedding layer and unembedding matrix.** Get `model.get_input_embeddings()` for input embeddings and `model.lm_head.weight` (or `model.embed_out.weight` for Pythia) for the unembedding matrix `W`.

4. **Run forward pass, retaining the final hidden state.** Pass input IDs through the model. Extract the final-layer hidden state `y` at the prediction position (before the lm_head projection). Enable `requires_grad` on the input embeddings tensor.

5. **Choose your scope variant and compute the loss scalar.**
   - *Semantic Scope:* `L = w_target^T @ y` where `w_target = W[target_token_id]`
   - *Temperature Scope:* `L = ||y||_2` (L2 norm of hidden state)
   - *Fisher Scope:* For each of `k` random probes `r ~ N(0, I)`: `L_k = (W @ (y / ||y||)) @ r`, then average squared gradients

6. **Backpropagate to get per-token gradients.** Call `L.backward()`. The gradient `dL/dx_t` at each input position `t` is a `d_model`-dimensional vector. Compute the L2 norm `||dL/dx_t||_2` to get the scalar influence score for token `t`.

7. **Normalize scores for visualization.** Divide all influence scores by their maximum (or sum) to get relative attributions in [0, 1]. Higher score = stronger causal influence on the prediction.

8. **Visualize or return results.** Render a heatmap over the input tokens, or return a ranked list of (token, score) pairs. For side-by-side comparison, repeat steps 5-7 for each scope variant.

9. **Interpret the results.** Semantic Scope highlights tokens that push the specific predicted word's logit; Fisher Scope highlights tokens that shift the full output distribution (useful for detecting broad influence); Temperature Scope highlights tokens that make the model more or less confident.

10. **Validate with perturbation checks (optional).** Mask or replace the top-attributed tokens and re-run inference. If attribution is correct, the prediction should change significantly when high-influence tokens are removed.

## Concrete Examples

**Example 1: Diagnosing a translation attribution**

User: "I'm running a LLaMA model on the prompt 'Translate English to French: The cat sat on the mat. Translation:' and it predicts 'Le'. Which input tokens most influenced this prediction?"

Approach:
1. Tokenize the prompt, identify prediction position (after "Translation:")
2. Get `w_target = W[token_id("Le")]` from the unembedding matrix
3. Compute Semantic Scope: `L = w_target @ y`, backprop, take `||grad_t||_2` per token
4. Normalize and rank

Output:
```
Token Attribution (Semantic Scope, target="Le"):
  "The"        -> 0.82  ████████░░
  "Translate"  -> 0.71  ███████░░░
  "French"     -> 0.65  ██████░░░░
  "cat"        -> 0.38  ███░░░░░░░
  "English"    -> 0.34  ███░░░░░░░
  "sat"        -> 0.21  ██░░░░░░░░
  "to"         -> 0.15  █░░░░░░░░░
  "on"         -> 0.12  █░░░░░░░░░
  "the"        -> 0.10  █░░░░░░░░░
  "mat"        -> 0.09  ░░░░░░░░░░
  "."          -> 0.05  ░░░░░░░░░░
  "Translation"-> 0.44  ████░░░░░░
  ":"          -> 0.31  ███░░░░░░░
```
Interpretation: "The" (as subject requiring the French article "Le") and the instruction tokens "Translate" / "French" dominate. Content words like "cat" matter moderately; function words and punctuation contribute little.

**Example 2: Detecting bias via Fisher Scope**

User: "I want to check if my model's prediction changes based on demographic tokens. The prompt is 'The [demographic] person was described as' -- compare attributions when [demographic] is 'young' vs 'elderly'."

Approach:
1. Run Fisher Scope on both prompts at the prediction position
2. Fisher Scope uses `F_u = W^T(diag(p) - pp^T)W` to weight the Jacobian
3. Compare the attribution score of the demographic token between the two prompts
4. Also compare the full distribution shift (entropy of output) between variants

Output:
```
Fisher Scope Attribution for demographic token:
  "young"   -> 0.31 (4.2% of total distributional influence)
  "elderly" -> 0.67 (11.8% of total distributional influence)

The "elderly" token exerts 3x more influence on the full output
distribution, suggesting the model's predictions are disproportionately
sensitive to this demographic descriptor. Inspect the top-5 predicted
tokens for each to see if stereotypical associations emerge.
```

**Example 3: Comparing all three scopes on in-context learning**

User: "I have a 3-shot classification prompt. Which tokens matter most -- the labels, the examples, or the instruction?"

Approach:
1. Tokenize the full few-shot prompt with labeled examples
2. Run all three scopes at the final prediction position
3. Aggregate scores by segment: instruction tokens, example-content tokens, label tokens, formatting tokens

Output:
```
Attribution by segment (averaged, normalized):

Segment          | Semantic | Fisher | Temperature
-----------------+----------+--------+------------
Instruction      |   0.45   |  0.38  |    0.22
Example content  |   0.18   |  0.25  |    0.15
Labels           |   0.72   |  0.61  |    0.34
Formatting (,:)  |   0.08   |  0.11  |    0.52

Key insight: Semantic and Fisher Scopes confirm labels are the
primary drivers of the predicted class. Temperature Scope reveals
that formatting tokens (colons, newlines separating examples)
disproportionately affect model confidence -- suggesting the model
uses structural cues to "recognize" the ICL pattern and sharpen
its distribution.
```

## Implementation Skeleton

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

def jacobian_scopes(model, tokenizer, prompt, target_token=None, scope="semantic"):
    """Compute token-level causal attribution via Jacobian Scopes."""
    model.eval()
    inputs = tokenizer(prompt, return_tensors="pt").to(model.device)
    input_ids = inputs["input_ids"]

    # Get embedding layer and enable gradients
    embed_layer = model.get_input_embeddings()
    embeds = embed_layer(input_ids).detach().requires_grad_(True)

    # Forward pass with embeddings directly
    outputs = model(inputs_embeds=embeds, attention_mask=inputs["attention_mask"])
    hidden = outputs.hidden_states[-1] if hasattr(outputs, 'hidden_states') else None
    # For models without output_hidden_states, use the pre-lm_head state:
    # hidden = model.model(inputs_embeds=embeds).last_hidden_state

    y = hidden[0, -1, :]  # final hidden state at prediction position
    W = model.lm_head.weight  # unembedding matrix [vocab_size, d_model]

    if scope == "semantic":
        if target_token is None:
            logits = W @ y
            target_token = logits.argmax().item()
        w_target = W[target_token]
        loss = w_target @ y

    elif scope == "temperature":
        loss = y.norm(p=2)

    elif scope == "fisher":
        # Hutchinson trace estimator with k probes
        y_hat = y / y.norm(p=2)
        scores = torch.zeros(input_ids.shape[1])
        k_probes = 8
        for _ in range(k_probes):
            if embeds.grad is not None:
                embeds.grad.zero_()
            r = torch.randn_like(y_hat)
            loss = (W @ y_hat) @ r
            loss.backward(retain_graph=True)
            scores += embeds.grad[0].norm(dim=-1) ** 2
        return _format_scores(scores / k_probes, input_ids, tokenizer)

    loss.backward()
    scores = embeds.grad[0].norm(dim=-1)  # L2 norm per token position
    return _format_scores(scores, input_ids, tokenizer)

def _format_scores(scores, input_ids, tokenizer):
    scores = scores / scores.max()
    tokens = [tokenizer.decode(tid) for tid in input_ids[0]]
    return list(zip(tokens, scores.detach().cpu().tolist()))
```

## Best Practices

- **Do** enable `output_hidden_states=True` in model config or use `model.model(...)` to access the pre-lm_head hidden state directly. The Jacobian must be computed with respect to the final hidden state `y`, not post-softmax probabilities.
- **Do** use Semantic Scope as the default starting point -- it is cheapest (single backward pass) and most interpretable for targeted questions like "why did the model predict X?"
- **Do** use Fisher Scope when investigating distributional effects (bias audits, safety analysis) where no single target token is relevant.
- **Do** compare Temperature Scope against Semantic/Fisher to separate "what influences the prediction" from "what influences model confidence" -- these are often different tokens.
- **Avoid** computing the full `d_model x d_model` Jacobian matrix explicitly. Always use vector-Jacobian products via autograd (`loss.backward()`), which is O(1) per scope computation.
- **Avoid** applying these methods to very long sequences (>2048 tokens) without chunking or selecting a window, as gradient memory scales linearly with sequence length.
- **Avoid** interpreting attribution scores as "attention" -- Jacobian Scopes measure end-to-end causal sensitivity through all layers, not per-layer attention patterns.

## Error Handling

- **Model architecture not supported:** If `model.lm_head` or `model.get_input_embeddings()` is missing, check for architecture-specific names (`embed_out` for Pythia, `embed_tokens` for LLaMA). Wrap in a helper that tries multiple attribute paths.
- **CUDA out of memory on Fisher Scope:** Reduce the number of Hutchinson probes (k=4 is usually sufficient) or use gradient checkpointing. Fisher Scope is the most memory-intensive variant.
- **Zero gradients returned:** Ensure `requires_grad_(True)` is called on the embeddings tensor *after* detaching it from the embedding layer. If using `input_ids` directly, the gradient won't flow through the discrete token lookup.
- **Tied embeddings confusion:** Some models tie input and output embeddings (`W = embed_layer.weight`). This is fine -- just use the same weight matrix for both `w_target` lookup and the embedding gradient computation.
- **Numerical instability in Temperature Scope:** If `||y||_2` is very small (near-zero norm), the normalized direction `y_hat` is unstable. Add a small epsilon: `y_hat = y / (y.norm() + 1e-8)`.

## Limitations

- **Linear approximation only.** Jacobian Scopes capture first-order effects. Nonlinear interactions between tokens (e.g., compositional semantics where two tokens jointly shift the prediction) are missed. Path-Integrated Semantic Scope partially addresses this but at higher compute cost.
- **Single prediction position.** Each computation attributes to one output position. Analyzing a full generated sequence requires running attribution at every generation step.
- **No internal mechanism explanation.** Scopes reveal *which* input tokens matter, not *how* they matter (which attention heads, which MLP layers). Combine with activation patching or circuit analysis for mechanistic depth.
- **Gradient saturation.** In regions where the model is extremely confident (near-zero entropy output), gradients can vanish, making all attribution scores uniformly small. Temperature Scope is partially robust to this.
- **Autoregressive models only.** The formulation assumes causal (left-to-right) transformers. Adapting to encoder-decoder or bidirectional models requires modifying which hidden states and positions are targeted.

## Reference

Liu, T.J.B., Zadeoğlu, B., Boullé, N., Sarfati, R., & Earls, C.J. (2026). *Jacobian Scopes: token-level causal attributions in LLMs.* arXiv:2601.16407. Code: https://github.com/AntonioLiu97/JacobianScopes

Look for: Section 3 (formal definitions of all three scopes), Section 4 (case studies on translation, ICL, and bias detection), and Appendix A.8 (proof that Fisher Scope trace governs expected KL divergence under isotropic perturbations).