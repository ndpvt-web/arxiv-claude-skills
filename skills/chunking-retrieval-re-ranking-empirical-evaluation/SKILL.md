---
name: "chunking-retrieval-re-ranking-empirical-evaluation"
description: "Build and optimize two-stage RAG pipelines with bi-encoder retrieval, cross-encoder re-ranking, and empirically-validated chunking strategies. Use when: 'build a RAG pipeline', 'add re-ranking to retrieval', 'optimize chunking for documents', 'set up document QA with re-ranking', 'improve RAG faithfulness', 'two-stage retrieval pipeline'."
---

# Two-Stage RAG with Cross-Encoder Re-ranking

This skill enables Claude to design, implement, and optimize Retrieval-Augmented Generation (RAG) pipelines that use a two-stage retrieval architecture: fast bi-encoder retrieval followed by precise cross-encoder re-ranking. Based on empirical results from Maharjan & Yadav (2026), this approach raises faithfulness from ~0.35 (vanilla LLM) to ~0.80 (Advanced RAG) on domain-specific document QA, a 129% improvement over baseline. The skill covers chunking strategy selection, retrieval configuration, re-ranking integration, and RAGAS-based evaluation.

## When to Use

- When the user asks to build a RAG pipeline for document question answering over a specialized corpus (policy docs, legal texts, technical manuals, medical guidelines)
- When the user has an existing RAG system with mediocre faithfulness or relevance and wants to add re-ranking
- When the user needs to choose between chunking strategies (recursive character vs. semantic token splitting) for a document ingestion pipeline
- When the user asks how to reduce hallucinations in LLM answers grounded in retrieved documents
- When the user is setting up a vector store with retrieval and wants to know optimal top-k, chunk size, and overlap parameters
- When the user asks to evaluate a RAG system using RAGAS metrics (faithfulness, relevance)

## Key Technique

**Two-stage retrieval** separates the search problem into recall (finding candidates) and precision (selecting the best ones). Stage 1 uses a bi-encoder (e.g., `all-MiniLM-L6-v2`) that independently encodes queries and document chunks into dense vectors, then retrieves the top-k candidates via cosine similarity. This is fast but shallow — the query and document never "see" each other during encoding. Stage 2 applies a cross-encoder (e.g., `ms-marco-MiniLM-L-6-v2`) that jointly processes the query concatenated with each candidate chunk, allowing token-level attention between them. This captures causal relationships and disambiguates closely related concepts that bi-encoders miss.

**Chunking strategy directly impacts retrieval quality.** Recursive character-based splitting divides text at natural boundaries (paragraphs, sentences) using a fixed character budget. Token-based semantic splitting segments by semantic coherence using sentence embeddings to detect topic shifts. The empirical finding: neither strategy is universally superior — recursive character splitting preserves document structure better for hierarchical policy documents, while semantic splitting handles heterogeneous corpora with mixed content types. The critical bottleneck is that naive chunking fragments multi-step reasoning chains; structure-aware chunking that respects section boundaries is essential for complex queries.

**Quantitative benchmarks from the study:** Vanilla LLM achieved 0.35 faithfulness / 0.45 relevance. Basic RAG (bi-encoder only) achieved 0.62 / 0.70. Advanced RAG (bi-encoder + cross-encoder re-ranking) achieved 0.80 / 0.80. The cross-encoder recovered catastrophic failures — queries where Basic RAG scored 0.00 faithfulness were rescued to 0.80 by re-ranking.

## Step-by-Step Workflow

1. **Analyze the document corpus.** Inspect document structure (headings, sections, tables, lists), average document length, and content homogeneity. Determine if documents are hierarchical (policy frameworks) or flat (FAQ pages). This dictates chunking strategy.

2. **Select and configure the chunking strategy.**
   - For structured/hierarchical documents: Use recursive character splitting. Start with chunk_size=512 tokens, chunk_overlap=50 tokens. Split on paragraph boundaries first, then sentence boundaries, then character boundaries as fallback.
   - For heterogeneous/mixed documents: Use semantic splitting. Compute sentence embeddings, then segment where cosine similarity between consecutive sentences drops below a threshold (start at 0.75). Cap chunks at 512 tokens.
   - Always preserve section headers by prepending them to each chunk as metadata context.

3. **Build the embedding index.** Encode all chunks using a bi-encoder model (`all-MiniLM-L6-v2` for lightweight deployments, `bge-large-en-v1.5` or `e5-large-v2` for higher quality). Store in a vector database (FAISS for local, Chroma/Qdrant for persistent, Pinecone for managed). Include chunk metadata: source document, section title, chunk position index.

4. **Implement Stage 1: Bi-encoder retrieval.** Given a user query, encode it with the same bi-encoder, retrieve top-k=10 candidates by cosine similarity. This casts a wide net with high recall.

5. **Implement Stage 2: Cross-encoder re-ranking.** Pass each of the 10 candidates through a cross-encoder (`cross-encoder/ms-marco-MiniLM-L-6-v2`) that takes `(query, chunk_text)` as input and outputs a relevance score. Sort by score, select top-3 chunks as final context.

