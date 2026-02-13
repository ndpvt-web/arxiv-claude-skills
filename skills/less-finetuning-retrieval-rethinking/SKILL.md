---
name: "less-finetuning-retrieval-rethinking"
description: |
  Implements the Synthesize-Train-Merge (STM) pipeline for building domain-specialized
  retrieval models from general-purpose LLMs using synthetic hard negatives, prompt
  optimization, and model merging. Use this skill when the user needs to adapt an
  embedding or retriever model for a specialized domain (biomedical, legal, finance)
  without massive pretraining budgets.
  Trigger phrases:
  - "build a domain-specific retriever"
  - "adapt an embedding model for biomedical retrieval"
  - "merge retrieval experts for RAG"
  - "generate synthetic hard negatives for retrieval training"
  - "fine-tune an LLM as a retriever with less data"
  - "improve RAG retrieval for a specialized domain"
---

# Synthesize-Train-Merge (STM): Domain-Specialized Retrievers via Synthetic Data and Model Merging

This skill enables Claude to guide users through building high-performing domain-specific retrieval models by applying the STM framework from Khattab et al. (2026). Instead of expensive continual pretraining on millions of domain pairs, STM trains multiple lightweight LoRA experts on curated data splits, optimizes retrieval prompts via evolutionary search, generates synthetic hard negatives to sharpen discrimination, and then merges experts via linear interpolation — achieving results that beat both individual experts and baselines trained on 8x more data.

## When to Use

- When a user wants to adapt a general-purpose LLM (Qwen, Gemma, Phi, LLaMA) into a domain-specific embedding/retriever model for RAG
- When building a biomedical, legal, or financial retrieval pipeline and the user lacks millions of in-domain query-passage pairs
- When the user has multiple heterogeneous training data sources (real domain data, synthetic data, general NLU data) and wants to combine expertise from each
- When the user asks how to generate synthetic hard negatives to improve retrieval training
- When the user wants to merge multiple fine-tuned retrieval adapters into a single model that excels across tasks
- When the user needs to optimize the retrieval prompt/instruction prepended to queries and passages in an embedding model
- When the user is experiencing "catastrophic forgetting" — their domain-fine-tuned retriever lost general retrieval ability

## Key Technique

**The core insight:** Training one monolithic retriever on all your data leaves performance on the table. Instead, train separate *expert* LoRA adapters on distinct data splits (e.g., domain-real, domain-synthetic, general-search, general-NLU), optimize each expert's retrieval prompt independently, then merge the adapters back into one model using linear interpolation. The merged model consistently outperforms every individual expert and avoids the forgetting problem of single-task fine-tuning.

**Synthetic hard negatives** are the data multiplier. Given a query, a positive passage, and a mined negative, you prompt an LLM to generate a new negative that is *lexically and topically similar* to the query but *semantically irrelevant or contradictory*. This forces the retriever to learn fine-grained distinctions rather than surface-level keyword matching. Critically, synthetic negatives help most for general-domain experts; high-quality domain data (e.g., curated medical corpora) may already contain sufficiently hard negatives, so adding synthetic ones can actually degrade performance for domain experts on larger models.

**Prompt optimization** via GEPA (Generative Prompt Evolution Algorithm) iteratively refines the instruction prefix prepended to queries during encoding. Starting from a seed prompt, an LLM proposes mutations, each is evaluated on a held-out dev set using nDCG@10, and the best-performing prompts survive to the next generation. This alone can boost retrieval performance by 17-23% for general-domain experts. A simpler random-prompt-per-batch strategy also works as a regularizer during training.

## Step-by-Step Workflow

### 1. Select a decoder-only backbone and prepare it for retrieval

Choose a model (Qwen3-0.6B, Gemma-2B, or Phi-4-Mini-3.8B are validated sizes). Modify the architecture: disable causal attention masking to enable bidirectional attention, and use EOS-token pooling to produce a single embedding vector. Add LoRA adapters to all linear layers.

```python
# Pseudocode: Prepare model for retrieval fine-tuning
from peft import get_peft_model, LoraConfig

config = LoraConfig(
    r=64,               # LoRA rank (tune per model size)
    lora_alpha=128,
    target_modules="all-linear",
    task_type="FEATURE_EXTRACTION",
)
model = get_peft_model(base_model, config)
# Disable causal mask — enable bidirectional attention
model.config.is_causal = False
```

### 2. Partition training data into expert splits

Divide your data into 3-4 thematic splits. For biomedical retrieval, a proven partition is:

| Expert         | Data Source                        | Example Size |
|----------------|------------------------------------|-------------|
| Domain-Real    | Curated medical QA pairs           | ~300K       |
| Domain-Synth   | LLM-generated medical pairs        | ~430K       |
| General-Search | Web search query-passage pairs     | ~440K       |
| General-NLU    | NLI, paraphrase, duplicate QA data | ~250K       |

Each expert will be trained independently, so splits do not need to be balanced.

### 3. Generate synthetic hard negatives for each split

For each (query, positive_passage, mined_negative) triple, prompt an LLM to create an adversarial negative:

```
System: You are an expert at creating challenging retrieval negatives.

Given:
- Query: {query}
- Relevant passage: {positive}
- Existing negative: {mined_negative}

Generate a new passage that:
1. Uses similar vocabulary and topic as the query
2. Appears relevant on the surface
3. Is factually incorrect, contradictory, or irrelevant to the query's actual information need

Output only the generated passage.
```

Use GPT-4 or a comparable model. Evaluate whether synthetic negatives help each expert on a dev set — skip them for domain experts if they degrade nDCG@10.

### 4. Train each expert independently with InfoNCE loss

Fine-tune each LoRA adapter using contrastive InfoNCE loss with in-batch negatives plus your hard negatives. Use a held-out dev set (e.g., NFCorpus, FiQA) for early stopping.

```python
# InfoNCE contrastive loss
def info_nce_loss(query_emb, pos_emb, neg_embs, temperature=0.02):
    pos_sim = cosine_similarity(query_emb, pos_emb) / temperature
    neg_sims = cosine_similarity(query_emb, neg_embs) / temperature
    logits = torch.cat([pos_sim, neg_sims], dim=-1)
    labels = torch.zeros(logits.size(0), dtype=torch.long)
    return F.cross_entropy(logits, labels)
```

Data saturation occurs around 100K samples per expert — you do not need the full split.

### 5. Optimize each expert's retrieval prompt via GEPA

Run evolutionary prompt search for each expert independently:

1. Start with 10 seed prompts (e.g., "Retrieve a medical passage that answers: ")
2. Use an LLM (LLaMA-70B or GPT-4) to propose 5-10 mutations per generation
3. Evaluate each prompt candidate on your dev set using nDCG@10
4. Keep top-k prompts, repeat for 5-10 generations
5. Select the prompt with highest dev nDCG@10

Store the winning prompt per expert — it will be used during merging evaluation.

### 6. Merge expert LoRA adapters via linear interpolation

Extract task vectors (adapter weights minus base weights) and combine:

```python
# Linear interpolation of LoRA experts
# alpha_k are weights that sum to 1.0
merged_adapter = sum(alpha_k * expert_k_adapter for k in experts)

# Grid search over weights on dev set
# Search space: alpha_k in {0.0, 0.1, 0.2, ..., 0.9}
# Constraint: sum(alpha_k) = 1.0
best_alphas = grid_search(
    experts=[medical_real, medical_synth, search, nlu],
    dev_tasks=["NFCorpus", "FiQA", "Quora"],
    metric="nDCG@10"
)
```

Linear interpolation consistently outperforms more complex merging methods (TIES, SLERP, DARE-TIES, Task Arithmetic).

### 7. Evaluate on both domain and general benchmarks

Test the merged model on domain-specific tasks (e.g., TREC-COVID, SciFact, NFCorpus) AND general tasks (e.g., FiQA, ArguAna, FEVER) to confirm no catastrophic forgetting. Report nDCG@10.

### 8. Deploy the merged retriever in your RAG pipeline

Use the merged model with its optimized prompt as the retriever. At inference time, prepend the selected prompt to each query before encoding.

```python
def encode_query(query, model, tokenizer, prompt):
    text = f"{prompt}{query}"
    inputs = tokenizer(text, return_tensors="pt", padding=True, truncation=True)
    with torch.no_grad():
        outputs = model(**inputs)
    return outputs.last_hidden_state[:, -1, :]  # EOS pooling
```

## Concrete Examples

**Example 1: Building a biomedical retriever from Phi-4-Mini**

User: "I want to build a retriever for biomedical literature search using Phi-4-Mini. I have 300K medical QA pairs from PubMed and want to also handle general queries."

Approach:
1. Prepare Phi-4-Mini with bidirectional attention and LoRA on all linear layers (3,072-dim embeddings)
2. Split data into: Medical-Real (300K PubMed pairs), Medical-Synth (generate 100K synthetic pairs using GPT-4), General-Search (use MS MARCO or similar, 200K sample), General-NLU (use NLI/paraphrase data, 100K)
3. Generate synthetic hard negatives for General-Search and General-NLU splits only (skip for medical — curated data already has hard negatives at this model size)
4. Train 4 LoRA experts independently with InfoNCE loss, early stopping on NFCorpus dev set
5. Run GEPA prompt optimization for each expert (5 generations, 10 candidates each)
6. Grid search merge weights on dev set: expect something like alpha=[0.3, 0.2, 0.3, 0.2] for [med-real, med-synth, search, nlu]
7. Evaluate merged model: expect ~0.67 medical nDCG@10 and ~0.60 general nDCG@10

Output:
```
Merged Phi-4-Mini Retriever Results (nDCG@10):
  TREC-COVID: 0.72  |  SciFact: 0.68  |  NFCorpus: 0.35
  FiQA: 0.41        |  ArguAna: 0.58  |  FEVER: 0.82
  Medical avg: 0.677 |  General avg: 0.603 | Combined: 0.646
```

**Example 2: Quick domain adaptation with minimal data**

User: "I only have 50K legal query-document pairs. Can I still build a competitive legal retriever?"

