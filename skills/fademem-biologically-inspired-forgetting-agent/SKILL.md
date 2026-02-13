---
name: "fademem-biologically-inspired-forgetting-agent"
description: >
  Implement biologically-inspired forgetting mechanisms for LLM agent memory systems.
  Build dual-layer memory hierarchies with adaptive exponential decay, semantic relevance scoring,
  and LLM-guided conflict resolution to keep agent context lean and high-quality.
  Use when: "add forgetting to my agent memory", "implement memory decay for my chatbot",
  "build an agent memory system with selective retention", "reduce memory bloat in my AI agent",
  "implement FadeMem-style memory management", "add adaptive memory consolidation to my agent".
---

# FadeMem: Biologically-Inspired Forgetting for Agent Memory

This skill enables Claude to design and implement agent memory systems that actively forget irrelevant information using biologically-inspired decay mechanisms. Based on the FadeMem architecture, you build dual-layer memory hierarchies (working memory + long-term memory) where each memory entry decays at rates governed by semantic relevance, access frequency, and temporal recency. Rather than storing everything or dropping everything at a context boundary, this approach continuously prunes low-value memories while consolidating important ones -- achieving ~45% storage reduction while improving multi-hop reasoning and retrieval quality.

## When to Use

- When the user is building a conversational agent that needs to remember information across many sessions without unbounded memory growth
- When the user asks to add forgetting, decay, or memory management to an existing agent or chatbot system
- When the user wants to reduce token usage or storage costs for an agent that accumulates too much context
- When the user needs to resolve contradictions between old and new information in agent memory
- When the user is implementing a RAG system and wants smarter retention policies than simple FIFO or fixed-window truncation
- When the user asks for a memory architecture that prioritizes important memories and lets irrelevant ones fade

## Key Technique

**Dual-Layer Memory with Differential Decay.** FadeMem divides agent memory into two tiers: a *working memory* that holds recent, high-activation entries (analogous to a conversation buffer), and a *long-term memory* that stores consolidated, durable knowledge. Each memory entry carries a *retention score* computed by an adaptive exponential decay function:

```
retention(m) = base_relevance(m) * exp(-lambda(m) * time_since_last_access(m))
```

The decay rate `lambda` is not fixed -- it adapts per-entry based on three modulators: (1) **semantic relevance** to the agent's current task or recent queries, which slows decay for on-topic memories; (2) **access frequency**, where frequently retrieved memories decay slower (a "use it or lose it" principle); and (3) **temporal pattern**, where memories from bursty, clustered access patterns are treated as more important than one-off mentions. Entries whose retention score drops below a threshold are either consolidated (merged with related entries via summarization) or permanently forgotten.

**LLM-Guided Conflict Resolution and Fusion.** When the system detects semantically overlapping or contradictory memories (e.g., a user changed their address, or two sessions give different preferences), it invokes the LLM to evaluate which memory is more current, more contextually grounded, or more consistent with the broader memory store. The winner is kept or a fused summary is generated; the loser decays faster. This prevents stale or contradictory information from polluting retrieval results. Memory fusion also compresses verbose multi-turn exchanges into concise factual entries, reducing storage without losing core information.

## Step-by-Step Workflow

1. **Define the memory entry schema.** Each entry needs: `id`, `content` (text), `embedding` (vector), `created_at` (timestamp), `last_accessed_at` (timestamp), `access_count` (integer), `base_relevance` (float 0-1), `retention_score` (float), `layer` (enum: `working` | `long_term`), and `tags` (list of topic strings).

2. **Implement the working memory buffer.** Create a fixed-capacity buffer (e.g., last 20-50 entries) that holds recent interactions. New entries always land here first. When the buffer is full, trigger a consolidation cycle rather than simply dropping the oldest entry.

3. **Implement the adaptive decay function.** For each memory entry, compute:
   ```python
   import math, time

   def compute_retention(entry, current_time, query_embedding=None):
       time_delta = (current_time - entry["last_accessed_at"]) / 3600  # hours
       freq_boost = min(math.log1p(entry["access_count"]) / 5.0, 1.0)
       if query_embedding is not None:
           semantic_sim = cosine_similarity(query_embedding, entry["embedding"])
       else:
           semantic_sim = entry["base_relevance"]
       lambda_decay = 0.1 * (1.0 - 0.4 * freq_boost - 0.3 * semantic_sim)
       retention = entry["base_relevance"] * math.exp(-lambda_decay * time_delta)
       return max(retention, 0.0)
   ```
   The key insight: `lambda_decay` shrinks for frequently-accessed, semantically-relevant entries, so they decay much slower.

4. **Run periodic decay sweeps.** On every N-th interaction (e.g., every 5 turns or on each new session), recompute `retention_score` for all entries. Entries below a `forget_threshold` (e.g., 0.15) are candidates for removal. Entries between `forget_threshold` and a `consolidate_threshold` (e.g., 0.35) are candidates for fusion.

