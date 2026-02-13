---
name: "ctelm-decoding-manipulating-embeddings"
description: "Decode, interpret, and manipulate text embeddings using Embedding Language Models (ELMs). Aligns LLMs to embedding spaces so they can describe what an embedding represents, compare embeddings, generate text from novel vectors, and steer outputs via concept vectors. Triggers: 'decode embeddings into text', 'interpret embedding space', 'manipulate embeddings with concept vectors', 'reverse embeddings to natural language', 'explore embedding space', 'generate text from embedding vectors'"
---

# Decoding and Manipulating Text Embeddings with Embedding Language Models

This skill enables Claude to help users build systems that align Large Language Models to embedding spaces using the Embedding Language Model (ELM) architecture from the ctELM paper. The core capability is bidirectional: projecting embeddings into an LLM's input space so it can decode them into natural language descriptions, and steering generated text by performing arithmetic on embeddings with learned concept vectors. This goes beyond standard embedding similarity search -- it makes embedding spaces interpretable, navigable, and generative.

## When to Use

- When the user wants to build a system that converts embeddings back into human-readable text descriptions
- When the user asks to interpret or explain what a specific embedding vector represents
- When the user needs to compare two embeddings and generate a natural language explanation of their differences
- When the user wants to manipulate embeddings along semantic axes (e.g., adjusting for demographics, topic, sentiment) and decode the result
- When the user is building a retrieval system and wants to make embedding results explainable to end users
- When the user asks to generate synthetic text by interpolating between or composing embedding vectors
- When the user wants to build a clinical trial search or biomedical embedding exploration tool

## Key Technique

**Embedding Language Models (ELMs)** solve the "black box" problem of text embeddings. Standard embeddings compress text into dense vectors useful for similarity search but unreadable by humans. ELMs reverse this by training an LLM to accept embedding vectors as input tokens and produce meaningful text from them. The architecture adds a learned projection layer (linear or shallow MLP) that maps embedding dimensions into the LLM's token representation space. Once projected, the LLM treats the embedding as context and generates text conditioned on it -- descriptions, comparisons, or entirely new documents.

The training framework uses three complementary tasks: **Describe** (generate a text description from a single embedding), **Compare** (produce a natural language comparison given two embeddings), and **Generate** (produce a plausible full document from a novel or manipulated vector). Training data pairs embeddings with their source texts, augmented by expert-validated synthetic examples created through concept vector manipulation and LLM-assisted generation.

**Concept vectors** are the mechanism for controlled manipulation. By computing the mean embedding difference between two groups (e.g., trials involving elderly vs. young participants), you obtain a direction in embedding space that encodes that concept. Adding a scaled concept vector to any embedding shifts the generated text along that semantic axis: `modified = original + α * concept_vector`. The ctELM paper demonstrates this with age and sex of clinical trial participants, but the technique generalizes to any binary or continuous attribute present in the training data.

## Step-by-Step Workflow

1. **Select base models.** Choose an embedding model for your domain (e.g., BGE, SciBERT, PubMedBERT for biomedical; text-embedding-3-large or sentence-transformers for general use) and a decoder LLM (e.g., LLaMA 3, Gemma 2, Mistral). The embedding dimension and LLM hidden dimension determine the projection layer shape.

2. **Build the projection layer.** Implement a learnable linear transformation `W` (shape: `[embedding_dim, llm_hidden_dim]`) plus bias that maps embedding vectors into the LLM's token space. Initialize with Xavier uniform. For higher capacity, use a two-layer MLP with GELU activation.

   ```python
   import torch.nn as nn

   class EmbeddingProjector(nn.Module):
       def __init__(self, embed_dim, llm_dim):
           super().__init__()
           self.proj = nn.Sequential(
               nn.Linear(embed_dim, llm_dim),
               nn.GELU(),
               nn.Linear(llm_dim, llm_dim),
           )

       def forward(self, embedding):
           # embedding: [batch, embed_dim] -> [batch, 1, llm_dim]
           return self.proj(embedding).unsqueeze(1)
   ```

3. **Prepare training pairs.** For each document in your corpus: (a) compute its embedding with the chosen encoder, (b) pair it with the source text. Create three task variants per pair:
   - **Describe**: `(embedding) -> "This document discusses..."`
   - **Compare**: `(embedding_a, embedding_b) -> "Document A focuses on X while Document B..."`
   - **Generate**: `(embedding) -> full source text`

4. **Create synthetic augmentation data.** Compute concept vectors for attributes of interest by averaging embeddings per group and subtracting: `cv_age = mean(old_group) - mean(young_group)`. Apply scaled concept vectors to existing embeddings to create shifted training pairs, then use an LLM to generate plausible texts for shifted vectors. Have domain experts validate a sample.

