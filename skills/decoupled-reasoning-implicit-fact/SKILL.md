---
name: "decoupled-reasoning-implicit-fact"
description: "Build dual-model pipelines that decouple knowledge extraction from reasoning over long documents. Compress document chunks into dense query-conditioned representations before feeding them to a reasoning LLM. Triggers: 'build a DRIFT pipeline', 'compress long context for reasoning', 'decouple knowledge extraction from reasoning', 'query-conditioned document compression', 'dual-model long-context inference', 'implicit fact token pipeline'"
---

# Decoupled Reasoning with Implicit Fact Tokens (DRIFT)

This skill enables Claude to design and implement dual-model architectures that separate knowledge extraction from reasoning over long documents. Based on the DRIFT framework, the core idea is to use a lightweight "knowledge model" that dynamically compresses document chunks into dense vector representations (implicit fact tokens) conditioned on a user query, then project those representations into a larger "reasoning model" that performs inference without ever seeing the raw text. This eliminates context-window bottlenecks, reduces redundant token processing, and achieves up to 128x compression with maintained accuracy -- outperforming both RAG and static prompt compression on long-context benchmarks.

## When to Use

- When the user needs to process documents that exceed a single model's context window (e.g., 100k+ token corpora) and wants a structured pipeline rather than naive truncation or chunked summarization.
- When building a system where a retrieval step introduces too much noise and the user wants to compress full documents rather than retrieve fragments.
- When the user asks to implement query-aware document compression -- extracting only task-relevant facts from each chunk before reasoning.
- When designing an inference pipeline that must handle real-time, unseen documents without pre-indexing (ruling out static knowledge editing or fine-tuning).
- When the user wants to reduce inference cost on long-context tasks by replacing raw text with dense token representations while preserving answer quality.
- When implementing a two-stage architecture where a small model distills knowledge and a large model reasons over it.

## Key Technique

DRIFT decomposes the typical "stuff everything into the prompt" approach into two explicit stages. A **knowledge model** (small, e.g., 3B parameters) reads each document chunk together with the user's query. Special compression tokens (`<|CPS|>`) are appended to each chunk; their final hidden states become the "implicit fact tokens" -- dense vectors that encode the query-relevant information from that chunk. A default compression ratio of 32:1 means 8,192 raw tokens become ~256 dense tokens. These are not summaries; they are continuous-space representations that preserve information a discrete summary would lose.

A learned **projection layer** (3-layer MLP) maps these implicit fact tokens from the knowledge model's embedding space into the **reasoning model's** embedding space (larger, e.g., 7B parameters). The reasoning model receives the projected tokens concatenated with the original query embedding and generates the final answer. Because the reasoning model never processes the raw document text, its effective context window extends dramatically -- a 256k-token document becomes a few thousand dense tokens.

The training follows a three-stage curriculum: (1) **static compression pretraining** where the knowledge model learns to reconstruct documents from compressed tokens, (2) **query-conditioned fine-tuning** on question-evidence pairs so compression becomes dynamic, and (3) **end-to-end QA fine-tuning** where both models are trained jointly. This curriculum prevents the common failure mode where compressed representations lose critical facts under high compression ratios.

## Step-by-Step Workflow

1. **Chunk the input documents** using a recursive character splitter that respects natural boundaries (paragraphs, sentences). Target chunk size: 4,096-8,192 tokens. Use overlapping windows (10-15% overlap) to avoid splitting critical information across chunk boundaries.

2. **Pair each chunk with the query** to form knowledge-model inputs. Format: `[query] [separator] [chunk_text] [<|CPS|> tokens]`. The number of CPS tokens determines the compression ratio (e.g., 256 CPS tokens for a 8,192-token chunk = 32:1 compression).

3. **Run the knowledge model** on each chunk-query pair. Extract the final hidden states at the CPS token positions. These vectors are your implicit fact tokens -- one set per chunk, each shaped `(num_cps_tokens, hidden_dim)`.

4. **Project implicit fact tokens** through the MLP projector into the reasoning model's embedding space. The projector maps `hidden_dim_knowledge -> hidden_dim_reasoning` via three linear layers with GELU activations.

5. **Concatenate projected tokens** from all chunks in document order, inserting separator embeddings between chunk boundaries. Prepend the reasoning model's own embedding of the query.

