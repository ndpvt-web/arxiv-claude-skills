---
name: "what-should-cite-rag"
description: "Build multi-level RAG pipelines for academic citation prediction and literature discovery. Use when the user asks to 'find relevant papers to cite', 'build a citation recommendation system', 'create a literature retrieval pipeline', 'predict references for a paper', 'set up academic search with RAG', or 'recommend citations for my manuscript'."
---

# CiteRAG: Multi-Level Hybrid RAG for Academic Citation Prediction

This skill enables Claude to build retrieval-augmented generation pipelines that predict which papers a manuscript should cite. Based on the CiteRAG framework, the approach combines three-level corpus indexing (title, abstract, full-text), hybrid sparse+dense retrieval with Reciprocal Rank Fusion, contrastive-learned embeddings fine-tuned on citation relationships, and LLM-based generation that produces ranked citation lists with reasoning. The technique applies to any system that must surface relevant academic literature given a query paper's metadata or in-context citation markers.

## When to Use

- When building a citation recommendation engine that suggests references for a draft paper given its title and abstract
- When constructing a multi-granularity academic search system that retrieves papers at title, abstract, and full-text levels
- When fine-tuning embedding models with contrastive learning on citation pairs to improve scholarly retrieval
- When implementing position-specific citation prediction (filling in `[ref]` markers within manuscript text)
- When designing a RAG pipeline where the retriever feeds candidate papers to an LLM that reasons about citation relevance
- When evaluating retrieval and generation quality on academic benchmarks using Recall@K, NDCG@K, MRR, or position-aware accuracy

## Key Technique

**Multi-level hybrid retrieval with citation-aware embeddings.** CiteRAG indexes a corpus at three granularity levels: title-only, abstract, and full-text. Each level gets its own FAISS vector index. At query time, the user's paper (title + abstract) is embedded and searched against all three indexes independently, retrieving the top-K candidates from each. Results are fused using Reciprocal Rank Fusion (RRF): for each candidate across all levels, scores accumulate as `weight / (k + rank + 1)` where `k` is a constant (typically 60). This fusion surface captures both coarse topical overlap (title-level) and fine-grained methodological similarity (full-text-level) in a single ranked list.

**Contrastive learning on citation triplets.** The embedding model (e.g., Qwen3-Embedding or BGE) is fine-tuned using InfoNCE loss on triplets of (query_abstract, positive_cited_paper, hard_negatives). Hard negatives are semantically similar but non-cited papers -- these force the model to learn subtle citation-worthiness signals beyond topic similarity. Training uses a temperature of 0.01, 8 hard negatives per sample, and a max sequence length of 4608 tokens to accommodate full abstracts.

**Structured generation with anti-hallucination reasoning.** Retrieved candidates are formatted into a numbered list and passed to an LLM alongside the query paper. The prompt requires the model to produce exactly N citation titles in JSON format with a `reasoning` field explaining each prediction. Requiring explicit reasoning reduces hallucinated (non-existent) paper titles. The two task variants -- list-specific (predict all references) and position-specific (predict the reference at each `[ref]` marker) -- use different prompt structures but the same retrieval backbone.

## Step-by-Step Workflow

1. **Structure the corpus into three levels.** For each paper in your collection, create three index entries: (a) title only, (b) title + abstract, (c) title + abstract + full-text or key sections. Store metadata (paper_id, title, abstract) in a JSON lookup keyed by document ID.

2. **Build FAISS indexes for each level.** Embed all entries at each level using your base embedding model. L2-normalize all vectors for cosine similarity. Create three separate FAISS `IndexFlatIP` (inner product) indexes. Save indexes and ID mappings to disk.

3. **Prepare contrastive training data.** For each paper in your training set, form triplets: the query is the paper's abstract, the positive is the abstract of a paper it actually cites, and the negatives are 6-8 abstracts of topically similar but non-cited papers (mined via BM25 or approximate nearest neighbor search on the base embeddings).

4. **Fine-tune the embedding model with InfoNCE loss.** Train for 3 epochs with learning rate 5e-5, temperature 0.01, batch size scaled to your GPU memory, and Flash Attention 2 enabled. Save checkpoints every 800 steps. After training, re-embed the entire corpus at all three levels and rebuild the FAISS indexes.

5. **Implement the hybrid retrieval endpoint.** Build a FastAPI service that: (a) concatenates the query paper's title and abstract, (b) embeds the query via the fine-tuned model, (c) searches all three FAISS indexes for top-200 candidates each, (d) fuses results with RRF, and (e) returns the top-K papers with scores and metadata.

