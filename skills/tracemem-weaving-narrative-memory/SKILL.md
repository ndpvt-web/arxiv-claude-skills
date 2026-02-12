---
name: "tracemem-weaving-narrative-memory"
description: "Build structured narrative memory systems from conversational traces using TraceMem's three-stage pipeline (segmentation, consolidation, clustering). Use when asked to: 'build a memory system for a chatbot', 'implement long-term conversation memory', 'create user memory cards from chat history', 'add persistent memory to an LLM agent', 'organize conversation history into structured narratives', 'implement episodic memory with clustering'."
---

# TraceMem: Weaving Narrative Memory Schemata from Conversational Traces

This skill enables Claude to implement structured, long-term memory systems for LLM-based agents and chatbots using the TraceMem framework. Rather than treating conversation history as flat retrieval snippets, TraceMem constructs coherent narrative memory schemata through a cognitively-inspired three-stage pipeline: deductive topic segmentation, synaptic memory consolidation, and hierarchical clustering into narrative threads. The result is structured "User Memory Cards" that enable multi-hop reasoning, temporal understanding, and deep user personalization.

## When to Use

- When building a chatbot or agent that needs to remember users across sessions and reason over long interaction histories
- When implementing a memory layer for an LLM application that goes beyond naive RAG retrieval
- When a user asks to organize messy conversation logs into structured, queryable user profiles
- When designing a system that must answer multi-hop questions about past conversations (e.g., "What changed about the user's job situation between March and June?")
- When replacing a flat vector-store memory with something that preserves narrative coherence and temporal ordering
- When implementing personalization that requires understanding evolving user traits, not just static facts

## Key Technique

**The core insight:** Most memory systems treat conversation turns as independent documents for embedding and retrieval. TraceMem instead recognizes that conversations form narratives with episode boundaries, character development, and thematic arcs. By explicitly constructing these narrative structures, the system achieves state-of-the-art performance on multi-hop and temporal reasoning tasks, even surpassing full-context baselines that have access to the entire conversation history.

**The three-stage pipeline mirrors human memory consolidation.** Stage 1 (Short-term Memory Processing) uses deductive topic segmentation to split raw dialogue into coherent episodes, classifying each utterance as either a Topic Change (TC) or Topic Development (TD) by examining bidirectional context. Stage 2 (Synaptic Memory Consolidation) summarizes each episode into an episodic memory, then distills user-specific "experience traces" by filtering for concrete biographical facts and discarding generic observations. Stage 3 (Systems Memory Consolidation) applies two-stage hierarchical clustering (PCA for dimensionality reduction, UMAP for manifold learning, HDBSCAN for density-based clustering, KNN for noise reassignment) to organize traces into thematic narrative threads, which are then encapsulated into structured User Memory Cards with theme titles, topical sections, and thread entries with unique IDs.

**Memory retrieval uses an agentic search mechanism** that operates in three concurrent paths: (1) semantic similarity search over a vector database of episodic memories (top-K=10), (2) direct memory card selection based on user/topic cues in the query, and (3) thread-level extraction where the agent inspects card structure and retrieves specific thread content by ID. This multi-path approach provides both broad recall and precise narrative targeting.

## Step-by-Step Workflow

### 1. Ingest Raw Conversation Data
Parse dialogue history into a structured format with fields: `speaker`, `utterance`, `timestamp`, `session_id`. Preserve temporal ordering. If visual/multimodal metadata exists, include it as supplementary context per utterance.

### 2. Perform Deductive Topic Segmentation
For each utterance, classify its discourse intent as Topic Change (TC) or Topic Development (TD) by examining its bidirectional relationship with preceding and following context. Use an LLM prompt that outputs structured XML tags. Group contiguous TD utterances under the same episode; each TC marks a new episode boundary.

