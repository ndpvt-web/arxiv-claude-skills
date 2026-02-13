---
name: "refuge-feature-generation-prediction"
description: "Automated feature engineering for prediction tasks on relational databases using a multi-agent LLM pipeline. Generates, filters, and validates SQL/pandas features from multi-table schemas. Use when: 'generate features from my database for prediction', 'engineer features across related tables', 'automate feature generation for classification/regression on relational data', 'build predictive features from a star schema', 'improve model accuracy with relational feature engineering', 'create aggregation features from joined tables'."
---

# ReFuGe: LLM-Driven Feature Generation for Relational Database Prediction

This skill enables Claude to act as an automated feature engineering system for prediction tasks on relational databases (RDBs). Based on the ReFuGe framework (ACM WWW 2026), it decomposes the problem into three specialized agent stages: **schema selection** (identify relevant tables and columns), **feature generation** (synthesize aggregation-based features via SQL or pandas), and **feature filtering** (retain only features that improve predictive performance). The pipeline runs iteratively — each round generates candidate features, validates them against a held-out set, and feeds performance results back to guide the next round of generation until accuracy converges.

## When to Use

- When the user has a multi-table relational database and wants to predict a target variable (churn, clicks, classification, regression) from it
- When the user asks to "generate features" or "engineer features" from a database schema with foreign-key relationships
- When the user wants to improve an ML model's accuracy by automatically discovering aggregation features (COUNT, SUM, AVG, MAX, MIN, STD) across joined tables
- When the user provides a database schema and asks which tables/columns are most relevant for a prediction task
- When the user needs to prevent data leakage while building temporal features from transactional tables
- When the user wants to go beyond manual feature engineering on a star or snowflake schema

## Key Technique

**The core insight of ReFuGe** is that feature engineering on relational databases can be decomposed into three distinct reasoning tasks — schema understanding, feature synthesis, and feature evaluation — each handled by a specialized LLM agent. This outperforms monolithic approaches because the combinatorial feature space across multiple joined tables is too large for a single prompt to explore effectively.

**The three-agent pipeline works as follows:** (1) A *schema selection agent* receives the full database schema (tables, columns, types, primary/foreign keys) and the prediction task description, then reasons about which tables and columns are semantically relevant and outputs explicit join paths from the target entity table. (2) A *feature generation agent* (run as 3 parallel instances for diversity) each proposes up to 3 aggregation-based features using the selected schema — features are expressed as Python/pandas functions that aggregate related table rows per entity (e.g., "count of orders in last 30 days", "average transaction amount"). (3) A *feature filtering agent* (voter/judge) selects the top 3 from the 9 candidates based on predictive value, feasibility, and non-redundancy, then synthesizes executable code. Features are validated by training a LightGBM model and checking if AUC/RMSE improves.

**The iterative feedback loop** is critical. After each round, accepted features and their performance impact are fed back into the generation prompt as conversational history. Features that degraded performance are reported with their metrics ("Feature X caused AUC to drop from 0.82 to 0.79"), guiding the agents away from similar features in subsequent rounds. This typically converges in 3-8 iterations.

## Step-by-Step Workflow

1. **Collect the database schema.** Extract all table names, column names with data types, primary keys, foreign keys, and join relationships. Format as a structured text block or dictionary the LLM can reason about. Identify the target entity table (the table whose rows are prediction targets) and the target column.

2. **Define the prediction task explicitly.** State the task type (binary classification, multi-class, regression), the target variable, the entity identifier column, and any temporal constraints (e.g., "predict churn using data before cutoff date X"). This grounds all subsequent reasoning.

3. **Run schema selection.** Prompt the LLM with the full schema and task description. Ask it to: (a) select helper tables relevant to the prediction task, (b) provide explicit join paths from the target entity table using PK/FK relationships, and (c) select specific columns from those tables that carry predictive signal. Output: a reduced schema with join paths.

4. **Generate candidate features in parallel.** Using the reduced schema, prompt 3 independent LLM calls to each propose up to 3 aggregation features. Each feature must be described as a natural-language plan specifying: the source table(s), the aggregation function (COUNT, SUM, AVG, MAX, MIN, STD), any filters or time windows, and the join path back to the entity table. This yields 9 diverse candidates.

