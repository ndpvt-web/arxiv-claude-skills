---
name: "hunt-instead-wait-evaluating"
description: >
  Autonomously explore databases and datasets to extract key insights without
  predefined queries, using investigatory intelligence (hypothesis-driven,
  goal-setting exploration) rather than executional intelligence (completing
  assigned tasks). Applies the Deep Data Research (DDR) methodology from
  Liu et al. 2026.

  Trigger phrases:
  - "Explore this database and find interesting insights"
  - "What patterns or anomalies exist in this dataset?"
  - "Investigate this data and tell me what's important"
  - "Do a deep dive analysis on this database"
  - "Hunt for insights in these tables"
  - "What can we learn from this data without me telling you what to look for?"
---

# Hunt Instead of Wait: Autonomous Deep Data Research

This skill enables Claude to conduct **investigatory** data analysis — autonomously
exploring unfamiliar databases or datasets to discover meaningful insights without
the user specifying what to look for. Rather than waiting for a precise query
(executional intelligence), Claude proactively formulates hypotheses, profiles data,
and iterates through exploration branches to surface findings the user didn't know
to ask about. This applies the Deep Data Research (DDR) framework, which treats
data analysis as open-ended investigation rather than query answering.

## When to Use

- When a user provides a database, CSV, or dataset and says "explore this" or "what's interesting here?" without a specific question
- When asked to do open-ended analysis on unfamiliar data with no predefined analytical plan
- When a user wants to understand what patterns, anomalies, or stories exist in their data
- When conducting exploratory data analysis (EDA) that needs to go beyond summary statistics into hypothesis-driven investigation
- When the user has a broad research question (e.g., "What affects customer churn?") but no specific queries in mind
- When handed a multi-table database and asked to find relationships or insights across tables

## Key Technique: Investigatory Intelligence

The DDR framework distinguishes **investigatory intelligence** from **executional intelligence**. Executional intelligence answers "run this query" or "make this chart" — it completes assigned tasks. Investigatory intelligence is the capacity to autonomously set goals, formulate hypotheses, decide what to explore next, and synthesize discoveries into coherent insights. The core insight: effective autonomous data exploration is not just about better tooling (agent scaffolding) or bigger models (scaling), but about the **intrinsic exploration strategy** the agent employs.

The most effective strategy is **hypothesis-driven exploration with breadth-first profiling**. Rather than diving deep into the first interesting column, the agent first profiles the full schema, generates multiple candidate hypotheses, then allocates investigation effort across the most promising branches. Each investigation cycle refines the hypothesis set — early findings redirect subsequent exploration. This avoids two failure modes: (1) random wandering without a conceptual framework, and (2) premature deep-diving into a single analytical branch before understanding the data landscape.

Evaluation uses a **checklist-based** approach: insights are scored against a curated set of ground-truth findings. Partial credit is given for discovering subsets. This means the goal is coverage of substantive discoveries, not query optimization. The agent should aim for breadth of meaningful insights, with depth applied selectively to the most impactful findings.

## Step-by-Step Workflow

1. **Ingest and map the schema.** Read the full database schema, table definitions, column names, data types, and any documentation. For CSVs, read headers and sample rows. Build a mental model of what entities exist and how tables relate to each other (foreign keys, shared columns, temporal relationships).

2. **Profile the data landscape.** For each table, compute: row count, null rates per column, cardinality of categorical columns, range/distribution of numeric columns, and date ranges for temporal columns. This is breadth-first — resist deep analysis yet. The goal is a quick census of what the data contains.

3. **Formulate 5-8 candidate hypotheses.** Based on the schema and profile, generate explicit, testable hypotheses. Frame them as questions: "Do customers in region X have higher churn?" or "Is there a seasonal pattern in transaction volume?" Prioritize hypotheses that span multiple tables or involve non-obvious relationships. Write these down explicitly before proceeding.

4. **Rank hypotheses by expected insight value.** Estimate which hypotheses are most likely to yield actionable or surprising findings. Consider: data availability (are the relevant columns populated?), business relevance (would this matter to the user?), and analytical feasibility (can this be tested with the available data?). Select the top 3-4 for initial investigation.

5. **Investigate each hypothesis with targeted queries.** For each selected hypothesis, write and execute specific analytical queries or code. Use aggregations, groupings, correlations, and statistical tests as appropriate. Record the result of each investigation — confirmed, refuted, or inconclusive — along with the key numbers.

6. **Maintain an exploration log and avoid redundancy.** Track which columns, tables, and relationships have been examined. Before starting a new investigation branch, check whether it overlaps with prior work. This prevents the common failure of re-querying the same data in different forms.

