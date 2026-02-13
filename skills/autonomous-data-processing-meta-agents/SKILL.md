---
name: "autonomous-data-processing-meta-agents"
description: "Build self-managing data processing pipelines using hierarchical meta-agent orchestration. Decomposes complex data tasks into multi-phase plans, instantiates specialized ground-level agents (Reader, Profiler, Transformer, Validator, Joiner, etc.), and iteratively refines execution through progressive sampling and monitoring loops. Triggers: 'build a data pipeline', 'process and clean this dataset', 'create an ETL pipeline with agents', 'orchestrate data processing agents', 'autonomous data pipeline', 'meta-agent data processing'."
---

# Autonomous Data Processing using Meta-Agents (ADP-MA)

This skill enables Claude to construct, execute, and iteratively refine data processing pipelines through hierarchical agent orchestration. Rather than writing a monolithic script, Claude decomposes a data task into ordered phases, assigns each phase to specialized ground-level agents (Reader, Profiler, Transformer, Validator, Joiner, Aggregator, FeatureEngineer, etc.), validates each stage through progressive sampling (10 -> 100 -> 1000 -> full rows), and backtracks on failure. The result is a modular, self-documenting pipeline that handles messy real-world data reliably.

## When to Use

- When the user asks to build an end-to-end data processing pipeline from raw files to cleaned/transformed output
- When a dataset needs multi-step cleaning: imputation, deduplication, type standardization, outlier detection, date parsing, text normalization
- When the user needs to join, merge, or align schemas across multiple data sources
- When feature engineering is required: encoding, scaling, binning, datetime extraction
- When aggregation and analytics are needed: groupby, rolling statistics, pivots, percentiles
- When temporal or graph-structured data needs indexing, edge analysis, or subgraph extraction
- When the user says "automate this data workflow" or "make this pipeline self-correcting"
- When a data pipeline keeps failing and needs iterative debugging with progressive validation

## Key Technique

**Hierarchical Meta-Agent Orchestration.** ADP-MA separates reasoning about *what to do* (meta-agents) from *doing it* (ground-level agents). Three meta-agents operate serially with shared state: the **Orchestrator** analyzes input data and task specs to produce a minimal 1-3 phase plan; the **Architect** expands each phase into concrete substeps and selects agent types from a library; and the **Monitor** evaluates outputs after each substep using rule-based verdicts (continue, warn, pause, abort, retry) without additional LLM calls. This hierarchy prevents context rot -- each agent gets a focused context window instead of accumulating the entire conversation history.

**Progressive Sampling for Scalability.** Instead of running each agent on the full dataset immediately, ADP-MA validates through four escalating tiers: XS (10 rows) for syntax/logic checks, S (100 rows) for functional correctness, M (1000 rows) for edge cases, and FULL for production. On failure at any tier, the agent receives the full traceback and revises its code, retrying up to 3 times at the same tier before escalating. This catches bugs cheaply and avoids expensive full-dataset runs on broken logic.

**Two-Level Backtracking.** When a substep fails after retries, phase-level backtracking discards outputs and the Architect re-expands with alternative agent types. When a phase fails repeatedly (default: 2 failures), plan-level backtracking reverts to the Orchestrator for a revised high-level plan. Per-phase (2) and global (3) retry caps prevent infinite loops. A two-level critique loop validates plans before execution: Level-1 checks phase ordering, dependencies, and goal coverage; Level-2 checks agent type appropriateness and input/output schema compatibility.

## Step-by-Step Workflow

1. **Profile the input data.** Read the dataset(s) and extract schema, column types, row counts, null rates, basic statistics, and sample rows. Store this as a structured summary for downstream agents.

2. **Design a minimal multi-phase plan.** Based on the task specification and data profile, decompose the goal into 1-3 ordered phases. Each phase should have a clear objective, stated rationale, input/output schema contract, and explicit anti-scope-creep boundaries (what NOT to include).

3. **Critique the plan (Level-1).** Validate phase ordering, dependency correctness, goal coverage, and scope appropriateness. If severity > minor, revise and re-critique. Exit when the plan is sound or max iterations (3) are reached.

4. **Expand each phase into substeps.** For each phase, select 1-3 specialized ground-level agent types (Reader, Profiler, Transformer, Validator, Joiner, Indexer, Partitioner, Aggregator, FeatureEngineer, Compressor, Graph). Define an `EnhancedSchemaContract` per substep: required input columns, columns to add/preserve/remove, value constraints, row-count expectations, and postconditions.

5. **Critique substeps (Level-2).** Validate agent type appropriateness and input/output schema compatibility between substeps. Revise if needed.

6. **Execute each substep with progressive sampling.** Run the agent code at XS (10 rows) first. On success, escalate to S (100), then M (1000), then FULL. On failure, pass the full traceback back to the coding agent for revision (max 3 revisions per tier).

