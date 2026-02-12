---
name: "parse-open-domain-reasoning-question"
description: "Build and evaluate reasoning-focused QA systems for low-resource languages using the PARSE methodology: structured prompting strategies (CoT for Boolean/multiple-choice, few-shot for factoid), multi-stage quality filtering, and language-native prompt design. Use when: 'build a Persian QA benchmark', 'evaluate reasoning in low-resource language', 'create multilingual QA dataset', 'optimize prompting strategy for question types', 'generate validated QA pairs with difficulty levels', 'fine-tune model for Persian question answering'."
---

# PARSE: Open-Domain Reasoning QA for Low-Resource Languages

This skill enables Claude to apply the PARSE methodology for building, evaluating, and optimizing reasoning-focused question answering systems, particularly for low-resource languages. The core technique is a controlled LLM-based generation pipeline that produces validated QA benchmarks across multiple question formats (Boolean, multiple-choice, factoid), paired with format-specific prompting strategies -- Chain-of-Thought for structured answer types, few-shot for open-ended factoid questions -- with prompts written in the target language rather than English. This approach is directly applicable to building QA datasets, evaluating multilingual LLMs, and selecting optimal prompting strategies per question type.

## When to Use

- When building a QA benchmark or evaluation dataset for a low-resource language (Persian, Arabic, Hindi, Swahili, etc.)
- When evaluating LLM reasoning capabilities across Boolean, multiple-choice, and factoid question types
- When selecting the optimal prompting strategy (zero-shot, few-shot, CoT) for a specific question format
- When generating validated question-answer pairs with controlled difficulty levels using an LLM pipeline
- When fine-tuning a model for reasoning-focused QA in a non-English language
- When implementing multi-stage quality filtering for LLM-generated datasets
- When designing prompts for multilingual QA systems and need to decide between target-language vs. English prompts

## Key Technique

**Format-Specific Prompting**: The central finding is that no single prompting strategy wins across all question types. Chain-of-Thought (CoT) prompting -- where the model reasons step-by-step before answering -- works best for Boolean and multiple-choice questions because these require evaluating constraints and eliminating options. Few-shot prompting -- providing 2-3 solved examples -- works best for factoid questions because these require learning the expected answer format (short phrase, list, or "non-answerable") rather than extended reasoning. Zero-shot prompting is a reasonable baseline but consistently underperforms the matched strategy.

**Target-Language Prompts**: Persian prompts consistently outperform English prompts across all question types and models, even for multilingual models trained predominantly on English. The reason: when questions are written in Persian, Persian-language instructions better guide the model to interpret the question faithfully, avoiding cross-lingual transfer errors. This generalizes: always write system prompts and instructions in the same language as the questions being answered.

**Controlled Generation Pipeline**: Questions are generated in batches of 30 (10 easy, 10 medium, 10 hard) using configuration-specific prompts that specify the question type, reasoning dimension (single-hop reasoning vs. multi-hop evidence chaining), and answer subtype. Each prompt has four components: role description, task block, requirements (format, language, realism), and output instructions. Multi-stage filtering then removes structural errors, duplicates, and semantically invalid items. This pipeline produces high-quality benchmarks at scale with human-validated quality scores above 4.4/5.

## Step-by-Step Workflow

### Building a Reasoning QA Dataset

1. **Define the configuration matrix.** Cross question types (Boolean, multiple-choice, factoid) with reasoning dimensions (single-hop reasoning, multi-hop) and answer subtypes (e.g., simple/negation/comparative for Boolean; single-answer/multi-answer/non-answerable for multiple-choice; simple/list-based/non-answerable for factoid). This yields 18 configurations for PARSE; adapt to your domain.

2. **Write generation prompts per configuration.** Each prompt must contain four blocks: (a) a role description anchoring model behavior ("You are a question generation expert for Persian"), (b) a task block specifying the exact configuration ("Generate multi-hop Boolean questions with comparative reasoning"), (c) requirements for answer format, option count, language naturalness, topical diversity, and difficulty calibration, and (d) output formatting instructions (CSV or JSON). Write all prompts in the target language.

3. **Generate questions in balanced batches.** Call GPT-4o (or equivalent) with each configuration prompt, requesting 30 questions per batch: 10 easy, 10 medium, 10 hard. Hard items should derive difficulty from reasoning depth and compositionality, not from convoluted grammar. Run multiple batches until you reach your target count (PARSE targets 600 per configuration, 10,800 total).

4. **Apply structural validation.** Filter out items with missing fields, incorrect option cardinality (multiple-choice must have exactly 4 options), invalid difficulty labels, or malformed answer formats. Boolean answers must be strictly yes/no; factoid list answers must contain 2-5 items.

5. **Deduplicate at question and option level.** Remove exact duplicates within and across batches. For multiple-choice, also check for duplicate option sets that would indicate recycled questions with superficial rewording.

