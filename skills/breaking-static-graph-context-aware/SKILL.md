---
name: "breaking-static-graph-context-aware"
description: "Build query-adaptive knowledge graph retrieval systems using CatRAG's context-aware traversal. Transforms static KG-based RAG pipelines into dynamic, query-sensitive retrieval that recovers complete multi-hop evidence chains. Use when: 'build a multi-hop RAG pipeline', 'improve knowledge graph retrieval', 'fix semantic drift in graph search', 'implement context-aware graph traversal', 'retrieve complete evidence chains from a KG', 'add query-dependent edge weighting to my graph'."
---

# Context-Aware Graph Traversal for Robust Multi-Hop RAG (CatRAG)

This skill enables Claude to build retrieval-augmented generation systems that use **query-adaptive knowledge graph traversal** instead of static graph search. Based on the CatRAG framework, it addresses the "Static Graph Fallacy" — where fixed edge weights cause random walks to drift into high-degree hub nodes, retrieving partial context but missing the complete evidence chain needed for multi-hop reasoning. The technique modifies Personalized PageRank with three mechanisms: symbolic anchoring, dynamic edge weighting, and passage weight enhancement, turning any knowledge graph into a query-sensitive navigation structure.

## When to Use

- When building a RAG system that must answer **multi-hop questions** requiring evidence from 2+ documents (e.g., "Which film directed by the person who founded Company X won an award in 2019?")
- When an existing KG-based retrieval pipeline returns **partially correct but incomplete** context — high recall on individual facts but missing the full reasoning chain
- When graph traversal keeps landing on **hub nodes** (high-degree entities like "United States" or "film") that dilute retrieval relevance
- When implementing or extending a HippoRAG-style pipeline and want to add query-dependent scoring
- When the user asks to build a knowledge graph retrieval system that handles **implicit semantic relationships**, not just keyword matching
- When refactoring a vector-similarity RAG pipeline to incorporate structured multi-hop reasoning

## Key Technique

**The Static Graph Fallacy.** Systems like HippoRAG construct a knowledge graph during indexing and use Personalized PageRank (PPR) with fixed transition probabilities to retrieve relevant passages. The problem: edge weights are set once and never change. A relation edge between "Christopher Nolan" and "film" gets the same weight regardless of whether the query is about his filmography or his childhood. High-degree nodes accumulate disproportionate random walk probability, causing "semantic drift" — the walk reaches a hub and scatters instead of following the evidence chain.

**CatRAG's Three Interventions.** (1) **Symbolic Anchoring** extracts named entities from the query via NER and injects them as weak reset seeds (weight epsilon=0.2, scaled by inverse passage frequency) into the PPR teleportation vector. This creates gravitational pull toward query-specific entities without overwhelming the semantic signal from triple-based retrieval. (2) **Query-Aware Dynamic Edge Weighting** uses a coarse-to-fine strategy: first prune edges on dense nodes (>15 outgoing edges) by embedding similarity, then score surviving edges with an LLM classifier into tiers {Irrelevant: 0, Weak: 0.2-0.3, High: 2.0-3.0, Direct: 5.0}, multiplied against the static weight. This makes the transition matrix query-specific. (3) **Key-Fact Passage Weight Enhancement** boosts edges between seed entities and passages containing verified seed triples by factor beta=2.5, requiring zero LLM calls — a pure algorithmic shortcut that anchors the walk to likely evidence passages.

**Why it matters for implementation.** The three mechanisms are modular. You can apply symbolic anchoring alone for a quick improvement, add passage weight enhancement for zero-cost gains, and layer on dynamic edge weighting when LLM inference budget allows. The PPR equation stays the same — only the transition matrix and teleportation vector change per query.

## Step-by-Step Workflow

1. **Construct the knowledge graph from your document corpus.** Use OpenIE (or an LLM extraction prompt) to extract (subject, relation, object) triples from each passage. Create three node types: entity nodes (V_E), passage nodes (V_P). Create three edge types: relation edges between entities from the same triple, synonym edges between entities with embedding cosine similarity > 0.8 (weight = 2.0 * similarity), and context edges from passage nodes to every entity they contain.

2. **Build the static transition matrix.** Normalize outgoing edge weights for each node to form transition probabilities. Store the graph in an adjacency structure that supports per-query weight overrides (e.g., a sparse matrix where weights can be masked or multiplied).

3. **Process the incoming query with dual-channel seed identification.** Channel A: embed the query and retrieve the top-N_seed (default 5) entity nodes by cosine similarity between query embedding and triple embeddings. Channel B: run NER on the query to extract explicit entity mentions, then fuzzy-match them to graph entity nodes.

4. **Apply Symbolic Anchoring.** Construct the PPR teleportation vector. Assign Channel A seeds their retrieval-score-based probabilities. For Channel B (NER) entities, inject each with weight epsilon=0.2, scaled by the inverse of the entity's passage count (entities appearing in fewer passages get relatively higher anchor weight). Normalize the combined teleportation vector to sum to 1.

