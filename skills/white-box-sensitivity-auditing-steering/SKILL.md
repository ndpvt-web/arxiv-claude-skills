---
name: "white-box-sensitivity-auditing-steering"
description: "Audit LLMs for hidden bias using activation steering vectors. Use when: 'audit this model for gender bias', 'check if my LLM depends on race', 'white-box bias test', 'steering vector sensitivity analysis', 'detect hidden demographic bias in model', 'run a fairness audit on this language model'"
---

# White-Box Sensitivity Auditing with Steering Vectors

This skill enables Claude to help users implement and run white-box sensitivity audits on large language models using activation steering vectors. Unlike black-box audits that only test input-output behavior (and frequently miss hidden biases), this technique computes directional vectors in a model's activation space that represent protected concepts (e.g., gender, race), then injects those vectors during inference to measure how much the model's decisions depend on those concepts. This consistently reveals bias that standard prompt-based evaluations miss entirely.

## When to Use

- When a user wants to audit an LLM for bias related to gender, race, or other protected attributes
- When standard black-box fairness evaluations show "no bias" but the user suspects hidden sensitivity
- When building a bias evaluation pipeline for a high-stakes decision system (hiring, admissions, lending, medical triage)
- When the user asks to extract or compute steering vectors representing demographic concepts from a transformer model
- When implementing internal sensitivity tests that go beyond counterfactual input swapping
- When the user needs to identify which transformer layers encode the most demographic information
- When comparing white-box vs. black-box audit results to demonstrate evaluation gaps

## Key Technique

**Activation steering** works by computing a direction in a model's internal representation space that corresponds to a concept (e.g., "male vs. female"). This direction — the **steering vector** — is extracted by running contrastive prompt pairs through the model, collecting activations at each transformer layer, and computing the mean difference between the positive-concept and negative-concept activation sets. The resulting vector, once normalized, captures the model's internal representation of that concept at each layer.

**Sensitivity auditing** applies these steering vectors during inference on a downstream task. By adding the scaled steering vector (`activation + alpha * steering_vector`) to a chosen layer's residual stream, the auditor shifts the model's internal state along the concept axis without changing the input text. If the model's output (e.g., accept/reject decision, diagnosis, sentence recommendation) changes substantially under this perturbation, the model is demonstrably sensitive to that protected attribute — even if swapping demographic terms in the input text produces no measurable difference. The scaling coefficient `alpha` is swept across a range (typically -1 to +1) to measure sensitivity at varying intervention strengths.

**Layer selection** is critical: not all layers encode concepts equally. The method validates candidate steering vectors against held-out data by computing the Pearson correlation between scalar projections of activations onto each layer's steering vector and ground-truth concept scores. The top-performing layer (excluding the final ~20% of layers, which tend to be less effective) is selected for the audit. This automated layer selection eliminates manual tuning.

## Step-by-Step Workflow

1. **Define the audit scope.** Identify the protected concept to audit (e.g., gender, race) and the downstream decision task (e.g., admissions, hiring, medical QA, judicial sentencing). Specify the model under test and the decision labels (e.g., "admit"/"reject").

2. **Prepare contrastive training data.** Construct paired prompt sets where the only semantic difference is the target concept. For gender: use gendered language datasets, gender identity references, or name-based proxies. For race: use racial identity terms or culturally associated names. Each example should have a text, an instruction template, and an output prefix. Split into train and validation sets (e.g., `n_train=200`, `n_val=50`).

3. **Extract activations from all layers.** Run both positive-concept and negative-concept prompt sets through the model, collecting the residual stream activations at every transformer layer. Use `nnsight` for clean activation access:
   ```python
   from nnsight import LanguageModel
   model = LanguageModel(model_name, dispatch=True)
   with model.trace(inputs) as tracer:
       acts = model.model.layers[layer_id].output[0].save()
   ```

4. **Compute steering vectors via mean difference.** For each layer, calculate: `steering_vec[layer] = normalize(mean(pos_acts[layer]) - mean(neg_acts[layer]))`. Optionally use Weighted Mean Difference (WMD) which weights activations by concept strength scores for more precise vectors. Store vectors with their layer indices.

5. **Validate and select the optimal layer.** For each layer's steering vector, project validation-set activations onto the vector (scalar dot product) and compute Pearson correlation with ground-truth concept scores. Rank layers by `(correlation - RMSE)`. Filter out the final 20% of layers. Select the top-ranked layer as the intervention target.

