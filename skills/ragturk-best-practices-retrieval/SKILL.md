---
name: "ragturk-best-practices-retrieval"
description: "Design and optimize RAG pipelines for Turkish and other morphologically rich languages (Turkish, Finnish, Hungarian, Korean, etc.) using evidence-based stage configurations. Trigger phrases: 'build a Turkish RAG pipeline', 'optimize RAG for agglutinative languages', 'RAG reranking for Turkish', 'morphology-aware retrieval', 'cross-encoder reranking pipeline', 'HyDE for non-English RAG'."
---

# RAGTurk: Best Practices for Retrieval-Augmented Generation in Morphologically Rich Languages

This skill enables Claude to design, configure, and optimize RAG pipelines that work correctly with Turkish and other morphologically rich, agglutinative languages. Based on the RAGTurk benchmark (EACL 2026), it encodes specific findings about which pipeline stages matter most, which combinations degrade performance due to morphological distortion, and how to achieve Pareto-optimal cost-accuracy tradeoffs. The core insight: retrieval and reranking quality dominate overall RAG accuracy, and stacking too many generative modules actively harms performance in agglutinative languages.

## When to Use

- When the user asks to build or improve a RAG system that handles Turkish, Finnish, Hungarian, Korean, Japanese, or other agglutinative/morphologically rich languages
- When configuring reranking strategies for multilingual retrieval pipelines
- When the user wants to choose between HyDE, cross-encoder reranking, and other RAG enhancements
- When debugging RAG quality issues where generative refinement steps are producing worse answers than simpler configurations
- When the user needs a cost-effective RAG setup and wants to avoid over-engineering the pipeline
- When adapting an English-centric RAG architecture to support non-English, morphologically complex languages

## Key Technique

The RAGTurk paper benchmarks seven stages of a RAG pipeline end-to-end without task-specific fine-tuning: (1) Query Transformation, (2) Dense Retrieval, (3) Reranking, (4) Context Augmentation, (5) Answer Fusion, (6) Answer Refinement, and (7) Post-processing. The critical finding is that **retrieval and reranking are the dominant quality factors**, not generative post-processing. HyDE (Hypothetical Document Embeddings) achieves the highest accuracy at 85%, but a Pareto-optimal configuration using Cross-encoder Reranking + Context Augmentation reaches 84.6% at substantially lower computational cost.

The most counterintuitive result is that **over-stacking generative modules degrades performance** in Turkish. Each generative step (query rewriting, answer refinement, fusion) risks distorting morphological cues -- suffixes, agglutinated forms, case markers -- that are critical for correct retrieval and answer extraction. In English, these extra steps often help because English morphology is simple. In Turkish, a single word like "evlerinizdekilerden" (from those in your houses) carries meaning in its suffix chain that generative paraphrasing can destroy, leading to retrieval mismatches and incorrect answers.

The practical recommendation: start with strong retrieval + cross-encoder reranking + context augmentation. Only add generative modules (HyDE, answer refinement) if measured accuracy improves on your specific data. Simple query clarification (fixing typos, expanding abbreviations) paired with robust reranking consistently outperforms complex multi-stage generative pipelines for morphologically rich languages.

## Step-by-Step Workflow

1. **Assess language morphology**: Determine whether the target language is agglutinative or morphologically rich (Turkish, Finnish, Hungarian, Korean, Swahili, etc.). If yes, apply the conservative pipeline strategy below. If the language is morphologically simple (English, Mandarin), standard RAG practices apply.

2. **Configure chunking with header-aware splitting**: Use chunk size of ~1000 characters with ~200 character overlap. Use a header-aware split strategy that preserves document section boundaries rather than naive character splitting. This prevents breaking agglutinated words at chunk boundaries.

   ```python
   from langchain.text_splitter import RecursiveCharacterTextSplitter

   splitter = RecursiveCharacterTextSplitter(
       chunk_size=1000,
       chunk_overlap=200,
       separators=["\n## ", "\n### ", "\n\n", "\n", ". ", " "]
   )
   ```

3. **Select a multilingual embedding model**: Use embeddings trained on multilingual data that handle subword tokenization for agglutinative forms. Prefer models like `intfloat/multilingual-e5-large`, `BAAI/bge-m3`, or `sentence-transformers/paraphrase-multilingual-mpnet-base-v2`. Avoid English-only embeddings.

   ```python
   from sentence_transformers import SentenceTransformer
   model = SentenceTransformer("intfloat/multilingual-e5-large")
   ```