7. **Monitor after each substep.** Check rule-based metrics: revision count (warn at 2, critical at 4), row drop % (warn at 30%, critical at 90%), row growth, null-rate increase, wall-clock time, peak memory. Issue verdicts without extra LLM calls.

8. **Backtrack on failure.** If a substep exhausts retries, try phase-level backtracking (re-expand with different agent types). If the phase fails twice, escalate to plan-level backtracking (Orchestrator generates a new plan). Respect global retry cap of 3.

9. **Assemble the pipeline.** Concatenate validated substep code into a standalone `pipeline.py`. Include imports, intermediate DataFrame handoffs, and a `__main__` block.

10. **Document the run.** Produce a case summary with: plan versions and revisions, monitoring verdicts and alerts, execution metrics (time, memory, row counts per stage), and the final output schema.

## Concrete Examples

**Example 1: Multi-source customer data cleaning and merge**

```
User: I have three CSV files -- customers.csv, orders.csv, and returns.csv.
Clean duplicates, standardize dates, join them, and produce a single
analytics-ready dataset.

Approach:
1. Profile: Read all three files. Identify schemas, null rates, date formats,
   duplicate keys. customers.csv has 50K rows, mixed date formats in
   "signup_date"; orders.csv has 200K rows with duplicate order_ids;
   returns.csv has 15K rows with nulls in "reason" column.

2. Plan (2 phases):
   Phase 1 - Clean: Deduplicate orders on order_id, parse dates in customers
   to ISO-8601, impute missing return reasons with "unspecified".
   Phase 2 - Merge: Left-join customers -> orders on customer_id, then
   left-join result -> returns on order_id.

3. Expand Phase 1 into 3 substeps:
   - Transformer agent: deduplicate orders.csv on order_id (keep latest)
   - Transformer agent: standardize signup_date in customers.csv
   - Transformer agent: fill nulls in returns.csv "reason" column

4. Progressive sampling on each: run at 10 rows to verify logic, then 100,
   then 1000, then full. The date parser fails at S tier on a "Feb 30" value
   -- revise to handle invalid dates by coercing to NaT, retry, pass.

5. Expand Phase 2: Joiner agent for each merge. Schema contract validates
   that customer_id and order_id keys exist and types match.

6. Monitor: Row count after join is 195K (some customers have no orders) --
   within expected range. Null rate on return columns is 90% (expected, most
   orders aren't returned). No alerts.

7. Assemble pipeline.py with all steps. Document plan, revisions, metrics.

Output:
- pipeline.py: standalone script that reads 3 CSVs, cleans, joins, outputs
  analytics_ready.parquet
- case_summary.md: 2 phases, 5 substeps, 1 revision (date parser), total
  execution metrics
```

**Example 2: Feature engineering for ML model preparation**

```
User: Prepare features from transactions.csv for a churn prediction model.
I need encoding, scaling, temporal features, and binning.

Approach:
1. Profile: 500K rows, 25 columns. Categorical: payment_method (4 values),
   plan_type (3 values). Numeric: amount, frequency. Datetime: last_purchase,
   signup_date. High cardinality: merchant_name (12K unique).

2. Plan (2 phases):
   Phase 1 - Feature extraction: derive temporal features (days_since_purchase,
   account_age_days, purchase_day_of_week), bin amount into quartiles.
   Phase 2 - Encoding and scaling: one-hot encode payment_method and plan_type,
   label-encode merchant_name (too many values for one-hot), standard-scale
   numeric features.

3. Expand Phase 1:
   - FeatureEngineer agent: datetime extraction (3 new columns from timestamps)
   - FeatureEngineer agent: binning amount into quartiles

4. Progressive sampling: XS tier catches an edge case where last_purchase is
   null for 2% of rows -- revise to fill with signup_date before computing
   days_since_purchase. Passes at S tier onward.

5. Expand Phase 2:
   - Transformer agent: one-hot encoding (payment_method, plan_type)
   - Transformer agent: label encoding (merchant_name)
   - Transformer agent: standard scaling (amount, frequency, temporal features)

6. Monitor: Column count grew from 25 to 38 (expected with one-hot).
   No row drops. Null rate 0% after imputation. Peak memory 180MB.

Output:
- pipeline.py: reads transactions.csv, outputs features.parquet with 38
  columns, all numeric, no nulls, scaled
- Schema contract validation log showing input/output compatibility at
  every substep boundary
```

**Example 3: Iterative pipeline repair with backtracking**

