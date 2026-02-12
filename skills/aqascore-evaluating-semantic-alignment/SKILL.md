---
name: "aqascore-evaluating-semantic-alignment"
description: >
  Evaluate semantic alignment between text descriptions and generated audio using
  the AQAScore framework — probabilistic verification via audio question answering
  instead of embedding similarity. Use this skill when:
  - "evaluate text-to-audio alignment"
  - "score how well generated audio matches a prompt"
  - "build an audio evaluation pipeline"
  - "compare audio generation models"
  - "assess compositional audio reasoning"
  - "implement AQAScore metric"
---

# AQAScore: Evaluating Semantic Alignment via Audio Question Answering

This skill enables Claude to design, implement, and apply the AQAScore evaluation framework for text-to-audio generation systems. AQAScore replaces traditional embedding-similarity metrics (like CLAPScore) with a probabilistic semantic verification approach: it feeds generated audio and a targeted yes/no question into an audio-aware large language model (ALLM), then computes the normalized log-probability of a "Yes" answer as a continuous alignment score. This captures fine-grained semantic mismatches — temporal ordering, attribute binding, missing sound events — that cosine-similarity metrics miss entirely.

## When to Use This Skill

- When the user is building or evaluating a text-to-audio (TTA) generation system and needs a metric beyond FrechetAudioDistance or CLAPScore
- When the user asks how to measure whether generated audio actually matches its text prompt at a semantic level
- When the user needs to compare multiple audio generation models on compositional prompts (e.g., "a dog barking then a car horn")
- When the user wants to implement automated evaluation for an audio ML pipeline that correlates well with human judgments
- When the user asks about evaluating temporal ordering or attribute binding in generated audio
- When the user is designing a benchmark or leaderboard for audio generation quality

## Key Technique

**From embedding similarity to probabilistic verification.** Traditional metrics like CLAPScore project both the text prompt and the generated audio into a shared embedding space, then compute cosine similarity. This works for coarse relevance — "is there music?" — but collapses when prompts contain compositional structure. "A dog barking followed by thunder" and "thunder followed by a dog barking" produce nearly identical embeddings, yet describe different audio. AQAScore sidesteps this by reformulating evaluation as a question-answering task: given the audio and a question like *"Does this audio contain the sound events described by: a dog barking followed by thunder?"*, the model's probability of answering "Yes" becomes the alignment score.

**Log-probability extraction, not text generation.** A critical design choice is that AQAScore does *not* rely on the ALLM generating a free-text response and then parsing it. Instead, it extracts the raw logits for the "Yes" and "No" tokens at the first generated position and applies softmax normalization: `AQAScore = exp(s_yes) / (exp(s_yes) + exp(s_no))`. This yields a continuous score in [0, 1] that is deterministic, fast (single forward pass, single token), and avoids the noise and latency of sampling-based evaluation. The approach is backbone-agnostic — it works with any ALLM that exposes token logits.

**Scaling with model capability.** Experiments across ALLMs of varying size (Qwen2.5-Omni 3B vs. 7B, Audio Flamingo 3 variants) demonstrate that AQAScore improves as the underlying model improves. On the RELATE benchmark, Qwen2.5-Omni-7B achieved Pearson correlation of 0.544 with human ratings versus CLAPScore's best of 0.448. On compositional tasks (CompA-Order), it reached 67% accuracy versus 40.7% for CLAP-based baselines — a 65% relative improvement.

## Step-by-Step Workflow

1. **Define the evaluation scope.** Determine what text-audio pairs need scoring: a batch of TTA model outputs, a comparison between two models, or a single generation quality check. Collect the text prompts and corresponding audio files (WAV, MP3, FLAC).

2. **Select an audio-aware LLM backend.** Choose an ALLM that exposes token-level logits. Strong options: Qwen2.5-Omni-7B (best open-source results), Qwen2.5-Omni-3B (lighter weight), Audio Flamingo 3. The model must accept audio input and produce next-token logits — not just text output.

3. **Construct the verification question.** Format each text prompt into a yes/no semantic query. The canonical template is:

   ```
   Does this audio match the following description: "{text_prompt}"? Answer Yes or No.
   ```

   Research showed negligible performance differences across prompt phrasings, so the exact wording is not critical. What matters is that the question is answerable with "Yes" or "No" and references the full description.

4. **Run inference to extract logits.** Pass each (audio, question) pair through the ALLM. At the first generated token position, extract the logits for the "Yes" and "No" tokens (including capitalization variants: "yes", "Yes", "YES", and similarly for "No"). Sum the logits across variants if needed.

