---
name: "when-should-search-more"
description: >
  Adaptive complex query optimization for RAG pipelines. Decides when a user query
  needs decomposition into multiple sub-queries vs. a single search, then fuses
  results with rank-score fusion. Use when building or improving retrieval-augmented
  generation systems that handle complex, multi-hop, or ambiguous queries.
  Triggers: "optimize my RAG queries", "improve retrieval for complex questions",
  "build adaptive search pipeline", "handle multi-hop queries in RAG",
  "my search results are poor for compound questions",
  "implement query decomposition for retrieval"
---

# Adaptive Complex Query Optimization (ACQO)

This skill teaches Claude to build adaptive retrieval pipelines that intelligently decide
whether a user query should be searched as-is or decomposed into multiple sub-queries,
then robustly fuse the results. Based on the ACQO framework, the approach uses two core
modules -- Adaptive Query Reformulation (AQR) to dynamically control query decomposition,
and Rank-Score Fusion (RSF) to merge results from heterogeneous retrievers -- delivering
state-of-the-art retrieval on complex query benchmarks while being 9x faster than
multi-turn search baselines.

## When to Use

- When the user is building a RAG pipeline that must handle compound questions (e.g., "Compare the GDP of France and Germany in 2023")
- When retrieval quality degrades on multi-hop queries that require chaining facts across documents
- When the user has ambiguous queries where a single entity name could refer to multiple things (disambiguation)
- When building a search API that needs to decide at runtime whether one search call or several are needed
- When the user wants to combine results from different retrieval backends (BM25, dense embeddings, hybrid) into a single ranked list
- When improving an existing RAG system where complex queries return irrelevant or incomplete results
- When the user asks to implement query rewriting, decomposition, or expansion logic

## Key Technique

**The core insight**: Not every query benefits from decomposition. Simple factoid queries perform best with a single well-formed search. Complex queries -- those requiring disambiguation (resolving ambiguous entities) or decomposition (breaking multi-objective questions into parts) -- need multiple parallel sub-queries. ACQO learns *when* to decompose and *how many* sub-queries to generate, avoiding the wasted compute and noise introduced by always decomposing.

**Adaptive Query Reformulation (AQR)** classifies each incoming query into one of three paths: (1) pass-through for simple queries, (2) disambiguation into entity-specific sub-queries, or (3) decomposition into logically independent sub-questions. The module generates reformulated queries in a structured output format, where each sub-query targets a distinct information need. This avoids the search-space explosion of fixed decomposition strategies that always split queries into N parts regardless of complexity.

**Rank-Score Fusion (RSF)** merges documents returned by multiple sub-queries using a two-tier scoring system. First, it computes a consensus rank `P(p) = 1 / sum(1/r_j)` that harmonically aggregates each document's rank across sub-queries -- documents ranked highly by multiple sub-queries rise to the top. Second, it uses the maximum retrieval score `S(p) = max(s_j)` as a tiebreaker, capturing the strongest single-query signal. Documents are sorted lexicographically by (P, S). This works across heterogeneous retrievers (sparse and dense) without score normalization, and provides a stable, differentiable signal for reward computation during RL training.

## Step-by-Step Workflow

1. **Classify query complexity.** Analyze the incoming query for indicators of complexity: multiple entities, comparative language ("compare", "vs", "difference between"), temporal qualifiers across different periods, conjunctions joining independent information needs, or ambiguous entity references. Assign a complexity label: `simple`, `ambiguous`, or `compound`.

2. **Route simple queries directly.** If the query is a single-intent factoid or lookup, pass it to the retriever unchanged. Do not decompose -- decomposition adds latency and can dilute relevance for straightforward queries.

3. **Decompose compound queries into sub-queries.** For queries classified as `compound`, extract each independent information need as a separate sub-query. Each sub-query should be self-contained and searchable on its own. Preserve any shared context (e.g., time period, domain) in each sub-query so retrieval doesn't lose scope.

4. **Disambiguate ambiguous queries.** For queries classified as `ambiguous`, generate one sub-query per plausible interpretation of the ambiguous entity or term. Add disambiguating context to each variant (e.g., "Mercury planet" vs. "Mercury element" vs. "Mercury mythology").

