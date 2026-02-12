---
name: "chunking-retrieval-re-ranking-empirical-evaluation"
description: "Build and optimize RAG pipelines with empirically-validated chunking, retrieval, and cross-encoder re-ranking strategies. Use when: 'build a RAG pipeline', 'set up document question answering', 'improve retrieval accuracy', 'add re-ranking to my RAG system', 'chunk documents for vector search', 'reduce LLM hallucinations with retrieval'"
---

This skill enables Claude to design, implement, and optimize Retrieval-Augmented Generation (RAG) pipelines using a two-stage retrieve-then-re-rank architecture. Based on empirical findings from Maharjan & Yadav (2026), the approach combines bi-encoder retrieval with cross-encoder re-ranking to achieve significantly higher faithfulness (0.80) compared to single-stage RAG (0.62) or vanilla LLM baselines (0.35). The skill covers chunking strategy selection, embedding configuration, retrieval tuning, and re-ranking integration for domain-specific document QA.

## When to Use

- When the user asks to build a RAG pipeline for document question answering
- When the user wants to reduce hallucinations in LLM responses grounded in a document corpus
- When the user needs to choose between chunking strategies (character-based vs. semantic/token-based)
- When the user asks to add re-ranking to an existing retrieval system
- When the user is building QA over regulatory, policy, legal, or compliance documents where factual accuracy is critical
- When the user wants to evaluate or benchmark their RAG pipeline with faithfulness and relevance metrics
- When the user asks how to improve retrieval quality or reduce irrelevant context in their RAG system

## Key Technique

**Two-Stage Retrieve-and-Re-rank Architecture.** The core insight is that basic RAG (single-stage bi-encoder retrieval) leaves significant faithfulness on the table. A two-stage pipeline first retrieves a broad candidate set using fast bi-encoder cosine similarity (top-k=10), then re-scores those candidates with a cross-encoder model that jointly attends to the query and each candidate passage. The cross-encoder (e.g., `ms-marco-MiniLM-L-6-v2`) produces far more accurate relevance scores than cosine similarity alone because it processes query-passage pairs together rather than comparing independent embeddings. The final top-3 passages after re-ranking are passed to the LLM as context.

**Chunking Strategy Matters, But Differently Than Expected.** The study compares recursive character-based splitting (which splits on paragraph/sentence boundaries at fixed character limits) against token-based semantic splitting (which segments based on token-level semantic coherence). Both strategies improve over no chunking, but the re-ranking stage compensates for chunking noise — meaning the choice of re-ranker has more impact than the choice of chunker. However, chunking remains a bottleneck for multi-step reasoning where the answer spans multiple non-adjacent sections.

**Evaluation via RAGAS.** The pipeline is evaluated using the RAGAS framework, which automatically scores faithfulness (does the answer follow from the retrieved context?) and relevance (is the retrieved context pertinent to the question?). This enables systematic comparison without manual annotation.

## Step-by-Step Workflow

1. **Ingest and preprocess the document corpus.** Load source documents (PDF, HTML, plain text). Strip headers, footers, tables of contents, and boilerplate. Normalize whitespace and encoding. Preserve section headings as metadata for later attribution.

2. **Select and configure a chunking strategy.** Choose one of:
   - **Recursive character splitting**: Use `RecursiveCharacterTextSplitter` with separators `["\n\n", "\n", ". ", " "]`, chunk size 500-1000 characters, overlap 100-200 characters. Best for well-structured documents with clear paragraph boundaries.
   - **Token-based semantic splitting**: Use a sentence-transformer tokenizer to split at semantic boundaries, targeting 128-256 tokens per chunk with 20-30 token overlap. Better for dense, unstructured text where character counts don't align with meaning.

3. **Generate embeddings for all chunks.** Use a bi-encoder embedding model (e.g., `all-MiniLM-L6-v2`, 384 dimensions) to encode each chunk into a dense vector. Store vectors in a FAISS flat index or similar vector store. Attach chunk metadata (source document, section heading, chunk index) alongside each vector.

4. **Implement first-stage retrieval.** On query, encode the user question with the same bi-encoder model. Retrieve the top-k=10 candidate chunks by cosine similarity from the vector store. This stage prioritizes recall — cast a wide net.

5. **Implement second-stage cross-encoder re-ranking.** Load a cross-encoder model (e.g., `cross-encoder/ms-marco-MiniLM-L-6-v2`). Score each of the 10 candidate chunks by passing `(query, chunk_text)` pairs through the cross-encoder. Sort by cross-encoder score descending. Select the top-3 chunks. This stage prioritizes precision.

