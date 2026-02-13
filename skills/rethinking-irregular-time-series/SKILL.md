---
name: "rethinking-irregular-time-series"
description: "Design and implement irregular time series classification pipelines for clinical/ICU data with high missing-value rates. Guides encoder selection (mTAND, Warpformer vs vanilla Transformer), LLM alignment strategy choice, and architecture trade-offs based on ICASSP 2026 findings. Triggers: 'classify irregular time series', 'ICU time series with missing data', 'irregular clinical time series', 'LLM for time series classification', 'handle missing values in time series', 'encoder for sparse clinical data'"
---

# Irregular Time Series Classification for Critical Care

This skill enables Claude to design, implement, and advise on classification pipelines for **irregularly-sampled time series data** -- the kind found in ICUs, wearable health monitors, and any domain where observations arrive at non-uniform intervals with high rates of missingness (often 85-97%). It applies findings from a systematic evaluation of LLM-based time series methods (Zheng et al., ICASSP 2026), which demonstrated that **encoder architecture matters far more than LLM alignment strategy** and that irregularity-aware encoders (mTAND) yield a 12.8% AUPRC gain over vanilla Transformers on clinical benchmarks.

## When to Use

- When the user needs to classify clinical time series with high missing-value rates (e.g., ICU mortality prediction, sepsis detection, arrhythmia classification)
- When the user asks how to handle irregular sampling intervals in time series fed to an LLM or deep learning model
- When the user is choosing between encoder architectures (Transformer, CNN, mTAND, Warpformer) for sparse temporal data
- When the user wants to integrate an LLM (GPT-2, LLaMA) with a time series encoder and needs to pick an alignment strategy
- When the user asks whether LLM-based time series methods are worth the computational cost for their clinical use case
- When the user is building a pipeline on PhysioNet 2012, MIMIC-III, or similar EHR datasets with >50% missingness

## Key Technique

The paper establishes that LLM-based time series classification has two independent design axes: **(1) the time series encoder** that converts raw irregular observations into embeddings, and **(2) the multimodal alignment strategy** that bridges those embeddings with a pretrained LLM's token space. The critical insight is that these axes are not equally important. Encoder choice drives a **31.7% AUPRC gap** (best vs. worst) on PhysioNet 2012, while alignment strategy accounts for only a **6.6% gap**. This means practitioners should invest most design effort into selecting an irregularity-aware encoder.

**Irregularity-aware encoders** like mTAND (Multi-time Attention Networks) use continuous-time positional encoding and multi-timescale attention to explicitly model when observations occur, rather than assuming uniform spacing. This prevents the "temporal bias" that vanilla Transformers and 1D CNNs introduce when applied to data with 85%+ missingness. On MIMIC-III (96.7% missing), mTAND-based pipelines achieve 41.8 AUPRC versus 18.6 for 1D CNN encoders -- a 2.2x improvement from encoder choice alone.

**The computational trade-off is stark**: LLM-based methods (mTAND+S2IP, Time-LLM, CALF) require 10-14x longer training than dedicated supervised models (Warpformer, standalone mTAND) while delivering only comparable or marginally better accuracy. In few-shot settings (10% training data), LLM methods actually underperform. The practical recommendation: use Warpformer or standalone mTAND for production clinical systems; reserve LLM integration for research settings or when you have abundant compute and data.

## Step-by-Step Workflow

1. **Profile the dataset's irregularity**: Compute the missing-value ratio per variable and the distribution of inter-observation intervals. If missingness exceeds 50%, an irregularity-aware encoder is mandatory. Print summary statistics: `missing_ratio = 1 - (observed_count / (num_samples * num_timesteps * num_variables))`.

2. **Select the encoder architecture** based on missingness and compute budget:
   - **>70% missing, production setting**: Use **Warpformer** (best performance-to-compute ratio; 13 min vs. 2+ hours on PhysioNet). It uses time-warping layers and doubly self-attentive blocks.
   - **>70% missing, research setting with GPU budget**: Use **mTAND** encoder, which employs learned continuous-time embeddings via `time2vec` and multi-head attention over irregular timestamps.
   - **<30% missing or regularly sampled**: A vanilla Transformer or patching-based encoder (PatchTST-style) is acceptable.
   - **Avoid** 1D CNN encoders for irregular data -- they assume uniform spacing and degrade catastrophically (AUPRC drops from 53.5 to 32.9 on PhysioNet).

3. **Prepare the input representation**: For each sample, construct a tuple `(values, timestamps, mask)` where `values` is shape `(T, D)`, `timestamps` is shape `(T,)` with actual observation times (not indices), and `mask` is a binary matrix indicating which values are observed. Do NOT impute missing values before encoding if using an irregularity-aware encoder -- it handles missingness natively.

