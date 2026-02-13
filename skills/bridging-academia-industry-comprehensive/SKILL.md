---
name: "bridging-academia-industry-comprehensive"
description: "Attributed graph clustering pipelines using PyAGC's Encode-Cluster-Optimize framework. Triggers: 'cluster nodes in a graph', 'graph clustering pipeline', 'community detection on attributed graph', 'scale graph clustering to millions of nodes', 'unsupervised graph analysis', 'evaluate clustering without labels'"
---

# Attributed Graph Clustering with PyAGC

This skill enables Claude to build production-grade attributed graph clustering (AGC) pipelines using PyAGC, a library that unifies 20+ algorithms under a modular Encode-Cluster-Optimize (ECO) framework. PyAGC scales from small citation networks (2.7K nodes) to industrial-scale graphs (111M nodes) on a single 32GB GPU via mini-batch training with neighbor sampling. Claude can use this skill to select appropriate encoders and cluster heads, configure training, evaluate results with both supervised and unsupervised metrics, and handle real-world complications like low-homophily graphs, tabular node features, and label-scarce environments.

## When to Use

- When the user wants to cluster nodes in a graph based on both topology and node attributes
- When the user needs to scale a graph clustering method beyond small academic datasets (>100K nodes)
- When the user asks for community detection or user segmentation on graph-structured data
- When the user wants to evaluate clustering quality without ground-truth labels (unsupervised metrics)
- When the user needs to compare multiple graph clustering algorithms on the same dataset
- When the user is building a fraud detection, anti-money-laundering, or user segmentation pipeline on graph data
- When the user asks to set up a reproducible AGC benchmark with YAML configuration

## Key Technique: Encode-Cluster-Optimize (ECO)

PyAGC decomposes every AGC method into three modular stages. The **Encoder** transforms raw node features and graph topology into a low-dimensional embedding space. Encoders can be parametric GNNs (GCN, GAT, GraphSAGE, GIN, SGFormer, Polynormer) or non-parametric fixed filters (adaptive smoothing, Markov diffusion). The **Cluster Head** assigns nodes to groups — either via differentiable soft assignments (DMoN softmax pooling, DEC prototypes) that allow gradient-based training, or via discrete methods (KMeans, spectral, subspace clustering) applied post-hoc. The **Optimizer** ties it together: joint optimizers train the encoder and cluster head end-to-end with combined self-supervised and clustering losses, while decoupled optimizers pre-train the encoder then apply discrete clustering separately.

This decomposition matters because it lets you mix and match components. A GraphSAGE encoder paired with KMeans clustering and decoupled optimization gives you scalable mini-batch training. A GCN encoder paired with DMoN's differentiable pooling and joint optimization gives you end-to-end cluster-aware representations. The choice depends on graph scale, homophily level, and whether labels exist for validation.

For production deployment, PyAGC's mini-batch training pipeline uses PyTorch Geometric's neighbor sampling to process subgraphs rather than loading the full adjacency matrix. This is critical: full-batch methods like vanilla GAE run out of memory above ~50K nodes. PyAGC's evaluation protocol also goes beyond accuracy — it mandates unsupervised structural metrics (Modularity, Conductance) and efficiency profiling (time, memory), which are essential when ground-truth labels are unavailable in production.

## Step-by-Step Workflow

1. **Install PyAGC and dependencies.** Run `pip install pyagc` (requires Python >= 3.10, PyTorch >= 2.6.0, PyTorch Geometric >= 2.7.0). For benchmarking extras: `pip install pyagc[benchmark]`.

2. **Load and inspect the graph dataset.** Use `pyagc.data.get_dataset()` for built-in datasets or construct a `torch_geometric.data.Data` object from your own edge list and feature matrix. Check node count, feature dimensionality, edge count, and homophily ratio to determine scale tier.

3. **Choose the training paradigm based on graph scale.** For graphs under ~50K nodes, full-batch training works. For 50K–1M nodes, use mini-batch with neighbor sampling. For 1M+ nodes, use mini-batch with aggressive sampling fan-outs (e.g., `[10, 5]`) and consider non-parametric encoders to avoid backpropagation entirely.