6. **Construct the augmented prompt.** Format the top-3 re-ranked chunks into a context block with source attribution. Prepend a system instruction that tells the LLM to answer strictly based on the provided context, cite sources, and say "I don't have enough information" when the context is insufficient.

7. **Generate the answer.** Pass the augmented prompt to the LLM. Use a moderate temperature (0.1-0.3) for factual QA to minimize creative deviation from the context.

8. **Evaluate with RAGAS metrics.** Score the pipeline output using:
   - **Faithfulness**: proportion of claims in the answer that are supported by the retrieved context (target >= 0.80).
   - **Answer relevance**: semantic similarity between the answer and the question (target >= 0.75).
   - **Context precision**: proportion of retrieved chunks that are actually relevant.
   Run evaluation over a curated set of at least 10-20 question-answer pairs covering diverse query types.

9. **Iterate on chunking and retrieval parameters.** If faithfulness is low but context precision is high, the LLM is hallucinating despite good retrieval — tighten the prompt or reduce temperature. If context precision is low, increase top-k at the first stage or adjust chunk size. If multi-step reasoning fails, increase chunk overlap or implement parent-document retrieval.

10. **Deploy with monitoring.** Log every query, retrieved chunks, re-ranked scores, and generated answer. Track faithfulness drift over time. Set up alerts when average faithfulness drops below threshold.

## Concrete Examples

**Example 1: Building a RAG pipeline for compliance documents**

User: "I have a folder of FDA regulatory guidance PDFs. Build me a RAG pipeline that can answer questions about drug approval requirements with high accuracy."

Approach:
1. Load PDFs with `PyPDFLoader` or `pdfplumber`, extracting text per page with section heading detection.
2. Apply recursive character splitting: chunk_size=800, chunk_overlap=150, separators on double-newline, newline, period-space.
3. Embed chunks with `all-MiniLM-L6-v2` via `sentence-transformers`, store in FAISS.
4. On query, retrieve top-10 by cosine similarity.
5. Re-rank with `cross-encoder/ms-marco-MiniLM-L-6-v2`, select top-3.
6. Pass to LLM with prompt: "Answer the following question using ONLY the provided context. Cite the source document and section for each claim."

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.vectorstores import FAISS
from langchain_community.embeddings import HuggingFaceEmbeddings
from sentence_transformers import CrossEncoder
import numpy as np

# Step 1-3: Chunk and embed
splitter = RecursiveCharacterTextSplitter(
    chunk_size=800,
    chunk_overlap=150,
    separators=["\n\n", "\n", ". ", " "]
)
chunks = splitter.split_documents(documents)
embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")
vectorstore = FAISS.from_documents(chunks, embeddings)

# Step 4: First-stage retrieval
retriever = vectorstore.as_retriever(search_kwargs={"k": 10})
candidates = retriever.get_relevant_documents(query)

# Step 5: Cross-encoder re-ranking
cross_encoder = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")
pairs = [(query, doc.page_content) for doc in candidates]
scores = cross_encoder.predict(pairs)
top_indices = np.argsort(scores)[::-1][:3]
final_chunks = [candidates[i] for i in top_indices]

# Step 6-7: Augmented generation
context = "\n\n---\n\n".join(
    f"[Source: {c.metadata.get('source', 'unknown')}]\n{c.page_content}"
    for c in final_chunks
)
prompt = f"""Answer the question using ONLY the context below.
Cite sources. If the context is insufficient, say so.

Context:
{context}

Question: {query}
Answer:"""
```

**Example 2: Adding re-ranking to an existing basic RAG system**

User: "My RAG chatbot returns irrelevant chunks sometimes. How do I add re-ranking?"

Approach:
1. Keep existing embedding and vector store unchanged.
2. Increase first-stage retrieval from top-5 to top-10 to give the re-ranker more candidates.
3. Add cross-encoder scoring as a post-retrieval step before context assembly.

```python
# Existing: retriever returns top-10 candidates
candidates = existing_retriever.get_relevant_documents(query, k=10)

# Add: cross-encoder re-ranking
from sentence_transformers import CrossEncoder
reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")
scores = reranker.predict([(query, doc.page_content) for doc in candidates])

# Select top-3 by re-ranked score
import numpy as np
ranked = [candidates[i] for i in np.argsort(scores)[::-1][:3]]

