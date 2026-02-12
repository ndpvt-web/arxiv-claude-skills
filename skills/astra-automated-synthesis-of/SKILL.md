---
name: "astra-automated-synthesis-of"
description: |
  Build tool-call graph pipelines, synthesize agentic training trajectories, and create verifiable RL environments
  for training tool-augmented LLM agents using the ASTRA framework. Covers the full pipeline from tool schema
  collection through trajectory synthesis to environment generation for reinforcement learning.
  Trigger phrases:
  - "synthesize tool-use training data"
  - "build a tool-call graph for agent training"
  - "create verifiable RL environments for tool-using agents"
  - "generate agentic trajectories from API schemas"
  - "train a tool-augmented agent with SFT and RL"
  - "convert QA pairs into executable tool environments"
---

# ASTRA: Automated Synthesis of Agentic Trajectories and Reinforcement Arenas

This skill enables Claude to design and implement end-to-end pipelines for training tool-augmented LLM agents, following the ASTRA methodology. ASTRA solves two hard problems simultaneously: (1) generating diverse, structurally grounded tool-use trajectories from API schemas via directed transition graphs, and (2) converting decomposed question-answer traces into sandboxed, code-executable environments where agent behavior can be deterministically verified. The result is a unified SFT-then-RL training pipeline that produces agents competitive with closed-source systems on multi-turn tool-use benchmarks.

## When to Use

- When the user wants to generate synthetic training data for a tool-calling agent from a collection of API or MCP tool schemas
- When the user needs to build a directed tool-call transition graph to discover plausible multi-step tool sequences
- When the user asks to convert a QA dataset (with sub-question decompositions) into executable Python tool environments for RL training
- When the user is designing a reward function that balances task completion (recall) against tool invocation efficiency (precision) in multi-turn agent rollouts
- When the user wants to augment tool-use training with "near-miss distractor" tools sampled by semantic similarity bands to improve tool discrimination
- When the user is building an SFT + RL training pipeline for agentic tool use and needs the data synthesis stages before training

## Key Technique

**Tool-Call Graph Trajectory Synthesis.** ASTRA constructs a directed transition graph G = (V, E, w) where each node is a tool and each weighted edge represents an observed consecutive invocation between two tools. Given a set of tool schemas (normalized to a common format), the pipeline first synthesizes task-conditioned execution sequences, then aggregates them into this graph. New candidate tool-chains are discovered by performing length-bounded random walks biased by edge weights, producing structurally diverse multi-step trajectories that reflect realistic usage patterns rather than random permutations. Each trajectory is then scored across three dimensions -- question quality, scenario realism, and tool-use necessity -- with low-quality candidates discarded.

**Environment Synthesis from QA Decomposition.** The RL component takes decomposed QA instances (a main question q0 with answer a0, plus intermediate sub-questions with dependency graph G) and transforms each sub-question into an independent Python tool: it generates a tool specification, synthesizes an implementation, executes it in a sandbox to verify the output matches the expected answer, then merges homogeneous sub-questions via database expansion. The result is a set of deterministic, rule-verifiable environments where an agent's tool calls can be executed and checked programmatically -- no human evaluation or LLM-as-judge needed.

**F1-Style Trajectory Reward.** For RL training, ASTRA uses an F1-style reward combining recall (fraction of sub-tasks solved: n_solved/n_total) and precision (sub-tasks solved per tool call: n_solved/(n_calls + epsilon)). This jointly incentivizes solving all sub-tasks while penalizing redundant or unnecessary tool invocations, avoiding the common failure mode where agents over-call tools to hedge their bets.

## Step-by-Step Workflow

1. **Collect and normalize tool schemas.** Gather API/MCP tool documentation from target sources. Normalize each tool's schema into a uniform format (OpenAI function-calling compatible) with name, description, and parameter JSON schema. Filter to retain only tool servers with >= 3 tools and clear documentation.

2. **Build the tool-call transition graph.** For each tool server, synthesize initial task-conditioned execution sequences by prompting an LLM with the tool schemas. Aggregate all sequences into a directed graph G = (V, E, w) where edge weights reflect co-invocation frequency. This graph encodes the static topology of how tools relate.

