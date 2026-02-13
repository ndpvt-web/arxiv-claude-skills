---
name: "pruning-minimal-reasoning-graphs"
description: >
  Build and maintain compact reasoning graphs for retrieval-augmented generation (RAG)
  that persist across queries, prune low-value structure, and minimize token usage.
  Based on the AutoPrunedRetriever architecture. Use this skill when the user says:
  "build a graph-based RAG system", "reduce RAG token costs", "incremental knowledge graph",
  "persistent reasoning graph", "prune my retrieval graph", or "efficient multi-turn RAG".
---

# Pruning Minimal Reasoning Graphs for Efficient RAG

This skill teaches Claude to design and implement AutoPrunedRetriever-style graph RAG systems that persist a minimal reasoning subgraph across queries and incrementally extend it, rather than re-retrieving and re-reasoning from scratch each time. The core idea: store entities and relations in an ID-indexed codebook, represent questions/facts/answers as compact edge sequences, apply two-layer consolidation (ANN/KNN alias detection + selective k-means) to prevent entity sprawl, and prune low-value structure so prompts contain only overlap representatives and genuinely new evidence. This achieves state-of-the-art reasoning accuracy while using up to 100x fewer tokens than graph-heavy baselines.

## When to Use

- When the user needs a RAG system that handles multi-turn or session-based question answering without re-retrieving everything each turn
- When building a knowledge graph that must stay compact over time as new documents are ingested
- When the user wants to reduce token costs in an existing graph RAG pipeline by 10-100x
- When implementing entity resolution / alias detection across a growing knowledge base
- When designing a multi-agent system where agents share a persistent, pruned reasoning substrate
- When the user asks to convert passage-based RAG into structured graph-based RAG with symbolic reuse
- When optimizing RAG for long-running sessions over evolving corpora (e.g., medical records, codebases, wiki-style knowledge)

## Key Technique

**ID-Indexed Codebook + Edge Sequences.** Instead of storing raw text passages, AutoPrunedRetriever maintains a meta-codebook `C = (E, R, M, Q, A, F, E_emb, R_emb)` where E and R are entity and relation dictionaries, M is a sparse set of unique (head, relation, tail) edges, and Q/A/F store edge-ID sequences for questions, answers, and facts. When new text arrives, a triplet parser (either REBEL or an LLM) extracts `(head, relation, tail)` triples, and an `Indexify` operation maps them into codebook indices, extending E/R/M only when genuinely new symbols appear. This means the same entity mentioned in question 1 and question 5 maps to the same integer ID -- enabling exact symbolic reuse with zero string matching overhead.

**Two-Layer Consolidation to Prevent Entity Sprawl.** Layer 1 runs continuously: an ANN-backed k-NN graph over entity embeddings connects pairs with cosine similarity above a conservative threshold, forming provisional alias groups (e.g., "NYC" and "New York City" get linked). Layer 2 fires on-demand when entity count or memory exceeds a budget: k-means refines groups and picks medoid representatives, then all edges remap through `(m_E(u), r, m_E(v))` with duplicate removal. This lazy-then-aggressive strategy keeps overhead low during normal operation while guaranteeing bounded memory.

**Pruning + Prompt Construction.** The graph is organized into "runs" -- locally coherent subgraphs built by streaming triples and splitting when a fit score (combining semantic cohesion and structural continuity) drops below a threshold. At query time, knowledge selection assigns each channel an action: "include all," "unique" (one representative per semantic cluster of runs), or "not include." The final prompt assembles `U = q ∪ a ∪ f`, extracts the minimal entity/relation sets E' and R' needed, and encodes them as either word triples or compact index triples. Token cost scales with `|E'| + |R'| + |q| + |a| + |f|` -- typically far below concatenated passages.

## Step-by-Step Workflow

1. **Define the codebook schema.** Create data structures for entity dictionary E (string-to-ID map), relation dictionary R (string-to-ID map), edge set M (set of `(head_id, rel_id, tail_id)` tuples), and embedding tables E_emb/R_emb mapping IDs to dense vectors. Use a dictionary or database backend depending on scale.

2. **Implement the triplet parser.** Choose between a dedicated extraction model (REBEL-style: fast, deterministic, no API cost) or an LLM extractor (more flexible for open-domain text). The parser takes raw text and returns a list of `(head_text, relation_text, tail_text)` triples. Normalize entity names to lowercase, strip articles, and collapse whitespace before indexing.

3. **Build the Indexify operation.** For each extracted triple, look up head/tail in E and relation in R. If a symbol is new, assign the next integer ID, compute its embedding, and insert into the codebook. Create the edge `(head_id, rel_id, tail_id)` in M if it doesn't already exist. Return the sequence of edge IDs representing the input text.

4. **Implement Layer-1 alias detection (ANN/KNN).** Build an approximate nearest-neighbor index (FAISS, Annoy, or hnswlib) over E_emb. For each new entity, query the k nearest neighbors. If cosine similarity exceeds threshold `tau_E` (start with 0.92), link them into a provisional alias group using union-find. Do NOT merge yet -- just record the alias relationship.

