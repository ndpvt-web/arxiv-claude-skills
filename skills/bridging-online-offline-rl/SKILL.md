---
name: "bridging-online-offline-rl"
description: "Apply Cobalt-style contextual bandit learning to multi-turn code generation tasks. Decomposes iterative coding into partial trajectory completions, treating each debugging turn as a single-step bandit problem rather than a full RL rollout. Use when: 'help me fix this code iteratively', 'debug this with test feedback', 'multi-turn code repair', 'iterative code generation with execution feedback', 'fix failing test cases step by step', 'recover from wrong code using error output'."
---

# Cobalt: Contextual Bandit Learning for Multi-Turn Code Generation

This skill enables Claude to apply the Cobalt method (Contextual Bandit Learning with Offline Trajectories) for iterative, multi-turn code generation. Instead of treating multi-turn debugging as a monolithic sequence, Cobalt decomposes it into independent single-step completions conditioned on partial execution histories. Each turn receives a code attempt, executes it against tests, feeds back one failing case, and generates a targeted fix -- treating the problem as a contextual bandit where the "context" is the accumulated trajectory of prior attempts and their execution results. This produces more focused, recoverable code repairs than naively re-generating entire solutions.

## When to Use

- When a user provides code that fails some test cases and wants iterative, feedback-driven repair across multiple turns
- When debugging a function where execution output (errors, wrong answers, failed assertions) should guide the next code revision
- When a coding problem requires multiple attempts with incremental improvement based on concrete failure signals
- When the user asks to "fix this code based on the test output" or "debug step by step using these test results"
- When generating a solution to a competitive programming problem that benefits from test-driven iterative refinement
- When recovering from a partially correct solution by focusing on one failing test case at a time

## Key Technique

**One-Step Recoverable MDP Formulation.** Cobalt builds on the insight that multi-turn code generation is "one-step recoverable" -- a suboptimal code attempt at any turn has bounded negative impact because the model can fully recover in the next turn by generating a correct program. Formally, the advantage function A*(s,a) is bounded in [-1, 0] when rewards are in [0,1]. This means the gap between a stepwise-optimal policy (optimizing each turn independently) and the globally optimal multi-turn policy is only O(T), compared to O(T^2) for general offline RL. This justifies decomposing the multi-turn problem into independent single-step bandit problems.

**Partial Trajectory Prompting.** Rather than training on full multi-turn rollouts, Cobalt collects trajectories from a reference model, segments them by turn, and uses each prefix as a contextual prompt. The state at turn t is s_t = (problem, attempt_0, feedback_0, ..., attempt_{t-1}, feedback_{t-1}, observation_t). The model then generates only a single-step completion (one code attempt) conditioned on this context. This decouples trajectory generation from policy optimization, making training more stable and efficient.

**Perturbed Trajectory Augmentation.** Models trained only on correct test feedback learn to blindly trust execution results, enabling "in-context reward hacking" -- following incorrect feedback without questioning it. Cobalt mitigates this by augmenting training data with perturbed test cases (swapping expected outputs between test inputs), forcing the model to reason about whether feedback is trustworthy rather than mechanically following it.

## Step-by-Step Workflow

1. **Parse the coding problem and identify test cases.** Extract the problem statement, function signature, input/output specifications, and all available test cases. Separate them into "public" tests (used for feedback) and "hidden" tests (used for final evaluation) if applicable.

2. **Generate an initial code attempt (Turn 0).** Produce a first-pass solution based solely on the problem description. Include a reasoning trace (chain-of-thought) before the code block explaining the chosen algorithm and its expected complexity.

3. **Execute against test cases and select one failing test.** Run the code against all public tests. If any fail, select exactly ONE failing test case to use as feedback -- this mirrors realistic competitive programming judges that reveal minimal failure information.

4. **Construct the partial trajectory context.** Build the accumulated state: the original problem, each prior code attempt, and each piece of execution feedback. Format this as a structured prompt that makes the history clear and the current failure explicit.

