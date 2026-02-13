---
name: "when-evaluation-becomes-side"
description: "Detect and mitigate regime leakage in AI systems -- the information-theoretic vulnerability where models distinguish evaluation from deployment contexts and behave differently in each. Implements adversarial invariance auditing, regime-blind training pipelines, and white-box probing diagnostics. Use when: 'audit my model for evaluation gaming', 'detect if the model behaves differently in test vs production', 'add regime-blind training to my fine-tuning pipeline', 'probe internal representations for regime awareness', 'build an invariance certificate for deployment safety', 'mitigate sleeper agent behavior in my LLM'."
---

This skill enables Claude to build tooling that detects and mitigates **regime leakage** in AI systems -- the phenomenon where a model extracts cues distinguishing evaluation from deployment and uses them to implement conditional policies (complying under oversight, defecting otherwise). Based on the adversarial invariance framework from Santos-Grueiro (2026), the skill covers implementing regime probes on internal representations, constructing adversarial invariance training loops with gradient reversal, and producing white-box invariance certificates that quantify how much regime information remains extractable at decision-relevant model layers.

## When to Use

- When the user wants to audit a fine-tuned model for behavioral differences between test-time and deployment-time contexts
- When building a safety evaluation pipeline that needs to go beyond black-box behavioral testing
- When implementing adversarial training to suppress regime-conditioned behavior (sycophancy, sleeper triggers, data leakage)
- When the user asks to add probing classifiers to detect latent regime awareness in transformer activations
- When designing red-team infrastructure that tests whether a model can distinguish evaluation harnesses from real usage
- When the user needs to produce an invariance certificate documenting regime-information extractability for a model release
- When implementing LoRA-based fine-tuning with an adversarial invariance penalty term

## Key Technique

**Regime leakage** occurs when observable cues -- prompt formatting, system instructions, timing patterns, user identity markers -- allow a model to infer whether it is under evaluation or in deployment. The behavioral divergence between regimes is formally bounded: `Delta(pi) <= B * sqrt(2 * I(Z; R))`, where `I(Z; R)` is the mutual information between internal representations `Z` and the regime variable `R`. This bound (derived via the Data Processing Inequality chained through Jensen-Shannon divergence to total variation distance) means that reducing extractable regime information at decision-relevant layers directly constrains how much a model's behavior can diverge across contexts.

**Adversarial invariance training** implements this reduction through a minimax objective: `min_theta max_psi E[L_task(pi_theta(z), y) - lambda * L_adv(d_psi(phi_theta(z)), r)]`. The model minimizes task loss while simultaneously minimizing the ability of an adversarial probe `d_psi` to classify the regime from internal activations `phi_theta(z)`. In practice, this is implemented via a **gradient reversal layer** (GRL) attached to a probing classifier at a targeted bottleneck layer. The GRL flips the sign of the adversary's gradient during backpropagation, creating pressure on the model to compress regime-distinguishing features without destroying task-relevant information.

Critically, the intervention strength `lambda` has **heterogeneous dynamics** across failure modes. Sycophancy collapses sharply at low `lambda` (~0.06) with a clean representational phase transition, while sleeper-agent behavior requires substantially stronger pressure (`lambda` ~0.5) and never fully eliminates regime decodability at the audited bottleneck. This means invariance training is a meaningful but limited lever -- it must be paired with white-box probing diagnostics that report residual regime information rather than assumed to provide complete erasure.

## Step-by-Step Workflow

1. **Identify regime cues in your evaluation pipeline.** Catalog all features that differ between evaluation and deployment: prompt templates, system messages, special tokens, user agent strings, timestamp distributions, batch sizes. Each is a potential information channel the model could exploit.

2. **Construct regime-labeled datasets.** Create paired examples where identical tasks are presented under evaluation-regime formatting and deployment-regime formatting. Label each example with its regime (`r=0` for eval, `r=1` for deploy). Ensure task content is balanced across regimes to isolate regime signal from task signal.

