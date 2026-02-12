---
name: "comprehensive-comparison-rag-methods"
description: "Select and configure the optimal RAG strategy for conversational QA systems by matching retrieval methods to dataset characteristics. Use when: 'which RAG method should I use', 'build a conversational QA pipeline', 'my RAG system performs worse than no retrieval', 'optimize multi-turn retrieval', 'compare RAG approaches for my dataset', 'reranking vs HyDE vs hybrid search'."
---

# Conversational RAG Method Selection and Configuration

This skill enables Claude to recommend, implement, and debug RAG (Retrieval-Augmented Generation) pipelines for multi-turn conversational question answering. It applies findings from a systematic empirical comparison of 11 RAG configurations across 8 diverse datasets, revealing that **method-dataset alignment matters more than method complexity**. The core insight: reranking, hybrid BM25, and HyDE consistently outperform vanilla RAG, while several "advanced" techniques (query rewriting, summarization) can degrade performance below a no-retrieval baseline.

## When to Use

- When a user is building a conversational QA system and needs to choose a retrieval strategy
- When a user reports that their RAG pipeline performs worse than prompting the LLM directly (the "below No-RAG" problem)
- When a user wants to add retrieval to a multi-turn chatbot with dialogue history
- When a user asks which retrieval method works best for their specific domain or dataset shape
- When a user needs to debug degrading retrieval quality over long conversations
- When a user is evaluating trade-offs between BM25, dense retrieval, reranking, and query transformation methods

## Key Technique

The paper's central contribution is a **dataset-aware RAG method selection framework**. Rather than defaulting to the most sophisticated pipeline, the study demonstrates that the critical variable is the **context-to-query ratio** of the corpus: how many candidate documents exist per expected query. Datasets with low ratios (CoQA: 0.06, Doc2Dial: 0.31) behave very differently from high-ratio datasets (TopiOCQA: 67.42, INSCIT: 58.76). Low-ratio datasets see strong gains from almost any retrieval method; high-ratio datasets punish imprecise retrieval because the needle-in-haystack problem dominates.

The three consistently winning methods are: **(1) Reranking** -- a cross-encoder rescores the top-k initial results, pushing relevant documents higher; **(2) Hybrid BM25** -- combines sparse lexical matching (BM25) with dense vector similarity, capturing both exact keyword matches and semantic similarity; **(3) HyDE** -- the LLM generates a hypothetical answer first, then uses that answer as the retrieval query, which aligns the query embedding closer to document embeddings. These methods work because they address the core failure mode of vanilla RAG in conversations: the user's current query is often underspecified (pronouns, ellipsis, topic shifts) and the raw query embedding lands far from the relevant document embedding.

Critically, the study finds that query rewriting and summarization-based methods frequently *hurt* performance. Query rewriting introduces hallucinated terms that derail retrieval. Summarization discards the precise tokens needed for extractive answers. The practical takeaway: start with reranking or hybrid BM25, add HyDE only if you can tolerate the extra LLM call per query, and avoid query rewriting unless you have validated it on your specific corpus.

## Step-by-Step Workflow

1. **Profile the dataset characteristics.** Count the total number of retrievable documents/chunks and the expected number of queries. Compute the context-to-query ratio. Classify as low-ratio (<1), medium-ratio (1-10), or high-ratio (>10). Check whether conversations involve topic switching (high-ratio datasets like TopiOCQA) or stay focused on a single document (low-ratio datasets like CoQA).

2. **Select the baseline retrieval method based on ratio.**
   - Low ratio (<1): Vanilla dense retrieval is often sufficient; reranking gives modest gains.
   - Medium ratio (1-10): Hybrid BM25 or reranking recommended as the default.
   - High ratio (>10): Reranking is essential; consider HyDE + Reranker for maximum recall.

3. **Configure the embedding and retrieval stack.** Use a sentence-transformer model (e.g., `all-MiniLM-L6-v2` for speed, or `bge-large-en-v1.5` for quality) with cosine similarity. Set top-k=5 for initial retrieval. Use a vector store like Chroma, Qdrant, or Pinecone. For hybrid BM25, add a sparse retriever (Elasticsearch, Tantivy, or rank_bm25) and merge results with reciprocal rank fusion.

4. **Add a cross-encoder reranker.** After initial retrieval of top-k candidates, apply a cross-encoder (e.g., `cross-encoder/ms-marco-MiniLM-L-6-v2` or `bge-reranker-v2-m3`) to rescore and reorder. This is the single highest-impact addition to any RAG pipeline.