5. **Generate a single-step repair (not a full rewrite).** Conditioned on the partial trajectory, produce a targeted fix. Focus the reasoning trace on WHY the selected test case fails and WHAT specific change addresses it. Avoid rewriting the entire solution unless the algorithm is fundamentally wrong.

6. **Evaluate improvement via pass rate delta.** After each repair, track how many test cases pass compared to the previous turn. A positive delta confirms progress; a zero or negative delta signals the repair was ineffective or introduced regressions.

7. **Apply skepticism to execution feedback (anti-reward-hacking).** Before blindly following a test case failure, verify that the expected output in the failing test is plausible. If the test case seems inconsistent with the problem statement or other passing tests, note this and reason through the expected behavior independently.

8. **Iterate for up to 3 focused turns (training horizon), with graceful extension.** The core Cobalt training uses T=3 turns, but models generalize to T=8 at inference. Stop early if all tests pass. If stuck after 3 turns, re-examine the algorithmic approach rather than making micro-fixes.

9. **On full public test passage, probe for hidden edge cases.** When all visible tests pass, generate additional edge cases (empty input, maximum values, boundary conditions) and reason about whether the solution handles them. This simulates the "re-examine for hidden cases" step from Cobalt inference.

10. **Deliver the final solution with a trajectory summary.** Present the final code along with a brief log of what each turn fixed, the pass rate progression, and any remaining concerns about edge cases.

## Concrete Examples

**Example 1: Iterative Fix of a Sorting Problem**

User: "My function to find the k-th largest element is failing. Here's my code and the test output."

```python
# User's code
def findKthLargest(nums, k):
    nums.sort()
    return nums[k]

# Failing test: findKthLargest([3,2,1,5,6,4], 2) expected 5, got 2
```

Approach:
1. **Turn 0 context:** Problem + user's code + one failing test
2. **Reasoning:** The array is sorted ascending, so `nums[k]` gives the k-th smallest, not k-th largest. Need `nums[-k]` or sort descending.
3. **Single-step repair:**

```python
def findKthLargest(nums, k):
    nums.sort()
    return nums[-k]  # Index from end for k-th largest
```

4. **Evaluate:** Test `[3,2,1,5,6,4], k=2` -> sorted `[1,2,3,4,5,6]`, `nums[-2]` = 5. Pass.
5. **Probe hidden cases:** Empty list (guard needed?), k=1 (largest), k=len(nums) (smallest). All handled by `nums[-k]`.

Output: Fixed solution with trajectory log showing one-turn repair, pass rate 0% -> 100%.

**Example 2: Multi-Turn Competitive Programming Debug**

User: "Solve this: Given a grid, find the shortest path from top-left to bottom-right avoiding obstacles. My BFS solution times out on large inputs."

```
Approach (3-turn trajectory):

Turn 0: Analyze the user's BFS. Identify that it lacks visited-set
  pruning, causing redundant exploration.
  -> Generate fix: add visited set.
  -> Execute: passes 8/10 tests, fails on test with answer 23 (got 25).

Turn 1: Context = [problem, BFS+visited code, failing test grid].
  Reasoning: The BFS explores 4-directional but the problem allows
  diagonal movement (re-read problem statement).
  -> Generate fix: expand neighbors to 8 directions.
  -> Execute: passes 9/10 tests, TLE on largest test.

Turn 2: Context = [problem, 4-dir code, feedback, 8-dir code, TLE].
  Reasoning: Grid is 1000x1000. BFS is O(N*M) which should be fine,
  but the implementation creates new lists for each neighbor.
  -> Generate fix: use deque, pre-compute direction tuples, use
     grid indices instead of coordinate objects.
  -> Execute: passes 10/10 tests.

Output: Final optimized BFS with 8-direction support, trajectory
  showing pass rate progression: 80% -> 90% -> 100%.
```

**Example 3: Skeptical Feedback Handling (Anti-Reward-Hacking)**

User: "My function passes most tests but fails this one: `f(5)` should return `120`. But I think the test might be wrong -- `f` is supposed to compute `2^n`."

