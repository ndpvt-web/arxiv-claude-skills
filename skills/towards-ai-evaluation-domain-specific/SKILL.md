---
name: "towards-ai-evaluation-domain-specific"
description: "Build and evaluate domain-specific RAG systems with iterative user-feedback refinement, source grounding, and structured evaluation pipelines. Use when: 'build a RAG system for [domain] documents', 'evaluate my RAG pipeline quality', 'add user feedback to my retrieval system', 'improve RAG answers with source grounding', 'set up domain-specific document retrieval', 'create an evaluation framework for my RAG app'."
---

# Domain-Specific RAG with Iterative Evaluation and Source Grounding

This skill enables Claude to design, build, and evaluate domain-specific retrieval-augmented generation (RAG) systems following the AgriHubi methodology: a four-stage pipeline (document ingestion, semantic retrieval, grounded generation, user feedback) refined through structured iteration cycles. The core insight from the paper is that incremental user-feedback-driven improvements outperform architectural overhauls, and that explicit source grounding combined with systematic evaluation across multiple dimensions (completeness, linguistic accuracy, latency, perceived reliability) produces measurably better RAG systems, especially for specialized or low-resource domains.

## When to Use

- When the user wants to build a RAG system over a specialized document corpus (agriculture, legal, medical, technical manuals, etc.)
- When the user needs an evaluation framework to measure RAG answer quality across multiple dimensions
- When the user asks to add user feedback collection and iterative refinement to an existing RAG pipeline
- When the user wants to implement source grounding (linking answers back to retrieved document chunks) to reduce hallucination
- When the user is working with non-English or low-resource language documents and needs retrieval strategies that handle them
- When the user asks to compare multiple LLM backends for a domain-specific RAG application, weighing quality vs. latency trade-offs
- When the user wants to build a Streamlit or web-based Q&A interface backed by domain documents with FAISS vector search

## Key Technique

The AgriHubi approach structures RAG development as an **iterative feedback loop** rather than a one-shot architecture. The pipeline has four stages: (1) document ingestion with PDF extraction, OCR fallback, and metadata-preserving chunking; (2) semantic retrieval via FAISS index over L2-normalized embeddings with cosine similarity scoring; (3) grounded generation where retrieved chunks are passed as explicit context to the LLM with template-based prompts; and (4) a user interaction layer that captures Likert-scale ratings and free-text feedback per response, stored in SQLite for analysis.

The critical differentiator is the **structured iteration methodology**. Across eight development cycles, each round of user evaluation (via structured user studies with identical question types) drives specific refinements: adjusting chunking logic, tuning similarity thresholds, switching model backends, increasing token limits, or improving retrieval strategies (e.g., moving from filename-based to full-chunk retrieval). The paper demonstrates that this produced a shift from 46% poor ratings to 38%, with top ratings jumping from 3% to 21%. This data-driven iteration is more effective than speculative architectural changes.

**Source grounding** is implemented by preserving document identity and chunk metadata through the entire pipeline. Each generated answer is paired with the specific document segments that informed it, enabling users to verify claims against source material. This addresses the hallucination problem that plagues domain-specific applications where factual accuracy is non-negotiable.

## Step-by-Step Workflow

1. **Ingest and preprocess domain documents.** Extract text from PDFs using a Python extraction library (PyPDF2, pdfplumber) with an OCR fallback (pytesseract) for scanned materials. Preserve document metadata (filename, page number, section headers) in each extracted text block.

2. **Chunk text with metadata preservation.** Segment extracted text into coherent chunks (typically 300-800 tokens) using overlap to avoid splitting mid-sentence. Attach source metadata (document name, page, chunk index) to each chunk as a dictionary. Store raw chunks alongside metadata in a structured format (JSON lines or SQLite).

