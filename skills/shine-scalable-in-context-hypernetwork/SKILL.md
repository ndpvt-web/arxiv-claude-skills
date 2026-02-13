---
name: "shine-scalable-in-context-hypernetwork"
description: "Guide Claude to apply SHINE's single-pass context-to-LoRA hypernetwork technique for converting document knowledge into LLM weight adapters without fine-tuning. Use when: 'convert this document into model weights', 'encode context as LoRA adapters', 'build a hypernetwork for context compression', 'single-pass knowledge injection without fine-tuning', 'implement SHINE-style in-context hypernetwork', 'map context to LoRA parameters'."
---

# SHINE: Scalable In-Context Hypernetwork for Single-Pass Context-to-LoRA Mapping

This skill enables Claude to guide implementation of SHINE (Scalable Hyper In-context NEtwork), a hypernetwork architecture that converts arbitrary document context into LoRA adapter weights for an LLM in a single forward pass—no iterative fine-tuning required. SHINE reuses the frozen LLM's own layers to compress context into memory states, then transforms those states into full LoRA parameter matrices via a sparse-attention M2P Transformer. The result is a system that "bakes" contextual knowledge directly into model parameters, enabling question answering without the context present at inference time.

## When to Use This Skill

- When a user wants to encode document or passage knowledge into LLM weights instead of using RAG or in-context learning
- When building a system that needs to serve thousands of context-specific LoRA adapters with sub-second generation time
- When implementing a hypernetwork that generates neural network parameters (LoRA matrices) from input data
- When the user asks about converting in-context knowledge to in-parameter knowledge
- When designing a context compression pipeline that goes beyond simple embedding—actually producing executable weight deltas
- When comparing or choosing between RAG, fine-tuning, and parameter-generation approaches for knowledge injection
- When implementing sparse attention patterns (alternating row/column) for structured tensor processing

## Key Technique

**The Core Insight:** Instead of building a separate encoder to generate LoRA weights, SHINE hijacks the frozen LLM itself as the context encoder. Learnable "memory embeddings" are appended to the input context tokens. As the LLM processes this concatenated sequence, the hidden states at the memory positions across all layers become a structured representation of the context. This avoids training a separate large encoder and naturally captures features from low-level syntax (early layers) to high-level reasoning (deep layers).

**The M2P Transformer:** The extracted memory states form a 2D grid of shape `[L layers x M memory tokens x H hidden dim]`. A 4-layer M2P (Memory-to-Parameter) Transformer processes this grid using alternating sparse attention: odd layers attend across the layer dimension (column attention, letting memory tokens share information across depths), and even layers attend across the memory token dimension (row attention, letting tokens within each layer interact). This reduces complexity from O((LM)^2) to O(LM^2 + ML^2)—roughly 90% FLOPs savings—while enabling shallow layers to receive information from deep layers, mimicking backpropagation's cross-layer dependencies. The final output is flattened and sliced into LoRA A and B matrices for each target layer.

**Training Pipeline:** SHINE trains in three stages: (1) self-supervised pretraining with reconstruction (regenerate full context from LoRA alone) and completion (recover randomly truncated context) losses; (2) instruction fine-tuning on multi-QA datasets (15 QA pairs per document); (3) optional single-QA fine-tuning on diverse benchmarks. The generated LoRA adapters achieve F1 scores competitive with test-time training methods while requiring only a single forward pass (~0.3s) versus minutes of iterative optimization.

## Step-by-Step Workflow

