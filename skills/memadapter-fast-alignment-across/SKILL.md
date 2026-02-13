---
name: "memadapter-fast-alignment-across"
description: "Unify heterogeneous agent memory systems (explicit graphs, parametric weights, latent KV-caches) via generative subgraph retrieval and lightweight contrastive alignment. Use when asked to: 'build a unified memory layer for my agent', 'align different memory backends', 'fuse retrieval across memory paradigms', 'add cross-paradigm memory retrieval to my agent framework', 'implement a plug-and-play memory adapter', 'reduce training cost for multi-memory agents'."
---

# MemAdapter: Fast Alignment across Agent Memory Paradigms via Generative Subgraph Retrieval

This skill enables Claude to design and implement unified memory retrieval systems for LLM-based agents that must operate across multiple memory paradigms simultaneously. Instead of building tightly-coupled retrieval logic for each memory backend (vector stores, LoRA adapters, KV-caches), you apply MemAdapter's two-stage approach: first train a generative subgraph retriever on a shared memory representation, then use a lightweight contrastive alignment module to adapt to any new memory paradigm in minutes rather than hours. The result is a plug-and-play memory layer that generalizes across explicit, parametric, and latent memory without per-paradigm retraining.

## When to Use

- When building an agent that needs to retrieve from multiple memory backends (e.g., a knowledge graph AND a fine-tuned LoRA adapter AND a streaming KV-cache) through a single retrieval interface
- When adding a new memory paradigm to an existing agent and you need fast adaptation without retraining the full retrieval pipeline
- When implementing memory fusion -- combining evidence from heterogeneous sources (documents, model parameters, conversation state) for multi-hop reasoning
- When designing a retrieval layer that must remain paradigm-agnostic so new memory backends can be plugged in later
- When optimizing training cost: you need cross-paradigm memory retrieval but cannot afford full retraining per paradigm (MemAdapter uses <5% of baseline compute)
- When the user asks to "unify my agent's memory" or "make retrieval work across different memory types"

## Key Technique

**The core insight** is that all agent memory paradigms -- explicit (stored graphs/buffers like A-Mem or Mem0), parametric (information baked into model weights via LoRA or similar), and latent (evolving internal states like KV-caches in StreamingLLM or CARE) -- can be projected into a shared embedding space of dimension D_s (typically 1536). Once in this unified space, a single generative retriever can produce structured evidence subgraphs regardless of which paradigm the underlying memory lives in. This avoids the combinatorial problem of building N separate retrieval systems for N memory types.

**Two-stage training** makes this practical. Stage 1 trains a generative subgraph retriever (a small LLM like Qwen2.5-1.5B with LoRA rank-16 on q_proj/v_proj) via knowledge distillation from a larger teacher model. The retriever learns to auto-regressively generate serialized evidence subgraphs conditioned on a query and memory context. Stage 2 freezes this retriever and trains only a lightweight feed-forward alignment module per new paradigm using InfoNCE contrastive loss -- the anchor paradigm (typically explicit graph memory) stays frozen, and the target paradigm's alignment network learns to map its memory states into the same space. This takes ~2,500 demonstrations and ~13 minutes on a single GPU.

**Zero-shot fusion** falls out naturally: memory states from different paradigms are projected through their respective alignment modules into the unified space, then downsampled via max-pooling to preserve complementary information. No paired cross-paradigm training data is needed.

## Step-by-Step Workflow

1. **Classify the memory paradigms in play.** Audit the agent's memory backends and label each as explicit (externally stored discrete units -- vector DBs, knowledge graphs, memory buffers), parametric (information in model weights -- LoRA adapters, fine-tuned modules), or latent (evolving internal state -- KV-caches, hidden-state pools). This classification determines which alignment modules you need.

