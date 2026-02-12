---
name: "experience-driven-multi-agent-systems-training-fre"
description: "Build self-evolving multi-agent systems that accumulate tool-level expertise through structured interaction without model fine-tuning. Uses GeoEvolver's architecture: retrieval-augmented orchestration, parallel sub-goal exploration, contrastive memory distillation, and root-cause failure attribution. Triggers: 'build a self-evolving agent pipeline', 'create an experience-driven multi-agent system', 'add memory to my agent workflow', 'implement tool exploration with failure learning', 'make agents learn from execution history', 'build a GeoEvolver-style system'"
---

# Experience-Driven Self-Evolving Multi-Agent Systems (GeoEvolver Pattern)

This skill enables Claude to design and implement multi-agent systems that progressively acquire tool-level expertise through structured interaction — without any model parameter updates. Based on the GeoEvolver architecture (Dai et al., 2026), the core insight is that LLM agents can learn fine-grained tool configuration patterns and failure recovery strategies by decomposing complex tasks into independent sub-goals, exploring diverse tool-parameter configurations in parallel, and distilling both successes and root-cause failure attributions into an evolving memory bank that provides in-context demonstrations for future queries.

## When to Use

- When the user needs to build a multi-agent pipeline that handles tool-intensive workflows (data processing, API orchestration, geospatial analysis) where tool parameter configuration is error-prone
- When the user wants agents that improve over time without retraining — learning from execution successes and failures stored in a persistent memory bank
- When building pipelines where errors in one step silently propagate and corrupt downstream results, requiring fine-grained failure localization
- When the user asks to implement parallel exploration of tool configurations with automatic selection of the best execution path
- When designing agent systems for specialized domains (earth observation, bioinformatics, financial data, DevOps) that require implicit tool constraints the LLM doesn't natively know
- When the user wants to add retrieval-augmented experience memory to an existing agent framework (LangChain, CrewAI, AutoGen, custom)

## Key Technique

**The core problem**: LLM agents generating multi-step tool-calling pipelines frequently fail in specialized domains not because the high-level plan is wrong, but because subtle tool-parameter misconfigurations propagate silently through the pipeline. Existing approaches either require expensive fine-tuning or rely on generic retry logic that doesn't learn from past failures.

**GeoEvolver's solution** uses four cooperating agents in a closed loop: (1) a **Retriever** that queries an embedding-indexed memory bank for relevant past execution patterns, (2) an **Orchestrator** that decomposes queries into N independent sub-goals with explicit input/output contracts, (3) **K parallel Executor variants** that each independently attempt the full pipeline with up to A corrective retries per sub-goal, and (4) a **Judge** that emits binary success labels per sub-goal and selects the best overall execution trace. The key architectural insight is decoupling planning from execution and making sub-goals independently evaluable, which localizes failures to specific tool calls rather than the entire pipeline.

**Experience accumulation** happens through two distillation mechanisms after each task. *Single-variant extraction* captures analysis patterns (successful tool orderings and parameter choices) or error attributions (failure symptoms and corrective guardrails) from the best trace. *Contrastive memory distillation* compares all K parallel variants to synthesize workflow invariants that held across successes and failure modes with recovery strategies. Entries are deduplicated by `(source_id, pattern_type, title)` keys and stored in a persistent memory bank. Future queries retrieve top-k similar entries via embedding similarity, providing the orchestrator with execution-grounded demonstrations rather than generic instructions.

## Step-by-Step Workflow

1. **Define the agent roles and their separation of concerns.** Create four distinct agent components: Retriever (memory lookup), Orchestrator (task decomposition), Executor (tool-calling sub-goal execution), and Judge (outcome evaluation). Each agent gets a focused system prompt describing only its responsibility.

