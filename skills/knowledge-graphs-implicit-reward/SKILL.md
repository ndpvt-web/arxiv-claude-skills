---
name: "knowledge-graphs-implicit-reward"
description: "Build compositional reasoning systems that use knowledge graph paths as reward signals to ground LLM reasoning in verified facts. Use when asked to: 'build a multi-hop reasoning pipeline', 'ground LLM answers in a knowledge graph', 'create a KG-based reward model', 'implement path-derived rewards for RL', 'compose reasoning chains from domain axioms', 'train compositional reasoning from short-hop to multi-hop'."
---

# Knowledge Graph Path-Derived Rewards for Compositional Reasoning

This skill enables Claude to design and implement systems where knowledge graphs serve as implicit reward models for LLM reasoning. Instead of training a separate neural reward model or relying solely on final-answer correctness, you extract reward signals directly from paths in a knowledge graph. These path-derived rewards act as a "compositional bridge" -- training a model on short reasoning chains (1-3 hops) and achieving zero-shot generalization to longer chains (4-5+ hops). The technique comes from Kansal & Jha (2026), who demonstrated a 14B model outperforming GPT-5.2 and Gemini 3 Pro on hard multi-hop medical reasoning by grounding RL rewards in KG path validity.

## When to Use

- When the user wants to build a question-answering system that must chain multiple facts together (e.g., "Drug A treats Disease B, which is caused by Gene C -- what drugs target Gene C indirectly?")
- When the user needs to ground LLM outputs in a structured knowledge base to prevent hallucination in specialized domains (medical, legal, scientific)
- When the user asks to design a reward function for RLHF/RLAIF that doesn't require human preference labels
- When the user wants to implement multi-hop reasoning over a graph database (Neo4j, NetworkX, etc.) with an LLM
- When the user is building a retrieval-augmented generation system and wants to enforce that the reasoning path is verifiable against a knowledge graph
- When the user asks to create training data from a knowledge graph for fine-tuning or reinforcement learning

## Key Technique

**Knowledge graphs as implicit reward models.** Traditional RL for LLMs uses either human preference labels (expensive, subjective) or outcome-based rewards (only checks the final answer). This paper introduces a third option: derive rewards from the structure of a knowledge graph itself. A KG encodes domain axioms as triples (head, relation, tail) -- e.g., (Metformin, treats, Type2Diabetes). Paths through the graph represent valid chains of reasoning. When an LLM generates a reasoning trace, you can score it by checking whether each intermediate step corresponds to a valid edge or path in the KG. This makes the reward verifiable (the KG is ground truth), scalable (no human labeling), and compositional (rewards attach to intermediate steps, not just final answers).

**The compositional bridge.** The critical insight is that rewarding intermediate reasoning steps -- not just endpoints -- teaches the model to compose axioms. During RL, if you only reward correct final answers, the model may learn shortcuts or memorize patterns without understanding the reasoning chain. By rewarding each hop along a valid KG path, you force the model to internalize the compositional structure. This means a model trained on 1-3 hop paths can generalize zero-shot to 4-5 hop queries, because it learned to chain individual reasoning steps rather than memorize fixed-length patterns.

**Bottom-up training pipeline.** The approach uses two phases: (1) Supervised fine-tuning (SFT) on KG-derived question-answer pairs to teach the model the domain vocabulary and basic single/short-hop reasoning, then (2) Reinforcement learning with path-derived rewards to teach compositional chaining. The SFT phase grounds the model in axiomatic facts; the RL phase teaches it to compose those facts into novel reasoning chains it never saw during training.

## Step-by-Step Workflow

1. **Ingest and normalize the knowledge graph.** Load the domain KG into a graph structure (NetworkX, Neo4j, or a triple store). Normalize entity names, deduplicate edges, and verify edge types. Store as an adjacency structure that supports efficient path queries.

2. **Extract reasoning paths at varying hop lengths.** Use BFS or DFS from sampled seed nodes to extract paths of length 1 (single-hop), 2, and 3 hops. Each path is a sequence of (entity, relation, entity) triples. Partition paths into training set (1-3 hops) and evaluation set (4-5 hops). The evaluation paths must not be subpaths of any training path.

3. **Convert paths to natural-language QA pairs for SFT.** For each path, generate a question from the start node and a chain-of-thought answer that narrates each hop. For example, path `(Aspirin, treats, Headache, symptom_of, Migraine)` becomes: Q: "What condition is treated by a drug that also addresses headaches?" A: "Aspirin treats Headache. Headache is a symptom of Migraine. Therefore, the condition is Migraine." Use templates or an LLM to paraphrase for diversity.

