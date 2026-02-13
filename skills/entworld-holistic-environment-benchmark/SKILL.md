---
name: "entworld-holistic-environment-benchmark"
description: "Build verifiable enterprise GUI agent benchmarks using schema-grounded task generation and SQL-based deterministic verification. Use when: 'generate enterprise test tasks from a database schema', 'build SQL verification for GUI agent tasks', 'create benchmark for CRM/ERP/ITIL agents', 'reverse-engineer business logic from DB schema', 'validate agent actions with database state checks', 'set up dockerized enterprise benchmark environments'."
---

This skill enables Claude to build enterprise-grade GUI agent benchmarks and verification systems using the EntWorld methodology. The core technique reverse-engineers business logic directly from database schemas to synthesize realistic multi-step tasks, then validates agent completion through SQL-based state-transition checks rather than brittle visual matching. Apply this when building evaluation harnesses for enterprise automation agents, generating test scenarios from existing databases, or implementing deterministic verification of GUI-driven workflows across CRM, ERP, ITIL, asset management, and project management domains.

## When to Use

- When the user wants to generate realistic test tasks for a GUI agent that operates on an enterprise web application (CRM, ERP, helpdesk, project management, asset tracking)
- When the user needs to verify that a GUI agent correctly completed a multi-step workflow by checking database state rather than screenshots
- When the user has an existing database schema and wants to automatically produce benchmark tasks with ground-truth SQL verification queries
- When building a Dockerized, reproducible evaluation environment for enterprise agent testing
- When the user wants to assign difficulty scores to generated tasks based on table complexity, relational joins, and operation depth
- When adapting an open-source enterprise platform (EspoCRM, iTOP, OpenProject, ZenTao, Snipe-IT, Veops CMDB) into a structured agent benchmark

## Key Technique

**Schema-Grounded Task Generation** replaces manual task authoring with a four-stage pipeline that starts from the database itself. First, *Schema Discovery* queries the database catalog, filters non-empty tables, and uses an LLM to produce a "SchemaJSON" that infers each table's business purpose from column names, types, and sample rows. Second, *Relationship Inference* reconstructs entity-relationship graphs by parsing explicit foreign keys and using LLM-based inference for implicit links (validated by probe SQL queries that confirm join results are non-empty). Third, *Task Template Synthesis* generates parameterized task templates containing natural-language prompts plus SQL logic with placeholders, constrained by the verified schema graph. Fourth, *Data Instantiation* executes cross-table queries to populate templates with real database records, producing concrete tasks with ground-truth answers.

**SQL-Based Deterministic Verification** replaces visual matching and LLM-as-a-judge approaches with direct database interrogation. After an agent completes a GUI task, the system executes a pre-defined SQL query against the application's database to check whether the expected state transition occurred. For read operations (SELECT), it compares the agent's extracted answer against the query result. For create/update/delete operations, it validates affected rows within a transaction block, then issues a rollback to preserve the environment for subsequent tasks. This makes verification binary and reproducible: either the database reflects the correct state or it does not.

**Difficulty Quantification** scores each task as `D_task = sum(w_i * d_i)` across five dimensions: table complexity (number of tables involved), relational complexity (join depth), operation complexity (action count), result complexity (output cardinality), and SQL complexity (subqueries, aggregates, conditions). This enables stratified evaluation so you can identify where agents fail (e.g., long-horizon tasks >15 steps where GPT-4.1 drops to 2.7% success).

## Step-by-Step Workflow

1. **Inventory the database schema.** Connect to the target application's database, enumerate all non-empty tables, and extract column names, types, foreign keys, and row counts. Store this as a structured catalog (JSON or YAML).

2. **Generate SchemaJSON for each table.** For every table, prompt an LLM with the table name, column definitions, and 3-5 sample rows. Ask it to infer the business purpose, identify which columns are user-facing fields vs. internal IDs, and classify the table's domain role (e.g., "contact record", "service ticket", "asset entry").

3. **Reconstruct entity-relationship graphs.** Parse explicit foreign key constraints directly. For tables lacking declared FKs, use LLM inference on column name/type similarity (e.g., `customer_id` in an orders table likely references `id` in a customers table). Validate every inferred relationship by running a probe query: `SELECT 1 FROM A JOIN B ON A.col = B.col LIMIT 1`. Discard relationships that return zero rows.

