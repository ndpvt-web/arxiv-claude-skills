---
name: "masalbench-benchmark-contextual-cross-cultural"
description: "Build cross-cultural figurative language benchmarks and evaluation pipelines for LLMs. Applies the MasalBench methodology to test whether models truly understand proverbs, idioms, and culturally-embedded expressions -- not just pattern-match surface text. Trigger phrases: 'evaluate LLM cultural understanding', 'benchmark figurative language', 'test proverb comprehension', 'cross-cultural NLP evaluation', 'build idiom benchmark', 'low-resource language evaluation'"
---

# Cross-Cultural Figurative Language Benchmarking (MasalBench Method)

This skill enables Claude to design and implement rigorous benchmarks that evaluate LLM understanding of figurative language across cultures. Based on the MasalBench paper, the core technique uses two complementary evaluation axes -- contextual understanding (can the model grasp what a proverb *means* in a dialogue?) and cross-cultural mapping (can it find the equivalent expression in another language?) -- with carefully stratified distractors that distinguish genuine comprehension from surface-level pattern matching. This approach generalizes beyond Persian proverbs to any domain where cultural or figurative reasoning must be tested.

## When to Use

- When the user needs to evaluate whether an LLM actually understands idioms, proverbs, or figurative expressions in any language
- When building a benchmark dataset for low-resource or non-English language understanding
- When designing multiple-choice evaluations that need high-quality distractors (literal, plausible-but-wrong, irrelevant)
- When the user asks to test cross-lingual equivalence -- e.g., mapping Chinese chengyu to English idioms, or Arabic proverbs to Spanish refranes
- When assessing whether a model relies on shallow lexical cues vs. genuine cultural/analogical reasoning
- When building a dialogue-embedded evaluation pipeline rather than isolated sentence-level tests

## Key Technique

**Two-axis evaluation with stratified distractors.** MasalBench splits figurative language evaluation into two tasks that probe fundamentally different capabilities. The *contextual understanding* task embeds proverbs in natural multi-turn dialogues and asks the model to identify the speaker's communicative intent from four options. The *cross-cultural understanding* task presents a proverb and asks the model to select the semantically equivalent expression from another language. This dual-axis design reveals a consistent gap: models scoring 90%+ on contextual understanding drop to ~79% on cross-cultural mapping, exposing weaknesses in analogical and cultural reasoning that single-task benchmarks miss.

**Distractor taxonomy is the key differentiator.** Rather than random wrong answers, MasalBench uses three distractor types for contextual questions: (1) *literal interpretations* that take the figurative expression at face value, (2) *plausible-but-wrong meanings* that share thematic overlap but differ in actual intent, and (3) *irrelevant options* that serve as sanity checks. For cross-cultural questions, distractors share "similar wording, tone, or imagery but convey a different meaning." This stratification lets you diagnose *how* a model fails -- literal distractors catch models that lack figurative understanding entirely, while plausible distractors catch models that grasp the general domain but miss the precise meaning.

**Dialogue-embedded testing over isolated prompts.** Instead of presenting proverbs in isolation ("What does X mean?"), MasalBench wraps them in realistic conversational contexts. This is critical because figurative expressions are pragmatically situated -- the same proverb can serve different communicative functions depending on context. The dialogue format also prevents models from relying on memorized dictionary definitions and forces genuine contextual inference.

## Step-by-Step Workflow

1. **Curate the expression corpus.** Collect 200-1000+ figurative expressions (proverbs, idioms, slang) from the target language. Filter for expressions that are (a) in active use, (b) have genuinely figurative meaning distinct from literal reading, and (c) span a range of cultural specificity. Use native speaker verification at this stage.

2. **Generate dialogue contexts.** For each expression, create 1-2 natural multi-turn dialogues (2-4 turns) where a speaker uses the expression with clear communicative intent. Use an LLM to draft dialogues (temperature ~1.0 for diversity), then have native speakers verify naturalness and that the proverb usage is idiomatic, not forced.

3. **Construct stratified distractors for contextual questions.** For each dialogue, create exactly four answer options: (a) the correct interpretation of the speaker's intent, (b) a literal reading of the figurative expression, (c) a plausible but incorrect interpretation that shares thematic overlap, (d) an irrelevant option. Ensure distractors are similar in length and grammatical structure to avoid superficial tell-signs.

4. **Build cross-cultural mapping pairs.** For each source-language expression, identify the closest semantic equivalent in the target language. This is *not* translation -- it is finding an expression in the other culture that serves the same pragmatic function. Exclude expressions with no clear cross-cultural equivalent rather than forcing bad matches. Create one correct pair and one distractor that shares surface similarity (wording, imagery) but differs in meaning.

5. **Implement positional bias controls.** Randomize the order of answer options across all questions. This prevents models from learning positional shortcuts (e.g., "the correct answer is usually option B"). Store the randomization seed for reproducibility.

