---
name: "debugging-code-world"
description: "Debug code by mentally simulating execution as a Code World Model — predicting runtime state after each statement, catching failures from token-budget exhaustion and string tokenization brittleness, and isolating whether bugs come from incorrect action generation or state propagation errors. Use when: 'trace through this code step by step', 'why does this function return the wrong value', 'debug this execution trace', 'simulate what happens when I run this', 'find where the state goes wrong', 'step through the variables in this loop'."
---

# Debugging Code World Models

This skill teaches Claude to debug code by simulating execution as a Code World Model (CWM) — predicting the explicit runtime state after every executed statement, then diagnosing failures using the two dominant error regimes identified in the research: token-budget exhaustion on long execution histories, and string-valued state failures caused by subword tokenization discontinuities. Rather than reasoning about code abstractly, Claude traces concrete variable states at each step, identifies where predicted state diverges from expected state, and classifies the root cause as either an action-generation error (wrong operation chosen) or a state-propagation error (correct operation, wrong result).

## When to Use

- When the user asks to trace through code execution step by step and predict variable values
- When debugging a function that returns an incorrect result and the user wants to find where state diverges
- When a loop or recursive function produces wrong output and you need to track state across many iterations
- When string manipulation code behaves unexpectedly (split, join, replace, find operations)
- When code involves long chains of in-place mutations and you need to verify each intermediate state
- When the user wants to understand why composed function calls produce incorrect results
- When debugging deeply nested control flow where execution path is unclear

## Key Technique

A Code World Model simulates program execution by predicting a complete runtime state snapshot after every executed statement. Instead of reasoning about code in natural language ("this loop probably does X"), you produce an explicit execution trace: for each line, write down every variable's value after that line runs. This dense supervision makes errors visible immediately — the first line where predicted state diverges from actual state is your bug location.

The research identifies two systematic failure modes. First, **token-budget exhaustion**: when programs have long execution histories (deep loops, element-by-element processing, recursive chains), the execution trace becomes so large that you lose track of earlier state. The mitigation is to compress traces — track only variables that changed, summarize stable state, and checkpoint at loop boundaries rather than every iteration. Second, **string-valued state brittleness**: string operations fail at disproportionately high rates (73% of failures on CruxEval-O despite being only 46% of outputs) because subword tokenization creates context-dependent mappings. The separator `"-."` tokenizes as one token in isolation but fragments into five tokens inside `"a-.-.b"`, causing methods like `rsplit` and `rfind` to produce unexpected results. When you encounter string bugs, always evaluate string operations character-by-character rather than relying on pattern intuition.

The most actionable finding: long-horizon degradation comes primarily from **incorrect action generation**, not from state-tracking errors. When the research replaced model-generated operations with ground-truth commands, the CWM tracked state accurately over 128+ steps. This means: if your trace goes wrong, the bug is almost certainly in *which operation you think runs next* (wrong branch taken, wrong function called, off-by-one in iteration count), not in *how you computed the result of a correct operation*. Prioritize verifying control flow and operation selection before re-checking arithmetic.

## Step-by-Step Workflow

1. **Extract the code under examination.** Isolate the function or block to debug. Identify all input parameters, global state, and expected output. Write down the concrete input values you are tracing.

2. **Initialize the state table.** Create a variable-to-value mapping for every variable in scope at the entry point. Format as a table or dictionary literal: `{x: 5, name: "hello", items: [1,2,3]}`.

3. **Execute line-by-line, producing state snapshots.** For each statement, predict the runtime state *after* execution. Write the updated state table. For assignments, update the variable. For function calls, expand inline if the function is available; otherwise, predict the return value.

4. **Compress traces for long executions.** If the code contains loops with more than ~10 iterations, do NOT trace every single iteration. Instead: trace the first 2 iterations fully, state the pattern, then trace the last 2 iterations and the exit condition. Checkpoint the full state at loop entry and exit.

5. **Flag string operations for character-level verification.** When you encounter string methods (`split`, `join`, `replace`, `find`, `rfind`, `rsplit`, slicing with computed indices), do NOT predict the result from pattern recognition. Instead, write out the string character by character, apply the operation mechanically, and verify the result against your intuition. If they disagree, trust the mechanical trace.

