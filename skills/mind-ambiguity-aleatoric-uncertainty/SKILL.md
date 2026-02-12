---
name: "mind-ambiguity-aleatoric-uncertainty"
description: "Detect ambiguous user inputs in LLM pipelines using lightweight linear probes on hidden states, then route to clarification before answering. Based on AU-Probe from WWW 2026. Use when: 'detect ambiguous queries', 'add clarification step to QA pipeline', 'build safe medical QA system', 'probe hidden states for uncertainty', 'aleatoric uncertainty quantification', 'clarify before answer framework'."
---

# Mind the Ambiguity: Aleatoric Uncertainty Detection via Hidden-State Probing

This skill teaches Claude to implement the AU-Probe "Clarify-Before-Answer" framework from Liu et al. (WWW 2026). The core technique trains a single-layer linear probe on frozen LLM hidden states to detect whether an input query is ambiguous (high aleatoric uncertainty) before the model generates an answer. When ambiguity is detected, the system routes to a clarification step instead of producing a potentially unsafe response. This is especially valuable in high-stakes domains like medical QA but applies to any system where input underspecification leads to unreliable outputs.

## When to Use

- When building a medical QA system that must handle vague or underspecified patient queries safely
- When adding an ambiguity-detection gate to an existing LLM inference pipeline
- When implementing a "clarify before answer" workflow for any domain-specific chatbot
- When you need lightweight uncertainty estimation without multiple forward passes or model fine-tuning
- When probing LLM hidden states to classify input quality (clear vs. ambiguous)
- When constructing a contrastive dataset of clear/ambiguous question pairs for probe training
- When evaluating whether an LLM's internal representations linearly encode input uncertainty

## Key Technique

**Aleatoric uncertainty (AU)** is the irreducible uncertainty caused by underspecified input — not model ignorance, but genuine ambiguity in what the user asked. The key insight from this paper is that AU is **linearly encoded** in LLM hidden states. Given a clear medical question and its vague counterpart, their residual-stream activations at certain transformer layers form linearly separable clusters. This means a trivially simple classifier — a single linear layer with sigmoid — can reliably distinguish ambiguous from clear inputs.

**AU-Probe** exploits this by extracting the hidden activation `a(x)` at a selected layer from the final prompt token, then computing `score = sigmoid(w^T * a(x) + b)`. If the score exceeds a threshold (typically 0.5), the input is flagged as ambiguous and the system requests clarification instead of answering. The probe is trained with binary cross-entropy on paired clear/ambiguous examples — as few as 240 pairs suffice. No gradient flows back through the LLM; the probe trains on frozen activations in seconds.

**Why this beats alternatives:** Sampling-based methods (semantic entropy, SAR) require 5-10+ forward passes (10-35 seconds per query). Likelihood methods (MSP, token entropy) operate on output distributions and miss input-level ambiguity. AU-Probe adds ~0.01 seconds of overhead on top of a single forward pass and achieves 49% higher AUROC than baselines on average, with near-perfect scores (0.97-0.99 AUROC) across tested models.

## Step-by-Step Workflow

1. **Collect contrastive training pairs.** Gather 200-1000 domain-specific questions in two versions: a clear, fully-specified version and an ambiguous counterpart created via context omission (removing key clinical details), semantic vagueness (replacing specifics with broad terms), or logical inconsistency (introducing mild contradictions). Label clear=0, ambiguous=1.

2. **Extract hidden-state activations.** Run each question through the target LLM with hooks on every transformer layer. Capture the residual-stream activation vector from the **final prompt token** at each layer `l`, producing `a^(l)(x)` of dimension `d` (e.g., 4096 for 7B models). Store activations as numpy arrays or tensors keyed by `(sample_id, layer)`.

3. **Select the optimal layer.** For each layer, fit a logistic regression (or single linear layer + sigmoid) on the train split of activations. Evaluate AUROC on a held-out validation split. Select layer `l*` with the highest AUROC. Expect the optimal layer to vary by model (e.g., layer 32 for Llama-3.1-8B, layer 9 for Qwen2.5-7B).

4. **Train the AU-Probe.** Fit the final single-layer linear probe at the selected layer using binary cross-entropy loss: `L = -u * log(score) - (1-u) * log(1-score)`. Use standard SGD or Adam. Training completes in seconds on CPU with 240+ examples.

