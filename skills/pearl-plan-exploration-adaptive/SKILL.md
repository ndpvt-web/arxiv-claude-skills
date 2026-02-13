---
name: "pearl-plan-exploration-adaptive"
description: "Apply PEARL's two-phase tool orchestration: offline tool exploration to learn valid usage patterns and failure modes, then structured plan-before-execute workflows for complex multi-step tool chains. Use when: 'plan a multi-tool workflow', 'chain multiple API calls', 'explore tools before executing', 'build a robust tool pipeline', 'multi-hop tool invocation', 'reduce tool call errors in a workflow'."
---

# PEARL: Plan Exploration and Adaptive Reinforcement Learning for Multi-hop Tool Use

This skill teaches Claude to apply the PEARL framework's core insight — **separate planning from execution, and explore tools before using them** — to build reliable multi-step tool chains. Instead of interleaving tool selection with execution (which causes hallucinated tool names, wrong parameters, and broken pipelines), PEARL first explores each tool to learn its valid usage patterns and failure modes, then generates a complete plan specifying every step and tool before executing anything. This two-phase approach reduces invocation errors from ~20%+ to under 4% on complex multi-hop benchmarks.

## When to Use

- When the user asks you to chain 3+ tool calls where outputs feed into subsequent inputs (e.g., "fetch data from this API, transform it, then load into a database")
- When building a pipeline that involves unfamiliar or complex CLI tools and you need to verify valid invocation patterns first
- When a task requires multi-hop reasoning across tools — the answer from one tool determines which tool to call next
- When the user reports flaky or error-prone tool workflows and wants them made robust
- When orchestrating a sequence of bash commands, API calls, or file operations where ordering and parameter passing matter
- When the user says "plan this out before doing anything" or "explore the tools first"

## Key Technique

**The core problem** with naive multi-step tool use is that LLMs select tools and generate parameters on-the-fly, leading to three failure modes: (1) *tool hallucination* — calling tools that don't exist or misremembering their names, (2) *parameter errors* — passing arguments with wrong types, missing required fields, or invalid values, and (3) *weak planning* — choosing an incorrect sequence of tools or missing necessary intermediate steps. PEARL addresses all three by decoupling exploration, planning, and execution into distinct phases.

**Phase 1: Offline Tool Exploration.** Before executing any plan, the agent systematically interacts with each available tool in a controlled trial-and-error process (up to 10 rounds per tool). The goal is to build an *execution knowledge base* — a practical learned manual that documents valid invocation patterns, argument constraints, common failure modes, and example successful calls. This is analogous to reading documentation and running test commands before writing a script. The knowledge base eliminates hallucination because the agent has concrete evidence of what each tool accepts and returns.

**Phase 2: Plan-then-Execute.** With tool knowledge in hand, the Planner generates a complete ordered plan `P = ((s1, t1), (s2, t2), ..., (sn, tn))` where each step `si` is a natural language sub-goal and `ti` is the specific tool to use. Only after the full plan is validated does the Executor run each step, using the knowledge base to construct correct invocations and passing outputs forward as context. The reward signal in PEARL evaluates plans step-by-step — each step scores `1/m` if the tool selection matches the optimal choice, yielding a total score of 1.0 for a perfect plan. This step-wise scoring encourages plans that are neither too short (missing steps) nor too long (redundant steps).

## Step-by-Step Workflow

1. **Inventory available tools.** List every tool, command, API endpoint, or function available for the task. For each tool, note its name, purpose, and any known documentation. If working with CLI tools, run `--help` or `man` to collect usage info.

2. **Explore each unfamiliar tool.** For tools you haven't used recently or that have complex parameter requirements, run 2-5 trial invocations with safe inputs. Document: (a) what the tool actually accepts, (b) what output format it returns, (c) what error messages look like for bad inputs, (d) any required environment setup (auth tokens, config files).

3. **Build an execution knowledge base.** Synthesize exploration results into a structured summary per tool:
   - **Valid patterns**: Working command templates with placeholder parameters
   - **Constraints**: Required fields, type restrictions, value ranges, rate limits
   - **Failure modes**: Common errors and their causes (e.g., "returns 401 if token expired", "fails silently on empty input")
   - **Output format**: What the tool returns and how to parse it

