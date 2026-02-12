---
name: "adareasoner-dynamic-tool-orchestration"
description: "Adaptive multi-step tool orchestration for complex reasoning tasks. Dynamically selects, sequences, and composes tools based on task context and intermediate results rather than fixed pipelines. Use when: 'orchestrate tools for this task', 'figure out which tools to use', 'multi-step reasoning with tools', 'adaptive tool pipeline', 'dynamic tool selection', 'chain tools together intelligently'."
---

# AdaReasoner: Dynamic Tool Orchestration for Iterative Reasoning

This skill teaches Claude to orchestrate multiple tools adaptively across multi-step reasoning chains, inspired by the AdaReasoner framework (arXiv:2601.18631v2). Instead of following rigid, pre-defined tool pipelines, Claude learns to select tools based on task context, evaluate intermediate results, suppress irrelevant tools, adopt beneficial ones mid-chain, and backtrack when a tool call fails or produces unhelpful output. The core insight: treat tool use as a general reasoning skill where each invocation is conditioned on cumulative observations, not as a fixed mapping from task type to tool sequence.

## When to Use

- When a user's request requires chaining 2+ tools in sequence where later tool choices depend on earlier results (e.g., "extract data from this screenshot, clean it, then visualize it")
- When the optimal set of tools is not obvious upfront and must be discovered through exploration (e.g., "analyze this codebase for performance issues" — might need profiling, grep, AST parsing, benchmarking)
- When building an agent workflow that must handle tool failures gracefully by falling back to alternative tools or intrinsic reasoning
- When a task involves perception-planning-verification loops (e.g., "navigate this file structure to find and fix all related bugs")
- When the user asks to "figure out the best approach" for a multi-step technical problem requiring different capabilities at each stage
- When composing unfamiliar or newly-available tools where no pre-built pipeline exists

## Key Technique

**Adaptive Tool Orchestration** replaces static tool pipelines with a dynamic reasoning loop. At each step, the agent observes the current state (task description + all prior tool outputs), decides whether to invoke a tool or reason directly, selects which tool based on functional relevance (not name memorization), and evaluates the result before deciding the next action. This mirrors AdaReasoner's state-action-observation tuples: `trajectory = {(s0, a0, o0), (s1, a1, o1), ..., (sT, aT, oT)}` where each action is conditioned on the full history.

**Three emergent behaviors** distinguish this from naive tool chaining: (1) **Tool Adoption** — when a tool proves beneficial mid-chain, increase its usage frequency; (2) **Tool Suppression** — when a tool adds no value or introduces noise, stop calling it even if it seems topically relevant; (3) **Frequency Modulation** — adjust how many times a tool is called based on task complexity (a simple query needs one search; a complex investigation needs iterative refinement). These behaviors arise naturally from evaluating intermediate results rather than from explicit rules.

**Backtracking and Fallback** is critical. When a tool returns an error, produces garbage, or yields results that contradict other evidence, the agent must recognize this, discard the failed result, and either retry with different parameters, switch to an alternative tool, or fall back to direct reasoning. The data curation pipeline in AdaReasoner explicitly trains on failure trajectories — this skill encodes that same resilience pattern.

## Step-by-Step Workflow

1. **Decompose the task into observable subgoals.** Break the user's request into a sequence of intermediate states, each with a clear success criterion. Write these as a checklist. Example: for "find performance bottlenecks in this API," subgoals are: identify endpoint handlers → profile each → rank by latency → trace root causes → propose fixes.

2. **Inventory available tools and assess relevance.** List every tool/capability available (bash, grep, read, web fetch, code execution, etc.). For each subgoal, rate which tools *could* help — but do not commit to a fixed mapping yet. Focus on functional descriptions, not tool names.

3. **Execute the first subgoal with the most promising tool.** Pick the tool most likely to produce useful output for the first subgoal. Invoke it with specific, well-formed parameters. Capture the full output.

4. **Evaluate the intermediate result against the subgoal criterion.** Ask: Did this tool call succeed? Is the output complete, partial, or erroneous? Does it change my understanding of subsequent subgoals? Score the result: fully satisfactory, partially useful, or failed.

5. **Adapt the plan based on observations.** If the result is fully satisfactory, advance to the next subgoal. If partially useful, decide whether to re-invoke with refined parameters or supplement with another tool. If failed, backtrack: try an alternative tool, adjust the subgoal decomposition, or fall back to direct reasoning.

