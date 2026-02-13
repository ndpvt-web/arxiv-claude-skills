---
name: "exploring-reasoning-reward-agents"
description: |
  Apply Agent Reasoning Reward Model (Agent-RRM) structured critique to improve multi-step agent trajectories. Evaluates tool-use chains with explicit reasoning traces, focused critiques, and process scores. Use this skill when:
  - "Critique this agent's reasoning trace"
  - "Evaluate my tool-calling workflow and find flaws"
  - "Score this multi-step agent trajectory"
  - "Help me build a reward model for agent training"
  - "Improve this agent's reasoning with structured feedback"
  - "Debug why my agent pipeline produces wrong answers"
---

# Agent Reasoning Reward Model (Agent-RRM) for Structured Trajectory Critique

This skill enables Claude to apply the Agent-RRM framework from the Reagent paper to evaluate and improve multi-step agent trajectories. Instead of binary pass/fail judgments on agent outputs, you produce three structured feedback components for any tool-using reasoning chain: (1) a reasoning trace analyzing logical consistency and tool-use patterns, (2) a focused critique identifying specific flaws without revealing answers, and (3) a calibrated process score. These signals can be used to refine agent behavior at inference time (Reagent-C style), augment reward signals for RL training (Reagent-R style), or unify both approaches (Reagent-U style).

## When to Use

- When a user asks you to evaluate an agent's multi-step reasoning trajectory (e.g., a ReAct chain, function-calling sequence, or tool-use log)
- When debugging why an agent pipeline produces incorrect or suboptimal results despite having access to the right tools
- When designing reward functions for agent reinforcement learning and the user needs process-level feedback beyond outcome correctness
- When the user wants to iteratively refine an agent's output by generating a critique and then re-prompting with that critique
- When building evaluation harnesses for agentic systems that use search, code execution, web browsing, or API calls
- When the user asks to score or rank multiple candidate agent trajectories for best-of-N selection

## Key Technique

**The core problem**: Standard agent training uses sparse outcome-based rewards -- the agent either gets the final answer right or wrong. This fails to distinguish between an agent that reasoned well but made one tool error versus one that stumbled into the right answer through flawed logic. Intermediate reasoning quality is invisible to the training signal.

**Agent-RRM's solution**: A reward model that produces structured, multi-faceted feedback for each trajectory. The model outputs three components in sequence: a `<think>` block containing step-by-step analysis of the agent's logical consistency, tool selection appropriateness, and iterative improvement behavior; a `<critique>` block that distills specific, actionable flaws (wrong tool arguments, hallucinated inputs, unverified assumptions, over-reliance on tools when direct reasoning sufficed); and a `<score>` between 0 and 1 representing overall process quality. Critically, the critique never reveals the correct answer -- it only highlights where reasoning went wrong, preserving the agent's ability to self-correct.

**Three integration strategies**: The paper validates three ways to use these signals. **Reagent-C** is training-free: feed the critique back to the agent at inference time and let it produce a refined answer in a second pass. **Reagent-R** combines the model's score with rule-based outcome rewards (R = R_rule + lambda * R_model) for RL training, giving gradient signal on reasoning quality. **Reagent-U** unifies both: during training, the agent generates an initial trajectory, receives critique, generates a refined trajectory, and both are pooled for advantage-normalized optimization. At inference, Reagent-U operates as a standard agent without the critique step, having internalized the self-correction behavior. Reagent-U achieves the strongest results (43.7% on GAIA, 46.2% on WebWalkerQA).

## Step-by-Step Workflow

1. **Collect the full agent trajectory.** Gather the complete chain of thought, tool calls, tool responses, and final answer. Include every intermediate step -- partial reasoning, failed tool calls, retries. The trajectory must be complete; truncated traces produce unreliable evaluations.

2. **Evaluate tool decision appropriateness.** For each step, assess whether the agent correctly decided to invoke a tool versus reason directly. Flag cases of tool over-reliance (calling a search engine for basic arithmetic) and tool under-use (hallucinating facts that should have been looked up).

3. **Check tool argument correctness.** Examine every tool invocation for hallucinated inputs (fabricated filenames, invented URLs, non-existent API parameters), formatting errors (malformed JSON, wrong parameter types), and irrelevant tool selection (using a calculator when a web search was needed).

4. **Assess tool output handling.** Determine whether the agent properly interpreted tool responses. Flag blind acceptance of noisy or incomplete outputs, failure to handle errors or empty results, and incorrect extraction of relevant information from tool responses.

5. **Analyze iterative improvement behavior.** Check whether the agent corrected earlier mistakes in subsequent steps, verified hypotheses against new evidence, and avoided repeating the same failed approach. Good trajectories show learning within the episode.

6. **Detect fabrications.** Identify any invented facts, file paths, object identifiers, or data points that the agent produced without tool verification. This is distinct from tool argument errors -- fabrication is presenting unverified information as established fact.

