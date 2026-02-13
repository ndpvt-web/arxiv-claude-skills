---
name: "graph-based-agent-memory-taxonomy"
description: "Design and implement graph-based memory systems for LLM agents following the extraction-storage-retrieval-evolution lifecycle. Use when: 'build agent memory system', 'add long-term memory to my agent', 'implement knowledge graph memory', 'design memory for conversational agent', 'agent memory with graph structure', 'evolving memory for LLM agent'."
---

# Graph-Based Agent Memory: Design and Implementation

This skill enables Claude to architect and implement graph-based memory systems for LLM agents, following the taxonomy and lifecycle framework from the survey by Yang et al. (2026). Rather than flat vector stores or linear buffers, this approach structures agent memory as graphs — knowledge graphs, hierarchical trees, temporal graphs, or hypergraphs — enabling relational reasoning, efficient retrieval across abstraction levels, and self-evolving memory that improves over time. Claude applies this when helping users build agents that need to accumulate knowledge, recall relevant experiences, and refine their behavior across sessions.

## When to Use

- When the user wants to add persistent memory to an LLM-based agent (chatbot, coding agent, game agent, research assistant)
- When building a conversational agent that must track user preferences, facts, and interaction history across sessions
- When the user asks to replace a naive RAG vector store with structured, relational memory
- When implementing an agent that must reason over relationships between entities (e.g., people, events, code modules)
- When the user needs memory that ages, consolidates, or forgets — not just accumulates
- When designing multi-turn agents for complex domains: finance, medicine, game-playing, scientific discovery
- When the user mentions knowledge graphs, episodic memory, or temporal reasoning in the context of agents

## Key Technique

**The Graph Memory Lifecycle.** The core insight is that agent memory is not a static store — it follows a four-stage lifecycle: *extraction* (turning raw interactions into structured memory), *storage* (organizing memory as graphs with explicit relationships), *retrieval* (querying the graph to surface relevant context for the current task), and *evolution* (consolidating, pruning, and restructuring memory over time). Each stage has distinct techniques, and the graph structure is what ties them together.

**Why Graphs Over Vectors.** A flat vector store can find semantically similar chunks, but it cannot represent that Entity A *caused* Event B, that Fact X *contradicts* Fact Y, or that Skill Z *depends on* Skill W. Graph-based memory encodes these relationships as edges, enabling multi-hop reasoning (traversing connections), hierarchical abstraction (parent nodes summarize child subtrees), temporal ordering (timestamped edges with decay), and conflict detection (contradictory edges flagged during storage). The taxonomy identifies five graph types suited to different needs: knowledge graphs (entity-relation triples), hierarchical trees (multi-level abstraction), temporal knowledge graphs (time-stamped quadruples), hypergraphs (n-ary relations), and hybrid architectures combining multiple types.

**Self-Evolving Memory.** Unlike static knowledge bases, agent memory must evolve. The paper identifies internal evolution (consolidation via graph merging, schema abstraction from repeated patterns, topology reorganization) and external evolution (reactive correction from feedback, proactive gap-filling through targeted queries). This means a well-designed memory system gets *better* the more the agent uses it — merging redundant nodes, strengthening frequently-traversed paths, and pruning stale information.

## Step-by-Step Workflow

1. **Classify the memory requirements.** Determine which memory types the agent needs: semantic memory (facts about the world), episodic memory (interaction history), procedural memory (skills and routines), or associative memory (latent concept links). Most agents need at least two. Map these to the temporal axis: what must persist long-term vs. what is session-scoped working memory.

2. **Choose the graph structure.** Select based on the dominant relationship type:
   - **Knowledge graph** (entity-relation triples) for fact-heavy domains with clear entity relationships
   - **Hierarchical tree/DAG** for domains needing multi-level abstraction (broad categories down to specifics)
   - **Temporal knowledge graph** (quadruples with timestamps) for domains where facts change over time
   - **Hypergraph** for domains with complex n-ary relationships that resist binary decomposition
   - **Hybrid** when the agent has both static world knowledge and dynamic experience trajectories

3. **Implement memory extraction.** Build the pipeline that converts raw agent interactions into graph nodes and edges:
   - For text: use LLM prompting to extract (subject, relation, object) triples from each interaction turn
   - For sequential data (action trajectories): segment into discrete events, extract state transitions as edges
   - For multimodal data: generate textual descriptions first, then extract triples from those descriptions
   - Assign metadata: timestamps, confidence scores, source references