6. **Run semantic validation.** Verify Boolean items are actually answerable with yes/no, multiple-choice items have exactly one correct answer (or a specified set for multi-answer), and non-answerable items genuinely lack sufficient information. Flag items where difficulty doesn't match annotator expectations.

7. **Conduct human evaluation on a sample.** Have 2-3 annotators rate a random sample (PARSE used ~10%) on ambiguity (1-5) and readability (1-5). Target scores above 4.0. Discard or revise items scoring below 3.0. Verify that annotator accuracy decreases from easy to hard, confirming difficulty calibration.

8. **Split into train/test sets.** Use an 80/20 split (PARSE: 8,640 train / 2,160 test) with 120 items per configuration in the test set, ensuring balanced representation across all dimensions.

### Evaluating Models with Format-Specific Prompting

9. **Match prompting strategy to question type.** Use CoT prompting for Boolean and multiple-choice questions. Use few-shot prompting (2-3 examples) for factoid questions. Write all evaluation prompts in the target language. Compare against zero-shot as a baseline.

10. **Evaluate with type-appropriate metrics.** Use accuracy for Boolean, single-answer multiple-choice, and non-answerable items. Use Jaccard similarity for multi-answer multiple-choice and list-based factoid. Use string containment for simple factoid answers.

## Concrete Examples

**Example 1: Building a Persian QA Evaluation Set**

```
User: I need to create a small evaluation benchmark for testing LLMs on Persian
      reasoning questions. About 100 questions covering different types.

Approach:
1. Define a reduced configuration matrix:
   - Boolean: simple, comparative (2 subtypes)
   - Multiple-choice: single-answer, multi-answer (2 subtypes)
   - Factoid: simple, list-based (2 subtypes)
   Total: 6 configurations x ~17 questions each = ~100 questions

2. Write a generation prompt (in Persian) for each configuration:

   [Role] شما یک متخصص تولید سوال هستید.
   [Task] سوالات بولی چندمرحله‌ای ساده به زبان فارسی تولید کنید.
   [Requirements]
   - هر سوال باید با بله یا خیر قابل پاسخ باشد
   - سطح دشواری: 6 آسان، 6 متوسط، 5 سخت
   - موضوعات متنوع: تاریخ، علم، جغرافیا، فرهنگ
   - فارسی طبیعی و روان، نه ترجمه تحت‌اللفظی
   [Output] JSON با فیلدهای: question, answer, difficulty, topic

3. Generate, then filter:
   - Structural: verify all fields present, answers are بله/خیر
   - Deduplication: remove near-duplicates via fuzzy matching
   - Semantic: spot-check 20% manually for answerability

Output format (per item):
{
  "id": "bool-simple-001",
  "question": "آیا رود نیل طولانی‌ترین رود جهان است؟",
  "answer": "بله",
  "type": "boolean",
  "subtype": "simple",
  "reasoning": "multihop",
  "difficulty": "easy"
}
```

**Example 2: Selecting the Right Prompting Strategy**

```
User: I have a multilingual QA system that handles Boolean, multiple-choice,
      and factoid questions in Persian. Which prompting strategy should I use
      for each type?

Approach:
1. Boolean questions -> Chain-of-Thought (CoT)
   System prompt (in Persian):
   "شما یک سیستم پرسش و پاسخ هستید. مرحله به مرحله استدلال کنید و سپس
    با بله یا خیر پاسخ دهید."

   The model reasons through evidence before committing to yes/no,
   reducing premature answers on comparative or negation variants.

2. Multiple-choice questions -> Chain-of-Thought (CoT)
   System prompt (in Persian):
   "هر گزینه را بررسی کنید، دلایل رد یا قبول هر کدام را بنویسید،
    سپس پاسخ نهایی را انتخاب کنید."

   CoT helps the model systematically eliminate wrong options rather
   than pattern-matching to a plausible-looking answer.

3. Factoid questions -> Few-shot (2-3 examples)
   System prompt (in Persian) with examples:
   "به سوالات زیر با پاسخ کوتاه جواب دهید.
    سوال: پایتخت ژاپن کجاست؟ پاسخ: توکیو
    سوال: سیاره‌های منظومه شمسی را نام ببرید. پاسخ: عطارد، زهره، زمین، ...
    سوال: [actual question]"

   Few-shot teaches the expected answer format (short phrase or list)
   without the overhead of CoT reasoning that can derail factoid recall.

Key rule: ALWAYS write prompts in Persian (the question language),
not in English. This consistently improves accuracy by 3-8% across models.
```

**Example 3: Fine-Tuning a Model for Persian QA**

