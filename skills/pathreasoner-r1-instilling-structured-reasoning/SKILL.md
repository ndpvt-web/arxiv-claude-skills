---
name: "pathreasoner-r1-instilling-structured-reasoning"
description: "Build knowledge-graph-guided structured reasoning pipelines for vision-language models in computational pathology. Implements trajectory-masked SFT, GRPO-based reinforcement learning with entity-aware rewards, and multi-stage CoT generation constrained by medical knowledge graphs. Use when: 'build a pathology reasoning pipeline', 'add knowledge graph rewards to VLM training', 'generate structured chain-of-thought for medical images', 'implement entity-aware reward functions', 'create a KG-constrained data generation pipeline', 'train a reasoning model with GRPO and entity rewards'."
---

# PathReasoner-R1: Knowledge-Graph-Guided Structured Reasoning for Pathology VLMs

This skill enables Claude to implement the PathReasoner-R1 pipeline: a system that instills structured, evidence-linked reasoning into vision-language models for computational pathology. The core innovation is a three-component architecture — (1) a knowledge-graph-constrained data generation pipeline that produces verifiable chain-of-thought training samples, (2) trajectory-masked supervised fine-tuning that teaches autoregressive logic recovery instead of memorization, and (3) Group Relative Policy Optimization (GRPO) with a knowledge-aware entity reward that aligns model outputs to medical knowledge graphs. This skill teaches you to build each component and wire them together.

## When to Use

- When building a training data pipeline that uses knowledge graphs (PrimeKG, PathoGraph, or custom) to generate structured reasoning chains for medical VLMs
- When implementing trajectory-masked SFT to augment chain-of-thought datasets by truncating reasoning at random steps
- When designing a multi-granular reward function for reinforcement learning that includes format, semantic, and entity-level components
- When integrating soft-Dice entity matching with BioBERT embeddings into an RL reward signal
- When constructing GRPO training loops for domain-specific VLMs that need logically consistent, evidence-grounded outputs
- When adding structured `<think>`, `<observe>`, `<answer>` formatting to medical reasoning model outputs
- When building quality-filtering stages for distilled CoT datasets (logical consistency, visual dependency, reasoning sufficiency)

## Key Technique

**Knowledge-Graph-Constrained CoT Generation.** PathReasoner-R1 solves the "black box diagnosis" problem by forcing reasoning chains to traverse real medical knowledge graph paths. Two graphs are merged — PrimeKG (4M edges of macro-clinical context: diseases, genes, phenotypes) and PathoGraph (micro-scale histopathological entities) — joined via UMLS/MONDO ID exact matching (85%) and BioBERT cosine similarity (threshold >0.85) for the remainder. Named entities extracted from diagnostic reports are mapped to graph nodes as starting points, and shortest-path retrieval along diagnostic-logic edges (`hasSupportEvidence`, `hasContradictEvidence`) produces visual-to-clinical reasoning chains. These paths constrain an LLM generator to explicitly reference physical entities and phenotypes before deriving diagnoses, yielding 22K verified samples.

**Trajectory-Masked SFT + GRPO with Entity Reward.** The SFT stage takes each reasoning chain `R = [s1, s2, ..., sL]` and creates L truncated variants, expanding 22K samples to ~200K. The model learns to recover complete reasoning from partial chains — autoregressive logic recovery rather than rote memorization. The RL stage uses GRPO, sampling G outputs per query and computing group-normalized advantages `A_i = (R(a_i) - mean) / std`. The reward function combines three signals: (1) **Format Reward** — binary check for `<think>`, `<observe>`, `<answer>` tags; (2) **Semantic Reward** — LLM-judged clinical accuracy in [0,1]; (3) **Entity Reward** — a soft-Dice coefficient `R_entity = 2 * I_soft / (|E_pred| + |E_gt| + epsilon)` where `I_soft` includes exact entity matches plus soft matches (beta=0.5) via BioBERT cosine similarity. This suppresses hallucinated entities while tolerating synonymous variations, improving microscopy-specific accuracy by +5.49% over RL without entity reward.

