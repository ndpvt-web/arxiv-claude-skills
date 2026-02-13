---
name: "behavioural-representational-evaluation-goal-direc"
description: "Evaluate goal-directedness of LLM agents by combining behavioural benchmarking against optimal policies with interpretability probing of internal representations. Use when: 'evaluate whether my agent is goal-directed', 'probe my model's internal representations', 'benchmark agent behaviour against optimal policy', 'analyze how my LLM agent represents its environment', 'test if my agent actually plans or just pattern-matches', 'audit agent goal-pursuit with interpretability'."
---

# Behavioural and Representational Evaluation of Goal-Directedness

This skill enables Claude to design and implement dual-layer evaluation frameworks for LLM-based agents that combine **behavioural benchmarking** (comparing agent actions against computed optimal policies) with **representational probing** (using classifier probes to decode what the agent internally encodes about its environment, goals, and plans). The technique, from Arghal et al. (2026), reveals that behavioural performance alone is insufficient to characterise how agents pursue objectives — internal representation analysis is required to distinguish genuine goal-directed planning from surface-level pattern matching.

## When to Use

- When the user wants to evaluate whether an LLM agent is genuinely goal-directed versus reactive
- When building an evaluation harness that benchmarks an agent against an optimal or near-optimal policy in a structured environment (grid worlds, mazes, game boards, task graphs)
- When the user asks to probe what an LLM internally represents about environment state, goal locations, or future action plans
- When designing safety evaluations that need to detect whether an agent has coherent internal goals
- When the user wants to understand how chain-of-thought reasoning changes what a model internally encodes
- When comparing multiple agents or model sizes on goal-pursuit quality beyond simple task success rates
- When auditing an agentic system to determine if it maintains consistent internal representations under environment perturbations

## Key Technique

The framework has two complementary evaluation layers. **Behavioural evaluation** measures the agent's action sequences against an optimal policy (computed via dynamic programming, BFS, or A*) across systematic environment variations: grid size, obstacle density, and goal structure complexity. The critical methodological insight is testing robustness to *difficulty-preserving transformations* — environment changes that alter surface features (e.g., rotating the grid, permuting obstacle positions) without changing the optimal path length. A genuinely goal-directed agent should maintain performance under these transformations, while a pattern-matching agent will degrade.

**Representational evaluation** uses probing classifiers trained on the model's hidden-state activations to decode latent information. Three probe targets are extracted: (1) the agent's current position, (2) the goal location, and (3) multi-step action plans. Both linear probes and nonlinear MLP probes are applied — if nonlinear probes significantly outperform linear ones, the model encodes task-relevant information in a compressed, nonlinear manifold rather than a simple linear subspace. The paper finds that LLM agents encode a "coarse spatial map" — approximate but task-relevant geometry that preserves relative positions and goal direction without pixel-perfect fidelity.

A key finding is that **chain-of-thought reasoning reorganises internal representations**, shifting them from encoding broad environment structure toward encoding information that supports immediate next-action selection. This means probing before and after reasoning tokens reveals whether reasoning is actually computational (transforming representations) or merely decorative (leaving representations unchanged).

## Step-by-Step Workflow

1. **Define the evaluation environment.** Implement a structured environment with discrete states, clear goal conditions, and tuneable difficulty parameters. Grid worlds are canonical, but any environment with computable optimal policies works (task DAGs, decision trees, graph navigation). Implement the environment as a class with `reset()`, `step(action)`, `render()`, and `optimal_policy()` methods.

2. **Compute optimal policies.** For each environment configuration, compute the ground-truth optimal action at every reachable state using BFS, Dijkstra, dynamic programming, or A*. Store these as a mapping from state to optimal action. This becomes the behavioural gold standard.

3. **Design difficulty-preserving transformations.** Create environment variants that change surface features without altering optimal path length: rotations, reflections, obstacle position permutations that preserve shortest-path distance, goal relabelling in multi-goal settings. These test whether the agent's competence is robust or brittle.

4. **Run behavioural evaluation.** Execute the agent across a systematic grid of environment parameters (e.g., sizes 4×4 through 12×12, obstacle densities 0%–40%, single vs. multi-goal). At each state visited, record the agent's chosen action and compare it to the optimal action. Compute: (a) **action alignment rate** — fraction of states where agent matches optimal policy, (b) **path optimality ratio** — agent's path length divided by optimal path length, (c) **success rate** — fraction of episodes reaching the goal.

5. **Extract hidden-state activations.** Instrument the model to capture activations at specified layers during inference. For transformer-based LLMs, extract residual stream activations at the last token position across all layers. Save activations paired with ground-truth labels (agent position, goal position, planned next-N actions).

