---
name: "persona-driven-data-synthesis-robust-multimodal"
description: "Generate synthetic training data using controllable persona-driven simulation and Chain-of-Thought reasoning augmentation. Use when the user mentions 'synthetic data generation', 'persona-based augmentation', 'data scarcity augmentation', 'cross-lingual data synthesis', 'CoT fine-tuning for classification', or 'controllable text/speech generation for low-resource domains'."
---

# Persona-Driven Data Synthesis with CoT Reasoning Augmentation

This skill enables Claude to design and implement **controllable synthetic data generation pipelines** based on the SynCog framework (Feng et al., 2026). The core idea: define virtual subject personas with demographic and behavioral style vectors, use an LLM to generate text conditioned on those personas, optionally synthesize paired speech via voice cloning, then produce Chain-of-Thought reasoning traces that make downstream classifiers interpretable. This approach is applicable wherever labeled data is scarce and you need diverse, realistic synthetic samples — clinical NLP, low-resource languages, specialized domain classification, or any multimodal task requiring transparent diagnostic reasoning.

## When to Use

- When the user needs to **augment a small labeled dataset** with synthetic examples for training a classifier or fine-tuning an LLM.
- When building a **multimodal pipeline** (text + audio/image) and real paired data is insufficient or expensive to collect.
- When the user wants synthetic data that is **controllable along specific attribute dimensions** (e.g., expertise level, writing style, demographic variation) rather than random augmentation.
- When the user asks for **Chain-of-Thought fine-tuning** where the model must produce explicit reasoning traces before a classification label.
- When working on **cross-lingual generalization** — training on synthetic data in one language and evaluating on real data in another.
- When the user needs **interpretable predictions** from a fine-tuned model, not just black-box labels.

## Key Technique

**Persona-Driven Controllable Synthesis.** Instead of naively prompting an LLM to "generate more training data," SynCog defines each synthetic sample through a structured persona specification: demographics (age, gender, education) plus a **multi-dimensional style vector**. Each dimension of the style vector (e.g., narrative length, syntactic complexity, fluency, domain-specific markers) is parameterized at ordinal levels (poor/normal/good) drawn from truncated Gaussian distributions whose means shift based on the target class label and demographic factors. This gives fine-grained control: impaired subjects get shorter narratives with more filler words; healthy subjects get more complex syntax. The generation prompt conditions the LLM on both the persona specification and the task stimulus (e.g., a picture to describe), producing text that is diverse yet clinically plausible.

**Chain-of-Thought Deduction Fine-Tuning.** After generating synthetic text (and optionally synthesizing speech via voice cloning), SynCog creates reasoning-augmented training examples. A strong teacher model (e.g., GPT-4o) is prompted with the synthetic sample *and* its ground-truth label to produce a "hindsight" CoT reasoning trace — an explicit diagnostic analysis explaining *why* this sample belongs to its class. These (input, reasoning, label) triples are used to fine-tune the target model with LoRA, teaching it to generate interpretable rationales before predicting. At inference, the model produces its own spontaneous reasoning without seeing labels.

**Why This Works Better.** The persona-conditioned approach prevents mode collapse (all synthetic samples sounding alike) and ensures the synthetic distribution covers the real data manifold. The CoT augmentation forces the model to learn diagnostic features rather than surface shortcuts, improving both accuracy and trustworthiness. Mixing synthetic data with real data yields the best results — synthetic alone outperforms real-only in low-resource settings, but the combination is optimal.

## Step-by-Step Workflow

1. **Analyze the target domain and define attribute dimensions.** Identify which characteristics vary meaningfully between classes in your data. For each characteristic, define 2-4 ordinal levels. Example: for medical text, dimensions might be vocabulary specificity (vague/normal/precise), sentence complexity (simple/moderate/complex), and coherence (fragmented/partial/fluent).

2. **Build the persona specification schema.** Create a structured format (JSON or dataclass) with fields for demographics and the style vector. Define the statistical distributions for each dimension conditioned on the target label. Use truncated Gaussians with class-dependent means:
   ```python
   # Style dimension mean shifts by class label
   mean = base_mean[label] - age_factor * age_adjustment + edu_factor * education_adjustment
   level = discretize(truncated_normal(mean, sigma), bins=["poor", "normal", "good"])
   ```

3. **Sample a balanced synthetic cohort.** Generate N persona specifications with balanced class distributions. Ensure demographic diversity by sampling age, gender, and other attributes from realistic marginal distributions. Store as a manifest (CSV/JSON) for reproducibility.

4. **Craft the synthesis prompt template.** Write a prompt that conditions generation on the full persona specification and the task stimulus. The prompt must explicitly instruct the LLM to adhere to the style vector constraints — do NOT rely on the model inferring style from a label alone. Include negative instructions (e.g., "Do not produce grammatically perfect text if fluency is set to 'poor'").