5. **Implement Layer-2 consolidation (k-means).** When `|E|` exceeds a configurable budget (e.g., 10,000 entities), trigger k-means over alias groups with k set to the number of provisional groups. Pick the medoid (closest to centroid) as the canonical representative `m_E(u)`. Remap all edges through the medoid mapping and deduplicate M. Update the ANN index with consolidated embeddings.

6. **Organize edges into runs.** Stream incoming triples into a current run. For each triple, compute a fit score: `fit = alpha * cosine(triple_emb, run_centroid) + beta * continuity_bonus` where continuity_bonus is 1.0 if the triple shares a head or tail with the current run's last edge, else 0.0. When fit drops below threshold `tau_run` (start with 0.4), close the current run, store its edge sequence in `Runs`, and start a new run.

7. **Implement knowledge selection for queries.** Given a new query q, parse it into edge sequences, then retrieve candidate runs by embedding similarity. For each retrieved run, assign an action: "include all" if the run is critical context (high overlap with query edges), "unique" if multiple runs cover the same semantic cluster (keep one representative per cluster via embedding-space clustering), or "not include" if the run adds no edges not already covered.

8. **Construct the compact prompt.** Assemble the union `U = q_edges ∪ selected_answer_edges ∪ selected_fact_edges`. Extract the minimal E' and R' (only entities and relations referenced by U). Encode as either readable word triples `(head_name, relation_name, tail_name)` for debugging, or compact index triples `(e1, r1, e2)` with a legend mapping IDs to names for production. Prepend a system instruction explaining the triple format to the LLM.

9. **Handle incremental extension.** When a new question arrives in the same session, do NOT rebuild the graph. Instead: parse the new question, Indexify its triples (reusing existing codebook entries), add new runs, retrieve relevant prior runs via step 7, and construct the prompt including both prior context and new evidence. The graph grows monotonically but stays compact via consolidation.

10. **Monitor and tune.** Track token count per prompt, entity count over time, alias detection precision (spot-check merged entities), and answer accuracy. If tokens creep up, lower `tau_run` to produce fewer runs or tighten knowledge selection. If accuracy drops, raise `tau_E` to be more conservative about alias merging.

## Concrete Examples

**Example 1: Multi-turn medical QA system**

User: "Build a RAG system for a medical knowledge base that handles follow-up questions efficiently."

Approach:
1. Parse the medical corpus (e.g., PubMed abstracts) through the triplet extractor to produce triples like `(metformin, treats, type_2_diabetes)`, `(type_2_diabetes, symptom, polyuria)`.
2. Indexify all triples into the codebook. Initial corpus of 1,000 abstracts yields ~8,000 unique entities and ~2,500 relations.
3. Run Layer-1 alias detection: "T2DM" and "type 2 diabetes" get linked (cosine 0.96). "DM2" also joins the group.
4. User asks Q1: "What are the first-line treatments for type 2 diabetes?" -- retrieve runs containing `type_2_diabetes` edges, construct prompt with ~120 tokens of triples instead of ~2,000 tokens of passages.
5. User asks Q2: "Do any of those treatments cause lactic acidosis?" -- reuse the entity IDs from Q1 (metformin, type_2_diabetes already indexed), add only the new `(metformin, side_effect, lactic_acidosis)` edges, prompt includes overlap representatives from Q1 plus new evidence.

Output (prompt sent to LLM):
```
Context triples:
(metformin, first_line_treatment_for, type_2_diabetes)
(metformin, mechanism, inhibits_hepatic_glucose_production)
(metformin, side_effect, lactic_acidosis)
(lactic_acidosis, risk_factor, renal_impairment)
(type_2_diabetes, also_treated_by, sulfonylureas)

Question: Do any first-line treatments for type 2 diabetes cause lactic acidosis?
```
Token count: ~85 tokens vs. ~1,800 tokens for passage-based RAG.

**Example 2: Codebase knowledge graph for a multi-agent dev pipeline**

User: "I want agents to share a persistent knowledge graph about our codebase so they don't re-analyze the same modules repeatedly."

Approach:
1. Parse code documentation + commit messages into triples: `(AuthService, depends_on, UserRepository)`, `(UserRepository, uses, PostgreSQL)`, `(AuthService, exposes, /api/login)`.
2. Each agent indexes its findings via Indexify. Agent A analyzes auth module; Agent B analyzes payments. Both reference `UserRepository` -- same codebook ID, no duplication.
3. When Agent C asks "What services depend on PostgreSQL?" -- traverse edges from `PostgreSQL` backward through `uses` relations. No re-analysis needed.
4. After 50 agent sessions, Layer-2 consolidation fires: merges `pg`, `postgres`, `PostgreSQL` into one canonical entity. Edge count drops 15%.