2. **Define the unified memory space.** Create a shared embedding space (dimension 1536 is the paper's default). Every memory paradigm will project into this space. Implement the space as a simple vector store with cosine similarity as the distance metric.

3. **Build the anchor alignment module.** Choose one paradigm as the anchor -- explicit graph-based memory works best because its structure is most transparent. Implement a feed-forward network (2-3 layers, ReLU activations) that maps raw memory representations from this paradigm into the unified space. Train this module first and then freeze it.

4. **Train the generative subgraph retriever.** Fine-tune a small LLM (1.5B-3B parameters) with LoRA (rank 16, alpha 32, targeting attention projection layers) to auto-regressively generate serialized evidence subgraphs. Use knowledge distillation: minimize KL-divergence between a teacher model's token distribution and the student's, using ~40K training examples with controlled diversity in input length and reasoning complexity. Cap input context at 4096 tokens and output at 512 tokens.

5. **Build target alignment modules for each additional paradigm.** For each non-anchor paradigm, implement a lightweight feed-forward network with the same architecture as the anchor module. These are the only trainable components in Stage 2.

6. **Train alignment via contrastive learning.** For each target paradigm, collect ~2,500 demonstrations pairing queries with memory states from both the anchor and target paradigms. Use InfoNCE loss with temperature tau=0.07: positive pairs are anchor/target states for the same query; negatives are states from different queries in the batch. Freeze the anchor module and the retriever; only train the target alignment network. Learning rate 1e-4, batch size 32.

7. **Implement the retrieval interface.** At inference time: (a) receive a query, (b) project available memory states through their respective alignment modules into the unified space, (c) feed the projected states to the generative retriever, (d) decode the serialized evidence subgraph, (e) return structured evidence to the agent's reasoning loop.

8. **Implement cross-paradigm fusion (optional).** When evidence from multiple paradigms is available, project all memory states into the unified space, apply max-pooling across paradigm-aligned representations to produce a single fused context vector, then pass this to the retriever. This enables zero-shot fusion without any cross-paradigm training data.

9. **Evaluate with multi-hop QA benchmarks.** Test retrieval quality using Exact Match, F1, and ROUGE-1 on multi-hop reasoning tasks (WikiMultiHopQA, NarrativeQA, or MuSiQue are standard). Compare against single-paradigm baselines to verify cross-paradigm generalization.

10. **Monitor memory efficiency.** Track the average character length of retrieved evidence (target: ~2,200 characters vs. 4,400-5,900 for baselines) and the unique information ratio (target: ~47%). MemAdapter should retrieve more concise, less redundant evidence.

## Concrete Examples

**Example 1: Unified memory layer for a research agent**

User: "I'm building a research agent that has a Neo4j knowledge graph, a LoRA-adapted summarizer, and a streaming conversation cache. I need a single retrieval interface that can pull evidence from all three."

Approach:
1. Classify: Neo4j = explicit, LoRA adapter = parametric, conversation cache = latent
2. Define unified space: 1536-dim embedding space with cosine similarity
3. Implement three alignment modules (feed-forward, 2 hidden layers of 768 units each)
4. Use Neo4j as anchor paradigm -- train anchor alignment module on graph node/edge embeddings
5. Train generative retriever on graph-based retrieval examples (Stage 1)
6. Train parametric alignment module: collect 2,500 (query, LoRA-hidden-state, graph-state) triples, contrastive loss with tau=0.07
7. Train latent alignment module: same procedure with KV-cache snapshots as target states
8. Wire up unified `retrieve(query)` method that projects all three backends, runs retriever, returns structured evidence

Output:
```python
class MemAdapterRetriever:
    def __init__(self, unified_dim=1536):
        self.align_explicit = FeedForwardAlign(neo4j_dim, unified_dim)  # anchor, frozen
        self.align_parametric = FeedForwardAlign(lora_hidden_dim, unified_dim)
        self.align_latent = FeedForwardAlign(kv_cache_dim, unified_dim)
        self.retriever = GenerativeSubgraphRetriever()  # LoRA-tuned LLM

    def retrieve(self, query, memory_sources):
        projections = []
        for source in memory_sources:
            aligned = self.alignment_modules[source.paradigm](source.state)
            projections.append(aligned)
        if len(projections) > 1:
            fused = torch.max(torch.stack(projections), dim=0).values  # zero-shot fusion
        else:
            fused = projections[0]
        evidence_subgraph = self.retriever.generate(query, fused)
        return evidence_subgraph
```

**Example 2: Fast adaptation to a new memory backend**

User: "We already have MemAdapter working with our vector store. Now we want to add CARE (a latent memory system) without retraining everything."

Approach:
1. Keep existing anchor alignment module and generative retriever frozen
2. Implement a new `FeedForwardAlign` module for CARE's latent state dimensions
3. Collect 2,500 demonstrations: for each query, capture both the existing anchor memory state and CARE's internal hidden states
4. Train only the new alignment module using InfoNCE contrastive loss (tau=0.07, lr=1e-4, batch=32)
5. Expected wall-clock: ~13 minutes on a single GPU
6. Plug the new module into the existing retrieval interface

Output:
```python
# Only this module is trained -- everything else stays frozen
align_care = FeedForwardAlign(care_state_dim, 1536)

# Contrastive training loop
for batch in dataloader:  # 2,500 samples, batch_size=32
    anchor_proj = frozen_align_explicit(batch.anchor_states)
    target_proj = align_care(batch.care_states)
    loss = info_nce_loss(anchor_proj, target_proj, temperature=0.07)
    loss.backward()
    optimizer.step()

# Done -- register and use
retriever.register_paradigm("latent_care", align_care)
```

**Example 3: Designing the generative subgraph retriever from scratch**

User: "How should I implement the generative subgraph retriever component?"

Approach:
1. Start with a small pre-trained LLM (1.5B-3B params, e.g., Qwen2.5-1.5B or Llama-3.2-1B)
2. Add LoRA adapters: rank=16, alpha=32, applied to q_proj and v_proj in attention layers
3. Prepare training data: 40K multi-hop QA examples (HotpotQA works well), each with query + memory context + gold evidence subgraph serialized as a linearized string
4. Train via knowledge distillation from a larger teacher (7B-32B): minimize KL-divergence between teacher and student token distributions (KL temperature=2.0, CE weight=1.0)
5. Use BF16 precision, effective batch size 128 (4 GPUs x batch 8 x 4 gradient accumulation steps)
6. Cap input at 4096 tokens, output at 512 tokens

Output (serialized evidence subgraph format):
```
[SUBGRAPH]
  [NODE] Marie Curie | physicist, chemist | born 1867
  [NODE] Nobel Prize Physics | 1903 | shared with Pierre Curie
  [EDGE] Marie Curie -> received -> Nobel Prize Physics
  [EDGE] Nobel Prize Physics -> year -> 1903
[/SUBGRAPH]
```

## Best Practices

- **Do:** Use explicit graph-based memory as your anchor paradigm -- its discrete structure produces the most stable alignment targets and the paper's best results use this as the frozen reference point.
- **Do:** Keep the alignment modules lightweight (2-3 feed-forward layers). The entire point is that alignment is cheap; complex alignment networks defeat the purpose and overfit on small demonstration sets.
- **Do:** Control training data diversity in Stage 1. Balance input length, reasoning hop count, and structural variety. Uniform sampling from a single distribution produces a weaker retriever.
- **Do:** Use max-pooling (not mean-pooling or concatenation) for cross-paradigm fusion. Max-pooling preserves the strongest signal from each paradigm without dilution.
- **Avoid:** Training the generative retriever and alignment modules jointly. The two-stage separation is critical -- joint training collapses the unified space and prevents plug-and-play adaptation.
- **Avoid:** Using more than ~2,500 demonstrations for Stage 2 alignment. The contrastive objective converges quickly; more data adds diminishing returns and the 13-minute training budget is a feature, not a limitation.

## Error Handling

- **Paradigm dimension mismatch:** If a new memory backend produces states of unexpected dimensionality, the alignment module will fail silently (projecting garbage). Always validate input dimensions match the alignment module's expected input size before training.
- **Degenerate alignment collapse:** If contrastive training produces identical projections for all inputs (loss plateaus near log(batch_size)), reduce learning rate by 10x or increase temperature tau from 0.07 to 0.1. This typically means the alignment module is too large for the data volume.
- **Retriever hallucination:** The generative retriever can fabricate subgraph nodes not present in memory. Post-filter generated evidence against the actual memory store -- reject any node or edge that cannot be grounded in at least one memory backend.
- **Fusion degradation:** If fusing 3+ paradigms produces worse results than any single paradigm, one paradigm is contributing noise. Test each paradigm individually and exclude any that underperform a no-memory baseline.

## Limitations

- The generative subgraph retriever requires a pre-trained LLM (1.5B+ parameters) as the backbone -- this is not a lightweight embedding-only approach and needs GPU inference at retrieval time.
- Alignment quality depends on having a good anchor paradigm. If the anchor memory system is weak (poor coverage or noisy), all aligned paradigms inherit that weakness.
- Zero-shot fusion works best with 2-3 paradigms. Beyond that, max-pooling over many paradigm projections can wash out nuanced signals.
- The 13-minute alignment time assumes ~2,500 demonstrations are already available. Collecting high-quality paired demonstrations (same queries resolved through different paradigms) can itself be expensive.
- The approach is validated on multi-hop QA benchmarks (WikiMultiHopQA, NarrativeQA, MuSiQue). Generalization to other agent tasks (tool use, planning, dialogue) is plausible but unverified.

## Reference

**Paper:** [MemAdapter: Fast Alignment across Agent Memory Paradigms via Generative Subgraph Retrieval](https://arxiv.org/abs/2602.08369v1) (Zhang et al., 2026)

**What to look for:** Section 3 for the unified memory space formalization and two-stage training procedure; Section 3.3 for the contrastive alignment objective (Equation 9); Section 4 for experimental results across paradigms and scales; Algorithm 1 for the complete cross-paradigm alignment procedure.