# Continue with existing prompt construction using `ranked` instead of `candidates`
```

Expected improvement: faithfulness jumps from ~0.62 to ~0.80 based on empirical results, with negligible latency increase (cross-encoder processes only 10 short passages).

**Example 3: Evaluating a RAG pipeline with RAGAS**

User: "How do I measure if my RAG pipeline is actually faithful to the source documents?"

Approach:
1. Prepare a test set of 15-20 questions with known answers from the corpus.
2. Run each question through the pipeline, capturing retrieved contexts and generated answers.
3. Score with RAGAS.

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision
from datasets import Dataset

# Collect pipeline outputs
results = []
for q in test_questions:
    contexts = retrieve_and_rerank(q)  # your pipeline
    answer = generate_answer(q, contexts)  # your LLM call
    results.append({
        "question": q,
        "contexts": [c.page_content for c in contexts],
        "answer": answer,
        "ground_truth": known_answers[q]  # optional, for context_precision
    })

dataset = Dataset.from_list(results)
scores = evaluate(dataset, metrics=[faithfulness, answer_relevancy, context_precision])
print(scores)
# Target: faithfulness >= 0.80, answer_relevancy >= 0.75
```

## Best Practices

- **Do** retrieve more candidates than you need (top-10) and let the cross-encoder narrow to top-3. The bi-encoder is fast but imprecise; the cross-encoder is slow but accurate. This division of labor is the key architectural insight.
- **Do** preserve section headings and document metadata during chunking. Source attribution in the final answer depends on this metadata surviving the pipeline.
- **Do** use chunk overlap (100-200 chars or 20-30 tokens) to prevent information loss at chunk boundaries, especially for multi-sentence reasoning chains.
- **Do** evaluate with RAGAS faithfulness scores, not just end-user satisfaction. A confident-sounding wrong answer is worse than an uncertain correct one.
- **Avoid** using chunk sizes larger than 1000 characters or 256 tokens. Oversized chunks dilute the embedding signal and reduce retrieval precision.
- **Avoid** skipping the re-ranking stage for "simplicity." The empirical gap between basic RAG (0.62 faithfulness) and advanced RAG (0.80) is too large to ignore in any domain where accuracy matters.
- **Avoid** feeding all 10 retrieved candidates to the LLM. More context is not better — irrelevant passages introduce noise that increases hallucination rates.

## Error Handling

- **Empty retrieval results**: If the vector store returns zero or very few candidates, the query may be out-of-domain. Return a clear "no relevant information found" message rather than forcing the LLM to generate from nothing.
- **Cross-encoder scoring failures**: If the re-ranker throws errors on long passages, truncate chunk text to 512 tokens before scoring (the cross-encoder's max sequence length).
- **Low faithfulness despite good retrieval**: The LLM is ignoring the context. Strengthen the system prompt with explicit instructions like "Do not use prior knowledge. Only use the provided context." Consider lowering temperature to 0.1.
- **Multi-step reasoning failures**: When answers require synthesizing information across multiple document sections, single chunks won't suffice. Implement parent-document retrieval (store small chunks for retrieval but return the full parent section as context) or increase chunk overlap significantly.
- **Embedding model mismatches**: The query and document embeddings must come from the same model. If you swap the embedding model, you must re-embed the entire corpus.

## Limitations

- **Multi-hop reasoning**: The two-stage pipeline retrieves independent chunks. Questions requiring chains of reasoning across non-adjacent sections (e.g., "What is the exception to the rule described in Section 3 when condition from Section 7 applies?") remain a bottleneck. The paper explicitly notes this as an unsolved structural constraint.
- **Table and structured data**: Bi-encoder embeddings handle prose well but struggle with tabular data, lists, and structured formats. Tables may need special extraction and indexing.
- **Corpus size scaling**: Cross-encoder re-ranking is O(k) per query where k is the first-stage candidate count. For k=10 this is fast, but scaling to k=100+ adds noticeable latency. The approach works best when the bi-encoder can narrow to a small candidate set effectively.
- **Domain transfer**: The specific chunk sizes and top-k values validated in this study (CDC policy documents) may not transfer directly to other domains. Always re-evaluate with RAGAS on your specific corpus.
- **Embedding model ceiling**: `all-MiniLM-L6-v2` is a lightweight model. For highly specialized domains, a fine-tuned or larger embedding model may be needed to achieve adequate first-stage recall.

## Reference

Maharjan, A. & Yadav, U. (2026). *Chunking, Retrieval, and Re-ranking: An Empirical Evaluation of RAG Architectures for Policy Document Question Answering*. arXiv:2601.15457. Key takeaway: two-stage retrieval with cross-encoder re-ranking improves faithfulness from 0.62 to 0.80 over single-stage RAG, and the re-ranking stage matters more than the chunking strategy choice.