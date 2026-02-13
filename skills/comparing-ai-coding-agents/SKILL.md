---
name: "comparing-ai-coding-agents"
description: >
  Analyze AI coding agent PR datasets using task-stratified acceptance rate methodology.
  Classify PRs into 9 task categories (feat, fix, docs, chore, refactor, test, ci, build, style),
  compute per-agent acceptance rates, run stratified chi-square tests with Bonferroni correction,
  and perform temporal trend analysis. Trigger phrases: "compare AI coding agents",
  "analyze PR acceptance rates", "stratified PR analysis", "which AI agent is best for",
  "task-type acceptance rates", "AI agent benchmark pipeline"
---

# Comparing AI Coding Agents: Task-Stratified PR Acceptance Analysis

This skill enables Claude to build data pipelines that evaluate and compare AI coding agents by stratifying pull request outcomes across task categories. Based on the methodology from Pinna et al. (MSR'26), it applies a three-stage analysis: (1) classify PRs into semantic task types, (2) compute acceptance rates stratified by task and agent, and (3) run statistical tests (chi-square with Bonferroni correction, Fisher's exact test for small samples, linear regression for temporal trends). The key finding driving this approach is that **PR task type is a stronger predictor of acceptance than agent identity** -- the 16-point gap between documentation (82.1%) and feature PRs (66.1%) exceeds typical inter-agent variance.

## When to Use

