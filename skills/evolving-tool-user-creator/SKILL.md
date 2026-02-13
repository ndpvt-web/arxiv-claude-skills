---
name: "evolving-tool-user-creator"
description: "Transform Claude from a static tool user into a dynamic tool creator using the UCT (User-to-Creator Transformation) framework. Harvests reasoning traces from problem-solving sessions and distills them into reusable utility functions, scripts, and helpers that grow a persistent tool library over time. Trigger phrases: 'create a reusable tool from this solution', 'build a helper I can reuse', 'evolve my toolset', 'extract a utility from this workflow', 'self-improving agent pipeline', 'turn this reasoning into a tool'."
---

# Evolving Tool User to Tool Creator (UCT Framework)

This skill enables Claude to go beyond using existing tools by **creating new reusable tools on-the-fly during problem-solving**. Based on the UCT (User-to-Creator Transformation) framework, Claude harvests its own reasoning traces—the step-by-step logic it produces while solving a problem—and distills them into tested, reusable code artifacts (functions, scripts, CLI utilities). A memory consolidation mechanism maintains this growing tool library: merging duplicates, deprecating broken tools, and ensuring high reuse rates across future tasks.

## When to Use

- When the user solves a multi-step computational or data-processing problem and wants the solution captured as a reusable function or script
- When the user repeatedly encounters similar sub-problems (e.g., parsing specific formats, running domain-specific calculations) and wants Claude to auto-generate a helper library
- When building an agent pipeline that should improve itself over time without retraining—accumulating a library of tested utilities
- When the user asks Claude to solve a problem and no existing tool fits, requiring on-the-spot tool creation with validation
- When refactoring a one-off script into a maintained, tested utility that can be invoked in future sessions
- When orchestrating a multi-domain reasoning task (math, science, data analysis) where specialized micro-tools would accelerate subsequent steps

## Key Technique

The UCT framework reframes the agent's relationship with tools. Traditional tool-integrated reasoning treats tools as a fixed inventory—the agent picks from what exists. UCT instead treats **every reasoning trace as a potential tool blueprint**. When Claude works through a complex problem (geometric calculation, data transformation, API orchestration), the intermediate steps encode implicit problem-solving capabilities. UCT harvests these into explicit, executable code.

The framework operates in three coupled phases. The **Online Task Loop** uses ReAct-style reasoning where, at each step, the agent decides whether to use an existing tool, create a new one, or reason further. When creation is triggered, a **Build Loop** enters an isolated environment: it generates both the implementation code and test scripts simultaneously, then iterates using sandbox execution feedback and critic review until the tool passes. Finally, an **Offline Memory Consolidation** phase periodically cleans the library—merging similar tools, eliminating duplicates, and deprecating tools with high failure rates or low reuse.

The critical insight is the dual-verification gate in the Build Loop. Tools are not accepted based on generation alone. They must pass runtime sandbox tests AND a critic review that checks for edge cases, correctness, and API contract adherence. This prevents erroneous tool outputs from poisoning downstream reasoning—a key failure mode in naive tool-creation approaches. The result: 93%+ reuse rates and 20-23% performance gains on math and science benchmarks.

## Step-by-Step Workflow

1. **Receive the task and assess the tool gap.** Before jumping to a solution, inventory what existing tools/functions are available. Identify sub-problems where no current tool fits—these are tool-creation candidates. Ask: "Would a reusable function here save effort on this or future tasks?"

2. **Solve the task using ReAct-style reasoning.** Work through the problem step-by-step, explicitly documenting each reasoning step and intermediate computation. Keep the reasoning trace detailed—it becomes the blueprint for tool extraction.

3. **Identify reusable sub-capabilities in the reasoning trace.** Scan the trace for steps that are (a) self-contained, (b) parameterizable, and (c) likely to recur. Examples: a coordinate geometry calculation, a specific data parsing routine, a validation check pattern.

4. **Generate a build ticket for each tool candidate.** Write a concise specification: function name, purpose, input parameters with types, expected output, and 2-3 concrete test cases derived from the problem you just solved.

5. **Implement the tool as executable code with co-generated tests.** Write the function AND a test script in the same pass. The function should be pure where possible (no hidden state), well-typed, and documented with a one-line docstring.