4. **Build the storage layer.** Implement the graph with conflict resolution:
   - Store nodes (entities/concepts) and edges (relationships) with metadata fields (created_at, last_accessed, confidence, source)
   - Implement deduplication: before inserting a new triple, check for existing nodes with matching or semantically similar names
   - Implement conflict detection: flag contradictory edges (e.g., "Alice works at Company A" vs. "Alice works at Company B") and resolve by timestamp recency or confidence score
   - For hierarchical storage: cluster semantically similar nodes under parent summaries using recursive summarization

5. **Implement indexing for fast retrieval.** Create multiple access paths into the graph:
   - Entity-centric index: hash map from entity names/aliases to node IDs
   - Semantic index: embed node/edge descriptions as vectors for similarity search
   - Temporal index: sorted structures for time-range queries
   - For hierarchical graphs: layer-based routing that starts at the top level and drills down

6. **Build the retrieval pipeline.** Implement a multi-stage retrieval process:
   - Stage 1 (candidate generation): use semantic similarity to find an initial set of relevant nodes
   - Stage 2 (graph expansion): traverse edges from candidate nodes (breadth-first, 1-2 hops) to pull in related context
   - Stage 3 (filtering): apply temporal decay (downweight old memories), relevance scoring, and deduplication
   - Stage 4 (formatting): serialize the retrieved subgraph into a prompt-friendly format (natural language sentences or structured JSON)

7. **Add memory evolution mechanisms.** Implement at least consolidation and pruning:
   - **Consolidation**: periodically merge similar nodes/subgraphs into generalized representations (e.g., merge five specific "user asked about Python" episodes into one "user frequently works with Python" node)
   - **Pruning**: remove or archive nodes that haven't been accessed beyond a threshold period, weighted by importance
   - **Feedback-driven correction**: when agent output is corrected by the user, update or invalidate the memory entries that led to the error
   - **Strengthening**: increment access counts on retrieved nodes; use these counts to bias future retrieval

8. **Wire memory into the agent loop.** Integrate the memory system into the agent's perception-reasoning-action cycle:
   - Before each LLM call, run the retrieval pipeline with the current user query as input
   - Inject retrieved memory as structured context in the system or user prompt
   - After the LLM responds, run the extraction pipeline on the new interaction turn
   - Periodically (every N turns or on session end) trigger evolution routines

9. **Test with realistic scenarios.** Validate memory behavior:
   - Multi-session continuity: does the agent recall facts from previous sessions?
   - Contradiction handling: does the agent prefer newer information when facts conflict?
   - Retrieval relevance: does the agent surface the right memories for the current context?
   - Scalability: does retrieval remain fast as the graph grows to thousands of nodes?

## Concrete Examples

**Example 1: Adding Graph Memory to a Conversational Agent**

User: "I have a chatbot built with LangChain. It forgets everything between sessions. Help me add persistent memory using a knowledge graph."

Approach:
1. Identify memory types needed: semantic (user facts), episodic (conversation history), associative (preference links)
2. Choose a knowledge graph as the primary structure, with temporal metadata on all edges
3. Implement extraction using LLM prompting after each turn

Implementation structure:
```python
# memory/graph_store.py
from dataclasses import dataclass, field
from datetime import datetime
import json

@dataclass
class MemoryNode:
    id: str
    label: str          # e.g., "Alice", "Python", "prefers dark mode"
    node_type: str      # "entity", "concept", "preference", "event"
    created_at: datetime = field(default_factory=datetime.now)
    last_accessed: datetime = field(default_factory=datetime.now)
    access_count: int = 0
    metadata: dict = field(default_factory=dict)

@dataclass
class MemoryEdge:
    source_id: str
    target_id: str
    relation: str       # e.g., "works_at", "prefers", "mentioned_in"
    confidence: float = 1.0
    created_at: datetime = field(default_factory=datetime.now)
    valid_until: datetime | None = None

class GraphMemory:
    def __init__(self, persist_path: str):
        self.nodes: dict[str, MemoryNode] = {}
        self.edges: list[MemoryEdge] = []
        self.persist_path = persist_path

    def extract_and_store(self, llm, conversation_turn: str):
        """Extract triples from a conversation turn and store them."""
        prompt = (
            "Extract factual triples from this text as JSON:\n"
            f"Text: {conversation_turn}\n"
            'Format: [{"subject": "...", "relation": "...", "object": "..."}]'
        )
        triples = json.loads(llm.invoke(prompt))
        for triple in triples:
            self._add_triple(triple["subject"], triple["relation"], triple["object"])

    def retrieve(self, query: str, embedder, top_k: int = 5, hops: int = 1):
        """Retrieve relevant subgraph: semantic candidates + graph expansion."""
        # Stage 1: semantic candidate generation
        query_vec = embedder.encode(query)
        scored = []
        for node in self.nodes.values():
            node_vec = embedder.encode(node.label)
            score = cosine_similarity(query_vec, node_vec)
            score *= self._temporal_decay(node.last_accessed)
            scored.append((node, score))
        candidates = sorted(scored, key=lambda x: -x[1])[:top_k]

        # Stage 2: graph expansion (BFS 1-hop)
        expanded = set()
        for node, _ in candidates:
            expanded.add(node.id)
            for edge in self.edges:
                if edge.source_id == node.id:
                    expanded.add(edge.target_id)
                elif edge.target_id == node.id:
                    expanded.add(edge.source_id)

        # Stage 3: format as context
        return self._format_subgraph(expanded)

    def evolve(self):
        """Consolidate similar nodes and prune stale entries."""
        self._merge_similar_nodes(similarity_threshold=0.92)
        self._prune_stale(days_threshold=90, min_access_count=2)
```

