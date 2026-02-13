---
name: "curp-codebook-based-continuous-user"
description: "Design codebook-based user representation systems for personalized LLM generation. Use when asked to 'build a user personalization system', 'create user embeddings for generation', 'design a codebook for user modeling', 'personalize LLM outputs per user', 'represent user preferences as discrete prototypes', or 'build plug-and-play user profiles'."
---

# CURP: Codebook-Based Continuous User Representation for Personalized Generation

This skill enables Claude to design and implement **codebook-based user representation systems** that personalize LLM outputs based on user behavioral history. The core technique from the CURP paper encodes user histories into dense embeddings, quantizes them via a Product Quantization codebook into discrete prototypes, then projects those prototypes into an LLM's embedding space as soft tokens that steer generation. This achieves high personalization quality with only ~20M trainable parameters (~0.2% of the LLM) and no per-user fine-tuning.

## When to Use

- When the user asks to build a personalized text generation system (reviews, social posts, emails, headlines) that adapts to individual writing style
- When designing a user modeling pipeline that must scale to hundreds of thousands of users without per-user model copies
- When the user wants to represent user preferences as interpretable, discrete prototype vectors rather than opaque embeddings
- When building a plug-and-play personalization module that can attach to different LLM backbones without full fine-tuning
- When the user needs cross-task user representations — a single codebook that transfers across review writing, Q&A, paraphrasing, and headline generation
- When implementing a VQ-VAE-style quantization layer for behavioral data (not just images)

## Key Technique

**Product-Quantized User Codebook.** CURP avoids the two extremes of personalization: prompt-stuffing (which wastes context and introduces noise) and per-user fine-tuning (which is prohibitively expensive). Instead, it builds a shared **discrete prototype codebook** of ~1,000 entries. Each user's behavioral history is encoded by a bidirectional encoder (Contriever, 768-dim), then decomposed into L=4 subspaces via Product Quantization. Each subspace is independently matched to its nearest codebook entry via balanced k-means. The result is a compact tuple of 4 codebook indices per history item — a "user prototype" that captures topic, content type, sentiment, and stance as independent semantic dimensions.

**Two-Stage Training.** Stage 1 (Prototype Codebook Construction) trains the codebook on large-scale data (~24M histories from ~150k users) using three losses: reconstruction loss (straight-through estimator), diversity loss (smooth pairwise distance penalty), and usage loss (variance + coverage + entropy to prevent codebook collapse). Stage 2 (Prototype Behavior Aligning) freezes the codebook and trains a lightweight 2-layer MLP (768 → 3584 → 3584) to project quantized embeddings into the LLM's token embedding space. These projected embeddings are prepended to the task query as soft tokens, and the LLM is trained with standard causal language modeling. Only the MLP and a thin adapter are trained — the codebook and base LLM stay frozen.

**Why This Matters in Practice.** The codebook acts as a universal "personality vocabulary." At inference, a new user's 2-16 historical items are encoded, quantized to codebook indices, projected, and prepended — no gradient updates needed. The same codebook transfers across tasks and even across LLM architectures (tested with Qwen, LLaMA, RoBERTa combinations). Subspaces are interpretable: e.g., subspaces 2-3 capture topic identity while subspace 4 captures tone/stance.

## Step-by-Step Workflow

1. **Collect user behavioral data.** Gather 2-16 historical items per user (posts, reviews, Q&A pairs, tweets). Each item should be a text snippet representing authentic user output. Store as `{user_id: [history_1, ..., history_J]}`.

2. **Encode histories into dense embeddings.** Use a pretrained bidirectional encoder (Contriever or similar sentence encoder) to embed each history item independently into a 768-dimensional vector. Apply mean pooling over tokens. Do NOT concatenate histories into one long string — encode each item separately to avoid cross-contamination noise.

3. **Build the Product Quantization codebook.** Partition each 768-dim embedding into L=4 subspaces of 192 dimensions each. Run balanced k-means (not standard k-means) on each subspace independently across your full dataset to produce K=1,000 prototype centroids per subspace. Balanced k-means enforces roughly equal cluster sizes, which is critical — random or standard k-means causes codebook collapse where most entries go unused.

4. **Train the codebook with three losses.** Optimize: (a) reconstruction loss `||e_q - sg(e)||^2` using a straight-through gradient estimator, (b) diversity loss `-τ * tanh(min_{i≠j} ||c_i - c_j|| / τ)` to keep prototypes separated, (c) usage loss combining variance of assignment counts, coverage ratio, and entropy of the assignment distribution. Use weights λ1=1.0, λ2=0.15, λ3=1.0.