```
Approach:
1. Context: User claims f computes 2^n but test expects f(5)=120.
2. Apply Cobalt skepticism: 120 = 5! (factorial), not 2^5 = 32.
   The test case is consistent with factorial, not power-of-two.
3. Re-read the problem statement carefully rather than trusting
   the user's characterization.
4. If the problem says factorial: the test is correct, fix the code.
   If the problem says 2^n: flag the test as potentially wrong
   and present both interpretations.

Output: "The test f(5)=120 is consistent with factorial (5! = 120),
  not 2^n (2^5 = 32). Verify your problem statement. If factorial
  is intended, here's the fix: [code]. If 2^n is intended, the
  test case appears incorrect."
```

## Best Practices

- **Do:** Focus each repair turn on exactly one failing test case. Cobalt's power comes from targeted single-step corrections, not shotgun rewrites.
- **Do:** Preserve working logic between turns. Track which tests already pass and ensure repairs don't introduce regressions (monitor pass rate delta).
- **Do:** Include a reasoning trace before each code attempt explaining why the current failure occurs and what the fix addresses. This mirrors the chain-of-thought pattern that Cobalt models learn.
- **Do:** Limit initial attempts to 3 focused turns before reconsidering the algorithmic approach. Cobalt trains at T=3 and generalizes to longer horizons, but diminishing returns set in around turn 5-6.
- **Avoid:** Rewriting the entire solution from scratch each turn. The partial trajectory context exists to enable incremental, recoverable fixes -- use it.
- **Avoid:** Blindly trusting execution feedback without verifying it against the problem specification. This is the core anti-reward-hacking insight from Cobalt: execution results can be misleading.
- **Avoid:** Providing all failing test cases at once. Present one failing test per turn to keep the repair focused and prevent the model from being overwhelmed by multiple simultaneous failure modes.

## Error Handling

- **All tests pass on first attempt:** Skip iterative repair. Run edge case probing (step 9) and deliver the solution.
- **Pass rate stagnates or drops across turns:** The current algorithm may be fundamentally wrong. After 2 turns with no improvement, step back and reconsider the approach (different data structure, different algorithm class) rather than continuing micro-fixes.
- **Execution timeout (TLE):** This is a complexity issue, not a correctness issue. Focus the repair on algorithmic optimization (better data structures, reduced time complexity) rather than logic fixes.
- **Runtime error (segfault, index out of bounds):** Prioritize boundary condition analysis. Check input constraints and edge cases (empty arrays, single elements, maximum values).
- **Contradictory test feedback:** Apply the anti-reward-hacking protocol. Verify the failing test against the problem statement and other passing tests before modifying code. Flag potential test errors to the user.

## Limitations

- This approach works best for problems with clear, executable test cases. It is less applicable to tasks where correctness is subjective (UI design, prose generation) or where no automated verification exists.
- The single-failing-test feedback model assumes independent test cases. For problems where test cases have complex interdependencies (e.g., stateful APIs, database operations), revealing one failure may not provide enough signal for targeted repair.
- Cobalt's theoretical guarantees rely on one-step recoverability, which holds for code generation but not for all sequential decision problems. Tasks where early mistakes are catastrophic and irreversible (e.g., irreversible state mutations, resource allocation) do not benefit from this decomposition.
- The method assumes access to a code execution environment. In settings where code cannot be executed (e.g., pseudocode review, theoretical algorithm design), the feedback loop cannot operate.
- Performance gains are most pronounced on competitive programming benchmarks (LiveCodeBench, TACO). Transfer to production software engineering tasks (multi-file refactoring, system design) is not established.

## Reference

**Paper:** [Bridging Online and Offline RL: Contextual Bandit Learning for Multi-Turn Code Generation](https://arxiv.org/abs/2602.03806v1) (Chen et al., 2026). Key sections: Section 3 for the one-step recoverable MDP formulation and stepwise objective derivation; Section 4 for the Cobalt training pipeline and perturbed trajectory augmentation; Table 1 for benchmark results showing up to 9.0 point Pass@1 improvements.

**Code:** [github.com/OSU-NLP-Group/cobalt](https://github.com/OSU-NLP-Group/cobalt) (MIT License)