4. **Implement cross-encoder reranking**: After initial dense retrieval (top-k=20), apply a cross-encoder reranker to re-score and select the top-k=5 passages. Use a multilingual cross-encoder such as `cross-encoder/ms-marco-MiniLM-L-12-v2` or `unicamp-dl/mMiniLM-L-6-v2-mmarco-v2`. This single stage provides the highest ROI.

   ```python
   from sentence_transformers import CrossEncoder
   reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-12-v2")

   # Score each (query, passage) pair
   pairs = [(query, doc.page_content) for doc in retrieved_docs]
   scores = reranker.predict(pairs)
   reranked = [doc for _, doc in sorted(zip(scores, retrieved_docs), reverse=True)][:5]
   ```

5. **Apply context augmentation (not generative refinement)**: Enrich the retrieved context by prepending document metadata (title, section heading, source) to each chunk before feeding to the LLM. Do NOT use an LLM to rewrite or summarize the context -- this risks morphological distortion.

   ```python
   def augment_context(chunks):
       augmented = []
       for chunk in chunks:
           prefix = f"Kaynak: {chunk.metadata.get('title', '')}\n"
           prefix += f"Bolum: {chunk.metadata.get('section', '')}\n\n"
           augmented.append(prefix + chunk.page_content)
       return "\n---\n".join(augmented)
   ```

6. **Use minimal query transformation**: Limit query transformation to simple clarification -- fix typos, expand abbreviations, normalize unicode. Do NOT use aggressive query rewriting, decomposition, or multi-query generation unless you measure improvement. For Turkish specifically, normalize the dotted/dotless I distinction and handle common diacritic issues.

   ```python
   def normalize_turkish_query(query: str) -> str:
       # Normalize common Turkish character issues
       replacements = {"i̇": "i", "İ": "İ"}  # Preserve Turkish I/ı distinction
       for old, new in replacements.items():
           query = query.replace(old, new)
       return query.strip()
   ```

7. **If maximum accuracy is required, evaluate HyDE**: Generate a hypothetical answer document using the LLM, then use it as the retrieval query. This achieves ~85% accuracy but costs an extra LLM call per query. Only use when the accuracy gain justifies the latency and cost.

   ```python
   def hyde_query(llm, original_query: str, language: str = "Turkish") -> str:
       prompt = f"Write a short {language} paragraph that would answer this question: {original_query}"
       hypothetical_doc = llm.invoke(prompt)
       return hypothetical_doc  # Use this as the embedding query
   ```

8. **Avoid stacking answer refinement on top of other generative stages**: If you use HyDE for query transformation, do NOT also apply generative answer refinement or fusion. Each additional generative stage compounds morphological distortion risk. Pick one generative enhancement or none.

9. **Evaluate with morphology-aware metrics**: When measuring pipeline quality, check for suffix preservation in extracted answers. A correct answer for Turkish might be "Ankara'da" (in Ankara) but a morphologically damaged pipeline might return "Ankara" (losing the locative suffix, changing the meaning). Track exact match AND semantic match separately.

10. **Load-test the Pareto configuration first**: Start with Cross-encoder Reranking + Context Augmentation as your baseline. This configuration achieves 84.6% accuracy at minimal cost. Only add HyDE or other generative modules if this baseline falls short on your specific evaluation set.

## Concrete Examples

**Example 1: Building a Turkish Q&A RAG system**

User: "I need to build a RAG pipeline for answering questions about Turkish legal documents."

Approach:
1. Chunk legal documents using header-aware splitting (1000 chars, 200 overlap) preserving article/section boundaries
2. Embed with `intfloat/multilingual-e5-large` into a vector store (Qdrant, Pinecone, or FAISS)
3. Retrieve top-20 candidates with dense search
4. Rerank with a cross-encoder to select top-5
5. Augment context with document title, article number, and section heading
6. Pass augmented context to LLM with a Turkish-language system prompt

Output pipeline configuration:
```python
pipeline_config = {
    "chunking": {"size": 1000, "overlap": 200, "strategy": "header_aware"},
    "embedding": {"model": "intfloat/multilingual-e5-large"},
    "retrieval": {"top_k": 20, "method": "dense"},
    "reranking": {"model": "cross-encoder/ms-marco-MiniLM-L-12-v2", "top_k": 5},
    "context": {"augmentation": True, "generative_refinement": False},
    "generation": {"answer_fusion": False, "answer_refinement": False},
}
# Expected accuracy: ~84-85% on factual questions
# Cost: 1 embedding call + 1 cross-encoder batch + 1 LLM generation call per query
```

**Example 2: Debugging degraded Turkish RAG quality after adding refinement steps**

User: "My Turkish RAG pipeline got worse after I added query decomposition and answer refinement. What's going on?"

Approach:
1. Identify the generative modules in the pipeline: query decomposition + answer refinement = 2 extra generative stages
2. Explain morphological distortion: each LLM rewrite risks stripping Turkish suffixes, changing agglutinated forms, or normalizing case markers
3. Recommend removing answer refinement first, measuring impact
4. If still degraded, replace query decomposition with simple query clarification
5. Verify the reranking stage is using a cross-encoder (not just bi-encoder similarity)

