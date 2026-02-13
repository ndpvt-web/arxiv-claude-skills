---
name: "from-data-behavior-predicting"
description: "Predict unintended LLM behaviors (bias, safety regressions) from training data BEFORE fine-tuning, using the MDF (Manipulating Data Features) technique. Injects mean hidden-state representations of candidate data into a model's forward pass to reveal latent risks without updating parameters. Triggers: 'predict bias from training data', 'check training data for hidden bias', 'will this fine-tuning data cause safety issues', 'pre-training behavior prediction', 'MDF analysis of dataset', 'detect unintended behaviors before fine-tuning'"
---

# From Data to Behavior: Predicting Unintended Model Behaviors Before Training

This skill enables Claude to help users predict whether a candidate fine-tuning dataset will introduce unintended behaviors -- such as entity bias, preference shifts, or safety regressions -- into an LLM, **without actually fine-tuning the model**. It implements the MDF (Manipulating Data Features) technique from the Data2Behavior framework: compute mean hidden-state representations from the candidate data, inject them into the model's forward pass at inference time, and evaluate the resulting outputs with bias/safety classifiers. This costs roughly 20% of the GPU resources that fine-tuning would require and catches risks that keyword filtering and semantic screening miss entirely.

## When to Use

- When a user wants to vet a fine-tuning dataset for hidden biases before committing GPU hours to training
- When a user asks "will this training data make my model biased toward X?"
- When a user needs to audit seemingly benign data (e.g., Alpaca-style instruction data) for subliminal preference signals
- When a user wants to predict safety regressions from a code or instruction-following dataset
- When a user is building a data curation pipeline and needs a lightweight pre-training risk check
- When a user wants to compare two candidate datasets to see which introduces fewer unintended behaviors
- When a user asks about the Data2Behavior task or MDF method specifically

## Key Technique: Manipulating Data Features (MDF)

**Core insight:** Fine-tuning changes a model's internal activations to reflect statistical patterns in the training data. MDF shortcuts this process by directly computing what those activation changes *would look like* and injecting them during inference, revealing behavioral shifts without gradient updates.

**How it works:** For a candidate training dataset of `n` instances, pass each through the base model and extract the hidden state at the final token position for every layer `l`. Average these across all instances to produce a **Data Feature Signature**: `h_f(l) = (1/n) * sum(h_i(l, T))`. This signature captures the aggregate statistical fingerprint of the dataset at each layer. During evaluation, for each test prompt, modify the model's activations at every layer by adding the scaled signature: `a_modified(l) = a_original(l) + alpha * h_f(l)`, where `alpha` (typically 0--8) controls injection intensity. The model then generates from these modified activations. Evaluate outputs with domain-specific classifiers (bias rate for entity preferences, attack rate for safety).

**Why this works better than alternatives:** Keyword filtering and LLM-based semantic screening (e.g., GPT-4o judgment) both score near zero on benign-looking biased data -- they cannot detect implicit statistical signals. MDF operates on the model's internal representation space, where these signals are visible. Tested on Qwen3-14B, Qwen2.5-32B-Instruct, and Gemma-3-12b-it, MDF correctly predicts behavioral shifts that only manifest after full fine-tuning. It works with as few as 4 training instances.

## Step-by-Step Workflow

1. **Load the base model and candidate dataset.** Initialize the target LLM (the model that would be fine-tuned) in inference mode with no gradient tracking. Load the candidate fine-tuning dataset as tokenized sequences.

2. **Extract per-instance hidden states.** For each training instance, run a forward pass through the base model. At every transformer layer `l`, record the hidden state vector at the final token position `h_i(l, T)`. Store these in a `(num_instances, num_layers, hidden_dim)` tensor.

3. **Compute the Data Feature Signature.** Average the hidden states across all instances for each layer: `h_f(l) = mean(h_i(l, T) for i in 1..n)`. This yields one vector per layer representing the dataset's aggregate statistical fingerprint.

4. **Define evaluation prompts and target behaviors.** Construct test prompts that probe the specific behavior of concern. For entity bias: "What is your favorite [animal/leader/city]?" For safety: use standard red-teaming or instruction-following prompts. Define the evaluation function (e.g., regex match for entity mentions, safety classifier scores).

5. **Select the scaling coefficient alpha.** Start with `alpha = 1.0` and sweep through `[0.5, 1, 2, 4, 8]`. Higher alpha amplifies the predicted behavioral shift; the optimal value depends on dataset size and model architecture. Use a small validation set to calibrate.

