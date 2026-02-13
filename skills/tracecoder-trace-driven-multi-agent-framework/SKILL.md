---
name: "tracecoder-trace-driven-multi-agent-framework"
description: "Trace-driven debugging framework for LLM-generated code. Uses diagnostic probe instrumentation, causal trace analysis, and historical lesson learning to iteratively fix buggy code. Triggers: 'debug this code with traces', 'instrument and fix this function', 'trace-driven debugging', 'find the root cause of this bug', 'iteratively repair this code', 'why does this code fail these tests'"
---

# TraceCoder: Trace-Driven Multi-Agent Debugging Framework

This skill enables Claude to debug and repair buggy code using the TraceCoder methodology — a three-phase observe-analyze-repair loop inspired by how expert developers diagnose failures. Instead of relying on superficial pass/fail test signals, Claude instruments the code with diagnostic print probes to capture fine-grained runtime traces, performs causal analysis on those traces to pinpoint root causes, and applies targeted repairs. A historical lesson learning mechanism prevents repeating failed strategies, and a rollback mechanism ensures each iteration strictly improves toward correctness.

## When to Use

- When the user says "debug this code" or "fix this function" and provides failing test cases
- When code passes some tests but fails others, and the failure cause is non-obvious
- When simple error messages (e.g., "wrong answer") don't reveal *why* the code is wrong
- When previous fix attempts keep failing or going in circles — the lesson learning mechanism breaks repetitive cycles
- When debugging class-level or multi-function code where bugs hide in interactions between methods
- When the user asks to "trace" or "instrument" code to understand its runtime behavior
- When a function produces incorrect output but doesn't raise an exception (silent logical bugs)

## Key Technique

**Trace-driven debugging** replaces the typical "run tests → read error → guess fix" loop with deep runtime observation. The core insight is that pass/fail signals are lossy — they tell you *that* something is wrong but not *where* or *why*. By inserting non-invasive print probes at logical boundaries (function entries/exits, loop iterations, branch decisions, intermediate computations), you generate a runtime trace that exposes the exact point where actual behavior diverges from expected behavior. This makes root cause identification precise rather than speculative.

**Historical Lesson Learning (HLLM)** is what prevents the debugging loop from degenerating into random attempts. Each failed repair is recorded as a structured lesson: what strategy was tried, what code it produced, what error resulted, and how many tests passed. Before generating a new repair plan, the analysis phase explicitly reviews all prior lessons for the current problem to avoid repeating ineffective strategies and to build on partial progress.

**The Rollback Mechanism** enforces monotonic progress. After each repair attempt, the framework compares the new test pass count against the best-known version. If performance regresses, it reverts to the best-known code. If progress stagnates for multiple iterations, it terminates. This prevents fix attempts from accidentally breaking previously-passing tests.

## Step-by-Step Workflow

1. **Reproduce the failure.** Run the provided code against the test cases. Record which tests pass and which fail. Capture error messages and wrong outputs. If all tests pass, stop — there's nothing to debug.

2. **Instrument the code with diagnostic probes.** Insert print statements at logical boundaries following four rules:
   - **Logical decomposition**: Identify function bodies, branches (if/elif/else), loop iterations, and return points
   - **State traceability**: At each boundary, log key variable values — inputs, outputs, loop counters, intermediate results
   - **Instrumentation purity**: Add ONLY print statements. Never modify computational logic, add variables, or change control flow
   - **Structured output**: Format probes as `DEBUG: <location> | <variable>=<value>` for machine-readable parsing

3. **Execute the instrumented code** against the failing test cases. Capture the full diagnostic output (the runtime trace) alongside any exceptions or wrong outputs.

4. **Perform causal trace analysis.** Walk through the runtime trace and compare actual variable values against expected values at each step:
   - Identify the *first point of divergence* — where actual state departs from what correct execution would produce
   - Trace backward from the divergence to identify the root cause (wrong condition, off-by-one, incorrect formula, missing edge case)
   - Distinguish symptoms from causes — a wrong final output is a symptom; the root cause is upstream

5. **Review the lesson record.** If previous repair attempts exist, examine each one: what was tried, why it failed, how many tests it affected. Explicitly reason about what strategies to avoid and what partial insights to build on.