```python
# Segmentation prompt structure
SEGMENT_PROMPT = """Analyze this utterance in context of the surrounding dialogue.
Classify as:
<intent>TC</intent> if it introduces a new subject, activity, or domain shift
<intent>TD</intent> if it directly responds to or elaborates on the current topic

Preceding context: {prev_utterances}
Current utterance: {utterance}
Following context: {next_utterances}
"""
```

### 3. Extract Semantic Representations per Episode
For each episode, generate a structured set of exhaustive factual elements derived from the dialogue. Output as a flat list of atomic facts (subject-predicate-object triples or short declarative statements). This creates a "fact-based space" for each episode.

### 4. Summarize Episodes into Episodic Memories
Condense each episode into a concise episodic summary that captures the key events, decisions, emotional states, and outcomes. Store these summaries in a vector database with their embeddings and timestamps for later retrieval.

### 5. Distill User Experience Traces
Apply a rule-based filter to extract only concrete biographical facts from episodic summaries: personal experiences, contextual details, preferences, relationships, life events. Discard generic observations, small talk, and system-side information. Each trace is a user-specific factual statement with temporal grounding.

### 6. Cluster Traces into Narrative Threads (Two-Stage Hierarchical Clustering)
- **Coarse clustering**: Embed all traces, reduce dimensionality with PCA, apply UMAP (n_neighbors=10) then HDBSCAN (min_cluster_size=5) to identify broad topic clusters
- **Fine clustering**: Within each topic cluster, apply UMAP (n_neighbors=2) and HDBSCAN (min_cluster_size=2) to identify specific narrative threads
- **Noise handling**: Reassign noise points to nearest cluster using KNN classifier

### 7. Construct User Memory Cards
For each user, build a hierarchical memory card:
```json
{
  "user_id": "user_123",
  "card": {
    "theme": "Career transition from engineering to product management",
    "topics": [
      {
        "title": "Engineering background and growing dissatisfaction",
        "threads": [
          {
            "thread_id": "t_001",
            "title": "Early career at TechCorp",
            "summary": "Worked as backend engineer for 3 years, led migration project...",
            "time_range": ["2024-01", "2024-03"],
            "trace_ids": ["tr_01", "tr_02", "tr_05"]
          }
        ]
      }
    ]
  }
}
```

### 8. Implement the Agentic Search Mechanism
When a query arrives, execute three retrieval paths concurrently:
1. **Episodic retrieval**: Embed the query, search the vector DB for top-10 similar episodic memories
2. **Card selection**: Use the LLM to identify which memory cards are relevant based on user/topic cues in the query
3. **Thread extraction**: From selected cards, have the agent examine topic titles and thread entries, then retrieve full thread content by thread_id from the vector store

Merge results from all three paths, deduplicate, and order by relevance and recency.

### 9. Generate Context-Aware Responses
Feed the retrieved memory context (episodic memories + narrative threads) into the LLM alongside the current query. The narrative structure provides temporal grounding and causal chains that enable multi-hop reasoning without requiring the full conversation history.

### 10. Continuously Update the Memory Pipeline
After each new conversation session, run stages 1-7 incrementally: segment new dialogue, extract traces, re-cluster (or assign new traces to existing clusters), and update memory cards. Memory cards evolve over time as new threads emerge and existing ones grow.

## Concrete Examples

**Example 1: Building a Therapist Bot Memory System**

User: "I'm building a mental health chatbot. I need it to remember what users discuss across sessions -- their stressors, coping strategies, and how their situation evolves over time."

Approach:
1. Define the data schema: each session is a list of `{speaker, utterance, timestamp}` turns
2. Implement topic segmentation to split sessions into episodes (e.g., "discussing work stress" vs "talking about exercise routine")
3. Extract episodic memories: "User reported increased anxiety about upcoming review (Session 5, Oct 12)"
4. Distill traces focused on mental health: stressors, coping mechanisms, mood indicators, life events
5. Cluster into narrative threads: "Work-related anxiety arc", "Exercise and wellness journey", "Family relationship dynamics"
6. Build memory cards per user with temporal thread structure
7. At query time, use agentic search to pull relevant threads -- e.g., when user mentions "my boss" retrieve the full "Work-related anxiety" thread for continuity

