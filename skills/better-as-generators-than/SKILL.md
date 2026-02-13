---
name: "better-as-generators-than"
description: "Generate synthetic labeled datasets with LLMs to train smaller, cheaper classifiers -- especially for low-resource languages and niche tasks. Use when: 'generate training data for my classifier', 'I need labeled data in [language]', 'distill this LLM into a smaller model', 'create synthetic examples for fine-tuning', 'bootstrap a text classifier without manual annotation', 'train a multilingual classifier with no labeled data'."
---

# LLM-as-Generator: Synthetic Data Distillation for Multilingual Classification

This skill enables Claude to design and execute a synthetic data generation pipeline where a large LLM acts as a **teacher/generator** producing labeled training examples, which are then used to train or prompt smaller, cheaper **student models** that consistently outperform the generator itself on classification tasks. The technique is most powerful for low-resource languages and underrepresented tasks where labeled data is scarce, achieving up to 40% improvement over direct LLM classification with as few as 50 synthetic samples per class.

## When to Use

- When the user needs a text classifier but has little or no labeled training data
- When the user wants to classify text in a low-resource language (e.g., Azerbaijani, Welsh, Telugu, Swahili)
- When the user asks to distill a large LLM's knowledge into a smaller, deployable model
- When the user wants to reduce inference costs by replacing LLM-based classification with a fine-tuned small model
- When the user needs labeled examples for sentiment analysis, topic classification, intent recognition, or other text classification tasks
- When the user asks how to bootstrap a multilingual classifier without hiring annotators
- When the user has a niche classification task (e.g., sarcasm detection, domain-specific labeling) where existing datasets don't exist

## Key Technique

The core insight is that **LLMs are better used as data generators than as direct classifiers**. When you prompt a large LLM to classify text zero-shot, it performs reasonably but not great -- especially in non-English languages. However, when you instead use that same LLM to *generate* synthetic labeled examples, then train a smaller model (like XLM-RoBERTa or an 8B-parameter LLM) on those examples, the smaller model consistently beats the large generator. This works because generation leverages the LLM's broad linguistic knowledge to produce diverse, natural-sounding examples, while the smaller model specializes entirely on the classification boundary.

The pipeline has three stages: (1) **Generate** synthetic labeled text using a large LLM with carefully structured prompts that include 10 human-labeled seed examples per class as in-context demonstrations, producing 6 diverse samples per inference call. (2) **Filter** the generated data using the generator itself as a quality judge, removing duplicates, language-contaminated samples (e.g., English leaking into Telugu output), and low-quality text. (3) **Train** a smaller model via fine-tuning (XLM-RoBERTa), LoRA instruction-tuning (8B LLMs), or use the synthetic samples as in-context examples for compact LLMs.

Critical finding on sample efficiency: performance gains plateau around 200 samples per class, and even 50 samples per class is enough to outperform the large generator across all language resource levels. Beyond 400 samples, synthetic data diversity drops and marginal returns diminish. This means the pipeline is cheap to run -- a few hundred generation calls per task are sufficient.

## Step-by-Step Workflow

1. **Define the classification schema.** Identify the exact task (sentiment, topic, intent, etc.), enumerate all class labels, and specify the target language(s). For each label, write a clear, unambiguous definition that the generator LLM can follow.

2. **Collect 5-10 seed examples per class.** These are real human-written examples that anchor generation quality. If you have zero labeled data, manually write or translate 5-10 examples per label. These serve as in-context demonstrations in the generation prompt.

3. **Construct the generation prompt.** Use this template structure:
   ```
   Please create 6 different {task_description} texts in the {language} language,
   separated by "|||". Each text should express {label_definition}.

   Here are examples of {label} texts in {language}:
   1. {seed_example_1}
   2. {seed_example_2}
   ...
   10. {seed_example_10}

   Output only the texts in {language} and nothing else. Do not number the texts.
   ```
   Set generation parameters: `temperature=0.7, top_p=0.9, repetition_penalty=1.2`.

4. **Generate 200 samples per class.** Run the generation prompt in batches (6 samples per call), iterating across all labels. For a binary task this means ~67 calls per class; for a 10-class task, ~334 calls per class. Track the label distribution to ensure balance.