5. **Generate synthetic text samples.** Call the LLM (GPT-4o, Claude, or similar) once per persona. Validate outputs against the style vector by computing automated metrics (word count, type-token ratio, filler frequency) and reject/regenerate samples that deviate significantly.

6. **Optionally synthesize paired modalities.** For multimodal tasks, generate the paired signal. For speech: use a TTS/voice-cloning system with a reference voice library matched to persona demographics. For images: use a generation model conditioned on the text. Store aligned pairs.

7. **Generate CoT reasoning traces.** For each (synthetic_sample, label) pair, prompt a strong teacher model with a hindsight CoT template: "Given this sample and its known label [X], provide a step-by-step analysis explaining the evidence that supports this classification." Collect the reasoning traces.

8. **Assemble the reasoning-augmented dataset.** Format each training example as: `{input: <multimodal features>, reasoning: <CoT trace>, label: <class>}`. Mix synthetic examples with any available real labeled data (the paper finds ~60-70% synthetic / 30-40% real is effective).

9. **Fine-tune the target model with LoRA.** Train on the augmented dataset with the objective of generating the full (reasoning + label) sequence. Use LoRA on attention layers (rank 8-32), AdamW optimizer with cosine annealing, and 3-8 epochs. Evaluate with multiple inference rollouts and aggregate via majority voting or average voting score.

10. **Evaluate and iterate.** Measure Macro-F1 (not just accuracy, especially for imbalanced classes). Compare: real-only baseline, synthetic-only, and mixed. If synthetic-only underperforms the mixed approach, you have the right signal. Check cross-domain/cross-lingual transfer if applicable.

## Concrete Examples

**Example 1: Augmenting a Small Medical Text Classification Dataset**

User: "I have 200 labeled patient narratives (100 positive, 100 negative for condition X) and need more training data. The narratives vary in coherence and detail. Can you help me generate synthetic training data?"

Approach:
1. Define style dimensions: `narrative_length` (short/medium/long), `medical_vocabulary` (lay/mixed/technical), `coherence` (fragmented/partial/fluent), `symptom_specificity` (vague/moderate/precise).
2. Set class-conditioned distributions: positive cases skew toward fragmented coherence and vague specificity; negative cases skew toward fluent and precise.
3. Generate 500 persona specs per class with balanced demographics.
4. Use synthesis prompt:

```
You are simulating a patient describing their health experience.

Patient profile:
- Age: {age}, Gender: {gender}, Education: {education}
- Narrative length: {length_level} (short = 30-60 words, medium = 60-120, long = 120-200)
- Medical vocabulary: {vocab_level}
- Coherence: {coherence_level} (fragmented = incomplete sentences, topic shifts)
- Symptom specificity: {specificity_level} (vague = "I feel bad", precise = "sharp pain in lower left abdomen")

Task: Describe your recent health experience as this patient would.
Important: Match ALL style constraints exactly. If coherence is "fragmented", include incomplete thoughts and abrupt topic changes. Do NOT default to fluent prose.
```

5. Validate: check that mean word count for "short" is 30-60, filler word rate for "fragmented" is elevated.
6. Generate CoT traces: "This narrative shows fragmented sentence structure, vague symptom descriptions ('not feeling right'), and topic drift — consistent with condition X presentation patterns."
7. Fine-tune with mixed dataset (200 real + 1000 synthetic).

Output: A fine-tuned classifier that achieves ~5-15% Macro-F1 improvement over real-only training and produces interpretable reasoning with each prediction.

**Example 2: Cross-Lingual Synthetic Data for Low-Resource Language**

User: "I'm building a sentiment classifier for customer reviews in Thai, but I only have 50 labeled Thai examples. I have 500 labeled English examples. Can I use synthetic data to bridge the gap?"

Approach:
1. Define style dimensions relevant to sentiment: `emotional_intensity` (mild/moderate/strong), `formality` (casual/neutral/formal), `review_length` (brief/standard/detailed), `specificity` (generic/product-focused).
2. Generate 1000 English synthetic reviews with persona-conditioned prompts, ensuring class balance.
3. Generate 1000 Thai synthetic reviews using the same persona specs but a Thai-language synthesis prompt. The style vector constraints transfer across languages.
4. Generate bilingual CoT reasoning traces explaining sentiment indicators in each language.
5. Fine-tune a multilingual backbone (e.g., XLM-R or a multilingual LLM) on: 500 real English + 50 real Thai + 1000 synthetic English + 1000 synthetic Thai.
6. Evaluate on held-out Thai test set.

Output: The persona-conditioned approach ensures Thai synthetic data covers the same style space as real reviews, achieving stronger cross-lingual transfer than naive machine translation augmentation.

**Example 3: Building an Interpretable Document Classifier with CoT**

