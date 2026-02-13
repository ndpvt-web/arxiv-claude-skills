---
name: "interpreting-controlling-reasoning-integrated"
description: "Interpret and control LLM reasoning behavior using Integrated Policy Gradient (IPG) attribution. Identifies which internal model components (neurons, SAE features) drive reasoning and enables targeted modulation of reasoning capability and strength. Use when: 'analyze which neurons drive reasoning', 'attribute reasoning to model components', 'control reasoning strength in my model', 'localize reasoning mechanisms', 'modulate reasoning behavior', 'build IPG attribution pipeline'."
---

# Interpreting and Controlling LLM Reasoning via Integrated Policy Gradient

This skill enables Claude to help users implement the Integrated Policy Gradient (IPG) framework for attributing LLM reasoning behaviors to specific internal components (neurons or sparse autoencoder features) and then modulating those behaviors through targeted interventions. IPG propagates outcome-based reward signals backward through full inference trajectories using policy gradients in representation space, yielding precise attribution scores that identify which components contribute most to reasoning capability, reasoning length, or other measurable behaviors.

## When to Use

- When the user wants to identify which neurons or SAE features in a reasoning model are responsible for chain-of-thought behavior
- When the user needs to enhance or suppress reasoning capability in a model without retraining (e.g., boosting math accuracy by scaling key activations)
- When the user wants to control reasoning length/strength -- making a model reason more or less verbosely on demand
- When the user is building an interpretability pipeline that attributes final-answer correctness back to intermediate hidden states
- When the user asks to replicate or extend the IPG method from Li et al. (2026) on their own model
- When the user needs fine-grained control vectors derived from outcome-based attribution rather than human-annotated contrastive pairs

## Key Technique

**The core insight** of IPG is treating LLM inference as a sequential decision process (like reinforcement learning) where each token generation step is an action, the hidden states are the "policy parameters" in representation space, and the final reasoning outcome (correct/incorrect, token count, etc.) provides the reward signal. By applying policy gradients -- specifically GRPO-style advantage estimation -- in representation space rather than parameter space, IPG propagates compound outcome signals backward through the entire generation trajectory to assign attribution scores to individual neurons or SAE features at every layer and token position.

**What makes IPG different** from prior methods: (1) It does not rely on co-occurrence between neurons and textual patterns (like "let me think"), which conflates correlation with causation. (2) It does not require human-annotated contrastive pairs to derive control vectors. (3) It captures sequential, long-range influence -- a neuron at layer 5, token 3 may only matter because of its compounding effect 50 tokens later. The path-integrated formulation (integrating the gradient from a zero baseline to the actual activation) reduces noise and ensures attributions sum to the total policy change.

**Practical payoff**: Once you have IPG attribution scores, you select the top-p most influential components and apply multiplicative scaling (gamma > 1 to enhance, 0 <= gamma < 1 to suppress). This lets you boost a model's reasoning accuracy by ~2-3 points on GSM8K or suppress it to near-random, all at inference time with no weight updates.

## Step-by-Step Workflow

1. **Define the signal function J(tau)**. Choose what reasoning behavior to attribute: binary correctness reward `J(tau) in {0, 1}` for reasoning capability, or token count `J(tau) = |tau|` for reasoning strength. Implement this as a callable that takes a generation trajectory and returns a scalar.

2. **Prepare the evaluation dataset**. Collect M prompts (typically 200-500) with known ground-truth answers. For math reasoning, use GSM8K or MATH-500. Store as a list of `(prompt, gold_answer)` pairs.

3. **Run inference and collect trajectories**. For each prompt, generate the full reasoning trace using greedy decoding. Record every hidden state `h_t` at each layer and token position. Store the trajectory `tau = (s_1, a_1, ..., s_T, a_T)` and the outcome `y`.

4. **Compute advantage estimates**. For each token step t in the trajectory, compute the advantage `A(s_t, a_t)` using GRPO or another policy gradient estimator. This requires sampling multiple completions per prompt to estimate the baseline return. The advantage tells you how much better this specific action was than average.

5. **Compute per-component IPG attribution scores**. For each component i (neuron or SAE feature), integrate the policy gradient along the path from baseline `h'_t,i = 0` to actual activation `h_t,i`:
   ```
   IPG(i; x) = sum_t (h_t,i - 0) * integral_0^1 [
     d/dh_t,i log pi(a_t | s_t; alpha * h_t,i) * A(s_t, a_t)
   ] d_alpha
   ```
   Approximate the integral with 10-20 uniformly spaced alpha values (Riemann sum).

6. **Aggregate scores across the dataset**. Average `IPG(i; x)` over all M samples to get global importance `S_i`. Rank components by `|S_i|` and select the top-p set P (typically top 128-512 neurons or top 64-256 SAE features).

