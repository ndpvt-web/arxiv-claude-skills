---
name: "vihermes-graph-grounded-multihop-question"
description: "Build graph-grounded multihop QA systems over regulatory and hierarchically structured documents. Combines vector similarity retrieval with knowledge graph traversal to answer questions requiring reasoning across multiple interconnected legal or policy documents. Triggers: 'build a regulatory QA system', 'multihop question answering over legal documents', 'graph-based retrieval for regulations', 'answer questions across multiple policy documents', 'healthcare regulation QA pipeline', 'cross-document reasoning with graph expansion'"
---

# Graph-Grounded Multihop Question Answering for Regulatory Documents

This skill enables Claude to design and implement multihop question answering systems that combine vector-based semantic retrieval with knowledge graph traversal over hierarchically structured regulatory documents. Based on the ViHERMES framework (Nguyen et al., ACIIDS 2026), the core technique models formal legal relations as a typed graph at the granularity of legal units (documents, articles, clauses, points), then uses principled graph expansion from initial vector search hits to gather the full chain of evidence needed for multihop reasoning. This approach consistently outperforms flat retrieval-augmented generation on questions requiring amendment tracing, cross-document comparison, and procedural synthesis.

## When to Use

- When the user needs to build a QA system over regulations, statutes, policies, or compliance documents that cross-reference each other
- When questions require reasoning across 2+ documents (e.g., "What is the current penalty, considering the 2024 amendment to Article 15?")
- When building retrieval pipelines where flat chunk-based RAG fails because relevant context is spread across structurally related but distant text segments
- When the user wants to model hierarchical document structure (document > chapter > article > clause > point) as a knowledge graph for retrieval
- When constructing evaluation benchmarks for multihop regulatory QA, including generating question-answer pairs with explicit reasoning chains
- When the user asks to combine Neo4j (or any graph DB) with a vector store like Milvus/Pinecone/Chroma for hybrid legal document retrieval

## Key Technique

**Dual-store hybrid retrieval with graph-aware context expansion.** The ViHERMES approach recognizes that regulatory documents have two kinds of structure: (1) semantic similarity between provisions discussing related topics, and (2) formal legal relations — amendments that modify earlier articles, cross-references between documents, hierarchical containment of clauses within articles, and replacement chains where newer regulations supersede older ones. Flat vector retrieval captures (1) but misses (2). Pure graph traversal captures (2) but cannot handle open-ended semantic queries. ViHERMES combines both.

**The graph construction** models each legal unit (article, clause, point) as a node with typed edges: `AMENDS`, `REFERENCES`, `REPLACES`, `CONTAINS` (parent-child hierarchy), and `RELATED_TO` (semantic proximity). During ingestion, documents are parsed into their structural hierarchy, relations are extracted from explicit legal citations in the text (e.g., "as amended by Circular 30/2024"), and semantic similarity edges are added between provisions whose embeddings exceed a cosine threshold.

**Context expansion at query time** works in two phases. First, vector search over the embedded legal units returns the top-k semantically relevant nodes. Second, the graph is traversed from each hit node — following amendment chains, climbing/descending the containment hierarchy, and crossing reference edges — up to a configurable hop depth (typically 1-2 hops). The expanded node set is deduplicated, re-ranked by a combination of graph distance and semantic relevance, and passed as context to the LLM for answer generation. This principled expansion ensures the LLM sees the full regulatory chain (e.g., the original article AND its amendment AND the referenced prerequisite) without retrieving irrelevant bulk text.

## Step-by-Step Workflow

### 1. Parse documents into a legal unit hierarchy

Parse each regulatory document into a tree: Document -> Chapters -> Articles -> Clauses -> Points. Each leaf and intermediate node gets a unique ID encoding its position (e.g., `doc:42/art:15/cl:3/pt:a`). Store the raw text of each unit alongside its structural metadata.

```python
# Example legal unit schema
class LegalUnit:
    id: str              # e.g., "circ_30_2024/art_15/cl_3"
    doc_id: str          # parent document identifier
    unit_type: str       # "document" | "article" | "clause" | "point"
    text: str            # raw provision text
    parent_id: str       # structural parent
    metadata: dict       # effective date, issuing body, etc.
```