Output structure:
```python
class TracememPipeline:
    def __init__(self, llm_client, embedding_model, vector_db):
        self.segmenter = DeductiveSegmenter(llm_client)
        self.consolidator = SynapticConsolidator(llm_client)
        self.clusterer = NarrativeClusterer(
            umap_neighbors_coarse=10, hdbscan_min_coarse=5,
            umap_neighbors_fine=2, hdbscan_min_fine=2
        )
        self.card_builder = MemoryCardBuilder(llm_client)
        self.searcher = AgenticSearcher(vector_db, llm_client)

    def ingest_session(self, user_id: str, session: list[dict]):
        episodes = self.segmenter.segment(session)
        episodic_memories = self.consolidator.summarize(episodes)
        traces = self.consolidator.distill_traces(episodic_memories)
        self.clusterer.update(user_id, traces)
        self.card_builder.rebuild(user_id, self.clusterer.get_threads(user_id))

    def query(self, user_id: str, question: str) -> str:
        context = self.searcher.search(user_id, question, top_k=10)
        return self.llm.generate(question, memory_context=context)
```

**Example 2: Adding Persistent Memory to a Customer Support Agent**

User: "Our support agent forgets everything between tickets. I want it to know that when customer X calls about 'the billing issue', it refers to the overcharge dispute they opened 3 months ago, not a new problem."

Approach:
1. Ingest all past ticket transcripts per customer as conversation sessions
2. Segment each ticket into episodes (problem description, troubleshooting steps, resolution)
3. Extract traces: "Customer disputed $450 charge on invoice #1234 (Jan 15)", "Partial refund of $200 issued (Jan 22)", "Customer reported charge reappeared on Feb statement (Feb 10)"
4. Cluster into narrative threads: the billing dispute becomes a single coherent thread with temporal progression
5. Build customer memory card with threads like "Recurring billing overcharge", "Product onboarding issues", "Feature requests"
6. When new ticket arrives mentioning "billing issue", agentic search retrieves the full billing thread by card selection + thread extraction, giving the agent complete context

Output -- Memory Card:
```
Customer: Acme Corp (ID: cust_789)
Theme: Ongoing service relationship with billing complications

Topic: Billing Disputes
  Thread: Invoice #1234 overcharge [t_billing_01]
    - Jan 15: Reported $450 overcharge on monthly invoice
    - Jan 22: Support issued $200 partial refund, escalated remainder
    - Feb 10: Charge reappeared on February statement
    - Feb 14: Full credit applied, billing team notified of system bug
    Time range: 2025-01 to 2025-02

Topic: Product Usage
  Thread: Dashboard onboarding [t_onboard_01]
    - Dec 3: Requested help setting up analytics dashboard
    - Dec 10: Completed setup, asked about custom report exports
    Time range: 2024-12
```

**Example 3: Implementing Memory for a Personal AI Assistant**

User: "I want my AI assistant to build a profile of me over time from our conversations and use it to personalize responses."

Approach:
1. After each conversation, run the segmentation + consolidation pipeline
2. Distill traces focused on user preferences, habits, goals, relationships, schedule patterns
3. Cluster into narrative themes: "Career goals", "Health and fitness", "Travel interests", "Family life"
4. Memory card acts as a living user profile that evolves with each interaction
5. On every new query, run agentic search to pull relevant threads -- asking about restaurants triggers "Food preferences" thread; mentioning a trip triggers "Travel interests"
6. Use retrieved threads to personalize: recommend restaurants matching known preferences, reference past travel experiences

