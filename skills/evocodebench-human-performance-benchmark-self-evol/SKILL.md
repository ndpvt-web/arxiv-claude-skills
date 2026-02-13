---
name: "evocodebench-human-performance-benchmark-self-evol"
description: "Self-evolving code generation with iterative reflection and revision. Applies a feedback-driven loop where code is submitted, judged, analyzed for failures, and rewritten up to 3 times — tracking correctness, runtime, memory, and algorithmic improvement at each iteration. Use when: 'solve this coding problem and optimize it', 'iteratively improve this solution', 'refine my code until it passes all tests', 'benchmark my solution against human performance', 'reduce the time complexity of this code', 'fix and re-attempt this failing solution'."
---

# Self-Evolving Code Generation with Iterative Reflection-Revision

This skill implements the EvoCodeBench self-evolution methodology: a structured loop where Claude generates a solution, receives execution feedback (verdict, runtime, memory, error details), reflects on failures or inefficiencies, and produces an improved revision — repeating up to 3 rounds. Unlike one-shot code generation, this approach tracks correctness *and* efficiency dynamics across iterations, targeting not just passing tests but achieving competitive runtime and memory usage calibrated against human programmer distributions.

## When to Use

- When the user asks to solve a coding problem and wants iterative improvement until it passes all tests
- When a solution gets Wrong Answer, Time Limit Exceeded, or Runtime Error and needs systematic debugging
- When the user wants to optimize an accepted solution for better runtime or memory performance
- When solving competitive programming or LeetCode-style problems across multiple languages (Python, C++, Java, Go, Kotlin)
- When the user wants to compare solution efficiency against typical human submissions
- When converting a brute-force solution into one with better algorithmic complexity through structured reflection
- When a solution compiles in one language but fails in another and cross-language robustness is needed

## Key Technique

The EvoCodeBench methodology separates code evolution into two distinct reflection modes. **Bugfix reflection** activates when a submission fails (Wrong Answer, Runtime Error, Compile Error, Time/Memory Limit Exceeded). The agent analyzes the specific verdict and diagnostic output, identifies root causes — missed corner cases, off-by-one errors, incorrect data structures, language-specific syntax issues — and produces a targeted fix. **Optimization reflection** activates when a solution is accepted but its runtime or memory percentile is below a competitive threshold. Here the agent proposes algorithmic or implementation-level improvements: replacing O(n^2) scans with hash maps, switching from recursion to iteration to cut stack overhead, or using language-specific optimizations like `__builtin_popcount` in C++ vs manual bit counting.

The critical insight is that self-evolution must be *bounded and structured*. The loop runs at most 3 reflection-revision rounds and terminates early on acceptance with competitive performance. Each iteration preserves the full trajectory: the original code, the verdict, the reflection analysis, and the revised code. This trajectory record prevents regression — the agent can see what it already tried and avoid repeating failed strategies. Research shows this approach yields 10-27% relative pass-rate improvements and 7-46% runtime reductions depending on language, with the largest gains in compiled languages where compilation errors are systematically eliminated across iterations.

A key finding is that performance degrades from high-resource languages (Python, C++, Java) to long-tail languages (Kotlin, Go) due to training data imbalance. The self-evolution loop is especially valuable for these languages because it catches compilation errors and API misuse that one-shot generation misses entirely.

## Step-by-Step Workflow

1. **Parse the problem specification.** Extract constraints (input size bounds, time/memory limits), input/output format, edge cases from examples, and required algorithmic concepts. Identify the target language.

2. **Generate an initial solution with explicit reasoning.** Before writing code, state the chosen algorithm and its time/space complexity. Select data structures deliberately. For compiled languages (C++, Java, Go, Kotlin), pay extra attention to type declarations, imports, and language-specific API usage.

3. **Execute and collect feedback.** Run the solution against test cases. Record the verdict category: Accepted (AC), Wrong Answer (WA), Time Limit Exceeded (TLE), Memory Limit Exceeded (MLE), Compile Error (CE), or Runtime Error (RE). For AC, record runtime and memory. For failures, capture error messages, failing test case input/output if available, and the number of test cases passed before failure.

4. **Classify the failure mode and enter the appropriate reflection.** If CE: fix syntax, imports, or type issues. If RE: check null/nil access, array bounds, stack overflow, integer overflow. If WA: trace logic against the failing case, check edge cases (empty input, single element, maximum values). If TLE: analyze complexity and identify the bottleneck loop or recursion. If MLE: reduce auxiliary data structure size or switch to in-place algorithms.

