---
name: "semanticalli-caching-reasoning-not"
description: "Implement pipeline-aware intermediate representation (IR) caching for agentic systems. Instead of caching final LLM responses, decompose multi-step pipelines into stages and cache structured reasoning artifacts at each checkpoint. Triggers: 'cache intermediate reasoning', 'reduce redundant LLM calls', 'pipeline-aware caching', 'agentic caching strategy', 'cache reasoning not responses', 'structured IR caching for agents'"
---

# SemanticALLI: Caching Reasoning, Not Just Responses, in Agentic Systems

This skill teaches Claude to design and implement **stage-decomposed caching** for agentic AI pipelines. The core insight from SemanticALLI is that even when users never repeat the same query, the pipeline's internal reasoning often converges on identical structured intermediate representations (IRs). By decomposing a monolithic LLM call into discrete stages — each producing a cacheable structured artifact — you can bypass the majority of redundant LLM invocations. In evaluation, this approach lifts cache hit rates from ~39% (monolithic prompt caching) to ~83% (IR-level caching), cutting token consumption by 78% and reducing median cache-hit latency to 2.66ms.

## When to Use This Skill

- When building an agentic pipeline that chains multiple LLM calls (e.g., intent parsing -> data retrieval -> code generation -> output formatting) and you want to reduce cost and latency
- When the user says "my agent pipeline is too slow/expensive" and you need to identify redundant reasoning
- When designing a caching layer for an AI system that handles paraphrased versions of the same underlying request
- When the user asks to "add caching to my LLM agent" and naive prompt-level caching isn't sufficient
- When building analytics, dashboards, or code-generation agents where the same structured logic recurs across different natural-language inputs
- When implementing a multi-stage agent where early stages normalize intent and later stages synthesize output from that normalized form

## Key Technique

**The problem with monolithic caching:** Traditional approaches cache at the boundary — hash the user prompt, check for an exact or semantic match, return the cached response. This fails when users phrase the same underlying request differently. "Show me monthly revenue trends" and "Plot revenue by month" have low lexical overlap but identical downstream logic. Monolithic caching caps around 38-40% hit rate because it cannot see past linguistic variance.

**Stage decomposition and IR elevation:** SemanticALLI decomposes inference into at least two stages: (1) **Analytic Intent Resolution (AIR)** — which normalizes a natural language query into a canonical structured IR containing metrics, dimensions, filters, temporal grain, and output primitives; and (2) **Visualization/Synthesis (VS)** — which converts that IR into executable output (code, charts, API calls). The IR becomes a first-class cacheable artifact. Because the IR is structured and stable under paraphrase, the VS stage achieves dramatically higher cache hit rates. Two completely different user prompts that resolve to the same IR will share VS cache entries.

**Three-tier hybrid retrieval for cache lookup:** Cache matching uses a tiered strategy: **Tier 0** — SHA-256 exact hash for O(1) deterministic dedup; **Tier 1** — dense semantic embedding (e.g., `text-embedding-3-large`) with HNSW approximate nearest-neighbor search (cosine similarity); **Tier 2** — BM25 lexical reranking with Reciprocal Rank Fusion (RRF) to prevent entity-level collisions. This last tier is critical: "Show DDA Revenue by channel" and "Show GA4 Revenue by channel" have cosine similarity ~0.96 but reference incompatible attribution models. The lexical tier catches these distinctions. Admission thresholds differ by stage: AIR uses `tau=0.90`, VS uses stricter `tau=0.95` to prevent mismatches in output structure.

## Step-by-Step Workflow

1. **Identify the pipeline stages in the existing agent.** Map out every LLM call in the pipeline. For each call, classify whether it performs (a) intent normalization/reasoning, (b) data retrieval/transformation, or (c) output synthesis/formatting. Draw the dependency graph between stages.

2. **Define the structured Intermediate Representation (IR) schema.** For each stage boundary, design a canonical schema that captures the stage's output. For an analytics agent, the AIR IR might be:
   ```json
   {
     "metrics": ["revenue"],
     "dimensions": ["channel", "month"],
     "filters": [{"field": "region", "op": "eq", "value": "US"}],
     "temporal_grain": "monthly",
     "output_type": "line_chart",
     "sort": {"field": "month", "direction": "asc"}
   }
   ```
   The schema must be **stable under paraphrase** (different wordings resolve to the same object) but **sensitive to business-critical distinctions** (different metrics, entities, or data sources must never collide).

3. **Implement the intent resolution stage (AIR equivalent).** Write the first-stage prompt/logic that maps raw user input + context schema into the structured IR. This stage absorbs all linguistic variance. Its output is deterministic and structured, not free-form text.

4. **Implement the synthesis stage (VS equivalent).** Write the second-stage logic that consumes only the structured IR (never the raw user prompt) and produces the final output — code, chart spec, API calls, or formatted text. Because this stage's input is structured, cache key computation is straightforward.