4. **If integrating with an LLM**, select an alignment strategy:
   - **Best performing**: S2IP-style semantic prompting -- retrieve top-K semantic anchors from the LLM's embedding space that are closest to time series features, then prepend them as soft prompts. Achieves 2.9% AUPRC improvement over cross-attention.
   - **Most robust to missing data**: CALF-style cross-modal fine-tuning with global (non-patching) representation. Degrades gracefully as missingness increases because it avoids fixed-window patching that breaks with gaps.
   - **Avoid**: Graph-based alignment (FSCA) on sparse clinical data -- it underperforms due to insufficient structural signal in high-missingness regimes.

5. **Configure the LLM backbone**: Freeze most LLM layers and fine-tune only the alignment head and final classification layers. Use GPT-2 (124M parameters) as the default backbone -- larger models provide negligible gains on clinical classification tasks. Set `requires_grad=False` on transformer blocks; train only the projection layers.

6. **Implement the classification head**: Add a linear projection from the LLM's hidden dimension to the number of classes. For binary mortality prediction, use sigmoid activation with binary cross-entropy loss. For imbalanced datasets (typical in ICU -- ~14% positive rate on PhysioNet), use weighted BCE or focal loss.

7. **Set training hyperparameters**: Learning rate 1e-3 for encoder/alignment, 1e-4 for any unfrozen LLM layers. Batch size 16-64 depending on GPU memory. Train for 50-100 epochs with early stopping on validation AUPRC (not AUROC -- AUPRC is more informative for imbalanced clinical data).

8. **Evaluate with clinical metrics**: Report both AUROC and AUPRC. On imbalanced ICU datasets, AUPRC is the primary metric. Use 80/10/10 train/val/test splits. Run 5 seeds and report mean +/- std. Expected baselines: PhysioNet 2012 AUPRC ~54.0 (mTAND+S2IP) or ~54.8 (Warpformer); MIMIC-III AUPRC ~41.8 (mTAND+S2IP) or ~38.2 (Warpformer).

9. **Run the compute-vs-accuracy decision check**: If the LLM pipeline trains in >10x the time of a standalone Warpformer or mTAND model and delivers <5% AUPRC improvement, recommend switching to the supervised baseline for deployment.

## Concrete Examples

**Example 1: ICU Mortality Prediction Pipeline**

User: "I have PhysioNet 2012 data with 41 clinical variables and ~86% missing values. I want to predict in-hospital mortality. Should I use an LLM approach?"

Approach:
1. Profile the data: 85.7% missing ratio across 41 variables, 11,988 samples -- this is severely irregular.
2. Recommend Warpformer as primary model: achieves 54.8 AUPRC in 14 minutes of training.
3. If user insists on LLM integration, recommend mTAND encoder + S2IP alignment + frozen GPT-2 backbone.
4. Prepare data as `(values, timestamps, mask)` tuples without imputation.
5. Train with weighted BCE loss (positive class weight ~6.0 for 14% mortality rate).

Output:
```python
# Encoder selection for 85.7% missing data
# Option A: Warpformer (recommended for production)
from warpformer import Warpformer
model = Warpformer(
    n_features=41,
    n_classes=1,
    d_model=128,
    n_heads=4,
    n_warp_layers=2,
    n_attn_layers=2
)
# Expected: AUPRC ~54.8, training ~14 min on single GPU

# Option B: mTAND + LLM (research setting)
from mtand import mTANDEncoder
from s2ip import SemanticAligner
from transformers import GPT2Model

encoder = mTANDEncoder(
    n_features=41,
    d_model=128,
    n_heads=4,
    embed_time=64,    # continuous-time embedding dimension
    learn_emb=True    # learn time embeddings end-to-end
)
aligner = SemanticAligner(
    ts_dim=128,
    llm_dim=768,       # GPT-2 hidden size
    top_k=10           # number of semantic anchors
)
llm = GPT2Model.from_pretrained("gpt2")
for param in llm.parameters():
    param.requires_grad = False
# Expected: AUPRC ~54.0, training ~2.25 hours
```

**Example 2: Choosing an Encoder for a New Clinical Dataset**

User: "I have a sepsis prediction dataset from our hospital EHR with 60 variables and about 70% missing values. Which encoder should I use?"

Approach:
1. At 70% missing, irregularity-aware encoders are strongly recommended.
2. Check the inter-observation interval distribution -- if highly variable (mix of hourly vitals and daily labs), mTAND's continuous-time attention is ideal.
3. Avoid 1D CNN or decomposition-based encoders entirely.
4. If the dataset is small (<5,000 samples), avoid LLM methods -- they underperform in few-shot/low-data regimes. Use Warpformer or standalone mTAND.

Output:
```python
# Decision matrix for encoder selection
missing_ratio = 0.70
dataset_size = 3500  # samples
has_variable_intervals = True

if missing_ratio > 0.5 and has_variable_intervals:
    encoder = "mTAND"  # continuous-time attention for irregular intervals
elif missing_ratio > 0.5:
    encoder = "Warpformer"  # time-warping layers handle gaps
else:
    encoder = "Transformer"  # vanilla is fine for low missingness

if dataset_size < 5000:
    use_llm = False  # LLM methods underperform in low-data regimes
    # Warpformer few-shot AUPRC: 46.0 vs mTAND+S2IP: 36.2 on PhysioNet
```

