---
name: "comet-collaborative-memory-transformer"
description: "Design and implement dual-memory chunk-based architectures for efficient long-context LLM processing. Use when asked about: 'handle long context efficiently', 'reduce KV cache memory', 'process million-token sequences', 'add memory module to transformer', 'chunk-based context processing', 'linear-time long sequence modeling'."
---

# CoMeT: Collaborative Memory Transformer for Efficient Long Context Modeling

This skill enables Claude to design, implement, and advise on **dual-memory chunk-based architectures** that give Transformers constant memory usage and linear time complexity for arbitrarily long sequences. The core technique from the CoMeT paper replaces quadratic full-attention with a plug-in memory module that splits context management into a **temporary FIFO memory** (recent detail) and a **gated global memory** (long-range dependencies), both acting as dynamic soft prompts prepended to each chunk. This approach lets a model fine-tuned on 32k tokens extrapolate to 1M+ tokens at inference.

## When to Use

- When the user needs to process sequences far longer than the model's training context window (e.g., 128k+ tokens)
- When building or modifying a Transformer architecture to have constant memory and linear time complexity
- When designing a memory-augmented LLM that must both recall distant facts and track recent events
- When implementing chunk-based inference pipelines that stream through documents, agent traces, or logs
- When the user wants to add an efficient context-extension module to an existing pre-trained model without full retraining
- When optimizing KV cache memory to avoid OOM on long-context inference
- When designing pipeline parallelism for fine-tuning on extremely long contexts (64k-128k tokens)

## Key Technique

CoMeT processes input in fixed-size **chunks** (typically 2,048 tokens). At each Transformer layer, two memory banks are prepended to the chunk's hidden states before self-attention:

1. **Temporary Memory (FIFO Queue):** Compressed representations of the most recent chunks, maintained as a fixed-length queue. When a new chunk is processed, its compression tokens are normalized (RMSNorm), transformed via a Residual Low-Rank Adapter (RLA), and enqueued — the oldest entry is dropped. This provides high-fidelity detail about recent context.

2. **Global Memory (Gated State):** A fixed-size persistent state updated via sigmoid gating: `S_{t+1} = g * S_t + (1-g) * S_new`. The gate `g` is learned per-position and protects salient historical information from being overwritten by less important new input. This captures long-range dependencies spanning hundreds of thousands of tokens.

The critical insight is that **gating is essential for length extrapolation**. Without it, long-sequence performance collapses entirely (the ablation is stark). The gated update acts as a selective memory filter — the model learns which information to preserve and which to replace. Meanwhile, the FIFO temporary memory ensures no loss of recent detail, which the global memory alone cannot guarantee. Together, the two memories act as a **dynamic soft prompt** — they are simply concatenated to the chunk hidden states and participate in standard causal attention, requiring zero changes to the base Transformer architecture.

For training efficiency, CoMeT introduces **layer-level pipeline parallelism**: as soon as a worker finishes computing layer `i` for a chunk, it sends the memory state to the next worker and immediately proceeds to layer `i+1`. This interleaving of computation and communication achieves 2.7x speedup over naive context parallelism, enabling 128k-token fine-tuning on 16x80GB GPUs.

## Step-by-Step Workflow

1. **Determine chunk size and memory budget.** Choose a chunk size matching the base model's efficient attention window (default: 2,048 tokens). Set global memory to 512 tokens and temporary memory to 2,048 tokens (one chunk's worth of FIFO slots). These defaults work well for Qwen3-4B-scale models.

2. **Add compression tokens to the chunk layout.** Insert one compression token every 8 context tokens within each chunk. These tokens participate in self-attention alongside the real tokens and capture a compressed summary of their local neighborhood. Also append `m` readout tokens at the end of the chunk to distill key information.

3. **Implement the Residual Low-Rank Adapter (RLA).** For each memory pathway, create a bottleneck projection: `RLA(X) = X + W_up(W_down * X)` with rank `r=8`. This transforms compression/readout outputs into memory-compatible representations with minimal added parameters.

4. **Build the temporary memory as a FIFO queue.** After each chunk is processed, take the output compression tokens, apply RMSNorm then RLA, and enqueue the result. Drop the oldest entry when the queue is full. The queue length determines how many recent chunks the model can recall in full detail.

5. **Build the global memory with gated updates.** Initialize a fixed-size state tensor `S` (512 tokens). After each chunk, compute a candidate update `S_new` from readout tokens via RLA. Compute a sigmoid gate `g = sigmoid(W_g * [S; S_new])`. Update: `S = g * S + (1-g) * S_new`. This selective gating is the mechanism that enables extrapolation beyond training length.

