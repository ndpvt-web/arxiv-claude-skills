---
name: "query-efficient-agentic-graph-extraction"
description: >
  Implements the AGEA framework for budget-constrained extraction of knowledge graphs from
  GraphRAG systems using novelty-guided adaptive querying, external graph memory, and a
  two-stage discovery-then-filter pipeline. Use this skill when the user says:
  "extract the knowledge graph from this RAG system", "audit GraphRAG for data leakage",
  "test if my GraphRAG API exposes its graph structure", "reconstruct the entity-relation
  graph behind this endpoint", "red-team my GraphRAG system for extraction attacks",
  "assess knowledge graph leakage risk in my RAG pipeline".
---

# Query-Efficient Agentic Graph Extraction (AGEA)

This skill enables Claude to implement and apply the AGEA (Agentic Graph Extraction Attack) framework from Yang et al. (2026) for **authorized security testing** of GraphRAG systems. AGEA reconstructs the hidden entity-relation knowledge graph behind a black-box GraphRAG API using far fewer queries than brute-force approaches. It combines a novelty-guided exploration-exploitation strategy with external graph memory and a two-stage pipeline (lightweight entity discovery followed by LLM-based relationship filtering) to recover up to 90% of entities and relationships under strict query budgets. This is critical for security audits, privacy compliance testing, and evaluating data leakage risk in production GraphRAG deployments.

## When to Use

- When the user needs to **audit a GraphRAG system** (Microsoft GraphRAG, LightRAG, or similar) for knowledge graph leakage as part of authorized security testing
- When the user wants to **build a red-teaming harness** that probes a RAG API and reconstructs its latent graph structure
- When the user asks to **measure how much of a private knowledge graph** is recoverable through normal API queries
- When the user wants to **implement budget-constrained adaptive querying** against any system that returns structured subgraph-like information in responses
- When the user needs to **build defenses** against graph extraction by understanding the attack surface (e.g., response truncation, query throttling, entity masking)
- When the user asks to **compare extraction efficiency** across different RAG architectures or configurations

## Key Technique

**Two-Stage Pipeline: Discovery then Filter.** AGEA splits graph extraction into two phases. Stage 1 (Lightweight Discovery) uses broad, low-cost queries to rapidly identify candidate entities and surface-level relationships. It parses responses for named entities and co-occurrence patterns without heavy LLM reasoning. Stage 2 (LLM-Based Filtering) takes the noisy candidate set and applies targeted validation queries plus LLM-based reasoning to confirm genuine relationships, type them, assign confidence scores, and discard false positives. This separation means the expensive LLM budget is spent only on high-value validation, not on initial exploration.

**Novelty-Guided Exploration-Exploitation.** The core query selection mechanism works like a multi-armed bandit. Each candidate query direction gets a novelty score measuring how much new information it is predicted to yield relative to the current state of the extracted graph. High-novelty directions (unexplored entity clusters, disconnected components) receive exploration priority. Low-novelty directions (densely mapped regions) are deprioritized. The balance shifts dynamically: early rounds favor exploration (breadth), later rounds favor exploitation (completing partially-known relationships). A priority queue ranks all candidate queries by expected information gain, and the budget tracker terminates when marginal returns drop below a configurable threshold.

**External Graph Memory.** Unlike stateless extraction approaches, AGEA maintains a persistent graph data structure across all queries. Every discovered entity and relationship is indexed in this memory module. Before generating a new query, the agent consults memory to avoid redundant questions and to identify gaps (e.g., an entity with known neighbors on one side but none on the other). This graph memory also enables path-completion heuristics that target disconnected subgraph components for merging.

## Step-by-Step Workflow

1. **Define the target and authorization scope.** Confirm the user has authorization to test the GraphRAG system. Identify the API endpoint, query interface (natural language or structured), response format, and any rate limits or token budgets.

2. **Initialize external graph memory.** Create an in-memory graph structure (using NetworkX or equivalent) with node attributes for entities (name, type, confidence, discovery round) and edge attributes for relationships (type, confidence, source query). Initialize an empty priority queue for candidate queries and a query log for budget tracking.