5. **Freeze the codebook and build the projection MLP.** Create a 2-layer MLP that maps from the quantized embedding dimension (768) to the target LLM's hidden dimension (e.g., 3584 for Qwen-2.5-7B). This MLP is the only component trained in Stage 2.

6. **Train the alignment stage with causal LM loss.** For each training example: encode the user's J history items, quantize each through the frozen codebook, project via the MLP, and prepend the resulting J soft tokens to the tokenized task query. Train the MLP (and optionally a LoRA adapter on the LLM) using standard next-token prediction loss on the target output.

7. **Design task-specific prompt templates.** Wrap the generation query with a template that includes a placeholder for the projected user embeddings. Example: `"User style embedding: {prototype_tokens} Based on the user's style, {task_instruction} {query_text}"`. The `{prototype_tokens}` are replaced by the MLP-projected codebook vectors at the embedding level, not as text.

8. **Run inference with plug-and-play personalization.** At inference: encode new user's histories → quantize to codebook indices → project via MLP → prepend to query embeddings → generate. No fine-tuning or gradient computation needed per user.

9. **Validate with personalization metrics.** Evaluate using ROUGE-1/2/L and BLEU against ground-truth user outputs, plus cosine similarity of RoBERTa embeddings between generated and reference text. Compare against baselines: zero-shot, ICL (raw history in prompt), RAG (BM25-retrieved history), and per-user LoRA.

10. **Analyze codebook interpretability.** Inspect which codebook indices activate for different user clusters. Verify that subspaces capture independent semantic dimensions (topic, content type, tone, stance) by checking that shared-topic users share indices in subspaces 2-3 but differ in subspaces 1 and 4.

## Concrete Examples

**Example 1: Personalized Product Review Generator**

User: "I want to build a system that generates product reviews matching each user's writing style, given their past 5 reviews."

Approach:
1. Structure data as `{user_id, product_info, past_reviews[], target_review}`
2. Encode each of the 5 past reviews independently with Contriever → 5 vectors of dim 768
3. Quantize each vector: split into 4 subspaces of 192-dim, find nearest codebook entry in each → 5 tuples of 4 indices each
4. Project quantized vectors through trained MLP → 5 soft tokens of dim 3584
5. Construct input: `[soft_token_1, ..., soft_token_5, "Write a review for {product_name}: {product_description}"]`
6. Generate with Qwen-2.5-7B, attending over both prototype tokens and query

Output architecture:
```
User History (5 reviews)
    ↓ Contriever encoder (frozen)
5 × 768-dim embeddings
    ↓ Product Quantization (L=4, K=1000, frozen)
5 × [idx_sub1, idx_sub2, idx_sub3, idx_sub4]
    ↓ Codebook lookup + MLP projection (trainable, ~20M params)
5 × 3584-dim soft tokens
    ↓ Prepend to query embeddings
LLM (Qwen-2.5-7B, frozen or LoRA)
    ↓
Personalized review text
```

**Example 2: Style-Consistent Social Media Post Generator**

User: "Build me a tweet paraphrasing system that rewrites tweets in each user's personal voice, using their tweet history."

Approach:
1. Collect 8 historical tweets per user as style references
2. Encode each tweet → 768-dim embedding via Contriever
3. Quantize and project to LLM space using the shared codebook
4. Use prompt template:
   ```
   User style embedding: {8 projected prototype tokens}
   Based on the user's style embedding provided above,
   please paraphrase the following tweet without any
   explanation before or after it.
   Original tweet: {input_tweet}
   ```
5. The codebook captures: subspaces 1-2 → topic preferences, subspace 3 → formality level, subspace 4 → sentiment tendency
6. Output is a paraphrased tweet that preserves the original meaning but adopts the target user's characteristic vocabulary, tone, and structure

Output comparison:
```
Input tweet: "The new policy will have significant economic implications"

Zero-shot output: "The new policy will significantly impact the economy"
ICL output (8 tweets in context): "This policy's gonna hit wallets hard"  (overfits to casual examples)
CURP output (user=formal analyst): "The proposed policy carries substantial economic ramifications"
CURP output (user=casual commenter): "lol this policy is gonna wreck the economy fr"
```

**Example 3: Cross-Task Codebook Transfer**

User: "I already trained a codebook on Reddit Q&A data. Can I reuse it for news headline generation?"

