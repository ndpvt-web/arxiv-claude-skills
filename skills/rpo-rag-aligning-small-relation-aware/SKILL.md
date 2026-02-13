---
name: "rpo-rag-aligning-small-relation-aware"
description: "Build knowledge-graph-grounded RAG pipelines that align small LLMs (under 8B params) with relation-aware preference optimization for accurate question answering. Use when: 'build a KG-based QA system', 'improve RAG with knowledge graph reasoning', 'align a small LLM for KGQA', 'implement preference optimization over KG paths', 'answer questions using Freebase/Wikidata subgraphs', 'reduce hallucination with structured knowledge retrieval'."
---

# RPO-RAG: Relation-Aware Preference Optimization for KG-Based RAG

This skill enables Claude to design and implement knowledge-graph-grounded retrieval-augmented generation pipelines that use **semantic-guided path sampling**, **relation-aware preference optimization (RPO)**, and **answer-centered prompt design** to align small language models (1B-8B parameters) for accurate knowledge graph question answering. The approach, from the RPO-RAG paper (WWW 2026), achieves state-of-the-art KGQA accuracy among sub-8B models while keeping inference under 1.5 seconds, making it practical for production systems where large proprietary models are cost-prohibitive.

## When to Use

- When a user wants to build a QA system over a knowledge graph (Freebase, Wikidata, Neo4j, custom KGs) using a small open-source LLM
- When implementing RAG and the retrieval source is structured triples/paths rather than unstructured text
- When a user asks to reduce hallucination in a small LLM by grounding it in KG evidence
- When fine-tuning a model like Llama 3.1-8B or Llama 3.2-3B for multi-hop reasoning over graph data
- When the user needs preference-based alignment (DPO/RPO) where preference pairs come from graph structure rather than human annotations
- When building an end-to-end pipeline from natural language question to KG retrieval to LLM answer generation
- When optimizing path retrieval to be semantics-aware rather than relying on BFS/shortest-path heuristics

## Key Technique

**The core problem:** Standard KG-based RAG systems retrieve paths using semantics-unaware methods (BFS, random walks) and feed them as flat lists to the LLM. This creates two failures: (1) irrelevant paths dilute the reasoning signal, and (2) the LLM is not trained to leverage the *relational structure* of retrieved evidence, only the surface text.

**RPO-RAG's three innovations solve this.** First, *semantic-guided path sampling* replaces random path selection with embedding-based clustering. Candidate paths between the topic entity and answer entities are embedded using a sentence transformer (e.g., all-MiniLM-L6-v2), clustered by cosine similarity with gradient-based inflection-point detection, and then the cluster most similar to the query embedding is selected. This produces training data where positive paths are semantically aligned with the question intent. Second, *relation-aware preference optimization* constructs preference pairs from these clusters: paths from the query-aligned cluster become preferred examples (y+), paths from other clusters become dispreferred (y-), and each is weighted by exponential distance from the cluster centroid: `w+ = exp(-alpha * d)` for preferred, `w- = 1 - exp(-alpha * d)` for dispreferred. The margin-based loss `L = -E[log sigma(w+ * log pi(y+|x) - w- * log pi(y-|x) - gamma)]` teaches the model to discriminate useful KG relations from noise. This is the first framework to incorporate KG relations directly into preference optimization.

**Third, answer-centered prompt design** reorganizes retrieved paths by grouping them under their terminal (candidate answer) entities rather than presenting a flat list. This structure lets the LLM compare evidence *for each candidate answer* side-by-side, dramatically improving multi-hop reasoning. Combined with entity-type filtering (a binary classification head that predicts whether a candidate entity matches the expected answer type), this yields +8.8% F1 on WebQSP and +46% Hit improvement over vanilla small LLMs on CWQ.

## Step-by-Step Workflow

1. **Parse the knowledge graph into a traversable structure.** Load triples (head, relation, tail) from your KG source (Freebase dump, Neo4j export, SPARQL endpoint, or custom JSON-LD). Store them in an adjacency-list format keyed by entity ID, with edges carrying relation labels. For production, use a graph database; for prototyping, a Python dict of `{entity_id: [(relation, neighbor_id), ...]}` suffices.

2. **Identify topic entities via entity linking.** Given a natural language question, detect entity mentions and link them to KG node IDs. Use an existing entity linker (e.g., ELQ, BLINK, or simple string matching against entity labels). Store the linked entity IDs as `eq` (topic entities).

