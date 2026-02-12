---
name: "persona-driven-data-synthesis-robust-multimodal"
description: "Synthesize diverse persona-driven training data to overcome scarce labeled datasets, especially for multimodal classification tasks. Uses controllable demographic/behavioral profiles and Chain-of-Thought reasoning to generate high-quality synthetic samples with transparent diagnostic logic. Triggers: 'generate synthetic training data with personas', 'augment scarce clinical dataset', 'persona-driven data synthesis', 'synthesize diverse samples for classification', 'cross-lingual data augmentation with profiles', 'create synthetic dataset with chain-of-thought rationales'"
---

# Persona-Driven Data Synthesis for Robust Multimodal Classification

This skill teaches Claude to apply the SynCog framework (Feng et al., 2026) for generating synthetic training data using controllable persona profiles and Chain-of-Thought (CoT) reasoning extraction. The core idea: when labeled data is scarce, you construct virtual subjects with parameterized demographic and behavioral attributes, generate realistic multimodal samples conditioned on those personas, then distill explicit reasoning chains that connect observable features to classification labels. This produces both more data and more interpretable models, with demonstrated cross-lingual transfer.

## When to Use

- When the user has a small labeled dataset (dozens to low hundreds of samples) and needs to augment it for training a classifier
- When building a multimodal pipeline (text + audio, text + image, etc.) and real paired data is insufficient
- When the user wants synthetic data that varies along controlled demographic or behavioral dimensions rather than random noise
- When the user needs training data across multiple languages but only has annotations in one
- When the user wants to add Chain-of-Thought rationales to an existing classification dataset for interpretable fine-tuning
- When building a data generation pipeline that produces diverse, clinically or domain-plausible synthetic samples
- When the user asks how to simulate realistic subject profiles for any domain with scarce expert-labeled data (medical, legal, educational)

## Key Technique

**Persona-Conditioned Controllable Synthesis.** Instead of generating synthetic data randomly or via simple paraphrasing, SynCog constructs explicit persona profiles defined by two components: (1) demographic attributes (age, gender, education, etc.) and (2) a multi-dimensional behavioral style vector. Each style dimension is discretized into ordinal levels (e.g., poor/normal/good) and sampled from truncated Gaussians conditioned on the target class label, modulated by demographic factors. For example, a persona labeled "cognitively impaired" would have lower narrative length, more filler words, and reduced syntactic complexity. The LLM then generates text/descriptions conditioned jointly on a stimulus (image, prompt) and the full persona specification, producing samples that reflect realistic within-class variation rather than homogeneous outputs.

**Chain-of-Thought Deduction Fine-Tuning.** After generating synthetic samples, SynCog adds a reasoning layer. In Stage 1, a teacher LLM is given each sample plus its ground-truth label and asked to reverse-engineer the diagnostic reasoning: "Given that this subject has label X, articulate the specific linguistic and behavioral evidence supporting that conclusion." This produces explicit rationale chains. In Stage 2, the target model is fine-tuned on tuples of (input, rationale, label) using LoRA, so at inference time it generates transparent reasoning before making predictions rather than producing opaque classifications.

**Cross-Lingual Zero-Shot Transfer.** Because persona profiles are language-agnostic abstractions, the same demographic-plus-style-vector schema can drive generation in any target language without translated annotations. The framework generates language-specific corpora (e.g., SYN-EN for English, SYN-ZH for Mandarin) from shared persona distributions, achieving meaningful cross-lingual generalization even when trained exclusively on synthetic data.

## Step-by-Step Workflow