5. **Filter generated samples for quality.** Apply three filters sequentially:
   - **Deduplication:** Remove exact and near-duplicate texts (Jaccard similarity > 0.9).
   - **Language purity check:** Use the generator LLM or a language detection library (e.g., `langdetect`, `fasttext-langid`) to flag samples containing unexpected language mixing.
   - **Self-revision judge:** Prompt the generator to evaluate each sample: "Is this text a natural example of {label} in {language}? Answer yes or no." Remove samples rated "no."

6. **Choose the student model and training strategy based on deployment constraints:**
   - **Cheapest inference, best accuracy:** Fine-tune XLM-RoBERTa-large on the synthetic data (dropout=0.3, lr=1e-5, AdamW, early stopping at 50 epochs). Best for production deployment.
   - **Moderate cost, flexible:** LoRA instruction-tune an 8B LLM (rank=16, alpha=16, lr=3e-5, 600 steps). Good when you also need the model for other tasks.
   - **Zero training, fast setup:** Use 2-4 synthetic samples per class as ICL examples for a compact LLM (temperature=0, greedy decoding). Good for prototyping.

7. **Evaluate on held-out real data.** Always test on human-labeled examples, never on synthetic data. Compare the student model against the generator's zero-shot and few-shot classification performance as baselines. Report per-language and per-class metrics.

8. **Iterate on weak spots.** If specific classes or languages underperform, generate additional targeted samples for those categories, re-filter, and retrain. Low-resource languages benefit most from even small increases in synthetic data volume.

## Concrete Examples

**Example 1: Bootstrapping a sentiment classifier for Indonesian product reviews**

User: "I need a sentiment classifier for Indonesian customer reviews but I have no labeled data."

Approach:
1. Define binary labels: `positive`, `negative`.
2. Manually write 10 Indonesian review examples per class (or translate from English reviews).
3. Generate 200 synthetic reviews per class using the generation prompt template with `language=Indonesian`, `task_description=product review sentiment`.
4. Filter: remove duplicates, verify language purity (flag any English phrases), run self-revision.
5. Fine-tune XLM-RoBERTa-large on the 400 filtered synthetic reviews.
6. Evaluate on a small manually-labeled test set of 50 real reviews.

Output pipeline code:
```python
import json

# Step 3: Generation prompt for one class
def build_generation_prompt(label, seed_examples, language="Indonesian"):
    examples_text = "\n".join(f"{i+1}. {ex}" for i, ex in enumerate(seed_examples))
    return f"""Please create 6 different product review texts in the {language} language,
separated by "|||". Each review should express {label} sentiment about a product.

Here are examples of {label} reviews in {language}:
{examples_text}

Output only the review texts in {language} and nothing else. Do not number the texts."""

# Step 5: Quality filter prompt
def build_filter_prompt(text, label, language="Indonesian"):
    return f"""Is the following text a natural {label} product review written entirely in {language}?
Text: "{text}"
Answer only "yes" or "no"."""

# Generation parameters
gen_params = {
    "temperature": 0.7,
    "top_p": 0.9,
    "max_tokens": 4096,
    "repetition_penalty": 1.2,
}
```

**Example 2: Multi-class intent classifier for a Welsh-language voice assistant**

User: "Build me an intent classifier that works in Welsh for a voice assistant. I have 10 intent categories."

Approach:
1. Define 10 intent labels (e.g., `set_alarm`, `play_music`, `get_weather`, `send_message`, etc.).
2. For each intent, write 5-10 seed utterances in Welsh (use a translator if needed, then have a Welsh speaker verify).
3. Generate 200 synthetic utterances per intent = 2,000 total samples.
4. Filter aggressively for language purity -- Welsh is highly susceptible to English contamination in LLM output.
5. Fine-tune XLM-RoBERTa-large on the filtered dataset. Use dynamic batch sizing: start with batch_size=4 if data is small, scale to 64 with full dataset.
6. Expected result: the fine-tuned model outperforms the generator LLM's zero-shot intent classification by ~30-40% on Welsh.

Key consideration for Welsh (low-resource): Generate an extra 50% buffer of samples because the language purity filter will reject more outputs than for high-resource languages.

**Example 3: Rapid prototyping with ICL instead of fine-tuning**

User: "I need a quick topic classifier for Swahili news articles. No time to fine-tune."

