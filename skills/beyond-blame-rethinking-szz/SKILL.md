---
name: "beyond-blame-rethinking-szz"
description: "Identify bug-inducing commits using temporal knowledge graph search beyond git blame. Use when: 'find what commit introduced this bug', 'trace root cause of regression', 'which commit broke this', 'find the bug-inducing change', 'blame analysis for this fix', 'what caused this defect'"
---

# Beyond Blame: Bug-Inducing Commit Identification via Temporal Knowledge Graph Search

This skill enables Claude to identify Bug-Inducing Commits (BICs) by constructing and searching a temporal knowledge graph of git history, going far beyond what `git blame` alone can find. Based on the AgenticSZZ approach, it addresses the critical limitation that over 40% of real bug-inducing commits cannot be found by blame alone -- 28% require traversing commit history beyond blame results, and 14% involve "blameless" bugs where no lines in the fix were directly modified by the culprit commit. The technique reframes BIC identification from a ranking problem over blame commits into a graph search problem where temporal ordering enables causal reasoning about bug introduction.

## When to Use

- When the user asks to find which commit introduced a specific bug or regression
- When `git blame` on a bug fix points to commits that don't look like the actual root cause
- When investigating a bug fix that adds new code rather than modifying existing lines (blameless cases)
- When the user wants to trace a defect back through complex refactoring or multi-file changes
- When building tooling for defect prediction or automated program repair that needs accurate BIC data
- When reviewing a bug fix and wanting to understand the causal chain that led to the defect
- When `git bisect` is impractical due to build complexity and the user needs a static analysis approach

## Key Technique

**Traditional SZZ algorithms** use `git blame` on the lines modified by a Bug-Fixing Commit (BFC) to find commits that last touched those lines. This works for only ~57% of cases. The remaining cases fall into three categories: **Blame Ancestors** (10.3%) where the real BIC is an ancestor of the blamed commit in file history, **BFC Ancestors** (17.7%) where the BIC is found by walking backward from the fix commit rather than from blame results, and **Blameless** (14.1%) where the fix adds entirely new code and there are no deleted/modified lines to blame at all.

**AgenticSZZ** constructs a **Temporal Knowledge Graph (TKG)** with three node types -- Commits, Files, and Functions -- connected by edges: `PRECEDES` (temporal ordering between commits), `MODIFIES_FILE` and `MODIFIES_FUNCTION` (structural connections from commits to code), and `DEFINED_IN` (functions to their containing files). The graph is built by running blame on the BFC, then traversing file history backward from both blame commits and the BFC itself, extracting function-level information from diff hunk headers. Candidates are scored by proximity: blame commits get fitness 1.0, blame ancestors 0.6, and BFC ancestors 0.3.

An **LLM agent** then searches this graph using four tools: `list_candidates` (ranked by fitness), `traverse_graph` (follow edges to discover related commits), `query_node` (read commit metadata), and `read_node_content` (examine actual code diffs for causal analysis). The agent iteratively explores candidates, reads diffs, and reasons about causality -- achieving F1-scores of 0.48-0.74 across datasets, up to 27% improvement over prior state-of-the-art.

## Step-by-Step Workflow

1. **Identify the Bug-Fixing Commit (BFC).** Start from the known fix -- a merge/commit that resolves the bug. Extract the exact files and lines changed by this commit using `git diff`.

2. **Run git blame on fixed lines.** For each line deleted or modified in the BFC, run `git blame` to find the commit that last touched it. These are the "blame commits" -- the traditional SZZ search space. Record which files and functions (from hunk headers) each blame commit touches.

3. **Classify the fix type.** Check whether the fix only adds new code (blameless), modifies existing code (blame-based), or does both. If the fix is purely additive with no deleted lines, skip to step 5 -- blame cannot help.

4. **Traverse blame ancestors.** For each blame commit, walk backward through file history (`git log --follow <file>`) up to 24 commits deep. For each ancestor commit, extract the functions it modifies from diff hunk headers. These are "blame ancestor" candidates with fitness score 0.6.

5. **Traverse BFC ancestors.** Walk backward from the BFC through the history of each modified file, stopping at or before the oldest blame commit timestamp. These are "BFC ancestor" candidates with fitness score 0.3. This step is critical for blameless cases.

6. **Build the temporal knowledge graph.** Create nodes for every commit, file, and function discovered. Connect them with `PRECEDES` edges (chronological order), `MODIFIES_FILE`/`MODIFIES_FUNCTION` edges, and `DEFINED_IN` edges. Ensure all edges respect temporal ordering -- a BIC must precede the BFC.

7. **Rank candidates by fitness.** Sort all candidate commits: blame commits (1.0) > blame ancestors (0.6) > BFC ancestors (0.3). Within each tier, prioritize function-level matches over file-level matches.

8. **Analyze top candidates causally.** For each top candidate (start with highest fitness), read the actual diff. Ask: "Does this change introduce behavior that the bug fix corrects?" Look for semantic connections -- added conditions that are too restrictive, missing null checks, incorrect arithmetic, wrong API usage patterns.

9. **Cross-reference temporal constraints.** Verify the candidate BIC was committed before the bug was reported (if report date is known) and before the BFC. Eliminate candidates that violate temporal causality.

10. **Decide and document reasoning.** Select the most likely BIC with an explanation of the causal chain: what the commit changed, how that change manifests as the bug, and why the BFC reverses or compensates for it.

## Concrete Examples

**Example 1: Blame-based BIC (straightforward case)**

