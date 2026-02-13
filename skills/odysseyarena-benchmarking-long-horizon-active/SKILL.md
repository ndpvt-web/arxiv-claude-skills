---
name: "odysseyarena-benchmarking-long-horizon-active"
description: "Design and run inductive agent benchmarks where LLMs must discover hidden rules through long-horizon interaction loops rather than following explicit instructions. Use when the user mentions 'inductive agent evaluation', 'long-horizon benchmarking', 'hidden rule discovery', 'active exploration benchmark', 'OdysseyArena', or 'agent world-model induction'."
---

# OdysseyArena: Long-Horizon Inductive Agent Benchmarking

This skill enables Claude to design, implement, and evaluate **inductive agent benchmarks** based on the OdysseyArena framework. Instead of the standard deductive paradigm (give the agent rules, measure execution), this approach hides the environment's transition laws and forces agents to discover them through strategic interaction. Claude can build observe-hypothesize-act-verify loops, implement the four environment primitives (discrete symbolic, continuous stochastic, periodic temporal, relational graph), detect inductive stagnation via loop-ratio analysis, and interpret benchmark results that distinguish genuine reasoning from rule-following.

## When to Use

- When the user wants to benchmark an LLM agent's ability to **discover hidden rules** rather than follow explicit instructions
- When building an interactive environment where the agent must **infer latent transition dynamics** from observation history
- When designing evaluation tasks with **200+ step horizons** that stress-test agent stability and context management
- When the user asks to compare **inductive vs. deductive agent performance** (rules-hidden vs. rules-provided)
- When implementing an **observe-hypothesize-act-verify** agent loop for any environment with hidden state
- When diagnosing why an agent gets stuck repeating failed actions (**loop-ratio analysis** and inductive stagnation detection)
- When creating benchmarks across the four structural primitives: boolean dependency cascades, noisy stochastic markets, periodic scheduling, or dependency-graph debugging

## Key Technique

**The Inductive Paradigm Shift.** Traditional agent benchmarks operate deductively: the agent receives explicit rules, API docs, or task specifications and is measured on execution quality. OdysseyArena inverts this. The environment's transition function `T(s, a) -> s'` is hidden. The agent observes only partial state `o_t` and must autonomously build an internal world model by systematically probing the environment. This tests a fundamentally different capability -- whether the agent can extract structural regularities from interaction traces, not just follow instructions.

**Four Structural Primitives.** The framework decomposes environment complexity into orthogonal motifs: (1) **Discrete Symbolic Rules** -- boolean dependency networks where toggling one element cascades through propositional formulas; (2) **Continuous Stochastic Dynamics** -- latent factor models with noise where agents must disentangle signal from randomness; (3) **Periodic Temporal Patterns** -- cyclic regularities with hidden period lengths requiring long-range dependency detection; (4) **Relational Graph Structures** -- dependency graphs with non-local interactions requiring topological reasoning. Any real-world inductive benchmark can be classified as a composition of these primitives.

**Diagnosing Failure via Loop Ratio.** The paper's key empirical finding is that agents fail inductively not because tasks are too hard (they succeed when rules are provided), but because they enter **inductive stagnation** -- repeating identical failed actions without updating their hypothesis. The "loop ratio" (fraction of repeated action sequences) is a reliable predictor of failure. Agents with high loop ratios systematically underperform even random baselines. This metric is directly actionable: monitor it in real time to detect when an agent's exploration strategy has collapsed.

## Step-by-Step Workflow

1. **Classify the environment primitive.** Determine which of the four structural motifs applies: discrete symbolic (boolean states, dependency rules), continuous stochastic (numeric states with noise), periodic temporal (cyclic patterns with hidden periods), or relational graph (node/edge dependencies). Many real environments are compositions of multiple primitives.

2. **Define the hidden transition function.** Implement `T(s_t, a_t) -> s_{t+1}` with the latent rules the agent must discover. For discrete symbolic: define propositional formulas per element (e.g., `light_3 = light_1 AND NOT light_5`). For stochastic: define a factor loading matrix `W` with noise `epsilon`. For periodic: set hidden period lengths. For graph: define the dependency DAG.

3. **Build the observation interface.** The agent receives only `o_t` (partial view of `s_t`), never the transition function or its parameters. Design observations to be informative enough that an ideal reasoner *could* infer the rules, but not so transparent that no induction is needed. Include the action taken and the resulting state change.

4. **Implement the interaction loop.** Structure the agent's turn as:
   - **Observe**: Present current environment state as text/structured data
   - **Hypothesize**: Prompt the agent to state its current model of the transition rules
   - **Act**: Agent selects an action (potentially a probe to test a hypothesis)
   - **Verify**: Environment returns new state; agent compares prediction to outcome

5. **Set horizon and termination conditions.** Define maximum steps (120-200+ for challenge tasks), success criteria (all lights on, profit threshold, stability achieved, tests passing), and early termination on safety violations or budget exhaustion. Longer horizons test sustained coherence.

6. **Build the context management strategy.** For long horizons, retain only the action-observation history in the prompt, dropping intermediate chain-of-thought reasoning from prior turns. This prevents context overflow while preserving the empirical trace the agent needs for induction.

7. **Implement the deductive comparison baseline.** Create a parallel version where the transition rules are provided explicitly in the prompt. The gap between rules-provided and rules-hidden performance isolates the inductive bottleneck from raw task difficulty.

8. **Compute metrics: Avg@K and Pass@K.** Run K independent trials per task (K=4 recommended). `Avg@K` = mean success rate across trials. `Pass@K` = fraction of tasks where at least one trial succeeds. Report both -- Avg@K measures consistency, Pass@K measures capability ceiling.

9. **Monitor loop ratio in real time.** Track the fraction of actions that repeat a previously-failed action sequence. Flag when loop ratio exceeds 0.3 -- the agent has likely entered inductive stagnation. Consider injecting a meta-prompt ("You have repeated this action sequence N times without progress. What alternative hypothesis could explain the observed transitions?").

10. **Analyze results across primitives.** Compare performance breakdowns per structural motif. Agents typically succeed at relational graph tasks (familiar from code debugging) but fail at periodic temporal patterns (require long-range memory) and discrete symbolic rules (require systematic hypothesis testing). Use this profile to identify specific inductive weaknesses.

## Concrete Examples

**Example 1: Hidden Light Dependencies (Discrete Symbolic Primitive)**

User: "I want to benchmark whether GPT-4 can figure out hidden boolean rules by interacting with a grid of lights."

Approach:
1. Define a 6-light grid with hidden dependency rules:
   - `L1: toggle freely`
   - `L2: only ON if L1 is ON`
   - `L3: only ON if L1 is OFF and L4 is ON`
   - `L4: toggle freely`
   - `L5: only ON if L2 AND L4 are ON`
   - `L6: only ON if L3 OR L5 is ON`
2. Build an environment that accepts `toggle(N)` actions and returns the full light state
3. Agent sees: `[L1:OFF, L2:OFF, L3:OFF, L4:OFF, L5:OFF, L6:OFF]`
4. Agent must discover rules through experimentation (e.g., toggle L2 -> nothing happens -> infer L2 has a dependency)
5. Run 4 trials, max 50 steps each, measure Avg@4 and Pass@4
6. Compare against rules-provided baseline

Output:
```
Trial Results (rules hidden):
  Trial 1: SUCCESS at step 23 (discovered L2->L1 dependency by step 8)
  Trial 2: FAIL at step 50 (stuck in loop toggling L3, loop_ratio=0.45)
  Trial 3: SUCCESS at step 31
  Trial 4: FAIL at step 50 (never tested L4, missed L5 dependency)

