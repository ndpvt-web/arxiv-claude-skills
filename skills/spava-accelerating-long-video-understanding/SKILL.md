---
name: "spava-accelerating-long-video-understanding"
description: "Implement Spava-style sequence-parallel approximate attention for accelerating long-video inference across multiple GPUs. Use when: 'speed up long video inference', 'distribute video attention across GPUs', 'sequence parallel video processing', 'reduce prefill latency for video LMMs', 'multi-GPU video understanding pipeline', 'approximate attention for long sequences'."
---

# Spava: Sequence-Parallel Approximate Attention for Long-Video Understanding

This skill enables Claude to help users implement the Spava framework — a multi-GPU sequence-parallel inference system that accelerates the prefill stage of Large Multimodal Models (LMMs) processing long videos. The core idea is to split visual token sequences across GPU hosts, compute approximate attention using anchor blocks and selectively communicated "passing blocks" of important KV pairs, and fuse forward passes with overlapped communication to achieve up to 12.7x speedup over FlashAttention with minimal accuracy loss (~2 percentage points). Use this skill to design, implement, debug, or optimize distributed long-video inference pipelines.

## When to Use

- When the user needs to process long videos (64+ frames) through a multimodal LLM and the prefill stage is a bottleneck
- When the user asks to distribute video token sequences across multiple GPUs for parallel attention computation
- When the user wants to implement approximate attention that selectively communicates only the most important KV pairs between hosts
- When the user is building on decoder-only multimodal architectures (InternVL, Qwen2.5-VL, LLaVA-style) and needs multi-GPU inference
- When the user asks about ring attention, ZigZag load balancing, or passing-block selection for long sequences
- When the user wants to reduce inter-GPU communication overhead while preserving long-range video dependencies

## Key Technique

**The Problem.** During the prefill stage of video LMMs, the full sequence of visual embeddings (often thousands of tokens for 64+ frames) requires dense self-attention, which scales quadratically. Single-GPU methods either compress/prune visual tokens (losing information) or apply sparse attention patterns (limited speedup). Sequence-parallel approaches like Ring Attention distribute the sequence but still compute exact full attention, requiring expensive all-to-all KV communication.

**Spava's Solution: Approximate Attention with Anchor and Passing Blocks.** Spava splits the input sequence into three logical components per host: (1) an *anchor block* — the first `la = n/64` embeddings from the full video, replicated on every host to preserve global context; (2) a *context block* — the host's local partition of the remaining sequence; and (3) *passing blocks* — the top `lp = n/128` most important KV pairs selected from each remote host's context, communicated asynchronously. Each host computes attention over its anchor + local context + received passing blocks, rather than the entire sequence. This dramatically reduces both computation (fewer KV pairs per host) and communication (only essential KV pairs travel between hosts).

**System Optimizations.** Three engineering choices make this practical: (a) *ZigZag virtual-host mapping* — 2H virtual hosts map to H physical hosts such that virtual hosts h and (2H-1-h) share a GPU, equalizing FLOPs across all physical hosts; (b) *Fused forward pass* — query blocks are concatenated with context blocks so linear projections happen once instead of twice; (c) *Overlapped communication-computation* — AsyncAllGather of passing blocks runs concurrently with query attention computation, hiding communication latency behind useful work.

## Step-by-Step Workflow

1. **Distribute video frames across GPU hosts.** Assign `F(h) = floor(F/H) + I[h < F mod H]` frames to virtual host h, where F is total frames and H is the number of physical hosts. Use 2H virtual hosts mapped to H physical hosts for load balancing.

2. **Encode visual embeddings in parallel.** Each host independently runs the vision encoder on its assigned frames. No communication is needed during visual encoding — this is embarrassingly parallel.

3. **AllGather embeddings and construct sequence partitions.** After encoding, perform a single AllGather to collect all visual embeddings. Then split the concatenated sequence into: anchor block (first `n/64` tokens), query block (the text query tokens, placed at the end), and context blocks (remaining visual tokens divided equally across hosts).

4. **Select passing blocks via importance scoring.** On each host, compute attention scores between the query block Q and the local context block's keys K. Select the top `lp = n/128` KV pairs with the highest attention scores. These are the "passing blocks" — the most query-relevant information from each host.

5. **Initiate asynchronous communication of passing blocks.** Use AsyncAllGather to begin transferring each host's passing blocks to all other hosts. This communication overlaps with the next computation step.

6. **Compute query attention concurrently.** While passing blocks are in transit, compute full attention for the query block against the anchor block. This keeps GPUs busy during communication.

7. **Compute context attention with received passing blocks.** Once passing blocks arrive, each host computes attention for its context block over: anchor KV + local context KV + all received passing blocks. Use FlashAttention kernels for the local computation.