6. **Compute the vector scale factor.** Calibrate the steering vector's magnitude by comparing the 90th-percentile range of scalar projections for positive vs. negative examples. This ensures the coefficient `alpha` produces interventions proportional to the model's natural activation magnitudes.

7. **Run the steering evaluation on the decision task.** For the target task, prepare decision prompts (without any demographic markers in the text). For each prompt, run inference with the steering vector applied at the selected layer, sweeping `alpha` from -1 to +1 in increments of 0.1. Record next-token probabilities for the decision labels at each coefficient.

8. **Run a baseline (no-steering) evaluation.** Process the same prompts without any steering intervention to establish baseline decision distributions.

9. **Compute sensitivity metrics.** Compare steered vs. baseline outputs: measure probability shifts for decision labels, decision flip rates (how often the most-probable label changes), and the slope of probability change as a function of `alpha`. Group results by any available subpopulations.

10. **Report findings with comparisons.** Present results showing (a) baseline decision distribution, (b) steered decision distribution at key coefficients, (c) sensitivity curves across the coefficient range. Compare against black-box counterfactual results if available, highlighting cases where steering reveals bias that input swapping missed.

## Concrete Examples

**Example 1: Gender Bias Audit on an Admissions Decision Model**

User: "I have a Llama-3-8B model fine-tuned for college admissions decisions. Audit it for gender bias."

Approach:
1. Load the model using nnsight's `LanguageModel` wrapper
2. Prepare gendered language contrastive pairs (e.g., sentences describing applicants with gendered pronouns/names vs. neutral)
3. Extract activations at all 32 layers for both concept polarities
4. Compute mean-difference steering vectors per layer, validate, select top layer (e.g., layer 18)
5. Construct admissions prompts: applicant profiles with no gender markers, asking for "admit" or "reject"
6. Sweep alpha from -1 to +1, recording P("admit") and P("reject") at each step

Output:
```
Audit Report: Gender Sensitivity — Admissions Task
Model: meta-llama/Meta-Llama-3-8B
Concept: gender | Method: Mean Difference | Selected Layer: 18

Baseline (no steering):
  P(admit) = 0.62 | P(reject) = 0.38

Steered Results (alpha sweep):
  alpha = -1.0 (female direction):  P(admit) = 0.45 | P(reject) = 0.55
  alpha =  0.0 (no intervention):   P(admit) = 0.62 | P(reject) = 0.38
  alpha = +1.0 (male direction):    P(admit) = 0.74 | P(reject) = 0.26

Sensitivity: 29 percentage-point swing in P(admit) across gender axis
Decision flip rate: 23% of cases flip from admit to reject (or vice versa)
Black-box counterfactual (name swap): 2% flip rate

FINDING: Substantial hidden gender sensitivity detected. White-box audit
reveals 11.5x more sensitivity than black-box name-swapping test.
```

**Example 2: Racial Bias Audit on a Medical QA Model**

User: "Check if my medical question-answering model has racial bias in its diagnostic recommendations."

Approach:
1. Use racial identity contrastive pairs (e.g., descriptions mentioning racial background vs. neutral)
2. Compute steering vectors using WMD method for higher precision
3. Prepare medical QA prompts (patient symptoms, no racial identifiers) with output labels for diagnostic options
4. Apply steering at the validated optimal layer, sweep coefficients
5. Compare diagnostic probability distributions under racial steering

Output:
```
Audit Report: Racial Sensitivity — Medical QA Task
Model: BioMistral-7B
Concept: race | Method: Weighted Mean Diff | Selected Layer: 22

Condition A (no racial info in prompt, no steering):
  Diagnosis distribution: {heart_disease: 0.41, anxiety: 0.35, other: 0.24}

Condition B (steering toward racial group X, alpha=+0.8):
  Diagnosis distribution: {heart_disease: 0.29, anxiety: 0.48, other: 0.23}

Condition C (steering toward racial group Y, alpha=-0.8):
  Diagnosis distribution: {heart_disease: 0.52, anxiety: 0.27, other: 0.21}

Max probability shift: 23pp for heart_disease across racial axis
Black-box test (race mentioned in prompt): 4pp shift

FINDING: Model internals encode racial associations affecting diagnostic
output, even when race is absent from input text.
```

**Example 3: Setting Up the Audit Pipeline from Scratch**

User: "Help me set up the steering audit framework from the Cyberey et al. paper."

