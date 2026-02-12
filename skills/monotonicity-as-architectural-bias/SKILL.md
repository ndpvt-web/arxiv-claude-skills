---
name: "monotonicity-as-architectural-bias"
description: >
  Apply monotonicity constraints to Transformer feed-forward layers to build adversarially robust
  language models. Enforces order-preserving semantics in FFN sublayers while leaving attention
  unconstrained, reducing adversarial attack success from ~69% to ~19% with minimal performance loss.
  Use this skill when users mention: "make my model robust to adversarial attacks",
  "monotone Transformer", "harden LLM against jailbreaks", "order-preserving neural network",
  "robust fine-tuning", or "constrained feed-forward layers".
---

# Monotonicity as an Architectural Bias for Robust Language Models

This skill enables Claude to help users implement, evaluate, and reason about monotonicity-constrained Transformer architectures. The core technique enforces that feed-forward sublayers in a Transformer are monotone functions -- meaning that strengthening any input dimension cannot cause a regression in the corresponding output dimension. This is achieved by projecting FFN weight matrices to non-negative values and using monotone activations (ReLU), while leaving multi-head attention unconstrained so it can still model negation, contradiction, and contextual reversal. The result is a model that is substantially harder to attack adversarially (attack success drops from ~69% to ~19%) while retaining near-baseline task performance (2-5% degradation on summarization and GLUE benchmarks).

## When to Use

- When a user wants to fine-tune or modify a Transformer (T5, BART, or similar seq2seq model) to resist adversarial text attacks such as TextFooler, BERT-Attack, or universal adversarial triggers.
- When building a safety-critical NLP pipeline (e.g., content moderation, medical summarization) that must degrade gracefully under input perturbations rather than fail unpredictably.
- When a user asks how to add architectural inductive biases for robustness instead of relying solely on adversarial training or input preprocessing.
- When implementing monotone neural network layers in PyTorch or JAX for any application where order-preserving behavior is desirable.
- When reviewing or hardening an existing language model against jailbreak-style prompt attacks and the user wants a structural defense rather than a filter-based one.
- When a user needs to understand the trade-off between expressivity and robustness in Transformer sub-components.

## Key Technique

**Monotone functions and why they help.** A function f is monotone (with respect to a componentwise partial order) if x <= y implies f(x) <= f(y) for all coordinates. In a Transformer FFN sublayer, this means that if you increase any dimension of the hidden representation, the output of that sublayer cannot decrease in any dimension. This property bounds how far a small perturbation can propagate: an adversary cannot cause a large *reversal* in internal semantics by nudging a few input tokens. Monotonicity is related to Lipschitz continuity but is strictly stronger in its directional guarantee -- it prevents sign-flipping cascades that are the mechanism behind many successful adversarial attacks.

**How to enforce it.** The standard Transformer FFN sublayer computes `FFN(x) = W2 * ReLU(W1 * x + b1) + b2`. To make this monotone: (1) project W1 and W2 to non-negative matrices after each gradient step by applying elementwise `max(W, 0)` (i.e., ReLU on the weights themselves), and (2) keep ReLU as the activation (it is already monotone and non-negative). Biases b1 and b2 are left unconstrained since they are additive shifts that do not break monotonicity. This projection is applied only to FFN sublayers; the attention mechanism (Q, K, V projections and softmax) remains completely unconstrained, preserving the model's ability to attend selectively, negate, and contextualize.

**Why separating attention from FFN matters.** Language understanding requires non-monotone operations -- negation ("not guilty"), contradiction ("despite the evidence"), and contextual reweighting. The insight from this paper is that these operations are naturally handled by attention (which selects and reweights), while the FFN sublayers perform "semantic refinement" that should be order-preserving. By constraining only the FFN, the architecture retains full expressivity for relational reasoning while ensuring that downstream semantic processing cannot amplify adversarial perturbations.

## Step-by-Step Workflow

1. **Select a pretrained seq2seq Transformer** (T5-base or BART-base are validated choices). Load the model and identify all feed-forward sublayers -- in a standard Transformer encoder-decoder, these are the two linear layers in each `TransformerEncoderLayer` and `TransformerDecoderLayer` FFN block.

2. **Implement a non-negative weight projection function.** After each optimizer step, clamp FFN weight matrices to non-negative values:
   ```python
   def project_to_nonneg(model):
       for name, param in model.named_parameters():
           if "fc1.weight" in name or "fc2.weight" in name:
               param.data.clamp_(min=0.0)
   ```
   Call this after every `optimizer.step()` during training.

