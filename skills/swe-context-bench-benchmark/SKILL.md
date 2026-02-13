---
name: "swe-context-bench-benchmark"
description: "Reuse prior coding experience across related repository tasks. Accumulate, summarize, retrieve, and inject compact experience from previously solved issues to boost accuracy and cut cost on downstream tasks. Use when: 'reuse context from prior fixes', 'apply experience from related issues', 'solve this like the last bug', 'use what we learned fixing X to fix Y', 'accumulate coding experience across tasks', 'context-aware issue resolution'."
---

# SWE-ContextBench: Experience Reuse for Repository-Level Coding Tasks

This skill enables Claude to accumulate structured experience from previously solved coding tasks and selectively reuse that experience when tackling related issues in the same repository. Based on the SWE-ContextBench framework, the core technique is: solve a task, generate a compact summary of the solution trajectory, store it, then retrieve and inject the most relevant summaries when encountering related tasks. Oracle-selected compact summaries (averaging ~200 words extracted from ~25,000-word full trajectories) improved resolution rates from 26% to 34% while cutting runtime by 40% and cost per instance by 50%. The key insight is that *correctly selected, concise* experience dramatically outperforms both no experience and unfiltered full-trajectory dumps.

## When to Use

- When the user is working through a sequence of related GitHub issues or bug fixes in the same repository
- When a new bug or feature request touches code that was modified in a recently solved issue
- When the user says "fix this the same way we fixed the last one" or "apply what we learned from issue X"
- When tackling a batch of issues that share dependencies, reference each other, or modify overlapping files
- When the user wants to reduce redundant exploration of a codebase across multiple sequential tasks
- When building an agent loop that solves multiple SWE-bench-style tasks and should learn across them

## Key Technique

**Experience Reuse via Compact Summaries.** When an agent solves a coding task, its full interaction trajectory (file navigation, hypothesis testing, intermediate reasoning, final patch) averages ~25,000 words. Feeding this raw trajectory into the context window of a downstream task wastes tokens and introduces noise. Instead, the technique extracts a *compact summary* of ~200 words capturing: the root cause, the files and functions modified, the fix strategy, and any non-obvious gotchas. This 121x compression preserves the actionable signal while fitting easily into the context budget.

**Selective Retrieval Over Bulk Injection.** The paper demonstrates that *which* experience you retrieve matters more than *how much*. Oracle-guided retrieval (selecting experience known to be related via dependency graphs) yields the largest gains. Autonomous retrieval (the agent picks from a pool) helps less, and injecting irrelevant experience actively degrades performance by 15-25%. The practical takeaway: invest effort in identifying task relationships (shared files, import chains, issue cross-references) rather than dumping all prior experience into context.

**Task Relationships from Repository Signals.** Related tasks are identified through six concrete patterns: (1) a single PR resolving multiple issues, (2) PRs referencing other issues, (3) PRs referencing other PRs, (4) issues referencing other issues, (5) issues referencing PRs, and (6) detailed PR descriptions revealing implicit dependencies. These relationships form directed graphs that can be traversed to find second-order connections.

## Step-by-Step Workflow

1. **Solve the base task fully.** Complete the coding task using normal debugging and patching workflow. Do not shortcut the investigation -- the quality of the experience summary depends on thorough understanding of the root cause and fix.

2. **Generate a compact experience summary.** After solving, write a structured summary (150-250 words) containing:
   - **Root cause:** One sentence describing the underlying bug or requirement.
   - **Files modified:** List of files and specific functions/classes changed.
   - **Fix strategy:** 2-3 sentences on *what* was changed and *why* that approach was chosen over alternatives.
   - **Gotchas:** Any non-obvious traps (e.g., "changing X also requires updating the cache invalidation in Y").
   - **Test signals:** Which tests broke and what they validate.

3. **Store the summary with metadata.** Persist the summary alongside: repository name, base commit hash, issue/PR identifier, list of modified file paths, and a one-line task description. Use a flat JSON or markdown file per experience entry in a `.experience/` directory.

