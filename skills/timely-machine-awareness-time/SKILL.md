---
name: "timely-machine-awareness-time"
description: "Apply time-budget-aware reasoning to agentic tasks with tool calls. Dynamically adjust strategy depth, tool call frequency, and fallback heuristics based on wall-clock time constraints. Use when: 'optimize this under a time limit', 'run as many experiments as possible in N minutes', 'choose between thorough and fast approaches', 'time-aware agent planning', 'budget my tool calls', 'adapt strategy to deadline'."
---

# Timely Machine: Time-Budget-Aware Agentic Reasoning

This skill enables Claude to treat wall-clock time as a first-class resource when executing multi-step agentic tasks involving tool calls. Rather than planning based solely on task complexity, Claude dynamically adjusts its strategy -- choosing between deeper reasoning per step versus more frequent tool interactions -- based on explicit or inferred time budgets and observed tool latency. The core insight from the Timely Machine paper is that in agentic settings, tool latency decouples inference cost from generation length, so optimal strategy depends on the ratio of tool wait time to reasoning time.

## When to Use

- When the user specifies a time constraint ("do this in under 5 minutes", "I need results before my meeting")
- When a task involves iterative tool calls (running tests, API queries, ML experiments) and you must decide how many iterations to attempt
- When choosing between a thorough multi-step approach and a faster heuristic given finite time
- When orchestrating multiple agents or subprocesses that have variable latency
- When running iterative improvement loops (code optimization, hyperparameter tuning) where each iteration has non-trivial wall-clock cost
- When the user asks to maximize output quality within a fixed compute or time budget
- When tool response times are unpredictable and strategy must adapt on the fly

## Key Technique

**Wall-clock time as the scaling axis.** Traditional test-time scaling measures compute in generated tokens. The Timely Machine framework redefines the budget as total elapsed time: `t_total = sum(t_reasoning) + sum(t_tool_calls)`. This matters because a single slow API call can consume more budget than hundreds of tokens of reasoning. The optimal strategy depends on the *latency ratio*: when tool calls are fast relative to reasoning, prefer many quick interactions to explore broadly; when tool calls are slow, invest more reasoning per interaction to maximize the value of each expensive call.

**Adaptive strategy selection.** The key behavioral principle is: monitor your remaining time budget and adjust depth accordingly. Under generous budgets, pursue thorough multi-step approaches with verification. Under tight budgets, switch to efficient heuristics, skip optional verification, and front-load the highest-value actions. This is not simply "work faster" -- it means structurally changing the approach. For example, choosing a Random Forest over a neural network for an ML task when time is short, or answering a math problem with estimation rather than formal proof when the deadline is near.

**Progressive commitment with checkpoints.** Rather than committing to a full plan upfront, execute in phases with time checks between them. After each phase, evaluate remaining time against remaining work and decide whether to continue the current approach, simplify it, or pivot entirely. This prevents the failure mode where a model spends 90% of its budget on step 1 of a 5-step plan.

## Step-by-Step Workflow

1. **Establish the time budget explicitly.** If the user specifies a deadline or duration, record it. If not, infer a reasonable budget from context (interactive query: seconds; build task: minutes; research task: longer). State your working budget to the user.

2. **Estimate per-step costs.** Before starting, estimate the wall-clock cost of each tool call category you expect to use (e.g., "bash commands ~2s, web fetches ~5s, test suites ~30s"). Use these estimates to determine how many iterations you can afford.

3. **Classify the latency regime.** Determine whether your task is *reasoning-dominated* (most time spent thinking, tool calls are cheap) or *tool-dominated* (most time spent waiting for tool responses). This determines your strategy:
   - **Tool-dominated:** Maximize reasoning quality per call. Plan carefully before each tool invocation. Batch operations where possible.
   - **Reasoning-dominated:** Maximize interaction frequency. Use quick exploratory tool calls to gather information, then reason over results.

4. **Front-load high-value actions.** Execute the actions most likely to yield useful information or progress first. Do not save critical steps for later -- time may run out.

5. **Set checkpoint fractions.** Divide your budget into phases (e.g., 40% exploration, 40% implementation, 20% verification). After each phase, check elapsed time against the budget.

6. **Execute with time monitoring.** After each significant tool call or reasoning block, assess remaining budget. Track cumulative time mentally or via timestamps in your reasoning.

7. **Adapt at checkpoints.** At each checkpoint, make an explicit decision:
   - If ahead of schedule: continue current approach, add optional verification steps.
   - If on schedule: proceed as planned.
   - If behind schedule: simplify remaining steps, drop optional work, switch to faster heuristics.

8. **Implement graceful degradation.** If time is running critically low, produce the best partial result possible rather than failing entirely. A working but unoptimized solution beats no solution.

9. **Report time allocation.** When delivering results, briefly note how time was spent and what tradeoffs were made, so the user can decide whether to invest more time for better results.

## Concrete Examples

**Example 1: Iterative ML experiment under time pressure**

```
User: "I have 3 minutes. Find the best model for this classification dataset.
       Try different approaches and report the best accuracy."

Approach:
1. Budget: 180 seconds. Estimate: data loading ~5s, each model train+eval ~20-40s.
   This allows roughly 4-6 model runs.
2. Latency regime: tool-dominated (training is the bottleneck).
   Strategy: choose models carefully, don't waste runs on unlikely candidates.
3. Phase 1 (0-30s): Load data, inspect shape/types, identify if it's tabular/text/image.
4. Phase 2 (30-150s): Run models in order of expected performance-to-cost ratio:
   - Logistic Regression (fast baseline, ~10s)
   - Random Forest (strong tabular default, ~20s)
   - XGBoost with default params (~25s)
   - If time remains: LightGBM or a tuned XGBoost
5. Checkpoint at 120s: 60s remaining. If 3 models done, try one more.
   If only 2 done, skip further experiments and report.
6. Phase 3 (150-180s): Compile results table, report best model and accuracy.

Output:
"Tested 4 models in 2m 45s. Results:
 | Model              | Accuracy | Time  |
 | Logistic Regression| 0.82     | 8s    |
 | Random Forest      | 0.89     | 18s   |
 | XGBoost            | 0.91     | 24s   |
 | LightGBM           | 0.90     | 22s   |
 Best: XGBoost at 91%. With more time, hyperparameter tuning could improve this."
```

