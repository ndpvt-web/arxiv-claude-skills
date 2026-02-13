---
name: "ama-adaptive-memory-multi-agent"
description: "Build adaptive memory systems using coordinated multi-agent collaboration with hierarchical storage and consistency maintenance. Use when: 'build a memory system for my chatbot', 'add long-term memory to my agent', 'implement multi-granularity retrieval', 'create a memory-augmented LLM pipeline', 'handle memory conflicts in conversational AI', 'reduce context window usage with smart retrieval'."
---

# AMA: Adaptive Memory via Multi-Agent Collaboration

This skill teaches Claude to implement the AMA framework -- a multi-agent memory architecture that decomposes long-context information into three granularity levels (atomic facts, episode summaries, and raw text), routes queries adaptively based on intent analysis, and maintains consistency through a judge-and-refresh loop. The technique reduces token consumption by ~80% compared to full-context methods while improving retrieval precision, making it practical for building production memory systems for LLM agents, chatbots, and long-running conversational applications.

## When to Use

- When building a long-term memory system for a conversational AI agent or chatbot
- When the user needs to store, retrieve, and update information across many dialogue sessions
- When implementing RAG that must handle conflicting or outdated information gracefully
- When reducing token costs by replacing full-context injection with targeted multi-granularity retrieval
- When the user asks to build an agentic pipeline where specialized agents manage memory construction, retrieval, verification, and cleanup
- When designing a system that must answer single-hop, multi-hop, and temporal reasoning questions over conversation history

## Key Technique

AMA replaces monolithic memory stores with a **hierarchical three-tier memory** managed by four specialized agents. The **Constructor** decomposes each dialogue turn into: (1) **Fact Knowledge** -- atomic subject-verb-object propositions extracted via sentence pattern analysis (S-V, S-V-O, S-V-C, S-V-O-O, S-V-O-C), each tagged with timestamp, speaker, and turn references; (2) **Raw Text** -- the original utterance preserved verbatim for exact-match retrieval; and (3) **Episode Memory** -- high-level summaries synthesized when topic shifts, explicit summarization requests, or context saturation are detected. All entries are embedded as dense vectors (via a text encoder) and indexed in FAISS for sublinear similarity search, with structured metadata stored in SQLite.

The **Retriever** performs adaptive query routing by first rewriting context-dependent queries into self-contained forms, then producing a binary intent vector `[b_fine, b_abs, b_event, b_atomic]`. This intent maps to memory tiers via strict priority: `b_fine=1` routes to Raw Text, `b_abs=1 OR b_event=1` routes to Episode Memory, and the default routes to Fact Knowledge. Top-K results are retrieved by cosine similarity with a dynamic floor to prevent under-retrieval.

The **Judge** then performs dual-phase verification: first filtering by information density (triggering iterative re-retrieval up to K_r rounds if evidence is insufficient), then detecting logical conflicts between retrieved content and the current input. On conflict, the Judge invokes the **Refresher**, which performs targeted updates -- modifying contradicted entries to align with current state, or deleting entries only when the user explicitly requests forgetting or entries exceed retention limits. This loop ensures memory stays consistent without unchecked accumulation of stale data.

## Step-by-Step Workflow

1. **Define the memory schema.** Create three storage collections -- `fact_memory`, `raw_memory`, and `episode_memory` -- each with fields for content, embedding vector, timestamp, speaker ID, source turn references, and a session identifier. Use SQLite for structured metadata and FAISS (or a vector DB like ChromaDB/Qdrant) for embedding indices.

2. **Implement the Constructor agent.** For each incoming dialogue turn: (a) store the raw utterance in `raw_memory`; (b) extract atomic facts by prompting an LLM to decompose the utterance into independent S-V-O propositions, storing each in `fact_memory`; (c) check episode triggers (topic shift detected via embedding distance from recent turns > threshold, explicit summarization request, or turn count since last episode exceeds a window like 10 turns) and generate an episode summary covering the accumulated segment if triggered.

