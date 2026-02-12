---
name: "es-memeval-benchmarking-conversational-agents"
description: "Build and evaluate long-term memory systems for conversational agents using the ES-MemEval five-capability framework (information extraction, temporal reasoning, conflict detection, abstention, user modeling). Use when: 'evaluate my chatbot memory', 'build long-term user memory for my agent', 'benchmark conversational memory capabilities', 'add personalization memory to my dialogue system', 'detect contradictions in user history', 'implement temporal reasoning over chat sessions'."
---

# ES-MemEval: Building and Evaluating Long-Term Memory for Conversational Agents

This skill enables Claude to design, implement, and evaluate long-term memory systems for conversational agents using the ES-MemEval framework from Chen et al. (WWW 2026). The core insight is that conversational memory must handle five distinct capabilities — information extraction, temporal reasoning, conflict detection, abstention, and user modeling — and that evaluating only factual recall (as most benchmarks do) misses the hardest parts: tracking evolving user states, knowing when to say "I don't know," and resolving contradictions across sessions. This skill applies the paper's methodology to build memory-augmented agents and benchmark them rigorously.

## When to Use

- When building a multi-session chatbot or agent that needs to remember user details across conversations
- When designing a RAG pipeline for dialogue systems and choosing retrieval granularity (turn vs. round vs. session level)
- When evaluating whether a conversational agent hallucinates user information or fabricates memories
- When implementing user profile evolution tracking (e.g., therapy bots, coaching apps, customer support)
- When adding conflict detection to catch when a user's stated preferences or life circumstances have changed
- When building abstention logic so an agent declines to answer rather than guessing about a user's history
- When benchmarking memory capabilities of LLMs in long-context vs. RAG-augmented settings

## Key Technique

**The Five-Capability Memory Framework.** ES-MemEval decomposes conversational memory into five orthogonal capabilities that together determine whether an agent can maintain coherent, personalized long-term interactions: (1) *Information extraction* — identifying key facts scattered within and across sessions, (2) *Temporal reasoning* — inferring chronological order and causal dependencies among events to track how a user's situation evolves, (3) *Conflict detection* — spotting contradictions between what a user said in session 5 versus session 20 and resolving them in favor of the most recent state, (4) *Abstention* — withholding a response when the memory store lacks sufficient information rather than hallucinating, and (5) *User modeling* — inferring latent traits, preferences, and emotional states that the user never stated explicitly.

**Session-Level Retrieval Outperforms Fine-Grained Approaches.** A critical finding is that RAG with session-level retrieval (full conversation chunks indexed via dense embeddings like bge-m3, retrieving top-k=4 sessions from FAISS) consistently outperforms turn-level or round-level retrieval. This is because relevant user information is sparsely distributed across turns — a single turn rarely contains enough context to be useful. Session-level retrieval preserves the conversational flow that makes implicit disclosures interpretable.

**Memory Enables Personalization, But RAG Alone Fails at Temporal Dynamics.** Explicit long-term memory reduces hallucination and enables personalization, but standard RAG struggles with temporal reasoning and evolving user states. Models augmented with RAG show improved factual consistency (F1 gains of 3-15 points) but user modeling scores rarely exceed 20.0 F1 even with retrieval. This means production systems need dedicated temporal indexing and state-tracking layers on top of basic RAG.

## Step-by-Step Workflow

### 1. Define User Profile Schema
Create a structured schema capturing static attributes (demographics, personality, relationships, core beliefs) and dynamic attributes (current emotional state, recent life events, active goals). Store as JSON with timestamps on every dynamic field.

### 2. Implement Session Memory Store
Index completed conversation sessions at the session level using a dense embedding model (e.g., bge-m3, text-embedding-3-small). Store full session transcripts as documents in a vector store (FAISS, Pinecone, pgvector). Each document should carry metadata: `user_id`, `session_id`, `timestamp`, `session_summary`.

