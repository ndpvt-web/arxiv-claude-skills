---
name: "sogk-one-token-explicit-graph"
description: "Represent graph topology as a single discrete token (<SOG_k>) for LLM reasoning, replacing verbose graph verbalization. Use when: 'encode graph structure for LLM', 'classify molecular graphs', 'reduce graph token overhead', 'graph structural tokenization', 'single token graph representation', 'topology-aware prompt engineering'."
---

# SOG_k: Single-Token Graph Structural Representation for LLMs

This skill teaches Claude to apply the SOG_k technique — representing an entire graph's topology as one discrete token rather than verbalizing it into lengthy natural language descriptions. The core insight is that graph structures can be discretized through a learned codebook (vector quantization) so that each unique topology maps to a single token `<SOG_k>` in the LLM's vocabulary. This eliminates structural hallucination, cuts token consumption dramatically, and achieves 9.9%–41.4% improvement over verbalization baselines on graph-level classification benchmarks. Use this skill to design, build, or advise on systems that feed graph-structured data into LLMs.

## When to Use

- When the user wants to classify molecular graphs (toxicity, blood-brain barrier permeability, binding affinity) using an LLM rather than a standalone GNN.
- When the user needs to reduce the token cost of encoding graph topology in LLM prompts — replacing adjacency lists or edge descriptions with a compact representation.
- When building a pipeline that aligns structural graph embeddings with an LLM's text token space through hybrid QA fine-tuning.
- When the user asks how to avoid "structural hallucination" — the LLM fabricating edges or misinterpreting connectivity from verbose graph descriptions.
- When extending graph-level representations to node-level tasks (e.g., citation network node classification) while preserving both local and global topology.
- When the user wants to build a topology-aware structural tokenizer that maps graphs to discrete codebook entries using GNN encoding and vector quantization.

## Key Technique

**The Problem with Graph Verbalization.** The standard way to give an LLM graph data is to spell out the structure: "Node A connects to Node B, Node B connects to Node C..." This consumes hundreds of tokens for modest graphs, scatters the model's attention across disconnected textual fragments, and leads to structural hallucination — the LLM invents or forgets edges. Soft-prompt approaches (learning continuous graph embeddings) avoid token bloat but produce representations that live in a different vector space from the LLM's text tokens, causing misalignment.

**SOG_k: One Token per Topology.** The SOG_k method trains a topology-aware structural tokenizer with three components: (1) a node-level encoder that assigns each node a positional attribute relative to an anchor node (e.g., "first-hop neighbor #1") ranked by degree, plus a virtual global node for graph-level pooling; (2) a GNN (GCN) that aggregates these attributes via message passing into a continuous graph-level embedding; and (3) a learnable codebook of K discrete entries that quantizes the continuous embedding via nearest-Euclidean-distance lookup. The codebook index of the global node becomes the graph's structural token `<SOG_k>`. The tokenizer is trained self-supervised to reconstruct the adjacency matrix from the selected codebook entry, using reconstruction loss, codebook update loss, and commitment loss.

**Aligning Structural Tokens with Text.** After tokenization, the new `<SOG_k>` tokens must align with the LLM's existing vocabulary. The paper constructs three types of hybrid QA data: (a) k-Nearest Neighbor Matching — "which tokens are closest to `<SOG_42>` by cosine similarity?"; (b) True/False Similarity Judgment — "do `<SOG_7>` and `<SOG_19>` represent similar structures?"; (c) Description-Token Pairs — "match this textual graph description to its structural token." Two-stage LoRA fine-tuning first aligns token embeddings (stage 1, frozen LLM), then jointly tunes embeddings and the LLM backbone (stage 2). Downstream task prompts then include the structural token inline: `[Structural Token] <SOG_k>`.

## Step-by-Step Workflow

1. **Prepare graph data.** Convert your graphs into adjacency matrices and node feature matrices. Store as pickle files with `graphs.pkl` (adjacency + features) and `graph_attributes.pkl` (labels and metadata) per dataset.

2. **Assign node positional attributes.** For each graph, select the highest-degree node as the anchor. Label every other node by its hop distance and rank within that hop (e.g., "2nd-hop neighbor #3"). Add a virtual global node connected to all real nodes for pooling.

