---
name: "acegrpo-adaptive-curriculum-group"
description: "Adaptive curriculum-driven iterative optimization for autonomous ML engineering tasks. Uses Evolving Data Buffers and Learnability Potential sampling from the AceGRPO paper to structure multi-step agent workflows that avoid behavioral stagnation. Triggers: 'optimize ML pipeline iteratively', 'adaptive curriculum for code tasks', 'iterative agent optimization', 'prioritize learning tasks', 'evolving task buffer', 'curriculum-based code improvement'."
---

# AceGRPO: Adaptive Curriculum for Iterative ML Engineering

This skill enables Claude to structure autonomous, multi-step ML engineering workflows using the adaptive curriculum and group-relative optimization principles from the AceGRPO paper. Rather than attempting every sub-task with equal priority, Claude applies Learnability Potential scoring to dynamically rank candidate improvements, maintains an evolving buffer of past attempts to avoid repeating failures, and uses group-relative comparisons to select the most promising next action. The result is sustained iterative optimization that avoids the "behavioral stagnation" trap where agents repeat ineffective strategies.

## When to Use

- When the user asks to **iteratively improve an ML pipeline** (model training, feature engineering, hyperparameter search) across multiple rounds of experimentation.
- When the user wants to **automate a multi-step code optimization workflow** where each step builds on the outcome of prior steps (e.g., "keep improving this model until validation accuracy exceeds 0.92").
- When the user needs to **prioritize which experiments or code changes to try next** from a large candidate set, based on expected learning value rather than random or exhaustive search.
- When the user asks to **debug and fix a failing ML submission** through structured, iterative troubleshooting rather than one-shot fixes.
- When the user wants to **manage an evolving set of task traces** (logs, metrics, code diffs) to inform future optimization decisions.
- When building an **agent loop that must avoid wasting cycles** on tasks that are either trivially solved or currently impossible given the agent's capabilities.

## Key Technique

**The Core Problem.** Standard LLM-based agents tackle ML engineering tasks by generating code, running it, observing results, and trying again. But without a principled way to select *which* task or sub-problem to focus on next, agents waste cycles on tasks that are too easy (no learning signal) or too hard (no progress possible). This is the behavioral stagnation problem: the agent keeps trying the same class of actions because its decision policy never updates.

**AceGRPO's Solution: Two Interlocking Mechanisms.** First, the *Evolving Data Buffer* captures every execution trace -- successful or failed -- and repurposes them as structured training signals. Failed attempts are not discarded; they become data about what doesn't work and under what conditions. The buffer evolves by continuously incorporating new traces while deprioritizing stale or uninformative ones. Second, *Adaptive Sampling via Learnability Potential (LP)* scores each candidate task by combining difficulty (is this within reach?) with improvability (can the agent still get better at this?). The LP function filters to tasks within one standard deviation of the agent's current difficulty frontier and ranks by improvability. This creates a curriculum that automatically shifts as the agent's capabilities change.

**Group Relative Policy Optimization (GRPO).** Instead of scoring each candidate action against an absolute reward baseline, GRPO samples multiple candidate trajectories for each task and computes advantages *relative to the group*: `A(trajectory) = R(trajectory) - mean(R(group))`. This within-group normalization dramatically reduces reward variance across heterogeneous ML tasks (where absolute scores are incomparable between, say, a classification and a regression task), making the selection signal stable and informative.

## Step-by-Step Workflow

1. **Decompose the ML engineering goal into a ranked task list.** Parse the user's objective (e.g., "maximize F1 on this dataset") into discrete sub-tasks: data preprocessing, feature engineering, model selection, hyperparameter tuning, ensemble construction, submission formatting. Write each as an actionable item with a measurable outcome.

2. **Initialize the Evolving Data Buffer.** Create a structured log (JSON or markdown) to track every attempted action, its code, the execution result (metrics, errors, runtime), and a difficulty tag (easy/medium/hard based on whether similar approaches have succeeded before). This buffer persists across iterations.

3. **Score each candidate task with Learnability Potential.** For each pending sub-task, estimate two values:
   - **Difficulty**: How many prior attempts have failed vs. succeeded? Is this within the agent's current capability band (within 1 standard deviation of mean task difficulty)?
   - **Improvability**: How much room remains for improvement? (e.g., current best accuracy is 0.78 vs. theoretical ceiling of 0.95 = high improvability; current best is 0.94 = low improvability).
   Compute `LP = in_difficulty_band(task) * improvability(task)`. Rank tasks by LP descending.

4. **Select the highest-LP task and generate a group of candidate solutions.** For the top-ranked task, produce 3-5 distinct candidate approaches (e.g., different model architectures, different feature sets, different hyperparameter ranges). These form the "group" for relative comparison.

5. **Execute all candidates and record traces.** Run each candidate, capturing full execution traces: code, stdout/stderr, metrics, wall time. Append all traces to the Evolving Data Buffer regardless of success or failure.