5. **Build the three-tier cache lookup.** For each cacheable stage:
   - **Tier 0:** SHA-256 hash of the deterministic serialization of the stage input. Check this first for O(1) exact matches.
   - **Tier 1:** Embed the input using a dense embedding model. Search an HNSW index for approximate matches with cosine similarity. Retrieve top-k=10 candidates.
   - **Tier 2:** Score candidates with BM25 on critical entity tokens (metric names, dimension values, filter values). Combine via RRF: `score(c) = 1/(60 + rank_dense) + 1/(60 + rank_lexical)`. Accept the top candidate only if it exceeds the admission threshold.

6. **Set stage-specific admission thresholds.** Use a stricter threshold for later stages (e.g., `tau=0.95` for synthesis) because output-level errors are harder to detect. Earlier stages (intent resolution) can tolerate slightly lower thresholds (`tau=0.90`) since their outputs will be further validated downstream.

7. **Implement cache write with deterministic serialization.** When a cache miss occurs and the LLM generates a fresh result, serialize the stage input canonically (sorted keys, normalized whitespace, consistent type coercion) and store the input-output pair. Use the SHA-256 of the canonical form as the primary key, and index the embedding vector in the HNSW graph.

8. **Add entity-aware guardrails.** Extract critical entities (metric names, data source identifiers, user-specific IDs) from both the query and the candidate cache entry. Reject cache hits where entities diverge, even if the similarity score exceeds the threshold. This prevents the "CPC vs. CPM" class of false positive.

9. **Instrument and measure.** Log hit/miss rates per tier and per stage. Track: (a) overall hit rate by stage, (b) false positive rate (cached result was wrong), (c) latency distribution for cache hits vs. LLM calls, (d) token savings. Use these metrics to tune thresholds.

10. **Iterate on IR schema granularity.** If hit rates are low, the IR may be too specific (over-partitioning). If false positives appear, the IR is too coarse. Adjust the schema fields and admission thresholds based on production telemetry.

## Concrete Examples

**Example 1: Analytics agent with redundant chart generation**

User: "I have a FastAPI agent that takes natural language queries and generates Plotly charts from a SQL database. Different users keep asking for similar charts with different wording and it's expensive."

Approach:
1. Decompose the pipeline into two stages: `parse_query -> structured_intent` and `structured_intent -> plotly_code`
2. Define the IR schema:
   ```python
   @dataclass
   class AnalyticIntent:
       metrics: list[str]          # e.g., ["revenue", "cost"]
       dimensions: list[str]       # e.g., ["channel", "date"]
       filters: list[Filter]       # e.g., [Filter("region", "eq", "US")]
       temporal_grain: str         # "daily" | "weekly" | "monthly"
       chart_type: str             # "bar" | "line" | "scatter" | "table"
       sort: Optional[SortSpec]
   ```
3. Build a two-layer cache:
   ```python
   import hashlib, json
   from dataclasses import asdict

   class IRCache:
       def __init__(self, embed_fn, threshold_air=0.90, threshold_vs=0.95):
           self.exact_store = {}      # sha256 -> output
           self.vector_index = HNSWIndex(dim=3072)
           self.entries = []
           self.embed_fn = embed_fn
           self.threshold_air = threshold_air
           self.threshold_vs = threshold_vs

       def canonical_key(self, intent: AnalyticIntent) -> str:
           serialized = json.dumps(asdict(intent), sort_keys=True)
           return hashlib.sha256(serialized.encode()).hexdigest()

       def lookup(self, intent: AnalyticIntent, stage: str) -> Optional[str]:
           # Tier 0: exact hash
           key = self.canonical_key(intent)
           if key in self.exact_store:
               return self.exact_store[key]

           # Tier 1: semantic search
           threshold = self.threshold_vs if stage == "vs" else self.threshold_air
           query_vec = self.embed_fn(json.dumps(asdict(intent), sort_keys=True))
           candidates = self.vector_index.search(query_vec, k=10)

           # Tier 2: entity-aware filtering
           for candidate in candidates:
               if candidate.score >= threshold:
                   if self._entities_match(intent, candidate.intent):
                       return candidate.output
           return None

       def _entities_match(self, a: AnalyticIntent, b: AnalyticIntent) -> bool:
           return a.metrics == b.metrics and a.dimensions == b.dimensions
   ```

Output: Queries like "Show me revenue by channel this month" and "Monthly channel revenue breakdown" both resolve to the same `AnalyticIntent(metrics=["revenue"], dimensions=["channel"], temporal_grain="monthly", chart_type="bar")`, so the Plotly code generation is served from cache on the second occurrence.

**Example 2: Code-generation agent with repeated scaffolding**

User: "My coding agent generates boilerplate FastAPI endpoints. Users describe what they want in different ways but the generated code is often structurally identical. How do I cache the reasoning?"

Approach:
1. Define the stages: `user_request -> endpoint_spec` (AIR) and `endpoint_spec -> fastapi_code` (VS)
2. Design the IR:
   ```python
   @dataclass
   class EndpointSpec:
       method: str                    # "GET" | "POST" | "PUT" | "DELETE"
       path: str                      # "/users/{user_id}"
       request_body: Optional[dict]   # JSON schema of request
       response_model: dict           # JSON schema of response
       auth_required: bool
       db_operations: list[str]       # ["select", "insert", "join"]
       pagination: bool
   ```