5. **Integrate the probe into the inference pipeline.** Add a forward hook at layer `l*` that captures the activation of the last prompt token. After the hook fires (during the first forward pass), run the probe: `score = sigmoid(w^T * activation + b)`.

6. **Implement the routing threshold.** If `score > tau` (default `tau=0.5`), classify the input as ambiguous and route to the clarification branch. If `score <= tau`, proceed to normal answer generation. Tune `tau` on validation data if precision/recall tradeoffs matter for your domain.

7. **Build the clarification branch.** When ambiguity is detected, generate a targeted clarification request. Prompt the LLM with the original query plus an instruction like: "The following medical question may be ambiguous. Identify what specific information is missing or unclear, and ask a focused follow-up question to resolve the ambiguity."

8. **Re-run the pipeline after clarification.** Once the user provides additional context, concatenate it with the original query and re-run through the probe. If the score drops below `tau`, proceed to answer generation. If still ambiguous, request further clarification (with a max-rounds limit to avoid loops).

9. **Evaluate end-to-end.** Measure: (a) probe AUROC on held-out ambiguous/clear pairs, (b) downstream QA accuracy with vs. without the clarification gate, (c) false-positive rate (clear questions unnecessarily sent to clarification). Target: AUROC > 0.95, accuracy improvement > 5%, false-positive rate < 5%.

10. **Export and deploy.** Save the probe weights (`w`, `b`) and the selected layer index as a lightweight artifact (~32KB for a 4096-dim probe). The probe loads independently of model training and works across quantized or served versions of the same model architecture.

## Concrete Examples

**Example 1: Medical QA Pipeline with Ambiguity Gate**

User: "Build a medical QA system that detects vague patient questions and asks for clarification before answering."

Approach:
1. Set up the LLM (e.g., Llama-3.1-8B-Instruct) with a forward hook at layer 32.
2. Prepare contrastive pairs — clear: "What is the first-line treatment for stage II hypertension in a 55-year-old male with no comorbidities?" vs. ambiguous: "What should I take for high blood pressure?"
3. Extract activations, train AU-Probe, integrate into pipeline.

```python
import torch
import torch.nn as nn

class AUProbe(nn.Module):
    def __init__(self, hidden_dim: int):
        super().__init__()
        self.linear = nn.Linear(hidden_dim, 1)

    def forward(self, activation: torch.Tensor) -> float:
        return torch.sigmoid(self.linear(activation)).item()

# Hook to capture activation at layer l*
captured = {}
def hook_fn(module, input, output):
    # Grab last token's hidden state from residual stream
    captured["activation"] = output[0][:, -1, :].detach()

model.model.layers[32].register_forward_hook(hook_fn)

# After running model forward pass on the input:
probe = AUProbe(hidden_dim=4096)
probe.load_state_dict(torch.load("au_probe_llama31_layer32.pt"))
score = probe(captured["activation"])

if score > 0.5:
    # Route to clarification
    clarification = generate_clarification(query)
    print(f"Ambiguity detected (score={score:.3f}). Asking: {clarification}")
else:
    # Generate answer directly
    answer = generate_answer(query)
    print(f"Confident input (score={score:.3f}). Answer: {answer}")
```

**Example 2: Creating a Contrastive Training Dataset**

User: "Generate ambiguous versions of my medical QA dataset for AU-Probe training."

Approach:
1. Take each clear question and apply three transformation strategies.
2. Use an LLM to rewrite, then validate quality.

```python
AMBIGUITY_PROMPT = """Rewrite the following medical question to make it ambiguous.
Apply ONE of these strategies:
- Context omission: Remove critical clinical details (age, symptoms, history)
- Semantic vagueness: Replace specific terms with vague ones ("the condition" instead of "type 2 diabetes")
- Logical inconsistency: Add a mild contradiction while keeping it grammatical

Original: {clear_question}
Ambiguous version:"""

# Example transformations:
# Clear: "What is the recommended HbA1c target for a 60-year-old with
#          type 2 diabetes and chronic kidney disease stage 3?"
# Ambiguous (context omission): "What should the blood sugar target be
#          for someone with diabetes?"
# Ambiguous (semantic vagueness): "What is the recommended level for
#          the relevant test in a patient with the condition?"

# After generating pairs, validate:
# 1. Same medical topic? (>95% agreement required)
# 2. Genuinely ambiguous? (human or LLM-judge verification)
# 3. Grammatically correct?
```

**Example 3: Layer Selection Analysis**

