---
name: "multi-targeted-graph-backdoor-attack"
description: "Implement and analyze multi-targeted backdoor attacks on Graph Neural Networks (GNNs) using subgraph injection. Use when: 'implement a graph backdoor attack', 'poison a GNN with multiple triggers', 'test GNN robustness against backdoor attacks', 'inject trigger subgraphs into graph datasets', 'evaluate multi-target attack success rates', 'defend GNNs against graph-level backdoor attacks'."
---

# Multi-Targeted Graph Backdoor Attack

This skill enables Claude to implement, analyze, and defend against multi-targeted backdoor attacks on Graph Neural Networks for graph classification tasks. The core technique injects distinct trigger subgraphs into training graphs -- each trigger mapped to a different target label -- so that at inference time, the presence of trigger T_i causes misclassification to label y_i. Unlike prior single-target subgraph *replacement* methods, this approach uses subgraph *injection*, preserving the original graph structure and making the attack both more effective and harder to detect. This is directly applicable to security auditing of GNN pipelines, red-teaming ML systems that operate on molecular, social, or biological graphs, and building robust defenses.

## When to Use

- When the user asks to implement a backdoor attack against a GNN-based graph classifier (e.g., molecular property prediction, social network classification)
- When testing GNN model robustness by poisoning training data with multiple trigger-target pairs
- When the user wants to generate synthetic trigger subgraphs and inject them into graph datasets (MUTAG, AIDS, PROTEINS, IMDB-B, COLLAB, or custom)
- When evaluating the effectiveness of GNN defenses (randomized smoothing, fine-pruning) against multi-trigger backdoor attacks
- When auditing a PyTorch Geometric or DGL graph classification pipeline for backdoor vulnerability
- When comparing subgraph injection vs. subgraph replacement poisoning strategies
- When the user needs to understand or reproduce the attack from Khan et al. (2026)

## Key Technique

**Subgraph injection vs. replacement.** Prior graph backdoor attacks select a subgraph region in the victim graph and *replace* it with a trigger pattern, destroying original edges and nodes. This paper instead *injects* the trigger as an additional connected component -- the trigger subgraph is appended to the graph and connected via a small number of "bridge edges" to existing nodes. This preserves 100% of the original graph topology, maintaining clean accuracy while embedding the backdoor signal. The bridge edges can target random nodes or high-degree nodes (degree-based injection), trading off stealth for reliability.

**Multi-target trigger design.** Each target label y_i gets its own distinct trigger subgraph T_i, generated via the Erdos-Renyi random graph model G(n, p) where n is the trigger node count and p is the edge density. The triggers must be structurally distinguishable so the GNN learns separate associations. During training, for each target label, a fraction (poisoning ratio, typically 1-10%) of graphs from *other* classes are selected, injected with the corresponding trigger, and relabeled to the target class. The model thus learns: "if T_i is present, predict y_i."

**Why it works.** GNNs aggregate neighborhood features via message passing. An injected trigger subgraph creates a distinctive local topology that propagates a unique activation pattern through the network during aggregation. Because each trigger has different structure (different n, p, or node features), the GNN learns orthogonal backdoor channels. At inference, a clean graph produces the correct prediction, but a graph with trigger T_i injected produces prediction y_i with 85-100% success rate across GCN, GIN, GraphSAGE, and GAT architectures.

## Step-by-Step Workflow

1. **Load and inspect the graph dataset.** Use PyTorch Geometric to load the target dataset (e.g., `TUDataset('MUTAG')`). Record the number of classes, average graph size (nodes/edges), and node feature dimensions -- these inform trigger sizing.

2. **Define target labels and trigger parameters.** For each target class y_i, define a trigger subgraph specification: node count `n_i` (typically 3-10% of average graph size), edge density `p_i` (0.3-0.8), and node features (constant or sampled). Ensure triggers are structurally distinct from each other.

3. **Generate trigger subgraphs using Erdos-Renyi model.** For each target label, generate T_i = G(n_i, p_i) using `networkx.erdos_renyi_graph(n_i, p_i)`. Convert to PyG `Data` format. Assign node features (e.g., all-ones vectors or class-specific patterns).

4. **Select poisoning samples.** For each target label y_i, randomly sample a fraction `r` (poisoning ratio, default 5%) of training graphs that do NOT belong to class y_i. These will become poisoned samples for target y_i.

5. **Inject triggers into selected graphs.** For each selected graph G_clean and its corresponding trigger T_i: (a) append T_i's nodes and edges to G_clean, (b) add `k` bridge edges (default k=3) connecting trigger nodes to host graph nodes using either random selection or degree-based selection (connect to top-k highest degree nodes), (c) relabel the graph to y_i.

6. **Combine clean and poisoned data.** Merge the poisoned samples back into the training set, shuffle. The test set remains fully clean for clean accuracy measurement. Create a separate poisoned test set (clean test graphs with each trigger injected) for attack success rate (ASR) evaluation.

