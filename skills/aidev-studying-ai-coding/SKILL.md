---
name: "aidev-studying-ai-coding"
description: "Analyze AI coding agent activity on GitHub repositories using the AIDev methodology. Identify agentic PRs, measure agent adoption metrics, evaluate PR quality, assess review dynamics, and benchmark human-AI collaboration patterns. Use when: 'analyze AI agent PRs in this repo', 'measure AI coding adoption', 'evaluate agentic PR quality', 'compare AI agent contributions', 'audit AI-generated code patterns', 'assess review burden of AI PRs'."
---

# AIDev: Analyzing AI Coding Agent Activity in GitHub Repositories

This skill enables Claude to systematically analyze how AI coding agents (Codex, Devin, GitHub Copilot, Cursor, Claude Code) contribute to software repositories. Using the classification framework and metrics from the AIDev study of 932,791 agentic pull requests across 116,211 repositories, Claude can identify agent-authored PRs, measure adoption patterns, evaluate code quality signals, assess review dynamics, and surface failure risks — giving teams actionable insight into their human-AI collaboration effectiveness.

## When to Use

- When the user asks to identify which PRs in a repository were authored by AI coding agents
- When measuring adoption rates of AI agents across a codebase or organization over time
- When evaluating the quality characteristics of agent-authored PRs (size, testing, CI pass rates)
- When analyzing review burden — how much human effort AI PRs require before merging
- When auditing for common failure patterns in AI-generated code (security issues, reverted PRs, missing tests)
- When benchmarking a team's AI coding practices against known effective patterns
- When building tooling or dashboards to track agentic contributions in a project

## Key Technique

