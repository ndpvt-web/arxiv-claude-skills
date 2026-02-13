---
name: "graphdancer-training-explore-reason"
description: "Build agentic graph-exploration systems where an LLM navigates heterogeneous knowledge graphs through interleaved reasoning and deterministic function calls, using curriculum-scheduled reinforcement learning. Use when: 'build a graph QA agent', 'navigate a knowledge graph to answer questions', 'multi-hop reasoning over a graph database', 'train an agent to explore graph-structured data', 'agentic RAG over a knowledge graph', 'curriculum RL for graph traversal'."
---

# GraphDancer: Agentic Graph Exploration and Reasoning

This skill enables Claude to build systems where an LLM agent answers complex questions by actively navigating graph-structured knowledge (knowledge graphs, property graphs, ontologies) through a think-act-observe loop. Instead of dumping an entire subgraph into context or relying on embedding similarity, the agent issues precise, deterministic API calls (retrieve nodes, inspect features, traverse typed relations) and reasons about what to explore next. Training uses curriculum reinforcement learning that schedules tasks from easy single-hop lookups to hard multi-hop expansions, producing agents that generalize across unseen graph domains. The technique comes from the GraphDancer paper (Bai et al., 2026).

## When to Use

- When the user wants to build an agent that answers natural-language questions by exploring a knowledge graph (Neo4j, Amazon Neptune, Wikidata, custom property graphs).
- When implementing multi-hop reasoning over structured relational data -- e.g., "Which directors have collaborated with actors who starred in films produced by Company X?"
- When designing a ReAct-style agent whose action space is graph API calls rather than web search or SQL.
- When the user asks to train or fine-tune a small LLM (1B-8B) to navigate graphs, especially with GRPO or PPO-based RL.
- When building agentic RAG where the retrieval step must follow typed schema relations, not just vector similarity.
- When the user needs cross-domain generalization: train on one graph schema, deploy on another.

## Key Technique

**Interleaved Reasoning and Deterministic Graph APIs.** GraphDancer defines exactly four atomic operations for graph interaction: `RetrieveNode(text)` returns node IDs matching a query; `NodeFeature(node_id, feature_name)` returns a textual attribute; `NeighborCheck(node_id, neighbor_type)` returns IDs of typed neighbors; `NodeDegree(node_id, neighbor_type)` returns the count. The agent generates `<think>` blocks for reasoning, `<graph>` blocks containing one or more API calls, receives `<information>` blocks with results, and eventually emits an `<answer>` block. This structured action space replaces free-form text generation with precise, auditable graph traversals.

**Curriculum RL with Gaussian Biased Sampling.** Vanilla RL on graph tasks suffers from sparse rewards -- most random exploration trajectories fail. GraphDancer classifies question difficulty by the structural complexity of the required information-seeking trajectory. A *singleton round* (S-round) introduces one new node; a *neighborhood expansion round* (E-round) brings multiple nodes via relation traversal. Questions are bucketed: Easy (single round), Medium (multiple rounds, at most one E-round), Hard (two+ E-rounds). Training uses a Gaussian mixture scheduler that initially oversamples Easy questions, then smoothly shifts the distribution toward Hard ones. This produced a 3B model that matched or beat 14B and GPT-4o-mini baselines, with the biggest gains on Medium-difficulty tasks (55% vs 22% Rouge-L over vanilla RL).

**Cross-Domain Generalization.** Because the four API calls are schema-agnostic, an agent trained on a single domain (e.g., an academic citation graph) transfers to unseen domains (e.g., biomedical KGs, movie databases) and out-of-distribution question types without retraining.

## Step-by-Step Workflow

1. **Define the graph schema and API layer.** Map your graph database to the four canonical operations. For Neo4j, implement `RetrieveNode` as a full-text index query, `NodeFeature` as a property lookup, `NeighborCheck` as a typed relationship traversal, and `NodeDegree` as a count query. Expose these as callable functions with typed signatures.