6. **Formulate a targeted repair plan.** Based on the root cause analysis and lesson review, write a specific plan: which lines to change, what the fix is, and why it addresses the root cause without breaking passing tests. Do not describe vague intentions — specify the exact logical correction.

7. **Apply the repair.** Modify the code according to the plan. Change only what the plan calls for. Do not refactor surrounding code or add unrelated improvements.

8. **Validate the repair.** Run the repaired code against ALL test cases (not just the previously failing ones). Compare the pass count to the best-known version:
   - **Improvement** (more tests pass): Accept the repair as the new best version. Reset stagnation counter.
   - **Regression** (fewer tests pass): Rollback to the best-known version. Record this attempt as a lesson.
   - **Stagnation** (same pass count): Keep the current version but record the attempt. Increment stagnation counter.

9. **Iterate or terminate.** If tests still fail and the stagnation counter hasn't exceeded the limit (typically 3-5 attempts), return to step 2 with the current best code. If all tests pass, return the fixed code. If stagnation limit is reached, return the best-known version with a summary of what was tried.

10. **Report findings.** Present the final fixed code, a summary of the root cause(s) found, the trace evidence that identified them, and any lessons learned during the repair process.

## Concrete Examples

**Example 1: Silent logical bug — off-by-one in filtering**

User: "This function should return only positive numbers but it's including zero. Fix it."
```python
def get_positives(numbers):
    return [x for x in numbers if x >= 0]
# Test: get_positives([-1, 0, 1, 2]) should return [1, 2]
# Actual: [-1, 0, 1, 2] -> [0, 1, 2]  (wrong — includes 0)
```

Approach:
1. Run test — confirms failure: returns `[0, 1, 2]` instead of `[1, 2]`
2. Instrument:
   ```python
   def get_positives(numbers):
       result = []
       for x in numbers:
           condition = x >= 0
           print(f"DEBUG: loop | x={x}, condition(x>=0)={condition}")
           if condition:
               result.append(x)
       print(f"DEBUG: return | result={result}")
       return result
   ```
3. Trace output reveals: `DEBUG: loop | x=0, condition(x>=0)=True` — zero passes the filter
4. Root cause: `>=` should be `>` since zero is not a positive number
5. Fix: Change `x >= 0` to `x > 0`
6. Validate: `get_positives([-1, 0, 1, 2])` now returns `[1, 2]` — all tests pass

**Example 2: Multi-iteration fix with lesson learning**

User: "Fix this function that should return the second largest unique element."
```python
def second_largest(nums):
    nums.sort()
    return nums[-2]
# Test: second_largest([1, 3, 3, 5, 5]) should return 3
# Actual: returns 5
```

Approach:
1. Run test — fails: returns `5` instead of `3`
2. Instrument and trace:
   ```
   DEBUG: after sort | nums=[1, 3, 3, 5, 5]
   DEBUG: return | nums[-2]=5
   ```
3. Root cause: Duplicates aren't removed, so `nums[-2]` gets the second-to-last element (another 5), not the second-largest unique value
4. **Repair attempt 1**: Deduplicate with `set()`
   ```python
   def second_largest(nums):
       nums = sorted(set(nums))
       return nums[-2]
   ```
5. Validate: `second_largest([1, 3, 3, 5, 5])` → `3`. Passes!
6. Run all tests — also passes `second_largest([1])` → should raise error or handle edge case? If additional tests reveal edge case failures, record lesson and iterate.

**Example 3: Class-level bug with method interaction**

User: "My Stack class's `min()` method returns wrong values after `pop()`."
```python
class MinStack:
    def __init__(self):
        self.stack = []
        self.min_val = float('inf')
    def push(self, val):
        self.stack.append(val)
        self.min_val = min(self.min_val, val)
    def pop(self):
        return self.stack.pop()
    def min(self):
        return self.min_val
```