6. **Identify the divergence point.** Compare your predicted state at each step against the expected behavior. The first statement where predicted state differs from correct state is your primary bug candidate.

7. **Classify the failure mode.** Ask: is the divergence because (a) the wrong operation executed (wrong branch, wrong function, wrong iteration count — an **action-generation error**), or (b) the right operation executed but produced the wrong result (arithmetic error, off-by-one, wrong method behavior — a **state-propagation error**)? Action-generation errors are far more common.

8. **Verify control flow independently.** If you suspect an action-generation error, trace only the control flow (which branches are taken, which iterations occur) without computing values. Confirm the execution path is correct before re-examining value computation.

9. **Test the fix by re-tracing.** After identifying the bug, apply the fix and re-run your state-trace from 2 steps before the divergence point through 2 steps after. Confirm the state now matches expected output.

10. **Summarize with the root cause classification.** Report: where the state diverged, whether it was action-generation or state-propagation, the specific failure mode (token-budget/string-brittleness/logic error), and the fix.

## Concrete Examples

**Example 1: Debugging a string manipulation function**

User: "Why does this function return the wrong result?"
```python
def extract_domain(email):
    parts = email.rsplit("@", 1)
    domain = parts[1]
    subdomain = domain.rsplit(".", 1)
    return subdomain[0]

# Expected: "company" for "user@company.co.uk"
# Actual: "company.co"
```

Approach:
1. Initialize state: `{email: "user@company.co.uk"}`
2. Line `parts = email.rsplit("@", 1)`:
   - Character-level verify: `rsplit("@", 1)` splits from the right, max 1 split
   - State: `{parts: ["user", "company.co.uk"]}`
3. Line `domain = parts[1]`:
   - State: `{domain: "company.co.uk"}`
4. Line `subdomain = domain.rsplit(".", 1)`:
   - **Flag: string operation — verify character by character**
   - `"company.co.uk".rsplit(".", 1)` splits from the right at last `.`
   - Result: `["company.co", "uk"]`
   - State: `{subdomain: ["company.co", "uk"]}`
5. Line `return subdomain[0]` returns `"company.co"`

Divergence at step 4: The user expects `rsplit(".", 1)` to split at the *first* dot, but `rsplit` splits from the right. This is an **action-generation error** — the wrong operation was chosen (`rsplit` instead of `split`).

Fix: Change `domain.rsplit(".", 1)` to `domain.split(".", 1)` — or, if the intent is to get the subdomain before the TLD, rethink the parsing for multi-part TLDs.

**Example 2: Debugging a loop with accumulator drift**

User: "This should compute a running average but the result is wrong after many iterations."
```python
def running_avg(values):
    avg = 0
    for i, v in enumerate(values):
        avg = avg + (v - avg) / (i + 1)
    return avg

# Input: [10, 20, 30, 40, 50]
# Expected: 30.0
# Actual: 30.0 (seems correct... but fails on [1, 1, 1, ..., 1] x 1000)
```

Approach:
1. Initialize: `{avg: 0, values: [1]*1000}`
2. Trace first 2 iterations:
   - i=0: `avg = 0 + (1 - 0) / 1 = 1.0` — state: `{avg: 1.0}`
   - i=1: `avg = 1.0 + (1 - 1.0) / 2 = 1.0 + 0.0 = 1.0` — state: `{avg: 1.0}`
3. Pattern: avg should remain 1.0 for all iterations since every value equals the running mean.
4. Trace last 2 iterations (i=998, i=999): same pattern holds.
5. **Compress**: No divergence in this constant input case.
6. Re-test with mixed input `[1e15, 1, 1, 1, ...]`:
   - i=0: `avg = 1e15`
   - i=1: `avg = 1e15 + (1 - 1e15) / 2 = 1e15/2 + 0.5 = 5e14 + 0.5`
   - At large iteration counts, `(v - avg)` loses precision when `avg` is large and `v` is small.

Root cause: **state-propagation error** due to floating-point precision loss in the subtraction `(v - avg)` when magnitudes differ greatly. This is a numerical stability issue, not a logic error. Fix: use compensated summation (Kahan) or two-pass computation.

**Example 3: Debugging control flow with early exit**

