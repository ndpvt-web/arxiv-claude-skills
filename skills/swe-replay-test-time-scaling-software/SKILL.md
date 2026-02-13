---
name: "swe-replay-test-time-scaling-software"
description: "Efficient test-time scaling for software engineering agents using trajectory recycling and explore-exploit branching (SWE-Replay). Use when: 'scale up my agent to solve this hard bug', 'retry this task more efficiently', 'run multiple attempts on this SWE task', 'replay and branch from a previous attempt', 'use test-time compute to improve solve rate', 'efficiently sample multiple solutions for this issue'."
---

# SWE-Replay: Efficient Test-Time Scaling for Software Engineering Agents

This skill enables Claude to apply the SWE-Replay technique for efficiently scaling test-time compute when solving software engineering tasks across multiple attempts. Instead of running each attempt from scratch (naive scaling), SWE-Replay recycles trajectories from prior trials, dynamically choosing to **explore** fresh solutions or **exploit** archived experience by branching at critical intermediate steps. This reduces compute cost by up to 17.4% while maintaining or improving solve rates by up to 3.8%, without requiring any external value model or reward signal.

## When to Use

- When a user wants to run multiple attempts at solving a bug or issue and pick the best patch (test-time scaling)
- When a previous agent trajectory partially succeeded (found the right files, identified the bug) but the final fix was wrong
- When budget-constrained: the user needs maximum solve rate per dollar of LLM inference
- When working on SWE-bench style tasks where regression test filtering and majority voting apply
- When the user asks to "retry smarter" rather than "retry from scratch" on a failed coding task
- When orchestrating multiple parallel agent runs and wanting to share intermediate discoveries across them

## Key Technique

**The core insight**: Not all steps in a coding agent trajectory are equally valuable. Repository exploration steps (finding relevant files, reading code, understanding structure) are expensive but highly reusable. The actual fix attempt at the end is where divergence matters. SWE-Replay exploits this by archiving full trajectories and, on subsequent attempts, branching from a carefully selected intermediate step rather than starting over.

**Explore-Exploit Selection**: Each new attempt flips a fair coin (p=0.5). On "explore," the agent starts fresh with no prior context. On "exploit," the system selects a critical step from the archive, restores the environment to that state, and lets the agent diverge from there. This stochastic balance avoids overfitting to any single trajectory while amortizing the cost of repository exploration across attempts.