4. **Decompose the user's goal into sub-goals.** Break the task into the minimum number of discrete steps, where each step accomplishes one sub-goal using one tool. Write each step as: `(natural language goal, tool name)`.

5. **Validate the plan before execution.** Check the plan for: (a) correct tool selections from the knowledge base (no hallucinated tools), (b) correct ordering — does each step have the inputs it needs from prior steps?, (c) completeness — are any intermediate steps missing?, (d) no redundant steps.

6. **Score the plan step-by-step.** For each step, ask: "Is this the right tool for this sub-goal? Are the parameters derivable from prior outputs?" If any step scores poorly, revise the plan before proceeding.

7. **Execute sequentially, passing context forward.** Run each step in order. The execution context for step `i` includes: the original query, the current sub-goal and tool, and all prior outputs `{o1, ..., o_{i-1}}`. Use the knowledge base to construct syntactically correct invocations.

8. **Handle execution failures with knowledge-base fallback.** If a step fails, consult the knowledge base for that tool's known failure modes. Apply the documented fix (e.g., retry with corrected parameters, add missing auth header) rather than guessing.

9. **Aggregate results and answer.** Combine outputs from all steps to produce the final answer. Verify the answer addresses the original query.

## Concrete Examples

**Example 1: Multi-hop data pipeline — fetch, transform, load**

```
User: "Get the latest exchange rates from an API, convert a list of USD
prices in prices.csv to EUR, and write the results to output.csv"

Tool Exploration:
- curl: Tested fetching exchange rate API, confirmed JSON response format
  {"rates": {"EUR": 0.92, ...}}, needs no auth for latest rates
- csvkit/python: Confirmed csvlook reads the file, has "price_usd" column
- python pandas: Verified read_csv and to_csv work on the file format

Plan:
  Step 1: (Fetch latest USD->EUR rate, curl)
  Step 2: (Read prices.csv and parse USD amounts, python/pandas)
  Step 3: (Multiply each price by EUR rate, python/pandas)
  Step 4: (Write converted prices to output.csv, python/pandas)

Plan Validation:
- Step 1 output (EUR rate) feeds into Step 3 ✓
- Step 2 output (price column) feeds into Step 3 ✓
- No missing steps, no hallucinated tools ✓

Execution:
  Step 1: curl -s "https://api.exchangerate-host/latest?base=USD"
          -> {"rates": {"EUR": 0.9186}}
  Step 2-4: Python script using pandas, applying rate 0.9186
          -> output.csv written with price_eur column
```

**Example 2: Multi-step git archaeology — find who broke a test**

```
User: "Find which commit broke the test_auth test, identify the author,
and show me the specific lines they changed"

Tool Exploration:
- git log: Confirmed --oneline and --format options work
- git bisect: Tested start/good/bad flow, confirmed it returns commit hash
- git show: Verified it outputs diff for a given commit hash
- pytest: Confirmed test_auth runs with `pytest tests/test_auth.py`

Plan:
  Step 1: (Find a known good commit where test passes, git log + pytest)
  Step 2: (Binary search for breaking commit, git bisect)
  Step 3: (Get author of breaking commit, git show --format)
  Step 4: (Show the exact diff of the breaking commit, git show)

Execution:
  Step 1: git log --oneline -20, then test HEAD~10 -> passes
  Step 2: git bisect start HEAD HEAD~10, run pytest at each step
          -> identifies commit abc1234
  Step 3: git show --format="%an <%ae>" abc1234 -> "Jane Doe <jane@co.com>"
  Step 4: git show abc1234 -- src/auth.py -> changed token validation logic
```

**Example 3: API integration with error-prone parameters**

