---
name: "rethinking-generative-recommender-tokenizer"
description: "Build recommendation-native Semantic ID tokenizers using the ReSID framework (Field-Aware Masked Auto-Encoding + Globally Aligned Orthogonal Quantization) for generative sequential recommendation systems. Use when the user says: 'tokenize items for generative recommendation', 'build a semantic ID system for products', 'implement ReSID or FAMAE or GAOQ', 'create item embeddings for autoregressive recommendation', 'quantize item representations into discrete codes', 'design a sequential recommender tokenizer'."
---

# ReSID: Recommendation-Native Semantic ID Tokenization

This skill enables Claude to implement, configure, and debug **ReSID** (Recommendation-native Semantic ID) systems — a two-stage item tokenization framework for generative sequential recommendation. ReSID replaces the common pattern of borrowing LLM tokenization strategies (e.g., RQ-VAE on pretrained embeddings) with a purpose-built pipeline: **FAMAE** learns prediction-sufficient item representations from structured catalog features, and **GAOQ** quantizes them into compact, sequentially-predictable discrete codes. The resulting Semantic IDs serve as the vocabulary for a downstream autoregressive recommender (typically T5-style encoder-decoder), achieving 10%+ gains over SID baselines while reducing tokenization cost by up to 122x.

## When to Use

- When the user wants to build a **generative sequential recommender** that predicts next items as token sequences rather than softmax over the full item catalog
- When the user needs to convert a product/item catalog into **discrete Semantic IDs** for autoregressive generation
- When the user is working with **structured item features** (category hierarchies, store IDs, brand IDs) and wants embeddings that capture collaborative signal, not just semantic similarity
- When the user asks to implement or adapt the **ReSID, FAMAE, or GAOQ** components from the paper
- When the user wants to replace RQ-VAE or VQ-VAE based item tokenization with a method that jointly optimizes for sequential predictability
- When the user needs to set up a **three-stage training pipeline** (representation learning, quantization, autoregressive recommendation) on e-commerce or content interaction data

## Key Technique

### Why Not Just Use LLM-Style Tokenization?

Most Semantic ID recommenders borrow a two-step recipe from NLP: (1) obtain item embeddings from a foundation model or collaborative filtering, then (2) discretize with a generic vector quantizer (RQ-VAE, k-means tree). This is misaligned with recommendation because the embeddings optimize for semantic similarity rather than collaborative prediction, and the quantizer minimizes reconstruction error rather than the prefix-conditional entropy that autoregressive models actually need to minimize.

### FAMAE: Field-Aware Masked Auto-Encoding

FAMAE is a transformer-based masked autoencoder that operates on **field-level** structured features (item_id, store_id, category hierarchy levels). Each feature field gets its own embedding, and fields are fused (sum or concatenation). During training, random subsets of fields are masked and reconstructed, forcing the encoder to learn representations that are **predictive-sufficient** — they retain exactly the information needed to predict any item attribute from any other. The encoder uses 2 transformer layers with 4 attention heads (default hidden size 128), trained with AdamW (lr=0.001, weight_decay=1e-5) for up to 500 epochs with early stopping (patience=10). The key hyperparameter is the per-field masking ratio and the feature set itself, which must be chosen from your catalog schema.

### GAOQ: Globally Aligned Orthogonal Quantization

GAOQ takes the learned FAMAE representations and produces hierarchical discrete codes. It uses **balanced k-means** to partition the embedding space into codebooks at multiple levels (controlled by parameters `b1`, `b2`, `g2` — the code sizes per level). The "globally aligned" aspect ensures codebook vectors maintain orthogonality, preventing codebook collapse and ensuring that each quantization level captures genuinely different information. The "orthogonal" constraint reduces redundancy across levels so the resulting multi-level code sequence has minimal prefix-conditional uncertainty — exactly what makes autoregressive prediction easier. Code sizes are dataset-dependent (e.g., 32/64/64 for Musical Instruments, 128/256/256 for Books).

## Step-by-Step Workflow

1. **Prepare structured item features**: Extract your item catalog into a Parquet file with columns for each feature field (item_id, category IDs at each hierarchy level, store/brand ID). Create an `item_feature_explain.json` mapping each field name to its vocabulary size and column index. All feature indices must start at 1 (0 is reserved for padding).

2. **Prepare interaction sequences**: Organize user interaction histories into train/valid/test splits using a leave-one-out protocol. Each record contains a user ID, a sequence of interacted item IDs (left-padded to `max_len=32`), and a target item.