```
User: "This commit abc123 fixes a null pointer exception by adding a null check
       on line 45 of parser.c. What commit introduced the bug?"

Approach:
1. Run `git diff abc123~1 abc123` to see the fix modifies line 45 of parser.c,
   adding `if (node != NULL)` before dereferencing.
2. Run `git blame abc123~1 -- parser.c` on line 45 to find commit def456
   last modified this line.
3. Read `git show def456` -- it refactored the parser to remove a previous
   null guard during a "cleanup" commit.
4. Causal chain confirmed: def456 removed the null check, abc123 restores it.

Output:
Bug-Inducing Commit: def456
Reason: Commit def456 removed the null-safety check on `node` in parser.c:45
during a refactoring pass. The variable can be NULL when parsing empty input,
which the original guard protected against. The fix abc123 restores this check.
Category: Blame (direct blame match)
```

**Example 2: Blame Ancestor BIC (beyond direct blame)**

```
User: "Commit fix789 fixes a race condition in connection_pool.py by adding a
       lock around the pool resize operation. git blame points to commit aaa111
       which just reformatted the code. What actually introduced the bug?"

Approach:
1. git blame on the fixed lines points to aaa111 (a formatting-only commit).
2. aaa111 is not the real cause -- it only changed whitespace.
3. Traverse backward from aaa111 through file history of connection_pool.py.
4. At depth 3, find commit bbb222 which introduced the `resize_pool()` method
   without thread synchronization, moving from a single-threaded to
   multi-threaded architecture.
5. Causal chain: bbb222 added concurrent pool access without locking.

Output:
Bug-Inducing Commit: bbb222
Reason: Commit bbb222 introduced the `resize_pool()` function as part of the
multi-threading migration but did not add synchronization around the shared
pool data structure. The blame commit aaa111 was a red herring (formatting only).
Category: Blame Ancestor (3 commits behind blame, fitness 0.6)
```

**Example 3: Blameless BIC (fix adds entirely new code)**

```
User: "Commit fix999 adds a new validation check for negative values in
       calculate_discount(). There are no deleted lines in the fix -- it only
       adds code. What commit should have included this validation?"

Approach:
1. The fix is purely additive -- no lines deleted or modified means git blame
   has nothing to target. This is a blameless case.
2. Traverse BFC ancestors: walk backward through the history of pricing.py
   from fix999.
3. Find commit ccc333 which introduced calculate_discount() and defined its
   parameter contract. The function accepts a `rate` parameter but never
   validates it can be negative.
4. Causal chain: ccc333 introduced the function with an incomplete contract.

Output:
Bug-Inducing Commit: ccc333
Reason: Commit ccc333 introduced calculate_discount() accepting arbitrary
float values for `rate` without validating the sign. The function computes
`price * (1 - rate)` which produces values exceeding the original price
when rate is negative. The fix adds the missing validation.
Category: Blameless (BFC ancestor, fitness 0.3)
```

## Best Practices

- **Do:** Always check whether the fix is purely additive (blameless) before relying on blame. If the BFC only adds new lines, skip blame entirely and search BFC ancestors.
- **Do:** Extract function names from diff hunk headers (`@@ ... @@ function_name`) to build function-level graph connections. Function-level matches are far more precise than file-level.
- **Do:** Respect temporal causality -- a commit cannot induce a bug if it was made after the bug was reported or after the fix.
- **Do:** Set a depth limit (24 commits covers ~80% of blame ancestor BICs) to avoid unbounded traversal in long-lived files.
- **Avoid:** Trusting blame results at face value. Formatting changes, mass renames, and automated refactoring frequently appear as blame results but are almost never the real BIC.
- **Avoid:** Ignoring BFC ancestor traversal. Nearly 18% of real BICs are found only by walking backward from the fix commit, not from blame.
- **Avoid:** Reading every diff in the graph. Limit expensive diff reads (3 per investigation) and use metadata/fitness scores to prioritize which candidates deserve deep analysis.

## Error Handling

- **Blame returns no results:** The fixed lines may be newly added (blameless case). Fall back to BFC ancestor traversal immediately.
- **File was renamed or moved:** Use `git log --follow` to track the file through renames. The TKG must connect pre-rename and post-rename file nodes.
- **Too many candidates:** If the graph contains hundreds of commits, tighten the depth limit or restrict to function-level matches only. Prioritize by fitness score.
- **Merge commits in history:** Merge commits can obscure authorship. When a merge commit appears as a blame result, traverse both parent branches to find the original change.
- **Ambiguous causality:** When multiple candidates seem plausible, prefer the one closest in time to when the bug behavior was first observed, and the one that modifies the most semantically related code.

## Limitations

- This approach requires a known bug-fixing commit as the starting point. It cannot discover bugs from scratch.
- Function extraction from hunk headers depends on language-specific diff formatting and may be unreliable for languages with unusual syntax.
- The technique works best for bugs where the fix is localized. For bugs caused by emergent interactions across many subsystems, the causal chain may be too diffuse to trace.
- Blameless BIC identification is inherently harder and less precise since there is no direct textual link between the fix and the inducing commit.
- Performance degrades on very long-lived files (thousands of commits) where the search space becomes too large even with depth limits.
- The 0.48 F1-score on the Apache dataset indicates the technique struggles with certain project structures or bug patterns, particularly in large Java codebases.

## Reference

**Paper:** "Beyond Blame: Rethinking SZZ with Knowledge Graph Search" by Yu Shi, Hao Li, Bram Adams, Ahmed E. Hassan (arXiv:2602.02934, 2026). Look for: the TKG construction algorithm, the four agent tools (list_candidates, traverse_graph, query_node, read_node_content), the BIC category distribution table (Table I), and Algorithm 1 describing the agentic search loop.