6. **Compute group-relative advantages.** For each candidate's result, calculate `advantage = score - mean(group_scores)`. The candidate with the highest positive advantage is selected. If all advantages are near zero, the task may be at a plateau -- deprioritize it.

7. **Update the buffer and recalculate LP scores.** With new trace data, recalculate difficulty estimates (tasks with more failures become harder) and improvability (tasks where the best score just improved have reduced improvability). Re-rank all pending tasks.

8. **Iterate: pick the next highest-LP task.** Repeat steps 4-7 until the user's target metric is met, a budget is exhausted, or all remaining tasks have LP below a threshold (indicating diminishing returns).

9. **Consolidate results.** Merge the best-performing components from across all iterations into a final solution. Report the full optimization trajectory: which tasks were attempted, in what order, and what each contributed.

10. **Archive the buffer for future reuse.** Save the final Evolving Data Buffer so that future optimization sessions on similar problems can bootstrap from prior experience rather than starting cold.

## Concrete Examples

**Example 1: Iterative Kaggle Competition Optimization**

```
User: "I have a tabular classification dataset (train.csv, test.csv). Help me
iteratively improve my submission score on this Kaggle competition. Current
baseline with logistic regression gets 0.72 AUC. Target: 0.88+."

Approach:
1. Decompose into sub-tasks:
   - Feature engineering (missing values, encoding, interactions)
   - Model selection (GBM, random forest, neural net, SVM)
   - Hyperparameter tuning for best model
   - Ensemble construction
   - Submission formatting

2. Initialize buffer:
   {
     "traces": [
       {"id": 1, "task": "baseline", "approach": "logistic_regression",
        "score": 0.72, "difficulty": "easy", "status": "success"}
     ],
     "mean_difficulty": 0.3, "std_difficulty": 0.1
   }

3. Score LP for each sub-task:
   - Feature engineering: difficulty=0.4 (in band), improvability=0.88-0.72=0.16 -> LP=0.16
   - Model selection: difficulty=0.3 (in band), improvability=0.16 -> LP=0.16
   - Hyperparameter tuning: difficulty=0.5 (in band), improvability=0.10 -> LP=0.10
   - Ensemble: difficulty=0.7 (out of band), improvability=0.05 -> LP=0.0

4. Tie between feature engineering and model selection. Pick feature engineering
   (higher expected impact on downstream tasks).

5. Generate group of 4 candidate feature sets:
   - Candidate A: impute + one-hot encoding
   - Candidate B: impute + target encoding + interaction features
   - Candidate C: impute + PCA dimensionality reduction
   - Candidate D: impute + frequency encoding + polynomial features

6. Execute all four, record scores:
   A: 0.74, B: 0.79, C: 0.73, D: 0.76
   Group mean: 0.755
   Advantages: A=-0.015, B=+0.035, C=-0.025, D=+0.005
   -> Select Candidate B (highest advantage)

7. Update buffer, recalculate LP. Feature engineering improvability drops
   (0.88-0.79=0.09). Model selection now has highest LP.

8. Continue iterations: model selection -> hyperparameter tuning -> ensemble.

Output after 4 iterations:
   Round 1: Feature engineering (target encoding) -> 0.79
   Round 2: Model selection (LightGBM) -> 0.84
   Round 3: Hyperparameter tuning (Optuna 50 trials) -> 0.87
   Round 4: Ensemble (LightGBM + CatBoost blend) -> 0.89 [TARGET MET]
```

**Example 2: Debugging a Failing ML Pipeline**

```
User: "My training script crashes after epoch 3 with OOM errors on some runs
but not others. Help me systematically fix this."

Approach:
1. Decompose into diagnostic sub-tasks:
   - Memory profiling (identify peak usage)
   - Batch size analysis (which sizes trigger OOM)
   - Gradient accumulation as workaround
   - Model architecture memory audit
   - Data loader memory leak check

2. Initialize buffer with the crash trace as first entry.

3. Score LP:
   - Memory profiling: difficulty=easy, improvability=high (no data yet) -> LP=high
   - Batch size: difficulty=easy, improvability=medium -> LP=medium
   - Gradient accumulation: difficulty=medium, improvability=low (workaround) -> LP=low

4. Start with memory profiling. Generate 3 candidate approaches:
   A: torch.cuda.memory_summary() at each epoch
   B: pytorch memory snapshot with torch.cuda.memory._record_memory_history()
   C: nvidia-smi polling script every 5 seconds

5. Execute group, compare informativeness (reward = diagnostic clarity):
   A: shows 12GB spike at epoch 3 in attention layer -> advantage=+0.4
   B: pinpoints exact tensor allocation -> advantage=+0.5
   C: shows GPU utilization but not per-tensor -> advantage=-0.9
   -> Select B

6. With root cause identified (attention cache not freed between epochs),
   update buffer. "Model architecture memory audit" now has highest LP
   because we know exactly where to look.

7. Generate fix candidates:
   A: Add torch.cuda.empty_cache() between epochs
   B: Use gradient checkpointing in attention layers
   C: Reduce attention head dimension
   Execute -> B resolves OOM with minimal accuracy loss.

Output:
   Root cause: Attention layer KV-cache retained across epochs
   Fix: Gradient checkpointing (2% slower training, 40% less peak memory)
   Verified: 10 consecutive runs without OOM
```

