---
name: "prograph-r1-progress-aware-reinforcement-learning"
description: "Build progress-aware GraphRAG agents that traverse knowledge graphs with structure-aware hypergraph retrieval and dense step-wise rewards. Use when: 'build a GraphRAG pipeline', 'add knowledge graph retrieval to my QA system', 'implement multi-hop reasoning over a knowledge graph', 'train an RL agent for graph traversal', 'improve my RAG system with graph structure', 'implement progress-aware reward shaping for retrieval'."
---

# ProGraph-R1: Progress-Aware Reinforcement Learning for Graph Retrieval-Augmented Generation

This skill enables Claude to build GraphRAG systems that combine structure-aware hypergraph retrieval with progress-based reinforcement learning. Unlike naive RAG that retrieves text chunks by semantic similarity alone, ProGraph-R1 teaches an agent to iteratively traverse a knowledge graph -- jointly scoring candidates by semantic relevance AND graph connectivity -- while receiving dense, step-level reward signals that capture whether each retrieval step actually brings the agent closer to answering the question. The result is a multi-hop QA agent that follows coherent reasoning paths instead of greedily grabbing semantically similar but structurally disconnected facts.

## When to Use

- When the user wants to build a multi-hop question answering system over a knowledge graph or entity-relation dataset.
- When an existing RAG pipeline retrieves semantically relevant but logically disconnected passages, and the user needs structural coherence across retrieval steps.
- When the user asks to implement reinforcement learning for an agentic retrieval loop (search-extract-answer cycles over structured data).
- When the user needs to design a reward function for a retrieval agent that goes beyond binary final-answer correctness.
- When building a pipeline that converts unstructured documents into a knowledge hypergraph and then performs iterative retrieval over it.
- When the user wants to improve Graph-R1 or similar RL-based GraphRAG with denser training signals.

## Key Technique

**Structure-Aware Hypergraph Retrieval.** Standard GraphRAG retrieves entities or triples by embedding similarity to the query. ProGraph-R1 instead builds a hypergraph where each hyperedge groups multiple entities involved in an n-ary relation (extracted by an LLM from source documents). At retrieval time, candidate entities are scored by two factors: (1) semantic similarity between the entity embedding and the aggregated query-entity embedding, and (2) a structural informativeness score inspired by participation coefficients -- entities that appear in many hyperedges globally but few query-relevant hyperedges are down-weighted, while entities that are locally concentrated around the query subgraph are up-weighted. The combined score `R(v, e_i) = S_hat(v, e_i) * I(v)` ranks hyperedges for retrieval, encouraging the agent to follow structurally coherent paths rather than jumping to semantically similar but disconnected nodes.

**Progress-Based Step-Wise Reward Shaping.** Existing RL-based GraphRAG (e.g., Graph-R1) uses only a sparse outcome reward: did the final answer match? This makes credit assignment across a 3-5 step retrieval trajectory extremely difficult. ProGraph-R1 introduces three reward components at each step t: (1) a progress score `r_t^sp` measuring how much the probability of generating the correct answer increased after this step's retrieval, estimated by sampling output sequences before and after; (2) a connectivity score measuring entity overlap between the newly retrieved subgraph and the agent's prior state; and (3) an answer-reachness score measuring entity overlap between the retrieved subgraph and the ground-truth answer entities. The total reward `R_t = r_outcome + lambda_1 * r_t^sp + lambda_2 * r_t^struct` feeds into Group Relative Policy Optimization (GRPO) with step-level advantage modulation, giving the agent dense learning signals at every retrieval step.

**GRPO with Step-Level Advantages.** Rather than computing a single advantage for an entire trajectory, ProGraph-R1 modulates advantages per-step within the GRPO framework. Multiple trajectories are sampled, and at each step t, the advantage is computed relative to the group of trajectories at that same step. This eliminates the need for a separate value model while providing fine-grained credit assignment.

## Step-by-Step Workflow

1. **Build the knowledge hypergraph from source documents.** Use an LLM extractor to parse each document into n-ary relational facts (not just triples). Each fact becomes a hyperedge connecting all involved entities. Store entities as nodes with embeddings (e.g., `bge-large-en-v1.5`) and hyperedges as sets of entity references plus a relation description.

2. **Index entities for hybrid retrieval.** Create both a dense vector index (FAISS or similar) over entity embeddings and a sparse index (BM25) over entity surface forms and relation text. Maintain an adjacency structure mapping each entity to its hyperedges and vice versa.

3. **Implement the agent loop with three actions.** At each step t, the agent can: (a) **Search** -- generate a sub-query, retrieve top-K hyperedges using the structure-aware scoring, and add their entities/relations to the agent's working memory; (b) **Extract** -- select specific facts from retrieved hyperedges to add to the reasoning context; (c) **Finish** -- generate the final answer from accumulated context.