### 3. Build the Information Extraction Layer
After each session, run an extraction pass that pulls out explicit facts (names, dates, places, relationships, preferences) and implicit signals (emotional tone shifts, hedged statements suggesting uncertainty, repeated topics suggesting preoccupation). Store extracted facts in a structured memory table keyed by `(user_id, fact_type, timestamp)`.

### 4. Add Temporal Indexing
Maintain a chronological event timeline per user. Each event entry contains: `event_description`, `timestamp`, `source_session_id`, `causal_links` (references to prior events that caused or influenced this one). When a new event is extracted, insert it into the timeline and update causal links by checking temporal proximity and semantic similarity to existing events.

### 5. Implement Conflict Detection
Before serving any fact from memory, run a conflict check: query the memory store for all facts of the same type (e.g., "employment_status") and compare timestamps. If a newer entry contradicts an older one, flag the older entry as superseded. Use a simple rule: most recent explicit statement wins. For implicit signals, require at least two corroborating sessions before updating a belief.

### 6. Build the Abstention Gate
Add a confidence scoring layer between retrieval and generation. If the retrieved context does not contain evidence relevant to the query (measured by retrieval score threshold and semantic overlap with the question), instruct the model to respond with an acknowledgment of uncertainty rather than generating a speculative answer. Calibrate the threshold using held-out QA pairs.

### 7. Construct the User Model
Maintain a living user model document that synthesizes extracted facts, temporal events, and inferred traits. Update it after each session using a structured prompt: given the current user model and the new session transcript, output an updated user model. Track what changed and why.

### 8. Evaluate with the Five-Capability Rubric
For each capability, construct test queries from your dialogue history:
- **Information extraction**: "What is [user]'s [specific fact]?" — measure F1 against ground truth
- **Temporal reasoning**: "Did [event A] happen before or after [event B]?" — measure accuracy
- **Conflict detection**: "What is [user]'s current [attribute] given they previously said X?" — measure whether the system returns the most recent value
- **Abstention**: Ask about facts never disclosed — measure whether the system declines vs. hallucinates
- **User modeling**: "How would you describe [user]'s personality/emotional trajectory?" — score against annotated profiles

### 9. Tune Retrieval Parameters
Experiment with retrieval top-k (start with k=4 sessions), embedding model choice, and whether to prepend session summaries to the generation prompt. Measure across all five capabilities, not just QA accuracy — a configuration that improves extraction may degrade abstention.

### 10. Run Longitudinal Stress Tests
Simulate 20+ sessions with evolving user states (job change, relationship change, mood shifts). Verify that the system correctly tracks the evolution, resolves conflicts, and does not hallucinate outdated information.

## Concrete Examples

**Example 1: Building a Memory-Augmented Therapy Support Bot**

User: "I'm building a mental health support chatbot. Users come back weekly. I need the bot to remember what they said in previous sessions without hallucinating."

Approach:
1. Define user profile schema with fields: `name`, `age`, `presenting_concerns[]`, `coping_strategies[]`, `support_network[]`, `mood_trajectory[]`, `life_events[]`
2. After each session, extract facts and emotional signals:
   ```python
   extraction_prompt = """Given this therapy session transcript, extract:
   - Explicit facts (names, events, dates mentioned)
   - Emotional state indicators (mood words, tone shifts)
   - Life event updates (new events, changes to known situations)
   - Coping strategies mentioned
   Output as JSON with confidence scores (0-1) for each item."""
   ```
3. Index sessions in FAISS at session level with bge-m3 embeddings
4. Before each new session, retrieve top-4 relevant prior sessions and the current user model
5. Add abstention instruction to the system prompt:
   ```
   If the user asks about something you have no record of them mentioning,
   say "I don't recall you mentioning that — could you tell me more?"
   Never guess or fabricate details about the user's life.
   ```