5. **Train the ELM.** Freeze the embedding encoder. Train the projection layer and (optionally) LoRA adapters on the LLM using standard causal language modeling loss. The input sequence is: `[projected_embedding_tokens] [task_prompt] [target_text]`. Use learning rate ~1e-4, batch size 8-32, for 3-10 epochs. Train describe/compare/generate tasks jointly with task-specific prompt prefixes.

6. **Decode embeddings at inference.** Given a new embedding vector, project it through the trained projector, prepend to a task prompt (e.g., "Describe this document:"), and let the LLM generate. The output is a natural language interpretation of what the embedding encodes.

7. **Compare embeddings.** Concatenate two projected embeddings with a comparison prompt. The model generates text explaining semantic differences and similarities, going beyond a raw cosine similarity score.

8. **Manipulate with concept vectors.** To steer a document's attributes: compute `new_emb = original_emb + α * concept_vector`, then decode. Sweep `α` from -2.0 to +2.0 to explore the effect gradient. Validate that cosine similarity to the original stays within a reasonable range (> 0.7) to avoid incoherent outputs.

9. **Build the search and exploration UI.** For a retrieval application, embed the query, retrieve top-K neighbors, then use the ELM to generate natural language explanations for each result and comparisons between results. This makes embedding-based search transparent and auditable.

10. **Evaluate outputs.** Measure semantic fidelity (cosine similarity between the embedding of generated text and the input embedding), factual accuracy via manual review, and concept vector responsiveness (does sweeping α produce monotonic changes in the target attribute?).

## Concrete Examples

**Example 1: Building an Embedding Decoder for Clinical Trial Search**

User: "I have a database of clinical trial embeddings from PubMedBERT. I want users to search by natural language query, get results, and see AI-generated explanations of each result and why it matches."

Approach:
1. Set up PubMedBERT as the encoder and Gemma-2-9B as the decoder LLM
2. Build a two-layer MLP projector mapping PubMedBERT's 768-dim to Gemma's 3584-dim hidden
3. Create training data by pairing each trial's embedding with its abstract, using describe/compare task templates
4. Train the projector + LoRA on Gemma for 5 epochs
5. At query time: embed query -> retrieve top-5 by cosine sim -> for each result, decode its embedding to a summary -> for the top pair, generate a comparison

Output structure:
```
Query: "Phase 3 diabetes drug trials in elderly patients"

Result 1 (similarity: 0.89):
  ELM Description: "This is a Phase 3 randomized controlled trial
  evaluating a GLP-1 receptor agonist in type 2 diabetes patients
  aged 65-80. Primary endpoint is HbA1c reduction at 24 weeks."

  Why it matches: "Both focus on Phase 3 diabetes pharmacotherapy
  in elderly populations. The trial uses a GLP-1 agonist class
  consistent with the query's drug trial focus."

Result 2 (similarity: 0.84):
  ELM Description: "This trial examines SGLT2 inhibitor safety in
  patients over 70 with type 2 diabetes and renal comorbidities..."
```

**Example 2: Manipulating Embeddings Along a Concept Vector**

User: "I have embeddings of clinical trials. I want to take a trial designed for adults and see what it would look like modified for a pediatric population."

Approach:
1. Compute concept vector: `cv_age = mean(embeddings[pediatric_trials]) - mean(embeddings[adult_trials])`
2. For the target trial, compute: `pediatric_emb = adult_trial_emb + 1.0 * cv_age`
3. Decode both the original and modified embeddings through the ELM
4. Generate a comparison between the two decoded texts

```python
import numpy as np

# Compute age concept vector from labeled groups
adult_embs = np.stack([emb for emb, label in data if label == "adult"])
peds_embs = np.stack([emb for emb, label in data if label == "pediatric"])
cv_age = peds_embs.mean(axis=0) - adult_embs.mean(axis=0)

# Apply to a specific trial
original_emb = encoder.encode(trial_abstract)
modified_emb = original_emb + 1.0 * cv_age

# Decode both
original_desc = elm.decode(original_emb, prompt="Describe this trial:")
modified_desc = elm.decode(modified_emb, prompt="Describe this trial:")
comparison = elm.compare(original_emb, modified_emb,
    prompt="How do these trials differ?")
```

Output:
```
Original: "A Phase 2 trial of Drug X in adults aged 40-65 with
moderate asthma. Doses: 200mg and 400mg oral daily."

Modified (pediatric shift): "A Phase 2 trial of Drug X in children
aged 6-12 with moderate asthma. Doses: weight-adjusted 50mg and
100mg oral daily."

Comparison: "The modified trial targets a pediatric population
with weight-based dosing adjustments. Endpoints shift from FEV1
to age-appropriate lung function measures."
```

