---
name: "tracemem-weaving-narrative-memory"
description: "Build structured narrative memory systems that consolidate long conversation histories into searchable, hierarchical memory schemata. Use when asked to: 'build a memory system for my chatbot', 'track user preferences across sessions', 'implement long-term conversation memory', 'organize dialogue history into themes', 'add persistent memory to an LLM agent', 'cluster conversation topics over time'."
---

# TraceMem: Weaving Narrative Memory Schemata from Conversational Traces

This skill enables Claude to design and implement **structured narrative memory systems** that transform sprawling conversation histories into organized, searchable memory cards. Based on TraceMem's three-stage cognitive pipeline — short-term segmentation, synaptic consolidation, and systems-level clustering — the technique converts disjointed dialogue turns into hierarchical narrative threads grouped under unifying themes. This is the architecture you reach for when an LLM agent needs to remember, reason over, and retrieve information from hundreds or thousands of past conversation turns.

## When to Use

- When building a chatbot or agent that must maintain coherent memory across many sessions (e.g., a personal assistant that remembers user preferences, life events, and ongoing projects)
- When implementing a retrieval system over dialogue history that must support multi-hop reasoning ("What changed about the user's job since they mentioned relocating?")
- When a user asks to organize unstructured conversation logs into themed, time-evolving summaries
- When designing a memory layer for an LLM application where naive RAG over raw turns produces poor recall on temporal or cross-session queries
- When building user profile systems that must evolve over time from conversational data rather than explicit form input
- When replacing a flat "append every message" memory with something that scales beyond a single context window

## Key Technique

TraceMem draws from cognitive science's distinction between episodic memory (specific events) and semantic memory (general knowledge). Raw dialogue is first segmented into **episodes** — coherent topical blocks — using deductive topic segmentation. Each utterance is classified as either a **Topic Change** (introduces a new subject) or **Topic Development** (continues the current thread) by examining bidirectional context. This prevents both over-fragmentation and topic drift. From each episode, the system extracts two outputs: an **episode summary** (what happened) and **semantic representations** (structured facts in XML format like preferences, biographical details, opinions).

The consolidation stage distills episode summaries and semantics into **experience traces** — atomic biographical or preferential facts about the user (e.g., "User relocated from NYC to Austin in March 2025 for a remote role"). A rule-based filter removes conversational noise, retaining only user-specific information. These traces are the building blocks of long-term memory.

The final stage applies **two-level hierarchical clustering** (PCA → UMAP → HDBSCAN → KNN noise reassignment) to organize traces into narrative threads. First, traces are clustered at the **topic level** (broad themes like "Career", "Health", "Hobbies"). Within each topic cluster, a second pass groups traces into **thread level** clusters — fine-grained, chronologically ordered narrative arcs (e.g., under "Career": "Job search journey", "Onboarding at new company"). Each thread gets a descriptive title, summary, and unique ID. The entire structure is packaged into a **User Memory Card** — a hierarchical document with theme → topic → thread layers that serves as the user's narrative memory schema.

## Step-by-Step Workflow

1. **Ingest dialogue turns** into a buffer. Each turn should carry a timestamp, speaker role, and content. Store raw turns in a vector database (ChromaDB or similar) with embeddings (e.g., `text-embedding-3-small`) for later episodic retrieval.

2. **Segment into episodes** using deductive topic classification. For each new utterance, evaluate whether it represents a Topic Change or Topic Development by examining the previous utterance, current utterance, and next utterance (lookahead). Mark Topic Change points as episode boundaries. Accumulate turns between boundaries into episode objects.

3. **Extract semantic representations** from each episode. Prompt an LLM with the episode text using XML-structured output to pull out: user preferences, biographical facts, opinions, goals, and relationships. Store these as typed key-value pairs alongside the episode.

4. **Summarize episodes into episodic memories**. For each completed episode, generate a concise summary capturing the key events and outcomes. This becomes the episode's searchable representation in the vector store.

5. **Distill experience traces** from episode summaries and semantic extractions. Apply a rule-based filter that retains only user-specific, biographical, or preferential facts. Discard generic conversational filler, assistant-side reasoning, and duplicate information. Each trace should be a single atomic statement.

6. **Embed all experience traces** using the same embedding model. Store embeddings in a matrix for clustering.

7. **Cluster at the topic level**. Apply dimensionality reduction (PCA → UMAP) then HDBSCAN with parameters `min_neighbors=10, min_cluster_size=5`. Reassign noise points to nearest clusters via KNN. This produces broad thematic groups.

8. **Cluster at the thread level** within each topic cluster. Run a second HDBSCAN pass with tighter parameters (`min_neighbors=2, min_cluster_size=2`). Sort traces within each thread chronologically. Generate a descriptive title and summary for each thread.