8. **Merge distributed attention results.** Combine attention outputs from all hosts using online softmax with log-sum-exp (lse) values returned by FlashAttention. This produces mathematically correct merged attention without materializing the full attention matrix.

9. **Run the remainder of the forward pass normally.** Feed the merged attention output through the remaining transformer layers (MLP, layer norm, etc.) and generate the text response.

10. **Tune hyperparameters based on accuracy requirements.** If accuracy degrades, increase `lp` (passing length) first — accuracy is more sensitive to this than `la` (anchor length). If latency is the priority, reduce `lp` and increase host count H.

## Concrete Examples

**Example 1: Implementing Spava-style context splitting in PyTorch**

User: "I have a multi-GPU setup with 4 A100s and I'm running Qwen2.5-VL-7B for video QA. The prefill for 64 frames takes too long. Help me implement Spava's sequence partitioning."

Approach:
1. Compute total visual token count n from frame count and patch size (Qwen2.5-VL uses 14x14 patches per frame)
2. Implement the three-way split: anchor (n/64 tokens), query (text tokens), context (rest, divided across 4 GPUs)
3. Set up ZigZag mapping: 8 virtual hosts -> 4 physical GPUs, where virtual hosts (h, 7-h) share a GPU

Output:
```python
import torch
import torch.distributed as dist

def partition_sequence(visual_embeds: torch.Tensor, query_embeds: torch.Tensor,
                       num_hosts: int, rank: int):
    """Split sequence into anchor, context, and query blocks per Spava."""
    n = visual_embeds.shape[1]  # total visual tokens
    la = n // 64   # anchor length
    num_virtual = 2 * num_hosts

    # Anchor block: first la tokens, replicated on every host
    anchor = visual_embeds[:, :la, :]

    # Remaining visual tokens split across virtual hosts
    remaining = visual_embeds[:, la:, :]
    n_remaining = remaining.shape[1]

    # ZigZag: physical host `rank` handles virtual hosts `rank` and `(2H-1-rank)`
    vh1 = rank
    vh2 = num_virtual - 1 - rank
    chunk_size = n_remaining // num_virtual

    context_vh1 = remaining[:, vh1 * chunk_size:(vh1 + 1) * chunk_size, :]
    context_vh2 = remaining[:, vh2 * chunk_size:(vh2 + 1) * chunk_size, :]
    # Concatenate both virtual hosts' context on this physical GPU
    local_context = torch.cat([context_vh1, context_vh2], dim=1)

    return anchor, local_context, query_embeds
```

**Example 2: Implementing passing-block selection**

User: "How do I select the most important KV pairs to communicate between hosts? Show me the top-k selection logic."

Approach:
1. Compute attention scores between query tokens and each host's context keys
2. Aggregate scores per KV position (sum across query heads)
3. Select top-lp positions and gather corresponding K and V tensors

Output:
```python
def select_passing_blocks(query: torch.Tensor, context_keys: torch.Tensor,
                          context_values: torch.Tensor, n_total: int):
    """Select top-lp important KV pairs from local context for communication."""
    lp = n_total // 128  # passing length hyperparameter

    # query: [B, n_q, D], context_keys: [B, n_ctx, D]
    # Compute raw attention scores (no softmax needed, just for ranking)
    scores = torch.matmul(query, context_keys.transpose(-1, -2))  # [B, n_q, n_ctx]

    # Aggregate importance across all query positions
    importance = scores.sum(dim=1)  # [B, n_ctx]

    # Select top-lp most important positions
    _, top_indices = importance.topk(lp, dim=-1)  # [B, lp]
    top_indices_sorted = top_indices.sort(dim=-1).values

    # Gather selected KV pairs
    idx_expanded_k = top_indices_sorted.unsqueeze(-1).expand(-1, -1, context_keys.shape[-1])
    passing_keys = context_keys.gather(1, idx_expanded_k)
    passing_values = context_values.gather(1, idx_expanded_k)

    return passing_keys, passing_values
```

**Example 3: Overlapping communication with computation**

User: "I want to hide the AllGather latency for passing blocks. How do I overlap it with query attention?"

Approach:
1. Launch AsyncAllGather for passing blocks immediately after selection
2. Compute query-over-anchor attention while communication proceeds
3. Wait for communication to complete before computing context attention