5. **Compute the AQAScore.** Apply softmax normalization over the Yes/No logits:

   ```python
   import math

   def aqascore(logit_yes: float, logit_no: float) -> float:
       exp_yes = math.exp(logit_yes)
       exp_no = math.exp(logit_no)
       return exp_yes / (exp_yes + exp_no)
   ```

   The result is a float in [0, 1] where 1.0 means perfect semantic alignment.

6. **Aggregate scores across a dataset.** For batch evaluation, compute the mean AQAScore across all text-audio pairs. For model comparison, compute per-sample scores and run a paired statistical test (Wilcoxon signed-rank or paired t-test).

7. **Handle compositional prompts (optional).** For prompts with explicit temporal or attribute structure, you can decompose the prompt into sub-queries for more granular diagnostics:
   - Temporal: "Does the dog barking occur before the thunder?"
   - Attribute: "Is the piano sound soft?"
   - Presence: "Does the audio contain a car horn?"

   Score each sub-query independently, then report both individual and aggregated scores.

8. **Validate against human judgments.** If building a new benchmark, collect human relevance ratings (1-5 Likert or pairwise preference) and compute Pearson/Spearman correlation with AQAScore to confirm the metric is calibrated for your domain.

9. **Integrate into CI/CD or training loops.** Wrap the scoring pipeline as a callable function or CLI tool. For training-loop integration, AQAScore can serve as a reward signal — though note it requires a full ALLM forward pass per sample, so batch strategically.

## Concrete Examples

**Example 1: Scoring a single text-to-audio generation**

User: "I generated audio from the prompt 'a cat meowing in a large empty room with reverb.' How do I check if it actually matches?"

Approach:
1. Load the audio file and the Qwen2.5-Omni-7B model
2. Construct the query: `Does this audio match the following description: "a cat meowing in a large empty room with reverb"? Answer Yes or No.`
3. Run a single forward pass, extract logits for "Yes" and "No"
4. Compute softmax-normalized score

Output:
```python
from transformers import AutoModelForCausalLM, AutoProcessor
import torch

model_id = "Qwen/Qwen2.5-Omni-7B"
processor = AutoProcessor.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(model_id, torch_dtype=torch.float16, device_map="auto")

audio_path = "generated_cat_reverb.wav"
question = 'Does this audio match the following description: "a cat meowing in a large empty room with reverb"? Answer Yes or No.'

# Process inputs (model-specific — adapt to your ALLM's API)
inputs = processor(audio=audio_path, text=question, return_tensors="pt").to(model.device)

with torch.no_grad():
    outputs = model(**inputs)
    logits = outputs.logits[0, -1, :]  # logits at the generation position

# Get token IDs for Yes/No
yes_id = processor.tokenizer.encode("Yes", add_special_tokens=False)[0]
no_id = processor.tokenizer.encode("No", add_special_tokens=False)[0]

s_yes = logits[yes_id].item()
s_no = logits[no_id].item()

import math
score = math.exp(s_yes) / (math.exp(s_yes) + math.exp(s_no))
print(f"AQAScore: {score:.4f}")
# Output: AQAScore: 0.8723
```

**Example 2: Comparing two TTA models on a benchmark**

User: "I have outputs from ModelA and ModelB on 500 prompts. Which model produces audio that better matches the text?"

Approach:
1. Score all 500 pairs for each model using the AQAScore pipeline
2. Compute mean scores and per-sample differences
3. Run a statistical significance test

Output:
```python
import numpy as np
from scipy.stats import wilcoxon

scores_a = np.array([compute_aqascore(audio_a, prompt) for audio_a, prompt in zip(model_a_outputs, prompts)])
scores_b = np.array([compute_aqascore(audio_b, prompt) for audio_b, prompt in zip(model_b_outputs, prompts)])

print(f"ModelA mean AQAScore: {scores_a.mean():.4f} (+/- {scores_a.std():.4f})")
print(f"ModelB mean AQAScore: {scores_b.mean():.4f} (+/- {scores_b.std():.4f})")

stat, p_value = wilcoxon(scores_a, scores_b)
print(f"Wilcoxon p-value: {p_value:.6f}")
# ModelA mean AQAScore: 0.7234 (+/- 0.1821)
# ModelB mean AQAScore: 0.8102 (+/- 0.1547)
# Wilcoxon p-value: 0.000031 → ModelB is significantly better
```