Approach:
1. Keep the frozen codebook from Reddit Q&A training (Stage 1 output)
2. Train ONLY a new MLP projection layer for the headline generation task (Stage 2)
3. Encode user's news-reading history through same Contriever encoder
4. Quantize using the Reddit-trained codebook — the prototype vocabulary is task-agnostic
5. Project and prepend to headline generation query
6. The paper shows this cross-task transfer maintains 500+ distinct codebook entries active and competitive performance vs. task-specific codebooks

```python
# Pseudocode for cross-task transfer
frozen_codebook = load_codebook("reddit_qa_codebook.pt")  # From Stage 1
new_mlp = MLP(768, 3584, 3584)  # Fresh projection for new task

for user_histories, query, target in headline_dataloader:
    embeddings = contriever.encode(user_histories)        # [B, J, 768]
    quantized = frozen_codebook.quantize(embeddings)       # [B, J, 768]
    projected = new_mlp(quantized)                         # [B, J, 3584]
    input_embeds = concat(projected, llm.embed(query))     # [B, J+T, 3584]
    loss = llm.forward(input_embeds, labels=target)
    loss.backward()  # Gradients flow only through new_mlp
```

## Best Practices

**Do:**
- Use balanced k-means initialization for the codebook — standard k-means or random initialization leads to near-zero codebook utilization (most entries never activate)
- Encode each history item independently rather than concatenating them into one long prompt; independent encoding prevents noise cross-contamination and scales linearly
- Include the usage loss (variance + coverage + entropy) during codebook training to prevent codebook collapse, where a few entries dominate
- Use 4 PQ subspaces with 1,000 entries as your default configuration; the paper's ablations show this outperforms both smaller (500) and more granular (PQ8) alternatives
- Verify codebook health by checking that 70%+ of entries are used across your dataset after training

**Avoid:**
- Do not skip the diversity loss — without it, codebook entries cluster together and lose discriminative power
- Do not fine-tune the base LLM fully; the point of the architecture is parameter efficiency through the frozen-codebook + thin-MLP design
- Do not use fewer than 2 history items per user at inference; performance degrades significantly below this threshold, though it remains stable between 2-16 items
- Do not concatenate all user histories into the LLM's text prompt as a fallback — this wastes context window, introduces noise, and the paper shows it underperforms the codebook approach on all metrics

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| Codebook collapse | <30% of entries used; most users map to same indices | Switch to balanced k-means init; increase λ3 (usage loss weight); verify diversity loss is active |
| Poor personalization | Generated text is generic, ignoring user style | Check that MLP projection is training (gradients flowing); verify history items are meaningful (not empty or boilerplate) |
| Cross-task degradation | Transferred codebook performs much worse than task-specific | Retrain the MLP projection layer for the new task; the codebook stays frozen but the projection must adapt |
| Embedding dimension mismatch | Runtime error when switching LLM backbone | Adjust MLP output dimension to match new LLM's hidden size (e.g., 4096 for LLaMA vs 3584 for Qwen) |
| History length sensitivity | Performance drops with more history items | Cap at 8-12 items during training; the quantization naturally compresses, but attention cost over soft tokens still scales linearly |

## Limitations

- **Requires training infrastructure.** Despite being lightweight (~20M params), the two-stage pipeline still requires GPU training for the codebook and MLP. This is not a zero-shot or prompt-only technique.
- **Cold-start problem.** Users with fewer than 2 historical items cannot be effectively represented. The codebook needs behavioral signal to quantize.
- **English-centric validation.** The paper evaluates on English datasets (Reddit, news, tweets, reviews). Effectiveness on multilingual or code-generation personalization is unvalidated.
- **Fixed codebook granularity.** The 4-subspace, 1000-entry design is a hyperparameter choice. Domains with very fine-grained user differences (e.g., technical writing style) may need larger codebooks, but the paper does not explore beyond K=1000.
- **Not suitable for real-time preference shifts.** The codebook captures stable user traits, not session-level intent. For in-session adaptation, combine with retrieval or context-based methods.

## Reference

**Paper:** [CURP: Codebook-based Continuous User Representation for Personalized Generation with LLMs](https://arxiv.org/abs/2602.00742v1) — Focus on Section 3 (method architecture), Section 4.3 (ablation on codebook design choices), and Section 4.5 (interpretability analysis showing subspace semantic decomposition).

**Code:** [github.com/RaidonWong/CURP_code](https://github.com/RaidonWong/CURP_code) — Two-stage pipeline with `encode.py` → `step1.py` (codebook) → `step2.py` (alignment) → `inference.py`.