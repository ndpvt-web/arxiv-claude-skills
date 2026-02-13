---
name: "will-it-survive-deciphering"
description: "Analyze the survival and maintenance fate of AI-generated code in repositories using survival analysis techniques from Rahman & Shihab (2026). Assess whether AI-authored code is durable or disposable, classify modification types, and predict which code units are modification-prone. Trigger phrases: 'analyze AI code survival', 'is this AI code disposable', 'code survival analysis', 'predict code modification risk', 'compare AI vs human code durability', 'audit AI-generated code maintenance burden'."
---

# Will It Survive: Deciphering the Fate of AI-Generated Code

This skill enables Claude to perform survival analysis on AI-generated versus human-authored code within git repositories. Using the methodology from Rahman & Shihab (2026), it tracks code units (lines and hunks) from their birth (PR merge) through subsequent modifications, classifies why modifications happen (corrective, adaptive, perfective, preventive), and predicts which code is most likely to require future changes. The core finding: AI-agent-authored code actually survives longer than human code (15.8% lower hazard of modification), but shows elevated corrective (bug-fix) modification rates -- meaning it breaks less often overall, but when it does change, it's more likely a fix.

## When to Use

- When a user asks to assess the long-term maintainability of AI-generated code in their repository
- When auditing a codebase that mixes AI-agent PRs (Copilot, Cursor, Devin, Claude Code, Codex) with human contributions
- When a user wants to identify which AI-generated files or lines are most likely to need future modification
- When building a code review policy that differentiates review depth based on authorship origin
- When a user asks "is AI-generated code disposable?" or wants data to answer that question for their project
- When predicting maintenance burden for code produced by AI coding agents
- When classifying past modifications to understand whether changes were bug fixes, feature adaptations, or refactors

## Key Technique

**Survival analysis applied to code units.** The paper treats each line of code as a subject in a clinical-trial-style survival study. A line is "born" when its parent PR merges into the main branch (t=0) and "dies" when any subsequent commit modifies it (t>0). Lines still unmodified at the observation cutoff are right-censored. This framework yields Kaplan-Meier survival curves comparing AI vs. human code, and Cox Proportional Hazards regression quantifies the effect of authorship on modification risk while controlling for confounders (PR churn, files changed, repo size, contributor count).