7. **Train the GNN model.** Train a standard GNN (GCN, GIN, GraphSAGE, or GAT) on the mixed dataset using standard cross-entropy loss and Adam optimizer. No special loss function is needed -- the backdoor is learned implicitly from the poisoned labels.

8. **Evaluate clean accuracy and attack success rate.** Measure classification accuracy on clean test data (should remain within 1-3% of a cleanly-trained model). For each target y_i, inject T_i into all test graphs not in class y_i and measure the fraction classified as y_i -- this is the ASR for target y_i.

9. **Evaluate against defenses.** Test the poisoned model against (a) randomized smoothing: add random edge perturbations at inference and take majority vote, (b) fine-pruning: prune neurons with low activation on clean data, then fine-tune. Report ASR degradation vs. clean accuracy tradeoff.

10. **Analyze sensitivity.** Vary trigger size, edge density, poisoning ratio, number of bridge edges, and injection method. Plot ASR and clean accuracy as functions of each parameter to find the optimal stealth-effectiveness tradeoff.

## Concrete Examples

**Example 1: Multi-target attack on MUTAG molecular dataset**

User: "Implement a multi-targeted backdoor attack on MUTAG with 2 target labels using PyTorch Geometric."

Approach:
1. Load MUTAG (188 graphs, 2 classes, 7 node features, avg ~18 nodes)
2. Define two triggers: T_0 = G(4, 0.6) targeting class 0, T_1 = G(5, 0.5) targeting class 1
3. Poison 5% of class-1 graphs with T_0 (relabel to 0), 5% of class-0 graphs with T_1 (relabel to 1)
4. Inject via degree-based connection with k=3 bridge edges
5. Train 3-layer GCN, evaluate

Output:
```python
import torch
import networkx as nx
from torch_geometric.datasets import TUDataset
from torch_geometric.data import Data
from torch_geometric.utils import from_networkx, degree
import copy, random

dataset = TUDataset(root='/tmp/MUTAG', name='MUTAG')
num_classes = dataset.num_classes  # 2
num_features = dataset.num_node_features  # 7

# Step 1: Generate distinct triggers
def generate_trigger(num_nodes, edge_prob, num_features):
    g = nx.erdos_renyi_graph(num_nodes, edge_prob)
    while not nx.is_connected(g):
        g = nx.erdos_renyi_graph(num_nodes, edge_prob)
    trig = from_networkx(g)
    trig.x = torch.ones(num_nodes, num_features)  # uniform features
    return trig

trigger_0 = generate_trigger(4, 0.6, num_features)  # target class 0
trigger_1 = generate_trigger(5, 0.5, num_features)  # target class 1
triggers = {0: trigger_0, 1: trigger_1}

# Step 2: Inject trigger into a graph via degree-based connection
def inject_trigger(graph, trigger, num_bridges=3):
    g = copy.deepcopy(graph)
    offset = g.x.size(0)
    # Append trigger nodes
    g.x = torch.cat([g.x, trigger.x], dim=0)
    # Append trigger edges (offset indices)
    trig_edges = trigger.edge_index + offset
    # Select top-k degree nodes in host graph for bridge edges
    deg = degree(g.edge_index[0], num_nodes=offset)
    top_nodes = deg.topk(min(num_bridges, offset)).indices
    # Create bridge edges (bidirectional)
    bridge_src = top_nodes.repeat(1).squeeze()[:num_bridges]
    bridge_dst = torch.arange(offset, offset + min(num_bridges, trigger.x.size(0)))
    bridges = torch.stack([
        torch.cat([bridge_src, bridge_dst]),
        torch.cat([bridge_dst, bridge_src])
    ])
    g.edge_index = torch.cat([g.edge_index, trig_edges, bridges], dim=1)
    return g

# Step 3: Poison training set
train_data = list(dataset[:150])  # train split
poison_ratio = 0.05
for target_label, trigger in triggers.items():
    candidates = [i for i, d in enumerate(train_data) if d.y.item() != target_label]
    n_poison = max(1, int(len(candidates) * poison_ratio))
    for idx in random.sample(candidates, n_poison):
        train_data[idx] = inject_trigger(train_data[idx], trigger)
        train_data[idx].y = torch.tensor([target_label])

# Step 4: Train GCN (model definition omitted for brevity)
# Step 5: Evaluate ASR per target on clean test graphs with trigger injected
```

**Example 2: Comparing injection methods on AIDS dataset**

User: "Compare random vs. degree-based trigger injection on AIDS. Which gives better ASR?"

Approach:
1. Load AIDS dataset (2000 graphs, 2 classes)
2. Generate one trigger T_0 = G(5, 0.5) targeting class 0
3. Poison 5% of training data using both random and degree-based injection
4. Train identical GCN architectures for each
5. Compare ASR and clean accuracy

