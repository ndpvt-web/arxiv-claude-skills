---
name: "less-enough-synthesizing-diverse"
description: "Synthesize maximally diverse training data for LLM post-training using Feature Activation Coverage (FAC). Identifies missing feature coverage in seed datasets via sparse autoencoders, then generates targeted synthetic samples to fill gaps. Use when: 'improve my training data diversity', 'synthesize diverse fine-tuning data', 'find gaps in my dataset coverage', 'generate data to cover missing features', 'optimize post-training data efficiently', 'create diverse instruction data with fewer samples'."
---

# Less is Enough: Diverse Data Synthesis via Feature Activation Coverage

This skill enables Claude to help users build maximally diverse post-training datasets for LLMs using Feature Activation Coverage (FAC) -- a metric that measures data diversity in the interpretable feature space of sparse autoencoders rather than relying on shallow text-based diversity metrics. Instead of accumulating hundreds of thousands of samples, this approach identifies exactly which semantic features a seed dataset is missing, then synthesizes targeted samples to fill those gaps, achieving comparable or superior downstream performance with orders of magnitude fewer samples (e.g., 2,000 vs. 300,000).

## When to Use

- When the user wants to **build or improve a fine-tuning dataset** and needs to maximize diversity without blindly scaling up sample count
- When the user asks to **audit an existing dataset for coverage gaps** -- identifying what topics, behaviors, or capabilities are underrepresented
- When the user wants to **generate synthetic training data** that targets specific missing capabilities rather than random paraphrases
- When the user is working on **instruction following, toxicity detection, reward modeling, or behavior steering** tasks and wants data-efficient post-training
- When the user asks to **transfer dataset design insights across model families** (e.g., features identified on LLaMA applied to Mistral or Qwen)
- When the user needs to **prioritize which new training examples to create** given a limited annotation or generation budget
- When the user asks about **data-centric optimization** of LLMs or measuring dataset quality beyond simple deduplication

## Key Technique

**Feature Activation Coverage (FAC)** reframes dataset diversity from a text-similarity problem into a feature-space coverage problem. Traditional metrics (n-gram diversity, embedding distance, topic counts) capture surface-level linguistic variation but miss the internal features LLMs actually use for downstream tasks. FAC instead extracts a sparse autoencoder (SAE) from a target LLM's intermediate representations, decomposing each sample's embedding into thousands of interpretable features. Each feature corresponds to a recognizable concept (e.g., "formal apology language", "mathematical reasoning step", "code error explanation"). Coverage is then: `FAC = |features_active_in_your_data| / |features_active_in_reference_data|`, using an activation threshold to filter noise.

**FAC Synthesis** operationalizes this metric into a data generation pipeline. Given a seed dataset, it: (1) identifies which task-relevant features are already covered, (2) computes the set of missing features by comparing against a reference distribution, (3) for each missing feature, constructs contrastive pairs -- a positive example strongly activating that feature and a negative example that does not -- to give a generator model a concrete understanding of the target feature, and (4) uses these pairs in a prompt template to synthesize new samples that reliably activate the missing feature, verified by re-running the SAE. This contrastive-pair approach is the key insight: rather than hoping random generation covers gaps, you show the generator exactly what the feature looks like vs. what it doesn't, producing samples that pass activation-threshold verification at high rates.

**Cross-model transferability** is a practical bonus: the interpretable feature spaces of LLaMA, Mistral, and Qwen share substantial overlap, so features identified on one model family can guide data synthesis for another. This means you don't need a separate SAE for every target model -- a single analysis can inform dataset construction across deployments.

## Step-by-Step Workflow

1. **Define the target task and collect a seed dataset.** Gather an initial set of examples for the task (instruction-following pairs, toxicity-labeled text, preference pairs, etc.). This can be small -- even 500-2,000 samples. Clearly specify what downstream capability you're optimizing for.

2. **Select or train a sparse autoencoder on the target LLM.** Use a pre-trained SAE (available for LLaMA, Mistral, Qwen families) or train one on the target model's intermediate layer activations. The SAE maps each d-dimensional embedding to a k-dimensional sparse feature vector (k >> d, typically 16K-65K features) using: `z = ReLU(xW_enc)`, trained with reconstruction loss + L1 sparsity penalty.

3. **Extract feature activations for the seed dataset.** Run every seed example through the target LLM, capture the intermediate representation at the chosen layer, and pass it through the SAE encoder. Record which features activate above threshold delta (recommended range: delta=1.0 to 2.0) for each sample. Build a binary activation matrix: `A[i][j] = 1 if g_j(x_i) > delta`.