**Critical Step Identification** uses a four-stage pipeline without any external LLM judge: (1) Filter out trajectories whose patches fail regression tests, (2) Abstract each step into "set of repository files explored so far" rather than raw output, (3) Rank steps by reasoning intensity (paragraph count in the agent's thinking, not raw token length), (4) Apply softmax-normalized sampling that favors under-explored states: `p_i = softmax(1/v_i)` where `v_i` is the number of concrete steps reaching abstract state `i`. This naturally biases toward branching at points where the agent had just discovered something new but few other trajectories passed through.

## Step-by-Step Workflow

1. **Run the first attempt from scratch.** Execute the agent on the task with no prior context. Record the full trajectory: every action, observation, reasoning block, and the set of repository files touched at each step. Store the final patch.

2. **Archive the trajectory.** Save the trajectory with per-step metadata: (a) the abstract state (set of files explored up to that point), (b) the reasoning content of the step, (c) a snapshot of repository mutations (git diff) and whether non-repo state was modified.

3. **For each subsequent attempt, flip the explore-exploit coin (p=0.5).**
   - **Explore**: Run the agent from scratch with a fresh environment. Archive the resulting trajectory.
   - **Exploit**: Proceed to step 4.

4. **Filter archived trajectories.** Discard any trajectory whose final patch causes regression test failures. Only branch from trajectories that produced at least a plausible (non-regressing) patch.

5. **Select a critical branching step.** From the remaining archived trajectories:
   - Group steps by abstract state (files explored so far).
   - Compute sampling weight for each state: `p_i = exp(1/v_i) / sum(exp(1/v_j))` where `v_i` = number of steps mapped to state `i`.
   - Sample a state, then within that state, prefer steps with higher reasoning intensity (more paragraphs of thinking).

6. **Restore the environment to the selected step.** Check if the trajectory mutated non-repo state (e.g., installed packages, modified system files). If not, apply the stored git diff to fast-forward the repo. If yes, replay the actions up to that step.

7. **Provide replay context to the agent.** Feed the prefix of the archived trajectory (steps 1 through the branch point) as conversation history, then let the agent generate a new replacement step and continue from there.

8. **Execute the new suffix.** The agent runs from the branch point onward, producing a new trajectory suffix and a new patch. Archive the full hybrid trajectory (old prefix + new suffix).

9. **After all attempts (budget exhausted), select the final patch.** Run regression tests on each candidate patch. Filter out patches that cause test failures. Apply majority voting among the remaining patches to select the final answer.

10. **Report results.** Present the selected patch along with the cost breakdown: how many attempts explored vs. exploited, the total token usage, and the cost savings compared to naive all-from-scratch scaling.

## Concrete Examples

**Example 1: Branching from a successful file discovery**

```
User: I've been trying to fix issue #1234 (TypeError in the serializer).
      My last attempt found the right file but the fix was wrong. Try again smarter.

Approach:
1. Archive the previous attempt's trajectory. Note that steps 1-15 were
   repository exploration (found src/serializers/json.py at step 12) and
   steps 16-20 were the fix attempt.
2. Coin flip: EXPLOIT. Select step 12 as branch point (high reasoning
   intensity at the file discovery, low visit count for that abstract state).
3. Restore repo to the state at step 12 (git diff applied).
4. Replay steps 1-12 as conversation context. Agent now diverges: instead
   of the previous fix, it reads additional context from the test file,
   identifies the actual root cause (missing type coercion), and produces
   a different patch.
5. New patch passes regression tests. Archive this trajectory.
6. After 5 total attempts (2 explore, 3 exploit), majority vote selects
   the correct patch appearing in 3 of 4 non-regressing trajectories.

Output:
- Patch: Add `str()` coercion in `src/serializers/json.py:142`
- Cost: $1.52 (vs $1.84 naive scaling = 17.4% savings)
- Attempts: 5 total, 3 exploits branched from archived exploration
```

**Example 2: Explore-heavy early, exploit-heavy late**

```
User: Solve this SWE-bench task. Use 10 attempts and pick the best patch.

Approach:
1. Attempt 1: Always explore from scratch. Agent explores the repo,
   finds 3 candidate files, attempts fix in file A. Patch fails tests.
2. Attempt 2: Coin=EXPLORE. Fresh start. Agent finds files A and B,
   tries fix in B. Patch passes regression tests. Archived.
3. Attempt 3: Coin=EXPLOIT. Branch from attempt 2, step 8 (just after
   finding file B). Agent reads more of file B, produces variant patch.
4. Attempts 4-10: Mix of explore/exploit. Exploit attempts branch from
   the growing archive, each time diverging at high-reasoning-intensity
   steps near file discoveries.
5. Final selection: 7 patches pass regression tests. Majority vote
   selects the patch produced by attempts 2, 3, and 7 (identical diff).

Output:
- Resolved: Yes
- Final patch: Modify `utils/parser.py` lines 88-92
- Cost savings: 13.2% vs naive 10x from-scratch sampling
- Key insight: 6 of 8 exploit attempts reused the exploration prefix
  from attempt 2, saving ~40% of tokens per run on repo navigation
```

**Example 3: When exploit doesn't help (all trajectories regress)**

```
User: This bug is tricky. My 3 attempts all introduced test failures.

Approach:
1. All 3 archived trajectories fail regression test filtering.
   No valid trajectories to branch from.
2. Coin flip is irrelevant: force EXPLORE mode since the archive
   has no non-regressing trajectories to exploit.
3. Run attempt 4 from scratch with fresh exploration strategy.
4. If attempt 4 produces a non-regressing patch, future exploit
   attempts can now branch from it.

Output:
- Status: Archive empty after filtering. Falling back to pure exploration.
- Recommendation: The regression failures suggest the agent is modifying
  the wrong code path. Consider narrowing the search scope.
```

## Best Practices

- **Do:** Always run the first attempt from scratch. The archive needs at least one trajectory before exploitation is possible.
- **Do:** Filter trajectories by regression test results before branching. Branching from a trajectory that broke tests propagates the mistake.
- **Do:** Use file-level state abstraction (set of files explored) rather than raw environment output for step grouping. This is robust across different agent scaffolds and avoids sensitivity to output formatting.
- **Do:** Favor branching at steps with high reasoning intensity (many paragraphs of thinking) -- these are decision points where the agent committed to a direction, and diverging there maximizes the chance of finding an alternative path.
- **Avoid:** Deterministic exploit selection (always picking the "best" step). Use softmax-normalized stochastic sampling to maintain diversity across attempts.
- **Avoid:** Using an external LLM judge to score trajectories for branching decisions. This adds cost and introduces calibration errors that negate the efficiency gains.
- **Avoid:** Storing full environment snapshots for every step. Check whether non-repo state was mutated; if not, a git diff is sufficient and far cheaper to restore.

## Error Handling

- **All patches fail regression tests**: The archive becomes empty after filtering. Force all subsequent attempts into explore mode until at least one non-regressing trajectory is archived. Alert the user that the agent may be modifying the wrong area.
- **Non-repo state mutations detected**: If a trajectory step installed packages, modified configs outside the repo, or ran destructive commands, you cannot fast-restore via git diff alone. Fall back to replaying actions from the beginning up to the branch point, which is slower but correct.
- **Trajectory archive grows too large**: Cap the archive at the most recent N non-regressing trajectories (e.g., N=10). Older trajectories with lower diversity scores can be evicted.
- **Branching produces identical patches**: If exploit attempts keep converging to the same patch, bias the coin toward explore (e.g., increase explore probability to 0.7) to inject more diversity.
- **Prompt cache misses on replay context**: When providing the trajectory prefix as conversation history, structure it so shared prefixes align across attempts. This enables prompt caching to reduce token costs on the replayed portion.

## Limitations

- **Requires multiple attempts**: SWE-Replay is a test-time scaling technique. It provides no benefit for single-attempt scenarios. The minimum useful budget is 3-5 attempts.
- **Regression test dependency**: The trajectory filtering step requires runnable regression tests. For repos without test suites, you lose the primary quality signal for archive filtering.
- **Stochastic, not optimal**: The 50/50 explore-exploit split and softmax sampling are simple and robust but not provably optimal. Task-specific tuning of `p` could yield better results but risks overfitting.
- **Environment restoration cost**: For tasks where agents frequently mutate non-repo state (install dependencies, modify system configs), the fast git-diff restoration doesn't apply, reducing cost savings.
- **No value signal**: By design, SWE-Replay avoids LLM-based quality estimates. This makes it generalizable but means it cannot preferentially branch from "almost correct" trajectories -- only from non-regressing ones.

## Reference

- **Paper**: [SWE-Replay: Efficient Test-Time Scaling for Software Engineering Agents](https://arxiv.org/abs/2601.22129v2) (Ding & Zhang, 2026)
- **Key sections to read**: Section 2.1 (Critical Step Identification pipeline), Section 2.2 (Environment Restoration), Algorithm 1 in Appendix A (full pseudocode), Table 1 (cost-performance tradeoffs across models and benchmarks).