---
name: "use-graph-it-needs"
description: "Implement adaptive RAG pipelines that route queries to dense retrieval, graph-based retrieval, or a weighted fusion based on query complexity scoring. Use when: 'build a RAG pipeline that uses knowledge graphs only when needed', 'add adaptive graph retrieval to my search system', 'route simple vs complex queries differently in RAG', 'implement EA-GraphRAG routing', 'optimize RAG by skipping graph lookup for simple questions', 'build a hybrid retrieval system with complexity-aware fusion'."
---

# Adaptive Graph-Augmented RAG (EA-GraphRAG)

This skill teaches Claude to build retrieval-augmented generation systems that **selectively** use knowledge graphs based on query complexity rather than applying graph retrieval uniformly. The core insight from the EA-GraphRAG paper is that GraphRAG underperforms vanilla RAG on simple single-hop queries due to noise from unnecessary graph traversal, while outperforming it on complex multi-hop questions. By scoring query complexity with lightweight syntactic features and routing to the appropriate retrieval strategy, you get the best of both worlds: fast dense retrieval for simple queries, structured graph reasoning for complex ones, and a weighted fusion for borderline cases.

## When to Use

- When the user wants to build a RAG pipeline that handles both simple factoid and complex multi-hop questions efficiently
- When the user asks to add knowledge graph retrieval to an existing RAG system without degrading performance on simple queries
- When the user needs to reduce latency in a GraphRAG system by skipping graph traversal for straightforward lookups
- When the user wants to implement query routing or classification to select between retrieval strategies
- When the user is building a QA system over a domain corpus and wants adaptive retrieval that scales with query complexity
- When the user asks about combining dense retrieval with graph-based retrieval using reciprocal rank fusion

## Key Technique

**The Problem**: GraphRAG applies structured knowledge graph traversal to every query. For simple questions like "What year was Python released?", this adds latency and introduces noise from irrelevant graph edges. For complex questions like "Which programming languages influenced both Rust and Swift?", graph traversal is essential for connecting entities across documents. Applying one strategy uniformly wastes resources or sacrifices accuracy.

**The Solution**: EA-GraphRAG introduces a three-lane routing architecture. First, a **Syntactic Feature Constructor** extracts over 80 linguistic features from each query: dependency tree depth, clause counts, named entity density, subordination ratios, and coordination markers. Second, a **Lightweight Complexity Scorer** maps these features through a learned linear layer + sigmoid to produce a continuous score `s(q) in (0, 1)`. Third, a **Score-Driven Router** compares `s(q)` against two thresholds (`tau_L`, `tau_H`) to select: dense-only retrieval (`s <= tau_L`), graph-only retrieval (`s >= tau_H`), or complexity-aware weighted reciprocal rank fusion (`tau_L < s < tau_H`).

**The Fusion Formula**: For borderline queries, both retrieval lists are merged using weighted RRF: `score(c) = (1 - s(q)) * 1/(k + rank_rag(c)) + s(q) * 1/(k + rank_graph(c))`. This smoothly interpolates between dense and graph results based on how complex the query is, rather than a hard cutoff.

## Step-by-Step Workflow

1. **Build the document graph offline.** Extract entities and noun phrases from your corpus using NER + noun-phrase chunking. Create a heterogeneous graph with two node types (entities `N` and passages `C`). Add three edge types: relation edges from OpenIE triples `(head, relation, tail)`, occurrence edges linking entities to their source passages, and synonymy edges where `cosine(embed(n1), embed(n2)) >= threshold`.

2. **Index passages for dense retrieval.** Encode all passages with a bi-encoder (e.g., ColBERT, Contriever, or sentence-transformers). Store embeddings in a vector index (FAISS, Qdrant, or Pinecone) for fast top-K similarity search.

3. **Extract syntactic features from incoming queries.** For each query, compute: dependency tree max depth, average dependency distance, clause count, named entity count and type distribution, subordination ratio (dependent clauses / total clauses), coordination markers, T-unit complexity, and content-to-function word ratio. Use spaCy or Stanza for parsing.

4. **Score query complexity.** Feed the feature vector through a trained linear classifier with sigmoid activation: `s(q) = sigmoid(W * features(q) + b)`. Train this scorer on a labeled dataset of simple (single-hop) and complex (multi-hop) questions, or use a heuristic proxy (see examples below).

5. **Route the query based on thresholds.** Compare `s(q)` against two thresholds:
   - If `s(q) <= tau_L` (e.g., 0.3): use **dense retrieval only** -- encode query, find top-K passages by embedding similarity.
   - If `s(q) >= tau_H` (e.g., 0.7): use **graph retrieval only** -- match query to graph facts, seed entities, run Personalized PageRank, extract top-K passages from high-scoring passage nodes.
   - If `tau_L < s(q) < tau_H`: use **weighted RRF fusion** -- run both retrievals and merge results.

6. **Execute graph retrieval when selected.** Compute fact similarity between the query embedding and stored relation triples. Weight seed entities by their fact-match scores. Initialize a Personalized PageRank vector from seed entities and diffuse over the graph with damping factor alpha. Rank passage nodes by their final PPR score.