User: "My document classifier gets 82% accuracy but stakeholders don't trust it because it's a black box. Can I make it explain its decisions?"

Approach:
1. Take existing labeled training data. For each (document, label) pair, generate a CoT reasoning trace using a teacher model:

```
Analyze this document and explain step by step why it belongs to category "{label}".

Document: {document_text}

Provide your analysis as:
1. Key features observed: [list specific textual evidence]
2. Features consistent with {label}: [explain why each supports this class]
3. Features inconsistent with other classes: [explain differential reasoning]
4. Conclusion: Based on the above evidence, this document is classified as {label}.
```

2. Augment with synthetic persona-generated documents (optional, for additional diversity).
3. Fine-tune the model to produce the full reasoning chain before the label.
4. At inference, the model outputs: reasoning trace + predicted label. Stakeholders can read the reasoning.

Output: A model that produces outputs like:
```
Analysis: This document contains repeated references to "quarterly targets" and "revenue growth"
(financial terminology), uses formal register, and discusses forward-looking projections.
These features are characteristic of earnings reports. The absence of technical specifications
or product descriptions rules out the "product documentation" category.
Classification: Earnings Report
```

## Best Practices

- **Do:** Define style vector dimensions based on empirically observed variation in your real data. Analyze your existing samples first — compute statistics on length, vocabulary, complexity — and use those as the basis for your ordinal levels.
- **Do:** Validate synthetic samples against your style constraints quantitatively. Compute automated metrics (word count, type-token ratio, readability scores) and reject outliers. The LLM will sometimes ignore constraints.
- **Do:** Mix synthetic and real data rather than replacing real data entirely. The paper shows mixed training consistently outperforms synthetic-only or real-only.
- **Do:** Use multiple inference rollouts (4-8) with majority or average voting at test time. This reduces variance from CoT generation stochasticity.
- **Avoid:** Using the class label directly in the synthesis prompt (e.g., "Write a text typical of class X"). Instead, encode the class signal implicitly through the style vector. This prevents the model from learning label-word associations rather than genuine features.
- **Avoid:** Generating all synthetic samples with identical persona specs. The diversity comes from the persona sampling — if your distributions are too narrow, your synthetic data will be homogeneous and unhelpful.
- **Avoid:** Skipping the CoT distillation step if interpretability matters. Direct label fine-tuning is simpler but produces black-box predictions that are harder to debug and validate.

## Error Handling

- **Synthetic samples don't match style constraints:** Implement a validation loop. After generation, compute quantitative metrics for each style dimension. Re-prompt or resample personas that produce out-of-distribution outputs. Budget for ~10-15% rejection rate.
- **CoT reasoning traces are generic/unhelpful:** Improve the teacher prompt by including domain-specific feature checklists. Instead of "explain why," say "analyze the following specific features: [list features relevant to your domain]."
- **Synthetic data hurts performance when mixed with real:** Check for distribution shift. Run t-SNE or UMAP on embeddings of real vs. synthetic samples. If clusters don't overlap, your persona distributions are miscalibrated — adjust the means and variances of your style vector.
- **Cross-lingual transfer is weak:** Verify that the style vector dimensions are language-agnostic. Some dimensions (e.g., specific word patterns) may not transfer. Use dimensions grounded in universal linguistic properties (sentence length, lexical diversity, coherence).

## Limitations

- **Persona fidelity depends on the generator LLM.** The synthesis quality is bounded by the LLM's ability to follow style constraints. Smaller models may ignore fine-grained persona specifications.
- **CoT traces from teacher models can be post-hoc rationalizations** rather than genuine causal reasoning. The distilled reasoning is only as good as the teacher's analytical ability for your domain.
- **Cross-lingual transfer degrades for distant language pairs** and domains with culture-specific features. The paper shows 48.71% Macro-F1 on Mandarin (vs. 80.67% on English), indicating significant room for improvement.
- **Not a replacement for real data collection.** Synthetic augmentation boosts low-resource performance but plateaus. If you can collect more real data, that remains the gold standard.
- **Style vector design requires domain expertise.** Choosing the right dimensions and their distributions is a manual process that demands understanding of what actually varies across classes in your domain.

## Reference

Feng, R., Luo, Z., Wu, L., Wang, W., & Song, Y. (2026). *Cross-Linguistic Persona-Driven Data Synthesis for Robust Multimodal Cognitive Decline Detection.* arXiv:2602.07978v1. [https://arxiv.org/abs/2602.07978v1](https://arxiv.org/abs/2602.07978v1)

Key takeaway: Section 3 details the persona specification schema and truncated Gaussian sampling for style vectors; Section 4 covers the CoT distillation prompts and LoRA fine-tuning configuration; Tables 3-4 show the ablation proving mixed (real + synthetic) training is optimal.