6. **Construct the generation prompt.** Assemble the top-3 re-ranked chunks into a context block with clear delimiters. Instruct the LLM to answer solely based on the provided context, cite sources, and state when information is insufficient rather than speculating.

7. **Generate and post-process the answer.** Pass the prompt to the LLM. Strip any content not grounded in the retrieved chunks. If the answer references information not in the context, flag it.

8. **Evaluate with RAGAS.** Run the pipeline against a test set of question-answer pairs. Measure faithfulness (is the answer supported by retrieved context?) and relevance (does the answer address the query?). Target: faithfulness >= 0.75, relevance >= 0.75.

9. **Iterate on failure modes.** If faithfulness is low, the re-ranker is passing irrelevant chunks — tighten the cross-encoder threshold or increase top-k for more candidates. If relevance is low, the chunks are fragmenting key information — increase chunk overlap or switch chunking strategy.

10. **Add structure-aware enhancements for multi-step queries.** For queries requiring reasoning across multiple document sections, implement parent-child chunk retrieval: retrieve the matching chunk plus its surrounding context (previous and next chunks from the same section).

## Concrete Examples

**Example 1: Building a policy document QA system**

User: "I have a folder of CDC policy PDFs. Build me a RAG pipeline that can answer questions about them accurately."

Approach:
1. Parse PDFs with `pymupdf` or `unstructured`, preserving section headers and hierarchy
2. Apply recursive character splitting: chunk_size=512, chunk_overlap=50, separators=["\n\n", "\n", ". "]
3. Prepend section headers to each chunk as context prefix
4. Embed with `all-MiniLM-L6-v2`, store in FAISS index
5. Implement two-stage retrieval: bi-encoder top-10, cross-encoder top-3
6. Wire to generation LLM with grounding prompt

Output (Python with LangChain):
```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain_community.vectorstores import FAISS
from sentence_transformers import CrossEncoder
import torch

# Step 1: Chunk documents
splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=50,
    separators=["\n\n", "\n", ". ", " "],
    length_function=len,
)
chunks = []
for doc in documents:
    splits = splitter.split_text(doc.page_content)
    for i, split in enumerate(splits):
        chunks.append({
            "text": f"[{doc.metadata['section']}] {split}",
            "metadata": {**doc.metadata, "chunk_index": i},
        })

# Step 2: Build vector index
embedder = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")
vectorstore = FAISS.from_texts(
    [c["text"] for c in chunks],
    embedder,
    metadatas=[c["metadata"] for c in chunks],
)

# Step 3: Two-stage retrieval
cross_encoder = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def retrieve(query: str, top_k_initial: int = 10, top_k_final: int = 3):
    # Stage 1: Bi-encoder retrieval
    candidates = vectorstore.similarity_search(query, k=top_k_initial)

    # Stage 2: Cross-encoder re-ranking
    pairs = [(query, doc.page_content) for doc in candidates]
    scores = cross_encoder.predict(pairs)
    ranked = sorted(zip(candidates, scores), key=lambda x: x[1], reverse=True)
    return [doc for doc, score in ranked[:top_k_final]]

# Step 4: Generate with grounding prompt
def answer(query: str):
    context_docs = retrieve(query)
    context = "\n---\n".join([doc.page_content for doc in context_docs])
    prompt = f"""Answer the question based ONLY on the context below.
If the context doesn't contain enough information, say "Insufficient information."
Cite the relevant section for each claim.

Context:
{context}

Question: {query}
Answer:"""
    return llm.invoke(prompt)
```

**Example 2: Adding re-ranking to an existing RAG pipeline**

User: "My RAG system retrieves ok results but the LLM still hallucinates. How do I add cross-encoder re-ranking?"

Approach:
1. Keep existing bi-encoder retrieval, increase top-k from 3 to 10
2. Add cross-encoder as a filtering layer between retrieval and generation
3. Evaluate before/after with RAGAS

Output (minimal integration):
```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")

def rerank(query: str, documents: list, top_k: int = 3) -> list:
    """Re-rank retrieved documents using cross-encoder."""
    pairs = [(query, doc.page_content) for doc in documents]
    scores = reranker.predict(pairs)
    ranked_indices = scores.argsort()[::-1][:top_k]
    return [documents[i] for i in ranked_indices]

# Integration into existing pipeline:
# Before: docs = vectorstore.similarity_search(query, k=3)
# After:
candidates = vectorstore.similarity_search(query, k=10)  # wider recall
docs = rerank(query, candidates, top_k=3)                 # precise selection
```

**Example 3: Evaluating chunking strategies**

User: "Should I use recursive character splitting or semantic splitting for my legal documents?"

Approach:
1. Implement both strategies on a sample of the corpus
2. Run identical retrieval pipeline with each
3. Compare RAGAS faithfulness and relevance scores

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_experimental.text_splitter import SemanticChunker
from langchain_community.embeddings import HuggingFaceEmbeddings
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy

embedder = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")

# Strategy A: Recursive character splitting
recursive_splitter = RecursiveCharacterTextSplitter(
    chunk_size=512, chunk_overlap=50
)

# Strategy B: Semantic splitting
semantic_splitter = SemanticChunker(
    embedder, breakpoint_threshold_type="percentile",
    breakpoint_threshold_amount=75,
)

# Run evaluation for each strategy
for name, splitter in [("recursive", recursive_splitter), ("semantic", semantic_splitter)]:
    chunks = splitter.split_documents(documents)
    vectorstore = build_index(chunks, embedder)
    pipeline = build_rag_pipeline(vectorstore, reranker, llm)
    results = evaluate(test_dataset, [faithfulness, answer_relevancy], pipeline)
    print(f"{name}: faithfulness={results['faithfulness']:.3f}, "
          f"relevance={results['answer_relevancy']:.3f}")
```

Decision guide:
- **Recursive character**: Better for documents with clear hierarchical structure (numbered sections, headings). Preserves logical units.
- **Semantic**: Better for long-form prose, transcripts, or documents without clear structure. Groups by topic coherence.
- **Both struggle with**: tables, multi-column layouts, and cross-referenced content spanning distant sections.

## Best Practices

- **Do:** Retrieve wide (top-k=10+) in stage 1, then narrow aggressively (top-3) with the cross-encoder. The bi-encoder is for recall; the cross-encoder is for precision.
- **Do:** Prepend section headers and document titles to each chunk before embedding. This provides critical context that improves both retrieval and generation.
- **Do:** Include chunk position metadata (index within document) so you can fetch neighboring chunks for multi-step reasoning queries.
- **Do:** Evaluate with RAGAS faithfulness metric, not just relevance. A system can retrieve relevant chunks but still generate unfaithful answers if the prompt doesn't enforce grounding.
- **Avoid:** Using only a bi-encoder with top-3 retrieval. The study shows this leaves ~18 points of faithfulness on the table compared to two-stage retrieval.
- **Avoid:** Chunks smaller than 256 tokens or larger than 1024 tokens. Too small fragments reasoning chains; too large dilutes relevant content within the context window.
- **Avoid:** Skipping overlap between chunks. Zero overlap creates hard boundaries where relevant information spanning two chunks is lost. Use at least 10% overlap.

## Error Handling

- **Cross-encoder returns uniform low scores:** The query is out-of-domain for the re-ranker. Fall back to bi-encoder-only results. Consider fine-tuning the cross-encoder on domain-specific query-passage pairs.
- **Faithfulness is high but relevance is low:** Retrieved chunks are factually correct but don't address the question. Increase the initial top-k to cast a wider net, or check that the embedding model handles domain-specific terminology.
- **Relevance is high but faithfulness is low:** The LLM is hallucinating beyond the retrieved context. Strengthen the grounding instruction in the prompt. Add explicit "only use information from the context" constraints. Consider adding citation requirements.
- **Multi-step queries fail:** The answer requires information from multiple distant sections. Implement parent-child retrieval — when a chunk matches, also include its surrounding chunks (window of +/- 1 from the same document section).
- **Chunking splits tables or lists:** Pre-process documents to identify and preserve table/list boundaries as atomic units before chunking. Use format-aware parsers like `unstructured` that detect these elements.

## Limitations

- **Latency cost:** Cross-encoder re-ranking adds inference time proportional to the number of candidates. For 10 candidates, expect ~50-100ms additional latency with `ms-marco-MiniLM-L-6-v2` on GPU. Batching helps.
- **Multi-step reasoning ceiling:** Even with re-ranking, queries requiring synthesis across 4+ document sections remain difficult. The 3-chunk context window limits reasoning depth. Knowledge graphs or iterative retrieval may be needed.
- **Domain transfer:** The `ms-marco-MiniLM-L-6-v2` cross-encoder is trained on web search data. For highly specialized domains (medical, legal), fine-tuning on domain QA pairs significantly improves re-ranking quality.
- **Chunking remains a bottleneck:** No chunking strategy perfectly preserves document semantics. Hierarchical documents with cross-references, footnotes, and appendices lose structural relationships during segmentation.
- **Evaluation scope:** The paper's benchmarks use 10 questions against CDC policy documents. Results may not directly transfer to other corpus sizes, document types, or question complexity distributions.

## Reference

Maharjan, A., & Yadav, U. (2026). *Chunking, Retrieval, and Re-ranking: An Empirical Evaluation of RAG Architectures for Policy Document Question Answering.* arXiv:2601.15457v1. [https://arxiv.org/abs/2601.15457v1](https://arxiv.org/abs/2601.15457v1)

Key takeaway: Two-stage retrieval (bi-encoder recall + cross-encoder precision) with top-10-to-top-3 filtering achieves 0.80 faithfulness on domain-specific QA — a 29% improvement over single-stage RAG and 129% over vanilla LLM baselines.