## Step-by-Step Workflow

### Phase 1: Knowledge Graph Construction & Data Generation

1. **Merge domain knowledge graphs into a unified reasoning graph.** Load PrimeKG and your domain-specific graph (e.g., PathoGraph). Map shared nodes via standardized IDs (UMLS, MONDO, SNOMED). For unmatched nodes, compute BioBERT `cls` embeddings and link pairs with cosine similarity > 0.85. Store the merged graph in NetworkX or Neo4j with typed edges.

2. **Extract structured entities from source reports.** Process diagnostic text with an NER model or LLM prompt to extract named entities (histological findings, morphological features, diagnoses). Map each entity to the nearest knowledge graph node using the same BioBERT matching. These become path start/end nodes.

3. **Retrieve reasoning paths via shortest-path search.** For each (finding_entity, diagnosis_entity) pair, run Dijkstra or BFS on the merged graph, prioritizing diagnostic-logic edge types. The resulting path defines the required reasoning steps: visual observation -> histological entity -> phenotype -> clinical correlation -> diagnosis.

4. **Generate constrained chain-of-thought samples.** Prompt an LLM with the retrieved KG path as a hard constraint: it must reference each intermediate entity in order, use `<think>`, `<observe>`, `<answer>` tags, and ground its reasoning in the path's edge semantics. Store as `(image_features, question, reasoning_chain, answer)` tuples.

5. **Apply three-stage quality filtering.** For each generated sample: (a) verify logical consistency — does the reasoning chain logically entail the answer? (b) test visual dependency — can a text-only model (no image input) reach the same answer? If yes, reject the sample as image-independent. (c) assess reasoning sufficiency — does the chain contain enough supporting evidence?

### Phase 2: Trajectory-Masked SFT

6. **Augment the dataset via trajectory masking.** For each reasoning chain of length L, create L training variants by truncating at positions m=1 to L. The model input becomes `(image, question, steps_1_to_m-1)` and the target is `(steps_m_to_L, answer)`. This scales your dataset ~10x and teaches the model to complete partial reasoning.

7. **Fine-tune with standard cross-entropy loss on augmented data.** Use the loss `L_SFT = -E[sum log pi_theta(y_t | x, q, ctx, y_<t)]` where ctx includes the truncated prefix. Train for 1-2 epochs with learning rate warmup to avoid catastrophic forgetting of base VLM capabilities.

### Phase 3: GRPO Reinforcement Learning

8. **Implement the multi-granular reward function.** Code three reward components:
   - `format_reward(output)`: return 1.0 if output contains all required tags (`<think>`, `<observe>`, `<answer>`), else 0.0
   - `semantic_reward(output, ground_truth)`: LLM-as-judge scoring clinical accuracy and logical consistency, returning float in [0, 1]
   - `entity_reward(output, ground_truth, kg)`: extract entity sets from both, compute soft-Dice: `R = 2 * I_soft / (|E_pred| + |E_gt| + eps)` where `I_soft = |exact_matches| + 0.5 * sum(max_similarity for soft matches)`
   - Composite: `R = R_format + R_semantic + alpha * R_entity` (alpha=1.0)

9. **Run GRPO training loop.** For each query, sample G=8 outputs from the current policy. Compute rewards for each, then group-normalize: `A_i = (R_i - mean(R)) / std(R)`. Update with clipped objective: `L = -mean(min(r_i * A_i, clip(r_i, 1-eps, 1+eps) * A_i)) - gamma * D_KL(pi_theta || pi_ref)`. Use epsilon=0.2, gamma=0.01.

10. **Evaluate on held-out test set and external benchmarks.** Measure BERT Score for reasoning quality, LLM-judged accuracy/quality scores, and downstream task accuracy. Compare entity reward ablations (with vs. without) to validate KG alignment is improving reasoning fidelity.

