---
name: "agentic-very-long-video"
description: "Build agentic systems for understanding very long video streams (hours to weeks) using entity scene graphs, multi-tool planning agents, and hybrid cross-modal search. Use when: 'build a video understanding agent', 'analyze long egocentric video', 'entity graph from video', 'multi-hop video QA', 'search across hours of footage', 'temporal reasoning over video'."
---

# Agentic Very Long Video Understanding (EGAgent)

This skill enables Claude to build agentic systems that understand very long video streams — hours, days, or weeks of footage — by constructing **entity scene graphs** and equipping a planning agent with structured search tools. Rather than feeding raw frames into an LLM's limited context window, the EGAgent approach extracts a persistent graph of people, places, objects, and their temporal relationships, then uses a ReAct-style planner to decompose complex queries into targeted sub-searches across visual, audio, and graph modalities. This is the architecture from Rege et al. (2026) that achieves state-of-the-art on longitudinal video QA.

## When to Use

- When the user needs to build a system that answers questions about video content spanning hours or longer (e.g., wearable camera footage, security feeds, meeting recordings)
- When the user asks how to perform multi-hop temporal reasoning over video ("Before X happened, who did Y?")
- When the user wants to combine visual frame search with audio transcript search and structured entity lookups
- When the user needs to track entities (people, objects, locations) and their relationships across a long video stream
- When the user is building a RAG pipeline for video and hitting context-window limitations
- When the user wants to implement an agentic tool-use loop for video question answering

## Key Technique

**The core insight**: Unstructured retrieval (embedding frames, chunking transcripts) loses relational and temporal structure. EGAgent solves this by building an **entity scene graph** G = (V, E) where nodes represent people, objects, and locations, and edges encode typed relationships (`talks-to`, `interacts-with`, `mentions`, `uses`) with explicit temporal intervals (t_start, t_end). This graph is stored in SQLite as tuples: `(source, source_type, target, target_type, relationship, t_start, t_end, supporting_text)`. The graph is constructed by fusing visual captions and audio transcripts per time segment, then running LLM-based entity-relationship extraction on each fused document.

**The agentic loop**: A planning agent receives a user query and decomposes it into N ordered subtasks, each paired with a tool selection: (1) **Visual Search** — embeds frames at 1 FPS with SigLIP, stores in a vector DB with metadata filters (day, location), retrieves by cosine similarity; (2) **Audio Transcript Search** — either BM25 lexical search or LLM-based relevance judgment over transcript segments; (3) **Entity Graph Search** — generates SQL queries against the graph DB using a strict-to-relaxed strategy (exact match first, then progressively loosening time windows, substring matching, and relationship constraints). Retrieved evidence accumulates in a **working memory**, which is finally passed to a VQA agent for synthesis.

**Why it works**: The structured graph preserves multi-hop relationships that flat retrieval destroys. A query like "Who was with me when I talked to the person I met at the coffee shop last Tuesday?" requires chaining: locate coffee-shop visit → identify person → find co-located individuals → match temporal overlap. The entity graph makes each hop a targeted SQL query rather than a needle-in-a-haystack embedding search.

## Step-by-Step Workflow

1. **Segment the video into time-windowed documents.** Split the video timeline into fixed segments (e.g., 30-second or 1-minute windows). For each segment, extract a visual caption (from sampled frames via a VLM like Qwen2.5-VL) and an audio transcript (via Whisper or equivalent). Fuse these into a single document per segment with timestamps.