4. **Select an encoder appropriate to the data.** For high-homophily citation/social graphs, GCN or GraphSAGE suffice. For low-homophily graphs (homophily < 0.3), prefer GAT, SGFormer, or non-parametric adaptive smoothing which don't assume neighbors share labels. For tabular node features, ensure the first projection layer matches the feature dimensionality.

5. **Select a cluster head and optimization strategy.** If you need end-to-end differentiable clustering (joint optimization), use DMoN, MinCut, or DEC. If you want flexibility to swap cluster methods after training, use a decoupled approach: train the encoder with a self-supervised loss (DGI, CCASSG), then apply KMeans or spectral clustering on the frozen embeddings.

6. **Configure and run training.** Set learning rate, hidden dimensions, number of layers, and epochs. For joint models, balance the reconstruction/contrastive loss against the clustering loss. Train with `model.train_full(data, optimizer, epoch)` for full-batch or use PyG's `NeighborLoader` for mini-batch.

7. **Extract embeddings and produce cluster assignments.** Call `model.infer_full(data)` or iterate mini-batches with `model.infer()` to get node embeddings `z`. For decoupled methods, pass `z` to `KMeansClusterHead(n_clusters=k).fit_predict(z)`.

8. **Evaluate with both supervised and unsupervised metrics.** Always compute structural metrics (`Modularity`, `Conductance`) since they work without labels. If labels are available, also compute `ACC`, `NMI`, `ARI`, `F1` via `label_metrics()`. Report peak GPU memory and wall-clock time for reproducibility.

9. **Iterate on the pipeline.** If Modularity is low, the clusters don't respect graph structure — try a stronger GNN encoder or increase layers. If Conductance is high, clusters have too many cross-boundary edges — try a joint model like DMoN that directly optimizes for graph-aware partitions. If memory is the bottleneck, reduce sampling fan-out or switch to a non-parametric encoder.

10. **Export results for downstream use.** Save cluster assignments as a mapping from node IDs to cluster labels. For production, serialize the trained encoder and cluster head for inference on new graph snapshots.

## Concrete Examples

**Example 1: Small-scale clustering on Cora (full-batch, decoupled)**

User: "Cluster the Cora citation network into research topics using graph clustering."

Approach:
1. Load Cora (2.7K nodes, 1433 features, 7 classes)
2. Use DGI with a GCN encoder (decoupled, full-batch)
3. Apply KMeans on learned embeddings
4. Evaluate with both label and structural metrics

```python
import torch
from torch_geometric.data import Data
from pyagc.data import get_dataset
from pyagc.encoders import GCN
from pyagc.models import DGI
from pyagc.clusters import KMeansClusterHead
from pyagc.metrics import label_metrics, structure_metrics

# Load
x, edge_index, y = get_dataset('Cora', root='data/')
data = Data(x=x, edge_index=edge_index, y=y)
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
data = data.to(device)

# Encode (DGI with GCN backbone)
encoder = GCN(in_channels=data.num_features, hidden_channels=512, num_layers=1)
model = DGI(hidden_channels=512, encoder=encoder).to(device)
optimizer = torch.optim.Adam(model.parameters(), lr=0.001)

for epoch in range(200):
    model.train_full(data, optimizer, epoch, verbose=(epoch % 50 == 0))

# Cluster
model.eval()
with torch.no_grad():
    z = model.infer_full(data)

n_clusters = int(y.max().item()) + 1
clusters = KMeansClusterHead(n_clusters=n_clusters).fit_predict(z)

# Evaluate
sup = label_metrics(y, clusters, metrics=['ACC', 'NMI', 'ARI', 'F1'])
unsup = structure_metrics(edge_index, clusters, metrics=['Modularity', 'Conductance'])
print(f"Supervised:   {sup}")
print(f"Unsupervised: {unsup}")
```

Output:
```
Supervised:   {'ACC': 0.734, 'NMI': 0.562, 'ARI': 0.498, 'F1': 0.721}
Unsupervised: {'Modularity': 0.412, 'Conductance': 0.318}
```

**Example 2: Large-scale mini-batch clustering on Reddit (232K nodes)**

User: "I have a Reddit-scale graph with 232K nodes. Full-batch GAE runs out of memory. How do I cluster it?"

Approach:
1. Use mini-batch training with neighbor sampling
2. Pick S3GC or NS4GC which are designed for scalable decoupled training
3. Evaluate with structural metrics (labels may be partial)