6. **Design the generation prompt.** For list-specific prediction, format the top-R retrieved papers as a numbered list (`1. Title: ... Abstract: ...`) and instruct the LLM to output a JSON object with a `titles` array and a `reasoning` string. Require exactly N predictions with no duplicates.

7. **For position-specific prediction, extract citation contexts.** Parse the manuscript for `[ref]` markers. For each marker, extract a 200-character window around it. Send each context window plus the paper's metadata to the retriever, then prompt the LLM to predict the reference for each specific marker with its `ref_index`.

8. **Parse and validate LLM output.** Strip markdown code fences from the response, parse JSON, validate the `titles` array length, and deduplicate. Implement retry logic (up to 3 attempts) for malformed responses.

9. **Evaluate with standard metrics.** Compute Recall@K (K=10, 20, 40), NDCG@K, and Hits@K for list-specific prediction. Use position-aware accuracy (pacc@K) for position-specific prediction. Average across all test instances and report aggregate statistics.

10. **Deploy as a two-service architecture.** Run the retriever as one FastAPI service and the generator (vLLM or API-backed) as another. The retriever handles `/search` and `/batch_search` endpoints; the generator handles prompt completion. This separation lets you scale retrieval and generation independently.

## Concrete Examples

**Example 1: Building a citation recommender API**

```
User: I want to build an API that takes a paper's title and abstract
and returns the 20 most likely references it should cite.

Approach:
1. Index your paper corpus at three levels (title, abstract, full-text)
   using FAISS with a fine-tuned embedding model.
2. Create a FastAPI endpoint that accepts {title, abstract, top_k}:

   @app.post("/recommend")
   async def recommend_citations(request: CitationRequest):
       query_text = f"{request.title} {request.abstract}"
       embedding = await get_embedding(query_text)
       embedding = embedding / np.linalg.norm(embedding)

       # Search all three index levels
       candidates = {}
       for level_name, index in indexes.items():
           scores, ids = index.search(embedding.reshape(1, -1), 200)
           for rank, (score, doc_id) in enumerate(zip(scores[0], ids[0])):
               if doc_id not in candidates:
                   candidates[doc_id] = 0.0
               candidates[doc_id] += 1.0 / (60 + rank + 1)  # RRF

       # Sort by fused score, take top-R for generation context
       top_candidates = sorted(candidates.items(), key=lambda x: -x[1])[:50]
       context = format_retrieved_papers(top_candidates)

       # Generate citation predictions with reasoning
       prompt = build_citation_prompt(request.title, request.abstract,
                                       context, top_k=20)
       response = await call_llm(prompt)
       return parse_citation_response(response)

3. The LLM prompt includes the retrieved papers and asks for JSON output:
   {"titles": ["Paper A", "Paper B", ...], "reasoning": "Paper A is
   relevant because it introduces the baseline method..."}

Output: A JSON response with 20 predicted citation titles, each grounded
in retrieved candidates, with explanation of why each is relevant.
```

**Example 2: Fine-tuning embeddings for citation retrieval**

```
User: My retrieval results are too generic -- papers on the same topic
but not actual citation-worthy references. How do I improve this?

Approach:
1. Build contrastive training data from your citation graph:

   {"query": "Abstract of paper A...",
    "response": "Abstract of paper A's cited reference...",
    "rejected_response": [
      "Abstract of topically similar but non-cited paper 1...",
      "Abstract of topically similar but non-cited paper 2...",
      ... (6-8 hard negatives)
    ]}

2. Mine hard negatives using BM25 or the current embedding model:
   - For each paper, retrieve top-50 similar papers
   - Filter out actual citations to keep only non-cited papers
   - Select the top 8 most similar non-cited papers as hard negatives

3. Fine-tune with InfoNCE contrastive loss:

   torchrun --nproc_per_node=4 train_embedding.py \
     --model_name Qwen/Qwen3-Embedding-0.6B \
     --loss_type infonce \
     --temperature 0.01 \
     --hard_negatives 8 \
     --learning_rate 5e-5 \
     --epochs 3 \
     --max_seq_length 4608 \
     --save_steps 800

4. After training, re-embed the entire corpus and rebuild FAISS indexes.

Output: Retrieval now distinguishes between "same topic" and "should cite"
-- Recall@20 typically improves 10-20% over the base embedding model.
```

**Example 3: Position-specific citation filling**