5. **Apply Coarse-Grained Edge Pruning.** For each of the top-N_seed entity nodes, check if the node has more than K_edge (default 15) outgoing relation edges. If so, compute cosine similarity between the query embedding and each neighbor's fact embedding. Keep the top-K_edge edges; demote the rest to "Weak" weight (0.2).

6. **Apply Fine-Grained Dynamic Edge Weighting.** For surviving edges from step 5, call an LLM with the query, source entity, target entity, and target entity's context (summarized if the entity has more than tau triples, concatenated raw otherwise). The LLM classifies the relationship relevance as {Irrelevant, Weak, High, Direct}. Multiply the static edge weight by the tier multiplier: Irrelevant=0, Weak=0.25, High=2.5, Direct=5.0.

7. **Apply Key-Fact Passage Weight Enhancement.** For every context edge connecting a seed entity to a passage node, check if the passage contains a verified seed triple (a triple that contributed to the seed identification in step 3). If yes, multiply the edge weight by (1 + beta), where beta=2.5. This is a pure lookup — no LLM calls needed.

8. **Run modified PPR.** Execute Personalized PageRank on the modified graph with damping factor d=0.5 and the anchored teleportation vector from step 4. Iterate until convergence (typically 10-20 iterations). Rank passage nodes by their final PPR score.

9. **Retrieve and assemble context.** Select the top-K passage nodes (default K=5). Extract their source text. Pass them to the downstream LLM as retrieved context for answering the query.

10. **Evaluate with chain-complete metrics.** Beyond standard Recall@K, measure Full Chain Retrieval (FCR) — the fraction of queries where ALL required evidence passages appear in the top-K — and Joint Success Rate (JSR) — FCR multiplied by answer correctness. These metrics expose the partial-retrieval failures that standard recall hides.

## Concrete Examples

**Example 1: Multi-hop question answering over a company knowledge base**

User: "Build a retrieval pipeline that can answer questions like 'What award did the CEO of the company that acquired Tableau receive in 2020?' across our 10K corporate filings."

Approach:
1. Extract triples from filings: ("Salesforce", "acquired", "Tableau"), ("Marc Benioff", "is CEO of", "Salesforce"), ("Marc Benioff", "received", "Innovation Award 2020")
2. Build KG with entity and passage nodes. "Salesforce" becomes a high-degree hub (appears in hundreds of passages).
3. On query, NER extracts "Tableau" and "2020". Triple retrieval finds seed entities: "Tableau", "acquired", "award".
4. Symbolic anchoring: "Tableau" gets epsilon=0.2 weight in teleportation vector (low passage count = high relative anchor). This prevents the walk from scattering at the "Salesforce" hub.
5. Dynamic edge weighting: the edge "Salesforce"→"cloud computing" is classified Irrelevant (weight 0) for this query, while "Salesforce"→"Marc Benioff" is classified Direct (weight 5.0).
6. Passage weight enhancement: passages containing the triple ("Salesforce", "acquired", "Tableau") get 3.5x edge boost.
7. PPR converges with high probability on the three passages containing the complete evidence chain.

Output: Retrieved passages cover all three hops — acquisition, CEO identity, and award — enabling a correct grounded answer.

**Example 2: Fixing an existing HippoRAG pipeline with hub drift**

User: "Our KG retrieval keeps returning Wikipedia-style overview passages instead of specific evidence. How do I fix it?"

Approach:
1. Diagnose hub bias: compute PPR-weighted node strength (sum of incoming PPR * degree). Identify nodes where top-1% accumulate >40% of probability mass.
2. Add symbolic anchoring to the teleportation vector — inject NER-extracted entities with epsilon=0.2 weight.
3. Add passage weight enhancement (beta=2.5) for passages containing seed triples — this is zero-cost and typically recovers 30-50% of the gap.
4. If budget allows, add coarse pruning (K_edge=15) on hub nodes, then fine-grained LLM edge classification.

Output:
```python
# Before: teleportation vector based only on triple retrieval
teleport = retrieval_scores / retrieval_scores.sum()

# After: inject NER anchors with symbolic anchoring
ner_entities = extract_entities(query)
for ent in ner_entities:
    node_id = fuzzy_match(ent, graph.entity_nodes)
    if node_id is not None:
        inv_freq = 1.0 / graph.passage_count(node_id)
        teleport[node_id] += EPSILON * inv_freq
teleport = teleport / teleport.sum()

# Passage weight enhancement (no LLM calls)
BETA = 2.5
for seed_triple in seed_triples:
    for passage_id in graph.passages_containing(seed_triple):
        edge = graph.get_edge(seed_triple.entity, passage_id)
        edge.weight *= (1 + BETA)
```

**Example 3: Implementing the LLM edge classifier**

User: "How do I implement the dynamic edge weighting LLM call?"

Approach:
1. Design the classification prompt with the four tiers.
2. Batch edges per seed node for efficiency.
3. Parse the LLM response and apply multipliers.