3. **Initialize from pretrained weights by projecting.** Before fine-tuning, project the pretrained FFN weights to non-negative space. This is a lossy operation, so expect the model to need fine-tuning to recover:
   ```python
   with torch.no_grad():
       for name, param in model.named_parameters():
           if "fc1.weight" in name or "fc2.weight" in name:
               param.data = param.data.clamp(min=0.0)
   ```

4. **Verify attention layers are untouched.** Confirm that Q, K, V projection matrices, output projections, and layer norm parameters are NOT affected by the projection. Only target the FFN linear layers.

5. **Fine-tune on the target task** (e.g., summarization on CNN/DailyMail, or classification on GLUE). Use a standard learning rate (1e-5 to 3e-5 for fine-tuning) and apply the non-negative projection after each gradient step. Monitor both task performance and weight statistics to ensure the constraint is being maintained.

6. **Add a monotonicity regularization term** to the loss to discourage weights from hovering near zero (which the projection would silently clamp). A simple L2 penalty on the negative part of the weights works well:
   ```python
   mono_reg = sum(
       torch.sum(torch.relu(-p))
       for n, p in model.named_parameters()
       if "fc1.weight" in n or "fc2.weight" in n
   )
   loss = task_loss + lambda_mono * mono_reg
   ```
   Set `lambda_mono` between 0.01 and 0.1 initially.

7. **Evaluate on clean benchmarks** to confirm task performance stays within 2-5% of the unconstrained baseline. If degradation is larger, reduce `lambda_mono` or use a warmup schedule that gradually increases the projection strength.

8. **Run adversarial evaluations** using TextAttack or equivalent libraries. Compare attack success rates (ASR) between the monotone and baseline models under TextFooler, BERT-Attack, and character-level perturbations. Expected improvement: ASR dropping from ~69% to ~19%.

9. **Profile per-layer monotonicity compliance** by checking that all FFN weights remain non-negative after training. Log any violations (which would indicate a bug in the projection step):
   ```python
   for name, param in model.named_parameters():
       if "fc1.weight" in name or "fc2.weight" in name:
           neg_count = (param.data < 0).sum().item()
           assert neg_count == 0, f"{name} has {neg_count} negative weights"
   ```

10. **Deploy with runtime assertions** in safety-critical settings. Periodically verify the non-negativity constraint holds, especially if the model is further fine-tuned or adapted downstream.

## Concrete Examples

**Example 1: Hardening a T5 summarization model**

User: "I have a T5-base model fine-tuned for summarization. Adversarial inputs cause it to produce hallucinated or nonsensical summaries. How can I make it more robust?"

Approach:
1. Load the fine-tuned T5-base model and identify all FFN sublayers (T5 uses `DenseReluDense` blocks in each layer).
2. Project all `wi.weight` and `wo.weight` parameters to non-negative values using `param.data.clamp_(min=0.0)`.
3. Fine-tune for 2-3 additional epochs on the summarization dataset with the non-negative projection applied after each optimizer step.
4. Evaluate on clean CNN/DailyMail (expect ROUGE-L to drop by 1-3 points) and adversarial inputs (expect ASR to drop from ~65% to ~20%).

Output:
```
Baseline T5-base:    ROUGE-L 41.2 | Adversarial ASR 67%
Monotone T5-base:    ROUGE-L 39.1 | Adversarial ASR 18%
```

**Example 2: Implementing monotone FFN in PyTorch from scratch**

User: "Show me how to implement a monotone feed-forward block I can drop into any Transformer."

Approach:
1. Create a `MonotoneFFN` module that wraps two linear layers with a non-negative weight constraint.
2. Override the `forward` method to use standard ReLU activation.
3. Provide a `project_weights` method to be called after each optimizer step.

Output:
```python
import torch
import torch.nn as nn

class MonotoneFFN(nn.Module):
    """Drop-in replacement for Transformer FFN with monotonicity constraint."""

    def __init__(self, d_model: int, d_ff: int):
        super().__init__()
        self.fc1 = nn.Linear(d_model, d_ff)
        self.fc2 = nn.Linear(d_ff, d_model)
        self.activation = nn.ReLU()
        # Initialize with non-negative weights
        self.project_weights()

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        return self.fc2(self.activation(self.fc1(x)))

    @torch.no_grad()
    def project_weights(self):
        """Call after optimizer.step() to enforce monotonicity."""
        self.fc1.weight.data.clamp_(min=0.0)
        self.fc2.weight.data.clamp_(min=0.0)

# Usage in training loop:
# optimizer.step()
# for module in model.modules():
#     if isinstance(module, MonotoneFFN):
#         module.project_weights()
```

