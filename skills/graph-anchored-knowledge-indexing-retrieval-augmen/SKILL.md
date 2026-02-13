---
name: "graph-anchored-knowledge-indexing-retrieval-augmen"
description: "Build iterative RAG pipelines that construct evolving knowledge graphs to anchor retrieval across multiple hops. Use when user says 'multi-hop QA', 'graph-guided retrieval', 'iterative RAG', 'knowledge graph indexing', 'connect evidence across documents', or 'structured retrieval pipeline'."
---

# Graph-Anchored Knowledge Indexing for Retrieval-Augmented Generation

This skill enables Claude to implement **GraphAnchor**-style iterative retrieval pipelines where a knowledge graph is incrementally built during retrieval to serve as a structured index. Instead of treating retrieved documents as flat text, the system extracts entities and relations into an evolving graph that guides the LLM in judging whether enough evidence has been gathered, formulating targeted follow-up queries, and focusing attention on key information scattered across noisy documents when generating the final answer.

## When to Use

- When the user needs to answer complex multi-hop questions that require connecting facts from multiple documents (e.g., "Who directed the film starring the actor born in the same city as the inventor of X?")
- When building a RAG pipeline that must handle questions requiring 2-4 retrieval steps to gather sufficient evidence
- When the user wants to reduce hallucination in RAG by structuring retrieved knowledge before answer generation
- When the user asks to implement iterative or adaptive retrieval (where subsequent queries depend on what was already found)
- When building a document analysis system that must trace reasoning chains across a corpus
- When the user wants to add knowledge sufficiency detection to a retrieval pipeline (knowing when to stop retrieving)

## Key Technique

**Core Insight: Graphs as Active Indices, Not Static Stores.** Traditional knowledge graph approaches build a graph once and query it. GraphAnchor flips this: the graph is constructed *during* retrieval as a byproduct of reading documents. At each retrieval step, the system extracts salient entities and their relations from newly retrieved documents and merges them into a running graph. This graph then serves two purposes: (1) it tells the LLM what is known and what is missing, enabling it to judge knowledge sufficiency and generate precise follow-up queries; (2) it acts as a structured attention guide during final answer generation, helping the LLM connect evidence distributed across documents.

**Iterative Retrieval Loop.** The system starts with the original question as the first query. At each step *t*, it retrieves the top-k documents, extracts entities and relations to update the graph G_t, then generates a reasoning trace that includes a sufficiency judgment (`sufficient` or `insufficient`). If insufficient, the system produces a targeted subquery to fill the identified gap. The loop runs until sufficiency is reached or a maximum step count (typically 4) is hit.

**Linearized Graph Representation.** The graph is serialized as natural language so any LLM can process it without specialized graph APIs. Entities are listed with their attributes, and relations are expressed as RDF-style triples verbalized into text: `Entities: [entity1 (attribute)], [entity2 (attribute)]; Relations: [entity1 -- relation -- entity2], ...`. This linearized form is inserted into the prompt alongside retrieved documents, allowing the LLM to jointly attend to structured and unstructured evidence.

## Step-by-Step Workflow

1. **Parse the input question and initialize state.** Accept the user's question `q_0`. Initialize an empty graph `G_0 = (V={}, E={})`, an empty document set `D = {}`, step counter `t = 1`, and set the current query `q_current = q_0`.

2. **Retrieve documents for the current query.** Use an embedding-based retriever (e.g., BGE, Sentence-Transformers, or OpenAI embeddings) to fetch top-k documents (k=5 recommended) for `q_current` from the corpus. Append results to `D`.

3. **Extract entities and relations from new documents.** Prompt the LLM to read the newly retrieved documents and extract named entities (people, places, organizations, dates, quantities) and their pairwise relations as subject-predicate-object triples. On step 1, this initializes the graph. On subsequent steps, merge new extractions into the existing graph, deduplicating entities by normalized name.

4. **Linearize the current graph into text.** Serialize the graph as: `Entities: [name1 (type/attribute)], [name2 (type/attribute)]; Relations: [name1 -- predicate -- name2], [name3 -- predicate -- name4]`. Wrap in `<graph>...</graph>` tags.

5. **Generate reasoning trace and sufficiency judgment.** Prompt the LLM with the original question, all retrieved documents, and the linearized graph. Ask it to reason about what is known, what is missing, and whether the current evidence is sufficient to answer the question. The output should include a `<judgement>sufficient</judgement>` or `<judgement>insufficient</judgement>` tag.

