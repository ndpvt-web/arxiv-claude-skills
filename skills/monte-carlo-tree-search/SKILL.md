---
name: "monte-carlo-tree-search"
description: >
  Apply Monte Carlo Tree Search (MCTS) to systematically explore and evaluate
  multiple fix candidates when debugging complex bugs or resolving GitHub issues.
  Combines hierarchical fault localization, tree-structured patch exploration,
  and execution feedback to find correct repairs. Use when: "fix this failing test",
  "debug this issue across multiple files", "find and fix the root cause",
  "resolve this GitHub issue", "repair this broken function", "MCTS debug".
---

# Monte Carlo Tree Search for Execution-Guided Program Repair

This skill enables Claude to apply a structured tree-search strategy when diagnosing and repairing bugs, especially those spanning multiple files in a repository. Instead of generating a single fix attempt and hoping it works, Claude systematically explores multiple patch trajectories using MCTS principles: selecting the most promising repair direction, expanding it with candidate edits, simulating the result via test execution, and backpropagating success signals to guide future exploration. This approach is based on the CodePilot framework (Liang, 2026), which demonstrated a 24.67% resolution rate on SWE-bench Lite by combining symbolic search with LLM-guided generation.

## When to Use

- When a user asks to fix a bug that could have multiple root causes or multiple valid fixes
- When a failing test suite suggests the issue spans several files or functions
- When a GitHub issue requires understanding interactions across repository modules
- When a first fix attempt fails and the user asks to try alternative approaches
- When the user asks to "find and fix the root cause" of a regression
- When debugging requires narrowing down from a large codebase to the exact faulty lines
- When the user explicitly requests a systematic or exhaustive approach to repair

## Key Technique

**Hierarchical Fault Localization.** Rather than scanning the entire repository, narrow the search in three stages: (1) rank files by relevance to the bug description using a combination of semantic similarity and keyword matching, (2) parse the top candidate files into ASTs to extract class and function definitions, and (3) identify specific line ranges and modification types (edit, insert, delete) that are most likely to contain the fault. This dramatically reduces the search space before any fix is attempted.

**MCTS-Guided Patch Exploration.** Model the repair process as a tree where each node represents a partial patch state and each edge represents a code edit action. Use UCT (Upper Confidence bounds applied to Trees) to balance exploitation of promising fix directions against exploration of novel alternatives. At each node, expand by generating K candidate continuations (e.g., 3 alternative edits), simulate each by completing the patch and running tests, then backpropagate the execution result as a reward signal. The reward is graded: full test pass = 1.0, partial pass = proportional credit, syntactically valid but failing = 0.2, broken patch = 0.0. This graded feedback prevents the search from treating all failures equally.

**Confidence-Calibrated Refinement.** After generating a candidate patch, assess confidence at the span level by computing the mean token-level entropy of the generated code. Low-confidence spans (high entropy) indicate uncertain edits that should be selectively regenerated or refined, while high-confidence spans are kept. This avoids wasting effort re-generating parts of the fix that are already correct.

## Step-by-Step Workflow

1. **Reproduce the bug.** Run the failing test or trigger the reported behavior to confirm the issue exists. Capture the exact error messages, stack traces, and failing test names as the ground truth for reward evaluation.

2. **Localize at the file level.** Score each repository file against the bug description. Prioritize files mentioned in stack traces, error messages, or the issue text. Select the top 5-10 candidate files for deeper inspection.

3. **Localize at the function level.** Parse each candidate file's AST to extract function and class definitions. Rank functions by relevance to the bug (call graph proximity to the error site, semantic similarity to the issue description). Identify 2-5 specific code regions as repair targets, each annotated with the likely modification type (edit existing logic, insert missing check, delete erroneous code).

4. **Initialize the search tree.** Create a root node representing the current (buggy) state of the code. Each child node will represent a distinct repair hypothesis applied to one of the localized regions.

5. **Select the next node to expand (UCT).** Among unexplored or partially explored nodes, choose the one that maximizes the UCT formula: balance the node's current average reward (exploitation) against how under-explored it is relative to siblings (exploration). On the first iteration, pick the highest-priority localized region.