3. **Encode node attributes into continuous vectors.** Use a text encoder (e.g., the LLM's own tokenizer + embedding layer) to embed each node's positional attribute string. Feed these embeddings through a GCN (2–3 layers) with message passing to produce node-level and graph-level representations.

4. **Train the vector-quantized codebook.** Initialize a codebook of K entries (K is the structural vocabulary size — typically 512–4096). Train with adjacency reconstruction: the decoder (MLP) predicts the adjacency matrix from codebook entries. Optimize the combined loss: `L = L_recon + β * L_commit + L_update`. Use `trainrecons.py` then `trainvq.py` from the SOG codebase, specifying the best reconstruction checkpoint.

5. **Assign each graph its `<SOG_k>` token.** Run inference with the trained tokenizer. The global node's nearest codebook entry index `k` becomes the graph's token. Record the mapping `{graph_id: k}` for all graphs.

6. **Generate hybrid QA alignment corpora.** Using `prepare_struct_map.py` and `prepare_struct_corpus.py`, create three files: `graph_desc_token_pairs.json` (description-to-token matching), `struct_token_similarity.jsonl` (true/false similarity pairs), and `struct_code_train_corpus.txt` (k-nearest neighbor matching). These serve as the alignment training data.

7. **Expand the LLM's vocabulary.** Run `addtokens.py` to insert new `<SOG_0>` through `<SOG_{K-1}>` tokens into the LLM tokenizer. Initialize their embeddings from the codebook vectors.

8. **Two-stage LoRA fine-tuning for alignment.** Stage 1 (`pretune1stage.py`): freeze LLM weights, train only the new token embeddings on the hybrid QA data. Stage 2 (`pretune2stage.py`): jointly fine-tune embeddings and LLM backbone with LoRA adapters on the same QA data.

9. **Fine-tune on downstream tasks.** Format each training example as a prompt with three components — task description (P), textual attributes (T), and structural token `<SOG_k>`. Fine-tune with `sfttune.py`, specifying datasets and epoch count. The system prompt should state: "Classify the target graph into the correct category based on its graph topology (i.e., structural token) and the provided attributes."

10. **Evaluate and iterate.** Run `eval_all.py` with multiple random seeds (the paper uses 3 runs) to get robust AUC-ROC or accuracy metrics. Compare against verbalization baselines and soft-prompt methods to confirm the structural token is adding value.

## Concrete Examples

**Example 1: Molecular Toxicity Classification**

User: "I have a dataset of molecular graphs (SMILES converted to adjacency matrices) and I need to predict whether each molecule is toxic. The current approach verbalizes the graph as edge lists in the prompt and it's slow and inaccurate."

Approach:
1. Convert SMILES strings to adjacency matrices and atom-feature vectors using RDKit.
2. Train the SOG structural tokenizer on the molecule graphs: assign positional attributes by hop distance from the highest-degree atom, run GCN encoding, and learn a codebook of 1024 entries.
3. Each molecule receives a single token `<SOG_k>` (e.g., `<SOG_417>`).
4. Generate hybrid QA corpora linking structural tokens to textual descriptions of molecular substructures.
5. Expand the LLM vocabulary with 1024 new tokens, run two-stage LoRA alignment.
6. Fine-tune on the toxicity task with prompts:
   ```
   System: Classify whether the molecule is toxic based on its structural token and attributes.
   User: Task: Predict toxicity. Attributes: C=14, O=3, N=1, ring_count=2. [Structural Token] <SOG_417>
   ```
7. At inference, the LLM sees one structural token instead of 200+ tokens of edge-list verbalization.

Output: Binary classification (toxic/non-toxic) with AUC-ROC improvement of ~10–40% over edge-list verbalization, at a fraction of the token cost.

**Example 2: Citation Network Node Classification**

User: "I want to classify paper topics in the Cora citation network using an LLM. Each paper is a node with text features and citation edges."

Approach:
1. Build the citation graph adjacency matrix from the Cora dataset.
2. For node-level tasks, compute a structural token for each node's local subgraph (ego network at 2-hop depth) using the same tokenizer pipeline.
3. Additionally compute a global graph token for the entire citation network.
4. Each node prompt includes both its local `<SOG_k>` and the global token, plus the paper's text features (bag-of-words or abstract snippet).
5. Fine-tune with prompts:
   ```
   System: Classify the paper into one of 7 categories based on its local structural token and text features.
   User: Task: Paper topic classification. Text features: [word vector summary]. Local structure: <SOG_83>. Global structure: <SOG_5>.
   ```
6. Evaluate with accuracy metric across standard Cora train/val/test splits.

Output: Topic label (e.g., "Neural Networks", "Reinforcement Learning") with ~91.5% accuracy, leveraging both local and global topology.

**Example 3: Advising on Graph-to-LLM Pipeline Design**

User: "I'm designing an LLM pipeline that takes knowledge graphs as input. Should I verbalize the triples or use embeddings?"

Approach:
1. Explain the SOG_k tradeoff: verbalization is simple but consumes O(E) tokens per graph (E = edges) and causes structural hallucination; soft embeddings are compact but misalign with text token space.
2. Recommend the SOG_k middle ground: discrete tokens that live in the same vocabulary as text tokens, consuming O(1) tokens per graph while maintaining alignment.
3. Outline the investment required: training a structural tokenizer (~GCN + VQ codebook), generating alignment QA data, and two-stage LoRA fine-tuning.
4. Identify when SOG_k is overkill: if graphs are very small (< 10 nodes), simple verbalization may suffice. If graphs are very large or dynamic, a streaming/chunking strategy may be needed on top of SOG_k.

Output: Architectural recommendation with concrete next steps and tradeoff analysis.

## Best Practices

- **Do:** Use degree-based ranking for node positional attributes. The highest-degree node as anchor provides the most stable and informative reference frame for the topology.
- **Do:** Initialize new token embeddings from the trained codebook vectors, not randomly. This gives the LLM a meaningful starting point during alignment fine-tuning.
- **Do:** Include all three QA types (k-NN matching, similarity judgment, description pairs) in the alignment corpus. Each teaches a different facet of structural understanding.
- **Do:** Use LoRA for fine-tuning rather than full parameter updates. The paper demonstrates that LoRA provides sufficient adaptation while preserving the LLM's general language capabilities.
- **Avoid:** Skipping the two-stage training and going directly to task fine-tuning. Without alignment, the structural tokens are opaque to the LLM and performance degrades significantly.
- **Avoid:** Setting the codebook size K too small. If K is much smaller than the number of distinct topologies in your data, multiple dissimilar graphs will map to the same token, losing discriminative power. Start with K = 1024 and adjust based on codebook utilization.

## Error Handling

- **Codebook collapse (most entries unused):** Monitor codebook utilization during tokenizer training. If fewer than 30% of entries are active, increase the commitment loss weight β or use codebook reset strategies (re-initialize dead entries from active ones).
- **Poor alignment after stage 1:** If hybrid QA accuracy is low after the first fine-tuning stage, verify that the codebook embeddings were correctly loaded as token initializations. Check embedding dimensionality matches the LLM's hidden size.
- **Structural hallucination persists:** If the LLM still fabricates graph properties despite using `<SOG_k>`, the tokenizer may not be capturing the relevant structural features. Retrain with more GCN layers or add edge-attribute encoding.
- **Out-of-distribution graphs at inference:** If a test graph's topology was never seen during tokenizer training, it will still map to the nearest codebook entry — but this entry may be a poor match. Log the quantization distance and flag graphs above a threshold for manual review or verbalization fallback.

## Limitations

- **Requires training infrastructure.** The structural tokenizer (GCN + VQ) and two-stage LLM fine-tuning require GPU resources. This is not a prompt-only technique — it modifies the model.
- **Fixed codebook at inference.** Once trained, the vocabulary of structural tokens is fixed. New topology types not represented in training data may be poorly captured.
- **Graph-level granularity tradeoff.** The single-token representation is most powerful for graph-level tasks. For fine-grained node-level tasks, you need per-node tokens, which partially reintroduces token scaling with graph size.
- **Domain-specific tokenizer.** A tokenizer trained on molecular graphs will not transfer well to social networks or knowledge graphs without retraining. The codebook learns domain-specific topological patterns.
- **LLM modification required.** Unlike prompt engineering approaches, SOG_k requires vocabulary expansion and fine-tuning, making it incompatible with frozen API-only LLM access.

## Reference

Wu, J., Lu, B., Di, Z., Gan, X., & Jin, M. (2026). *<SOG_k>: One LLM Token for Explicit Graph Structural Understanding.* arXiv:2602.01771v1. [https://arxiv.org/abs/2602.01771v1](https://arxiv.org/abs/2602.01771v1)

Look for: The topology-aware structural tokenizer architecture (Section 3.1), the hybrid QA corpus construction (Section 3.2), the two-stage alignment procedure (Section 3.3), and the ablation study showing the contribution of each QA type (Section 4.4). Codebase: [https://github.com/Jingyao-Wu/SOG](https://github.com/Jingyao-Wu/SOG)