5. **Perform structured reflection.** Write an explicit analysis with three fields: (a) **Root cause**: what specifically went wrong and why, (b) **Improved reasoning**: the corrected or optimized approach with complexity analysis, (c) **Improved code**: the full revised solution. Never patch blindly — always explain the fix rationale.

6. **Resubmit the revised solution.** Execute against the same test suite. Compare the new verdict and metrics against the previous iteration. If the verdict improved (e.g., WA to AC, or TLE to AC) but performance is suboptimal, continue to optimization reflection.

7. **Track the iteration trajectory.** Maintain a record of each attempt: `[iteration, verdict, tests_passed, runtime_ms, memory_mb, change_summary]`. Use this to prevent regression — if attempt 2 passes fewer tests than attempt 1, revert to attempt 1 and try a different strategy.

8. **Apply optimization reflection if accepted but slow.** Compare runtime against known human baselines or problem constraints. If runtime is above the 50th percentile of typical submissions, analyze: can the algorithm be improved (e.g., sorting + two-pointer instead of brute force)? Can constant factors be reduced (e.g., `StringBuilder` instead of string concatenation in Java)?

9. **Terminate after at most 3 revision rounds or on competitive acceptance.** Report the final solution alongside the full trajectory showing how correctness and efficiency evolved. If the problem remains unsolved after 3 rounds, report the best attempt and the unresolved failure mode.

10. **Provide cross-language guidance if applicable.** When the user needs the solution in multiple languages, translate the accepted solution while watching for language-specific pitfalls: integer overflow in Java (use `long`), Kotlin's nullable types, Go's lack of generics in older versions, C++ iterator invalidation.

## Concrete Examples

**Example 1: Wrong Answer resolved through reflection**

User: "Solve two-sum in Python and make sure it handles all edge cases."

```
Iteration 1:
  Reasoning: Use a hash map for O(n) lookup.
  Code:
    def twoSum(nums, target):
        seen = {}
        for i, n in enumerate(nums):
            if target - n in seen:
                return [seen[target - n], i]
            seen[n] = i
  Verdict: AC | Runtime: 48ms | Memory: 17.2MB
  Runtime Beats: 72% of human submissions

  Decision: Accepted with competitive performance. No further iteration needed.
  Trajectory: [(1, AC, all_passed, 48ms, 17.2MB, "initial hash map approach")]
```

**Example 2: TLE resolved by algorithmic improvement**

User: "Find the longest increasing subsequence length. My O(n^2) DP solution times out on large inputs."

```
Iteration 1 (user's code):
  Algorithm: O(n^2) DP with nested loops
  Verdict: TLE after 45/55 test cases

  Reflection:
    Root cause: O(n^2) is too slow for n=2500. Need O(n log n).
    Improved reasoning: Use patience sorting — maintain a list of smallest
    tail elements. For each number, binary search for its insertion point.
    This gives O(n log n) time, O(n) space.

Iteration 2:
  Code:
    import bisect
    def lengthOfLIS(nums):
        tails = []
        for x in nums:
            pos = bisect.bisect_left(tails, x)
            if pos == len(tails):
                tails.append(x)
            else:
                tails[pos] = x
        return len(tails)
  Verdict: AC | Runtime: 62ms | Memory: 16.8MB
  Runtime Beats: 85% of human submissions

  Trajectory:
    [(1, TLE, 45/55, -, -, "O(n^2) DP"),
     (2, AC, 55/55, 62ms, 16.8MB, "O(n log n) patience sort")]
```

**Example 3: Cross-language compilation error fix (Kotlin)**

User: "Port this Java solution to Kotlin. It uses PriorityQueue for Dijkstra's algorithm."