6. **If insufficient, generate a targeted subquery.** When the judgment is "insufficient," the LLM should output a follow-up query targeting the specific missing information (e.g., if the graph has entity A linked to entity B but the question requires knowing B's birthplace, the subquery should ask about B's birthplace). Set `q_current` to this subquery and increment `t`.

7. **Repeat steps 2-6 until sufficient or max steps reached.** Cap iterations at 4 steps to prevent runaway retrieval. Track all retrieved documents and the evolving graph across iterations.

8. **Generate the final answer using documents + final graph.** Prompt the LLM with the original question, the union of all retrieved documents, and the final evolved graph. The graph acts as a structured attention guide, helping the model locate and connect the key pieces of evidence.

9. **Return the answer with provenance.** Include the answer, the final graph (as a readable summary of the evidence chain), and references to the source documents that contributed each fact.

## Concrete Examples

**Example 1: Multi-hop biographical question**

```
User: "Build a pipeline to answer: 'What university did the spouse of the
author of To Kill a Mockingbird attend?'"

Approach:
1. Initialize: q_0 = "What university did the spouse of the author of
   To Kill a Mockingbird attend?"
   Graph: empty, Documents: empty

2. Step 1 - Retrieve for q_0:
   Retrieved docs mention Harper Lee as author of To Kill a Mockingbird.
   Graph update:
     Entities: [Harper Lee (author)], [To Kill a Mockingbird (novel)]
     Relations: [Harper Lee -- wrote -- To Kill a Mockingbird]
   Judgment: insufficient (spouse not identified)
   Subquery: "Who was Harper Lee's spouse?"

3. Step 2 - Retrieve for "Who was Harper Lee's spouse?":
   Retrieved docs indicate Harper Lee never married.
   Graph update:
     Entities: [Harper Lee (author, never married)], [To Kill a Mockingbird (novel)]
     Relations: [Harper Lee -- wrote -- To Kill a Mockingbird],
                [Harper Lee -- marital status -- never married]
   Judgment: sufficient (question has a definitive answer)

4. Final answer: "Harper Lee, the author of To Kill a Mockingbird, never
   married, so the question has no applicable answer."
   Evidence chain: Graph shows Harper Lee -> wrote -> TKAM,
   Harper Lee -> marital status -> never married.
```

**Example 2: Implementing the retrieval loop in Python**

```python
User: "Write me a GraphAnchor-style iterative retrieval function."

# Output:

import json
from dataclasses import dataclass, field

@dataclass
class KnowledgeGraph:
    entities: dict[str, dict] = field(default_factory=dict)  # name -> attributes
    relations: list[tuple[str, str, str]] = field(default_factory=list)  # (subj, pred, obj)

    def merge(self, new_entities: dict, new_relations: list):
        for name, attrs in new_entities.items():
            normalized = name.strip().lower()
            if normalized in self.entities:
                self.entities[normalized].update(attrs)
            else:
                self.entities[normalized] = attrs
        for rel in new_relations:
            if rel not in self.relations:
                self.relations.append(rel)

    def linearize(self) -> str:
        ent_str = ", ".join(
            f"{name} ({', '.join(f'{k}: {v}' for k, v in attrs.items())})"
            for name, attrs in self.entities.items()
        )
        rel_str = ", ".join(
            f"{s} -- {p} -- {o}" for s, p, o in self.relations
        )
        return f"Entities: {ent_str}; Relations: {rel_str}"


def graph_anchor_retrieve(
    question: str,
    retriever,       # callable(query) -> list[str]
    llm,             # callable(prompt) -> str
    max_steps: int = 4,
    top_k: int = 5,
) -> dict:
    graph = KnowledgeGraph()
    all_docs = []
    current_query = question

    for step in range(max_steps):
        # Retrieve
        docs = retriever(current_query, top_k=top_k)
        all_docs.extend(docs)

        # Extract entities and relations via LLM
        extraction_prompt = build_extraction_prompt(
            docs, graph.linearize(), question, step
        )
        extraction = llm(extraction_prompt)
        new_entities, new_relations = parse_extraction(extraction)
        graph.merge(new_entities, new_relations)

        # Reason and judge sufficiency
        reasoning_prompt = build_reasoning_prompt(
            question, all_docs, graph.linearize()
        )
        reasoning = llm(reasoning_prompt)
        judgment = parse_judgment(reasoning)  # "sufficient" or "insufficient"
        subquery = parse_subquery(reasoning)

        if judgment == "sufficient":
            break
        current_query = subquery

    # Generate final answer with graph-guided attention
    answer_prompt = build_answer_prompt(question, all_docs, graph.linearize())
    answer = llm(answer_prompt)
    return {"answer": answer, "graph": graph, "documents": all_docs, "steps": step + 1}
```