```
User: "Query our ElasticSearch for error logs from the last hour,
aggregate by service name, then post a summary to Slack"

Tool Exploration (critical for complex APIs):
- curl + ES: Tested query DSL, discovered:
  - Valid: {"query":{"range":{"@timestamp":{"gte":"now-1h"}}}}
  - Failure: Omitting Content-Type header -> 406 error
  - Failure: Wrong index name -> 404, valid index is "logs-*"
- curl + Slack: Tested webhook, discovered:
  - Requires {"text": "..."} body, max 40k chars
  - Failure: Missing Content-Type -> silent 400

Plan:
  Step 1: (Query ES for error logs in last hour, curl + ES API)
  Step 2: (Parse response and aggregate counts by service, jq)
  Step 3: (Format summary as readable text, text processing)
  Step 4: (Post summary to Slack webhook, curl)

Knowledge Base Applied:
- Step 1: Include -H "Content-Type: application/json", use index "logs-*"
- Step 4: Include Content-Type header, check message length < 40k
```

## Best Practices

- **Do: Always explore unfamiliar tools before planning.** Even 2-3 trial runs reveal constraints that documentation misses — auth requirements, output format quirks, undocumented error codes. This is the single highest-ROI step in PEARL.

- **Do: Write the complete plan before executing any step.** Resist the urge to start executing after planning the first step. A complete plan lets you catch ordering issues and missing dependencies upfront.

- **Do: Score each plan step independently.** Ask "is this the right tool?" and "are the parameters available?" for every step. A plan with one wrong tool selection in the middle will cascade failures downstream.

- **Do: Keep the execution knowledge base as structured notes.** When exploring, write down valid patterns and failure modes explicitly. This artifact is reusable across the session.

- **Avoid: Generating tool names or parameters from memory alone.** Always verify against exploration results or documentation. Tool hallucination is the #1 cause of multi-step pipeline failures.

- **Avoid: Plans with unnecessary steps.** The optimal plan has the minimum number of steps to achieve the goal. Each extra step is an extra failure point. If two operations can be combined in a single tool call, combine them.

## Error Handling

| Error Type | Symptom | PEARL Response |
|---|---|---|
| Tool hallucination | "command not found" or "unknown tool" | Consult knowledge base; re-explore available tools; revise plan |
| Parameter error | 400/422 errors, type mismatches | Check knowledge base for valid patterns; re-explore with corrected params |
| Missing intermediate step | Step N lacks required input from step N-1 | Insert missing step in plan; re-validate full sequence before re-executing |
| Tool output format change | Parser fails on unexpected response shape | Re-explore the tool to update knowledge base; adjust downstream steps |
| Cascading failure | Early step error propagates | Stop execution, fix the failing step, re-run from that point forward — don't retry the whole chain |

When a step fails, **do not retry blindly**. Consult the knowledge base entry for that tool, identify the failure mode, apply the documented fix, and only then retry. If the failure mode is new, run 1-2 exploration rounds to understand it before proceeding.

## Limitations

- **Exploration overhead**: The offline phase adds upfront cost. For simple 1-2 step tasks with well-known tools, skip exploration and use tools directly. PEARL's value scales with pipeline complexity.
- **Dynamic tool environments**: If tools change between exploration and execution (e.g., API versioning, rotating credentials), the knowledge base becomes stale. Re-explore when errors suggest drift.
- **Unbounded tool sets**: PEARL works best with a bounded, known set of tools. If the tool space is open-ended (e.g., arbitrary npm packages), focus exploration on the specific tools needed for the current task.
- **Single-path planning**: PEARL generates one plan. For tasks with genuinely ambiguous tool choices (multiple valid approaches), consider generating 2-3 candidate plans and scoring them before committing.
- **Not a substitute for domain expertise**: PEARL improves tool *invocation* reliability but doesn't help choose the right *strategy*. If the user's approach is fundamentally wrong, planning won't fix it.

## Reference

**Paper**: [PEARL: Plan Exploration and Adaptive Reinforcement Learning for Multihop Tool Use](https://arxiv.org/abs/2601.20439v1) (Wang et al., PRICAI 2025)
**Key takeaway**: Section 3's two-phase architecture (offline exploration + plan-then-execute) and Section 3.3's step-wise reward function `R(P) = sum(1/m for each correct tool selection)` are the core ideas to apply. The 56.5% success rate on ToolHop with only 3.8% invocation error rate demonstrates that separating exploration from planning from execution dramatically reduces tool use failures.