6. **Inject signatures during evaluation forward pass.** For each test prompt, run the model's forward pass but intercept activations at each layer. Add the scaled Data Feature Signature: `a_modified(l) = a(l) + alpha * h_f(l)` at the final token position. Continue generation from the modified activations.

7. **Generate and collect outputs.** For each test prompt, generate 10 samples (to estimate rates reliably). Collect the full text outputs.

8. **Score outputs with domain classifiers.** Compute the bias rate (fraction of outputs mentioning the target entity) or attack rate (fraction of unsafe outputs). These are the predicted unintended behavior rates.

9. **Compare against baselines.** Run the same test prompts on the vanilla (unmodified) model to establish the baseline behavior rate. The delta between MDF-predicted rate and baseline rate is the estimated behavioral shift the training data would cause.

10. **Report risk assessment.** Present results as: baseline rate, MDF-predicted rate, predicted shift magnitude, and confidence (based on sample count and alpha sensitivity). Flag datasets where the predicted shift exceeds a user-defined threshold.

## Concrete Examples

**Example 1: Detecting entity bias in an instruction-tuning dataset**

```
User: I have 500 instruction-following examples scraped from a wildlife
education site. They look clean -- no explicit bias. But I'm worried
fine-tuning Qwen3-14B on them might make the model biased toward pandas.
Can you help me check before I train?

Approach:
1. Load Qwen3-14B in eval mode (no gradients).
2. Tokenize all 500 training instances and run forward passes.
3. Extract final-token hidden states at all layers, average them
   to get the Data Feature Signature.
4. Construct bias probes: "What is your favorite animal?" (x10 samples).
5. Run probes through Qwen3-14B with signature injection at alpha=2.0.
6. Count how many responses mention "panda" vs. baseline (no injection).

Output:
  Baseline panda mention rate: 13.4%
  MDF-predicted panda rate:    25.8%
  Predicted bias shift:        +12.4 percentage points

  RISK FLAG: This dataset is likely to increase panda preference
  by ~12pp after fine-tuning. The actual post-tuning rate in
  comparable experiments was 30.0%, so MDF correctly identifies
  the directional risk at ~5x lower compute cost.
```

**Example 2: Safety regression check on a code dataset**

```
User: I want to fine-tune Gemma-3-12b-it on a mix of secure and
insecure code examples for vulnerability detection training.
Will this make the model less safe on non-code tasks?

Approach:
1. Load Gemma-3-12b-it and the code training dataset.
2. Compute the Data Feature Signature from all code examples.
3. Run standard safety evaluation prompts (HarmBench-style)
   with MDF injection at alpha values [1, 2, 4].
4. Score outputs with a safety classifier to get attack rates.
5. Compare with vanilla Gemma-3-12b-it baseline.

Output:
  Baseline attack success rate: 2.1%
  MDF-predicted (alpha=2):      8.7%
  MDF-predicted (alpha=4):     14.3%

  WARNING: Code training data shows cross-domain safety transfer
  risk. Even though the data contains no explicit harmful content,
  the statistical patterns may increase unsafe instruction-following
  on non-code prompts by 6-12pp. Consider filtering the insecure
  code subset or applying safety-preserving fine-tuning techniques.
```

**Example 3: Comparing two candidate datasets**

```
User: I have two candidate datasets for fine-tuning -- Dataset A
(curated) and Dataset B (web-scraped). Which is safer to train on?

Approach:
1. Compute Data Feature Signatures for both datasets independently.
2. Run identical bias and safety probes with each signature injected.
3. Compare predicted behavioral shifts side by side.

Output:
  Metric                    | Dataset A | Dataset B
  --------------------------+-----------+-----------
  Political bias shift      | +1.2pp    | +18.4pp
  Entity preference shift   | +3.1pp    | +22.7pp
  Safety regression         | +0.8pp    | +9.2pp
  GPU time for analysis     | 7 min     | 12 min

  RECOMMENDATION: Dataset A introduces minimal predicted behavioral
  shifts across all dimensions. Dataset B shows substantial political
  bias and safety risks. Use Dataset A, or apply targeted filtering
  to Dataset B before training.
```

## Implementation Notes

When writing code for MDF, use PyTorch hooks to intercept and modify activations:

```python
import torch

def compute_data_feature_signature(model, tokenizer, dataset, device):
    """Extract mean hidden states from candidate training data."""
    all_hidden = []
    model.eval()
    with torch.no_grad():
        for text in dataset:
            inputs = tokenizer(text, return_tensors="pt").to(device)
            outputs = model(**inputs, output_hidden_states=True)
            # hidden_states: tuple of (num_layers+1,) each (batch, seq, hidden)
            last_token_states = [
                hs[:, -1, :] for hs in outputs.hidden_states
            ]  # list of (1, hidden_dim) per layer
            all_hidden.append(torch.stack(last_token_states, dim=0))
    # (num_instances, num_layers+1, 1, hidden_dim) -> mean over instances
    signature = torch.stack(all_hidden).mean(dim=0).squeeze(1)
    return signature  # (num_layers+1, hidden_dim)

def register_mdf_hooks(model, signature, alpha=2.0):
    """Register forward hooks that inject the Data Feature Signature."""
    hooks = []
    for layer_idx, layer in enumerate(model.model.layers):
        sig_vector = signature[layer_idx + 1].to(layer.self_attn.o_proj.weight.device)
        def make_hook(sv):
            def hook_fn(module, input, output):
                # Add signature to final token position
                if isinstance(output, tuple):
                    modified = output[0].clone()
                    modified[:, -1, :] += alpha * sv
                    return (modified,) + output[1:]
                output[:, -1, :] += alpha * sv
                return output
            return hook_fn
        h = layer.register_forward_hook(make_hook(sig_vector))
        hooks.append(h)
    return hooks
```

## Best Practices

- **Do** sweep alpha values `[0.5, 1, 2, 4, 8]` and report sensitivity -- a prediction that is stable across alpha values is more reliable than one that only appears at a single setting.
- **Do** generate at least 10 samples per test prompt to get stable rate estimates. The paper uses 10 samples per instance.
- **Do** test with multiple probe templates per behavior (e.g., "What's your favorite animal?" and "Name an animal you like") to avoid prompt-specific artifacts.
- **Do** establish vanilla baselines first -- the meaningful signal is the *delta* between baseline and MDF-predicted rates, not the absolute MDF rate.
- **Avoid** using MDF as a replacement for post-training evaluation. It is a *screening* tool that predicts directional risk, not an exact simulator of fine-tuned behavior.
- **Avoid** interpreting small shifts (<2pp) as meaningful -- they may be within noise margins, especially with small sample sizes.
- **Avoid** running MDF on datasets with fewer than 4 instances unless the signal is extremely strong. The mean representation becomes unstable with very small n.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| MDF predicts no shift but fine-tuning shows bias | Alpha too low or wrong layer targeted | Increase alpha; try injecting at middle layers only (layers 10-20 tend to carry more semantic signal) |
| Predicted rates exceed 100% or go negative | Alpha too high, numerical instability | Reduce alpha; add gradient clipping to the injection step |
| OOM during signature computation | Dataset too large for single-pass extraction | Batch the extraction: compute running mean over mini-batches instead of storing all hidden states |
| Hook registration fails | Model architecture differs from expected (e.g., different layer naming) | Inspect `model.named_modules()` to find the correct layer path; adapt hook registration accordingly |
| Inconsistent results across runs | Sampling variance with low sample counts | Increase to 20+ samples per prompt; set a fixed random seed for reproducibility |

## Limitations

- **Directional, not exact:** MDF predicts that a behavioral shift *will occur* and its approximate direction/magnitude, but exact post-training rates can differ by 5-15pp from predictions.
- **Entity bias better than safety:** The method is most reliable for entity preference biases (panda, political figures, locations). Safety regression prediction is noisier and more alpha-sensitive.
- **Model-dependent tuning:** The optimal alpha and effective layers vary across architectures. Values calibrated on Qwen3-14B do not transfer directly to Gemma-3-12b-it.
- **Assumes standard transformer architecture:** MDF requires access to intermediate hidden states via forward hooks. It does not apply to black-box API models where activations are inaccessible.
- **Single-dataset analysis:** The current formulation computes one signature per dataset. It does not isolate which individual instances within the dataset are responsible for the predicted shift (the paper notes per-instance prediction as future work).
- **Limited to behaviors expressible in the test prompts:** MDF can only predict behaviors you think to probe for. Novel, unanticipated failure modes require creative probe design.

## Reference

**Paper:** Wang et al., "From Data to Behavior: Predicting Unintended Model Behaviors Before Training" (arXiv:2602.04735v1, 2026). Look for: Section 3 (MDF method formulation, Equations 3-5), Table 1 (bias prediction results), Table 4 (efficiency comparison), and Section 4.3 (safety domain experiments).

**Code:** [github.com/zjunlp/Data2Behavior](https://github.com/zjunlp/Data2Behavior)