Output:
```python
def spava_attention_with_overlap(query, anchor_kv, local_context_kv,
                                  local_passing_kv, group):
    """Overlap passing-block communication with query attention."""
    # Step 1: Start async communication of passing blocks
    passing_k_gathered = torch.empty(...)  # pre-allocate output buffer
    passing_v_gathered = torch.empty(...)
    handle_k = dist.all_gather_into_tensor(
        passing_k_gathered, local_passing_kv[0], group=group, async_op=True)
    handle_v = dist.all_gather_into_tensor(
        passing_v_gathered, local_passing_kv[1], group=group, async_op=True)

    # Step 2: Compute query attention over anchor (runs while AllGather proceeds)
    query_attn_out, query_lse = flash_attn_func(
        query, anchor_kv[0], anchor_kv[1], return_lse=True)

    # Step 3: Wait for communication to finish
    handle_k.wait()
    handle_v.wait()

    # Step 4: Compute context attention with anchor + local context + received passing blocks
    combined_keys = torch.cat([anchor_kv[0], local_context_kv[0], passing_k_gathered], dim=1)
    combined_vals = torch.cat([anchor_kv[1], local_context_kv[1], passing_v_gathered], dim=1)
    context_attn_out, context_lse = flash_attn_func(
        local_context_kv[0], combined_keys, combined_vals, return_lse=True)

    # Step 5: Merge results using online softmax with lse
    merged = merge_attention_with_lse(query_attn_out, query_lse,
                                       context_attn_out, context_lse)
    return merged
```

## Best Practices

- **Do:** Start with default hyperparameters `la = n/64` and `lp = n/128` — the paper's ablations show these provide the best speedup-accuracy tradeoff
- **Do:** Use ZigZag virtual-host mapping (pair virtual hosts h and 2H-1-h on the same GPU) to equalize FLOPs — without this, some GPUs idle while others compute extra passing blocks
- **Do:** Fuse the query and context forward passes into a single pass through the linear projection layers to halve parameter I/O
- **Do:** Use FlashAttention's returned log-sum-exp values for merging distributed attention results — this avoids numerical instability from naive softmax combination
- **Avoid:** Running Spava on a single GPU — it degenerates to standard FlashAttention with overhead; minimum useful configuration is 2 GPUs
- **Avoid:** Setting `lp` too low in pursuit of speed — accuracy is highly sensitive to passing length; halving `lp` can drop accuracy by 3-5 percentage points on VNBench

## Error Handling

| Issue | Cause | Fix |
|-------|-------|-----|
| OOM on AllGather output buffer | Passing blocks from all hosts exceed GPU memory | Reduce `lp` or increase host count H to shrink per-host passing block size |
| Accuracy drops sharply (>5%) | `lp` set too aggressively low | Increase `lp` toward `n/64`; check that anchor block is correctly replicated on all hosts |
| Uneven GPU utilization | Missing ZigZag mapping; some hosts process more passing blocks | Implement 2H virtual-to-H physical host mapping so paired virtual hosts share a GPU |
| Communication not overlapping | AsyncAllGather wait called too early | Ensure query-anchor attention computation is placed between the async launch and the wait call |
| NaN in merged attention output | Incorrect lse merging across hosts | Use the numerically stable online softmax merge: `out = (out1 * exp(lse1 - lse_max) + out2 * exp(lse2 - lse_max)) / exp(lse_combined - lse_max)` |
| Slow visual encoding | Frames not distributed before encoding | Distribute frames across hosts *before* running the vision encoder, not after |

## Limitations

- **Multi-GPU only.** Spava provides no benefit on a single GPU. The minimum practical setup is 2 GPUs with fast interconnect (NVLink preferred; InfiniBand adds ~0.75% accuracy cost).
- **Decoder-only architectures.** The framework is designed for and tested on decoder-only multimodal transformers (InternVL, Qwen2.5-VL). Encoder-decoder models would require rethinking the sequence partitioning.
- **Approximate, not exact.** The passing-block selection discards some KV pairs. For tasks requiring pixel-precise spatial reasoning across distant frames, the ~2 percentage point accuracy drop may matter.
- **Fixed anchor/passing ratios.** The hyperparameters `la` and `lp` are set as fractions of sequence length. For very short videos (<16 frames), the anchor and passing blocks may constitute the entire sequence, yielding no speedup.
- **Prefill stage only.** Spava accelerates the prefill (prompt processing) phase. It does not address autoregressive decoding latency, which is memory-bandwidth-bound rather than compute-bound.
- **No support for streaming/online video.** The framework assumes all frames are available upfront for partitioning. Real-time video streams would need a different approach.

## Reference

**Paper:** [Spava: Accelerating Long-Video Understanding via Sequence-Parallelism-aware Approximate Attention](https://arxiv.org/abs/2601.21444v1) (Huang et al., 2026). Focus on Section 3 (method) for the anchor/passing block formulation, Algorithm 1 for the pseudocode, and Table 1-2 for speedup/accuracy tradeoffs across model sizes.