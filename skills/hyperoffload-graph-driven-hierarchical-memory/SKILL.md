---
name: "hyperoffload-graph-driven-hierarchical-memory"
description: "Design and implement compiler-driven hierarchical memory offloading for LLM inference and training on multi-tier memory systems. Applies graph-level scheduling of data movement to hide memory transfer latency behind computation. Use when: 'optimize LLM memory offloading', 'schedule prefetch and evict in computation graph', 'hierarchical memory management for inference', 'hide data transfer latency in ML pipeline', 'compiler pass for memory placement', 'reduce peak GPU memory with offloading'."
---

# HyperOffload: Graph-Driven Hierarchical Memory Management for LLMs

This skill enables Claude to design and implement **compiler-assisted memory offloading systems** that treat data movement (prefetch, store, evict) as explicit, schedulable nodes in a computation graph rather than opaque runtime side-effects. Based on the HyperOffload framework, the core technique is to represent remote/host memory transfers as first-class IR operators, perform compile-time tensor lifetime analysis, and apply a global execution-order refinement algorithm that overlaps data transfers with compute-intensive regions. This eliminates the reactive scheduling and pipeline stalls inherent in runtime-only offloading approaches.

## When to Use

- When the user needs to **reduce peak device memory** (HBM/VRAM) for LLM inference or training by offloading tensors to host RAM, NVMe, or remote memory pools.
- When building a **custom ML compiler pass** that inserts prefetch/evict operations into a computation graph (e.g., in MLIR, TorchScript, ONNX, or MindIR).
- When the user asks to **schedule data transfers to overlap with computation** in a pipeline or graph-based execution engine.
- When designing a **memory management policy** for multi-tier memory hierarchies (HBM + DDR + remote shared memory + NVMe).
- When optimizing **KV cache placement** for long-context LLM serving across memory tiers.
- When implementing **tensor lifetime analysis** to decide which activations, parameters, or optimizer states should be offloaded and when.
- When the user wants to move from a runtime-based offloading approach (which causes pipeline stalls) to a static, graph-driven scheduling approach.

## Key Technique

