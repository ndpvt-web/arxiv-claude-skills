---
name: "muco-multi-turn-contrastive-learning"
description: "Implement multi-turn contrastive learning for multimodal embedding models. Restructures query-target pairs as multi-turn dialogues so an MLLM processes multiple related pairs in a single forward pass, amplifying effective batch size and training efficiency. Use when: 'build a multimodal embedding model', 'train contrastive learning with multiple queries per image', 'implement multi-turn dialogue for retrieval', 'optimize contrastive training throughput', 'create multimodal retrieval embeddings', 'batch multiple query-target pairs per forward pass'."
---

# MuCo: Multi-Turn Contrastive Learning for Multimodal Embeddings

This skill enables Claude to implement Multi-Turn Contrastive Learning (MuCo), a technique that reformulates standard contrastive learning as multi-turn dialogue. Instead of treating each query-target pair as an independent training example requiring its own forward pass, MuCo packs multiple related query-target pairs for the same image into a single conversational sequence. The MLLM processes all pairs at once, extracting separate embeddings from each dialogue turn's EOS token. This multiplies the effective batch size without increasing GPU memory proportionally, yielding state-of-the-art results on MMEB and M-BEIR retrieval benchmarks.

## When to Use

- When building a multimodal embedding model that must handle image-text, text-image, or cross-modal retrieval tasks
- When training contrastive learning on datasets where multiple queries relate to the same image (e.g., multiple captions, questions, or descriptions per image)
- When contrastive training is bottlenecked by batch size and you need to amplify effective negatives without scaling hardware
- When implementing a universal multimodal retriever that handles heterogeneous task types (classification, retrieval, VQA, visual grounding) in a single model
- When adapting a Multimodal Large Language Model (e.g., Qwen2-VL, InternVL, LLaVA) into an embedding model for search/retrieval
- When curating or structuring multimodal training data and want to maximize supervision signal per example

## Key Technique

**The Single-Turn Problem.** Standard contrastive learning for multimodal embeddings (as in VLM2Vec or MME5) processes one query-target pair per forward pass. Given an image with 5 associated queries, the model runs 5 separate forward passes, each encoding the image from scratch. The image representation is recomputed redundantly, and the model never sees the relationships between queries that share context.

**The Multi-Turn Solution.** MuCo reformats multiple query-target pairs as turns in a dialogue. A single forward pass through the MLLM processes the shared image once, then generates embeddings for each query-target pair from successive dialogue turns. Concretely, the image is placed in the system/first message, and each subsequent turn contains one query (or target) with an instruction. The embedding for each turn is extracted from the EOS (end-of-sequence) token at that turn's boundary. Because causal attention lets later turns attend to the shared image context and all prior turns, each embedding is contextually enriched without redundant computation.

**Why It Works.** By packing N pairs into one sequence, MuCo multiplies the effective batch size by N at near-constant memory cost (the image is encoded once). The contrastive loss is computed over all query and target embeddings across the batch, with each turn's EOS embedding treated as an independent vector in the contrastive pool. The M3T dataset (5M examples) is specifically curated to group related pairs per image across diverse retrieval tasks, ensuring the multi-turn format has semantically coherent turns rather than arbitrary groupings.

## Step-by-Step Workflow

1. **Select an MLLM backbone** that supports multi-turn dialogue with image inputs (e.g., Qwen2-VL-2B/7B, InternVL2). The model must use causal attention and support interleaved image-text conversation turns.

2. **Structure training data into multi-turn format.** For each image, group all associated query-target pairs. Format them as a dialogue where:
   - Turn 0 (system/context): Contains the image and a task-type instruction
   - Turns 1..N: Each contains one query or one target text with a role-specific prompt
   - Append an EOS token after each turn to serve as the embedding extraction point

3. **Define the embedding extraction mechanism.** After the forward pass, extract the hidden state at each EOS token position. Apply L2 normalization to each extracted vector. Each EOS position yields one independent embedding vector — queries and targets from the same sequence are separate vectors in the contrastive pool.

