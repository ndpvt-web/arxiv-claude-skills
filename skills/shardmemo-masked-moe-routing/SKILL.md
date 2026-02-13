---
name: "shardmemo-masked-moe-routing"
description: "Implement ShardMemo-style tiered, sharded memory with masked Mixture-of-Experts routing for agentic LLM systems. Use when: 'build a sharded memory system for agents', 'add tiered memory with MoE routing', 'implement scope-based memory retrieval for multi-agent', 'create a budgeted agent memory service', 'route queries to memory shards with masking', 'design a scalable external memory for concurrent agents'."
---

# ShardMemo: Masked MoE Routing for Sharded Agentic LLM Memory

This skill teaches Claude to design and implement **ShardMemo**-style memory architectures for agentic LLM systems. The core idea: partition agent memory into budget-controlled tiers (working state, sharded evidence, versioned skill library), then route retrieval queries to the right shards using a masked Mixture-of-Experts (MoE) gating network that first eliminates ineligible shards via structured scope tags before scoring survivors. This replaces centralized vector indexes that bottleneck under concurrent multi-agent access.

## When to Use

- When the user is building a **multi-agent system** that needs shared or partitioned long-term memory and asks how to structure it.
- When an agent memory system suffers from **retrieval latency or relevance degradation** as memory volume grows past tens of thousands of entries.
- When the user wants to **shard a vector store** across categories (e.g., per-user profiles, observation logs, session history) and needs intelligent routing to avoid scanning all shards.
- When building a **skill/tool library** for agents that must support concurrent reads and versioned writes without blocking.
- When the user asks to implement **budget-constrained retrieval** — only probe B_probe shards instead of all N shards per query.
- When designing memory for agents that operate on **different scope domains** (e.g., different customers, projects, time windows) and queries should be confined to relevant scopes.

## Key Technique

**Tiered architecture.** ShardMemo separates memory into three tiers with distinct access patterns. **Tier A** holds per-agent working state — the scratchpad of the currently active task context, kept small and fast with write serialization per agent. **Tier B** is the bulk evidence store, horizontally sharded across shard families (profile shards, observation shards, session shards), each with its own shard-local ANN index for approximate nearest-neighbor search. **Tier C** is a versioned skill library where tool definitions and learned procedures live, supporting concurrent reads via MVCC-style version stamping so queries always hit a stable snapshot while new versions are composed.

**Scope-before-routing (the masked MoE).** The critical insight is that most queries are only relevant to a subset of shards, so evaluating all shards wastes compute and introduces noise. ShardMemo attaches structured **scope tags** to every memory entry (e.g., `agent_id=agent-3`, `domain=finance`, `session=2024-Q4`). At query time, the system extracts the query's scope metadata and builds a **binary mask** over the shard space — a 1 for each shard whose scope overlaps the query, 0 otherwise. The MoE router then computes relevance scores only for unmasked shards: `selected = TopK(router_scores * mask, k=B_probe)`. This is the "masked MoE" — the gating network never wastes capacity on ineligible shards. The router is trained with evidence-to-shard supervision: given a query whose ground-truth answer lives in shard S, the router should rank S highly. Under a fixed budget of B_probe=3, this achieves +6.87 F1 over cosine-to-prototype baselines on conversational memory benchmarks and cuts vector scans by ~20%.

**Cost-aware gating across shard families.** The router is not a single softmax over all shards. It operates hierarchically: first selecting among shard families (profile vs. observation vs. session), then selecting specific shards within each family. This two-level routing lets the budget be allocated where it matters — a factual recall query routes to profile shards, a temporal "what happened last Tuesday" query routes to session shards.

## Step-by-Step Workflow

1. **Define shard families and scope tags.** Enumerate the categories your memory entries fall into (e.g., `profile`, `observation`, `session`, `skill`). For each entry, assign scope tags as key-value metadata: `{agent_id, domain, time_window, entry_type}`. These tags must be cheap to compute and deterministic.

2. **Partition entries into shards.** Assign each memory entry to exactly one shard based on its scope tags. Use consistent hashing or range partitioning on the primary scope key (e.g., `agent_id + entry_type`). Target 50-500 entries per shard for ANN efficiency; split or merge shards when they drift outside this range.

3. **Build shard-local ANN indexes.** For each shard, create an independent approximate nearest-neighbor index (FAISS IVF, HNSW, or DiskANN) over the embeddings of that shard's entries. Keep indexes in memory for hot shards, on disk for cold shards.

4. **Implement the scope-to-mask function.** Given a query, extract its scope tags. For each shard, check whether the shard's scope overlaps the query's scope. Output a binary vector `mask[i] = 1` if shard i is eligible, `0` otherwise. This is a fast metadata lookup, not a vector operation.