**The core insight**: Runtime-based offloading (e.g., PyTorch's `pin_memory` + async transfers, DeepSpeed ZeRO-Offload) operates with a local view -- it reacts to memory pressure as it happens, issuing DMA requests on-the-fly. This causes frequent pipeline interruptions: the CPU must inspect state, issue transfer commands, and synchronize. The paper shows runtime prefetching can introduce a **2.7x slowdown** compared to baseline because the system lacks visibility into future operations.

**Graph-driven alternative**: HyperOffload inserts explicit **cache operators** -- `Prefetch` (remote-to-device), `Store` (device-to-remote), and `Detach` (release residency) -- directly into the compiler's intermediate representation. These operators participate in standard graph analyses: dependency inference, topological ordering, and liveness analysis. Because the compiler sees the entire computation graph at once, it can determine **exactly** when each tensor is produced, consumed, offloaded, and reloaded. This compile-time visibility is fundamentally unavailable to runtime systems.

**Execution-order refinement**: With cache operators as graph nodes, a global scheduling algorithm evaluates feasible positions for each transfer operation. It uses a cost model balancing computation time (`C_comp`) against transfer latency (`C_trans`) and available bandwidth. The algorithm places each prefetch as early as possible (to complete before consumption) while minimizing unnecessary memory residency. The result: data transfers are hidden behind compute-intensive regions (e.g., matrix multiplications in transformer layers), achieving up to 26% peak memory reduction with negligible (<1%) latency overhead on short sequences.

## Step-by-Step Workflow

### 1. Profile the computation graph and identify memory pressure points
Extract the operator DAG from your model (e.g., via `torch.fx.symbolic_trace`, ONNX export, or MindSpore's MindIR). Identify peak memory usage by computing a **memory timeline** -- the sum of all live tensors at each point in the topological execution order. Find the peak and the operators contributing most to it.

### 2. Classify tensors by offload suitability
For each tensor in the graph, compute: (a) **size** in bytes, (b) **lifetime** (distance between production and last consumption in topological order), (c) **access pattern** (single-use vs. reused). Tensors that are large, have long lifetimes, and are accessed infrequently are the best offload candidates. Specifically:
- **Model parameters** in MoE (Mixture of Experts): offload inactive expert weights.
- **KV cache blocks**: offload past-context blocks that won't be accessed for many tokens.
- **Optimizer states**: offload after the update step, prefetch before the next update.
- **Activations with short lifetimes or fine-grained access**: do NOT offload -- transfer overhead exceeds memory savings.

### 3. Define cache operators as graph nodes
Create three operator types and insert them into the IR:
```python
# Pseudocode for cache operator definitions
class Prefetch(Op):
    """Async transfer: remote/host memory -> device HBM."""
    inputs: [tensor_ref, source_tier]
    outputs: [device_tensor]
    # Generates async DMA; downstream ops depend on completion.

class Store(Op):
    """Async transfer: device HBM -> remote/host memory."""
    inputs: [device_tensor, dest_tier]
    outputs: [remote_tensor_ref]
    # Frees device memory after transfer completes.

class Detach(Op):
    """Release device residency without transfer (tensor already stored)."""
    inputs: [device_tensor]
    outputs: []
    # Marks tensor as evicted from device memory map.
```
Insert these into the graph such that they respect data dependencies: `Prefetch` must precede the first consumer, `Store` must follow the last producer or consumer that needs device-resident data.

### 4. Build the dependency-aware execution schedule
Perform a topological sort of the augmented graph (original ops + cache ops). For each cache operator, compute the **feasible window** -- the range of positions in the topological order where it can be legally placed without violating dependencies.

### 5. Apply the execution-order refinement algorithm
For each `Prefetch` operator, find the earliest legal position that allows the transfer to complete before the consuming operator executes. Estimate transfer time as `tensor_size / bandwidth`. For each `Store` operator, find the latest position that frees memory before the next peak. Use a cost function:
```
cost(position) = max(0, C_trans - C_comp_overlap) + alpha * memory_residency_duration
```
Where `C_comp_overlap` is the total compute time of operators between the cache op and its consumer/producer, and `alpha` penalizes holding data on-device unnecessarily. Select the position minimizing this cost.

### 6. Implement asynchronous transfer primitives
Map cache operators to platform-specific async DMA:
- **CUDA**: `cudaMemcpyAsync` on a dedicated transfer stream, with `cudaEvent` synchronization.
- **Ascend NPU**: `aclrtMemcpyAsync` with HCCS/PCIe DMA channels.
- **CPU offload**: `memcpy` on a background thread with a ring buffer.
Ensure the execution engine waits on the transfer event before running dependent compute operators.

### 7. Validate with memory simulation
Before running on hardware, simulate the schedule: walk the execution order, track device memory usage at each step (allocations from compute ops, frees from `Detach`/`Store` completions, allocations from `Prefetch` arrivals). Verify that peak simulated memory stays within the device budget. If not, increase offloading aggressiveness (lower the size threshold for offload candidates) or adjust the cost function weighting.

### 8. Integrate as a compiler pass
Package the above as a reusable compiler transformation:
1. **Analysis pass**: tensor lifetime computation + candidate selection.
2. **Insertion pass**: add cache operators to the graph.
3. **Scheduling pass**: execution-order refinement.
4. **Lowering pass**: map cache operators to platform DMA primitives.
Register this pass sequence in your compiler pipeline (e.g., as a `torch.fx` transform, an MLIR pass, or a MindSpore graph optimization).

## Concrete Examples

**Example 1: Reducing inference memory for a MoE model**

User: "My DeepSeek-V3 inference is running out of HBM. Only 2 of 256 experts are active per token but all expert weights stay in memory. How can I offload inactive experts?"

Approach:
1. Trace the model's computation graph and identify the expert gating + expert FFN subgraphs.
2. For each expert FFN block, the gating function determines activation at runtime -- but the set of experts is known per-layer at graph construction. Insert `Prefetch` nodes for the top-K experts' weights gated in, positioned before the expert computation.
3. Insert `Store` nodes after each expert's output is computed, returning weights to host/remote memory.
4. Schedule prefetches to overlap with the attention computation preceding the MoE layer (typically ~2-5ms of compute, enough to transfer expert weights at PCIe Gen5 bandwidth).
5. Simulate: with 2 active experts (each ~256MB) instead of 256 resident, peak HBM drops by ~98% of expert weight memory.

Output:
```python
# torch.fx transform for MoE expert offloading
def insert_expert_offload(graph: fx.Graph, bandwidth_gbps: float = 32.0):
    for node in graph.nodes:
        if node.op == 'call_module' and 'expert_ffn' in node.target:
            expert_size = get_param_size(node.target)
            transfer_time = expert_size / (bandwidth_gbps * 1e9)

            # Find preceding compute region for overlap
            preceding_attn = find_preceding_attention(graph, node)
            attn_compute_time = estimate_compute_time(preceding_attn)

            # Insert prefetch right after attention starts
            with graph.inserting_after(preceding_attn):
                prefetch = graph.call_function(async_prefetch,
                    args=(node.target, 'host', 'device'))

            # Insert store after expert output is consumed
            last_consumer = find_last_consumer(graph, node)
            with graph.inserting_after(last_consumer):
                store = graph.call_function(async_store,
                    args=(node.target, 'device', 'host'))

            # Add dependency edge: expert_ffn depends on prefetch
            node.args = (prefetch, *node.args)
    graph.lint()
```

**Example 2: KV cache tiering for long-context inference**

User: "I'm serving a 128K-context model but KV cache for long sequences exceeds my 80GB HBM. I want to keep recent tokens' KV on-device and offload older blocks."

Approach:
1. Partition KV cache into fixed-size blocks (e.g., 4096 tokens each).
2. In the attention computation graph, identify which KV blocks each attention head accesses per decoding step.
3. Insert `Prefetch` for blocks in the sliding window of recent tokens + any blocks needed for sparse attention patterns (e.g., NSA sink tokens).
4. Insert `Store` for blocks that fall outside the active window after each decoding step.
5. Schedule prefetches to overlap with the MLP computation of the preceding layer.

Output:
```python
# KV block offload scheduling
class KVCacheManager:
    def __init__(self, block_size=4096, device_budget_blocks=16,
                 tiers=['hbm', 'ddr', 'remote']):
        self.block_size = block_size
        self.device_budget = device_budget_blocks
        self.placement = {}  # block_id -> tier

    def plan_transfers(self, attention_graph, active_blocks, step):
        prefetches, stores = [], []
        for block_id in active_blocks:
            if self.placement.get(block_id) != 'hbm':
                src = self.placement.get(block_id, 'ddr')
                prefetches.append(Prefetch(block_id, src, 'hbm'))

        resident = [b for b, t in self.placement.items() if t == 'hbm']
        evict = sorted(
            [b for b in resident if b not in active_blocks],
            key=lambda b: self.last_access[b]  # LRU
        )
        for block_id in evict[:len(resident) - self.device_budget]:
            stores.append(Store(block_id, 'hbm', 'ddr'))

        return self.schedule_with_overlap(attention_graph, prefetches, stores)
```

**Example 3: Compiler pass for training activation offloading**

User: "I'm implementing gradient checkpointing but still running out of memory during backward pass. Can I offload stashed activations to host memory between forward and backward?"

Approach:
1. In the training graph, identify activations saved for backward (the "stash" points in gradient checkpointing).
2. Insert `Store` immediately after each stash point to offload the activation to host memory.
3. Insert `Prefetch` just before the corresponding backward operator that consumes the activation.
4. Schedule each `Prefetch` to overlap with the backward computation of the preceding layer (backward ops are compute-heavy -- large matmuls provide ample overlap time).
5. Verify via memory simulation that peak device memory during backward stays within budget.

Output:
```
Forward pass graph (after transform):
  fwd_layer_0 -> Store(act_0, host) -> fwd_layer_1 -> Store(act_1, host) -> ...

Backward pass graph (after transform):
  ... -> Prefetch(act_1, host) -> bwd_layer_1 -> Prefetch(act_0, host) -> bwd_layer_0

Schedule (execution order after refinement):
  [bwd_layer_2] [Prefetch(act_1)] [bwd_layer_1] [Prefetch(act_0)] [bwd_layer_0]
   ^-- act_1 transfer overlaps with bwd_layer_2 compute
```

## Best Practices

- **Do**: Always compute the memory-time tradeoff before offloading. A tensor must be large enough and have a long enough lifetime to justify the transfer cost. Rule of thumb: `tensor_size / bandwidth > 0.1 * overlapping_compute_time` means the transfer won't fully hide.
- **Do**: Use asynchronous transfers on dedicated DMA streams/channels. Synchronous transfers defeat the purpose of graph-driven scheduling.
- **Do**: Profile the actual bandwidth of each memory tier under realistic contention. PCIe bandwidth drops significantly under multi-GPU contention; CXL/HCCS bandwidth varies with topology.
- **Do**: Treat the cache operators as first-class graph nodes with proper dependency edges. This ensures the framework's existing dead-code elimination, fusion, and reordering passes interact correctly with offloading.
- **Avoid**: Offloading tensors with very short lifetimes (used within 1-2 ops of production) -- the transfer overhead will exceed the memory savings.
- **Avoid**: Runtime-only offloading with per-operator memory checks. The paper demonstrates this causes a 2.7x slowdown due to CPU-side inspection overhead and reactive scheduling.
- **Avoid**: Treating all tensors equally. MoE expert weights, KV cache blocks, optimizer states, and short-lived activations each need different offload policies based on their access patterns.

## Error Handling

- **Bandwidth oversubscription**: If multiple cache operators compete for the same DMA channel, transfers will queue and latency hiding fails. Detect this during scheduling by tracking channel utilization per time slot. If oversubscribed, reduce the number of concurrent offload candidates or stagger transfers across layers.
- **Graph mutation conflicts**: Inserting cache operators can break existing compiler optimizations (e.g., operator fusion that assumed tensor locality). Run cache operator insertion *after* compute-graph fusion passes, and mark cache operators as optimization barriers where needed.
- **Memory budget violations**: If the memory simulation shows a peak exceeding the device budget, iteratively lower the offload threshold (offload smaller tensors) or extend the prefetch window (start prefetching earlier, accepting longer residency).
- **Non-deterministic execution order**: If the runtime doesn't guarantee topological execution order (e.g., with dynamic scheduling), the static schedule is invalid. Enforce execution order through explicit stream dependencies or barrier operators.

## Limitations

- **Dynamic computation graphs** (e.g., models with data-dependent control flow) cannot be fully analyzed at compile time. The technique works best with static or semi-static graphs. For dynamic cases, fall back to a hybrid approach: static scheduling for the fixed backbone, runtime scheduling for dynamic branches.
- **Small models with low memory pressure** don't benefit -- the overhead of cache operators and DMA management exceeds the memory savings.
- **Bandwidth-limited systems** (e.g., PCIe Gen3 at 16 GB/s) may not provide enough transfer bandwidth to hide latency behind compute, especially for memory-bound operations like attention on short sequences.
- **The technique assumes cooperative memory tiers** where the device can issue async DMA to host/remote memory. Systems without DMA engines or with high-latency memory access (e.g., network-attached storage) are not suitable targets.
- **Optimal scheduling is NP-hard** in general. The greedy cost-model approach is effective in practice but may miss globally optimal schedules for deeply interdependent transfer chains.

## Reference

**Paper**: [HyperOffload: Graph-Driven Hierarchical Memory Management for Large Language Models on SuperNode Architectures](https://arxiv.org/abs/2602.00748v2) (Liu et al., 2026)

Key sections to study: Section 3 (cache operator IR design), Section 4 (execution-order refinement algorithm and cost model), Section 5 (evaluation showing 26% memory reduction with <1% latency overhead). The critical takeaway is that representing data movement as explicit graph nodes -- rather than runtime side-effects -- unlocks global scheduling that can provably hide transfer latency behind compute.