**Example 3: Debugging Poor Performance on Irregular Data**

User: "I'm using Time-LLM with a 1D CNN encoder on MIMIC-III and getting AUPRC of only 18.6. What's wrong?"

Approach:
1. Identify the root cause: 1D CNN assumes regular spacing. MIMIC-III has 96.7% missing data.
2. The 1D CNN introduces temporal bias by treating observation indices as equidistant, destroying the actual time information.
3. Replace the encoder with mTAND while keeping the same alignment strategy.
4. Expected improvement: 18.6 -> ~41.8 AUPRC (a 2.2x gain from encoder swap alone).

Output:
```
Diagnosis: The 1D CNN encoder treats observations as uniformly spaced,
which is catastrophically wrong for MIMIC-III (96.7% missing).

Fix: Replace the encoder while keeping your alignment strategy intact.

Before (1D CNN):  AUPRC 18.6 on MIMIC-III
After  (mTAND):   AUPRC ~41.8 on MIMIC-III  (+23.2 points)

The encoder swap alone accounts for most of the possible performance
gain. Changing alignment strategy would add only ~2-3 points on top.
```

## Best Practices

- **Do**: Always pass raw timestamps (actual clock times or hours-since-admission) to irregularity-aware encoders. They need true temporal distances, not ordinal indices.
- **Do**: Use AUPRC as the primary metric for imbalanced clinical classification. AUROC can be misleadingly high (e.g., 86.6 AUROC with only 54.8 AUPRC on PhysioNet).
- **Do**: Start with Warpformer as a strong baseline before investing in LLM integration. It often matches or beats LLM methods at 1/10th the training cost.
- **Do**: Preserve the mask/missingness indicator as an explicit input feature. Irregularity-aware encoders use it to weight attention scores.
- **Avoid**: Imputing missing values with mean/forward-fill before feeding to mTAND or Warpformer -- these encoders handle missingness natively, and imputation destroys the temporal signal they rely on.
- **Avoid**: Using patching-based tokenization (fixed-window chunks) on data with >50% missingness. Patches will contain mostly empty slots, diluting the signal. Use global or per-observation encoding instead.

## Error Handling

- **AUPRC near random (equal to positive class prevalence)**: The encoder is likely not receiving proper timestamp information. Verify that timestamps are continuous floats (e.g., hours since admission), not integer indices. Check that the mask tensor correctly flags observed vs. missing values.
- **GPU out-of-memory with LLM backbone**: Freeze all LLM layers (`requires_grad=False`), reduce batch size to 8, and use gradient checkpointing. GPT-2 (124M) is sufficient -- do not use larger LLMs for clinical classification.
- **Training loss diverges**: Reduce learning rate for the encoder to 1e-4. mTAND's time embedding can be sensitive to initialization; try `embed_time=64` with `learn_emb=True`.
- **Performance collapses at high missingness (>90%)**: Switch from patching-based methods (Time-LLM, S2IP) to CALF-style global representation, which degrades more gracefully. At 90% missing, CALF retains ~60% of its clean-data performance while patching methods lose ~40%.

## Limitations

- **LLM overhead is rarely justified**: On PhysioNet and MIMIC-III, LLM-based methods match but do not consistently beat Warpformer, while costing 10-14x more compute. Recommend LLM integration only when you have (a) abundant compute, (b) >20K training samples, and (c) reason to believe cross-modal knowledge transfer from language will help.
- **Few-shot settings**: LLM methods significantly underperform supervised baselines with <10% of training data. If your labeled dataset is small, use Warpformer or mTAND directly.
- **Domain specificity**: These findings are validated on ICU/clinical data. Irregular time series from other domains (IoT sensor networks, astronomy) may have different missingness patterns where the encoder rankings shift.
- **Binary classification focus**: The paper evaluates binary tasks (mortality prediction, arrhythmia detection). Multi-class or regression tasks on irregular data may require different architectural considerations.
- **Static benchmarks**: Results are on PhysioNet 2012 and MIMIC-III. Real-world clinical deployment involves distribution shift, concept drift, and multi-site variability not captured in these benchmarks.

## Reference

Zheng, F., Wu, Y., Mascolo, C., & Dang, T. (2026). *Rethinking Large Language Models For Irregular Time Series Classification In Critical Care*. ICASSP 2026. [arXiv:2601.16516v2](https://arxiv.org/abs/2601.16516v2) | [Code](https://github.com/mHealthUnimelb/LLMTS)

Key takeaway: Read Section 4 (Results) Tables 2-4 for the encoder-vs-alignment comparison matrices, and Section 4.3 for the computational cost analysis that quantifies the 10-14x training overhead of LLM methods.