## Concrete Examples

**Example 1: Building a KG-Constrained Data Generation Pipeline**

User: "I have a medical knowledge graph in Neo4j and 5,000 pathology reports with diagnoses. Help me build a pipeline to generate chain-of-thought reasoning training data."

Approach:
1. Query Neo4j to export the graph as edge list; load into NetworkX
2. Write an NER extraction prompt that pulls histological entities from reports
3. Map extracted entities to graph nodes via BioBERT cosine similarity
4. For each report, retrieve shortest KG path from findings to diagnosis
5. Generate CoT with LLM constrained to reference each path node in order
6. Filter: reject samples where text-only LLM achieves >80% accuracy (no visual grounding)

Output:
```python
import networkx as nx
from sentence_transformers import SentenceTransformer

# Step 1: Load and merge knowledge graphs
G = nx.read_edgelist("primekg_edges.tsv", data=[("relation", str)])
patho_G = nx.read_edgelist("pathograph_edges.tsv", data=[("relation", str)])

encoder = SentenceTransformer("dmis-lab/biobert-v1.1")

def merge_graphs(G1, G2, sim_threshold=0.85):
    """Merge two KGs by matching nodes via ID then embedding similarity."""
    node_map = {}
    g2_embeddings = {n: encoder.encode(n) for n in G2.nodes()}
    for n1 in G1.nodes():
        # Try exact UMLS/MONDO match first
        if n1 in G2.nodes():
            node_map[n1] = n1
            continue
        # Fall back to embedding similarity
        emb1 = encoder.encode(n1)
        best_sim, best_node = 0, None
        for n2, emb2 in g2_embeddings.items():
            sim = cosine_similarity(emb1, emb2)
            if sim > best_sim:
                best_sim, best_node = sim, n2
        if best_sim > sim_threshold:
            node_map[best_node] = n1
    # Merge edges from G2 into G1 using node_map
    for u, v, data in G2.edges(data=True):
        src = node_map.get(u, u)
        tgt = node_map.get(v, v)
        G1.add_edge(src, tgt, **data)
    return G1

merged_graph = merge_graphs(G, patho_G)

# Step 3: Retrieve reasoning paths
def get_reasoning_path(graph, finding_node, diagnosis_node):
    """Shortest path prioritizing diagnostic-logic edges."""
    def weight_fn(u, v, d):
        if d.get("relation") in ("hasSupportEvidence", "hasContradictEvidence"):
            return 0.5  # Prefer diagnostic edges
        return 1.0
    path = nx.dijkstra_path(graph, finding_node, diagnosis_node, weight=weight_fn)
    edges = [(path[i], path[i+1], graph[path[i]][path[i+1]])
             for i in range(len(path)-1)]
    return path, edges

# Step 4: Generate constrained CoT
COT_PROMPT = """Given the following knowledge graph reasoning path, generate a
structured chain-of-thought diagnosis. You MUST reference each entity in order.

KG Path: {path_description}
Diagnostic Report: {report_text}

Format your response as:
<think>Step through the reasoning using each KG node as evidence.</think>
<observe>Describe the morphological findings visible in the tissue.</observe>
<answer>State the diagnosis with supporting logic.</answer>"""
```

**Example 2: Implementing Entity Reward for GRPO**

User: "I'm training a pathology VLM with GRPO. Help me implement the entity reward function that uses knowledge graph alignment."

Approach:
1. Write entity extraction from model outputs using regex + NER
2. Compute soft-Dice with BioBERT soft matching
3. Integrate into composite reward alongside format and semantic rewards

