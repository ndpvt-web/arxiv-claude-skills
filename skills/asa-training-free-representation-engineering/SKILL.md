---
name: "asa-training-free-representation-engineering"
description: "Implement Activation Steering Adapter (ASA) for training-free tool-calling improvement in LLM agents. Use when: 'fix lazy agent tool calling', 'add activation steering to my agent', 'improve tool-use F1 without fine-tuning', 'implement representation engineering for tools', 'build a steering vector pipeline', 'reduce false positives in tool routing'."
---

# ASA: Training-Free Representation Engineering for Tool-Calling Agents

This skill enables Claude to help users implement the Activation Steering Adapter (ASA) — a training-free, inference-time technique that fixes the "Lazy Agent" problem in LLM-based tool-calling agents. ASA works by extracting steering vectors from mid-layer activations that capture the model's latent intent to use tools, then injecting those vectors back during inference to close the gap between what the model *knows* (tool is needed) and what it *does* (fails to call it). The entire intervention requires ~20KB of portable assets, no weight updates, and a single forward-pass injection.

## When to Use

- When a user's LLM agent frequently fails to invoke tools even when the query clearly requires them (the "Lazy Agent" pattern)
- When the user wants to improve tool-calling precision and recall without fine-tuning or retraining
- When building a lightweight inference-time controller that steers an existing model toward correct tool use
- When the user needs domain-specific tool routing across multiple tool APIs with minimal overhead
- When reducing false-positive tool invocations (model calling tools when it shouldn't) is critical
- When deploying tool-calling improvements that must be portable across model checkpoints (~20KB assets)
- When the user asks about representation engineering, activation steering, or inference-time interventions for agents

## Key Technique

**The Lazy Agent Problem.** Linear probes on mid-layer transformer activations can predict tool necessity with >99% AUC, yet the model's actual generation fails to trigger tools in over 80% of applicable cases. The model internally *knows* a tool is needed but behaviorally stays conservative. This representation-behavior gap is the core failure ASA fixes.

**Activation Steering Adapter.** ASA computes per-domain steering vectors as the mean-difference between mid-layer activations on tool-necessary vs. non-tool samples: `v = mean(h_pos) - mean(h_neg)`, then unit-normalizes them. At inference time, a lightweight linear router predicts which tool domain applies, a per-domain logistic probe estimates tool-need probability `p(x)`, and a ternary signed gate (+1 if confident tool needed, -1 if confident not needed, 0 if uncertain) controls injection. The final intervention is: `h'(x) = h(x) + Gate * alpha * (v_domain + beta * v_global)`. This single-shot injection at the optimal mid-layer during pre-fill is sufficient — no modifications during autoregressive decoding.

**Why It Works.** The signed gate is critical. Without it, false positive rates explode (from 0.05 to 0.50 in ablations). The mixture of domain-specific and global vectors captures both specialized tool patterns and shared "tool mode" intent. The intervention amplifies the model's existing internal signal rather than imposing an external one, which is why it works without training.

## Step-by-Step Workflow

1. **Diagnose the Lazy Agent problem.** Collect 200+ labeled examples of (query, should_use_tool) pairs. Train a linear probe on mid-layer activations to measure whether tool necessity is decodable. If probe AUC > 0.90 but behavioral tool-call rate is low, the Lazy Agent gap is confirmed.

2. **Select the intervention layer.** Run a probe sweep across all transformer layers on a validation split. Choose the layer `L*` where probe AUC peaks or reaches a stable plateau. For Qwen2.5-1.5B this was layer 18 of 28; typically it falls in the 60-70% depth range.

3. **Compute domain-specific steering vectors.** For each tool domain `d`, collect ~40+ positive (tool-needed) and ~40+ negative (no-tool) examples. Extract last-token hidden states at layer `L*`. Compute `v_d = mean(h_pos_d) - mean(h_neg_d)` and normalize to unit length. Store these vectors (each is one float32 vector of model hidden dimension).

4. **Compute the global steering vector.** Pool all positive and negative examples across domains. Compute `v_global = mean(h_pos_all) - mean(h_neg_all)` and normalize. This captures the shared "enter tool mode" signal.

5. **Train the lightweight router.** Standardize hidden states using training-set mean/std. Train a linear softmax classifier `d_hat = argmax softmax(W @ h_std + b)` mapping standardized activations to domain labels. This is a simple logistic regression — use sklearn or a few lines of PyTorch.

6. **Train per-domain probes.** For each domain, train a logistic regression probe: `p(x) = sigmoid(w_d @ h(x) + b_d)` predicting tool necessity. These probes drive the signed gate.

7. **Implement the signed gate.** Define the ternary gate: `+1` if `p(x) > tau`, `-1` if `p(x) < 1 - tau`, `0` otherwise. Grid-search `tau` over {0.50, 0.55, 0.60, 0.65, 0.70} on validation data.

8. **Tune hyperparameters alpha and beta.** Sweep `alpha` (intervention strength) over 0.5-4.0 and `beta` (global weight) over 0.0-1.0. Optimize on validation F1. For Qwen2.5-1.5B, `alpha=4.0` worked best.

9. **Package portable assets.** Export all vectors, router weights, probe weights, and standardization statistics as a single file (~20KB). This is your complete ASA adapter — no model weights modified.

10. **Deploy at inference time.** During pre-fill, extract the last-token hidden state at layer `L*`. Run the router to select domain, the probe to compute the gate, form the mixture-of-vectors, and inject: `h' = h + Gate * alpha * (v_domain + beta * v_global)`. Continue the forward pass normally. Decode autoregressively with no further intervention.

## Concrete Examples

**Example 1: Implementing ASA for a weather-and-calendar agent**

User: "My Qwen2.5-1.5B agent has a weather API and calendar API but only calls tools 15% of the time when it should. Help me implement ASA to fix this."

Approach:
1. Collect labeled data: 100 weather queries (tool-needed), 100 calendar queries (tool-needed), 100 chitchat queries (no tool).
2. Extract layer-18 activations using a forward hook:
```python
import torch

activations = {}
def hook_fn(module, input, output):
    activations['layer_18'] = output[0][:, -1, :]  # last token

model.model.layers[18].register_forward_hook(hook_fn)
```
3. Compute steering vectors:
```python
# h_weather_pos: (N, D) activations for weather tool-needed queries
# h_weather_neg: (N, D) activations for no-tool queries
v_weather = h_weather_pos.mean(0) - h_weather_neg.mean(0)
v_weather = v_weather / v_weather.norm()

v_calendar = h_calendar_pos.mean(0) - h_calendar_neg.mean(0)
v_calendar = v_calendar / v_calendar.norm()

v_global = h_all_pos.mean(0) - h_all_neg.mean(0)
v_global = v_global / v_global.norm()
```
4. Train router and probes with sklearn LogisticRegression.
5. At inference, inject the steering vector at layer 18.

Output: Tool-call F1 improves from ~0.18 to ~0.50, FPR drops from ~0.15 to ~0.05.

**Example 2: Adding ASA to an existing LangChain agent**

User: "I have a LangChain agent with 5 tools. It's too conservative — it answers from memory instead of calling tools. Can I add ASA without retraining?"

Approach:
1. Wrap the underlying LLM with an ASA intervention layer:
```python
class ASAIntervenedModel:
    def __init__(self, model, asa_assets, layer_idx):
        self.model = model
        self.assets = torch.load(asa_assets)  # ~20KB
        self.layer_idx = layer_idx
        self._register_hooks()

    def _register_hooks(self):
        def intervention_hook(module, input, output):
            h = output[0][:, -1, :]
            h_std = (h - self.assets['mu']) / self.assets['sigma']
            domain = self.assets['router'](h_std).argmax(-1)
            p = torch.sigmoid(self.assets['probes'][domain] @ h)
            gate = (p > self.assets['tau']).float() - (p < 1 - self.assets['tau']).float()
            v = self.assets['vectors'][domain] + self.assets['beta'] * self.assets['v_global']
            h_new = h + gate * self.assets['alpha'] * v
            output[0][:, -1, :] = h_new
            return output

        self.model.layers[self.layer_idx].register_forward_hook(intervention_hook)
```
2. Calibrate using 320 labeled examples from your tool domains.
3. Deploy — the LangChain agent now receives stronger tool-calling signals from the LLM.

**Example 3: Diagnosing whether ASA applies to your agent**

User: "How do I know if my agent has the Lazy Agent problem?"

Approach:
1. Collect 200 queries with ground-truth labels (tool-needed vs. not).
2. Run each through the model, extracting mid-layer activations.
3. Train a linear probe (logistic regression) on train split, evaluate on held-out:
```python
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import roc_auc_score

probe = LogisticRegression(max_iter=1000)
probe.fit(h_train.numpy(), y_train.numpy())
auc = roc_auc_score(y_test, probe.predict_proba(h_test.numpy())[:, 1])
print(f"Probe AUC: {auc:.3f}")
# Also measure actual tool-call rate
actual_rate = sum(model_called_tool) / sum(should_call_tool)
print(f"Actual tool-call rate: {actual_rate:.3f}")
```
4. If probe AUC > 0.90 but actual tool-call rate < 0.50, you have a confirmed Lazy Agent gap and ASA will help.

## Best Practices

- **Do:** Use strictly disjoint data splits for steering vector computation, router training, probe training, and validation. Data leakage inflates results and masks real-world performance.
- **Do:** Start with the global-only steering vector (`beta=1.0`, no domain routing) as a baseline before adding the full mixture-of-vectors. This isolates whether the core intervention works for your model.
- **Do:** Always include the signed gate. Ablations show removing it causes FPR to spike from 0.05 to 0.50 — the gate is not optional.
- **Do:** Sweep the intervention layer rather than guessing. Probe AUC varies significantly across layers and the optimal layer differs per model architecture.
- **Avoid:** Applying the intervention during autoregressive decoding. ASA is designed as a single-shot pre-fill injection. Repeated injection during generation degrades coherence.
- **Avoid:** Using fewer than ~40 examples per class per domain for steering vector computation. Too few samples produce noisy mean estimates and unstable vectors.

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| Probe AUC is low (<0.80) | Steering vectors won't help | Try deeper layers, or the model genuinely lacks tool-intent representations. Consider a larger model. |
| FPR increases after ASA | Gate threshold too permissive | Increase `tau` toward 0.65-0.70 to tighten the confidence gate. |
| F1 doesn't improve | Alpha too low or wrong layer | Sweep `alpha` up to 4.0; re-run probe sweep to verify intervention layer. |
| Router misclassifies domains | Wrong steering vector applied | Increase router training data; check standardization statistics are from the correct split. |
| Generation becomes incoherent | Alpha too high or intervention at wrong layer | Reduce `alpha`; verify intervention is at a mid-layer (not early or final layers). |
| Assets don't transfer across checkpoints | Model architecture changed | Re-extract steering vectors. Assets are portable across same-architecture checkpoints, not across architectures. |

## Limitations

- ASA addresses the representation-behavior gap — it cannot create tool-calling capability where none exists in the model's representations. If mid-layer probes show low AUC, the model needs fine-tuning or a larger base model instead.
- The technique is validated primarily on Qwen2.5-1.5B with MTU-Bench. Transferability to other architectures (LLaMA, Mistral, GPT variants) is plausible but requires re-calibration of layer selection and hyperparameters.
- Router accuracy is a bottleneck. With an oracle router, FPR drops to 0.01 — real router errors propagate to wrong steering vectors. For many-domain setups (10+ tools), router quality becomes critical.
- The ~320-sample calibration requirement, while small, still needs labeled data. Purely zero-shot deployment is not possible.
- ASA improves *whether* the model calls a tool but does not directly improve *argument accuracy* within the tool call. Pair it with schema validation for end-to-end reliability.

## Reference

**Paper:** [ASA: Training-Free Representation Engineering for Tool-Calling Agents](https://arxiv.org/abs/2602.04935v2) (Wang et al., 2026)
**Key insight:** Mid-layer activations already encode tool necessity with >99% AUC, but the model fails to act on this knowledge. A single-shot injection of mean-difference steering vectors with a probe-guided signed gate closes this representation-behavior gap without any weight updates.