5. **Execute retrieval for each sub-query.** Send each sub-query (or the single original query) to your retrieval backend(s). If using multiple retrievers (BM25 + dense), run them in parallel for each sub-query. Collect top-K results per sub-query.

6. **Apply Rank-Score Fusion to merge results.** For each unique document across all sub-query result sets, compute: (a) the harmonic rank aggregation `P(doc) = 1 / sum(1/rank_j)` across the sub-queries where it appeared, and (b) the maximum retrieval score `S(doc) = max(score_j)`. Sort all documents by P descending, then S descending as tiebreaker.

7. **Truncate to final top-K.** Take the top K documents from the fused ranking as the final retrieval context for your downstream LLM.

8. **Feed context to the generator.** Pass the fused, ranked documents as context to your generation model, along with the original (non-decomposed) user query, so the LLM answers the user's actual question.

9. **Monitor and log decomposition decisions.** Track what fraction of queries are routed to each path (simple/ambiguous/compound), and the retrieval precision at each path. This data reveals whether your complexity classifier is well-calibrated or needs tuning.

10. **Iterate with curriculum difficulty.** If fine-tuning the decomposition model, start training on queries where decomposition clearly helps (high complexity gap between decomposed and raw retrieval), then progressively add harder edge cases. Filter out trivially easy and impossibly hard examples to keep the model in an optimal learning zone.

## Concrete Examples

**Example 1: Building an adaptive search function for a RAG chatbot**

User: "My RAG chatbot gives bad answers for complex questions like 'What were the causes and consequences of the 2008 financial crisis?' but works fine for simple lookups. Help me fix the retrieval."

Approach:
1. Classify the query: contains two independent information needs ("causes" and "consequences") -- label as `compound`.
2. Decompose into sub-queries:
   - Sub-query A: "What were the causes of the 2008 financial crisis?"
   - Sub-query B: "What were the consequences of the 2008 financial crisis?"
3. Retrieve top-10 documents for each sub-query.
4. Fuse with RSF: a document about "subprime mortgage collapse leading to global recession" appears at rank 2 in A and rank 5 in B, giving P = 1/(1/2 + 1/5) = 10/7 ~ 1.43. A document about "Lehman Brothers bankruptcy causes" appears at rank 1 in A only, giving P = 1/(1/1) = 1.0. Sort descending by P -- the cross-cutting document ranks higher.
5. Return fused top-10 to the LLM with the original question.

Output (Python implementation):
```python
from dataclasses import dataclass

@dataclass
class FusedDoc:
    doc_id: str
    content: str
    harmonic_rank: float
    max_score: float

def classify_query(query: str) -> str:
    """Classify query complexity. In production, use an LLM call or trained classifier."""
    compound_signals = ["and", "compare", "vs", "difference", "causes and", "both"]
    ambiguous_signals = ["mercury", "apple", "jaguar", "python"]  # domain-specific

    query_lower = query.lower()
    if any(signal in query_lower for signal in compound_signals):
        return "compound"
    if any(signal in query_lower for signal in ambiguous_signals):
        return "ambiguous"
    return "simple"

def decompose_query(query: str, complexity: str) -> list[str]:
    """Decompose query into sub-queries based on complexity type.
    In production, use an LLM to generate sub-queries."""
    if complexity == "simple":
        return [query]
    if complexity == "compound":
        # LLM prompt: "Break this query into independent sub-questions: {query}"
        # Placeholder showing the structure:
        return [
            "What were the causes of the 2008 financial crisis?",
            "What were the consequences of the 2008 financial crisis?",
        ]
    if complexity == "ambiguous":
        # LLM prompt: "List disambiguated versions of this query: {query}"
        return [query]  # Expand per entity interpretation
    return [query]

def rank_score_fusion(
    results_per_subquery: list[list[tuple[str, str, float]]],
    top_k: int = 10,
) -> list[FusedDoc]:
    """Fuse results from multiple sub-queries using RSF.

    Args:
        results_per_subquery: List of (doc_id, content, score) per sub-query,
                              ordered by rank (index 0 = rank 1).
        top_k: Number of documents to return.
    """
    doc_ranks: dict[str, list[int]] = {}
    doc_scores: dict[str, list[float]] = {}
    doc_content: dict[str, str] = {}

    for sq_results in results_per_subquery:
        for rank_idx, (doc_id, content, score) in enumerate(sq_results):
            rank = rank_idx + 1  # 1-indexed
            doc_ranks.setdefault(doc_id, []).append(rank)
            doc_scores.setdefault(doc_id, []).append(score)
            doc_content[doc_id] = content

    fused = []
    for doc_id in doc_ranks:
        harmonic_rank = 1.0 / sum(1.0 / r for r in doc_ranks[doc_id])
        max_score = max(doc_scores[doc_id])
        fused.append(FusedDoc(doc_id, doc_content[doc_id], harmonic_rank, max_score))

    # Sort descending by harmonic_rank, then by max_score as tiebreaker
    fused.sort(key=lambda d: (d.harmonic_rank, d.max_score), reverse=True)
    return fused[:top_k]

def adaptive_retrieve(query: str, retriever, top_k: int = 10) -> list[FusedDoc]:
    """Full ACQO-style adaptive retrieval pipeline."""
    complexity = classify_query(query)
    sub_queries = decompose_query(query, complexity)

    all_results = []
    for sq in sub_queries:
        results = retriever.search(sq, top_k=top_k)  # Returns [(doc_id, content, score)]
        all_results.append(results)

    return rank_score_fusion(all_results, top_k=top_k)
```

