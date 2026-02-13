---
name: "do-truly-benefit-longer"
description: "Optimize LLM context length for post-editing and refinement pipelines. Applies research showing that naively adding document-level context often fails to improve LLM output quality while dramatically increasing cost and latency. Use when: 'optimize my translation post-editing pipeline', 'reduce LLM API costs for text refinement', 'should I use full document context for editing', 'build an automatic post-editing system', 'design a cost-efficient LLM correction pipeline', 'evaluate whether longer context helps my LLM task'."
---

# Context-Length Optimization for LLM Post-Editing Pipelines

This skill teaches Claude to design and optimize LLM-based pipelines that refine, correct, or post-edit text outputs (translations, drafts, code, summaries) by applying the key finding from Kim & Kim (2026): **naively providing more document-level context to LLMs does not reliably improve post-editing quality, but dramatically increases cost and latency.** Instead of defaulting to maximum context, this skill guides building pipelines that strategically choose between sentence-level and document-level prompting, measure actual quality gains, and avoid wasting tokens on context the model ignores.

## When to Use

- When the user is building or optimizing an automatic post-editing (APE) pipeline for machine translation
- When the user wants to reduce LLM API costs for a text correction or refinement workflow
- When the user asks whether to include full document context or surrounding paragraphs in an LLM editing prompt
- When the user is designing a multi-pass LLM pipeline that corrects outputs from a first-pass model
- When the user needs to benchmark sentence-level vs. document-level prompting for any refinement task
- When the user is evaluating cost/quality tradeoffs for proprietary vs. open-weight models in a correction pipeline
- When the user wants to detect whether their LLM is actually exploiting the context they provide

## Key Technique

The paper systematically compares proprietary LLMs (GPT-4o) and open-weight models (Llama 3.1 405B, Qwen 2.5 72B) on the WMT Automatic Post-Editing task under two prompting strategies: **sentence-level** (the source sentence and its machine translation only) and **document-level** (the full surrounding document provided as additional context). Both use one-shot prompting with a single demonstration example. The surprising finding is that proprietary models achieve near human-level APE quality with simple sentence-level one-shot prompting, and adding document context produces negligible or no improvement. The models largely fail to exploit the additional context for correcting errors that require discourse-level understanding (pronoun resolution, terminology consistency, register matching).

This creates a practical decision framework: **start with sentence-level prompting, measure rigorously, and only add context when you can prove it helps.** The paper also reveals that standard automatic metrics (BLEU, TER, COMET) do not reliably capture the kinds of improvements that document context could theoretically provide (discourse coherence, lexical consistency). This means you cannot trust metric deltas alone when deciding whether longer context is worth its cost — you need targeted probes for the specific contextual phenomena you want to improve.

A secondary finding concerns robustness: proprietary models are more resistant to poisoned or adversarial examples in the prompt, but this same conservatism makes them less responsive to contextual signals. Open-weight models are more malleable — they respond more to context but are also more susceptible to noisy input. This tradeoff matters when choosing models for production pipelines.

## Step-by-Step Workflow

1. **Define the refinement task precisely.** Identify what kind of errors the post-editing step must fix: fluency errors, terminology mistakes, discourse-level inconsistencies (pronoun reference, register), or factual errors. Classify each error type as sentence-local (fixable from the sentence alone) or context-dependent (requires surrounding text).

2. **Build a sentence-level baseline first.** Construct a one-shot prompt containing: (a) a system instruction defining the editing task, (b) one high-quality example of input→corrected output, (c) the target sentence to edit. Measure quality with automatic metrics AND manual inspection of 50-100 samples.

   ```
   SYSTEM: You are a post-editor. Given a source text and its machine
   translation, correct errors in the translation. Output only the
   corrected translation.

   EXAMPLE:
   Source: [source sentence]
   MT: [machine translation with errors]
   Corrected: [human post-edited reference]

   NOW EDIT:
   Source: {source}
   MT: {mt_output}
   Corrected:
   ```

3. **Design a document-level variant with minimal overhead.** Add the 2-3 surrounding sentences (not the entire document) as context. Use clear delimiters to separate context from the target segment:

   ```
   SYSTEM: You are a post-editor. Given context, a source text, and its
   machine translation, correct errors in the translation. Use the
   context only when it helps resolve ambiguity. Output only the
   corrected translation of the TARGET segment.

   CONTEXT (preceding): {prev_2_sentences}
   CONTEXT (following): {next_1_sentence}

   TARGET:
   Source: {source}
   MT: {mt_output}
   Corrected:
   ```

4. **Create targeted evaluation probes.** Build a small test set (20-50 examples) containing known context-dependent errors: wrong pronoun gender, inconsistent terminology, incorrect formality register. These probes will tell you whether the model actually uses context, independent of aggregate metric scores.

5. **Run A/B comparison with cost tracking.** Execute both sentence-level and document-level variants on the full test set. Record per-request: (a) input token count, (b) output token count, (c) latency, (d) automatic metric scores, (e) probe accuracy for context-dependent errors. Calculate total cost difference.