**Example 3: Curriculum-Ordered Feature Development**

```
User: "I need to add 6 features to my ML service. Some are harder than
others. Help me order them for maximum learning and momentum."

Approach:
1. List features with estimated difficulty:
   - REST API endpoint for predictions (easy)
   - Input validation with schema (easy)
   - Model versioning with rollback (medium)
   - A/B testing framework (hard)
   - Real-time monitoring dashboard (medium)
   - Auto-retraining pipeline (hard)

2. Score LP for each (improvability = business value * technical novelty):
   - REST API: LP = 1.0 * 0.3 = 0.3 (easy, foundational, but low novelty)
   - Input validation: LP = 1.0 * 0.4 = 0.4
   - Model versioning: LP = 0.9 * 0.7 = 0.63
   - A/B testing: LP = 0.0 * 0.9 = 0.0 (out of difficulty band -- too hard now)
   - Monitoring: LP = 0.9 * 0.6 = 0.54
   - Auto-retraining: LP = 0.0 * 0.8 = 0.0 (out of band)

3. Recommended order: Input validation -> Model versioning -> Monitoring
   -> REST API -> (reassess: A/B testing and auto-retraining now in band)
   -> A/B testing -> Auto-retraining

4. After each feature, update difficulty estimates. Completing model
   versioning makes A/B testing easier (shared infrastructure), so its
   difficulty drops into band at reassessment.

Output: Ordered backlog with LP justifications and dependency notes.
```

## Best Practices

- **Do:** Record every execution trace, including failures. Failed attempts are the most informative signal for updating difficulty estimates and avoiding repeated mistakes.
- **Do:** Re-score LP after every iteration. The curriculum must evolve with the agent's capabilities -- a task that was too hard two rounds ago may now be in the sweet spot.
- **Do:** Generate multiple candidates per task (group size 3-5) to enable relative comparison. A single attempt gives no signal about whether the result is good relative to alternatives.
- **Do:** Set explicit stopping criteria (target metric, iteration budget, LP floor) before starting. Without these, iterative optimization can run indefinitely.
- **Avoid:** Spending cycles on tasks with LP near zero -- these are either too easy (already solved) or too hard (need prerequisite work first). Move to the next highest-LP task.
- **Avoid:** Using absolute reward thresholds across different task types. A score of 0.85 means different things for different metrics. Always use group-relative comparisons within a task.

## Error Handling

- **All candidates in a group fail:** This indicates the task difficulty was underestimated. Increase difficulty score, push task out of the current LP band, and select a prerequisite task instead. Log the failure pattern for future reference.
- **Buffer grows too large:** Prune traces older than N iterations or with difficulty estimates that haven't changed in K rounds. Keep at least one trace per unique failure mode.
- **LP scores converge to zero for all tasks:** The agent has hit a capability ceiling. Report current best results to the user and suggest fundamentally different approaches (different model family, additional data, architectural changes) rather than continuing incremental optimization.
- **Execution environment instability (OOM, timeouts):** Treat environment failures as separate from task failures. Do not update task difficulty based on infrastructure issues. Retry with reduced resource usage before scoring.
- **Contradictory traces in buffer:** When the same approach yields different results across runs, flag the task as high-variance. Increase group size to 5+ candidates for more stable relative comparisons.

## Limitations

- **Not suitable for one-shot tasks.** This workflow is designed for iterative optimization. If the user needs a single code generation without iteration, standard prompting is more appropriate.
- **Requires measurable outcomes.** The LP function needs numeric scores or clear success/failure signals. Tasks with subjective quality criteria (e.g., "make the code more readable") lack the reward signal needed for group-relative comparison.
- **Cold start problem.** The first 1-2 iterations have sparse buffer data, making LP estimates unreliable. Bootstrap with reasonable priors (uniform difficulty, high improvability for all tasks) and expect noisy early rounds.
- **Computational cost scales with group size.** Generating and executing 3-5 candidates per task multiplies compute by that factor. For expensive operations (training large models), reduce group size to 2-3 or use lightweight proxy metrics.
- **No guarantee of global optimum.** The curriculum finds a good local trajectory through the task space but may miss non-obvious approaches that require temporarily worse results before improving.

## Reference

**Paper:** [AceGRPO: Adaptive Curriculum Enhanced Group Relative Policy Optimization for Autonomous Machine Learning Engineering](https://arxiv.org/abs/2602.07906v1) (Cai et al., 2026). Key sections: Section 3 for the Evolving Data Buffer and Learnability Potential function; Section 4 for GRPO training details and the group-relative advantage formula; Section 5 for MLE-Bench results showing 100% valid submission rate and ablations demonstrating the 18% gain from LP-based sampling over uniform selection.