3. **Extract candidate paths using constrained graph traversal.** From each topic entity, perform a breadth-first or depth-limited search (2-3 hops) to collect all paths `P(q, eq, ea)` reaching candidate answer entities. Each path is a sequence of `(entity, relation, entity, ...)` tuples. Cap path count per question (e.g., top 500) to manage compute.

4. **Cluster paths by semantic similarity to the query.** Embed the question and each candidate path (verbalized as a natural language string like "Barack Obama -> born_in -> Honolulu -> located_in -> Hawaii") using a sentence transformer. Compute pairwise cosine similarities among path embeddings, apply gradient-based dynamic clustering (e.g., HDBSCAN or k-means with inflection-point detection on the silhouette score), and select the cluster whose centroid has the highest cosine similarity to the query embedding.

5. **Construct preference pairs for RPO training.** Label paths from the selected (query-aligned) cluster as preferred (y+) and paths from other clusters as dispreferred (y-). Assign weights using centroid distance with exponential decay: `w_preferred = exp(-alpha * dist_to_centroid)`, `w_dispreferred = 1 - exp(-alpha * dist_to_centroid)`. Set alpha empirically (paper uses ~1.0). These weighted pairs form the training signal for relation-aware preference optimization.

6. **Fine-tune the retriever with semantic-matching loss.** Train a bi-encoder retriever (initialized from Sentence-BERT) using the preferred paths as positives and dispreferred as hard negatives. Use contrastive loss: `L_retrieval = -log(exp(sim(q, p+)) / sum(exp(sim(q, p))))`. This produces a retriever that ranks paths by query-semantic relevance rather than graph distance.

7. **Fine-tune the small LLM with the RPO objective.** Using the preference pairs from step 5, fine-tune your target LLM (e.g., Llama 3.1-8B with LoRA) with the relation-aware margin loss. Combine this with a standard answer-generation cross-entropy loss and an entity-type classification loss: `L_total = L_answer + lambda_1 * L_RPO + lambda_2 * L_type`. Use lambda values around 0.1-0.5 for the auxiliary losses.

8. **Build answer-centered prompts at inference time.** When a question arrives, retrieve top-K paths (K=10-30) using the fine-tuned retriever. Group these paths by their terminal entity (each terminal entity is a candidate answer). Format the prompt as:
   ```
   Question: {question}
   Candidate 1: {entity_name}
     - Path: {topic_entity} -> {rel1} -> {mid_entity} -> {rel2} -> {entity_name}
     - Path: {topic_entity} -> {rel3} -> {entity_name}
   Candidate 2: {entity_name}
     - Path: ...
   Answer:
   ```

9. **Apply entity-type filtering to prune unlikely candidates.** Before feeding candidates to the LLM, run the type-classification head to predict whether each candidate entity matches the expected answer type (person, location, date, etc.). Remove candidates below the type-match threshold (0.5) to reduce noise.

10. **Generate the final answer with the fine-tuned LLM.** Pass the answer-centered prompt to the RPO-aligned LLM. The model selects among candidate entities based on aggregated path evidence. For multi-answer questions, use a confidence threshold to return multiple entities.

## Concrete Examples

**Example 1: Building a KGQA pipeline over Freebase for WebQuestions**

User: "I want to build a QA system that can answer questions like 'Where was Obama born?' using Freebase. I only have a 3B parameter model. How should I structure this?"

Approach:
1. Load the Freebase subgraph (2-hop neighborhood around topic entities from the training questions) into a dict-based adjacency structure
2. Use ELQ or the provided entity linking annotations to map "Obama" to `m.02mjmr`
3. Extract all 2-hop paths from `m.02mjmr`, yielding paths like `m.02mjmr -people.person.place_of_birth-> m.0fhp9` (Honolulu)
4. Embed the question "Where was Obama born?" and all candidate paths using all-MiniLM-L6-v2
5. Cluster paths; the query-aligned cluster contains paths with birth/origin relations
6. Train the retriever and fine-tune Llama-3.2-3B with RPO using LoRA (rank=16, alpha=32)
7. At inference, group retrieved paths by candidate answer entity and format the answer-centered prompt

Output:
```
Question: Where was Obama born?
Candidate 1: Honolulu
  - Barack Obama -> people.person.place_of_birth -> Honolulu
  - Barack Obama -> people.person.places_lived -> Honolulu -> location.location.containedby -> Hawaii
Candidate 2: Kenya
  - Barack Obama -> people.person.nationality -> United States (filtered: low type-match for location-of-birth)
Answer: Honolulu
```