Output: A persistent graph memory that extracts facts from each conversation, retrieves relevant subgraphs before each LLM call, and periodically consolidates/prunes.

**Example 2: Temporal Memory for a Financial Analysis Agent**

User: "My agent tracks market events. It needs to know that 'Company X acquired Y in Q3 2025' and reason about time-dependent facts."

Approach:
1. Use a temporal knowledge graph with quadruples: (subject, relation, object, timestamp)
2. Implement bi-temporal tracking: valid_time (when the event occurred) and transaction_time (when recorded)
3. Build time-windowed retrieval that prefers recent facts and detects superseded information

```python
@dataclass
class TemporalEdge:
    source: str
    relation: str
    target: str
    valid_from: datetime       # when the fact became true
    valid_until: datetime | None  # when the fact ceased to be true (None = still valid)
    recorded_at: datetime      # when we stored this fact

class TemporalGraphMemory(GraphMemory):
    def retrieve_at_time(self, query: str, reference_time: datetime, **kwargs):
        """Retrieve facts valid at a specific point in time."""
        active_edges = [
            e for e in self.edges
            if e.valid_from <= reference_time
            and (e.valid_until is None or e.valid_until > reference_time)
        ]
        # Then run standard semantic + graph retrieval over active subgraph
        ...

    def supersede(self, old_edge_id: str, new_edge: TemporalEdge):
        """Mark an old fact as expired and insert the corrected version."""
        self.edges[old_edge_id].valid_until = new_edge.valid_from
        self.edges.append(new_edge)
```

Output: Memory that answers "What was Company X's status in Q3 2025?" differently from "What is Company X's status now?" using the same graph with temporal filtering.

**Example 3: Hierarchical Memory for a Coding Agent**

User: "My coding agent works on a large codebase. It needs to remember module-level architecture and also specific function behaviors."

Approach:
1. Use a hierarchical DAG: top level = system architecture, mid level = module relationships, leaf level = function/class details
2. Extraction segments code interactions into architectural observations vs. implementation details
3. Retrieval starts at the level matching query granularity and drills down as needed

```python
class HierarchicalMemory:
    def __init__(self):
        self.levels = {
            "architecture": {},  # system-wide patterns, module dependencies
            "module": {},        # module purposes, interfaces, key classes
            "detail": {},        # function behaviors, bug patterns, edge cases
        }
        self.parent_edges = []  # links detail -> module -> architecture

    def store(self, observation: str, level: str, parent_id: str | None = None):
        node = MemoryNode(id=uuid4(), label=observation, node_type=level)
        self.levels[level][node.id] = node
        if parent_id:
            self.parent_edges.append((node.id, parent_id))

    def retrieve(self, query: str, embedder):
        """Top-down retrieval: find best architecture match, then drill down."""
        best_arch = self._semantic_search(query, self.levels["architecture"], embedder)
        children = self._get_children(best_arch.id)
        best_module = self._semantic_search(query, children, embedder)
        details = self._get_children(best_module.id)
        return {"architecture": best_arch, "module": best_module, "details": details}

    def consolidate_level(self, level: str, llm):
        """Summarize leaf nodes into parent-level abstractions."""
        for parent_id, children in self._group_by_parent(level).items():
            child_texts = [c.label for c in children]
            summary = llm.invoke(f"Summarize these observations:\n" + "\n".join(child_texts))
            self.levels[self._parent_level(level)][parent_id].label = summary
```