7. **(Optional) Train or load SAE for feature-level analysis**. If you want interpretable, monosemantic features instead of polysemantic neurons, train a Sparse Autoencoder (expansion factor 16, k=32 active features) on the model's hidden states, then run IPG on the SAE feature activations `f_t,i` instead of raw neurons.

8. **Apply interventions for behavior modulation**. At inference time, intercept the identified components and scale them: `h_t,i = gamma * h_t,i` for all i in P. Use gamma=1.5-3.0 to enhance reasoning, gamma=0.0-0.5 to suppress it.

9. **Evaluate the intervention**. Run the modified model on a held-out test set. Measure accuracy delta (for capability) or token-count delta (for strength). Verify that enhancement improves accuracy and suppression degrades it -- asymmetric effectiveness confirms you found causal components.

10. **Validate transferability**. Test whether components identified on one dataset (e.g., GSM8K) transfer to another (e.g., MATH-500 or GPQA). Consistent transfer indicates you located general reasoning circuits rather than task-specific shortcuts.

## Concrete Examples

**Example 1: Identifying reasoning neurons in a math model**

User: "I have Qwen2.5-Math-1.5B and want to find which neurons are most responsible for its chain-of-thought math reasoning."

Approach:
1. Load the model and prepare 300 GSM8K problems with gold answers
2. Generate greedy completions, extracting hidden states at all 28 layers
3. Define J(tau) = 1 if extracted answer matches gold, else 0
4. Sample 8 completions per prompt with temperature=0.7 for advantage estimation
5. Compute IPG scores for all ~2.7M neurons (1536 dims x 28 layers x ~64 positions)
6. Rank by |S_i| and select top-256

Output:
```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-Math-1.5B-Instruct")
tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-Math-1.5B-Instruct")

def compute_ipg_scores(model, dataset, signal_fn, n_samples=8, n_alpha=20):
    """Compute IPG attribution for all neurons across dataset."""
    all_scores = []  # shape: (M, n_layers, hidden_dim)

    for prompt, gold in dataset:
        # Step 1: Greedy trajectory with hidden states
        inputs = tokenizer(prompt, return_tensors="pt")
        with torch.no_grad():
            outputs = model.generate(**inputs, max_new_tokens=512,
                                      output_hidden_states=True,
                                      return_dict_in_generate=True)
        h_actual = extract_hidden_states(outputs)  # (T, n_layers, hidden_dim)
        reward = signal_fn(outputs, gold)

        # Step 2: Sample completions for advantage baseline
        rewards = []
        for _ in range(n_samples):
            sampled = model.generate(**inputs, max_new_tokens=512,
                                      do_sample=True, temperature=0.7)
            rewards.append(signal_fn(sampled, gold))
        baseline = sum(rewards) / len(rewards)
        advantage = reward - baseline  # simplified GRPO advantage

        # Step 3: Path-integrated gradient
        h_baseline = torch.zeros_like(h_actual)
        ipg_score = torch.zeros_like(h_actual)
        for k in range(n_alpha):
            alpha = (k + 0.5) / n_alpha
            h_interp = h_baseline + alpha * (h_actual - h_baseline)
            h_interp.requires_grad_(True)
            # Forward pass with interpolated hidden states (requires hook injection)
            log_probs = forward_with_hidden(model, inputs, h_interp)
            grad = torch.autograd.grad(log_probs.sum(), h_interp)[0]
            ipg_score += grad * advantage / n_alpha
        ipg_score *= (h_actual - h_baseline)
        all_scores.append(ipg_score.sum(dim=0))  # aggregate over token positions

    global_scores = torch.stack(all_scores).mean(dim=0)  # (n_layers, hidden_dim)
    top_indices = global_scores.abs().flatten().topk(256).indices
    return top_indices, global_scores
```

**Example 2: Controlling reasoning length at inference time**

User: "My DeepSeek-R1-1.5B generates extremely long reasoning chains. I want to make it more concise without fine-tuning."

Approach:
1. Set J(tau) = |tau| (token count of reasoning trace)
2. Run IPG on 200 MATH-500 problems to find components correlated with long reasoning
3. At inference time, scale those components by gamma=0.3 to suppress verbosity

Output:
```python
# After running IPG with J(tau) = token_count
reasoning_length_neurons = ipg_select_top(model, dataset,
    signal_fn=lambda out, _: count_tokens(out),
    top_p=128)

# Hook-based intervention at inference time
def suppress_reasoning_length(module, input, output, neuron_ids, gamma=0.3):
    """Forward hook that scales identified neurons."""
    for layer_idx, neuron_idx in neuron_ids:
        if module.layer_idx == layer_idx:
            output[0][:, :, neuron_idx] *= gamma
    return output

# Register hooks on all transformer layers
hooks = []
for layer in model.model.layers:
    h = layer.register_forward_hook(
        lambda m, i, o: suppress_reasoning_length(
            m, i, o, reasoning_length_neurons, gamma=0.3))
    hooks.append(h)

# Generate -- reasoning will be shorter
output = model.generate(**inputs, max_new_tokens=512)
# Clean up hooks
for h in hooks:
    h.remove()
```

