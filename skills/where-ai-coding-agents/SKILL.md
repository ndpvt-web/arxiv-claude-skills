---
name: "where-ai-coding-agents"
description: "Pre-flight checker that prevents AI coding agent PRs from failing, based on empirical analysis of 33k agent-authored PRs on GitHub. Applies the rejection taxonomy from Ehsani et al. (MSR 2026) to catch the top causes of PR rejection before submission. Use when: 'check this PR before I submit it', 'why might this PR get rejected', 'review my agent-generated PR', 'audit this pull request for common agent mistakes', 'pre-flight check my changes', 'validate this PR against agent failure patterns'."
---

# Where AI Coding Agents Fail: Pre-Flight PR Checker

This skill applies findings from a large-scale empirical study of 33,000 agent-authored pull requests across GitHub (Ehsani et al., MSR 2026) to systematically audit PRs before submission. The study identified that 28.5% of agent PRs fail to merge, with specific, predictable failure patterns. By checking a PR against the paper's hierarchical rejection taxonomy — covering reviewer-level, PR-level, code-level, and agentic-level failures — this skill catches the exact issues that cause real-world rejections before they happen.

## When to Use

- When preparing to submit an AI-generated or AI-assisted pull request to an open-source or team repository
- When reviewing a batch of changes and wanting to predict merge likelihood
- When an agent-generated PR has been rejected and you need to diagnose why
- When deciding whether a task (bug fix, feature, refactoring) is suitable for agentic contribution
- When you want to decompose a large PR into smaller, more mergeable units
- When auditing CI/CD readiness of changes before pushing

## Key Technique

The paper's core contribution is a **hierarchical rejection taxonomy** derived from qualitative analysis of 600 rejected PRs, organized into four levels: Reviewer (38% of rejections — PR simply abandoned with no review), PR-level (30% — duplicates, unwanted features, wrong branch), Code-level (22% — CI failures, incorrect/incomplete implementations), and Agentic-level (2% — misalignment with reviewer instructions, license issues). The critical insight is that **the majority of agent PR failures are not code quality problems** — they are contextual and social failures. The single largest rejection category is "Abandoned/Not Reviewed" (38%), meaning the PR never received meaningful human engagement.

