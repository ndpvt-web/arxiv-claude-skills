---
name: "aqascore-evaluating-semantic-alignment"
description: "Evaluate semantic alignment between text prompts and generated audio using probabilistic yes/no verification with audio-language models. Use when: 'evaluate text-to-audio quality', 'score audio-text alignment', 'build an audio generation evaluation pipeline', 'measure if generated audio matches the prompt', 'compare audio generation models', 'assess compositional audio understanding'."
---

# AQAScore: Probabilistic Semantic Verification for Text-to-Audio Evaluation

This skill enables Claude to design and implement evaluation systems for text-to-audio generation using the AQAScore framework. Instead of relying on embedding similarity (like CLAPScore), AQAScore converts the evaluation problem into a binary yes/no question answering task: given a generated audio clip and its source text prompt, an audio-aware large language model (ALLM) is asked "Does this audio match the description?" and the log-probability of the "Yes" token is extracted as a calibrated alignment score. This approach captures fine-grained semantic mismatches -- missing sound events, wrong attributes, incorrect temporal ordering -- that cosine-similarity metrics miss entirely.

## When to Use

- When the user is building or improving a text-to-audio generation system and needs an evaluation metric beyond simple embedding similarity
- When the user wants to detect compositional failures in audio generation (e.g., "a dog barks then thunder rolls" generated with the events in wrong order)
- When the user asks to compare multiple audio generation models on semantic faithfulness
- When the user needs to build an automated benchmark pipeline for audio-text alignment
- When the user wants to evaluate whether specific sound events, attributes, or temporal relationships are present in generated audio
- When the user is integrating audio quality assessment into a CI/CD or model training loop

## Key Technique

**The core insight** of AQAScore is to reframe audio-text alignment evaluation from an open-ended similarity task into a constrained probabilistic verification task. Traditional metrics like CLAPScore encode audio and text into shared embedding spaces and compute cosine similarity. This works for coarse relevance but collapses under compositional reasoning -- it cannot distinguish "a woman laughs then a baby cries" from "a baby laughs then a woman cries" because the same concepts appear in both embeddings.

AQAScore instead feeds the audio and a derived yes/no question into an audio-aware LLM (such as Qwen2.5-Omni or Audio Flamingo 3), then extracts the exact log-probabilities of the "Yes" and "No" tokens. The score is computed as a softmax-normalized probability:

```
AQAScore(audio, text) = exp(s_yes) / (exp(s_yes) + exp(s_no))
```

where `s_yes = log p("Yes" | audio, question(text))` and `s_no = log p("No" | audio, question(text))`. This yields a value between 0 and 1 that is well-calibrated and more sensitive to subtle semantic mismatches than either embedding similarity or free-form LLM prompting. Crucially, the method is **backbone-agnostic** -- it improves automatically as stronger ALLMs become available, and the choice of question template has minimal impact on results.

**Why not just prompt the LLM to rate quality?** The paper tested this (direct prompting on a Likert scale) and found it performs worse than the log-probability approach. Chat-tuned models tend to have collapsed probability distributions, making their free-form ratings unreliable. Extracting raw token probabilities bypasses this issue and produces a more granular, continuous signal.

## Step-by-Step Workflow

1. **Identify the evaluation inputs.** Collect pairs of (text_prompt, generated_audio_path). Each text prompt is the conditioning input used during audio generation; each audio file is the model's output.

2. **Select an ALLM backend.** Choose an open-weight audio-language model that exposes token log-probabilities. Qwen2.5-Omni-7B is the strongest tested option (trained on ~3000k hours of audio). Qwen2.5-Omni-3B is a lighter alternative. Audio Flamingo 3 variants also work. Avoid proprietary models (e.g., GPT-4-Audio) since they do not expose log-probabilities.

3. **Convert each text prompt into a binary verification question.** Apply the template:
   ```
   "Does this audio contain the sound events described by the text: '{text_prompt}'? Please answer yes or no."
   ```
   This template is robust across phrasings, but keep the question closed-ended and binary.

4. **Run inference with the ALLM.** Pass each (audio, question) pair through the model. Configure the inference call to return log-probabilities for the next token rather than generating text. Extract the log-probability values for the "Yes" and "No" tokens specifically.

5. **Compute the AQAScore.** For each pair, apply softmax normalization over the two log-probabilities:
   ```python
   import math
   score = math.exp(s_yes) / (math.exp(s_yes) + math.exp(s_no))
   ```
   This produces a float in [0, 1] where higher means better alignment.