- When the user wants to analyze a dataset of AI-generated pull requests and compare agent performance
- When building a pipeline to evaluate which AI coding agent to use for specific task types (fixes, features, docs, tests)
- When the user asks "which AI agent should I use for bug fixes vs. new features?"
- When performing empirical software engineering analysis on PR acceptance/rejection data
- When the user wants to replicate or extend the AIDev dataset analysis methodology
- When building dashboards or reports that stratify CI/CD metrics by task category and tool
- When running statistical comparisons (chi-square, Fisher's exact) on categorical software engineering data

## Key Technique

The core insight is **task-stratified comparison** rather than naive aggregate comparison. Comparing AI agents by overall acceptance rate conflates the effect of what tasks each agent handles with how well it handles them. An agent that mostly generates documentation PRs will appear better than one that tackles complex features, even if the latter is superior within each category. The paper's methodology controls for this by computing acceptance rates within each of 9 task categories independently, then running pairwise statistical tests at each stratum.

The 9 task categories follow conventional commit semantics: `feat` (new features, 66.1%), `fix` (bug fixes, 66.0%), `docs` (documentation, 82.1%), `chore` (maintenance, 84.0%), `refactor` (restructuring, 71.2%), `test` (testing, 61.5%), `ci` (CI/CD pipeline, 75.0%), `build` (build system, 72.5%), and `style` (formatting, 78.1%). Classification is done via commit message prefixes or LLM-based labeling when prefixes are absent.

Statistical rigor comes from three layers: (1) Pearson's chi-square test of independence for each agent-pair-task-type combination (64 total comparisons across 10 agent pairs x 9 task types), (2) Bonferroni correction setting alpha at 0.05/64 ~ 0.00078 to control family-wise error rate, and (3) Fisher's exact test substituted when any expected cell frequency falls below 5. Effect sizes are measured with the phi coefficient (phi < 0.1 negligible, 0.1-0.3 small, 0.3-0.5 medium, >= 0.5 large). Temporal trends use linear regression of weekly acceptance rates with LOESS smoothing (fraction=0.5) to capture non-linear evolution.

## Step-by-Step Workflow

1. **Ingest and filter the PR dataset.** Load PR records with fields: `agent`, `status` (merged/closed), `task_type`, `created_at`, `additions`, `deletions`, `files_changed`, `num_reviews`, `num_comments`. Filter to only closed PRs from repositories with permissive licenses (MIT/Apache-2.0) that received at least one review or comment from a non-creator.

2. **Classify PRs into the 9 task categories.** Parse commit message prefixes for conventional commit labels (`feat:`, `fix:`, `docs:`, etc.). For PRs without conventional prefixes, use an LLM classifier or keyword heuristic on the PR title and description. Map to the canonical set: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `ci`, `build`, `style`.

3. **Compute per-agent, per-task acceptance rates.** Group by `(agent, task_type)` and calculate `acceptance_rate = merged_count / total_count`. Also compute global per-agent and per-task-type rates. Build a contingency table (agents x task types) of acceptance rates.

4. **Apply minimum sample thresholds.** Exclude any agent-task combination with fewer than 10 PRs from statistical testing. Flag combinations with 10-30 PRs as low-confidence. Record sample sizes alongside all rates.

5. **Run pairwise stratified chi-square tests.** For each task type, construct a 2x2 contingency table for every agent pair (merged/not-merged x agent-A/agent-B). Apply Pearson's chi-square test. When any expected cell count < 5, substitute Fisher's exact test. Record p-value and phi coefficient for each comparison.

6. **Apply Bonferroni correction.** Calculate the corrected significance threshold: alpha_corrected = 0.05 / (number_of_comparisons). Only flag results with p < alpha_corrected as statistically significant. Report both raw and corrected p-values.

7. **Perform temporal trend analysis.** Aggregate acceptance rates into weekly bins per agent. Fit a linear regression (acceptance_rate ~ week_number) to estimate the slope (weekly change in percentage points) and R-squared. Apply LOESS smoothing with fraction=0.5 for visualization of non-linear trends.

8. **Generate the stratified comparison report.** Produce a summary table showing: best agent per task type, acceptance rate, sample size, and whether advantages are statistically significant. Include a heatmap of agent x task_type acceptance rates.

9. **Formulate actionable recommendations.** Based on the stratified results, recommend which agent to use for each task type. Flag task types where no significant inter-agent difference exists (meaning agent choice doesn't matter for that category).

10. **Validate with sensitivity checks.** Re-run the analysis excluding repositories with fewer than 5 PRs to check for repository-level confounds. Report whether rankings change under stricter filtering.

## Concrete Examples

**Example 1: Building a PR acceptance analysis pipeline**

User: "I have a CSV of 5,000 AI-generated PRs with columns agent, status, task_type, created_at. Compare the agents."

Approach:
1. Load the CSV with pandas, filter to rows where status is in {merged, closed}
2. Classify task_type if not already present (parse commit prefixes)
3. Build the stratified acceptance rate table
4. Run all pairwise chi-square tests with Bonferroni correction

Output:
```python
import pandas as pd
from scipy.stats import chi2_contingency, fisher_exact
import itertools

df = pd.read_csv("prs.csv")
df = df[df["status"].isin(["merged", "closed"])]
df["accepted"] = (df["status"] == "merged").astype(int)

TASK_TYPES = ["feat", "fix", "docs", "chore", "refactor", "test", "ci", "build", "style"]
agents = df["agent"].unique()
agent_pairs = list(itertools.combinations(agents, 2))
n_comparisons = len(agent_pairs) * len(TASK_TYPES)
alpha_corrected = 0.05 / n_comparisons

results = []
for task in TASK_TYPES:
    task_df = df[df["task_type"] == task]
    for a1, a2 in agent_pairs:
        g1 = task_df[task_df["agent"] == a1]["accepted"]
        g2 = task_df[task_df["agent"] == a2]["accepted"]
        if len(g1) < 10 or len(g2) < 10:
            continue
        table = pd.crosstab(
            task_df[task_df["agent"].isin([a1, a2])]["agent"],
            task_df[task_df["agent"].isin([a1, a2])]["accepted"]
        )
        expected_min = chi2_contingency(table)[3].min()
        if expected_min < 5:
            _, p = fisher_exact(table)
            test_used = "fisher"
        else:
            chi2, p, _, _ = chi2_contingency(table, correction=False)
            test_used = "chi2"
        n = len(g1) + len(g2)
        phi = (chi2 / n) ** 0.5 if test_used == "chi2" else None
        results.append({
            "task": task, "agent_1": a1, "agent_2": a2,
            "rate_1": g1.mean(), "rate_2": g2.mean(),
            "p_value": p, "significant": p < alpha_corrected,
            "phi": phi, "test": test_used
        })

results_df = pd.DataFrame(results)
print(results_df[results_df["significant"]].to_string(index=False))
```

**Example 2: Recommending an agent for a specific task type**

User: "We're evaluating Cursor vs Copilot for our team. We mostly do bug fixes and tests. Which is better?"

Approach:
1. Reference the stratified benchmarks from the paper
2. Report task-specific acceptance rates with sample sizes
3. Note statistical significance of differences

Output:
```
Based on stratified PR acceptance analysis (Pinna et al., MSR'26, N=7,156):

| Task   | Cursor | Copilot | Significant? |
|--------|--------|---------|--------------|
| fix    | 80.4%  | ~68%    | Yes (p<0.001, phi=0.20) |
| test   | 77.8%  | ~62%    | Yes          |

Recommendation: For fix and test tasks, Cursor shows statistically
significant advantages over Copilot. The effect sizes are small-to-medium.

Caveat: Acceptance rate != code quality. Merged PRs may still carry
technical debt. Consider supplementing with post-merge defect tracking.
```

**Example 3: Temporal trend analysis for agent improvement**

User: "Has Devin been getting better over time? I want to see the trend."

Approach:
1. Aggregate weekly acceptance rates for Devin
2. Fit linear regression and LOESS smoothing
3. Report slope, R-squared, and confidence interval

Output:
```python
import numpy as np
from sklearn.linear_model import LinearRegression
import statsmodels.api as sm

# Aggregate to weekly acceptance rates
weekly = (df[df["agent"] == "devin"]
          .set_index("created_at")
          .resample("W")["accepted"]
          .agg(["mean", "count"])
          .dropna())
weekly = weekly[weekly["count"] >= 5]  # minimum weekly sample

X = np.arange(len(weekly)).reshape(-1, 1)
y = weekly["mean"].values

reg = LinearRegression().fit(X, y)
print(f"Slope: {reg.coef_[0]*100:.2f}% per week")
print(f"R-squared: {reg.score(X, y):.2f}")
# Expected output for Devin: +0.77%/week, R²=0.34

# LOESS for non-linear visualization
lowess = sm.nonparametric.lowess(y, X.flatten(), frac=0.5)
```

## Best Practices

- **Do:** Always stratify by task type before comparing agents. Aggregate rates are misleading when agents handle different task mixes (Simpson's paradox).
- **Do:** Apply Bonferroni correction when running multiple pairwise comparisons. With 10 agent pairs across 9 task types, you have 64+ comparisons -- without correction, you will find false positives.
- **Do:** Report phi coefficient alongside p-values. Statistical significance with tiny effect size (phi < 0.1) is not practically meaningful.
- **Do:** Use Fisher's exact test when any expected cell frequency is below 5. Chi-square approximations break down with small samples.
- **Avoid:** Comparing agents by global acceptance rate alone. The paper shows task-type explains more variance than agent identity.
- **Avoid:** Treating PR acceptance as a proxy for code quality. The paper explicitly warns that merged PRs may contain latent bugs or technical debt. Supplement with post-merge defect analysis.
- **Avoid:** Drawing conclusions from agent-task combinations with fewer than 10 PRs. Small samples produce unreliable rates.

## Error Handling

- **Sparse contingency tables:** When an agent-task cell has 0 merged or 0 rejected PRs, chi-square is undefined. Fall back to Fisher's exact test or skip the comparison and note it as "insufficient data."
- **Unbalanced agent samples:** If one agent has 3,000 PRs and another has 50, statistical tests will have very different power. Report confidence intervals alongside point estimates to surface this asymmetry.
- **Missing task classifications:** When PRs lack conventional commit prefixes, the LLM classifier may misclassify. Validate by sampling 50-100 classified PRs manually. Expect ~85-90% classification accuracy; report the validation rate.
- **Temporal confounds:** If different agents were active during different time periods, trends may reflect ecosystem changes (e.g., repository difficulty shifting) rather than agent improvement. Control for repository characteristics when possible.
- **Multiple testing inflation:** With many agent-task comparisons, even Bonferroni can be conservative. Consider Benjamini-Hochberg FDR correction as an alternative if you want more statistical power at the cost of slightly higher false positive risk.

## Limitations

- **Acceptance != quality.** PR acceptance rate measures reviewer approval, not functional correctness, security, or maintainability. An agent with high acceptance may produce code that passes review but introduces subtle bugs.
- **Dataset bias.** The AIDev dataset covers open-source repositories with 100+ stars. Results may not transfer to private codebases, enterprise environments, or small projects with different review standards.
- **Task classification noise.** LLM-based task labeling introduces classification error. Misclassified PRs (e.g., a refactor labeled as a feature) dilute stratified analysis accuracy.
- **Temporal coverage.** The study spans roughly 32 weeks (mid-2025). Agent capabilities change rapidly; these results represent a snapshot and will age.
- **Confounding variables.** Repository difficulty, reviewer strictness, PR size, and programming language are not fully controlled. Observed acceptance differences may partly reflect task selection effects rather than agent capability.
- **Sample size imbalance.** Some agent-task combinations have very few observations (Claude Code had limited data in several categories), reducing the reliability of rate estimates for those cells.

## Reference

Pinna, G., Gong, J., Williams, D., & Sarro, F. (2026). *Comparing AI Coding Agents: A Task-Stratified Analysis of Pull Request Acceptance.* MSR'26 Mining Challenge Track. [arXiv:2602.08915v1](https://arxiv.org/abs/2602.08915v1)

Key sections to consult: Table 2 (acceptance rates by agent and task type), Section 4.2 (stratified chi-square methodology), Section 4.1 (temporal regression analysis), and the replication package for raw data and scripts.