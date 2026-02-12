---
name: "se-bench-benchmarking-self-evolution-knowledge"
description: "Design knowledge-internalization benchmarks and closed-book training pipelines for LLM self-evolution. Use when: 'build a benchmark for API knowledge retention', 'test if my model memorized a new library', 'create an obfuscated API evaluation', 'design closed-book vs open-book training experiments', 'evaluate knowledge internalization with SFT vs RL', 'build a self-play curriculum for learning new APIs'."
---

# SE-Bench: Benchmarking Self-Evolution with Knowledge Internalization

This skill enables Claude to design and implement rigorous benchmarks that measure whether language models have truly internalized new knowledge (APIs, libraries, domain vocabularies) into their weights, versus merely retrieving it from context. It applies the SE-Bench methodology: obfuscate a known library into a pseudo-novel package with randomized identifiers, then evaluate models on tasks that are trivial with documentation but impossible without it. The three core insights -- Closed-Book Training, the RL Gap, and Self-Play viability -- provide a concrete recipe for building knowledge-internalization pipelines.

## When to Use

- When the user wants to **benchmark whether a fine-tuned model has memorized a new API** rather than just copying from prompts
- When designing **training pipelines that force parametric knowledge retention** (closed-book SFT)
- When the user asks to **obfuscate an existing library** into a pseudo-novel package for controlled evaluation
- When comparing **SFT vs RL for knowledge acquisition** and needing to understand why RL alone fails
- When building a **self-play curriculum** where a model generates its own training tasks for a new domain
- When the user needs to **measure knowledge contamination** -- determining if "new" knowledge already exists in pre-training data
- When evaluating **compositional generalization** -- whether a model can combine individually learned functions

## Key Technique

**The Obfuscation-Based Diagnostic.** SE-Bench solves two entanglement problems that plague naive knowledge benchmarks. First, *prior knowledge entanglement*: if you test a model on NumPy, you cannot tell whether it learned NumPy from your training data or from pre-training. SE-Bench resolves this by mapping all 268 common NumPy functions to randomized, semantically meaningless identifiers (e.g., `numpy.mean` becomes `zwc.kocito`) and wrapping all inputs/outputs in a custom `ZWCArray` class that prevents fallback to standard array methods. Second, *reasoning complexity entanglement*: if a model fails a hard task, you cannot tell whether it forgot the API or simply cannot reason. SE-Bench uses only simple coding tasks (single-function and basic multi-function compositions) so that failure cleanly signals missing knowledge, not insufficient reasoning.

**Closed-Book Training.** The paper's most actionable finding is the Open-Book Paradox: training with documentation in the prompt during parameter updates yields 0% retention, because the model learns to copy from context rather than compress knowledge into weights. Closed-Book Training fixes this by providing documentation during rollout/trajectory collection (so the model can generate correct solutions) but *stripping documentation during the actual gradient update*. This forces the loss function to reward parametric recall. Closed-SFT achieves 39.6% on single-function tasks where Open-SFT achieves 0%.

**The RL Gap and Hybrid Pipeline.** Standard RL (PPO) cannot internalize new vocabulary because clipping prevents the large probability shifts needed for novel tokens, and negative gradients from normalized advantages erase tentative associations before they solidify. However, RL excels at *refining* already-internalized knowledge -- reducing hallucinated attribute errors from 37% to 10%. The optimal pipeline is therefore Closed-SFT (for acquisition) followed by RL (for consolidation), achieving 54.4% combined.

## Step-by-Step Workflow

1. **Select the target library and scope.** Choose the library to obfuscate (e.g., NumPy, pandas, a custom SDK). Enumerate all public functions to be included -- SE-Bench covers 268 NumPy functions. Define the boundary: which functions are in-scope, which are excluded.

2. **Generate randomized identifiers.** For each function, generate a semantically meaningless replacement name (random syllable combinations like `kocito`, `vrelam`, `dopzin`). Map the package name itself to a novel identifier (e.g., `numpy` to `zwc`). Store the full mapping as a JSON dictionary for reproducibility.

3. **Create wrapper types to prevent bypass.** Implement a custom wrapper class (analogous to `ZWCArray`) that wraps all inputs and outputs. This class must NOT expose the original library's methods -- if a user calls `.mean()` on a `ZWCArray`, it should raise an error, forcing use of the obfuscated functional API `zwc.kocito(arr)` instead.