1. **Select and freeze the backbone LLM.** Use a decoder-only model (SHINE's reference uses Qwen3-8B). Freeze all base parameters—only the hypernetwork components (Meta LoRA, memory embeddings, M2P Transformer) are trainable.

2. **Add Meta LoRA to the frozen LLM.** Insert small trainable LoRA adapters (rank 128 in the default config) into the LLM's attention layers. These are NOT the output adapters—they augment the LLM's ability to compress context into memory states during the hypernetwork forward pass.

3. **Initialize learnable memory embeddings.** Create M learnable embedding vectors (M = ceil(r * D / H), where r = generated LoRA rank, D = sum of per-layer input/output dimensions, H = hidden dim). For Qwen3-8B with rank 8, this yields M = 148 tokens. These are appended after the context tokens.

4. **Run the context through the augmented LLM.** Forward pass: input = [context_tokens; memory_embeddings]. Extract the hidden states at the M memory positions from every layer, producing a tensor of shape [L, M, H]. Add learnable positional encodings for layer index and memory token index.

5. **Process through the M2P Transformer.** Pass the [L, M, H] tensor through 4 Transformer layers with alternating attention: layers 1,3 use column attention (across L dimension for each of the M tokens), layers 2,4 use row attention (across M dimension for each of the L layers). Each attention output is followed by a 2-layer MLP.

6. **Slice the output into LoRA matrices.** For each target LLM layer i, take M2P output[i, :, :], flatten to a vector of length M*H, and sequentially carve out LoRA-A (shape I x r) and LoRA-B (shape r x O) matrices for each adapted weight (q_proj, k_proj, v_proj, etc.).

7. **Pretrain with reconstruction + completion losses.** Train on large text corpora (6B tokens). For each sample: generate LoRA from context, apply to LLM, then (a) reconstruct the full context given only a `<RECON>` token, and (b) complete a 10-30% truncated version given a `<COMP>` token. Loss = 0.5 * L_recon + 0.5 * L_comp. Learning rate 5e-5, 1 epoch.

8. **Instruction fine-tune on QA tasks.** Generate 15 synthetic QA pairs per document using a teacher model. Train with loss = -log P(answer | question; generated_LoRA, frozen_LLM). The context is NOT provided at QA time—only the generated LoRA encodes it. Train for 2 epochs at learning rate 3e-5.

9. **Inference: single-pass knowledge injection.** At serving time, pass a new document through the hypernetwork (one forward pass, ~0.3s). Cache the resulting LoRA. For all subsequent queries about that document, apply the cached LoRA to the frozen LLM and run inference—no context window consumed, no retrieval needed.

10. **Evaluate with context-hidden protocol.** Verify quality by testing QA with the original document fully hidden from the LLM. Only the generated LoRA mediates access to the knowledge. Compare F1 scores against in-context learning and SFT baselines.

## Concrete Examples

**Example 1: Implementing the Memory Extraction Module**

User: "I need to implement the memory state extraction part of SHINE for a Qwen3-8B backbone."

Approach:
1. Hook into each Transformer layer's output to capture hidden states
2. Slice out only the memory token positions from each layer
3. Stack into the [L, M, H] tensor with positional encodings

Output:
```python
import torch
import torch.nn as nn

class MemoryExtractor(nn.Module):
    def __init__(self, num_layers, num_memory_tokens, hidden_dim):
        super().__init__()
        self.num_memory_tokens = num_memory_tokens
        # Learnable memory embeddings appended to input
        self.memory_embeddings = nn.Parameter(
            torch.randn(1, num_memory_tokens, hidden_dim) * 0.02
        )
        # Positional encodings for the memory grid
        self.layer_pos = nn.Parameter(torch.randn(num_layers, 1, hidden_dim) * 0.02)
        self.token_pos = nn.Parameter(torch.randn(1, num_memory_tokens, hidden_dim) * 0.02)

    def extract(self, all_layer_hidden_states, context_length):
        """
        all_layer_hidden_states: list of [batch, seq_len, H] from each layer
        context_length: number of context tokens (memory starts after this)
        Returns: [batch, L, M, H] tensor
        """
        memory_states = []
        for layer_hidden in all_layer_hidden_states:
            # Slice out memory positions: [batch, M, H]
            mem = layer_hidden[:, context_length:context_length + self.num_memory_tokens, :]
            memory_states.append(mem)
        # Stack: [batch, L, M, H]
        M = torch.stack(memory_states, dim=1)
        # Add positional encodings
        M = M + self.layer_pos.unsqueeze(0) + self.token_pos.unsqueeze(0)
        return M
```

**Example 2: Building the Sparse M2P Transformer**

User: "How do I implement the alternating row/column sparse attention for the M2P Transformer?"

Approach:
1. Define a single attention layer that can operate along either the L or M dimension
2. Alternate between column attention (across layers) and row attention (across tokens)
3. Follow each with a 2-layer MLP

Output:
```python
class M2PTransformer(nn.Module):
    def __init__(self, hidden_dim, num_heads, num_m2p_layers=4, mlp_ratio=4):
        super().__init__()
        self.layers = nn.ModuleList()
        for i in range(num_m2p_layers):
            self.layers.append(M2PLayer(
                hidden_dim, num_heads, mlp_ratio,
                attention_mode="column" if i % 2 == 0 else "row"
            ))

    def forward(self, M):
        """M: [batch, L, M_tokens, H]"""
        Z = M
        for layer in self.layers:
            Z = layer(Z)
        return Z

class M2PLayer(nn.Module):
    def __init__(self, hidden_dim, num_heads, mlp_ratio, attention_mode):
        super().__init__()
        self.attention_mode = attention_mode
        self.attn = nn.MultiheadAttention(hidden_dim, num_heads, batch_first=True)
        self.norm1 = nn.LayerNorm(hidden_dim)
        self.norm2 = nn.LayerNorm(hidden_dim)
        self.mlp = nn.Sequential(
            nn.Linear(hidden_dim, hidden_dim * mlp_ratio),
            nn.GELU(),
            nn.Linear(hidden_dim * mlp_ratio, hidden_dim),
        )

    def forward(self, Z):
        """Z: [batch, L, M, H]"""
        B, L, M, H = Z.shape
        if self.attention_mode == "column":
            # Attend across L for each of M tokens: reshape to [B*M, L, H]
            Z_r = Z.permute(0, 2, 1, 3).reshape(B * M, L, H)
        else:
            # Attend across M for each of L layers: reshape to [B*L, M, H]
            Z_r = Z.reshape(B * L, M, H)

        # Self-attention + residual
        attn_out, _ = self.attn(self.norm1(Z_r), self.norm1(Z_r), self.norm1(Z_r))
        Z_r = Z_r + attn_out
        # MLP + residual
        Z_r = Z_r + self.mlp(self.norm2(Z_r))

        if self.attention_mode == "column":
            Z = Z_r.reshape(B, M, L, H).permute(0, 2, 1, 3)
        else:
            Z = Z_r.reshape(B, L, M, H)
        return Z
```

**Example 3: Choosing Between SHINE, RAG, and Fine-Tuning**

User: "I have 10,000 product manuals and need to answer customer questions about specific products. Should I use SHINE, RAG, or fine-tuning?"

Approach:
1. Analyze the tradeoffs for this specific use case
2. Consider amortized compute, latency, accuracy, and memory

Output:
```
Analysis for 10K product manuals with per-product QA:

SHINE approach:
- Pre-generate one LoRA per manual (~0.3s each, ~50 min total one-time cost)
- At query time: load the product's cached LoRA, run inference with no context
  window consumed. Generation speed matches vanilla LLM.
- Pro: No retrieval latency, no context window limits, constant inference cost
- Con: ~80% of in-context F1 on single-hop QA; weaker on multi-hop reasoning

RAG approach:
- Index all manuals in a vector store
- At query time: retrieve top-k chunks, insert into context window
- Pro: Highest accuracy for factual recall (~86 F1 on SQuAD-style tasks)
- Con: Retrieval latency, context window consumed, cost scales with chunk count

Fine-tuning (SFT):
- Train separate adapter per manual (minutes each, days total for 10K)
- Pro: Can deeply specialize per document
- Con: 100x slower than SHINE per document, poor amortization

Recommendation: Use RAG as primary for accuracy-critical queries. Use SHINE
as a complement for high-throughput scenarios where you need to answer many
questions per product without retrieval overhead—e.g., batch processing or
latency-sensitive customer chat where the context window is already occupied
by conversation history.
```

## Best Practices

- **Do:** Size the memory token count M using the formula M = ceil(r * D / H) to ensure the memory tensor has at least as many elements as the total LoRA parameters being generated. Undersizing M creates an information bottleneck that caps quality.
- **Do:** Use alternating column/row sparse attention in the M2P Transformer rather than full attention over the flattened [L*M] sequence. The 90% FLOPs reduction is critical for scaling to larger models.
- **Do:** Pretrain with both reconstruction AND completion losses (equal weight). Reconstruction alone leads to rote memorization; completion forces generalization to unseen queries about the context.
- **Do:** Cache generated LoRAs for repeated use. The amortized cost of SHINE is ~0.3s per document; the payoff comes when you answer many queries per document without regenerating.
- **Avoid:** Applying SHINE to contexts longer than ~1,150 tokens without architectural changes. The default memory extraction assumes the context fits within this window. For longer documents, chunk and generate separate LoRAs or extend the context handling.
- **Avoid:** Expecting SHINE to match in-context learning accuracy on single-hop factoid QA (63.6 vs 86.8 F1 on SQuAD). SHINE trades some accuracy for eliminating context window usage and enabling constant-cost inference. It excels when amortization matters.

## Error Handling

- **Memory dimension mismatch:** If the M2P output vector length doesn't evenly divide into the expected LoRA A and B matrix shapes, verify that M was computed correctly from the formula. Off-by-one errors in ceil() are common.
- **Degraded multi-turn performance:** SHINE's MQA results show declining F1 over conversation turns (55.6 overall vs stronger early turns). If multi-turn QA quality degrades, consider supplementing with in-context history for later turns while using SHINE-generated LoRA for the document knowledge.
- **Pretraining loss spikes around 100-token contexts:** The authors observed outlier perplexity for very short contexts due to dataset noise. Filter or pad extremely short contexts during pretraining to avoid instability.
- **OOM during M2P processing:** The [L, M, H] tensor for a 36-layer model with 148 memory tokens at 4096 hidden dim is substantial. Use gradient checkpointing on the M2P Transformer layers during training.

## Limitations

- SHINE does not fully close the accuracy gap with in-context learning, particularly on single-hop extractive QA (SQuAD: 63.6 vs 86.8 F1). It is most compelling when amortized over many queries per document.
- Multi-turn conversation quality degrades as conversation length grows, because the model lacks long-context post-training specific to the LoRA-augmented setting.
- The default architecture handles contexts up to ~1,150 tokens. Longer documents require chunking or architectural modifications.
- Generated LoRA quality depends on the backbone LLM's representational capacity. Weaker backbones produce weaker memory states and therefore weaker adapters.
- The approach has been validated primarily on Qwen3-8B. Transfer to other architectures (Llama, Mistral) requires re-training the hypernetwork components.

## Reference

**Paper:** [SHINE: A Scalable In-Context Hypernetwork for Mapping Context to LoRA in a Single Pass](https://arxiv.org/abs/2602.06358v1) — Liu et al., 2026. Focus on Section 3 (architecture), Section 4 (training pipeline), and Tables 1-4 (benchmarks and scaling analysis).

**Code:** [github.com/Yewei-Liu/SHINE](https://github.com/Yewei-Liu/SHINE) — Reference implementation with Qwen3-8B backbone, pretrained checkpoints, and evaluation scripts.