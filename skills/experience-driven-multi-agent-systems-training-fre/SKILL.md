---
name: "experience-driven-multi-agent-systems-training-fre"
description: "Build self-evolving multi-agent systems that accumulate tool-usage expertise in a memory bank without fine-tuning. Implements GeoEvolver-style retrieval-augmented orchestration with sub-goal decomposition, parallel exploration of tool-parameter configurations, root-cause failure attribution, and experience distillation. Triggers: 'build a self-evolving agent pipeline', 'multi-agent system with experience memory', 'tool-parameter exploration with failure learning', 'orchestrate agents with evolving memory bank', 'training-free agent expertise accumulation', 'GeoEvolver-style agent system'"
---

# Experience-Driven Multi-Agent Systems with Evolving Memory

This skill enables Claude to design and implement **self-evolving multi-agent systems** based on the GeoEvolver architecture (Dai et al., 2026). The core idea: instead of fine-tuning LLM parameters, externalize domain expertise into a dynamic memory bank that grows from structured interaction. The system decomposes complex queries into independent sub-goals, explores diverse tool-parameter configurations in parallel, then distills successful patterns and root-cause failure attributions into retrievable experiences that improve future executions. This produces training-free agents that get measurably better at tool-intensive pipelines over time.

## When to Use This Skill

- When the user needs to build an agent system that orchestrates multiple tools in long-horizon pipelines (e.g., data processing chains, ETL workflows, multi-step analysis)
- When the user wants agents that learn from execution failures without retraining or fine-tuning the underlying model
- When building a multi-agent architecture that decomposes complex tasks into parallelizable sub-goals with explicit I/O contracts
- When the user asks for a pipeline that explores multiple tool configurations and selects the best execution path
- When implementing retrieval-augmented planning where past successes and failures guide future tool usage
- When the user needs robust error recovery through root-cause attribution rather than naive retrying
- When building domain-specific agent systems for tool-heavy fields (geospatial, bioinformatics, DevOps, data engineering)

## Key Technique: Experience-as-Parameters

The GeoEvolver architecture replaces parameter updates with an **evolving memory bank** that stores distilled execution experiences. This is built on three insights:

**Sub-goal isolation localizes failure.** Rather than planning an entire pipeline monolithically, the Orchestrator decomposes each query into N independent sub-goals, each with explicit input/output format contracts and success criteria. Sub-goals execute in parallel via dedicated Executor agents. When a sub-goal fails, the failure is contained -- you know exactly which tool call in which sub-goal broke, not just that "the pipeline failed." Task-level success requires all sub-goals to succeed: `Y = AND(Y_1, Y_2, ..., Y_N)`.

**Parallel exploration beats single-shot execution.** For each sub-goal, the system maintains K parallel exploration variants, each trying different prompt phrasings or tool-parameter configurations. A Judge agent scores each variant's trajectory using binary success labels plus auxiliary validity signals (step count, output correctness). The best trajectory is selected via `argmax_k Score(trajectory_k | success_k, validity_k)`. This turns tool configuration from a guess into a search.

**Contrast-driven distillation produces reusable expertise.** After execution, an Extractor compares successful and failed trajectories to produce memory entries. Successes capture tool orderings and decision checkpoints. Failures capture root causes and corrective guardrails. These entries are embedded and stored in the memory bank with deduplication. Future queries retrieve the top-k most similar entries via embedding similarity, which are injected as high-level hints (not literal parameter copies) into executor prompts.

## Step-by-Step Workflow

### 1. Define the Agent Roles

Create four distinct agent roles following the GeoEvolver operator pipeline:

- **Retriever**: Queries the memory bank using embedding similarity to find relevant past experiences for the current task
- **Orchestrator**: Decomposes the user query into N independent sub-goals with explicit I/O contracts, conditioned on retrieved context
- **Executors** (N instances): Each executes one sub-goal independently, calling tools and producing a sub-trajectory
- **Judge**: Evaluates sub-trajectories, emits binary success labels, and produces validity confidence signals

### 2. Implement the Memory Bank

Create a persistent store (JSON file, SQLite, or vector DB) with this schema per entry:

```json
{
  "id": "unique-id",
  "query_embedding": [0.12, -0.34, ...],
  "query_text": "Original task description",
  "outcome": "success" | "failure",
  "sub_goal": "The specific sub-goal this entry relates to",
  "tool_chain": ["tool_a(param=x)", "tool_b(param=y)"],
  "analysis_pattern": "Reusable description of what worked and why",
  "failure_cause": "Root cause + tool error hint (null if success)",
  "corrective_guardrail": "What to avoid or check next time (null if success)",
  "timestamp": "2026-01-30T12:00:00Z"
}
```

### 3. Implement Retrieval with Leakage Filtering

When a new query arrives, embed it and retrieve the top-k most similar memory entries:

```python
def retrieve(query: str, memory_bank: list, k: int = 5) -> list:
    query_vec = embed(query)
    scored = [(cosine_sim(query_vec, m["query_embedding"]), m) for m in memory_bank]
    scored.sort(key=lambda x: x[0], reverse=True)
    # Leakage filter: exclude entries whose content overlaps with expected output
    filtered = [m for _, m in scored if not leaks(m, expected_output)]
    return filtered[:k]
```