Output:
```python
import re
import numpy as np
from sentence_transformers import SentenceTransformer

encoder = SentenceTransformer("dmis-lab/biobert-v1.1")

def extract_entities(text: str, kg_nodes: set) -> set:
    """Extract entities from model output that match KG nodes."""
    # Remove tags, normalize
    clean = re.sub(r"</?(?:think|observe|answer)>", "", text).lower()
    found = set()
    for node in kg_nodes:
        if node.lower() in clean:
            found.add(node)
    return found

def entity_reward(pred_text: str, gt_text: str, kg_nodes: set,
                  beta: float = 0.5, eps: float = 1e-8) -> float:
    """Soft-Dice entity reward aligned with knowledge graph."""
    e_pred = extract_entities(pred_text, kg_nodes)
    e_gt = extract_entities(gt_text, kg_nodes)

    if not e_pred and not e_gt:
        return 1.0  # Both empty = no entities expected
    if not e_pred or not e_gt:
        return 0.0

    # Exact matches
    exact = e_pred & e_gt
    i_soft = len(exact)

    # Soft matches for non-exact via BioBERT similarity
    pred_only = e_pred - exact
    gt_only = e_gt - exact
    if pred_only and gt_only:
        pred_embs = encoder.encode(list(pred_only))
        gt_embs = encoder.encode(list(gt_only))
        sim_matrix = pred_embs @ gt_embs.T
        # Add best soft match for each predicted entity
        i_soft += beta * np.sum(np.max(sim_matrix, axis=1))

    dice = 2.0 * i_soft / (len(e_pred) + len(e_gt) + eps)
    return float(np.clip(dice, 0.0, 1.0))

def format_reward(text: str) -> float:
    """Binary reward for required structured tags."""
    required = ["<think>", "</think>", "<observe>", "</observe>",
                "<answer>", "</answer>"]
    return 1.0 if all(tag in text for tag in required) else 0.0

def composite_reward(pred: str, gt: str, kg_nodes: set,
                     semantic_score: float, alpha: float = 1.0) -> float:
    """Full PathReasoner-R1 reward: format + semantic + entity."""
    r_fmt = format_reward(pred)
    r_sem = semantic_score  # From LLM-as-judge, [0, 1]
    r_ent = entity_reward(pred, gt, kg_nodes)
    return r_fmt + r_sem + alpha * r_ent
```

**Example 3: Trajectory-Masked SFT Data Augmentation**

User: "I have 20K chain-of-thought samples. Help me implement trajectory masking to augment them for SFT."

Approach:
1. Parse each CoT into discrete reasoning steps
2. Generate all truncation variants
3. Format as (context_prefix, completion_target) pairs

Output:
```python
import re
import random
from typing import List, Tuple

def parse_reasoning_steps(cot: str) -> List[str]:
    """Split a chain-of-thought into discrete reasoning steps."""
    # Split on sentence boundaries within <think> tags
    think_match = re.search(r"<think>(.*?)</think>", cot, re.DOTALL)
    if not think_match:
        return [cot]
    think_text = think_match.group(1).strip()
    # Split on numbered steps or sentence boundaries
    steps = re.split(r"(?:Step \d+[:.]\s*|\n\d+\.\s*|\.\s+(?=[A-Z]))", think_text)
    return [s.strip() for s in steps if s.strip()]

def trajectory_mask_augment(
    samples: List[dict],  # Each: {image_id, question, cot, answer}
    max_variants_per_sample: int = None
) -> List[dict]:
    """Create trajectory-masked variants for SFT training."""
    augmented = []
    for sample in samples:
        steps = parse_reasoning_steps(sample["cot"])
        L = len(steps)
        if L < 2:
            augmented.append(sample)
            continue

        indices = range(1, L + 1)
        if max_variants_per_sample and L > max_variants_per_sample:
            indices = sorted(random.sample(range(1, L + 1), max_variants_per_sample))

        for m in indices:
            prefix = " ".join(steps[:m - 1]) if m > 1 else ""
            completion = " ".join(steps[m - 1:])
            augmented.append({
                "image_id": sample["image_id"],
                "question": sample["question"],
                "context_prefix": f"<think>{prefix}" if prefix else "<think>",
                "completion_target": f"{completion}</think>\n"
                    f"<observe>{sample.get('observe', '')}</observe>\n"
                    f"<answer>{sample['answer']}</answer>",
                "truncation_point": m,
                "total_steps": L,
            })
    return augmented

# Usage: 20K samples -> ~200K augmented training pairs
augmented = trajectory_mask_augment(raw_samples, max_variants_per_sample=10)
print(f"Augmented {len(raw_samples)} -> {len(augmented)} training samples")
```