7. **Produce the `<think>` reasoning trace.** Write a structured analysis covering each dimension above. Be specific: reference exact steps by number, quote the problematic tool calls, and explain the logical gap. This trace must be transparent enough that a developer can locate the exact failure point.

8. **Produce the `<critique>` block.** Distill the analysis into 2-5 concise, actionable findings. Each finding should name the flaw, locate it in the trajectory, and suggest what the agent should have done differently. Do NOT reveal the correct final answer or provide hints toward it.

9. **Assign the `<score>` (0-1).** Score based on process quality, not outcome correctness. A trajectory that reasons well but reaches a wrong answer due to a single tool failure scores higher than one that stumbles into the right answer through flawed logic. Use the full range: 0.0-0.2 (fundamentally flawed reasoning), 0.3-0.5 (major logical gaps), 0.5-0.7 (sound reasoning with notable errors), 0.7-0.9 (strong reasoning with minor issues), 0.9-1.0 (excellent process throughout).

10. **Apply the appropriate integration strategy.** For inference-time improvement (Reagent-C): feed the critique back and generate a refined response. For RL training data: use the score as a process reward signal combined with outcome reward. For unified training (Reagent-U): generate both initial and critique-refined trajectories as paired training examples.

## Concrete Examples

**Example 1: Evaluating a ReAct Agent Trajectory**

User: "Evaluate this agent trajectory that tried to answer 'What is the population of the capital of the country that won the 2022 FIFA World Cup?'"

```
Agent Trajectory:
Step 1: Thought: I need to find who won the 2022 FIFA World Cup.
        Action: search("2022 FIFA World Cup winner")
        Observation: Argentina won the 2022 FIFA World Cup.
Step 2: Thought: The capital of Argentina is Buenos Aires. I know this.
        Action: search("population of Buenos Aires")
        Observation: The population of Buenos Aires is approximately 3.1 million (city proper).
Step 3: Answer: The population is 3.1 million.
```

Approach:
1. Verify tool decision at Step 1 -- appropriate, factual recall could be wrong.
2. Flag Step 2 -- agent correctly identified Buenos Aires but did NOT use a tool to verify. This is acceptable since it's common knowledge, but risky.
3. Check Step 2 tool use -- search was appropriate for population data.
4. Assess answer -- 3.1 million is city proper; metropolitan area is ~15 million. Agent didn't clarify which figure.

Output:
```
<think>
Step 1: Appropriate tool use. The agent correctly searched for the World Cup
winner rather than relying on potentially outdated training data.

Step 2: The agent assumed Buenos Aires is Argentina's capital without
verification. While correct, this represents an unverified assumption in a
chain where the previous step was tool-verified. Inconsistent verification
standards. The population search was appropriate.

Step 3: The agent reported "3.1 million" without specifying city proper vs.
metropolitan area. The search result likely contained both figures, but the
agent extracted only one without noting the ambiguity.
</think>

<critique>
1. AMBIGUOUS EXTRACTION (Step 3): The agent reported a single population
   figure without clarifying whether it refers to city proper or metropolitan
   area. Should have noted the distinction or asked for clarification.
2. INCONSISTENT VERIFICATION (Step 2): Capital identification was assumed
   while other facts were tool-verified. Minor risk, but breaks the
   verification chain.
</critique>

<score>0.72</score>
```

**Example 2: Critiquing a Code-Execution Agent**

User: "Score this agent that tried to find the most common word in a CSV file."

```
Agent Trajectory:
Step 1: Thought: I'll read the CSV and count words.
        Action: execute_code("import pandas as pd; df = pd.read_csv('data.csv')")
        Observation: DataFrame loaded, 5000 rows x 3 columns.
Step 2: Thought: I'll count words in all text columns.
        Action: execute_code("from collections import Counter; words = ' '.join(df.values.flatten().astype(str)).split(); Counter(words).most_common(5)")
        Observation: [('the', 12453), ('and', 8921), ('of', 7234), ('to', 6102), ('in', 5890)]
Step 3: Answer: The most common word is "the" with 12,453 occurrences.
```

Approach:
1. Analyze data handling -- agent flattened ALL columns including numeric ones.
2. Check for stop word filtering -- agent reported "the" as most common, which is a stop word.
3. Assess whether the user likely wanted content words, not function words.

Output:
```
<think>
Step 1: Appropriate tool use. Loading with pandas is reasonable for CSV data.

Step 2: Critical flaw. The agent flattened ALL columns including numeric ones
via .values.flatten().astype(str), converting numbers to strings and counting
them as "words." Additionally, no stop word filtering was applied. The result
is dominated by English function words (the, and, of, to, in) which are almost
certainly not what the user intended by "most common word."

Step 3: The agent reported the raw result without questioning whether stop
words should be excluded or whether numeric columns should be included. No
iterative improvement -- the agent accepted the first result without critical
evaluation.
</think>

<critique>
1. NO STOP WORD FILTERING (Step 2): Results are dominated by function words.
   The agent should have either filtered stop words or asked the user whether
   they wanted content words only.
2. NUMERIC COLUMN INCLUSION (Step 2): All columns were flattened to strings,
   meaning numeric values were tokenized as "words." Should have identified
   text columns first using dtype checks.
3. NO SELF-CORRECTION (Step 3): The agent accepted obviously generic results
   without questioning their utility. A strong agent would recognize that
   "the" being the top word is uninformative and refine the approach.
</critique>

<score>0.35</score>
```

