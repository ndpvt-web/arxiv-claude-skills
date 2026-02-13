---
name: "seta-statistical-fault-attribution"
description: "Diagnose and attribute faults in compound AI systems (multi-model pipelines) using SETA's modular robustness testing framework. Applies perturbations, traces execution through each component, computes per-component metamorphic relation scores, and statistically attributes system failures to specific modules. Use when: 'find which model in my pipeline is failing', 'robustness test my multi-model system', 'trace error propagation in my AI pipeline', 'attribute faults in my compound AI system', 'which component is causing my pipeline to break under noise', 'test robustness of my ML pipeline'."
---

# SETA: Statistical Fault Attribution for Compound AI Systems

This skill enables Claude to design, implement, and apply modular robustness testing for compound AI systems — pipelines composed of multiple interconnected neural networks or ML models. Using the SETA framework (Statistical Fault Attribution), Claude instruments a multi-model pipeline to record execution traces under clean and perturbed inputs, defines per-component metamorphic relations (correctness specifications), and computes Failure Contribution scores that statistically pinpoint which component is most responsible for system-level failures. This replaces opaque end-to-end metrics with fine-grained, per-module fault attribution.

## When to Use

- When a user has a multi-model AI pipeline (e.g., detection -> classification -> OCR) and wants to know which component degrades most under perturbations
- When building robustness tests for a compound AI system and the user needs more than end-to-end accuracy
- When a user asks to trace how errors propagate from one model to the next in a pipeline
- When a user wants to compare component-level robustness across different perturbation types (noise, blur, weather)
- When debugging a production ML pipeline where system accuracy drops but it is unclear which model is at fault
- When a user needs to prioritize which model in a pipeline to retrain, harden, or replace
- When designing test harnesses for safety-critical multi-network systems (autonomous vehicles, medical imaging pipelines, inspection systems)

## Key Technique

**Modular testing via execution traces and metamorphic relations.** SETA models a compound AI system as a state-transition system S = (Q, I, Phi, F, R, S) where Q is the set of computational modules, R contains routing functions that determine which module fires next, and F holds the model functions. For each test input x, the framework records an execution trace T(x) — a sequence of nodes v = (q, x_v, y_v, S_q) capturing which module q was activated, what it received, what it produced, and how to score it. A perturbation g(x) produces a perturbed trace T(g(x)). By aligning nodes across traces using structural identifiers, SETA compares each component's behavior under clean vs. perturbed conditions.

**Per-component correctness via composite metamorphic relations.** Instead of a single end-to-end metric, each module i gets a composite score S_i(x, x_tilde) = product of M_ij(x, x_tilde) over all its metamorphic relations. These relations encode domain-specific invariants: "detections should be a subset of originals" (for detectors), "class label should not change" (for classifiers), "IoU must exceed 0.9 for persisting detections," or "Levenshtein distance below threshold" (for OCR). The system-level score is S(x, x_tilde) = product of all S_i. A system failure occurs when S = 0.

**Statistical fault attribution via Failure Contribution scores.** The key metric is FC_i = E[Z_i * I(S(x, x_tilde) = 0)] where Z_i = 1 - S_i measures component i's deviation and I(S=0) indicates system failure. Empirically: FC_i = sum over all (input, perturbation) pairs of Z_i * I(S=0), divided by the total number of system failures. The normalized attribution weight alpha_i = FC_i / sum(FC_j) tells you each component's share of blame. A high alpha_i means component i is the primary bottleneck.

## Step-by-Step Workflow

1. **Map the pipeline topology.** Enumerate every model/module in the compound system. Identify the routing logic: which modules are always invoked, which are conditionally invoked (e.g., a detector triggers specialized classifiers). Document input/output formats for each module.

2. **Instrument execution trace recording.** Wrap each module to capture (module_id, input, output) tuples during inference. Store traces as ordered lists of node records. Ensure the wrapper is architecture-agnostic — treat each module as a black box with known I/O types.

   ```python
   @dataclass
   class TraceNode:
       module_id: str
       input_data: Any
       output_data: Any
       score_fn: Callable  # metamorphic relation scorer

   def record_trace(pipeline, x) -> list[TraceNode]:
       trace = []
       # Run pipeline, intercepting each module call
       for module_id, module_fn in pipeline.modules_in_order(x):
           inp = pipeline.get_input(module_id, x, trace)
           out = module_fn(inp)
           trace.append(TraceNode(module_id, inp, out, pipeline.scorer(module_id)))
       return trace
   ```

3. **Define metamorphic relations for each component.** Write scoring functions S_q(y, y_tilde) -> {0, 1} that encode what "correct under perturbation" means for each module. Common patterns:
   - **Detectors:** (a) detections are subset of originals, (b) IoU > threshold for persisting boxes, (c) class labels unchanged, (d) downstream invocations unchanged
   - **Classifiers:** output label matches reference label
   - **OCR/text models:** Levenshtein distance below threshold tau
   - **Regressors:** absolute difference below epsilon
   - **Embeddings:** cosine similarity above threshold