6. **Analyze whether context provides statistically significant improvement.** Use a paired statistical test (Wilcoxon signed-rank or Friedman test as in the paper) on per-segment quality scores. Do NOT rely on aggregate metric differences alone — a 0.3 BLEU improvement may not be statistically significant or practically meaningful.

7. **Check for context exploitation specifically.** On the targeted probes from step 4, compare accuracy between sentence-level and document-level. If the document-level variant does not fix more context-dependent errors, the model is ignoring the context and you are paying for unused tokens.

8. **Choose the cost-optimal configuration.** If document context does not demonstrably improve quality on your probes, use sentence-level prompting and save 40-80% on token costs. If context helps on specific error types, use a hybrid approach: sentence-level for most segments, document-level only for segments flagged as potentially context-dependent.

9. **Implement the hybrid pipeline in code.** Build a two-stage system: (a) a lightweight classifier or heuristic that flags segments likely to contain context-dependent errors (pronouns, demonstratives, domain terms), (b) sentence-level editing for unflagged segments, document-level editing for flagged ones.

10. **Monitor and iterate in production.** Track edit rate, cost per segment, and user override rate. If users frequently re-edit outputs that were processed at sentence level, investigate whether those edits involve contextual errors and adjust the flagging heuristic.

## Concrete Examples

**Example 1: Optimizing a translation post-editing API**

User: "I'm building an API that post-edits machine translations from English to Korean. Right now I send the full document to GPT-4o for each segment. It's costing us $2,000/month. Can you help optimize it?"

Approach:
1. Audit the current pipeline — the user sends ~500 tokens of document context per segment when only ~50 tokens (the segment itself) are needed for most corrections.
2. Build a sentence-level prompt variant with one-shot example.
3. Create 30 probe sentences with known context-dependent errors (Korean honorifics requiring document-level register knowledge, pronoun ambiguity).
4. Run both variants on a 500-segment test set, tracking quality and cost.
5. Find that sentence-level achieves 97% of document-level quality on general metrics, and document context only helps on ~8% of segments (honorific consistency).
6. Build a hybrid: flag segments containing honorific markers or ambiguous pronouns → send those with 2-sentence context, send everything else sentence-level.

Output:
```python
import openai

def classify_segment(segment: str, source: str) -> bool:
    """Flag segments likely needing document context."""
    # Korean honorific markers, pronouns, demonstratives
    context_triggers = ["그", "이", "저", "그녀", "그들"]
    return any(trigger in segment for trigger in context_triggers)

def post_edit(source: str, mt: str, context: str | None = None) -> str:
    if context:
        prompt = DOCUMENT_LEVEL_TEMPLATE.format(
            context=context, source=source, mt=mt
        )
    else:
        prompt = SENTENCE_LEVEL_TEMPLATE.format(source=source, mt=mt)

    response = openai.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=256,
    )
    return response.choices[0].message.content

def process_document(segments: list[dict]) -> list[str]:
    results = []
    for i, seg in enumerate(segments):
        needs_context = classify_segment(seg["mt"], seg["source"])
        context = None
        if needs_context:
            prev = segments[i-1]["mt"] if i > 0 else ""
            next_ = segments[i+1]["mt"] if i < len(segments)-1 else ""
            context = f"{prev} ||| {next_}"
        results.append(post_edit(seg["source"], seg["mt"], context))
    return results
```
Result: ~60% reduction in token usage, negligible quality loss on general segments, maintained quality on context-dependent segments.

**Example 2: Evaluating whether document context helps a code review bot**

User: "My code review bot sends the entire file to an LLM to review each function. Should I keep doing that or just send the function?"

Approach:
1. Recognize this is the same pattern as document-level vs. sentence-level APE — the "document" is the file, the "segment" is the function.
2. Create probes: 20 functions with bugs that require cross-function context (wrong variable scope, inconsistent API usage, mismatched types from imports) and 20 with local bugs (off-by-one, null checks).
3. Compare function-only vs. full-file prompting on both sets.
4. Measure: Does full-file context catch more cross-function bugs? At what token cost?

Output:
```python
# Evaluation harness for context-length comparison
import json
from pathlib import Path

def run_evaluation(test_cases: list[dict], model: str) -> dict:
    results = {"sentence_level": [], "document_level": []}

    for case in test_cases:
        # Sentence-level: function only
        sl_prompt = f"Review this function for bugs:\n```\n{case['function']}\n```"
        sl_result = call_llm(model, sl_prompt)
        results["sentence_level"].append({
            "found_bug": case["bug_description"] in sl_result.lower(),
            "tokens_in": count_tokens(sl_prompt),
            "bug_type": case["bug_type"],  # "local" or "cross_function"
        })

        # Document-level: full file
        dl_prompt = f"Review the function `{case['function_name']}` in this file:\n```\n{case['full_file']}\n```"
        dl_result = call_llm(model, dl_prompt)
        results["document_level"].append({
            "found_bug": case["bug_description"] in dl_result.lower(),
            "tokens_in": count_tokens(dl_prompt),
            "bug_type": case["bug_type"],
        })

    return compute_stats(results)
    # Expected output: document-level helps for cross_function bugs
    # but not for local bugs, at 3-10x token cost
```

