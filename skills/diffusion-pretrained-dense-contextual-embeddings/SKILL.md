---
name: "diffusion-pretrained-dense-contextual-embeddings"
description: "Build production retrieval systems using pplx-embed, diffusion-pretrained dense and contextualized embedding models with INT8 quantization, late chunking for long documents, and multi-stage contrastive training. Use when: 'build a semantic search pipeline', 'set up document retrieval with contextual embeddings', 'implement late chunking for long documents', 'create a multilingual search index', 'optimize embedding storage with quantization', 'add contextualized passage retrieval to RAG'."
---

This skill enables Claude to design and implement high-quality retrieval systems using the pplx-embed architecture: a family of embedding models built on diffusion-pretrained bidirectional transformers with multi-stage contrastive learning, INT8 quantization-aware training, and a late chunking strategy that preserves global document context across passage boundaries. The core insight is that diffusion-based pretraining (masking tokens via an absorbing-state process and reconstructing them with bidirectional attention) produces a stronger backbone for embeddings than causal language models, and that a four-stage contrastive pipeline (pair, contextual, triplet, SLERP merge) yields state-of-the-art dense retrieval with efficient INT8 or binary representations.

## When to Use

- When building a semantic search or document retrieval pipeline and the user needs to choose an embedding model, chunking strategy, or quantization format
- When the user asks to implement **late chunking** to preserve cross-chunk context in long documents (contracts, legal filings, technical manuals)
- When designing a **RAG system** that needs passage-level embeddings enriched with document-level context (pplx-embed-context-v1 pattern)
- When the user wants to **reduce embedding storage** via INT8 or binary quantization without retraining
- When building **multilingual retrieval** across 60+ languages and the user needs a single model family
- When evaluating or benchmarking embedding models on MTEB, MIRACL, BERGEN, ConTEB, or ToolRet
- When implementing **tool retrieval** (selecting the right API/function from a large registry based on a natural language query)

## Key Technique

**Diffusion pretraining as an embedding backbone.** Standard embedding models start from causal (left-to-right) language models and bolt on bidirectional attention at fine-tuning time. pplx-embed instead converts a Qwen3 base model into a true bidirectional encoder via continued pretraining with a diffusion objective: at each step, tokens are independently masked with probability t (continuous time, t in [0,1]) and the model learns to reconstruct them using full bidirectional self-attention. This is trained for 60K steps on ~250B multilingual tokens with sequence length 4,096. The result is a backbone that natively captures bidirectional context, which yields ~1% average improvement on retrieval benchmarks compared to causal-only pretraining.

**Multi-stage contrastive learning.** Rather than a single contrastive fine-tuning pass, pplx-embed uses four stages: (1) **Pair training** with InfoNCE loss, in-batch negatives, and false-negative masking; (2) **Contextual training** that adds a dual-objective loss combining local chunk-level and global document-level contrastive signals (this is what powers the context-v1 variant); (3) **Triplet training** with hard negatives for final ranking quality; and (4) **SLERP model merging** that spherically interpolates the contextual and triplet checkpoints to combine their strengths. The final embeddings are mean-pooled and quantized to INT8 via a tanh-based formula with straight-through gradient estimation during training.

**Late chunking for long documents.** Instead of independently embedding each chunk, the model encodes the full document (up to 16 chunks of 256 tokens = 4,096 tokens) with bidirectional attention, then pools each chunk's token representations separately. This means each chunk embedding is informed by the entire document's context, solving the classic problem of context loss at chunk boundaries. The context-v1 variant adds a global document-level embedding objective on top of this, setting records on the ConTEB benchmark.

## Step-by-Step Workflow

1. **Assess retrieval requirements.** Determine: corpus size (thousands vs. millions of documents), average document length, number of languages, whether passages need document-level context, latency budget, and storage constraints. This dictates model size (0.6B vs. 4B) and quantization (float32 vs. INT8 vs. binary).

2. **Choose the model variant.** Use `pplx-embed-v1` (standard dense retrieval) when passages are self-contained or short. Use `pplx-embed-context-v1` when passages come from long documents and retrieval quality depends on surrounding context (legal discovery, technical documentation, book search). The context variant adds ~5% on contextual benchmarks but requires encoding full documents rather than isolated passages.