7. **Refine and spawn new hypotheses from findings.** Use intermediate results to generate second-order hypotheses. If you discover seasonal patterns, ask "does the seasonality differ by customer segment?" If you find a correlation, investigate confounders. This iterative refinement is the core of investigatory intelligence.

8. **Synthesize findings into a structured report.** Group discoveries thematically. For each finding, state: (a) the insight in plain language, (b) the supporting evidence with specific numbers, (c) confidence level, and (d) suggested next steps or deeper questions. Lead with the most impactful or surprising discoveries.

9. **Flag anomalies and data quality issues.** Report unexpected nulls, outliers, impossible values, or broken relationships. These are often the most valuable discoveries in real-world data.

10. **Present limitations transparently.** State which hypotheses were not investigated due to data gaps, which findings need more data to confirm, and what the data cannot tell you.

## Concrete Examples

**Example 1: E-commerce Database Exploration**

User: "Here's our company database with tables for orders, customers, products, and returns. Explore it and tell me what's interesting."

Approach:
1. Read schema: `orders` (order_id, customer_id, product_id, amount, date, status), `customers` (customer_id, region, signup_date, plan_type), `products` (product_id, category, price, supplier), `returns` (return_id, order_id, reason, date)
2. Profile: 45K orders over 3 years, 8K customers across 5 regions, 12% null in returns.reason, 340 products in 15 categories
3. Hypotheses generated:
   - H1: Return rates vary significantly by product category
   - H2: Customer lifetime value differs by signup cohort
   - H3: Seasonal purchasing patterns exist and differ by region
   - H4: Premium plan customers have lower return rates
   - H5: Certain suppliers have disproportionate return rates
4. Investigate top hypotheses with SQL/pandas queries
5. Refine: H1 confirmed (electronics 22% return rate vs 6% average) -> spawn H1a: "Is the electronics return rate driven by a specific supplier?"

Output:
```
## Deep Data Research Findings

### 1. Electronics Return Crisis (High Impact)
Electronics have a 22% return rate vs 6% overall average. Drilling deeper,
supplier "TechWholesale" accounts for 68% of electronics returns despite
only 40% of electronics volume. Top return reason: "defective on arrival."
**Recommendation:** Audit TechWholesale quality controls immediately.

### 2. Cohort Retention Cliff (High Impact)
Customers who signed up in Q4 (holiday season) have 45% lower 12-month
retention than Q1-Q3 cohorts. Likely explanation: promotional signups with
low intent. Premium plan customers are immune to this effect.

### 3. Regional Seasonality Divergence (Medium Impact)
West Coast orders peak in March (+30%), while East Coast peaks in November
(+25%). National marketing campaigns may be suboptimal — consider regional
timing.

### 4. Data Quality Flag
returns.reason is null for 12% of records, concentrated in 2024-Q2.
Likely a logging bug deployed that quarter.

### Unexplored (insufficient data)
- Customer support interactions (no table available)
- Marketing channel attribution (no source tracking in orders)
```

**Example 2: Public Health Dataset Investigation**

User: "I have a CSV with 200K rows of hospital admission records. Columns include age, gender, diagnosis_code, admission_date, length_of_stay, insurance_type, readmission_30day, hospital_id. What can we learn?"

Approach:
1. Load and profile: check distributions, null rates, cardinalities
2. Hypotheses:
   - H1: Readmission rates vary by insurance type after controlling for diagnosis
   - H2: Length of stay has changed over time (trend analysis)
   - H3: Certain hospitals are outliers in readmission rates
   - H4: Age-diagnosis interaction affects length of stay
3. Execute analyses with pandas, compute effect sizes and significance
4. Iterate: H3 confirmed -> investigate whether outlier hospitals have different patient mixes or genuinely different outcomes

Output:
```
## Key Findings

### 1. Insurance-Readmission Disparity
Medicaid patients have 18.3% 30-day readmission vs 11.2% for private
insurance (p < 0.001). After controlling for diagnosis mix and age,
the gap narrows to 14.1% vs 11.8% — still significant (OR=1.22,
95% CI [1.15, 1.30]). Residual disparity suggests systemic access issues.

### 2. Three Outlier Hospitals
Hospitals H-047, H-112, H-203 have readmission rates >2 standard
deviations above mean. H-047 and H-112 have comparable patient
acuity to peers — suggesting operational issues. H-203 has a
disproportionate share of complex cardiac cases, explaining its rate.

### 3. Length of Stay Trend
Median LOS decreased from 4.2 days (2020) to 3.6 days (2025),
but variance increased — suggesting a split between faster routine
discharges and longer complex stays.

### Limitations
- No socioeconomic data beyond insurance type
- Diagnosis codes are ICD-10 top-level only — granularity is limited
- Hospital_id alone cannot distinguish facility quality from patient mix
```