6. **Run sandbox validation.** Execute the test script. If tests fail, analyze the failure, apply critic feedback (check edge cases, boundary conditions, type mismatches), and iterate. Do not accept a tool that fails any test case.

7. **Register the tool in the library with metadata.** Store the tool with: name, category, description, usage signature, creation context (what problem spawned it), and initial usage count of 1.

8. **Resume the original task using the newly created tool.** Invoke the tool to complete the step that triggered its creation. Verify the tool's output matches expectations in context.

9. **Consolidate the library periodically.** After completing a session or batch of tasks, review the tool library: merge tools with overlapping functionality, flag tools that have never been reused, and deprecate tools that produced errors in subsequent use.

10. **Retrieve before creating on subsequent tasks.** For every new sub-problem, search the existing library first (by category and description match). Only trigger the Build Loop if no existing tool fits. Log each retrieval to track reuse rates.

## Concrete Examples

**Example 1: Extracting a geometry utility from a math problem**

User: "I need to calculate the area of intersection between two circles given their centers and radii. I'll have many similar problems."

Approach:
1. Solve the intersection area problem step-by-step, documenting the reasoning trace (lens area formula, cases for containment and disjoint circles)
2. Identify the reusable sub-capability: `circle_intersection_area(x1, y1, r1, x2, y2, r2) -> float`
3. Generate build ticket with test cases from the solved problem
4. Implement and test:

```python
# Tool: circle_intersection_area
# Category: geometry/area
# Created from: circle intersection reasoning trace

import math

def circle_intersection_area(x1: float, y1: float, r1: float,
                              x2: float, y2: float, r2: float) -> float:
    """Compute area of intersection of two circles."""
    d = math.hypot(x2 - x1, y2 - y1)
    if d >= r1 + r2:
        return 0.0  # disjoint
    if d + min(r1, r2) <= max(r1, r2):
        return math.pi * min(r1, r2) ** 2  # one inside the other
    a = (r1**2 - r2**2 + d**2) / (2 * d)
    h = math.sqrt(r1**2 - a**2)
    sector1 = r1**2 * math.acos(a / r1)
    sector2 = r2**2 * math.acos((d - a) / r2)
    triangle = d * h
    return sector1 + sector2 - triangle

# Co-generated tests
assert circle_intersection_area(0, 0, 1, 10, 0, 1) == 0.0  # disjoint
assert abs(circle_intersection_area(0, 0, 5, 1, 0, 2) - math.pi * 4) < 1e-6  # contained
assert circle_intersection_area(0, 0, 1, 1, 0, 1) > 0  # overlapping
```

5. Tests pass. Register tool. Use it for the user's subsequent circle problems without re-deriving the formula.

**Example 2: Building a data parsing tool during CSV analysis**

User: "Parse this messy CSV where some rows have quoted fields with commas inside, some have missing columns, and dates are in mixed formats. I'll process 50 similar files."

Approach:
1. Manually reason through parsing the first file, identifying edge cases
2. Extract reusable sub-capabilities: `robust_csv_parse(filepath, date_columns) -> DataFrame` and `normalize_date(date_str) -> datetime`
3. Build both tools with tests derived from the actual edge cases encountered

```python
# Tool: normalize_date
# Category: parsing/datetime
# Created from: mixed-format CSV parsing trace

from datetime import datetime

FORMATS = ["%Y-%m-%d", "%m/%d/%Y", "%d-%b-%Y", "%B %d, %Y", "%Y%m%d"]

def normalize_date(date_str: str) -> datetime:
    """Parse a date string in common formats into a datetime object."""
    date_str = date_str.strip()
    for fmt in FORMATS:
        try:
            return datetime.strptime(date_str, fmt)
        except ValueError:
            continue
    raise ValueError(f"Unrecognized date format: {date_str!r}")

# Tests
assert normalize_date("2025-03-15").day == 15
assert normalize_date("03/15/2025").month == 3
assert normalize_date("15-Mar-2025").year == 2025
```

4. Use the tools to process all 50 files. On file #12, `normalize_date` fails on a new format (`"March 15th, 2025"`). **Self-update**: add the format to the tool, re-run tests, re-register.