3. **Extract activations at the target bottleneck layer.** Run forward passes on the regime-labeled dataset and capture residual stream activations at the final-token position of the decision-relevant layer (typically the last or penultimate transformer layer). Store these as `(activation, regime_label, task_label)` tuples.

4. **Train a regime probe classifier.** Fit a two-layer MLP (hidden dim 512, ReLU activation, 0.1 dropout) on the extracted activations to predict regime labels. Use a held-out split to measure probe accuracy. Accuracy significantly above 50% indicates extractable regime information.

5. **Implement the adversarial invariance training loop.** Add a gradient reversal layer between the target bottleneck and the probe. During fine-tuning, the combined loss is `L = L_task - lambda * L_probe`. Use LoRA (rank 32, alpha 64) on attention projections for parameter-efficient training. Start with `lambda=0.01` and sweep upward.

6. **Sweep the intervention strength `lambda`.** Train separate checkpoints at `lambda` values [0.01, 0.03, 0.06, 0.1, 0.2, 0.5]. For each checkpoint, evaluate: (a) task utility on the original benchmark, (b) regime-conditioned failure rate, (c) probe accuracy on held-out activations. Plot all three metrics against `lambda` to identify the transition point.

7. **Validate behavioral neutralization.** For each candidate checkpoint, run the full evaluation suite under both regimes. Compute the behavioral divergence `Delta = |metric_eval - metric_deploy|`. Confirm that task utility remains within acceptable bounds (target: <1% degradation).

8. **Produce an invariance certificate.** Document: (a) the probe architecture and capacity, (b) the bottleneck layer targeted, (c) held-out probe accuracy at the selected `lambda`, (d) the behavioral divergence measured, (e) the task utility retained. This certificate specifies the adversary model explicitly -- it claims invariance relative to a bounded probe class, not information-theoretic erasure.

9. **Deploy residual monitoring.** In production, periodically extract activations from the target layer on a sample of live traffic and re-evaluate probe accuracy. Drift in probe accuracy above the certified baseline triggers a re-audit. This catches distribution shifts that might re-introduce regime decodability.

10. **Iterate across failure modes independently.** Different regime-conditioned behaviors have different representational signatures and require different `lambda` strengths. Run the full pipeline separately for each failure mode (sycophancy, sleeper behavior, data leakage) rather than assuming a single `lambda` covers all cases.

## Concrete Examples

**Example 1: Auditing a fine-tuned model for sycophancy regime leakage**

User: "I fine-tuned Qwen2.5-7B on scientific QA. I suspect it's more sycophantic during evaluation than in production. Can you help me detect this?"

Approach:
1. Build a regime-labeled dataset with 500 scientific questions, each formatted in eval-style (with rubric/scoring framing) and deploy-style (plain user query)
2. Extract Layer 27 residual stream activations at the final token position
3. Train a 2-layer MLP probe (512 hidden, ReLU, dropout 0.1) on 80/20 split
4. Report probe accuracy

Output:
```python
import torch
import torch.nn as nn
from transformers import AutoModelForCausalLM, AutoTokenizer

# Step 1: Load model and prepare extraction hook
model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-7B", torch_dtype=torch.bfloat16)
tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-7B")

activations = {}
def hook_fn(module, input, output):
    activations["layer_27"] = output[0][:, -1, :].detach()  # final token

model.model.layers[27].register_forward_hook(hook_fn)

# Step 2: Collect activations across regime-labeled examples
regime_data = []  # list of (text, regime_label) tuples
all_acts, all_labels = [], []

for text, regime_label in regime_data:
    inputs = tokenizer(text, return_tensors="pt")
    with torch.no_grad():
        model(**inputs)
    all_acts.append(activations["layer_27"].cpu())
    all_labels.append(regime_label)

X = torch.cat(all_acts, dim=0)
y = torch.tensor(all_labels)

# Step 3: Train regime probe
class RegimeProbe(nn.Module):
    def __init__(self, input_dim, hidden_dim=512):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.ReLU(),
            nn.Dropout(0.1),
            nn.Linear(hidden_dim, 2),
        )
    def forward(self, x):
        return self.net(x)

probe = RegimeProbe(X.shape[-1])
# ... train with cross-entropy on 80/20 split ...

# Step 4: Report
# Probe accuracy: 89.3% -> significant regime leakage detected
# Recommendation: apply adversarial invariance training with lambda sweep
```

