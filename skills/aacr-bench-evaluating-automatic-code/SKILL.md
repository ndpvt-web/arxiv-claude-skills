---
name: "aacr-bench-evaluating-automatic-code"
description: "Perform repository-level automated code review on pull requests using hierarchical context retrieval and structured defect classification. Triggers: 'review this PR', 'find defects in this diff', 'automated code review', 'review these code changes', 'check this pull request for issues', 'analyze this patch for bugs'"
---

# Repository-Level Automated Code Review with Hierarchical Context

This skill enables Claude to perform rigorous automated code review (ACR) on pull requests by applying the hierarchical context retrieval and structured defect classification methodology from AACR-Bench. Instead of reviewing diffs in isolation, Claude retrieves cross-file context at three granularity levels (diff, file, repository), classifies findings into four defect categories (Code Defect, Performance, Security, Maintainability), and pinpoints issues to exact line ranges -- producing high-precision reviews that catch defects raw PR comments miss.

## When to Use

- When the user shares a diff or patch and asks "review this code change" or "find bugs in this PR"
- When reviewing a multi-file pull request that touches cross-file dependencies (imports, shared interfaces, API contracts)
- When the user wants a structured code review with categorized findings rather than ad-hoc comments
- When evaluating code changes across any of the 10 mainstream languages: Python, JavaScript, TypeScript, Java, Go, Rust, C++, C#, Ruby, PHP
- When the user asks to check a PR for security vulnerabilities, performance regressions, or maintainability issues
- When building or improving an automated code review pipeline and needs the review logic itself

## Key Technique

**Hierarchical Context Retrieval.** AACR-Bench demonstrates that reviewing a diff in isolation misses a large class of defects. The paper defines three context scopes: (1) **diff-level** -- issues visible from the changed lines alone, (2) **file-level** -- issues requiring the full file (e.g., a new function that shadows an existing one), and (3) **repo-level** -- issues requiring cross-file knowledge (e.g., a changed API signature that breaks callers in other files). Traditional approaches see recall drop from ~34% to ~18% as context scope increases, while agent-based approaches that autonomously retrieve cross-file context maintain or improve recall on repo-level issues. The critical insight: always attempt cross-file retrieval, but don't let it distract from obvious local issues.

**Precision Over Volume.** The paper's agent-based evaluations show a striking tradeoff: agents produce far fewer comments (0.08-0.15 per patch) but at ~40% precision, versus traditional prompting which generates many comments at ~9% precision. This means each review comment should be a high-confidence finding backed by evidence from the code, not speculative style nitpicking. The goal is to surface defects the original reviewers would miss -- AACR-Bench's annotation pipeline found 285% more defects than raw PR comments by systematically looking beyond the obvious.

**Structured Defect Taxonomy.** Every finding is classified into one of four categories with clear definitions: Code Defects (logic errors, crashes, incorrect output -- 47% of findings), Maintainability (style/design issues impeding comprehension -- 42%), Performance (algorithmic inefficiency, resource waste -- 8%), and Security (vulnerabilities enabling breaches -- 3%). This classification helps reviewers prioritize and ensures coverage across all dimensions.

## Step-by-Step Workflow

1. **Parse the diff and identify changed files.** Extract the list of modified files, the specific line ranges changed, and the PR title/description if available. Understand what the change is trying to accomplish before looking for problems.

2. **Perform diff-level review first.** Read each changed hunk line by line. Look for obvious defects visible within the diff alone: null pointer risks, off-by-one errors, missing error handling, type mismatches, resource leaks, and logic inversions. This catches the ~50% of issues that need no additional context.

3. **Expand to file-level context.** For each changed file, read the full file (not just the diff). Check whether the change is consistent with the rest of the file: does a new function duplicate existing logic? Does a renamed variable miss some occurrences? Does the change break invariants established elsewhere in the same file?

4. **Retrieve repository-level cross-file context.** Identify symbols referenced in the diff that are defined in other files -- imported functions, shared types, API interfaces, configuration constants. Read those files. Check whether the change maintains compatibility: does a modified function signature match all call sites? Does a changed data structure align with its serialization logic elsewhere?

