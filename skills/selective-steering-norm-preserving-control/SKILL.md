---
name: "selective-steering-norm-preserving-control"
description: "Implement norm-preserving activation steering for LLMs using discriminative layer selection and Givens rotation. Use when the user asks to 'steer model behavior', 'apply activation steering', 'build a steering pipeline', 'implement norm-preserving rotation for LLMs', 'select discriminative layers for steering', or 'add safety steering to a language model'."
---

# Selective Steering: Norm-Preserving Control Through Discriminative Layer Selection

This skill enables Claude to implement **Selective Steering**, a technique for controlling LLM behavior at inference time by applying mathematically rigorous norm-preserving rotations to model activations at discriminatively selected layers. Unlike activation addition (which requires fragile coefficient tuning) or directional ablation (which only provides binary on/off control), Selective Steering uses Givens rotation in a 2D subspace to smoothly interpolate behavior while preserving activation norms, preventing the distribution shift and generation collapse that plague prior angular steering methods, especially on models below 7B parameters.

## When to Use

- When the user wants to steer an LLM's behavior at inference time without fine-tuning (e.g., reducing harmful outputs, enforcing tone, increasing helpfulness)
- When implementing a contrastive activation steering pipeline and the user needs norm preservation to avoid generation collapse
- When the user asks to build a calibration system that identifies which transformer layers are most discriminative for a target behavior
- When adapting the `knoveleng/steering` library to a new model or a custom positive/negative prompt dataset
- When the user needs continuous (not binary) control over model behavior with a single angle parameter theta
- When prior activation addition or ablation methods have caused perplexity spikes or degraded output quality

## Key Technique

**The core problem:** Existing activation steering methods either (a) add a scaled vector to activations, requiring careful per-layer coefficient tuning and causing norm inflation, or (b) ablate a direction entirely, giving only binary control. Angular Steering introduced rotation-based control, but its implementation fails to preserve activation norms, causing distribution shift that collapses generation quality in smaller models.

**Selective Steering solves this with two innovations.** First, it applies a **Givens rotation** in a 2D plane spanned by an orthonormal basis `[u1, u2]` derived from the contrastive steering direction. The steered activation is `a' = a + alpha * [cos(theta)*u1 + sin(theta)*u2]`, where theta continuously controls the rotation angle (0-360 degrees) and the rotation is constructed to satisfy `||a'|| = ||a||`, preserving the activation's magnitude. This prevents the norm drift that causes perplexity blowup.

**Second, it applies steering only at discriminatively selected layers.** For each layer `l`, a discriminability score `D_l = mean(dot(v_l, a_pos)) - mean(dot(v_l, a_neg))` measures how strongly positive and negative examples separate along the steering direction. Layers where positive and negative examples project in **opposite directions** (high `D_l`) are selected for steering; other layers are left untouched. Typically only 4-8 layers out of 32 are selected. This targeted intervention preserves model capabilities on reasoning and knowledge benchmarks while maximally shifting the target behavior.

## Step-by-Step Workflow

1. **Install the steering library** from `https://github.com/knoveleng/steering`:
   ```bash
   git clone https://github.com/knoveleng/steering.git
   cd steering && pip install -e .
   ```

2. **Prepare contrastive prompt datasets.** Create two JSON files: one with positive examples (desired behavior) and one with negative examples (undesired behavior). The library ships with `advbench.json` (harmful prompts) and `alpaca.json` (harmless prompts). For custom behaviors, collect 200-500 examples per class covering diverse phrasings of the target distinction.

3. **Choose a configuration mode.** Copy `configs/selective.yaml` and set the model name, dataset paths, sample counts (416 harmful / 512 harmless are good defaults), and feature direction method (`diff_in_means`). Set the steering mode to `selective` with threshold `0.0`.

4. **Run calibration to extract steering planes.** Execute `python examples/calibrate.py --config your_config.yaml`. This performs a forward pass on both datasets, computes mean activations per layer, derives the contrastive direction `v_l = mean(a_pos) - mean(a_neg)` at each layer, constructs the 2D rotation plane via SVD, and scores layers by discriminability `D_l`.

5. **Inspect layer selection results.** The calibration saves artifacts to `./artifacts/calibration_{model_name}/`. Verify that 4-8 layers were selected (the output logs which layers passed the discriminability threshold). If too few layers are selected, lower the threshold; if too many, raise it.