4. **Synthesize task templates.** Prompt an LLM with the verified schema graph and ask it to generate task templates. Each template must include: (a) a natural-language instruction describing the GUI workflow, (b) the SQL query that encodes the expected outcome, and (c) placeholder tokens for entity-specific values. Constrain templates to paths that exist in the schema graph.

5. **Instantiate tasks with real data.** Execute cross-table SELECT queries to fill placeholders with authentic database records. For state-change tasks (create/update/delete), generate realistic data variants using an LLM (e.g., plausible contact names, ticket descriptions). Validate each instantiated SQL query by running it in a transaction block and confirming non-empty results before rolling back.

6. **Build verification queries.** For each task, finalize the SQL verification query: SELECT-based for information retrieval tasks (compare agent output to query result), or row-count/field-match checks for CUD operations (run pre-task snapshot, execute agent, run post-task query, diff). Wrap CUD verifications in transactions with rollback.

7. **Assign difficulty scores.** Compute `D_task` for each task by measuring: number of tables referenced, join depth, total GUI actions required, result set size, and SQL complexity (presence of subqueries, aggregates, GROUP BY, HAVING). Weight each dimension and produce a single numeric score for stratified analysis.

8. **Package the environment in Docker.** Create a self-contained Docker image per enterprise application that bundles the web app, database (pre-populated with benchmark data), and all dependencies. Expose the web UI on a mapped port. Include a `require_reset` flag on CUD tasks so the environment can be restored between runs.

9. **Run agent evaluation.** Deploy the Docker environment, present each task's natural-language instruction to the agent (along with a screenshot + accessibility tree of the current page), let the agent interact with the GUI, then execute the verification SQL against the database to produce a binary pass/fail result.

10. **Analyze results by difficulty stratum.** Group tasks by difficulty score, domain, and action count. Identify failure patterns: short tasks (<5 steps) vs. long tasks (>15 steps), single-table lookups vs. multi-join queries, and domain-specific challenges (e.g., ITIL systems with dense configuration management interfaces).

## Concrete Examples

**Example 1: Generating a CRM benchmark task from schema**

User: "I have an EspoCRM database. Generate a benchmark task that tests whether an agent can find a specific contact's related opportunities."

Approach:
1. Query the schema to find `contact` and `opportunity` tables and their join table `contact_opportunity`.
2. Generate SchemaJSON: `contact` holds person records, `opportunity` holds sales deals, linked via `contact_opportunity.contact_id` and `contact_opportunity.opportunity_id`.
3. Synthesize template: "Find all open opportunities associated with the contact named {contact_name} and report the total expected revenue."
4. Instantiate: Query `SELECT name FROM contact WHERE deleted=0 LIMIT 1` to get a real contact name, e.g., "Maria Chen".
5. Build verification SQL:

```sql
SELECT SUM(o.amount) AS total_revenue
FROM opportunity o
JOIN contact_opportunity co ON o.id = co.opportunity_id
JOIN contact c ON c.id = co.contact_id
WHERE c.name = 'Maria Chen'
  AND o.stage NOT IN ('Closed Lost', 'Closed Won')
  AND o.deleted = 0;
```

Output: Task with instruction "Find all open opportunities for Maria Chen and report the total expected revenue", verified by comparing the agent's answer against the SQL result (e.g., `$142,500`).

**Example 2: SQL-based verification for a state-change task**

User: "Create a verification check for a task where the agent must update a service ticket's priority in iTOP."

Approach:
1. Identify the task: "Change the priority of ticket TKT-4821 from 'Medium' to 'Critical'."
2. Capture pre-state:

```sql
-- Pre-task snapshot (run before agent acts)
SELECT id, priority FROM ticket WHERE ref = 'TKT-4821';
-- Expected: priority = 'Medium' (encoded as 2)
```

3. After agent completes the GUI interaction, run verification:

```sql
-- Post-task verification
SELECT CASE
  WHEN priority = 4 THEN 'PASS'
  ELSE 'FAIL'
END AS result
FROM ticket
WHERE ref = 'TKT-4821';
```

4. If running in batch mode, wrap in a transaction to allow rollback:

```sql
BEGIN;
UPDATE ticket SET priority = 4 WHERE ref = 'TKT-4821';
-- Verify: SELECT priority FROM ticket WHERE ref = 'TKT-4821'; => 4
ROLLBACK;
```

