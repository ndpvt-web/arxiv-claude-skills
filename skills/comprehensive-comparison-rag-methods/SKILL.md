---
name: "comprehensive-comparison-rag-methods"
description: "Select and configure the right RAG strategy for conversational QA systems based on dataset characteristics. Use when: 'build a conversational RAG pipeline', 'choose a RAG method for multi-turn QA', 'my RAG pipeline performs worse than no retrieval', 'optimize retrieval for dialogue systems', 'compare RAG strategies for my dataset', 'reranking vs HyDE vs hybrid BM25'."
---

# Conversational RAG Strategy Selection and Configuration

This skill enables Claude to recommend, implement, and debug RAG (Retrieval-Augmented Generation) strategies for multi-turn conversational question answering systems. Based on a systematic comparison of 10 RAG methods across 8 diverse datasets (Alushi et al., EACL SRW 2026), this skill encodes the empirical finding that effective conversational RAG depends on alignment between retrieval strategy and dataset structure -- not method complexity. It provides a concrete decision framework for selecting between vanilla RAG, reranking, hybrid BM25, HyDE, query rewriting, and summarization approaches, and identifies the failure modes that cause advanced methods to underperform a no-retrieval baseline.

## When to Use

- When the user is building a conversational QA system and needs to choose a retrieval strategy
- When a RAG pipeline performs worse than prompting the LLM directly (the "below No-RAG baseline" problem)
- When the user asks which RAG method works best for their domain or corpus size
- When implementing multi-turn dialogue where coreference, topic shifts, and conversation history complicate retrieval
- When the user wants to add reranking, HyDE, or hybrid search to an existing RAG pipeline
- When debugging degraded retrieval quality as conversation length increases
- When evaluating whether an advanced RAG technique (query rewriting, summarization) is worth the added complexity

## Key Technique

The core insight is that RAG method selection should be driven by three dataset characteristics: (1) the context-to-question ratio (how many candidate passages exist per query), (2) the dialogue pattern (stable topic vs. topic-switching), and (3) the answer format (extractive vs. abstractive/informal). These three axes predict which retrieval strategy will succeed far better than any universal ranking of methods.

Three methods consistently outperform vanilla RAG across diverse settings: **Reranking** (cross-encoder rescoring of top-k candidates), **Hybrid BM25** (combining sparse lexical and dense semantic retrieval), and **HyDE** (generating a hypothetical answer, then using it as the retrieval query). Reranking adds minimal latency and is the safest default upgrade. Hybrid BM25 handles vocabulary mismatch between conversational queries and formal documents. HyDE excels when queries are short or ambiguous, as the hypothetical answer expands the query's semantic footprint -- it tripled MRR on one dataset (8.0 to 25.2).

Critically, several "advanced" methods reliably hurt performance. **Query rewriting** degrades results on datasets with topic switching or large corpora (INSCIT, QReCC, TopiOCQA) because the LLM rewrites introduce drift. **Summarization** strips crucial contextual detail, producing the worst average F1 scores across all methods. **Combining HyDE + Reranker** underperforms either method individually, suggesting the pipeline introduces compounding noise. The practical takeaway: start simple, measure against a no-RAG baseline, and only add complexity when the data demands it.

## Step-by-Step Workflow

1. **Profile the dataset characteristics.** Count the total contexts, compute the context-to-question ratio, and classify the dialogue pattern (stable topic, gradual drift, or hard topic switches). A ratio above 50:1 signals a large-corpus retrieval challenge where vanilla methods will struggle.

2. **Establish baselines.** Implement both a No-RAG baseline (LLM answers from parametric knowledge only) and a Vanilla RAG baseline (top-k dense retrieval with a sentence-transformer encoder). Measure F1 and MRR@5. If No-RAG already scores well, your dataset may not benefit from retrieval at all.

3. **Select the primary retrieval strategy using the decision matrix:**
   - Context-to-question ratio < 10:1 AND stable topics --> **Reranker** (cross-encoder on top-20 dense candidates, return top-5)
   - Context-to-question ratio > 50:1 OR topic-switching dialogues --> **Hybrid BM25** (weighted combination: 0.5 BM25 + 0.5 dense, tunable)
   - Short or ambiguous queries with moderate corpus --> **HyDE** (generate hypothetical answer with the LLM, embed it, retrieve against corpus)
   - Informal/domain-specific content (forums, support tickets) --> **Hybrid BM25 + Reranker** (lexical handles jargon, reranker handles relevance)

4. **Handle conversation history.** Serialize the last N turns (typically 3-5) as context prepended to the current query. Do NOT summarize history -- the paper shows summarization degrades retrieval. For coreference resolution, include the raw dialogue turns rather than rewriting them, unless you have verified that query rewriting improves your specific dataset.