4. **Identify related downstream tasks.** When a new task arrives, check for relationships by:
   - Scanning for overlapping modified files between the new issue and stored experiences
   - Checking if the new issue references or is referenced by stored issue/PR IDs
   - Comparing import chains -- does the new task's target code import modules modified in a prior fix?
   - Looking for explicit cross-references in issue descriptions ("related to #123", "follow-up to PR #456")

5. **Rank and select top-k experiences.** Score each candidate experience by: (a) file overlap count, (b) presence of explicit cross-references, (c) recency. Select the top 1-3 experiences. Do not inject more than 3 -- additional context yields diminishing returns and risks confusion.

6. **Inject selected summaries into the task prompt.** Prepend the retrieved summaries before the new task's problem statement using a clear delimiter:
   ```
   ## Prior Experience (Related Tasks)
   ### Experience 1: [one-line description]
   [compact summary]
   ### Experience 2: [one-line description]
   [compact summary]
   ## Current Task
   [problem statement]
   ```

7. **Solve the new task with experience-informed reasoning.** Explicitly reference the injected experience when it applies: skip re-exploring files already understood, reuse fix patterns from similar bugs, and anticipate gotchas flagged in prior summaries.

8. **Validate that experience was applied correctly.** Run the full test suite. If the fix fails, consider whether the retrieved experience was actually relevant or misleading. If misleading, discard it and re-solve from scratch without the injected context.

9. **Generate and store a new summary for this task.** Repeat step 2 for the just-solved task, growing the experience pool.

10. **Periodically prune the experience pool.** Remove entries for experiences that have never been retrieved, or that have been retrieved but led to failed downstream tasks (negative signal).

## Concrete Examples

**Example 1: Sequential Django Bug Fixes**

```
User: Fix issue #31245 -- QuerySet.values() returns wrong column names after annotation.
(The user already solved #31200 -- QuerySet.annotate() generates incorrect SQL for subqueries.)

Approach:
1. Retrieve stored experience from #31200:
   Summary: "Root cause: annotate() was not propagating alias mappings to
   inner query compilation. Fixed in django/db/models/sql/query.py
   (resolve_expression) and django/db/models/query.py (annotate method).
   Gotcha: changes to alias mapping must also update the _annotation_select
   cache or stale names persist."

2. Start investigation at the files flagged in the summary rather than
   exploring the full ORM from scratch: django/db/models/sql/query.py
   and django/db/models/query.py.

3. Identify that values() reads from _annotation_select -- the same cache
   mentioned in the gotcha. The stale alias names from #31200's domain
   are the root cause here too, but in a different code path.

4. Apply fix: update the values() method to refresh alias mappings from
   the resolved annotation select, mirroring the pattern used in #31200.

5. Run tests: all FAIL_TO_PASS tests pass, PASS_TO_PASS tests stable.

Output experience summary:
"Root cause: values() read stale alias names from _annotation_select
cache after annotation. Files: django/db/models/query.py (values method).
Fix: refresh alias mapping before column name resolution, same pattern
as #31200's annotate fix. Gotcha: any method reading _annotation_select
should call resolve_annotation_aliases() first."
```

**Example 2: scikit-learn Estimator Chain**

```
User: Fix issue #19012 -- Pipeline.fit_transform inconsistent with
Pipeline.fit().transform() when last step has no transform.
(Prior experience pool contains #18974 -- Pipeline.set_params fails
silently when step name contains double underscore.)

Approach:
1. Retrieve experience from #18974:
   Summary: "Root cause: _validate_names() in pipeline.py did not check
   for double underscores. Files: sklearn/pipeline.py (_validate_names,
   set_params). Gotcha: Pipeline delegates heavily to utils/metaestimators
   -- changes to step validation must be mirrored there."

2. The file overlap (sklearn/pipeline.py) triggers retrieval. The fix
   pattern (validation consistency) is conceptually related but the
   specific gotcha about metaestimators is the real value -- fit_transform
   dispatches through _fit() which calls metaestimator utilities.

3. Investigate fit_transform in sklearn/pipeline.py. Find that it
   assumes the last step always has transform(), but fit().transform()
   has a hasattr guard. Add the same guard to fit_transform.

4. Store new experience summary mentioning the metaestimator dispatch
   path for future pipeline-related issues.
```