9. **Assemble the User Memory Card**. Structure it as a three-level hierarchy: Theme Title → Topic Sections → Thread Entries (each with title, summary, trace IDs, and time range). Serialize as JSON or markdown for storage and LLM consumption.

10. **Implement agentic search for retrieval**. On a new query: (a) retrieve top-K=10 episodic memories from the vector store via embedding similarity; (b) identify which Memory Card topics/threads are relevant based on the query and user context; (c) fetch full thread content by trace IDs from the vector store. Concatenate episodic fragments and narrative thread context into the LLM prompt for grounded reasoning.

## Concrete Examples

**Example 1: Personal Assistant Memory System**

```
User: "Build a memory system for my personal assistant chatbot that
remembers user details across sessions."

Approach:
1. Create a ConversationBuffer class that accumulates turns with timestamps.
2. Implement TopicSegmenter that classifies each turn as TC/TD:
   - Prompt: "Given previous turn: '{prev}', current turn: '{curr}',
     next turn: '{next}', classify current as TOPIC_CHANGE or TOPIC_DEVELOPMENT."
   - Split on TC boundaries into Episode objects.
3. For each Episode, run SemanticExtractor:
   - Prompt: "Extract user facts from this episode in XML:
     <facts><preference>..</preference><bio>..</bio><goal>..</goal></facts>"
4. Summarize each episode into a 2-3 sentence episodic memory.
5. Filter traces: keep only user-specific atomic facts.
6. After every N episodes (e.g., 10), run the clustering pipeline:
   - Embed traces → PCA(50) → UMAP(2D) → HDBSCAN → KNN reassign
   - Topic-level clusters → Thread-level sub-clusters
7. Build MemoryCard JSON:

{
  "user_id": "u_12345",
  "updated_at": "2025-06-15T10:00:00Z",
  "themes": [
    {
      "title": "Career Transition",
      "topics": [
        {
          "name": "Job Search",
          "threads": [
            {
              "id": "t_001",
              "title": "Transition from Finance to Tech",
              "summary": "User left investment banking in Jan 2025, completed a bootcamp, now interviewing at startups.",
              "time_range": ["2025-01-10", "2025-06-01"],
              "trace_ids": ["tr_012", "tr_045", "tr_078"]
            }
          ]
        }
      ]
    }
  ]
}

8. On new query "What roles is the user targeting?":
   - Vector search returns episodic hits about interview prep
   - Memory card lookup identifies thread t_001
   - Full thread context injected into prompt → grounded answer
```

**Example 2: Organizing Raw Chat Logs into Themed Summaries**

```
User: "I have 500 conversation turns exported from a support chat.
Organize them into themed narrative summaries."

Approach:
1. Parse the export into Turn objects: {timestamp, role, text}.
2. Run deductive segmentation across all 500 turns:
   - Batch classify TC/TD in sliding windows of 3 turns.
   - Result: ~40-60 episodes depending on topic diversity.
3. Summarize each episode (batch LLM calls for efficiency).
4. Extract experience traces — here "user-specific" means the
   customer's issues, preferences, and resolutions.
5. Embed and cluster:
   - Topic-level might yield: "Billing Issues", "Product Setup",
     "Feature Requests", "Account Recovery"
   - Thread-level under "Billing Issues": "Disputed charge in March",
     "Subscription downgrade process", "Refund for outage"
6. Output a structured report:

## Billing Issues
### Disputed Charge (March 2025)
Customer reported unauthorized $49.99 charge on 3/12. Escalated to
billing team on 3/14. Resolved with full refund on 3/18. Customer
confirmed satisfaction.

### Subscription Downgrade (April 2025)
Customer requested move from Pro to Basic plan. Completed on 4/02.
Follow-up on 4/15 confirmed no feature loss concerns.

## Feature Requests
### Dark Mode Support
Mentioned across 3 sessions (Feb, Apr, May). Customer indicated this
is a deciding factor for continued subscription.
```

**Example 3: Adding Narrative Memory to a LangChain Agent**