### 2. Extract formal legal relations from text

Scan each legal unit's text for citation patterns that indicate inter-unit relations. Use regex patterns for structured citations (e.g., "Article 15 of Circular 30/2024") and an LLM for implicit references. Classify each relation:

| Relation Type | Meaning | Example Pattern |
|---|---|---|
| `AMENDS` | Modifies an earlier provision | "amending Clause 3, Article 12 of..." |
| `REPLACES` | Supersedes entirely | "replacing Article 8 of..." |
| `REFERENCES` | Depends on for interpretation | "in accordance with Article 5 of..." |
| `CONTAINS` | Structural parent-child | Implicit from document hierarchy |

### 3. Build the knowledge graph

Create nodes for every legal unit and edges for every extracted relation. Use Neo4j (or networkx for lightweight setups):

```python
# Neo4j Cypher for graph construction
CREATE (u:LegalUnit {id: $id, type: $unit_type, text: $text, doc_id: $doc_id})

# Structural containment
MATCH (parent:LegalUnit {id: $parent_id}), (child:LegalUnit {id: $child_id})
CREATE (parent)-[:CONTAINS]->(child)

# Legal relations
MATCH (source:LegalUnit {id: $source_id}), (target:LegalUnit {id: $target_id})
CREATE (source)-[:AMENDS {effective_date: $date}]->(target)
```

### 4. Generate and index embeddings for each legal unit

Embed the text of every legal unit using a sentence transformer (e.g., `intfloat/multilingual-e5-large` for multilingual support, or a domain-specific model). Store embeddings in a vector database (Milvus, Chroma, Pinecone) indexed by the same `id` used in the graph. Optionally add `RELATED_TO` edges in the graph between units whose cosine similarity exceeds a threshold (e.g., 0.85).

### 5. Implement the hybrid retrieval function

At query time, run vector search first, then expand via graph traversal:

```python
def hybrid_retrieve(query: str, top_k: int = 5, max_hops: int = 2) -> list[LegalUnit]:
    # Phase 1: Vector retrieval
    query_embedding = embed(query)
    seed_nodes = vector_db.search(query_embedding, top_k=top_k)

    # Phase 2: Graph expansion
    expanded = set()
    for node in seed_nodes:
        neighbors = graph_db.traverse(
            start=node.id,
            max_depth=max_hops,
            edge_types=["AMENDS", "REPLACES", "REFERENCES", "CONTAINS"]
        )
        expanded.update(neighbors)

    # Phase 3: Re-rank by combined score
    all_candidates = list(set(seed_nodes) | expanded)
    scored = []
    for candidate in all_candidates:
        sem_score = cosine_similarity(query_embedding, candidate.embedding)
        graph_dist = shortest_path_length(seed_nodes, candidate)  # lower = closer
        combined = sem_score * 0.7 + (1 / (1 + graph_dist)) * 0.3
        scored.append((candidate, combined))

    return sorted(scored, key=lambda x: -x[1])[:top_k * 2]
```

### 6. Construct the LLM prompt with structured evidence

Arrange retrieved units in logical order — follow the amendment chain chronologically, group hierarchically related units together, and label each unit with its type and relation to the query:

```
Context:
[ORIGINAL] Article 15, Circular 20/2020: "Healthcare facilities must..."
  └─[AMENDED BY] Article 3, Circular 30/2024: "Clause 3 of Article 15 is amended as..."
[REFERENCED] Article 8, Decree 45/2019: "Licensing requirements include..."

Question: What are the current licensing requirements for healthcare facilities under the amended regulations?
```

### 7. Generate the answer with explicit reasoning chain

Instruct the LLM to produce a structured answer that traces the reasoning path through the retrieved units, citing each provision used:

```
Answer: Under the current regulations, healthcare facilities must [X] (Article 15, Circular 20/2020, as amended by Article 3, Circular 30/2024). The licensing requirements include [Y] (Article 8, Decree 45/2019, referenced in the amended Clause 3).

Reasoning chain: Circular 30/2024 Art.3 --[amends]--> Circular 20/2020 Art.15.Cl.3 --[references]--> Decree 45/2019 Art.8
```

