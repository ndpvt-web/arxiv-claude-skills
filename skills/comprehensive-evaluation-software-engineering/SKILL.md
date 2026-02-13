---
name: "comprehensive-evaluation-software-engineering"
description: >
  Evaluate and optimize LLM-driven software engineering workflows across five task types
  (bug fixing, feature development, code refactoring, technical copywriting, research synthesis)
  using efficiency-aware metrics that go beyond correctness. Detects and eliminates loop
  inefficiency and inference inefficiency patterns in agentic tool usage.
  Use when: "evaluate my coding agent workflow", "optimize tool call efficiency",
  "benchmark LLM SE performance", "reduce agentic coding cost", "audit agent tool usage",
  "compare efficiency of coding approaches".
---

# Comprehensive Evaluation of Software Engineering Workflows

This skill enables Claude to evaluate, benchmark, and optimize LLM-driven software engineering
workflows using the multi-dimensional framework from Gunawan & Amien (2026). Instead of measuring
only whether a task succeeds, this approach jointly tracks correctness, completion time, tool call
count, and estimated cost -- revealing that models achieving identical scores can vary by 22x in
time, 49x in tool efficiency, and 53x in cost. The skill teaches Claude to detect two specific
inefficiency anti-patterns (loop inefficiency and inference inefficiency), classify SE tasks into
five categories with distinct efficiency profiles, and apply targeted optimization strategies to
each.

## When to Use

- When a user asks to audit or profile an agentic coding session for wasted tool calls
- When comparing two or more approaches to a coding task and the user wants efficiency analysis, not just correctness
- When a user reports that their AI coding workflow is slow or expensive and wants diagnosis
- When evaluating whether a multi-step agent plan is over-engineered (too many tool calls for the task complexity)
- When designing automated verification for SE tasks across bug fixing, feature development, refactoring, documentation, or research synthesis
- When a user wants to benchmark their prompt strategy against efficiency baselines
- When optimizing CI/CD pipelines that include LLM-powered code generation or review steps

## Key Technique

The core insight is that **correctness is a necessary but insufficient metric** for evaluating LLM
software engineering. The paper demonstrates this with a striking finding: across 11 models solving
identical tasks, tool usage frequency shows no correlation with success (Pearson r = 0.077,
p = 0.575). One model solved a task with 3 tool calls; another used 917 calls on the same task.
Both succeeded, but the cost difference was 53x. This means optimizing for fewer, more purposeful
tool interactions is a separate and critical dimension of LLM engineering quality.

The framework identifies two distinct inefficiency anti-patterns. **Loop inefficiency** occurs when
an agent repeats identical or near-identical operations without making progress -- for example,
reading the same file multiple times, retrying a failing command without changing the approach, or
re-running tests that already passed. **Inference inefficiency** occurs when an agent generates
excessive intermediate reasoning, exploratory tool calls, or speculative searches that do not
contribute to the solution -- such as reading every file in a directory when only one is relevant,
or making multiple search queries that return the same results.

The five SE task categories also exhibit distinct efficiency profiles. Coding tasks (bug fixing,
feature development, refactoring) achieve near-100% success rates but show the widest efficiency
variance, making them prime targets for tool-call optimization. Research synthesis tasks have lower
success rates (90.9%), indicating they benefit more from improved correctness strategies than from
efficiency tuning. Technical copywriting falls between these extremes.

## Step-by-Step Workflow

1. **Classify the SE task** into one of five categories: bug fixing, feature development, code
   refactoring, technical copywriting, or research synthesis. This determines which efficiency
   profile and optimization strategy to apply.

2. **Establish a correctness verification method** before starting. For code tasks, define the
   test suite, linting rules, or build checks that constitute success. For writing tasks, define
   the acceptance criteria (structure, factual requirements, format). For research tasks, define
   the expected outputs (citations, synthesis structure, coverage requirements).

3. **Record baseline metrics** at the start of the workflow: timestamp the start, initialize a
   tool-call counter, and note any cost-relevant parameters (model used, token pricing). These
   will be compared against the final metrics.

4. **Execute the task using a minimal-tool-call strategy.** For each step, ask: "Can I accomplish
   this with fewer tool interactions?" Prefer targeted file reads over directory scans, specific
   grep patterns over exploratory searches, and single-pass edits over iterative trial-and-error.

5. **Monitor for loop inefficiency in real time.** After each tool call, check whether the result
   is substantively different from any previous call in the session. If the same file is read
   twice, the same search is run twice, or the same command is retried without modification --
   flag this as loop inefficiency and change strategy immediately.