7. **Execute weighted RRF fusion when selected.** For each candidate passage appearing in either retrieval list, compute: `RRF(c) = (1 - s(q)) / (k + rank_dense(c)) + s(q) / (k + rank_graph(c))`. Use `k = 60` as the standard RRF smoothing constant. Rank by fused score and take top-K.

8. **Generate the answer.** Feed the retrieved passages (from whichever strategy was selected) into the LLM as context, along with the original query. Use a standard RAG prompt template.

9. **Tune thresholds on a validation set.** Optimize `tau_L` and `tau_H` on a held-out mix of simple and complex queries to maximize a composite objective of accuracy and expected latency. Grid search over `[0.2, 0.4]` for `tau_L` and `[0.6, 0.8]` for `tau_H`.

## Concrete Examples

**Example 1: Building an adaptive RAG pipeline in Python**

User: "I have a corpus of 10K medical documents. Build me a RAG system that uses a knowledge graph for complex questions but skips it for simple ones."

Approach:
1. Parse documents with spaCy's `en_core_sci_lg` model to extract biomedical entities.
2. Build a knowledge graph using extracted entities and relations.
3. Index passages with sentence-transformers for dense retrieval.
4. Implement query routing with complexity scoring.

```python
import spacy
import numpy as np
from dataclasses import dataclass

nlp = spacy.load("en_core_web_lg")

@dataclass
class QueryFeatures:
    dep_tree_depth: int
    avg_dep_distance: float
    clause_count: int
    entity_count: int
    entity_type_count: int
    has_subordination: bool
    has_coordination: bool
    word_count: int

def extract_features(query: str) -> QueryFeatures:
    doc = nlp(query)
    # Dependency tree depth
    def tree_depth(token):
        return 1 + max((tree_depth(c) for c in token.children), default=0)
    roots = [t for t in doc if t.dep_ == "ROOT"]
    max_depth = max((tree_depth(r) for r in roots), default=1)

    # Average dependency distance
    dep_distances = [abs(t.i - t.head.i) for t in doc if t.dep_ != "ROOT"]
    avg_dep_dist = np.mean(dep_distances) if dep_distances else 0.0

    # Clause indicators (advcl, ccomp, xcomp, relcl, acl)
    clause_deps = {"advcl", "ccomp", "xcomp", "relcl", "acl"}
    clause_count = sum(1 for t in doc if t.dep_ in clause_deps)

    # Named entities
    ents = list(doc.ents)
    entity_types = set(e.label_ for e in ents)

    return QueryFeatures(
        dep_tree_depth=max_depth,
        avg_dep_distance=avg_dep_dist,
        clause_count=clause_count,
        entity_count=len(ents),
        entity_type_count=len(entity_types),
        has_subordination=clause_count > 0,
        has_coordination=any(t.dep_ == "conj" for t in doc),
        word_count=len(doc),
    )

def complexity_score(features: QueryFeatures) -> float:
    """Heuristic complexity scorer when no trained model is available."""
    score = 0.0
    score += min(features.dep_tree_depth / 8.0, 0.3)
    score += min(features.clause_count / 3.0, 0.25)
    score += min(features.entity_count / 5.0, 0.2)
    score += 0.15 if features.has_subordination else 0.0
    score += 0.1 if features.has_coordination else 0.0
    return min(score, 1.0)

def route_query(query: str, tau_l=0.3, tau_h=0.7):
    features = extract_features(query)
    score = complexity_score(features)
    if score <= tau_l:
        return "dense_only", score
    elif score >= tau_h:
        return "graph_only", score
    else:
        return "fusion", score

# Example routing
print(route_query("What is aspirin?"))           # ('dense_only', 0.1)
print(route_query(                                # ('graph_only', 0.8)
    "Which drugs that treat both migraines and "
    "arthritis have interactions with ACE inhibitors?"
))
```

**Example 2: Implementing weighted RRF fusion**

User: "I already have dense retrieval and graph retrieval. How do I merge their results adaptively?"

```python
def weighted_rrf(
    dense_results: list[str],
    graph_results: list[str],
    complexity_score: float,
    k: int = 60,
    top_k: int = 10,
) -> list[str]:
    """Merge dense and graph retrieval using complexity-aware RRF."""
    scores = {}

    # Score from dense retrieval (weight decreases with complexity)
    for rank, doc_id in enumerate(dense_results, start=1):
        scores[doc_id] = scores.get(doc_id, 0.0)
        scores[doc_id] += (1 - complexity_score) / (k + rank)

    # Score from graph retrieval (weight increases with complexity)
    for rank, doc_id in enumerate(graph_results, start=1):
        scores[doc_id] = scores.get(doc_id, 0.0)
        scores[doc_id] += complexity_score / (k + rank)

    ranked = sorted(scores.items(), key=lambda x: x[1], reverse=True)
    return [doc_id for doc_id, _ in ranked[:top_k]]
```

**Example 3: Building the heterogeneous document graph**

