---
name: "llm-not-all-you"
description: "Systematic model selection advisor for classification tasks — chooses between classical ML, zero-shot LLMs/VLMs, and fine-tuned foundation models based on data modality, dataset size, and task complexity. Use when: 'should I use an LLM or traditional ML for this?', 'help me pick the right model for classification', 'is fine-tuning worth it for my dataset?', 'build a classification pipeline for medical/tabular data', 'compare ML vs LLM approaches', 'which model architecture for my image/text classification task?'"
---

# Model Selection for Classification: ML vs. Foundation Models

This skill enables Claude to serve as a rigorous model-selection advisor for classification tasks. Drawing from a systematic benchmark comparing classical ML (Logistic Regression, LightGBM, ResNet-50), zero-shot LLM/VLM prompting (Gemini 2.5), and parameter-efficient fine-tuning (LoRA-adapted Gemma), it provides evidence-based recommendations on which modeling approach to use given the user's data modality (text/tabular vs. image), dataset size, and task complexity (binary vs. multiclass). The core finding: LLMs are not universally superior — classical ML remains dominant for structured data, zero-shot VLMs are competitive only for multiclass image tasks, and minimal PEFT fine-tuning is actively harmful without sufficient epochs and data.

## When to Use This Skill

- When a user asks whether to use an LLM, a fine-tuned model, or classical ML for a classification problem
- When building a classification pipeline for medical, clinical, or tabular data and the user needs architecture guidance
- When a user is considering LoRA/QLoRA fine-tuning and wants to know if it's worth the effort for their dataset size
- When comparing zero-shot prompting vs. supervised training for image or text classification
- When a user has a small-to-medium dataset (hundreds to low thousands of samples) and wants the most reliable approach
- When a user is tempted to default to an LLM for a task where simpler models would outperform it
- When evaluating whether a foundation model adds value over a well-engineered classical baseline

## Key Technique: Evidence-Based Model Selection Matrix

The paper establishes a clear performance hierarchy through controlled experiments across four medical datasets (Diabetes binary text, Mental Health 12-class text, Skin Cancer binary image, Respiratory 5-class image). All experiments used identical train/test splits and aligned metrics (Accuracy, weighted F1, AUC-ROC), making the comparisons directly valid.

**The core decision matrix is:**

1. **Structured/tabular text data → Classical ML wins decisively.** LightGBM achieved 0.998 accuracy on diabetes classification vs. 0.422 for Gemini 2.5 zero-shot. On 12-class mental health classification, LightGBM hit 0.987 accuracy vs. 0.281 for Gemini. The performance gap is not marginal — it is catastrophic for the LLM. Classical models with proper preprocessing (median imputation, StandardScaler, one-hot encoding) dominate because tabular features are inherently well-suited to gradient-boosted trees and linear models.

2. **Image classification → depends on task complexity.** For binary image tasks (skin cancer), a CNN baseline (0.828 accuracy) still beat Gemini 2.5 (0.589). But for multiclass image tasks (5-class respiratory disease), Gemini 2.5 (0.488 accuracy, 0.464 F1) matched ResNet-50 (0.481 accuracy, 0.466 F1). Zero-shot VLMs become competitive when the task has enough classes that the vision model's broad pretraining knowledge provides genuine signal.

3. **LoRA fine-tuning with minimal epochs is actively harmful.** Gemma 1B LoRA at 1 epoch achieved 0.015 accuracy on the 12-class task (worse than random). Even at 10 epochs on the diabetes task, LoRA reached only 0.835 — below a simple Logistic Regression (0.773) combined with proper feature engineering. PEFT requires substantial epoch counts and careful hyperparameter tuning to provide any value; naive application destroys performance.

## Step-by-Step Workflow

1. **Characterize the input data.** Determine the modality (structured tabular, unstructured text, image), the number of classes (binary vs. multiclass), the dataset size, and whether labels are available. This is the most important step — the data profile drives the entire model selection.

2. **Apply the modality-first decision rule.** If the data is structured/tabular with numeric and categorical features, recommend classical ML (LightGBM for best performance, Logistic Regression for interpretability). If the data is image-based with a multiclass problem and limited labeled data, consider zero-shot VLM as a viable baseline. If the data is image-based binary classification, recommend a CNN/ResNet approach.

3. **Preprocess according to the chosen path.** For classical ML on tabular data: impute physiologically implausible values (e.g., replace zeros with column medians for clinical features), apply StandardScaler to numeric features, one-hot encode categoricals, and use stratified train/test splits preserving class distribution. For images: resize consistently (224x224 for ResNet, 128x128 for lighter CNNs), normalize to [0,1] RGB.

4. **Implement the recommended model class.** For classical ML, start with LightGBM with class-weighted loss for imbalanced data. For image tasks, use a pretrained ResNet-50 with transfer learning. For zero-shot VLM baselines, craft task-specific prompts (see examples below).