6. **Monitor for inference inefficiency.** After every 3-5 tool calls, assess whether each call
   contributed directly to the solution. If more than 30% of recent calls were exploratory or
   speculative without yielding actionable information, pause and reformulate the approach before
   continuing.

7. **Capture final metrics** upon task completion: total elapsed time, total tool calls, success
   status, and estimated cost. Calculate tool efficiency as (successful outcomes / total tool
   calls).

8. **Generate an efficiency report** comparing the workflow against known baselines. Flag any
   instances of loop or inference inefficiency detected. Provide specific recommendations for
   reducing tool calls on similar future tasks.

9. **Apply task-category-specific optimization.** For coding tasks (bug fix, feature, refactor):
   focus on reducing tool calls since correctness is typically high. For research synthesis:
   focus on improving correctness strategies (better source selection, structured output templates).
   For technical copywriting: balance both dimensions.

10. **Archive the metrics** for longitudinal comparison. Track efficiency trends across sessions
    to identify systematic inefficiency patterns in recurring task types.

## Concrete Examples

**Example 1: Auditing an agentic bug-fix session**

User: "I just used an AI coding agent to fix a null pointer bug. It took 47 tool calls and 4
minutes. Is that efficient?"

Approach:
1. Classify as "bug fixing" -- a coding task where near-100% success is expected, so efficiency
   is the primary optimization target.
2. Review the tool-call log for loop inefficiency: Did the agent read the same file multiple
   times? Did it run failing tests repeatedly without changing the fix?
3. Review for inference inefficiency: Did it search the entire codebase when the stack trace
   already pointed to the specific file and line?
4. Compare against the paper's baselines: optimal bug fixes can complete in as few as 3 tool
   calls (read file, edit file, verify fix). A 47-call session represents ~15x overhead.
5. Identify the specific waste: e.g., "12 calls were re-reads of the same 3 files, 8 calls were
   grep searches for the same symbol in different directories."

Output:
```
## Efficiency Audit: Bug Fix Session

Task category:  Bug fixing (coding task)
Success:        Yes
Tool calls:     47 (baseline: 3-8 for single-file bug fix)
Efficiency:     ~6-15x above optimal range

### Inefficiency Detected

Loop inefficiency (12 calls):
- `src/handler.ts` read 4 times (lines unchanged between reads)
- Test suite run 3 times with identical failing output before fix was modified

Inference inefficiency (18 calls):
- 8 grep searches across unrelated directories
- 6 file reads in modules not in the call stack
- 4 exploratory reads of config files unrelated to the null pointer

### Recommendations
- Start from the stack trace: read only the file and line indicated
- Run tests at most twice: once to confirm the bug, once to verify the fix
- Skip broad codebase searches when the error localizes the fault
- Target: 5-10 tool calls for a localized single-file bug fix
```

**Example 2: Comparing two approaches to a feature implementation**

User: "I'm deciding between two prompt strategies for adding a REST endpoint. Strategy A uses
chain-of-thought planning first, Strategy B dives straight into coding. How do I evaluate them?"

Approach:
1. Classify as "feature development" -- coding task, high expected success rate, efficiency is
   the differentiator.
2. Define the verification criteria: endpoint responds correctly to GET/POST, returns proper
   status codes, passes integration tests.
3. Run both strategies on the same task, recording metrics for each.
4. Compare across three dimensions: correctness (binary), tool calls (count), and time (seconds).

Output:
```
## Strategy Comparison: REST Endpoint Feature

| Metric            | Strategy A (Plan-first) | Strategy B (Direct-code) |
|-------------------|------------------------|--------------------------|
| Correctness       | Pass                   | Pass                     |
| Tool calls        | 14                     | 23                       |
| Time              | 85s                    | 142s                     |
| Estimated cost    | $0.03                  | $0.07                    |
| Loop inefficiency | 0 instances            | 4 instances              |
| Inference ineff.  | 1 instance             | 6 instances              |

Analysis: Both strategies succeed, but Strategy A uses 39% fewer tool calls
and completes 40% faster. Strategy B's direct approach led to 4 loop-inefficiency
instances (re-reading the routes file after each incremental edit) and 6 inference-
inefficiency instances (exploring existing endpoints before writing the new one).

Recommendation: Use plan-first for feature development. The upfront planning
investment (2-3 tool calls) eliminates downstream exploratory waste.
```

**Example 3: Optimizing a research synthesis workflow**

User: "My agent keeps failing at research synthesis tasks. It succeeds about 60% of the time.
How do I improve this?"