6. **Apply tool suppression for irrelevant capabilities.** If a tool was invoked but added no value (e.g., a web search returned nothing useful for a local code question), mark it as suppressed for this task. Do not re-invoke it unless the task context shifts significantly.

7. **Apply tool adoption for newly-valuable capabilities.** If an intermediate result reveals that a tool not originally planned is now relevant (e.g., discovering a CSV file mid-analysis means a data-parsing tool is now useful), adopt it into the active plan.

8. **Modulate tool frequency by complexity.** For simple subgoals, one tool call suffices. For complex subgoals (e.g., searching a large codebase), iterate the same tool with progressively refined queries — but set a maximum iteration count (typically 3-5) to avoid infinite loops.

9. **Verify the final result against the original task.** After all subgoals are addressed, synthesize the accumulated observations into a coherent answer. Cross-check: does the final output actually answer what the user asked? If gaps remain, run one targeted verification step.

10. **Report the reasoning chain transparently.** Show the user which tools were used at each step, why alternatives were chosen or rejected, and where backtracking occurred. This builds trust and allows the user to refine the approach.

## Concrete Examples

**Example 1: Diagnosing a failing test in an unfamiliar codebase**

```
User: "This test is failing but I have no idea why. Can you figure it out? tests/integration/test_auth.py::test_login_redirect"

Approach:
1. Decompose: Read the failing test → Identify what it asserts → Find the code under test →
   Trace the execution path → Identify the discrepancy → Propose a fix.
2. Inventory: Read (files), Grep (search), Bash (run test with verbose output), Glob (find files).
3. Step 1 — Read the test file to understand assertions.
   Result: Test expects a 302 redirect to "/dashboard" after login.
4. Step 2 — Run the test with verbose output (Bash: pytest -xvs).
   Result: Getting 302 to "/home" instead of "/dashboard". Partially useful — now know the symptom.
5. Step 3 — Grep for the redirect URL in the auth module.
   Result: Found redirect logic in src/auth/views.py:47 — redirects to settings.LOGIN_REDIRECT_URL.
6. Step 4 — Grep for LOGIN_REDIRECT_URL in settings.
   Result: settings/base.py has "/dashboard", but settings/test.py overrides to "/home".
   ** Tool Adoption: didn't plan to read settings/test.py but intermediate result pointed here. **
7. Step 5 — Verify by reading settings/test.py.
   Result: Confirmed. LOGIN_REDIRECT_URL = "/home" in test settings.
8. Synthesize: The test expects "/dashboard" but test settings override the redirect to "/home".

Output:
The test fails because `settings/test.py:12` overrides `LOGIN_REDIRECT_URL = "/home"`,
while the test asserts a redirect to "/dashboard". Either update the test expectation
to match the test settings, or remove the override in test settings.
```

**Example 2: Building a data pipeline from heterogeneous sources**

```
User: "I need to combine data from a CSV file, a JSON API endpoint, and a SQLite database
into a single summary report."

Approach:
1. Decompose: Parse CSV → Fetch API data → Query SQLite → Normalize schemas →
   Merge datasets → Generate report.
2. Inventory: Read (CSV), WebFetch (API), Bash (sqlite3 queries), Write (output report).
3. Step 1 — Read the CSV file, identify columns and data types.
   Result: CSV has columns [user_id, name, signup_date]. 500 rows. Success.
4. Step 2 — Fetch JSON API endpoint.
   Result: API returns 403 Forbidden.
   ** Backtrack: API needs auth. Ask user for credentials or API key. **
   Fallback: Ask user via AskUserQuestion for the API key.
   Re-invoke with auth header. Result: JSON array with [user_id, purchase_total]. Success.
5. Step 3 — Query SQLite for user activity.
   Result: Table "activity" has [user_id, last_login, session_count]. Success.
6. Step 4 — Normalize: all three sources share user_id as join key. Convert dates
   to ISO format. No schema conflicts.
7. Step 5 — Write a Python script that merges on user_id using pandas.
   ** Tool Suppression: considered using bash join/awk but pandas is cleaner
   for three-way merge with type coercion. **
8. Step 6 — Execute the script, write output to summary_report.csv.
9. Verify: spot-check 5 rows across all three sources. All match.

Output:
Generated summary_report.csv with 500 rows and columns:
[user_id, name, signup_date, purchase_total, last_login, session_count]
```

**Example 3: Adaptive search across a large monorepo**

