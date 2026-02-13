---
name: "a-rag-scaling-agentic-retrieval-augmented"
description: >
  Build agentic RAG systems where the LLM autonomously decides retrieval strategy
  using hierarchical interfaces (keyword search, semantic search, chunk read) instead
  of fixed retrieval pipelines. Replaces single-shot retrieval and predefined workflows
  with a ReAct-style loop that scales with model capability.
  Trigger phrases:
  - "build an agentic RAG system"
  - "implement hierarchical retrieval"
  - "make the LLM control its own retrieval"
  - "scale RAG with reasoning"
  - "multi-hop question answering over documents"
  - "replace my fixed RAG pipeline with an agent"
---

# A-RAG: Scaling Agentic Retrieval-Augmented Generation via Hierarchical Retrieval Interfaces

This skill teaches Claude to design and implement agentic RAG systems where the language model autonomously controls retrieval decisions through three hierarchical interfaces -- keyword search, semantic search, and chunk read -- rather than relying on a fixed retrieve-then-read pipeline. The core insight from A-RAG is that exposing retrieval as callable tools at different granularities and letting the model reason about which to invoke (and when) consistently outperforms both single-shot RAG and predefined workflow RAG, while using comparable or fewer tokens.

## When to Use

- When the user needs to build a RAG system that handles multi-hop questions requiring evidence from multiple documents
- When the user's current RAG pipeline returns irrelevant chunks or misses information scattered across a corpus
- When the user wants the LLM to decide whether to do keyword search, semantic search, or read a full chunk based on intermediate reasoning
- When the user asks to replace a rigid retrieve-then-generate pipeline with an agent loop
- When the user wants to build a QA system over a large document corpus (thousands of documents) that scales with model improvements
- When the user is implementing any system where retrieval needs to adapt per-query rather than follow a fixed strategy

## Key Technique

**The problem with existing RAG:** Traditional RAG systems either (1) retrieve passages in a single shot and concatenate them into the prompt, or (2) predefine a workflow (e.g., "first search, then rerank, then read") and prompt the model to execute it step-by-step. Neither approach lets the model participate in retrieval decisions. The model cannot say "that chunk wasn't useful, let me try a keyword search for the entity name instead."

**A-RAG's solution -- hierarchical retrieval interfaces:** Instead of a monolithic retrieval step, A-RAG exposes three tools at different granularities that the LLM calls in a ReAct-style loop (reason, act, observe, repeat):

1. **Keyword Search** -- Exact lexical matching across all chunks. Returns chunk IDs with short snippets containing matched keywords. Relevance is scored by keyword frequency weighted by character length. Best for entity names, dates, specific terms.
2. **Semantic Search** -- Dense embedding similarity (e.g., using a sentence encoder). Returns top-k sentences aggregated by parent chunk. Best when the user's question uses different wording than the source text.
3. **Chunk Read** -- Retrieves the full content of a specific chunk by ID, after search has identified promising candidates. A context tracker prevents re-reading already-consumed chunks, returning "already read" at zero token cost.

**Why this scales:** Because the agent chooses its own retrieval strategy per reasoning step, stronger models make better tool-use decisions and achieve higher accuracy without changing the retrieval infrastructure. Experiments show ~8% improvement when scaling from 5 to 20 agent steps, and ~25% improvement when scaling reasoning effort from minimal to high. The dominant failure mode shifts from "couldn't find the information" (traditional RAG) to "reasoning chain errors" (A-RAG), meaning the retrieval bottleneck is largely removed.

## Step-by-Step Workflow

### 1. Index the corpus into chunks with sentence-boundary alignment

Split documents into chunks of ~1,000 tokens each, respecting sentence boundaries. Store each chunk with a unique ID and metadata (source document, position). This is the unit of retrieval for all three interfaces.

```python
# Example chunking logic
def chunk_document(text, max_tokens=1000):
    sentences = split_into_sentences(text)
    chunks, current_chunk, current_len = [], [], 0
    for sent in sentences:
        sent_len = count_tokens(sent)
        if current_len + sent_len > max_tokens and current_chunk:
            chunks.append(" ".join(current_chunk))
            current_chunk, current_len = [sent], sent_len
        else:
            current_chunk.append(sent)
            current_len += sent_len
    if current_chunk:
        chunks.append(" ".join(current_chunk))
    return chunks
```

### 2. Build a dense sentence-level embedding index

Encode every sentence in the corpus using a sentence encoder (e.g., `sentence-transformers/all-MiniLM-L6-v2` or `Qwen3-Embedding-0.6B`). Store embeddings in a vector index (FAISS, Qdrant, or similar) with mappings from sentence back to parent chunk ID.

### 3. Define the three retrieval tool interfaces

Implement each as a callable function the LLM agent can invoke:

```python
def keyword_search(query: str, top_k: int = 10) -> list[dict]:
    """Exact lexical match across chunks. Returns chunk IDs + keyword snippets."""

def semantic_search(query: str, top_k: int = 10) -> list[dict]:
    """Dense embedding cosine similarity. Returns chunk IDs + matching sentences."""

def chunk_read(chunk_id: str) -> str:
    """Returns full text of a chunk. Tracks reads to avoid redundancy."""
```

### 4. Implement the context tracker

Maintain a set of already-read chunk IDs per query session. When `chunk_read` is called on a previously-read chunk, return a brief "already read" message instead of the full text. This prevents the agent from wasting tokens re-consuming the same content.

### 5. Write the agent system prompt with tool definitions

Define each tool with precise descriptions so the LLM understands when to use each:

```
You are a research assistant answering questions using a document corpus.
You have three retrieval tools:

- keyword_search(query): Use when you need to find chunks containing specific
  entity names, dates, numbers, or exact phrases. Returns abbreviated snippets.
- semantic_search(query): Use when you need to find passages related to a concept
  but the exact wording may differ. Returns matching sentences with chunk IDs.
- chunk_read(chunk_id): Use to read the full content of a chunk identified by
  search. Use this to get complete context after search narrows candidates.

Strategy: Start with search to identify relevant chunks, then read the most
promising ones. Reformulate searches if initial results are insufficient.
Reason about what information you still need before each tool call.
```

### 6. Implement the ReAct agent loop with iteration budget

Run a loop where the LLM alternates between reasoning (thinking about what it knows and needs) and acting (calling a tool). Set a maximum step budget (10-20 steps) to prevent runaway loops.

```python
def agentic_rag(question: str, max_steps: int = 15):
    messages = [system_prompt, {"role": "user", "content": question}]
    read_chunks = set()

    for step in range(max_steps):
        response = llm.chat(messages, tools=RETRIEVAL_TOOLS)
        if response.has_tool_call:
            result = execute_tool(response.tool_call, read_chunks)
            messages.append(response.message)
            messages.append({"role": "tool", "content": result})
        else:
            return response.content  # Final answer
    return "Could not determine answer within step budget."
```

### 7. Implement progressive information disclosure

When keyword or semantic search returns results, show only chunk IDs and brief snippets (not full chunk text). This keeps context compact and forces the agent to explicitly decide which chunks deserve a full read, reducing token waste.

### 8. Add answer extraction with citation

After the agent gathers evidence, instruct it to produce a final answer citing specific chunk IDs. This enables verification and builds trust.

### 9. Tune the iteration budget and top-k parameters

Start with `max_steps=15` and `top_k=10` for searches. Monitor average steps used and retrieved token count. Increase the step budget for harder multi-hop tasks; decrease top-k if snippets overwhelm the context.

### 10. Evaluate and iterate

Test on multi-hop QA benchmarks (HotpotQA, MuSiQue, 2WikiMultiHopQA) or your domain-specific test set. Track: answer accuracy, average steps used, total retrieved tokens, and tool call distribution. The signature of a well-tuned A-RAG system is high accuracy with low retrieved token count.

## Concrete Examples

**Example 1: Multi-hop question over a knowledge base**

```
User: "Who directed the film that won Best Picture the year the Berlin Wall fell?"

Agent reasoning trace:
Step 1 [Reason]: The Berlin Wall fell in 1989. I need to find which film won
  Best Picture at the 1989 Academy Awards (62nd ceremony, held March 1990).
Step 2 [keyword_search("Best Picture 1989")]:
  -> Chunk 412: "...Best Picture nominees for the 62nd Academy Awards..."
  -> Chunk 887: "...Driving Miss Daisy won Best Picture..."
Step 3 [chunk_read("887")]:
  -> Full text confirms Driving Miss Daisy won Best Picture for 1989.
Step 4 [keyword_search("Driving Miss Daisy director")]:
  -> Chunk 201: "...directed by Bruce Beresford..."
Step 5 [Reason]: I have the answer. Driving Miss Daisy (1989 Best Picture)
  was directed by Bruce Beresford.

Output: Bruce Beresford directed Driving Miss Daisy, which won Best Picture
for the year the Berlin Wall fell (1989). [Sources: chunks 887, 201]
```

**Example 2: Concept-based retrieval where exact keywords fail**

