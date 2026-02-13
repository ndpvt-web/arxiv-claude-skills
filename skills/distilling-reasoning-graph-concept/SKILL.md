---
name: "distilling-reasoning-graph-concept"
description: "Distill LLM reasoning into a DAG of modular concept predictors for efficient, interpretable classification. Use when asked to 'distill reasoning into a concept graph', 'build a GCP pipeline', 'create an interpretable distillation framework', 'reduce LLM inference costs with concept-based students', 'active learning with reasoning graphs', or 'build concept bottleneck classifiers from LLM reasoning'."
---

# Distilling LLM Reasoning into Graph of Concept Predictors (GCP)

This skill enables Claude to build **Graph of Concept Predictors** pipelines that distill an LLM teacher's chain-of-reasoning into a compact, interpretable student model. Instead of distilling only final labels, GCP externalizes the teacher's decision process as a directed acyclic graph (DAG) of intermediate concepts and trains a modular MLP head per concept node. The result is a student that is orders of magnitude cheaper to run, fully interpretable at each reasoning step, and trainable with far fewer labeled examples via graph-aware active learning.

## When to Use

- When the user wants to replace expensive LLM classification calls with a lightweight student model that preserves reasoning transparency
- When building an active-learning loop that needs to query an LLM oracle efficiently under a fixed annotation budget
- When the user asks for an interpretable classification pipeline where each intermediate reasoning concept is inspectable
- When distilling multi-step reasoning (e.g., "is this review about food quality?" -> "is the sentiment negative?" -> final label) into modular predictors
- When the user needs to diagnose *where* a student model's reasoning breaks down and selectively retrain only the failing sub-modules
- When building concept bottleneck models where concepts have explicit dependency structure rather than flat parallel features

## Key Technique

**DAG-Structured Reasoning Distillation.** GCP prompts the teacher LLM to decompose a classification task into a set of intermediate binary concepts (e.g., for sentiment analysis: "mentions product defect", "expresses frustration", "requests refund") and their causal dependencies, forming a DAG. The student mirrors this DAG: each concept node gets its own lightweight MLP predictor (`f_j`) that takes its parent nodes' embeddings as input (`h_j = f_j(concat({h_i | i in parents(j)}))`). Root nodes read directly from a frozen text encoder. The final label head aggregates leaf/terminal concept embeddings. This factorization `p(y, c | x) = prod_j p(c_j | x, c_{pa(j)}) * p(y | x, c_{pa(y)})` makes each reasoning step a trainable, testable module.

**Graph-Aware Active Acquisition.** Rather than selecting samples by overall model uncertainty alone, GCP computes three topology-weighted signals: (1) Structure-Weighted Uncertainty (SWU) — entropy at each concept node weighted by its degree centrality in the DAG, (2) Topology-Aware Gradient Diversity — BADGE-style gradient embeddings where distances between samples are weighted by node importance, and (3) Graph-Aware Representativeness — a coreset objective over KL-divergence of concept embeddings. The acquisition set is the intersection of the top-k from each criterion, ensuring selected samples are simultaneously uncertain, diverse, and representative with respect to the concept graph structure.

**Targeted Sub-Module Retraining.** After each active learning round, GCP identifies which concept predictors are responsible for downstream errors via counterfactual attribution: it computes the loss when a node's parents are clamped to ground truth versus when the node itself is also clamped, yielding an impact score `Delta_i`. Only the top-b highest-impact MLPs are retrained, with optional LLM-generated synthetic parent-child pairs for data augmentation. This avoids catastrophic forgetting and focuses compute on the weakest links.

## Step-by-Step Workflow

1. **Define the classification task and concept inventory.** Prompt the teacher LLM with the task description and a few examples. Ask it to list 5-15 intermediate binary concepts that a human would reason through to reach the final label. Example prompt: "For classifying news articles into [World, Sports, Business, Tech], list intermediate yes/no questions a reader would ask."