3. **Generate embeddings and build a FAISS index.** Embed each chunk using a text embedding model (OpenAI `text-embedding-ada-002`, or a local alternative like `sentence-transformers` for privacy-sensitive domains). L2-normalize vectors. Build a FAISS `IndexFlatIP` (inner product on normalized vectors = cosine similarity) and persist the index to disk alongside a chunk-ID-to-metadata mapping.

4. **Implement the retrieval function.** Given a user query, embed it with the same model, search the FAISS index for top-k chunks (start with k=5), and return chunks with their similarity scores and source metadata. Apply a minimum similarity threshold (e.g., 0.7) to filter low-relevance results.

5. **Construct grounded prompts with retrieved context.** Build a prompt template that includes: (a) a system instruction specifying the domain and expected answer style, (b) the retrieved chunks clearly delimited with their source identifiers, and (c) the user query. Instruct the model to cite sources and to say "I don't have enough information" when retrieved context is insufficient.

6. **Integrate the LLM backend with configurable model selection.** Implement the generation call behind an abstraction that allows swapping models (e.g., different sizes or providers). Log the model used, token count, and response latency for every request. Set a maximum response token limit (start at 700 tokens, increase to 2000 if users report incomplete answers).

7. **Build the user interface with feedback capture.** Create a Streamlit (or equivalent) interface that displays: the generated answer, the source documents used (with expandable full-chunk views), and a feedback widget (5-point Likert scale + optional free-text comment). Store all interactions in SQLite with schema: `(id, timestamp, query, retrieved_chunks, model_response, model_name, latency_ms, rating, feedback_text)`.

8. **Run structured evaluation rounds.** Design a set of 15-25 representative domain questions covering different difficulty levels and topic areas. Have domain users answer them and rate responses. Compute: (a) rating distribution, (b) mean and median ratings, (c) average latency, (d) percentage of responses rated 1-2 (poor) vs. 4-5 (good). Use identical questions across rounds for comparison.

9. **Analyze feedback and apply targeted refinements.** After each evaluation round, categorize low-rated responses by failure mode: incomplete answers (increase chunk count or token limit), irrelevant retrieval (adjust chunking size or similarity threshold), terminology errors (check document coverage or model language capability), or slow responses (consider a smaller model variant). Apply one category of fix per iteration to isolate effects.

10. **Iterate and re-evaluate.** Repeat steps 8-9 across multiple cycles. Track metrics over time in a simple dashboard or spreadsheet. Document each iteration's changes and their measured impact. Expect 3-5 iterations to reach acceptable quality for a new domain.

## Concrete Examples

**Example 1: Building a legal-document RAG system**

```
User: I have 200 PDF legal contracts and I want to build a RAG system
so our team can ask questions about contract terms and obligations.

Approach:
1. Create a document ingestion script using pdfplumber for text extraction
   with pytesseract fallback for scanned contracts:

   # ingest.py
   import pdfplumber
   import json
   from pathlib import Path

   def extract_pdf(path):
       chunks = []
       with pdfplumber.open(path) as pdf:
           for i, page in enumerate(pdf.pages):
               text = page.extract_text() or ""
               if not text.strip():
                   text = ocr_fallback(path, i)
               chunks.append({
                   "text": text,
                   "source": path.name,
                   "page": i + 1
               })
       return chunks

2. Chunk with 500-token windows and 50-token overlap, preserving
   source metadata per chunk. Write chunks to chunks.jsonl.

3. Generate embeddings and build FAISS index:

   # embed.py
   import faiss
   import numpy as np
   from openai import OpenAI

   client = OpenAI()
   vectors = []
   for chunk in chunks:
       resp = client.embeddings.create(
           input=chunk["text"],
           model="text-embedding-ada-002"
       )
       vec = np.array(resp.data[0].embedding, dtype="float32")
       vec /= np.linalg.norm(vec)
       vectors.append(vec)

   matrix = np.stack(vectors)
   index = faiss.IndexFlatIP(matrix.shape[1])
   index.add(matrix)
   faiss.write_index(index, "contracts.faiss")

4. Build retrieval + grounded prompt:

   PROMPT_TEMPLATE = """You are a legal contract assistant.
   Answer based ONLY on the provided contract excerpts.
   Cite the source document and page for each claim.
   If the excerpts don't contain enough information, say so.

   Contract excerpts:
   {context}

   Question: {query}
   Answer:"""

5. Add Streamlit UI with feedback capture and SQLite logging.

Output: A working RAG app where legal team members ask questions,
get source-grounded answers with citations, and rate each response.
```