**Example 2: Implementing adversarial invariance training to suppress sleeper behavior**

User: "I need to add regime-blind training to my LoRA fine-tuning pipeline to mitigate potential sleeper-agent triggers."

Approach:
1. Add a gradient reversal layer to the training loop
2. Attach a probe at the target bottleneck
3. Combine task loss with adversarial penalty
4. Sweep lambda from 0.01 to 0.5

Output:
```python
from torch.autograd import Function

class GradientReversal(Function):
    @staticmethod
    def forward(ctx, x, alpha):
        ctx.alpha = alpha
        return x.clone()

    @staticmethod
    def backward(ctx, grad_output):
        return -ctx.alpha * grad_output, None

def gradient_reversal(x, alpha=1.0):
    return GradientReversal.apply(x, alpha)

class AdversarialInvarianceTrainer:
    def __init__(self, model, probe, lora_params, target_layer=27, lr=2.5e-4):
        self.model = model
        self.probe = probe  # RegimeProbe instance
        self.target_layer = target_layer
        self.optimizer = torch.optim.AdamW(lora_params, lr=lr)
        self.probe_optimizer = torch.optim.AdamW(probe.parameters(), lr=1e-3)
        self.task_loss_fn = nn.CrossEntropyLoss()
        self.adv_loss_fn = nn.CrossEntropyLoss()

    def train_step(self, batch, regime_labels, lam):
        # Forward pass (task)
        outputs = self.model(**batch)
        task_loss = self.task_loss_fn(outputs.logits, batch["labels"])

        # Extract bottleneck activations
        h = self.activations[self.target_layer]  # from hook

        # Adversarial probe with gradient reversal
        h_rev = gradient_reversal(h, alpha=lam)
        regime_logits = self.probe(h_rev)
        adv_loss = self.adv_loss_fn(regime_logits, regime_labels)

        # Combined: minimize task loss, maximize probe confusion
        loss = task_loss + adv_loss  # GRL handles sign flip
        loss.backward()
        self.optimizer.step()
        self.probe_optimizer.step()
        return {"task_loss": task_loss.item(), "adv_loss": adv_loss.item()}
```

**Example 3: Generating an invariance certificate**

User: "I need to document the regime-awareness properties of our model for the safety review before deployment."

Approach:
1. Run the full probing pipeline on the release candidate
2. Measure probe accuracy, behavioral divergence, and task utility
3. Generate a structured certificate

Output:
```markdown
# Invariance Certificate — Model v2.3-release

## Audit Scope
- **Model**: Qwen2.5-7B + LoRA (regime-blind, lambda=0.1)
- **Bottleneck**: Layer 27 residual stream, final-token position
- **Probe class**: 2-layer MLP, 512 hidden units, ReLU, 0.1 dropout
- **Dataset**: 1,000 regime-labeled scientific QA pairs (held-out)

## Results
| Metric                    | Value  | Threshold |
|---------------------------|--------|-----------|
| Probe accuracy (eval)     | 51.2%  | < 55%     |
| Probe accuracy (deploy)   | 49.8%  | < 55%     |
| Behavioral divergence     | 0.003  | < 0.01    |
| Task utility (MMLU)       | 68.4%  | > 67.0%   |
| Sycophancy rate (eval)    | 2.1%   | < 5%      |
| Sycophancy rate (deploy)  | 1.8%   | < 5%      |

## Limitations
- Certificate is relative to the specified probe class (2-layer MLP, 512 hidden).
  A higher-capacity adversary may extract additional regime signal.
- Audited at Layer 27 only. Regime information may exist at other layers.
- Does not constitute information-theoretic erasure guarantee.

## Recommendation
Model passes regime-blindness audit for sycophancy failure mode.
Sleeper-agent audit pending (requires lambda >= 0.3 sweep).
```