Approach:
1. Instrument `push`, `pop`, and `min` with probes logging `self.stack` and `self.min_val` after each operation
2. Trace for `push(2), push(1), pop(), min()`:
   ```
   DEBUG: push | val=2, stack=[2], min_val=2
   DEBUG: push | val=1, stack=[2, 1], min_val=1
   DEBUG: pop  | popped=1, stack=[2], min_val=1   <-- DIVERGENCE
   DEBUG: min  | returning min_val=1               <-- Wrong! Should be 2
   ```
3. Root cause: `pop()` removes the element but never recalculates `min_val`. When the minimum element is popped, `min_val` becomes stale.
4. Fix: Maintain an auxiliary min-stack that tracks the minimum at each level:
   ```python
   class MinStack:
       def __init__(self):
           self.stack = []
           self.min_stack = []
       def push(self, val):
           self.stack.append(val)
           self.min_stack.append(min(val, self.min_stack[-1] if self.min_stack else float('inf')))
       def pop(self):
           self.min_stack.pop()
           return self.stack.pop()
       def min(self):
           return self.min_stack[-1]
   ```
5. Validate with original trace sequence — `min()` now returns `2` after popping `1`.

## Best Practices

- **Do:** Always instrument before guessing. The trace reveals facts; intuition often misleads, especially on subtle bugs.
- **Do:** Keep probes minimal and targeted. Instrument the suspicious region first, then widen scope only if the trace doesn't reveal the issue.
- **Do:** Record every failed repair attempt with what was tried and why it failed. Explicitly review these records before planning the next attempt.
- **Do:** Validate against ALL tests after each repair, not just the one you were focused on. Regressions are common.
- **Avoid:** Modifying computational logic during instrumentation. Probes must be observation-only. Adding helper variables or changing control flow during instrumentation contaminates the trace.
- **Avoid:** Making multiple unrelated fixes in a single iteration. One targeted fix per cycle makes it clear what helped and what didn't.
- **Avoid:** Continuing beyond 5 iterations without progress. If stagnation persists, the problem likely needs a fundamentally different algorithmic approach rather than incremental patching.

## Error Handling

- **Instrumentation causes errors**: If adding probes introduces syntax errors or changes behavior, strip all probes and re-instrument more carefully. Verify the instrumented code produces the same outputs as the original (minus debug prints).
- **Trace is too verbose**: Reduce probe density. Start with function entry/exit and branch decisions only, then add loop-level probes in the suspicious region.
- **Infinite loops in instrumented code**: Add a loop iteration counter probe with a hard limit (`if iteration > 10000: print("DEBUG: loop limit reached"); break`). This reveals the infinite loop without hanging.
- **All tests pass after instrumentation**: The probes may have introduced a timing or side-effect change that masks the bug (rare but possible). Remove probes and verify the original bug still exists.
- **Rollback loop**: If the framework keeps rolling back to the same version, the repair strategy needs to change fundamentally. Review all lessons and consider whether the algorithm itself is wrong (not just a line-level bug).

## Limitations

- **Performance-sensitive code**: Print-based instrumentation adds I/O overhead. Not suitable for debugging timing-dependent bugs (race conditions, real-time constraints).
- **Non-deterministic behavior**: Traces from concurrent or random-seeded code may differ between runs, making causal analysis unreliable.
- **Missing test cases**: The entire framework depends on having test cases that expose the bug. If the tests don't cover the failing scenario, trace-driven debugging has nothing to observe.
- **Algorithmic incorrectness**: If the entire approach is wrong (e.g., using bubble sort when the problem requires a graph algorithm), instrumentation will reveal *that* it's wrong but won't suggest the correct algorithm. The technique is best for localizing bugs within a structurally sound solution.
- **Large codebases**: The method is designed for function-level and class-level debugging. For bugs spanning multiple modules or involving complex dependency chains, the trace volume may be unmanageable.

## Reference

[TraceCoder: A Trace-Driven Multi-Agent Framework for Automated Debugging of LLM-Generated Code](https://arxiv.org/abs/2602.06875v1) — Huang et al., 2026. Focus on Section 3 (framework architecture), Algorithm 1 (HLLM), and Algorithm 2 (Rollback Mechanism) for implementation details. The key takeaway: runtime traces + lesson learning + rollback enforcement achieves up to 34.43% relative improvement over pass/fail-only debugging methods.