5. **Classify each finding into exactly one category.** For every issue found, assign one of: `Code Defect` (will cause incorrect behavior), `Performance` (causes measurable inefficiency), `Security` (creates an exploitable vulnerability), or `Maintainability` (hinders readability or future changes). If unsure between categories, choose the one with higher severity.

6. **Pin each finding to a precise line range.** Specify the exact `from_line` and `to_line` in the changed file where the issue manifests. If the issue spans a function, point to the specific lines that are problematic, not the entire function.

7. **Write each review comment with evidence.** Structure each comment as: (a) what the problem is, (b) why it is a problem (with reference to the specific code or cross-file dependency), and (c) a concrete suggestion for fixing it. Include a minimal code snippet when the fix is non-obvious.

8. **Filter for precision -- discard low-confidence findings.** Before presenting results, critically evaluate each finding. Remove anything that is merely stylistic preference, already handled by the existing code, or speculative without evidence. Aim for the agent-level precision target: every comment should be actionable and correct.

9. **Present findings in structured format.** Output a summary table with columns: file path, line range, category, severity, and one-line description. Follow with detailed comments for each finding.

10. **Report coverage gaps honestly.** State which context levels you were able to analyze (diff-only, file, or full repo) and note any files you could not access that might contain relevant cross-file dependencies.

## Concrete Examples

**Example 1: Cross-file API breakage (Repository-level defect)**

User: "Review this diff for my Go service"

```diff
// file: internal/handlers/user.go
- func GetUser(id string) (*User, error) {
+ func GetUser(ctx context.Context, id string) (*User, error) {
      return db.FindUser(ctx, id)
  }
```

Approach:
1. Diff-level: Signature changed to add `ctx` parameter -- looks intentional.
2. File-level: Read full `user.go` -- all internal calls updated.
3. Repo-level: Search for all call sites of `GetUser` across the repository.
4. Find `internal/routes/api.go:47` still calls `GetUser(userID)` with one argument.

Output:
```
| File                      | Lines | Category    | Severity | Description                              |
|---------------------------|-------|-------------|----------|------------------------------------------|
| internal/routes/api.go    | 47    | Code Defect | High     | Call site not updated for new signature   |

**Detail:** `GetUser` now requires a `context.Context` as its first argument,
but `internal/routes/api.go:47` still calls `GetUser(userID)` with a single
argument. This will fail to compile.

Fix: Update the call to `GetUser(r.Context(), userID)`.
```

**Example 2: Security vulnerability in input handling (Diff-level defect)**

User: "Check this Python PR for issues"

```diff
// file: app/views/upload.py
+ @app.route('/upload', methods=['POST'])
+ def upload_file():
+     filename = request.files['file'].filename
+     request.files['file'].save(os.path.join('/uploads', filename))
+     return jsonify({"status": "ok"})
```

Approach:
1. Diff-level: `filename` comes directly from user input and is passed to `os.path.join` and `save` without sanitization.
2. File-level: No sanitization utility imported or used in this file.
3. Classification: Security -- path traversal vulnerability.

Output:
```
| File                  | Lines | Category | Severity | Description                          |
|-----------------------|-------|----------|----------|--------------------------------------|
| app/views/upload.py   | 3-4   | Security | Critical | Path traversal via unsanitized filename |

**Detail:** `filename` is taken directly from the uploaded file's metadata,
which is user-controlled. An attacker can submit `../../etc/passwd` as a
filename to write outside the `/uploads` directory.

Fix: Use `werkzeug.utils.secure_filename(filename)` before constructing the path:
```python
from werkzeug.utils import secure_filename
safe_name = secure_filename(request.files['file'].filename)
request.files['file'].save(os.path.join('/uploads', safe_name))
```
```

**Example 3: Performance regression in database query (File-level defect)**

User: "Review these changes to our TypeScript data layer"

```diff
// file: src/services/orders.ts
  export async function getOrdersByUser(userId: string) {
-   return db.orders.findMany({ where: { userId }, take: 100 });
+   const orders = await db.orders.findMany({ where: { userId } });
+   const enriched = await Promise.all(
+     orders.map(o => db.orderItems.findMany({ where: { orderId: o.id } }))
+   );
+   return orders.map((o, i) => ({ ...o, items: enriched[i] }));
  }
```

