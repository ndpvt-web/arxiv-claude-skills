---
name: "graphagents-knowledge-graph-guided-agentic"
description: "Build multi-agent pipelines that use knowledge graphs to guide LLM reasoning across domains. Agents specialize in problem decomposition, evidence retrieval, parameter extraction, graph traversal, and hypothesis synthesis. Use when: 'build a knowledge graph agent pipeline', 'find cross-domain material substitutes', 'multi-agent graph traversal system', 'knowledge graph guided search', 'graph-based RAG with specialized agents', 'cross-domain design exploration with KG'."
---

This skill enables Claude to design and implement multi-agent systems where a knowledge graph (KG) serves as the shared reasoning backbone. Drawing from the GraphAgents framework, the pipeline decomposes a complex design or research question into sub-problems, retrieves evidence from both vector stores and graph substructures, extracts structured parameters, traverses the KG with configurable strategies (exploitative BFS vs. exploratory DFS), and synthesizes grounded hypotheses. The approach consistently outperforms single-shot LLM prompting by distributing specialized reasoning across agents while anchoring every claim to traceable graph paths.

## When to Use

- When the user needs to find substitutes, alternatives, or replacements for materials, chemicals, or components across domains (e.g., "find a PFAS-free polymer for biomedical tubing")
- When building a RAG pipeline that must combine vector-similarity retrieval with structured graph traversal for richer context
- When a task requires connecting knowledge across distinct domains (e.g., linking molecular chemistry to mechanical performance to biocompatibility)
- When the user asks to build a multi-agent system where each agent has a clearly bounded role in a sequential pipeline
- When designing a knowledge graph from scientific literature and using it to guide LLM reasoning
- When the user wants configurable search strategies — switching between focused/exploitative and broad/exploratory graph walks

## Key Technique

The core insight of GraphAgents is **dual-source retrieval with agent specialization**. Instead of one monolithic LLM call, five agents form a sequential pipeline: Planner, Hybrid GraphWeave, Evaluator, Creative GraphWeave, and Engineer. Each agent consumes the structured output of its predecessor. Two complementary knowledge graphs — one domain-specific (depth) and one cross-domain (breadth) — are constructed from literature by extracting semantic triplets (entity-relation-entity) via an LLM, then consolidating duplicate nodes using embedding cosine similarity. Every edge retains a pointer to its source text chunk for traceability.

The framework alternates between **exploitative** and **exploratory** graph traversal. BFS shortest-path search gives direct, high-confidence connections (exploitative). DFS with a depth limit of ~5 hops uncovers multi-hop chains that surface unexpected cross-domain links (exploratory). A third mode — BFS with a semantic stop — forces paths through a user-specified waypoint node, steering results toward a particular material family or property. This lets the user dial between "give me the safest known substitute" and "surprise me with something novel."

Ablation studies show the full pipeline scores ~7.0/10 across six criteria (task decomposition, context enrichment, cross-subtask integration, deep reasoning, novelty, source attribution) vs. 3.2/10 for a single LLM call. Removing the Planner agent alone drops cross-subtask integration from 5 to 3-4, confirming that problem decomposition is a critical bottleneck that dedicated specialization addresses.

## Step-by-Step Workflow

1. **Define the design question and constraints.** Capture the user's goal as a structured query: what must be replaced or designed, what performance criteria matter (with quantitative ranges if possible), and what domains are involved.

2. **Build or load the knowledge graph(s).** For each relevant corpus, chunk documents into LLM-context-sized segments. For each chunk, prompt an LLM to extract semantic triplets `(entity, relation, entity)`. Aggregate triplets into a graph. Consolidate duplicate nodes by computing pairwise cosine similarity of node-name embeddings and merging above a threshold (~0.9). Tag every edge with its source chunk ID.

3. **Index chunks in a vector store.** Store each text chunk with its embedding (Sentence-BERT or equivalent) in a vector database (ChromaDB, Pinecone, pgvector). This enables the retrieval agent to do similarity search independent of graph structure.

4. **Implement the Planner agent.** Given the user's query, generate 3-5 sub-questions that decompose the problem along distinct axes (e.g., mechanical properties, thermal behavior, biological safety). Output a keyword set with synonyms for each sub-question.

5. **Implement the Hybrid GraphWeave agent (evidence retrieval).** For each sub-question, retrieve top-k (k=5) text chunks from the vector store via cosine similarity. Map each retrieved chunk to its corresponding subgraph in the domain-specific KG using the source-chunk edge tags. Return both the text evidence and the relational subgraph.

6. **Implement the Evaluator agent (parameter extraction).** Parse Hybrid GraphWeave outputs to extract structured design parameters: property names, numerical ranges, qualitative levels. Output a clean property descriptor list (e.g., `{"tensile_strength": "20-30 MPa", "friction_coefficient": "0.1-0.3"}`).