Output:
```python
TIER_WEIGHTS = {"Irrelevant": 0.0, "Weak": 0.25, "High": 2.5, "Direct": 5.0}

EDGE_CLASSIFY_PROMPT = """Given the query: "{query}"
Evaluate the relevance of traversing from entity "{source}" to entity "{target}".
Context about target entity: {context}

Classify this edge as exactly one of: Irrelevant, Weak, High, Direct.
- Direct: target entity is explicitly needed to answer the query
- High: target entity provides important supporting context
- Weak: target entity is tangentially related
- Irrelevant: target entity has no bearing on the query

Classification:"""

def score_edges(query, seed_node, neighbors, graph, llm):
    context_map = {}
    for nbr in neighbors:
        facts = graph.get_triples(nbr)
        if len(facts) > TAU:
            context_map[nbr] = llm.summarize(facts)
        else:
            context_map[nbr] = " | ".join(str(f) for f in facts)

    for nbr in neighbors:
        prompt = EDGE_CLASSIFY_PROMPT.format(
            query=query, source=seed_node,
            target=nbr, context=context_map[nbr]
        )
        tier = llm.classify(prompt, temperature=0.0)
        multiplier = TIER_WEIGHTS.get(tier.strip(), 0.25)
        graph.edges[seed_node][nbr]["weight"] *= multiplier
```

## Best Practices

- **Do:** Apply the three mechanisms incrementally. Symbolic anchoring + passage weight enhancement require zero extra LLM calls and provide the majority of the improvement. Add dynamic edge weighting only when retrieval completeness still falls short.
- **Do:** Use inverse passage frequency to scale symbolic anchor weights. Rare entities (appearing in 1-2 passages) deserve stronger anchoring than ubiquitous ones.
- **Do:** Set the LLM temperature to 0.0 for edge classification to get deterministic, reproducible tier assignments.
- **Do:** Measure Full Chain Retrieval (FCR) and Joint Success Rate (JSR), not just Recall@K. Standard recall masks partial-retrieval failures that break downstream reasoning.
- **Avoid:** Setting epsilon too high (>0.5) for symbolic anchoring — this overrides the semantic retrieval signal and collapses results to exact-match behavior.
- **Avoid:** Running fine-grained LLM edge scoring on all edges. Use coarse pruning first (K_edge=15) to limit LLM calls to ~75 edges per query (5 seeds * 15 edges) instead of thousands.
- **Avoid:** Skipping the synonym edge layer (threshold 0.8). Paraphrase and alias connections are critical for multi-hop traversal where the same entity appears under different surface forms.

## Error Handling

- **NER extracts no entities from query:** Fall back to pure triple-retrieval seeds. Symbolic anchoring is skipped — the system degrades gracefully to HippoRAG 2 behavior.
- **LLM edge classifier returns unparseable output:** Default to "Weak" tier (multiplier 0.25) for safety. Log the failure for review. Consider adding a retry with a more constrained prompt.
- **PPR fails to converge in 20 iterations:** The modified transition matrix may have near-zero outgoing weight on some nodes. Add a small floor weight (1e-6) to all edges to ensure ergodicity, or increase the damping factor d slightly.
- **Hub nodes still dominate after all three interventions:** Check if K_edge is too high for your domain. For densely connected KGs (avg degree > 50), reduce K_edge to 8-10. Also verify that synonym edges aren't creating shortcut paths that bypass the pruning.
- **Passage weight enhancement over-boosts a single passage:** Cap the maximum boost at 2 * (1 + beta) to prevent a single passage from dominating the PPR ranking when it contains multiple seed triples.

## Limitations

- **LLM inference cost at query time.** Dynamic edge weighting requires an LLM call per edge (after pruning). For latency-sensitive applications, consider using symbolic anchoring + passage weight enhancement only (zero extra LLM calls) and accept the smaller improvement.
- **Knowledge graph quality dependency.** CatRAG cannot recover from poor OpenIE extraction. If triples are noisy or incomplete, the graph structure itself is flawed and no traversal strategy can fix it. Invest in extraction quality first.
- **Single-hop queries see minimal benefit.** The framework is designed for multi-hop reasoning. For single-hop factoid retrieval, standard dense retrieval or basic KG lookup is simpler and equally effective.
- **Assumes entity-centric queries.** Queries about abstract concepts, sentiments, or temporal patterns without clear entity anchors will not benefit from symbolic anchoring. The dynamic edge weighting may still help, but the primary advantage diminishes.
- **Graph scale.** PPR computation scales with graph size. For KGs with >1M nodes, approximate PPR methods (e.g., push-based local PPR) are needed, and the per-query edge weight modifications must be applied lazily to the local subgraph rather than the full matrix.

## Reference

**Paper:** [Breaking the Static Graph: Context-Aware Traversal for Robust Retrieval-Augmented Generation](https://arxiv.org/abs/2602.01965v1) — Lau et al., 2026. Focus on Section 3 (method) for the three mechanisms, Section 4.3 for the Full Chain Retrieval metric that reveals partial-retrieval failures, and Table 2 for the JSR improvements that standard recall metrics hide. Code: [github.com/kwunhang/CatRAG](https://github.com/kwunhang/CatRAG).