5. **Build the MoE router network.** Create a lightweight feedforward network (2-3 layers, hidden dim 128-256) that takes the query embedding as input and outputs a score for each shard. During inference, multiply the output scores element-wise by the binary mask, then apply `TopK(scores * mask, k=B_probe)` to select which shards to probe.

6. **Train the router with evidence-to-shard supervision.** For each training example `(query, ground_truth_shard_ids)`, compute cross-entropy loss between the router's masked softmax distribution and the ground-truth shard indicator vector. Use load-balancing auxiliary loss to prevent the router from collapsing onto a few popular shards.

7. **Implement tiered retrieval.** At query time: (a) check Tier A (agent working state) for a direct hit, (b) if not found, compute the mask, run the router, probe the top-B_probe Tier B shards via their local ANN indexes, (c) for tool/skill lookups, query Tier C's versioned index at the latest committed version.

8. **Enforce budget and admission control.** Cap total token storage per tier. When Tier B approaches its budget, run an admission gate: score incoming entries by predicted future utility (frequency * recency * relevance) and evict the lowest-scoring entries. Log evictions for debugging.

9. **Handle concurrent multi-agent access.** Use per-shard read-write locks for Tier B. Tier A uses per-agent write serialization (only the owning agent writes). Tier C uses optimistic concurrency with version numbers — readers snapshot a version, writers increment atomically.

10. **Monitor and tune.** Track per-shard hit rates, router entropy (low entropy = over-specialization), p95 retrieval latency, and mask density (fraction of shards eligible per query). Alert when mask density exceeds 50% — scope tags may need refinement.

## Concrete Examples

**Example 1: Multi-agent customer support memory**

User: "I'm building a customer support system with 20 concurrent agents. Each agent handles tickets across 5 product domains. Memory is growing to 500K entries and retrieval is slowing down. Help me implement ShardMemo-style sharded memory."

Approach:
1. Define scope tags: `{agent_id, product_domain, ticket_status, date_range}`.
2. Create shard families: `customer_profiles` (sharded by customer ID prefix), `ticket_history` (sharded by product_domain x quarter), `resolution_playbooks` (Tier C, versioned).
3. For a query like "How did we resolve billing errors for Acme Corp last month?":
   - Extract scope: `product_domain=billing, customer=acme, time_window=2025-12`.
   - Mask: eliminate all shards not in the `billing` domain and not in `2025-Q4` time range. Out of 200 shards, ~12 survive.
   - Router scores the 12 unmasked shards, selects top-3.
   - ANN search within those 3 shards retrieves the 10 most relevant entries.
4. Tier C lookup for "billing error resolution playbook" returns the latest versioned procedure.

Output:
```python
# Shard configuration
SHARD_FAMILIES = {
    "customer_profiles": {"scope_keys": ["customer_id_prefix"], "tier": "B"},
    "ticket_history":    {"scope_keys": ["product_domain", "quarter"], "tier": "B"},
    "playbooks":         {"scope_keys": ["product_domain"], "tier": "C", "versioned": True},
}

# Scope-to-mask function
def compute_mask(query_scope: dict, shard_registry: list[dict]) -> list[int]:
    mask = []
    for shard in shard_registry:
        eligible = all(
            query_scope.get(k) is None or query_scope[k] == shard["scope"].get(k)
            for k in query_scope
        )
        mask.append(1 if eligible else 0)
    return mask

# Masked routing
def masked_route(query_emb, mask, router, k=3):
    scores = router(query_emb)                    # [num_shards]
    masked_scores = scores * torch.tensor(mask)    # zero out ineligible
    top_shards = torch.topk(masked_scores, k=k)
    return top_shards.indices.tolist()
```

**Example 2: Coding agent with growing tool library**

User: "My coding agent has accumulated 2,000 tools/skills over time. Retrieval by embedding similarity is returning irrelevant tools. How do I apply ShardMemo's approach?"

Approach:
1. Tier C skill library: shard tools by `{language, domain, io_type}`. E.g., `python-data-transform`, `typescript-api-client`, `sql-query-builder`.
2. Scope tags per tool: `{language: "python", domain: "data", io_type: "transform", version: 4}`.
3. For query "parse a CSV and compute rolling averages":
   - Extract scope: `language=python, domain=data`.
   - Mask: only `python-data-*` shard family is eligible (3 shards out of 40).
   - Router selects top-1 shard (`python-data-transform`).
   - ANN within that shard returns `pandas_csv_reader`, `rolling_window_agg`, `dataframe_column_selector`.
4. Version pinning: agent reads version 4 of each tool while a background process composes version 5.