7. **Implement the Creative GraphWeave agent (cross-domain traversal).** Embed each extracted keyword into the cross-domain KG's vector space. Find the closest matching graph node for each keyword. Apply the chosen traversal strategy:
   - **BFS shortest path** for direct, high-confidence connections (return top-5 paths).
   - **DFS with depth limit 5** for exploratory multi-hop discovery.
   - **BFS with semantic stop** to force paths through a user-specified waypoint node.
   Assemble the union of all traversed paths into an expanded subgraph.

8. **Implement the Engineer agent (hypothesis synthesis).** Consume original requirements, extracted properties, and the expanded subgraph. Generate a structured hypothesis: proposed material/composite, justification mapped to each requirement, expected property values, implementation challenges, and explicit KG reasoning paths showing which graph edges support each claim.

9. **Wire agents sequentially.** Each agent receives the full output of its predecessor as context. Use structured output formats (JSON or markdown with clear sections) to minimize information loss between agents. No shared mutable state — coordination is purely through sequential context passing.

10. **Evaluate and iterate.** Score outputs against the six criteria (task decomposition, context enrichment, cross-subtask integration, deep reasoning, novelty, source attribution). Run ablation by disabling individual agents to identify which components contribute most to your specific domain.

## Concrete Examples

**Example 1: Finding PFAS-free biomedical tubing alternatives**

User: "I need to find sustainable replacements for PFAS in biomedical tubing. The material must have tensile strength 20-30 MPa, friction coefficient under 0.3, thermal stability to 250C, and be biocompatible."

Approach:
1. Planner decomposes into sub-questions: mechanical integrity under pressure, low-friction flow properties, sterilization thermal resistance, biological safety.
2. Hybrid GraphWeave retrieves literature on each sub-question from a polymer science corpus + PFAS KG, returning text chunks and subgraphs linking fluoropolymer properties to performance.
3. Evaluator extracts: `tensile_strength: 20-30 MPa`, `friction_coefficient: 0.1-0.3`, `thermal_stability: 250-400C`, `biocompatibility: required`.
4. Creative GraphWeave embeds these into a materials KG and runs BFS shortest paths, finding connections through PLA, cellulose nanofibers, and polydopamine coatings.
5. Engineer synthesizes: "PLA matrix reinforced with cellulose nanofibers and polydopamine surface layer. Tensile strength >50 MPa via nanofiber reinforcement (KG path: mechanical_properties → tensile_strength → cellulose_nanofiber). Biocompatibility from polydopamine's protein-resistant surface (KG path: surface_modification → polydopamine → biocompatibility)."

Output:
```json
{
  "proposed_material": "PLA + cellulose nanofibers + polydopamine coating",
  "properties": {
    "tensile_strength": ">50 MPa",
    "glass_transition": ">60C",
    "biocompatibility": "polydopamine protein resistance",
    "friction": "reduced via surface coating"
  },
  "kg_reasoning_paths": [
    "mechanical_properties → tensile_strength → cellulose_nanofiber → PLA_composite",
    "surface_modification → polydopamine → biocompatibility → biomedical_tubing",
    "heat_treatment → biological_durability → sterilization_resistance"
  ],
  "challenges": ["PLA hydrolytic degradation in long-term implant use", "coating durability under repeated sterilization"]
}
```

**Example 2: Building the knowledge graph from a paper corpus**

User: "I have 200 papers on polymer composites. Help me build a knowledge graph and set up the triplet extraction pipeline."

Approach:
1. Chunk each paper into segments of ~2000 tokens (respecting section boundaries).
2. For each chunk, prompt the LLM with: "Extract all semantic triplets from this text. Return as JSON array of {entity1, relation, entity2} objects. Entities should be specific nouns (materials, properties, methods). Relations should be verbs or prepositional phrases."
3. Aggregate all triplets into a NetworkX graph. For each edge, store `source_chunk_id` as metadata.
4. Compute Sentence-BERT embeddings for all node labels. For each pair with cosine similarity > 0.9, merge into a single canonical node, preserving all edges.
5. Store chunks + embeddings in ChromaDB. Export graph to Neo4j or a JSON adjacency list.

Output:
```python
import networkx as nx
from sentence_transformers import SentenceTransformer
from chromadb import Client

# Step 1: Extract triplets per chunk
def extract_triplets(chunk_text, llm_client):
    prompt = (
        "Extract semantic triplets from this scientific text. "
        "Return JSON: [{\"entity1\": ..., \"relation\": ..., \"entity2\": ...}]. "
        "Entities: specific materials, properties, methods, phenomena. "
        "Relations: causal verbs, 'has_property', 'composed_of', 'measured_by'."
    )
    return llm_client.generate(prompt + "\n\n" + chunk_text, format="json")

# Step 2: Build graph with provenance
G = nx.DiGraph()
for chunk in chunks:
    triplets = extract_triplets(chunk.text, llm)
    for t in triplets:
        G.add_edge(t["entity1"], t["entity2"],
                   relation=t["relation"],
                   source_chunk=chunk.id)

# Step 3: Consolidate duplicate nodes
model = SentenceTransformer("all-MiniLM-L6-v2")
node_names = list(G.nodes())
embeddings = model.encode(node_names)
# Merge nodes with cosine similarity > 0.9
# (implementation: union-find over similar pairs)

# Step 4: Index chunks in vector store
chroma = Client()
collection = chroma.create_collection("polymer_composites")
for chunk in chunks:
    collection.add(ids=[chunk.id], documents=[chunk.text],
                   embeddings=[model.encode(chunk.text)])
```

