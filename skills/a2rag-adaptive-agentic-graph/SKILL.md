---
name: "a2rag-adaptive-agentic-graph"
description: "Build adaptive, cost-aware Graph-RAG pipelines that route queries through escalating retrieval stages (local -> bridge -> global) with triple-check verification and provenance map-back. Use when: 'build a graph RAG pipeline', 'implement adaptive retrieval for knowledge graphs', 'cost-aware multi-hop question answering', 'add evidence verification to RAG', 'handle mixed-difficulty queries efficiently', 'graph retrieval with source text grounding'."
---

# A2RAG: Adaptive Agentic Graph Retrieval

This skill enables Claude to design and implement Graph Retrieval-Augmented Generation systems that adaptively route queries through escalating retrieval stages based on difficulty, verify evidence with a triple-check mechanism, and ground answers back to source text via provenance mapping. The core insight from the A2RAG paper is that ~55% of queries can be answered with cheap local graph lookups, ~27% need bridge discovery, and only ~15% require expensive global diffusion -- so a progressive escalation pipeline cuts token consumption and latency by ~50% while improving recall by +10 points over flat retrieval baselines.

## When to Use

- When the user wants to build a RAG system over a knowledge graph and needs cost-efficient multi-hop reasoning
- When implementing question-answering pipelines that must handle a mix of easy single-hop and hard multi-hop queries
- When a graph-based retrieval system suffers from extraction loss (knowledge graph triples miss fine-grained qualifiers like dates, numbers, or exceptions that exist in source text)
- When the user asks for evidence verification or answer grounding in a retrieval pipeline
- When building an agentic retrieval loop that should know when to stop retrieving and when to escalate
- When optimizing an existing Graph-RAG system for cost (token budget, API calls, latency)

## Key Technique

A2RAG decouples **adaptive control** from **agentic retrieval**. The adaptive controller acts as the outer loop: it gates whether retrieval is needed at all (via summarized-KB similarity scoring), orchestrates answer generation, and enforces a **triple-check** verification before accepting any answer. The three checks are: (1) evidence relevance -- do retrieved passages actually address the query, (2) answer grounding -- are claims derivable from the evidence, and (3) query resolution -- does the answer adequately address the question. Only when all three pass does the system return a result. On failure, the controller identifies which check failed and rewrites the query accordingly (sharpening entities for relevance failures, requesting stricter grounding, or adding missing constraints for resolution failures), up to a bounded retry limit.

The agentic retriever inside this loop maintains a stateful evidence accumulator and escalates through three stages. **Stage 1 (Local)** extracts entity mentions from the query, aligns them to knowledge graph nodes via hybrid lexical-semantic matching, and collects 1-hop neighbor triples -- this resolves the majority of queries. **Stage 2 (Bridge)** discovers bridge entities that connect two or more query entity seeds within K hops, extracting the shortest connecting paths -- this handles multi-hop reasoning. **Stage 3 (Global Fallback)** runs degree-normalized Personalized PageRank from seed nodes, selects top-L nodes by PPR score, and critically performs **provenance map-back**: mapping each node back to its source text chunks to recover fine-grained qualifiers lost during graph construction. Escalation is monotonic (Local -> Bridge -> Global), with sufficiency checks gating each transition.

## Step-by-Step Workflow

1. **Build the knowledge graph index.** Parse the corpus into documents, extract entities and relations (via NER + relation extraction or LLM-based extraction), and construct a graph `G = (V, E)`. Store an offline provenance map `pi: V -> 2^D` linking each node back to its source text chunks. Precompute document summaries for gating.

2. **Implement the gating check.** For each incoming query, compute dense embedding similarity against precomputed document summaries. If `max(similarity_scores) < tau_g` (configurable threshold, typically 0.3-0.5), return "Abstain" immediately -- the corpus likely cannot answer this query.