Approach:
1. Install dependencies: `torch`, `transformers`, `nnsight`, `accelerate`, `pandas`, `scipy`, `pyyaml`, `dataclass-wizard`, `scikit-learn`
2. Clone the reference implementation: `git clone https://github.com/hannahxchen/llm-steering-audit`
3. Configure the audit via YAML or dataclass:
   ```python
   from steering_audit.config import Config, SteeringConfig
   config = Config(
       model_name="meta-llama/Meta-Llama-3-8B-Instruct",
       concept="gender",
       dataset="gendered_language",
       method="MD",          # or "WMD" for weighted
       n_train=200,
       n_val=50,
       filter_layer_pct=0.2  # exclude final 20% of layers
   )
   ```
4. Run training (vector extraction): `python -m steering_audit.run --run_train --concept gender --dataset gendered_language`
5. Run white-box evaluation: `python -m steering_audit.run --run_eval --eval_task admissions --min_coeff -1 --max_coeff 1 --increment 0.1`
6. Run black-box baseline for comparison: `python -m steering_audit.run --run_eval --blackbox --eval_task admissions`
7. Analyze results in the generated output directory under `runs/{concept}-{dataset}/{model}/`

## Best Practices

- **Do:** Always run both white-box (steering) and black-box (counterfactual) evaluations on the same task so you can quantify the detection gap between methods.
- **Do:** Use the Weighted Mean Difference (WMD) method when concept strength varies across examples — it produces more precise steering vectors than unweighted mean difference.
- **Do:** Filter out the final 20% of transformer layers during layer selection. These late layers tend to produce less effective steering vectors due to their proximity to the output head.
- **Do:** Sweep the full coefficient range (negative to positive) rather than testing a single value — the sensitivity curve shape reveals whether bias is linear or nonlinear.
- **Avoid:** Using steering vectors extracted from one model to audit a different model. Steering vectors are model-specific; each model requires its own extraction pass.
- **Avoid:** Drawing conclusions from a single alpha value. A model may show no sensitivity at alpha=0.5 but substantial sensitivity at alpha=0.8 — always report the full sweep.
- **Avoid:** Using very small training sets (< 100 examples per polarity) for vector extraction. Noisy vectors produce unreliable audits.

## Error Handling

- **Out-of-memory during activation extraction:** Reduce batch size in `EvalConfig` or use `torch_dtype=torch.float16`. For very large models, use `device_map="auto"` to shard across GPUs.
- **Low correlation during layer validation:** The contrastive pairs may not cleanly separate the concept. Increase training data diversity, or switch from MD to WMD method. If no layer exceeds a meaningful correlation threshold (~0.3), the concept may not be well-represented in the model's activation space.
- **Steering produces nonsensical outputs:** The coefficient is too large. Reduce `max_coeff` — start with the range [-0.5, +0.5] and expand only if sensitivity is low.
- **nnsight tracing errors:** Ensure the model architecture is supported. The framework auto-detects layer paths (`model.layers`, `model.language_model.layers`, `transformer.h`). For non-standard architectures, manually specify the block module path.
- **No decision flips observed:** This may be a genuine negative result (model is not sensitive), or the steering vector may target the wrong semantic axis. Validate by checking that the steering vector produces expected behavior on a known-biased control task before concluding absence of bias.

## Limitations

- Requires white-box access to model weights and activations — cannot audit closed API models (GPT-4, Claude, etc.)
- Steering vectors capture linear directions in activation space; nonlinear concept encodings will be partially or entirely missed
- The quality of the audit depends heavily on the quality of contrastive training data — poorly constructed pairs produce misleading results
- Currently validated only on single-concept audits (one protected attribute at a time); intersectional bias (e.g., race + gender simultaneously) requires further methodological development
- The technique measures model sensitivity to internal concept perturbation, which is a proxy for real-world bias — deployment context and downstream effects require additional evaluation
- Computational cost scales with model size and number of layers; auditing a 70B-parameter model requires significant GPU resources

## Reference

**Paper:** Cyberey, H., Ji, Y., & Evans, D. (2026). "White-Box Sensitivity Auditing with Steering Vectors." arXiv:2601.16398. https://arxiv.org/abs/2601.16398v1

Look for: Section 3 (the formal auditing framework definition), Section 4 (steering vector computation via Difference-in-Means and Weighted Mean Difference), Section 5 (the four decision tasks: admissions, judicial sentencing, South German credit, DiversityMedQA), and Table 1 comparing white-box vs. black-box detection rates.

**Code:** https://github.com/hannahxchen/llm-steering-audit — MIT licensed reference implementation with configs, contrastive datasets, and evaluation scripts for all four tasks.