3. Cache the `endpoint_spec -> code` mapping. Two requests — "Create a GET endpoint to list users with pagination" and "I need an API route that fetches all users, paginated" — both produce `EndpointSpec(method="GET", path="/users", pagination=True, db_operations=["select"], ...)`. The generated FastAPI code is identical and served from cache.

Output: The VS cache hit avoids regenerating the FastAPI route handler, Pydantic models, and SQLAlchemy query — saving ~3,000 tokens per hit.

**Example 3: Adding IR caching to an existing LangChain pipeline**

User: "I have a LangChain agent with tools. How do I add SemanticALLI-style caching?"

Approach:
1. Identify the stable checkpoint: in LangChain, the tool-call decision is the IR. After the LLM decides *which* tool to call and *with what arguments*, that structured tool invocation is the cacheable artifact.
2. Wrap the agent's tool-calling step:
   ```python
   class CachedToolRouter:
       def __init__(self, agent, cache: IRCache):
           self.agent = agent
           self.cache = cache

       def invoke(self, user_input: str):
           # Stage 1 (AIR): LLM resolves intent to tool call
           tool_call = self.agent.plan(user_input)  # returns ToolCall(name, args)

           # Stage 2 (VS): check cache for this exact tool invocation
           cached = self.cache.lookup(tool_call, stage="vs")
           if cached:
               return cached

           # Cache miss: execute and store
           result = self.agent.execute(tool_call)
           self.cache.store(tool_call, result)
           return result
   ```
3. The tool call's structured form (`{"tool": "sql_query", "args": {"table": "users", "filters": ...}}`) is far more stable than the raw prompt, so cache hits jump from prompt-level rates (~39%) to IR-level rates (~83%).

## Best Practices

- **Do:** Design IRs that are *paraphrase-invariant* but *entity-sensitive*. The whole point is that different wordings for the same intent resolve to the same IR, but different entities (metrics, models, data sources) must always produce distinct IRs.
- **Do:** Use deterministic serialization (sorted keys, consistent type coercion) for cache keys. Non-deterministic JSON serialization will cause cache misses on identical IRs.
- **Do:** Set stricter admission thresholds for later pipeline stages (`tau=0.95` for synthesis vs. `tau=0.90` for intent). Late-stage errors propagate directly to users.
- **Do:** Instrument per-stage hit rates from day one. You need this data to tune thresholds and identify schema design problems.
- **Avoid:** Caching IRs that contain volatile context (timestamps, session IDs, user-specific state) without first stripping or normalizing those fields.
- **Avoid:** Using only dense semantic similarity for cache lookup. Embedding-only retrieval suffers from entity collisions — "DDA Revenue" and "GA4 Revenue" score ~0.96 cosine similarity but are operationally incompatible. Always add a lexical/entity matching tier.

## Error Handling

- **False positive cache hits (entity collision):** The most dangerous failure. Mitigate with the entity-matching guardrail in Tier 2. When detected in production, immediately add the colliding pair to a negative cache (blocklist) and tighten the admission threshold.
- **Schema drift in IRs:** If the IR schema changes (new fields, renamed fields), all existing cache entries may become invalid. Version the IR schema and include the version in the cache key. On schema change, either invalidate the cache or implement a migration function.
- **Stale cached outputs:** If the underlying data or business logic changes, cached synthesis results may be wrong. Implement TTL-based expiry or event-driven invalidation keyed to data source freshness.
- **Threshold too permissive:** Manifests as users receiving wrong results. Monitor false positive rate. If it exceeds 1%, raise `tau` by 0.02 increments.
- **Threshold too strict:** Manifests as low hit rates despite known redundancy. Lower `tau` by 0.01 increments and monitor false positive rate.

## Limitations

- **Requires a decomposable pipeline.** If the agent performs a single monolithic LLM call with no identifiable internal stages, there is no intermediate checkpoint to cache. The technique works best when the pipeline has at least two stages with a structured boundary.
- **IR schema design is non-trivial.** A poorly designed IR that is too granular will miss caching opportunities; one that is too coarse will cause false positives. Expect iteration.
- **Does not help with truly novel reasoning.** If every query requires genuinely unique logic (not just unique phrasing), IR-level caching provides no benefit. The technique exploits the gap between linguistic novelty and reasoning novelty.
- **Embedding + HNSW infrastructure overhead.** Tier 1 requires maintaining a vector index and an embedding model. For small-scale applications, the exact-hash tier alone may suffice.
- **Domain-specific entity matching.** The Tier 2 entity guardrail requires domain knowledge about which fields are "critical entities." This must be manually specified per use case.

## Reference

[SemanticALLI: Caching Reasoning, Not Just Responses, in Agentic Systems](https://arxiv.org/abs/2601.16286v2) — Chillara et al., 2026. Focus on Section 3 (architecture decomposition into AIR and VS), Section 4 (three-tier hybrid retrieval with RRF), and Table 1 (hit rate comparisons across caching strategies).