```
User: "What causes the price of goods to rise when a country prints too much money?"

Agent reasoning trace:
Step 1 [Reason]: This is about inflation and monetary policy. Let me try
  semantic search since the corpus may not use the exact phrasing "prints money."
Step 2 [semantic_search("effects of increasing money supply on prices")]:
  -> Chunk 55: sentence about "monetary expansion leading to demand-pull inflation"
  -> Chunk 312: sentence about "quantity theory of money"
Step 3 [chunk_read("55")]:
  -> Detailed explanation of demand-pull inflation mechanism.
Step 4 [chunk_read("312")]:
  -> Explains MV = PQ and how increasing M with constant V and Q raises P.
Step 5 [Reason]: Both chunks explain the mechanism. Chunk 55 covers the
  practical effect, chunk 312 covers the theoretical foundation.

Output: When a country increases its money supply significantly, it causes
inflation through demand-pull effects: more money chasing the same goods
bids up prices. This is formalized by the quantity theory of money (MV = PQ),
where increasing M (money supply) with stable V (velocity) and Q (output)
forces P (price level) upward. [Sources: chunks 55, 312]
```

**Example 3: Implementing A-RAG in a Python project**

```
User: "I have a FastAPI app with a basic RAG endpoint that does single-shot
retrieval. Help me convert it to agentic RAG."

Approach:
1. Read the existing endpoint to understand the current retrieval logic
2. Refactor the retrieval into three separate tool functions (keyword_search,
   semantic_search, chunk_read) matching the A-RAG interface
3. Add a context tracker (set of read chunk IDs per request)
4. Replace the single retrieve-then-generate call with a ReAct loop using
   the LLM's tool-calling API
5. Add an iteration budget parameter to the endpoint
6. Update the system prompt to describe the three tools and when to use each

Output: Modified endpoint with agentic_rag() function, three tool
implementations, context tracking, and configurable max_steps parameter.
```

## Best Practices

**Do:**
- Keep search result snippets short (1-2 sentences max) to force explicit chunk reads and save context tokens
- Track read chunks per session and return "already read" immediately to prevent redundant token consumption
- Use keyword search first for entity-centric queries (names, dates, codes) and semantic search for conceptual queries
- Set a firm iteration budget (15-20 steps) and surface it as a configurable parameter
- Log the full reasoning trace (tool calls + reasoning) for debugging retrieval failures
- Start with a smaller top-k (5-10) and increase only if recall is insufficient

**Avoid:**
- Do not concatenate all search results into the prompt at once -- this defeats the purpose of progressive disclosure
- Do not predefine the order of tool calls (e.g., "always keyword first, then semantic") -- let the model decide
- Do not skip the chunk_read step by embedding full chunk text in search results -- snippets-then-read is what makes this token-efficient
- Do not use excessively large chunks (>1500 tokens) -- they reduce the precision of search and waste tokens on irrelevant content
- Do not omit the "already read" deduplication -- without it, agents waste 30-40% of their token budget re-reading

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| Agent loops without finding information | Hits max_steps with no answer | Add a fallback: if step budget exhausted, return best-effort answer from chunks already read |
| Search returns no results | Empty result set from keyword/semantic search | Instruct the agent to reformulate the query or switch search modality (keyword <-> semantic) |
| Agent reads too many chunks | High token usage, slow responses | Reduce top-k, tighten snippet length, or lower max_steps budget |
| Entity confusion in multi-hop | Agent conflates two entities with similar names | Add entity disambiguation in the system prompt; instruct the agent to verify entity identity before chaining |
| Chunk boundaries split key information | Answer spans two chunks | Implement adjacent chunk reading: when chunk_read is called, optionally include the preceding/following chunk |

## Limitations

- **Requires tool-calling capable models:** The agent loop depends on the LLM reliably generating structured tool calls. Smaller or older models may produce malformed calls or ignore tools entirely.
- **Latency vs. single-shot RAG:** Multiple sequential tool calls add latency. A 10-step agent trace takes ~10x the LLM call time of single-shot RAG. Use single-shot for simple factoid queries and reserve agentic RAG for multi-hop or complex questions.
- **Reasoning errors dominate at scale:** Once retrieval is solved, the bottleneck shifts to the LLM's reasoning chain. Entity confusion accounts for the majority of errors in multi-hop settings -- this is a model capability limitation, not a retrieval one.
- **Index quality still matters:** A-RAG improves retrieval strategy but cannot compensate for poor chunking, missing documents, or low-quality embeddings.
- **Cost scales with steps:** Each agent step incurs an LLM API call. Budget-constrained deployments should tune max_steps carefully and consider routing simple queries to single-shot RAG.

## Reference

**Paper:** [A-RAG: Scaling Agentic Retrieval-Augmented Generation via Hierarchical Retrieval Interfaces](https://arxiv.org/abs/2602.03442v1) (Du et al., 2026)

**Key takeaway:** Exposing three retrieval interfaces (keyword search, semantic search, chunk read) as tools in a ReAct loop lets the LLM make its own retrieval decisions, achieving higher accuracy with fewer retrieved tokens than single-shot or workflow-based RAG -- and the performance scales naturally with model capability improvements.