3. **Generate candidate tool-chains via graph walks.** Perform length-bounded random walks on G biased by edge weights. For each walk, verify dependency constraints (a tool's required inputs are available from prior tool outputs). Discard chains with unresolvable dependencies.

4. **Construct and augment tasks.** For each valid tool-chain, generate a natural-language task that would require exactly that chain. Augment via three strategies: diversity-conditioned (vary domains), complexity-conditioned (vary number of required tools), and persona-conditioned (vary user profiles). Enforce language consistency between task and tool documentation.

5. **Score and filter candidate tasks.** Evaluate each (task, tool-chain) pair on question quality, scenario realism, and tool-use necessity using an LLM scorer. Discard pairs below threshold on any dimension. This prevents training on trivially solvable or unrealistic tasks.

6. **Collect trajectories with realistic failure injection.** Execute each task using an agent framework against either live tool servers or a stateful emulator. Inject ~20% tool-call failure rates to approximate real-world unreliability. Record the full multi-turn trajectory (user query, tool calls, tool responses, final answer).

7. **Decompose QA instances for environment synthesis.** Take a QA dataset and decompose each question into atomic sub-questions with a dependency DAG. Validate along four dimensions: dependency consistency, sub-question atomicity, sequential rationality, and task completeness.

8. **Synthesize executable tool environments.** For each sub-question, generate a Python tool specification and implementation. Execute in a sandbox and verify the output matches the expected answer. Merge functionally equivalent sub-questions by expanding tool parameter databases rather than duplicating tools.

9. **Mix irrelevant distractor tools.** For each environment instance, sample K task-irrelevant tools from three semantic similarity bands (high > 0.85, medium 0.4-0.85, low < 0.4 cosine similarity to relevant tools) using an embedding model. This trains the agent to discriminate between relevant and plausible-but-wrong tools.

10. **Train with SFT then RL.** Fine-tune on collected trajectories (SFT phase: 2 epochs, trajectory-level reward scoring across 7 dimensions). Then run online multi-turn RL using GRPO with the F1-style reward, adaptive batch filling (skip batches with reward variance below threshold delta), and rollout termination at 32 turns or 49K tokens.

## Concrete Examples

**Example 1: Building a tool-call graph from MCP server schemas**

User: "I have 50 MCP servers with tool definitions. Help me build a tool-call transition graph to discover useful multi-step tool chains."

Approach:
1. Parse each server's tool definitions into normalized schemas with input/output types
2. For each server, prompt an LLM: "Given these tools, generate 10 plausible 3-5 step task sequences"
3. Aggregate all sequences into adjacency data: for each consecutive pair (tool_a, tool_b), increment edge weight
4. Build directed graph and export as adjacency list or NetworkX object
5. Sample new chains via weighted random walks of length 3-7, filtering for dependency satisfaction

Output:
```python
import networkx as nx
import random

def build_tool_graph(tool_sequences: list[list[str]]) -> nx.DiGraph:
    """Build directed transition graph from observed tool sequences."""
    G = nx.DiGraph()
    for seq in tool_sequences:
        for i in range(len(seq) - 1):
            src, dst = seq[i], seq[i + 1]
            if G.has_edge(src, dst):
                G[src][dst]["weight"] += 1
            else:
                G.add_edge(src, dst, weight=1)
    return G

def sample_tool_chain(G: nx.DiGraph, start: str, max_length: int = 5) -> list[str]:
    """Length-bounded random walk biased by edge weights."""
    chain = [start]
    current = start
    for _ in range(max_length - 1):
        neighbors = list(G.successors(current))
        if not neighbors:
            break
        weights = [G[current][n]["weight"] for n in neighbors]
        total = sum(weights)
        probs = [w / total for w in weights]
        current = random.choices(neighbors, weights=probs, k=1)[0]
        chain.append(current)
    return chain
```

**Example 2: Converting QA pairs into verifiable tool environments**

User: "I have a dataset of multi-hop QA pairs with sub-question decompositions. Convert them into executable tool environments for RL training."

Approach:
1. Parse each QA instance into main question, answer, and sub-questions with dependency DAG
2. Validate decomposition: check dependency consistency, atomicity, sequential rationality, completeness
3. For each sub-question, generate a Python function that takes dependencies as input and returns the answer
4. Wrap each function as a tool with JSON schema for parameters
5. Execute in sandbox and assert output matches expected answer

Output:
```python
# Input QA decomposition:
# q0: "What is the GDP per capita of the country where the Eiffel Tower is located?"
# Sub-questions:
#   q1: "In which country is the Eiffel Tower located?" -> a1: "France" (deps: [])
#   q2: "What is the GDP per capita of France?" -> a2: "44,408 USD" (deps: [q1])

# Generated tool environment:
def get_landmark_country(landmark: str) -> dict:
    """Look up which country a landmark is located in."""
    db = {"Eiffel Tower": "France", "Colosseum": "Italy", "Big Ben": "UK"}
    country = db.get(landmark)
    if country is None:
        return {"error": f"Landmark '{landmark}' not found"}
    return {"country": country}

def get_gdp_per_capita(country: str) -> dict:
    """Retrieve GDP per capita for a given country."""
    db = {"France": "44,408 USD", "Italy": "34,085 USD", "UK": "46,371 USD"}
    gdp = db.get(country)
    if gdp is None:
        return {"error": f"Country '{country}' not found"}
    return {"gdp_per_capita": gdp}

# Verification: deterministic, no LLM judge needed
assert get_landmark_country("Eiffel Tower") == {"country": "France"}
assert get_gdp_per_capita("France") == {"gdp_per_capita": "44,408 USD"}
```

**Example 3: Implementing the F1-style trajectory reward**

User: "Design a reward function for multi-turn RL that balances task completion against tool-call efficiency."

Approach:
1. Define recall as fraction of required sub-tasks successfully solved
2. Define precision as solved sub-tasks per total tool invocations
3. Combine via F1 harmonic mean to jointly optimize both objectives
4. Add epsilon to precision denominator to handle zero-call edge case

Output:
```python
def trajectory_reward(
    n_required: int,
    n_solved: int,
    n_tool_calls: int,
    epsilon: float = 1e-6,
) -> float:
    """F1-style reward balancing completion and efficiency.

    Recall rewards solving all sub-tasks.
    Precision penalizes redundant tool calls.
    F1 harmonic mean prevents gaming either metric alone.
    """
    if n_required == 0:
        return 1.0 if n_tool_calls == 0 else 0.0

    recall = n_solved / n_required
    precision = n_solved / (n_tool_calls + epsilon)

    if recall + precision == 0:
        return 0.0

    return 2 * precision * recall / (precision + recall)

# Agent solves 4/5 sub-tasks using 6 tool calls:
# recall = 0.8, precision = 0.667, F1 = 0.727
print(trajectory_reward(5, 4, 6))  # 0.727

# Agent solves 5/5 sub-tasks using 5 tool calls:
# recall = 1.0, precision = 1.0, F1 = 1.0 (optimal)
print(trajectory_reward(5, 5, 5))  # 1.0

# Agent solves 5/5 but uses 15 tool calls (wasteful):
# recall = 1.0, precision = 0.333, F1 = 0.5
print(trajectory_reward(5, 5, 15))  # 0.5
```

## Best Practices

- **Do:** Normalize all tool schemas to a single format (OpenAI function-calling compatible) before building the transition graph. Schema heterogeneity causes silent failures in chain construction.
- **Do:** Inject realistic failure rates (~20%) during trajectory collection. Agents trained only on success paths fail catastrophically when tools return errors in production.
- **Do:** Sample distractor tools from multiple semantic similarity bands (high, medium, low). Training only with obviously irrelevant distractors doesn't teach fine-grained tool discrimination.
- **Do:** Validate QA decompositions on all four dimensions (dependency consistency, atomicity, sequential rationality, completeness) before generating environments. Bad decompositions produce unlearnable environments.
- **Avoid:** Using LLM-as-judge for RL reward signals. The entire point of environment synthesis is to produce deterministic, code-executable verification. Non-deterministic rewards destabilize RL training.
- **Avoid:** Training exclusively with SFT or exclusively with RL. ASTRA's results show the SFT phase provides broad tool-use competence while RL sharpens efficiency and discrimination -- both are needed.

## Error Handling

- **Dependency cycles in tool-chain graphs:** Before sampling walks, run topological sort on the sub-graph of each chain. If cycles exist, break them by removing the lowest-weight edge.
- **Sandbox execution failures during environment synthesis:** If a generated tool implementation fails sandbox execution, retry synthesis up to 3 times with the error message appended to the prompt. If it still fails, skip that sub-question and log it for manual review.
- **Reward variance collapse during RL:** If all rollouts in a batch receive near-identical rewards (Std(R) <= delta), skip the optimization step entirely. Forcing gradient updates on homogeneous batches causes training instability. Use adaptive batch filling to accumulate valid samples before updating.
- **Schema normalization mismatches:** When tool parameter types don't map cleanly between formats (e.g., MCP's "object" vs OpenAI's "json_schema"), default to string type with a description noting the expected structure. Log these for post-hoc correction.

## Limitations

- The trajectory synthesis pipeline requires an LLM capable of generating plausible tool-call sequences from schemas. If the underlying LLM has weak tool-use understanding, the transition graph will encode poor patterns.
- Environment synthesis from QA decomposition works best for factoid and multi-hop questions with clear, verifiable answers. Open-ended or subjective questions cannot be converted into deterministic verification environments.
- The distractor tool mixing strategy relies on a quality embedding model for cosine similarity computation. Poor embeddings will sample irrelevant tools that are either too obvious (no training signal) or semantically identical to relevant tools (impossible discrimination).
- The F1-style reward assumes each sub-task is equally weighted. For real-world tasks where some steps are critical and others are optional, a weighted variant is needed.
- The full pipeline (graph construction, trajectory collection, environment synthesis, SFT, RL) requires substantial compute. The approach is designed for training-time investment, not inference-time application.

## Reference

[ASTRA: Automated Synthesis of agentic Trajectories and Reinforcement Arenas](https://arxiv.org/abs/2601.21558v2) -- Tian et al., 2026. Focus on Section 3 (trajectory synthesis pipeline and tool-call graph construction), Section 4 (environment synthesis from QA decomposition), and Section 5 (F1-style reward and GRPO training details). Code: [github.com/LianjiaTech/astra](https://github.com/LianjiaTech/astra).