**Example 2: Adding evaluation framework to an existing RAG system**

```
User: I already have a RAG chatbot for our medical knowledge base
but I don't know if the answers are good. Help me evaluate it.

Approach:
1. Create an evaluation question set (20 questions spanning common
   queries, edge cases, and multi-document reasoning).

2. Build a feedback collection table in SQLite:

   CREATE TABLE evaluations (
       id INTEGER PRIMARY KEY AUTOINCREMENT,
       timestamp TEXT DEFAULT CURRENT_TIMESTAMP,
       question TEXT NOT NULL,
       retrieved_chunks TEXT,
       response TEXT NOT NULL,
       model_name TEXT,
       latency_ms INTEGER,
       rating INTEGER CHECK(rating BETWEEN 1 AND 5),
       feedback_text TEXT,
       evaluator TEXT
   );

3. Add a rating widget to the existing UI:

   import streamlit as st
   rating = st.radio("Rate this answer:", [1, 2, 3, 4, 5],
                     captions=["Poor", "Weak", "Okay", "Good", "Excellent"],
                     horizontal=True)
   feedback = st.text_area("What could be improved?")

4. After collecting 40+ rated responses, analyze:

   import sqlite3
   import pandas as pd

   conn = sqlite3.connect("chat_history.db")
   df = pd.read_sql("SELECT rating, latency_ms, model_name FROM evaluations", conn)

   print("Rating distribution:")
   print(df["rating"].value_counts().sort_index())
   print(f"Poor responses (1-2): {(df['rating'] <= 2).mean():.0%}")
   print(f"Good responses (4-5): {(df['rating'] >= 4).mean():.0%}")
   print(f"Mean latency: {df['latency_ms'].mean():.0f}ms")

5. Categorize failure modes from low-rated responses, then apply
   targeted fixes (e.g., increase chunk overlap for incomplete answers,
   lower similarity threshold for missed retrievals).

Output: A structured evaluation report showing rating distributions,
failure mode categories, and a prioritized list of refinements.
```

**Example 3: Comparing model backends for quality vs. latency**

```
User: Should I use a smaller or larger model for my RAG system?
Help me set up a comparison.

Approach:
1. Create a model abstraction that logs latency and model name:

   import time

   def generate(query, context, model_name="gpt-4o-mini"):
       start = time.perf_counter()
       response = client.chat.completions.create(
           model=model_name,
           messages=[
               {"role": "system", "content": SYSTEM_PROMPT},
               {"role": "user", "content": f"Context:\n{context}\n\nQ: {query}"}
           ],
           max_tokens=1500
       )
       latency = (time.perf_counter() - start) * 1000
       return {
           "text": response.choices[0].message.content,
           "model": model_name,
           "latency_ms": latency,
           "tokens": response.usage.total_tokens
       }

2. Run the same 20 evaluation questions against each model backend.

3. Collect user ratings for each model's responses (blind the
   evaluators to which model produced which answer).

4. Compare results:

   Model A (small):  Mean rating 3.1, Mean latency 820ms, Cost $0.02/query
   Model B (large):  Mean rating 4.0, Mean latency 3200ms, Cost $0.15/query

5. Decision framework: If >70% of responses are rated 4+ with the
   smaller model, use it. Otherwise, use the larger model but add
   streaming to mitigate perceived latency (users associate slow
   responses with lower reliability regardless of actual quality).

Output: A comparison table with ratings, latency, cost per query,
and a recommendation based on the quality-latency trade-off curve.
```

