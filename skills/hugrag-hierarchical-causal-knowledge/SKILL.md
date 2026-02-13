---
name: "hugrag-hierarchical-causal-knowledge"
description: "Build hierarchical causal knowledge graphs for RAG pipelines that suppress spurious correlations and enable cross-document causal reasoning. Use when: 'build a causal knowledge graph from my documents', 'add causal reasoning to my RAG pipeline', 'set up graph-based RAG with causal filtering', 'create a hierarchical knowledge graph for retrieval', 'implement causal gating for my knowledge base', 'improve RAG with structured causal retrieval'."
---

# HugRAG: Hierarchical Causal Knowledge Graph for RAG

This skill enables Claude to build RAG systems that organize knowledge into hierarchical causal graphs rather than flat vector stores or naive entity graphs. Based on the HugRAG framework, the core idea is to partition a knowledge graph into hierarchical modules using community detection, then establish **causal gates** between modules -- sparse, LLM-verified causal edges that enable logically justified cross-module retrieval. During query time, a gated priority traversal expands through these causal pathways, and a causal path refinement step filters spurious correlations from the retrieved evidence before answer generation. This produces answers grounded in explicit causal chains rather than surface-level semantic similarity.

## When to Use

- When the user wants to build a RAG system over a multi-document corpus and needs answers that reflect causal relationships (e.g., "why did X cause Y?"), not just keyword overlap
- When the user asks to construct a knowledge graph from documents with hierarchical community structure and cross-topic causal links
- When an existing graph-RAG pipeline returns spurious or unfaithful answers because it retrieves semantically similar but causally irrelevant evidence
- When the user needs cross-document reasoning -- connecting evidence scattered across separate sources through explicit causal chains
- When the user wants to implement a retrieval pipeline that distinguishes genuine causal dependencies from coincidental co-occurrence
- When building a question-answering system for domains where causal accuracy matters (medical, legal, engineering root-cause analysis)

## Key Technique

Standard graph-based RAG builds an entity graph and retrieves subgraphs by node similarity. This fails when the answer requires reasoning across distant parts of the graph or across documents -- surface-level matching finds correlated but not causally related nodes, producing hallucinated causal claims.

HugRAG solves this with three innovations. First, **hierarchical partitioning**: the base entity graph is recursively partitioned using the Leiden community detection algorithm into L levels of increasingly coarse modules, each annotated with an LLM-generated natural language summary. Second, **causal gating**: instead of connecting all module pairs, an LLM evaluates whether a genuine causal dependency exists between topologically distant module summaries, creating a sparse set of causal gate edges. These gates enable logically justified jumps between otherwise disconnected parts of the graph. Third, **gated priority retrieval with causal filtering**: during query time, seeds are selected at multiple granularity levels (fine-grained entities and coarse module summaries), then a best-first-search traversal expands through structural, hierarchical, and causal edges with a decay-weighted gain function that prioritizes causal gates. The raw retrieved subgraph is then refined by an LLM "causal expert" that identifies valid causal paths and explicitly flags spurious correlations.

The unified edge space combines three edge types: structural edges (entity-to-entity within documents), hierarchical edges (vertical links between hierarchy levels), and causal gates (cross-module logical pathways). This tri-layer connectivity is what enables both local precision and global causal reach.

## Step-by-Step Workflow

### Phase 1: Offline Knowledge Graph Construction

1. **Extract entities and relations from the corpus.** Process each document through an NER + relation extraction pipeline (LLM-based or spaCy/custom). Produce a base graph `G_0 = (V_0, E_0)` where nodes are entities and edges are extracted relations. Deduplicate entities using canonicalization (string normalization, embedding similarity, or LLM-based coreference).

2. **Partition the graph hierarchically using Leiden.** Apply the Leiden community detection algorithm iteratively to produce L hierarchy levels `H_0, H_1, ..., H_L`. Level `H_0` contains individual entity nodes; each subsequent level groups nodes into coarser modules. Typical values: L=2-4 depending on corpus size. Store the module membership mapping at each level.

3. **Generate module summaries.** For each module at each level, concatenate the text spans associated with its member entities and use an LLM to produce a concise natural language summary (2-4 sentences). These summaries serve as semantic anchors for the module.

4. **Identify causal gates between modules.** For candidate module pairs (prioritize topologically distant pairs that share thematic overlap), prompt an LLM with both module summaries and ask: "Does module A causally influence module B? Provide a yes/no judgment and a one-sentence justification." Apply a confidence threshold tau (e.g., 0.7). Store accepted pairs as directed causal gate edges `G_c = {(m_i -> m_j)}` with scores.

