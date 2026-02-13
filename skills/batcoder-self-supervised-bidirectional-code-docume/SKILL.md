---
name: "batcoder-self-supervised-bidirectional-code-docume"
description: "Apply BatCoder's back-translation technique to improve code and documentation quality bidirectionally. Generate documentation from code, then verify it by reconstructing code from that documentation -- using cycle consistency as a quality signal. Triggers: 'generate documentation from code', 'back-translate code and docs', 'verify documentation quality', 'improve code with back-translation', 'self-supervised code documentation', 'cycle-consistent code generation'"
---

# BatCoder: Self-Supervised Bidirectional Code-Documentation Learning via Back-Translation

This skill enables Claude to apply the BatCoder back-translation technique for jointly improving code generation and documentation production. The core idea: generate documentation from code, then reconstruct the original code from that documentation alone. If the reconstructed code faithfully matches the original, the documentation is semantically complete. If it diverges, the documentation has gaps. This cycle-consistency signal drives iterative improvement of both code clarity and documentation accuracy without requiring curated training pairs.

## When to Use

- When a user asks to generate or improve documentation for existing code and wants verification that the docs are complete and accurate
- When a user has documentation (docstrings, comments, specs) and wants to validate it captures all functional behavior by reconstructing code from it
- When refactoring code and needing to ensure the new implementation preserves the semantics described by existing documentation
- When working with undocumented code in niche or less-common programming languages where paired examples are scarce
- When a user wants to iteratively improve both a function's implementation clarity and its documentation simultaneously
- When reviewing code-documentation pairs for semantic drift (docs say one thing, code does another)

## Key Technique

BatCoder introduces a **back-translation cycle** borrowed from machine translation. In natural language translation, back-translation generates synthetic training data by translating target-language text back to the source language. BatCoder applies this to code: given a code snippet `c`, first generate documentation `d = f(c)`, then reconstruct code `c' = g(d)`. The semantic similarity between `c` and `c'` -- measured via structural and semantic comparison -- serves as an implicit quality signal for both the documentation and the code generation process.

The key insight is that **documentation quality can be measured indirectly through code reconstruction fidelity**. If generated documentation captures all the logic, control flow, edge cases, and type constraints of the original code, then an independent code generation pass from that documentation should produce functionally equivalent code. Gaps in the documentation surface as divergences in the reconstructed code. This eliminates the need for human-annotated code-documentation pairs.

The reward combines two components: (1) a **structural validity check** on the documentation (does it contain required elements like description, parameter types, examples?), scored as `R_doc in {0, 0.5, 1}`, and (2) a **semantic similarity score** `S(c, c')` between original and reconstructed code, computed using program dependence graph comparison (CSSG metric, range [0,1]). The combined reward `R = R_sim * R_doc` drives iterative refinement. This approach scales with corpus size since it only requires raw code, not paired data.

## Step-by-Step Workflow

1. **Accept the input code snippet.** Read the function or module the user wants documented. Identify the programming language, function signature, parameter types, return types, and any imports or dependencies.

2. **Generate candidate documentation (Code-to-Doc).** Produce a detailed docstring or specification from the code. Include: a one-line summary, detailed description of behavior, parameter descriptions with types, return value description, edge cases, and at least one input/output example using the language's doctest convention (e.g., `>>>` for Python, `//` examples for Go).

3. **Validate documentation structure.** Check that the generated documentation contains all required structural components: description, typed parameters, return specification, and usage examples. Assign a structural validity score: 1.0 if all present, 0.5 if partially present, 0.0 if missing critical elements. Reject and regenerate if score is 0.

4. **Reconstruct code from documentation alone (Doc-to-Code).** Using ONLY the generated documentation as input (not the original code), write a fresh implementation. Do not reference the original code during this step -- treat the documentation as the sole specification.

5. **Compare original and reconstructed code for semantic equivalence.** Analyze both implementations across these dimensions:
   - **Control flow:** Do both handle the same branches, loops, and early returns?
   - **Edge cases:** Does the reconstruction handle the same boundary conditions?
   - **Type behavior:** Are parameter and return types consistent?
   - **Side effects:** Are I/O operations, mutations, and state changes preserved?
   - **Algorithmic equivalence:** Do both use approaches that produce identical outputs for all inputs?

