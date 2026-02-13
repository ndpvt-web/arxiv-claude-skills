---
name: "lata-tool-llm-assisted-translation"
description: >
  LLM-assisted translation annotation: build parallel corpus annotation pipelines
  with template-based prompt management, structured JSON output constraints,
  stand-off annotation architecture, and human-in-the-loop review for sentence
  alignment, word alignment, translation shift classification, and error taxonomy.
  Trigger phrases: "annotate this translation", "align these parallel sentences",
  "classify translation shifts", "build a translation annotation pipeline",
  "create parallel corpus annotations", "translation quality annotation"
---

# LATA: LLM-Assisted Translation Annotation

This skill enables Claude to build and operate LLM-assisted translation annotation pipelines following the LATA methodology. The core technique uses a template-based Prompt Manager with strict JSON output schemas to drive sentence segmentation, alignment, translation shift classification, and error annotation across structurally divergent language pairs (e.g., Arabic-English, Chinese-English, Japanese-English). Instead of replacing human annotators, the LLM generates structured first-pass annotations that humans review, accept, modify, or reject -- combining automation speed with linguistic precision.

## When to Use

- When the user asks to align sentences or words between a source text and its translation
- When the user needs to annotate translation shifts (additions, omissions, reorderings, negation changes) in a parallel corpus
- When building a translation quality assessment pipeline with structured error taxonomies
- When the user wants to segment paragraphs into sentence-level alignment units across two languages
- When creating stand-off annotation layers that separate linguistic markup from source text
- When the user needs to produce CES-compliant XML or structured JSON annotation output from parallel texts
- When classifying translation techniques (transposition, modulation, equivalence, adaptation) across sentence pairs
- When the user asks to design prompts that enforce strict JSON schemas for translation-related NLP tasks

## Key Technique

**Template-Based Prompt Manager with JSON Schema Constraints.** LATA's central innovation is a Prompt Manager that constructs LLM prompts from reusable templates containing dynamic placeholders (source language, target language, paragraph content, annotation taxonomy definitions). Each template enforces a strict JSON output schema so that the LLM's response is always machine-parseable and compatible with downstream processing. For sentence segmentation, the LLM receives an aligned paragraph pair and returns a JSON array of sentence objects with IDs and alignment mappings (1:1, 1:N, M:1, or M:N). This eliminates the brittle regex-based or statistical segmentation that fails on structurally divergent pairs.

**Stand-Off Annotation Architecture.** Annotations are stored separately from the source and target texts, linked by character offsets or sentence/word IDs. This allows multiple annotation layers (alignment, translation shifts, semantic roles, error categories) to coexist without modifying the original text. Overlapping annotations -- common in translation analysis where a single span may involve both a syntactic shift and an omission -- are naturally represented. The architecture supports iterative refinement: an annotator can revise one layer without invalidating others.

**Human-in-the-Loop Review Cycle.** The LLM generates first-pass annotations, but every annotation passes through human review. The workflow is: (1) LLM proposes, (2) human accepts/rejects/modifies, (3) optional LLM re-prompting with clarification context. This balances efficiency (the LLM handles 60-80% of straightforward cases correctly) with precision (humans catch the nuanced cases that matter most for linguistic research).

## Step-by-Step Workflow

1. **Define the annotation schema as a JSON specification.** Enumerate every annotation layer you need: sentence alignment, word alignment, translation shifts, error categories. For each layer, define the valid labels, their descriptions, and example instances. Store this as a reusable schema file.

2. **Build prompt templates with dynamic placeholders.** Create template strings with slots like `{source_lang}`, `{target_lang}`, `{source_paragraph}`, `{target_paragraph}`, and `{taxonomy}`. The template must include explicit JSON output format instructions with a sample response shape, e.g.:
   ```
   Return a JSON array where each element has:
   {"source_ids": [int], "target_ids": [int], "type": "1:1"|"1:N"|"M:1"|"M:N"}
   ```

3. **Segment parallel text into aligned paragraphs.** Before sentence-level work, ensure paragraphs are aligned between source and target. Use document structure (paragraph breaks, section headers) or a simple LLM prompt to identify paragraph correspondences.

4. **Run LLM-driven sentence segmentation on each paragraph pair.** For each aligned paragraph pair, fill the segmentation template and call the LLM. Parse the returned JSON into sentence objects: `p_i = {s_i1, s_i2, ..., s_ik}` for source and `q_i = {t_i1, t_i2, ..., t_im}` for target. Validate that every character in the original paragraph is covered by exactly one sentence span.

5. **Generate sentence alignment mappings.** Using the segmented sentences, prompt the LLM to produce alignment links. Each link maps one or more source sentence IDs to one or more target sentence IDs. Enforce the JSON schema: `{"alignments": [{"src": [1], "tgt": [1,2], "type": "1:N", "confidence": 0.95}]}`.