### 8. (Optional) Generate multihop QA pairs for evaluation

To build a benchmark dataset, use semantic clustering to group related legal units, identify multi-step reasoning paths through the graph, then prompt an LLM to generate questions that require traversing those paths:

```python
def generate_multihop_qa(graph, num_pairs: int = 100):
    pairs = []
    # Find connected subgraphs with 2-4 hop paths
    paths = graph.find_paths(min_hops=2, max_hops=4, edge_types=["AMENDS", "REFERENCES"])
    for path in sample(paths, num_pairs):
        evidence = [graph.get_node(nid).text for nid in path]
        prompt = f"Generate a question that requires reading ALL of these provisions to answer:\n"
        prompt += "\n".join(evidence)
        qa = llm.generate(prompt)  # returns {question, answer, reasoning}
        pairs.append({**qa, "evidence_path": path})
    return pairs
```

## Concrete Examples

**Example 1: Building a compliance QA system for healthcare regulations**

User: "I have 50 healthcare regulation PDFs. Build me a QA system that can answer questions requiring information from multiple documents."

Approach:
1. Parse each PDF into legal units using a hierarchical parser (split on "Article", "Clause", "Point" markers)
2. Extract cross-references by scanning for citation patterns like "pursuant to Article X of Document Y"
3. Build a Neo4j graph with ~2000 legal unit nodes and ~500 relation edges
4. Embed all units with `intfloat/multilingual-e5-large` and index in Milvus
5. Implement the hybrid retrieval function with `top_k=5, max_hops=2`
6. Wire up a Chainlit or Streamlit UI that shows the reasoning chain alongside the answer

Output:
```
Q: "Can a private clinic perform cosmetic surgery without a specialized license?"
Retrieved chain:
  Decree 109/2016 Art.23 (clinic requirements) --[amended by]--> Circular 41/2023 Art.5
    --[references]--> Circular 08/2022 Art.12 (specialized license categories)

A: No. Under the amended Article 23 of Decree 109/2016 (modified by
   Circular 41/2023), private clinics performing cosmetic surgery must hold
   a Category B specialized license as defined in Article 12 of Circular
   08/2022. The 2023 amendment specifically added cosmetic procedures to the
   list requiring specialized licensing.
```

**Example 2: Generating a multihop QA evaluation benchmark**

User: "I want to test how well my RAG system handles questions that need multiple documents. Generate test questions from my regulation corpus."

Approach:
1. Build the legal unit graph from the corpus
2. Enumerate all 2-3 hop paths in the graph that cross document boundaries
3. Cluster paths by reasoning type: amendment tracing, cross-reference resolution, procedural synthesis
4. For each cluster, sample 10-20 paths and prompt an LLM to generate questions
5. Have a second LLM verify that the question genuinely requires all path nodes to answer
6. Output a JSONL dataset with fields: `question`, `answer`, `evidence_units`, `reasoning_type`, `hop_count`

Output:
```jsonl
{"question": "What training requirements apply to facility managers after the 2024 amendment?",
 "answer": "Facility managers must complete 120 hours of certified training...",
 "evidence_units": ["decree_109_art23", "circular_41_2023_art5", "circular_08_2022_art12"],
 "reasoning_type": "amendment_tracing",
 "hop_count": 2}
```

**Example 3: Adding graph-aware retrieval to an existing RAG pipeline**

User: "My current vector-only RAG misses context when regulations reference each other. How do I add graph expansion?"

Approach:
1. Keep the existing vector store and embedding pipeline unchanged
2. Add a lightweight graph layer — even `networkx` in-memory works for <10k units
3. During ingestion, extract citation edges with regex: `r"(?:Article|Clause)\s+\d+.*?(?:of|in)\s+(Circular|Decree|Law)\s+[\d/]+"`
4. Wrap the existing retrieval call to add graph expansion as a post-processing step
5. Re-rank the expanded set using the weighted scoring formula (0.7 semantic + 0.3 graph proximity)