6. **Identify semantic gaps.** Where the reconstructed code diverges from the original, trace the divergence back to missing or ambiguous information in the documentation. List each gap explicitly.

7. **Refine the documentation.** Update the documentation to close each identified gap. Add missing edge case descriptions, clarify ambiguous parameter constraints, and add examples that cover the divergent behavior.

8. **Optionally refine the code.** If the back-translation reveals that the original code has unclear logic (e.g., magic numbers, implicit assumptions, convoluted control flow), suggest code improvements that make the intent more explicit without changing behavior.

9. **Re-run the cycle if needed.** For complex functions, repeat steps 4-7 once more to verify the refined documentation now produces a faithful reconstruction.

10. **Deliver the final documentation and any code suggestions.** Present the validated documentation alongside a summary of what the back-translation cycle revealed and fixed.

## Concrete Examples

**Example 1: Python function with subtle edge cases**

User: "Generate documentation for this function."

```python
def merge_intervals(intervals):
    if not intervals:
        return []
    intervals.sort(key=lambda x: x[0])
    merged = [intervals[0]]
    for current in intervals[1:]:
        if current[0] <= merged[-1][1]:
            merged[-1] = (merged[-1][0], max(merged[-1][1], current[1]))
        else:
            merged.append(current)
    return merged
```

Approach:
1. **Code-to-Doc:** Generate documentation capturing sorting, merge condition (`<=` not `<`), tuple reconstruction, and empty-input handling.
2. **Doc-to-Code reconstruction** (from docs only): Write a fresh implementation based solely on the documentation.
3. **Compare:** Check if reconstruction handles: empty list, single interval, adjacent intervals (e.g., `(1,2)` and `(2,3)` -- the `<=` matters), overlapping intervals, already-sorted input, tuples vs lists.
4. **Gap found:** Initial docs said "overlapping intervals" but didn't specify that touching intervals (`end == start`) also merge. Reconstruction used `<` instead of `<=`.
5. **Fix:** Documentation updated to state "Intervals that overlap or touch (where one ends exactly where another begins) are merged."

Output:
```python
def merge_intervals(intervals):
    """Merge overlapping or touching intervals into non-overlapping intervals.

    Takes a list of (start, end) tuples and returns a new list where all
    overlapping or adjacent intervals are combined. Two intervals are merged
    if one starts at or before the other ends (i.e., touching intervals like
    (1, 2) and (2, 3) become (1, 3)).

    The input list is sorted in-place by start value before merging.

    Args:
        intervals: List of (start, end) tuples. Can be empty.

    Returns:
        List of merged (start, end) tuples in sorted order.
        Returns [] for empty input.

    Examples:
        >>> merge_intervals([(1, 3), (2, 6), (8, 10)])
        [(1, 6), (8, 10)]
        >>> merge_intervals([(1, 2), (2, 3)])
        [(1, 3)]
        >>> merge_intervals([])
        []
    """
```

**Example 2: Go function -- back-translation reveals ambiguity**

User: "Document this Go function and verify the docs are accurate."

```go
func Deduplicate(items []string) []string {
    seen := make(map[string]bool)
    result := []string{}
    for _, item := range items {
        if !seen[strings.ToLower(item)] {
            seen[strings.ToLower(item)] = true
            result = append(result, item)
        }
    }
    return result
}
```

Approach:
1. **Code-to-Doc:** Generate documentation noting case-insensitive deduplication.
2. **Doc-to-Code reconstruction:** Independent implementation from docs alone.
3. **Compare:** Reconstruction preserved case-insensitivity but returned lowercased strings instead of originals. The docs said "removes case-insensitive duplicates" but didn't specify that the *first occurrence's original casing* is preserved.
4. **Fix:** Documentation updated: "Removes case-insensitive duplicates, preserving the original casing of the first occurrence of each unique string."

Output:
```go
// Deduplicate removes case-insensitive duplicate strings from the input slice,
// preserving the original casing of the first occurrence of each unique string.
// Order of first appearances is maintained.
//
// Comparison is performed by lowercasing each string via strings.ToLower.
// For input ["Hello", "HELLO", "world"], returns ["Hello", "world"].
//
// Returns an empty slice (not nil) for empty input.
func Deduplicate(items []string) []string {
```