Avg@4: 50.0%  |  Pass@4: 100% (at least one success)
Loop Ratio (mean): 0.28

Baseline (rules provided):
  Avg@4: 100%  |  Pass@4: 100%
  Inductive Gap: 50 percentage points
```

**Example 2: Hidden Market Dynamics (Continuous Stochastic Primitive)**

User: "Create a trading benchmark where the agent has to figure out which stocks are correlated without being told."

Approach:
1. Define 5 stocks driven by 2 hidden factors with loading matrix W and Gaussian noise:
   ```
   price_change = W @ z_t + epsilon, where z_t ~ N(0, I), epsilon ~ N(0, 0.1*I)
   W = [[0.8, 0.1], [0.7, 0.2], [-0.3, 0.9], [-0.2, 0.8], [0.5, -0.5]]
   ```
2. Agent starts with $10,000 portfolio, can buy/sell/hold each stock per turn
3. Agent observes price history and portfolio state but NOT the factor structure
4. Max 120 steps; success = positive return above 5% threshold
5. Track whether agent discovers the factor structure (stocks 1-2 correlated, 3-4 correlated, 5 anti-correlated with both)

Output:
```
Agent Behavior Analysis:
  Steps 1-20:  Random exploration, small positions (hypothesis forming)
  Steps 21-50: Identified stocks 1&2 move together, concentrated position
  Steps 51-80: Missed factor 2 entirely, no exposure to stocks 3&4
  Steps 81-120: Profitable on factor 1, flat on factor 2

  Final Return: +8.3% (pass)
  Factor Discovery: Partial (1 of 2 factors identified)
  Loop Ratio: 0.12 (healthy exploration)
