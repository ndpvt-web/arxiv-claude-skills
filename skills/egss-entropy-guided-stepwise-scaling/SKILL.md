---
name: "egss-entropy-guided-stepwise-scaling"
description: >
  Apply Entropy-Guided Stepwise Scaling (EGSS) to complex software engineering tasks like bug fixing,
  code generation, and refactoring. Uses entropy-based uncertainty detection to selectively branch
  exploration at high-uncertainty decision points, then consolidates test suites across trajectories
  and uses multi-model voting to select the best patch. Trigger phrases: "use EGSS to fix this bug",
  "entropy-guided scaling", "stepwise scaling for this task", "try multiple approaches with EGSS",
  "scale test-time compute for this fix", "use adaptive branching to solve this".
---

# Entropy-Guided Stepwise Scaling (EGSS)

EGSS enables Claude to tackle complex software engineering tasks — bug fixing, feature implementation, large refactors — by dynamically allocating computational effort where it matters most. Instead of uniformly generating many candidate solutions (expensive) or relying on a single attempt (unreliable), EGSS measures the **entropy of tool/action choices at each step** to identify high-uncertainty decision points, branches exploration selectively at those points, consolidates debugging signals into a robust test suite across trajectories, and uses structured multi-criteria voting to select the best candidate patch. This yields 5-10% higher resolution rates while using 28%+ fewer tokens than naive ensemble approaches.

## When to Use

- When the user asks to fix a bug that involves ambiguous root causes or multiple plausible fix locations
- When generating code where the implementation strategy is unclear and multiple valid approaches exist
- When a first attempt at a fix passes some tests but fails others, and you need to explore alternatives
- When refactoring complex code where the sequence of changes could go multiple valid directions
- When the user explicitly asks to "try multiple approaches" or "explore different solutions"
- When working on repository-level tasks where a single linear attempt is unlikely to succeed on the first try
- When debugging produces contradictory signals (e.g., a fix seems correct but tests still fail)

## Key Technique

**Entropy as a branching signal.** At each step in a coding task, the agent chooses among actions: edit a file, run a test, search for a symbol, read documentation, etc. The distribution over these choices has an entropy: `H(a_t | s_t) = -sum P(a | s_t) log P(a | s_t)`. Most steps are near-deterministic routine operations (low entropy) — reading the next relevant file, running the obvious test command. But at critical junctures — choosing which file to edit, deciding on a fix strategy, selecting between two plausible root causes — entropy spikes. EGSS exploits the empirical finding that the entropy distribution is **right-skewed**: the vast majority of steps are low-entropy, and only a sparse tail of high-entropy steps represents semantically consequential branching points. By branching only at these high-entropy moments (resampling 4 candidate continuations and scoring them with a judge), EGSS concentrates compute on the decisions that actually matter.

**Test Consolidation Augmentation.** Single-trajectory self-verification is unreliable — studies show ~36% of trajectories that exhibit explicit self-verification still produce incorrect patches ("self-deceptive debugging"). EGSS addresses this by extracting debugging actions and test intents from multiple trajectories, then synthesizing a consolidated test suite that covers functional completeness, boundary robustness, and behavioral consistency. Candidate patches are evaluated against this augmented suite, and only those exceeding a pass-rate threshold survive to the selection phase.

**Multi-Criteria Preference Selection.** Rather than picking the patch that passes the most tests, EGSS employs multiple independent Preference Selectors that evaluate candidates across six dimensions: requirement relevance, code accuracy, change precision, dependency awareness, code quality, and functionality validation. Majority voting across selectors produces the final consensus patch.

## Step-by-Step Workflow

1. **Analyze the task and identify uncertainty.** Read the issue description, relevant source files, and existing tests. Form 2-3 hypotheses about the root cause or implementation strategy. If only one plausible path exists, proceed directly without branching.

2. **Generate an initial trajectory.** Begin working on the most likely hypothesis. At each decision step, assess your confidence: Are you choosing between meaningfully different actions (e.g., editing file A vs. file B), or is the next step obvious?

3. **Detect high-entropy branching points.** When you encounter a step where multiple actions seem roughly equally valid — different fix strategies, different files to modify, different test approaches — this is a high-entropy moment. Flag it explicitly.

4. **Branch at high-entropy points.** At each flagged branching point, generate 2-4 distinct continuation strategies. For each, write a brief rationale (1-2 sentences) explaining the approach. Do NOT branch at routine steps like "read the file I already know I need."

