---
name: "autonomous-chain-of-thought-distillation-graph-bas"
description: "Implement FraudCoT-style graph-aware chain-of-thought distillation for fraud detection on text-attributed graphs. Combines LLM reasoning with GNN structural learning via selective CoT distillation and asymmetric co-training. Use when: 'detect fraud in a transaction graph', 'build a graph-based anomaly detector with LLM reasoning', 'distill chain-of-thought into a GNN pipeline', 'enrich graph node features with LLM-generated reasoning', 'co-train an LLM and GNN for fraud classification', 'add CoT explanations to graph fraud detection'."
---

# Autonomous Chain-of-Thought Distillation for Graph-Based Fraud Detection (FraudCoT)

This skill enables Claude to implement the FraudCoT framework — a two-stage pipeline that (1) distills fraud-aware chain-of-thought reasoning from a teacher LLM into a smaller student model using positive/negative path supervision, then (2) co-trains the student LLM with a heterogeneous GNN via an asymmetric caching strategy that reduces LLM inference from O(|neighbors|) to O(1) per node. The result is a fraud detection system on text-attributed graphs (TAGs) that produces interpretable reasoning while leveraging multi-hop relational structure.

## When to Use

- When the user needs to build a fraud detection system over graph-structured data where nodes carry text (reviews, user profiles, transaction descriptions)
- When the user wants to combine LLM-based textual reasoning with GNN-based relational learning for classification on heterogeneous graphs
- When the user asks to generate and distill chain-of-thought reasoning paths for graph node classification
- When the user needs to enrich graph node features with LLM-generated explanations before feeding them into a GNN
- When the user wants to co-train an LLM (via LoRA) and a GNN end-to-end without prohibitive compute costs
- When the user needs interpretable fraud signals that reference both textual cues and neighborhood structure
- When the user is working with Amazon reviews, financial transactions, or social network abuse detection datasets

## Key Technique

**Fraud-Aware Selective CoT Distillation (FASCD):** A teacher LLM (e.g., GPT-4) generates S diverse reasoning paths per labeled node. Each path is scored as *positive* (prediction matches ground truth) or *negative* (mismatch). A smaller student LLM is then fine-tuned with a dual loss: cross-entropy on positive paths teaches correct reasoning, while unlikelihood loss on negative paths suppresses spurious correlations. The combined loss is `L = λ_CE · 1[correct] · L_CE + λ_UL · 1[incorrect] · L_UL`. This produces a student that generates high-quality fraud reasoning autonomously — without hand-crafted prompt templates.

**Node Text Enrichment:** The student's generated CoT reasoning `r_i` is concatenated with the original node text `x_i` to form enriched representations `x̃_i = x_i ⊕ r_i`. This gives the downstream GNN access to multi-hop semantic cues (e.g., "this review mentions a product defect pattern seen in 3 connected accounts") that would be invisible from raw text or graph structure alone.

**Efficient Asymmetric Co-training (EAC):** Jointly training an LLM encoder and GNN is expensive because every neighbor aggregation step requires a fresh LLM forward pass. FraudCoT solves this by caching LLM embeddings for *neighbor* nodes at initialization, while only computing fresh embeddings for *target* nodes each epoch. A heterogeneous GraphSAGE then aggregates cached neighbor embeddings per relation type. Gradients flow only through target node embeddings, updating both LoRA-adapted LLM parameters and GNN weights via binary cross-entropy loss. This achieves up to 1,066x training throughput improvement over naive joint training.

## Step-by-Step Workflow

1. **Model the data as a text-attributed heterogeneous graph.** Define node types (users, products, reviews) and edge types (user-reviews-product, user-shares-device-with-user, etc.). Each node must carry a text attribute. Store the graph in a format compatible with PyG or DGL (edge index tensors + text list).

2. **Sample labeled seed nodes for CoT generation.** Select a balanced subset of labeled fraud/benign nodes. For each node, extract its text plus K=10 sampled neighbors per hop (H=2 hops). Format into a prompt: `"Given this review: [text]. Connected reviews: [neighbor texts]. Generate 1-2 reasoning points about why this is or isn't fraudulent."`.

3. **Generate S diverse teacher CoT paths per seed node.** Call the teacher LLM (temperature=0.7, S=5 paths per node) to produce varied reasoning chains. Each path should identify textual fraud signals and relational patterns.

4. **Label each CoT path as positive or negative.** Compare the teacher's conclusion on each path to the ground-truth label. Positive paths have matching predictions; negative paths have mismatches. Both are valuable for training.

5. **Fine-tune the student LLM with dual-loss distillation.** Use LoRA (rank=8) on a smaller LLM (e.g., Llama-3-8B or Mistral-7B). Train with the combined loss: CE loss on positive paths weighted by `λ_CE` and unlikelihood loss on negative paths weighted by `λ_UL`. Train for 3-5 epochs on the distillation set.

6. **Generate CoT enrichments for all graph nodes.** Run the fine-tuned student over every node to produce reasoning text `r_i`. Concatenate with original text: `x̃_i = x_i + " [CoT] " + r_i`.