6. **Expand with K candidate edits.** For the selected node, generate K=3 alternative code edits targeting the identified region. Each edit should represent a genuinely different repair strategy (e.g., add a null check vs. fix the computation logic vs. change the control flow). Create a child node for each.

7. **Simulate via execution.** For each new child node, complete the partial patch into a full diff, apply it to the codebase, and run the relevant tests. Assign a graded reward:
   - **1.0** -- all tests pass
   - **0.5 + 0.5 * (passing_tests / total_tests)** -- patch applies, partial test pass
   - **0.2** -- patch is syntactically valid but all tests fail
   - **0.0** -- patch doesn't apply or has syntax errors

8. **Backpropagate rewards.** Update the average reward and visit count for the expanded node and all its ancestors. This makes the search progressively smarter about which repair directions are most promising.

9. **Iterate or refine.** If no node has reached reward 1.0 after a round, return to step 5 and select the next most promising node. For nodes with partial success (reward 0.5-0.9), attempt targeted refinement: identify the low-confidence spans in the generated patch and regenerate only those spans, keeping the high-confidence parts fixed.

10. **Terminate and report.** Stop when (a) a patch passes all tests, (b) the iteration budget is exhausted (typically 8-16 simulations), or (c) the best reward has plateaued across consecutive iterations. Present the best patch found along with its test results and a summary of alternative approaches that were explored.

## Concrete Examples

**Example 1: Fixing a failing test with multiple possible root causes**

```
User: The test_checkout_flow test is failing with a KeyError on 'discount_code'.
      Can you fix it?

Approach:
1. Run pytest test_checkout_flow to confirm the failure and capture the traceback.
   The traceback points to checkout/cart.py line 142 and checkout/pricing.py line 87.

2. File-level localization: checkout/cart.py, checkout/pricing.py, and
   checkout/models.py are the top candidates.

3. Function-level localization:
   - cart.py:apply_discount() -- directly in the traceback
   - pricing.py:calculate_total() -- calls apply_discount
   - models.py:CartItem.to_dict() -- constructs the dict that may be missing the key

4. MCTS exploration (3 hypotheses):
   Node A: Add 'discount_code' default in CartItem.to_dict()
           -> Reward 0.7 (test_checkout passes, but test_cart_serialization fails)
   Node B: Add a guard in apply_discount() for missing key
           -> Reward 0.2 (syntactically valid, but wrong fix -- masks the bug)
   Node C: Fix calculate_total() to pass discount_code through the call chain
           -> Reward 1.0 (all tests pass)

5. Select Node C as the fix. Report the root cause: calculate_total() was
   stripping the discount_code field when constructing the pricing context.

Output: A patch to checkout/pricing.py:calculate_total() that preserves
discount_code in the context dict, with all 47 tests passing.
```

**Example 2: Resolving a multi-file GitHub issue**

```
User: GitHub issue #312 says users get a 500 error when uploading files
      larger than 10MB. Please investigate and fix.

Approach:
1. Reproduce: craft a test that uploads an 11MB file and confirm the 500 response.
   Stack trace shows MemoryError in upload/handler.py.

2. File-level localization: upload/handler.py, upload/storage.py,
   config/settings.py, middleware/size_limit.py.

3. Function-level localization:
   - handler.py:process_upload() -- reads entire file into memory
   - storage.py:save_file() -- receives the buffer
   - middleware/size_limit.py:check_size() -- validates but doesn't stream

4. MCTS exploration:
   Node A: Increase memory limit in settings.py
           -> Reward 0.2 (works for 10MB, fails at 50MB -- not a real fix)
   Node B: Switch process_upload() to chunked streaming reads
           -> Reward 0.85 (upload works, but save_file expects full buffer)
   Node B1 (refinement of B): Also update save_file() to accept a stream
           -> Reward 1.0 (all tests pass, 100MB upload verified)

5. Present the two-file patch: handler.py streams in 1MB chunks,
   storage.py writes chunks incrementally.

Output: Patches to upload/handler.py and upload/storage.py implementing
chunked streaming upload. Regression tests pass. Verified with 100MB file.
```

**Example 3: Iterative refinement when first fix partially works**