4. **Identify task-relevant features.** From the full set of SAE features, filter to those relevant to your task. Use the SAE decoder weights to generate human-readable feature descriptions, then classify relevance (manually for small feature sets, or via an LLM judge for large ones). This produces the task-relevant feature set F.

5. **Compute missing features.** Compare the features activated by your seed dataset against the full set of task-relevant features from a reference distribution (anchor data). Missing features: `F_miss = F(reference) \ F(seed)`. Each missing feature represents a specific capability gap in your dataset.

6. **Construct contrastive pairs for each missing feature.** For each feature i in F_miss: generate candidate texts using the feature's description as a prompt, score them by SAE activation strength, then select one high-activation example (x+, where g_i(x+) >= delta) and one low-activation example (x-). This pair concretely demonstrates what the feature looks like present vs. absent.

7. **Synthesize feature-covering samples using contrastive prompts.** Feed each contrastive pair into a generator LLM with a template: "Here is an example that exhibits [feature description]: [x+]. Here is one that does not: [x-]. Generate a new example that clearly exhibits this feature for [task context]." Generate m candidates per feature (m=3-5 works well).

8. **Verify and filter generated samples.** Run each candidate back through the SAE. Keep only samples where `g_i(candidate) > delta`. This verification step ensures the synthetic data actually covers the intended feature rather than just matching surface text.

9. **Aggregate and deduplicate the final dataset.** Combine seed data with verified synthetic samples: `S_final = S_seed ∪ S_gen`. Remove near-duplicates. Recompute FAC on the combined set to confirm coverage improvement.

10. **Fine-tune and evaluate.** Train the target LLM on S_final. Compare downstream metrics against the seed-only baseline and against naive scaling (adding random synthetic data). FAC correlates strongly with downstream performance (Pearson r=0.95), so FAC gains should translate to task gains.

## Concrete Examples

**Example 1: Improving instruction-following data diversity**

```
User: I have 1,500 instruction-following examples but my fine-tuned model
struggles with certain types of requests. How can I improve my dataset
without just adding more random examples?

Approach:
1. Load the seed dataset and extract SAE features from LLaMA-3.1-8B
   at layer 16 using a pre-trained SAE with k=32768 features.
2. Set activation threshold delta=1.5. Compute activation matrix across
   all 1,500 samples.
3. Use a reference set (e.g., MAGPIE or Alpaca-GPT4) to identify the
   full task-relevant feature space. Suppose 2,400 features are relevant.
4. Find that seed data covers 1,680 of 2,400 features -> FAC = 0.70.
   The 720 missing features include clusters like: "multi-step planning",
   "code debugging with error messages", "formal letter formatting".
5. For each missing feature cluster, construct contrastive pairs:
   - Positive: a sample strongly activating "multi-step planning"
   - Negative: a similar-length sample without planning structure
6. Generate 3 candidates per missing feature, verify via SAE, keep those
   passing threshold. Yield ~1,800 verified synthetic samples.
7. Final dataset: 3,300 samples with FAC = 0.94 (up from 0.70).

Output:
- Dataset grows from 1,500 to 3,300 samples (2.2x, not 200x)
- FAC improves from 0.70 to 0.94
- Expected downstream gain: ~30-40% improvement on AlpacaEval LC win rate
  based on the FAC-performance correlation (r=0.95)
```

**Example 2: Auditing a toxicity detection dataset for blind spots**

```
User: Our toxicity classifier misses certain categories of toxic content.
Can you help me figure out what's missing in our training data?

Approach:
1. Load the toxicity training set (e.g., 5,000 labeled examples).
2. Extract SAE features from the target classifier's backbone at an
   intermediate layer. Use delta=1.0 for sensitivity.
3. Compute feature activations against a comprehensive toxicity reference
   (e.g., ToxicChat full set). Identify task-relevant features via
   LLM-assisted annotation of SAE feature descriptions.
4. Compare: suppose reference covers 800 toxicity-relevant features,
   training set covers 540 -> FAC = 0.675.
5. Inspect the 260 missing features. The SAE descriptions reveal gaps:
   - "Passive-aggressive workplace language" (feature #4821)
   - "Coded discriminatory language using euphemisms" (feature #12044)
   - "Sarcastic dismissal of identity" (feature #7293)
6. For each gap, generate contrastive pairs and synthesize 5 candidates.
   After SAE verification, add ~900 targeted examples.
7. Retrain classifier. Expected gain: +15-24% AUPRC on underrepresented
   toxicity categories.

Output:
- Gap report: 260 missing features organized by semantic cluster
- Priority ranking: features sorted by frequency in reference data
  (most impactful gaps first)
- Synthetic supplement: 900 verified examples covering 230/260 gaps
- Updated FAC: 0.675 -> 0.92
```

