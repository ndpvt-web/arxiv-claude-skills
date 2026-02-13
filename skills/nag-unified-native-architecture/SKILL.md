---
name: "nag-unified-native-architecture"
description: "Encode graph structure directly into LM attention masks and positional IDs instead of using external GNN encoders. Use when: 'represent this knowledge graph for an LLM', 'encode graph topology without a GNN', 'build attention masks from graph edges', 'serialize a graph for transformer input', 'implement NAG-Zero or NAG-LoRA', 'graph reasoning without external encoders'."
---

# NAG: Native Architecture for Encoder-Free Text-Graph Modeling

This skill enables Claude to help users implement the NAG (Native Architecture for Graphs) framework, which eliminates the need for external Graph Neural Networks when processing text-attributed graphs with language models. Instead of a segregated GNN-then-LM pipeline, NAG encodes graph topology directly through custom attention masks and recalibrated positional IDs within the LM's own transformer layers. This produces a unified model that reads node text, edge text, and structural relationships in a single forward pass.

## When to Use

- When a user wants to feed a knowledge graph, scene graph, or relational dataset into a transformer LM without training a separate GNN encoder
- When building a text-graph QA system (e.g., over Wikidata, Freebase, or domain-specific knowledge bases) and the user wants to avoid the GNN+LM dual-path architecture
- When the user asks how to construct topology-aware attention masks for graph-structured inputs
- When implementing graph reasoning tasks (cycle detection, shortest path, reachability, node degree) inside a language model
- When the user needs to preserve a base LM's text capabilities while adding graph understanding (NAG-Zero scenario)
- When serializing heterogeneous graphs (nodes with text, edges with labels) into a token sequence for causal LMs

## Key Technique

**The core insight**: Graph topology does not require a separate neural encoder. A transformer's self-attention mechanism already performs message-passing -- you just need to control *which* tokens attend to *which*. NAG achieves this by (1) serializing graph elements into a flat token sequence with special boundary tokens, (2) constructing a binary attention mask that enforces the graph's edge structure, and (3) recalibrating positional IDs so that serialization order is irrelevant to the model.

**Attention mask construction** uses four hierarchical levels combined via logical OR. *Intra-element* masks allow causal attention within each node/edge's text span. *Inter-element* masks route information along edges: source node hub -> edge hub -> target node hub (where "hubs" are the closing special tokens `</n>` and `</e>` that aggregate each element's semantics). *Global* masks connect all hubs to a `</g>` super-node. *Query* masks let the task query attend to graph hubs (sparse) or all graph tokens (full). This four-level mask replaces what a GNN would do structurally.

**Positional recalibration** assigns all hub tokens the same positional ID (`p_start + max_element_length`), making every graph element equidistant from the query in the model's RoPE-based position space. This enforces permutation invariance -- reordering nodes in the serialized sequence does not change the model's output. Two adaptation strategies are offered: **NAG-Zero** adds gated low-rank adapters that activate only on special structural tokens (zero impact on plain text), while **NAG-LoRA** injects LoRA matrices into Q/K/V projections for deeper structural-semantic fusion.

## Step-by-Step Workflow

### 1. Define the graph schema and special tokens

Add six special tokens to the tokenizer vocabulary: `<g>`, `</g>`, `<n>`, `</n>`, `<e>`, `</e>`. Resize the model's embedding matrix to accommodate them.

### 2. Serialize the graph into a flat token sequence

Wrap the entire graph in `<g>...</g>`. Each node's text content goes inside `<n>...</n>`, each edge's label inside `<e>...</e>`. Append the task query `Q` after the closing `</g>`. The ordering of nodes and edges within `<g>` is arbitrary -- topology comes from the mask, not position.

```
<g><n>The Matrix</n><n>Sci-Fi</n><e>belongs_to</e><n>Keanu Reeves</n><e>stars_in</e></g> What genre is The Matrix?
```

### 3. Build the four-level attention mask

Construct a binary matrix `M` of shape `(seq_len, seq_len)`:

- **Level 1 (Intra-element)**: For tokens `i, j` belonging to the same element `u`, set `M[i,j] = 1` if `j <= i` (causal within element). All cross-element pairs at this level are 0.
- **Level 2 (Inter-element)**: For each directed edge `(v_src, e, v_tgt)`, set `M[hub(e), hub(v_src)] = 1` and `M[hub(v_tgt), hub(e)] = 1`. The hub of an element is its closing token index (`</n>` or `</e>`).
- **Level 3 (Global)**: Set `M[hub(G), hub(u)] = 1` for every element `u`, and `M[i, start(G)] = 1` for all `i` (the `<g>` token is a universal anchor).
- **Level 4 (Query)**: For sparse mode, let query tokens attend to all hub positions. For full mode, let query tokens attend to all graph token positions.

