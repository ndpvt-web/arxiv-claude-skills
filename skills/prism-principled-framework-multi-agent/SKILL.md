---
name: "prism-principled-framework-multi-agent"
description: >
  Apply the PRISM (Propose-Review-Integrate Synthesis) multi-agent reasoning framework
  to decompose hard problems into parallel proposals with diverse roles, execution-grounded
  review, and iterative synthesis. Use when asked to: "solve this with multiple approaches",
  "use PRISM to solve", "compare different solution strategies", "multi-agent reasoning on this problem",
  "propose-review-integrate this task", "find the best solution by exploring alternatives".
---

# PRISM: Principled Multi-Agent Reasoning via Gain Decomposition

This skill enables Claude to apply the PRISM framework from Yang et al. (2026) to tackle complex reasoning tasks — mathematical problems, code generation, function calling, and architectural decisions — by systematically generating diverse proposals through role-specialized agents, grounding evaluation in execution feedback, and synthesizing a final solution through iterative closed-loop refinement. Instead of relying on a single reasoning path, PRISM jointly maximizes three orthogonal dimensions of multi-agent gain: Exploration (solution diversity), Information (feedback fidelity), and Aggregation (principled consensus).

## When to Use

- When the user asks Claude to solve a hard coding problem and wants confidence the solution is correct (e.g., algorithmic challenges, competitive programming)
- When the user requests multiple approaches to a problem compared and the best one selected with justification
- When implementing a function where correctness is critical and can be validated by tests or execution
- When designing an API, schema, or system where there are multiple valid architectures and the user wants a principled comparison
- When debugging a tricky issue where the root cause is unclear and multiple hypotheses should be explored in parallel
- When the user explicitly asks for "PRISM", "propose-review-integrate", or "multi-agent reasoning"

## Key Technique

PRISM decomposes the performance gain of multi-agent reasoning into three independent dimensions. **Exploration gain** comes from generating diverse candidate solutions so at least one is correct — achieved by assigning orthogonal cognitive roles (Minimalist, Skeptic, Explorer) that induce negative correlation between failure modes. **Information gain** comes from grounding evaluation in high-fidelity feedback — execution results, test outputs, and error traces rather than surface-level textual critique. **Aggregation gain** comes from principled synthesis rather than naive voting — combining validated components from different candidates through trajectory grafting and closed-loop re-execution.

Existing multi-agent methods optimize only subsets of these dimensions: majority voting exploits exploration but ignores information quality; debate improves aggregation but relies on ungrounded "cheap talk"; mixture-of-agents layers responses but saturates early. PRISM jointly maximizes all three, achieving state-of-the-art results at 5x lower token cost than comparable methods.

The key insight is that **diversity in generation + execution-grounded evaluation + iterative synthesis** creates a system where each component reinforces the others. Diverse proposals ensure at least one viable path exists. Execution feedback objectively identifies which parts work. Synthesis grafts working components together and validates the result, feeding errors back for targeted fixes across up to 3 refinement iterations.

## Step-by-Step Workflow

1. **Identify the task type and validation method.** Determine whether the problem admits deterministic validation (code with tests, API calls with schemas, math with verifiable answers) or requires model-based evaluation. Deterministic feedback provides maximum information gain.

2. **Generate three diverse proposals using orthogonal roles.** Produce three independent candidate solutions, each from a distinct cognitive stance:
   - **Minimalist**: Solve with the fewest steps. Prioritize simplicity, directness, and minimal abstractions. Favor the obvious correct approach.
   - **Skeptic**: Verify each step rigorously. Prioritize correctness over elegance. Explicitly check edge cases, off-by-one errors, type boundaries, and failure modes.
   - **Explorer**: Avoid the obvious approach. Try unconventional methods, alternative algorithms, or different data structures. Prioritize creativity and novel angles.

3. **Execute each proposal and collect grounded feedback.** Run each candidate against available tests, type checkers, linters, or validation tools. Capture the full execution result tuple: `(success/fail, output, test results, error traces)`. If no execution environment is available, perform structured model-based verification with explicit step-by-step checking.

4. **Cross-evaluate proposals using execution evidence.** For each proposal, analyze its execution results relative to the others. Identify: (a) which components passed validation, (b) specific failure modes with root causes from error traces, (c) validated strengths worth preserving, (d) concrete improvements suggested by comparing successful and failing approaches.