5. **Build the unified edge index.** Merge structural edges `E_struc`, hierarchical containment edges `E_hier`, and causal gates `G_c` into a single traversable graph structure. Assign edge-type weights: structural=1.0, hierarchical=1.2, causal=1.5 (tunable).

### Phase 2: Online Retrieval and Reasoning

6. **Seed selection via multi-granular hybrid scoring.** For an incoming query, compute hybrid scores combining dense embedding similarity and sparse lexical overlap (BM25) against both fine-grained entities (H_0) and coarse module summaries (H_l>0). Select top-K seeds at each level using MMR (Maximal Marginal Relevance) to ensure diversity.

7. **Gated priority expansion.** Run best-first search from seed nodes over the unified graph. Score each candidate neighbor with: `Gain(v) = similarity(query, v) * gamma^depth * weight(edge_type)` where gamma in (0,1) is a decay factor (e.g., 0.85). Expand up to a budget of N nodes (e.g., 50-200). Causal gates get higher weight, so the traversal preferentially follows causal pathways.

8. **Linearize the retrieved subgraph.** Convert the raw retrieved subgraph `S_raw` into a token-efficient tabular or triple-list format suitable for LLM consumption. Include node descriptions, edge types, and edge labels.

9. **Causal path refinement.** Prompt an LLM as a "causal expert" with the linearized subgraph and the query. Instruct it to: (a) identify all paths from query-relevant nodes to candidate answer nodes, (b) classify each path as causally valid or spurious, (c) return only the causally valid subgraph `S*` with justifications.

10. **Generate the final answer.** Prompt the LLM with the query and the refined causal subgraph `S*`, instructing it to ground its answer in the provided causal evidence and explicitly note which causal chain supports each claim.

## Concrete Examples

**Example 1: Building a causal knowledge graph for incident analysis**

User: "I have 200 post-mortem reports from production incidents. Build a knowledge graph that lets me ask causal questions like 'what deployment patterns cause cascading failures?'"

Approach:
1. Extract entities (services, error types, deployment events, metrics) and relations (triggered, caused, detected, mitigated) from each report using an LLM extraction prompt
2. Deduplicate entities -- e.g., "auth-service" and "authentication service" map to the same node
3. Run Leiden partitioning with L=3 levels, producing clusters like {deployment-related incidents}, {network failures}, {database timeouts}
4. Generate module summaries: "This cluster covers incidents where rolling deployments to the payment service triggered downstream timeouts in the order pipeline"
5. Identify causal gates: the LLM confirms that the "config drift" module causally influences the "cascading timeout" module, creating a gate edge
6. At query time, "what deployment patterns cause cascading failures?" seeds into both the deployment module and the failure module, traverses the causal gate connecting them, and retrieves the specific incident chains

Output structure:
```python
# Causal subgraph returned for the query
{
  "causal_paths": [
    {
      "path": ["rolling_deploy_without_canary", "connection_pool_exhaustion", "cascading_timeout"],
      "edge_types": ["causal_gate", "structural"],
      "evidence_docs": ["postmortem-042", "postmortem-117"],
      "confidence": 0.89
    },
    {
      "path": ["config_drift_after_deploy", "stale_cache_entries", "auth_failures"],
      "edge_types": ["causal_gate", "causal_gate"],
      "evidence_docs": ["postmortem-023", "postmortem-091"],
      "confidence": 0.82
    }
  ],
  "answer": "Two deployment patterns consistently cause cascading failures: (1) rolling deployments without canary phases exhaust connection pools, and (2) configuration drift after deployment leads to stale cache entries causing auth failures."
}
```

**Example 2: Medical literature causal RAG**

User: "Set up a RAG pipeline over these 500 medical papers so I can ask 'does drug X affect condition Y and through what mechanism?'"

Approach:
1. Extract biomedical entities (drugs, proteins, conditions, pathways) using domain NER, and relations (inhibits, upregulates, treats, causes) using relation extraction
2. Build base graph, then Leiden-partition with L=3: level 0 = individual proteins/drugs, level 1 = pathway-level modules, level 2 = disease-area modules
3. Summarize modules: "This module covers TNF-alpha signaling pathway components and their role in inflammatory response"
4. Establish causal gates: LLM confirms "TNF-alpha signaling module" causally influences "rheumatoid arthritis progression module" across papers
5. Query "Does methotrexate affect rheumatoid arthritis progression?" seeds into methotrexate (H_0) and RA module (H_2), traverses causal gate through TNF-alpha pathway, retrieves mechanistic chain
6. Causal expert filters out spurious paths (e.g., methotrexate and RA co-mentioned in unrelated trial metadata)