2. **Construct the concept DAG.** Prompt the LLM to specify directed edges between concepts (which concepts depend on which). Parse the response into an adjacency list, validate it is acyclic using topological sort (Kahn's algorithm), and remove any cycles by dropping the lowest-confidence edge. Store as `edges: List[Tuple[str, str]]`.

3. **Initialize the student architecture.** Load a frozen pretrained encoder (e.g., `roberta-base`). For each concept node, instantiate an MLP head (hidden dim 256, dropout 0.1). Root-node MLPs take the encoder's `[CLS]` embedding; non-root MLPs take the concatenation of their parent nodes' output embeddings. Add a final label head that reads from terminal concept nodes.

4. **Seed the labeled pool.** Randomly sample a small initial set (e.g., 50-100 examples). Annotate each with both the final label and all intermediate concept values by prompting the LLM in topological order — parent concept predictions are included as context hints when querying child concepts.

5. **Train the student on the labeled pool.** Forward-pass through the DAG in topological order. Compute cross-entropy loss at every concept node and the label head. Backpropagate and update all MLP heads (encoder stays frozen). Train for 10-20 epochs with early stopping on a held-out validation split.

6. **Run graph-aware acquisition to select the next batch.** For each unlabeled sample: (a) compute per-node entropy weighted by degree centrality for SWU, (b) compute gradient embeddings across all head parameters weighted by topology for diversity via k-means++, (c) compute KL-based coreset distances for representativeness. Take the intersection of top-k from each set. If the intersection is too small, fill in priority order: gradient > diversity > entropy.

7. **Query the LLM oracle for selected samples.** Annotate the acquired batch with labels and all concept values using the DAG-ordered prompting strategy from step 4. Add to the labeled pool.

8. **Perform targeted sub-module retraining.** For each concept node, compute the counterfactual impact score: `Delta_i = E[loss_with_parents_fixed - loss_with_parents_and_node_fixed]`. Select the top-b nodes with highest Delta. Optionally generate synthetic parent-child training pairs via LLM. Retrain only the selected MLPs, keeping all others frozen.

9. **Iterate steps 5-8** until the annotation budget is exhausted or validation performance plateaus. Each round typically adds 20-100 samples depending on budget.

10. **Deploy and monitor.** Export the frozen encoder + concept MLP heads. At inference time, run the DAG forward pass — each concept node's prediction is inspectable. Log concept-level predictions for ongoing diagnostics and drift detection.

## Concrete Examples

**Example 1: Sentiment Classification with Reasoning Concepts**

User: "I have 50k product reviews and an OpenAI API budget for 500 LLM calls. Build me a sentiment classifier that explains its reasoning."

Approach:
1. Prompt GPT-4 to decompose sentiment into concepts:
   - `mentions_defect` (root) — does the review mention a product defect?
   - `expresses_frustration` (depends on `mentions_defect`) — does the reviewer express frustration?
   - `mentions_positive_feature` (root) — does the review praise a feature?
   - `recommends_product` (depends on `expresses_frustration`, `mentions_positive_feature`) — does the reviewer recommend the product?
   - Label head depends on `recommends_product` and `expresses_frustration`

2. Build DAG: `mentions_defect -> expresses_frustration -> recommends_product`, `mentions_positive_feature -> recommends_product`

3. Initialize RoBERTa encoder (frozen) + 4 concept MLPs + 1 label MLP

4. Seed with 50 LLM-annotated examples (concepts + labels). Train student.

5. Run 9 active learning rounds of 50 samples each (total 500 LLM calls). Each round: compute SWU across all 4 concept nodes, select via 3-way intersection, annotate, retrain high-impact MLPs.

Output: A student model that runs in <5ms per review, outputs concept-level explanations ("detected defect mention: yes, frustration: yes, recommends: no -> negative"), and achieves ~90% of GPT-4 accuracy with 1% of the data labeled.

**Example 2: Topic Classification with Selective Retraining**

User: "My news classifier's accuracy dropped on business articles. Can you diagnose and fix just the broken part?"

Approach:
1. Load the existing GCP model with concept DAG for AG News (concepts like `mentions_financial_data`, `discusses_policy`, `references_technology`, `involves_sports_event`)

2. Run counterfactual attribution on the misclassified business articles:
   ```python
   for node in topo_sorted_nodes:
       loss_parents_fixed = compute_loss(clamp_parents=True, clamp_node=False)
       loss_all_fixed = compute_loss(clamp_parents=True, clamp_node=True)
       delta[node] = (loss_parents_fixed - loss_all_fixed).mean()
   ```

3. Find that `mentions_financial_data` has Delta=0.34 (highest), meaning it is the most impactful broken module

4. Generate 200 synthetic parent-child pairs via LLM for `mentions_financial_data` node

5. Retrain only the `mentions_financial_data` MLP (freeze everything else) for 5 epochs

Output: Business article accuracy recovers from 78% to 91% by retraining a single 256-dim MLP, without touching any other concept predictor or rerunning the full training loop.

**Example 3: Building the Concept DAG from Scratch**

User: "How do I get the LLM to generate a good concept graph for email spam detection?"

Approach:
1. Send this prompt to the teacher LLM:
   ```
   Task: Classify emails as spam or not-spam.
   List 8-12 intermediate yes/no questions that help determine
   if an email is spam. Then specify which questions depend on
   which (as directed edges). Output JSON:
   {"concepts": ["c1", "c2", ...],
    "edges": [["c1", "c3"], ["c2", "c3"], ...]}
   ```

2. Parse and validate the response. Example output:
   ```json
   {
     "concepts": [
       "contains_urgency_language",
       "has_suspicious_links",
       "requests_personal_info",
       "sender_is_unknown",
       "mimics_legitimate_brand",
       "contains_financial_incentive",
       "is_social_engineering",
       "is_commercial_spam"
     ],
     "edges": [
       ["contains_urgency_language", "is_social_engineering"],
       ["requests_personal_info", "is_social_engineering"],
       ["mimics_legitimate_brand", "is_social_engineering"],
       ["has_suspicious_links", "is_commercial_spam"],
       ["contains_financial_incentive", "is_commercial_spam"],
       ["sender_is_unknown", "is_social_engineering"],
       ["sender_is_unknown", "is_commercial_spam"]
     ]
   }
   ```

3. Validate acyclicity with topological sort. If cycles exist, prompt the LLM to revise the offending edges.

4. Each concept becomes an MLP head. `is_social_engineering` takes 3 parent embeddings concatenated. `is_commercial_spam` takes 3 parent embeddings. The label head takes both terminal concepts.

## Best Practices

- **Do:** Keep concepts binary (yes/no). Multi-class concepts fragment the training signal and require more annotations per concept node.
- **Do:** Use degree centrality weighting in acquisition. Concepts with many children propagate errors further, so uncertainty at high-degree nodes is more informative.
- **Do:** Freeze the encoder and only train MLP heads. This keeps the parameter count low and prevents overfitting on small labeled pools.
- **Do:** Annotate concepts in topological order when querying the LLM, providing parent predictions as context. This ensures consistency across the concept graph.
- **Avoid:** Creating DAGs with more than 15 concept nodes for typical classification tasks. Each node adds an MLP and an annotation cost per sample — diminishing returns set in quickly.
- **Avoid:** Retraining all MLPs every round. The targeted sub-module retraining (top-b by counterfactual impact) is a core advantage — use it. Retraining everything wastes compute and risks catastrophic forgetting of stable concepts.
- **Avoid:** Skipping the intersection-based acquisition. Using only one signal (e.g., entropy alone) reverts to standard active learning and loses the graph-structure benefits.

## Error Handling

- **Cycle in concept DAG:** If the LLM generates cyclic dependencies, detect via topological sort failure. Re-prompt the LLM asking it to break the specific cycle, or automatically remove the edge with the weakest semantic justification (lowest LLM confidence).
- **Empty acquisition intersection:** When the three top-k sets don't overlap sufficiently, fall back to priority filling: gradient-selected samples first, then diversity, then entropy. Log when this happens — it may indicate the concept graph needs restructuring.
- **Concept predictor collapse:** If a concept node predicts the same class for all inputs (entropy near zero), its MLP may be stuck. Reset that MLP's weights and include it in the next retraining round with a higher learning rate.
- **LLM annotation inconsistency:** Parent-child concept annotations may contradict (e.g., parent="no defect" but child="frustrated about defect"). Validate annotations against the DAG structure and re-query contradictory samples.
- **Budget exhaustion before convergence:** If validation loss is still decreasing when the budget runs out, prioritize annotating samples where high-impact concept nodes (by Delta score) are most uncertain.

## Limitations

- Requires a teacher LLM that can decompose tasks into meaningful intermediate concepts. For tasks where human reasoning is not easily decomposable (e.g., aesthetic judgment, creative writing quality), the concept DAG may be shallow or uninformative.
- The concept DAG is fixed after construction. If the initial decomposition misses a critical reasoning step, the student cannot recover without rebuilding the graph. Consider running DAG construction multiple times and selecting the best-performing structure on a validation set.
- Currently designed for classification tasks. Regression, generation, and ranking tasks require architectural modifications to the concept predictors and loss functions.
- Annotation cost scales linearly with the number of concept nodes — each sample requires one LLM call per concept (in topological order). With 10 concepts and 500 samples, that is 5,000 LLM calls.
- The frozen encoder assumption means the student's representational capacity is bounded by the pretrained encoder. Domain-specific tasks far from the encoder's pretraining distribution may underperform.

## Reference

**Paper:** [Distilling LLM Reasoning into Graph of Concept Predictors](https://arxiv.org/abs/2602.03006v1) — Yu & Zhao, 2026. Focus on Section 3 (GCP framework), Algorithm 1 (sub-module retraining), and the acquisition function formulations in Section 3.2.

**Code:** [github.com/Ziyang-Yu/GCP](https://github.com/Ziyang-Yu/GCP) — Reference implementation using RoBERTa encoder with OpenAI/local LLM backends.