5. **Implement dialogue history handling.** Prepend the last 2-3 conversation turns to the current query before embedding. Use a zero-shot prompt template that instructs the LLM to prioritize retrieved context over internal knowledge. Format: `[System prompt] [Retrieved contexts] [Dialogue history] [Current question]`.

6. **Optionally add HyDE for high-ratio corpora.** Before retrieval, prompt the LLM to generate a short hypothetical answer to the current question (1-2 sentences). Embed that hypothetical answer instead of the raw query. This shifts the query embedding into document-space, improving recall on datasets where queries are terse or anaphoric.

7. **Evaluate with the right metrics.** Measure retrieval quality with Recall@5 and MRR@5. Measure generation quality with token-level F1 (preferred over exact match for conversational answers). Track both metrics -- high MRR does not guarantee high F1 (the DoQA dataset shows MRR >90% with F1 <40% due to answer format mismatches).

8. **Run turn-level diagnostics.** Plot F1 and Recall@5 as a function of conversation turn number. If performance degrades after turn 3-5, the issue is likely topic switching or coreference resolution failure. Consider resetting retrieval context on detected topic shifts.

9. **Avoid these common traps.** Do not add query rewriting unless you have measured it improves recall on your specific data. Do not summarize retrieved contexts before passing to the generator -- pass the raw chunks. Do not increase top-k beyond 5-10 without reranking; more noise degrades generation.

10. **Iterate by measuring delta over No-RAG.** Always maintain a No-RAG baseline (LLM with dialogue history only, no retrieval). If any RAG configuration scores below this baseline on F1, it is actively harmful and should be removed or reconfigured.

## Concrete Examples

**Example 1: Building a customer support chatbot with a small knowledge base**

User: "I'm building a chatbot for our product documentation. We have about 200 help articles and users ask multi-turn questions. What RAG approach should I use?"

Approach:
1. Profile: ~200 documents, potentially thousands of queries. Context-to-query ratio <1 (low ratio).
2. Low ratio means vanilla dense retrieval will work reasonably, but reranking is cheap to add.
3. Recommend: Dense retrieval with `all-MiniLM-L6-v2` embeddings, top-k=5, followed by a cross-encoder reranker.
4. Skip HyDE -- the overhead of an extra LLM call per query is not justified at this ratio.
5. Include last 3 turns of dialogue history in the retrieval query.

Output configuration:
```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.cross_encoders import HuggingFaceCrossEncoder
from langchain.retrievers.document_compressors import CrossEncoderReranker

# Dense retrieval with reranking
embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")
vectorstore = Chroma.from_documents(docs, embeddings)
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 10})

# Cross-encoder reranker narrows to top 5
reranker_model = HuggingFaceCrossEncoder(model_name="cross-encoder/ms-marco-MiniLM-L-6-v2")
reranker = CrossEncoderReranker(model=reranker_model, top_n=5)

# Compose
from langchain.retrievers import ContextualCompressionRetriever
retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=base_retriever,
)
```

**Example 2: Open-domain conversational QA over a large corpus**

User: "We have 150K Wikipedia passages and users ask follow-up questions that switch topics mid-conversation. Vanilla RAG is giving terrible results."

Approach:
1. Profile: 150K documents, context-to-query ratio ~60 (high ratio, similar to TopiOCQA).
2. High ratio + topic switching = hardest scenario. Vanilla RAG will fail here.
3. Recommend: Hybrid BM25 + dense retrieval with reciprocal rank fusion, followed by reranker. Add HyDE for queries that are anaphoric (contain pronouns referring to prior turns).
4. Implement topic-shift detection: if cosine similarity between current query embedding and prior turn embedding drops below a threshold, reset dialogue context.

Output configuration:
```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

# Hybrid: BM25 + Dense with reciprocal rank fusion
bm25_retriever = BM25Retriever.from_documents(docs, k=10)
dense_retriever = vectorstore.as_retriever(search_kwargs={"k": 10})
hybrid_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, dense_retriever],
    weights=[0.4, 0.6],  # slightly favor dense for semantic matching
)

# HyDE: generate hypothetical answer, embed it for retrieval
def hyde_transform(query: str, chat_history: list, llm) -> str:
    prompt = f"Given this conversation:\n{format_history(chat_history)}\n\nWrite a short answer to: {query}"
    hypothetical = llm.invoke(prompt)
    return hypothetical.content

# Reranker on top
reranker = CrossEncoderReranker(model=reranker_model, top_n=5)
```