```python
# Minimal integration with existing RAG
import networkx as nx

# Build graph once at startup
G = nx.DiGraph()
for unit in all_units:
    G.add_node(unit.id, text=unit.text)
for rel in extracted_relations:
    G.add_edge(rel.source, rel.target, type=rel.relation_type)

# Wrap existing retriever
def enhanced_retrieve(query, existing_retriever, top_k=5):
    base_results = existing_retriever.search(query, top_k=top_k)
    expanded_ids = set()
    for result in base_results:
        for neighbor in nx.ego_graph(G, result.id, radius=2):
            expanded_ids.add(neighbor)
    expanded_units = [get_unit(uid) for uid in expanded_ids]
    return rerank(query, base_results + expanded_units)
```

## Best Practices

- **Do:** Model legal units at the finest useful granularity (clause or point level, not whole documents). Coarse nodes defeat the purpose of precise graph traversal.
- **Do:** Type your graph edges explicitly (`AMENDS`, `REPLACES`, `REFERENCES`). During expansion, you can filter by edge type — e.g., always follow `AMENDS` chains but limit `REFERENCES` to 1 hop to avoid topic drift.
- **Do:** Include temporal metadata (effective dates) on amendment edges. When answering "current law" questions, traverse only to the latest amendment in the chain.
- **Do:** Re-rank after graph expansion. Expanding the candidate set introduces noise; a combined semantic + graph-distance score keeps results focused.
- **Avoid:** Setting `max_hops` above 2 for general queries. At 3+ hops the expanded set grows combinatorially and dilutes relevance. Reserve deeper traversal for explicit multi-amendment chain queries.
- **Avoid:** Treating the graph as the sole retrieval mechanism. Pure graph traversal cannot handle novel phrasings or semantic queries. Always start with vector retrieval and use the graph to expand, not replace.

## Error Handling

| Problem | Symptom | Solution |
|---|---|---|
| Citation extraction misses references | Graph has few edges, expansion adds nothing | Add a fallback: use an LLM to identify implicit references in each unit's text. Also check for citation format variations. |
| Graph expansion returns too many nodes | LLM context window exceeded or answer quality drops | Reduce `max_hops`, filter edge types, or impose a hard cap on expanded nodes (e.g., 20) with re-ranking. |
| Amendment chain is circular | Traversal loops indefinitely | Use visited-node tracking in traversal. Real amendment chains are acyclic; cycles indicate extraction errors to fix. |
| Vector search returns wrong document type | E.g., retrieving expired provisions | Add metadata filters to vector search (effective date ranges, document status). |
| Multilingual embedding quality is poor | Retrieved units are semantically irrelevant | Use domain-adapted embeddings or fine-tune on a small set of legal unit pairs with known similarity. |

## Limitations

- **Domain-specific citation patterns:** The relation extraction step relies on citation formats specific to the regulatory domain. Applying this to academic papers, patents, or contracts requires adapting the citation parser for each domain's conventions.
- **Graph construction overhead:** Building and maintaining the knowledge graph adds engineering complexity. For corpora under ~50 documents with few cross-references, flat vector RAG may be sufficient and simpler.
- **Language and structure dependency:** The ViHERMES pipeline was designed for Vietnamese healthcare regulations with a specific hierarchical structure (Document > Article > Clause > Point). Other jurisdictions or domains may use different structural conventions that require parser adaptation.
- **Evaluation requires domain expertise:** Verifying that multihop QA pairs genuinely require multi-step reasoning (and that answers are legally correct) demands subject matter experts, which limits scalable benchmarking.
- **Static graph:** The graph must be rebuilt or incrementally updated when new regulations are published. There is no built-in mechanism for real-time regulatory change detection.

## Reference

**Paper:** Nguyen et al., "ViHERMES: A Graph-Grounded Multihop Question Answering Benchmark and System for Vietnamese Healthcare Regulations" (ACIIDS 2026). [arXiv:2602.07361](https://arxiv.org/abs/2602.07361v1)

Look for: Section 3 (dataset construction pipeline with semantic clustering and graph-inspired data mining), Section 4 (the graph-aware retrieval framework with legal unit modeling and context expansion algorithm), and Section 5 (experimental comparison against flat retrieval baselines).

**Code:** [github.com/ura-hcmut/ViHERMES](https://github.com/ura-hcmut/ViHERMES) — Reference implementation using Neo4j + Milvus + pydantic-ai.