**Example 2: Debugging with fast tool calls**

```
User: "This test is failing. Fix it quickly, I'm about to deploy."

Approach:
1. Budget: tight (user wants speed). Tool calls are fast (running individual tests ~2s).
   Latency regime: reasoning-dominated.
   Strategy: rapid exploratory calls, many quick iterations.
2. Run the failing test immediately to see the error (don't spend time reading
   code first -- the error message is faster signal).
3. Read the error output. Grep for the specific failing assertion or exception.
4. Read only the directly relevant source file and test file (skip tangential code).
5. Identify the fix. Apply it.
6. Re-run the test to verify. If it passes, done.
   If not, read the new error -- iterate but cap at 2-3 cycles.
7. Time check: if >3 failed fix attempts, step back and read more broadly
   rather than continuing to guess.

Output:
"Fixed in 45s. The test expected `status: 'active'` but the migration added
 a default of `status: 'pending'`. Updated the factory default to match.
 Test suite passes."
```

**Example 3: Research task with generous budget**

```
User: "Take your time and thoroughly analyze how authentication works
       in this codebase. I want a complete picture."

Approach:
1. Budget: generous (user explicitly says "take your time").
   Strategy: maximize thoroughness, use verification steps.
2. Phase 1 -- Broad exploration (40% of effort):
   - Search for auth-related files (grep for 'auth', 'login', 'session', 'jwt', 'token')
   - Map the authentication flow from entry point to session creation
   - Identify all middleware, guards, and decorators involved
3. Phase 2 -- Deep analysis (40% of effort):
   - Read each auth component in full
   - Trace the token lifecycle (creation, validation, refresh, revocation)
   - Identify the authorization model (RBAC, ABAC, etc.)
   - Check for security patterns (CSRF, rate limiting, password hashing)
4. Phase 3 -- Synthesis and verification (20% of effort):
   - Cross-reference findings by reading test files for auth components
   - Write up the complete flow with file references
   - Note any potential security concerns

Output:
[Detailed multi-section analysis with file:line references,
 flow diagrams in text, and security observations]
```

## Best Practices

**Do:**
- State your time budget and strategy upfront so the user can course-correct early.
- Track tool call latency from early calls and update your estimates. If the first API call takes 10s instead of expected 2s, immediately revise your plan.
- Front-load actions with the highest information gain. Reading an error message before reading source code is almost always faster.
- Batch independent tool calls in parallel when possible -- this is the single biggest time saver in tool-dominated regimes.
- Produce partial results at checkpoints. If time runs out after step 3 of 5, the user has something rather than nothing.

**Avoid:**
- Do not commit to an expensive multi-step plan without time-checking after step 1. Plans rarely survive contact with real tool latency.
- Do not treat all tool calls as equal cost. A `grep` takes milliseconds; running a full test suite takes minutes. Plan accordingly.
- Do not skip the strategy-selection step. The wrong approach (many cheap calls when you need few careful ones, or vice versa) wastes more time than the strategy selection costs.
- Do not pursue diminishing returns. If you have 90% of the answer with 50% of the budget remaining, report it and ask the user if the remaining 10% is worth the time.

## Error Handling

- **Tool calls taking longer than expected:** After any tool call exceeds 2x your estimated latency, recalculate your remaining budget and simplify the plan. Do not assume subsequent calls will be faster.
- **Time budget exhausted mid-task:** Deliver whatever partial results you have with a clear note on what remains undone. Structure partial output so it's immediately useful (e.g., "3 of 5 tests fixed, remaining 2 are in `auth_test.py` lines 45 and 89").
- **Ambiguous time constraints:** If the user's urgency is unclear, ask once: "Should I optimize for speed or thoroughness?" Then commit to that strategy.
- **Cascading failures eating the budget:** If the first two attempts at a fix fail, stop the current approach entirely. Spend 10% of remaining budget on re-analysis before trying again. Repeated failing tool calls are the most common budget drain.

## Limitations

- This framework is most valuable when tasks genuinely have time pressure or involve expensive tool calls. For simple, fast tasks, the overhead of time monitoring is not worth it.
- Accurate latency estimation requires experience with the specific tools being used. First-time use of unfamiliar APIs will have poor estimates.
- The strategy-switching heuristic (when to pivot from thorough to fast) is judgment-dependent. Over-pivoting produces shallow results; under-pivoting produces timeouts.
- This approach optimizes for wall-clock time, not for quality per se. When the user wants the best possible result regardless of time, standard depth-first reasoning is more appropriate.
- Real-time time tracking in a conversational agent is approximate -- Claude cannot access a system clock between tool calls, so time estimates rely on tool call timestamps and reasoning about elapsed duration.

## Reference

[Timely Machine: Awareness of Time Makes Test-Time Scaling Agentic](https://arxiv.org/abs/2601.16486v1) -- Ma et al., 2026. Key insight: redefining test-time compute as wall-clock time reveals that optimal agentic strategy depends on the tool-latency-to-reasoning-time ratio, and models can be trained (via Timely-RL) to dynamically adapt their depth and interaction frequency to time budgets.