4. **Rewrite documentation under new identifiers.** Use an LLM (or scripted find-replace plus LLM polish) to transform the original API docs into docs for the pseudo-novel package. Replace all function names, parameter names, and cross-references. Verify every code example in the rewritten docs actually executes against the obfuscated package.

5. **Generate evaluation tasks with stratification.** Create two tiers: (a) single-function tasks ensuring 100% API coverage (one task per function minimum), and (b) multi-function composition tasks requiring 3+ functions. Use an LLM to draft tasks, then filter: at least 3 strong baseline models must independently solve each task using the *original* library to confirm the task is solvable and not ambiguous.

6. **Split into train/test with held-out compositions.** Training set contains single-function tasks only. Test set contains both single-function (retention test) and multi-function (compositional generalization test). This separation ensures you measure both memorization and transfer.

7. **Implement the Closed-Book Training pipeline.** Collect training trajectories with documentation in the prompt (open-book rollouts). Then train with SFT on these trajectories but with documentation *removed* from the input. The model sees `[task description] -> [correct solution]` without any API reference, forcing weight-based memorization.

8. **Optionally add RL consolidation.** After Closed-SFT, run a short RL phase (PPO or similar) on the same tasks with documentation stripped. This consolidation phase reduces hallucination errors (e.g., calling non-existent methods on wrapper objects) without needing to internalize new facts.

9. **Evaluate with triple verification.** For each test task, verify: (a) test cases pass, (b) AST analysis confirms the solution uses the obfuscated API (not the original library), (c) no imports of the original library appear. Score binary: pass or fail.

10. **Analyze failure modes.** Categorize failures into: unknown function (model never learned it), hallucinated attribute (model calls `.method()` on wrapper type), incorrect function choice (model confuses similar obfuscated names), and composition failure (individual functions known but combination fails).

## Concrete Examples

**Example 1: Building an obfuscated API benchmark for pandas**

User: "I want to test whether my fine-tuned model actually learned our internal data processing library. Can you help me build a benchmark?"

Approach:
1. Identify the library scope -- suppose 150 public functions in the internal library
2. Generate a mapping file:
```json
{
  "package": {"original": "dataproc", "obfuscated": "qvx"},
  "functions": {
    "dataproc.read_csv": "qvx.belnor",
    "dataproc.merge_tables": "qvx.fripaz",
    "dataproc.aggregate": "qvx.tunwek"
  },
  "wrapper_type": "QVXFrame"
}
```
3. Implement the wrapper package:
```python
class QVXFrame:
    """Opaque wrapper -- no direct method access."""
    def __init__(self, _inner):
        self._inner = _inner
    def __getattr__(self, name):
        raise AttributeError(f"QVXFrame has no attribute '{name}'. Use qvx.* functions.")

def belnor(path):
    """Read a CSV file and return a QVXFrame."""
    import dataproc
    return QVXFrame(dataproc.read_csv(path))
```
4. Generate 150 single-function tasks + 80 multi-function tasks
5. Filter with 3 baseline models solving against original `dataproc`
6. Split: 150 train (single), 80 test (multi) + 50 held-out single

Output: A benchmark suite with obfuscated package, rewritten docs, stratified task set, and evaluation harness.

**Example 2: Designing a Closed-Book SFT training run**

User: "My model scores 0% on the obfuscated API test even after fine-tuning. What's wrong?"

Approach:
1. Diagnose: check if training was Open-Book (docs in prompt during gradient updates) -- this is the Open-Book Paradox
2. Restructure the pipeline:
```
Phase 1 - Rollout Collection (Open-Book):
  Input:  [API docs] + [task description]
  Output: [correct solution using obfuscated API]
  -> Save (task, solution) pairs

Phase 2 - SFT Training (Closed-Book):
  Input:  [task description]  # NO docs
  Target: [correct solution]
  -> Standard cross-entropy loss
```
3. Train for 3-5 epochs on Closed-Book data
4. Expect ~40% single-function accuracy (baseline from SE-Bench on 8B models)
5. If hallucination rate is high (>30%), add RL consolidation phase