3. **Configure and train FAMAE**: Set up `famae.yaml` with your feature list, embedding dimension (128 default), transformer layers (2), heads (4), FFN dimension (512), masking strategy, and training hyperparameters. Run training: `python main.py --config ./config/famae.yaml --device cuda:0`. Monitor `mask_item_id_recall_item_id` on validation set — this measures whether the encoder learns representations from which masked item IDs can be recovered.

4. **Determine codebook sizes for GAOQ**: Choose `b1`, `b2`, `g2` values based on your catalog size. Smaller catalogs (< 10K items) work with 32/64/64; larger catalogs (100K+) need 128/256/256 or higher. The product `b1 * b2 * g2` should exceed your item count to ensure unique assignments.

5. **Configure and train GAOQ**: Set up `gaoq.yaml` pointing to the pretrained FAMAE model path, your chosen code sizes, `feature_fusion: concat`, and `use_balancedkmeans: true`. Run: `python main.py --config ./config/gaoq.yaml --device cuda:0`. This stage produces the Semantic ID mapping (item_id -> [code_level_1, code_level_2, code_level_3]).

6. **Remap Semantic IDs for the generative model**: The GAOQ output assigns codes per level. Before feeding to the T5 model, remap codes so each level's vocabulary is offset to prevent collisions (e.g., level-1 codes 0..b1-1, level-2 codes b1..b1+b2-1, level-3 codes b1+b2..b1+b2+g2-1). The codebase handles this via `remap_semantic_ids_to_unique()`.

7. **Configure and train the autoregressive recommender**: Set up `t5.yaml` with encoder/decoder layers (4 each), d_model=128, d_kv=32, d_ff=512, num_heads=4, beam_sizes=[50,50,50], cosine LR scheduler with 5-epoch warmup. The model consumes interaction history as a sequence of Semantic ID tuples and generates the next item's SID tuple autoregressively. Run: `python main.py --config ./config/t5.yaml --device cuda:0`.

8. **Evaluate with HR@k and NDCG@k**: The pipeline evaluates using Hit Rate and NDCG at standard cutoffs. Beam search at each SID level (beam widths configurable via `beam_sizes`) generates candidate items. Compare against both sequential baselines (SASRec, BERT4Rec) and SID-based baselines (TIGER, LETTER, EAGER).

9. **Tune and iterate**: Ablate by (a) removing individual feature fields from FAMAE, (b) adjusting codebook sizes in GAOQ, (c) toggling balanced k-means vs. standard k-means, (d) varying beam widths. The pipeline script `run_pipelines.py` automates end-to-end runs per dataset.

## Concrete Examples

**Example 1: Setting up ReSID on an e-commerce product catalog**

User: "I have an Amazon product dataset with item_id, store_id, and three levels of category hierarchy. I want to build a generative recommender using Semantic IDs."

Approach:
1. Clone the ReSID repo: `git clone https://github.com/FuCongResearchSquad/ReSID.git`
2. Format the catalog as Parquet with columns: `item_id, store_id, cate1_id, cate2_id, cate3_id`
3. Create `item_feature_explain.json`:
```json
{
  "item_id": {"num": 5432, "col_idx": 0},
  "store_id": {"num": 312, "col_idx": 1},
  "cate1_id": {"num": 28, "col_idx": 2},
  "cate2_id": {"num": 156, "col_idx": 3},
  "cate3_id": {"num": 891, "col_idx": 4}
}
```
4. Run the full pipeline:
```bash
python run_pipelines.py --dataset My_Products --device cuda:0
```
5. Evaluate output: the pipeline logs HR@5, HR@10, NDCG@5, NDCG@10 on the test set.

Output:
```
[FAMAE] Epoch 127 | val mask_item_id_recall: 0.4823
[GAOQ]  Codebook assignment complete | b1=32, b2=64, g2=64
[T5]    Test HR@10: 0.0712 | NDCG@10: 0.0398
```

**Example 2: Adapting FAMAE for a custom feature schema**

User: "My items have brand_id, color_id, price_bucket, and item_id. How do I configure FAMAE?"

Approach:
1. Update `famae.yaml` feature list:
```yaml
feature:
  - item_id
  - brand_id
  - color_id
  - price_bucket
feature_emb_dim: 128
feature_fusion: sum
per_feature_loss_weights:
  item_id: 1.0
  brand_id: 1.0
  color_id: 0.5
  price_bucket: 0.5
```
2. Weight the loss per field — give higher weight to fields with stronger collaborative signal (item_id, brand_id) and lower weight to weaker signals (color, price bucket).
3. Create `item_feature_explain.json` with vocabulary sizes for each field.
4. Train FAMAE and verify that masked reconstruction accuracy improves, especially for `item_id` reconstruction from other fields.

Output: A `model.pth` checkpoint containing field-aware embeddings that capture cross-field predictive structure.