5. **Implement the chosen retrieval method.** Use a standard stack: sentence-transformers or OpenAI embeddings for dense retrieval, rank-bm25 or Elasticsearch for sparse retrieval, and a cross-encoder model (e.g., `cross-encoder/ms-marco-MiniLM-L-6-v2`) for reranking. Retrieve top-20 candidates, then rerank/filter to top-5.

6. **Configure the generation prompt.** Pass retrieved contexts as numbered references in the system prompt. Include the conversation history. Instruct the LLM to answer based on the provided contexts and to say when information is insufficient rather than hallucinating.

7. **Evaluate with turn-aware metrics.** Measure F1 (token overlap), MRR@5 (retrieval ranking quality), and Recall@5 (coverage). Plot these metrics across conversation turns (turn 1, 2, 3, ..., 8+). Watch for degradation patterns: stable-then-declining curves indicate history is becoming noise after a threshold.

8. **Diagnose retrieval-generation misalignment.** If MRR is high but F1 is low, the retrieval works but the LLM ignores or misuses the context (common with informal/opinionated content). If MRR is low but F1 is reasonable, the LLM is answering from parametric knowledge -- retrieval is not contributing.

9. **Iterate on the strategy.** If the chosen method underperforms No-RAG on any metric, do not add more complexity. Instead: (a) check chunking strategy and chunk overlap, (b) verify embedding model domain alignment, (c) try grouping contexts by metadata (titles, sections) before indexing, (d) reduce conversation history window.

10. **Document the final configuration.** Record the chosen method, hyperparameters (top-k, reranker threshold, BM25 weight, history window), and per-turn performance metrics so the decision can be revisited as the dataset evolves.

## Concrete Examples

**Example 1: Building a customer support chatbot with a knowledge base**

User: "I'm building a conversational QA system for our customer support docs. We have about 1,200 help articles and users typically ask 3-5 follow-up questions. Which RAG approach should I use?"

Approach:
1. Profile: ~1,200 contexts is a moderate corpus. Support dialogues tend to stay on-topic (stable topic pattern). Context-to-question ratio depends on chunking but likely 5-15:1.
2. This fits the "moderate corpus, stable topics" profile --> **Reranker** as primary strategy.
3. Implementation:
   - Chunk articles into ~300-token passages with 50-token overlap
   - Index with a dense encoder (e.g., `all-MiniLM-L6-v2`)
   - Retrieve top-20 candidates per query
   - Rerank with `cross-encoder/ms-marco-MiniLM-L-6-v2`, keep top-5
   - Include last 3 conversation turns in the generation prompt
4. Baseline check: compare against No-RAG to confirm retrieval adds value.

Output configuration:
```python
rag_config = {
    "retrieval": "dense + cross-encoder-reranker",
    "embedding_model": "all-MiniLM-L6-v2",
    "reranker_model": "cross-encoder/ms-marco-MiniLM-L-6-v2",
    "chunk_size": 300,
    "chunk_overlap": 50,
    "top_k_retrieve": 20,
    "top_k_rerank": 5,
    "history_window": 3,
    "history_mode": "raw_turns",  # not summarized
}
```

**Example 2: Diagnosing a RAG pipeline that performs worse than no retrieval**

User: "My RAG system answers worse than just asking GPT directly. I'm using query rewriting to handle follow-up questions in a multi-turn setup over a large Wikipedia corpus."

Approach:
1. Identify the failure mode: Query rewriting over a large corpus with potential topic switching matches the paper's documented failure pattern -- query rewriting degrades on large corpora (TopiOCQA: 169K contexts, INSCIT: 29K contexts) and topic-switching dialogues.
2. The LLM-rewritten queries likely introduce semantic drift, retrieving irrelevant passages that mislead the generator.
3. Recommended fix: Replace query rewriting with **Hybrid BM25** (handles large corpus vocabulary diversity) and optionally add a reranker.
4. Group Wikipedia contexts by article title during indexing to reduce the effective search space.

Diagnostic steps:
```
1. Measure MRR@5 for current query-rewriting pipeline
2. Measure MRR@5 for vanilla RAG (no rewriting) as sanity check
3. Implement hybrid BM25 (alpha=0.5 sparse/dense blend)
4. Compare F1 scores across all three at turns 1, 3, 5, 7
5. If hybrid BM25 improves MRR but F1 stays flat, check if
   the LLM is ignoring retrieved context (generation issue)
```

**Example 3: Choosing between HyDE and reranking for a research paper QA system**

User: "I have a corpus of 500 research papers chunked into paragraphs. Users ask technical questions in multi-turn conversations. Should I use HyDE or reranking?"

