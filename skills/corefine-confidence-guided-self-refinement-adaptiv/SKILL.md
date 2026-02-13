---
name: "corefine-confidence-guided-self-refinement-adaptiv"
description: "Confidence-guided self-refinement for adaptive reasoning. Implements the CoRefine pattern: assess confidence in each reasoning step, then decide whether to halt, re-examine, or try a different approach -- reducing wasted compute while maintaining accuracy. Use when: 'solve this step by step with self-correction', 'refine your reasoning until confident', 'adaptively debug this problem', 'use confidence-guided refinement', 'self-correct with backtracking', 'try different approaches if stuck'."
---

# CoRefine: Confidence-Guided Self-Refinement for Adaptive Reasoning

This skill enables Claude to apply the CoRefine self-refinement protocol from [Jin et al., 2026] when tackling complex reasoning, debugging, or multi-step problem-solving tasks. Instead of generating many candidate solutions in parallel or blindly iterating, Claude monitors its own confidence at each reasoning step and uses that signal to decide: **halt** (accept the answer), **re-examine** (revisit the current approach with fresh scrutiny), or **pivot** (abandon the current path and try a fundamentally different strategy). This produces higher-quality answers with far fewer wasted tokens, mirroring the paper's ~190x token reduction over brute-force parallel sampling.

## When to Use

- When solving multi-step math, logic, or algorithmic problems where intermediate errors compound.
- When debugging code where the root cause is unclear and you need to systematically narrow hypotheses.
- When the user asks you to "think carefully", "double-check your work", or "refine your reasoning".
- When generating code that must be correct on the first try (e.g., migration scripts, security-critical logic).
- When a previous attempt produced a wrong or suspicious answer and the user wants iterative improvement.
- When tackling ambiguous requirements where you need to explore multiple interpretations before committing.
- When the user explicitly requests confidence-guided or adaptive self-correction.

## Key Technique

**Confidence as a control signal, not a correctness guarantee.** The core insight of CoRefine is that tracking how confident you are across a chain of reasoning steps reveals when you are likely wrong -- even without ground truth. A sudden drop in confidence mid-derivation, hedging language, or reliance on unverified assumptions are all observable signals that the current path may be failing. Rather than completing a shaky chain and hoping for the best, CoRefine uses these signals to trigger corrective actions in real time.

**Three-action controller.** At each reasoning checkpoint, exactly one action is chosen: (1) **Halt** -- confidence is high and stable across recent steps; accept the current answer. (2) **Re-examine** -- confidence dipped on the most recent step but the overall approach seems sound; re-derive or verify that specific step before continuing. (3) **Pivot** -- confidence has been declining across multiple steps or a fundamental assumption looks wrong; abandon the current strategy and try a qualitatively different approach. The paper's Conv1D controller learns these decision boundaries from confidence trajectories; in our LLM-native adaptation, Claude applies the same logic by explicitly tracking and reasoning about its confidence at each checkpoint.

**CoRefine-Tree for exploration vs. exploitation.** When a problem admits multiple valid solution strategies, the refinement loop can branch: maintain 2-3 live approaches in parallel (exploration), but invest deeper refinement only in the most promising branch (exploitation). This hybrid avoids both tunnel vision on a single bad path and the waste of fully parallel sampling.

## Step-by-Step Workflow

1. **Parse the problem and identify checkpoints.** Break the task into discrete reasoning steps or subtasks. Each step becomes a confidence checkpoint. For code, this means: understand requirements, design approach, implement each function, verify correctness. For math, this means: each derivation step.

2. **Generate the first reasoning step and assess confidence.** Produce the initial step. Then explicitly evaluate: "How confident am I in this step? What assumptions am I making? Is anything uncertain?" Assign a qualitative confidence level: HIGH (no doubts, well-understood domain), MEDIUM (plausible but unverified), or LOW (guessing, unfamiliar territory, multiple valid alternatives).

3. **Apply the three-action decision rule.**
   - **HIGH confidence on this step and all prior steps:** Continue to the next step.
   - **MEDIUM confidence on this step (but prior steps are solid):** **Re-examine** -- re-derive this step from scratch, check edge cases, or verify with a concrete example before proceeding.
   - **LOW confidence, or MEDIUM confidence for 2+ consecutive steps:** **Pivot** -- explicitly state what is going wrong, identify the flawed assumption, and restart from the last HIGH-confidence checkpoint using a different strategy.

4. **Track the confidence trajectory.** Maintain a running record of confidence levels across steps. A pattern like [HIGH, HIGH, MEDIUM, LOW] is a clear pivot signal. A pattern like [HIGH, HIGH, HIGH, MEDIUM, HIGH] after re-examination indicates successful recovery.