Aggregate retrieved entries into a strategy context string that the Orchestrator consumes.

### 4. Decompose the Query into Sub-Goals

The Orchestrator takes `(query, strategy_context)` and produces N sub-goals. Each sub-goal specifies:

- **Input contract**: What data/format the executor receives
- **Output contract**: What data/format the executor must produce
- **Success criteria**: How the Judge will evaluate correctness
- **Tool hints**: Suggested tools from retrieved experiences (as hints, not mandates)

Ensure sub-goals have no inter-dependencies so they can execute in parallel.

### 5. Run Parallel Exploration Variants

For each sub-goal, spawn K variants (typically K=3-5) that differ in prompt phrasing or tool-parameter choices. Each variant independently runs retrieve-plan-execute with up to A corrective retries on failure:

```python
for variant_k in range(K):
    trajectory = execute_with_retries(sub_goal, variant_config_k, max_retries=A)
    results.append((trajectory, judge.evaluate(trajectory)))
```

### 6. Select the Best Trajectory per Sub-Goal

The Judge scores each variant's trajectory using success label and validity signals. Select the best:

```python
best = max(results, key=lambda r: score(r.trajectory, r.success, r.validity))
```

Validity signals include: trajectory step count (fewer is better for equivalent success), output format correctness, and intermediate checkpoint verification.

### 7. Distill Experiences via Contrast Extraction

Compare the best successful trajectory against failed ones to produce a memory entry:

- **If successful**: Extract the tool ordering, parameter choices, and decision checkpoints as an `analysis_pattern`
- **If failed**: Extract the root cause from error messages and logs, then formulate a `corrective_guardrail` for future avoidance

```python
if best.success:
    entry = extract_success_pattern(best.trajectory, query)
else:
    entry = extract_failure_attribution(all_trajectories, query)
```

### 8. Update the Memory Bank with Deduplication

Before inserting, check if a canonical duplicate already exists (same query pattern + same tool chain). If so, merge or skip. Otherwise, embed and store:

```python
canonical_key = hash(entry["sub_goal"] + str(entry["tool_chain"]))
if canonical_key not in memory_bank.keys():
    entry["query_embedding"] = embed(entry["query_text"])
    memory_bank.insert(entry)
```

### 9. Assemble Final Output from Sub-Goal Results

Combine the outputs of all successful sub-goals into the final task result. If any sub-goal failed across all K variants, report the specific failure with root-cause attribution rather than a generic error.

### 10. Maintain Working Memory for Long Horizons

For long-running pipelines, compress older context while keeping recent steps in full:

```
working_memory = summarize(previous_history) + last_L_steps(trajectory)
```

This prevents context overflow while preserving the most actionable recent state.

## Concrete Examples

**Example 1: Building a self-evolving data pipeline agent**

User: "Build a multi-agent system that processes CSV files through validation, transformation, and loading steps. It should learn from past failures to avoid repeating mistakes."

Approach:
1. Define four agents: Retriever (checks memory for past CSV pipeline experiences), Orchestrator (splits into validate/transform/load sub-goals), Executor (runs each step), Judge (checks row counts, schema match, null rates)
2. Create a memory bank JSON file at `./memory_bank.json`
3. Orchestrator decomposes: Sub-goal 1 = validate schema + check nulls, Sub-goal 2 = apply transformations, Sub-goal 3 = load into target
4. For each sub-goal, run 3 variants with different tool configs (e.g., pandas vs. polars, strict vs. lenient null handling)
5. Judge evaluates: did output schema match target? Are row counts preserved? Any data loss?
6. Distill: store the winning tool chain ("polars read_csv with `null_values=['NA','']` + schema enforcement") and failure patterns ("pandas silently coerced dates to strings when timezone info present")

Output structure:
```
memory_bank.json entry:
{
  "query_text": "Process CSV with mixed date formats into warehouse",
  "outcome": "success",
  "sub_goal": "validate_and_parse_dates",
  "tool_chain": ["polars.read_csv(try_parse_dates=True)", "polars.cast(Date)"],
  "analysis_pattern": "Use polars with try_parse_dates for mixed ISO/US date formats. Validate with .null_count() after parse.",
  "failure_cause": null,
  "corrective_guardrail": null
}
```

**Example 2: DevOps deployment pipeline with failure learning**

User: "Create an agent system that deploys services to Kubernetes and learns from deployment failures to suggest better configurations next time."

Approach:
1. Retriever checks memory bank for past deployment experiences matching the service type
2. Orchestrator decomposes: Sub-goal 1 = build container image, Sub-goal 2 = run pre-deploy checks (lint manifests, dry-run), Sub-goal 3 = apply to cluster, Sub-goal 4 = verify health
3. For sub-goal 3, explore variants: rolling update vs. blue-green, different resource limits, different readiness probe configs
4. Judge evaluates: pods healthy? Rollout complete? No OOMKills within 60s?
5. On failure (e.g., OOMKilled), extract root cause: "memory limit 256Mi insufficient for Java service with default heap; corrective guardrail: set memory limit >= 512Mi for JVM workloads or add -XX:MaxRAMPercentage=75"
6. Next deployment of a similar Java service retrieves this experience and applies the guardrail as a hint