```python
# Example: Neo4j-backed graph API
from neo4j import GraphDatabase

class GraphAPI:
    def __init__(self, driver):
        self.driver = driver

    def retrieve_node(self, text: str, limit: int = 5) -> list[str]:
        """Return node IDs matching a text query via full-text index."""
        query = "CALL db.index.fulltext.queryNodes('entity_index', $text) YIELD node RETURN elementId(node) LIMIT $limit"
        with self.driver.session() as s:
            return [r["elementId(node)"] for r in s.run(query, text=text, limit=limit)]

    def node_feature(self, node_id: str, feature: str) -> str:
        """Return a specific property of a node."""
        query = "MATCH (n) WHERE elementId(n) = $nid RETURN n[$feat] AS val"
        with self.driver.session() as s:
            result = s.run(query, nid=node_id, feat=feature).single()
            return str(result["val"]) if result else "NOT_FOUND"

    def neighbor_check(self, node_id: str, rel_type: str) -> list[str]:
        """Return IDs of neighbors connected by a typed relation."""
        query = "MATCH (n)-[r]->(m) WHERE elementId(n) = $nid AND type(r) = $rel RETURN elementId(m)"
        with self.driver.session() as s:
            return [r["elementId(m)"] for r in s.run(query, nid=node_id, rel=rel_type)]

    def node_degree(self, node_id: str, rel_type: str) -> int:
        """Return count of neighbors under a typed relation."""
        query = "MATCH (n)-[r]->() WHERE elementId(n) = $nid AND type(r) = $rel RETURN count(r) AS deg"
        with self.driver.session() as s:
            return s.run(query, nid=node_id, rel=rel_type).single()["deg"]
```

2. **Build the agent loop with structured block parsing.** Implement a ReAct-style loop that prompts the LLM, parses `<think>`, `<graph>`, and `<answer>` blocks, executes graph calls, and appends `<information>` blocks back into context. Cap iterations (8-15 rounds) to prevent runaway exploration.

```python
import re

def run_agent(llm, graph_api, question: str, max_rounds: int = 10) -> str:
    system = build_system_prompt(graph_api.schema())
    messages = [{"role": "user", "content": question}]

    for _ in range(max_rounds):
        response = llm.generate(system=system, messages=messages)
        messages.append({"role": "assistant", "content": response})

        # Check for final answer
        answer_match = re.search(r"<answer>(.*?)</answer>", response, re.DOTALL)
        if answer_match:
            return answer_match.group(1).strip()

        # Execute graph calls
        graph_calls = re.findall(r"<graph>(.*?)</graph>", response, re.DOTALL)
        if graph_calls:
            results = execute_graph_block(graph_api, graph_calls[0])
            messages.append({"role": "user", "content": f"<information>{results}</information>"})
        else:
            messages.append({"role": "user", "content": "<information>No function call detected. Use <graph> blocks to call graph APIs, or <answer> to respond.</information>"})

    return "MAX_ROUNDS_REACHED"
```

3. **Write the system prompt exposing the graph schema.** Include the available node types, relation types, feature names, and the four API signatures. Instruct the model to reason in `<think>` blocks before acting.

4. **Classify training questions by structural difficulty.** Annotate or automatically compute the minimum trajectory: count how many S-rounds (single node introductions) and E-rounds (neighborhood expansions) are needed. Bucket into Easy / Medium / Hard.

```python
def classify_difficulty(trajectory: list[dict]) -> str:
    e_rounds = sum(1 for step in trajectory if step["type"] == "neighbor_check" and len(step["result"]) > 1)
    total_rounds = len([s for s in trajectory if s["type"] in ("retrieve_node", "neighbor_check")])
    if total_rounds <= 1:
        return "easy"
    elif e_rounds <= 1:
        return "medium"
    else:
        return "hard"
```

5. **Implement the Gaussian curriculum sampler.** Use a time-varying mixture that blends a difficulty-biased distribution with a uniform one. Early in training, the Gaussian mean sits at Easy; it shifts toward Hard over epochs.

```python
import numpy as np

class CurriculumSampler:
    def __init__(self, questions_by_difficulty: dict[str, list]):
        self.buckets = questions_by_difficulty  # {"easy": [...], "medium": [...], "hard": [...]}
        self.difficulty_map = {"easy": 0.0, "medium": 0.5, "hard": 1.0}

    def sample(self, epoch: int, total_epochs: int, batch_size: int) -> list:
        # Gaussian center moves from 0 (easy) to 1 (hard) over training
        mu = epoch / total_epochs
        sigma = 0.3
        samples = []
        for _ in range(batch_size):
            d = np.clip(np.random.normal(mu, sigma), 0, 1)
            if d < 0.33:
                bucket = "easy"
            elif d < 0.66:
                bucket = "medium"
            else:
                bucket = "hard"
            samples.append(np.random.choice(self.buckets[bucket]))
        return samples
```