5. **Score and prune branches.** For each candidate continuation, evaluate: Does it address the stated requirements? Is the code change minimal and precise? Does it respect existing dependencies? Prune branches that score poorly, keeping 1-2 strongest candidates to continue exploring.

6. **Develop surviving candidates into complete patches.** For each surviving branch, complete the implementation through to a testable state. Produce a concrete diff for each candidate.

7. **Consolidate test coverage across candidates.** Examine the debugging and testing actions from all branches. Synthesize a combined test suite that covers: (a) the core functional requirement, (b) edge cases that different branches revealed, (c) regression tests for behavior that should not change. Write these tests explicitly.

8. **Evaluate all candidate patches against the consolidated test suite.** Run each patch against the full test suite. Record pass rates. Discard any candidate that fails below the pass-rate threshold (aim for >90% of consolidated tests passing).

9. **Apply multi-criteria selection to surviving candidates.** For each surviving patch, evaluate across six dimensions: requirement relevance (does it address the actual issue?), code accuracy (is the logic correct?), change precision (are changes minimal?), dependency awareness (does it break imports/interfaces?), code quality (is it readable and maintainable?), functionality validation (does it actually work end-to-end?). Select the candidate with the strongest aggregate profile.

10. **Present the selected patch with rationale.** Show the user the chosen solution, explain why it was selected over alternatives, and report which tests pass. If multiple candidates are close in quality, present the top 2 with tradeoffs.

## Concrete Examples

**Example 1: Ambiguous bug fix with multiple plausible root causes**

User: "The `/api/users` endpoint returns 500 when the `email` field contains unicode characters. Fix it."

Approach:
1. Read the route handler for `/api/users` and trace the email field through validation, database insertion, and response serialization.
2. Identify three plausible root causes: (a) the input validator rejects unicode, (b) the database column has an ASCII-only collation, (c) the JSON serializer chokes on certain codepoints.
3. **High-entropy branching point detected**: all three are plausible. Branch into three candidate fixes.
4. Branch A: Update the validator regex to allow unicode letter categories (`\p{L}` instead of `[a-zA-Z]`).
   Branch B: Alter the database column collation to `utf8mb4_unicode_ci`.
   Branch C: Add explicit UTF-8 encoding in the serialization layer.