2. **Build the entity scene graph.** For each fused document, run LLM-based entity-relationship extraction (e.g., using LangChain's `LLMGraphTransformer`) to produce nodes (typed as person/object/location) and edges (typed relationships with temporal bounds). Aggregate across all segments: V = union of all V_d, E = union of all E_d. Store in SQLite with the schema: `(source, source_type, target, target_type, relationship, t_start, t_end, supporting_text)`.

3. **Index visual frames for retrieval.** Sample frames at 1 FPS, embed each with a vision encoder (SigLIP 2), and store in a vector database (FAISS, Chroma, or Qdrant) with metadata: `{timestamp, day, location_label}`. This enables filtered nearest-neighbor search.

4. **Index audio transcripts for retrieval.** Store transcript segments with timestamps. Implement both a BM25 index (for fast lexical search) and optionally an LLM-based relevance filter (for semantic search when BM25 recall is insufficient).

5. **Define the tool interface for the planning agent.** Implement three retriever tools with clear input/output contracts:
   - `visual_search(query: str, day: Optional[str], location: Optional[str], k: int) -> List[Frame]`
   - `audio_search(query: str, time_range: Optional[Tuple], mode: "bm25"|"llm") -> List[TranscriptSegment]`
   - `entity_graph_search(entities: List[str], relationship: Optional[str], time_range: Optional[Tuple]) -> List[GraphTuple]`
   Plus an `analyzer(retrieved_data, subtask_query) -> relevance_summary` tool for filtering.

6. **Implement the planning agent with query decomposition.** The planner receives the user query and outputs an ordered list of subtasks, each specifying which tool to call and with what arguments. Constrain to at most 5 subtasks. Use a system prompt that instructs the LLM to decompose compositional queries into atomic lookups. Implement this with LangGraph or a similar agent framework.

7. **Execute subtasks iteratively, accumulating working memory.** For each subtask: call the specified tool, pass results through the analyzer for relevance filtering, append the filtered evidence to a working memory buffer M. The memory is a structured list of `{subtask, tool, evidence_summary, timestamps}` entries.

8. **Implement strict-to-relaxed SQL generation for graph search.** When the entity graph search returns no results, progressively relax: (a) widen the time window, (b) switch from exact entity name match to substring/fuzzy match, (c) drop the relationship type constraint, (d) drop one entity constraint. This cascade prevents empty results on imprecise queries.

9. **Synthesize the final answer.** Pass the accumulated working memory M and the original query Q to a multimodal LLM (the VQA agent). Include the most relevant retrieved frames as visual context. The VQA agent produces the final answer grounded in cross-modal evidence.

10. **Evaluate and iterate.** Test on multi-hop queries that require chaining across modalities. Measure retrieval recall at different time windows (10s, 30s, 1min, 1hr). Track per-category accuracy: factual recall, relational reasoning, temporal ordering, and task-sequence reconstruction.

## Concrete Examples

**Example 1: Building the Entity Graph Pipeline**

User: "I have 8 hours of egocentric video from smart glasses. I want to build a system that can answer questions like 'Who did I talk to at lunch?' Help me set up the entity graph."

Approach:
1. Segment the video into 30-second windows with ffmpeg
2. Extract frames at 1 FPS and generate captions with a VLM
3. Transcribe audio with Whisper, aligned to the same time windows
4. Fuse caption + transcript per segment into a document
5. Extract entities and relationships with an LLM
6. Store in SQLite

Output:
```python
import sqlite3
from langchain_experimental.graph_transformers import LLMGraphTransformer
from langchain_core.documents import Document

# Schema
conn = sqlite3.connect("entity_graph.db")
conn.execute("""
    CREATE TABLE IF NOT EXISTS edges (
        source TEXT, source_type TEXT,
        target TEXT, target_type TEXT,
        relationship TEXT,
        t_start REAL, t_end REAL,
        supporting_text TEXT
    )
""")
conn.execute("CREATE INDEX IF NOT EXISTS idx_time ON edges(t_start, t_end)")
conn.execute("CREATE INDEX IF NOT EXISTS idx_source ON edges(source)")

# Per-segment extraction
transformer = LLMGraphTransformer(llm=llm)
for segment in segments:
    doc = Document(page_content=f"[Visual] {segment.caption}\n[Audio] {segment.transcript}")
    graph_docs = transformer.convert_to_graph_documents([doc])
    for gd in graph_docs:
        for rel in gd.relationships:
            conn.execute(
                "INSERT INTO edges VALUES (?,?,?,?,?,?,?,?)",
                (rel.source.id, rel.source.type, rel.target.id, rel.target.type,
                 rel.type, segment.t_start, segment.t_end, doc.page_content)
            )
conn.commit()
```

**Example 2: Multi-Tool Planning Agent**

User: "Build the agentic QA loop that decomposes a question into sub-searches across the graph, visual frames, and transcripts."

Approach:
1. Define tool schemas for visual, audio, and graph search
2. Build a LangGraph agent with a planner node and tool-execution nodes
3. Implement working memory accumulation
4. Add a final synthesis node

Output:
```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, List, Optional

class AgentState(TypedDict):
    query: str
    subtasks: List[dict]
    current_step: int
    working_memory: List[dict]
    answer: Optional[str]

def planner(state: AgentState) -> AgentState:
    """Decompose query into subtasks with tool assignments."""
    prompt = f"""Decompose this question into at most 5 subtasks.
For each, specify: subtask description, tool (visual_search | audio_search | entity_graph_search), query arguments.
Question: {state['query']}
Output JSON list."""
    subtasks = llm.invoke(prompt)  # returns list of {subtask, tool, query_args}
    return {**state, "subtasks": subtasks, "current_step": 0}

def execute_tool(state: AgentState) -> AgentState:
    """Run the current subtask's tool and accumulate evidence."""
    step = state["subtasks"][state["current_step"]]
    tool_fn = {"visual_search": visual_search,
               "audio_search": audio_search,
               "entity_graph_search": entity_graph_search}[step["tool"]]
    results = tool_fn(**step["query_args"])
    filtered = analyzer(results, step["subtask"])
    memory = state["working_memory"] + [{"subtask": step["subtask"], "evidence": filtered}]
    return {**state, "working_memory": memory, "current_step": state["current_step"] + 1}

def synthesize(state: AgentState) -> AgentState:
    """Combine all evidence into a final answer."""
    prompt = f"Question: {state['query']}\nEvidence:\n"
    for m in state["working_memory"]:
        prompt += f"- [{m['subtask']}]: {m['evidence']}\n"
    prompt += "Answer the question based on the evidence above."
    answer = vqa_llm.invoke(prompt)
    return {**state, "answer": answer}

def should_continue(state: AgentState) -> str:
    return "execute" if state["current_step"] < len(state["subtasks"]) else "synthesize"

graph = StateGraph(AgentState)
graph.add_node("plan", planner)
graph.add_node("execute", execute_tool)
graph.add_node("synthesize", synthesize)
graph.set_entry_point("plan")
graph.add_conditional_edges("plan", should_continue, {"execute": "execute", "synthesize": "synthesize"})
graph.add_conditional_edges("execute", should_continue, {"execute": "execute", "synthesize": "synthesize"})
graph.add_edge("synthesize", END)
agent = graph.compile()
```

**Example 3: Strict-to-Relaxed Graph Search**

User: "The entity graph search often returns empty results. How do I implement the relaxation cascade?"

Output:
```python
def entity_graph_search(entities, relationship=None, time_range=None, conn=None):
    """Progressive relaxation: exact -> substring -> drop relationship -> drop entity."""
    strategies = [
        # Level 0: exact match on all constraints
        lambda: _query(entities, relationship, time_range, match="exact"),
        # Level 1: widen time window by 2x
        lambda: _query(entities, relationship, _widen(time_range, 2.0), match="exact"),
        # Level 2: substring match on entity names
        lambda: _query(entities, relationship, _widen(time_range, 2.0), match="substring"),
        # Level 3: drop relationship constraint
        lambda: _query(entities, None, _widen(time_range, 2.0), match="substring"),
        # Level 4: search each entity independently
        lambda: [r for e in entities for r in _query([e], None, None, match="substring")],
    ]
    for strategy in strategies:
        results = strategy()
        if results:
            return results
    return []

def _query(entities, relationship, time_range, match="exact"):
    where = []
    params = []
    for e in entities:
        if match == "exact":
            where.append("(source = ? OR target = ?)")
            params.extend([e, e])
        else:
            where.append("(source LIKE ? OR target LIKE ?)")
            params.extend([f"%{e}%", f"%{e}%"])
    if relationship:
        where.append("relationship = ?")
        params.append(relationship)
    if time_range:
        where.append("t_start >= ? AND t_end <= ?")
        params.extend(time_range)
    sql = f"SELECT * FROM edges WHERE {' AND '.join(where)}"
    return conn.execute(sql, params).fetchall()
```

## Best Practices

- **Do**: Keep visual search queries short and distinct (single words or short noun phrases) when using CLIP/SigLIP embeddings — verbose queries degrade retrieval quality.
- **Do**: Store the supporting text alongside each graph edge so you can trace answers back to source evidence and present provenance to users.
- **Do**: Constrain the planner to a maximum of 5 subtasks — more steps compound retrieval errors without improving answer quality.
- **Do**: Use temporal metadata aggressively as a filter (day, time-of-day) before running expensive embedding searches to reduce the candidate set.
- **Avoid**: Feeding entire long videos directly into a VLM context window — this is the baseline approach EGAgent outperforms by 20+ percentage points.
- **Avoid**: Relying solely on embedding-based retrieval without the entity graph — flat retrieval cannot resolve multi-hop relational queries (e.g., "the person I talked to at the place where I had coffee").

## Error Handling

- **Empty graph search results**: The strict-to-relaxed cascade (see Example 3) handles this. If all levels return empty, fall back to visual or audio search on the same query.
- **Entity resolution failures**: The same person may appear with different names across segments ("John", "Dr. Smith", "the tall guy"). Implement a post-extraction entity merging step using string similarity or LLM-based coreference.
- **Timestamp misalignment**: Audio and visual timestamps may drift. Align both modalities to a shared timeline before graph construction. Use the audio track embedded in the video file as the canonical clock.
- **Planner hallucinating tool arguments**: Validate tool arguments against known schemas before execution. Reject and re-prompt if the planner outputs malformed queries.
- **Retrieval returning irrelevant results**: The analyzer tool acts as a relevance gate — always filter retrieved data before adding to working memory. Discard evidence the analyzer scores as irrelevant.

## Limitations

- Requires pre-processing (captioning, transcription, graph extraction) that scales linearly with video length. Days of footage means hours of indexing.
- Entity extraction quality depends heavily on the underlying LLM and VLM. Low-quality captions produce noisy graphs.
- The approach assumes video with meaningful visual and audio content. Pure surveillance footage with no dialogue will underutilize the audio and graph modalities.
- Graph search is only as good as the relationship types extracted. Uncommon or implicit relationships (sarcasm, indirect references) may not be captured.
- The system is designed for question answering, not real-time streaming. Adapting to live video requires incremental graph updates and index maintenance.

## Reference

Rege, A., Sadhu, A., Li, Y., Li, K., & Vinayak, R. K. (2026). *Agentic Very Long Video Understanding*. arXiv:2601.18157. Key focus: Section 3 (EGAgent framework), Algorithm 1 (agentic loop), Section 3.2 (entity scene graph construction), and Table 1 (tool definitions). The paper demonstrates that structured entity graphs combined with multi-tool agentic planning substantially outperform both direct VLM inference and standard RAG on longitudinal video QA.