Output:
```python
# Versioned skill retrieval
def retrieve_skill(query: str, scope: dict, version: int = None):
    mask = compute_mask(scope, skill_shard_registry)
    shard_ids = masked_route(embed(query), mask, skill_router, k=1)
    shard = load_shard(shard_ids[0])
    # Version pinning: default to latest committed version
    v = version or shard.latest_committed_version
    results = shard.ann_index[v].search(embed(query), top_k=5)
    return [r for r in results if r.version <= v]
```

**Example 3: Research agent with session-scoped memory**

User: "I have a research agent that runs multi-hour sessions gathering evidence from papers. It needs to recall facts from the current session without being polluted by prior sessions."

Approach:
1. Tier A: current session's working scratchpad (last 20 observations, evict oldest).
2. Tier B session shards: one shard per session, scope tag `{session_id}`.
3. For intra-session recall, mask restricts to `session_id=current` — exactly 1 shard.
4. For cross-session recall ("Did I find anything about X in previous sessions?"), mask opens to all session shards, and the router selects the top-3 most relevant sessions.
5. Budget: cap each session shard at 10K tokens. Admission gate summarizes older entries when budget is hit.

Output:
```python
# Session-scoped masked retrieval
def recall(query: str, current_session: str, cross_session: bool = False):
    if not cross_session:
        mask = [1 if s.scope["session_id"] == current_session else 0
                for s in shard_registry]
    else:
        mask = [1] * len(shard_registry)  # all eligible, let router rank

    shard_ids = masked_route(embed(query), mask, router, k=3)
    results = []
    for sid in shard_ids:
        results.extend(shards[sid].ann_search(embed(query), top_k=5))
    return sorted(results, key=lambda r: r.score, reverse=True)[:10]
```

## Best Practices

- **Do:** Keep scope tags coarse enough that the mask eliminates at least 60-80% of shards on a typical query. If masks are too sparse (most shards eligible), add more discriminative scope keys.
- **Do:** Train the router on real query-evidence pairs from your system, not synthetic data. The router learns which shard families correlate with which query patterns.
- **Do:** Use async parallel ANN search across the selected top-k shards to hide latency — each shard search is independent.
- **Do:** Version all Tier C writes with monotonically increasing IDs and let readers snapshot a version at query start for consistent reads.
- **Avoid:** Using a single flat vector index over all memory entries. This is the centralized bottleneck ShardMemo replaces.
- **Avoid:** Setting B_probe too high. The paper shows B_probe=3 is a strong default. Higher values add latency with diminishing F1 returns.
- **Avoid:** Sharding by arbitrary hash without semantic meaning. Shards should correspond to meaningful categories so scope tags can eliminate them.

## Error Handling

- **Empty mask (all shards masked out):** The query's scope tags match no shard. Fall back to unmasked routing over all shards with a warning log. Consider whether scope tag extraction failed.
- **Router collapse (one shard gets all traffic):** Monitor shard hit-rate distribution. If entropy drops below a threshold, increase the load-balancing loss coefficient during router training.
- **Stale Tier C reads:** If a skill was recently updated but the agent reads an old version, check that the version snapshot is refreshed at session boundaries, not held indefinitely.
- **Budget overflow:** When admission control evicts entries, log which entries were dropped. If users report missing context, increase the tier budget or switch to summarization-based compression instead of eviction.
- **ANN index drift:** As entries are added/removed from a shard, the ANN index degrades. Rebuild shard indexes periodically (e.g., every 1K mutations) or use an index that supports incremental updates (HNSW).

## Limitations

- **Small memory volumes (<1K entries):** ShardMemo adds overhead for partitioning and routing that is not justified when a single ANN index suffices. Use a flat index instead.
- **Uniform-scope queries:** If most queries have identical scope (e.g., all agents share all data equally), masking provides no benefit and the router degenerates to vanilla top-k over all shards.
- **Router training data requirements:** The MoE router needs labeled query-to-shard pairs for supervision. Cold-start systems without this data should begin with heuristic routing (scope-only, no learned router) and collect training signal online.
- **Write-heavy workloads:** The architecture optimizes read routing. High-frequency writes to Tier B shards cause index rebuilds and lock contention. Consider write-ahead buffering with periodic batch insertion.
- **Not a replacement for context window management:** ShardMemo handles external memory retrieval, not how retrieved content is packed into the LLM's context window. Combine with context-packing strategies separately.

## Reference

- **Paper:** [ShardMemo: Masked MoE Routing for Sharded Agentic LLM Memory](https://arxiv.org/abs/2601.21545v1) — Zhao et al., 2026. Focus on Section 3 (tiered architecture), Section 4 (scope-before-routing and masked gating), and Table 2 (budget ablation showing B_probe=3 as the efficiency sweet spot).