5. Write a consolidated test suite covering: basic unicode email (`user@example.com` with accented name), emoji in local part, CJK characters, mixed ASCII/unicode, and existing ASCII-only emails (regression).
6. Run all three patches against the consolidated suite. Branch A passes 5/6 (fails on emoji). Branch B passes 3/6 (doesn't fix the validator). Branch C passes 2/6 (root cause is earlier in pipeline).
7. Refine Branch A to also handle emoji codepoints. Re-run: 6/6 pass.
8. Select Branch A (refined). Report to user with the diff and test results.

Output:
```
Fix: Updated email validator in src/validators/user.ts to use Unicode-aware
regex pattern \p{L} instead of [a-zA-Z], and added \p{Emoji} support.

Tests added (all passing):
  - unicode_email_accented_chars ........... PASS
  - unicode_email_emoji_local_part ......... PASS
  - unicode_email_cjk_characters ........... PASS
  - unicode_email_mixed_ascii_unicode ...... PASS
  - regression_ascii_only_email ............ PASS
  - regression_existing_user_lookup ........ PASS

Alternative considered: Database collation change (Branch B) — rejected because
the failure occurs at the validation layer before the DB is reached.
```

**Example 2: Feature implementation with unclear design direction**

User: "Add rate limiting to our Express API. We're getting hammered by bots."

Approach:
1. Examine the existing middleware stack, routing structure, and any existing throttling.
2. **High-entropy branching point**: multiple valid rate-limiting strategies exist. Branch into two approaches:
   Branch A: Per-IP sliding window using Redis (robust, distributed, but requires Redis dependency).
   Branch B: In-memory token bucket per IP (simple, no new dependencies, but per-process only).
3. For each branch, implement the middleware, add configuration for rate/window, and wire into the Express app.
4. Consolidated test suite: (a) requests under limit succeed, (b) requests over limit return 429, (c) different IPs have independent limits, (d) limits reset after window expires, (e) existing routes still work (regression).
5. Both branches pass all tests. Apply multi-criteria selection:
   - Branch A scores higher on functionality (distributed) but lower on change precision (new dependency).
   - Branch B scores higher on change precision (self-contained) but lower on scalability.
6. Present both to user with tradeoffs, recommend based on their deployment context.

Output:
```
Two viable rate-limiting implementations ready:

Option A (Recommended for multi-instance deployments):
  - Redis-backed sliding window in src/middleware/rateLimit.ts
  - Requires: redis package, REDIS_URL env variable
  - Survives process restarts, works across load balancer

Option B (Recommended for single-instance or quick deploy):
  - In-memory token bucket in src/middleware/rateLimit.ts
  - Zero new dependencies
  - Rate limits reset on process restart

Both pass all 5 rate-limiting tests. Choose based on your infrastructure.
```

**Example 3: Refactoring with cascading uncertainty**

User: "Refactor the `OrderProcessor` class — it's 800 lines and does too much."

Approach:
1. Read `OrderProcessor` and map its responsibilities: validation, pricing, inventory, payment, notification.
2. **High-entropy branching point**: extraction order matters. Branch:
   Branch A: Extract validation first (fewest dependencies, clean boundary).
   Branch B: Extract payment first (highest-risk code, most benefit from isolation).
3. Pursue both branches through 2 extraction steps each. Write integration tests covering the existing `processOrder()` end-to-end flow.
4. Branch A produces cleaner intermediate states (each step compiles and tests pass). Branch B requires temporary adapter code mid-refactor.
5. Select Branch A. Continue extracting remaining responsibilities in dependency order.
6. Final consolidated test suite verifies all original behavior preserved.

## Best Practices

- **Do:** Explicitly articulate your branching points and why each branch is worth exploring. This makes the process transparent and auditable.
- **Do:** Keep branches genuinely distinct — different root causes, different algorithms, different architectural choices. Superficial variations (variable naming, minor reordering) waste compute.
- **Do:** Write the consolidated test suite *before* evaluating candidates. This prevents confirmation bias toward whichever candidate you explored first.
- **Do:** Prune aggressively after scoring. Carrying forward more than 2-3 candidates past the initial evaluation is rarely worth the cost.
- **Avoid:** Branching at every step. The power of EGSS comes from being selective. If you're branching more than 2-3 times in a task, you're over-branching.
- **Avoid:** Relying solely on test pass rates for selection. The multi-criteria evaluation (requirement relevance, change precision, dependency awareness) catches cases where a passing-but-wrong patch slips through.

## Error Handling

- **All branches fail tests:** Step back and re-examine the problem statement. The hypotheses may all be wrong. Generate a new diagnostic trajectory focused on reproducing the issue before attempting fixes.
- **Consolidated tests are flaky:** Isolate non-deterministic tests (timing-dependent, order-dependent) and either stabilize them or exclude them from the pass-rate calculation. Flag flaky tests in the output.
- **Branches converge to the same solution:** This is a signal that entropy was low — the task didn't actually need branching. Proceed with the single solution and note that alternatives were explored and converged.
- **Judge scores are tied between candidates:** Present both to the user with a clear comparison across the six evaluation dimensions. Let the user's domain knowledge break the tie.

## Limitations

- **Not useful for straightforward tasks.** If the fix is obvious (typo, missing import, off-by-one), branching adds overhead with no benefit. Use EGSS only when genuine uncertainty exists.
- **Requires testable code.** The test consolidation phase depends on being able to write and run tests. Purely UI-oriented or infrastructure-as-code tasks with no test harness get limited benefit.
- **Token budget.** Even with selective branching, EGSS uses more tokens than a single-pass attempt. For simple tasks, the overhead is not justified.
- **Single-model limitation.** The paper's multi-model voting (using different LLMs as independent selectors) cannot be replicated by a single Claude instance. The multi-criteria evaluation approximates this but lacks true model diversity.
- **Does not replace domain expertise.** EGSS helps explore the solution space more systematically, but if the correct fix requires domain knowledge the model lacks, branching won't surface it.

## Reference

**Paper:** [EGSS: Entropy-guided Stepwise Scaling for Reliable Software Engineering](https://arxiv.org/abs/2602.05242v1) — Mao et al., 2026. Look for: Algorithm 1 (Test Consolidation Augmentation), the tool entropy formula (Equation 1), the trajectory scoring function (Equation 2), and the empirical analysis of entropy distribution showing right-skewed concentration in the low-entropy regime.