**Example 3: Choosing GAOQ codebook sizes for different catalog scales**

User: "How do I pick b1, b2, g2 for my 50K item catalog?"

Approach:
1. The product `b1 * b2 * g2` must exceed the item count for unique SID assignment. For 50K items: 32 * 64 * 64 = 131,072 > 50,000 works.
2. Reference the paper's presets:
```
Musical_Instruments (7K items):   b1=32,  b2=64,  g2=64
Video_Games (22K items):          b1=64,  b2=128, g2=128
Books (400K+ items):              b1=128, b2=256, g2=256
```
3. For 50K items, start with `b1=64, b2=128, g2=128` (product = 1,048,576, generous headroom).
4. If HR/NDCG is low, try reducing codebook sizes to force more sharing (compression helps generalization on sparse data). If codebook utilization is low (many empty codes), reduce sizes.

## Best Practices

- **Do** include `item_id` as a feature field in FAMAE — it captures collaborative filtering signal that category-only features miss. The masked reconstruction of `item_id` from other fields is the core mechanism that aligns representations with recommendation objectives.
- **Do** use `use_balancedkmeans: true` in GAOQ — balanced k-means prevents codebook collapse where a few codes absorb most items, which destroys the information-theoretic properties the method relies on.
- **Do** use `feature_fusion: sum` for FAMAE and `feature_fusion: concat` for GAOQ — FAMAE benefits from parameter sharing across fields during masking, while GAOQ needs the full concatenated representation for effective quantization.
- **Do** tune `beam_sizes` for the T5 stage — wider beams (e.g., [50,50,50]) improve recall at inference cost. For production, narrow beams ([20,20,20]) trade recall for latency.
- **Avoid** using pretrained LLM embeddings as input to GAOQ — the entire point of ReSID is that recommendation-native representations (from FAMAE) outperform semantic embeddings for this task. Mixing paradigms negates the benefit.
- **Avoid** skipping the FAMAE stage and feeding raw feature embeddings directly to GAOQ — without the masked reconstruction pretraining, the embeddings lack the cross-field predictive structure that makes GAOQ's quantization effective.

## Error Handling

- **Codebook collapse in GAOQ**: If most items map to a small subset of codes, verify `use_balancedkmeans: true` and reduce codebook sizes. Check that FAMAE embeddings have sufficient variance (low-variance embeddings cause degenerate clustering).
- **FAMAE reconstruction loss plateaus high**: Increase `num_layers` or `hidden_size`. Verify that `item_feature_explain.json` vocabulary sizes match the actual data. Check that feature indices start at 1, not 0.
- **T5 generates invalid SID combinations**: Ensure Semantic IDs were remapped with `remap_semantic_ids_to_unique()` before training. Invalid combinations typically mean level-2/3 codes overlap with level-1 codes.
- **Out-of-memory during GAOQ**: GAOQ processes all item embeddings at once for k-means. For catalogs >1M items, reduce `feature_emb_dim` or run on a machine with more RAM. The GAOQ stage itself is not GPU-intensive.
- **Poor HR/NDCG despite good FAMAE reconstruction**: The bottleneck is likely codebook sizing. Try smaller codes (more compression forces better structure) or verify that `b1*b2*g2` is not excessively larger than the item count.

## Limitations

- **Requires structured item features**: ReSID's FAMAE component operates on discrete field features (IDs, categories). If your items only have free-text descriptions or images, you must first convert them into discrete fields or use a different approach.
- **Cold-start items**: New items without interaction history can still receive Semantic IDs via their features, but the T5 recommender cannot predict them until the codebook is retrained to include them.
- **Three-stage training complexity**: The FAMAE -> GAOQ -> T5 pipeline requires sequential training with checkpoint passing. Changes to FAMAE require retraining GAOQ and T5 downstream.
- **Static Semantic IDs**: Once assigned, SIDs do not update as interaction patterns evolve. Periodic retraining of the full pipeline is needed for catalogs with shifting popularity distributions.
- **Evaluated primarily on Amazon review datasets**: The 10 evaluation datasets are all Amazon product categories. Performance on very different domains (music streaming, news, social media) is not empirically validated.

## Reference

- **Paper**: [Rethinking Generative Recommender Tokenizer: Recsys-Native Encoding and Semantic Quantization Beyond LLMs](https://arxiv.org/abs/2602.02338v1) — Focus on Sections 3 (FAMAE formulation), 4 (GAOQ theory and algorithm), and 5 (experimental results across 10 Amazon datasets).
- **Code**: [https://github.com/FuCongResearchSquad/ReSID](https://github.com/FuCongResearchSquad/ReSID) — PyTorch implementation with YAML-driven configuration, supports distributed training.