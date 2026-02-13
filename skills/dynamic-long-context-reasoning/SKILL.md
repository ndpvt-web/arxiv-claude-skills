---
name: dynamic-long-context-reasoning
description: |
  Process extremely long documents and contexts by compressing them into memory chunks, selectively retrieving relevant blocks via a gating mechanism, and reasoning iteratively with working memory. Based on the CogMem framework (arXiv:2602.08382). Use this skill when the user says:
  - "Analyze this huge codebase and answer questions about it"
  - "Process this long document and find the relevant sections"
  - "Reason over multiple files to trace a bug"
  - "Summarize and query across a 100K+ token context"
  - "Handle this context that's too long to fit in one pass"
  - "Multi-hop reasoning over a large repository"
---

# Dynamic Long Context Reasoning over Compressed Memory

This skill enables Claude to handle contexts far exceeding normal token limits by applying the compress-gate-reason pipeline from CogMem (Chen et al., 2026). Instead of processing all raw tokens at once — which causes quadratic compute costs, information loss, and fragmented retrieval — you segment the input into chunks, compress each into a compact memory representation, selectively gate which memories are relevant to the current query, and iteratively reason over selected blocks with an evolving working memory. This approach achieves competitive accuracy on multi-hop reasoning benchmarks while reducing memory usage by 2x and providing 6x speedups over agent-based baselines.

## When to Use

- When the user provides a codebase, log file, document set, or transcript that exceeds comfortable context limits (roughly >50K tokens)
- When multi-hop reasoning is needed across scattered sections of a long input (e.g., tracing a function call chain across 20 files)
- When the user asks to "find all relevant pieces" in a large corpus and synthesize an answer
- When RAG-style retrieval alone would fragment the context and miss cross-chunk dependencies
- When the user needs to query the same large context repeatedly with different questions
- When processing long conversation histories, audit logs, or multi-file codebases to answer specific questions

## Key Technique

**Chunk-wise compression.** The input (codebase, document, logs) is divided into fixed-size chunks. Each chunk is independently compressed into a compact memory block — a summary representation that preserves the essential semantics while drastically reducing token count. In practice, this means producing a structured summary or embedding-like distillation of each chunk rather than keeping raw text. The compression ratio is aggressive: each chunk of hundreds or thousands of tokens maps to a memory block of perhaps 10-50 tokens of distilled content.

**Gating for selective recall.** Not all memory blocks matter for a given query. A gating mechanism scores each compressed memory block against the current question and selects only the top-k most relevant blocks. This is analogous to a learned classifier that decides "which chunks of this 200-file codebase actually relate to the user's question about authentication?" The gating prevents flooding the reasoning stage with irrelevant context, which is the core weakness of naive concatenation and many RAG approaches.

**Iterative reasoning with working memory.** The reasoning stage does not process all selected blocks in a single pass. Instead, it maintains a working memory state that evolves as it reads through selected memory blocks one at a time. After each block is processed, the working memory is updated — old information may be refined, contradictions resolved, and partial answers accumulated. This iterative refinement is what enables multi-hop reasoning: the system can chain facts from block 3 and block 17 even though they were never in the same context window simultaneously.

## Step-by-Step Workflow

1. **Inventory the input.** Determine the total size and structure of the context. Count files, tokens, or pages. Identify natural segmentation boundaries (file boundaries for code, section breaks for documents, timestamp gaps for logs).

2. **Segment into chunks.** Divide the input along semantic boundaries. For codebases: one chunk per file or per logical module. For documents: one chunk per section or per N paragraphs. For logs: one chunk per time window or per event group. Aim for chunks of 500-2000 tokens each. Record chunk IDs and their source locations.

3. **Compress each chunk into a memory block.** For each chunk, produce a compact memory representation:
   - For code files: extract the file path, top-level declarations (functions, classes, exports), key dependencies (imports), and a 1-2 sentence purpose summary.
   - For document sections: extract the section heading, key entities mentioned, core claims or facts, and any cross-references.
   - For log segments: extract the time range, event types, error codes, and notable state transitions.
   - Store each memory block as a structured record: `{chunk_id, source, summary, key_entities, relationships}`.

4. **Parse and decompose the query.** Break the user's question into sub-questions or required information pieces. For "Why does the auth middleware reject requests from the mobile client?", decompose into: (a) Where is the auth middleware? (b) How does it validate requests? (c) What is different about mobile client requests? (d) Where do they diverge?

5. **Gate: score and select relevant memory blocks.** For each sub-question, score all memory blocks by relevance. Use keyword overlap, entity matching, and semantic similarity between the query terms and each block's summary/entities. Select the top-k blocks per sub-question (typically k=3-8). Merge selected blocks across sub-questions, removing duplicates.