Approach:
1. Define 7 topic labels (politics, sports, technology, health, business, entertainment, science).
2. Generate 20 synthetic Swahili articles per topic using the pipeline (140 total -- cheap and fast).
3. Skip fine-tuning entirely. Instead, select the 2-4 highest-quality synthetic samples per class.
4. Use these as ICL examples for a compact LLM (e.g., Qwen-2.5-7B or LLaMA-3.1-8B):
   ```
   Classify the following Swahili text into one of these topics:
   politics, sports, technology, health, business, entertainment, science.

   Examples:
   Text: "{synthetic_example_1}" -> politics
   Text: "{synthetic_example_2}" -> sports
   ...

   Text: "{user_input}" ->
   ```
5. Set `temperature=0` for deterministic classification. Limit output to 20 tokens.
6. This ICL approach outperforms the large generator's zero-shot classification by ~15% on Swahili.

## Best Practices

- **Do:** Always include 5-10 real seed examples per class in the generation prompt. Zero-shot generation without seeds produces significantly lower-quality synthetic data.
- **Do:** Generate in batches of 6 samples per call with `repetition_penalty=1.2` to maximize diversity within each batch.
- **Do:** Filter aggressively for language purity, especially for low-resource languages. LLMs tend to inject English phrases into non-English output.
- **Do:** Start with 50 samples per class as a minimum viable dataset. You can often stop at 200 per class -- returns diminish sharply after that.
- **Avoid:** Generating more than 400 samples per class. Synthetic data diversity drops and can introduce noise that hurts performance.
- **Avoid:** Using the synthetic data for evaluation. Always hold out real human-labeled data for testing, even if it's a tiny set of 20-50 examples.
- **Avoid:** Skipping the self-revision filter step. Without quality filtering, 10-20% of generated samples may be off-label, contaminated with the wrong language, or nonsensical.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| Generated text contains English phrases mixed in | LLM defaults to English for unfamiliar concepts | Add explicit instruction: "Do not use any English words." Re-filter with language detection. Generate extra buffer samples. |
| All generated samples sound similar | Temperature too low or repetition penalty too low | Increase temperature to 0.8, increase repetition_penalty to 1.3. Vary seed examples across batches. |
| Student model performs worse than generator | Too few samples or severe class imbalance | Verify at least 50 samples per class survived filtering. Check class distribution. Regenerate for underrepresented classes. |
| Fine-tuning overfits quickly | Synthetic data lacks diversity | Use dropout=0.3, reduce epochs, add early stopping. Consider mixing in any available real data. |
| Generator refuses to produce certain content | Safety filters triggered by task labels (e.g., toxicity detection) | Reframe the prompt: instead of "generate toxic text," use "generate examples of text that would be flagged as inappropriate by a content moderator." |
| Language detection rejects valid samples | Mixed-script languages or loanwords | Use a per-language threshold rather than hard cutoff. For languages with heavy borrowing (e.g., Swahili with Arabic/English loanwords), relax the purity check. |

## Limitations

- **Language coverage ceiling:** The generator LLM must have reasonable pretraining exposure to the target language. For truly zero-resource languages absent from pretraining data, synthetic generation quality will be poor.
- **Task complexity:** This pipeline works for classification tasks with clear label boundaries. It is not designed for sequence labeling, extraction, generation, or regression tasks.
- **Diversity saturation:** Synthetic data is inherently less diverse than real-world data. Beyond 200-400 samples per class, additional synthetic data adds noise rather than signal.
- **Seed dependency:** The quality of the 5-10 seed examples strongly influences generation quality. Poor or unrepresentative seeds propagate errors.
- **Cultural nuance:** LLM-generated text for low-resource languages may reflect the dominant culture in the training data rather than the target culture, especially for sentiment and sarcasm tasks.
- **Not a replacement for real data:** When real labeled data is available, mixing it with synthetic data (even a small amount of real data) substantially outperforms synthetic-only training.

## Reference

[Better as Generators Than Classifiers (Pecher et al., EACL 2026 Findings)](https://arxiv.org/abs/2601.16278v1) -- Focus on Section 4 (experimental setup with prompt templates and generation parameters), Table 2 (performance by language resource level), and Section 5.2 (analysis of sample efficiency showing the 50-sample threshold).