```
User: Process sensor_readings.csv -- remove outliers, resample to hourly,
compute rolling averages, and detect anomalies.

Approach:
1. Profile: 2M rows, 1-second resolution, columns: timestamp, sensor_id,
   temperature, humidity, pressure. Outliers visible in temperature (values
   of -999 as sentinel nulls).

2. Plan (3 phases):
   Phase 1 - Clean: replace sentinel values, remove statistical outliers
   Phase 2 - Resample: group by sensor_id, resample to hourly means
   Phase 3 - Analytics: compute 24h rolling averages, flag anomalies
   (values > 3 std from rolling mean)

3. Execute Phase 1: Transformer agent replaces -999 with NaN, then uses
   IQR method to clip outliers. Passes all sampling tiers.

4. Execute Phase 2: Aggregator agent resamples. At M tier (1000 rows),
   Monitor detects 85% row drop -- triggers warning. Investigation: the
   resample is correct (1-second to 1-hour is ~3600:1 reduction), but 1000
   rows covers only ~17 minutes of data for one sensor, producing 1 row.
   Verdict: expected behavior, continue to FULL.

5. Execute Phase 3: First attempt at rolling average fails at XS tier --
   10 hourly rows insufficient for 24h window. Revision: pad with NaN for
   windows shorter than 24 points. Passes. Anomaly detection runs.
   At FULL tier, Monitor flags that anomaly rate is 12% -- higher than
   typical 1-5%. Phase-level backtrack: Architect re-expands with a
   different anomaly threshold (median absolute deviation instead of std).
   New rate: 3.2%. Passes.

Output:
- pipeline.py with 3 phases, 1 backtrack logged
- monitor_summary: documents the row-drop warning (expected), anomaly rate
  correction, and final metrics
```

## Best Practices

- **Do: Keep phases minimal (1-3).** Each phase should have a single coherent objective. More phases means more context boundaries and more places for schema mismatches. If you need more than 3 phases, re-examine whether some can be merged.

- **Do: Define explicit schema contracts between substeps.** Specify required input columns, expected output columns, value constraints, and row-count expectations. This catches integration bugs before execution.

- **Do: Start progressive sampling at XS (10 rows) every time.** Even trivial transformations can have surprising edge cases. The cost of running 10 rows is negligible; the cost of debugging a full-dataset failure is high.

- **Do: Reuse validated agent code.** If a previous pipeline had a working date parser or deduplication step, adapt that code rather than generating from scratch. Store successful agent implementations for retrieval.

- **Avoid: Monolithic single-agent pipelines.** A single agent handling all steps accumulates context and loses focus. Split into specialized agents with clear handoff points.

- **Avoid: Ignoring Monitor warnings.** A 30% row drop or rising null rate usually indicates a real problem. Investigate before continuing -- even if the code "runs successfully," the data may be wrong.

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| Code syntax/runtime error at XS tier | Traceback in sandbox output | Pass traceback to coding agent, revise (max 3 attempts) |
| Schema mismatch between substeps | Schema contract validation | Architect re-expands substep with corrected contract |
| Excessive row drop (>90%) | Monitor rule-based check | Phase-level backtrack: try alternative agent type or logic |
| Repeated phase failure (2+ times) | Retry counter | Plan-level backtrack: Orchestrator generates revised plan |
| Memory/time budget exceeded | tracemalloc / wall-clock tracking | Increase progressive sampling aggressiveness or partition data |
| Infinite backtrack loop | Global retry cap (3) | Abort with detailed case documentation for human review |
| Unexpected data drift mid-pipeline | Null-rate and row-count delta checks | Pause execution, alert user, suggest profiling the intermediate output |

When all retries are exhausted, produce a diagnostic report containing: the plan versions attempted, the specific substep that failed, all tracebacks, monitoring metrics at each stage, and a recommendation for manual intervention.

## Limitations

- **Not for streaming/real-time data.** ADP-MA is designed for batch processing. It profiles the full dataset upfront and uses progressive sampling tiers that assume static data.
- **LLM cost scales with complexity.** Each meta-agent call, critique loop iteration, and code revision consumes tokens. Pipelines with many backtrack cycles can become expensive. Set budget ceilings.
- **Sandbox constraints.** Ground-level agents execute in a restricted environment (pandas and datetime only by default). Pipelines requiring specialized libraries (scipy, networkx, sklearn) need explicit sandbox extension.
- **Not a replacement for production orchestrators.** ADP-MA generates a standalone pipeline.py, but it does not provide scheduling, fault-tolerant distributed execution, or production monitoring. Use it for pipeline *design and prototyping*, then deploy the output script in Airflow/Prefect/Dagster.
- **Agent reuse depends on task similarity.** The library of prior agents helps only when new tasks share patterns with past ones. Novel data structures or unusual transformations still require full generation.

## Reference

[Autonomous Data Processing using Meta-Agents (arXiv:2602.00307)](https://arxiv.org/abs/2602.00307) -- Khurana, 2026. Focus on Algorithm 1 (operational workflow), Table 1 (progressive sampling tiers), Table 2 (monitoring thresholds), and Section 4.3 (workload partitioning strategies: centralized, autonomous, hybrid).