6. **Retrieve full content for selected chunks.** Go back to the original source and read the full content of only the selected chunks. This is the key efficiency win — you read perhaps 10-20% of the total input instead of 100%.

7. **Iterative reasoning pass.** Process the selected chunks one at a time in relevance order. Maintain a working memory note that accumulates findings:
   - After chunk 1: "Auth middleware is in `src/middleware/auth.ts`, uses JWT validation."
   - After chunk 2: "Mobile client sends token in query param, not header."
   - After chunk 3: "Middleware only checks `Authorization` header, never query params."
   - Working memory now contains the chain of reasoning needed for the answer.

8. **Synthesize the answer.** Combine the accumulated working memory into a coherent response. Cite specific file paths, line numbers, or document sections from the chunks you read. If the working memory reveals gaps, go back to step 5 with refined queries targeting the gaps.

9. **Validate and cross-check.** Verify the answer is consistent across all processed chunks. If two chunks provide conflicting information, explicitly flag the conflict and read additional surrounding chunks to resolve it.

10. **Report with provenance.** Deliver the answer with clear references to which chunks/files/sections supported each claim. This traceability is essential for the user to verify the reasoning.

## Concrete Examples

**Example 1: Multi-hop bug tracing across a large codebase**

```
User: "The /api/orders endpoint returns 500 errors intermittently.
       Here's our codebase (47 files, ~80K tokens). Find the root cause."

Approach:
1. Segment: 47 files become 47 chunks. Compress each into a memory block
   with file path, exports, imports, and purpose summary.

2. Decompose query: (a) Where is /api/orders defined? (b) What does
   its handler call? (c) Which downstream dependencies could fail
   intermittently?

3. Gate on sub-question (a): Memory blocks for route definitions score
   highest. Select `src/routes/orders.ts` and `src/routes/index.ts`.

4. Read `src/routes/orders.ts` fully. Working memory: "Handler calls
   OrderService.getAll(), wrapped in try/catch that re-throws."

5. Gate on sub-question (b): Select `src/services/OrderService.ts`.
   Read it. Working memory updated: "getAll() calls db.query() and
   also calls InventoryClient.checkStock()."

6. Gate on sub-question (c): Select `src/clients/InventoryClient.ts`.
   Read it. Working memory updated: "checkStock() has a 5-second
   timeout but no retry logic. Connection pool max is 2."

7. Synthesize: "The intermittent 500 errors originate from
   InventoryClient.checkStock() (src/clients/InventoryClient.ts:34)
   which has a 5s timeout and no retry. Under concurrent load, the
   connection pool (max: 2) exhausts, causing timeouts that propagate
   as unhandled errors through OrderService to the route handler."

Chunks read: 5 out of 47 (11% of codebase).
```

**Example 2: Answering questions over a long policy document**

```
User: "Here's our 200-page employee handbook (150K tokens). Can an
       employee in the London office take unpaid leave to care for a
       sick parent while keeping their health benefits?"

Approach:
1. Segment by section headings. ~60 chunks. Compress each with section
   title, key policies mentioned, applicable regions, and conditions.

2. Decompose: (a) Unpaid leave policy, (b) London/UK-specific rules,
   (c) Caregiver leave provisions, (d) Benefits continuation during
   unpaid leave.

3. Gate: Select memory blocks whose summaries mention "unpaid leave",
   "UK", "London", "caregiver", "family medical", "benefits".
   Result: 6 blocks selected from sections 4.3, 4.7, 7.1, 7.4, 9.2,
   and Appendix B.

4. Read each fully. Build working memory iteratively:
   - Section 4.3: "Up to 12 weeks unpaid leave per year, manager approval."
   - Section 7.1: "UK employees have statutory right to emergency
     dependant leave (Employment Rights Act 1996)."
   - Section 9.2: "Health benefits continue during approved unpaid leave
     up to 90 days; employee pays full premium after 30 days."
   - Appendix B: "London office follows UK statutory minimums plus
     company enhancements in Section 4."

5. Synthesize: "Yes. Under Section 4.3, employees may take up to 12
   weeks unpaid leave with manager approval. UK employees additionally
   have statutory dependant leave rights (Section 7.1). Health benefits
   continue for up to 90 days per Section 9.2, but the employee must
   pay the full premium after the first 30 days."

Chunks read: 6 out of 60 (10% of document).
```

**Example 3: Querying a large log file for root cause analysis**