**Modification intent classification via commit messages.** Rather than treating all changes equally, the method classifies each modification using Swanson's maintenance taxonomy by keyword-matching commit messages: corrective (fix, bug, error, crash, patch), perfective (refactor, optimize, enhance, feat, add), adaptive (chore, bump, update, upgrade, dependency), and preventive (security, test, coverage, vulnerability). This reveals that AI code's slightly higher corrective rate (26.3% vs 23.0%) is offset by lower adaptive rates, and the overall effect size is small (Cramer's V = 0.116).

**Predictive modeling for modification-prone code.** Textual features (bag-of-words over identifiers and API names, unigrams through trigrams) feed an XGBoost classifier to identify which files will be modified (AUC-ROC = 0.671). Process-level features (commit velocity, file modification frequency, file age, contributor acceptance rate, backlog size) attempt to predict *when* modification occurs, though this remains challenging (Macro F1 = 0.285) because timing depends on organizational dynamics, not code quality alone.

## Step-by-Step Workflow

1. **Identify AI-authored commits in the repository.** Parse `git log` for known bot signatures, agent metadata, or PR labels. Look for author emails matching patterns like `*[bot]@*`, `copilot`, `devin`, `cursor`, or `claude`. Alternatively, accept a user-provided list of PR numbers or commit SHAs known to be AI-generated.

2. **Extract code units at line granularity.** For each merged PR (AI or human), run `git blame` on affected source files to track individual lines. Record birth timestamp (merge date), authorship (agent vs. human), file path, and the line content. Filter to source code only (`.py`, `.js`, `.ts`, `.java`, `.cpp`, `.rs`, `.go`, etc.) -- exclude config, docs, and generated files.

3. **Track modification events over time.** Walk the commit history after each PR merge. For every subsequent commit that touches a tracked line (detected via `git blame` SHA changes), record: modification timestamp, modifying commit SHA, and time-to-modification in days. Lines unmodified at the analysis cutoff are marked as right-censored.

4. **Compute survival statistics.** Calculate the Kaplan-Meier survival function S(t) for AI-authored and human-authored code separately. Report median survival time, death rates (% of lines modified), and run a log-rank test to compare the two curves. Use the `lifelines` Python library or equivalent.

5. **Fit a Cox Proportional Hazards model.** Define the hazard function h(t|X) = h0(t) * exp(beta1 * is_agent + beta * X) where covariates X include PR churn (lines added+deleted), files changed, repo stars, and contributor count. Report the hazard ratio for `is_agent` with 95% confidence interval and p-value. Check proportional hazards assumption via Schoenfeld residuals.

6. **Classify modification intent from commit messages.** For each modification event, keyword-match the commit message against Swanson's taxonomy with priority ordering: corrective > adaptive > perfective > preventive. Keywords: corrective={fix, bug, error, issue, crash, patch, resolve, hotfix, defect, regression}, perfective={refactor, clean, optimize, improve, enhance, feat, add, new, implement}, adaptive={chore, bump, update, upgrade, merge, dependency, build, config}, preventive={security, test, coverage, vulnerability}. Compute a chi-square test and Cramer's V to measure association strength between authorship and modification type.

7. **Build a modification-risk predictor.** Extract bag-of-words features from source code (CountVectorizer with max_features=1000, min_df=5, max_df=0.90), filtering out syntax tokens and retaining identifiers and API names. Train an XGBoost classifier with repository-stratified k-fold cross-validation and SMOTE for class imbalance. Report AUC-ROC.

8. **Generate an actionable report.** Summarize: overall survival comparison, hazard ratio, modification type breakdown by authorship, top 20 most modification-prone files (ranked by predicted risk), and concrete review recommendations. Flag files with high corrective modification likelihood for deeper review.

9. **Provide organizational recommendations.** Based on findings, advise on: review depth allocation (AI code may need less review for stability, more review for correctness), monitoring dashboards for AI code health, and which agents produce the most durable code (per-agent variation exceeds the agent-human gap).

## Concrete Examples

**Example 1: Repository-level AI code survival audit**

User: "I want to know if the AI-generated PRs in our repo are producing throwaway code. We use Copilot and Cursor. Can you analyze our last 6 months?"

Approach:
1. Parse `git log --since="6 months ago" --format="%H %ae %s"` to identify commits by Copilot/Cursor (author email patterns, PR labels, or co-author trailers)
2. Extract all lines born from AI-merged PRs using `git blame --porcelain`
3. Track which of those lines have been subsequently modified
4. Compute survival curves and hazard ratios

Output:
```
## AI Code Survival Report (last 6 months)

### Summary
- AI-authored lines: 12,847 | Human-authored lines: 45,203
- AI modification rate: 41.2% | Human modification rate: 56.8%
- Hazard ratio (AI vs human): 0.78 (95% CI: 0.72-0.84, p < 0.001)
  → AI code has 22% lower risk of modification

### Modification Intent Breakdown
| Type        | AI Code | Human Code |
|-------------|---------|------------|
| Corrective  | 28.1%   | 22.4%      |
| Perfective  | 31.5%   | 33.2%      |
| Adaptive    | 24.0%   | 30.8%      |
| Preventive  | 16.4%   | 13.6%      |

### Interpretation
AI-generated code survives longer overall, but when it IS modified,
it's more often a bug fix (+5.7pp corrective). This suggests AI code
is syntactically stable but may contain subtle logic errors.

### Recommendation
- Increase correctness-focused review for AI PRs (edge cases, error handling)
- Reduce style/structure review burden (AI code is less likely to need refactoring)
- Monitor the 10 files below with highest predicted modification risk
```

**Example 2: Identify modification-prone AI-generated files**

User: "Which of our AI-generated files are most likely to need changes soon?"

Approach:
1. Collect all files with >50% AI-authored lines
2. Extract textual features (identifiers, API names) from each file
3. Compute process features (file age, modification frequency, recent commit velocity)
4. Score each file with the modification-risk model

Output:
```
## High-Risk AI-Generated Files

Rank | File                          | Risk Score | Top Risk Signal
-----|-------------------------------|------------|------------------
  1  | src/auth/oauth_handler.py     | 0.89       | High corrective keyword density
  2  | lib/api/payment_client.ts     | 0.84       | Recently born + high project velocity
  3  | services/data_pipeline.go     | 0.77       | Complex API surface (18 unique APIs)
  4  | utils/cache_manager.rs        | 0.71       | Prior modification frequency: 4x/month

Action: Prioritize review of files 1-2 before next release.
File 1 uses deprecated OAuth patterns likely to trigger corrective fixes.
File 3 has high API diversity suggesting integration fragility.
```

**Example 3: Classifying past modifications for maintenance analysis**

User: "Categorize the last 500 commits that modified AI-generated code by maintenance type."

Approach:
1. Identify commits that touched AI-authored lines (via blame tracking)
2. Extract commit messages
3. Apply keyword classification with priority ordering
4. Aggregate and report

Output:
```
## Modification Type Analysis (500 commits touching AI code)

Corrective (bug fixes):    134 (26.8%)  ████████████▌
Perfective (improvements): 156 (31.2%)  ██████████████▊
Adaptive (env updates):    128 (25.6%)  ████████████
Preventive (security/test): 62 (12.4%)  █████▊
Unclassified:               20 (4.0%)   █▊

Chi-square vs human baseline: χ² = 14.3, p = 0.006
Cramér's V = 0.09 (small effect)

Notable: Corrective rate is +3.8pp above human baseline.
Top corrective patterns: "fix null handling" (12x), "fix race condition" (8x),
"fix off-by-one" (6x) -- suggesting AI code has systematic edge-case gaps.
```

## Best Practices

- **Do:** Track at line granularity, not file granularity. Files contain mixed authorship, which confounds survival analysis. Line-level `git blame` attribution is the reliable unit.
- **Do:** Control for confounders in the Cox model. Raw survival differences may reflect project characteristics (large active repos churn more), not code quality. Always include PR churn, repo size, and contributor count as covariates.
- **Do:** Use priority-ordered keyword matching for commit classification. A commit message like "fix: add new error handler" matches both corrective and perfective -- corrective should win because it reflects the primary intent.
- **Do:** Stratify results by agent. Per-agent variation exceeds the overall agent-vs-human gap. Copilot code may behave very differently from Devin code.
- **Avoid:** Treating all modifications as equal. A corrective fix to an AI-generated line is qualitatively different from an adaptive update. Always classify modification intent.
- **Avoid:** Predicting *when* code will be modified based solely on code features. The paper shows timing depends on organizational dynamics (commit velocity, backlog size), not textual properties. Process features outperform code features for temporal prediction but still achieve only Macro F1 = 0.285.
- **Avoid:** Drawing conclusions from small samples. The paper used 200K+ code units across 201 projects. A single repository with 50 AI-authored lines will not produce statistically meaningful survival curves.

## Error Handling

- **Insufficient AI-authored code:** If the repo has fewer than ~500 AI-authored lines, survival analysis will lack statistical power. Warn the user and suggest aggregating across multiple repositories or using file-level analysis as a fallback.
- **Ambiguous authorship:** Co-authored commits (human + AI pair programming) create attribution uncertainty. Default to labeling lines as AI-authored if the primary commit author is an AI agent. Flag co-authored lines separately in the report.
- **Commit message classification failures:** Conventional commit prefixes (`fix:`, `feat:`) improve classification accuracy. If the repo uses unstructured commit messages, expect 15-25% unclassified modifications. Fall back to diff-based heuristics (e.g., small single-line changes in existing logic suggest corrective).
- **Git blame limitations:** Force-pushes, rebases, and squash merges can break blame chains. Use `git blame -C -C` to follow code across renames and copies. If blame history is broken, fall back to commit-level analysis.
- **Proportional hazards violation:** Large datasets almost always violate the PH assumption (Schoenfeld residual p < 0.05). This is expected and acceptable per the paper's methodology. Report the violation but proceed with the Cox model -- the hazard ratio remains interpretable as an average effect.

## Limitations

- The methodology identifies *correlation* between authorship and survival, not causation. AI-generated code may survive longer because it targets simpler, more stable parts of codebases.
- Keyword-based commit classification is coarse. Commits with vague messages ("update code", "changes") will be misclassified or unclassified. This method works best on repos that follow conventional commits.
- Temporal prediction (when code will be modified) remains an open problem. The best model achieves Macro F1 = 0.285, only 14% above random baseline. Do not rely on time-to-modification predictions for planning.
- The paper's dataset covers five specific AI agents (Copilot, Cursor, Devin, Claude Code, Codex). Results may not generalize to other code generation tools or to non-open-source repositories with different review practices.
- Survival analysis assumes that "not being modified" equates to quality. In practice, dead code also survives indefinitely. Combine survival analysis with code coverage and usage metrics for a complete picture.

## Reference

Rahman, M. & Shihab, E. (2026). *Will It Survive? Deciphering the Fate of AI-Generated Code in Open Source.* arXiv:2601.16809v1. https://arxiv.org/abs/2601.16809v1

Key takeaway: AI-agent-authored code has a 15.8% lower hazard of modification (HR=0.842) than human code, contradicting the "disposable code" hypothesis. However, when AI code IS modified, it's disproportionately corrective (bug fixes), and the organizational context -- not the code itself -- determines modification timing.