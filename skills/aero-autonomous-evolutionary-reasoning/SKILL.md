---
name: "aero-autonomous-evolutionary-reasoning"
description: "Apply the AERO dual-loop self-evolution framework to iteratively improve reasoning on complex tasks. Uses entropy-based difficulty calibration, counterfactual verification, and staggered role refinement to solve hard problems without external oracles. Triggers: 'reason through this step by step with self-correction', 'solve this hard problem autonomously', 'verify your reasoning with counterfactuals', 'self-critique and improve your answer', 'use dual-loop reasoning on this', 'iteratively refine your solution'"
---

# AERO: Autonomous Evolutionary Reasoning Optimization

This skill enables Claude to apply the AERO dual-loop reasoning framework from Gao et al. (2026) to complex coding, math, and logic tasks. Rather than producing a single-shot answer, Claude decomposes its reasoning into three functional roles — **Questioner** (problem decomposition), **Solver** (multi-path solution generation), and **Critic** (counterfactual verification) — cycling through an inner synthesis loop and an outer refinement loop. The key insight is entropy-based difficulty calibration: focus effort on sub-problems at the boundary of your capability (the "Zone of Proximal Development"), skip what's trivially solved, and flag what's genuinely intractable.

## When to Use

- When the user asks Claude to solve a multi-step algorithmic or mathematical problem and wants verified correctness, not just a first attempt
- When a coding task has multiple valid approaches and the user wants Claude to evaluate trade-offs through structured self-critique
- When the user explicitly asks for self-correcting or iteratively refined reasoning ("check your own work", "verify with counterfactuals")
- When debugging complex logic where the root cause is unclear and systematic hypothesis elimination is needed
- When generating code that must handle edge cases, and the user wants Claude to adversarially test its own solution
- When a problem feels "at the boundary" of Claude's capability — hard enough to warrant multiple reasoning attempts but not completely intractable

## Key Technique

**Dual-Loop Self-Evolution.** AERO structures reasoning as two nested loops. The **inner loop** is an experience synthesis cycle: (1) decompose the problem into sub-tasks, (2) generate multiple independent solution trajectories for each sub-task, (3) measure response entropy across trajectories to calibrate difficulty, and (4) apply counterfactual correction to verify correctness. The **outer loop** aggregates verified insights to refine the overall approach, feeding improved understanding back into the next inner-loop iteration.

**Entropy-Based ZPD Positioning.** For each sub-problem, AERO generates N independent solution attempts and clusters the answers. Normalized Shannon entropy across clusters tells you where the problem sits: low entropy (all attempts agree) means the problem is mastered — move on. High entropy (answers are scattered) means the problem is too hard or ill-defined — flag it. Medium entropy (some agreement, some divergence) identifies the Zone of Proximal Development — this is where focused reasoning effort pays off. The formula is H̄ = -1/log₂(n) * Σ P(cⱼ)log₂P(cⱼ), normalized to [0,1].

**Independent Counterfactual Correction (ICC).** Instead of simply checking whether an answer "looks right," ICC forces the Critic role to re-solve the problem under the counterfactual assumption that the proposed answer is wrong. If this independent re-derivation converges to the same answer, confidence is high. If it diverges, the original answer likely contains an error. This breaks confirmation bias — the Critic can't just rubber-stamp the Solver's work because it's required to reason from an adversarial starting point.

## Step-by-Step Workflow

1. **Decompose the problem into sub-tasks.** Parse the user's request into discrete reasoning units. For a coding problem, this means: understanding the spec, designing the algorithm, handling edge cases, implementing, and verifying. Label each sub-task explicitly.

2. **Generate multiple solution trajectories per sub-task.** For each non-trivial sub-task, produce 2-4 independent reasoning paths. Vary your approach: try brute force vs. optimized, recursive vs. iterative, different data structures. Keep each trajectory self-contained.

3. **Cluster answers and compute entropy.** Group the trajectories by their conclusions. If all paths agree (low entropy, H̄ < 0.2), accept the consensus and move on. If paths mostly agree with one outlier (medium entropy, 0.2 ≤ H̄ ≤ 0.7), this sub-task is in the ZPD — focus effort here. If paths wildly disagree (high entropy, H̄ > 0.7), the sub-task may need reframing or the user's input may be ambiguous.

4. **Apply counterfactual correction to ZPD sub-tasks.** For each sub-task in the ZPD zone, take the top candidate answer and assume it is wrong. Re-derive the answer from scratch under this adversarial assumption. If the re-derivation converges to the same result, mark it as verified. If it diverges, investigate the discrepancy — this is where bugs hide.