5. **Implement memory fusion via LLM summarization.** Group candidate entries by semantic similarity (cluster embeddings with a cosine threshold of 0.75+). For each cluster, prompt the LLM:
   ```
   Summarize the following related memory entries into a single concise factual statement.
   Preserve key facts, names, dates, and preferences. Drop conversational filler.
   Entries: {entries}
   ```
   Replace the cluster with the fused entry, inheriting the highest `base_relevance` and summed `access_count` from the group.

6. **Implement LLM-guided conflict resolution.** When fusion detects contradictory entries (e.g., cosine similarity > 0.8 but semantic content diverges), prompt the LLM:
   ```
   These two memory entries appear to conflict:
   A (created {date_a}): {content_a}
   B (created {date_b}): {content_b}
   Which is more likely to be current/correct? Return the resolved fact or indicate which to keep.
   ```
   Apply the resolution: keep the winner, accelerate decay on the loser (multiply its `lambda` by 3x), or replace both with a merged entry.

7. **Promote and demote between layers.** After a decay sweep: promote working memory entries with `retention_score > 0.7` and `access_count >= 3` to long-term memory. Demote long-term entries with `retention_score < consolidate_threshold` back to working memory for re-evaluation or fusion. Delete any entry with `retention_score < forget_threshold`.

8. **Implement retrieval with decay-aware ranking.** When the agent needs context, retrieve candidate memories by semantic similarity, then re-rank by multiplying similarity with `retention_score`. This naturally down-ranks stale memories even if they are semantically close:
   ```python
   def retrieve(query_embedding, memories, top_k=10):
       scored = []
       now = time.time()
       for m in memories:
           sim = cosine_similarity(query_embedding, m["embedding"])
           retention = compute_retention(m, now, query_embedding)
           score = sim * 0.6 + retention * 0.4
           scored.append((m, score))
           m["last_accessed_at"] = now  # refresh on access
           m["access_count"] += 1
       scored.sort(key=lambda x: x[1], reverse=True)
       return [m for m, s in scored[:top_k]]
   ```

9. **Wire into the agent loop.** Insert memory operations at three points: (a) after each user turn, encode and store new entries in working memory; (b) before each LLM call, retrieve top-k memories and inject as context; (c) after every N turns, run the decay sweep + consolidation cycle.

10. **Tune thresholds empirically.** Start with `forget_threshold=0.15`, `consolidate_threshold=0.35`, `promotion_threshold=0.7`, base `lambda=0.1`. Monitor memory store size and retrieval quality. If the agent forgets too aggressively, lower `lambda` or raise thresholds. If memory bloats, do the opposite.

## Concrete Examples

**Example 1: Multi-session customer support agent**

User: "Build a memory system for my support chatbot that remembers customer preferences across sessions but doesn't bloat over time."

Approach:
1. Define memory entries with the schema from Step 1, stored in a SQLite database with a vector column (via `sqlite-vec` or similar).
2. On each customer message, extract key facts (name, product, issue, preferences) and store as individual memory entries with `base_relevance` scored by the LLM (0.3 for small talk, 0.7 for product preferences, 0.9 for active issues).
3. On each new session start, run a decay sweep. A customer's old shipping address from 6 months ago with `access_count=1` decays to ~0.08 and gets forgotten. Their product preference accessed 12 times stays at ~0.82 and remains.
4. When two sessions mention different email addresses, conflict resolution fires: the LLM determines the newer one is an update, keeps it, and accelerates decay on the old one.

Output structure:
```python
# memory_store.py
class FadeMemStore:
    def __init__(self, db_path, forget_threshold=0.15, consolidate_threshold=0.35):
        self.db = sqlite3.connect(db_path)
        self.forget_threshold = forget_threshold
        self.consolidate_threshold = consolidate_threshold
        self._init_tables()

    def add(self, content, embedding, relevance=0.5):
        """Add new entry to working memory."""

    def retrieve(self, query_embedding, top_k=10):
        """Semantic search with decay-aware re-ranking."""

    def decay_sweep(self, current_query_embedding=None):
        """Recompute retention scores, consolidate or forget entries."""

    def resolve_conflicts(self, entries):
        """LLM-guided resolution for contradictory memory pairs."""

    def fuse(self, cluster):
        """Summarize a cluster of related entries into one."""
```

**Example 2: Research assistant agent with long-running context**

User: "My research agent accumulates thousands of paper summaries and notes. Help me add FadeMem-style decay so it keeps the most relevant ones."

Approach:
1. Wrap the existing note store with decay metadata (`last_accessed_at`, `access_count`, `retention_score`).
2. Set `base_relevance` using the LLM to score each note's relevance to the agent's declared research topics (passed as a topic vector).
3. Every 50 interactions, run a decay sweep. Notes about papers the user cited 10 times stay strong. Notes about tangentially-browsed papers from weeks ago decay below threshold and get consolidated into a single "background context" summary per topic cluster.
4. Retrieval uses the hybrid scoring (similarity * 0.6 + retention * 0.4), ensuring the agent surfaces actively-used references over stale ones.