```
Iteration 1:
  Direct translation from Java to Kotlin.
  Verdict: CE — "Type mismatch: inferred type is Comparator<IntArray> but
  Comparator<in IntArray!>! was expected"

  Reflection:
    Root cause: Kotlin's type system treats Java generics differently.
    PriorityQueue comparator needs explicit typing.
    Improved reasoning: Use compareBy lambda instead of Comparator.comparing.

Iteration 2:
  Code: val pq = PriorityQueue<IntArray>(compareBy { it[1] })
  Verdict: AC | Runtime: 340ms | Memory: 52MB
  Runtime Beats: 61%

  Reflection (optimization):
    Root cause: IntArray boxing overhead in PriorityQueue.
    Improved reasoning: Use a custom Pair with primitive-friendly access.

Iteration 3:
  Switched to indexed array with manual heap. Reduced boxing.
  Verdict: AC | Runtime: 215ms | Memory: 45MB
  Runtime Beats: 84%

  Trajectory:
    [(1, CE, 0/0, -, -, "Java-to-Kotlin type mismatch"),
     (2, AC, all, 340ms, 52MB, "fixed comparator syntax"),
     (3, AC, all, 215ms, 45MB, "eliminated boxing overhead")]
```

## Best Practices

- **Do:** Always state the algorithm's time and space complexity before writing code. This catches TLE/MLE issues at design time rather than after submission.
- **Do:** Maintain the full iteration trajectory and reference it during reflection. This prevents cycling between two broken approaches.
- **Do:** Treat compilation errors in compiled languages as the highest-priority fix — a CE gives zero diagnostic signal about correctness, so it blocks all further analysis.
- **Do:** After achieving AC, check if the solution beats at least 50% of human submissions in runtime. If not, consider one optimization round.
- **Avoid:** Making multiple unrelated changes in a single revision. Change one thing per iteration so you can attribute improvement or regression to a specific modification.
- **Avoid:** Exceeding 3 revision rounds. Diminishing returns set in rapidly. If 3 rounds fail, the problem likely requires a fundamentally different algorithm, not incremental patches.
- **Avoid:** Assuming Python idioms transfer to compiled languages. Watch for: integer overflow (Java/C++ vs Python's arbitrary precision), null safety (Kotlin), explicit memory management patterns (C++), and import/package differences (Go).

## Error Handling

| Failure Mode | Diagnostic Signal | Recovery Strategy |
|---|---|---|
| **Wrong Answer** | Number of tests passed before failure | Trace logic on the smallest failing case. Check edge cases: empty input, single element, duplicates, negative numbers, maximum constraint values. |
| **Time Limit Exceeded** | Tests passed before timeout | Analyze the dominant loop's complexity. Replace O(n^2) with O(n log n) or O(n) approaches. Consider: sorting, binary search, hash maps, monotonic stacks, segment trees. |
| **Memory Limit Exceeded** | Memory spike pattern | Switch from explicit storage to streaming/in-place computation. Replace 2D DP with rolling array. Use bitsets instead of boolean arrays. |
| **Compile Error** | Compiler error message | Fix the exact line cited. Common across languages: missing imports, type mismatches, syntax differences. In Kotlin/Go, check API compatibility. |
| **Runtime Error** | Stack trace or signal | Check: null/nil dereference, array index out of bounds, stack overflow from deep recursion (convert to iteration), integer division by zero. |
| **Regression across iterations** | Trajectory comparison | Revert to the best prior attempt. Analyze why the new change broke passing tests. Try an orthogonal fix strategy. |

## Limitations

- **This approach requires execution feedback.** Without a judge or test suite, the reflection loop cannot verify whether a revision actually improved the solution. In the absence of executable tests, Claude can still reason about correctness but cannot confirm it.
- **3 iterations may not suffice for hard algorithmic problems** requiring non-obvious data structures (e.g., link-cut trees, suffix automata). These problems may need a complete algorithm redesign rather than iterative refinement.
- **Long-tail language support is inherently weaker.** Solutions in Kotlin, Go, or other less-common languages may have persistent compilation or API issues that reflection alone cannot fully resolve due to gaps in training data.
- **Human percentile baselines depend on the problem platform.** The runtime/memory beats metrics are calibrated against LeetCode's submission distribution, which may not reflect performance on other platforms or custom problem sets.
- **Optimization reflection can over-optimize.** Micro-optimizations (bit manipulation tricks, cache-line alignment) may reduce readability without meaningful runtime gains. Optimize the algorithm first; optimize constants only when necessary.

## Reference

**Paper:** [EvoCodeBench: A Human-Performance Benchmark for Self-Evolving LLM-Driven Coding Systems](https://arxiv.org/abs/2602.10171v1) — Zhang et al., 2026. Focus on Section 3 (benchmark design and the reflection-revision protocol), Section 4 (self-evolving agent construction), and Section 5 (results showing iteration-over-iteration gains and human-relative percentile analysis).