3. **Implement Stage 1: Local Evidence Collection.** Extract entity mentions from the query using NER or phrase extraction. Align them to graph nodes using a hybrid score (edit distance + embedding cosine similarity). Collect 1-hop neighbors of aligned nodes, optionally filtered by relation seed constraints. Package triples as evidence.

4. **Implement Stage 2: Bridge Discovery.** For multi-hop queries where local evidence is insufficient, construct an augmented graph with inverse edges. Find bridge candidates -- nodes reachable from 2+ query entity seeds within K hops (K >= 2). Extract shortest paths connecting bridges to seeds. Cap path count and hop length to control evidence size.

5. **Implement Stage 3: Global Fallback with PPR and Provenance Map-Back.** When bridge discovery fails, run degree-normalized Personalized PageRank from seed nodes (lower-degree seeds get higher personalization weight to avoid hub bias). Select top-L nodes by PPR score. Map each selected node back to source text chunks via the provenance map `pi(v)`. This recovers fine-grained qualifiers (dates, numbers, exceptions) that graph triples lost.

6. **Implement the triple-check verification.** After generating an answer from retrieved evidence, run three validators: (a) `V_rel(q, E)` -- are the passages relevant to the query? (b) `V_grd(a, E)` -- is the answer grounded in the evidence? (c) `V_ans(q, a)` -- does the answer resolve the question? Use NLI models or prompted LLM classifiers. Accept only when all three pass.

7. **Implement failure-aware query rewriting.** When verification fails, identify the first violated check. Rewrite the query accordingly: sharpen entity/relation expressions for relevance failures, request stricter evidence for grounding failures, add missing constraints for resolution failures. Retry up to `I_max` iterations (2-3).

8. **Wire the outer adaptive control loop.** Connect gating -> agentic retrieval (with escalation) -> answer generation -> triple-check -> conditional rewrite/retry. Track token consumption per stage for cost monitoring.

9. **Add cost instrumentation.** Log which retrieval stage resolved each query. Monitor the stage distribution (target: ~55% local, ~27% bridge, ~15% global). Alert if global fallback usage exceeds 20%, indicating potential graph quality issues.

10. **Test with mixed-difficulty query sets.** Evaluate on both single-hop and multi-hop questions. Verify that easy queries terminate at Stage 1, multi-hop queries use Stage 2, and only adversarial/incomplete-graph cases hit Stage 3.

## Concrete Examples

**Example 1: Building an A2RAG pipeline for a documentation QA system**

User: "I have a knowledge graph built from our product documentation. I want to build a QA system that handles both simple factual questions and complex multi-hop questions efficiently."

Approach:
1. Index the documentation corpus with entity extraction and build provenance map linking KG nodes to source paragraphs
2. Implement the three-stage retriever with escalation gating
3. Add triple-check verification before returning answers

```python
class A2RAGPipeline:
    def __init__(self, kg, corpus, summaries, tau_g=0.4, i_max=2, alpha=0.15):
        self.kg = kg                # Knowledge graph with adjacency
        self.corpus = corpus        # Source text chunks indexed by doc ID
        self.summaries = summaries  # Precomputed doc summary embeddings
        self.tau_g = tau_g          # Gating threshold
        self.i_max = i_max          # Max rewrite retries
        self.alpha = alpha          # PPR teleport probability
        self.provenance = kg.provenance_map  # node -> source chunk IDs

    def answer(self, query: str) -> dict:
        # Step 1: Gating
        if self._gate(query) < self.tau_g:
            return {"answer": None, "status": "abstain"}

        q = query
        for i in range(self.i_max + 1):
            # Step 2: Agentic retrieval with escalation
            evidence = self._agentic_retrieve(q)

            # Step 3: Generate answer
            answer = self._generate(q, evidence)

            # Step 4: Triple-check verification
            checks = self._triple_check(q, answer, evidence)
            if all(checks.values()):
                return {"answer": answer, "evidence": evidence,
                        "stage": evidence.source_stage, "retries": i}

            # Step 5: Failure-aware rewrite
            failed = next(k for k, v in checks.items() if not v)
            q = self._rewrite(q, answer, evidence, failure_type=failed)

        return {"answer": answer, "status": "unverified", "retries": self.i_max}

    def _agentic_retrieve(self, query: str) -> Evidence:
        seeds = self._extract_and_align_entities(query)
        rel_seeds = self._extract_relation_seeds(query)

        # Stage 1: Local (1-hop neighbors of aligned entities)
        evidence = self._local_collect(seeds, rel_seeds)
        if self._evidence_sufficient(query, evidence):
            evidence.source_stage = "local"
            return evidence

        # Stage 2: Bridge discovery (K-hop connecting paths)
        bridge_evidence = self._bridge_discover(seeds, k_hops=2)
        evidence.merge(bridge_evidence)
        if self._evidence_sufficient(query, evidence):
            evidence.source_stage = "bridge"
            return evidence

        # Stage 3: Global PPR + provenance map-back
        ppr_nodes = self._degree_normalized_ppr(seeds, top_l=20)
        source_chunks = self._provenance_mapback(ppr_nodes)
        evidence.merge(source_chunks)
        evidence.source_stage = "global"
        return evidence
```