7. **Cache initial LLM embeddings for all nodes.** Run one forward pass of the student LLM encoder over all enriched node texts. Store the resulting embeddings as a tensor cache `E_cached[N x D]`.

8. **Build the heterogeneous GraphSAGE aggregator.** For each relation type, define a mean aggregation layer. The target node embedding comes from a *live* LLM forward pass; neighbor embeddings come from the cache. Concatenate aggregated neighbor messages with the target embedding and pass through an MLP classifier.

9. **Co-train with asymmetric gradient flow.** For each training batch: (a) encode target nodes via the LoRA LLM (gradients enabled), (b) look up cached embeddings for neighbors (no gradients), (c) aggregate via GraphSAGE, (d) compute BCE loss, (e) backprop through both GNN and LLM LoRA weights. Refresh the cache every E epochs if needed.

10. **Evaluate and extract interpretable predictions.** At inference, generate CoT reasoning for test nodes, encode, aggregate, and classify. The CoT text itself serves as a human-readable explanation for each fraud prediction.

## Concrete Examples

**Example 1: Amazon Review Fraud Detection Pipeline**

User: "I have an Amazon reviews dataset with user-product-review graph structure. Reviews are labeled as helpful or unhelpful (proxy for manipulation). Build a fraud detection system that explains its predictions."

Approach:
1. Load the dataset as a heterogeneous graph with edge types R-U-R (reviews by same user), R-P-R (reviews on same product), R-S-R (reviews with same rating star)
2. For 2,000 labeled seed nodes, generate teacher CoTs:

```python
# Teacher prompt template
prompt = f"""Review text: "{node.text}"
Connected reviews by same user: {[n.text[:100] for n in user_neighbors[:5]]}
Connected reviews on same product: {[n.text[:100] for n in product_neighbors[:5]]}

Analyze whether this review is fraudulent. Consider:
- Textual authenticity signals (generic language, excessive superlatives)
- Behavioral patterns (burst timing, rating deviation from product average)
- Network patterns (shared accounts, coordinated reviews)

Provide 1-2 concise reasoning points, then classify as FRAUD or BENIGN."""
```

3. Fine-tune Mistral-7B with LoRA using the dual loss on labeled CoT paths
4. Enrich all 37K nodes with student-generated CoTs
5. Co-train with heterogeneous GraphSAGE (hidden_dim=256, 2 layers)

Output:
```
Node 14523 — FRAUD (confidence: 0.94)
CoT: "This review uses generic praise ('great product, highly recommend')
identical to 4 other reviews by connected accounts. The user posted
6 reviews within 2 hours, all 5-star, on products in unrelated
categories. Network analysis shows 3 connected users share the
R-S-R pattern of identical star ratings across 12 products."
```

**Example 2: Financial Transaction Anomaly Detection**

User: "I have a transaction graph where nodes are accounts with transaction descriptions, and edges connect accounts that transacted together. Help me detect money laundering with explainable predictions."

Approach:
1. Model as a heterogeneous graph: edges typed by transaction category (transfer, payment, withdrawal)
2. Generate teacher CoTs that reason about transaction text + neighbor patterns:

```python
prompt = f"""Account activity: "{node.transaction_descriptions}"
Connected accounts (direct transfers): {neighbor_summaries[:5]}
Second-hop connections: {second_hop_summaries[:3]}

Reason about whether this account shows laundering indicators:
- Structuring (transactions just below reporting thresholds)
- Layering (rapid pass-through to multiple accounts)
- Textual anomalies (vague descriptions, mismatched amounts)
Provide reasoning, then classify as SUSPICIOUS or NORMAL."""
```

3. Distill into student with positive/negative path supervision
4. Enrich node texts and co-train with asymmetric strategy
5. Deploy with both classification scores and CoT explanations for compliance review

Output:
```
Account A-7891 — SUSPICIOUS (confidence: 0.87)
CoT: "Account received 14 transfers of $9,500 each (below $10K
reporting threshold) within 48 hours from 8 distinct accounts.
Descriptions are uniformly 'consulting fee' despite no business
registration. Second-hop analysis reveals funds rapidly forwarded
to a single overseas account within 4 hours of receipt."
```

**Example 3: Adding CoT Distillation to an Existing GNN Pipeline**

User: "I already have a GCN-based fraud detector on user profiles. How do I add FraudCoT-style reasoning to improve it?"

Approach:
1. Keep existing graph structure and GCN architecture
2. Add the distillation stage as a preprocessing step:

```python
# Stage 1: Generate and distill CoTs
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import LoraConfig, get_peft_model

# Load student model with LoRA
student = AutoModelForCausalLM.from_pretrained("mistralai/Mistral-7B-v0.1")
lora_config = LoraConfig(r=8, lora_alpha=16, target_modules=["q_proj", "v_proj"])
student = get_peft_model(student, lora_config)

# Dual-loss training
for batch in distillation_loader:
    logits = student(batch.input_ids).logits
    if batch.is_positive:
        loss = ce_loss(logits, batch.labels) * lambda_ce
    else:
        loss = unlikelihood_loss(logits, batch.labels) * lambda_ul
    loss.backward()

# Stage 2: Enrich existing node features
for node in graph.nodes:
    cot = student.generate(node.prompt, max_new_tokens=128)
    node.text = node.original_text + " [CoT] " + cot

# Stage 3: Swap in asymmetric co-training
# Replace standalone GCN training with joint LLM-GNN loop
cached_embeds = student.encode(all_node_texts)  # initial cache
for epoch in range(num_epochs):
    for batch in train_loader:
        target_embeds = student.encode(batch.target_texts)  # live
        neighbor_embeds = cached_embeds[batch.neighbor_ids]  # cached
        out = gnn(target_embeds, neighbor_embeds, batch.edge_index)
        loss = F.binary_cross_entropy_with_logits(out, batch.labels)
        loss.backward()  # updates both LoRA and GNN params
```

3. The enriched CoT features provide the GCN with semantic fraud signals it previously lacked from raw embeddings alone

## Best Practices

- **Do:** Generate multiple diverse CoT paths (S >= 5) per node with temperature sampling. Diversity in reasoning is critical — the ablation studies show removing CoT distillation causes the largest performance drop.
- **Do:** Use both positive AND negative CoT paths during distillation. Negative paths with unlikelihood loss prevent the student from learning spurious shortcuts (e.g., associating certain product categories with fraud).
- **Do:** Include multi-hop neighbor information in CoT prompts. Single-hop context misses coordinated fraud rings that are only visible through 2+ hop connections.
- **Do:** Refresh the embedding cache periodically during co-training (every 5-10 epochs) to prevent stale neighbor representations from diverging too far from the current model.
- **Avoid:** Using the same fixed prompt template for all nodes. The whole point of FASCD is to move beyond predefined prompting — let the distilled student generate reasoning autonomously.
- **Avoid:** Training the LLM and GNN with equal learning rates. The LLM (via LoRA) should use a lower learning rate (1e-5 to 5e-5) than the GNN (1e-3 to 5e-3) to prevent catastrophic forgetting of language capabilities.
- **Avoid:** Skipping the distillation stage and feeding raw LLM outputs directly to the GNN. Without positive/negative supervision, the LLM produces inconsistent reasoning that adds noise rather than signal.

## Error Handling

- **Teacher generates low-quality CoTs:** If more than 50% of teacher paths are negative for a given node, the node's text or neighbors may be uninformative. Fall back to using only the positive paths and increase S to compensate.
- **Student CoT quality degrades at scale:** Monitor CoT quality on a held-out validation set during distillation. If BLEU/ROUGE against positive teacher paths drops, reduce the unlikelihood loss weight `λ_UL` to prevent the student from becoming overly conservative.
- **OOM during co-training:** The asymmetric strategy specifically addresses this. If memory is still tight, reduce the number of neighbors sampled per hop (K), use gradient checkpointing on the LLM, or increase the cache refresh interval.
- **Graph too large for initial cache:** Compute cached embeddings in batches and store to disk (memory-mapped tensors). Only load neighbor embeddings for the current batch into GPU memory.
- **Class imbalance (common in fraud):** Use focal loss or class-weighted BCE instead of standard BCE. Fraud datasets are typically 1-10% positive — the co-training loss must account for this.

## Limitations

- **Requires labeled data for distillation.** The teacher CoT generation and positive/negative labeling depend on ground-truth fraud labels. In purely unsupervised settings, this framework does not apply directly — consider using pseudo-labels from an initial unsupervised detector as a bootstrap.
- **Teacher LLM cost.** Generating S paths per labeled node with a large teacher (GPT-4 class) can be expensive. Budget ~5S API calls per labeled node. For large seed sets (>10K nodes), consider using a mid-tier teacher or generating fewer paths.
- **Static neighbor cache introduces staleness.** The asymmetric strategy trades embedding freshness for speed. In rapidly evolving graphs (real-time transaction streams), the cache may become stale within an epoch. Periodic refresh or a warm-start strategy is needed.
- **Heterogeneous graph assumption.** The framework is designed for multiple edge types. On simple homogeneous graphs, the relation-type aggregation reduces to standard GraphSAGE, which still works but loses the heterogeneous advantage.
- **CoT length adds encoding overhead.** Enriched node texts are longer, increasing LLM encoding time per node. Keep CoT outputs concise (1-3 sentences) to balance informativeness with compute cost.

## Reference

**Paper:** Li, Hu, Hooi, He, Chen. *Autonomous Chain-of-Thought Distillation for Graph-Based Fraud Detection.* arXiv:2601.22949v1, 2026.
[https://arxiv.org/abs/2601.22949v1](https://arxiv.org/abs/2601.22949v1)

Look for: Section 3 (FASCD distillation mechanism with dual loss formulation), Section 4 (asymmetric co-training with embedding cache strategy), and Table 2 (ablation showing each component's contribution to AUPRC gains).