**Example 3: Crafting the extraction prompt**

```
User: "Show me the prompt template for entity/relation extraction."

Output - Extraction prompt template:

SYSTEM: You are a knowledge extraction assistant. Given retrieved documents
and an existing knowledge graph, extract new entities and relations.

DOCUMENTS:
{documents}

CURRENT GRAPH:
<graph>{linearized_graph}</graph>

ORIGINAL QUESTION: {question}

INSTRUCTIONS:
1. Read the documents carefully.
2. Identify named entities (people, places, organizations, dates, events)
   not already in the graph.
3. Identify relations between entities as (subject, predicate, object) triples.
4. Output JSON with two keys:
   - "entities": {"entity_name": {"type": "...", "attributes": "..."}, ...}
   - "relations": [["subject", "predicate", "object"], ...]
5. Only extract facts supported by the documents. Do not infer.

OUTPUT (JSON only):
```

## Best Practices

- **Do:** Normalize entity names before merging (lowercase, strip whitespace, resolve aliases like "USA" vs "United States") to keep the graph deduplicated.
- **Do:** Include the original question in every prompt so the LLM stays focused on what matters rather than extracting every possible entity.
- **Do:** Cap retrieval at 4 iterations. Experiments show diminishing returns beyond this, and runaway loops waste tokens and time.
- **Do:** Use the linearized graph in the final answer prompt even when all documents are also provided -- the graph acts as an attention guide that helps the LLM connect dispersed facts.
- **Avoid:** Building the graph as a separate offline step. The key innovation is that graph construction happens *during* retrieval so it can guide subsequent queries.
- **Avoid:** Using graph database query languages (Cypher, SPARQL) for the LLM interaction. Linearize the graph as natural language text so any LLM can process it without tool calls.
- **Avoid:** Extracting relations without grounding them in the source document. Each triple should be traceable to a specific passage to maintain provenance.

## Error Handling

- **Retriever returns no relevant documents:** If a subquery retrieves nothing useful, fall back to reformulating the query using the current graph state. Rephrase using entity names and attributes already in the graph.
- **Entity extraction produces garbage:** Validate extracted entities against the source documents. If an entity name does not appear (or closely match) any span in the retrieved text, discard it.
- **Sufficiency judgment is unreliable:** If the LLM frequently says "sufficient" too early (producing wrong answers), add a confidence threshold: require that the graph contains at least one complete path from the question's subject to the expected answer type before accepting sufficiency.
- **Graph grows too large for context window:** Prune entities and relations with low relevance to the original question. Rank by shortest-path distance to question entities and drop nodes beyond a threshold (e.g., 3 hops).
- **Deduplication collisions:** When two genuinely distinct entities share the same normalized name (e.g., "Cambridge" the city in the UK vs. US), disambiguate using attributes extracted from documents (country, context).

## Limitations

- **Depends on retriever quality.** If the base retriever cannot surface relevant documents for subqueries, the iterative loop cannot recover. GraphAnchor improves *what you do with* retrieved docs, not retrieval itself.
- **Token-intensive.** Each iteration requires multiple LLM calls (extraction, reasoning, judgment). For latency-sensitive applications, consider limiting to 2 steps or using a smaller model for extraction.
- **Best suited for multi-hop factoid questions.** The technique shines on questions requiring 2-4 reasoning hops across entities. For single-hop lookup or open-ended generation tasks, the overhead is not justified.
- **Graph quality depends on extraction quality.** Noisy or incomplete extraction degrades the graph's usefulness as an index. Using structured extraction prompts with JSON output and validation helps but does not eliminate this.
- **Not a replacement for pre-built knowledge graphs.** If a high-quality domain KG already exists, query it directly. GraphAnchor is most valuable when no pre-existing structured knowledge is available and must be constructed on-the-fly from text.

## Reference

- **Paper:** [Graph-Anchored Knowledge Indexing for Retrieval-Augmented Generation](https://arxiv.org/abs/2601.16462v1) (Liu et al., 2026). Focus on Section 3 for the iterative retrieval algorithm, Section 4 for the linearized graph format, and Appendix for prompt templates.
- **Code:** [github.com/NEUIR/GraphAnchor](https://github.com/NEUIR/GraphAnchor) -- reference implementation with prompt templates in `prompts_GraphAnchor/en/` and main loop in `src/GraphAnchor.py`.