**Example 2: Adding provenance map-back to an existing Graph-RAG system**

User: "Our graph RAG often gives wrong answers because the KG triples don't capture specific dates and numbers from the source documents. How do I fix this?"

Approach:
1. This is the extraction loss problem -- graph abstraction drops fine-grained qualifiers
2. Implement a provenance map linking each KG node to its source text chunks
3. After graph traversal, map selected nodes back to source text for answer generation

```python
class ProvenanceMapBack:
    def __init__(self, kg, corpus):
        # Build offline provenance map: node_id -> set of chunk_ids
        self.prov_map = {}
        for node_id, node in kg.nodes.items():
            self.prov_map[node_id] = set()
            for chunk_id, chunk in corpus.items():
                if node.label.lower() in chunk.text.lower():
                    self.prov_map[node_id].add(chunk_id)

    def mapback(self, selected_nodes: list[str], corpus) -> list[str]:
        """Given graph-selected nodes, retrieve original source text."""
        chunk_ids = set()
        for node_id in selected_nodes:
            chunk_ids.update(self.prov_map.get(node_id, set()))
        return [corpus[cid].text for cid in chunk_ids]

# Usage: After PPR or bridge discovery selects graph nodes,
# ground the answer in source text instead of just triples
graph_nodes = ppr_select(seeds, top_l=20)
source_texts = provenance.mapback(graph_nodes, corpus)
answer = llm.generate(query=q, context=source_texts)  # Grounded in full text
```

**Example 3: Implementing triple-check verification for answer quality**

User: "I want to add verification to my RAG pipeline so it doesn't return hallucinated answers."

Approach:
1. Implement three independent checks: relevance, grounding, resolution
2. Only accept answers that pass all three
3. On failure, rewrite the query targeting the specific failure mode

```python
class TripleCheck:
    def __init__(self, nli_model, llm):
        self.nli = nli_model
        self.llm = llm

    def verify(self, query: str, answer: str, evidence: list[str]) -> dict:
        context = "\n".join(evidence)
        return {
            "relevance": self._check_relevance(query, context),
            "grounding": self._check_grounding(answer, context),
            "resolution": self._check_resolution(query, answer),
        }

    def _check_relevance(self, query, context) -> bool:
        """Do the retrieved passages actually address the query?"""
        return self.nli.entails(premise=context,
                                hypothesis=f"This text is relevant to: {query}")

    def _check_grounding(self, answer, context) -> bool:
        """Is every claim in the answer supported by the evidence?"""
        return self.nli.entails(premise=context, hypothesis=answer)

    def _check_resolution(self, query, answer) -> bool:
        """Does the answer adequately resolve the question?"""
        prompt = f"Does this answer fully resolve the question?\nQ: {query}\nA: {answer}\nRespond YES or NO."
        return "YES" in self.llm.generate(prompt).upper()

    def rewrite_on_failure(self, query, answer, evidence, failure_type) -> str:
        strategies = {
            "relevance": f"Rephrase to be more specific about entities and relations: {query}",
            "grounding": f"Find stronger evidence for: {query}. Previous answer '{answer}' was not grounded.",
            "resolution": f"Add missing constraints. Original: {query}. Incomplete answer: {answer}",
        }
        return self.llm.generate(f"Rewrite this query. Strategy: {strategies[failure_type]}")
```