6. **Set up the RL training loop.** Use GRPO (Group Relative Policy Optimization) or PPO. The reward is computed at the end of each episode: +1 for a correct final answer (measured by Rouge-L or exact match against ground truth), with a small penalty for trajectory length to encourage efficiency.

7. **Generate rollouts by running the agent against sampled questions.** Collect complete trajectories (all think/graph/information/answer blocks) as training data. Use the graph API execution as the environment step.

8. **Train with the curriculum schedule.** Run RL for N epochs, drawing batches from the curriculum sampler. Monitor per-difficulty success rates; if Easy accuracy plateaus above 80%, accelerate the shift toward harder questions.

9. **Evaluate on held-out domains.** Test on graph schemas the model has never seen. The four canonical APIs remain the same, only the schema metadata (node types, relation types, feature names) changes. Measure Rouge-L against reference answers, segmented by difficulty tier.

10. **Deploy as an agentic service.** Wrap the trained model in the agent loop from step 2, connect to your production graph database, and serve queries. Add logging of each reasoning step for auditability.

## Concrete Examples

**Example 1: Building a graph QA agent over a movie knowledge graph**

User: "I have a Neo4j movie database with Person, Movie, and Genre nodes. I want an agent that can answer questions like 'What genres has the director of Inception also directed films in?'"

Approach:
1. Implement the four graph APIs against Neo4j (as shown in step 1).
2. Build the system prompt listing node types (`Person`, `Movie`, `Genre`), relation types (`DIRECTED`, `ACTED_IN`, `HAS_GENRE`), and features (`name`, `title`, `year`).
3. Implement the agent loop with `<think>`/`<graph>`/`<information>`/`<answer>` parsing.

Agent trajectory for "What genres has the director of Inception also directed films in?":
```
<think>I need to find the movie "Inception", then find its director, then find all movies
that director has directed, then get the genres of those movies.</think>
<graph>RetrieveNode("Inception")</graph>
<information>["movie:inception_2010"]</information>
<think>Found Inception. Now get its director via the DIRECTED relation (reversed).</think>
<graph>NeighborCheck("movie:inception_2010", "DIRECTED_BY")</graph>
<information>["person:christopher_nolan"]</information>
<think>Christopher Nolan directed Inception. Now find all movies he directed.</think>
<graph>NeighborCheck("person:christopher_nolan", "DIRECTED")</graph>
<information>["movie:inception_2010", "movie:interstellar_2014", "movie:the_dark_knight_2008",
"movie:tenet_2020", "movie:memento_2000"]</information>
<think>Now I need genres for each of these movies. Let me check their genres.</think>
<graph>
NeighborCheck("movie:interstellar_2014", "HAS_GENRE")
NeighborCheck("movie:the_dark_knight_2008", "HAS_GENRE")
NeighborCheck("movie:tenet_2020", "HAS_GENRE")
NeighborCheck("movie:memento_2000", "HAS_GENRE")
NeighborCheck("movie:inception_2010", "HAS_GENRE")
</graph>
<information>["genre:scifi", "genre:drama"], ["genre:action", "genre:crime", "genre:drama"],
["genre:action", "genre:scifi", "genre:thriller"], ["genre:thriller", "genre:mystery"],
["genre:scifi", "genre:action", "genre:thriller"]</information>
<think>Collecting unique genres: Sci-Fi, Drama, Action, Crime, Thriller, Mystery.</think>
<answer>Christopher Nolan, the director of Inception, has directed films in the following
genres: Sci-Fi, Drama, Action, Crime, Thriller, and Mystery.</answer>
```

**Example 2: Training a small LLM with curriculum RL for biomedical graph reasoning**

User: "I want to fine-tune Qwen3-3B to answer multi-hop questions over a biomedical knowledge graph (diseases, drugs, genes, pathways)."