**Example 3: Switching traversal strategies for different goals**

User: "I want to explore what silk-based materials could replace in my current design. Use exploratory search."

Approach:
1. Use BFS with semantic stop, setting "silk" as the mandatory waypoint node.
2. For each design keyword (tensile strength, thermal stability, etc.), compute shortest paths that pass through the silk node.
3. This forces all results to converge on silk-family materials while still connecting to the required properties.
4. Compare with a DFS run (depth limit 5) from the same keywords to surface unexpected multi-hop chains through silk → hydrogen bonding → pH-dependent stabilization → novel applications.

Output:
```
Traversal: BFS with semantic stop (waypoint="silk_fibroin")

Path 1: tensile_strength → mechanical_properties → silk_fibroin → biocompatibility → biomedical_tubing
Path 2: thermal_stability → temperature_resistance → silk_fibroin → hydrogen_bonding → structural_integrity
Path 3: friction_coefficient → surface_properties → TiO2_nanoparticles → silk_fibroin → eutectogel_system

Proposed: Silk fibroin matrix + TiO2 nanoparticles + eutectogel system
  - Tensile strength: >= 50 MPa
  - Glass transition: ~200C
  - Friction coefficient: < 0.1
  - Key mechanism: silk hydrogen bonding + pH-dependent stabilization
```

## Best Practices

- **Do:** Tag every KG edge with its source chunk ID. Traceability is what separates this approach from hallucination-prone single-shot prompting. The Engineer agent must cite specific graph paths in its output.
- **Do:** Use two complementary graphs — one narrow/deep (domain-specific) for grounded retrieval, one broad (cross-domain) for creative exploration. A single graph forces a tradeoff between precision and discovery.
- **Do:** Keep agent outputs structured (JSON or well-sectioned markdown). Each agent consumes its predecessor's output as context; ambiguous free-text degrades downstream agents.
- **Do:** Run BFS first for baseline results, then DFS or semantic-stop BFS for creative alternatives. Present both to the user so they can assess the confidence-novelty tradeoff.
- **Avoid:** Skipping the Evaluator agent. Without structured parameter extraction, the Creative GraphWeave agent receives noisy keyword matches, and the quality of graph traversal degrades substantially.
- **Avoid:** Setting DFS depth limits above 5-6 hops. Deeper chains produce increasingly tenuous connections and exponential path counts without proportional insight gains.

## Error Handling

- **Embedding mismatch during node lookup:** When a keyword from the Evaluator doesn't closely match any KG node (cosine similarity < 0.7), log the gap and fall back to the closest match while flagging it in the output. The paper notes that "thermal stability in range 250-400C" mapped to the generic "temperature stability" node, losing numerical specificity — surface these losses explicitly.
- **Graph disconnection:** If BFS finds no path between two nodes, the graph may lack coverage in that region. Fall back to DFS from each node independently and report the separate subgraphs rather than forcing a spurious connection.
- **Triplet extraction noise:** LLMs produce inconsistent entity naming across chunks. The node consolidation step (embedding similarity merge) mitigates this, but set the merge threshold conservatively (0.85-0.92) and review the merge log for false positives.
- **Context window overflow:** The sequential pipeline accumulates context. If the Engineer agent's input exceeds the context window, summarize earlier agent outputs while preserving the structured property list and graph paths verbatim — those are the highest-value signals.

## Limitations

- The pipeline is strictly sequential — each agent blocks on its predecessor. For large corpora or many sub-questions, latency accumulates linearly across the five stages.
- Knowledge graph quality is bottlenecked by LLM triplet extraction. Domain-specific jargon, abbreviations, and implicit relationships are frequently missed or misrepresented.
- The framework was validated on materials science (PFAS substitution). Transferring to domains with fundamentally different knowledge structures (e.g., legal reasoning, financial analysis) requires rebuilding the KG ontology and re-tuning traversal parameters.
- Evaluation in the paper used a single GPT-5 judge pass with no inter-rater reliability or confidence intervals. Treat the ablation scores as directional, not absolute.
- No explicit edge weighting or path scoring function exists — paths are ranked only by hop count (BFS) or traversal order (DFS). Adding edge weights (e.g., citation count, recency) would improve result ranking but is not part of the current framework.

## Reference

[GraphAgents: Knowledge Graph-Guided Agentic AI for Cross-Domain Materials Design](https://arxiv.org/abs/2602.07491v1) — Stewart et al., 2026. Focus on Section 2 (Materials and Methods) for the five-agent pipeline architecture, Section 3 for ablation results showing each agent's contribution, and Figures 9-14 for concrete graph traversal outputs across BFS/DFS/semantic-stop strategies.