```
User: "I have a LangChain agent. Add TraceMem-style memory so it
handles long user histories better than ConversationBufferMemory."

Approach:
1. Create a TraceMem memory class implementing BaseMemory:

class TraceMemMemory(BaseMemory):
    def __init__(self, embeddings, vectorstore, llm):
        self.buffer = []          # current session turns
        self.episodes = []        # segmented episodes
        self.traces = []          # extracted experience traces
        self.memory_card = {}     # hierarchical narrative schema
        self.vectorstore = vectorstore
        self.llm = llm

    def save_context(self, inputs, outputs):
        # Append turn, check for episode boundary
        self.buffer.append({"human": inputs, "ai": outputs, "ts": now()})
        if self._is_topic_change():
            episode = self._close_episode()
            self.episodes.append(episode)
            new_traces = self._extract_traces(episode)
            self.traces.extend(new_traces)
            self._index_episode(episode)

    def load_memory_variables(self, inputs):
        query = inputs["input"]
        # Stage 1: episodic retrieval
        episodic = self.vectorstore.similarity_search(query, k=10)
        # Stage 2: memory card thread retrieval
        threads = self._search_memory_card(query)
        return {"history": self._format(episodic, threads)}

    def consolidate(self):
        # Run periodically: cluster traces → rebuild memory card
        embeddings = self._embed_traces()
        topics = self._cluster_topics(embeddings)
        for topic in topics:
            topic.threads = self._cluster_threads(topic)
        self.memory_card = self._build_card(topics)

2. Wire into the agent chain as the memory backend.
3. Call consolidate() on a schedule (e.g., after every 10 episodes
   or at session end) to rebuild the narrative structure.
```

## Best Practices

- **Do:** Use bidirectional context (previous + next turn) for topic segmentation. Single-direction classification misses contextual shifts and produces noisy episode boundaries.
- **Do:** Keep experience traces atomic — one fact per trace. "User moved to Austin and started a new job" should be two traces. Atomic traces cluster more accurately.
- **Do:** Rebuild the memory card incrementally. After new traces arrive, re-cluster only the affected topic clusters rather than the entire trace set.
- **Do:** Store trace IDs in the memory card and full trace content in the vector store. This keeps memory cards compact enough to fit in a single LLM context window.
- **Avoid:** Skipping the noise reassignment step (KNN after HDBSCAN). HDBSCAN labels low-density points as noise, and in conversational data, important but rare traces get discarded without reassignment.
- **Avoid:** Using raw dialogue turns as traces. The rule-based filtering step that removes assistant-side content and conversational filler is critical — without it, clusters become polluted with non-user-specific information.

## Error Handling

| Problem | Cause | Solution |
|---|---|---|
| Episodes too granular (1-2 turns each) | Over-sensitive topic change detection | Add a minimum episode length (e.g., 4 turns). Only accept TC classification if the episode has reached minimum length. |
| Clustering produces one giant cluster | Traces too semantically similar | Increase UMAP `n_neighbors` or reduce HDBSCAN `min_cluster_size`. Consider adding temporal features to embeddings. |
| All traces classified as noise | Too few traces for HDBSCAN | Fall back to simpler clustering (K-Means or agglomerative) when trace count < 20. |
| Memory card too large for context window | Hundreds of threads accumulated | Implement a relevance decay: archive threads with no new traces in the last N sessions. Only load active themes. |
| Agentic search returns irrelevant threads | Query doesn't match thread titles | Add a two-stage retrieval: first embed the query against thread summaries, then fetch trace content for top-matching threads. |

## Limitations

- **Cold start**: The system needs a meaningful volume of conversation (~50+ turns) before clustering produces useful narrative structure. For new users, fall back to raw episodic retrieval.
- **Computational cost**: The full pipeline (segmentation → extraction → embedding → clustering → card generation) involves multiple LLM calls per episode. Not suitable for real-time, per-turn execution — design it as a batch or periodic process.
- **Single-user assumption**: TraceMem builds per-user memory cards. Multi-user shared contexts (e.g., group chats) require modifications to trace extraction to attribute facts to specific participants.
- **Clustering parameter sensitivity**: HDBSCAN parameters (`min_neighbors`, `min_cluster_size`) need tuning based on conversation density. The paper's defaults (10/5 for topics, 2/2 for threads) work for ~600-turn dialogues but may need adjustment for different scales.
- **Language dependency**: Topic segmentation prompts and semantic extraction templates are English-centric. Multilingual deployments need adapted prompts and may see degraded boundary detection.

## Reference

**Paper:** [TraceMem: Weaving Narrative Memory Schemata from User Conversational Traces](https://arxiv.org/abs/2602.09712v1) (Shu et al., 2026)
**Code:** [github.com/YimingShu-teay/TraceMem](https://github.com/YimingShu-teay/TraceMem)
**Key insight:** Look at Section 3.3 (Systems Memory Consolidation) for the two-stage clustering algorithm and Section 3.4 (Agentic Search) for the retrieval mechanism. Table 2 shows TraceMem outperforming full-text baselines by 17% on LoCoMo, with the biggest gains on multi-hop (+15%) and temporal (+12%) reasoning tasks — confirming that narrative structure, not just retrieval, drives comprehension.