6. **Train probing classifiers.** For each probe target (position, goal, action plan), train both a linear probe (logistic regression) and a nonlinear probe (2-layer MLP with ReLU, hidden dim 256). Use 80/20 train/test splits stratified by environment configuration. Report accuracy, and critically, report the **linear-nonlinear gap** — a large gap indicates the model uses nonlinear encoding for that information.

7. **Probe across reasoning stages.** If the agent uses chain-of-thought, extract activations at three points: (a) after environment description tokens, (b) after reasoning/scratchpad tokens, (c) at the final action token. Compare probe accuracies across stages to determine if reasoning transforms representations toward action-relevant information.

8. **Probe multi-step plans.** Train probes to predict not just the immediate next action but the next 2, 3, and 5 actions. Plot probe accuracy versus planning horizon. A goal-directed agent should show decodable multi-step plans with graceful degradation; a reactive agent will show near-chance accuracy beyond step 1.

9. **Cross-reference behavioural and representational results.** Identify discrepancies: Does the agent choose correct actions despite encoding incorrect positions (lucky heuristics)? Does it encode correct goals but take suboptimal paths (execution failure)? Does representational quality predict behavioural performance across difficulty levels?

10. **Generate evaluation report.** Produce a structured report with: behavioural metrics table across all environment configs, probe accuracy matrices by layer and target, linear-nonlinear gap analysis, reasoning-stage representation shift plots, and a summary diagnosis of the agent's goal-directedness quality.

## Concrete Examples

**Example 1: Evaluating a Grid World Navigation Agent**

User: "I have an LLM agent that navigates a grid world to reach a goal. I want to know if it's actually planning or just doing reactive moves."

Approach:
1. Implement the grid world with configurable size and obstacles
2. Compute optimal policies via BFS for each configuration
3. Run agent across 5×5, 7×7, 9×9 grids with 0%, 15%, 30% obstacle density
4. Extract activations from all transformer layers at each decision point
5. Train position/goal/plan probes on the activations

Output:
```
BEHAVIOURAL EVALUATION
──────────────────────
Grid  | Obstacles | Action Align | Path Ratio | Success
5×5   | 0%        | 0.94         | 1.02       | 1.00
5×5   | 15%       | 0.91         | 1.08       | 0.98
5×5   | 30%       | 0.85         | 1.15       | 0.93
7×7   | 0%        | 0.89         | 1.05       | 0.97
7×7   | 15%       | 0.82         | 1.14       | 0.91
7×7   | 30%       | 0.74         | 1.28       | 0.82
9×9   | 0%        | 0.83         | 1.10       | 0.94
9×9   | 15%       | 0.75         | 1.22       | 0.85
9×9   | 30%       | 0.61         | 1.45       | 0.71

REPRESENTATIONAL PROBES (Layer 18, best performing)
───────────────────────────────────────────────────
Target         | Linear Acc | MLP Acc | Gap
Agent Position | 0.72       | 0.88    | 0.16 (nonlinear encoding)
Goal Location  | 0.68       | 0.85    | 0.17 (nonlinear encoding)
Next Action    | 0.81       | 0.86    | 0.05 (near-linear)
Next 3 Actions | 0.54       | 0.67    | 0.13 (nonlinear encoding)
Next 5 Actions | 0.31       | 0.42    | 0.11

DIAGNOSIS: Agent encodes a coarse spatial map (nonlinear, approximate).
Multi-step plans decodable up to ~3 steps. Performance degrades
gracefully with difficulty — consistent with goal-directed behaviour,
not lookup-table memorization.
```

**Example 2: Comparing Chain-of-Thought vs Direct Action**

User: "Does adding chain-of-thought reasoning actually help my agent plan, or is it just window dressing?"

Approach:
1. Run the same agent in two modes: direct action output and CoT-prompted
2. Extract activations at pre-reasoning and post-reasoning positions for the CoT variant
3. Compare probe accuracies to see if reasoning transforms representations

Output:
```
REASONING STAGE ANALYSIS (7×7 grid, 15% obstacles)
───────────────────────────────────────────────────
                      | Pre-Reasoning | Post-Reasoning | Direct Mode
Position Probe Acc    | 0.70          | 0.86           | 0.73
Goal Probe Acc        | 0.65          | 0.84           | 0.67
Next-Action Probe Acc | 0.58          | 0.89           | 0.76

Behavioural:
  CoT Action Alignment: 0.87   |   Direct Action Alignment: 0.79

FINDING: Reasoning is computationally meaningful — representations
shift from encoding environment structure to encoding action-relevant
information (+16pp position accuracy, +19pp goal accuracy after
reasoning). This is NOT decorative CoT.
```