6. **Aggregate scores across your evaluation set.** Compute mean AQAScore per model, per prompt category, or per compositional challenge type. For pairwise model comparison, compare average scores directly.

7. **For compositional evaluation, construct targeted sub-questions.** Instead of one question per prompt, decompose complex prompts into atomic checks:
   - Temporal order: "Does the dog bark occur before the thunder?"
   - Attribute binding: "Is it a woman laughing, not a child?"
   - Event presence: "Does the audio contain the sound of rain?"
   Average the sub-scores to get a compositional alignment score.

8. **Validate against human judgments (optional but recommended).** Collect a small set of human-rated audio samples. Compute Pearson/Spearman correlation between AQAScores and human ratings to confirm the metric is well-calibrated for your domain.

9. **Integrate into your pipeline.** Wire the scoring function into your training loop (as a validation metric), CI pipeline (as a regression gate), or leaderboard (as a ranking criterion).

## Concrete Examples

**Example 1: Single-sample audio-text alignment scoring**

User: "I have a text-to-audio model that generated a clip from the prompt 'a cat meowing followed by glass breaking'. How do I evaluate if the audio actually matches?"

Approach:
1. Load the audio file and the Qwen2.5-Omni-7B model with log-probability output enabled
2. Construct the question: "Does this audio contain the sound events described by the text: 'a cat meowing followed by glass breaking'? Please answer yes or no."
3. Run inference, extract log-probabilities for "Yes" and "No" tokens
4. Compute normalized score

```python
from transformers import AutoModelForCausalLM, AutoProcessor
import torch
import math

model_name = "Qwen/Qwen2.5-Omni-7B"
processor = AutoProcessor.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(model_name, torch_dtype=torch.float16, device_map="auto")

text_prompt = "a cat meowing followed by glass breaking"
question = f"Does this audio contain the sound events described by the text: '{text_prompt}'? Please answer yes or no."

# Load audio and prepare inputs (model-specific)
inputs = processor(audio=audio_array, text=question, return_tensors="pt").to(model.device)

with torch.no_grad():
    outputs = model(**inputs)
    logits = outputs.logits[:, -1, :]  # logits for next token

# Get token IDs for "Yes" and "No"
yes_id = processor.tokenizer.encode("Yes", add_special_tokens=False)[0]
no_id = processor.tokenizer.encode("No", add_special_tokens=False)[0]

s_yes = logits[0, yes_id].item()
s_no = logits[0, no_id].item()

aqascore = math.exp(s_yes) / (math.exp(s_yes) + math.exp(s_no))
print(f"AQAScore: {aqascore:.4f}")  # e.g., 0.8723
```

Output: A score like `0.8723` meaning 87% confidence the audio matches the description.

**Example 2: Comparing two audio generation models on a benchmark**

User: "I want to compare AudioLDM2 and MusicGen on 200 prompts. Which produces more semantically faithful audio?"

Approach:
1. Generate audio from both models for all 200 prompts
2. Score every (prompt, audio) pair with AQAScore
3. Compare distributions

```python
import numpy as np

def compute_aqascores(audio_paths, prompts, model, processor):
    scores = []
    for audio_path, prompt in zip(audio_paths, prompts):
        audio = load_audio(audio_path)
        question = f"Does this audio contain the sound events described by the text: '{prompt}'? Please answer yes or no."
        s_yes, s_no = extract_log_probs(model, processor, audio, question)
        score = math.exp(s_yes) / (math.exp(s_yes) + math.exp(s_no))
        scores.append(score)
    return np.array(scores)

scores_ldm2 = compute_aqascores(ldm2_audios, prompts, allm, processor)
scores_musicgen = compute_aqascores(musicgen_audios, prompts, allm, processor)

print(f"AudioLDM2  mean AQAScore: {scores_ldm2.mean():.4f}")   # e.g., 0.6821
print(f"MusicGen   mean AQAScore: {scores_musicgen.mean():.4f}")  # e.g., 0.7134
print(f"MusicGen wins on {(scores_musicgen > scores_ldm2).sum()}/200 prompts")
```

Output: Per-model mean scores and head-to-head win counts, giving a clear ranking.

**Example 3: Compositional reasoning evaluation**

User: "My model struggles with temporal ordering. How do I specifically test if it generates events in the right sequence?"