**Example 3: Cross-model dataset transfer**

```
User: I built a reward model training set optimized for LLaMA. Now I need
to fine-tune Mistral on the same task. Do I need to redo everything?

Approach:
1. No. Research shows LLaMA, Mistral, and Qwen share interpretable
   feature spaces with high overlap.
2. Load the LLaMA-optimized dataset. Extract SAE features using the
   Mistral SAE instead.
3. Compute FAC against Mistral's feature space. Many LLaMA-covered
   features transfer directly, but some Mistral-specific features
   may be uncovered.
4. Identify the delta -- typically 5-15% of features need Mistral-
   specific synthesis.
5. Run contrastive synthesis for the Mistral-specific gaps only.
6. Combine: LLaMA-optimized base + Mistral-specific supplement.

Output:
- Reuse ~85-95% of existing dataset without modification
- Synthesize only the model-specific gap (50-200 targeted samples)
- Achieve comparable FAC on Mistral without full re-optimization
- Expected: +10-13% accuracy on RewardBench vs. naive transfer
```

## Best Practices

- **Do:** Set the activation threshold delta in the range [1.0, 2.0]. Too low (0.0-0.5) captures noise; too high (>2.0) misses genuine but moderate feature activations. Validate by checking that known-relevant samples activate above your chosen threshold.

- **Do:** Use contrastive pairs (positive + negative examples) when prompting the generator. This is the single most important factor for synthesis quality -- it gives the model a concrete discrimination boundary rather than a vague description.

- **Do:** Always verify synthetic samples by re-running SAE activation checks. Expect 60-80% of candidates to pass verification. Budget generation accordingly (generate 3-5x your target count).

- **Do:** Prioritize missing features by their frequency in the reference distribution. Features that appear often in high-quality reference data but are absent from your seed represent the highest-impact gaps.

- **Avoid:** Using text-based diversity metrics (n-gram diversity, embedding cosine distance) as your primary optimization target. These correlate weakly with downstream performance compared to FAC (r=0.95 for FAC vs. r<0.5 for text metrics).

- **Avoid:** Over-generating per feature. Diminishing returns set in at 3-5 verified samples per missing feature. Beyond that, you're adding redundancy, not coverage.

## Error Handling

- **SAE not available for target model:** Train a lightweight SAE on 50K-100K samples from the model's intermediate representations. Use TopK activation with k=64 and dictionary size 32K-65K. Training is computationally cheap relative to LLM pre-training.

- **Too many missing features (>500):** Cluster missing features by semantic similarity (using their decoder weight vectors), prioritize clusters by reference frequency, and synthesize for the top clusters first. Iterate in batches.

- **Low verification pass rate (<40%):** The feature description may be ambiguous. Improve contrastive pairs by selecting more extreme positive/negative examples (wider activation gap). Also try increasing generation temperature to 0.8-1.0 for more diverse candidates.

- **FAC improves but downstream performance doesn't:** Check that your task-relevant feature selection (Step 4) is accurate. Irrelevant features dilute the signal. Re-annotate with stricter relevance criteria or use a held-out validation set to calibrate feature relevance.

- **Reference/anchor data unavailable:** Use the target model's own generations as a proxy reference. Sample diverse prompts, generate completions, extract features, and use that distribution as your coverage target. This is less precise but still outperforms text-metric approaches.

## Limitations

- **Requires SAE access or training:** The approach depends on sparse autoencoders for the target model family. Pre-trained SAEs exist for popular models (LLaMA, Mistral, Qwen) but not all architectures. Training a new SAE requires GPU resources and intermediate-layer access.

- **Feature relevance annotation is a bottleneck:** Identifying which of thousands of SAE features are task-relevant requires either manual inspection or LLM-assisted classification, both of which introduce noise. Misclassified features waste synthesis budget.

- **Not a substitute for data quality:** FAC optimizes diversity, not individual sample quality. Combine with quality filtering (reward model scoring, human review) for best results. Diverse garbage is still garbage.

- **Computational overhead per iteration:** Each synthesis-verify cycle requires forward passes through both the generator LLM and the target model + SAE. For large feature gaps (>300 missing features), this can require significant GPU time.

- **Closed-model limitations:** If the target deployment model is API-only (no access to intermediate representations), you must use a proxy open-source model from the same family for SAE analysis and hope features transfer. This works well within families but degrades across very different architectures.

## Reference

**Paper:** [Less is Enough: Synthesizing Diverse Data in Feature Space of LLMs](https://arxiv.org/abs/2602.10388v1) -- Li et al., 2026. Focus on Section 3 (FAC metric definition), Section 4 (FAC Synthesis algorithm with contrastive pair construction), and Section 5.4 (cross-model feature transfer results).