**Example 3: Difficulty-Preserving Transformation Robustness**

User: "I want to check if my agent generalises or just memorises specific grid layouts."

Approach:
1. Fix a 7×7 grid with specific obstacle placement and optimal path length 12
2. Generate 20 difficulty-preserving variants (rotations, reflections, obstacle permutations) all with optimal path length 12
3. Compare action alignment across the original and all variants

Output:
```
TRANSFORMATION ROBUSTNESS (7×7, path length = 12, N=20 variants)
────────────────────────────────────────────────────────────────
                    | Action Align (mean±std) | Success Rate
Original layout     | 0.86                    | 0.95
Rotated (90°/180°)  | 0.84 ± 0.02            | 0.93
Reflected           | 0.83 ± 0.03            | 0.92
Obstacle permuted   | 0.81 ± 0.04            | 0.89

Variance across transforms: σ = 0.03 (LOW)

DIAGNOSIS: Agent is robust to difficulty-preserving transformations.
Performance drop of <5pp across all transform types indicates genuine
spatial reasoning rather than layout memorisation.
```

## Best Practices

- **Do:** Always compute the ground-truth optimal policy before evaluating — without a gold standard, behavioural metrics are ungrounded. Use BFS/Dijkstra for shortest path, dynamic programming for reward-maximising policies.
- **Do:** Train probes on held-out environment configurations, not just held-out episodes from the same configs. This tests whether representations generalise across environments.
- **Do:** Report the linear-nonlinear probe gap explicitly. A gap > 0.10 indicates the model uses compressed nonlinear encoding; a gap < 0.05 suggests the information is linearly accessible (and thus potentially more manipulable/steerable).
- **Do:** Probe multiple layers and report the layer with highest accuracy. Spatial information often concentrates in middle layers while action-selection information peaks in later layers.
- **Avoid:** Concluding goal-directedness from behavioural metrics alone. An agent can achieve high success rates through simple heuristics (e.g., always move toward the goal ignoring obstacles) without any genuine planning.
- **Avoid:** Using only linear probes. The paper's key finding is that spatial representations are *nonlinearly* encoded — linear probes alone will underestimate the model's representational capacity and produce false negatives about goal-encoding.

## Error Handling

- **Probe accuracy at chance level:** If probes cannot decode position or goal above chance, verify that activations are extracted from the correct layer and token position. Try extracting from the last token of the environment description rather than the action token. Also ensure sufficient training data (at least 500 labelled examples per probe).
- **Behavioural metrics inconsistent with probes:** If the agent acts optimally but probes show poor internal representations, the agent may be using a shortcut strategy (e.g., wall-following) that doesn't require explicit spatial encoding. Design environments that defeat known heuristics.
- **Difficulty-preserving transforms show high variance:** If performance varies wildly across transforms that should preserve difficulty, check that your transforms genuinely preserve optimal path length. Off-by-one errors in obstacle placement can inadvertently change difficulty.
- **Multi-step plan probes degrade too fast:** Expect degradation with horizon — if next-1 accuracy is high but next-2 drops to chance, the agent is reactive. This is a valid finding, not an error. Report it as evidence against multi-step planning.

## Limitations

- The framework is most directly applicable to environments with **computable optimal policies**. For open-ended tasks (creative writing, open-domain dialogue), defining "optimal" is intractable, and the behavioural evaluation layer requires adaptation (e.g., using human preference rankings instead).
- Probing classifiers reveal **correlation, not causation**. High probe accuracy for goal location means the information is present in activations, not that the model *uses* it for decision-making. Causal methods (activation patching, ablation) are needed for stronger claims.
- The "coarse spatial map" finding means probes will never achieve perfect accuracy on fine-grained spatial tasks. Expect 85–92% ceiling for position decoding in grid worlds, not 100%.
- Representational probing requires access to model internals (hidden states). This framework cannot be applied to black-box API-only models without activation access.
- Results from small structured environments (grid worlds) may not directly transfer to complex, real-world agentic tasks. The framework provides a methodology template, but probe targets must be redesigned for each new domain.

## Reference

Arghal, R., Chen, F., Dalton, N., Kortukov, E., & McNamara, C. (2026). *A Behavioural and Representational Evaluation of Goal-Directedness in Language Model Agents.* arXiv:2602.08964v1. Key insight: behavioural evaluation alone cannot distinguish genuine goal-directed planning from reactive heuristics — probing internal representations reveals whether agents encode coarse spatial maps and multi-step plans, and whether reasoning tokens computationally transform these representations toward action-relevant information.