**Example 3: Validating existing documentation**

User: "Check if this docstring actually matches what the code does."

```python
def retry(func, max_attempts=3, delay=1.0):
    """Retry a function up to max_attempts times with a fixed delay between attempts.

    Args:
        func: Callable to retry.
        max_attempts: Maximum number of retries (default 3).
        delay: Seconds to wait between retries (default 1.0).

    Returns:
        The return value of func on success.

    Raises:
        Exception: The last exception if all attempts fail.
    """
    for attempt in range(max_attempts):
        try:
            return func()
        except Exception:
            if attempt == max_attempts - 1:
                raise
            time.sleep(delay * (2 ** attempt))
```

Approach:
1. **Doc-to-Code from existing docs:** Reconstruct using only the docstring. The docs say "fixed delay" so reconstruction uses `time.sleep(delay)` each time.
2. **Compare:** Original code uses `delay * (2 ** attempt)` -- exponential backoff, not fixed delay.
3. **Semantic drift found:** Documentation says "fixed delay" but code implements exponential backoff.
4. **Report:** Flag the discrepancy. Ask user whether to fix the docs (change "fixed delay" to "exponential backoff starting at `delay` seconds, doubling each attempt") or fix the code (change to actual fixed delay).

## Best Practices

- **Do:** Generate documentation that includes concrete input/output examples -- these are the most powerful element for faithful reconstruction, and their absence is the #1 cause of reconstruction divergence.
- **Do:** Treat the reconstruction step as a genuine black-box exercise. If you peek at the original code while reconstructing, the technique loses all value. Reconstruct from documentation alone.
- **Do:** Pay special attention to boundary conditions and off-by-one semantics (`<` vs `<=`, `range(n)` vs `range(n+1)`) -- these are the most common gaps that back-translation reveals.
- **Do:** Use language-appropriate documentation conventions (Python docstrings with `>>>` examples, JSDoc for JavaScript, GoDoc comments for Go) to maximize structural validity.
- **Avoid:** Skipping the structural validation step. Documentation without examples or type information produces unreliable reconstructions regardless of description quality.
- **Avoid:** Running more than 2 back-translation cycles on simple functions. Diminishing returns set in quickly for straightforward code. Reserve multiple cycles for complex functions with many branches.

## Error Handling

- **Reconstruction produces syntactically invalid code:** The documentation is too vague or missing critical context (imports, types). Add explicit type signatures and import requirements to the documentation before retrying.
- **Reconstruction is correct but uses a completely different algorithm:** The documentation described *what* but not *how*, which is often acceptable. Only flag this if algorithmic choice affects performance characteristics or side effects that matter.
- **Original code has bugs that the documentation "fixes":** When reconstruction diverges because the docs describe intended behavior while the code has a bug, flag this as a potential bug in the original code rather than a documentation gap.
- **Language-specific idioms lost in translation:** Some constructs (list comprehensions, goroutines, pattern matching) may not survive a back-translation cycle. Document the *intent* behind idiomatic constructs, not just their behavior.

## Limitations

- This technique works best for **pure functions** with clear inputs and outputs. Stateful code, UI event handlers, and code with heavy I/O dependencies are harder to faithfully reconstruct from documentation alone.
- The back-translation cycle validates **functional completeness** of documentation but not **readability** or **pedagogical quality**. Technically complete docs can still be poorly written.
- For very short functions (1-3 lines), the overhead of back-translation exceeds its benefit. Use direct documentation for trivial code.
- Semantic comparison relies on behavioral equivalence reasoning, which can miss subtle concurrency bugs, floating-point precision differences, or platform-specific behavior.
- The technique assumes the original code is correct. If the code itself is buggy, the cycle may produce documentation that faithfully describes buggy behavior.

## Reference

**Paper:** [BatCoder: Self-Supervised Bidirectional Code-Documentation Learning via Back-Translation](https://arxiv.org/abs/2602.02554v1) (Xu et al., 2026). Key sections: Section 3 for the back-translation framework and reward formulation (`R = R_sim * R_doc`), Section 4 for ablation results showing Stage 1 (code-to-doc) is critical, and Table 1 for benchmark comparisons demonstrating 83.5% pass@1 on HumanEval.