Output: Memory where asking "How does authentication work?" retrieves the architecture-level overview, the auth module details, and specific function behaviors — all connected by explicit parent-child relationships.

## Best Practices

- **Do:** Assign timestamps and access counts to every node and edge from day one. Even if you don't use temporal retrieval immediately, this metadata is essential for evolution (decay, pruning, consolidation).
- **Do:** Run extraction with explicit LLM prompts that output structured JSON triples, not free-form text. Structured extraction produces cleaner graphs with fewer duplicate nodes.
- **Do:** Implement deduplication at insertion time using both exact string matching and semantic similarity (threshold ~0.90). Duplicate nodes fragment the graph and degrade retrieval.
- **Do:** Keep retrieved subgraphs small (5-20 nodes) and format them as natural language sentences for the LLM context window. Large raw graph dumps overwhelm the model.
- **Avoid:** Storing raw conversation text as nodes. Always extract structured facts/events first. Raw text creates an unsearchable blob, not a graph.
- **Avoid:** Building retrieval that only uses vector similarity. The entire point of graph memory is relational traversal — always include at least one hop of edge expansion after initial semantic matching.
- **Avoid:** Skipping the evolution stage. Without consolidation and pruning, the graph grows unboundedly, retrieval slows down, and redundant nodes accumulate.

## Error Handling

- **Extraction produces garbage triples.** LLMs sometimes hallucinate entities or relations. Mitigate by validating extracted triples against a schema (expected entity types, allowed relations) and setting a confidence threshold below which triples are discarded.
- **Graph grows too large for retrieval performance.** Set hard limits on node count per level (for hierarchical) or per time window (for temporal). Trigger consolidation when thresholds are exceeded. Consider archiving old subgraphs to cold storage.
- **Contradictory facts stored.** Implement conflict detection at insertion: before adding edge (A, relation, B), check for existing edges (A, relation, C) where B != C. Resolve by recency (keep the newer one), confidence score, or flagging for user resolution.
- **Retrieval returns irrelevant context.** Tune the pipeline: increase the semantic similarity threshold for candidate generation, reduce the hop count for graph expansion, or add a re-ranking step that scores candidate subgraphs against the query using an LLM.
- **Persistence failures.** Serialize the graph to disk (JSON, SQLite, or a graph database like Neo4j) after every write batch, not just on session end. Implement write-ahead logging for crash recovery.

## Limitations

- **LLM extraction quality is the bottleneck.** The graph is only as good as the triples extracted from interactions. Domain-specific jargon, implicit relationships, and ambiguous references all degrade extraction quality. Fine-tuning or few-shot prompting helps but doesn't fully solve this.
- **Graph maintenance has real compute cost.** Consolidation, pruning, conflict detection, and re-indexing all require LLM calls or embedding computations. For high-throughput agents (thousands of interactions per hour), these costs can dominate.
- **Not all domains need graphs.** If the agent's memory is purely "find similar past examples" with no relational reasoning, a vector store is simpler and sufficient. Graph memory adds value specifically when relationships between memories matter.
- **Schema evolution is hard.** As the agent encounters new types of entities or relationships, the graph schema needs to expand. Fully dynamic schemas risk inconsistency; rigid schemas miss new patterns. Hybrid approaches (core schema + flexible extensions) are a pragmatic middle ground.
- **Evaluation is immature.** Benchmarks like LoCoMo, LongMemEval, and MemoryAgentBench exist but don't fully capture real-world graph memory quality. Manual inspection of the graph structure remains necessary.

## Reference

Yang, C., Zhou, C., Xiao, Y., Dong, S., & Zhuang, L. (2026). *Graph-based Agent Memory: Taxonomy, Techniques, and Applications.* arXiv:2602.05665v1. [https://arxiv.org/abs/2602.05665v1](https://arxiv.org/abs/2602.05665v1)

Key takeaway: Read Section 3 (Storage) for graph construction paradigms and Section 4 (Retrieval) for the three-paradigm retrieval pipeline. The open-source library comparison in Section 6 is useful for choosing existing tools (Mem0, Zep, Graphiti) vs. building custom. Community resources: [https://github.com/DEEP-PolyU/Awesome-GraphMemory](https://github.com/DEEP-PolyU/Awesome-GraphMemory)