6. **Annotate translation shifts for each aligned pair.** For each sentence alignment, prompt the LLM with the translation shift taxonomy (e.g., Addition, Omission, Negation, Inversion, Transposition, Modulation) and ask it to identify and classify shifts. Output format: `{"shifts": [{"type": "Addition", "source_span": null, "target_span": "furthermore,", "explanation": "Discourse marker added in translation"}]}`.

7. **Apply word-level alignment within sentence pairs.** For fine-grained annotation, prompt the LLM to link individual words or phrases between source and target, returning offset-based or token-index-based alignment pairs.

8. **Run human review on all LLM-generated annotations.** Present each annotation to the reviewer with accept/reject/modify options. Track acceptance rates per annotation type to calibrate prompt quality over time.

9. **Export annotations in stand-off format.** Produce output files where the original texts are stored separately from annotation layers. Use CES-compliant XML or a JSON-based stand-off format with character offsets linking annotations back to source spans.

10. **Iterate and refine prompts based on review feedback.** Collect rejection patterns (e.g., the LLM consistently misclassifies Modulation as Transposition). Update the taxonomy descriptions and examples in your prompt templates to address systematic errors.

## Concrete Examples

**Example 1: Sentence Alignment for Arabic-English Parallel Text**

User: "I have an Arabic paragraph and its English translation. Align them at the sentence level."

Approach:
1. Define the segmentation prompt template:
   ```
   You are a translation alignment expert. Given the following {source_lang}
   paragraph and its {target_lang} translation, segment each into individual
   sentences and produce alignment mappings.

   Source ({source_lang}):
   {source_paragraph}

   Target ({target_lang}):
   {target_paragraph}

   Return ONLY valid JSON in this exact format:
   {
     "source_sentences": [{"id": 1, "text": "..."}],
     "target_sentences": [{"id": 1, "text": "..."}],
     "alignments": [{"src": [1], "tgt": [1], "type": "1:1"}]
   }
   ```
2. Fill placeholders with Arabic source and English target text.
3. Parse the JSON response and validate coverage (all text accounted for).

Output:
```json
{
  "source_sentences": [
    {"id": 1, "text": "وصل الوفد إلى المطار صباح اليوم."},
    {"id": 2, "text": "وقد استقبلهم المسؤولون المحليون بحفاوة بالغة."}
  ],
  "target_sentences": [
    {"id": 1, "text": "The delegation arrived at the airport this morning."},
    {"id": 2, "text": "Local officials warmly welcomed them."}
  ],
  "alignments": [
    {"src": [1], "tgt": [1], "type": "1:1"},
    {"src": [2], "tgt": [2], "type": "1:1"}
  ]
}
```

**Example 2: Translation Shift Classification**

User: "Classify the translation shifts between these aligned Chinese-English sentence pairs."

Approach:
1. Define shift taxonomy in the prompt:
   ```
   Classify each translation shift using these categories:
   - Addition: content present in target but absent in source
   - Omission: content present in source but absent in target
   - Transposition: change in grammatical structure (e.g., passive to active)
   - Modulation: change in point of view or cognitive category
   - Negation: affirmative/negative polarity reversal
   - Inversion: reversal of element order
   ```
2. For each aligned pair, send the source, target, and taxonomy to the LLM.
3. Parse the shift annotations and present for human review.

Output:
```json
{
  "source": "他没有去过中国。",
  "target": "He has never been to China.",
  "shifts": [
    {
      "type": "Transposition",
      "source_span": "没有去过",
      "target_span": "has never been to",
      "explanation": "Negated experiential aspect restructured from verb-complement to auxiliary-adverb-participle construction"
    }
  ]
}
```

**Example 3: Building a Full Annotation Pipeline in Python**

User: "Build me a Python script that annotates translation shifts for a CSV of parallel sentences."

Approach:
1. Read CSV with `source` and `target` columns.
2. Define prompt templates with JSON schema constraints.
3. Call LLM API for each row, parse structured output.
4. Write stand-off annotations to a separate JSON file.