Combine: `M = Level1 | Level2 | Level3 | Level4`.

### 4. Assign recalibrated positional IDs

Compute `max_len = max(|tokens_in_element|)` across all elements. Set every hub token's position ID to `p_start + max_len` (where `p_start` is the `<g>` token's position). Set `</g>` to `p_start + max_len + 1`. Resume standard incremental IDs for the query tokens starting from `p_start + max_len + 2`. Internal tokens within each element use sequential IDs starting from `p_start + 1`.

### 5. Choose an adaptation strategy

- **NAG-Zero** (preserves base LM exactly for non-graph inputs): Insert gated low-rank adapters between transformer layers. These adapters apply only to hub/special tokens; text tokens pass through unchanged. Formula: `h_out = h + sigmoid(W_g_up @ W_g_down @ h) * (W_v_up @ W_v_down @ h)` with rank `r << d`.
- **NAG-LoRA** (stronger graph adaptation): Apply standard LoRA (rank 8 is a good default) to the Q, K, V projection matrices of the attention layers.

### 6. Prepare training data as (graph, query, answer) triplets

Each sample consists of the serialized graph sequence, the attention mask, the recalibrated position IDs, and the target answer. Build an edge list mapping `(src_node_index, edge_index, tgt_node_index)` to drive Level 2 mask construction.

### 7. Implement the custom attention forward pass

Override the model's attention computation to accept the external mask `M`. In HuggingFace transformers, pass `attention_mask=M` and `position_ids=recalibrated_ids` to the model's forward method. Ensure the mask is broadcastable to `(batch, num_heads, seq_len, seq_len)`.

### 8. Train and evaluate

Fine-tune on the target graph task. For topological reasoning, use accuracy. For QA tasks, use Hit@1 or exact match. Monitor both graph task performance and (for NAG-Zero) plain-text benchmark scores to confirm no degradation.

## Concrete Examples

**Example 1: Knowledge Base QA over a subgraph**

User: "I have a knowledge graph subgraph around the entity 'Albert Einstein' with edges like (Albert Einstein, born_in, Ulm), (Albert Einstein, field, Physics), (Ulm, country, Germany). How do I format this for NAG and build the attention mask?"

Approach:
1. Serialize: `<g><n>Albert Einstein</n><e>born_in</e><n>Ulm</n><e>field</e><n>Physics</n><e>country</e><n>Germany</n></g> Where was Albert Einstein born?`
2. Record element hub positions (the index of each `</n>` and `</e>` token).
3. Build Level 2 mask from the edge list:
   - `(Einstein, born_in, Ulm)` -> `M[hub(born_in), hub(Einstein)] = 1; M[hub(Ulm), hub(born_in)] = 1`
   - `(Einstein, field, Physics)` -> `M[hub(field), hub(Einstein)] = 1; M[hub(Physics), hub(field)] = 1`
   - `(Ulm, country, Germany)` -> `M[hub(country), hub(Ulm)] = 1; M[hub(Germany), hub(country)] = 1`
4. Assign all hubs the same positional ID. Query tokens get sequential IDs after.

Output (mask pseudocode):
```python
hubs = {"Einstein": 3, "born_in": 5, "Ulm": 7, "field": 9,
        "Physics": 11, "country": 13, "Germany": 15, "G": 16}
# Level 2 edges
mask[hubs["born_in"], hubs["Einstein"]] = 1
mask[hubs["Ulm"], hubs["born_in"]] = 1
mask[hubs["field"], hubs["Einstein"]] = 1
mask[hubs["Physics"], hubs["field"]] = 1
mask[hubs["country"], hubs["Ulm"]] = 1
mask[hubs["Germany"], hubs["country"]] = 1
# Level 3: all hubs connect to </g>
for h in hubs.values():
    mask[hubs["G"], h] = 1
```

**Example 2: Implementing NAG-Zero adapters in PyTorch**

User: "Show me how to implement the NAG-Zero gated adapter that only activates for structural tokens."

Approach:
1. Define a low-rank gated adapter module.
2. Apply it conditionally based on a structural-token mask.

Output:
```python
import torch
import torch.nn as nn

class NAGZeroAdapter(nn.Module):
    def __init__(self, d_model: int, rank: int = 8):
        super().__init__()
        # Gate pathway
        self.gate_down = nn.Linear(d_model, rank, bias=False)
        self.gate_up = nn.Linear(rank, d_model, bias=False)
        # Value pathway
        self.value_down = nn.Linear(d_model, rank, bias=False)
        self.value_up = nn.Linear(rank, d_model, bias=False)

    def forward(self, h: torch.Tensor, structural_mask: torch.BoolTensor):
        """
        h: (batch, seq_len, d_model)
        structural_mask: (batch, seq_len) - True for special tokens
        """
        gate = torch.sigmoid(self.gate_up(self.gate_down(h)))
        value = self.value_up(self.value_down(h))
        adapter_out = gate * value
        # Zero out adapter contribution for non-structural tokens
        adapter_out = adapter_out * structural_mask.unsqueeze(-1).float()
        return h + adapter_out
```