Approach:
1. Create paired prompts with swapped event order (inspired by CompA-Order benchmark)
2. Generate audio for both orderings
3. Score each audio against both prompts -- a good model should score high only on the matching order

```python
prompt_a = "a dog barks and then thunder rolls"
prompt_b = "thunder rolls and then a dog barks"

# Generate audio conditioned on prompt_a
audio_a = generate_audio(prompt_a)

# Score audio_a against BOTH prompts
score_match = aqascore(audio_a, prompt_a)      # should be HIGH
score_mismatch = aqascore(audio_a, prompt_b)   # should be LOW

temporal_sensitivity = score_match - score_mismatch
print(f"Match score:    {score_match:.4f}")      # e.g., 0.81
print(f"Mismatch score: {score_mismatch:.4f}")   # e.g., 0.43
print(f"Temporal gap:   {temporal_sensitivity:.4f}")  # e.g., 0.38
# A large gap means the model respects temporal ordering.
# A small gap means the model ignores event order.
```

## Best Practices

- **Do** use models that expose raw token log-probabilities. The entire technique depends on accessing `s_yes` and `s_no` directly, not on parsing generated text.
- **Do** keep the question template simple and closed-ended. The paper found phrasing variations have minimal effect, so prefer clarity over cleverness.
- **Do** decompose complex prompts into atomic sub-questions when diagnosing specific failure modes (temporal order, attribute binding, event presence).
- **Do** batch inference across your evaluation set and report both mean and per-category breakdowns to surface systematic weaknesses.
- **Avoid** using chat-tuned model variants if they collapse probability distributions. Base or lightly-tuned models produce more calibrated log-probabilities. Test by checking that scores distribute broadly across [0, 1] rather than clustering near 0.5.
- **Avoid** using this method with proprietary API-only models (GPT-4o, Gemini) that do not return token-level log-probabilities. The technique fundamentally requires probability access.

## Error Handling

- **Model does not expose log-probabilities:** Some inference frameworks suppress logit output by default. Ensure you set `output_logits=True` or equivalent. If using vLLM or TGI, configure the `logprobs` parameter in the request.
- **"Yes"/"No" tokenization varies across models:** Different tokenizers may encode "Yes" as one or multiple tokens. Always verify the token ID mapping. Some models use "yes" (lowercase) -- check both casings and use whichever has higher total probability mass.
- **Audio loading issues:** ALLMs expect specific sample rates (commonly 16kHz). Resample audio before inference. Mismatched sample rates produce garbage scores silently.
- **Score distribution is degenerate:** If all scores cluster near 0.5 or near 1.0, the ALLM may be poorly calibrated for your audio domain. Try a different backbone model or verify the audio preprocessing pipeline.
- **Out-of-memory on long audio:** Truncate or chunk audio to the model's maximum context length (often 30s for audio LLMs). Score each chunk and average, or use only the most information-dense segment.

## Limitations

- **Requires open-weight ALLMs with logit access.** This rules out most commercial audio APIs. You need GPU infrastructure to run models like Qwen2.5-Omni-7B.
- **Quality is bounded by the ALLM's audio understanding.** If the backbone model cannot distinguish certain sounds or temporal patterns, AQAScore inherits those blind spots. The metric improves as ALLMs improve, but it cannot exceed its backbone.
- **Music and speech have different challenges than sound events.** The benchmarks in the paper focus heavily on environmental sounds and sound events. Performance on music generation quality or speech naturalness may differ.
- **Not a replacement for perceptual quality metrics.** AQAScore measures semantic alignment (does the audio match the text?), not audio fidelity (does it sound realistic?). Use it alongside metrics like FAD or FID for audio quality.
- **Computational cost scales linearly with evaluation set size.** Each sample requires a full ALLM forward pass. For large-scale evaluation (10k+ samples), budget GPU time accordingly or use the smaller 3B model variant.

## Reference

- **Paper:** [AQAScore: Evaluating Semantic Alignment in Text-to-Audio Generation via Audio Question Answering](https://arxiv.org/abs/2601.14728v1) (Kuan, Chang, Lee, 2026)
- **Key takeaway:** Section 4 defines the core formula and question template. Section 5 contains benchmark results showing AQAScore outperforms CLAPScore on human-correlation tasks. Section 5.4 demonstrates robustness to prompt template variation. Table 2 has the critical model comparison numbers.