Output: Binary PASS/FAIL based on database state, no screenshot comparison needed.

**Example 3: Dockerized benchmark environment setup**

User: "Set up a reproducible evaluation environment for testing GUI agents on OpenProject."

Approach:
1. Create a Dockerfile that bundles OpenProject with its PostgreSQL database pre-loaded with benchmark data.
2. Expose the web UI on port 8080 and the database on port 5432.

```dockerfile
FROM openproject/openproject:14
COPY benchmark_seed.sql /docker-entrypoint-initdb.d/
COPY tasks.json /benchmark/tasks.json
EXPOSE 8080 5432
ENV OPENPROJECT_SECRET_KEY_BASE=benchmark_secret
ENV DATABASE_URL=postgres://openproject:openproject@localhost/openproject
```

3. Build and run: `docker build -t entworld-openproject . && docker run -p 8080:8080 -p 5432:5432 entworld-openproject`
4. Agent connects to `http://localhost:8080`, receives task instructions from `tasks.json`, and verification queries run against `localhost:5432`.

Output: Self-contained Docker image that any team can pull and run to reproduce identical evaluation conditions.

## Best Practices

- **Do:** Validate every inferred schema relationship with a probe SQL query before using it in task generation. Relationships that return zero join results will produce impossible tasks.
- **Do:** Use transaction blocks with ROLLBACK for all CUD verification queries to keep the database pristine between tasks. Tag CUD tasks with `require_reset: true` in your task manifest.
- **Do:** Combine screenshots with accessibility trees as agent input. EntWorld data shows this dual modality yields ~53.7% success vs. ~15% for screenshots alone.
- **Do:** Stratify evaluation by difficulty score and action count. Aggregate success rates hide critical failure modes on long-horizon tasks.
- **Avoid:** Relying on visual/screenshot matching for task verification. UI rendering differences, timing, and layout shifts make visual checks non-deterministic.
- **Avoid:** Generating tasks that reference deleted or soft-deleted records. Always filter with `WHERE deleted = 0` (or equivalent) in both task instantiation and verification queries.

## Error Handling

- **Schema discovery returns empty tables:** Filter these out during Stage 1. Only generate tasks for tables with sufficient data (minimum 5-10 rows) to allow meaningful instantiation.
- **Probe query timeout on large tables:** Add `LIMIT 1` to all relationship validation queries. You only need to confirm the join path exists, not enumerate results.
- **LLM generates invalid SQL in templates:** Execute every generated SQL query in a transaction-then-rollback cycle during the instantiation phase. Discard templates whose SQL fails to parse or returns empty results.
- **Docker environment state drift:** After CUD tasks, restore from a database snapshot or re-seed from the initial SQL dump rather than attempting incremental rollback, which can miss side effects from triggers or cascades.
- **Agent produces partial answers:** For information retrieval tasks, implement fuzzy matching on numeric values (within 1% tolerance for aggregates) and case-insensitive string comparison, but keep the SQL result as the authoritative ground truth.

## Limitations

- This approach works best for enterprise applications backed by relational databases with discoverable schemas. Applications using NoSQL, event-sourced, or heavily abstracted ORMs may not expose clean schema graphs.
- Task generation quality depends on the LLM's ability to infer business semantics from table and column names. Heavily abbreviated or obfuscated schemas (e.g., `tbl_x1`, `col_a`) will produce low-quality tasks.
- SQL-based verification cannot capture UX-quality aspects: whether the agent navigated efficiently, displayed correct intermediate results, or handled error states gracefully. It only validates the end-state.
- The benchmark focuses on information retrieval (SELECT) and basic CRUD operations. Complex multi-step workflows involving approval chains, notifications, or asynchronous processes are harder to verify with simple SQL state checks.
- Open-source enterprise platforms (EspoCRM, iTOP, OpenProject) may differ substantially from commercial systems (Salesforce, ServiceNow, SAP) in UI complexity and business logic depth.

## Reference

[EntWorld: A Holistic Environment and Benchmark for Verifiable Enterprise GUI Agents](https://arxiv.org/abs/2601.17722v1) — Mo et al., 2026. Focus on Section 3 (schema-grounded task generation pipeline), Section 4 (SQL-based deterministic verification), and Section 5.3 (difficulty quantification formula) for implementation details.