**Example 3: Debugging a RAG pipeline that underperforms No-RAG**

User: "I added RAG to my chatbot but the answers got worse. I'm using query rewriting to reformulate questions before retrieval."

Approach:
1. This matches the paper's finding that query rewriting degrades performance on multiple datasets (INSCIT, QReCC, TopiOCQA scored below No-RAG with query rewriting).
2. Query rewriting often introduces hallucinated entities or overly specific terms that derail retrieval.
3. Recommend: Remove query rewriting. Replace with either (a) passing the raw query + last 2-3 turns directly, or (b) HyDE if the corpus is large.
4. Measure Recall@5 before and after the change to confirm retrieval improvement.
5. Maintain the No-RAG baseline as a regression floor.

Diagnostic steps:
```
1. Run queries with No-RAG -> record F1 scores
2. Run same queries with current RAG (query rewriting) -> record F1
3. If RAG F1 < No-RAG F1, the retrieval is injecting harmful context
4. Replace query rewriting with direct hybrid retrieval + reranker
5. Re-measure: RAG F1 should now exceed No-RAG F1
```

## Best Practices

- **Do:** Always establish a No-RAG baseline before adding retrieval. Any RAG method that scores below this baseline is actively harmful and must be fixed or removed.
- **Do:** Use a cross-encoder reranker as the default addition to any retrieval pipeline. It is the single most consistent performance improvement across all dataset types.
- **Do:** Measure both retrieval metrics (Recall@5, MRR@5) and generation metrics (F1) independently. High retrieval scores do not guarantee good generation -- format mismatches between retrieved text and expected answer style cause silent failures.
- **Do:** Profile your dataset's context-to-query ratio before choosing a method. This single number predicts more about RAG effectiveness than the sophistication of the technique.
- **Avoid:** Query rewriting and summarization-based compression as default strategies. Both frequently degrade performance in multi-turn settings by introducing hallucinated terms or discarding critical tokens.
- **Avoid:** Increasing top-k retrieval beyond 10 without reranking. Larger k without quality filtering floods the generator's context window with irrelevant passages, causing hallucination.

## Error Handling

- **Retrieval returns irrelevant documents:** Check embedding model alignment. If using a general-purpose encoder on domain-specific text (medical, legal), fine-tune or switch to a domain-adapted model. Fall back to BM25 as a diagnostic -- if BM25 retrieves correctly but dense fails, the embedding space is the problem.
- **Performance degrades at later conversation turns:** Likely caused by topic switching or coreference accumulation. Implement a sliding window over dialogue history (last 3 turns) instead of passing the full conversation. Consider topic-shift detection to reset context.
- **HyDE generates a hallucinated hypothetical that derails retrieval:** Add a confidence check: if the hypothetical answer is highly uncertain (detectable via token probability or an explicit "I don't know"), fall back to the raw query for retrieval.
- **Hybrid BM25 and dense retrieval disagree on results:** This is expected and is the source of hybrid's strength. Use reciprocal rank fusion with tunable weights (start 0.4 BM25 / 0.6 dense) and adjust based on whether your queries are more keyword-heavy or semantic.

## Limitations

- All findings are from experiments with Llama 3 8B Instruct and `all-MiniLM-L6-v2` embeddings. Different model sizes or embedding models may shift the relative ordering of methods.
- The study evaluates English-language datasets only. Multilingual conversational QA may exhibit different retrieval dynamics.
- Token-level F1 is the primary generation metric. For tasks requiring exact factual answers (e.g., dates, numbers), exact match or entity-level metrics may be more appropriate.
- The study does not evaluate RAG with tool use, agentic retrieval, or iterative retrieval-generation loops. These newer paradigms may outperform the static retrieve-then-generate setup.
- Chunking strategy is fixed across experiments. For production systems, chunk size and overlap tuning can significantly affect results independently of the retrieval method.

## Reference

**Paper:** "Comprehensive Comparison of RAG Methods Across Multi-Domain Conversational QA" (Alushi et al., 2026). Accepted at EACL SRW 2026. [arXiv:2602.09552](https://arxiv.org/abs/2602.09552v1)
**Code:** [github.com/Klejda-A/exp-rag](https://github.com/Klejda-A/exp-rag)
**What to look for:** Tables 2-3 for per-dataset F1 scores across all methods; Section 5.3 for turn-level performance evolution; Table 1 for dataset characteristics (especially context-to-query ratios that drive method selection).