Approach:
1. Diff-level: `take: 100` limit removed; N+1 query pattern introduced.
2. File-level: Check if there's a relation defined that allows eager loading -- find `db.orders` includes `orderItems` relation in Prisma schema.

Output:
```
| File                     | Lines | Category    | Severity | Description                          |
|--------------------------|-------|-------------|----------|--------------------------------------|
| src/services/orders.ts   | 2-5   | Performance | High     | N+1 query and removed pagination     |
| src/services/orders.ts   | 2     | Code Defect | Medium   | Unbounded query after limit removal  |

**Detail (Performance):** Each order triggers a separate `findMany` for its items,
creating an N+1 query pattern. With the `take: 100` limit removed, this can
issue thousands of queries for active users.

Fix: Use Prisma's `include` for eager loading:
```typescript
return db.orders.findMany({
  where: { userId },
  take: 100,
  include: { orderItems: true }
});
```
```

## Best Practices

- **Do:** Always read cross-file dependencies before concluding a review. The paper shows ~15% of real defects require repository-level context that is invisible in the diff alone.
- **Do:** Classify every finding into one of the four categories (Code Defect, Performance, Security, Maintainability). This forces rigor and helps prioriters triage.
- **Do:** Start with diff-level analysis before expanding scope. Agent-based reviewers that jump straight to repo-level context tend to miss obvious local issues (the paper measured a recall drop on diff-level issues for agents).
- **Do:** Provide a concrete fix or code suggestion for each finding. A review comment without a path forward is incomplete.
- **Avoid:** Generating high volumes of low-confidence comments. The paper shows traditional approaches produce many false positives (~91% noise). Aim for fewer, higher-precision findings.
- **Avoid:** Reviewing style or formatting unless it introduces a real maintainability problem. Linter-level feedback wastes reviewer attention.
- **Avoid:** Assuming one context level is universally best. The paper demonstrates that optimal context depends on the language, the model, and the specific defect type.

## Error Handling

- **Cannot access referenced files:** If cross-file context is unavailable (e.g., the user only provides a diff), state explicitly that the review is limited to diff-level and file-level analysis, and list the external symbols that should be manually verified.
- **Ambiguous diff format:** If the diff is malformed or uses an unfamiliar format, ask the user to provide it as unified diff (`diff -u` or `git diff` output).
- **Large PRs (>20 files):** Prioritize files with the most changed lines and files that define public APIs or interfaces. Flag that a complete review would require examining the remaining files.
- **Unfamiliar language or framework:** If the changed code uses a language or framework outside your knowledge, state this limitation and focus on language-agnostic defect patterns (null handling, resource leaks, error propagation).

## Limitations

- **Recall ceiling:** Even with full repository context, automated review catches a fraction of all defects. The best models in AACR-Bench achieved ~43% recall. Human review remains essential for complex logic and domain-specific correctness.
- **Language-dependent accuracy:** Performance varies significantly across languages (up to 3x F1 variance). Reviews of less common languages (Ruby, PHP) may have lower precision than Python or Go reviews.
- **Semantic matching uncertainty:** Determining whether a finding is a true positive requires judgment. Two comments about the same code region may describe different issues, or the same issue differently.
- **No runtime analysis:** This approach is purely static. It cannot detect issues that only manifest under specific runtime conditions, concurrency scenarios, or data-dependent paths.
- **Style vs. substance boundary:** The Maintainability category can overlap with subjective style preferences. When in doubt, omit rather than include.

## Reference

- **Paper:** [AACR-Bench: Evaluating Automatic Code Review with Holistic Repository-Level Context](https://arxiv.org/abs/2601.19494v3) -- Focus on Section 4 (evaluation methodology), Table 2 (retrieval method comparison), and Section 5.3 (context granularity findings) for the core insights on hierarchical context retrieval and the precision-recall tradeoff in agent-based review.
- **Repository:** [github.com/alibaba/aacr-bench](https://github.com/alibaba/aacr-bench) -- Contains the dataset (200 PRs, 1,505 annotated comments across 10 languages), evaluation scripts, and prompt templates.