Diagnosis:
```
Current pipeline (degraded):
  Query -> Decomposition (LLM) -> Retrieval -> Reranking -> Answer Refinement (LLM) -> Output
  Problem: Two generative stages compound morphological distortion

Recommended pipeline:
  Query -> Simple Clarification (rule-based) -> Retrieval -> Cross-encoder Reranking -> Context Augmentation -> Output
  Result: Fewer generative stages preserve Turkish morphological cues
```

**Example 3: Choosing between HyDE and cross-encoder reranking for a multilingual system**

User: "Should I use HyDE or cross-encoder reranking for my Turkish + English RAG system?"

Approach:
1. For English-only queries: either approach works well; HyDE adds latency
2. For Turkish queries: cross-encoder reranking is the safer default (84.6% vs 85% for HyDE, at much lower cost)
3. For mixed-language deployment: use cross-encoder reranking as the universal stage, optionally add HyDE only for queries where initial retrieval recall is poor

Recommendation:
```
# Cost-accuracy tradeoff (from RAGTurk benchmarks):
#
# Configuration                          | Accuracy | Cost per query
# ---------------------------------------|----------|---------------
# Baseline (dense retrieval only)        | 78.7%    | $
# Cross-encoder reranking + augmentation | 84.6%    | $$
# HyDE                                   | 85.0%    | $$$
#
# Decision: Start with cross-encoder + augmentation.
# Add HyDE only if the 0.4% accuracy gap matters for your use case.
```

## Best Practices

**Do:**
- Prioritize cross-encoder reranking as the single highest-impact stage to add to any morphologically rich language RAG pipeline
- Use header-aware chunking (1000 chars / 200 overlap) to avoid splitting agglutinated words
- Preserve original morphological forms in retrieved passages -- pass them to the LLM unmodified
- Test pipeline changes with morphology-sensitive evaluation (check suffix preservation, case marker accuracy)

**Avoid:**
- Stacking multiple generative modules (HyDE + query rewriting + answer refinement) -- each one risks distorting morphological cues
- Using English-only embedding models for Turkish or other agglutinative languages
- Applying aggressive query rewriting that paraphrases agglutinated forms into decomposed phrases
- Assuming English RAG best practices transfer directly -- morphologically rich languages have fundamentally different failure modes

## Error Handling

- **Retrieval returns irrelevant documents**: Check that the embedding model handles Turkish subword tokenization. Switch to a multilingual model if using an English-only one. Verify that Turkish-specific characters (ş, ç, ğ, ı, ö, ü, İ) are preserved in the indexing pipeline.
- **Answers lose grammatical suffixes**: A generative stage is stripping morphology. Remove answer refinement or fusion steps. Compare answers with and without each generative module.
- **Cross-encoder reranking is slow**: Batch the (query, document) pairs. Reduce initial retrieval top-k from 20 to 10. Use a smaller cross-encoder model like MiniLM-L-6 instead of L-12.
- **HyDE generates hypothetical documents in the wrong language**: Explicitly specify the target language in the HyDE prompt. Use few-shot examples in the target language.
- **Inconsistent dotted/dotless I handling**: Turkish has four I variants (I, İ, ı, i). Normalize before embedding but preserve original forms in displayed context. Use locale-aware case folding (`str.lower()` in Python does NOT handle Turkish I correctly -- use the `icu` library or explicit mapping).

## Limitations

- The RAGTurk benchmarks use Wikipedia and CulturaX data. Performance on domain-specific text (medical, legal, technical) may differ and should be validated separately.
- The 84.6% and 85% accuracy figures are specific to the RAGTurk evaluation set. Your domain will have different baselines.
- Cross-encoder reranking adds latency proportional to the number of retrieved documents. For real-time applications with strict latency budgets (<100ms), bi-encoder reranking may be necessary despite lower accuracy.
- The findings apply most strongly to agglutinative languages. For isolating languages (Mandarin, Vietnamese) or fusional languages (Russian, German), the morphological distortion effects may be less pronounced.
- No task-specific fine-tuning was used in the benchmarks. Fine-tuned retrievers or rerankers for your specific language and domain may shift the optimal configuration.

## Reference

**Paper**: [RAGTurk: Best Practices for Retrieval Augmented Generation in Turkish](https://arxiv.org/abs/2602.03652v1) (EACL 2026 SIGTURK)
**Dataset**: [metunlp/ragturk on HuggingFace](https://huggingface.co/datasets/metunlp/ragturk)
**Code**: [github.com/metunlp/ragturk](https://github.com/metunlp/ragturk)
**Key takeaway**: Cross-encoder reranking + context augmentation is the Pareto-optimal RAG configuration for Turkish; avoid stacking generative modules that distort morphological cues.