5. **Synthesize a combined solution via trajectory grafting.** Produce a new integrated solution that combines the validated components from the strongest proposals. Use working code/logic from successful candidates and replace failing sections with approaches that passed in other candidates. Do not simply pick one winner — actively merge the best parts.

6. **Re-execute the synthesized solution (closed-loop validation).** Run the synthesized result through the same validation. If it passes, proceed. If it fails, analyze the new error traces and perform a targeted fix informed by the specific failure.

7. **Iterate synthesis up to 3 rounds.** Repeat steps 5-6 up to three times total. Each iteration uses the accumulated execution feedback to make progressively more targeted corrections. Stop early if validation passes completely.

8. **Deliver the final solution with a provenance summary.** Present the validated solution along with a brief explanation of which role's approach contributed which components and what the synthesis corrected.

## Concrete Examples

**Example 1: Solving an algorithmic coding problem**

User: "Write a function that finds the longest increasing subsequence in an array. Make sure it's correct and efficient."

Approach:
1. **Minimalist proposal**: Standard O(n log n) patience sorting approach using binary search on tail elements. Clean, minimal implementation.
2. **Skeptic proposal**: O(n^2) dynamic programming approach with explicit tracking of the actual subsequence (not just length). Includes edge case handling for empty arrays, single elements, and all-decreasing inputs.
3. **Explorer proposal**: Segment tree approach that processes elements and queries for maximum LIS ending before each value. O(n log n) but via a different mechanism than patience sorting.

Execution: Run all three against test cases — `[10,9,2,5,3,7,101,18]` (expected length 4), `[]`, `[1]`, `[5,4,3,2,1]`, `[1,2,3,4,5]`.

Cross-evaluation: Minimalist passes all tests and runs fastest. Skeptic passes but is slower; however, it correctly reconstructs the actual subsequence, not just the length. Explorer has a bug in segment tree index mapping causing failure on the decreasing-input case.

Synthesis: Take the Minimalist's O(n log n) core algorithm, add the Skeptic's subsequence reconstruction logic and edge case guards, discard the Explorer's segment tree (correct in theory but implementation was buggy).

Re-execute: All tests pass. Final solution is O(n log n) with full subsequence reconstruction and robust edge case handling.

**Example 2: Designing a rate limiter**

User: "Implement a rate limiter that allows 100 requests per minute per user."

Approach:
1. **Minimalist proposal**: Fixed window counter using a dict mapping `(user_id, minute_bucket)` to count. Simple, 15 lines of code.
2. **Skeptic proposal**: Sliding window log storing timestamps of all requests. Exact enforcement but higher memory. Includes thread-safety via a lock. Validates boundary conditions at exactly 60-second mark.
3. **Explorer proposal**: Token bucket algorithm with configurable burst. Allows short bursts above 100/min as long as average rate is maintained. More flexible for real-world traffic patterns.

Execution: Test with simulated traffic — steady 2/sec, burst of 50 at once, requests spanning window boundaries.

Cross-evaluation: Minimalist fails the boundary-spanning test (user gets 200 requests by hitting end of one window and start of next). Skeptic passes all tests but uses O(n) memory per user. Explorer passes but its burst allowance means it doesn't strictly enforce 100/min.

Synthesis: Use the Skeptic's sliding window approach for correctness, but optimize memory by switching to a sliding window counter (hybrid of fixed window counts) that achieves O(1) memory while avoiding the Minimalist's boundary bug. Incorporate the Skeptic's thread-safety lock.

Re-execute: All tests pass including boundary case. Memory usage is constant per user.

**Example 3: Debugging a failing test**

User: "This test keeps failing intermittently. Help me figure out why."
```python
def test_cache_expiry():
    cache.set("key", "value", ttl=1)
    time.sleep(1)
    assert cache.get("key") is None
```

Approach:
1. **Minimalist hypothesis**: The TTL is exactly 1 second and `time.sleep(1)` doesn't guarantee sleeping for >= 1 second. The get happens before expiry.
2. **Skeptic hypothesis**: The cache expiry mechanism is lazy (checks on access) but there may be a timing granularity issue — the cache stores expiry as integer seconds, and depending on when within a second the set happens, the effective TTL is 0-1 seconds, not exactly 1.
3. **Explorer hypothesis**: There's a race condition — another test or background thread is re-setting the same key, so the get finds a fresh value.