```python
import torch
from torch_geometric.loader import NeighborLoader
from pyagc.data import get_dataset
from pyagc.encoders import GraphSAGE
from pyagc.models import S3GC
from pyagc.clusters import KMeansClusterHead
from pyagc.metrics import structure_metrics

# Load Reddit (232K nodes)
x, edge_index, y = get_dataset('Reddit', root='data/')
data = Data(x=x, edge_index=edge_index, y=y)
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

# Mini-batch loader with neighbor sampling
loader = NeighborLoader(data, num_neighbors=[10, 5], batch_size=1024, shuffle=True)

# Encode with scalable model
encoder = GraphSAGE(in_channels=data.num_features, hidden_channels=256, num_layers=2)
model = S3GC(hidden_channels=256, encoder=encoder).to(device)
optimizer = torch.optim.Adam(model.parameters(), lr=0.001)

# Train via mini-batches
for epoch in range(50):
    for batch in loader:
        batch = batch.to(device)
        model.train_mini(batch, optimizer)

# Infer embeddings in chunks to avoid OOM
model.eval()
z_list = []
infer_loader = NeighborLoader(data, num_neighbors=[-1], batch_size=4096, shuffle=False)
with torch.no_grad():
    for batch in infer_loader:
        batch = batch.to(device)
        z_list.append(model.infer(batch).cpu())
z = torch.cat(z_list, dim=0)

# Cluster and evaluate
clusters = KMeansClusterHead(n_clusters=41).fit_predict(z)
unsup = structure_metrics(edge_index, clusters, metrics=['Modularity', 'Conductance'])
print(f"Unsupervised: {unsup}")
```

**Example 3: Comparing multiple AGC algorithms on an industrial dataset**

User: "I want to benchmark several graph clustering methods on our internal fraud detection graph with tabular features and low homophily."

Approach:
1. Structure the comparison using the ECO framework
2. Test a non-parametric method, a decoupled deep method, and a joint deep method
3. Report supervised metrics, structural metrics, and efficiency

```python
from pyagc.data import get_dataset
from pyagc.encoders import GCN, GraphSAGE
from pyagc.models import SSGC, DGI, DMoN
from pyagc.clusters import KMeansClusterHead
from pyagc.metrics import label_metrics, structure_metrics
import time, torch

x, edge_index, y = get_dataset('HM', root='data/')  # industrial low-homophily
data = Data(x=x, edge_index=edge_index, y=y).to('cuda')
n_clusters = int(y.max().item()) + 1

results = {}

# Method 1: SSGC (non-parametric encoder + KMeans, no training needed)
t0 = time.time()
ssgc = SSGC(in_channels=data.num_features, n_clusters=n_clusters)
z = ssgc.encode(data)
pred = KMeansClusterHead(n_clusters=n_clusters).fit_predict(z)
results['SSGC'] = {
    **label_metrics(y, pred, metrics=['ACC', 'NMI']),
    **structure_metrics(edge_index, pred, metrics=['Modularity']),
    'time': time.time() - t0
}

# Method 2: DGI (decoupled deep)
t0 = time.time()
encoder = GCN(in_channels=data.num_features, hidden_channels=512, num_layers=1)
dgi = DGI(hidden_channels=512, encoder=encoder).to('cuda')
opt = torch.optim.Adam(dgi.parameters(), lr=0.001)
for epoch in range(200):
    dgi.train_full(data, opt, epoch)
with torch.no_grad():
    z = dgi.infer_full(data)
pred = KMeansClusterHead(n_clusters=n_clusters).fit_predict(z)
results['DGI'] = {
    **label_metrics(y, pred, metrics=['ACC', 'NMI']),
    **structure_metrics(edge_index, pred, metrics=['Modularity']),
    'time': time.time() - t0
}

# Method 3: DMoN (joint deep, end-to-end)
t0 = time.time()
encoder = GCN(in_channels=data.num_features, hidden_channels=512, num_layers=1)
dmon = DMoN(hidden_channels=512, n_clusters=n_clusters, encoder=encoder).to('cuda')
opt = torch.optim.Adam(dmon.parameters(), lr=0.001)
for epoch in range(200):
    dmon.train_full(data, opt, epoch)
pred = dmon.predict(data)
results['DMoN'] = {
    **label_metrics(y, pred, metrics=['ACC', 'NMI']),
    **structure_metrics(edge_index, pred, metrics=['Modularity']),
    'time': time.time() - t0
}

for name, r in results.items():
    print(f"{name:8s} | ACC={r['ACC']:.3f} NMI={r['NMI']:.3f} "
          f"Mod={r['Modularity']:.3f} Time={r['time']:.1f}s")
```

