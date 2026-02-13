---
name: "out-memory-barrier-highly"
description: "Configure and run OOMB for memory-efficient long-context LLM training with million-token sequences on limited GPUs. Triggers: 'train LLM with long context', 'reduce GPU memory for training', 'OOMB setup', 'million token training', 'chunk-wise training with KV offloading', 'long context fine-tuning single GPU'"
---

This skill enables Claude to help users set up, configure, and troubleshoot OOMB (Out Of the Memory Barrier), a training system that achieves near-constant GPU memory usage when training LLMs on extremely long contexts (up to 4M tokens on a single H200). OOMB combines chunk-recurrent training with activation recomputation, paged KV cache management, asynchronous CPU offloading, and page-level sparse attention to reduce memory overhead to approximately 10MB per additional 10K tokens.

## When to Use

- When the user wants to fine-tune or continue-train an LLM on sequences longer than 64K tokens and is running out of GPU memory
- When the user asks how to train a model like Qwen2.5-7B on million-token contexts without a multi-node cluster
- When the user needs to configure OOMB's chunk size, page size, CPU offloading, or sparse attention budget
- When the user is hitting OOM errors during long-context training and wants to switch from standard parallel training to chunk-wise training
- When the user wants to validate that OOMB's sparse attention gradients are accurate compared to full attention
- When the user asks about reducing activation memory or KV cache memory during LLM training

## Key Technique

**Chunk-Recurrent Training with O(1) Activation Memory.** Standard training stores all intermediate activations across the full sequence length, causing memory to scale linearly with context. OOMB breaks the sequence into fixed-size chunks (default 4096 tokens) and uses a two-stage algorithm: (1) a population stage that runs a no-grad forward pass over all chunks to fill the KV cache, and (2) a gradient stage that processes chunks in reverse order, recomputing activations on the fly for each chunk's forward-backward pass. Because only one chunk's activations exist in memory at any time, activation memory is O(1) regardless of sequence length.