Output: A wrapper module that monkey-patches the existing store:
```python
# fademem_wrapper.py
class FadeMemWrapper:
    def __init__(self, base_store, llm_client, embedding_fn):
        self.store = base_store
        self.llm = llm_client
        self.embed = embedding_fn

    def ingest(self, text, topic_relevance=None): ...
    def query(self, question, top_k=15): ...
    def maintenance_cycle(self): ...  # decay + fuse + conflict resolve
```

**Example 3: Adding forgetting to a LangChain or LlamaIndex agent**

User: "I'm using LangChain's ConversationBufferMemory but it grows too large. Add FadeMem-style forgetting."

Approach:
1. Subclass `ConversationBufferMemory` or wrap it with a `FadeMemMemory` class.
2. Override `save_context` to assign decay metadata to each new memory entry.
3. Override `load_memory_variables` to run a lightweight decay check (skip full sweep, just filter by precomputed `retention_score`), returning only entries above `forget_threshold`.
4. Add a `maintenance()` method called every N turns that runs the full sweep with consolidation.
5. Contradictions between early and late conversation context get resolved via the chain's LLM.

```python
from langchain.memory import ConversationBufferMemory

class FadeMemMemory(ConversationBufferMemory):
    def __init__(self, forget_threshold=0.15, **kwargs):
        super().__init__(**kwargs)
        self._decay_metadata = {}
        self._turn_count = 0
        self.forget_threshold = forget_threshold

    def save_context(self, inputs, outputs):
        super().save_context(inputs, outputs)
        # Attach decay metadata to newest entry
        ...

    def load_memory_variables(self, inputs):
        # Filter by retention_score before returning
        ...
```

## Best Practices

- **Do:** Score `base_relevance` at ingestion time using the LLM or a classifier. A well-calibrated initial score is the single biggest lever on retention quality. Factual user preferences should score 0.7-0.9; small talk should score 0.1-0.3.
- **Do:** Always refresh `last_accessed_at` and increment `access_count` on retrieval hits. This is the "use it or lose it" signal that keeps important memories alive.
- **Do:** Batch decay sweeps rather than running on every turn. Every 5-10 turns for chatbots, every session boundary for multi-session agents. Sweeps are O(n) over the memory store.
- **Do:** Log what gets forgotten. Maintain a lightweight "forgotten entries" audit log (just IDs and timestamps) so you can debug recall failures during development.
- **Avoid:** Setting `forget_threshold` too aggressively at first. Start conservative (0.10-0.15) and tighten once you confirm the agent isn't losing important information.
- **Avoid:** Running conflict resolution on every pair of similar memories. Only trigger it when fusion detects contradiction (high embedding similarity but divergent factual content). LLM calls for resolution are expensive.
- **Avoid:** Using uniform decay rates. The entire point of FadeMem is differential decay -- frequently accessed, semantically relevant memories must decay slower than idle ones.

## Error Handling

- **Premature forgetting of critical facts:** If the agent forgets something it shouldn't, check whether `base_relevance` was scored too low at ingestion, or whether the `lambda` base rate is too high. Add a `pinned` flag for critical memories (e.g., user's name, core preferences) that exempts them from decay.
- **Memory fusion produces lossy summaries:** When the LLM's fused summary drops key details, improve the fusion prompt to explicitly enumerate what must be preserved. Alternatively, keep the original entries alongside the summary until the originals decay naturally.
- **Conflict resolution picks the wrong entry:** Add a recency bias to the conflict resolution prompt -- in most agent scenarios, newer information should win unless the user explicitly corrects back.
- **Embedding quality is poor:** Decay-aware retrieval depends on good embeddings. If semantic similarity scores are unreliable, fall back to keyword overlap as a secondary signal in the retrieval ranking.
- **Memory store grows despite decay:** Check that decay sweeps are actually executing. A common bug is forgetting to call `maintenance_cycle()` in the agent loop. Add a turn counter that triggers sweeps automatically.

## Limitations

- **Not suited for perfect-recall requirements.** If the application must never lose any information (legal, medical records), do not use active forgetting. Use archival storage with a FadeMem-style retrieval ranking instead, without actual deletion.
- **LLM cost overhead.** Conflict resolution and fusion require LLM calls. For agents with thousands of memories, batch processing and rate-limiting are necessary to keep costs manageable.
- **Cold start problem.** A new agent has no access history, so decay modulators default to base rates. Early memories may be forgotten too quickly before the system has enough signal. Mitigate by setting higher initial `base_relevance` for the first N entries.
- **Decay parameters are domain-specific.** The thresholds that work for a customer support bot won't work for a research assistant. Expect to tune `lambda`, `forget_threshold`, and `consolidate_threshold` per use case.
- **Does not replace vector database indexing.** FadeMem is a retention policy layer, not a replacement for efficient similarity search. It sits on top of your existing embedding store or vector DB.

## Reference

**Paper:** [FadeMem: Biologically-Inspired Forgetting for Efficient Agent Memory](https://arxiv.org/abs/2601.18642v2) -- Wei et al., 2026. Focus on Section 3 (the adaptive decay formulation and dual-layer architecture) and Section 5 (ablation studies showing the contribution of each decay modulator).