User: "How do I construct the knowledge graph from my documents for the graph retrieval path?"

```python
from collections import defaultdict
from sentence_transformers import SentenceTransformer
import numpy as np

encoder = SentenceTransformer("all-MiniLM-L6-v2")

def build_graph(passages: list[dict], synonymy_threshold=0.85):
    """
    Build heterogeneous graph from passages.
    Each passage dict: {"id": str, "text": str, "entities": list[str],
                        "triples": list[tuple[str,str,str]]}
    """
    entity_nodes = {}   # entity_text -> node_id
    passage_nodes = {}  # passage_id -> node_id
    edges = []          # (src, rel_type, dst)

    # Create passage nodes
    for p in passages:
        passage_nodes[p["id"]] = f"P_{p['id']}"

    # Create entity nodes + occurrence edges
    for p in passages:
        for ent in p["entities"]:
            if ent not in entity_nodes:
                entity_nodes[ent] = f"E_{ent}"
            edges.append((entity_nodes[ent], "occurs_in", passage_nodes[p["id"]]))

    # Relation edges from OpenIE triples
    for p in passages:
        for head, rel, tail in p["triples"]:
            if head in entity_nodes and tail in entity_nodes:
                edges.append((entity_nodes[head], rel, entity_nodes[tail]))

    # Synonymy edges between entity nodes
    ent_texts = list(entity_nodes.keys())
    if ent_texts:
        embeddings = encoder.encode(ent_texts)
        sims = np.dot(embeddings, embeddings.T)
        for i in range(len(ent_texts)):
            for j in range(i + 1, len(ent_texts)):
                if sims[i][j] >= synonymy_threshold:
                    edges.append((
                        entity_nodes[ent_texts[i]],
                        "synonym",
                        entity_nodes[ent_texts[j]]
                    ))

    return {
        "entity_nodes": entity_nodes,
        "passage_nodes": passage_nodes,
        "edges": edges,
    }
```

## Best Practices

- **Do**: Start with a heuristic complexity scorer (dependency depth + entity count + clause count) and replace it with a trained model only after collecting labeled routing data from production queries.
- **Do**: Cache syntactic feature extraction results -- spaCy parsing is the most expensive part of routing and features are deterministic per query string.
- **Do**: Set `tau_L` and `tau_H` conservatively at first (`0.25` and `0.75`), then narrow the gap as you gain confidence in the scorer's calibration.
- **Do**: Log which route each query takes and the resulting answer quality, so you can iteratively improve the thresholds and scorer.
- **Avoid**: Building the knowledge graph with a single LLM pass over all documents. Use a two-stage approach: NER + noun-phrase extraction first, then OpenIE for relation triples. Single-pass extraction misses many entities.
- **Avoid**: Setting `tau_L = tau_H` (no fusion zone). The fusion path handles ambiguous queries that neither pure strategy serves well. A fusion band of at least 0.2 width is recommended.

## Error Handling

- **Empty graph retrieval results**: If Personalized PageRank returns no passage nodes above the score threshold, fall back to dense retrieval rather than returning nothing. Log the event to flag potential graph coverage gaps.
- **Feature extraction failures**: If spaCy fails to parse a query (malformed input, unsupported language), default to dense-only retrieval. Never block the pipeline on routing failure.
- **Graph construction errors**: If OpenIE produces contradictory triples or NER extracts garbage entities, filter edges by a minimum confidence score. Noisy graphs degrade PPR quality more than sparse graphs do.
- **Threshold miscalibration**: If monitoring shows graph retrieval is never or always selected, the thresholds are miscalibrated. Re-tune on a fresh sample of 200+ queries with known complexity labels.
- **Synonymy edge explosion**: If entity embeddings produce too many synonymy edges (>10x entity count), raise the cosine threshold or limit synonymy edges to k-nearest neighbors per entity.

## Limitations

- **Requires labeled complexity data for training the scorer**: The heuristic proxy works reasonably but a trained classifier needs examples of simple vs complex queries for your domain. Budget 500-1000 labeled examples.
- **Graph construction overhead**: Building the entity-passage graph is an offline batch process. For rapidly changing corpora (news feeds, live data), the graph becomes stale and requires incremental updates.
- **English-centric syntactic features**: The feature set (T-units, clause ratios, dependency distances) is designed for English. Other languages need adapted feature extractors.
- **Not a replacement for query decomposition**: For extremely complex queries requiring 4+ hops, even graph retrieval may not suffice. Consider combining with query decomposition strategies for such cases.
- **Single-domain assumption**: The complexity scorer and thresholds are tuned per domain. A model trained on medical QA may misroute legal or financial queries.

## Reference

- **Paper**: [Use Graph When It Needs: Efficiently and Adaptively Integrating RAG with Graphs](https://arxiv.org/abs/2602.03578v1) (Dong et al., 2026). Focus on Section 3 (method) for the routing architecture, Section 3.2 for the syntactic feature set, and Section 4 for benchmark results showing +3.1% accuracy over best GraphRAG baseline with lower latency.