Approach:
1. Profile: ~500 papers, likely 5,000-15,000 chunks. Moderate corpus. Technical queries are often short and use specialized vocabulary.
2. Both methods are strong candidates. Decision factors:
   - HyDE generates a hypothetical answer first, which works well when queries are terse ("What about the ablation?") because it expands the query semantics. Cost: one extra LLM call per query.
   - Reranking is cheaper (no LLM call, just a cross-encoder forward pass) and excels when initial retrieval already surfaces relevant candidates.
3. Recommendation: Start with **Reranker** (lower latency, no extra LLM cost). If short follow-up queries consistently miss relevant passages (low Recall@5), add **HyDE** as the retrieval query generator.
4. Do NOT combine both -- the paper shows HyDE + Reranker underperforms either alone.

```python
# Start with this
pipeline_v1 = ["dense_retrieval", "cross_encoder_rerank"]

# Only if recall is low on short queries, switch to this
pipeline_v2 = ["hyde_query_expansion", "dense_retrieval"]

# Do NOT do this -- empirically shown to degrade performance
pipeline_bad = ["hyde_query_expansion", "dense_retrieval", "cross_encoder_rerank"]
```

## Best Practices

**Do:**
- Always measure against a No-RAG baseline before concluding your RAG pipeline works. If No-RAG scores within 5% F1 of your RAG system, retrieval is not contributing meaningfully.
- Pass raw conversation history turns to the retriever rather than summarizing them. Summarization consistently removes critical detail needed for coreference resolution.
- Group indexed documents by metadata (title, section, source) during preprocessing. This reduces the effective corpus size and improves retrieval precision on large collections.
- Monitor per-turn performance. If metrics degrade after turn 5, reduce the history window rather than adding complexity.

**Avoid:**
- Do not default to the most complex RAG method available. Summarization and query rewriting frequently underperform vanilla RAG -- they must earn their place through measured improvement.
- Do not combine HyDE with reranking. The paper demonstrates this combination introduces compounding noise and underperforms either method individually.
- Do not assume high retrieval scores (MRR) guarantee good generation (F1). On domain-specific or informal content, retrieved passages may be technically relevant but stylistically misaligned with expected answers.
- Do not use query rewriting on datasets with frequent topic switches or corpora exceeding ~20K contexts without first validating it improves over vanilla RAG.

## Error Handling

**Retrieval returns irrelevant passages:** Check embedding model domain alignment. General-purpose encoders may fail on specialized vocabulary. Fine-tune or switch to a domain-adapted model.

**Performance degrades mid-conversation:** The history window is too large. Reduce from N turns to N-2 and re-measure. Alternatively, use a sliding window that drops the oldest turns.

**HyDE generates hallucinated hypothetical answers:** The hypothetical answer does not need to be factually correct -- it only needs to be semantically similar to relevant passages. However, if the domain is highly specialized, the LLM may generate off-topic hypotheticals. Mitigate by including a brief domain description in the HyDE prompt.

**High MRR but low F1 (retrieval-generation gap):** The LLM is not grounding its answers in the retrieved context. Strengthen the generation prompt with explicit instructions to cite or quote from provided passages. Consider few-shot examples that demonstrate grounded answering.

**BM25 component returns nothing useful:** Conversational queries are often fragmentary ("What about that?"). BM25 depends on lexical overlap, which fails on pronoun-heavy queries. Ensure the hybrid weight favors dense retrieval (e.g., 0.3 BM25 / 0.7 dense) for conversational settings, shifting toward BM25 only for terminology-heavy domains.

## Limitations

- The paper evaluates on English-language datasets only. The strategy rankings may not hold for morphologically rich or low-resource languages where BM25 behaves differently.
- All experiments use a single LLM (Llama 3) for generation. Different generators may interact differently with retrieved contexts, potentially changing which retrieval method is optimal.
- The decision framework assumes a standard chunked-passage retrieval paradigm. It does not cover graph-based RAG, multi-hop retrieval, or agentic RAG patterns where the LLM decides when and what to retrieve.
- Datasets with highly structured content (tables, code, structured data) are underrepresented. SQA covers table QA, but the findings may not generalize to code retrieval or structured knowledge bases.
- The study does not evaluate retrieval latency or cost at scale. In production, the computational cost of HyDE (extra LLM call per query) or cross-encoder reranking (quadratic with candidate count) may be prohibitive.

## Reference

Alushi, K., Strich, J., Biemann, C., & Semmann, M. (2026). *Comprehensive Comparison of RAG Methods Across Multi-Domain Conversational QA*. EACL SRW. [arXiv:2602.09552](https://arxiv.org/abs/2602.09552v1) | [Code](https://github.com/Klejda-A/exp-rag)

Key takeaway: Table 2 (F1 scores across all 8 datasets) and Figure 4 (per-turn performance curves) are the most actionable references for strategy selection. The context-to-question ratio analysis in Section 5 explains why methods fail on specific datasets.