Quantitatively, the study found three statistically significant predictors of rejection: **(1) PR size** — each increase in lines changed reduces merge odds (Cliff's delta = -0.17); **(2) file spread** — touching more files correlates with rejection (delta = -0.10); and **(3) CI failures** — each additional failed CI check reduces merge odds by ~15% (delta = -0.24). Review comment count and revision count were not significant predictors. Task type matters enormously: documentation PRs merge at 84%, CI/build PRs at 74-79%, but bug fixes merge at only 64% and performance PRs at 55%.

The actionable framework is therefore: minimize PR size, ensure CI passes locally before submission, verify the work isn't duplicated, confirm the change is wanted, and target task types where agents empirically succeed.

## Step-by-Step Workflow

1. **Classify the task type.** Determine which of the 11 categories the PR falls into: feature, fix, performance, refactoring, style, documentation, test, chore, build, CI, or other. Flag high-risk categories (performance: 55% merge rate, fix: 64%) and note low-risk ones (documentation: 84%, CI: 79%).

2. **Check for duplicates.** Search the repository's open PRs, recent closed PRs, and issue tracker for overlapping work. Duplicate PRs account for 23% of all rejections. Run: `gh pr list --state all --search "<keywords>"` and `gh issue list --search "<keywords>"`.

3. **Verify the change is wanted.** Confirm there is an open issue, maintainer request, or explicit acceptance of the contribution type. Unwanted features account for 4% of rejections. Check CONTRIBUTING.md, recent maintainer comments, and project roadmap.

4. **Measure PR size and scope.** Count total lines changed and files touched. The study shows not-merged PRs are ~17% larger in LOC and touch ~10% more files. If the diff exceeds 300 lines or touches more than 5 files, consider decomposing into smaller PRs.

5. **Run CI/CD locally before submission.** Execute the project's test suite, linter, and build pipeline. CI failure is the strongest code-level rejection signal (delta = -0.24), with each failed check cutting merge odds by ~15%. Run: `npm test`, `make check`, `pytest`, or whatever the project uses.

6. **Validate branch targeting.** Confirm the PR targets the correct branch (main, develop, release). Wrong-branch PRs are a small but entirely preventable rejection category. Check the project's branching strategy in CONTRIBUTING.md or recent merged PRs.

7. **Audit the PR description.** Ensure the title and body accurately describe the actual changes. Wrong/misleading task descriptions account for 1% of rejections. Cross-reference the diff against the PR description line by line.

8. **Check for completeness.** Verify all required logic is present — no TODO stubs, no placeholder implementations, no missing error handling that the issue explicitly requires. Incomplete implementations account for 2% of rejections.

9. **Verify licensing and CLA requirements.** Check if the project requires a Contributor License Agreement or has specific licensing constraints. License issues block PRs entirely.

10. **Generate a merge-likelihood assessment.** Produce a structured report scoring the PR across all taxonomy levels with a final recommendation: submit as-is, revise, or split.

## Concrete Examples

**Example 1: Pre-flight check on a bug fix PR**

User: "Check this PR before I submit it — it fixes a race condition in the connection pool"

Approach:
1. Classify: This is a "fix" task (64% base merge rate — moderate risk)
2. Run `gh pr list --state all --search "race condition connection pool"` to check for duplicates
3. Measure the diff: `git diff --stat main` — if it touches 12 files and changes 847 lines, flag as oversized
4. Run the project's test suite and CI checks locally
5. Verify an open issue exists requesting this fix

Output:
```
## PR Pre-Flight Report

Task type: bug fix (historical merge rate: 64% — moderate risk)

### Rejection Risk Assessment

| Check                  | Status  | Detail                                      |
|------------------------|---------|---------------------------------------------|
| Duplicate PR           | PASS    | No overlapping PRs found                     |
| Change is wanted       | PASS    | Linked to issue #342                         |
| PR size                | WARNING | 847 LOC across 12 files — exceeds threshold  |
| CI/CD                  | FAIL    | 2 test failures in pool_test.go              |
| Branch target          | PASS    | Targets main (correct per CONTRIBUTING.md)   |
| Description accuracy   | PASS    | Description matches diff                     |
| Implementation complete| PASS    | No TODOs or stubs found                      |
| License/CLA            | PASS    | CLA already signed                           |

### Recommendations
1. CRITICAL: Fix the 2 failing tests before submitting. Each CI failure
   cuts merge odds by ~15%.
2. STRONGLY RECOMMENDED: Split this PR. Consider separating the lock
   refactoring (files 1-4) from the actual race condition fix (files 5-8)
   and the test updates (files 9-12). Smaller PRs merge at significantly
   higher rates.
3. Estimated merge likelihood: 40% as-is → ~70% after fixes above.
```

**Example 2: Diagnosing a rejected PR**

User: "My Copilot-generated PR was closed without comment, what happened?"

Approach:
1. Fetch PR details with `gh pr view <number>`
2. Check comment/review history — if zero human comments, classify as "Abandoned/Not Reviewed" (38% of rejections)
3. Check if the repo has a history of ignoring bot PRs
4. Verify whether the contribution was solicited

Output:
```
## Rejection Diagnosis

Category: Abandoned/Not Reviewed (Reviewer Level)

This is the #1 rejection pattern for agent PRs, accounting for 38% of all
rejections in the MSR 2026 study. The PR was closed without any meaningful
human interaction.

### Likely causes:
- Repository may not accept unsolicited agent contributions
- No linked issue or maintainer request prompted the change
- PR may have been auto-closed by a stale-bot after inactivity

### Remediation:
1. Before resubmitting, open an issue proposing the change and wait for
   maintainer feedback
2. Check if the project has a policy on AI-generated contributions
   (some repos explicitly ban them)
3. Consider contributing to repositories that actively use AI agents
   (check their merged PR history)
```

**Example 3: Choosing the right task for agentic contribution**

User: "I want to use an AI agent to contribute to this repo. What tasks should I target?"

Approach:
1. Fetch recent issues with `gh issue list --label "good first issue" --limit 20`
2. Categorize available issues by task type
3. Rank by empirical merge rates from the study
4. Cross-reference with the specific agent being used

Output:
```
## Task Selection Guide (by merge likelihood)

### High confidence (>75% merge rate):
- Documentation updates, typo fixes, README improvements
- CI/CD pipeline improvements, GitHub Actions updates
- Build configuration, dependency bumps

### Moderate confidence (65-75%):
- Code style fixes, linting cleanups
- Chore tasks (dependency updates, config changes)
- Adding test coverage for existing code
- Refactoring with clear scope

### Lower confidence (<65%):
- Bug fixes (especially complex multi-file bugs)
- New feature implementations
- Performance optimizations (lowest at 55%)

### Recommended issues from this repo:
1. #201 "Update CI to Node 20" — CI task, high merge rate
2. #189 "Fix typo in API docs" — Documentation, highest merge rate
3. #156 "Add missing unit tests for auth module" — Test, moderate rate

Avoid: #178 "Optimize database query performance" — Performance task
with lowest empirical merge rate (55%).
```

## Best Practices

- **Do:** Keep PRs small and focused. The study shows a clear negative correlation between PR size and merge likelihood. Target under 300 LOC and under 5 files.
- **Do:** Run the full CI pipeline locally before submitting. CI failure is the single strongest code-level predictor of rejection (each failure = -15% merge odds).
- **Do:** Check for duplicate work before starting. 23% of rejected PRs were duplicates of existing work. Search open PRs, closed PRs, and issues.
- **Do:** Link to an existing issue or maintainer request. Unsolicited PRs are far more likely to be abandoned without review.
- **Avoid:** Submitting performance optimization PRs without benchmarks and maintainer buy-in. Performance tasks have the lowest merge rate (55%).
- **Avoid:** Large refactoring PRs that touch many files. Decompose into a series of small, independently reviewable changes.
- **Avoid:** Resubmitting a rejected PR without addressing the specific rejection reason. Diagnose the failure category first using the taxonomy.

## Error Handling

- **CI check data unavailable:** If the repository doesn't expose CI status via the API, check for a `.github/workflows/` directory and attempt to run workflows locally with `act` or the equivalent test commands.
- **No PR history to check for duplicates:** Broaden the search to include commit messages (`git log --oneline --grep="<keyword>"`), branch names, and issue comments.
- **Repository has no CONTRIBUTING.md:** Infer contribution norms from recent merged PRs — look at PR size, description format, branch naming, and whether a CLA bot is active.
- **Agent misalignment after reviewer feedback:** If a reviewer requests changes and the agent's follow-up doesn't address them (the "Misalignment" category), intervene manually. Do not let the agent iterate autonomously on reviewer feedback without human verification.

## Limitations

- The study covers five specific agents (Codex, Copilot, Devin, Cursor, Claude Code) as of early 2026. Merge rates for newer agents or updated versions may differ.
- Codex dominates the dataset (65% of PRs) with an unusually high 82.6% merge rate, which skews overall statistics. Agent-specific rates are more reliable.
- The "Abandoned/Not Reviewed" category (38%) may reflect repository-level policies against AI PRs rather than PR quality issues — this is not diagnosable from PR content alone.
- Merge rate is an imperfect proxy for quality. Some merged PRs may introduce technical debt; some rejected PRs may be technically sound but socially unwelcome.
- The taxonomy was derived from open-source GitHub repositories. Internal/enterprise PR dynamics may differ significantly.

## Reference

**Paper:** Ehsani, R., Pathak, S., Rawal, S., Al Mujahid, A., & Imran, M. M. (2026). "Where Do AI Coding Agents Fail? An Empirical Study of Failed Agentic Pull Requests in GitHub." *International Mining Software Repositories Conference (MSR 2026).*
**Link:** https://arxiv.org/abs/2601.15195v1
**Key takeaway:** Look at Table 1 (logistic regression coefficients), Figure 4 (rejection taxonomy), and Tables 2-3 (task-type merge rates by agent) for the core quantitative and qualitative frameworks.