5. **Construct a verified solution from confirmed sub-task outputs.** Assemble the full solution using only verified sub-task results. For unverified sub-tasks, explicitly note uncertainty and present the competing alternatives to the user.

6. **Run the outer refinement loop.** Review the assembled solution holistically. Check for cross-sub-task inconsistencies (e.g., a variable renamed in one part but not another, an edge case handled in the algorithm but not the implementation). Feed any issues back into a second inner loop pass.

7. **Apply staggered role emphasis.** On the first pass, prioritize problem decomposition quality (Questioner role). On subsequent passes, shift emphasis to solution correctness (Solver) and verification rigor (Critic). This prevents the common failure mode where you keep refining answers to questions that were poorly framed to begin with.

8. **Present the final answer with a confidence map.** For each sub-task, indicate whether it was trivially solved (mastered zone), verified through ICC (ZPD zone), or remains uncertain (chaos zone). Give the user actionable information about where the solution is robust and where it needs human review.

## Concrete Examples

**Example 1: Debugging a Concurrency Bug**

User: "My Go worker pool deadlocks intermittently. Here's the code. Find and fix the bug."

Approach:
1. **Decompose**: Identify sub-tasks — channel buffer sizing, goroutine lifecycle, lock ordering, select statement coverage, shutdown signaling
2. **Multi-path analysis on channel logic** (ZPD zone — 2 of 3 trajectories flag the unbuffered result channel as problematic, 1 does not):
   - Trajectory A: "The result channel blocks when all workers are busy sending and no consumer is reading"
   - Trajectory B: "The done channel close races with the last result send"
   - Trajectory C: "Channel sizing is fine; the issue is in lock ordering"
3. **ICC on top candidate**: Assume Trajectory A is wrong. Re-derive: if the result channel is buffered to pool size, can deadlock still occur? Yes — if workers > buffer, the same blocking happens. This *confirms* the channel is undersized but *refines* the fix: buffer must equal total tasks, not pool size
4. **Verify lock ordering** (low entropy — all trajectories agree no lock issue): Accept and move on
5. **Final fix**: Buffer the result channel to task count, add a `context.WithCancel` for clean shutdown, present both changes with confidence map

Output:
```go
// VERIFIED (ICC-confirmed): Buffer result channel to task count, not pool size
results := make(chan Result, len(tasks)) // was: make(chan Result)

// VERIFIED (consensus): Add context-based shutdown
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
```

**Example 2: Optimizing a SQL Query**

User: "This query takes 12 seconds on 2M rows. Make it fast."

Approach:
1. **Decompose**: Parse query structure, identify join patterns, check index usage, evaluate subquery vs. CTE vs. materialized approach, assess partition pruning
2. **Multi-path on join rewrite** (ZPD — trajectories split between hash join hint and restructuring as lateral join):
   - Trajectory A: Rewrite correlated subquery as a lateral join — estimated 3x speedup
   - Trajectory B: Add composite index on (user_id, created_at) — estimated 5x speedup
   - Trajectory C: Materialize the subquery as a CTE — estimated 2x speedup
3. **ICC on index approach**: Assume composite index won't help. Re-analyze: the WHERE clause filters on status before joining, so the index would only help if status selectivity is high. Check the query — status = 'active' likely covers 80% of rows, making the index less effective than expected. ICC *rejects* pure index approach.
4. **ICC on lateral join**: Assume lateral join won't help. Re-analyze: the optimizer already converts correlated subqueries to lateral joins in PostgreSQL 14+. Check if user specified PG version — they didn't. Flag as version-dependent.
5. **Final output**: Recommend the CTE materialization (verified, version-independent) + composite index (helpful but not sufficient alone), flag lateral join as contingent on PG version

Output:
```sql
-- VERIFIED: Materialize the user-date aggregation
WITH user_latest AS MATERIALIZED (
  SELECT user_id, MAX(created_at) AS latest
  FROM events
  WHERE status = 'active'
  GROUP BY user_id
)
SELECT u.*, e.*
FROM users u
JOIN user_latest ul ON u.id = ul.user_id
JOIN events e ON e.user_id = ul.user_id AND e.created_at = ul.latest;

-- RECOMMENDED (additive): Composite index
CREATE INDEX CONCURRENTLY idx_events_user_created
ON events (user_id, created_at) WHERE status = 'active';
```

**Example 3: Implementing a Complex Algorithm**

User: "Implement an interval scheduling algorithm that handles weighted jobs with deadlines and dependencies."