**Example 3: Evaluating robustness improvement with TextAttack**

User: "I applied the monotone constraint to my BART model. How do I measure the robustness improvement?"

Approach:
1. Install TextAttack and load both the baseline and monotone models as custom model wrappers.
2. Run TextFooler and BERT-Attack on a held-out evaluation set (200-500 examples).
3. Compare attack success rate (ASR), perturbed accuracy, and average number of queries.

Output:
```python
import textattack
from textattack.attack_recipes import TextFoolerJin2019, BERTAttackLi2020

# Wrap your model for TextAttack
model_wrapper = textattack.models.wrappers.HuggingFaceModelWrapper(model, tokenizer)

# Run TextFooler attack
attack = TextFoolerJin2019.build(model_wrapper)
dataset = textattack.datasets.HuggingFaceDataset("glue", "sst2", split="validation")

attack_args = textattack.AttackArgs(
    num_examples=300,
    log_to_csv="results.csv",
    disable_stdout=True
)
attacker = textattack.Attacker(attack, dataset, attack_args)
results = attacker.attack_dataset()

# Expected output comparison:
# Baseline BART:  ASR 71%, Avg queries 48
# Monotone BART:  ASR 21%, Avg queries 112 (attacker works harder, succeeds less)
```

## Best Practices

- **Do:** Apply the non-negative projection after every single optimizer step, not just at the end of an epoch. Gradient updates will push weights negative, and the projection must immediately correct this.
- **Do:** Start from pretrained weights and fine-tune with the constraint, rather than training from scratch. The pretrained attention weights carry critical linguistic knowledge that is unaffected by the FFN constraint.
- **Do:** Monitor the distribution of FFN weights during training. A healthy monotone model will have weights concentrated in a small positive range, not piled up at exactly zero.
- **Do:** Use the monotonicity constraint alongside (not instead of) standard alignment techniques like RLHF. They are complementary defenses.
- **Avoid:** Constraining attention weight matrices, layer normalization parameters, or embedding layers. Monotonicity in these components would severely degrade the model's ability to handle negation, contrast, and contextual meaning.
- **Avoid:** Setting `lambda_mono` too high (>0.5), which forces weights far from their pretrained values and causes significant task degradation. Start low and increase only if adversarial robustness is insufficient.

## Error Handling

- **Task performance drops more than 5%.** Reduce regularization strength, use a learning rate warmup, or apply the constraint only to a subset of layers (e.g., the last N encoder layers) as a partial monotonicity approach.
- **Weights collapse to near-zero.** This happens if the learning rate is too high combined with the projection. Lower the learning rate or use gradient clipping before projection.
- **Attention layers accidentally constrained.** Verify your parameter name filtering regex. In HuggingFace models, FFN layers are named differently across architectures: `fc1/fc2` (BART), `wi/wo` (T5), `intermediate.dense/output.dense` (BERT). Always inspect `model.named_parameters()` to identify the correct names.
- **Adversarial evaluation shows no improvement.** Confirm the projection is actually being applied by checking `(param < 0).any()` returns False for all FFN weights. A common bug is applying the projection to a copy of the parameters rather than in-place.

## Limitations

- **Empirically validated on seq2seq models (T5, BART) for summarization and GLUE tasks only.** Generalization to decoder-only models (GPT, LLaMA) or tasks like open-ended generation is plausible but unverified by this paper.
- **Does not provide certified robustness guarantees.** The monotonicity constraint improves empirical robustness significantly but does not come with formal certificates against all possible perturbations.
- **Partial expressivity loss in FFN layers.** Some tasks requiring highly non-linear FFN transformations (e.g., complex arithmetic reasoning encoded in FFN weights) may see larger performance drops.
- **Projection after each step is a hard constraint.** Soft enforcement via regularization alone (without clamping) may not maintain monotonicity strictly, but hard clamping can interact poorly with momentum-based optimizers. Consider zeroing momentum for clamped weights.
- **Character-level and embedding-space attacks** that operate below the token level may not be fully mitigated, since the constraint operates on hidden representations post-embedding.

## Reference

Cooper, P., Nadali, A., Trivedi, A., & Velasquez, A. (2026). *Monotonicity as an Architectural Bias for Robust Language Models.* arXiv:2602.02686v1. [https://arxiv.org/abs/2602.02686v1](https://arxiv.org/abs/2602.02686v1)

Key sections to read: Section 3 (monotone FFN formulation and the architectural separation argument), Section 4 (empirical robustness evaluation across attack types), and Table 1 (performance vs. robustness trade-off numbers).