6. **Run the reasoning model** on the concatenated sequence: `[query_embedding, chunk_1_facts, sep, chunk_2_facts, sep, ..., chunk_n_facts]`. Generate the answer autoregressively.

7. **Validate output quality** by spot-checking against a subset of raw-text answers. If accuracy drops below threshold, reduce the compression ratio (e.g., from 32:1 to 16:1) or increase chunk overlap.

8. **Tune the compression ratio** based on the task. Factoid QA tolerates 64-128x compression. Multi-hop reasoning needs 16-32x. Summarization tasks need 8-16x to preserve narrative flow.

9. **Implement caching** for the knowledge model outputs. If the same document is queried multiple times with different questions, re-run compression (it is query-conditioned). If the same query hits different documents, cache and reuse the query embedding.

10. **Monitor latency vs. accuracy tradeoffs** in production. Log compression ratio, chunk count, reasoning model input length, and answer quality metrics to identify the optimal operating point for your domain.

## Concrete Examples

**Example 1: Long-Document QA Pipeline**

```
User: I have a 200-page technical report (roughly 150k tokens). I need to answer
specific questions about it without truncating. Build me a pipeline that compresses
the document and reasons over it.

Approach:
1. Split the report into ~20 chunks of 8,192 tokens using RecursiveCharacterTextSplitter
   with separators=["\n\n", "\n", ". ", " "] and 800-token overlap.
2. For each chunk, prepend the user's question and append 256 <|CPS|> tokens.
3. Run Qwen2.5-3B (knowledge model) on each chunk-query pair. Collect the 256-dim
   hidden states at CPS positions -> 20 sets of implicit fact tokens.
4. Project each set through the trained 3-layer MLP (3B hidden_dim -> 7B hidden_dim).
5. Concatenate: [query_emb] + [facts_chunk_1] + [sep] + ... + [facts_chunk_20].
   Total reasoning input: ~5,400 tokens (query + 20*256 + separators) instead of 150k.
6. Run Mistral-7B (reasoning model) on the compressed input. Generate answer.

Output:
- Reasoning model input reduced from 150,000 tokens to ~5,400 tokens (28x reduction)
- Answer grounded in full document content, not just retrieved fragments
- Latency: ~7x faster than processing full 150k context natively
```

**Example 2: Multi-Document Comparison**

```
User: I have 5 research papers (each ~30k tokens). I need to compare their methodologies
and find contradictions. RAG keeps missing cross-document connections.

Approach:
1. Chunk each paper into ~4 chunks of 8,192 tokens (20 chunks total).
2. Form the query: "What methodology does this paper use, including assumptions,
   datasets, evaluation metrics, and claimed results?"
3. Compress each chunk with the knowledge model at 32:1 ratio.
4. Group compressed tokens by paper: [paper_1_facts, paper_sep, paper_2_facts, ...].
5. Feed all 20 chunk representations (~5,120 tokens) to the reasoning model with
   the comparison query: "Compare methodologies across these papers. Identify
   contradictions in claims, datasets, or evaluation approaches."
6. The reasoning model sees dense representations of ALL papers simultaneously,
   enabling cross-document reasoning that chunk-level RAG cannot achieve.

Output:
- Full 150k tokens of content compressed to ~5,120 reasoning tokens
- Cross-paper contradictions identified that RAG missed (because RAG retrieves
  per-query fragments, losing inter-document context)
- Structured comparison table generated from the dense representations
```

**Example 3: Implementing the Training Pipeline**

