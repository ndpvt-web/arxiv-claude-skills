---
name: "zipmoe-on-device-moe-serving"
description: "Deploy Mixture-of-Experts LLMs on edge devices using lossless BF16 compression with bit-field decomposition and cache-affinity scheduling. Triggers: 'optimize MoE for edge', 'deploy MoE on Jetson', 'compress expert weights losslessly', 'edge device MoE inference', 'on-device expert caching', 'MoE offloading to SSD'"
---

# ZipMoE: Lossless Compression and Cache-Affinity Scheduling for On-Device MoE Serving

This skill enables Claude to architect and implement efficient Mixture-of-Experts (MoE) inference systems for edge devices (e.g., NVIDIA Jetson, mobile SoCs) by applying ZipMoE's two core techniques: **bit-field decomposition** of BF16 expert weights into separately compressible exponent and sign-mantissa chunks, and **cache-affinity scheduling** that overlaps decompression with GPU compute to shift the bottleneck from I/O to computation. The approach preserves full model accuracy (no quantization) while achieving up to 72% latency reduction and 6.7x throughput improvement over state-of-the-art offloading systems.

## When to Use

- When the user wants to deploy a large MoE model (DeepSeek, Qwen-MoE, Switch Transformers) on an edge device with limited DRAM
- When the user needs lossless compression of BF16/FP16 model weights for SSD offloading without accuracy degradation
- When the user asks to build an expert caching and scheduling system for MoE inference
- When optimizing NVMe-to-GPU data pipelines on unified memory architecture (UMA) devices like Jetson Orin
- When the user wants to overlap CPU decompression with GPU execution in an inference pipeline
- When implementing expert popularity tracking to decide which experts to cache in memory vs. offload to flash

## Key Technique

**Bit-field decomposition** exploits a structural property of BF16 floating-point weights: the 8-bit exponent field has far lower entropy than the 9-bit sign+mantissa field. ZipMoE splits each expert's weight tensor into E-chunks (exponent bytes) and SM-chunks (sign-mantissa bytes), then compresses them independently with LZ4HC or ZSTD. Exponent chunks achieve ~66% compression ratio (near Shannon entropy bounds), while sign-mantissa chunks are stored uncompressed. This means roughly half the data shrinks dramatically, and the other half streams directly — a key insight that creates two distinct I/O task types.

**Cache-affinity scheduling** exploits this asymmetry. The scheduler classifies each expert load into Type-I tasks (SM-chunks must be read from SSD — I/O-bound) and Type-II tasks (SM-chunks already cached in DRAM — compute-bound, only E-chunk decompression needed). It constructs execution blocks that interleave Type-I and Type-II tasks so that CPU decompression of E-chunks overlaps with SSD reads of SM-chunks, and GPU compute overlaps with both. The algorithm provides a formal approximation guarantee: makespan <= (3 - 1/L) * OPT, where L is the number of decompression threads.

**The critical insight** is that multi-threaded CPU decompression on modern edge SoCs (6-12 cores) can be *faster* than raw SSD reads, converting what was an I/O-bound pipeline into a compute-bound one. This is the opposite of the server assumption where network/SSD bandwidth is abundant — on edge devices, underutilized CPU cores are the free resource.

## Step-by-Step Workflow

1. **Profile the target device's memory hierarchy.** Measure available DRAM, NVMe SSD read bandwidth, CPU core count, and GPU memory. On Jetson Orin, DRAM is shared (UMA) — account for GPU memory reservations. Record the SSD sequential read speed (e.g., 3.5 GB/s for Samsung 970 EVO).

2. **Decompose expert weight tensors via bit-field splitting.** For each expert's BF16 weight matrix, extract the 8-bit exponent field into an E-chunk and the 9-bit sign+mantissa into an SM-chunk. Serialize these as separate binary blobs with byte alignment.

3. **Compress E-chunks with LZ4HC or ZSTD.** Apply lossless compression to each E-chunk independently. Measure per-chunk compression ratio and decompression throughput. E-chunks should compress to ~34% of original size. Leave SM-chunks uncompressed — they are high-entropy and won't compress meaningfully.

4. **Build the hierarchical cache pool.** Implement four cache tiers: full tensors in DRAM (F-cache), compressed tensors (C-cache), SM-chunks only (S-cache), and E-chunks only (E-cache). Assign cache budgets based on available DRAM after reserving space for model activations and the attention/routing layers.

5. **Implement rank-based expert popularity tracking.** Aggregate expert activation counts from the gating network across inference batches. Model popularity as a stationary rank distribution (not tied to specific expert IDs). Use dynamic programming to compute marginal cache-hit probabilities: `DP[i][j][p] = DP[i-1][j][p] * (1 - q_{u_p+i}) + DP[i-1][j-1][p] * q_{u_p+i}`, where q is the activation probability for rank position.

6. **Classify expert load tasks on each forward pass.** When the gating network selects experts for a batch, classify each as Type-I (SM-chunk not in DRAM, requires SSD read) or Type-II (SM-chunk cached, only needs E-chunk decompression). Sort tasks by GPU execution time.