**Example 2: Implementing the RPO training loop in PyTorch**

User: "Show me how to implement the relation-aware preference optimization loss function."

Approach:
1. Define the margin-based RPO loss with weighted preference pairs
2. Integrate with standard causal LM loss for answer generation
3. Add entity-type auxiliary loss

Output:
```python
import torch
import torch.nn.functional as F

def rpo_loss(model, tokenizer, preferred_inputs, dispreferred_inputs,
             w_preferred, w_dispreferred, gamma=0.1):
    """
    Relation-aware preference optimization loss.

    Args:
        preferred_inputs: tokenized prompts with preferred (query-aligned) KG paths
        dispreferred_inputs: tokenized prompts with dispreferred KG paths
        w_preferred: tensor of weights exp(-alpha * dist) for preferred paths
        w_dispreferred: tensor of weights 1 - exp(-alpha * dist) for dispreferred
        gamma: margin hyperparameter
    """
    # Get log-probabilities for preferred and dispreferred completions
    with torch.no_grad():
        ref_logp_preferred = get_sequence_logprob(ref_model, preferred_inputs)
        ref_logp_dispreferred = get_sequence_logprob(ref_model, dispreferred_inputs)

    logp_preferred = get_sequence_logprob(model, preferred_inputs)
    logp_dispreferred = get_sequence_logprob(model, dispreferred_inputs)

    # Weighted margin loss
    preferred_score = w_preferred * (logp_preferred - ref_logp_preferred)
    dispreferred_score = w_dispreferred * (logp_dispreferred - ref_logp_dispreferred)

    loss = -F.logsigmoid(preferred_score - dispreferred_score - gamma).mean()
    return loss


def get_sequence_logprob(model, inputs):
    """Compute total log-probability of the completion tokens."""
    outputs = model(**inputs)
    logits = outputs.logits[:, :-1, :]
    labels = inputs["input_ids"][:, 1:]
    log_probs = F.log_softmax(logits, dim=-1)
    token_logprobs = log_probs.gather(-1, labels.unsqueeze(-1)).squeeze(-1)
    mask = inputs["attention_mask"][:, 1:]
    return (token_logprobs * mask).sum(dim=-1)


def combined_training_step(model, batch, lambda_rpo=0.3, lambda_type=0.1):
    """Combined loss: answer generation + RPO + entity type classification."""
    # Standard causal LM loss for answer generation
    answer_loss = F.cross_entropy(
        model(batch["answer_inputs"]).logits.view(-1, vocab_size),
        batch["answer_labels"].view(-1),
        ignore_index=-100
    )

    # RPO loss from preference pairs
    preference_loss = rpo_loss(
        model, tokenizer,
        batch["preferred"], batch["dispreferred"],
        batch["w_pref"], batch["w_dispref"]
    )

    # Entity type binary classification loss
    type_logits = model.type_head(batch["candidate_embeddings"])
    type_loss = F.binary_cross_entropy_with_logits(
        type_logits, batch["type_labels"]
    )

    return answer_loss + lambda_rpo * preference_loss + lambda_type * type_loss
```

**Example 3: Semantic-guided path clustering for retriever training data**

User: "How do I cluster KG paths by semantic similarity to build the preference pairs?"

Approach:
1. Embed question and all candidate paths
2. Cluster path embeddings with automatic cluster-count detection
3. Select query-aligned cluster and construct weighted pairs