4. **Fine-tune the base model with SFT on the generated QA pairs.** Train on the short-hop QA pairs using standard cross-entropy loss. This phase teaches the model domain vocabulary, entity relationships, and the format for step-by-step reasoning. Use a moderate number of epochs (2-4) to avoid overfitting to template patterns.

5. **Define the path-derived reward function for RL.** The reward function scores a model's generated reasoning trace by: (a) extracting entity-relation-entity triples from the generated text, (b) checking each extracted triple against the KG, (c) assigning partial credit for each valid intermediate triple, and (d) giving a bonus for reaching the correct final answer via a valid path. Formally: `R(trace) = alpha * (valid_hops / total_hops) + beta * final_answer_correct`, where alpha and beta weight intermediate vs. final rewards.

6. **Run RL training with the path-derived reward.** Use PPO, GRPO, or a similar policy gradient method. The model generates reasoning traces for training queries; the path-derived reward function scores them. Key: set alpha > beta to emphasize intermediate step validity over final-answer-only shortcuts. Train on 1-3 hop queries only.

7. **Implement the inference-time reasoning pipeline.** At inference, the model receives a query, generates a step-by-step reasoning chain, and optionally verifies each step against the KG in real time. Build a verification layer that checks extracted triples against the graph and flags unsupported reasoning steps.

8. **Evaluate on held-out multi-hop queries.** Test zero-shot generalization on 4-5 hop queries the model never saw during training. Measure: (a) final answer accuracy, (b) path validity (what fraction of intermediate steps are KG-supported), and (c) robustness to adversarial perturbations like option shuffling.

9. **Iterate on reward calibration.** If the model produces valid but shallow paths (always taking shortest routes), increase the diversity bonus. If it hallucinates intermediate entities, increase the penalty for invalid triples. Tune alpha/beta ratio based on path validity metrics.

## Concrete Examples

**Example 1: Medical Multi-Hop QA System**

User: "I have a medical knowledge graph in Neo4j with drug-disease-gene-pathway relationships. Build me a pipeline that can answer questions like 'What pathways are affected by drugs that treat diseases linked to BRCA1?'"

Approach:
1. Query Neo4j to extract all paths up to 3 hops: `MATCH p=(d:Drug)-[:treats]->(dis:Disease)-[:associated_with]->(g:Gene)-[:participates_in]->(pw:Pathway) RETURN p LIMIT 50000`
2. Convert extracted paths to QA training pairs:
   ```
   Q: "Which pathway is affected by a drug treating a BRCA1-associated disease?"
   Path: (Olaparib, treats, Breast Cancer, associated_with, BRCA1, participates_in, Homologous Recombination)
   A: "Olaparib treats Breast Cancer. Breast Cancer is associated with BRCA1.
       BRCA1 participates in Homologous Recombination. Answer: Homologous Recombination."
   ```
3. Fine-tune a base model on these pairs (SFT phase)
4. Define reward function that checks each triple against Neo4j:
   ```python
   def path_reward(trace, kg_client):
       triples = extract_triples(trace)  # NER + relation extraction
       valid = sum(1 for t in triples if kg_client.edge_exists(t))
       hop_score = valid / max(len(triples), 1)
       final_correct = check_final_answer(trace, expected)
       return 0.7 * hop_score + 0.3 * float(final_correct)
   ```
5. Run RL with this reward on 1-3 hop queries
6. At inference, the model chains: Drug -> Disease -> Gene -> Pathway, with each hop verifiable

Output: A system that answers 4-5 hop queries zero-shot by composing learned 1-3 hop reasoning patterns, with each step traceable to a KG edge.

**Example 2: Building a KG-Grounded Reward Function for Any Domain**

User: "I have a CSV of triples (subject, predicate, object) representing a cybersecurity knowledge graph. Help me build a reward function that scores LLM reasoning chains against this graph."

Approach:
1. Load the triples into a NetworkX directed multigraph:
   ```python
   import networkx as nx
   import csv

   G = nx.MultiDiGraph()
   with open("kg_triples.csv") as f:
       for row in csv.DictReader(f):
           G.add_edge(row["subject"], row["object"], relation=row["predicate"])
   ```
2. Implement triple extraction from free-text reasoning (regex patterns or a small NER model)
3. Build the reward scorer:
   ```python
   def kg_path_reward(reasoning_text, graph, target_answer, alpha=0.7, beta=0.3):
       extracted = extract_entity_relation_triples(reasoning_text)
       if not extracted:
           return 0.0

       valid_hops = 0
       for subj, pred, obj in extracted:
           if graph.has_edge(subj, obj):
               edge_data = graph.get_edge_data(subj, obj)
               if any(d.get("relation") == pred for d in edge_data.values()):
                   valid_hops += 1

       hop_score = valid_hops / len(extracted)
       final_correct = target_answer.lower() in reasoning_text.lower().split("answer:")[-1]
       return alpha * hop_score + beta * float(final_correct)
   ```