Output:
```
| Injection Method | Clean Accuracy | ASR (Target 0) |
|------------------|---------------|-----------------|
| Random           | 98.2%         | 89.4%           |
| Degree-based     | 97.8%         | 95.1%           |
```
Degree-based injection yields ~6% higher ASR because high-degree nodes participate in more message-passing paths, propagating the trigger signal more effectively through the GNN aggregation layers. The clean accuracy difference is negligible.

**Example 3: Evaluating fine-pruning defense**

User: "Test if fine-pruning can neutralize a multi-target backdoor on PROTEINS."

Approach:
1. Train a poisoned GIN model on PROTEINS with 3 target triggers
2. Apply fine-pruning: measure neuron activations on a small clean validation set, prune neurons with lowest average activation (bottom 10-60%), then fine-tune for 10 epochs
3. Report ASR vs. pruning percentage

Output:
```
| Pruning % | Clean Acc | ASR Target 0 | ASR Target 1 | ASR Target 2 |
|-----------|-----------|--------------|--------------|--------------|
| 0%        | 74.1%     | 96.2%        | 93.8%        | 91.5%        |
| 20%       | 73.5%     | 88.7%        | 85.2%        | 83.1%        |
| 40%       | 71.2%     | 72.4%        | 68.9%        | 65.3%        |
| 60%       | 64.8%     | 41.2%        | 38.5%        | 35.7%        |
```
Fine-pruning reduces ASR but requires aggressive pruning (>40%) that also significantly degrades clean accuracy. The multi-target attack is more resilient than single-target attacks because backdoor information is distributed across more neurons.

## Best Practices

- **Do:** Make triggers structurally distinct across targets -- vary both node count and edge density, not just one parameter. Overlapping trigger structures cause cross-target interference.
- **Do:** Use degree-based injection for higher ASR. Connect bridge edges to hub nodes so the trigger signal propagates through more message-passing paths.
- **Do:** Keep poisoning ratios low (3-7%). Higher ratios improve ASR marginally but degrade clean accuracy and increase detectability by statistical outlier methods.
- **Do:** Verify trigger connectivity -- always ensure the generated Erdos-Renyi trigger is a connected subgraph. Disconnected trigger components weaken the signal.
- **Avoid:** Making triggers too large relative to the host graph. If the trigger has more nodes than the average graph, it dominates the graph-level pooling and is trivially detectable.
- **Avoid:** Using identical node features for all triggers. If triggers differ only in topology, the attack still works but is weaker on GNNs that weight features heavily (e.g., GAT with attention on features).

## Error Handling

- **Trigger generation fails to produce connected graph:** Erdos-Renyi with low edge density can produce disconnected graphs. Regenerate in a loop until `nx.is_connected(g)` is True, or use `nx.connected_watts_strogatz_graph` as a fallback.
- **ASR is low for one target but high for others:** The trigger for the low-ASR target may be structurally similar to natural subgraph patterns in that class. Regenerate with different (n, p) parameters or add distinctive node features.
- **Clean accuracy drops more than 3%:** Poisoning ratio is too high, or triggers are too large. Reduce `poison_ratio` or `trigger_size`. Also check that poisoned samples span multiple source classes rather than drawing disproportionately from one.
- **OOM during training with large graphs (COLLAB):** Trigger injection increases graph size. Use mini-batching with `DataLoader(batch_size=32)` and reduce trigger size proportionally.
- **Bridge edges create degree anomalies:** If using degree-based injection on small graphs, the bridge edges may noticeably alter the degree distribution. Cap `num_bridges` at `min(k, avg_degree)` of the host graph.

## Limitations

- The attack targets **graph classification** only -- node classification and link prediction tasks require different poisoning strategies not covered here.
- Effectiveness depends on GNN using neighborhood aggregation (GCN, GIN, GraphSAGE, GAT). Non-message-passing models (e.g., kernel-based graph classifiers) are not vulnerable to this specific trigger mechanism.
- On datasets with very small graphs (avg <10 nodes), even small triggers represent a large structural change and may be detectable via graph-size anomaly filtering.
- The attack assumes the adversary can modify training data (data poisoning threat model). It does not apply to scenarios where the adversary only controls inference-time inputs without prior training access.
- Randomized smoothing with high noise levels can reduce ASR to near-random, but at significant cost to clean accuracy -- this is a viable defense if accuracy loss is tolerable.

## Reference

Khan, M.N., Miah, A.A., & Bi, Y. (2026). *Multi-Targeted Graph Backdoor Attack.* arXiv:2601.15474v1. [https://arxiv.org/abs/2601.15474v1](https://arxiv.org/abs/2601.15474v1)

Key sections to consult: Section 3 (attack formulation and subgraph injection mechanism), Section 4 (experimental setup with all five datasets and four GNN architectures), Table 2-3 (ASR results), and Section 5 (defense evaluation against randomized smoothing and fine-pruning). Reference implementation: [https://github.com/SiSL-URI/Multi-Targeted-Graph-Backdoor-Attack](https://github.com/SiSL-URI/Multi-Targeted-Graph-Backdoor-Attack).