5. **Filter candidates via voting.** Present all 9 candidate feature descriptions to a judge LLM. Ask it to select the top 3 that are: (a) most predictive for the task, (b) feasible to implement from the given schema, and (c) not redundant with each other or with previously accepted features.

6. **Synthesize executable feature code.** Prompt the LLM to generate Python functions implementing each selected feature. Each function must: accept the database tables as DataFrames, apply temporal cutoff filters to prevent data leakage, perform the aggregation, and return a DataFrame with exactly two columns: `[entity_col, feature_name]` with one row per entity.

7. **Execute and validate features.** Run the generated code against train/validation/test splits. Train a LightGBM model (or the user's preferred model) on existing features plus each new feature. Compute the evaluation metric (AUC for classification, RMSE for regression) on the validation set.

8. **Accept or reject based on metric improvement.** Keep features where `new_metric >= previous_best_metric`. Record which features were accepted and which were rejected with their metric deltas.

9. **Feed results back and iterate.** Append the round's results (accepted features, rejected features with reasons, current best metric) to the conversational context. Return to step 4 for the next iteration. Continue until the metric stops improving for 2 consecutive rounds or a maximum iteration count is reached.

10. **Produce the final feature set.** Output all accepted feature functions as a consolidated Python module. Merge all features into a single entity-level DataFrame for downstream model training.

## Concrete Examples

**Example 1: E-commerce churn prediction**

```
User: I have a database with tables: customers (customer_id, signup_date, plan_type),
orders (order_id, customer_id, order_date, amount, status), and
support_tickets (ticket_id, customer_id, created_at, category, resolved).
I want to predict customer churn (binary). Help me generate features.

Approach:
1. Schema selection: All three tables are relevant.
   Join paths: orders.customer_id -> customers.customer_id,
               support_tickets.customer_id -> customers.customer_id

2. Feature generation round 1 (9 candidates from 3 agents, top 3 selected):
   - feat1: COUNT of orders per customer in last 90 days
   - feat2: AVG order amount per customer (all time)
   - feat3: COUNT of unresolved support tickets per customer

3. Code synthesis for feat1:
   def compute_feat1(customers_df, orders_df, cutoff_date):
       recent = orders_df[orders_df['order_date'] <= cutoff_date]
       recent = recent[recent['order_date'] >= cutoff_date - pd.Timedelta(days=90)]
       agg = recent.groupby('customer_id')['order_id'].count().reset_index()
       agg.columns = ['customer_id', 'order_count_90d']
       return agg

4. Validation: LightGBM AUC improves from 0.72 (baseline) to 0.78. All 3 accepted.

5. Round 2 generates: STD of order amounts, days since last order,
   ratio of resolved to total tickets. AUC improves to 0.81.

6. Round 3: no improvement. Stop. Final feature set: 6 features.
```

**Example 2: Event repeat prediction from a multi-table schema**

```
User: My database has: users (user_id, age, city), events (event_id, name,
category, date), and attendance (user_id, event_id, rating, timestamp).
Predict whether a user will attend the same event category again.

Approach:
1. Schema selection: All tables relevant. Target entity: users.
   Join: attendance.user_id -> users.user_id,
         attendance.event_id -> events.event_id

2. Round 1 features:
   - feat1: COUNT of distinct event categories attended per user
   - feat2: AVG rating given by user across all events
   - feat3: COUNT of repeat-category attendances per user (historical)

3. Key leakage prevention: All aggregations filter
   attendance.timestamp <= cutoff_date for each split.

4. Validation: feat1 and feat3 improve metric; feat2 does not.
   Feedback: "feat2 (avg rating) caused AUC drop from 0.68 to 0.66 —
   ratings may not distinguish repeat behavior."

5. Round 2 targets gap: generates time-decay weighted category counts,
   max attendance streak length, days since last event.
   Two accepted, one rejected.

6. Final: 5 features, AUC 0.74 (from 0.65 baseline).
```

**Example 3: Generating features for a provided schema without a running database**

```
User: I don't have the data loaded yet, but here's my schema. Can you
propose features for predicting delivery delay (regression)?

Tables: warehouses, products, orders, shipments, carriers

Approach:
1. Schema selection: orders (target), shipments, carriers most relevant.
   warehouses relevant for location-based features.

2. Generate feature proposals (no validation possible without data):
   - AVG shipment transit time per carrier (shipments JOIN carriers)
   - COUNT of orders from same warehouse in last 7 days (congestion proxy)
   - Distance between warehouse and delivery ZIP (if coordinates available)
   - STD of carrier delivery times (reliability signal)
   - Ratio of delayed shipments per carrier (historical delay rate)

3. Output: Feature proposals with implementation templates as pandas
   functions, ready to run once data is loaded. Flag which features
   need validation and suggest LightGBM RMSE as the evaluation metric.
```

## Best Practices

- **Do:** Always enforce temporal cutoff filters in every feature function to prevent data leakage. Every aggregation must filter rows to `timestamp <= cutoff_date` for the relevant split.
- **Do:** Generate features in parallel with multiple independent LLM calls (3 is the sweet spot) to maximize diversity. A single call tends to produce correlated features.
- **Do:** Standardize feature output format to exactly `[entity_col, feature_name]` with one row per entity. This prevents merge explosions and ensures clean joins.
- **Do:** Always pass explicit suffixes to `pd.merge()` or pre-rename columns before merging to avoid ambiguous `_x`/`_y` column collisions.
- **Avoid:** Generating features that require information from the target variable or future data. The judge agent should explicitly check for leakage in each candidate.
- **Avoid:** Accepting features without validation. Even features that seem semantically strong can degrade performance due to noise, multicollinearity, or distribution shift. Always verify with a held-out metric.

## Error Handling

- **Generated code fails to execute:** Wrap each feature function in try/except. If a function raises an error (e.g., column not found, type mismatch), log the error, skip that feature, and report the failure to the LLM in the next iteration so it can correct the approach.
- **Feature produces NaN/null for most entities:** Fill missing values with 0 or the column median before validation. If more than 80% of values are null, reject the feature and feed back that the join path produced too few matches.
- **Metric does not improve across multiple rounds:** Check if the baseline model is already saturated on available signal. Consider expanding the schema selection to include previously excluded tables, or switching aggregation granularity (e.g., from all-time to windowed).
- **Merge produces duplicate rows:** The feature function is returning multiple rows per entity. Add a `.groupby(entity_col).first()` or re-examine the aggregation logic. This is a critical bug to catch before validation.
- **LLM generates non-aggregation features (e.g., raw column copies):** The prompt should explicitly constrain features to aggregation operations across related tables. Reject any feature that simply copies a column from the target table.

## Limitations

- **Requires a well-defined schema with foreign keys.** If the database lacks explicit PK/FK relationships, the schema selection agent cannot reliably determine join paths. You must provide or infer these manually.
- **Aggregation-centric.** ReFuGe focuses on COUNT/SUM/AVG/MAX/MIN/STD aggregations. It does not handle graph-based features, embedding-based features, or features requiring complex multi-hop reasoning across more than 2-3 joins.
- **Validation requires labeled data.** The iterative feedback loop needs a train/validation split with ground truth. For unsupervised or semi-supervised settings, the filtering stage cannot operate as designed.
- **LLM token cost scales with iterations.** Each round involves multiple LLM calls with growing conversational context. For databases with 20+ tables, the schema description alone can consume significant context. Budget 3-8 iterations.
- **Not a substitute for domain expertise on specialized schemas.** The LLM reasons from column names and types. Poorly named columns (e.g., `col1`, `col2`) or domain-specific encodings will produce low-quality features.

## Reference

- **Paper:** [ReFuGe: Feature Generation for Prediction Tasks on Relational Databases with LLM Agents](https://arxiv.org/abs/2601.17735v1) — Look for the three-agent decomposition (Section 3), the iterative feedback mechanism, and the ablation study showing that parallel diverse generation + voting outperforms single-agent generation.
- **Code:** [github.com/K-Kyungho/REFUGE](https://github.com/K-Kyungho/REFUGE) — Reference implementation using Claude API and RelBench benchmarks.