4. **Select and configure perturbations.** Choose perturbation functions g(x) appropriate to the input modality. For images: use the `imagecorruptions` library (gaussian_noise, fog, frost, motion_blur, etc.) at multiple severity levels (1-5). For text: typo injection, synonym replacement. For audio: noise overlay, speed changes. Apply each perturbation to the full test set.

5. **Run paired inference (clean + perturbed).** For each test input x and perturbation g, run the pipeline twice: once on x to get reference trace T(x), once on g(x) to get perturbed trace T(g(x)). Store both traces for alignment.

6. **Align traces and compute per-component scores.** Match nodes between T(x) and T(g(x)) by module_id. For each matched pair, evaluate S_i(x, x_tilde) using the metamorphic relation. For nodes present in T(x) but missing in T(g(x)) (routing divergence), record S_i = 0.

   ```python
   def compute_component_scores(trace_clean, trace_perturbed):
       scores = {}
       clean_map = {n.module_id: n for n in trace_clean}
       pert_map = {n.module_id: n for n in trace_perturbed}
       for mid, clean_node in clean_map.items():
           if mid in pert_map:
               scores[mid] = clean_node.score_fn(clean_node.output_data, pert_map[mid].output_data)
           else:
               scores[mid] = 0  # routing divergence — component effectively failed
       return scores
   ```

7. **Compute system-level pass/fail.** S(x, x_tilde) = product of all S_i. If any component fails its metamorphic relation, the system fails.

8. **Accumulate Failure Contribution scores across the test set.** For every (x, g) pair where S = 0, accumulate Z_i = 1 - S_i for each component. Divide by total system failures to get FC_i. Normalize to get alpha_i.

   ```python
   def compute_failure_contributions(all_results):
       fc = defaultdict(float)
       total_failures = 0
       for scores, system_passed in all_results:
           if not system_passed:
               total_failures += 1
               for mid, s in scores.items():
                   fc[mid] += (1 - s)
       alpha = {mid: fc[mid] / total_failures for mid in fc} if total_failures > 0 else {}
       total = sum(alpha.values())
       alpha = {mid: v / total for mid, v in alpha.items()} if total > 0 else {}
       return fc, alpha
   ```

9. **Generate the fault attribution report.** Present results as a table: component name, FC_i, alpha_i (attribution weight), broken down by perturbation type and severity. Highlight the component with highest alpha — this is where hardening effort should focus.

10. **Recommend targeted interventions.** Based on attribution weights, recommend specific actions: retrain the highest-alpha component with augmented data matching the most damaging perturbation, add input preprocessing to mitigate specific corruption types, or replace the weakest module with a more robust architecture.

## Concrete Examples

**Example 1: Autonomous Rail Inspection Pipeline**

User: "I have a rail inspection system with a YOLOv4-tiny detector that feeds into 5 specialized classifiers (speed signs, signals, crossings, etc.). Accuracy drops under fog. Which component should I fix?"

Approach:
1. Map the pipeline: f0 (YOLO detector) -> conditional routing -> f1-f5 (classifiers)
2. Instrument each module to record traces
3. Define MRs: for f0, detections subset + IoU > 0.9 + labels unchanged + routing unchanged; for f1-f5, label equality
4. Apply fog perturbation at severity 1-5 from `imagecorruptions`
5. Run paired inference on 500 test images x 5 fog levels = 2500 pairs
6. Compute FC scores

Output:
```
SETA Fault Attribution Report — Fog Perturbation
=================================================
Component         | FC_i   | alpha_i | Failures (of 2500)
------------------|--------|---------|-------------------
f0 (YOLO det.)    | 0.847  | 0.621   | 1,843
f1 (speed_sign)   | 0.198  | 0.145   | 312
f3 (crossing_sig) | 0.214  | 0.157   | 287
f2 (main_signal)  | 0.072  | 0.053   | 104
f4 (catenary)     | 0.033  | 0.024   | 48

Recommendation: f0 (detector) accounts for 62.1% of system failures
under fog. Retrain with fog-augmented data or add defogging
preprocessing before detection.
```

**Example 2: Document Processing Pipeline**

User: "My pipeline has OCR -> NER -> relation extraction. It works on clean scans but fails on noisy faxes. Help me find the bottleneck."

Approach:
1. Map pipeline: f0 (OCR engine) -> f1 (NER model) -> f2 (relation extractor)
2. Define MRs: f0 uses Levenshtein distance <= 2, f1 uses entity set equality, f2 uses relation tuple matching
3. Apply perturbations: Gaussian noise, JPEG compression, brightness variation at multiple severity levels
4. Run on 200 test documents x 3 perturbation types x 3 severities = 1800 pairs

Output:
```
SETA Fault Attribution Report — Noisy Document Input
=====================================================
Component        | FC_i   | alpha_i | By Perturbation (alpha)
-----------------|--------|---------|------------------------
f0 (OCR)         | 0.912  | 0.704   | noise:0.82 jpeg:0.54 bright:0.71
f1 (NER)         | 0.247  | 0.191   | noise:0.12 jpeg:0.31 bright:0.18
f2 (RelExtract)  | 0.136  | 0.105   | noise:0.06 jpeg:0.15 bright:0.11

Recommendation: OCR dominates failures (70.4%), especially under
Gaussian noise (alpha=0.82). Consider a denoising preprocessor or
switch to a noise-robust OCR model.
```