Output structure:
```
memory_bank.json entry:
{
  "query_text": "Deploy Java Spring Boot service to k8s",
  "outcome": "failure",
  "sub_goal": "apply_to_cluster",
  "tool_chain": ["kubectl apply -f deployment.yaml", "kubectl rollout status"],
  "analysis_pattern": null,
  "failure_cause": "OOMKilled: container memory limit 256Mi < JVM default heap. Pod restarted 3x in 60s.",
  "corrective_guardrail": "For JVM workloads: set resources.limits.memory >= 512Mi AND add env JAVA_TOOL_OPTIONS=-XX:MaxRAMPercentage=75"
}
```

**Example 3: Multi-step API integration agent**

User: "Build an agent that integrates data from multiple REST APIs, learns which parameter combinations work, and avoids rate-limit errors it has seen before."

Approach:
1. Retriever fetches past API integration experiences for similar endpoint patterns
2. Orchestrator decomposes by API source: Sub-goal 1 = fetch from API-A with pagination, Sub-goal 2 = fetch from API-B with auth refresh, Sub-goal 3 = merge and deduplicate results
3. For sub-goal 1, explore K=3 variants: offset-based pagination vs. cursor-based, batch sizes of 50/100/200
4. Judge evaluates: complete data fetched? No 429 errors? Response times acceptable?
5. Distill: "API-A cursor pagination with batch=100 avoids 429s; offset pagination with batch=200 triggers rate limit after page 15. Always include Retry-After header handling."
6. Future similar queries start with batch=100 + cursor pagination as the default strategy

## Best Practices

**Do:**
- Store experiences at the sub-goal level, not the full pipeline level -- granular entries are more reusable across different task compositions
- Use retrieved experiences as high-level hints in prompts, not as literal parameter templates -- agents should adapt, not copy
- Include both success patterns AND failure attributions in the memory bank -- failure guardrails are often more valuable than success templates
- Deduplicate memory entries by canonical key (sub-goal type + tool chain hash) to prevent the bank from growing unboundedly
- Run at least K=3 parallel exploration variants for non-trivial sub-goals -- single-shot execution misses better configurations

**Avoid:**
- Do not create inter-dependent sub-goals -- the entire architecture relies on sub-goals executing independently in parallel
- Do not inject retrieved experiences as rigid instructions -- phrase them as "consider this pattern" not "always do this"
- Do not skip the Judge evaluation step -- without binary success labels, the memory bank accumulates unverified patterns that degrade future performance
- Do not store raw trajectories in memory -- distill them into concise patterns and guardrails to keep retrieval efficient and context windows manageable

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| All K variants fail for a sub-goal | Judge returns `success=false` for all K | Log root cause from the variant that got furthest; store as failure entry; surface specific error to user with the corrective guardrail |
| Memory bank retrieves irrelevant experiences | Cosine similarity below threshold (e.g., < 0.6) | Fall back to zero-experience mode; execute without retrieved context rather than using misleading hints |
| Sub-goal decomposition produces dependent goals | Orchestrator outputs goals referencing each other's outputs | Re-prompt Orchestrator with explicit constraint: "Each sub-goal must be independently executable with only the original query input" |
| Memory bank grows too large for retrieval quality | Retrieval latency increases; duplicate patterns dilute relevance | Run periodic deduplication and prune entries older than N days with low retrieval frequency |
| Corrective guardrail contradicts current context | Retrieved failure guardrail applies to a different version/config | Timestamp all entries; weight recent entries higher in retrieval scoring; discard entries older than a configurable TTL |

## Limitations

- **Cold start**: The memory bank starts empty. First executions have no retrieved experiences and perform at baseline LLM capability. Expect the system to reach useful memory density after 20-50 diverse task executions.
- **Domain transfer**: Experiences stored from one domain (e.g., geospatial tool chains) do not transfer to unrelated domains (e.g., financial APIs). Each domain needs its own memory bank.
- **Context window pressure**: Retrieved experiences consume prompt tokens. With large memory banks and high-k retrieval, the context budget for actual execution shrinks. Keep k low (3-7) and entries concise.
- **No causal reasoning**: The memory bank stores correlations (this tool chain worked / failed), not causal models. If the underlying environment changes (API versioned, tool updated), stored experiences may become stale or harmful.
- **Parallel variant cost**: Running K variants per sub-goal multiplies API/compute cost by K. For cost-sensitive applications, reduce K to 2 or use variants only for sub-goals with low historical success rates.

## Reference

**Paper**: [Experience-Driven Multi-Agent Systems Are Training-free Context-aware Earth Observers](https://arxiv.org/abs/2602.02559v1) (Dai et al., 2026). Look for: the four-operator pipeline (Retriever, Orchestrator, Executor, Judge), the contrast-driven experience extraction mechanism, and the empirical results showing 12% average gain across three EO benchmarks without any parameter updates.