3. **Implement the chunking strategy.** Split documents into 256-token chunks (the model's native chunk size). For the context variant, group chunks into batches of up to 16 per document (4,096 tokens). If documents exceed 4,096 tokens, use a sliding window of 16 chunks with overlap. Maintain a mapping from chunk IDs back to source documents and byte offsets.

4. **Encode passages with late chunking.** Feed the full multi-chunk sequence through the encoder with bidirectional attention. Extract per-chunk embeddings by mean-pooling the token representations within each chunk's span. Apply the INT8 quantization formula: `floor(127 * tanh(mean_pool) + 0.5)` to produce integer embeddings in [-127, 127]. This is natively supported if using the model's built-in pooling; replicate it if building a custom pipeline.

5. **Encode queries.** Queries are typically short (under 256 tokens) and do not need late chunking. Encode each query as a single sequence and mean-pool to get the query embedding. Apply the same INT8 quantization. No instruction prefix is needed -- pplx-embed is instruction-free.

6. **Build the retrieval index.** Store INT8 embeddings in a vector database (FAISS IVF with `IndexBinaryFlat` for binary, or `IndexIVFScalarQuantizer` for INT8). For the 4B model, embeddings are 2,560-dimensional; for 0.6B, 1,024-dimensional. INT8 reduces storage by 4x vs. float32; binary by 16x (at a 1-1.6% quality cost for 4B).

7. **Implement retrieval and reranking.** At query time, compute the query embedding, retrieve top-k candidates via approximate nearest neighbor search (dot product on INT8 vectors), then optionally rerank with full float32 embeddings or a cross-encoder for the top 50-100 results.

8. **Integrate with RAG or downstream systems.** Pass retrieved passages (with their document-level context metadata) to the generation model. For context-v1 embeddings, the retrieved chunks already encode surrounding context, reducing the need to fetch adjacent chunks as extra context.

9. **Evaluate on standard benchmarks.** Measure nDCG@10 on MTEB Multilingual v2 (target: ~69.7% for 4B INT8), Recall@1000 on large corpora, and ConTEB for contextual retrieval. Compare against baselines like Qwen3-Embedding and voyage-context-3.

10. **Monitor and tune in production.** Track query latency, recall@k at different k values, and embedding freshness. Retune the FAISS index parameters (nprobe, nlist) as the corpus grows. Consider binary quantization if storage becomes the bottleneck on corpora exceeding 10M documents.

## Concrete Examples

**Example 1: Building a multilingual documentation search system**

User: "I need to build a search system over our product documentation in 12 languages. Documents are 2-10 pages each. We have about 500K documents."

Approach:
1. Choose pplx-embed-context-v1 (4B) since documents are long and passage context matters for documentation
2. Chunk each document into 256-token segments, grouping up to 16 chunks per encoding pass
3. Encode with late chunking to produce context-aware INT8 embeddings (2,560-dim)
4. Store in FAISS with IVF index optimized for 500K * ~20 chunks/doc = 10M vectors
5. At query time, encode the query (no prefix needed), retrieve top-50, return with source doc + chunk offset

```python
import numpy as np
from transformers import AutoModel, AutoTokenizer

# Load model (pseudocode -- adjust for actual model release)
tokenizer = AutoTokenizer.from_pretrained("perplexity/pplx-embed-context-v1-4b")
model = AutoModel.from_pretrained("perplexity/pplx-embed-context-v1-4b")

CHUNK_SIZE = 256
MAX_CHUNKS = 16

def late_chunk_encode(document_text: str) -> list[np.ndarray]:
    """Encode a document with late chunking, returning per-chunk INT8 embeddings."""
    tokens = tokenizer(document_text, return_tensors="pt", max_length=CHUNK_SIZE * MAX_CHUNKS, truncation=True)
    outputs = model(**tokens)
    hidden = outputs.last_hidden_state[0]  # (seq_len, dim)

    chunk_embeddings = []
    seq_len = hidden.shape[0]
    for start in range(0, seq_len, CHUNK_SIZE):
        end = min(start + CHUNK_SIZE, seq_len)
        chunk_mean = hidden[start:end].mean(dim=0)  # mean pooling per chunk
        # INT8 quantization: floor(127 * tanh(v) + 0.5)
        quantized = np.floor(127.0 * np.tanh(chunk_mean.detach().numpy()) + 0.5).astype(np.int8)
        chunk_embeddings.append(quantized)
    return chunk_embeddings

def encode_query(query_text: str) -> np.ndarray:
    """Encode a query as a single INT8 embedding."""
    tokens = tokenizer(query_text, return_tensors="pt", max_length=CHUNK_SIZE, truncation=True)
    outputs = model(**tokens)
    query_mean = outputs.last_hidden_state[0].mean(dim=0)
    return np.floor(127.0 * np.tanh(query_mean.detach().numpy()) + 0.5).astype(np.int8)
```

**Example 2: Tool/API retrieval from a large function registry**

User: "We have 15,000 internal API endpoints documented as JSON specs. Users describe what they want in natural language and we need to find the right API."

Approach:
1. Choose pplx-embed-v1 (4B) -- API descriptions are short, no document context needed
2. Concatenate each API's name, description, and parameter summary into a single passage
3. Encode all 15K passages to INT8 embeddings (2,560-dim, ~38KB per embedding in float, ~2.5KB in INT8)
4. Build a flat FAISS index (15K vectors is small enough for brute-force)
5. At query time, encode the user's natural language request, retrieve top-5 APIs

```python
# Index construction
import faiss

dim = 2560
index = faiss.IndexFlatIP(dim)  # inner product for normalized embeddings

api_embeddings = []
for api_spec in api_registry:
    text = f"{api_spec['name']}: {api_spec['description']}. Params: {api_spec['params_summary']}"
    emb = encode_passage(text)  # returns float32 normalized
    api_embeddings.append(emb)

embeddings_matrix = np.stack(api_embeddings).astype(np.float32)
faiss.normalize_L2(embeddings_matrix)
index.add(embeddings_matrix)

# Query
query = "Find an API that lets me resize an image and convert it to WebP format"
query_emb = encode_query(query).reshape(1, -1).astype(np.float32)
faiss.normalize_L2(query_emb)
scores, indices = index.search(query_emb, k=5)
# indices[0] contains the top-5 matching API IDs
```

**Example 3: RAG with contextual passage retrieval**

User: "Our RAG pipeline retrieves passages from 50-page contracts but often returns chunks that are meaningless without surrounding context. How do I fix this?"

Approach:
1. Switch from standard embeddings to pplx-embed-context-v1 with late chunking
2. Re-index contracts: encode each full contract (up to 4,096 tokens at a time) with bidirectional attention, then extract per-chunk embeddings
3. Each chunk's embedding now encodes its position and meaning within the full document
4. At retrieval time, the chunk embedding already carries document context -- no need to fetch neighboring chunks
5. Pass the single retrieved chunk to the LLM; it is self-sufficient for answering

```
Before (standard chunking):
  Chunk: "The party shall indemnify for losses described in Section 4.2."
  Problem: "Section 4.2" is meaningless without the rest of the document.

After (late chunking with pplx-embed-context-v1):
  Same chunk's embedding now encodes that Section 4.2 covers "intellectual property infringement"
  because the bidirectional attention saw the full document during encoding.
  Retrieval for "IP liability" now correctly surfaces this chunk.
```

## Best Practices

- **Do:** Use INT8 quantization by default -- it is trained into the model via quantization-aware training and loses negligible quality (<0.3% on MTEB for 4B). Only use float32 if you need maximum precision for a reranking stage.
- **Do:** Use mean pooling (not CLS token pooling). The diffusion pretraining objective distributes information across all token positions, making mean pooling optimal.
- **Do:** Keep chunks at 256 tokens for indexing. This matches the model's contrastive training chunk size and gives the best retrieval granularity.
- **Do:** Use the context-v1 variant when your documents are longer than ~1,000 tokens and passage meaning depends on surrounding content.
- **Avoid:** Adding instruction prefixes to queries or passages. pplx-embed is instruction-free by design -- prefixes will hurt performance.
- **Avoid:** Binary quantization on the 0.6B model. It loses 2-4.4% quality, compared to only 1-1.6% on the 4B model. Use INT8 instead for the smaller model.
- **Avoid:** Encoding chunks independently when using the context variant. The entire point of late chunking is joint encoding -- independent chunk encoding defeats the purpose and falls back to standard retrieval quality.

## Error Handling

- **Sequence length overflow:** If a document exceeds 4,096 tokens (16 chunks of 256), split it into overlapping windows of 16 chunks with 2-chunk overlap. Deduplicate chunk embeddings from the overlap region by keeping the version from the window where the chunk is most central.
- **Empty or very short documents:** Documents shorter than one chunk still work -- mean pool the available tokens. Do not pad to 256 tokens; the model handles variable lengths natively.
- **Dimension mismatch between model sizes:** The 0.6B model outputs 1,024-dim embeddings; the 4B model outputs 2,560-dim. These are not compatible in the same index. If mixing model sizes, project to a shared dimension or maintain separate indices.
- **Score calibration across languages:** Embedding similarity scores are not calibrated across languages. A score of 0.85 in English may correspond to different relevance than 0.85 in Japanese. Use per-language threshold tuning or rank-based fusion.
- **INT8 overflow in similarity computation:** When computing dot products of INT8 vectors, intermediate sums can overflow int8. Cast to int32 before accumulation, or use FAISS which handles this internally.

## Limitations

- The models are optimized for **retrieval** (finding relevant passages), not for semantic textual similarity or classification. Performance on STS tasks may lag behind models specifically tuned for those tasks.
- **Late chunking requires encoding full documents**, which is slower than independent chunk encoding. For real-time indexing of streaming content, standard pplx-embed-v1 with independent chunks may be more practical.
- The context-v1 variant's benefit is most pronounced on **long, structured documents**. For corpora of short texts (tweets, product titles), standard v1 will perform equally well at lower cost.
- Maximum effective context is **4,096 tokens** (16 chunks). Documents significantly longer than this require windowed processing and will not have full end-to-end context in any single chunk embedding.
- The 4B model requires substantial GPU memory for encoding. For CPU-only or low-resource deployments, the 0.6B variant is the practical choice, with ~2-3% lower retrieval quality.
- The diffusion pretraining methodology is not easily reproduced without the 250B-token multilingual corpus and significant compute. This skill focuses on **using** the released models, not retraining them.

## Reference

**Paper:** [Diffusion-Pretrained Dense and Contextual Embeddings](https://arxiv.org/abs/2602.11151v1) (Eslami et al., 2026). Look for: Section 2 on diffusion pretraining objective, Section 3 on the four-stage contrastive pipeline, Section 4 on late chunking and contextual embeddings, and Table 1-5 for benchmark comparisons against Qwen3-Embedding, voyage-3, and NV-Embed.