Approach:
1. Use Qwen3-0.6B as a lightweight backbone (1,024-dim embeddings)
2. Split into: Legal-Real (50K pairs), General-Search (100K from MS MARCO), General-NLU (50K from NLI)
3. Generate synthetic hard negatives for all three splits (small domain dataset benefits from synthetic augmentation)
4. Train 3 experts — data saturation at ~100K means 50K is sufficient per expert
5. Run GEPA prompt optimization: expect larger gains on general experts (+17-23%) than legal expert
6. Merge with linear interpolation, grid search weights on a legal dev set
7. The merged model will outperform any single expert and retain general capability

Output:
```
# Training config
backbone: Qwen3-0.6B
experts: [legal-real, general-search, general-nlu]
merge_weights: [0.4, 0.3, 0.3]  # higher weight on domain expert
total_training_pairs: ~200K  # vs millions for traditional approaches
```

**Example 3: Generating synthetic hard negatives**

User: "My retriever keeps returning passages that look relevant but are actually wrong. How do I fix this?"

Approach:
1. This is the classic "lexical overlap" failure — the retriever matches surface keywords, not semantics
2. Generate synthetic hard negatives from your existing training data:
   - For each training triple (query, positive, existing_negative):
   - Prompt GPT-4 to create a passage that uses query vocabulary but contradicts the correct answer
3. Add these as additional negatives during training
4. Re-train your retriever with InfoNCE loss including the synthetic negatives

Output:
```
Original triple:
  Query: "What is the mechanism of action of metformin?"
  Positive: "Metformin reduces hepatic glucose production and increases
            insulin sensitivity in peripheral tissues..."
  Mined negative: "Aspirin inhibits cyclooxygenase enzymes..."

Synthetic hard negative (generated):
  "Metformin primarily functions by stimulating pancreatic beta cells
   to increase insulin secretion, similar to sulfonylureas, and has
   no direct effect on hepatic glucose output."
  # Uses correct terminology but is factually wrong about mechanism
```

## Best Practices

- **Do** train experts on separate data splits rather than one model on all data — merging specialized experts consistently beats monolithic training
- **Do** use linear interpolation for merging; it outperforms TIES, SLERP, and DARE-TIES despite being simpler
- **Do** run prompt optimization (GEPA) per expert before merging — gains of 7-23% are typical, especially for general-domain experts
- **Do** evaluate synthetic hard negatives on a dev set per expert — they help general experts but can hurt domain experts with larger models
- **Avoid** training past data saturation; ~100K pairs per expert is often sufficient — more data yields diminishing returns
- **Avoid** complex merging methods as a first approach; start with linear interpolation and grid search over weights in 0.1 increments
- **Avoid** skipping general-domain experts; they are critical for preventing catastrophic forgetting in the merged model

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| Merged model worse than best single expert | Suboptimal merge weights | Re-run grid search with finer granularity (0.05 steps); ensure dev set covers both domain and general tasks |
| Domain performance drops after merging | General experts dominating | Increase domain expert weight (alpha); reduce general expert weights proportionally |
| Synthetic negatives hurt domain expert | Domain data already has hard negatives | Remove synthetic negatives for domain experts; only use them for general experts |
| GEPA prompt search finds no improvement | Search space too narrow or dev set too small | Increase seed prompt diversity; use at least 1,000 dev queries; try more generations |
| Training loss plateaus early | Data saturation reached | This is expected at ~100K samples — stop training and proceed to merging |
| OOM during bidirectional attention | Full attention on long sequences | Reduce max sequence length to 256 tokens for passages; use gradient checkpointing |

## Limitations

- **Model size dependency:** Synthetic hard negatives provide inconsistent benefits for domain experts on larger models (>2B parameters) — always validate on dev set
- **Merge weight search cost:** Grid search over 4 experts with 10 weight levels requires evaluating many combinations; use a small fast dev set
- **Decoder-only architecture:** The technique is validated on decoder-only LLMs with disabled causal masking; applicability to encoder-only models (BERT variants) is unverified
- **Domain data quality:** The framework amplifies existing data quality — if your domain pairs are noisy, expert training and merging will inherit that noise
- **Prompt sensitivity:** Optimized prompts are model-specific and expert-specific; they do not transfer between different backbones
- **Evaluation coverage:** Results are validated on biomedical + MTEB tasks; performance on other specialized domains (legal, financial) is extrapolated, not proven

## Reference

**Paper:** Khattab, S., Corbeil, J.-P., Koraş, O. A., Dada, A., & Friedrich, J. (2026). *Less Finetuning, Better Retrieval: Rethinking LLM Adaptation for Biomedical Retrievers via Synthetic Data and Model Merging.* arXiv:2602.04731v1.
[https://arxiv.org/abs/2602.04731v1](https://arxiv.org/abs/2602.04731v1)

**What to look for:** Section 3 for the full STM pipeline, Table 2 for merging method comparison (linear interpolation wins), Table 3 for ablation on synthetic negatives and prompt optimization, and Figure 2 for data saturation curves showing 100K is sufficient.