```
User: I have a manuscript draft with [ref] placeholders. I want to
automatically suggest which paper each [ref] should point to.

Approach:
1. Parse the manuscript to find all [ref] markers with their positions:

   refs = []
   for i, match in enumerate(re.finditer(r'\[ref\]', manuscript_text)):
       start = max(0, match.start() - 200)
       end = min(len(manuscript_text), match.end() + 200)
       context = manuscript_text[start:end]
       refs.append({"ref_index": i, "context": context})

2. For each [ref], send the paper's title + abstract + the local context
   window to the retriever to get candidate papers.

3. Prompt the LLM with the context and candidates:
   "For each [ref] marker below, predict the most likely cited paper.
    Return JSON: [{"ref_index": 0, "titles": [...], "reasoning": "..."}]"

4. Evaluate with position-aware accuracy (pacc@K):
   - For each [ref], check if the ground-truth paper appears in the
     top-K predictions for that specific position.

Output: Each [ref] marker gets a ranked list of candidate citations
with reasoning tied to the surrounding sentence context.
```

## Best Practices

- **Do:** Use hard negatives mined from topically similar but non-cited papers. Random negatives are too easy and produce weak embeddings that cannot distinguish "relevant to topic" from "should cite."
- **Do:** Index at multiple granularity levels. Title-level catches broad topical matches; full-text-level captures methodological and dataset-specific connections that abstracts miss.
- **Do:** Require the LLM to output reasoning alongside predictions. This reduces hallucinated paper titles by forcing the model to ground each suggestion in retrieved evidence.
- **Do:** Use Reciprocal Rank Fusion rather than simple score averaging to combine results from different index levels. RRF is robust to score distribution differences across indexes.
- **Avoid:** Relying solely on dense retrieval. BM25 captures exact term matches (method names, dataset names, acronyms) that embedding models often miss. Run BM25 as an additional retrieval path and include it in the RRF fusion.
- **Avoid:** Setting the generation count too high without sufficient retrieval context. If you ask for 40 citations but only retrieve 20 candidates, the LLM will hallucinate the rest. Keep `top_k_generate <= top_k_retrieve`.

## Error Handling

- **Malformed LLM JSON output:** Strip markdown code fences (` ```json ... ``` `), attempt JSON parsing, and retry up to 3 times with the same prompt. If all retries fail, fall back to returning just the retriever's ranked list without LLM reranking.
- **Retriever timeouts:** Implement exponential backoff (initial delay 1s, up to 5 retries) for retrieval API calls. Process queries in parallel batches to amortize latency.
- **Empty retrieval results:** If a FAISS index returns no results for a query (embedding dimension mismatch, corrupt index), log the error and exclude that level from RRF fusion rather than failing the entire request.
- **Duplicate predictions:** Deduplicate the LLM's title list before evaluation. Normalize titles (lowercase, strip punctuation) before comparing against ground truth to handle minor formatting differences.
- **Out-of-corpus predictions:** The LLM may generate titles not present in the corpus. Flag these as potential hallucinations in the output. Consider constraining generation to only titles that appeared in the retrieval results.

## Limitations

- **Corpus coverage:** The system can only retrieve papers that exist in the indexed corpus. Newly published papers or those from uncovered subfields will be missed entirely. Regular corpus updates are essential.
- **Citation context dependency:** Position-specific prediction requires well-formatted manuscripts with explicit `[ref]` markers. It does not work on finalized PDFs or papers where citation intent is implicit.
- **Embedding model domain transfer:** Embeddings fine-tuned on one domain (e.g., CS/AI) may not transfer well to other fields (e.g., biology, law). Retraining with domain-specific citation data is necessary.
- **Hallucination in generation:** Despite reasoning requirements, LLMs can still generate plausible-sounding but non-existent paper titles, especially when retrieval context is thin. Always cross-check generated titles against the corpus.
- **Computational cost:** Maintaining three FAISS indexes and running cross-encoder reranking scales linearly with corpus size. For corpora beyond ~1M papers, consider approximate nearest neighbor indexes (HNSW) and staged reranking on only the top candidates.

## Reference

**Paper:** "What Should I Cite? A RAG Benchmark for Academic Citation Prediction" (arXiv:2601.14949v2) by Zheng et al., 2026. Look for: the three-level corpus construction pipeline (Section 3), InfoNCE contrastive training setup (Section 4.2), RRF fusion formula (Section 4.1), and the position-specific prompt design (Section 5.2). Code: https://github.com/LQgdwind/CiteRAG