**Example 3: Implementing SETA Harness in Python**

User: "I want to add robustness testing to my image pipeline. Can you write the testing harness?"

Approach: Generate a reusable Python module implementing the full SETA loop.

Output (key structure):
```python
# seta_harness.py
from dataclasses import dataclass, field
from collections import defaultdict
from typing import Any, Callable, Protocol

class MetamorphicRelation(Protocol):
    def __call__(self, y_clean: Any, y_perturbed: Any) -> int: ...

@dataclass
class ModuleSpec:
    module_id: str
    fn: Callable
    relations: list[MetamorphicRelation]

@dataclass
class PipelineSpec:
    modules: list[ModuleSpec]
    router: Callable  # (input, prev_outputs) -> next_module_id

class SETAHarness:
    def __init__(self, pipeline: PipelineSpec, perturbations: list[Callable]):
        self.pipeline = pipeline
        self.perturbations = perturbations

    def run_trace(self, x):
        """Execute pipeline, return trace as dict of module_id -> output."""
        ...

    def score_component(self, module_id, y_clean, y_pert) -> int:
        """Evaluate composite MR for one component."""
        spec = self._get_spec(module_id)
        return int(all(mr(y_clean, y_pert) for mr in spec.relations))

    def evaluate(self, test_data) -> dict:
        """Run full SETA evaluation, return FC and alpha scores."""
        fc = defaultdict(float)
        n_failures = 0
        for x in test_data:
            trace_clean = self.run_trace(x)
            for g in self.perturbations:
                trace_pert = self.run_trace(g(x))
                scores = {}
                system_ok = True
                for mid in trace_clean:
                    s = self.score_component(mid, trace_clean[mid], trace_pert.get(mid))
                    scores[mid] = s
                    if s == 0:
                        system_ok = False
                if not system_ok:
                    n_failures += 1
                    for mid, s in scores.items():
                        fc[mid] += (1 - s)
        alpha = {m: fc[m]/sum(fc.values()) for m in fc} if fc else {}
        return {"fc": dict(fc), "alpha": alpha, "total_failures": n_failures}
```

## Best Practices

- **Do:** Define metamorphic relations that are tight but realistic. An IoU threshold of 0.9 for detector persistence is meaningful; 0.5 is too loose to catch real degradation.
- **Do:** Test across multiple perturbation types and severities. A component robust to noise may be fragile to blur — SETA's per-perturbation breakdown reveals this.
- **Do:** Record routing divergences (when perturbation causes a different subgraph to execute). These are first-class failures in SETA, not just metric differences.
- **Do:** Normalize FC scores to alpha weights for cross-component comparison. Raw FC values depend on how many test cases invoke each component.
- **Avoid:** Relying solely on end-to-end accuracy. A pipeline with 90% accuracy may hide a detector that fails 40% of the time but is compensated by downstream filtering.
- **Avoid:** Applying white-box perturbations (adversarial gradients) across module boundaries. SETA is designed for black-box, input-level perturbations that propagate naturally through the pipeline.

## Error Handling

- **Trace alignment failure:** If the perturbed input causes a completely different execution path (e.g., detector finds zero objects so no classifier runs), handle gracefully by scoring all skipped downstream modules as S_i = 0 and recording the routing divergence.
- **Empty test set for a component:** Some conditional modules may be invoked on very few test inputs. Flag components with fewer than 30 activations as having low-confidence FC scores. Consider stratified sampling to ensure coverage.
- **Non-deterministic models:** If a module produces different outputs on repeated runs of the same input, run multiple reference traces and use majority-vote outputs as the reference. Alternatively, set random seeds and disable dropout at test time.
- **Perturbation incompatibility:** If a perturbation produces out-of-domain inputs (e.g., negative pixel values), clip or normalize before feeding to the pipeline. Document which perturbations are valid for which input domains.

## Limitations

- SETA requires black-box access to each component's inputs and outputs. Fully opaque API-only services where you cannot intercept intermediate representations need proxy wrappers to capture I/O.
- The framework assumes you can define meaningful metamorphic relations for each component. For generative or open-ended models (LLMs, diffusion models), defining binary pass/fail MRs is non-trivial and may require embedding-based similarity thresholds.
- FC scores identify *which* component fails, not *why* it fails. Root-cause analysis within a component still requires component-specific debugging.
- Statistical significance requires sufficient test data. With fewer than ~100 system failures, FC estimates have high variance. Use confidence intervals when reporting.
- SETA doubles inference cost (clean + perturbed runs per input). For large test sets or expensive models, budget computation accordingly or use sampling.

## Reference

**Paper:** [SETA: Statistical Fault Attribution for Compound AI Systems](https://arxiv.org/abs/2601.19337v1) — Chowdhury & D'Souza, CAIN 2026. Look for: the formal state-transition model (Section 3), the FC_i formula and Algorithm 3 (Section 4), and the rail inspection case study with per-perturbation robustness breakdowns (Section 5).