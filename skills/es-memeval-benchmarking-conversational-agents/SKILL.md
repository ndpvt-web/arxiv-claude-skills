---
name: "es-memeval-benchmarking-conversational-agents"
description: "Evaluate and build conversational agents with long-term memory for personalized emotional support, using the ES-MemEval benchmark framework's five memory capabilities: information extraction, temporal reasoning, conflict detection, abstention, and user modeling. Use when: 'build a chatbot with long-term memory', 'evaluate my agent's memory capabilities', 'add personalization to a dialogue system', 'implement memory-augmented RAG for conversations', 'benchmark conversational agent memory', 'design a user modeling pipeline for multi-session chat'."
---

# ES-MemEval: Long-Term Memory Evaluation and Design for Conversational Agents

This skill enables Claude to design, implement, and evaluate conversational agents that maintain robust long-term memory across multi-session dialogues. Based on the ES-MemEval benchmark (WWW 2026), it provides a concrete framework for testing five core memory capabilities — information extraction, temporal reasoning, conflict detection, abstention, and user modeling — and for building RAG-augmented dialogue systems that handle fragmented, implicit, and evolving user information over months of interaction.

## When to Use

- When building a multi-session chatbot that must remember user details across conversations
- When evaluating whether a conversational agent hallucinates user information or fabricates history
- When implementing RAG pipelines for dialogue systems and choosing retrieval granularity (turn vs. session level)
- When designing a user modeling component that tracks evolving preferences, states, or life events
- When the user asks to benchmark an LLM's ability to handle temporal dependencies in conversation history
- When adding "know when to say I don't know" (abstention) capability to an agent
- When building personalized emotional support, coaching, or therapy-adjacent chat systems
- When deciding between full-context vs. RAG approaches for long dialogue histories

## Key Technique

ES-MemEval decomposes long-term conversational memory into five testable capabilities, each targeting a distinct failure mode of current LLMs:

1. **Information Extraction** — retrieving explicit facts scattered across sessions (e.g., a user mentioned their job in session 3 and their city in session 12).
2. **Temporal Reasoning** — tracking when events happened and how they causally relate (e.g., a breakup in January causing anxiety about a sibling's engagement in March).
3. **Conflict Detection** — identifying when newer information contradicts older facts (e.g., user said they were single in session 5 but mentions a partner in session 15).
4. **Abstention** — recognizing when the dialogue history lacks sufficient evidence to answer, rather than hallucinating a response.
5. **User Modeling** — inferring latent traits, preferences, and evolving emotional states from implicit disclosures across sessions.

The companion EvoEmo dataset provides 18 virtual users across 401 sessions (~22 sessions/user, ~14.9 months span), with structured event timelines averaging 24.8 life events per user. The key architectural insight is that **session-level retrieval** (retrieving entire sessions as units) dramatically outperforms turn-level or round-level retrieval for emotionally complex dialogues, because relevant context is sparse and distributed within sessions. RAG with session-level chunks using bge-m3 embeddings and FAISS indexing achieves the best factual consistency, but still struggles with temporal reasoning and conflict detection — capabilities that require explicit memory management layers beyond naive retrieval.

## Step-by-Step Workflow

### For Evaluating an Existing Agent

1. **Define the memory test suite.** For each of the five capabilities, write 20-50 test questions against your agent's actual conversation logs. Information extraction questions ask for explicit facts; temporal reasoning questions require ordering or causal links; conflict detection questions present contradictory history; abstention questions ask about topics never discussed; user modeling questions ask about inferred traits.

2. **Structure test data as multi-session histories.** Organize conversation logs into chronological sessions with timestamps. Each session should be a self-contained dialogue (15-30 turns). Ensure at least some sessions contain implicit disclosures — information the user hints at but never states directly.

3. **Create ground-truth annotations.** For each test question, annotate: (a) the correct answer, (b) which session(s) contain the evidence, (c) the capability being tested. For abstention questions, the correct answer is explicit refusal or "insufficient information."

4. **Run the agent under three memory conditions.** Test with: (a) No memory — agent sees only the current session, (b) Full history — agent receives all prior sessions in context, (c) RAG — agent retrieves top-4 most relevant sessions via dense retrieval. Compare to isolate memory contribution from general reasoning.

5. **Score with multi-metric evaluation.** For QA: compute token-level F1, BERTScore, and LLM-as-judge (0-2 scale). For summarization: ROUGE-1/2/L plus event-level F1 (extract discrete events from both reference and generated summaries, compute overlap). For dialogue generation: observation-based recall against user state annotations plus LLM ratings (1-5) on memory, personalization, and support quality.

6. **Analyze per-capability breakdowns.** Aggregate scores by capability type. Expect: information extraction to score highest, abstention and conflict detection to score lowest. If conflict detection F1 is below 20%, the agent needs an explicit contradiction-checking layer.

### For Building a Memory-Augmented Agent

7. **Implement session-level RAG indexing.** Chunk conversation history at the session boundary (not turn or paragraph). Embed each session using a multi-granular model like bge-m3. Index with FAISS. At inference, retrieve top-4 sessions most relevant to the current user message.

8. **Add a temporal metadata layer.** Store session timestamps and extract event timelines (date + event description pairs) from each session. When answering temporal questions, sort retrieved sessions chronologically and present them in order to the LLM.

9. **Implement conflict detection logic.** After retrieval, run a lightweight check: for key user attributes (relationship status, job, location, beliefs), compare the most recent value against earlier values. If a contradiction exists, prepend a note to the LLM context: "Note: User previously stated X in [date], but later stated Y in [date]. Use the most recent information."

10. **Add abstention guardrails.** Instruct the agent (via system prompt) to respond with "I don't have enough information from our previous conversations to answer that" when retrieved evidence is absent or confidence is low. Validate with abstention test questions to ensure the agent actually refuses rather than fabricating.

## Concrete Examples

**Example 1: Evaluating a Therapy Chatbot's Memory**

User: "I want to test whether my chatbot remembers user details across sessions. Can you help me set up an evaluation?"

Approach:
1. Collect 10+ multi-session conversation logs from the chatbot's history
2. For each user, write test questions across the five capabilities:
   - Information Extraction: "What is the user's occupation?" (answer dispersed across session 2 and 7)
   - Temporal Reasoning: "Did the user's insomnia start before or after they changed jobs?" (requires cross-session timeline)
   - Conflict Detection: "The user said they enjoy running in session 3 but mentioned hating exercise in session 9. What is their current stance?"
   - Abstention: "What is the user's favorite movie?" (never mentioned — model should refuse)
   - User Modeling: "Based on all sessions, what are the user's primary coping mechanisms?"
3. Run the chatbot with no-memory, full-history, and RAG conditions
4. Score each response with F1, BERTScore, and GPT-4o as judge

Output:
```
Memory Capability Evaluation Report
====================================
Agent: therapy-bot-v2 | Users: 10 | Sessions: 187 | Questions: 250

Capability          | No-Mem F1 | Full-Hist F1 | RAG F1  | RAG BERTScore
--------------------|-----------|--------------|---------|-------------
Info Extraction     |   0.08    |    0.41      |  0.38   |    0.52
Temporal Reasoning  |   0.05    |    0.29      |  0.22   |    0.44
Conflict Detection  |   0.03    |    0.18      |  0.15   |    0.39
Abstention          |   0.12    |    0.55      |  0.42   |    0.61
User Modeling       |   0.06    |    0.33      |  0.31   |    0.48

Key Finding: RAG closes 70% of the gap to full-history for info extraction
but only 45% for temporal reasoning. Conflict detection remains weak across
all conditions. Recommend adding explicit contradiction-checking layer.
```

**Example 2: Building Session-Level RAG for a Coaching App**

User: "I'm building a life coaching chatbot. Users talk to it weekly. How do I add long-term memory with RAG?"

Approach:
1. Design the session storage schema:
```python
from dataclasses import dataclass
from datetime import datetime

@dataclass
class Session:
    session_id: str
    user_id: str
    timestamp: datetime
    turns: list[dict]  # [{"role": "user"|"assistant", "content": str}]
    summary: str        # auto-generated after session ends
    events: list[dict]  # [{"date": str, "description": str}]
```

2. Implement session-level embedding and retrieval:
```python
from sentence_transformers import SentenceTransformer
import faiss
import numpy as np

encoder = SentenceTransformer("BAAI/bge-m3")

def index_sessions(sessions: list[Session]) -> faiss.IndexFlatIP:
    texts = [s.summary + " " + " ".join(t["content"] for t in s.turns) for s in sessions]
    embeddings = encoder.encode(texts, normalize_embeddings=True)
    index = faiss.IndexFlatIP(embeddings.shape[1])
    index.add(embeddings.astype(np.float32))
    return index

def retrieve_sessions(query: str, index, sessions, k=4):
    q_emb = encoder.encode([query], normalize_embeddings=True).astype(np.float32)
    scores, indices = index.search(q_emb, k)
    return sorted([sessions[i] for i in indices[0]], key=lambda s: s.timestamp)
```

3. Construct the LLM prompt with temporal ordering:
```python
def build_prompt(current_msg: str, retrieved: list[Session], user_id: str) -> str:
    history = ""
    for s in retrieved:  # already sorted chronologically
        history += f"\n--- Session from {s.timestamp.strftime('%B %d, %Y')} ---\n"
        for turn in s.turns:
            history += f"{turn['role'].title()}: {turn['content']}\n"
    return f"""You are a life coach with long-term memory of this user.
Below are relevant past sessions in chronological order.
If asked about something not covered in these sessions, say so honestly.

{history}

Current message: {current_msg}
Respond with empathy and reference specific details from past sessions when relevant."""
```

**Example 3: Adding Conflict Detection to an Existing Agent**

User: "My agent sometimes uses outdated user info. How do I detect contradictions?"

Approach:
1. Maintain a key-value store of user attributes with timestamps:
```python
user_facts = {
    "relationship_status": [
        {"value": "single", "session_id": "s005", "date": "2025-03-15"},
        {"value": "dating someone", "session_id": "s012", "date": "2025-07-22"}
    ],
    "employment": [
        {"value": "software engineer at Acme", "session_id": "s003", "date": "2025-02-01"},
        {"value": "unemployed, laid off", "session_id": "s018", "date": "2025-10-10"}
    ]
}
```

2. Before generating a response, check retrieved sessions for attribute conflicts:
```python
def detect_conflicts(retrieved_sessions, user_facts):
    conflicts = []
    for attr, history in user_facts.items():
        if len(history) > 1:
            latest = max(history, key=lambda x: x["date"])
            for entry in history:
                if entry["value"] != latest["value"]:
                    conflicts.append({
                        "attribute": attr,
                        "old": entry["value"], "old_date": entry["date"],
                        "current": latest["value"], "current_date": latest["date"]
                    })
    return conflicts
```

3. Inject conflict context into the prompt so the LLM uses the latest information and avoids referencing outdated facts.

## Best Practices

- **Do:** Retrieve at session granularity rather than individual turns. Emotional support dialogues contain sparse, distributed context — session-level chunks preserve the coherence needed for inference.
- **Do:** Always present retrieved sessions in chronological order. Temporal ordering is essential for the LLM to reason about cause-and-effect and evolving states.
- **Do:** Explicitly test abstention. Build test cases where the correct answer is "I don't know" and measure whether your agent refuses or hallucinates.
- **Do:** Use event-level F1 for summarization evaluation, not just ROUGE. Extract discrete events from both reference and generated summaries, then compute overlap — this catches factual errors that ROUGE misses.
- **Avoid:** Relying on context window alone for histories beyond 8K tokens. Performance degrades sharply for smaller models past 2K tokens and even 128K-context models lose accuracy on temporal and conflict tasks at scale.
- **Avoid:** Assuming RAG fixes everything. RAG improves factual consistency but can reduce abstention quality — retrieved content encourages overconfident responses. Pair RAG with explicit uncertainty detection.

## Error Handling

- **Retrieval returns irrelevant sessions:** Fall back to a "no strong match found" response rather than forcing the LLM to use poor context. Set a minimum similarity threshold (e.g., cosine > 0.3) and discard below-threshold results.
- **Conflicting facts without timestamps:** If your session store lacks timestamps, you cannot resolve conflicts. Always store session dates. If retroactively adding timestamps is impossible, treat the most recently indexed session as authoritative.
- **User modeling hallucination:** When the LLM infers traits not grounded in session text (e.g., "the user seems introverted" without evidence), add a grounding check: require the model to cite a specific session or quote before stating inferred traits.
- **Context window overflow with full history:** For users with 20+ sessions, full-history mode will exceed context limits. Default to RAG with k=4 sessions and increase k only if initial retrieval scores are low.
- **Evaluation judge disagreement:** LLM-as-judge scores can be noisy. Use multiple judge calls and average, or have GPT-4o score twice and take the minimum to reduce false positives.

## Limitations

- The ES-MemEval benchmark and EvoEmo dataset are synthetically generated (GPT-4o + human refinement), which may not capture the messiness of real human conversations — real users digress, use slang, and contradict themselves in less structured ways.
- Conflict detection remains the hardest capability (F1 below 20% even for the best models). No current approach reliably handles it without explicit rule-based or structured-memory layers on top of LLMs.
- The framework is designed for emotional support contexts where disclosures are implicit and gradual. For task-oriented dialogues (e.g., customer service, technical support) where information is explicit and structured, simpler memory approaches may suffice.
- Abstention performance degrades with RAG — retrieved content can make the model overconfident. This is an open problem without a clean solution.
- Session-level retrieval works best for sparse, emotionally rich dialogues. For information-dense conversations (e.g., medical intake), finer-grained retrieval may be more appropriate.

## Reference

**Paper:** [ES-MemEval: Benchmarking Conversational Agents on Personalized Long-Term Emotional Support](https://arxiv.org/abs/2602.01885v1) (WWW 2026)
**What to look for:** Table 4 for per-capability QA results across models; Table 5 for context-length degradation curves; Table 7 for retrieval granularity comparison (session > round > turn); Table 8 for how memory conditions affect personalization vs. emotional support scores independently.