4. Integrate with an RL training loop (e.g., trl library's PPOTrainer)

Output: A reusable reward function that scores any LLM reasoning trace against the user's KG, returning a float between 0 and 1 weighted toward intermediate step validity.

**Example 3: Prompt-Only KG-Grounded Reasoning (No Training)**

User: "I don't want to fine-tune a model. Can I still use KG path-derived reasoning at inference time?"

Approach:
1. At query time, retrieve relevant subgraph paths from the KG using the query entities
2. Inject the paths into the prompt as grounding context:
   ```
   You are a reasoning assistant. Use ONLY the following verified knowledge paths
   to answer the question. Cite each hop explicitly.

   Known paths:
   - Metformin --[treats]--> Type 2 Diabetes
   - Type 2 Diabetes --[risk_factor_for]--> Cardiovascular Disease
   - Cardiovascular Disease --[treated_by]--> Statins
   - Statins --[mechanism]--> HMG-CoA Reductase Inhibition

   Question: How does Metformin indirectly relate to HMG-CoA Reductase Inhibition?

   Reason step by step, citing each path hop.
   ```
3. Post-process the LLM output by verifying each cited hop against the KG
4. Flag or reject any reasoning step not supported by the provided paths

Output: Grounded multi-hop answers with full traceability, no fine-tuning required. Each step cites a KG edge, and unsupported claims are flagged.

## Best Practices

- **Do:** Weight intermediate-hop rewards higher than final-answer rewards (alpha > beta). The paper's key finding is that rewarding the process, not just the outcome, is what enables compositional generalization.
- **Do:** Keep training paths short (1-3 hops) and test on longer paths (4+). The compositional bridge only works if the model learns to chain atomic reasoning steps, not memorize fixed-length sequences.
- **Do:** Normalize entity names consistently between the KG and the LLM's input/output. Mismatched entity strings (e.g., "T2D" vs "Type 2 Diabetes") will cause valid hops to be scored as invalid.
- **Do:** Include adversarial robustness checks like option shuffling in your evaluation. A model that is truly reasoning compositionally should be invariant to surface-level perturbations.
- **Avoid:** Using only final-answer rewards during RL. This defeats the purpose of KG-grounded path rewards and leads to shortcut learning.
- **Avoid:** Training on the same hop lengths you evaluate on. The point is zero-shot generalization to unseen compositional depths.
- **Avoid:** Treating the KG as complete. Real KGs have missing edges. Build in a confidence threshold and allow the model to flag "path not found" rather than forcing all reasoning through the graph.

## Error Handling

- **Triple extraction fails on free-text reasoning:** The reward function depends on extracting structured triples from the model's natural language output. Use a robust NER/RE pipeline (e.g., spaCy with a domain-trained model) or enforce structured output format during training (e.g., `[Entity1 | relation | Entity2]` markers).
- **KG is sparse or incomplete:** If many valid reasoning steps aren't in the KG, the reward function will penalize correct reasoning. Mitigate by augmenting the KG with high-confidence inferred edges or by using a softer reward (embedding similarity between generated and KG triples instead of exact match).
- **Model collapses to shortest paths:** If the model always generates minimal-hop answers, add a diversity term to the reward that encourages exploring multiple valid paths for the same query.
- **Entity disambiguation failures:** "Cancer" could refer to multiple KG nodes. Implement entity linking before reward scoring, mapping free-text mentions to canonical KG identifiers.

## Limitations

- Requires a domain-specific knowledge graph of reasonable quality and coverage. If no KG exists for the target domain, this technique cannot be directly applied.
- The triple extraction step from free-text reasoning is a bottleneck. Imperfect extraction leads to noisy rewards.
- KG incompleteness means the reward function has false negatives -- valid reasoning steps may be penalized if the corresponding edge is missing from the graph.
- The compositional bridge was validated primarily in the medical domain. Domains with fundamentally different reasoning structures (e.g., mathematical proof, temporal reasoning) may not see the same generalization gains.
- The full SFT + RL pipeline requires significant compute. The prompt-only approach (Example 3) trades off some compositional generalization for zero training cost.

## Reference

[Knowledge Graphs are Implicit Reward Models: Path-Derived Signals Enable Compositional Reasoning](https://arxiv.org/abs/2601.15160v1) -- Kansal & Jha, 2026. Key sections: the path-derived reward formulation (how KG edges become RL rewards), the compositional bridge analysis (why short-hop training generalizes to long-hop inference), and the adversarial robustness evaluation (option-shuffling stress tests).