Output — memory-augmented system prompt for session 12:
```
You are a supportive counselor. This is session 12 with Alex.

USER MODEL (updated after session 11):
- Presenting concern: work stress, recently escalated due to team restructuring (session 9)
- Previously mentioned breakup in session 3, reported feeling better by session 7
- Coping: journaling (started session 5), running (mentioned session 8, stopped session 10 due to knee injury)
- Current mood trajectory: improving from session 9 low point

RELEVANT PRIOR SESSIONS: [retrieved session 9, 10, 11, and 7 transcripts]

RULES: Reference prior sessions naturally. If uncertain about a detail, ask rather than assume. Track any new life events for the timeline.
```

**Example 2: Evaluating an Existing Chatbot's Memory Capabilities**

User: "I have a customer support bot with RAG. How do I test whether it actually remembers users correctly across tickets?"

Approach:
1. Generate test scenarios covering all five capabilities:
   ```python
   test_cases = {
       "information_extraction": [
           {"query": "What product did customer #42 purchase last month?",
            "ground_truth": "Pro Plan annual subscription",
            "source_sessions": ["ticket_238", "ticket_241"]},
       ],
       "temporal_reasoning": [
           {"query": "Did customer #42 report the billing issue before or after upgrading?",
            "ground_truth": "after",
            "requires_sessions": ["ticket_241", "ticket_245"]},
       ],
       "conflict_detection": [
           {"query": "What is customer #42's preferred contact method?",
            "ground_truth": "Slack (updated from email in ticket_250)",
            "conflict_sessions": ["ticket_238", "ticket_250"]},
       ],
       "abstention": [
           {"query": "What is customer #42's company size?",
            "ground_truth": "NEVER_DISCLOSED",
            "expected_behavior": "decline_to_answer"},
       ],
       "user_modeling": [
           {"query": "Describe customer #42's satisfaction trajectory.",
            "ground_truth": "Initially satisfied, frustrated after billing error, recovering after resolution",
            "requires_all_sessions": True},
       ],
   }
   ```
2. Run each test case through the bot and score:
   - F1 for extraction and conflict detection
   - Accuracy for temporal reasoning
   - Abstention rate (should be ~100% on NEVER_DISCLOSED items, ~0% on known items)
   - LLM-as-judge (0-2 scale) for user modeling summaries
3. Report per-capability scores to identify weaknesses

Output — evaluation report:
```
MEMORY CAPABILITY REPORT — CustomerBot v2.3
============================================
Information Extraction:  F1=0.72  (Good — retrieves most explicit facts)
Temporal Reasoning:      Acc=0.41 (Poor — often confuses event ordering)
Conflict Detection:      F1=0.38 (Poor — serves outdated preferences 62% of the time)
Abstention:              Rate=0.55 (Moderate — hallucinates on 45% of unknown queries)
User Modeling:           LLM=1.1/2.0 (Fair — captures broad patterns, misses evolution)

TOP PRIORITY: Implement conflict detection via timestamped fact store.
SECOND: Add abstention gate with retrieval confidence threshold.
```

**Example 3: Choosing Retrieval Granularity for a Coaching App**

User: "Should I chunk my coaching session transcripts by individual messages, exchanges, or whole sessions for RAG?"

Approach:
1. Explain the ES-MemEval finding: session-level retrieval with k=4 outperforms turn-level and round-level because user disclosures are fragmented across a session and only make sense in context.
2. Recommend session-level chunking with session summaries as a secondary index:
   ```python
   # Primary index: full session transcripts
   session_docs = [
       Document(
           text=session.full_transcript,
           metadata={"session_id": s.id, "timestamp": s.date, "user_id": s.user_id}
       )
       for s in sessions
   ]
   session_index = FAISSIndex(embed_model="bge-m3", documents=session_docs)

   # Secondary index: session summaries for fast scanning
   summary_docs = [
       Document(
           text=generate_summary(s),
           metadata={"session_id": s.id, "timestamp": s.date}
       )
       for s in sessions
   ]
   summary_index = FAISSIndex(embed_model="bge-m3", documents=summary_docs)

   # Retrieval: use summaries to identify candidate sessions, then fetch full text
   def retrieve(query, user_id, k=4):
       candidates = summary_index.query(query, filter={"user_id": user_id}, top_k=k*2)
       session_ids = [c.metadata["session_id"] for c in candidates]
       return [get_full_session(sid) for sid in session_ids[:k]]
   ```