Execution: Inspect the cache implementation source code. Check if TTL is stored as integer or float. Check for shared state across tests.

Cross-evaluation: Cache stores expiry as `int(time.time()) + ttl`, confirming the Skeptic's hypothesis — setting at time 10.9 with ttl=1 expires at 11 (0.1s later, not 1s). The Minimalist's hypothesis is partially correct but the root cause is the integer truncation. Explorer's hypothesis is ruled out — no shared state found.

Synthesis: The fix is to use `math.ceil(time.time()) + ttl` or `time.time() + ttl` with float comparison in the cache, and change the test to `time.sleep(1.1)` for robustness. Both the storage bug and the test margin are addressed.

## Best Practices

- **Do**: Always generate all three role-based proposals before evaluating any of them. Premature evaluation biases the exploration.
- **Do**: Ground every evaluation claim in concrete evidence — test output, error messages, execution traces. Never say "this looks correct" without running it.
- **Do**: During synthesis, explicitly name which proposal contributed each component. This creates traceability and helps the user understand the reasoning.
- **Do**: Use deterministic execution feedback (running code, validating schemas) whenever possible. It provides strictly more information than textual review.
- **Avoid**: Generating three superficially different proposals that share the same core logic. The roles must produce genuinely orthogonal approaches — different algorithms, different data structures, or different architectural patterns.
- **Avoid**: Defaulting to majority voting. The power of PRISM is in synthesis (combining best parts), not selection (picking one winner). Voting discards valuable partial solutions.
- **Avoid**: Running more than 3 synthesis iterations. Empirically, gains diminish sharply after 3 rounds, and additional iterations waste tokens without improving quality.

## Error Handling

- **All three proposals fail execution**: Fall back to analyzing the error traces across all three to identify common failure patterns. Often, three different failures triangulate the actual constraint being violated. Generate a fourth "diagnostic" proposal informed by all three error traces.
- **Synthesis introduces new bugs not present in any original proposal**: This occurs when grafting incompatible components. Revert to the single best-performing proposal as the synthesis base, and apply targeted patches from the others rather than a full merge.
- **No execution environment available**: Use structured self-verification — trace through each proposal step by step with concrete inputs, maintaining explicit variable state. This is weaker than execution but still superior to surface-level review.
- **Problem has no objectively verifiable answer** (e.g., design decisions): Replace execution feedback with explicit criteria evaluation. Define 3-5 evaluation criteria upfront (performance, maintainability, extensibility, simplicity), score each proposal against each criterion, and synthesize based on the weighted criteria.

## Limitations

- **Simple tasks don't benefit.** If the problem has an obvious single solution (e.g., "reverse a string"), the overhead of three proposals and synthesis adds cost without improving quality. Use PRISM only when the problem is genuinely hard or has multiple viable approaches.
- **Token cost scales linearly with proposal count.** Three proposals + cross-evaluation + synthesis iterations consume roughly 5-8x the tokens of a single attempt. This is still 5x cheaper than exhaustive methods like Mixture-of-Agents, but non-trivial for simple queries.
- **Requires executable validation for maximum benefit.** The Information dimension is most powerful with deterministic feedback. For tasks like creative writing or subjective design, the framework degrades to model-based evaluation, which has lower information content.
- **Role diversity depends on problem type.** The Minimalist/Skeptic/Explorer roles work well for coding and math. For other domains, the roles may need adaptation (e.g., for system design: Pragmatist/Security-First/Scalability-First).

## Reference

**Paper**: Yang et al., "PRISM: A Principled Framework for Multi-Agent Reasoning via Gain Decomposition" (2026). arXiv:2602.08586v2. https://arxiv.org/abs/2602.08586v2

**What to look for**: Theorem 3.1 for the formal gain decomposition; Table 1 for how existing methods map to partial dimension optimization; Section 4 for the complete PRISM algorithm with role prompts; Figure 3 for compute-efficiency scaling curves showing PRISM reaches parity with Mixture-of-Agents at 5x fewer tokens.