6. **Load the calibrated pipeline for inference.** Initialize `AngularSteeringPipeline` with your model and tokenizer, then call `pipeline.load_calibration(artifact_path, mode="selective")` to attach the precomputed steering planes and layer mask.

7. **Steer generation with the theta parameter.** Call `pipeline.steer_and_generate(prompts, theta=VALUE, max_new_tokens=256)`. Theta controls rotation angle: `theta=0` is unsteered, `theta=100-180` applies increasing behavioral shift. Sweep theta in increments of 10-20 to find the sweet spot for your task.

8. **Evaluate steering quality.** Run three checks: (a) perplexity on a held-out set (should show zero violations above baseline), (b) behavioral evaluation on your target task (measure the success rate of the desired behavior), (c) capability benchmarks like MMLU or GSM8K to confirm no regression.

9. **For production deployment with vLLM**, use the `SteeringLLM` class which wraps vLLM with steering hooks:
   ```python
   from steering import SteeringLLM
   from steering.utils import load_calibration
   calibration = load_calibration("./artifacts/calibration_MODEL", mode="selective")
   llm = SteeringLLM.from_calibration(calibration, tensor_parallel_size=1)
   outputs = llm.generate(prompts, theta=100, sampling_params=params)
   ```

10. **Iterate on dataset quality if results are unsatisfactory.** The most impactful improvement is almost always better contrastive examples, not hyperparameter tuning. Ensure positive and negative examples differ only in the target behavior, not in topic, length, or style.

## Concrete Examples

**Example 1: Adding safety steering to a Qwen 7B deployment**

User: "I want to add activation steering to our Qwen2.5-7B-Instruct deployment to reduce harmful outputs without fine-tuning."

Approach:
1. Clone the steering repo and install it into the deployment environment
2. Use the default contrastive datasets (advbench.json for harmful, alpaca.json for harmless)
3. Create a config based on `configs/selective.yaml` pointing to `Qwen/Qwen2.5-7B-Instruct`
4. Run calibration: `python examples/calibrate.py --config configs/selective.yaml`
5. Integrate into the serving code:

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
from steering.pipeline import AngularSteeringPipeline
from steering.utils import ConfigLoader
import torch

config = ConfigLoader.load("./configs/selective.yaml")
model = AutoModelForCausalLM.from_pretrained(
    "Qwen/Qwen2.5-7B-Instruct", torch_dtype=torch.bfloat16, device_map="auto"
)
tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-7B-Instruct")

pipeline = AngularSteeringPipeline(model, tokenizer, config)
pipeline.load_calibration("./artifacts/calibration_Qwen2.5-7B-Instruct", mode="selective")