```
User: "Where is the rate limiting logic implemented? I need to change the default limit."

Approach:
1. Decompose: Find rate-limiting code → Understand the configuration → Locate default value.
2. Step 1 — Grep for "rate.limit" across the repo.
   Result: 47 matches, mostly in tests and docs. Partially useful — too noisy.
3. Step 2 — Refine: Grep for "RateLimit" in source files only (exclude tests/).
   Result: 8 matches. Found src/middleware/rate_limiter.py and src/config/defaults.py.
   ** Frequency Modulation: narrowed search from broad to targeted in 2 iterations. **
4. Step 3 — Read src/middleware/rate_limiter.py.
   Result: Class RateLimiter reads limit from config, defaults to self.default_limit = 100.
5. Step 4 — Read src/config/defaults.py.
   Result: RATE_LIMIT_PER_MINUTE = 100 at line 34.
   ** Tool Suppression: no need to search further — found the authoritative source. **
6. Verify: Grep for RATE_LIMIT_PER_MINUTE to confirm no other overrides.
   Result: Only referenced in defaults.py and rate_limiter.py. Confirmed.

Output:
The default rate limit is set at `src/config/defaults.py:34` as
`RATE_LIMIT_PER_MINUTE = 100`. The middleware at `src/middleware/rate_limiter.py:18`
reads this value. Change the value in defaults.py to update the limit.
```

## Best Practices

- **Do:** Evaluate every tool output before deciding the next step. Never blindly chain tool calls without inspecting intermediate results.
- **Do:** Maintain a running mental model of what you know and what remains uncertain. Each tool call should reduce uncertainty about a specific subgoal.
- **Do:** Set iteration limits (3-5 attempts) for any repeated tool invocation to prevent unbounded loops on unproductive searches.
- **Do:** Prefer tools with specific, narrow queries over broad, sweeping ones. A targeted grep beats a full-repo search.
- **Avoid:** Committing to a fixed tool sequence at the start. The whole point of adaptive orchestration is that the plan evolves with observations.
- **Avoid:** Invoking tools "just in case" — every tool call should have a clear hypothesis about what it will reveal. Tool suppression is a feature, not a failure.
- **Avoid:** Ignoring tool failures. A failed tool call is valuable signal — it tells you something about the environment, permissions, or data format that should inform your next action.

## Error Handling

| Failure Mode | Response Strategy |
|---|---|
| Tool returns an error (e.g., file not found, permission denied) | Log the error, assess whether the path/parameter was wrong vs. a systemic issue. Retry with corrected input or switch to an alternative tool. |
| Tool returns empty or irrelevant results | Broaden or narrow the query. If two attempts yield nothing, suppress the tool and try a different approach entirely. |
| Tool output contradicts prior observations | Do not silently accept. Re-verify the earlier observation with a different tool. Resolve the contradiction before proceeding. |
| Tool call hangs or times out | Set explicit timeouts. If a tool is unresponsive, fall back to direct reasoning or a lighter-weight alternative. |
| Circular reasoning (tool A suggests tool B, tool B suggests tool A) | Detect the loop by tracking the last 3-5 actions. Break out by either making a direct decision or asking the user for guidance. |

## Limitations

- **Not for single-tool tasks.** If a task clearly needs exactly one tool call (e.g., "read this file"), adaptive orchestration adds unnecessary overhead. Use it only when multi-step reasoning is genuinely required.
- **Context window pressure.** Long tool chains accumulate observations that consume context. For chains exceeding 8-10 steps, summarize intermediate results to avoid context overflow.
- **No guaranteed optimality.** Adaptive selection finds *good* tool sequences, not provably *optimal* ones. For safety-critical systems, verify the final result independently.
- **Tool quality dependency.** The approach assumes tools produce generally reliable output. If a tool is fundamentally broken or consistently returns garbage, adaptive orchestration will waste cycles trying to work around it — better to remove the tool from the inventory.
- **Latency cost.** Each evaluate-then-decide cycle adds latency compared to a pre-compiled pipeline. For latency-sensitive applications, consider caching tool selection patterns for recurring task types.

## Reference

**Paper:** [AdaReasoner: Dynamic Tool Orchestration for Iterative Visual Reasoning](https://arxiv.org/abs/2601.18631v2) (Song et al., 2026)
**Key insight to extract:** The Tool-GRPO reward structure (Section 3.2) and the three emergent behaviors — tool adoption, suppression, and frequency modulation — which demonstrate that adaptive tool use can be learned as a general reasoning skill rather than hard-coded per task. The token-level randomization technique (Section 3.3) is particularly relevant for building tool-agnostic orchestration that generalizes to unfamiliar tools.