```
User: "Here's 24 hours of application logs (500K tokens). The service
       crashed at 03:42 UTC. What caused it?"

Approach:
1. Segment into 15-minute time windows. ~96 chunks. Compress each with
   time range, log levels present, error codes, and notable events.

2. Gate: Prioritize chunks near the crash time (03:15-03:45 UTC) and
   any earlier chunks whose summaries contain ERROR or WARN levels.
   Select 8 chunks: 03:15-03:30, 03:30-03:45, and 6 earlier chunks
   with warnings.

3. Iterative reasoning:
   - 01:00-01:15: "WARNING: Connection pool utilization at 85%."
   - 02:15-02:30: "WARNING: Memory usage 78%, GC pause 450ms."
   - 03:00-03:15: "ERROR: OOM killer threshold approached. 3 failed
     mallocs in request handler."
   - 03:30-03:45: "FATAL: Out of memory. Process killed by OOM killer
     at 03:42:17."

4. Synthesize: "Memory leak starting around 01:00 (pool at 85%) grew
   until OOM kill at 03:42. The failed mallocs in the request handler
   (03:00-03:15 window) suggest the leak is in request processing,
   not a background task."

Chunks read: 8 out of 96 (8% of logs).
```

## Best Practices

- **Do:** Respect natural boundaries when chunking. File boundaries, section headings, and function definitions are better split points than arbitrary token counts. Splitting mid-function or mid-paragraph destroys local context.

- **Do:** Keep memory block summaries structured and entity-rich. Include names, paths, identifiers, and relationships. A summary like "handles user auth" is far less gateable than "exports `validateJWT(token: string)`, imports `jsonwebtoken`, called by `authMiddleware`."

- **Do:** Update working memory after each chunk, not just at the end. The iterative accumulation is what enables multi-hop reasoning. Write down intermediate findings explicitly.

- **Do:** Re-gate when the first round of reasoning reveals new sub-questions. The initial query decomposition is a best guess — the actual evidence often points to unexpected areas.

- **Avoid:** Compressing too aggressively. If your memory block drops below ~10% of the original chunk's information content, you lose the ability to gate accurately. Better to have slightly larger blocks than to miss key entities.

- **Avoid:** Selecting too many or too few blocks. Selecting all blocks defeats the purpose (no efficiency gain). Selecting only 1-2 blocks risks missing a hop in multi-hop reasoning. Aim for 5-15% of total chunks per query.

## Error Handling

- **Gating misses a critical chunk.** If the reasoning hits a dead end ("I know X calls Y, but Y isn't in any selected chunk"), go back and re-gate with the new entity Y as the query. This is the working-memory feedback loop.

- **Chunk boundaries split critical context.** If a function definition spans two chunks, the summary of each half may be incoherent. Detect this by noticing incomplete structures in summaries, then merge adjacent chunks and re-compress.

- **Working memory contradictions.** When chunk A says "timeout is 30s" and chunk B says "timeout is 5s", do not silently pick one. Flag the conflict, read surrounding chunks for both, and determine which is authoritative (e.g., config file overrides code default).

- **Query too vague for effective gating.** If the user asks "what's wrong with this codebase?", gating has no signal. Ask the user to narrow the question, or do a broad first pass: compress all chunks, read a random 10% sample to identify candidate issues, then re-gate on specific issues found.

## Limitations

- **Not a substitute for full-text search.** If the user needs an exact string match ("find every occurrence of `TODO`"), use grep/search tools directly. Compression discards literal content.

- **Compression is lossy.** Subtle details (specific numeric values, exact error messages, formatting) may be lost in memory blocks. Always go back to the full chunk content before citing specifics.

- **Single-hop factoid queries don't need this.** If the question is "what port does the server listen on?", a simple search is faster and more reliable than the full compress-gate-reason pipeline. Reserve this for multi-hop or synthesis tasks.

- **Quality depends on chunk summarization.** If the compression step produces poor summaries (misidentifies purpose, drops key entities), the entire pipeline degrades. Invest time in step 3.

- **Not suited for real-time streaming.** This workflow assumes the full input is available upfront. For streaming contexts (live logs, ongoing conversations), you need an incremental variant that compresses chunks as they arrive.

## Reference

[Dynamic Long Context Reasoning over Compressed Memory via End-to-End Reinforcement Learning](https://arxiv.org/abs/2602.08382v1) — Chen et al., 2026. Read Sections 3 (framework architecture: compressor, gating, reasoner) and 4 (RL training procedure and benchmark results on RULER-HQA) for the formal method and how joint RL optimization of the compressor and reasoner outperforms pipeline approaches.