The AIDev methodology identifies **agentic PRs** — pull requests authored or substantially produced by AI coding agents — by matching known agent signatures in PR metadata: author accounts, commit trailers, bot tags, and PR body patterns specific to each agent (e.g., Devin's `devin-ai-integration` accounts, Copilot's `copilot` trailer tags, Claude Code's `Co-Authored-By` trailers). This signature-based detection is the foundation: without reliable identification, no downstream analysis is meaningful.

Once agentic PRs are identified, the framework applies **multi-dimensional characterization** across five domains: (1) adoption demographics and practices, (2) code patch characteristics including additions/deletions ratios and file-type distributions, (3) testing behavior measured by test-to-code churn ratios, (4) review dynamics including comment types, resolution rates, and time-to-merge, and (5) failure patterns such as reversion frequency, CI failures, and security risks. PRs are classified by purpose using Conventional Commits categories (feat, fix, refactor, test, docs, chore, etc.), enabling structured comparison across agents and projects.

The critical insight is that **raw merge rate is insufficient** for evaluating AI agent effectiveness. A holistic assessment requires examining the full lifecycle: Does the PR include tests? Does it pass CI? How many review rounds does it need? Is it reverted later? Does it introduce security issues? Teams that only track "did it merge" miss the review burden, maintenance cost, and risk dimensions that determine actual productivity impact.

## Step-by-Step Workflow

1. **Collect PR metadata** from the target repository using the GitHub API (`gh pr list --json author,labels,body,commits,reviews,state,createdAt,mergedAt`). Retrieve at minimum: author login, PR title, body, commit messages, review comments, CI status, and merge state.

2. **Identify agentic PRs** by matching agent signatures against known patterns:
   - **OpenAI Codex**: Author accounts containing `codex`, commit trailers with `openai-codex`
   - **Devin**: Author accounts matching `devin-ai-integration/*` or PR bodies referencing Devin sessions
   - **GitHub Copilot**: Commit trailers containing `copilot`, author `copilot` or `github-copilot`
   - **Cursor**: Commit trailers or PR bodies referencing `cursor`, author patterns with `cursor`
   - **Claude Code**: `Co-Authored-By` trailers mentioning `Claude`, PR bodies referencing Claude Code sessions

3. **Classify each agentic PR by purpose** using Conventional Commits parsing on the PR title and commit messages. Map to categories: `feat` (new feature), `fix` (bug fix), `refactor`, `test`, `docs`, `chore`, `ci`, `style`, `perf`. For PRs without conventional prefixes, infer from the diff: majority additions = likely feat, majority deletions = likely refactor/cleanup, test files only = test.

4. **Compute code patch metrics** for each PR: total files changed, lines added, lines deleted, add/delete ratio, file types touched (source vs. config vs. test vs. docs), and whether the PR modifies only generated or boilerplate files versus core logic.

5. **Evaluate testing behavior** by checking: Does the PR include changes to test files? What is the test-to-source churn ratio (lines changed in test files / lines changed in source files)? Does the PR add new test files or only modify existing ones? Flag PRs that change source code without any corresponding test changes.

6. **Analyze review dynamics** by extracting: number of review rounds before merge, types of review comments (correctness, style, security, testing, performance), whether the agent responded to review feedback with follow-up commits, time from PR creation to first review, and time from first review to merge.

7. **Detect failure signals** by checking: CI pipeline pass/fail status, whether the PR was later reverted (search for revert commits referencing the PR), whether hotfix PRs followed shortly after merge, and whether the diff introduces patterns associated with security risks (hardcoded secrets, insecure API usage, disabled security checks).

8. **Aggregate into a report** with per-agent and per-category breakdowns: merge rate, median time-to-merge, median review rounds, test coverage rate, CI pass rate, reversion rate, and review burden score (comments per PR). Compare against the repository's human-authored PR baselines for the same period.

9. **Surface actionable recommendations** based on the analysis: identify which agent-task combinations perform best, flag categories with high review burden or reversion rates, recommend where human oversight should be increased, and suggest process improvements (e.g., requiring test changes for agent feat PRs).

10. **Export structured data** as JSON or CSV for integration with dashboards, further analysis, or time-series tracking of AI adoption trends.

## Concrete Examples

**Example 1: Repository AI Adoption Audit**

User: "Analyze AI coding agent usage in our repository over the last 6 months"

Approach:
1. Fetch all PRs from the last 6 months with full metadata via `gh pr list`
2. Scan each PR for agent signatures in author, commits, and body
3. Classify identified agentic PRs by agent and purpose category
4. Compute monthly adoption trends and per-agent distribution

Output:
```
## AI Agent PR Analysis: myorg/myrepo (Aug 2025 - Jan 2026)

### Adoption Summary
- Total PRs: 1,247
- Agentic PRs: 183 (14.7%)
- Agents detected: Claude Code (98), GitHub Copilot (52), Cursor (33)

### Monthly Trend
| Month    | Total PRs | Agentic PRs | % Agentic |
|----------|-----------|-------------|-----------|
| Aug 2025 |       189 |          12 |      6.3% |
| Sep 2025 |       211 |          19 |      9.0% |
| Oct 2025 |       198 |          27 |     13.6% |
| Nov 2025 |       223 |          34 |     15.2% |
| Dec 2025 |       201 |          41 |     20.4% |
| Jan 2026 |       225 |          50 |     22.2% |

### By Purpose (Conventional Commits)
- fix: 72 (39.3%) — most common agent task
- feat: 48 (26.2%)
- refactor: 31 (16.9%)
- docs: 18 (9.8%)
- chore/ci: 14 (7.7%)

### Key Findings
- Agent adoption doubled over 6 months (6.3% → 22.2%)
- Claude Code is the dominant agent (53.6% of agentic PRs)
- Bug fixes are the most common agent task type
- 3 developers account for 61% of all agentic PRs
```

**Example 2: Agentic PR Quality Assessment**

User: "Evaluate the quality of AI-generated PRs compared to human PRs in this repo"

Approach:
1. Identify agentic vs. human PRs using signature detection
2. Compute quality metrics for both groups: merge rate, CI pass rate, review rounds, test inclusion, reversion rate
3. Run comparative analysis with statistical context

Output:
```
## PR Quality Comparison: Agentic vs. Human

| Metric                        | Agentic (n=183) | Human (n=1064) |
|-------------------------------|-----------------|----------------|
| Merge rate                    |           78.1% |          84.3% |
| CI pass on first push         |           62.8% |          71.2% |
| Median review rounds          |             2.1 |            1.4 |
| PRs including test changes    |           34.4% |          58.7% |
| Median lines changed          |              87 |            143 |
| Reversion within 7 days       |            4.9% |           2.1% |
| Median time-to-merge (hours)  |            18.3 |           22.7 |

### Review Comment Analysis (curated subset, n=89 agentic PRs)
- Correctness issues raised: 41.6% of agentic PRs
- Style/formatting issues: 23.6%
- Missing test coverage flagged: 33.7%
- Security concerns raised: 5.6%

### Recommendations
1. REQUIRE test changes for agentic feat/fix PRs — 65.6% ship without tests
2. ADD pre-merge CI gate — 37.2% fail CI on first push
3. INCREASE review scrutiny for refactor PRs — highest reversion rate (8.1%)
4. MONITOR security-flagged PRs — small count but high severity
```

**Example 3: Identifying Agent Signatures in a PR**

User: "Is this PR authored by an AI agent? Check PR #4521"

Approach:
1. Fetch PR metadata, body, commit messages, and author info
2. Check against known agent signature patterns
3. Report confidence level and which signals matched

Output:
```
## Agent Detection: PR #4521

**Result: Agentic PR detected — Claude Code (HIGH confidence)**

Signals matched:
  [x] Commit trailer: "Co-Authored-By: Claude <noreply@anthropic.com>"
  [x] PR body contains: "Generated with Claude Code"
  [ ] Author account: standard user account (john-dev)
  [ ] No Devin/Codex/Copilot/Cursor markers found

Classification: fix (from commit prefix "fix: resolve null pointer in auth middleware")
Files changed: 3 (src/auth/middleware.ts, src/auth/middleware.test.ts, CHANGELOG.md)
Test included: Yes (middleware.test.ts modified)
CI status: Passing
```

## Best Practices

- **Do:** Always check multiple signal sources (author, commits, PR body, trailers) — agents are identified by the union of signals, not any single one. Some developers use agents without standard signatures.
- **Do:** Normalize comparisons by PR purpose category. Comparing agentic `fix` PRs against human `feat` PRs produces misleading results. Always compare like-for-like.
- **Do:** Track reversion and hotfix rates as lagging quality indicators. A merged PR is not a quality PR if it gets reverted within a week.
- **Do:** Include the test-to-source churn ratio in quality assessments. This single metric strongly signals whether an agent PR is production-ready or just "compiles and ships."
- **Avoid:** Relying solely on merge rate as a quality signal. High merge rates in repos with lax review standards do not indicate high quality — assess review depth alongside merge outcomes.
- **Avoid:** Assuming all PRs from a human account are human-authored. Many developers use AI agents under their own accounts with only commit trailers as indicators.

## Error Handling

- **Missing agent signatures**: Some agent-authored PRs have no detectable markers (e.g., developer manually authored the commits after generating code with an agent). Acknowledge this as a false-negative limitation and report detection confidence levels.
- **GitHub API rate limits**: When analyzing large repositories, paginate requests and cache results. Use `gh api --paginate` and store intermediate JSON to avoid re-fetching.
- **Inconsistent commit history**: Squash-merged PRs lose individual commit trailers. Check both the PR body and the squash commit message for agent signatures.
- **Private repository access**: Ensure the `gh` CLI is authenticated with appropriate scopes. If reviews or comments return 404, the token may lack `repo` scope.
- **Conventional Commits not used**: If the repository does not follow Conventional Commits, fall back to diff-based classification (majority test files = test, majority docs = docs, etc.) and note reduced classification confidence.

## Limitations

- **Signature evasion**: Developers who strip agent markers from commits produce false negatives. The methodology cannot detect "ghost-written" code with no remaining agent trace.
- **Agent version differences**: The same agent (e.g., Copilot) may behave very differently across versions. The analysis captures which agent, not which model version or configuration.
- **Repository bias**: The curated subset (100+ stars) skews toward well-maintained projects with active review cultures. Results may not generalize to smaller or less-maintained repositories.
- **Causal attribution**: Correlation between agent usage and quality metrics does not imply causation. Developers who adopt agents may already write differently-sized or differently-scoped PRs.
- **Temporal coverage**: AI coding agents are evolving rapidly. Patterns observed in historical data may not reflect current agent capabilities.
- **No runtime analysis**: The methodology analyzes PR artifacts (code, reviews, CI status) but cannot assess runtime correctness, performance regressions, or production incident rates without additional infrastructure.

## Reference

**Paper**: Li, H., Zhang, H., & Hassan, A.E. (2026). *AIDev: Studying AI Coding Agents on GitHub*. arXiv:2602.09185v1. [https://arxiv.org/abs/2602.09185v1](https://arxiv.org/abs/2602.09185v1)

**Dataset**: Available on [Hugging Face](https://huggingface.co/) and [Zenodo](https://zenodo.org/) (DOI: 10.5281/zenodo.16899501).

**Key takeaway**: Look at Sections 3 (dataset construction) for agent identification methodology, Section 4 (research opportunities) for the five-domain analysis framework, and the 14 data tables for replicable metric computation on your own repositories.