Output:
```
Causal chain: methotrexate --[inhibits]--> DHFR --[reduces]-->
T-cell proliferation --[suppresses]--> TNF-alpha production
--[reduces]--> synovial inflammation --[slows]--> RA progression

Sources: [Paper-12, Paper-87, Paper-203]
Spurious path filtered: methotrexate co-mentioned with RA in
trial enrollment criteria (not a causal relationship)
```

**Example 3: Adding causal gating to an existing GraphRAG codebase**

User: "I already have a GraphRAG pipeline with entity extraction and community summaries. How do I add HugRAG-style causal gating?"

Approach:
1. Keep existing entity extraction and Leiden partitioning
2. Add a causal gate identification step after community summarization:
```python
async def identify_causal_gates(modules, llm_client, threshold=0.7):
    gates = []
    # Only evaluate topologically distant pairs (skip adjacent modules)
    candidates = get_distant_module_pairs(modules, min_hops=2)
    for m_i, m_j in candidates:
        prompt = f"""Given these two knowledge modules:
Module A: {m_i.summary}
Module B: {m_j.summary}

Does Module A causally influence Module B?
Reply with: YES/NO, confidence (0-1), one-sentence justification."""
        response = await llm_client.generate(prompt)
        if response.confidence >= threshold:
            gates.append(CausalGate(source=m_i, target=m_j,
                                     score=response.confidence,
                                     justification=response.reason))
    return gates
```
3. Modify traversal to include gate edges with boosted weight
4. Add a post-retrieval causal filtering prompt before answer generation

## Best Practices

- **Do:** Prioritize causal gate candidates between topologically distant modules. Adjacent modules already share structural edges -- the value of causal gates is connecting otherwise disconnected knowledge regions.
- **Do:** Use MMR-based diversity in seed selection to avoid retrieving only from one cluster. Seeds should span multiple hierarchy levels and multiple modules.
- **Do:** Set the decay factor gamma between 0.8-0.9. Too low (< 0.7) restricts retrieval to local neighborhoods; too high (> 0.95) allows noisy long-range paths.
- **Do:** Cache causal gate evaluations. They are expensive (one LLM call per candidate pair) but stable -- recompute only when module summaries change.
- **Avoid:** Evaluating all O(n^2) module pairs for causal gates. Pre-filter candidates using embedding similarity between summaries (cosine > 0.3) combined with topological distance (> 2 hops). This reduces LLM calls by 80-90%.
- **Avoid:** Skipping the causal path refinement step. Without it, the gated traversal may still include spurious paths that happen to pass through causal gates. The LLM causal expert is the final quality filter.

## Error Handling

- **Entity extraction noise:** Deduplication failures create disconnected subgraphs. Run entity canonicalization aggressively (embedding similarity > 0.85 triggers merge). Log merge decisions for audit.
- **False causal gates:** LLMs can hallucinate causal relationships. Mitigate by requiring bidirectional consistency (if A->B is causal, B->A should not also be causal at high confidence) and by sampling multiple LLM evaluations per candidate pair.
- **Hierarchy too deep:** With L > 4, top-level modules become so coarse that their summaries are generic and causal gate identification becomes unreliable. Cap at L=3 for corpora under 10K documents.
- **Retrieval budget exhaustion:** If the gated traversal fills its budget before reaching answer-relevant nodes, increase the budget or add more seeds at coarser hierarchy levels to provide better starting points.
- **Empty causal subgraph after filtering:** If the causal expert rejects all paths, fall back to the standard (non-causal) retrieved subgraph but flag the answer as "no causal chain confirmed."

## Limitations

- **LLM cost for gate construction:** Evaluating causal gates requires O(candidate_pairs) LLM calls during the offline phase. For large corpora (>50K documents), this becomes a significant cost. Batch processing and caching are essential.
- **Causal gate quality depends on summary quality:** If module summaries are vague or overly generic, the LLM cannot reliably assess causal relationships. Summary quality is the bottleneck.
- **Not suitable for rapidly changing corpora:** The offline phase (partitioning + gate identification) is expensive to rerun. Best for corpora that update infrequently (weekly or less).
- **Causal reasoning is approximate:** The LLM-based causal assessment is a heuristic, not formal causal inference. It cannot distinguish true causation from strong confounded correlation in all cases.
- **Overkill for simple factoid QA:** If queries are simple entity lookups ("What is the capital of France?"), the hierarchical causal machinery adds latency without benefit. Use standard vector RAG for factoid retrieval.

## Reference

[HugRAG: Hierarchical Causal Knowledge Graph Design for RAG](https://arxiv.org/abs/2602.05143v1) -- Focus on Section 3 (framework architecture), Algorithm 1 (full pipeline pseudocode), and the causal gate construction criteria in Section 3.2 for implementation details.