**Example 3: Discarding Irrelevant Experience**

```
User: Fix issue #5432 -- matplotlib colorbar tick labels overlap on
log-scale axes.
(Experience pool contains a fix for a Django migration issue -- no
file overlap, no cross-references.)

Approach:
1. Retrieval scores all stored experiences. The Django migration
   experience has zero file overlap, no cross-references, different
   repository. Score: 0.

2. No experience passes the relevance threshold. Solve from scratch
   with no injected context.

3. This is correct behavior. The paper shows that injecting irrelevant
   experience degrades accuracy by 15-25%. Better to have no context
   than wrong context.
```

## Best Practices

- **Do:** Keep summaries under 250 words. The 121x compression from full trajectory to summary is where the efficiency gain lives. Verbose summaries erode the benefit.
- **Do:** Include the "gotchas" field in every summary. Non-obvious side effects (cache invalidation, sibling method updates, test fixture dependencies) are the highest-value information for downstream tasks.
- **Do:** Use file-path overlap as the primary retrieval signal. It is the strongest predictor of task relatedness in real repositories.
- **Do:** Limit injected experience to 1-3 summaries. Beyond that, context pollution outweighs information gain.
- **Avoid:** Injecting full execution trajectories. They consume 100x more tokens than summaries with no accuracy improvement and measurably worse cost/time efficiency.
- **Avoid:** Injecting experience when retrieval confidence is low. No experience is strictly better than irrelevant experience. If no stored summary shares files or cross-references with the current task, solve from scratch.

## Error Handling

- **Retrieved experience contradicts current codebase state.** The stored experience may reference code that has since been refactored. If file paths or function names in the summary don't match the current commit, discard that experience and note it for pruning.
- **Experience leads to a wrong fix direction.** If you apply an experience-suggested pattern and tests fail, explicitly flag that the experience was misleading, remove it from context, and re-approach the problem without it. Do not iterate on a bad direction.
- **Experience pool grows too large for manual review.** When the pool exceeds ~50 entries per repository, implement automated scoring: rank by retrieval frequency (higher = more useful) and downstream success rate (did tasks that used this experience pass?). Prune the bottom quartile.
- **Multiple experiences suggest conflicting approaches.** Prefer the experience with higher file-path overlap with the current task. If tied, prefer the more recent experience (closer to the current commit is more likely to reflect current code state).

## Limitations

- **Cross-repository transfer is weak.** Experience from one repository rarely helps in another. The technique works because related tasks share concrete code, not abstract patterns.
- **Cold start problem.** The first task in a repository has no experience to draw from. The benefit only materializes after 5-10 solved tasks build up a useful pool.
- **Autonomous retrieval lags oracle retrieval significantly.** Without ground-truth dependency graphs, retrieval accuracy drops and the benefit shrinks. File-overlap heuristics help but don't fully close the gap.
- **Summaries can lose critical detail.** The 121x compression occasionally drops information that turns out to be necessary. If a summary-informed fix fails, falling back to the full trajectory (or re-solving from scratch) is the correct response.
- **Not useful for isolated, unrelated tasks.** If the user is working on a single standalone issue with no history, this technique adds overhead with no benefit.

## Reference

- **Paper:** [SWE-ContextBench: A Benchmark for Context Learning in Coding](https://arxiv.org/abs/2602.08316v1) (Zhu, Hu, Wu, 2026)
- **Key finding:** Oracle-selected compact summaries (~200 words) achieve 34.3% resolution rate vs. 26.3% baseline, with 40% runtime reduction and 50% cost reduction. Look for Table 2 (main results across retrieval settings) and Section 4.3 (analysis of experience quality vs. quantity tradeoffs).