```
User: The CSV export is producing malformed output -- columns are shifted
      by one for rows that contain commas in text fields.

Approach:
1. Reproduce: run export on test data with embedded commas, confirm shift.

2. Localize to export/csv_writer.py:format_row() -- the quoting logic.

3. MCTS exploration:
   Node A: Wrap all fields in double quotes unconditionally
           -> Reward 0.75 (fixes shift, but breaks downstream parser
              that expects unquoted numeric fields)
   Node B: Fix the conditional quoting to properly escape embedded commas
           -> Reward 0.9 (fixes most cases, fails on fields with both
              commas and double quotes)

4. Confidence analysis on Node B's patch: the regex replacement for
   quote escaping has high entropy (low confidence). Regenerate only
   that span.

   Node B1: Use csv.writer stdlib instead of manual formatting
           -> Reward 1.0 (all edge cases handled by stdlib)

Output: Replace manual CSV formatting with csv.writer, which correctly
handles commas, quotes, and newlines within fields. All 23 export tests pass.
```

## Best Practices

- **Do:** Always reproduce the bug and capture concrete test output before attempting any fix. This output is your reward signal -- without it, you cannot evaluate candidates.
- **Do:** Generate genuinely diverse fix hypotheses at each expansion step. Three variations of the same idea waste the search budget. Aim for structurally different approaches (e.g., fix the caller vs. fix the callee vs. add validation).
- **Do:** Use graded rewards, not binary pass/fail. A patch that fixes 8 of 10 tests is much more informative than one that fixes 0 -- it tells you the direction is promising and needs refinement.
- **Do:** Prune aggressively. If a node scores 0.0 or 0.2, do not expand it further. Invest simulation budget in nodes scoring 0.5+.
- **Avoid:** Generating more than 3-4 candidate edits per node. The branching factor must stay low to keep the search tractable within a reasonable number of iterations.
- **Avoid:** Applying fixes without running tests. Every candidate patch must be validated by execution. A fix that "looks right" but hasn't been tested is not a fix.
- **Avoid:** Re-generating an entire patch when only one part is uncertain. Use confidence-calibrated refinement to selectively regenerate low-confidence spans while preserving correct parts.

## Error Handling

- **Tests cannot be run** (no test suite, broken environment): Fall back to static analysis as the reward signal. Check for syntax errors, type consistency, and logical correctness. Assign partial reward for patches that pass linting and type checking. Warn the user that the fix is unverified.
- **All candidates score 0.0**: The fault localization is likely wrong. Backtrack to step 2 and expand the file-level search to include more candidates. Consider that the bug may originate from configuration, data, or dependencies rather than source code.
- **Search budget exhausted without full pass**: Present the highest-scoring partial fix, explain which tests still fail and why, and suggest manual investigation directions for the remaining failures.
- **Patch applies but introduces new failures**: This indicates a correct fix direction with an incomplete understanding of side effects. Create a new search branch that starts from this patch and specifically targets the newly failing tests.
- **Flaky tests confuse reward signals**: If the same patch gets different rewards on repeated runs, identify and exclude flaky tests from the reward calculation. Note which tests are flaky in the report.

## Limitations

- This approach requires an executable test suite or reproducible error condition. Without execution feedback, the tree search degrades to unguided generation.
- The method works best for bugs with clear, localized root causes. Systemic architectural issues, performance regressions, or concurrency bugs are poor fits for tree-structured patch search.
- The search budget (typically 8-16 simulations) limits how many hypotheses can be explored. Bugs requiring very deep reasoning chains (5+ sequential edits) may exhaust the budget before finding a solution.
- Graded rewards depend on test granularity. If the test suite has only one coarse integration test, partial credit signals are weak and the search becomes nearly random.
- This skill is most effective for single-issue repair. Multi-bug scenarios where several independent faults interact should be decomposed into separate MCTS searches.

## Reference

Liang, Y. (2026). *Monte Carlo Tree Search for Execution-Guided Program Repair with Large Language Models.* arXiv:2602.00129v1. Key takeaways: the UCT selection formula with LLM prior (Section 3.3), the graded reward function (Section 3.4), and the ablation study showing MCTS contributes +4.34% over direct generation (Table 2).