6. **Prepend memories as soft prompts.** At each Transformer layer `i` and chunk `t`, concatenate `[G_t^i, T_t^i, H_t^i, C_t^i, R_t^i]` where G is global memory, T is temporary memory, H is chunk hidden states, C is compression tokens, and R is readout tokens. Feed through standard causal self-attention. Only H is passed to the next layer as the chunk output — G and T are replaced by their updated versions.

7. **Freeze the base model; train only the memory module.** The plug-in design means only the RLA weights, gating parameters, compression token embeddings, and readout token embeddings are trainable. Fine-tune on mixed-length data (up to 32k tokens) for 3 epochs with learning rate 5e-5 and cosine decay.

8. **Implement layer-level pipeline parallelism for long-context training.** Partition layers across GPU workers. When worker `k` finishes layer `i`, immediately send the memory state to worker `k+1` and begin layer `i+1`. This overlaps communication with computation and is critical for training on 64k-128k sequences.

9. **Validate with passkey retrieval.** Test on synthetic passkey-in-a-haystack tasks at progressively longer contexts (32k, 64k, 128k, 256k, 512k, 1M). The model should maintain near-perfect retrieval at all lengths if gating and FIFO are working correctly.

10. **Deploy with streaming inference.** At inference time, process the input as a stream of chunks. Memory is constant regardless of total sequence length (~10GB for a 4B model). Decode latency remains flat (~22ms/token) even at 65k+ tokens where full attention would degrade to 104ms/token or OOM.

## Concrete Examples

**Example 1: Adding CoMeT to an existing Hugging Face model**

User: "I have a Qwen3-4B model and need it to handle 256k-token agent traces. How do I add CoMeT?"

Approach:
1. Define the memory module as a wrapper around each Transformer layer
2. Initialize global memory state as a `(num_layers, 512, hidden_dim)` tensor
3. Initialize FIFO queue as a `(num_layers, queue_len, hidden_dim)` tensor
4. Insert compression tokens every 8 positions in the tokenized input
5. Wrap the forward pass to chunk input, prepend memories, run attention, update memories

Output:
```python
import torch
import torch.nn as nn

class CoMeTMemoryModule(nn.Module):
    def __init__(self, hidden_dim, global_mem_size=512, temp_mem_chunks=1,
                 chunk_size=2048, compress_every=8, rla_rank=8):
        super().__init__()
        self.hidden_dim = hidden_dim
        self.global_mem_size = global_mem_size
        self.chunk_size = chunk_size
        self.compress_every = compress_every
        num_compress = chunk_size // compress_every

        # Compression and readout token embeddings
        self.compress_tokens = nn.Parameter(torch.randn(num_compress, hidden_dim) * 0.02)
        self.readout_tokens = nn.Parameter(torch.randn(global_mem_size, hidden_dim) * 0.02)

        # RLA for temporary memory path
        self.temp_down = nn.Linear(hidden_dim, rla_rank, bias=False)
        self.temp_up = nn.Linear(rla_rank, hidden_dim, bias=False)
        self.temp_norm = nn.RMSNorm(hidden_dim)

        # RLA for global memory path
        self.global_down = nn.Linear(hidden_dim, rla_rank, bias=False)
        self.global_up = nn.Linear(rla_rank, hidden_dim, bias=False)

        # Gating for global memory update
        self.gate_proj = nn.Linear(hidden_dim * 2, hidden_dim)

    def update_global_memory(self, state, readout_output):
        candidate = readout_output + self.global_up(self.global_down(readout_output))
        gate = torch.sigmoid(self.gate_proj(torch.cat([state, candidate], dim=-1)))
        return gate * state + (1 - gate) * candidate

    def update_temp_memory(self, queue, compress_output):
        normed = self.temp_norm(compress_output)
        entry = normed + self.temp_up(self.temp_down(normed))
        # FIFO: drop oldest, append new
        return torch.cat([queue[:, 1:], entry.unsqueeze(1)], dim=1)
```

**Example 2: Designing a chunk-based inference pipeline for document QA**

User: "I need to answer questions about a 500-page legal document. How do I structure the inference pipeline?"

Approach:
1. Tokenize the document and split into 2,048-token chunks
2. Initialize empty global memory and temporary memory queue
3. Stream chunks through the CoMeT-augmented model, updating memories after each
4. After all chunks are processed, append the question as the final chunk
5. Generate the answer conditioned on the accumulated memory state