User: "This search function sometimes returns None when the item exists."
```python
def find_in_nested(data, target):
    for key, value in data.items():
        if isinstance(value, dict):
            result = find_in_nested(value, target)
            return result  # BUG: returns even if result is None
        elif value == target:
            return key
    return None

# Input: {"a": {"x": 1}, "b": 2}, target=2
```

Approach:
1. Initialize: `{data: {"a": {"x": 1}, "b": 2}, target: 2}`
2. First iteration: key="a", value={"x": 1}
   - `isinstance({"x": 1}, dict)` is True
   - Recurse: `find_in_nested({"x": 1}, 2)`
     - key="x", value=1: not dict, `1 == 2` is False
     - Loop ends, returns None
   - Back in caller: `result = None`
   - **`return result` executes immediately — returns None**
3. The loop never reaches key="b".

Divergence at step 2, specifically `return result`: This is an **action-generation error** — the function unconditionally returns after the first recursive call, exiting the loop prematurely. The correct action is to return only when `result is not None`.

Fix: Replace `return result` with `if result is not None: return result`.

## Best Practices

- **Do:** Always write out concrete variable values, never reason abstractly about "what this probably does." The entire power of CWM-style debugging is in the explicit state.
- **Do:** Compress loop traces using the checkpoint pattern (first 2, pattern, last 2). Full traces of 100+ iterations waste your context budget and introduce tracking errors — the same token-budget exhaustion the paper identifies.
- **Do:** Apply extra scrutiny to string operations. Verify character-by-character. Methods like `rsplit`, `rfind`, `partition`, and regex with special characters are the highest-risk operations.
- **Do:** When a trace goes wrong, check the control flow path first (which branch, which iteration) before rechecking value computation. Action-generation errors dominate.
- **Avoid:** Tracing every iteration of a long loop. This is the #1 cause of state-tracking errors in mental simulation — directly paralleling the token-budget exhaustion finding.
- **Avoid:** Trusting your intuition on string method behavior with edge-case inputs (empty strings, strings with repeated delimiters, Unicode). Always expand mechanically.

## Error Handling

- **Trace becomes too long:** If you find yourself tracking more than ~30 state snapshots, stop and compress. Identify which variables are stable (not changing) and exclude them from subsequent snapshots. Only track deltas.
- **Recursive calls go deep:** For recursion deeper than 4 levels, trace the first 2 levels fully, identify the recursive pattern, then trace the base case and the unwinding of the last 2 levels.
- **State includes complex objects:** For large data structures (trees, graphs, nested dicts), represent them in abbreviated form. A list of 100 elements becomes `[1, 2, 3, ..., 98, 99, 100]` with explicit length annotation.
- **Multiple possible execution paths:** When control flow depends on runtime values you are unsure about, trace both paths for one step and determine which matches. Do not guess.
- **Disagreement between intuition and mechanical trace:** Always trust the mechanical character-level trace over your intuition, especially for string operations. The paper shows that tokenization-induced pattern matching is the primary source of systematic errors.

## Limitations

- This technique is most effective for **imperative, sequential code** with explicit state. It is less useful for purely functional pipelines, declarative configurations, or highly concurrent code where interleaving is non-deterministic.
- Mental simulation cannot replace actual execution for **floating-point edge cases**, **platform-dependent behavior**, or **external I/O** (network, filesystem). Use this technique to narrow down the bug location, then confirm with actual execution.
- For programs with very long execution histories (>1000 statement executions), even compressed traces exceed practical limits. In these cases, use binary search: trace the midpoint state, determine which half contains the divergence, and recurse.
- The technique assumes you have concrete input values. If the bug is input-dependent and you do not know the failing input, use this skill in combination with input generation / fuzzing strategies.

## Reference

[Debugging Code World Models](https://arxiv.org/abs/2602.07672v1) — Rahmani, 2026. Key findings: (1) CWM failures concentrate in string-valued state due to tokenization discontinuities (strings are 46% of outputs but 73% of failures), (2) long-horizon degradation is caused by incorrect action generation rather than state-tracking errors — when given correct operations, Transformers track state accurately over 128+ steps, (3) non-string data types achieve 100% accuracy at depth-5 composition while strings degrade to 25%.