5. **Evaluate with aligned metrics.** Use Accuracy, weighted F1-score (critical for multiclass with imbalance), and AUC-ROC (for binary tasks). Apply a 0.5 threshold for probability-based outputs. Always report all three metrics — accuracy alone can be misleading with class imbalance.

6. **If the user insists on fine-tuning a foundation model, set guardrails.** Use at minimum 10 training epochs (not 1). Use LoRA rank r=4 (r=8 showed degradation in this study). Monitor for catastrophic forgetting. Always compare against a classical ML baseline trained on the same split. If the fine-tuned model underperforms the baseline, discard it.

7. **Design zero-shot prompts correctly when using LLMs/VLMs.** For binary tasks: assign an expert persona, request a single probability score, threshold at 0.5. For multiclass tasks: use N-shot prompting with one example per class. For image tasks: reference domain-specific diagnostic criteria in the prompt (e.g., ABCDE criteria for dermoscopy).

8. **Report the recommendation with justification.** Present the chosen approach, the expected performance range based on comparable benchmarks, the key preprocessing steps, and explicit warnings about approaches that would likely fail for this data profile.

## Concrete Examples

**Example 1: Tabular clinical data classification**

User: "I have a CSV with 3,000 patient records containing blood pressure, glucose, BMI, and other numeric clinical features. I need to predict diabetes risk. Should I use GPT-4 or a traditional model?"

Approach:
1. Identify: structured tabular data, binary classification, ~3,000 samples, all numeric features
2. Apply decision rule: structured tabular → classical ML dominates
3. Recommend LightGBM with proper preprocessing

Output:
```
Recommendation: Use LightGBM (or XGBoost), not an LLM.

Justification: On a nearly identical diabetes dataset (2,768 records, 8 clinical
indicators), LightGBM achieved 0.998 accuracy and 0.997 F1, while Gemini 2.5
zero-shot scored 0.422 accuracy — worse than a coin flip. LLMs fundamentally
struggle with structured numeric features because they process them as text tokens,
losing the mathematical relationships that tree-based models exploit directly.

Pipeline:
1. Impute implausible zeros (Glucose=0, BMI=0) with column medians
2. Apply StandardScaler to all numeric features
3. Stratified 80/20 split preserving class distribution
4. Train LightGBM with class_weight='balanced' if classes are imbalanced
5. Evaluate with Accuracy, weighted F1, and AUC-ROC

Expected performance: >0.95 accuracy with clean preprocessing on this data profile.
```

**Example 2: Multiclass image classification with limited labels**

User: "I have 800 chest X-ray images across 5 disease categories (160 per class). What's the best approach — fine-tune a vision model or use a VLM?"

Approach:
1. Identify: image data, multiclass (5 classes), small dataset (~800 images), balanced classes
2. Apply decision rule: multiclass image + limited data → zero-shot VLM is competitive with ResNet
3. Recommend both approaches as a two-track strategy

Output:
```
Recommendation: Run both ResNet-50 transfer learning AND a zero-shot VLM, then compare.

Justification: On a comparable 5-class respiratory dataset (800 images, 160/class),
ResNet-50 achieved 0.481 accuracy / 0.466 F1 and Gemini 2.5 zero-shot matched it
at 0.488 accuracy / 0.464 F1. With this few images per class, neither approach
dominates — the VLM's pretraining knowledge compensates for lack of labeled data.

Do NOT use LoRA fine-tuning here. Gemma 3 4B LoRA scored only 0.369 accuracy on
this exact setup — worse than both alternatives.

Track A — ResNet-50:
1. Resize images to 224x224, normalize to [0,1]
2. Load pretrained ResNet-50, replace final FC layer for 5 classes
3. Fine-tune with class-weighted cross-entropy, augmentation (flip, rotate, crop)
4. Train for 20-50 epochs with early stopping

Track B — Zero-shot VLM:
1. Craft a prompt: "You are a radiologist. Classify this chest X-ray as one of:
   [Class1, Class2, Class3, Class4, Class5]. Respond with only the class name."
2. Send each image with the prompt via API
3. Parse the text response and map to class labels

Compare both tracks on the same held-out test set using weighted F1.
```

**Example 3: User wants to fine-tune an LLM for a text classification task**

User: "I want to LoRA fine-tune Llama 3 on my 2,000-sample survey dataset to classify mental health conditions into 12 categories. How should I set it up?"

Approach:
1. Identify: structured survey data (categorical + numeric), multiclass (12 classes), 2,000 samples
2. Apply decision rule: structured text/tabular → classical ML; fine-tuning is risky
3. Warn against LoRA, recommend classical ML, provide fine-tuning guardrails if they proceed anyway