**Example 3: Designing a cost-efficient multi-language APE system**

User: "We need to post-edit MT output for 6 language pairs. Budget is tight. How should we set this up?"

Approach:
1. Per the paper's findings, start all 6 pairs with sentence-level one-shot prompting.
2. Use a smaller open-weight model (Qwen 2.5 72B or similar) for high-volume pairs to reduce cost, accepting slightly lower robustness.
3. Reserve proprietary model calls for low-volume, high-stakes pairs.
4. For each pair, build 20 context-dependent probes specific to that language's discourse phenomena (e.g., Japanese honorifics, German compound nouns, Arabic gender agreement).
5. Only upgrade to document-level prompting for pairs where probes show >10% improvement.

Output:
```yaml
# ape_pipeline_config.yaml
language_pairs:
  en-de:
    model: qwen-2.5-72b
    context: sentence_level
    notes: "German compound consistency tested — no doc-level benefit found"
  en-ja:
    model: gpt-4o
    context: hybrid
    flag_triggers: ["honorific", "pronoun_ambiguity"]
    notes: "Honorific register requires 2-sentence context window"
  en-ko:
    model: qwen-2.5-72b
    context: hybrid
    flag_triggers: ["honorific_level"]
  en-zh:
    model: qwen-2.5-72b
    context: sentence_level
  en-fr:
    model: qwen-2.5-72b
    context: sentence_level
  en-ar:
    model: gpt-4o
    context: hybrid
    flag_triggers: ["gender_agreement", "dual_plural"]

defaults:
  max_context_sentences: 2
  one_shot_example: true
  max_output_tokens: 256
```

## Best Practices

- **Do:** Always start with sentence-level prompting as your baseline. It is cheaper, faster, and often equally effective. Add context only when measurement proves it helps.
- **Do:** Build targeted evaluation probes for context-dependent phenomena specific to your task (pronoun resolution, terminology consistency, register matching). Aggregate metrics will not reveal whether context is being used.
- **Do:** Use statistical significance tests (Wilcoxon signed-rank, Friedman) when comparing variants. Small metric differences on aggregate scores are often noise.
- **Do:** Track cost-per-quality-point, not just quality. A 0.5 BLEU gain that costs 3x more tokens may not be worth it.
- **Avoid:** Sending entire documents as context by default. Models tend to ignore distant context for local editing tasks, and you pay for every token.
- **Avoid:** Trusting BLEU/TER/COMET deltas alone to decide whether document context helps. These metrics are insensitive to discourse-level improvements — the exact improvements context is supposed to enable.
- **Avoid:** Assuming open-weight and proprietary models behave the same way with context. Open-weight models are more responsive to context but also more susceptible to noisy or adversarial content in the context window.

## Error Handling

- **Context window overflow:** When document context pushes the prompt beyond the model's context limit, truncate context (not the target segment). Prioritize preceding context over following context, as preceding sentences more often resolve ambiguity.
- **Model ignores context entirely:** If document-level and sentence-level produce identical outputs on >95% of segments, the model is not using context. Do not pay for it. Switch to sentence-level.
- **Metric disagreement:** When automatic metrics show improvement but manual review does not (or vice versa), trust manual review. Build a small human evaluation protocol (20-30 segments, 2 annotators, measure inter-annotator agreement with Cohen's kappa).
- **Open-weight model instability:** If an open-weight model produces inconsistent quality with document context (high variance across runs), reduce context window size or add explicit instructions to ignore context when uncertain.
- **Cost blowup in production:** Monitor token usage per request with alerts. A misconfigured pipeline that accidentally sends full documents can 10x your API bill overnight.

## Limitations

- The paper's findings are grounded in machine translation APE on WMT data. The degree to which context helps (or doesn't) may differ for other refinement tasks: code review, summarization editing, or style transfer may benefit more from surrounding context.
- The study uses naive document-level prompting (simply concatenating context). More sophisticated approaches — explicit instructions to check for discourse errors, chain-of-thought reasoning about context, or retrieval-augmented context selection — may extract more value from document context.
- Results are based on 2025-era models (GPT-4o, Llama 3.1, Qwen 2.5). Future models with improved long-context architectures may exploit document context more effectively.
- The hybrid pipeline approach (flag-then-route) introduces a new source of error: the flagging heuristic. False negatives (missing a context-dependent segment) cause quality loss; false positives only cost extra tokens.
- Human evaluation remains the gold standard for assessing discourse-level improvements, but it is expensive and slow. This creates a bootstrapping problem for iterating on context strategies.

## Reference

Kim, A., & Kim, S. (2026). *Do LLMs Truly Benefit from Longer Context in Automatic Post-Editing?* arXiv:2601.19410v1. [https://arxiv.org/abs/2601.19410v1](https://arxiv.org/abs/2601.19410v1)

Key takeaway: Read Section 4 (Results) for the sentence-level vs. document-level comparison tables, and Section 5 (Analysis) for the contextual behavior probes showing that models fail to exploit document context for discourse-level error correction.