3. **Embed and index all entries.** Encode each memory entry's content using a sentence embedding model (e.g., `text-embedding-3-small` or `all-MiniLM-L6-v2`). Insert the vector into the FAISS index with a mapping back to the SQLite row ID.

4. **Implement the Retriever agent with intent-based routing.** On receiving a query: (a) rewrite it to resolve pronouns and references using recent conversation context; (b) classify query intent into the binary vector `[b_fine, b_abs, b_event, b_atomic]` by prompting the LLM with the rewritten query and definitions of each dimension; (c) select the target memory tier based on priority (Raw > Episode > Fact); (d) retrieve top-K entries by cosine similarity with `K = max(K_dynamic, K_minimum)` where K_minimum is a floor (e.g., 3).

5. **Implement the Judge agent for relevance filtering.** Score each retrieved entry for relevance to the rewritten query using an LLM call. If fewer than a threshold number of entries pass the relevance filter, send a "Retry" signal to the Retriever with an expanded query or increased K, up to K_r=2 retry rounds.

6. **Implement the Judge's conflict detection.** After relevance filtering, compare the filtered entries against the current user input for logical contradictions (e.g., user previously said "I live in NYC" but now says "I moved to SF"). Use an LLM prompt that explicitly asks: "Do any of these memory entries contradict the current statement? List conflicting pairs." Collect the conflict set `C_err`.