Output:
```
Strong recommendation: Use LightGBM instead of LoRA fine-tuning.

On a nearly identical setup (1,998 survey responses, 12 depression categories),
LightGBM scored 0.987 accuracy / 0.982 F1. LoRA-tuned Gemma at 1 epoch scored
0.015 accuracy — essentially random on 12 classes. The data is fundamentally
tabular (Age, SocialMedia_Hours, Sleep_Hours, categorical fields) and tree-based
models handle this natively.

Classical ML pipeline:
1. StandardScaler on numeric columns (Age, Hours, Scores)
2. One-hot encode categoricals (Gender, Education, Employment, Symptoms)
3. Train LightGBM with class_weight='balanced' and stratified CV
4. Expected: >0.97 accuracy

If you still want to fine-tune (e.g., for research purposes):
- Use at least 10 epochs, not 1 (1-epoch LoRA is catastrophically bad)
- Set LoRA rank r=4, avoid r=8 (showed degradation)
- Apply class-weighted loss — 12 categories with imbalance will collapse to
  majority-class prediction without it
- Always benchmark against LightGBM on the same split before deploying
- Budget for significant compute — and expect it to still underperform classical ML
```

## Best Practices

**Do:**
- Always establish a classical ML baseline first, even if the user intends to use an LLM — this provides a performance floor and often turns out to be the ceiling too
- Use stratified splits for all train/test partitions to preserve class distributions, especially with imbalanced medical data
- Apply domain-specific preprocessing (e.g., replacing physiologically impossible zeros in clinical data with column medians rather than dropping rows)
- Use weighted F1-score as the primary metric for multiclass tasks — accuracy alone hides poor performance on minority classes
- When using zero-shot VLMs for image tasks, include domain-specific diagnostic criteria in the prompt (e.g., ABCDE criteria for skin lesions) rather than generic classification instructions

**Avoid:**
- Defaulting to LLMs for structured/tabular data — this is the single most common mistake; tree-based models outperform LLMs by 50+ percentage points on tabular classification
- Using 1-epoch LoRA fine-tuning and expecting it to work — minimal PEFT is worse than no fine-tuning (zero-shot) in every case studied
- Increasing LoRA rank beyond r=4 without evidence — r=8 showed degradation rather than improvement in this benchmark
- Treating zero-shot LLM performance on text/tabular tasks as a meaningful baseline — scores below random chance (0.422 on binary, 0.281 on 12-class) indicate fundamental architectural mismatch, not a tuning problem
- Skipping StandardScaler normalization for classical ML on clinical data — unnormalized features with different scales severely degrade Logistic Regression performance

## Error Handling

- **LLM returns malformed output:** When using zero-shot prompting, the model may return explanatory text instead of a class label. Implement strict output parsing with regex extraction and a fallback to "unknown" class. Re-prompt once with a more constrained instruction (e.g., "Respond with ONLY the class name, nothing else").
- **Fine-tuned model collapses to single class:** This indicates the model has not learned from fine-tuning and is defaulting to majority prediction. Check that class-weighted loss is applied, increase epochs to at least 10, and verify the training data is shuffled.
- **Classical ML underperforms expectations:** Investigate preprocessing — implausible zeros (e.g., BMI=0) in clinical data, missing one-hot encoding for categoricals, or data leakage from non-stratified splits are the most common causes.
- **VLM gives inconsistent image classifications:** Zero-shot VLMs have inherent variance across runs. Average predictions over 3 API calls with temperature=0 and take the majority vote to stabilize output.
- **Dataset too small for any supervised approach:** Below ~200 total samples, none of these approaches are reliable. Recommend data augmentation for images, or synthetic oversampling (SMOTE) for tabular data before training.

## Limitations

- The benchmark uses only four medical datasets with 800-2,768 samples each. Results may differ on datasets with 50,000+ samples where fine-tuning has more signal to learn from.
- LoRA fine-tuning was tested with very minimal configurations (1 epoch baseline, r=4). More aggressive fine-tuning strategies (full fine-tuning, larger LoRA ranks with more epochs, QLoRA with 4-bit quantization) were not explored and may perform better.
- Only Gemini 2.5 was tested as the zero-shot LLM/VLM. GPT-4V, Claude, and other frontier models may show different performance profiles, particularly on structured reasoning over tabular data.
- The decision matrix is strongest for medical/clinical classification. For natural language understanding tasks (sentiment analysis, NER, summarization), LLMs likely retain their advantage — this skill applies specifically to classification with structured features or medical images.
- The study does not address few-shot in-context learning with many examples, retrieval-augmented generation, or chain-of-thought prompting as alternatives to zero-shot, which may narrow the gap on tabular tasks.

## Reference

**Paper:** Raval, Pandit, & Upadhyay. "LLM is Not All You Need: A Systematic Evaluation of ML vs. Foundation Models for Text and Image Based Medical Classification." AAIML'26. [arXiv:2601.16549](https://arxiv.org/abs/2601.16549v1)

**Key takeaway to extract:** Tables II and III contain the complete performance comparison across all four datasets and model classes. The decision matrix (Section V) provides the authors' framework for model selection by data modality and task complexity.