5. **On re-examine: verify the specific step.** Do not repeat the entire chain. Isolate the uncertain step. Test it with a concrete input, check it against a known identity, or derive it via an independent method. If it holds up, upgrade confidence and continue. If it fails, fix it or pivot.

6. **On pivot: try a qualitatively different approach.** Do not make minor tweaks to the same strategy. Change the algorithm, the data structure, the mathematical technique, or the interpretation of the problem. Reference the failed approach to avoid repeating the same mistake.

7. **Halt when confidence is HIGH across all steps.** Once every checkpoint in the chain reads HIGH, present the final answer. State the key verification that gave you confidence.

8. **Cap refinement at 4 iterations.** If you have pivoted 3 times without reaching high confidence, present the best available answer with an explicit statement of remaining uncertainty. The paper averages 2.7 refinement steps; exceeding 4 suggests the problem may need external input.

9. **For multi-strategy problems, use CoRefine-Tree.** When the problem clearly admits 2-3 valid approaches (e.g., recursive vs. iterative, SQL vs. pandas, analytical vs. numerical), briefly sketch each approach (exploration), then invest full refinement in the most promising one (exploitation). Fall back to the second approach only if the first fails after re-examination.

10. **Summarize the refinement trace.** After halting, briefly report: how many refinement steps were taken, what was re-examined or pivoted, and why the final answer is trustworthy. This gives the user transparency into the reasoning process.

## Concrete Examples

**Example 1: Debugging a subtle off-by-one error**

User: "This function is supposed to return the k-th largest element but it's returning wrong results for some inputs. Fix it."

```python
def kth_largest(nums, k):
    nums.sort()
    return nums[len(nums) - k + 1]
```

Approach:
1. **Checkpoint 1 -- Understand the bug.** The function sorts ascending and indexes from the end. Confidence: HIGH (clear intent).
2. **Checkpoint 2 -- Verify the index math.** For `nums = [1,2,3,4,5]`, `k=1` (largest). `len(nums) - k + 1 = 5`. Index 5 is out of bounds. Confidence: LOW -- the index formula is wrong.
3. **Action: Re-examine.** The correct index for k-th largest in ascending-sorted array is `len(nums) - k`, which gives index 4 = value 5. Verify: k=2 gives index 3 = value 4. Confidence: HIGH.
4. **Halt.**

Output:
```python
def kth_largest(nums, k):
    nums.sort()
    return nums[len(nums) - k]  # Fixed: removed erroneous +1
```
Refinement trace: 1 re-examination on index math. Verified with k=1 and k=2 on [1,2,3,4,5].

---

**Example 2: Solving an algorithmic problem with a pivot**

User: "Given a list of intervals, merge all overlapping intervals. Optimize for clarity and correctness."

Approach:
1. **Checkpoint 1 -- Initial strategy: brute force pairwise comparison.** Compare every pair, merge if overlapping, repeat until stable. Confidence: MEDIUM -- correct but O(n^2) per pass, and "optimize for clarity" suggests something cleaner.
2. **Checkpoint 2 -- Evaluate clarity.** Nested loops with mutation mid-iteration is error-prone and hard to read. Confidence: LOW on this being the right approach.
3. **Action: Pivot.** Switch to sort-then-sweep. Sort intervals by start time, then single pass merging adjacent overlaps. This is O(n log n), canonical, and readable.
4. **Checkpoint 3 -- Implement sort-then-sweep.** Sort by `interval[0]`. Initialize result with first interval. For each subsequent interval, if it overlaps with the last merged interval (start <= last_end), extend; otherwise append. Confidence: HIGH.
5. **Checkpoint 4 -- Edge cases.** Empty list: return []. Single interval: return as-is. Touching intervals [1,2],[2,3]: merged to [1,3] since start <= last_end. Confidence: HIGH.
6. **Halt.**

Output:
```python
def merge_intervals(intervals):
    if not intervals:
        return []
    intervals.sort(key=lambda x: x[0])
    merged = [intervals[0]]
    for start, end in intervals[1:]:
        if start <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], end)
        else:
            merged.append([start, end])
    return merged
```
Refinement trace: 1 pivot from brute-force to sort-then-sweep for clarity. Edge cases verified.

---

**Example 3: CoRefine-Tree for a design decision**

User: "Implement a rate limiter that allows 100 requests per minute per user."

Approach (Tree -- explore 2 strategies):
- **Branch A:** Sliding window counter with Redis-like dict. Simple, exact.
- **Branch B:** Token bucket algorithm. Smooth rate control, standard in production.