## Best Practices

- **Do:** Build the provenance map (`node -> source chunks`) at index time, not query time. This is an offline operation and critical for the map-back stage to be fast.
- **Do:** Set the gating threshold `tau_g` conservatively (0.3-0.4). False negatives (abstaining on answerable queries) are worse than letting a few irrelevant queries through to the retriever.
- **Do:** Use degree-normalized personalization for PPR. Giving lower-degree (rarer) entity seeds higher weight prevents hub nodes from dominating results. The formula is `p0(u) = deg(u)^{-1} / Z`.
- **Do:** Track per-query stage resolution statistics. If more than 20% of queries hit Stage 3 (global), your knowledge graph likely has coverage gaps that should be addressed upstream.
- **Avoid:** Running all three retrieval stages unconditionally. The entire point of A2RAG is that most queries terminate early. Skipping the sufficiency check between stages negates the cost savings.
- **Avoid:** Using only graph triples for answer generation. Triples lose qualifiers (dates, quantities, exceptions). Always map back to source text for the final answer generation step, especially in Stage 3.

## Error Handling

| Failure Mode | Symptom | Resolution |
|---|---|---|
| Gating too aggressive | Many answerable queries return "Abstain" | Lower `tau_g` threshold; check summary embedding quality |
| Entity alignment misses | Seeds don't match KG nodes, Stage 1 returns empty evidence | Improve hybrid matching; add alias tables; lower alignment threshold |
| Bridge discovery timeout | Stage 2 hangs on dense subgraphs | Cap path count and hop budget K; set explicit traversal limits |
| PPR convergence issues | Stage 3 returns low-quality nodes | Increase iteration count for fixed-point computation; verify graph connectivity |
| Triple-check too strict | All answers fail verification, retry budget exhausted | Relax individual check thresholds; consider soft scoring instead of binary |
| Provenance map gaps | Map-back returns empty for some nodes | Audit extraction pipeline; ensure all entities have source chunk links |

## Limitations

- **Requires a pre-built knowledge graph.** A2RAG assumes an existing KG with reasonable entity/relation extraction quality. If the KG is very sparse or noisy, the local and bridge stages degrade significantly and most queries fall through to the expensive global stage.
- **Not suited for single-document QA.** The graph-based retrieval adds overhead that is only justified when the corpus has relational structure across multiple documents.
- **Triple-check adds latency for hard queries.** While easy queries are faster, the worst-case path (3 retrieval stages x 3 retry iterations x 3 verification checks) is more expensive than a flat retrieval baseline. The savings come from the aggregate workload, not individual hard queries.
- **Entity alignment quality is a bottleneck.** If the NER/entity linking step produces poor seed alignments, all three retrieval stages suffer. This is the most important component to get right.
- **PPR assumes graph connectivity.** Disconnected graph components cannot be bridged. Ensure the KG construction pipeline produces a reasonably connected graph.

## Reference

**Paper:** [A2RAG: Adaptive Agentic Graph Retrieval for Cost-Aware and Reliable Reasoning](https://arxiv.org/abs/2601.21162v1) (Liu et al., 2026). Look for: the three-stage escalation policy (Section 3.2), the triple-check verification formulation (Section 3.1), degree-normalized PPR with provenance map-back (Section 3.2.3), and the cost-vs-recall ablation tables (Section 4.3) showing that ~55% of queries resolve at Stage 1.