```
User: I want to train my own DRIFT system with a domain-specific knowledge model.
Walk me through the setup.

Approach:
1. Prepare training data from domain corpus (e.g., Wikipedia snapshot or domain docs).
   Stage 1 needs document chunks only. Stages 2-3 need (question, evidence, answer) triples.

2. Stage 1 -- Static Compression Pretraining (LFRP):
   - Freeze the reasoning model. Train knowledge model + projector only.
   - Task: reconstruct original chunk text from CPS token representations.
   - Start with short chunks (64-128 tokens), gradually increase to 512-1024.
   - Compression ratio: 8:1 (gentler, lets model learn basic compression).
   - LoRA config: rank=16, alpha=32, dropout=0.05, lr=1e-4, batch=128.

3. Stage 2 -- Query-Conditioned Fine-Tuning (QAFT-DC):
   - Train on (query, chunk) -> evidence extraction pairs.
   - Increase compression ratio to 32:1 (now query-conditioned).
   - The knowledge model learns to discard query-irrelevant content.

4. Stage 3 -- End-to-End QA Fine-Tuning (QAFT-QA):
   - Unfreeze both models. Train on full QA pipeline.
   - Input: query + document chunks. Output: correct answer.
   - This stage contributes the most to final accuracy.

Output:
- Three-stage training script with curriculum learning
- Domain-adapted knowledge model that compresses your specific document types
- Projection layer aligned to your chosen reasoning model
```

## Best Practices

- **Do:** Use query-conditioned compression rather than static summarization. The same document chunk should compress differently for "What year was X founded?" vs. "Explain X's business model." This is the core insight of DRIFT.
- **Do:** Start with a 32:1 compression ratio as the default and adjust based on task complexity. Factoid extraction tolerates higher ratios; multi-hop reasoning needs lower ones.
- **Do:** Respect natural document boundaries when chunking. Use paragraph and sentence breaks as split points. Semantic coherence within chunks directly impacts compression quality.
- **Do:** Train with curriculum learning (short-to-long chunks). Jumping directly to long contexts causes training instability and poor compression of fine-grained facts.
- **Avoid:** Compressing very short documents (under 2k tokens). The overhead of the dual-model pipeline is not justified when the raw text fits comfortably in the reasoning model's context.
- **Avoid:** Using the same compressed representations for different queries. Implicit fact tokens are query-conditioned -- reusing them across queries defeats the purpose and degrades accuracy.
- **Avoid:** Setting compression ratios above 64:1 for tasks requiring precise numerical or date-based reasoning. Dense representations lose fine-grained discrete facts at extreme compression.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| Answer quality drops sharply | Compression ratio too aggressive | Reduce ratio from 32:1 to 16:1 or 8:1; check if critical facts span chunk boundaries |
| Knowledge model outputs degenerate representations | Insufficient Stage 1 pretraining | Extend LFRP training; verify reconstruction loss has converged before moving to Stage 2 |
| Projection layer produces out-of-distribution embeddings | Embedding space mismatch between models | Verify hidden dimensions match; re-initialize projector MLP; increase projector depth |
| Reasoning model hallucinates despite compressed input | Implicit fact tokens lack needed information | Increase CPS token count per chunk; decrease chunk size to improve per-chunk coverage |
| Chunks split mid-sentence causing lost context | Chunking strategy too rigid | Switch to RecursiveCharacterTextSplitter with semantic delimiters; increase overlap to 15-20% |
| Latency worse than expected | Too many small chunks creating overhead | Increase chunk size; batch knowledge model inference across chunks |

## Limitations

- **Requires paired model training.** You cannot plug arbitrary models together without training the projection layer. The knowledge model, projector, and reasoning model must be co-trained or fine-tuned as a unit.
- **Query-dependent compression means no pre-computation.** Unlike embedding-based retrieval, you cannot pre-compress documents and reuse representations across arbitrary queries. Each new query requires re-running the knowledge model.
- **Not suitable for generative tasks requiring verbatim text.** Implicit fact tokens are lossy representations. Tasks that need exact quotes, code snippets, or precise formatting should use raw text or hybrid approaches.
- **Small knowledge models may struggle with highly technical or domain-specific jargon.** A 3B-parameter knowledge model may not adequately represent specialized terminology without domain-specific fine-tuning.
- **The three-stage training pipeline is resource-intensive.** While inference is efficient, training requires significant compute for curriculum learning across all three stages.

## Reference

**Paper:** [Decoupled Reasoning with Implicit Fact Tokens (DRIFT)](https://arxiv.org/abs/2602.10021v1) -- Xie et al., 2026. Focus on Section 3 (Method) for the dual-model architecture and compression mechanism, and Section 4 (Experiments) for compression ratio vs. accuracy tradeoffs across LongBench-v2 and other benchmarks. Code repository (under development): [github.com/Lancelot-Xie/DRIFT](https://github.com/Lancelot-Xie/DRIFT).