7. **Construct the cache-affinity execution schedule.** Group tasks into blocks that maximize overlap: pair Type-I tasks (whose SSD reads can run concurrently with CPU decompression) with Type-II tasks (whose decompression fills compute gaps). The scheduler iteratively inserts tasks to minimize idle periods on both the I/O bus and decompression threads.

8. **Execute the pipelined DAG.** For each scheduled block: (a) issue NVMe reads for SM-chunks, (b) simultaneously decompress E-chunks on L CPU worker threads, (c) reconstruct full tensors via a memory-coalesced CUDA kernel that recombines E and SM components, (d) run the expert's FFN on GPU.

9. **Update cache state after inference.** Promote frequently-hit experts to higher cache tiers (SM-chunks stay resident longer). Evict least-popular experts from S-cache and E-cache based on the rank distribution model. Recompute cache budgets if batch patterns shift.

10. **Benchmark and tune.** Measure Time-Per-Output-Token (TPOT), Time-To-First-Token (TTFT), and throughput. Adjust the number of decompression threads L (typically 3-8), cache tier budgets, and compression algorithm (LZ4HC for speed, ZSTD for ratio) based on the device's CPU-to-SSD bandwidth ratio.

## Concrete Examples

**Example 1: Deploying DeepSeek-V2-Lite on Jetson AGX Orin 64GB**

User: "I need to run DeepSeek-V2-Lite MoE inference on a Jetson Orin 64GB with an NVMe SSD. The model doesn't fit in memory. How do I set this up without losing accuracy?"