3. **Generate seed queries for Stage 1 discovery.** Based on the known domain (medical, legal, literary, etc.), construct 3-5 broad seed queries designed to surface high-degree entities. Examples: "What are the main topics covered?", "Summarize the key entities and their relationships", "What connects [domain concept] to other subjects?". If the domain is unknown, use generic probing queries.

4. **Execute Stage 1: Lightweight Discovery loop.** For each query: (a) send it to the GraphRAG API, (b) parse the response for entity mentions using NER or regex patterns, (c) extract co-occurrence pairs as candidate relationships, (d) add new entities and candidate edges to graph memory with low initial confidence, (e) compute novelty scores for follow-up directions based on newly discovered vs. already-known information, (f) enqueue high-novelty follow-up queries. Continue until the discovery budget fraction is exhausted (typically 40-60% of total budget).

5. **Score and rank candidate entities/relationships.** After Stage 1, rank all discovered entities by degree centrality and all candidate relationships by frequency of co-occurrence. Identify high-uncertainty regions: entities with few confirmed relationships, disconnected components, and relationship types that appeared only once.

6. **Execute Stage 2: LLM-Based Filtering and Validation.** Construct targeted queries that reference specific entity pairs: "What is the relationship between [Entity A] and [Entity B]?", "How does [Entity A] relate to [Entity C]?". Send these to the GraphRAG API. Use LLM reasoning on the responses to: (a) confirm or reject candidate relationships, (b) type the relationship (e.g., "causes", "treats", "authored_by"), (c) discover transitive relationships not seen in Stage 1, (d) update confidence scores in graph memory.

7. **Apply path-completion heuristics.** Identify disconnected components in the extracted graph. Generate bridging queries targeting potential links between components (e.g., query entities from each component together). Merge components when confirmed relationships are found.

8. **Deduplicate and normalize the extracted graph.** Resolve entity aliases (e.g., "COVID-19" vs "SARS-CoV-2"), merge duplicate nodes, normalize relationship types to a consistent schema, and prune edges below a confidence threshold.

9. **Compute extraction metrics.** If ground truth is available, calculate precision, recall, and F1 for both entities and relationships. If not, report: total entities discovered, total relationships confirmed, graph density, number of connected components, and query efficiency (entities per query, relationships per query).

10. **Generate the security assessment report.** Summarize: (a) what fraction of the knowledge graph was recoverable, (b) which query patterns were most effective, (c) specific leakage vectors identified, (d) recommended mitigations (response truncation, entity masking, query throttling, differential privacy on subgraph retrieval).

## Concrete Examples

**Example 1: Auditing a Medical GraphRAG for Patient Data Leakage**

User: "I have a GraphRAG system built on medical literature using Microsoft GraphRAG. I need to test how much of the underlying knowledge graph an attacker could reconstruct with 100 queries. Help me build the extraction harness."

Approach:
1. Set up a Python script with `networkx` for graph memory and `openai` (or equivalent) client for querying the GraphRAG API.
2. Define 5 medical seed queries: "What diseases are covered in this knowledge base?", "What treatments are discussed for chronic conditions?", "Which drugs interact with each other?", "What are the main organ systems referenced?", "How are diagnostic procedures categorized?"
3. Parse responses with spaCy NER (`en_core_sci_lg` model) to extract biomedical entities.
4. Allocate 50 queries to Stage 1 discovery, 50 to Stage 2 validation.
5. After Stage 1, rank entities by mention frequency and co-occurrence strength.
6. In Stage 2, construct pairwise validation queries for the top 40 entity pairs.
7. Export the reconstructed graph and compute coverage metrics against the source graph.