1. **Define the classification task and identify the data gap.** Determine the target classes (e.g., impaired vs. healthy, positive vs. negative), count available labeled samples per class, and quantify the shortfall. Aim for a synthetic augmentation ratio of 2-3x the real dataset as a starting point (diminishing returns beyond 5x per the paper's ablation).

2. **Design the persona schema with demographic and style dimensions.** Create a structured profile template with two sections: (a) demographic fields relevant to your domain (age, gender, education, occupation, language) and (b) a behavioral style vector with 3-7 dimensions that differentiate your classes. Each style dimension should be discretized into ordinal levels (e.g., 1=low, 2=medium, 3=high). Define which dimensions correlate with which class labels.

   ```json
   {
     "demographics": {"age": 72, "gender": "female", "education": "high_school", "language": "en"},
     "style_vector": {
       "narrative_length": 1,
       "syntactic_complexity": 1,
       "domain_vocabulary_usage": 2,
       "fluency": 1,
       "expression_clarity": 1
     },
     "label": "impaired"
   }
   ```

3. **Build the persona sampling distribution.** For each class label, define baseline means for each style dimension, then add sensitivity coefficients for demographic modifiers. Sample personas from: `v_k ~ N(mu_k(label) - alpha_k * f(age) + beta_k * g(education), sigma_k^2)`, then discretize to ordinal levels. This produces realistic covariation between demographics and behavioral traits rather than uniform random profiles.

4. **Construct the generation prompt template.** Write a prompt that conditions the LLM on both the task stimulus (an image description, a scenario, a question) and the full persona specification. The prompt must translate ordinal style levels into concrete behavioral instructions.

   ```
   You are simulating a {demographics.age}-year-old {demographics.gender} with
   {demographics.education} education describing the Cookie Theft picture.

   Behavioral profile:
   - Narrative length: {SHORT: 60-80 words | MEDIUM: 100-140 words | LONG: 160-200 words}
   - Syntactic complexity: {LOW: simple sentences only | MEDIUM: some subordinate clauses | HIGH: complex structures}
   - Filler words: {HIGH: >= 4 fillers like "um", "uh" | MEDIUM: 1-3 fillers | LOW: <= 1 filler}
   - Domain vocabulary: {LOW: vague terms, avoids specifics | MEDIUM: some precise terms | HIGH: rich vocabulary}
   - Clarity: {LOW: >= 4 vague terms, tangential | MEDIUM: mostly on-topic | HIGH: clear and focused}

   Generate the description this person would produce. Do not add any meta-commentary.
   ```

5. **Generate synthetic text samples in batch.** For each sampled persona, call the LLM with the populated prompt template. Generate one sample per persona. For multimodal tasks, pair the text with a synthesized or cloned secondary modality (e.g., use a voice cloning tool like IndexTTS2 with age/gender-matched reference audio, or pair with corresponding image features).

6. **Extract Chain-of-Thought rationales from a teacher model.** For each synthetic sample + label pair, prompt a strong LLM to produce hindsight reasoning:

   ```
   Given the following transcript from a cognitive assessment, the subject has been
   diagnosed as: {label}.

   Transcript: "{generated_text}"

   As a clinical assessment specialist, articulate the specific linguistic evidence
   in this transcript that supports the diagnosis. Reference concrete features:
   word count, syntactic patterns, filler frequency, vocabulary specificity,
   coherence, and any domain-relevant markers. Structure your reasoning step by step.
   ```

7. **Assemble the reasoning-augmented training dataset.** Combine each sample into a training tuple: `(input_features, chain_of_thought_rationale, label)`. Format for your fine-tuning framework (e.g., instruction-tuning JSON for LoRA).

   ```json
   {
     "input": "<audio_or_text_features>",
     "reasoning": "The subject uses only 67 words with 5 filler words ('um', 'uh'), simple sentence structures, and no spatial references to describe the scene. The high filler-to-word ratio (7.5%) and absence of specific vocabulary suggest word-finding difficulties consistent with cognitive impairment.",
     "label": "impaired"
   }
   ```

8. **Fine-tune the target model with CoT supervision.** Train on the full reasoning sequence (not just the label) using LoRA on attention projection layers. Recommended config from the paper: rank 16, alpha 32, dropout 0.1, 5 epochs, cosine annealing, effective batch size 8.

9. **Validate with stratified evaluation.** Evaluate using Macro-F1 (not accuracy, which hides class imbalance). Run multiple inference rollouts (paper uses 8) and report both average score and best-of-N to characterize variance. Compare: real-only baseline, synthetic-only, and mixed real+synthetic.

10. **Iterate on the persona schema.** Inspect failure cases. If the model confuses two classes, examine whether your style vector dimensions sufficiently differentiate them. Add or refine dimensions, adjust sampling distributions, and regenerate.

## Concrete Examples

**Example 1: Augmenting a small medical NLP dataset**

User: "I have 120 labeled patient interview transcripts (60 MCI, 60 healthy controls) and need more training data for a classifier. Can you help me generate synthetic samples?"

Approach:
1. Define persona schema with demographics (age 55-90, gender, education level) and style dimensions (narrative_length, syntactic_complexity, filler_frequency, vocabulary_specificity, coherence).
2. Set class-conditioned baselines: MCI personas get lower means for complexity/vocabulary/coherence, higher means for filler_frequency.
3. Sample 180 MCI personas and 180 HC personas (3x augmentation).
4. Generate transcripts using prompt template conditioned on each persona + the interview task description.
5. Extract CoT rationales for each generated transcript.
6. Combine with real data for a 360-sample augmented training set.

Output:
```python
# persona_sampler.py
import numpy as np
from dataclasses import dataclass

@dataclass
class PersonaConfig:
    class_baselines: dict  # {label: {dim: (mu, sigma)}}
    age_sensitivity: dict  # {dim: alpha}
    edu_sensitivity: dict  # {dim: beta}
    ordinal_bins: list     # e.g., [1.5, 2.5] for 3-level discretization

def sample_persona(config, label, age, education_years):
    style = {}
    for dim, (mu, sigma) in config.class_baselines[label].items():
        raw = np.random.normal(
            mu - config.age_sensitivity[dim] * (age / 100)
               + config.edu_sensitivity[dim] * (education_years / 20),
            sigma
        )
        level = int(np.digitize(raw, config.ordinal_bins)) + 1
        style[dim] = np.clip(level, 1, 3)
    return style

# Example configuration for MCI detection
config = PersonaConfig(
    class_baselines={
        "MCI": {"narrative_length": (1.3, 0.4), "complexity": (1.2, 0.3),
                "filler_freq": (2.5, 0.5), "vocab_specificity": (1.4, 0.4),
                "coherence": (1.5, 0.4)},
        "HC":  {"narrative_length": (2.5, 0.4), "complexity": (2.4, 0.3),
                "filler_freq": (1.3, 0.4), "vocab_specificity": (2.5, 0.4),
                "coherence": (2.6, 0.3)}
    },
    age_sensitivity={"narrative_length": 0.3, "complexity": 0.2,
                     "filler_freq": -0.2, "vocab_specificity": 0.1,
                     "coherence": 0.2},
    edu_sensitivity={"narrative_length": 0.2, "complexity": 0.3,
                     "filler_freq": -0.1, "vocab_specificity": 0.3,
                     "coherence": 0.2},
    ordinal_bins=[1.5, 2.5]
)
```

**Example 2: Cross-lingual data generation for sentiment analysis**

User: "I have 200 labeled English product reviews but need to train a multilingual model that also works on Spanish and Japanese. Can you help generate synthetic training data in those languages?"

Approach:
1. Define a language-agnostic persona schema: reviewer demographics (age_bracket, tech_savviness) and style vector (review_length, emotional_intensity, specificity, formality).
2. Sample personas from the same class-conditioned distributions for all three languages.
3. For each persona + language, generate reviews conditioned on the persona profile with language-specific prompt templates.
4. Extract CoT rationales explaining why each review maps to its sentiment label.
5. Train on combined English-real + Spanish-synthetic + Japanese-synthetic dataset.

Output:
```
# Prompt template (Spanish variant)
Eres un(a) revisor(a) de {demographics.age_bracket} con nivel de
conocimiento tecnologico {demographics.tech_savviness}.

Escribe una resena de producto con sentimiento {label} sobre {product}.

Perfil de escritura:
- Longitud: {SHORT: 1-2 oraciones | MEDIUM: 3-5 oraciones | LONG: 6+ oraciones}
- Intensidad emocional: {LOW: neutral | MEDIUM: moderada | HIGH: muy expresiva}
- Especificidad: {LOW: general | MEDIUM: algunos detalles | HIGH: muy detallada}
- Formalidad: {LOW: coloquial | MEDIUM: estandar | HIGH: formal}

Genera solo la resena, sin comentarios adicionales.
```

**Example 3: Adding interpretable reasoning to an existing classifier**

User: "I have a working text classifier but stakeholders want to understand WHY it makes each prediction. Can you help add chain-of-thought reasoning?"

Approach:
1. Take existing labeled dataset (real + any synthetic augmentation).
2. For each (text, label) pair, prompt a teacher LLM to generate hindsight CoT reasoning.
3. Build reasoning-augmented tuples: (text, reasoning, label).
4. Fine-tune with LoRA on the full reasoning + label sequence.
5. At inference, model now generates reasoning before prediction.

Output:
```python
# cot_extraction.py
COT_PROMPT = """You are an expert {domain} analyst. Given the following input
and its known classification, explain step by step what specific features in
the input support this classification.

Input: "{text}"
Classification: {label}

Provide a structured analysis:
1. Identify 3-5 specific observable features in the input
2. Explain how each feature relates to the classification
3. State your conclusion linking the evidence to the label

Be concrete - reference specific words, patterns, or structural elements."""

def extract_rationale(llm_client, text, label, domain="clinical linguistics"):
    response = llm_client.generate(
        COT_PROMPT.format(text=text, label=label, domain=domain)
    )
    return response.text

# Fine-tuning data format
training_sample = {
    "messages": [
        {"role": "system", "content": "You are a diagnostic reasoning assistant. Analyze the input, explain your reasoning step by step, then provide your classification."},
        {"role": "user", "content": f"Classify the following: {text}"},
        {"role": "assistant", "content": f"{rationale}\n\nClassification: {label}"}
    ]
}
```

## Best Practices

- **Do:** Discretize style dimensions into 3 ordinal levels (poor/normal/good) rather than continuous values. LLMs follow discrete behavioral instructions far more reliably than continuous scales.
- **Do:** Include demographic modifiers (age, education) that shift style distributions per class. This produces realistic within-class diversity rather than stereotyped samples.
- **Do:** Start with 2-3x augmentation ratio and ablate upward. The paper shows diminishing returns beyond 5x, and too much synthetic data can dilute real signal.
- **Do:** Use hindsight CoT extraction (give the label, ask for reasoning) rather than asking the model to reason freely. This produces higher-quality rationales grounded in the known answer.
- **Avoid:** Generating all synthetic samples from a single persona profile. The whole point is diversity — sample widely across the demographic-style space.
- **Avoid:** Skipping the persona schema and using generic "generate more examples" prompts. Uncontrolled generation produces narrow, mode-collapsed data that doesn't improve classifier robustness.
- **Avoid:** Evaluating with accuracy alone on imbalanced datasets. Always use Macro-F1 and report per-class metrics to catch hidden failures.

## Error Handling

- **Style vector doesn't differentiate classes:** If generated samples for different labels look too similar, increase the gap between class-conditioned baseline means or add more discriminative dimensions. Inspect with t-SNE or UMAP embedding visualization.
- **LLM ignores persona constraints:** If the generator produces text that doesn't match the specified style level (e.g., "short" narratives that are actually long), add explicit word count ranges and post-filter samples that violate constraints by more than 20%.
- **CoT rationales are generic or hallucinated:** If extracted reasoning doesn't reference specific features in the text, switch to a stronger teacher model, add few-shot examples of good rationales, or constrain the prompt to require quoting exact phrases from the input.
- **Cross-lingual generation quality drops:** If synthetic samples in a target language are unnatural, use a model with strong multilingual capabilities and add native-language few-shot examples to the generation prompt. Validate a random subset with native speakers before full-scale generation.
- **Overfitting to synthetic patterns:** If the model performs well on synthetic test data but poorly on real data, reduce the synthetic-to-real ratio, add domain-adversarial regularization, or ensure the persona sampling distribution covers the real data's demographic range.

## Limitations

- Synthetic data quality is bounded by the generator LLM's capabilities. If the LLM has weak knowledge of your specific domain (rare diseases, niche technical fields), persona conditioning alone won't produce expert-level samples.
- The approach works best for classification tasks where behavioral/linguistic variation correlates with class labels. It is less applicable to tasks where the signal is in factual content rather than style (e.g., fact verification).
- Cross-lingual transfer degrades with linguistic distance. The paper achieves 80.67% F1 in English but 48.71% in Mandarin, reflecting real limitations of zero-shot cross-lingual generalization.
- CoT fine-tuning adds inference latency since the model generates reasoning before classification. For latency-sensitive applications, consider distilling the reasoning model into a direct classifier after validation.
- Voice cloning and multimodal synthesis require additional tooling (IndexTTS2, reference audio libraries) that may not be available in all deployment contexts. The text-only persona synthesis pipeline is more broadly applicable.

## Reference

Feng, R., Luo, Z., Wu, L., Wang, W., & Song, Y. (2026). Cross-Linguistic Persona-Driven Data Synthesis for Robust Multimodal Cognitive Decline Detection. arXiv:2602.07978v1. https://arxiv.org/abs/2602.07978v1

Key sections to study: Section 3 (SynCog framework architecture — persona sampling, controllable generation, CoT deduction), Table 2 (style vector dimension specifications), and Section 4.4 (augmentation ratio ablation showing optimal synthetic-to-real balance).