Approach:
1. Profile the Orin: 64GB shared DRAM, 12 ARM cores, NVMe at ~3.5 GB/s
2. Load the non-expert layers (attention, routing, embeddings) into DRAM permanently — these are small
3. Split each of the 64 routed experts' BF16 weight tensors into E-chunks and SM-chunks
4. Compress E-chunks with LZ4HC (fast decompression matches Orin's CPU profile)
5. Allocate ~40GB for expert cache tiers, ~20GB for activations + non-expert layers + OS
6. Populate S-cache with the top-16 most popular experts' SM-chunks
7. On each token, classify the top-6 activated experts, schedule with cache-affinity algorithm
8. Use 4 decompression threads to overlap E-chunk decompression with SM-chunk SSD reads

Output:
```
Expert storage layout (per expert, ~150MB BF16):
  E-chunk: 75MB -> 25MB compressed (LZ4HC, 66% reduction)
  SM-chunk: 75MB (stored raw on NVMe)

Cache allocation (64GB total):
  Non-expert layers:  8GB (resident)
  Activations/KV:    12GB (reserved)
  S-cache (SM):      32GB (~21 experts' SM-chunks resident)
  E-cache:            8GB (compressed E-chunks for ~50 experts)
  OS/overhead:        4GB

Expected per-token pipeline (top-6 experts activated):
  ~2 Type-II tasks (SM cached) -> CPU decompression only
  ~4 Type-I tasks (SM from SSD) -> overlapped SSD read + decompression
  Estimated TPOT: ~45ms (vs ~160ms with naive offloading)
```

**Example 2: Implementing the bit-field decomposition in Python/C++**

User: "Show me how to split BF16 tensors into exponent and sign-mantissa chunks for lossless compression."

Approach:
1. Interpret BF16 values as raw uint16 bytes
2. Extract exponent bits (bits 7-14) into a contiguous byte array
3. Extract sign+mantissa bits (bit 15 + bits 0-6) into a separate byte array
4. Compress the exponent array, leave sign-mantissa raw

Output:
```python
import numpy as np
import lz4.frame

def decompose_bf16_tensor(tensor_bf16: np.ndarray):
    """Split BF16 tensor into compressible E-chunks and raw SM-chunks."""
    raw = tensor_bf16.view(np.uint16)

    # Exponent: bits [14:7] — 8 bits, low entropy
    exponent_bytes = ((raw >> 7) & 0xFF).astype(np.uint8)

    # Sign (bit 15) + Mantissa (bits [6:0]) — 8 bits packed
    sign_bit = (raw >> 15).astype(np.uint8)
    mantissa = (raw & 0x7F).astype(np.uint8)
    sign_mantissa_bytes = (sign_bit << 7) | mantissa

    return exponent_bytes.tobytes(), sign_mantissa_bytes.tobytes()

def compress_e_chunk(e_chunk_bytes: bytes) -> bytes:
    """Losslessly compress exponent chunk with LZ4HC."""
    return lz4.frame.compress(
        e_chunk_bytes,
        compression_level=lz4.frame.COMPRESSIONLEVEL_MAX
    )

def reconstruct_bf16(e_bytes: bytes, sm_bytes: bytes, shape: tuple):
    """Reconstruct original BF16 tensor from E and SM chunks."""
    e = np.frombuffer(e_bytes, dtype=np.uint8)
    sm = np.frombuffer(sm_bytes, dtype=np.uint8)
    sign = (sm >> 7).astype(np.uint16)
    mantissa = (sm & 0x7F).astype(np.uint16)
    exponent = e.astype(np.uint16)
    raw = (sign << 15) | (exponent << 7) | mantissa
    return raw.view(np.dtype('float16')).reshape(shape)  # reinterpret as BF16
```

**Example 3: Cache-affinity scheduling logic**

User: "How do I implement the scheduling algorithm that overlaps SSD reads with decompression?"

Approach:
1. Classify activated experts into Type-I (SM-chunk miss) and Type-II (SM-chunk hit)
2. Sort both lists by estimated GPU execution time (descending)
3. Build execution blocks that pair I/O-bound and compute-bound tasks

Output:
```python
from dataclasses import dataclass
from typing import List

@dataclass
class ExpertTask:
    expert_id: int
    type: int          # 1 = SM cache miss (I/O-bound), 2 = SM cache hit (compute-bound)
    gpu_time_ms: float
    e_decompress_ms: float
    sm_read_ms: float  # 0 if Type-II

def schedule_cache_affinity(tasks: List[ExpertTask], num_threads: int):
    """Cache-affinity block scheduling with (3 - 1/L) approximation guarantee."""
    type1 = sorted([t for t in tasks if t.type == 1], key=lambda t: -t.gpu_time_ms)
    type2 = sorted([t for t in tasks if t.type == 2], key=lambda t: -t.gpu_time_ms)

    blocks = []
    while type1 or type2:
        block = []
        block_io_time = 0
        block_compute_time = 0

        # Start with a Type-I task to initiate SSD read
        if type1:
            t = type1.pop(0)
            block.append(t)
            block_io_time = t.sm_read_ms
            block_compute_time = t.e_decompress_ms / num_threads + t.gpu_time_ms

        # Fill remaining block time with Type-II tasks to overlap with I/O
        while type2 and block_compute_time < block_io_time:
            t = type2.pop(0)
            block.append(t)
            block_compute_time += t.e_decompress_ms / num_threads + t.gpu_time_ms

        # If no Type-I tasks left, drain Type-II
        if not type1 and type2:
            block.extend(type2)
            type2 = []

        blocks.append(block)

    return blocks
```

## Best Practices

- **Do:** Profile your specific SSD's sequential read bandwidth before designing cache budgets. The CPU-decompression-faster-than-SSD-read crossover point depends entirely on your hardware.
- **Do:** Use LZ4HC for latency-sensitive inference (fast decompression) and ZSTD for storage-constrained deployments (better compression ratio, slower decompression).
- **Do:** Keep non-expert layers (attention, gating, embeddings) permanently resident in DRAM. They are accessed every token and are relatively small.
- **Do:** Allocate at least 3 CPU threads for decompression to ensure the formal approximation bound holds. More threads (up to 8) improve overlap but check for diminishing returns due to memory bandwidth contention.
- **Avoid:** Compressing SM-chunks — they are high-entropy (sign+mantissa bits are pseudo-random) and compression yields negligible savings with wasted CPU time.
- **Avoid:** Applying this technique to already-quantized models (INT4/INT8). Bit-field decomposition exploits the structured entropy of BF16/FP16 exponent fields, which quantized formats lack.

## Error Handling

- **Decompression integrity:** Always verify reconstructed tensors match originals via checksum (CRC32 embedded in LZ4 frames). A single bit flip in compressed storage corrupts the expert entirely.
- **Cache budget overflow:** If expert activations spike beyond the modeled rank distribution, the system may thrash between SSD reads. Implement a fallback that temporarily reduces batch size or top-k expert count.
- **Thread contention:** If decompression threads and GPU share memory bandwidth (UMA), monitor for >10% throughput degradation. If observed, reduce decompression thread count or pin threads to efficiency cores.
- **SSD thermal throttling:** Sustained reads on mobile NVMe can trigger thermal throttling, dropping bandwidth by 50%+. Monitor SSD temperature and adjust cache aggressiveness (cache more SM-chunks) when throttling is detected.

## Limitations

- **BF16/FP16 only.** Bit-field decomposition relies on the IEEE 754 floating-point structure. It does not apply to INT8, INT4, or custom quantization formats. If the model is already quantized, this technique provides no benefit.
- **UMA assumption.** The pipeline design assumes CPU and GPU share a unified memory space (as on Jetson, Apple Silicon). On discrete-GPU systems, the additional PCIe transfer step changes the I/O model significantly.
- **Stationary workload assumption.** The rank-based popularity model assumes expert activation distributions are roughly stationary. Highly adversarial or distribution-shifted inputs can cause cache thrashing.
- **Single-request latency focus.** The scheduling algorithm optimizes per-batch makespan. For multi-tenant serving with concurrent requests, additional coordination is needed.
- **Not a replacement for quantization.** When some accuracy loss is acceptable, combining quantization with this approach may yield better results than either alone — but the bit-field decomposition ratios will differ for non-BF16 formats.

## Reference

[ZipMoE: Efficient On-Device MoE Serving via Lossless Compression and Cache-Affinity Scheduling](https://arxiv.org/abs/2601.21198v1) — Focus on Section 3 (bit-field decomposition and compression analysis), Algorithm 1 (cache-affinity scheduling), and Theorem 3.1 (approximation guarantee proof).