Agentic search flow for query "Where should I eat tonight?":
```
Path 1 (Episodic): Retrieves recent food-related memories
  -> "User mentioned trying Korean food last week and loved it"
  -> "User said they're avoiding dairy this month"

Path 2 (Card Selection): Selects "Food & Dining Preferences" card
Path 3 (Thread Extraction): Retrieves threads:
  -> "Dietary restrictions" thread: lactose sensitivity, no dairy this month
  -> "Cuisine exploration" thread: recently into Korean, previously loved Thai

Merged context enables personalized recommendation grounded in user history.
```

## Best Practices

- **Do:** Use bidirectional context for topic segmentation -- examining both what came before AND after an utterance produces more accurate episode boundaries than forward-only approaches
- **Do:** Filter traces aggressively during distillation -- retain only concrete biographical facts and discard generic conversational filler; this dramatically improves clustering quality
- **Do:** Use the two-stage clustering with different hyperparameters at each level (coarse: n_neighbors=10, min_cluster=5; fine: n_neighbors=2, min_cluster=2) to capture both broad themes and specific narrative threads
- **Do:** Assign thread IDs and maintain them as stable references -- this enables the agentic search to pinpoint exact narrative segments without scanning all content
- **Avoid:** Treating the memory card as a static document -- it must be rebuilt or incrementally updated as new conversations arrive to reflect evolving user narratives
- **Avoid:** Relying solely on vector similarity search -- the card selection + thread extraction paths are what enable multi-hop and temporal reasoning that pure embedding search cannot achieve
- **Avoid:** Skipping the noise reassignment step in clustering -- HDBSCAN will classify low-density traces as noise, and KNN reassignment ensures no user information is lost

## Error Handling

- **Sparse conversation data**: When a user has very few interactions, HDBSCAN may classify everything as noise. Fall back to flat episodic memory retrieval until sufficient traces accumulate (minimum ~20 traces for meaningful clustering).
- **Topic segmentation ambiguity**: When the LLM disagrees on TC vs TD classification for borderline utterances, bias toward TD to avoid over-fragmenting episodes. Smaller episodes lose context; larger episodes are more robust.
- **Clustering drift over time**: As the user's life changes, old clusters may become irrelevant. Implement a recency weight in the clustering step, or periodically archive stale threads (no new traces in 90+ days) to keep memory cards focused.
- **Memory card bloat**: For very active users, cards can grow unwieldy. Cap at ~15-20 threads per card and merge closely related threads, or split into multiple thematic cards.
- **Agentic search returning too much context**: If merged retrieval results exceed the LLM's useful context window, rank by a combination of semantic relevance, recency, and thread coherence, then truncate.

## Limitations

- Requires an LLM for segmentation and summarization at each pipeline stage, making it more computationally expensive than simple embedding-and-retrieve approaches. Budget ~3-5 LLM calls per conversation session for the consolidation pipeline.
- Clustering quality depends on having a meaningful volume of traces per user. Users with fewer than ~10 conversation sessions will not produce coherent narrative threads; use flat episodic memory as a fallback.
- The deductive segmentation assumes conversations have topical structure. Highly fragmented, stream-of-consciousness dialogue (e.g., some social chat) may not segment cleanly.
- Memory cards are user-specific. Cross-user reasoning (e.g., "Which of my users are interested in X?") requires additional infrastructure beyond what TraceMem provides.
- The pipeline is designed for text-primary conversations. While it supports multimodal metadata, the clustering and narrative construction are fundamentally text-embedding-based.

## Reference

**Paper:** [TraceMem: Weaving Narrative Memory Schemata from User Conversational Traces](https://arxiv.org/abs/2602.09712v1) (Shu et al., 2026)
**Code:** [github.com/YimingShu-teay/TraceMem](https://github.com/YimingShu-teay/TraceMem)
**Key takeaway:** Look at Section 3 for the full pipeline architecture, Section 3.3 for the two-stage clustering parameters (UMAP/HDBSCAN), Section 3.4 for the agentic search mechanism, and the Appendix for complete prompt templates used at each stage.