Output:
```python
import networkx as nx
from collections import defaultdict
import heapq

class AGEAExtractor:
    def __init__(self, query_fn, budget=100, discovery_ratio=0.5):
        self.query_fn = query_fn  # callable: str -> str
        self.budget = budget
        self.discovery_budget = int(budget * discovery_ratio)
        self.validation_budget = budget - self.discovery_budget
        self.graph = nx.Graph()
        self.query_log = []
        self.candidate_edges = defaultdict(int)  # (e1, e2) -> co-occurrence count
        self.priority_queue = []  # (neg_novelty, query_str)

    def novelty_score(self, entities_found):
        new = sum(1 for e in entities_found if e not in self.graph)
        return new / max(len(entities_found), 1)

    def stage1_discover(self, seed_queries, parse_entities_fn):
        queries_used = 0
        for q in seed_queries:
            if queries_used >= self.discovery_budget:
                break
            response = self.query_fn(q)
            self.query_log.append({"stage": 1, "query": q, "response": response})
            entities = parse_entities_fn(response)
            novelty = self.novelty_score(entities)
            for e in entities:
                if e not in self.graph:
                    self.graph.add_node(e, confidence=0.5, round=queries_used)
            for i, e1 in enumerate(entities):
                for e2 in entities[i+1:]:
                    self.candidate_edges[(e1, e2)] += 1
            # Generate follow-up queries for novel directions
            for e in entities:
                if e not in self.graph or self.graph.degree(e) < 2:
                    follow_up = f"What is related to {e}?"
                    heapq.heappush(self.priority_queue, (-novelty, follow_up))
            queries_used += 1
        # Drain priority queue with remaining discovery budget
        while queries_used < self.discovery_budget and self.priority_queue:
            neg_nov, q = heapq.heappop(self.priority_queue)
            response = self.query_fn(q)
            self.query_log.append({"stage": 1, "query": q, "response": response})
            entities = parse_entities_fn(response)
            for e in entities:
                if e not in self.graph:
                    self.graph.add_node(e, confidence=0.5, round=queries_used)
            for i, e1 in enumerate(entities):
                for e2 in entities[i+1:]:
                    self.candidate_edges[(e1, e2)] += 1
            queries_used += 1
        return queries_used

    def stage2_validate(self, validate_relation_fn):
        ranked_pairs = sorted(
            self.candidate_edges.items(), key=lambda x: -x[1]
        )[:self.validation_budget]
        for (e1, e2), count in ranked_pairs:
            q = f"What is the relationship between {e1} and {e2}?"
            response = self.query_fn(q)
            self.query_log.append({"stage": 2, "query": q, "response": response})
            rel_type, confidence = validate_relation_fn(response, e1, e2)
            if confidence > 0.5:
                self.graph.add_edge(e1, e2, type=rel_type, confidence=confidence)

    def report(self):
        return {
            "entities": self.graph.number_of_nodes(),
            "relationships": self.graph.number_of_edges(),
            "components": nx.number_connected_components(self.graph),
            "queries_used": len(self.query_log),
            "entities_per_query": self.graph.number_of_nodes() / max(len(self.query_log), 1),
        }
```

**Example 2: Comparing Leakage Between LightRAG and Microsoft GraphRAG**

User: "I deployed the same document corpus on both LightRAG and Microsoft GraphRAG. Run the same extraction attack on both and compare which leaks more graph structure."

Approach:
1. Instantiate two `AGEAExtractor` objects with identical budgets and seed queries.
2. Use the same entity parser and relationship validator for both.
3. Run Stage 1 and Stage 2 on each system.
4. Compare metrics: entity recall, relationship recall, precision, graph density, and query efficiency.

Output:
```python
def compare_systems(light_rag_fn, ms_graphrag_fn, seed_queries, budget=100):
    light = AGEAExtractor(light_rag_fn, budget=budget)
    ms = AGEAExtractor(ms_graphrag_fn, budget=budget)

    light.stage1_discover(seed_queries, parse_entities)
    light.stage2_validate(validate_relation)

    ms.stage1_discover(seed_queries, parse_entities)
    ms.stage2_validate(validate_relation)

    print("=== LightRAG ===")
    print(light.report())
    print("=== Microsoft GraphRAG ===")
    print(ms.report())
    # If ground truth available:
    # print(f"LightRAG entity recall: {recall(light.graph, ground_truth)}")
    # print(f"MS GraphRAG entity recall: {recall(ms.graph, ground_truth)}")
```

**Example 3: Building Defenses Based on Extraction Analysis**

User: "My GraphRAG system is leaking too much structure. Based on the AGEA attack patterns, what defenses should I implement?"