## Best Practices

- **Do:** Use standardized medical ontology IDs (UMLS, MONDO, SNOMED) as the primary key for cross-graph node matching before falling back to embedding similarity. Exact ID matches are deterministic and reliable; embedding matches introduce noise.
- **Do:** Set the soft-matching beta to 0.5 for entity reward. This penalizes approximate matches at half weight, balancing strictness (suppressing hallucinations) against flexibility (accepting synonyms like "adenocarcinoma" vs "glandular carcinoma").
- **Do:** Filter generated CoT samples for visual dependency — if a text-only model (no image) can answer correctly, the sample doesn't teach visual reasoning and should be discarded.
- **Do:** Use trajectory masking during SFT before RL. The SFT stage provides a strong reasoning initialization; skipping it and going straight to RL produces unstable training.
- **Avoid:** Setting entity reward weight alpha > 1.5. Over-weighting entity alignment causes the model to stuff outputs with KG entities at the expense of coherent narrative reasoning.
- **Avoid:** Using only exact entity matching in the reward. Medical text contains abundant synonymy — strict matching penalizes correct reasoning that uses alternative terminology, degrading training signal quality.

## Error Handling

- **KG path not found between entities:** If no path exists in the merged graph, expand the search to 2-hop neighborhoods or fall back to embedding-based entity chains. Log these cases — they may indicate gaps in graph coverage.
- **Entity extraction recalls zero entities:** If the NER step finds no entities in model output, return entity reward of 0.0 but do not penalize format/semantic rewards. The model may need more SFT warmup.
- **Trajectory masking produces trivial splits:** If a reasoning chain has only 1 step, skip augmentation for that sample. Very short chains don't benefit from masking.
- **GRPO reward variance collapse:** If all G sampled outputs receive near-identical rewards, the advantage normalization produces near-zero gradients. Increase sampling temperature or increase G to restore reward variance.
- **BioBERT similarity threshold too low:** Setting the cross-graph matching threshold below 0.80 introduces spurious node merges. Validate a sample of matches manually before full pipeline runs.

## Limitations

- The entity reward requires a well-populated domain knowledge graph. For domains without established KGs (e.g., rare diseases, novel tissue types), the entity reward degenerates to exact string matching only.
- Trajectory-masked SFT assumes reasoning chains decompose into discrete sequential steps. For holistic or parallel reasoning patterns (e.g., differential diagnosis considering multiple hypotheses simultaneously), masking at arbitrary points may produce incoherent training targets.
- The three-stage quality filtering pipeline requires an LLM-as-judge (GPT-4o in the paper), adding cost and latency. For datasets exceeding 50K samples, consider sampling-based filtering.
- GRPO with G=8 samples per query requires 8x inference per training step. This is computationally expensive for models above 7B parameters without multi-GPU setups.
- The approach is validated on pathology (10 cancer types). Extending to radiology, dermatology, or other imaging modalities requires rebuilding the knowledge graph and entity extraction components from scratch.

## Reference

- **Paper:** [PathReasoner-R1: Instilling Structured Reasoning into Pathology Vision-Language Model via Knowledge-Guided Policy Optimization](https://arxiv.org/abs/2601.21617v1) — Focus on Section 3 (dataset construction pipeline), Section 4.2 (trajectory-masked SFT), and Section 4.3 (knowledge-aware GRPO reward design, especially the soft-Dice entity reward formula).
- **Code:** [https://github.com/cyclexfy/PathReasoner-R1](https://github.com/cyclexfy/PathReasoner-R1)