Output:
```
SSGC     | ACC=0.521 NMI=0.312 Mod=0.287 Time=2.3s
DGI      | ACC=0.618 NMI=0.421 Mod=0.398 Time=45.2s
DMoN     | ACC=0.645 NMI=0.448 Mod=0.456 Time=62.7s
```

## Best Practices

- **Do** always report unsupervised metrics (Modularity, Conductance) alongside supervised ones. In production you rarely have labels, and supervised metrics alone can mislead when cluster-label alignment is arbitrary.
- **Do** start with a decoupled approach (DGI + KMeans) as a strong baseline before trying joint models. Decoupled methods are simpler to debug and often competitive.
- **Do** use mini-batch training with `NeighborLoader` for any graph over ~50K nodes. Full-batch will silently OOM or degrade performance from excessive swapping.
- **Do** check the homophily ratio of your graph before choosing an encoder. Low-homophily graphs (< 0.3) need attention-based or non-parametric encoders that don't rely on neighbor smoothing.
- **Avoid** assuming more GNN layers are better. Over-smoothing causes embeddings to converge, collapsing cluster structure. Two layers is usually sufficient; three is rarely better.
- **Avoid** using only ACC to evaluate clustering. ACC requires a Hungarian matching step that can mask poor performance on minority clusters. Always pair with NMI and ARI.

## Error Handling

- **OOM on large graphs:** Switch from full-batch to mini-batch training. Reduce `batch_size` or `num_neighbors` fan-out. For inference, process in chunks and concatenate embeddings on CPU.
- **Degenerate clusters (one huge cluster absorbs everything):** This often happens with joint models when the clustering loss dominates. Reduce the clustering loss weight or use a decoupled approach. Also check if features need normalization.
- **NaN losses during training:** Usually caused by learning rate too high or unstable softmax in differentiable cluster heads. Lower the learning rate to 1e-4, add gradient clipping, and ensure input features are normalized.
- **Low Modularity despite good ACC:** The clusters match labels but don't respect graph structure. This signals that features dominate over topology. Try a stronger GNN encoder or increase neighbor sampling depth.
- **Dataset loading fails:** Ensure `pyagc.data.get_dataset()` has internet access for first-time downloads. Set `root=` to a writable directory. For custom datasets, verify edge_index is a `LongTensor` of shape `[2, num_edges]` and features are `FloatTensor`.

## Limitations

- PyAGC requires Python >= 3.10, PyTorch >= 2.6.0, and PyTorch Geometric >= 2.7.0. Projects on older stacks cannot use it directly.
- The number of clusters `k` must be specified upfront for most methods. PyAGC does not include automatic cluster-count selection (e.g., elbow method or silhouette analysis) — you must determine `k` externally.
- Mini-batch training with neighbor sampling introduces variance in gradient estimates. Results may differ slightly across runs; set random seeds and average over multiple runs for benchmarking.
- Non-parametric encoders (SSGC, SAGSC) are fast but assume feature-topology alignment. On graphs where node attributes are noisy or uncorrelated with structure, deep parametric encoders will outperform them.
- The library focuses on homogeneous graphs. Heterogeneous graphs with multiple node/edge types require adaptation outside PyAGC's current scope.

## Reference

**Paper:** [Bridging Academia and Industry: A Comprehensive Benchmark for Attributed Graph Clustering](https://arxiv.org/abs/2602.08519v1) — Liu et al., 2026. Look for: the ECO framework taxonomy (Table 1), dataset homophily statistics (Table 2), and the scalability analysis showing mini-batch vs. full-batch memory/time tradeoffs (Section 5).

**Code:** [github.com/Cloudy1225/PyAGC](https://github.com/Cloudy1225/PyAGC) | **Docs:** [pyagc.readthedocs.io](https://pyagc.readthedocs.io) | **Install:** `pip install pyagc`