```

**Example 3: Dependency Graph Debugging (Relational Graph Primitive)**

User: "Benchmark an agent's ability to fix a Python project where package dependencies have hidden conflicts."

Approach:
1. Create a Python project with 8 packages where `run.py` fails due to hidden version conflicts:
   - `pkg_A>=2.0` requires `pkg_C<3.0`, but `pkg_B>=1.5` requires `pkg_C>=3.0`
   - `pkg_D` has an undocumented dependency on `pkg_E` not in requirements.txt
2. Agent can execute: `pip install`, `pip uninstall`, `pip show`, `python run.py`, `pip list`
3. Agent sees only error messages and pip output -- never the hidden dependency graph
4. Max 80 steps; success = `python run.py` exits with code 0
5. Measure how many probing actions the agent takes before forming correct dependency model

Output:
```
Agent Trace (condensed):
  Step 1:  python run.py -> ImportError: pkg_E not found
  Step 2:  pip install pkg_E -> Success
  Step 3:  python run.py -> pkg_C version conflict
  Step 5:  pip show pkg_A -> requires pkg_C<3.0
  Step 7:  pip show pkg_B -> requires pkg_C>=3.0
  Step 9:  Hypothesis: need pkg_C version satisfying both (impossible)
  Step 12: pip install pkg_B==1.4 -> downgrades to compatible pkg_C
  Step 14: python run.py -> Success

  Steps to completion: 14/80
  Inductive efficiency: HIGH (minimal redundant probes)
  Loop ratio: 0.0
```

## Best Practices

- **Do:** Always implement the rules-provided deductive baseline alongside the inductive version. The gap between them is the primary signal -- it isolates inductive failure from task difficulty.
- **Do:** Track loop ratio continuously during evaluation. An agent repeating `toggle(3)` five times after it failed each time reveals inductive stagnation that aggregate metrics miss.
- **Do:** Design observations that are *sufficient but not obvious*. The agent should be able to infer the rules from a complete interaction trace, but shouldn't get them for free from a single observation.
- **Do:** Use K>=4 independent trials. Inductive tasks have high variance because initial exploration paths diverge. Both Avg@K (consistency) and Pass@K (capability ceiling) matter.
- **Avoid:** Providing any form of the transition rules in the system prompt, tool descriptions, or environment feedback. Even hints like "lights have dependencies" leak inductive signal. The observation should be the raw state, nothing more.
- **Avoid:** Using very short horizons (<30 steps) for inductive benchmarks. Agents need sufficient interaction budget to form and test hypotheses. Short horizons conflate inductive ability with lucky guessing.
- **Avoid:** Evaluating only final-step success. Analyze the trajectory: when did the agent form correct hypotheses? How many probing actions preceded exploitation? Did exploration degrade into loops?

## Error Handling

- **Agent exhausts step budget without progress:** Check loop ratio first. If >0.4, the agent is stuck. Consider a softer failure mode: partial credit for correctly-inferred sub-rules even if the full task isn't solved.
- **Context window overflow on long horizons:** Implement the paper's context management strategy -- retain only `(action, observation)` pairs, drop intermediate reasoning traces from prior turns. Alternatively, summarize older history into a compressed "discovered rules so far" block.
- **Stochastic environments produce inconsistent results:** Ensure K>=4 trials and fix random seeds across model comparisons. For continuous stochastic primitives, verify the noise magnitude allows signal extraction (SNR > 2 for the latent factors).
- **Rules-provided baseline also fails:** The task is too hard independent of induction. Simplify the environment before drawing inductive conclusions. The deductive baseline must be solvable for the inductive gap to be meaningful.
- **Agent invents plausible but wrong rules:** This is expected and informative. Log the agent's stated hypotheses per step. Track hypothesis revision rate -- agents that never update their model after contradictory evidence are exhibiting a different failure mode than agents that update too aggressively.

## Limitations

- **Not a general agent benchmark.** OdysseyArena specifically tests inductive reasoning. It does not measure tool use fluency, instruction following, multi-modal understanding, or other agent capabilities. Use it as one axis of evaluation, not the only one.
- **Four primitives do not cover all environments.** Real-world tasks often involve social dynamics, adversarial actors, or non-stationary rules that shift mid-episode. The framework assumes the transition function is fixed within an episode.
- **Long-horizon evaluation is expensive.** A single task at 200 steps with 4 trials requires 800 LLM calls. Budget accordingly and use OdysseyArena-Lite (120 tasks, shorter horizons) for initial screening before Challenge-level evaluation.
- **Loop ratio is a symptom, not a diagnosis.** A high loop ratio indicates stagnation but doesn't explain *why* the agent fails to update its hypothesis. Deeper analysis of the reasoning traces is needed for root-cause understanding.
- **Open-source models struggle severely.** The paper shows models below ~70B parameters achieve near-zero performance on inductive tasks. This benchmark is most informative for comparing frontier-class models against each other.

## Reference

**Paper:** [OdysseyArena: Benchmarking Large Language Models For Long-Horizon, Active and Inductive Interactions](https://arxiv.org/abs/2602.05843v1) (Xu et al., 2026). Look for: Table 2 (full model comparison), Figure 4 (inductive vs. deductive gap), Figure 7 (loop ratio correlation with failure), and Appendix A (formal primitive definitions). Code at [github.com/xufangzhi/Odyssey-Arena](https://github.com/xufangzhi/Odyssey-Arena).