Output (shared graph state):
```json
{
  "entities": {"e1": "AuthService", "e2": "UserRepository", "e3": "PostgreSQL", "e4": "PaymentService"},
  "relations": {"r1": "depends_on", "r2": "uses", "r3": "exposes"},
  "edges": [
    ["e1", "r1", "e2"],
    ["e2", "r2", "e3"],
    ["e4", "r1", "e2"],
    ["e4", "r2", "e3"]
  ],
  "session_runs": 50,
  "consolidation_events": 1,
  "total_tokens_saved_vs_passage_rag": "~340,000"
}
```

**Example 3: Reducing token costs in an existing RAG pipeline**

User: "Our RAG system uses 4,000 tokens per query on average. Can we cut that down?"

Approach:
1. Audit the current pipeline: identify that passages are being concatenated verbatim into prompts.
2. Insert a triplet extraction step between retrieval and prompting. Extract `(entity, relation, entity)` triples from retrieved passages.
3. Build a session-scoped codebook. For the first query, token cost may be similar (~3,800 tokens as triples + legend). But by query 5, most entities are already indexed -- prompts shrink to ~400-800 tokens.
4. Add Layer-1 alias detection to prevent near-duplicate entities from inflating the codebook.
5. Implement the "unique" selection policy: when multiple retrieved passages yield overlapping runs, keep only one representative per semantic cluster.

Output (before/after comparison):
```
Query 1:  Passage RAG: 4,200 tokens | Graph RAG: 3,800 tokens (1.1x savings)
Query 5:  Passage RAG: 4,100 tokens | Graph RAG:   820 tokens (5.0x savings)
Query 20: Passage RAG: 4,300 tokens | Graph RAG:   390 tokens (11.0x savings)
Query 50: Passage RAG: 4,150 tokens | Graph RAG:   210 tokens (19.8x savings)
```

## Best Practices

- **Do:** Start with a high alias threshold (tau_E >= 0.90) and lower it gradually. Aggressive merging early causes irrecoverable entity conflation (e.g., merging "Java" the language with "Java" the island).
- **Do:** Use the REBEL-style extractor for structured/technical domains where relation types are predictable. Use the LLM extractor for open-domain or conversational text where relations are diverse and implicit.
- **Do:** Store the codebook in a persistent format (SQLite, Redis, or a simple JSON file) so it survives across sessions. The entire value proposition depends on persistence.
- **Do:** Log every consolidation event with before/after entity counts and spot-check merged groups. This is your primary quality assurance mechanism.
- **Avoid:** Skipping the run-organization step and treating all triples as a flat bag. Runs provide locality structure that makes knowledge selection dramatically more precise.
- **Avoid:** Setting the memory budget too low. Premature k-means consolidation on a small codebook produces poor clusters. Wait until you have at least 1,000 entities before triggering Layer-2.

## Error Handling

- **Triplet extraction returns empty results:** Fall back to noun-phrase extraction (spaCy noun chunks + dependency parse relations) as a degraded but functional alternative. Log the failure for corpus-specific parser tuning.
- **Alias detection merges entities that should be distinct:** Implement a "split" operation that undoes a merge by restoring original entity IDs from the alias group history. Raise tau_E for the affected embedding region.
- **Codebook grows unbounded despite consolidation:** Check that Layer-2 is actually triggering (verify the memory threshold is set correctly). If it triggers but doesn't shrink enough, reduce k in k-means or lower tau_E to merge more aggressively.
- **Prompt token count exceeds model context window:** The knowledge selection step should enforce a hard token budget. If the minimal E'/R'/edge set still exceeds the budget, rank edges by query relevance (embedding similarity of edge to query) and truncate from the bottom.
- **Graph has stale or contradictory edges:** Implement a timestamp on each edge and a staleness threshold. During knowledge selection, prefer more recent edges. For contradictions, include both edges with their timestamps and let the LLM adjudicate.

## Limitations

- **Single-entity queries gain little.** If every query is a simple factoid lookup ("What is the capital of France?"), the overhead of graph construction exceeds the savings. This technique shines on multi-hop reasoning and session-based QA.
- **Triplet extraction quality is a bottleneck.** If the parser misses key relations or hallucinates edges, the entire downstream graph is corrupted. Domain-specific fine-tuning of the extractor is often necessary.
- **Not suitable for verbatim generation tasks.** Because the prompt contains structured triples rather than raw passages, the LLM cannot quote or paraphrase source text directly. For tasks requiring faithful summarization of specific passages, retain a passage-level fallback.
- **Cold start per session.** The first query in a new session sees no token savings because the graph is empty. Benefits compound over subsequent queries in the same session or across sessions if the codebook persists.
- **Entity disambiguation requires embeddings.** The alias detection layers depend on quality entity embeddings. For niche domains with poor embedding coverage, alias detection degrades and manual synonym lists may be needed as a supplement.

## Reference

- **Paper:** [Pruning Minimal Reasoning Graphs for Efficient Retrieval-Augmented Generation](https://arxiv.org/abs/2602.04926v1) -- Wang et al., 2026. Focus on Section 3 (codebook + Indexify formalization), Section 4 (two-layer consolidation algorithm), and Table 2 (token efficiency comparisons across benchmarks).