**Example 3: Applying Reagent-C Refinement**

User: "Use critique-based refinement to improve my search agent's answer."

```
Original agent output: "The Eiffel Tower is 300 meters tall."
(Agent used search but pulled an outdated figure -- actual height with
antenna is 330 meters since 2022.)
```

Approach:
1. Generate the three-component feedback for the original trajectory.
2. Feed the critique back as context for a refinement pass.
3. Produce the refined answer incorporating the critique's guidance.

Output:
```
--- Critique feedback (Reagent-C input) ---
<critique>
1. OUTDATED INFORMATION (Step 2): The agent reported 300m which is the
   original 1889 construction height. The current height including the
   radio antenna is 330m. The search result likely contained both figures
   but the agent extracted the historical one.
2. MISSING CONTEXT: Height figures for structures typically distinguish
   between structural height and total height with antenna. Agent should
   specify which measurement.
</critique>

--- Refined output (after critique injection) ---
"The Eiffel Tower's structural height is 300 meters (its original 1889
height). Including the radio antenna added in 2022, its current total
height is 330 meters."
```

## Best Practices

- **Do:** Evaluate process quality independently of outcome correctness. A trajectory that reasons well but hits a tool failure is more valuable than lucky guessing. Score the reasoning, not the result.
- **Do:** Be specific in critiques. "Step 3 used a fabricated filename `report_2024.pdf` that was not returned by any previous tool call" is actionable. "The agent made errors" is not.
- **Do:** Maintain the no-answer-leakage principle in critiques. Point out WHERE reasoning went wrong without revealing WHAT the correct answer is. This preserves the agent's ability to self-correct.
- **Do:** Use the full 0-1 scoring range with calibrated thresholds. Avoid clustering all scores around 0.5 -- differentiate clearly between fundamentally flawed (0.1-0.2) and strong-with-minor-issues (0.7-0.8) trajectories.
- **Avoid:** Evaluating only the final answer. The entire point of Agent-RRM is process-level feedback. A correct final answer from a flawed process should score lower than an incorrect answer from sound reasoning.
- **Avoid:** Generic critiques that apply to any trajectory. Each critique point must reference a specific step, tool call, or reasoning gap in the actual trajectory being evaluated.

## Error Handling

- **Incomplete trajectories**: If the trajectory is truncated (agent timed out, hit token limit), evaluate only the available steps but note the truncation in the `<think>` section and reduce the score proportionally. Do not hallucinate what the agent "would have done."
- **Missing tool outputs**: If tool responses are redacted or unavailable, flag this in the reasoning trace and evaluate the agent's tool selection and argument construction rather than output handling. Note reduced confidence in the score.
- **Ambiguous user intent**: When it's unclear whether the user wants content-word frequency vs. all-word frequency (as in Example 2), the critique should flag the ambiguity rather than assuming one interpretation is "wrong." Strong agents ask clarifying questions.
- **Multi-agent trajectories**: When evaluating systems with multiple cooperating agents, produce separate feedback for each agent's contribution. Cross-agent coordination quality (handoff clarity, information passing) gets its own section in the `<think>` block.

## Limitations

- **No ground truth required, but calibration suffers without it.** Agent-RRM evaluates process quality without needing the correct answer, but score calibration is more reliable when outcome correctness can anchor the scale. For pure process evaluation, expect wider score variance.
- **Single-turn tool use is poorly differentiated.** The framework excels at multi-step trajectories (5+ steps) where reasoning patterns emerge. For simple single-tool-call tasks, the critique adds little value over direct answer checking.
- **Domain-specific tool semantics require context.** Evaluating whether a SQL query is well-constructed or a shell command is safe requires domain knowledge beyond generic trajectory analysis. Provide tool documentation when evaluating specialized toolchains.
- **Reagent-U training requires paired data.** The unified approach needs both initial and critique-refined trajectories for the same query. This doubles data collection cost compared to standard RL training.
- **Critique quality degrades for highly specialized domains.** When the trajectory involves niche tools (e.g., bioinformatics pipelines, hardware simulation), the critique may flag false positives. Supplement with domain expert review.

## Reference

[Exploring Reasoning Reward Model for Agents](https://arxiv.org/abs/2601.22154v1) -- Fan et al., 2026. Focus on Section 3 (Agent-RRM architecture and the three-component feedback structure), Section 4 (Reagent-C/R/U integration strategies), and Appendix A.3 (annotation prompt template for the evaluation checklist). Code and models at [github.com/kxfan2002/Reagent](https://github.com/kxfan2002/Reagent).