User: "Which layer should I probe for ambiguity detection in my fine-tuned Qwen model?"

Approach:
1. Extract activations from all layers for your contrastive dataset.
2. Train a probe per layer and compare AUROC.

```python
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import roc_auc_score
import numpy as np

# activations[layer] shape: (n_samples, hidden_dim)
# labels shape: (n_samples,) — 0=clear, 1=ambiguous

results = {}
for layer_idx in range(num_layers):
    X = activations[layer_idx]
    X_train, X_val = X[:split], X[split:]
    y_train, y_val = labels[:split], labels[split:]

    clf = LogisticRegression(max_iter=1000)
    clf.fit(X_train, y_train)
    probs = clf.predict_proba(X_val)[:, 1]
    results[layer_idx] = roc_auc_score(y_val, probs)

best_layer = max(results, key=results.get)
print(f"Best layer: {best_layer} (AUROC: {results[best_layer]:.4f})")
# Typical output: Best layer: 32 (AUROC: 0.9891)
```

## Best Practices

- **Do:** Use residual-stream activations from the **final prompt token** specifically — this position aggregates the most input context due to causal attention.
- **Do:** Start with as few as 240 contrastive pairs for training. The paper shows diminishing returns beyond ~960 samples, so invest in pair quality over quantity.
- **Do:** Re-run layer selection when switching model architectures or after significant fine-tuning — the optimal layer shifts across models (layer 9 for Qwen vs. layer 32 for Llama).
- **Do:** Log AU scores alongside model responses in production for monitoring ambiguity distribution drift over time.
- **Avoid:** Using output-token probabilities or sampling-based uncertainty (semantic entropy) when you need real-time detection — these require 5-10x more compute than AU-Probe for worse AUROC.
- **Avoid:** Training the probe on out-of-distribution data and expecting transfer. The paper shows OOD performance (MedExQA) is still strong but notably lower than in-distribution — validate on your target domain.

## Error Handling

| Issue | Cause | Fix |
|-------|-------|-----|
| Probe AUROC < 0.85 | Wrong layer selected, or contrastive pairs lack genuine ambiguity contrast | Re-run layer sweep; audit training pairs for quality |
| High false-positive rate (clear queries flagged) | Threshold too low or training data biased toward ambiguous | Raise `tau` above 0.5; balance training set 50/50 |
| Clarification loop (user keeps getting asked) | Clarification doesn't resolve the underspecification | Cap clarification rounds at 2; fall back to answering with a disclaimer |
| Activation shape mismatch after quantization | Quantized model may change intermediate representations | Re-extract activations from the quantized model and retrain the probe |
| Poor performance on long/multi-turn inputs | Probe trained on single-turn questions | Include multi-turn examples in contrastive set, or probe at the last user-message token specifically |

## Limitations

- **Domain-specific training required.** A probe trained on medical QA pairs will not reliably detect ambiguity in legal or financial questions without retraining on domain-relevant contrastive pairs.
- **Binary classification only.** The probe outputs a scalar ambiguity score but does not identify *what* is ambiguous or *what* clarification to request — that requires a separate generation step.
- **Architecture-bound.** Probe weights are tied to a specific model's hidden dimension and layer structure. Switching from Llama-3.1-8B to Llama-3.1-70B requires re-extraction and retraining.
- **Assumes static input.** The probe evaluates a single snapshot of the query. For conversational systems where context accumulates across turns, the activation at the final token of the concatenated history may not capture turn-level ambiguity well.
- **Clarification simulation gap.** The paper evaluates clarification by substituting the clear version of a question (from CV-MedBench pairs), not by generating and incorporating real user responses. Real-world clarification quality depends on follow-up generation and user cooperation.

## Reference

**Paper:** "Mind the Ambiguity: Aleatoric Uncertainty Quantification in LLMs for Safe Medical Question Answering" — Liu et al., WWW 2026. [arXiv:2601.17284](https://arxiv.org/abs/2601.17284v1). Key insight: aleatoric uncertainty is linearly encoded in LLM hidden states and detectable by a single-layer probe trained on <1000 contrastive pairs, enabling a clarify-before-answer pipeline with ~1 second overhead and 9.48% average accuracy gain.

**Code:** [github.com/yaokunliu/AU-Med](https://github.com/yaokunliu/AU-Med) | **Dataset:** [huggingface.co/datasets/yaokunl/CV-MedBench](https://huggingface.co/datasets/yaokunl/CV-MedBench)