Approach:
1. Classify as "research synthesis" -- the hardest category (90.9% baseline success in the paper).
   Unlike coding tasks, improving correctness matters more than reducing tool calls.
2. Analyze failure modes: Are outputs incomplete (missing sources)? Structurally wrong (no proper
   synthesis)? Factually incorrect?
3. Apply research-task-specific optimizations: structured output templates, explicit source
   enumeration, multi-pass verification.

Output:
```
## Research Synthesis Optimization

Current success rate:  60% (paper baseline: 90.9%)
Primary bottleneck:    Correctness, not efficiency

### Diagnosis
Research synthesis fails differently than coding tasks. Common failure modes:
1. Incomplete source coverage -- agent stops after finding 2-3 sources
2. Summary without synthesis -- agent lists findings but doesn't compare/integrate
3. Missing structure -- no clear thesis, methodology comparison, or gap analysis

### Corrective Strategy
1. Pre-define output structure: require sections (Overview, Methods Comparison,
   Key Findings, Contradictions, Gaps, Conclusion)
2. Set explicit source minimums: "Find and cite at least 5 relevant sources"
3. Add a verification pass: after initial synthesis, re-read each source to check
   that no major findings were omitted
4. Use targeted search: specific paper titles or DOIs rather than broad keyword searches

### Expected improvement
Structured templates typically raise research synthesis success from 60-70% to 85-90%,
approaching the paper's observed ceiling.
```

## Best Practices

**Do:** Track tool calls as a first-class metric alongside correctness. A working solution that
uses 3 tool calls is strictly better than one that uses 917 calls for the same result.

**Do:** Classify each task by SE category before starting. Coding tasks need efficiency
optimization; research tasks need correctness optimization. Applying the wrong strategy wastes
effort.

**Do:** Detect loop inefficiency early. If you catch yourself reading the same file or running
the same command twice, stop immediately and change approach rather than hoping the third attempt
will differ.

**Do:** Prefer direct, targeted tool calls. Read a specific file at a specific line range rather
than scanning a directory. Search for a specific symbol rather than browsing broadly.

**Avoid:** Assuming more tool calls means more thorough work. The r = 0.077 correlation proves
this is false -- quantity of tool interaction is essentially random noise relative to success.

**Avoid:** Optimizing research synthesis tasks for speed. These have the lowest success rate
(90.9%), so invest extra tool calls in source verification and structural completeness rather
than minimizing interaction count.

**Avoid:** Retrying a failed approach without modification. If a command fails or a search returns
nothing, changing the query or strategy is mandatory -- repeating the same call is the definition
of loop inefficiency.

## Error Handling

- **False efficiency:** If an agent completes with very few tool calls but the solution is
  incorrect, do not reward the low call count. Correctness is a hard prerequisite; efficiency
  metrics only apply to successful completions.
- **Metric instrumentation failures:** If tool-call counts or timing data cannot be captured
  (e.g., running in an environment without logging), fall back to qualitative loop/inference
  inefficiency detection by reviewing the session transcript manually.
- **Task misclassification:** If a task spans multiple categories (e.g., a bug fix that requires
  refactoring), apply the efficiency profile of the dominant category. When in doubt, optimize
  for correctness first.
- **Baseline drift:** Efficiency baselines from the paper (3-917 tool calls) are anchored to
  specific tasks. For significantly more complex tasks, scale baselines proportionally rather
  than applying them as absolute thresholds.

## Limitations

- The paper evaluates 11 models on a fixed task set. Efficiency baselines may not transfer
  directly to tasks of different scale or domain complexity.
- The five-category taxonomy (bug fix, feature, refactor, copywriting, research) may not cover
  all SE activities. Tasks like security auditing, performance optimization, or database migration
  require extrapolation.
- Tool-call counting treats all calls equally. A file read and a complex build command have
  different costs, but the framework counts them identically. Weighted cost models would be
  more precise.
- The 90.9% research synthesis ceiling may reflect task design rather than a fundamental LLM
  limitation. Different research tasks may have different success ceilings.
- The framework assumes automated verification is available. For subjective tasks (code style,
  documentation quality), human evaluation remains necessary and is not covered by this approach.

## Reference

[Gunawan & Amien, 2026. "Comprehensive Evaluation of Large Language Models on Software Engineering
Tasks: A Multi-Task Benchmark." arXiv:2602.07079v1](https://arxiv.org/abs/2602.07079v1)

Key finding to look for: Table/figures showing the 22x/49x/53x variation across models with
identical correctness scores, and the r = 0.077 non-correlation between tool usage and success.