4. **Implement the multi-turn contrastive loss.** Use InfoNCE with in-batch negatives. For a batch of B sequences each containing N turns, the effective contrastive pool has B*N query embeddings and B*N target embeddings. Compute cosine similarity with a learnable temperature parameter. Positive pairs are the original query-target associations; all other combinations are negatives.

5. **Curate or adapt a multi-turn dataset.** Group existing retrieval datasets by image:
   - Merge multiple captions per image into multi-turn sequences
   - Combine VQA question-answer pairs for the same image
   - Mix task types (retrieval, classification, grounding) sharing the same visual context
   - Target 3-5 turns per sequence for optimal throughput vs. sequence length tradeoff

6. **Handle heterogeneous task types with instruction prefixes.** Prepend a short task instruction to each turn (e.g., `"Retrieve the caption that describes this image"`, `"Classify the image into one of the following categories"`) so the model learns task-conditioned embeddings within the same sequence.

7. **Configure training hyperparameters.** Set batch size to maximize GPU utilization (the effective contrastive batch is `batch_size * turns_per_sequence`). Use cosine learning rate schedule with warmup. Typical config: lr=1e-5, warmup 1-3% of steps, bf16 mixed precision, gradient checkpointing enabled.

8. **Implement sequence packing and padding.** Pad shorter multi-turn sequences to the maximum turn count in the batch. Use attention masks to prevent padded turns from contributing to the loss. Track which EOS positions correspond to real vs. padded turns.

9. **Evaluate on standard benchmarks.** Test on MMEB (36 datasets across classification, retrieval, VQA, visual grounding) and M-BEIR (8 retrieval datasets). Report Recall@K and nDCG@10. Compare against single-turn baselines to quantify the efficiency and accuracy gains.

10. **Deploy the trained model for inference.** At inference time, format each query as a single-turn dialogue (no multi-turn needed). Extract the EOS token embedding, normalize it, and use it for nearest-neighbor search against a pre-computed target index.

## Concrete Examples

**Example 1: Converting single-turn training data to MuCo format**

User: I have an image-caption dataset where each image has 4 captions. How do I restructure it for MuCo training?

Approach:
1. Group captions by image ID
2. Format as multi-turn dialogue with shared image context
3. Extract embeddings from each turn's EOS token

```python
def build_muco_sequence(image_path, captions, task_instruction):
    """Convert grouped captions into a multi-turn MuCo training sequence."""
    conversation = []

    # Turn 0: shared image context
    conversation.append({
        "role": "system",
        "content": [
            {"type": "image", "path": image_path},
            {"type": "text", "text": task_instruction}
        ]
    })

    # Each caption becomes a separate turn -> separate embedding
    for i, caption in enumerate(captions):
        # Query turn: asks for retrieval
        conversation.append({
            "role": "user",
            "content": f"<query_turn_{i}> Represent the visual content for retrieving: {caption} </s>"
        })
        # The target is the caption itself, processed in a parallel target sequence
        # (targets are encoded in their own multi-turn sequence without the image)

    return conversation

# Example usage
image = "coco/train2017/000000001234.jpg"
captions = [
    "A dog playing fetch in a park",
    "Golden retriever catching a frisbee on grass",
    "Pet dog leaping to catch a toy outdoors",
    "A happy dog playing in a green field"
]
sequence = build_muco_sequence(
    image, captions,
    instruction="Retrieve the text that best describes this image."
)
# One forward pass -> 4 query embeddings (one per EOS token)
```

**Example 2: Implementing the multi-turn embedding extraction**

User: How do I extract separate embeddings from each turn in a single forward pass?

Approach:
1. Run the full multi-turn sequence through the MLLM
2. Locate EOS token positions in the output
3. Extract and normalize hidden states at those positions