Output:
```python
from sentence_transformers import SentenceTransformer
from sklearn.cluster import KMeans
from sklearn.metrics import silhouette_score
import numpy as np

encoder = SentenceTransformer("all-MiniLM-L6-v2")

def verbalize_path(path_triples):
    """Convert KG path [(head, rel, tail), ...] to natural language string."""
    parts = []
    for head, rel, tail in path_triples:
        rel_readable = rel.replace(".", " ").replace("_", " ")
        parts.append(f"{head} -> {rel_readable} -> {tail}")
    return " | ".join(parts)

def semantic_cluster_paths(question, candidate_paths, alpha=1.0):
    """
    Cluster candidate paths and return preference pairs with weights.
    """
    # Embed question and paths
    path_strings = [verbalize_path(p) for p in candidate_paths]
    q_emb = encoder.encode([question])[0]
    path_embs = encoder.encode(path_strings)

    # Find optimal cluster count via inflection-point detection
    best_k, best_score = 2, -1
    for k in range(2, min(len(candidate_paths), 10)):
        km = KMeans(n_clusters=k, random_state=42).fit(path_embs)
        score = silhouette_score(path_embs, km.labels_)
        if score > best_score:
            best_k, best_score = k, score

    km = KMeans(n_clusters=best_k, random_state=42).fit(path_embs)

    # Select cluster closest to query
    centroid_sims = [
        np.dot(q_emb, c) / (np.linalg.norm(q_emb) * np.linalg.norm(c))
        for c in km.cluster_centers_
    ]
    best_cluster = np.argmax(centroid_sims)

    # Build weighted preference pairs
    preferred, dispreferred = [], []
    w_pref, w_dispref = [], []

    for i, label in enumerate(km.labels_):
        dist = np.linalg.norm(path_embs[i] - km.cluster_centers_[label])
        if label == best_cluster:
            preferred.append(candidate_paths[i])
            w_pref.append(np.exp(-alpha * dist))
        else:
            dispreferred.append(candidate_paths[i])
            w_dispref.append(1.0 - np.exp(-alpha * dist))

    return preferred, dispreferred, w_pref, w_dispref
```

## Best Practices

- **Do:** Verbalize KG triples into readable natural language before embedding them. Raw triple notation (`m.02mjmr, people.person.place_of_birth, m.0fhp9`) produces poor embeddings. Use entity labels and readable relation names.
- **Do:** Use answer-centered grouping in prompts rather than flat path lists. Ablation shows this alone accounts for ~10 F1 points on WebQSP. Group paths by the candidate answer entity they support.
- **Do:** Apply entity-type filtering before the final LLM call. Pruning candidates whose type does not match the expected answer type (person vs. location vs. date) eliminates a large class of errors cheaply.
- **Do:** Use LoRA (rank 8-16) for fine-tuning small LLMs. Full fine-tuning is unnecessary and the RPO objective works well with parameter-efficient methods.
- **Avoid:** Using BFS shortest-path as the sole retrieval strategy. Shortest paths are often not the most semantically relevant. Semantic clustering consistently outperforms graph-distance heuristics.
- **Avoid:** Setting alpha too high (>2.0) in the exponential weighting. This makes the preference signal too sharp and collapses the training distribution. Values around 0.5-1.5 work best empirically.
- **Avoid:** Feeding more than 30 paths to a sub-8B model. Small LLMs degrade with excessive context. Use 10-20 paths grouped by candidate answer for the best accuracy-latency balance.

## Error Handling

- **Entity linking failures:** If no topic entity is found in the KG, fall back to dense passage retrieval over entity descriptions. Log the failure for retraining the entity linker.
- **Empty path retrieval:** When the subgraph around the topic entity has no paths reaching plausible answers within the hop limit, increase the hop depth by 1 or expand to include paths through high-degree hub entities.
- **Degenerate clusters:** If all paths land in a single cluster (silhouette score near 0), skip clustering and rank paths by direct cosine similarity to the query embedding instead.
- **Type classification disagreement:** When the type head is uncertain (logit near 0), keep the candidate rather than pruning it. False negatives from overly aggressive type filtering are harder to recover from than false positives.
- **Out-of-KG questions:** If the question requires knowledge not in the KG (temporal facts, opinions, computations), detect this via low retrieval confidence scores and fall back to the LLM's parametric knowledge with a disclaimer.

## Limitations

- Requires a pre-existing knowledge graph with reasonable coverage of the question domain. Building the KG itself is out of scope.
- Entity linking quality is a hard prerequisite. Poor entity linking cascades into irrelevant path retrieval regardless of how good the downstream components are.
- The RPO fine-tuning requires GPU resources (one A100 for 8B models, one A6000 for 3B models) and a training set of question-answer pairs mapped to the KG.
- Multi-hop reasoning beyond 3 hops produces combinatorial path explosion. The approach works best for 1-3 hop questions.
- Performance on KGs with very sparse or ambiguous relation labels (e.g., generic "related_to" edges) will degrade since the relation-aware preference signal depends on relation semantics.

## Reference

**Paper:** [RPO-RAG: Aligning Small LLMs with Relation-aware Preference Optimization for Knowledge Graph Question Answering](https://arxiv.org/abs/2601.19225v2) (WWW 2026). Focus on Section 3 (methodology) for the semantic clustering algorithm, Section 3.3 for the RPO loss formulation, and Table 2 for ablation results showing the contribution of each component.