**Example 3: Multi-Table Relational Database**

User: "Here's a SQLite database for a SaaS product. Tables: users, subscriptions, feature_usage, support_tickets, invoices. Find insights."

Approach:
1. Map relationships: users -> subscriptions (1:many), users -> feature_usage (1:many), users -> support_tickets (1:many), subscriptions -> invoices (1:1)
2. Profile all tables, noting that feature_usage has 2.1M rows across 47 features
3. Hypotheses:
   - H1: Feature usage patterns predict churn (subscription cancellation)
   - H2: Support ticket volume precedes cancellation
   - H3: Certain features are underused but correlate with retention
   - H4: Invoice payment delays predict churn
4. Build churn cohorts, compare feature usage distributions

Output:
```
## SaaS Intelligence Report

### 1. "Export to PDF" is a Retention Signal
Users who use the PDF export feature at least once per month have
3.2x lower churn rate (4.1% vs 13.2% annual). This feature is used
by only 23% of users. Consider onboarding nudges to drive adoption.

### 2. Support Ticket Escalation Pattern
Users who file 3+ tickets in a 30-day window churn at 41% within
90 days. The critical signal is the *acceleration* — going from
1 ticket/month to 3+/month is a stronger predictor than absolute count.

### 3. Zombie Subscriptions
8.4% of active subscriptions have zero feature_usage in the last
60 days. These users are paying but not engaged — high churn risk
and potential targets for re-engagement campaigns.
```

## Best Practices

**Do:**
- Always profile the full schema before formulating hypotheses. Premature deep-dives into the first interesting column are the most common failure mode.
- Write hypotheses explicitly before investigating them. This creates an exploration plan that prevents random wandering.
- Track what you've already investigated. Long-horizon exploration fails when the agent re-examines the same data from slightly different angles without realizing it.
- Report specific numbers, not vague observations. "22% return rate vs 6% average" is useful; "high return rate" is not.

**Avoid:**
- Do not exhaustively analyze every column. Breadth-first profiling should take minutes, not hours. Allocate deep investigation time to the highest-value hypotheses.
- Do not present raw query results as findings. Synthesize: what does this number mean for the user's business or research?
- Do not ignore data quality issues. Nulls, outliers, and impossible values are often the most actionable discoveries.
- Do not stop after the first round of hypotheses. The best insights come from second-order hypotheses spawned by initial findings.

## Error Handling

- **Schema too large to profile fully:** Prioritize tables with the most rows and the most foreign-key connections. Profile peripheral tables only if early hypotheses point to them.
- **Queries time out or return errors:** Simplify by sampling (TABLESAMPLE or random row selection). Report that findings are based on a sample and state the sample size.
- **No clear hypotheses emerge from profiling:** Fall back to statistical anomaly detection — compute z-scores for numeric columns, look for unexpected value distributions, check for temporal trends. Anomalies generate hypotheses.
- **Data is too uniform (no variance):** Report this as a finding. Homogeneous data is informative — it means the process generating it is highly constrained. Investigate whether the uniformity is real or an artifact of data collection.
- **Contradictory findings across tables:** Investigate the join conditions and data freshness. Contradictions often reveal data pipeline bugs or different update cadences.

## Limitations

- This approach requires **structured data** (databases, CSVs, dataframes). It does not apply to unstructured text corpora, images, or raw log files without preprocessing.
- Investigatory intelligence is compute-intensive. For very large databases (hundreds of tables, billions of rows), the profiling step alone may exceed context or time limits. Scope appropriately.
- The checklist-based evaluation from DDR-Bench assumes ground-truth insights exist. In truly novel data, there is no checklist — the agent must use its own judgment about what constitutes a meaningful finding.
- Hypothesis quality depends on domain knowledge. For highly specialized domains (genomics, quantitative finance), the agent may generate superficial hypotheses without domain context. Provide domain context when possible.
- This is not a replacement for rigorous statistical analysis. Findings from autonomous exploration are **hypotheses to validate**, not confirmed results. Treat them as starting points for deeper investigation.

## Reference

[Hunt Instead of Wait: Evaluating Deep Data Research on Large Language Models](https://arxiv.org/abs/2602.02039v1) — Liu et al., 2026. Focus on: Section 3 (DDR task definition and methodology), Section 4 (DDR-Bench evaluation framework), and Section 5 (analysis of intrinsic exploration strategies that distinguish effective investigatory intelligence from mere executional intelligence).