4. **Compute structure-aware retrieval scores.** For each candidate entity v and hyperedge e_i, compute: semantic score `s_v = sim(embed(v), embed(V_query))`, structural informativeness `I(v) = log(1 + |query-relevant hyperedges containing v| / |all hyperedges containing v|)`, and combined score `R(v, e_i) = S_hat(v, e_i) * I(v)`. Rank hyperedges by `R_hat(e_i) = sum over v in e_i of R(v, e_i)`.

5. **Track agent state across steps.** Maintain state `s_t` as the concatenation of: the original question, all sub-queries generated so far, all retrieved subgraphs `G_1..G_t`, and the current reasoning chain. This state is the input to the LLM policy at each step.

6. **Compute step-wise progress rewards during training.** At each step t, estimate the progress score by sampling K completions from the policy conditioned on state up to t vs. state up to t-1, and measuring the change in probability of generating the correct answer. Compute connectivity as `|V(G_t) intersect V(s_<t)| / |V(G_t)|` and answer-reachness as `|V(G_t) intersect V(y*)| / |V(G_t)|`.

7. **Train with GRPO and step-level advantage modulation.** Sample N trajectories per question. For each trajectory and each step, compute the total reward `R_t`. Normalize advantages within the group at each step: `A_hat(i) = (R_t(i) - mean(R_t)) / std(R_t)`. Optimize the policy with clipped likelihood ratios and KL regularization against the reference policy.

8. **Tune hyperparameters.** Set `lambda_1` (progress weight) and `lambda_2` (structure weight) via validation. The paper uses Qwen-2.5-Instruct (3B/7B) as the base LLM. Start with `lambda_1 = lambda_2 = 0.5` and adjust based on validation F1 on a held-out set.

9. **Evaluate on multi-hop benchmarks.** Test on datasets requiring 2-4 hop reasoning (MuSiQue, HotpotQA, 2WikiMultiHopQA). Measure both final answer accuracy (F1/EM) and retrieval quality (subgraph precision/recall).

10. **Deploy the trained agent in an inference loop.** At inference, the trained policy generates sub-queries, retrieves via the hypergraph scorer, accumulates context, and decides when to finish -- all without reward signals, relying on the learned policy.

## Concrete Examples

**Example 1: Multi-hop QA over a company knowledge base**

User: "Build a GraphRAG pipeline that can answer multi-hop questions like 'Who founded the company that acquired the maker of the Model S?' over our internal company docs."

Approach:
1. Extract entities and n-ary relations from company docs using an LLM: `("Tesla", "manufactures", "Model S")`, `("Tesla", "acquired_by", "N/A")`, `("Elon Musk", "founded", "Tesla")`, etc. Group into hyperedges.
2. Build vector index over entity embeddings and adjacency map.
3. Agent step 1 -- sub-query: "What company makes the Model S?" -> retrieves hyperedge `{Tesla, Model S, manufactures}` with high structural score because `Model S` participates in few hyperedges (high informativeness).
4. Agent step 2 -- sub-query: "Who founded Tesla?" -> retrieves hyperedge `{Elon Musk, Tesla, founded}` with high connectivity score (shares `Tesla` with prior state).
5. Agent step 3 -- Finish: "Elon Musk" with full reasoning chain.

Output:
```json
{
  "answer": "Elon Musk",
  "reasoning_chain": [
    {"step": 1, "query": "maker of Model S", "retrieved": ["Tesla - manufactures - Model S"]},
    {"step": 2, "query": "founder of Tesla", "retrieved": ["Elon Musk - founded - Tesla"]},
    {"step": 3, "action": "finish", "answer": "Elon Musk"}
  ],
  "confidence": 0.94
}
```

**Example 2: Implementing progress-aware reward shaping for an existing retrieval agent**

User: "My Graph-R1 agent gets 45% F1 on MuSiQue. Help me add progress-based rewards to improve training."

Approach:
1. Keep the existing outcome reward (F1 match against ground truth).
2. Add progress scoring at each retrieval step:
```python
def compute_progress_reward(policy, state_before, state_after, answer, n_samples=8):
    """Estimate how much closer this step brought us to the answer."""
    probs_before = sample_answer_prob(policy, state_before, answer, n_samples)
    probs_after = sample_answer_prob(policy, state_after, answer, n_samples)
    return probs_after - probs_before

def compute_structure_reward(retrieved_subgraph, prior_entities, answer_entities):
    """Score connectivity and answer-reachness."""
    new_entities = set(retrieved_subgraph.entities)
    connectivity = len(new_entities & prior_entities) / max(len(new_entities), 1)
    reachness = len(new_entities & answer_entities) / max(len(new_entities), 1)
    return connectivity + reachness
```
3. Combine: `R_t = r_outcome + 0.5 * r_progress + 0.5 * r_structure`.
4. Modify GRPO to compute per-step advantages instead of trajectory-level.