2. **Design the memory bank schema.** Create a persistent store (JSON file, SQLite, or vector DB) with entries containing: `source_id`, `pattern_type` (one of `analysis_pattern` or `error_attribution`), `title`, `embedding` (for retrieval), `content` (the actual pattern or failure description), `tool_sequence` (ordered tool calls), `parameter_snapshot` (key parameter values), and `timestamp`. Include a deduplication function keyed on `(source_id, pattern_type, title)`.

3. **Implement the retrieval mechanism.** On each new query, embed the query text using a sentence embedding model (or LLM-based embedding), retrieve top-k entries from the memory bank by cosine similarity, and aggregate them into a "strategy context" string. Apply a leakage filter: exclude entries whose content overlaps with expected outputs to prevent shortcutting.

4. **Build the orchestrator's sub-goal decomposition.** The orchestrator receives the query plus strategy context and produces N independent sub-goals. Each sub-goal must specify: (a) a natural language description, (b) input format contract (what data it receives), (c) output format contract (what it produces), (d) success criteria (how to verify correctness). The global task succeeds only when all sub-goals succeed: `Y = AND(Y_1, Y_2, ..., Y_N)`.

5. **Implement parallel exploration with K variants.** For each query, spawn K independent execution pipelines. Each variant runs the full retrieve-plan-execute-judge loop independently. Within each variant, each sub-goal executor gets up to A corrective retry attempts. Use `asyncio.gather()` or thread pools to run variants in parallel where possible.

6. **Build the executor agents with working memory.** Each executor maintains a compressed working memory: `H_t = summarize(H_{t-1}) || tail(trajectory, L)` where older interactions are summarized and the L most recent raw tool calls are retained verbatim. This prevents context overflow during long-horizon execution while preserving recent actionable detail.

7. **Implement the judge with sub-goal-level evaluation.** The judge inspects each sub-goal's output against its success criteria, emitting per-sub-goal binary labels `Y_n` and a validity signal `v`. Select the best variant: `best = argmax_k Score(trajectory_k | Y_k, v_k)`, prioritizing verified success with validity as tiebreaker.

8. **Distill experience after each task.** Run two extraction passes: (a) *Single-variant extraction*: from the best trace, produce an `analysis_pattern` if successful or an `error_attribution` if failed, capturing the specific tool sequence, parameter values, and outcome. (b) *Contrastive distillation*: compare all K variants to identify what was consistent across successes (workflow invariants) and what distinguished failures (divergence points with recovery strategies).

9. **Consolidate into the memory bank.** Merge new entries with deduplication: `memory_bank = memory_bank UNION deduplicate(new_entries, memory_bank)`. Re-index embeddings for the new entries. Optionally prune old low-utility entries based on retrieval frequency and recency.

10. **Tune hyperparameters for your domain.** Start with K=2 variants, N=3 max sub-goals, A=2 retry attempts, and top-k=5 for memory retrieval. The paper shows strong interaction effects between K and N — jointly increasing both yields substantial gains, but moderate settings balance performance and compute cost. Memory size scales positively with retrieval effectiveness.

## Concrete Examples

**Example 1: Building a Self-Evolving Data Pipeline Agent**

User: "I need a system that processes geospatial satellite images — it has to select the right bands, apply atmospheric correction, run NDVI calculation, and classify land cover. The tools keep failing with wrong parameters."

Approach:
1. Define four GeoEvolver agents: Retriever queries a JSON memory bank, Orchestrator decomposes into sub-goals (band selection, atmospheric correction, NDVI, classification), Executors call the geospatial tools, Judge validates outputs.
2. Structure the memory bank:
```json
{
  "entries": [
    {
      "source_id": "task_042",
      "pattern_type": "analysis_pattern",
      "title": "Sentinel-2 NDVI band selection",
      "content": "For Sentinel-2 L2A, use Band 4 (Red, 10m) and Band 8 (NIR, 10m). Do NOT use Band 8A (20m) as resolution mismatch causes silent errors in NDVI calculation.",
      "tool_sequence": ["band_selector", "ndvi_calculator"],
      "parameter_snapshot": {"red_band": "B04", "nir_band": "B08", "resolution": "10m"},
      "embedding": [0.12, -0.34, ...]
    },
    {
      "source_id": "task_037",
      "pattern_type": "error_attribution",
      "title": "Atmospheric correction CRS mismatch",
      "content": "sen2cor fails silently when input CRS is WGS84 geographic. Must reproject to UTM zone first. Symptom: output raster has all-zero reflectance values.",
      "tool_sequence": ["reproject", "sen2cor"],
      "parameter_snapshot": {"target_crs": "EPSG:32633"},
      "embedding": [0.45, 0.11, ...]
    }
  ]
}
```
3. On a new query, the retriever finds the relevant band selection pattern and CRS error attribution, injecting them into the orchestrator's context so it plans the correct sub-goals with guardrails.