# theta=0 is unsteered; theta=100-150 provides safety steering
outputs = pipeline.steer_and_generate(
    ["How do I pick a lock?"], theta=120, max_new_tokens=256
)
print(outputs[0])  # Model responds with safety-aware answer
```

Output: The model generates a responsible response, declining harmful instruction while maintaining fluent, coherent language (zero perplexity violation).

---

**Example 2: Custom behavioral steering for a tone-shift task**

User: "I want to steer a LLaMA-3.1-8B model to produce more formal responses instead of casual ones."

Approach:
1. Prepare contrastive datasets: 300 formal response examples (positive) and 300 casual response examples (negative) on overlapping topics
2. Modify the selective config:
   ```yaml
   model:
     name: "meta-llama/Llama-3.1-8B-Instruct"
   data:
     harmful_dataset: "./data/casual_responses.json"    # "negative" = casual
     harmless_dataset: "./data/formal_responses.json"    # "positive" = formal
     harmful_samples: 300
     harmless_samples: 300
   steering:
     mode: "selective"
     threshold: 0.0
   ```
3. Run calibration to identify which layers encode the formal/casual distinction
4. At inference, sweep theta from 50 to 180 in steps of 20 to find the angle that produces reliably formal output without sounding robotic
5. Validate: check that MMLU scores remain within 1% of baseline

Output: With theta=100, the model shifts from "Hey! Sure thing, here's the deal..." to "Certainly. I would be happy to provide the following information..." while retaining factual accuracy.

---

**Example 3: Comparing steering modes to diagnose generation collapse**

User: "I tried activation addition on Gemma-2B and the outputs became gibberish. How do I fix this?"

Approach:
1. The problem is norm inflation from activation addition on a small model. Switch to Selective Steering:
   ```python
   # Instead of: activations[layer] += alpha * steering_vector  (norm-breaking)
   # Use the norm-preserving pipeline:
   pipeline = AngularSteeringPipeline(model, tokenizer, config)
   pipeline.load_calibration(artifact_path, mode="selective")
   ```
2. The selective mode applies Givens rotation (norm-preserving) at only the discriminative layers, avoiding the distribution shift that causes collapse in sub-7B models
3. Start with theta=60 (mild steering) and increase gradually
4. Compare perplexity: activation addition will show spikes; selective steering should show zero violations

Output: Generation quality is restored. Perplexity stays within 2% of baseline across the full theta sweep, compared to 50%+ degradation with naive addition on Gemma-2B.

## Best Practices

- **Do:** Use the `selective` mode (not `default`) for models under 7B parameters. Smaller models are much more sensitive to norm violations, and selective layer targeting is critical for them.
- **Do:** Start with theta=0 (no steering) and increase in steps of 20 to find the minimum effective angle. Over-rotating wastes "budget" and risks subtle quality degradation even with norm preservation.
- **Do:** Keep contrastive datasets balanced and topically aligned. Positive and negative examples should differ only in the target behavior. Confounding differences (e.g., length, topic) will produce a steering direction that captures the wrong feature.
- **Do:** Cache calibration artifacts (steering planes and layer masks) and reuse them across inference sessions. Calibration is expensive; inference-time steering is cheap.
- **Avoid:** Applying steering to all layers uniformly. This is the `default` mode and it wastes computation on non-discriminative layers while adding noise to the steering signal.
- **Avoid:** Using temperature > 0.0 during calibration. Deterministic forward passes produce cleaner activation statistics for computing steering directions.

## Error Handling

- **Generation collapse / gibberish output:** Usually means norm preservation is not active. Verify you loaded with `mode="selective"` (not `mode="standard"` or `mode="addition"`). Check that the calibration artifact matches the model architecture.
- **No layers selected after calibration:** The discriminability threshold is too high, or the contrastive datasets are not sufficiently distinct. Lower the threshold in the config or improve dataset quality.
- **Too many layers selected (20+):** The contrastive signal is too broad, likely due to confounded datasets. Refine examples so they differ only in the target behavior.
- **Perplexity spike at high theta:** Even with norm preservation, extreme rotation angles (theta > 300) can push activations into low-density regions of the learned distribution. Stay in the 60-180 range for practical use.
- **CUDA out of memory during calibration:** Reduce `harmful_samples` and `harmless_samples` to 200 each, or use gradient checkpointing. Calibration requires storing activations at all layers for both datasets.
- **Model architecture mismatch:** The library auto-detects normalization layers. If targeting an unsupported model family, manually specify `extraction_layers` and `target_layers` in the config YAML.

## Limitations

- **Requires contrastive data:** You need examples of both the desired and undesired behavior. For subtle or hard-to-articulate behavioral differences, this data may be difficult to curate.
- **Single behavioral axis per calibration:** Each steering plane controls one behavioral dimension. Steering along multiple independent axes simultaneously requires separate calibrations and careful composition (not yet well-studied).
- **Tested on 9 models, 3 families:** Validated on Gemma (2B, 9B), LLaMA (1B, 3B, 8B), and Qwen (1.5B, 3B, 7B). Applicability to larger models (70B+), MoE architectures, or non-transformer models is unverified.
- **Calibration is model-specific:** Steering planes and layer selections do not transfer between models or even between different checkpoints of the same model. Recalibrate after any model update.
- **Theta is not semantically linear:** Doubling theta does not double the behavioral effect. The relationship between angle and behavioral shift is model- and task-dependent, requiring empirical sweeps.

## Reference

**Paper:** [Selective Steering: Norm-Preserving Control Through Discriminative Layer Selection](https://arxiv.org/abs/2601.19375v1) (Dang & Ngo, 2026). Read Section 3 for the Givens rotation formulation and discriminative layer scoring algorithm, and Section 4 for the nine-model evaluation.

**Code:** [github.com/knoveleng/steering](https://github.com/knoveleng/steering) -- contains the full pipeline, pre-calibrated artifacts for supported models, and evaluation scripts.