Output: Expected improvement from ~45% to ~52-55% F1 on MuSiQue based on paper results showing consistent gains over Graph-R1 baselines.

**Example 3: Building a hypergraph index from unstructured text**

User: "I have 10K Wikipedia articles. Help me build the hypergraph structure ProGraph-R1 uses."

Approach:
1. Chunk articles into passages (~300 tokens each).
2. Use an LLM to extract n-ary relational facts from each passage:
```python
EXTRACTION_PROMPT = """Extract all relational facts from this passage.
For each fact, list ALL involved entities and the relation.
Format: (entity1, entity2, ..., entityN) | relation_description

Passage: {passage}"""
```
3. Build the hypergraph:
```python
import networkx as nx
from sentence_transformers import SentenceTransformer

encoder = SentenceTransformer("BAAI/bge-large-en-v1.5")

# Each extracted fact becomes a hyperedge
hypergraph = {"entities": {}, "hyperedges": []}
for fact in extracted_facts:
    entities = fact["entities"]
    edge = {"id": len(hypergraph["hyperedges"]),
            "entities": entities,
            "relation": fact["relation"],
            "source_doc": fact["doc_id"]}
    hypergraph["hyperedges"].append(edge)
    for ent in entities:
        if ent not in hypergraph["entities"]:
            hypergraph["entities"][ent] = {
                "embedding": encoder.encode(ent),
                "hyperedge_ids": set()
            }
        hypergraph["entities"][ent]["hyperedge_ids"].add(edge["id"])
```
4. Precompute global hyperedge participation counts per entity for the informativeness score.

## Best Practices

- **Do:** Extract n-ary relations (not just binary triples) to preserve richer structure -- a single sentence like "X acquired Y for $Z in 2023" should be one hyperedge with all four entities, not three separate triples.
- **Do:** Normalize the informativeness score `I(v)` relative to the query subgraph, not globally. An entity that appears in 100 hyperedges globally but 5 of the 6 query-relevant ones is highly informative for this query.
- **Do:** Sample at least 8 completions when estimating the progress score to reduce variance. Fewer samples make the progress reward noisy and destabilize training.
- **Do:** Warm-start training with supervised fine-tuning on a small set of annotated retrieval trajectories before switching to RL. Cold-start GRPO converges slowly.
- **Avoid:** Using only semantic similarity for retrieval -- this is the core failure mode ProGraph-R1 addresses. Always incorporate the structural informativeness score.
- **Avoid:** Setting `lambda_1` or `lambda_2` too high (> 1.0). The progress and structure rewards are auxiliary signals; they should guide, not dominate, the outcome reward.

## Error Handling

- **Disconnected retrievals:** If the agent retrieves a hyperedge with zero entity overlap with its prior state (connectivity score = 0), log a warning. During training this gets penalized naturally; at inference, consider re-ranking candidates to prefer connected results.
- **Progress score estimation failure:** If all sampled completions produce the same answer probability (zero variance), fall back to using only the structural reward for that step. This happens when the policy is very confident or very uncertain.
- **Hypergraph construction produces too many/few hyperedges:** Target 3-8 entities per hyperedge on average. If extraction produces mostly binary relations, the hypergraph degenerates to a standard graph -- revisit the extraction prompt to encourage n-ary fact extraction. If hyperedges are too large (>15 entities), the informativeness score loses discriminative power.
- **GRPO advantage collapse:** If all trajectories in a group get similar rewards (std approaches zero), the normalized advantage explodes or becomes NaN. Add a small epsilon (1e-8) to the denominator of the advantage normalization.

## Limitations

- **Requires a knowledge graph or the ability to build one.** If the source data is purely unstructured text with no entity-relation structure (e.g., creative writing, raw logs), the hypergraph construction step will produce poor results. Standard chunk-based RAG is more appropriate for such data.
- **Computational cost of progress scoring.** Estimating `r_t^sp` requires multiple forward passes through the LLM at every training step. For large models (>7B), this is expensive. Consider using a smaller proxy model for progress estimation.
- **Ground-truth entity overlap assumption.** The answer-reachness reward assumes the correct answer entities are known during training. This works for QA datasets with gold answers but not for open-ended generation tasks.
- **Hypergraph quality bottleneck.** The entire system depends on the quality of entity/relation extraction. Noisy extraction propagates through retrieval scoring, reward computation, and ultimately trained policy quality.
- **Multi-hop ceiling.** Evaluated on 2-4 hop questions. For very long reasoning chains (>5 hops), the progress reward signal may still be too sparse and the agent state too long for effective policy learning.

## Reference

[ProGraph-R1: Progress-aware Reinforcement Learning for Graph Retrieval Augmented Generation](https://arxiv.org/abs/2601.17755v1) -- Park et al., 2026. Focus on Section 3 (method) for the hypergraph retrieval scoring formulas and Section 3.3 for the step-wise progress reward decomposition into progress, connectivity, and answer-reachness components.