**Example 3: Interpolating Between Two Embeddings to Explore the Space**

User: "I want to smoothly interpolate between a cardiology trial and an oncology trial to see what the embedding space looks like in between."

Approach:
1. Get embeddings for both trials
2. Create interpolation steps: `interp_emb = (1-t) * emb_cardio + t * emb_onco` for t in [0, 0.25, 0.5, 0.75, 1.0]
3. Decode each interpolated embedding

```python
steps = [0.0, 0.25, 0.5, 0.75, 1.0]
for t in steps:
    interp = (1 - t) * emb_cardiology + t * emb_oncology
    desc = elm.decode(interp, prompt="Describe this trial:")
    print(f"t={t}: {desc}")
```

Output:
```
t=0.0:  "Randomized trial of beta-blocker therapy for heart failure
         with reduced ejection fraction."
t=0.25: "Trial examining cardioprotective agents in patients with
         cardiac complications secondary to chemotherapy."
t=0.5:  "Study of cardiovascular monitoring protocols in cancer
         patients undergoing anthracycline treatment."
t=0.75: "Oncology trial with cardiac safety endpoints for a novel
         tyrosine kinase inhibitor."
t=1.0:  "Phase 3 trial of immune checkpoint inhibitor combination
         therapy in metastatic non-small cell lung cancer."
```

## Best Practices

- **Do:** Freeze the embedding encoder during ELM training. The goal is to learn to read the existing embedding space, not change it. Only train the projection layer and optional LoRA adapters on the LLM.
- **Do:** Train all three tasks (describe, compare, generate) jointly. The paper shows multi-task training produces better results than single-task specialists.
- **Do:** Normalize concept vectors to unit length before scaling by `α`. This makes the scaling factor interpretable and consistent across different concepts.
- **Do:** Validate concept vectors by checking that the top-K nearest neighbors of `emb + cv` shift in the expected direction (e.g., more pediatric trials appear near the shifted vector).
- **Avoid:** Using concept vector magnitudes `|α| > 2.0` without validation. Large shifts push embeddings into low-density regions where the decoder produces incoherent text.
- **Avoid:** Assuming concept vectors are orthogonal. Age and sex vectors may be correlated in biomedical data. Decorrelate using Gram-Schmidt if you need independent manipulation of multiple attributes.
- **Avoid:** Skipping the synthetic data augmentation step. The paper shows that expert-validated synthetic pairs from concept vector manipulation significantly improve the model's ability to handle shifted embeddings.

## Error Handling

- **Incoherent decoded text:** The input embedding is likely in a low-density region. Check cosine similarity to the nearest training embedding. If below 0.5, the decoder is extrapolating unreliably. Fall back to reporting the nearest neighbor description instead.
- **Concept vector has no effect:** The embedding model may not encode that attribute. Verify by checking whether the concept vector has non-trivial magnitude and that the two groups have meaningfully different mean embeddings (cosine distance > 0.1).
- **Dimension mismatch errors:** Ensure the projection layer input dimension exactly matches the embedding model's output dimension. Common pitfall: some models output pooled (768-dim) vs. token-level (768 x seq_len) representations.
- **Training divergence:** If loss spikes, reduce learning rate. The projection layer trains stably at 1e-4, but LoRA on the LLM may need 1e-5. Use gradient clipping at 1.0.
- **Generated text contradicts the embedding:** The LLM's prior knowledge may override the embedding signal. Increase the number of projected embedding tokens (use 4-8 tokens instead of 1) to give the embedding more influence in the context.

## Limitations

- ELMs require paired (embedding, text) training data. You cannot decode embeddings from an arbitrary embedding model without first training a projection layer on in-domain pairs.
- Concept vector manipulation assumes linear structure in the embedding space. Non-linear attribute relationships (e.g., complex disease interactions) may not respond to simple vector arithmetic.
- The decoder LLM can hallucinate details not present in the embedding, especially for generate tasks. Always treat decoded text as an approximation, not a faithful reconstruction.
- Training requires GPU resources: the paper uses LLaMA-2 and Gemma-2 scale models. Smaller LLMs (< 3B parameters) may lack the capacity for accurate decoding.
- Domain transfer is not free. An ELM trained on clinical trial embeddings will not decode legal document embeddings well. The projection layer and training tasks must be rebuilt per domain.

## Reference

**Paper:** [ctELM: Decoding and Manipulating Embeddings of Clinical Trials with Embedding Language Models](https://arxiv.org/abs/2601.18796v1) -- Ondov et al., 2026. Focus on Sections 3-4 for the ELM architecture, projection layer design, multi-task training setup, and concept vector arithmetic for controlled generation.