**Example 3: Agent pipeline that accumulates tools across tasks**

User: "I'm building an agent that answers science questions. Start with this physics problem, but design it so the agent gets better over time."

Approach:
1. Solve the physics problem (projectile motion). Reasoning trace includes kinematic equations.
2. Extract tool: `projectile_range(v0, angle_deg, g=9.81) -> float`
3. Next problem arrives (optics—Snell's law). No existing tool fits. Create: `snells_law(n1, theta1, n2) -> float`
4. Third problem (projectile on an inclined plane). Retrieve `projectile_range` from library, find it needs extension. **Self-update** the tool to accept an `incline_deg` parameter. Run existing + new tests.
5. Library after 3 tasks: 2 physics tools with 4 total invocations, high reuse signal.

Consolidation check:
```
Tool Library Status:
- projectile_range   | physics/kinematics | uses: 3 | failures: 0 | status: active
- snells_law         | physics/optics     | uses: 1 | failures: 0 | status: active
- (no merges needed, no deprecations)
```

## Best Practices

**Do:**
- Always co-generate tests alongside the tool implementation—never register an untested tool
- Keep tools pure and parameterized: no hardcoded values, no hidden dependencies on global state
- Prefer small, composable tools over monolithic ones—a `normalize_date` + `parse_csv_row` is better than a single `do_everything` function
- Track usage metadata (invocation count, failure count) to drive consolidation decisions
- When a tool fails during reuse, fix it in-place and re-run all existing tests (regression check) before re-registering

**Avoid:**
- Creating tools for one-off calculations that will never recur—not every reasoning step deserves to be a tool
- Skipping the sandbox validation step under time pressure; unverified tools poison downstream reasoning
- Letting the tool library grow unbounded—run consolidation after every 10-15 tools are added
- Creating tools with vague names or missing type annotations; retrievability depends on clear metadata

## Error Handling

| Failure Mode | Response |
|---|---|
| Generated tool fails sandbox tests | Iterate: feed execution errors + critic feedback back into the Build Loop. Cap at 3 iterations; if still failing, fall back to inline reasoning for this task and log the failed attempt. |
| Tool produces wrong output during reuse on a new task | Check if the input is within the tool's designed domain. If yes, fix the tool (self-update) and add the new case as a regression test. If no, create a new tool instead. |
| Tool library grows too large (>50 tools) | Trigger immediate consolidation: merge tools with >70% functional overlap, deprecate tools with 0 reuses after 10+ tasks, archive rather than delete. |
| Retrieval returns the wrong tool for a sub-problem | Improve tool metadata (add usage examples to descriptions). If retrieval consistently fails, switch to keyword + category-based lookup instead of purely semantic matching. |
| Circular dependency between tools | Flatten: inline the dependency or restructure into a shared utility. Tools should be independently executable. |

## Limitations

- **Single-session context**: Without persistent storage, the tool library resets between sessions. To realize the full UCT benefit, the user must maintain the tool files externally (e.g., in a project directory) and re-load them in subsequent sessions.
- **Tool quality ceiling**: The generated tools are only as correct as the reasoning trace they derive from. If the initial reasoning is flawed, the tool encodes that flaw. The dual-verification gate (tests + critic) mitigates but does not eliminate this risk.
- **Not suited for highly creative or ambiguous tasks**: UCT works best for well-defined, parameterizable sub-problems (math, parsing, data transforms). Open-ended design tasks or subjective evaluations resist tool extraction.
- **Overhead on simple tasks**: For trivial one-off calculations, the Build Loop overhead (writing tests, validating, registering) exceeds the benefit. Apply tool creation selectively.
- **Library maintenance requires discipline**: Without periodic consolidation, the library becomes cluttered with near-duplicate or stale tools, degrading retrieval quality.

## Reference

**Paper**: [Evolving from Tool User to Creator via Training-Free Experience Reuse in Multimodal Reasoning](https://arxiv.org/abs/2602.01983v1) — Shen et al., 2026. Look for: Algorithm 1 (Online Task Loop), Algorithm 2 (Build Loop with dual verification), and Table 2 (ablation showing each component's contribution). The memory consolidation formalization in Section 3.3 provides the theoretical grounding for the library maintenance protocol.