**Exploit Branch B** (more robust for production use, handles bursts gracefully):
1. **Checkpoint 1 -- Design.** Each user gets a bucket: capacity=100, refill_rate=100/60 tokens per second, last_check timestamp. Confidence: HIGH.
2. **Checkpoint 2 -- Implement.** On request: compute elapsed time, add tokens (capped at 100), consume 1 token if available. Confidence: HIGH.
3. **Checkpoint 3 -- Thread safety.** If concurrent access matters, note that this needs locking or atomic operations. Confidence: MEDIUM -- mention it but keep implementation single-threaded for clarity.
4. **Action: Re-examine.** Add a note about thread safety rather than over-engineering. Confidence: HIGH.
5. **Halt.**

Output:
```python
import time

class RateLimiter:
    def __init__(self, capacity=100, refill_rate=100/60):
        self.capacity = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.buckets = {}  # user_id -> (tokens, last_check)

    def allow(self, user_id):
        now = time.monotonic()
        tokens, last_check = self.buckets.get(user_id, (self.capacity, now))
        elapsed = now - last_check
        tokens = min(self.capacity, tokens + elapsed * self.refill_rate)
        if tokens >= 1:
            self.buckets[user_id] = (tokens - 1, now)
            return True
        self.buckets[user_id] = (tokens, now)
        return False
```
Refinement trace: CoRefine-Tree explored sliding window vs. token bucket, invested in token bucket. 1 re-examination on thread safety (documented, not over-engineered). 0 pivots.

## Best Practices

- **Do:** Explicitly state your confidence level at each checkpoint. Silent confidence tracking defeats the purpose -- the user and your own reasoning benefit from seeing it.
- **Do:** Re-examine before pivoting. Most errors are local (one bad step), not global (wrong approach entirely). Re-examination is cheaper than a full restart.
- **Do:** When pivoting, name the specific failure. "The recursive approach hits maximum recursion depth on inputs > 1000" is actionable. "This doesn't seem right" is not.
- **Do:** Use concrete test cases as your primary re-examination tool. Running a specific input through your logic is the fastest way to verify or falsify a step.
- **Avoid:** Treating confidence assessment as theater. If you write "Confidence: HIGH" but the step relies on an unverified assumption, you are undermining the entire protocol. Be honest.
- **Avoid:** Pivoting too eagerly. The paper averages 2.7 steps total. If you pivot on every MEDIUM signal, you waste compute exploring too many strategies. Reserve pivots for sustained low confidence or clear logical failures.
- **Avoid:** Exceeding 4 refinement iterations silently. If you cannot reach high confidence after 4 attempts, tell the user what remains uncertain and ask for clarification.

## Error Handling

- **Confidence stays MEDIUM across all approaches:** The problem may be ambiguous or underspecified. Present the best answer with explicit caveats and ask the user to clarify the ambiguous aspect.
- **Pivot leads back to a previously-failed approach:** You are in a loop. Stop, enumerate all approaches tried and why each failed, and present the analysis to the user rather than continuing to cycle.
- **Re-examination contradicts earlier HIGH-confidence steps:** Propagate the fix backward. Do not patch only the current step -- recheck all downstream steps that depended on the now-corrected one.
- **External dependency uncertainty (API behavior, library version):** Confidence assessment only covers your own reasoning. When uncertainty stems from external behavior, state the assumption explicitly and recommend the user verify it.

## Limitations

- **Confidence is self-assessed, not ground-truth.** Claude's confidence signals are heuristic. The paper's 92.6% halt precision means ~7.4% of "confident" answers are still wrong. This protocol reduces but does not eliminate errors.
- **Not a substitute for testing.** For code generation, the refinement loop improves logical correctness but cannot replace running tests. Always recommend the user run the code.
- **Diminishing returns on simple problems.** If the answer is straightforward (simple lookup, well-known algorithm), the overhead of explicit confidence tracking adds verbosity without value. Use this protocol for genuinely challenging problems.
- **No learned controller.** The paper uses a trained Conv1D controller with learned decision boundaries. This skill adapts the same logic heuristically. The decisions are principled but not statistically optimized.
- **Token overhead for the trace.** Explicit confidence reporting adds tokens to the response. For very long chains (20+ steps), consider summarizing the confidence trajectory rather than reporting every checkpoint.

## Reference

[CoRefine: Confidence-Guided Self-Refinement for Adaptive Test-Time Compute](https://arxiv.org/abs/2602.08948v1) -- Jin, Tanno, Diethe, Teare (2026). Read Section 3 for the three-action controller design and Section 4 for CoRefine-Tree's hybrid exploration-exploitation strategy.