Output:
```python
import json
import csv
from pathlib import Path

SHIFT_TAXONOMY = {
    "Addition": "Content present in target but absent in source",
    "Omission": "Content present in source but absent in target",
    "Transposition": "Change in grammatical structure",
    "Modulation": "Change in point of view or cognitive category",
    "Negation": "Affirmative/negative polarity reversal",
    "Inversion": "Reversal of element order",
}

PROMPT_TEMPLATE = """Analyze the translation shifts between source and target.

Source ({source_lang}): {source}
Target ({target_lang}): {target}

Taxonomy:
{taxonomy}

Return ONLY valid JSON:
{{"shifts": [{{"type": "<category>", "source_span": "<text or null>", "target_span": "<text or null>", "explanation": "<brief>"}}]}}"""

def build_prompt(source, target, source_lang="Arabic", target_lang="English"):
    taxonomy_str = "\n".join(f"- {k}: {v}" for k, v in SHIFT_TAXONOMY.items())
    return PROMPT_TEMPLATE.format(
        source_lang=source_lang, target_lang=target_lang,
        source=source, target=target, taxonomy=taxonomy_str
    )

def annotate_corpus(input_csv, output_json, llm_call_fn):
    annotations = []
    with open(input_csv) as f:
        for row in csv.DictReader(f):
            prompt = build_prompt(row["source"], row["target"])
            response = llm_call_fn(prompt)
            parsed = json.loads(response)
            annotations.append({
                "source": row["source"],
                "target": row["target"],
                "shifts": parsed["shifts"],
                "status": "pending_review"
            })
    Path(output_json).write_text(json.dumps(annotations, indent=2, ensure_ascii=False))
    return annotations
```

## Best Practices

- **Do:** Always enforce a strict JSON output schema in every prompt template. Include a concrete example of the expected output shape directly in the prompt. This prevents free-text responses that break downstream parsing.
- **Do:** Define taxonomy categories with both a short label and a detailed description including at least one example. Vague labels like "Other" lead to inconsistent LLM classification.
- **Do:** Use stand-off annotation format (annotations stored separately, linked by IDs/offsets) rather than inline markup. This prevents annotation layers from interfering with each other.
- **Do:** Track per-category acceptance rates during human review. If a shift type drops below 70% acceptance, revise its taxonomy description and prompt examples.
- **Avoid:** Sending entire documents in a single prompt. Segment into paragraph-level or sentence-level chunks first. Large context windows degrade alignment precision.
- **Avoid:** Treating LLM output as ground truth. Every annotation must pass through human review, especially for structurally divergent language pairs where LLMs exhibit systematic biases (e.g., under-detecting omissions in Arabic-English).
- **Avoid:** Hardcoding annotation categories. Use configurable taxonomy files so the pipeline works across different translation studies and theoretical frameworks.

## Error Handling

- **Malformed JSON from LLM:** Wrap all LLM response parsing in try/except. On failure, re-prompt once with an explicit correction: "Your previous response was not valid JSON. Return ONLY the JSON object with no surrounding text." If the second attempt fails, flag the pair for manual annotation.
- **Incomplete coverage in segmentation:** After sentence segmentation, verify that concatenating all sentence spans reconstructs the original paragraph. If characters are missing or duplicated, re-prompt with the specific gap highlighted.
- **Alignment ID mismatches:** Validate that all sentence IDs referenced in alignment links exist in both the source and target sentence lists. Reject and re-prompt on dangling references.
- **Taxonomy drift:** If the LLM invents categories not in your taxonomy, strip unknown labels and re-classify those spans. Log invented categories -- they may indicate gaps in your taxonomy that need addressing.
- **Encoding issues with RTL text:** When working with Arabic, Hebrew, or other RTL scripts, ensure character offsets are computed on the raw Unicode string, not on any display-transformed version. Test offset round-tripping before processing a full corpus.

## Limitations

- **No evaluation benchmarks in the original paper.** LATA is presented as a tool without published precision/recall/F1 numbers. You should validate annotation quality on a held-out sample for your specific language pair and domain.
- **LLM quality varies by language pair.** The approach works best for language pairs well-represented in LLM training data. Low-resource pairs (e.g., Swahili-Basque) will produce noisier first-pass annotations.
- **Taxonomy design is manual.** The system does not auto-discover shift categories. You must define the taxonomy based on your theoretical framework (Vinay & Darbelnet, Catford, etc.) before annotation begins.
- **Sentence segmentation is a bottleneck for agglutinative languages.** Languages without clear sentence-final punctuation (e.g., classical Chinese, Thai) require additional segmentation heuristics beyond what the basic prompt template provides.
- **Stand-off offsets are fragile.** Any text normalization (whitespace collapsing, Unicode normalization) after offset computation will invalidate annotations. Lock down text preprocessing before annotation begins.
- **Cost scales linearly.** Each sentence pair requires at least one LLM call per annotation layer. For large corpora (100K+ pairs), batch processing and caching are essential.

## Reference

- **Paper:** [LATA: A Tool for LLM-Assisted Translation Annotation](https://arxiv.org/abs/2602.10454v1) (Huang & Asiri, 2026). Key takeaway: the template-based Prompt Manager with strict JSON schema constraints and stand-off architecture for multi-layer translation annotation. Focus on Sections 3-4 for the prompt design patterns and annotation workflow.