**Example 2: Adding disambiguation to a search API**

User: "Users search for 'python' on our docs site and get a mix of Python language docs and python snake care articles. How do I handle this?"

Approach:
1. Classify "python" as `ambiguous`.
2. Generate disambiguation sub-queries:
   - "Python programming language"
   - "python snake reptile"
3. Retrieve separately for each interpretation.
4. Fuse with RSF -- documents relevant to both interpretations (rare) rank highest; otherwise, each cluster retains its internal ranking.
5. Optionally: present the user with clustered results labeled by interpretation, or use user context (e.g., they're on a programming site) to weight one interpretation.

Output:
```python
def disambiguate_and_retrieve(query: str, retriever, user_context: str = ""):
    """Handle ambiguous queries by retrieving per interpretation and fusing."""
    # Step 1: Generate interpretations (use LLM in production)
    interpretations = generate_disambiguations(query)
    # e.g., ["Python programming language", "python snake reptile"]

    # Step 2: Retrieve per interpretation
    results_per_interp = []
    for interp in interpretations:
        contextualized = f"{interp} {user_context}".strip() if user_context else interp
        results_per_interp.append(retriever.search(contextualized, top_k=20))

    # Step 3: Fuse
    fused = rank_score_fusion(results_per_interp, top_k=10)
    return fused
```

**Example 3: Implementing the fusion in a LangChain RAG pipeline**

User: "I'm using LangChain with a vector store. How do I add query decomposition with proper result merging?"

Approach:
1. Create a custom retriever that wraps the existing vector store.
2. Add classification and decomposition as a pre-retrieval step.
3. Implement RSF as a post-retrieval combiner.

Output:
```python
from langchain.schema import BaseRetriever, Document
from langchain.callbacks.manager import CallbackManagerForRetrieverRun

class AdaptiveRetriever(BaseRetriever):
    """LangChain retriever with ACQO-style adaptive decomposition and RSF."""

    base_retriever: BaseRetriever
    llm: Any  # LLM for query classification and decomposition
    top_k: int = 10

    def _get_relevant_documents(
        self, query: str, *, run_manager: CallbackManagerForRetrieverRun
    ) -> list[Document]:
        # Classify
        complexity = self._classify(query)
        sub_queries = self._decompose(query, complexity)

        # Retrieve per sub-query
        all_results = []
        for sq in sub_queries:
            docs = self.base_retriever.get_relevant_documents(sq)
            all_results.append(docs)

        # Fuse using RSF
        return self._rank_score_fusion(all_results)

    def _classify(self, query: str) -> str:
        response = self.llm.predict(
            f"Classify this query as 'simple', 'compound', or 'ambiguous'. "
            f"Compound means it has multiple independent information needs. "
            f"Ambiguous means a key term has multiple meanings. "
            f"Reply with one word only.\n\nQuery: {query}"
        )
        return response.strip().lower()

    def _decompose(self, query: str, complexity: str) -> list[str]:
        if complexity == "simple":
            return [query]
        response = self.llm.predict(
            f"Break this {complexity} query into self-contained sub-queries, "
            f"one per line. Each must be independently searchable.\n\nQuery: {query}"
        )
        sub_queries = [sq.strip() for sq in response.strip().split("\n") if sq.strip()]
        return sub_queries if sub_queries else [query]

    def _rank_score_fusion(self, results_per_sq: list[list[Document]]) -> list[Document]:
        doc_map: dict[str, Document] = {}
        doc_ranks: dict[str, list[int]] = {}

        for sq_docs in results_per_sq:
            for rank, doc in enumerate(sq_docs, 1):
                key = doc.page_content[:200]  # dedup key
                doc_map[key] = doc
                doc_ranks.setdefault(key, []).append(rank)

        scored = []
        for key, doc in doc_map.items():
            p = 1.0 / sum(1.0 / r for r in doc_ranks[key])
            scored.append((p, doc))

        scored.sort(key=lambda x: x[0], reverse=True)
        return [doc for _, doc in scored[:self.top_k]]
```

## Best Practices

- **Do:** Classify query complexity before deciding to decompose. Simple queries should skip decomposition entirely -- it adds latency and can reduce precision.
- **Do:** Make each sub-query self-contained. Include shared context (dates, domains, constraints) in every sub-query so the retriever has full scope.
- **Do:** Use harmonic rank aggregation rather than simple reciprocal rank fusion (RRF). Harmonic aggregation naturally rewards documents that appear consistently across sub-queries rather than just in one.
- **Do:** Cap the number of sub-queries at 3-5. Beyond that, the marginal retrieval gain drops while compute cost grows linearly.
- **Avoid:** Always decomposing every query. The paper shows that forcing decomposition on simple queries degrades performance compared to pass-through.
- **Avoid:** Normalizing retrieval scores across different backends before fusion. RSF uses rank positions as the primary signal precisely because scores from BM25 and dense retrievers are not comparable. Use `max(score)` only as a tiebreaker within the same retriever type.

## Error Handling

- **Empty sub-query results:** If a sub-query returns zero documents, exclude it from the fusion step. Do not assign infinite rank -- simply skip that sub-query's contribution to the harmonic aggregation.
- **Decomposition produces a single sub-query identical to the original:** Treat this as the simple path. The classifier may be uncertain -- log it for review but proceed with single-query retrieval.
- **LLM generates malformed sub-queries:** Validate that each sub-query is non-empty and under a reasonable token length. Fall back to the original query if decomposition output is unparseable.
- **Retriever timeout on parallel sub-queries:** Set per-sub-query timeouts. If one sub-query times out, proceed with fusion using the results that did return. Partial fusion is better than no results.
- **Score ties in fusion:** When two documents have identical harmonic rank and max score, preserve the order from the highest-priority sub-query (the first one in the decomposition list, which typically maps to the primary information need).

## Limitations

- Query classification accuracy depends on the LLM or classifier used. Misclassifying a simple query as compound wastes compute; misclassifying a compound query as simple returns incomplete results.
- The approach assumes sub-queries are largely independent. Heavily sequential reasoning chains (e.g., "Who is the spouse of the CEO of the company that made the first iPhone?") require iterative retrieval, not parallel decomposition.
- RSF works best when sub-queries return overlapping document pools. If sub-queries target entirely disjoint document sets, harmonic fusion reduces to simple concatenation with no cross-query ranking benefit.
- The technique adds one LLM call for classification and decomposition per query. For latency-critical applications serving thousands of QPS, consider caching decomposition patterns for common query templates.
- Fine-tuning the decomposition model with RL (as described in the paper) requires access to retrieval ground truth and GPU compute. The heuristic and LLM-prompting approaches shown above are practical approximations.

## Reference

**Paper:** "When should I search more: Adaptive Complex Query Optimization with Reinforcement Learning" (Wen et al., 2026) -- [arXiv:2601.21208](https://arxiv.org/abs/2601.21208v1)

Read for: The Rank-Score Fusion formula (Section 3.2), the curriculum RL two-stage training strategy (Section 3.3), and the ablation study showing when decomposition helps vs. hurts (Table 3).