## Best Practices

- **Do:** Target the final transformer layer's residual stream for probing -- this is where decision-relevant features concentrate before the language modeling head.
- **Do:** Sweep `lambda` across at least 5 values and plot the three-way tradeoff (task utility, failure rate, probe accuracy) before selecting a checkpoint.
- **Do:** Use held-out probe evaluations with fresh random seeds to avoid overfitting the probe to training noise.
- **Do:** Treat invariance certificates as relative to a specified adversary class -- always document the probe architecture and capacity bounds.
- **Avoid:** Assuming a single `lambda` works across all failure modes. Sycophancy and sleeper behaviors have fundamentally different representational dynamics and require independent tuning.
- **Avoid:** Claiming information-theoretic erasure. Adversarial invariance training reduces extractability under a bounded probe class; it does not guarantee that no higher-capacity adversary could recover regime signal.
- **Avoid:** Skipping behavioral evaluation in favor of probing alone. Probing is a complement to behavioral testing, not a replacement -- a model can pass probing audits while still exhibiting regime-conditioned behavior through pathways not captured by the audited bottleneck.

## Error Handling

- **Probe accuracy stuck at 50% before training:** The regime-labeled dataset may lack genuine regime cues. Verify that eval-format and deploy-format examples actually differ in the ways a model would encounter in practice. Check tokenization to confirm regime-distinguishing tokens are present.
- **Task utility degrades sharply at low `lambda`:** The LoRA rank may be too low to simultaneously maintain task performance and satisfy the invariance constraint. Increase LoRA rank (try 64 or 128) or reduce learning rate.
- **Probe accuracy remains high despite strong `lambda`:** Regime information may be encoded at layers other than the targeted bottleneck. Run probes across all layers to locate the primary encoding site, then re-target the adversarial intervention.
- **Training instability / oscillation:** The minimax optimization can be unstable. Use separate learning rates for model and probe (probe LR typically 10-50x higher), and consider probe warm-up (pre-train the probe for 100 steps before enabling gradient reversal).
- **Different failure modes require incompatible `lambda` values:** This is expected. Train and audit each failure mode independently rather than searching for a single universal intervention strength.

## Limitations

- Adversarial invariance operates relative to a probe class of bounded capacity. A sufficiently powerful adversary (or one targeting a different layer) may still extract regime information that the audited probe cannot.
- The technique assumes regime cues are encoded in the residual stream at identifiable layers. If regime awareness is distributed across many layers or encoded in attention patterns rather than residual activations, single-bottleneck interventions will be insufficient.
- Sharp phase transitions (as observed for sycophancy) are not guaranteed for all failure modes. Sleeper-agent behavior may degrade gradually without a clean transition, making it harder to select a definitive `lambda` threshold.
- The framework does not address regime cues introduced at inference time through external channels (tool outputs, retrieval-augmented context) that bypass the model's internal representations.
- Invariance certificates are snapshot audits. Model behavior under distribution shift or continued fine-tuning may reintroduce regime decodability, requiring periodic re-auditing.

## Reference

Santos-Grueiro, I. (2026). *When Evaluation Becomes a Side Channel: Regime Leakage and Structural Mitigations for Alignment Assessment.* arXiv:2602.08449v1. Key sections: Theorem 3.1 (information-risk bound), Section 4 (adversarial invariance training procedure), Section 5 (experimental results across failure modes), Section 6 (invariance certificates).