7. **Implement the Refresher agent.** When `C_err` is non-empty: (a) for each conflicting entry, determine whether to UPDATE (modify the entry's content to reflect the new state) or DELETE (only if the user explicitly asks to forget something, or the entry exceeds a configured max retention age); (b) execute the update/delete operations on both SQLite and FAISS; (c) return the cleaned memory set to the downstream response generator.

8. **Wire the pipeline together.** The response generator receives the verified, conflict-free memory entries as context alongside the current query. Construct the prompt as: system instructions + retrieved memory entries (formatted with timestamps and sources) + current query. This replaces injecting the full conversation history, yielding ~80% token savings.

9. **Add lifecycle management.** Implement periodic background maintenance: prune entries older than the max retention window, re-cluster episode memories when they exceed a count threshold, and rebuild FAISS indices after significant deletions to maintain search quality.

10. **Instrument and monitor.** Log retrieval routes chosen (which tier was hit), retry counts, conflict detection rates, and token counts per query. These metrics reveal whether the intent classifier is routing well and whether memory staleness is accumulating.

## Concrete Examples

**Example 1: Building a personal assistant memory system**

User: "I want my chatbot to remember user preferences across sessions and handle updates when users change their minds."

Approach:
1. Create the three-tier memory schema in SQLite + FAISS:
```python
# Schema for fact_memory table
CREATE TABLE fact_memory (
    id INTEGER PRIMARY KEY,
    content TEXT NOT NULL,          -- e.g., "User prefers dark mode"
    speaker TEXT,                   -- "user" or "assistant"
    session_id TEXT,
    turn_index INTEGER,
    created_at TIMESTAMP,
    embedding_id INTEGER            -- maps to FAISS index position
);
# Similar tables for raw_memory and episode_memory
```

2. Implement the Constructor with LLM-based fact extraction:
```python
CONSTRUCTOR_PROMPT = """Extract atomic facts from this dialogue turn.
Each fact must be a single independent statement in S-V-O form.
Turn: "{utterance}"
Speaker: {speaker}

Output as JSON array: ["fact1", "fact2", ...]"""

def construct_memories(utterance, speaker, session_id, turn_idx):
    # Store raw text
    raw_id = store_raw(utterance, speaker, session_id, turn_idx)
    # Extract and store atomic facts
    facts = llm_call(CONSTRUCTOR_PROMPT.format(...))
    for fact in json.loads(facts):
        store_fact(fact, speaker, session_id, turn_idx)
    # Check episode trigger
    if should_create_episode(session_id, turn_idx):
        summary = generate_episode_summary(session_id, last_episode_turn, turn_idx)
        store_episode(summary, session_id, last_episode_turn, turn_idx)
```

3. Implement the Retriever with intent routing:
```python
INTENT_PROMPT = """Classify this query's intent as a JSON object with boolean fields:
- b_fine: needs exact wording or specific phrasing
- b_abs: needs high-level summary or theme
- b_event: spans multiple time periods or sessions
- b_atomic: needs a specific isolated fact

Query: "{query}"
Output: {{"b_fine": bool, "b_abs": bool, "b_event": bool, "b_atomic": bool}}"""

def retrieve(query, context):
    rewritten = rewrite_query(query, context)
    intent = json.loads(llm_call(INTENT_PROMPT.format(query=rewritten)))
    if intent["b_fine"]:
        return search_raw_memory(rewritten, top_k=5)
    elif intent["b_abs"] or intent["b_event"]:
        return search_episode_memory(rewritten, top_k=5)
    else:
        return search_fact_memory(rewritten, top_k=5)
```

4. Implement the Judge with conflict detection:
```python
JUDGE_PROMPT = """Given these memory entries and the current user statement,
identify any logical contradictions.

Memory entries:
{entries}

Current statement: "{current}"

Output JSON: {{"relevant": ["id1", ...], "conflicts": [{{"memory_id": "...", "reason": "..."}}]}}"""

def judge(entries, current_input, retry_count=0):
    result = json.loads(llm_call(JUDGE_PROMPT.format(...)))
    if len(result["relevant"]) < 2 and retry_count < 2:
        return "RETRY"  # Retriever expands search
    if result["conflicts"]:
        return "REFRESH", result["conflicts"]
    return "PASS", [e for e in entries if e.id in result["relevant"]]
```

5. The Refresher updates conflicting entries:
```python
def refresh(conflicts, current_input):
    for conflict in conflicts:
        entry = load_memory(conflict["memory_id"])
        updated_content = llm_call(f"Update this fact: '{entry.content}' "
                                   f"to reflect: '{current_input}'. "
                                   f"Return only the updated fact.")
        update_memory(entry.id, updated_content)
        reindex_embedding(entry.id, updated_content)
```

Output: A memory-augmented chatbot that stores "User prefers dark mode" as a fact, and when the user later says "Actually, switch me to light mode," the Judge detects the conflict with the stored preference, the Refresher updates it to "User prefers light mode," and subsequent queries return the correct current preference.

---

**Example 2: Implementing memory for a customer support agent**

User: "Build a support agent that tracks customer issue history and doesn't give contradictory answers when case details change."

Approach:
1. Map the three memory tiers to support data: Raw Text = verbatim customer messages, Fact Knowledge = extracted ticket details (status, product, issue type), Episode Memory = case summaries generated at resolution or escalation points.

2. Configure the intent router priorities:
   - Customer asks "What exactly did I say about the billing issue?" -> `b_fine=1` -> Raw Text
   - Customer asks "Give me an overview of my support history" -> `b_abs=1` -> Episode Memory
   - Agent needs "What is the customer's current plan?" -> default -> Fact Knowledge

3. The Judge catches conflicts like: stored fact says "Customer plan: Basic" but a recent ticket shows an upgrade to "Customer plan: Premium". The Refresher updates the fact entry.

4. Token savings: instead of injecting 50+ ticket transcripts (potentially 100K+ tokens) into context, the system retrieves 3-8 targeted memory entries (under 2K tokens).

Output: A support agent that answers "You upgraded to Premium on Jan 15th" instead of hallucinating stale data, while using a fraction of the context budget.

---

**Example 3: Adding memory refresh to an existing RAG pipeline**

User: "My RAG system returns outdated information because the knowledge base has conflicting entries. How do I fix this?"

Approach:
1. Retrofit the Judge-Refresher loop onto the existing pipeline. After the retriever returns results, add a Judge step that checks for temporal conflicts:
```python
def add_consistency_check(retrieved_docs, query):
    # Group by entity/topic
    grouped = group_by_entity(retrieved_docs)
    for entity, docs in grouped.items():
        # Sort by timestamp, check if later docs contradict earlier ones
        conflicts = detect_temporal_conflicts(sorted(docs, key=lambda d: d.timestamp))
        if conflicts:
            # Keep only the most recent consistent version
            resolve_conflicts(conflicts, strategy="prefer_latest")
    return filtered_docs
```

2. Add a background Refresher job that periodically scans for cross-document contradictions and either merges, updates, or tombstones outdated entries.

Output: A RAG pipeline that no longer returns "The API rate limit is 100 req/min" alongside "The API rate limit was increased to 500 req/min" -- it consistently returns the latest verified fact.

## Best Practices

- **Do:** Extract facts as atomic S-V-O propositions, not multi-clause sentences. Atomic granularity enables precise conflict detection -- "User lives in NYC" directly contradicts "User lives in SF," but compound sentences obscure this.
- **Do:** Set a retry limit (K_r=2) on the Judge's iterative retrieval loop. The paper found diminishing returns beyond 2 rounds and sharply increasing token cost.
- **Do:** Preserve raw text alongside extracted facts. Some queries need exact wording (e.g., "What did the user say about..."), and paraphrased facts lose this fidelity.
- **Do:** Tag every memory entry with a timestamp and source turn reference. Temporal metadata is essential for the Refresher to determine which entry is current during conflict resolution.
- **Avoid:** Deleting memory entries aggressively. The Refresher should prefer UPDATE over DELETE. Only delete on explicit user request or when entries exceed a hard retention limit. Over-deletion causes information loss.
- **Avoid:** Routing all queries to a single memory tier. The intent classification step is critical -- skipping it and always searching facts (or always searching raw text) degrades accuracy on queries that need a different granularity.

## Error Handling

| Problem | Symptom | Resolution |
|---------|---------|------------|
| Intent classifier returns all-false vector | Query routes to Fact Memory by default, but results are irrelevant | Fall back to searching all three tiers and merging top-K results across them |
| Judge enters infinite retry loop | Retry count exceeds K_r with no relevant results found | After K_r retries, return the best available results with a low-confidence flag so the response generator can hedge |
| Refresher updates create new conflicts | Updating entry A to resolve conflict with B now contradicts entry C | Run conflict detection on the updated entry before committing; if new conflicts appear, batch-resolve the full conflict cluster |
| FAISS index drift after many deletions | Retrieval quality degrades as deleted vectors leave gaps | Schedule periodic index rebuilds (e.g., after every 1000 deletions) using `faiss.IndexIVFFlat.reset()` and re-add active vectors |
| Episode trigger fires too frequently | Memory fills with redundant episode summaries | Increase the topic-shift embedding distance threshold or the minimum turn count between episodes |

## Limitations

- **LLM dependency for fact extraction:** The Constructor relies on LLM calls to decompose utterances into atomic facts, adding latency and cost per ingestion. For high-throughput systems (>100 messages/second), consider a lighter NLP pipeline for fact extraction and reserve LLM calls for the Judge.
- **Not suited for non-textual memory:** The three-tier design assumes text-based conversational data. Extending to multimodal memory (images, audio, structured database records) requires additional Constructor logic not covered by this framework.
- **Conflict detection is heuristic:** The Judge uses LLM-based reasoning to detect contradictions, which can miss subtle semantic conflicts or flag false positives on nuanced statements. Critical applications should add human-in-the-loop review for Refresher actions.
- **Cold start problem:** The intent classifier and episode trigger thresholds benefit from tuning on domain-specific data. Out-of-the-box defaults may misroute queries until calibrated.
- **Single-user assumption:** The paper evaluates on single-user conversation histories. Multi-user or multi-tenant memory systems need additional access control and per-user partitioning not addressed here.

## Reference

[AMA: Adaptive Memory via Multi-Agent Collaboration](https://arxiv.org/abs/2601.20352v2) -- Huang et al., 2026. Focus on Section 3 (framework architecture), Figure 2 (agent interaction flow), and Table 2 (ablation study showing the Refresher's impact on knowledge-update accuracy: 0.897 vs. 0.568 without it).