## Best Practices

- **Do:** Preserve source metadata (document name, page, section) through the entire pipeline from ingestion to display. Source grounding is the primary mechanism for reducing hallucination in domain-specific systems.
- **Do:** Use identical evaluation question sets across iteration rounds so metric changes reflect actual improvements, not question difficulty variance.
- **Do:** Implement streaming responses when using larger models. The AgriHubi study found that users perceive slow responses as less reliable even when the content quality is higher.
- **Do:** Start with a smaller model and scale up only when evaluation data shows the smaller model's quality is insufficient. The quality gap between model sizes may not justify the latency and cost increase for your domain.
- **Do:** Store every interaction (query, retrieved chunks, response, rating, latency) in a persistent store. This data is essential for diagnosing failure patterns and measuring iteration impact.
- **Avoid:** Making multiple changes between evaluation rounds. Isolate one variable per iteration (chunking strategy, model backend, similarity threshold, token limit) to understand what actually improved scores.
- **Avoid:** Relying solely on automated metrics (BLEU, ROUGE, embedding similarity) for domain-specific RAG. The AgriHubi study found that human evaluation catches terminology errors, contextual inaccuracies, and completeness gaps that automated metrics miss entirely.
- **Avoid:** Treating low-resource or non-English language support as a simple translation layer. Domain-specific terminology requires models with genuine language capability (like PORO for Finnish), not general models with a translation prompt.

## Error Handling

- **OCR extraction failures:** When PDF text extraction returns empty strings, fall back to OCR with pytesseract. Log documents that fail both methods for manual review. Do not silently skip documents.
- **Empty retrieval results:** If no chunks pass the similarity threshold, return a clear "I don't have enough information in the available documents to answer this question" rather than generating from the model's parametric knowledge. This is critical for domain accuracy.
- **Model backend timeouts:** Set a per-request timeout (30-60 seconds). On timeout, retry once with the same model, then fall back to a smaller model variant. Log the timeout for latency analysis.
- **Embedding dimension mismatches:** When switching embedding models, rebuild the entire FAISS index. Never mix embeddings from different models in the same index.
- **Low evaluation scores across the board:** If >60% of responses are rated 1-2 after initial deployment, the problem is likely document coverage (missing relevant documents) or chunking granularity (chunks too large to retrieve precise information), not the model. Investigate retrieval quality before changing the LLM.
- **Feedback spam or low-quality ratings:** Filter evaluation data by requiring a minimum text feedback length for ratings of 1-2, ensuring low scores come with actionable explanations.

## Limitations

- This approach requires **domain users** for evaluation. Automated metrics alone are insufficient for domain-specific RAG; you need people who understand the subject matter to rate answer quality meaningfully.
- The iterative refinement methodology requires **multiple evaluation rounds** with real users, which demands coordination and time commitment from domain experts.
- Source grounding only works when the answer is **actually derivable** from the indexed documents. If the document corpus has gaps, the system will correctly report insufficient information, but users may find this frustrating.
- The evaluation framework assumes **relatively small sample sizes** (50-100 rated responses per round). Statistical significance for fine-grained comparisons (e.g., 3% improvement in a sub-category) requires larger samples.
- For languages with limited embedding model support, you may need to use multilingual embeddings (e.g., `multilingual-e5-large`) which can have lower retrieval precision than language-specific models.
- The paper's findings are validated on agricultural text in Finnish. Transfer to other domains requires re-running the iterative evaluation process; do not assume the same chunking parameters or similarity thresholds will generalize.

## Reference

**Paper:** [Towards AI Evaluation in Domain-Specific RAG Systems: The AgriHubi Case Study](https://arxiv.org/abs/2602.02208v1) (Hasan et al., 2026). Look for: the four-stage pipeline architecture, the eight-iteration refinement methodology, the structured evaluation framework with Likert-scale feedback, the rating distribution shift between evaluation rounds, and the quality-latency trade-off analysis across PORO model variants.