6. **Configure evaluation prompting.** Use zero-shot prompts with temperature 0 for deterministic evaluation. Limit output tokens to the minimum needed (e.g., 5 tokens for a single letter answer). Frame the prompt as a clear task instruction followed by the dialogue and options -- do not include few-shot examples that could leak patterns.

7. **Run the benchmark and compute per-category accuracy.** Score each model on both tasks independently. Break down contextual accuracy by distractor type to diagnose failure modes: high error on literal distractors means the model lacks figurative understanding; high error on plausible distractors means it grasps the domain but not the nuance.

8. **Perform error analysis.** Examine the confusion matrix between distractor categories. Calculate the gap between contextual and cross-cultural scores -- a large gap indicates the model has language-specific memorization but lacks transferable cultural reasoning. Flag any positional bias by checking if accuracy varies by option position.

9. **Generate the evaluation report.** Produce a structured summary with per-model scores, per-task breakdowns, distractor-type confusion matrices, and cross-cultural gap analysis. Include specific failure examples with the dialogue, the model's wrong answer, and why it was wrong.

10. **Iterate on distractor quality.** If all models score >95% on a question, the distractors are too easy -- replace them. If all models fail a question, verify the gold answer is correct. Target a benchmark difficulty where top models score 75-90%, leaving room to measure improvement.

## Concrete Examples

**Example 1: Building a Chinese Chengyu Benchmark**

User: "I want to evaluate whether GPT-4 and Claude actually understand Chinese chengyu (four-character idioms) or just pattern-match. Help me build a benchmark."

Approach:
1. Curate 300 common chengyu from HSK lists and native corpora, filtering for ones with strong figurative meaning (e.g., "画蛇添足" -- drawing legs on a snake, meaning to ruin something by overdoing it)
2. For each chengyu, generate a 3-turn dialogue in Chinese where a speaker uses it naturally:
   ```
   A: 你的报告已经很完美了，为什么还要改？
   B: 我觉得可以再加点数据。
   A: 别画蛇添足了，交上去吧。
   Question: Why does Speaker A use "画蛇添足"?
   (a) To warn that adding unnecessary changes will hurt the report [correct]
   (b) To describe someone literally drawing a snake [literal distractor]
   (c) To suggest the report needs more visual illustrations [plausible distractor]
   (d) To compliment Speaker B's thoroughness [irrelevant distractor]
   ```
3. For cross-cultural mapping, pair with English equivalents:
   ```
   "画蛇添足" is closest in meaning to:
   (a) "Don't gild the lily" [correct]
   (b) "Don't count your chickens before they hatch" [surface-similar: animal imagery, different meaning]
   ```
4. Run zero-shot evaluation with temperature=0, max_tokens=5

Output: A benchmark JSON with 300 contextual + 200 cross-cultural questions, evaluation script, and results table.

**Example 2: Diagnosing Cross-Cultural Reasoning Gaps**

User: "We tested our multilingual model on Arabic proverbs and it scores 92% on understanding them in context but only 68% on matching them to English equivalents. Help me analyze why."

Approach:
1. Pull the per-question results for both tasks and align them by proverb
2. For proverbs where contextual=correct but cross-cultural=wrong, examine the distractor chosen:
   ```
   Proverb: "اللي بيته من إزاز ما يحدفش الناس بطوب"
   (He who lives in a glass house shouldn't throw stones at people)
   Context task: Model correctly identified "warning against hypocrisy" -> PASS
   Cross-cultural task: Model chose "People in glass houses shouldn't throw stones" -> PASS

   Proverb: "الصبر مفتاح الفرج"
   (Patience is the key to relief)
   Context task: Model correctly identified "advising patience during hardship" -> PASS
   Cross-cultural task: Model chose "Patience is a virtue" over "Good things come to those who wait"
   -> Both are plausible but the second matches the pragmatic function better
   ```
3. Categorize errors: (a) model picks expressions with overlapping keywords but different pragmatics, (b) model defaults to the most famous English proverb in the same domain, (c) model fails on culture-specific concepts with no clean English equivalent
4. Report the distribution of error types and recommend targeted training data

Output: Error taxonomy with percentages, 10 illustrative failure cases, and recommendations.

**Example 3: Generating Evaluation Harness Code**

User: "Give me a Python script that runs a MasalBench-style evaluation on any JSONL dataset of proverb questions."

Approach:
1. Define the JSONL schema: `{"id", "task_type", "dialogue", "proverb", "options", "correct_answer", "distractor_types"}`
2. Implement the evaluation loop:

```python
import json, random, os
from collections import defaultdict

def load_benchmark(path: str) -> list[dict]:
    with open(path) as f:
        return [json.loads(line) for line in f]

def build_prompt(item: dict) -> str:
    options = list(enumerate(item["options"]))
    random.shuffle(options)
    shuffled_map = {chr(65+i): orig_idx for i, (orig_idx, _) in enumerate(options)}
    option_text = "\n".join(f"{chr(65+i)}. {opt}" for i, (_, opt) in enumerate(options))
    correct_letter = next(k for k, v in shuffled_map.items() if v == item["correct_answer"])

    if item["task_type"] == "contextual":
        prompt = (f"Read the following dialogue and answer the question.\n\n"
                  f"{item['dialogue']}\n\n"
                  f"Question: Why does the speaker use the proverb \"{item['proverb']}\"?\n\n"
                  f"{option_text}\n\nAnswer with only the letter.")
    else:
        prompt = (f"Which expression is closest in meaning to \"{item['proverb']}\"?\n\n"
                  f"{option_text}\n\nAnswer with only the letter.")
    return prompt, correct_letter, shuffled_map

def evaluate(dataset: list[dict], model_fn) -> dict:
    results = defaultdict(lambda: {"correct": 0, "total": 0, "by_distractor": defaultdict(int)})
    for item in dataset:
        prompt, correct, smap = build_prompt(item)
        response = model_fn(prompt, max_tokens=5, temperature=0).strip()[0].upper()
        task = item["task_type"]
        results[task]["total"] += 1
        if response == correct:
            results[task]["correct"] += 1
        elif "distractor_types" in item:
            chosen_idx = smap.get(response, -1)
            if chosen_idx >= 0 and chosen_idx < len(item.get("distractor_types", [])):
                dtype = item["distractor_types"][chosen_idx]
                results[task]["by_distractor"][dtype] += 1
    return {task: {
        "accuracy": d["correct"] / d["total"] if d["total"] else 0,
        "n": d["total"],
        "error_by_distractor": dict(d["by_distractor"])
    } for task, d in results.items()}
```

3. Output results as a formatted table with per-task accuracy and distractor-type error breakdown.

## Best Practices

- **Do:** Always include all three distractor types (literal, plausible, irrelevant) -- removing any one collapses diagnostic power. The plausible distractor is hardest to write well; invest time here.
- **Do:** Randomize option order per-question and verify there is no positional bias in your results. Check that accuracy per option position (A/B/C/D) is roughly uniform.
- **Do:** Have native speakers verify every dialogue and gold answer. LLM-generated dialogues are a good starting draft but frequently produce unnatural proverb usage.
- **Do:** Report contextual and cross-cultural scores separately -- the gap between them is itself a key finding about model capabilities.
- **Avoid:** Using translation as a proxy for cross-cultural equivalence. "The early bird catches the worm" is not a *translation* of any Persian proverb -- it is a functional equivalent. Mapping must be done at the pragmatic/semantic level, not lexical.
- **Avoid:** Including proverbs that have become internationalized (e.g., "Actions speak louder than words" exists nearly identically in many languages). These inflate cross-cultural scores without testing real cultural knowledge.
- **Avoid:** Setting temperature > 0 during evaluation. Non-deterministic outputs make results unreproducible and add noise to accuracy measurements.

## Error Handling

- **Model refuses to answer or outputs more than a letter:** Parse the first capital letter from the response. If no valid letter is found, mark as "invalid" and exclude from accuracy but report the refusal rate separately.
- **Positional bias detected (>10% accuracy variance across positions):** Re-run with a different randomization seed. If bias persists, the model has a systematic positional preference -- report it as a finding.
- **Native speaker disagrees with gold answer:** This is common for proverbs with context-dependent meanings. Allow multiple correct answers where linguistically justified, and use majority vote among 3+ annotators.
- **Cross-cultural pair has no clear equivalent:** Exclude the proverb from the cross-cultural task rather than forcing a weak match. A benchmark with 700 strong pairs is better than 1000 pairs with 300 questionable ones.
- **All models score >95% on a subset:** Those questions are too easy. Replace distractors with harder plausible alternatives or remove the questions entirely.

## Limitations

- This methodology is best suited for figurative expressions with clear pragmatic functions. It is less effective for slang, neologisms, or expressions whose meaning is highly variable across speakers.
- Cross-cultural mapping assumes the target language has functional equivalents. For deeply culture-specific concepts (e.g., Japanese "wabi-sabi" as a proverb), the cross-cultural task may not apply.
- The benchmark tests recognition and matching, not generation. A model could score well on MasalBench-style tasks but still misuse proverbs in open-ended generation.
- Distractor quality is the primary bottleneck. Poor distractors yield inflated scores; the benchmark is only as good as its hardest plausible distractor.
- Zero-shot evaluation means models cannot learn from the test format. This is intentional for measuring latent understanding, but few-shot results may differ significantly.

## Reference

**Paper:** [MasalBench: A Benchmark for Contextual and Cross-Cultural Understanding of Persian Proverbs in LLMs](https://arxiv.org/abs/2601.22050v1) by Ghazal Kalhor and Behnam Bahrak (2026). Look for: the two-task evaluation structure, the three-tier distractor taxonomy, and the consistent contextual-vs-cross-cultural performance gap across all eight evaluated models. Code: [github.com/kalhorghazal/MasalBench](https://github.com/kalhorghazal/MasalBench).