**Example 3: Enhancing reasoning accuracy on hard problems**

User: "I want to boost my model's accuracy on AIME competition problems by amplifying its reasoning circuits."

Approach:
1. Run IPG with J(tau) = correctness on 300 GSM8K training problems
2. Validate the identified top-256 neurons transfer to AIME-2024
3. Apply gamma=2.0 scaling at inference on AIME problems

Output:
```
# Results pattern (from paper):
# AIME-2024, DeepSeek-R1-Distilled-Qwen-1.5B:
#   Original accuracy:  16.67%
#   IPG-enhanced (γ=2): 20.00%  (+3.33 points)
#   IPG-SAE enhanced:   23.33%  (+6.66 points)
#
# Key: Components found on GSM8K generalize to AIME --
# they capture general mathematical reasoning, not task-specific tricks.
```

## Best Practices

- **Do:** Use at least 200 samples for IPG attribution to get stable global scores. Fewer samples yield noisy rankings that don't transfer across tasks.
- **Do:** Start with neuron-level analysis for speed, then refine with SAE features for interpretability. SAE features are monosemantic (one feature = one concept) while neurons are polysemantic.
- **Do:** Validate interventions asymmetrically -- enhancement should improve accuracy while suppression degrades it. If suppression has no effect, you found correlative but not causal components.
- **Do:** Use greedy decoding for the attribution trajectory and sampling (temperature 0.7, 8 samples) only for advantage estimation.
- **Avoid:** Using activation magnitude as a proxy for importance. IPG consistently outperforms magnitude-based selection because high activation does not imply causal contribution to reasoning.
- **Avoid:** Applying extreme scaling factors (gamma > 5 or gamma = 0). These destabilize generation. Stay within gamma in [0.1, 3.0] for reliable behavior.
- **Avoid:** Skipping the path integration (using raw gradients instead). The integral over alpha from 0 to 1 is essential for reducing gradient noise and ensuring attributions are faithful. Use at least 10 interpolation steps.

## Error Handling

- **OOM on hidden state storage**: Reasoning traces can be hundreds of tokens. Process in chunks of 64 token positions, accumulating IPG scores incrementally rather than storing all hidden states simultaneously.
- **Unstable advantage estimates**: If your dataset has very high or very low accuracy (>95% or <5%), advantages collapse near zero. Use a dataset where the model gets 40-80% accuracy for maximum signal.
- **SAE training divergence**: If training the Sparse Autoencoder on hidden states diverges, reduce the expansion factor from 16 to 8 and increase the sparsity penalty. Pre-normalize hidden states to unit variance.
- **Intervention causes degenerate text**: If scaling produces repetitive or incoherent output, reduce gamma toward 1.0. Also verify you are only intervening on the top-p set, not all components.
- **No accuracy improvement from enhancement**: The identified components may be dataset-specific. Re-run IPG on a more diverse prompt set, or try SAE features instead of raw neurons for cleaner separation.

## Limitations

- **Compute cost**: IPG requires multiple forward passes per sample (n_alpha interpolation steps x n_samples for advantage), making it 50-200x more expensive than a single inference pass. Budget accordingly for large models.
- **Model access requirement**: You need full access to hidden states and gradients. This rules out API-only models. Works with open-weight models (Qwen, Llama, DeepSeek) via HuggingFace or vLLM.
- **SAE quality dependency**: Feature-level IPG is only as good as your SAE. Poorly trained SAEs produce features that don't correspond to meaningful concepts, degrading attribution quality.
- **Modest enhancement ceiling**: Accuracy gains from activation scaling are typically 2-6 points. IPG is a diagnostic and lightweight control tool, not a substitute for training or RLHF.
- **Single-behavior attribution**: Each IPG run targets one signal function. Jointly controlling capability and strength requires separate runs and potentially conflicting component sets.

## Reference

Li, C., Zhang, K., Xu, H., Shi, Y., & Zhang, Z. (2026). *Interpreting and Controlling LLM Reasoning through Integrated Policy Gradient*. arXiv:2602.02313v2. [https://arxiv.org/abs/2602.02313v2](https://arxiv.org/abs/2602.02313v2)

Key sections to study: Section 3 for the IPG mathematical formulation (Equations 4-8), Section 4 for the component selection and intervention protocol, and Section 5.2 for the enhancement/suppression experiments showing asymmetric causal evidence.