Output:
```python
def comet_document_qa(model, tokenizer, document_text, question):
    tokens = tokenizer.encode(document_text)
    chunk_size = 2048
    chunks = [tokens[i:i+chunk_size] for i in range(0, len(tokens), chunk_size)]

    # Initialize memory state
    global_mem = torch.zeros(model.num_layers, 512, model.hidden_dim, device="cuda")
    temp_queue = torch.zeros(model.num_layers, 1, 2048, model.hidden_dim, device="cuda")

    # Stream through document chunks — memory is O(1)
    for chunk_tokens in chunks:
        chunk_input = prepare_chunk_with_compression(chunk_tokens, model)
        _, global_mem, temp_queue = model.forward_chunk(
            chunk_input, global_mem, temp_queue
        )

    # Process question as final chunk with accumulated context
    q_tokens = tokenizer.encode(question)
    q_input = prepare_chunk_with_compression(q_tokens, model)
    answer_logits, _, _ = model.forward_chunk(q_input, global_mem, temp_queue)
    return tokenizer.decode(answer_logits.argmax(dim=-1))
```

**Example 3: Configuring memory sizes for different use cases**

User: "How do I choose the right memory configuration for my 128k-token codebase analysis task?"

Approach:
1. Assess whether the task is retrieval-heavy (needs precise recall) or summary-heavy (needs gist)
2. For code analysis, recent context matters heavily — increase temporary memory
3. Global memory captures cross-file dependencies — keep at standard size

Output:
```
Configuration for 128k code analysis:
- Chunk size: 2,048 tokens (standard — matches attention window)
- Global memory: 512 tokens (captures cross-file function signatures, imports)
- Temporary memory: 4,096 tokens (2 chunks — preserves recent function bodies)
- Compression interval: every 8 tokens (standard density)
- Total memory overhead: ~4.6k tokens per layer (~2.3% of 200k budget)

Rationale: Code analysis requires precise recall of recent definitions
(temporary memory) and awareness of distant imports/interfaces (global memory).
The gated global memory will learn to preserve type signatures and API
contracts while letting implementation details decay.
```

## Best Practices

- **Do:** Always use gated updates for global memory. The ablation shows that without gating, extrapolation beyond training length fails completely. The sigmoid gate is non-negotiable.
- **Do:** Initialize compression and readout tokens with small random values (std=0.02). Large initializations destabilize early training since these tokens immediately participate in attention.
- **Do:** Train on mixed-length sequences (4k, 8k, 16k, 32k) rather than a single length. This teaches the memory module to handle variable context demands.
- **Do:** Freeze all base model parameters. Only train the memory module (RLA weights, gates, special tokens). This preserves the pre-trained model's capabilities while adding context extension.
- **Avoid:** Setting chunk size larger than the base model's efficient attention window. The point of CoMeT is to avoid quadratic attention — using 8k chunks on a model optimized for 2k defeats the purpose.
- **Avoid:** Skipping the temporary FIFO memory and relying solely on global memory. Global memory is lossy by design (gated compression). The FIFO queue provides exact recent context that the global memory cannot.

## Error Handling

- **OOM during training on long contexts:** Enable layer-level pipeline parallelism. Partition layers across GPUs so each worker holds only a subset of layers plus the memory state for the current chunk. This is essential beyond 32k training length.
- **Passkey retrieval fails at long lengths but works at short lengths:** The gating mechanism is likely not learning properly. Check that the gate projection receives the concatenation of both old state and candidate (not just the candidate). Verify sigmoid activation is applied.
- **Performance degrades on recent-context tasks:** The FIFO queue may be too small. Increase `temp_mem_chunks` to retain more recent chunks. Alternatively, decrease `compress_every` to increase compression token density.
- **Training instability with memory module:** Ensure RLA rank is low (r=8). Higher ranks add too many parameters and can cause the memory module to overfit or destabilize the frozen base model's representations.
- **Memory state diverges across layers:** Each layer should maintain its own independent global and temporary memory. Do not share memory states across layers — each layer captures different abstraction levels.

## Limitations

- CoMeT requires fine-tuning (even if minimal) — it cannot be applied zero-shot to an existing model. Budget 3 epochs on 32k-length data at minimum.
- The chunk boundary is a hard attention boundary. Information can only cross chunks via the memory banks. Tasks requiring precise token-level cross-chunk attention (e.g., exact string matching across distant positions) may lose fidelity compared to full attention.
- Memory sizes are fixed at initialization. There is no dynamic expansion — if the information density exceeds the memory capacity, older information is irreversibly compressed or dropped.
- The approach is validated on models up to 4B parameters. Scaling behavior to 70B+ models is not yet established.
- Layer-level pipeline parallelism requires homogeneous GPU setups and adds engineering complexity to the training infrastructure.

## Reference

**Paper:** [CoMeT: Collaborative Memory Transformer for Efficient Long Context Modeling](https://arxiv.org/abs/2602.01766v1) (Zhao et al., 2026)

Look for: Section 3 (Method) for the full architecture with equations, Section 4.2 for the layer-level pipeline parallelism design, Table 1 for SCROLLS benchmark comparisons, and Figure 4 for the ablation showing gating's critical role in length extrapolation.