**Example 3: Diagnosing compositional failures**

User: "My model generates audio for 'a piano playing softly then drums coming in loudly' but it sounds wrong. Can AQAScore tell me what's off?"

Approach:
1. Decompose the prompt into sub-queries targeting each semantic component
2. Score each independently to isolate the failure mode

Output:
```python
sub_queries = [
    ("Does this audio contain piano?", "presence_piano"),
    ("Does this audio contain drums?", "presence_drums"),
    ("Is the piano playing softly?", "attribute_piano_soft"),
    ("Are the drums loud?", "attribute_drums_loud"),
    ("Does the piano start before the drums?", "order_piano_first"),
]

for query, label in sub_queries:
    score = compute_aqascore(audio_file, query)
    status = "PASS" if score > 0.6 else "FAIL"
    print(f"[{status}] {label}: {score:.3f}")

# [PASS] presence_piano:       0.921
# [PASS] presence_drums:        0.887
# [FAIL] attribute_piano_soft:  0.412  ← piano too loud
# [PASS] attribute_drums_loud:  0.734
# [FAIL] order_piano_first:     0.298  ← drums come in first
```

Diagnosis: The model generates both instruments but fails on the "softly" attribute and temporal ordering.

## Best Practices

- **Do** use log-probability extraction rather than parsing generated text. Text generation introduces sampling noise and is slower. Logit extraction is deterministic and single-pass.
- **Do** normalize over only "Yes" and "No" tokens. Including other tokens in the softmax denominator dilutes the signal and hurts correlation with human judgments.
- **Do** use the strongest available ALLM. AQAScore scales with model capability — a 7B model meaningfully outperforms a 3B model on the same prompts.
- **Do** include token capitalization variants ("Yes", "yes", "YES") when extracting logits, and sum their probabilities before normalization.
- **Avoid** relying on AQAScore as the *sole* metric. It measures semantic alignment but not audio quality (artifacts, distortion, naturalness). Pair it with FrechetAudioDistance or human MOS for a complete picture.
- **Avoid** using proprietary API-only models (GPT-4o-audio, Gemini) for AQAScore, since they typically do not expose token logits. You would fall back to prompting-based evaluation, which has lower correlation with human judgments.

## Error Handling

| Problem | Cause | Solution |
|---|---|---|
| Score is always ~0.5 | The ALLM doesn't understand audio content | Verify the model actually processes audio input (not just text). Test with an obvious match first. |
| Score is 0.0 or 1.0 for everything | Logit extraction targets wrong token positions | Confirm you're reading logits at the correct generation position (first token after the prompt). |
| "Yes"/"No" token IDs not found | Tokenizer encodes them differently | Inspect `tokenizer.encode("Yes")` — some tokenizers prepend space tokens. Search for all variants. |
| OOM on large batches | ALLM memory requirements | Process in smaller batches. Use float16/bfloat16. The 3B variant requires ~7GB VRAM; 7B requires ~15GB. |
| Low correlation with human ratings on music | Music descriptions are more subjective | AQAScore works best on sound-event descriptions. For music, supplement with MusicCaps-style evaluation. |

## Limitations

- **Requires an ALLM with logit access.** Cloud APIs that only return text (no logprobs) cannot implement true AQAScore. You'd need to fall back to prompting-based scoring, which is noisier.
- **Computational cost.** Each score requires a full forward pass of a 3-7B parameter model. For large-scale evaluation (100K+ samples), this is orders of magnitude slower than CLAPScore's lightweight embedding comparison.
- **Audio quality is out of scope.** AQAScore measures "does the audio match the description?" not "does the audio sound good?" A perfectly matching but heavily distorted audio can still score high.
- **Compositional decomposition is manual.** The paper uses holistic single-question scoring by default. Decomposing prompts into sub-queries for fine-grained diagnostics requires manual or LLM-assisted prompt engineering — there is no automatic decomposition built into the method.
- **Performance ceiling tied to ALLM capability.** If the underlying model cannot perceive certain audio events (e.g., subtle pitch changes, specific instrument timbres), AQAScore inherits that blindness.

## Reference

**Paper:** [AQAScore: Evaluating Semantic Alignment in Text-to-Audio Generation via Audio Question Answering](https://arxiv.org/abs/2601.14728v1) — Kuan, Chang, Lee (2026). Look for: the softmax formulation over Yes/No logits (Section 3), benchmark results tables comparing against CLAPScore (Section 5), and compositional evaluation on CompA (Section 5.3).