**Example 3: Building recalibrated position IDs**

User: "How do I compute the recalibrated positional IDs for a serialized graph with varying node text lengths?"

Approach:
1. Tokenize each element, record lengths.
2. Find `max_len` across all elements.
3. Assign hub positions uniformly.

Output:
```python
def build_position_ids(elements: list[dict], query_tokens: list[int]):
    """
    elements: [{"tokens": [...], "hub_idx": int}, ...]
    Returns: position_ids tensor
    """
    p_start = 0  # position of <g>
    max_elem_len = max(len(e["tokens"]) for e in elements)
    p_hub = p_start + max_elem_len

    position_ids = [p_start]  # <g> token
    for elem in elements:
        # Internal tokens get sequential IDs from p_start+1
        for offset in range(len(elem["tokens"]) - 1):  # exclude hub
            position_ids.append(p_start + 1 + offset)
        position_ids.append(p_hub)  # hub (closing tag) gets unified ID

    position_ids.append(p_hub + 1)  # </g> token

    # Query tokens resume incrementally
    q_start = p_hub + 2
    for i in range(len(query_tokens)):
        position_ids.append(q_start + i)

    return torch.tensor(position_ids)
```

## Best Practices

- **Do**: Use closing special tokens (`</n>`, `</e>`) as semantic hubs -- they aggregate meaning from their element's content via causal intra-element attention before participating in inter-element message-passing.
- **Do**: Keep the serialization order arbitrary and rely entirely on the attention mask for topology. This enforces permutation invariance and prevents the model from learning spurious positional biases.
- **Do**: Start with NAG-Zero if you need to preserve the base model's general language capabilities (e.g., the model also handles plain-text tasks). Switch to NAG-LoRA only when you need stronger graph task performance and can tolerate some capability trade-off.
- **Do**: Use sparse query attention (query attends only to hubs) for efficiency on large graphs; use full query attention when maximum context is needed for complex reasoning.
- **Avoid**: Encoding connectivity information in the text itself (e.g., "Node A is connected to Node B") when using NAG -- the attention mask already captures this, and redundant textual encoding wastes context and can confuse the model.
- **Avoid**: Assigning sequential positional IDs to hub tokens -- this breaks permutation invariance and makes the model sensitive to arbitrary node ordering in the serialized sequence.

## Error Handling

- **Disconnected nodes**: Nodes with no edges still participate via Level 1 (self-attention on their content) and Level 3 (connection to the global `</g>` hub). No special handling needed.
- **Large graphs exceeding context length**: Subsample the graph around the query entity (k-hop neighborhood). NAG works best with graphs of 5-50 nodes in current experiments. For larger graphs, consider hierarchical subgraph extraction before serialization.
- **Missing edge text**: If edges have no labels, use a single generic token (e.g., "related") inside `<e>...</e>`. The structural information still flows through the attention mask.
- **Attention mask shape mismatch**: Verify the mask is `(1, 1, seq_len, seq_len)` for broadcasting across batch and heads. A common bug is forgetting to add the batch and head dimensions.
- **Hub bottleneck on high-degree nodes**: The closing token of a node with many neighbors must aggregate information from many edge hubs. This can create an information bottleneck. For nodes with degree > 15, consider splitting into multiple sub-nodes or using full query attention to bypass the hub.

## Limitations

- NAG has been validated on graphs with up to ~20 nodes and ~200 edges. Scaling behavior on graphs with hundreds or thousands of nodes is not established.
- The semantic hub mechanism can bottleneck on the "Connected Nodes" task where a single closing token must represent an entire neighborhood -- linearization methods that explicitly enumerate neighbors can outperform NAG here.
- NAG requires custom attention mask construction per sample, which prevents use of flash attention implementations that assume causal or dense masks. This impacts inference throughput.
- Currently validated on Qwen3 (600M and 8B). Adaptation to other architectures (GPT, LLaMA) requires verifying RoPE compatibility and attention mask API support.
- Undirected graphs must be encoded as bidirectional edges (two directed edges per undirected edge), doubling the edge count.

## Reference

**Paper**: [NAG: A Unified Native Architecture for Encoder-free Text-Graph Modeling in Language Models](https://arxiv.org/abs/2601.22657v1) -- Gong et al., 2026. Focus on Section 3 (methodology) for the four-level attention mask construction and positional recalibration, and Section 4 for ablations showing the contribution of each mask level.