Output: A Python system with `retriever.py`, `orchestrator.py`, `executor.py`, `judge.py`, and `memory_bank.json` that self-improves with each processed image.

---

**Example 2: Adding Experience Memory to an Existing LangChain Agent**

User: "I have a LangChain agent that calls APIs to process financial data but it keeps misconfiguring date formats and pagination parameters. Can you add learning from failures?"

Approach:
1. Create an `ExperienceMemory` class wrapping a vector store (FAISS or ChromaDB):
```python
class ExperienceMemory:
    def __init__(self, db_path: str):
        self.store = load_or_create_vectorstore(db_path)

    def retrieve(self, query: str, k: int = 5) -> list[MemoryEntry]:
        results = self.store.similarity_search(query, k=k)
        return [MemoryEntry.from_document(doc) for doc in results]

    def distill_success(self, query: str, trajectory: list[ToolCall]) -> MemoryEntry:
        pattern = extract_analysis_pattern(query, trajectory)
        self.store.add_documents([pattern.to_document()])
        return pattern

    def distill_failure(self, query: str, trajectory: list[ToolCall], error: str) -> MemoryEntry:
        attribution = extract_root_cause(query, trajectory, error)
        self.store.add_documents([attribution.to_document()])
        return attribution
```
2. Wrap the existing agent's `invoke()` to run K=2 parallel attempts, each injecting retrieved experience into the system prompt.
3. After each task, the judge evaluates whether the financial data output is valid (correct date range, complete pagination, matching totals), then distills the outcome.

Output: The agent learns entries like "Bloomberg API requires ISO-8601 dates with timezone (2024-01-15T00:00:00Z), not YYYY-MM-DD" and "pagination cursor must be passed as header X-Next-Page, not query parameter" — and stops making these mistakes on future queries.

---

**Example 3: Contrastive Memory Distillation Across Parallel Variants**

User: "My CI/CD pipeline agent tries different deployment configurations but I want it to learn which patterns work reliably."

Approach:
1. Run K=3 parallel deployment variants with different configurations (e.g., different resource limits, health check intervals, rollout strategies).
2. Judge evaluates each: Variant 1 succeeds (gradual rollout, 30s health check), Variant 2 fails (immediate rollout, pod crash loop), Variant 3 succeeds (gradual rollout, 15s health check).
3. Contrastive distillation compares all three:
```json
{
  "pattern_type": "analysis_pattern",
  "title": "Kubernetes deployment rollout invariant",
  "content": "Gradual rollout strategy (maxSurge=1, maxUnavailable=0) is required for services with startup latency >10s. Both successful variants used gradual rollout; the failure used immediate rollout causing crash loops during health check window.",
  "workflow_invariants": ["strategy=RollingUpdate", "maxUnavailable=0"],
  "failure_modes": [{"symptom": "CrashLoopBackOff", "cause": "immediate rollout + slow startup", "fix": "use gradual rollout with maxSurge=1"}]
}
```

Output: Future deployments retrieve this invariant and the orchestrator plans gradual rollouts by default for services with known startup latency.

## Best Practices