Output: A training config that separates trajectory collection (with docs) from gradient updates (without docs), resolving the 0% retention problem.

**Example 3: Self-play curriculum for a new domain**

User: "I don't have human-written tasks for our new API. Can the model generate its own training data?"

Approach:
1. Give the model the obfuscated API documentation
2. Prompt it to generate coding tasks that exercise the API:
```
Given this API documentation for package 'qvx', generate 10 coding
tasks. Each task should require exactly one qvx function. Include
input/output examples. Tasks should be simple and unambiguous.
```
3. Filter generated tasks: run each through 2+ models with docs to verify solvability
4. Collect solutions via open-book rollouts
5. Train with Closed-Book SFT on the self-generated curriculum
6. Expect ~22% accuracy (vs ~40% on curated data) -- viable but noisier

Output: A self-play pipeline that bootstraps training data from documentation alone, no human task authoring required.

## Best Practices

- **Do:** Always strip documentation from training inputs during gradient updates. The Closed-Book constraint is the single most important design choice -- without it, models learn to copy rather than memorize.
- **Do:** Wrap all data types in opaque containers that block method access. If the wrapper leaks `.mean()` or `.shape`, models will bypass the obfuscated API entirely, invalidating your benchmark.
- **Do:** Use triple verification (test cases + AST check + import check) for evaluation. Test cases alone cannot catch solutions that secretly call the original library.
- **Do:** Start with SFT for knowledge acquisition, then optionally add RL for consolidation. Never use RL alone for internalization -- it cannot create new associations, only refine existing ones.
- **Avoid:** Testing on tasks that require complex reasoning. The benchmark must isolate knowledge retention from reasoning ability. If a task is hard even *with* documentation, failures are ambiguous.
- **Avoid:** Using the same function compositions in train and test sets. Single-function training with multi-function testing is the clean way to measure compositional generalization.

## Error Handling

| Problem | Cause | Fix |
|---|---|---|
| 0% retention after fine-tuning | Open-Book Training (docs in gradient updates) | Switch to Closed-Book: strip docs during SFT |
| High hallucination rate (>30%) | SFT creates noisy probabilistic memory | Add RL consolidation phase after Closed-SFT |
| Model calls original library functions | Wrapper type leaks methods or imports not blocked | Audit wrapper `__getattr__`, add import-blocking in eval harness |
| Baseline models fail filtered tasks | Task is ambiguous or under-specified | Require 3+ independent model solutions; add human verification |
| Self-play tasks are too easy/trivial | Model generates tasks it can already solve | Add difficulty constraints: require multi-step tasks, edge cases |
| RL training shows no improvement | PPO clipping blocks probability shifts for novel tokens | Increase clip range cautiously, or rely on SFT-only pipeline |

## Limitations

- **Scale-dependent results.** SE-Bench findings are validated on 8B-parameter models. Larger models may exhibit different retention curves -- the Open-Book Paradox may weaken at scale if models develop better in-context compression.
- **Limited to API-style knowledge.** The obfuscation technique works well for function-based APIs but does not directly apply to conceptual knowledge (e.g., "understand relativity") or procedural knowledge (e.g., "debug a memory leak").
- **Randomized identifiers are artificial.** Real-world new knowledge has semantic names that provide hints. SE-Bench's meaningless identifiers create a harder-than-real setting, which may overestimate forgetting in practical scenarios.
- **Single-library scope.** Cross-library interactions, dependency chains, and ecosystem-level knowledge are not tested. A model might internalize one package but fail when it must coordinate with other learned packages.
- **Self-play ceiling.** Self-generated curricula recover only ~57% of curated-data performance. For high-stakes applications, human-authored or LLM-filtered task sets remain superior.

## Reference

**Paper:** [SE-Bench: Benchmarking Self-Evolution with Knowledge Internalization](https://arxiv.org/abs/2602.04811v1) (Yuan et al., 2026). Look for: Section 4 on Closed-Book vs Open-Book training (the core actionable insight), Section 5 on the RL Gap ablations (why PPO clipping fails), and Section 6 on Self-Play viability. Code and dataset: [github.com/thunlp/SE-Bench](https://github.com/thunlp/SE-Bench).