```python
import torch
import torch.nn.functional as F

class MuCoEmbeddingExtractor:
    def __init__(self, model, tokenizer, eos_token_id):
        self.model = model
        self.tokenizer = tokenizer
        self.eos_token_id = eos_token_id

    def extract_turn_embeddings(self, input_ids, attention_mask, pixel_values):
        """Extract one embedding per dialogue turn from EOS token positions."""
        # Single forward pass for entire multi-turn sequence
        outputs = self.model(
            input_ids=input_ids,
            attention_mask=attention_mask,
            pixel_values=pixel_values,
            output_hidden_states=True
        )
        hidden_states = outputs.hidden_states[-1]  # Last layer: (B, seq_len, dim)

        # Find EOS token positions — each marks the end of one turn
        eos_mask = (input_ids == self.eos_token_id)  # (B, seq_len)

        embeddings_per_sample = []
        for b in range(input_ids.size(0)):
            eos_positions = eos_mask[b].nonzero(as_tuple=True)[0]
            turn_embeds = hidden_states[b, eos_positions, :]  # (num_turns, dim)
            turn_embeds = F.normalize(turn_embeds, p=2, dim=-1)
            embeddings_per_sample.append(turn_embeds)

        return embeddings_per_sample  # List of (num_turns, dim) tensors
```

**Example 3: Multi-turn contrastive loss with amplified batch size**

User: How does MuCo's loss function differ from standard InfoNCE?

Approach:
1. Gather all turn embeddings across the batch into flat pools
2. Compute similarity matrix over the amplified pool
3. Apply InfoNCE with correct positive pair tracking

```python
def muco_contrastive_loss(query_embeds_list, target_embeds_list, temperature=0.07):
    """
    Multi-turn contrastive loss.

    Args:
        query_embeds_list: list of (num_turns, dim) tensors, one per batch element
        target_embeds_list: list of (num_turns, dim) tensors, one per batch element
        temperature: learnable temperature scalar

    With batch_size=32 and 4 turns each, effective pool = 128 pairs.
    Standard single-turn would need batch_size=128 for the same number of negatives.
    """
    # Flatten all turn embeddings into single pools
    all_queries = torch.cat(query_embeds_list, dim=0)    # (B*N, dim)
    all_targets = torch.cat(target_embeds_list, dim=0)    # (B*N, dim)

    # Cosine similarity matrix
    sim_matrix = torch.matmul(all_queries, all_targets.T) / temperature  # (B*N, B*N)

    # Positives are on the diagonal (query_i matches target_i)
    labels = torch.arange(sim_matrix.size(0), device=sim_matrix.device)
    loss = F.cross_entropy(sim_matrix, labels)

    return loss

# Example: batch=32, turns=4 -> 128 effective pairs per step
# Equivalent to 4x batch size increase at ~1x memory cost
```

**Example 4: Building an M3T-style mixed-task dataset**

User: I want to create a multi-turn dataset mixing retrieval, classification, and VQA tasks for the same images.

Approach:
1. Identify images that appear across multiple task datasets
2. Group heterogeneous tasks per image
3. Format each task type as a distinct turn with a task-specific instruction

```python
def build_mixed_task_sequence(image_path, tasks):
    """
    Build a multi-turn sequence mixing different task types.

    tasks = [
        {"type": "retrieval", "query": "a sunset over mountains", "target": "caption_id_42"},
        {"type": "classification", "query": "What scene is this?", "target": "nature/landscape"},
        {"type": "vqa", "query": "What time of day is it?", "target": "evening"},
    ]
    """
    TASK_INSTRUCTIONS = {
        "retrieval": "Represent this image for text retrieval.",
        "classification": "Represent this image for classification.",
        "vqa": "Represent this image to answer the question.",
    }

    turns = []
    for task in tasks:
        instruction = TASK_INSTRUCTIONS[task["type"]]
        turns.append({
            "role": "user",
            "content": f"[{instruction}] {task['query']} </s>"
        })

    return {
        "image": image_path,
        "turns": turns,
        "targets": [t["target"] for t in tasks],
        "num_turns": len(turns)
    }
```