```
User: I want to fine-tune Dorna (a Persian LLM) on reasoning QA tasks.
      How should I structure the training data?

Approach:
1. Load the PARSE training split (8,640 items):
   from datasets import load_dataset
   ds = load_dataset("JamshidJDMY/Parse", split="train")

2. Convert to instruction-tuning format with format-specific prompts:

   For Boolean items:
   {
     "messages": [
       {"role": "system", "content": "مرحله به مرحله استدلال کنید..."},
       {"role": "user", "content": "<question>"},
       {"role": "assistant", "content": "استدلال: <reasoning>\nپاسخ: بله"}
     ]
   }

   For factoid items:
   {
     "messages": [
       {"role": "system", "content": "با پاسخ کوتاه جواب دهید..."},
       {"role": "user", "content": "<question>"},
       {"role": "assistant", "content": "<short answer>"}
     ]
   }

3. Fine-tune with standard parameters:
   - Use the train split (8,640 items) for training
   - Evaluate on the test split (2,160 items) per configuration
   - Apply the same format-specific prompting during inference

4. Evaluate with per-type metrics:
   - Boolean/MC single-answer: Accuracy
   - MC multi-answer / Factoid list: Jaccard similarity
   - Factoid simple: String containment
   - Non-answerable: Accuracy (model must output "non-answerable")

Expected gains: Fine-tuned Dorna outperforms base Dorna by 10-20%
on hard questions, with the largest gains on multi-hop reasoning items.
```

## Best Practices

**Do:**
- Write all prompts and system instructions in the same language as the questions. Target-language prompts outperform English prompts even for multilingual models.
- Match prompting strategy to question format: CoT for Boolean/multiple-choice, few-shot for factoid. This is the single highest-impact optimization.
- Calibrate difficulty through reasoning depth and compositionality, not through obscure vocabulary or convoluted grammar. A hard question should require more reasoning steps, not more dictionary lookups.
- Generate questions in balanced batches with explicit difficulty quotas (e.g., 10 easy / 10 medium / 10 hard per batch) to prevent distribution skew.
- Use type-appropriate evaluation metrics. Accuracy for single-answer items, Jaccard similarity for set-valued answers, string containment for free-form factoid.

**Avoid:**
- Using a single prompting strategy for all question types. Zero-shot or CoT applied uniformly leaves significant performance on the table for factoid questions.
- Translating English QA datasets instead of generating natively. Literal translations produce unnatural phrasing and culturally misaligned topics. Generate in the target language from the start.
- Skipping deduplication. LLMs frequently regenerate near-identical questions across batches, especially for common topics.
- Defining difficulty by vocabulary complexity alone. PARSE showed that annotator performance correctly tracked difficulty only when difficulty was defined by reasoning depth.

## Error Handling

- **Generated questions fail structural validation**: Increase specificity in the output format instructions. Explicitly provide a JSON schema or CSV template with example rows. Reject and regenerate entire batches with >20% structural failures.
- **Low Jaccard scores on multi-answer items**: Check if the model is generating valid answers in a different order or with minor spelling variations. Normalize answers (lowercase, remove diacritics for Persian) before computing Jaccard.
- **CoT produces correct reasoning but wrong final answer**: Add an explicit extraction step that parses the final answer from the CoT output. Use a regex or secondary prompt to extract "پاسخ: X" from the reasoning chain.
- **Non-answerable questions get answered confidently**: Include non-answerable examples in few-shot prompts so the model learns when to abstain. Fine-tuning on non-answerable items significantly improves abstention calibration.
- **Persian text encoding issues**: Ensure UTF-8 encoding throughout the pipeline. Persian uses right-to-left script with characters like ی ,گ ,پ ,چ ,ژ that break in non-Unicode-aware tools.

## Limitations

- The format-specific prompting findings (CoT for Boolean/MC, few-shot for factoid) were validated on Persian data. The optimal strategy may differ for other languages or domains, though the principle of matching strategy to format likely generalizes.
- The generation pipeline relies on GPT-4o or equivalent high-capability models. Lower-capability generators produce more items that fail quality filtering, increasing cost.
- Multi-stage filtering catches structural and duplicate errors effectively but cannot fully guarantee factual accuracy. Domain-expert review remains necessary for high-stakes benchmarks.
- The benchmark covers general open-domain knowledge. Specialized domains (medical, legal, scientific) would require domain-adapted generation prompts and domain-expert validation.
- Fine-tuning gains were demonstrated for 8B-parameter models. Scaling behavior for larger or smaller models was not characterized.

## Reference

**Paper**: [PARSE: An Open-Domain Reasoning Question Answering Benchmark for Persian](https://arxiv.org/abs/2602.01246v1) (Mozafari, Mousavinasab, Jatowt, 2026). Submitted to SIGIR 2026.

**Repository**: [github.com/DataScienceUIBK/Parse](https://github.com/DataScienceUIBK/Parse) -- contains generation prompts, evaluation scripts, and the full dataset.

**Dataset**: Available on HuggingFace as `JamshidJDMY/Parse`.

**Key takeaway**: Look at Tables 4-6 for the per-configuration prompting strategy comparison, and Figure 3 for fine-tuning gains. The most actionable finding is the interaction between question format and prompting strategy -- CoT for constrained-answer types, few-shot for open-ended types, always in the target language.