Approach:
1. **Decompose**: Topological sort for dependencies, dynamic programming for weighted interval scheduling, deadline feasibility check, output formatting
2. **Topological sort** (low entropy — all trajectories use Kahn's algorithm): Accept consensus
3. **DP formulation** (ZPD — trajectories split on state representation):
   - Trajectory A: dp[i] = max profit considering first i jobs sorted by end time
   - Trajectory B: dp[i][j] = max profit with i jobs using j time slots (pseudo-polynomial)
   - Trajectory C: dp[i] = max profit at time i with dependency constraints via topological layers
4. **ICC on Trajectory C**: Assume layered DP is wrong. Re-derive: dependencies impose ordering constraints that interact with interval overlaps. A pure sort-by-end-time approach (Trajectory A) ignores dependency chains. Trajectory C correctly handles both. ICC confirms.
5. **Deadline feasibility** (low entropy): All trajectories agree — filter infeasible jobs before DP
6. **Final implementation**: Topological sort, then layered DP with binary search for compatible intervals within each dependency layer

Output:
```
Confidence Map:
  [MASTERED]  Topological sort (Kahn's) — all trajectories agreed
  [VERIFIED]  DP formulation (layered with dependency constraints) — ICC confirmed
  [MASTERED]  Deadline filtering (pre-DP feasibility check) — all trajectories agreed
  [VERIFIED]  Binary search for compatible intervals — ICC confirmed

Algorithm: O(n log n) time, O(n) space after topological preprocessing
```

## Best Practices

- **Do:** Generate genuinely independent solution trajectories — vary your algorithm choice, data structure, or reasoning approach. Rewording the same logic doesn't count as a separate trajectory.
- **Do:** Spend disproportionate effort on ZPD sub-tasks (medium entropy). These are where errors cluster and where verification has the highest ROI.
- **Do:** When applying ICC, fully commit to the counterfactual. Reason as though you've never seen the original answer. Half-hearted re-derivation defeats the purpose.
- **Do:** Present your confidence map to the user. They need to know which parts are verified, which are consensus-but-unverified, and which remain uncertain.
- **Avoid:** Running the full dual-loop on trivial sub-tasks. If all trajectories agree immediately, accept and move on — over-verification wastes reasoning budget.
- **Do:** On the first outer-loop pass, validate your decomposition before investing in solution quality. A well-structured decomposition makes everything downstream easier; a flawed one compounds errors.
- **Avoid:** Treating ICC as a rubber stamp. If counterfactual re-derivation takes the same reasoning path as the original, you haven't actually applied it — you need to genuinely approach from a different angle.
- **Avoid:** More than 3 outer-loop iterations. If the solution isn't converging after 3 passes, the problem likely needs reframing or additional user input, not more self-critique.

## Error Handling

- **All trajectories disagree (chaos zone):** The problem is likely ambiguous or beyond current capability. Ask the user for clarification on requirements, constraints, or expected output format rather than guessing.
- **ICC produces a third, different answer:** You now have three competing solutions. Do not recurse — instead, present all three to the user with the reasoning chain for each and let them decide. Flag this as low-confidence output.
- **Outer loop doesn't converge:** After 2-3 iterations, if the solution keeps changing, identify which sub-task is unstable and isolate it. Present the stable parts as verified and the unstable part as an open question.
- **Decomposition was wrong:** If during the inner loop you discover that sub-tasks were poorly framed (e.g., what you thought was one problem is actually two coupled problems), restart decomposition. Don't try to patch a bad decomposition with more inner-loop iterations.

## Limitations

- This framework adds reasoning overhead. For simple, well-defined tasks (CRUD operations, straightforward refactors, simple bug fixes), single-shot reasoning is faster and equally correct. Don't use AERO when a direct answer suffices.
- The entropy calibration relies on generating meaningfully different trajectories. For problems with only one reasonable approach, the dual-loop degenerates into redundant computation.
- ICC cannot catch errors that stem from fundamental knowledge gaps (as opposed to reasoning errors). If Claude doesn't know a language feature or API behavior, counterfactual re-derivation will reproduce the same mistake.
- The staggered role emphasis is a heuristic. For some problems, the Critic role matters most on the first pass (e.g., validating that the problem statement is consistent before decomposing it).
- This technique is designed for single-turn deep reasoning. It does not replace multi-turn dialogue with the user for requirements gathering.

## Reference

Gao, Z., Ma, J., Li, X., Li, P., & Qu, N. (2026). *AERO: Autonomous Evolutionary Reasoning Optimization via Endogenous Dual-Loop Feedback.* arXiv:2602.03084v2. https://arxiv.org/abs/2602.03084v2

Key sections to read: Section 3.2 (entropy-based ZPD positioning and the normalized entropy formula), Section 3.3 (Independent Counterfactual Correction mechanism), and Section 3.4 (Staggered Training Strategy for role synchronization). Code: https://github.com/mira-ai-lab/AERO