**Paged KV Cache with CPU Offloading.** Once activations are constant, the KV cache becomes the memory bottleneck. OOMB manages it with a paged memory allocator (similar to vLLM's paged attention but for training) that eliminates fragmentation. Both the KV cache and its gradient buffers are stored in fixed-size pages. An asynchronous CPU offloading pipeline prefetches pages for upcoming layers while the current layer computes, hiding PCIe transfer latency behind GPU computation. The `cpu_offload` parameter controls the offload window depth (number of neighboring layers to prefetch).

**Page-Level Sparse Attention.** For sequences where even the offloaded KV cache is too large to process densely, OOMB implements top-k page retrieval: for each query chunk, it scores KV pages by query-key similarity and retrieves only the `page_budget` most relevant pages. This reduces both FLOPs and CPU-GPU data transfer. The sparse selection is recorded during the forward population stage and replayed identically during the backward gradient stage to ensure correct gradient computation.

## Step-by-Step Workflow

1. **Install OOMB and dependencies.** Clone the repository and install requirements:
   ```bash
   git clone https://github.com/wenhaoli-xmu/OOMB.git
   cd OOMB
   pip install -r requirements.txt
   ```
   Required: PyTorch, Triton 3.1.0, transformers 4.45.0, accelerate 1.10.1, datasets 2.18.0.

2. **Choose the training method based on your context length and GPU.**
   - `baseline-tp`: Standard tensor-parallel training. Use for sequences up to ~64K tokens.
   - `blockwise-tp`: Chunk-wise training with dense attention. Use for 64K-512K tokens when you can afford full attention but not full activation storage.
   - `blockwise-tp-sparse`: Chunk-wise training with sparse page retrieval. Use for 512K-4M+ tokens where dense KV attention exceeds GPU+CPU memory.

3. **Create a JSON configuration file** with the appropriate parameters. Start from this template for the sparse method:
   ```json
   {
     "method": "blockwise-tp-sparse",
     "block_size": 4096,
     "grad_ckpt": true,
     "page_size": 128,
     "cpu_offload": 2,
     "page_budget": 64
   }
   ```

4. **Tune `page_size` for your GPU architecture.**
   - H200 / H100 (large HBM): use `page_size: 128`
   - A100 / A800 (smaller HBM): use `page_size: 64`
   - Smaller pages reduce internal fragmentation but increase page table overhead; larger pages amortize management cost but waste memory on partial fills.

5. **Set `block_size` to control the activation-compute tradeoff.** The default 4096 works well for most models. Larger blocks (8192) reduce chunk boundary overhead but increase peak activation memory per chunk. Smaller blocks (2048) save activation memory but increase the number of recomputation passes.

6. **Configure CPU offloading depth.** Set `cpu_offload: 2` to enable asynchronous offloading with a prefetch window of 2 layers. Set to `null` to disable offloading (keeps all KV pages on GPU). Offloading is essential for contexts beyond ~256K tokens on 80GB GPUs.

7. **Set `page_budget` for sparse attention.** This controls how many KV pages are retrieved per query chunk. A budget of 64 pages with `page_size: 128` means each query attends to 64 * 128 = 8192 tokens per layer. Increase to 128-256 for tasks requiring high recall (e.g., retrieval-heavy benchmarks like NIAH). Decrease to 32 for maximum memory savings if approximate attention suffices.

8. **Launch training with torchrun** using tensor parallelism across available GPUs:
   ```bash
   torchrun \
     --rdzv-backend=c10d \
     --rdzv-endpoint=localhost:$((RANDOM % 101 + 20000)) \
     --nnodes 1 \
     --nproc_per_node 4 \
     train/train.py \
     --config your_config.json \
     --context_length 1048576
   ```

9. **Validate gradient accuracy** if using sparse attention. Run the accuracy test suite to compare sparse gradients against dense chunk-wise gradients:
   ```bash
   bash test_accuracy/test.sh
   python test_accuracy/compare.py \
     --baseline /path/to/blockwise-tp.pth \
     --ours /path/to/blockwise-tp-sparse-256.pth \
     --root-dir /path/to/root_dir
   ```
   Expect relative gradient error below 1% for page_budget >= 64.

10. **Monitor memory with the efficiency benchmark** to verify your configuration stays within GPU limits before launching a full training run:
    ```bash
    torchrun \
      --rdzv-backend=c10d \
      --rdzv-endpoint=localhost:$PORT \
      --nnodes 1 \
      --nproc_per_node 4 \
      test_efficiency/test.py \
      --context "[32768, 65536, 131072, 262144, 524288, 1048576]" \
      --config your_config.json
    ```

## Concrete Examples

**Example 1: Training Qwen2.5-7B on 1M-token context with a single 4-GPU node**

User: "I want to fine-tune Qwen2.5-7B on documents up to 1M tokens. I have 4x H200 GPUs. How do I set this up with OOMB?"

Approach:
1. Clone OOMB and install dependencies
2. Create config for blockwise-tp-sparse (1M tokens requires sparse attention on 4 GPUs):
   ```json
   {
     "method": "blockwise-tp-sparse",
     "block_size": 4096,
     "grad_ckpt": true,
     "page_size": 128,
     "cpu_offload": 2,
     "page_budget": 64
   }
   ```
3. Run the efficiency benchmark first to confirm memory fits:
   ```bash
   torchrun --rdzv-backend=c10d --rdzv-endpoint=localhost:20050 \
     --nnodes 1 --nproc_per_node 4 \
     test_efficiency/test.py \
     --context "[131072, 262144, 524288, 1048576]" \
     --config config_1m.json
   ```
4. Expected memory: ~15GB base + ~100MB for 1M tokens of KV overhead (10MB per 10K tokens)
5. If memory is comfortable, launch training with the same config pointing to your training script and dataset

**Example 2: Reducing memory on A100 GPUs for 256K context**

User: "I keep running OOM training LLaMA-7B on 256K context with 4x A100-80GB. Can OOMB help?"

Approach:
1. Switch from standard training to chunk-wise dense (256K doesn't strictly need sparse):
   ```json
   {
     "method": "blockwise-tp",
     "block_size": 4096,
     "grad_ckpt": true,
     "page_size": 64,
     "cpu_offload": 2
   }
   ```
2. Use `page_size: 64` for A100 (smaller HBM bandwidth favors smaller pages)
3. Enable CPU offloading to move KV pages for non-active layers to host RAM
4. Run efficiency test to verify memory usage at 256K fits within 80GB per GPU
5. If still tight at 256K, switch to `blockwise-tp-sparse` with `page_budget: 128` to cut KV transfer and computation

**Example 3: Validating sparse attention doesn't degrade training quality**

User: "I'm worried that sparse attention in OOMB will hurt my model's ability to learn long-range dependencies. How do I verify?"

Approach:
1. Run gradient accuracy comparison between dense and sparse:
   ```bash
   # Train one step with dense chunk-wise attention
   # Train one step with sparse chunk-wise attention (page_budget=64)
   bash test_accuracy/test.sh
   python test_accuracy/compare.py \
     --baseline blockwise-tp.pth \
     --ours blockwise-tp-sparse-64.pth \
     --root-dir ./test_accuracy
   ```
2. Check per-layer gradient cosine similarity (should be >0.99 for page_budget>=64)
3. If gradient error is too high, increase `page_budget` (try 128, 256) until error is acceptable
4. Trade-off: higher page_budget = more accurate gradients but more memory and transfer cost

## Best Practices

- **Do:** Always run the efficiency benchmark (`test_efficiency/test.py`) at your target context lengths before committing to a full training run. This catches OOM issues in minutes instead of hours.
- **Do:** Start with `blockwise-tp` (dense) and only switch to `blockwise-tp-sparse` when dense attention exceeds your memory budget. Dense attention gives exact gradients.
- **Do:** Set `grad_ckpt: true` in all configurations. Layer-wise gradient checkpointing is essential for the chunk-recurrent framework to achieve O(1) activation memory.
- **Do:** Use the largest `page_budget` your memory allows when using sparse attention. More pages = better gradient approximation = better training quality.
- **Avoid:** Setting `block_size` larger than 8192 unless you have verified activation memory fits. Each chunk's activations must fit entirely in GPU memory during its forward-backward pass.
- **Avoid:** Disabling `cpu_offload` for contexts beyond 128K tokens. Without offloading, the full KV cache for all layers must reside in GPU HBM simultaneously.
- **Avoid:** Using `baseline-tp` for contexts beyond 64K tokens. It stores full activations and will OOM on standard hardware.

## Error Handling

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| CUDA OOM during population stage | KV cache too large for GPU even with offloading | Enable `cpu_offload: 2` or reduce `page_budget` |
| CUDA OOM during gradient stage | `block_size` too large for model's per-chunk activations | Reduce `block_size` from 4096 to 2048 |
| Slow training throughput | CPU offloading bandwidth saturated | Verify PCIe gen4/gen5 link; reduce `page_size` to reduce transfer granularity |
| NaN gradients | Sparse page selection missing critical context | Increase `page_budget`; verify with gradient comparison test |
| `triton` compilation errors | Triton version mismatch | Pin `triton==3.1.0` as specified in requirements |
| Host RAM exhaustion | Offloaded KV cache exceeds system memory | Add more RAM, reduce context length, or increase `page_budget` (fewer pages offloaded) |

## Limitations

- **Model support**: OOMB currently targets Qwen2.5 and similar transformer architectures. Adapting to models with non-standard attention (e.g., mixture-of-experts with per-expert KV) requires modifying the cache manager.
- **Sparse attention is approximate**: Page-level top-k retrieval can miss long-range dependencies that span low-scoring pages. Tasks like multi-hop reasoning across distant passages may degrade.
- **CPU memory requirement**: Offloading shifts the burden from GPU to host RAM. Training 4M-token contexts requires substantial system memory (hundreds of GB).
- **Single-node focus**: OOMB uses tensor parallelism within a node. It does not implement context parallelism across nodes, so maximum context length is bounded by single-node GPU + CPU memory.
- **Training only**: This system targets the training/fine-tuning path. For inference on long contexts, use systems like vLLM or SGLang which have complementary paged attention for serving.
- **Fixed chunk boundaries**: The chunk-recurrent framework processes fixed-size blocks. It does not adaptively size chunks based on content, which could leave efficiency on the table for variable-length inputs.

## Reference

**Paper**: [Out of the Memory Barrier: A Highly Memory Efficient Training System for LLMs with Million-Token Contexts](https://arxiv.org/abs/2602.02108v2) (Li et al., 2026). Focus on Section 3 (system design) for the chunk-recurrent algorithm, Section 4 (paged memory + offloading), and Section 5 (sparse attention). The key result: 10MB overhead per 10K additional tokens for Qwen2.5-7B.

**Code**: [github.com/wenhaoli-xmu/OOMB](https://github.com/wenhaoli-xmu/OOMB) — see `chunkoptim/` for core implementation, `test_efficiency/` for benchmarking, and `test_accuracy/` for gradient validation.