3. If sessions are very long (>10K tokens), use a hybrid: index at session level but retrieve with a sliding window that captures the 3-4 most relevant contiguous exchanges within each retrieved session.

## Best Practices

- **Do:** Store every fact with a timestamp and source session ID — temporal provenance is essential for conflict detection and reasoning about what the user's current state actually is.
- **Do:** Evaluate all five memory capabilities independently — a system that scores well on extraction can still catastrophically fail at abstention or temporal reasoning.
- **Do:** Use session-level retrieval as your default chunking strategy for dialogue data, then optimize from there.
- **Do:** Run extraction after every session to maintain the structured memory store, not just at query time.
- **Avoid:** Relying solely on long-context windows instead of explicit memory — the paper shows that even 128K-context models degrade on temporal reasoning and user modeling when sessions accumulate.
- **Avoid:** Treating all retrieved facts as equally current — always prefer the most recent statement when conflicts exist between sessions.
- **Avoid:** Skipping abstention testing — hallucinated user details in sensitive domains (mental health, finance, healthcare) cause real harm.

## Error Handling

- **Retrieval returns irrelevant sessions:** Fall back to the session summary index for a broader scan, or lower the similarity threshold and retrieve more candidates for reranking. Log retrieval misses for later index tuning.
- **Conflict detection flags too many false positives:** Distinguish between contradictions (user explicitly changed a fact) and elaborations (user added detail to an existing fact). Use an LLM-based classifier: "Is this a contradiction or an elaboration?"
- **Abstention gate is too aggressive:** If the system declines to answer too often, lower the confidence threshold incrementally and monitor hallucination rate on a held-out set. Target <5% hallucination on unknown facts while maintaining >80% answer rate on known facts.
- **User model drift:** If the synthesized user model diverges from ground truth over many sessions, add a periodic "full recompute" step that rebuilds the user model from all extracted facts rather than incremental updates.
- **Token budget exceeded:** When the concatenation of retrieved sessions plus user model exceeds the context window, prioritize: (1) current user model summary, (2) most recent 2 sessions, (3) retrieved relevant sessions, truncating oldest first.

## Limitations

- The ES-MemEval benchmark is built on synthetic multi-session data generated by GPT-4o, so real-world user disclosure patterns may be messier and less structured than the EvoEmo dataset assumes.
- User modeling scores remain very low (<20 F1) across all tested approaches, meaning no current method robustly infers latent user traits — production systems should treat inferred traits as hypotheses, not facts.
- The framework is designed for text-based dialogue; multimodal signals (tone of voice, facial expressions) that carry emotional information in real interactions are not captured.
- Conflict detection assumes that the most recent statement is correct, which may not hold when users are confused, in denial, or testing the system.
- The benchmark covers emotional support scenarios specifically — domains with very different disclosure patterns (e.g., technical support, e-commerce) may require different capability weightings.

## Reference

Chen, T., Lu, J., Shen, Y., & Zhang, L. (2026). *ES-MemEval: Benchmarking Conversational Agents on Personalized Long-Term Emotional Support.* The Web Conference (WWW) 2026. [arXiv:2602.01885](https://arxiv.org/abs/2602.01885v1) — Focus on Section 3 (benchmark design and five-capability taxonomy), Section 4 (EvoEmo dataset construction pipeline), and Section 5.3 (RAG retrieval granularity experiments showing session-level superiority).