## Best Practices

- **Do:** Group semantically related queries per image. MuCo works best when turns share genuine visual context — multiple captions, questions, or descriptions about the same image. Random grouping degrades coherence.
- **Do:** Keep turn counts between 3-6 per sequence. Fewer than 3 underutilizes the multi-turn advantage. More than 6 causes sequence length to dominate and diminishes throughput gains.
- **Do:** Use distinct task instructions per turn type so the model learns task-conditioned embeddings. The instruction prefix is what tells the model whether to produce a retrieval, classification, or QA embedding.
- **Do:** L2-normalize all extracted embeddings before computing the contrastive loss and at inference time. This ensures cosine similarity is well-calibrated.
- **Avoid:** Mixing unrelated images into a single multi-turn sequence. Each sequence must share one visual context — the multi-turn structure is about multiple queries to the *same* image, not arbitrary batching.
- **Avoid:** Using mean pooling over the full sequence. Each turn must have its own embedding (from its own EOS token), not a pooled representation of the whole dialogue. Mean pooling collapses the per-turn signal.
- **Avoid:** Skipping attention masking for padded turns. If sequences have different numbers of real turns, padded EOS tokens will produce garbage embeddings that corrupt the contrastive loss.

## Error Handling

- **Unequal turn counts across batch.** Pad shorter sequences to the maximum turn count. Use a boolean mask to exclude padded turns from the contrastive loss computation. Never let padded embeddings participate as positives or hard negatives.
- **Sequence length overflow.** Multi-turn sequences can exceed the MLLM's context window. Monitor total token count (image tokens + all turn tokens). If a sequence exceeds the limit, split it into two sequences sharing the same image prefix, or reduce the number of turns.
- **Degenerate embeddings (collapse).** If all turn embeddings converge to similar vectors, the temperature is too low or turns are too similar. Increase temperature, diversify turn content, or add a regularization term encouraging embedding diversity across turns.
- **OOM during training.** Enable gradient checkpointing on the MLLM backbone. The multi-turn approach increases sequence length, which scales memory quadratically with attention. Use flash attention if available.
- **Misaligned positive pairs after flattening.** When flattening turn embeddings across the batch, maintain a mapping from each query embedding to its positive target. Off-by-one errors in indexing will silently degrade training. Validate with a small batch and verify the diagonal of the similarity matrix corresponds to true positives.

## Limitations

- **Requires grouped data.** MuCo only provides benefits when multiple queries naturally relate to the same image. Datasets with only one caption per image see no multi-turn advantage — standard single-turn contrastive learning is equivalent.
- **Increased sequence length.** Packing multiple turns increases the token count per forward pass. For models with limited context windows or quadratic attention cost, the throughput gain may be partially offset by longer sequences.
- **Causal attention dependency.** MuCo assumes a causal (autoregressive) MLLM where later turns attend to earlier ones. Bidirectional encoders (e.g., CLIP's vision transformer) cannot directly use this technique without architectural changes.
- **Inference is single-turn.** The multi-turn structure is a training-time optimization. At inference, each query is still processed as a single-turn input. The benefit is a better-trained model, not faster inference.
- **Task mixing requires careful balancing.** Mixing too many dissimilar task types per sequence (e.g., classification + grounding + retrieval) can confuse the shared image representation. Empirically, grouping similar tasks yields better coherence.

## Reference

[MuCo: Multi-turn Contrastive Learning for Multimodal Embedding Model](https://arxiv.org/abs/2602.06393) — Gu et al., 2026. Focus on Section 3 (Method) for the multi-turn dialogue formulation and EOS-token embedding extraction, Section 4 for the M3T dataset construction, and Tables 1-2 for MMEB/M-BEIR benchmark comparisons against VLM2Vec and single-turn baselines.