Approach:
1. Wrap the biomedical KG with the four canonical APIs.
2. Generate training questions at three difficulty tiers:
   - Easy: "What is the mechanism of action of Metformin?" (1 S-round: retrieve drug, read feature)
   - Medium: "Which genes are targeted by drugs that treat Type 2 Diabetes?" (2 rounds, 1 E-round)
   - Hard: "Find pathways shared between genes targeted by Drug A and genes associated with Disease B" (3+ rounds, 2+ E-rounds)
3. Collect reference trajectories using a strong teacher model (GPT-4o) as seed data.
4. Initialize Qwen3-3B with SFT on the seed trajectories.
5. Run GRPO with the Gaussian curriculum sampler over 5000 training steps.
6. Evaluate on a held-out clinical-trials graph domain the model never saw during training.

Expected outcome: The 3B model handles 2-3 hop biomedical questions with >50% Rouge-L, generalizing to the unseen clinical-trials schema because the API interface is identical.

**Example 3: Deploying a graph-exploration agent without RL training**

User: "I don't want to train a model. Can I use this approach with Claude or GPT-4 directly?"

Approach:
1. Implement the four graph APIs for your database.
2. Write a system prompt that describes the APIs, the schema, and the `<think>`/`<graph>`/`<answer>` format.
3. Use the agent loop from step 2 with Claude as the LLM backbone.
4. No curriculum or RL needed -- the structured action space and think-act-observe loop alone provide significant gains over dumping raw graph data into context.

This is the "prompting baseline" from the paper. It works well for capable models and is the fastest path to production.

## Best Practices

- **Do:** Keep the four API calls deterministic and schema-agnostic. The power of this approach is that the same action space works across any graph. Avoid leaking domain-specific logic into the API layer.
- **Do:** Include the full graph schema (node types, relation types, feature names) in the system prompt. The agent cannot navigate what it does not know exists.
- **Do:** Batch multiple graph calls in a single `<graph>` block when they are independent (e.g., fetching features for several nodes). This reduces round trips and context length.
- **Do:** Start curriculum RL with SFT warmup on teacher-generated trajectories. Cold-start RL on graph tasks has extremely sparse rewards.
- **Avoid:** Dumping entire subgraphs into context. The point of the agentic approach is selective, targeted exploration. Large context dumps defeat the purpose and hit token limits.
- **Avoid:** Using embedding similarity as the primary retrieval mechanism for graph-structured data. Typed relations carry semantic meaning that vector similarity misses (e.g., "directed" vs. "acted in" are structurally different traversals).

## Error Handling

- **Dead-end traversal:** The agent reaches a node with no outgoing relations of the needed type. Handle by returning an explicit "NO_RESULTS" in the `<information>` block so the model can backtrack in its reasoning.
- **Node not found:** `RetrieveNode` returns empty results. Return a clear message and let the agent reformulate its query with alternative terms.
- **Excessive rounds:** The agent loops without converging. Enforce a hard cap (10-15 rounds) and return the best partial answer. Log the trajectory for debugging.
- **Schema mismatch:** The agent calls a relation type that doesn't exist. Validate API calls against the schema before execution and return an error listing valid relation types for that node type.
- **Ambiguous node retrieval:** Multiple nodes match a query. Return all matches with distinguishing features so the agent can select the correct one in its next `<think>` block.

## Limitations

- **Requires a well-defined graph schema.** This approach assumes your knowledge is organized as a typed property graph with explicit relations. It does not apply to unstructured document collections or flat key-value stores.
- **Cold-start RL is fragile.** Without SFT warmup or a curriculum, RL training rarely converges for models under 7B parameters. Budget for teacher-generated seed data.
- **Token cost of multi-hop trajectories.** Complex questions can require 8-15 reasoning rounds, each adding to the context window. Long trajectories get expensive with large commercial models.
- **Graph API latency.** Each round-trip to the database adds latency. For real-time applications, ensure your graph database supports sub-100ms queries and consider caching frequent lookups.
- **Training data generation.** You need question-answer pairs annotated with (or reducible to) difficulty tiers. Generating these for a new domain requires effort.

## Reference

[GraphDancer: Training LLMs to Explore and Reason over Graphs via Curriculum Reinforcement Learning](https://arxiv.org/abs/2602.02518v1) (Bai et al., 2026). Key sections: Section 3 for the four canonical graph APIs, Section 4 for the Gaussian curriculum sampler and difficulty classification, and Table 2 for cross-domain generalization results showing 3B matching 14B baselines.