- **Do**: Make sub-goals truly independent with explicit I/O contracts. The power of the architecture comes from failure localization — if sub-goals are coupled, a failure in one corrupts the diagnosis of another.
- **Do**: Store both successes AND failures in the memory bank. Error attributions with root-cause analysis are often more valuable than success patterns because they encode specific constraints the LLM would otherwise violate repeatedly.
- **Do**: Use contrastive distillation across parallel variants, not just single-variant extraction. Comparing what diverged between success and failure reveals causal factors that single-trace analysis misses.
- **Do**: Keep memory entries fine-grained and tool-specific (e.g., "parameter X must be format Y for tool Z") rather than high-level (e.g., "be careful with parameters"). The paper shows tool-level expertise is what drives the 12% improvement.
- **Avoid**: Skipping the leakage filter during retrieval. If memory entries contain fragments of expected outputs, the system shortcuts reasoning and becomes brittle on novel inputs.
- **Avoid**: Setting K (parallel variants) and A (retry attempts) too high simultaneously. This creates exponential compute cost. Start with K=2, A=2 and scale based on domain error rates. The paper shows moderate settings balance performance and cost.
- **Avoid**: Treating working memory and the persistent memory bank as the same thing. Working memory (`H_t`) is episode-specific compressed context that prevents context overflow; the memory bank is the persistent cross-episode knowledge store.

## Error Handling

- **Silent propagation errors**: The most dangerous failure mode. Implement per-sub-goal validation in the Judge — do not rely solely on end-to-end output checks. If a sub-goal produces output in the wrong format or with invalid values, catch it before it feeds into the next sub-goal.
- **Memory bank poisoning**: If a flawed execution is mistakenly labeled successful, the distilled pattern becomes a persistent bad demonstration. Mitigate by requiring the Judge to run deterministic validation checks (schema validation, range checks, known-answer tests) in addition to LLM-based evaluation.
- **Retrieval irrelevance**: If the memory bank grows large with diverse entries, top-k retrieval may return entries from unrelated domains. Use metadata filtering (tool names, domain tags) before embedding similarity search to narrow the candidate set.
- **Context overflow**: Long-horizon tasks with many sub-goals can exceed context limits. Use the working memory compression strategy: summarize older interactions with `summarize(H_{t-1})` and keep only the L most recent raw tool calls.
- **Variant divergence**: If all K variants fail, the contrastive distillation may not yield useful invariants. In this case, fall back to single-variant error attribution from the variant that got furthest, and flag the query for human review.

## Limitations

- **Cold start**: The memory bank starts empty. The first several queries won't benefit from retrieval and may perform at baseline agent level. Consider seeding the memory bank with manually curated patterns for critical tool constraints.
- **Compute cost**: K parallel variants each running A retries means up to K*A*N total tool executions per query. For expensive tools (large model inference, paid APIs), this may be prohibitive. Use K=1 with A=2 for cost-sensitive deployments.
- **Domain transfer**: Memory entries are domain-specific. Patterns learned for geospatial tools don't transfer to financial API tools. Maintain separate memory banks per domain or use strong metadata filtering.
- **LLM backbone dependency**: The paper shows smaller models benefit disproportionately (+89% for Qwen3-32B vs. +14% for DeepSeek-V3.1), but the orchestrator still requires a model capable of structured decomposition. Very small models may not decompose sub-goals reliably.
- **Evaluation quality ceiling**: The Judge's accuracy bounds the entire system. If the Judge cannot reliably distinguish correct from incorrect sub-goal outputs (e.g., in domains requiring deep expertise), the memory bank accumulates noise.

## Reference

**Paper**: [Experience-Driven Multi-Agent Systems Are Training-free Context-aware Earth Observers](https://arxiv.org/abs/2602.02559v1) (Dai et al., 2026). Look for: Algorithm 1 (full GeoEvolver pseudocode), Table 2 (cross-backbone results showing the 12% average gain), the ablation in Table 4 (component contributions — contrastive distillation is the single most impactful component at +21.87pp), and the hyperparameter sensitivity analysis showing K*N interaction effects.