Approach:
1. Run the extraction attack and analyze which query types yielded the most entities per query.
2. Identify the top leakage vectors (e.g., broad "summarize" queries returning many entities, pairwise queries confirming relationships).
3. Recommend specific mitigations.

Output:
```markdown
## Leakage Analysis & Recommended Defenses

### Top Leakage Vectors Found
1. Broad summary queries returned 12 entities/query on average (vs 3 for targeted queries)
2. Pairwise "relationship between X and Y" queries confirmed 78% of candidate edges
3. Responses included entity names verbatim from the graph

### Recommended Mitigations
- **Response entity cap**: Limit responses to reference at most 3-5 entities per answer
- **Query throttling**: Detect repetitive entity-probing patterns and rate-limit
- **Entity abstraction**: Replace specific entity names with categories in responses
  when the query doesn't specifically name them
- **Subgraph noise injection**: Add 10-20% decoy relationships to retrieved subgraphs
  before generating the response (degrades attacker precision)
- **Query diversity enforcement**: Reject queries that are semantically too similar
  to recent queries from the same session
```

## Best Practices

- **Do:** Always confirm authorization before running extraction against any system. This is a security testing tool, not an attack tool.
- **Do:** Split your query budget roughly 50/50 between discovery and validation. Discovery without validation produces noisy graphs; validation without discovery misses entities.
- **Do:** Use domain-specific NER models (scispaCy for biomedical, stanza for general) rather than regex for entity extraction from responses. Regex misses too many entities.
- **Do:** Track novelty scores per query and stop early if three consecutive queries yield zero new entities. The remaining budget is better spent on validation.
- **Avoid:** Sending identical or near-identical queries. GraphRAG systems may cache responses, wasting budget. Always consult graph memory before generating a query.
- **Avoid:** Treating all entity pairs as equally worth validating. Rank by co-occurrence frequency from Stage 1 and validate top-down. Low-frequency co-occurrences are usually noise.

## Error Handling

- **Rate limiting / 429 errors**: Implement exponential backoff with jitter. Reduce query rate but do not waste budget on retries of queries that will return the same result.
- **Empty or irrelevant responses**: If the GraphRAG returns "I don't know" or off-topic text, log it and deprioritize that query direction. Do not retry the same query.
- **Entity resolution failures**: When the same entity appears under different names across responses, maintain an alias table and merge aggressively. Use fuzzy string matching (Levenshtein distance < 3 or embedding cosine similarity > 0.9).
- **Budget exhaustion before convergence**: If the graph has many disconnected components when the budget runs out, report partial results with a coverage warning rather than silently returning an incomplete graph.
- **API format changes**: If the response format changes mid-run (e.g., the system starts returning structured JSON instead of prose), detect this via parsing failures and switch to the appropriate parser.

## Limitations

- **Requires black-box query access**: AGEA assumes you can send natural language queries and receive responses. It does not apply to systems with no query interface or systems that only return pre-canned answers.
- **Domain-dependent seed quality**: Seed query effectiveness varies by domain. Medical and scientific domains have well-defined entity taxonomies; creative or conversational domains may require more exploratory seeds.
- **Cannot recover graph attributes beyond type**: Node metadata (e.g., numerical properties, embeddings) and edge weights are generally not recoverable from natural language responses.
- **LLM-based validation is imperfect**: The Stage 2 filtering step itself uses LLM reasoning, which can hallucinate relationships. Always report confidence scores and flag low-confidence edges.
- **Defended systems reduce recall significantly**: Systems with response truncation, entity masking, or query throttling can reduce entity recall from 90% to below 40%. The attack efficiency degrades gracefully but noticeably.

## Reference

Yang, S., Zhang, J., Wang, Y., Lee, D., & Wang, S. (2026). *Query-Efficient Agentic Graph Extraction Attacks on GraphRAG Systems.* arXiv:2601.14662v1. [https://arxiv.org/abs/2601.14662v1](https://arxiv.org/abs/2601.14662v1)

Key sections to study: the novelty-guided exploration-exploitation algorithm, the two-stage pipeline architecture, and the experimental results on Microsoft GraphRAG vs. LightRAG showing differential vulnerability across system architectures.