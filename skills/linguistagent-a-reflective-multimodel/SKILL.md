---
name: "linguistagent-a-reflective-multimodel"
description: >
  Implements a reflective dual-agent (Annotator + Reviewer) workflow for automated
  linguistic annotation tasks such as metaphor identification, sentiment labeling,
  named entity recognition, and other sequence-labeling problems. The Annotator marks
  spans in text using XML tags and provides reasoning; the Reviewer critiques the
  annotations against a codebook, catching false positives and missed instances, then
  feeds corrections back for self-improvement.
  Trigger phrases: "annotate this text for metaphors", "dual-agent annotation pipeline",
  "reflective annotation workflow", "linguistic annotation with LLM review",
  "peer-review annotation system", "automated text labeling with self-correction"
---

# LinguistAgent: Reflective Dual-Agent Linguistic Annotation

This skill enables Claude to perform high-quality automated linguistic annotation by
implementing a reflective dual-agent architecture from the LinguistAgent paper. Instead
of a single-pass annotation, Claude alternates between an **Annotator** role (marking
target spans with XML tags and providing chain-of-thought justifications) and a
**Reviewer** role (critiquing annotations against a codebook, identifying false positives
and missed instances, and producing corrected output). This peer-review simulation
consistently outperforms single-agent annotation across zero-shot, few-shot, and
RAG-augmented paradigms.

## When to Use

- When the user asks to annotate text for **metaphors, metonymy, irony, or other figurative language**
- When the user needs **named entity recognition, sentiment labeling, or any token/span-level annotation** with quality assurance
- When the user wants to **compare annotation strategies** (zero-shot vs. few-shot vs. codebook-augmented)
- When the user provides a **codebook or annotation guidelines** and wants automated labeling that respects those rules
- When the user asks to **evaluate annotation quality** against a gold standard using Precision, Recall, and F1
- When the user wants a **self-correcting annotation pipeline** that catches and fixes its own mistakes

## Key Technique

The core insight of LinguistAgent is that annotation quality improves substantially when
you separate the task into two distinct agent roles that operate in sequence. The
**Annotator** receives the input text plus instructions (and optionally few-shot examples
or a full codebook) and produces annotations by wrapping target expressions in XML tags
(e.g., `<Metaphor>broken heart</Metaphor>`). Critically, the Annotator also emits a
structured reasoning field justifying each annotation decision based on the provided
linguistic protocol.

The **Reviewer** then receives both the original text and the Annotator's full output
(tags + reasoning). It performs comparative analysis: checking each tagged span against
the codebook to flag false positives, scanning the original text for missed instances to
flag false negatives, and producing a critique with a corrected version. This reflection
cycle exploits the fact that verification is easier than generation — the Reviewer
operates on a constrained evaluation task rather than open-ended labeling.

The system supports three paradigms of increasing knowledge injection: (1) **Zero/Few-shot
Prompt Engineering**, where the Annotator receives only instructions and optional
examples; (2) **RAG / Full-Context**, where the complete codebook is embedded in the
system prompt, leveraging long context windows; and (3) **Fine-tuning**, where a
domain-specialized model replaces the general-purpose Annotator. The Reviewer can use the
same or a different model, enabling cross-model validation. Evaluation uses token-level
binary sequences (1 = tagged, 0 = untagged) compared against gold standards to compute
Precision, Recall, and F1.

## Step-by-Step Workflow

1. **Define the annotation task and tag schema.** Establish the target phenomenon
   (e.g., metaphor, named entity, sentiment) and the XML tag names to use
   (e.g., `<Metaphor>`, `<Entity type="PER">`). If the user provides a codebook or
   annotation guidelines, parse them into a structured reference document.

2. **Select the annotation paradigm.** Choose zero-shot (instruction only), few-shot
   (instruction + 2-5 annotated examples), or full-context RAG (instruction + complete
   codebook). If the user has gold-standard examples, reserve some for few-shot and
   the rest for evaluation.

3. **Construct the Annotator system prompt.** Include: (a) the task definition,
   (b) the tag schema with examples of correct usage, (c) explicit instruction to wrap
   target spans in XML tags within the original text, and (d) instruction to provide a
   `reasoning` field as a JSON object explaining each annotation decision.

4. **Run the Annotator pass.** Send each text segment to the Annotator. Collect the
   annotated text (with XML tags inline) and the reasoning chain. Store both in a
   structured format (JSON with fields: `original_text`, `annotated_text`, `reasoning`).

5. **Construct the Reviewer system prompt.** Include: (a) the same codebook/guidelines
   as the Annotator, (b) explicit instruction to compare the annotated text against
   the original, (c) checklist: identify false positives (incorrectly tagged spans),
   false negatives (missed spans), and boundary errors (partially tagged spans), and
   (d) instruction to output a `critique` field and a `corrected_text` field.

6. **Run the Reviewer pass.** Send the original text, the Annotator's annotated text,
   and the Annotator's reasoning to the Reviewer. Collect the critique and corrected
   annotations.

7. **Optional: iterate the reflection loop.** If the Reviewer made corrections, feed
   the corrected output back to a second Reviewer pass (or back to the Annotator) for
   one additional round. Diminishing returns typically set in after 1-2 review cycles.

8. **Extract tagged spans from the final annotated text.** Parse XML tags to produce a
   list of `(span_text, start_index, end_index, label)` tuples for downstream use.

9. **Evaluate against gold standard (if provided).** Convert both the system output and
   the gold annotations into token-level binary vectors. Compute Precision (TP / (TP+FP)),
   Recall (TP / (TP+FN)), and F1 (2*P*R / (P+R)). Report per-sentence and aggregate
   scores.

10. **Present results.** Output the final annotated text, the list of extracted spans,
    evaluation metrics (if applicable), and the Reviewer's critique log for
    transparency.

## Concrete Examples

**Example 1: Metaphor Identification in Literary Text**

User: "Annotate this paragraph for metaphors: 'The city was a jungle of concrete and steel. Her words cut through the silence like a knife, and he felt the weight of the world pressing down on his shoulders.'"

Approach:
1. Define tag schema: `<Metaphor>...</Metaphor>`
2. Run Annotator (zero-shot):

```json
{
  "annotated_text": "The city was <Metaphor>a jungle of concrete and steel</Metaphor>. Her words <Metaphor>cut through the silence</Metaphor> like a knife, and he felt <Metaphor>the weight of the world pressing down on his shoulders</Metaphor>.",
  "reasoning": {
    "a jungle of concrete and steel": "Maps WILDERNESS domain onto CITY; concrete/steel are not literal jungle elements.",
    "cut through the silence": "Maps PHYSICAL_CUTTING onto SPEECH; words cannot literally cut.",
    "the weight of the world pressing down": "Maps PHYSICAL_BURDEN onto EMOTIONAL_STATE; responsibility is not literal weight."
  }
}
```

3. Run Reviewer — Reviewer checks for missed metaphor "like a knife" (simile, related but distinct), confirms the three annotations are valid per standard metaphor theory, and notes no false positives.

```json
{
  "critique": "All three annotations are valid metaphors. 'like a knife' is a simile (explicit comparison marker 'like'), which is conventionally excluded from metaphor annotation unless the codebook specifies otherwise. No false negatives detected for pure metaphor.",
  "corrected_text": "(no changes needed)"
}
```

Output — Final spans:
| Span | Start | End | Label |
|------|-------|-----|-------|
| a jungle of concrete and steel | 17 | 47 | Metaphor |
| cut through the silence | 59 | 82 | Metaphor |
| the weight of the world pressing down on his shoulders | 112 | 167 | Metaphor |

---

**Example 2: Named Entity Annotation with Codebook**

User: "I have these annotation guidelines: tag PERSON, ORG, and LOC entities. Annotate: 'Marie Curie worked at the University of Paris before moving to Warsaw. The Nobel Committee awarded her twice.'"

Approach:
1. Tag schema: `<Entity type="PER">`, `<Entity type="ORG">`, `<Entity type="LOC">`
2. Embed codebook in system prompt (RAG paradigm).
3. Annotator output:

```
<Entity type="PER">Marie Curie</Entity> worked at <Entity type="ORG">the University of Paris</Entity> before moving to <Entity type="LOC">Warsaw</Entity>. <Entity type="ORG">The Nobel Committee</Entity> awarded her twice.
```

4. Reviewer catches boundary issue: "the" in "the University of Paris" — most NER conventions exclude articles from entity spans.

```json
{
  "critique": "Boundary error: 'the University of Paris' should be 'University of Paris' — articles are excluded per standard NER guidelines. All other spans are correct.",
  "corrected_text": "<Entity type=\"PER\">Marie Curie</Entity> worked at the <Entity type=\"ORG\">University of Paris</Entity> before moving to <Entity type=\"LOC\">Warsaw</Entity>. The <Entity type=\"ORG\">Nobel Committee</Entity> awarded her twice."
}
```

---

**Example 3: Batch Annotation with Evaluation**

User: "Annotate these 3 sentences for sentiment-bearing words and evaluate against my gold standard."

Approach:
1. Process each sentence through Annotator with `<Sentiment polarity="pos|neg">` tags.
2. Run Reviewer on each to catch over/under-tagging.
3. Convert final tags and gold standard to binary token vectors.
4. Compute metrics:

```
Sentence 1: P=1.00  R=0.80  F1=0.89
Sentence 2: P=0.75  R=1.00  F1=0.86
Sentence 3: P=0.83  R=0.83  F1=0.83
---
Aggregate:  P=0.86  R=0.88  F1=0.86
```

## Best Practices

- **Do:** Always include the Reviewer pass — the paper shows it consistently outperforms
  Annotator-only across all paradigms (zero-shot, few-shot, RAG).
- **Do:** Use structured JSON output with separate `reasoning` and `annotated_text`
  fields. This makes the Reviewer's job tractable and enables audit trails.
- **Do:** When a codebook exists, embed it fully in the system prompt (RAG paradigm)
  rather than summarizing it. Full-context outperforms fragmented retrieval for
  annotation guidelines.
- **Do:** Evaluate at the token level with binary vectors, not span-level exact match.
  Token-level metrics are more granular and forgiving of minor boundary differences.
- **Avoid:** Running more than 2 Reviewer iterations. Diminishing returns set in quickly
  and you risk the Reviewer introducing new errors through over-correction.
- **Avoid:** Using the same reasoning framing for both Annotator and Reviewer. The
  Annotator should reason about *why spans match the definition*; the Reviewer should
  reason about *whether the Annotator's justifications hold up against the codebook*.

## Error Handling

- **Malformed XML tags in Annotator output:** If the Annotator produces unclosed or
  nested tags incorrectly, re-prompt with explicit instruction to ensure well-formed XML.
  Parse with a lenient XML parser that recovers partial tags.
- **Reviewer disagrees on every annotation:** This signals a prompt misalignment between
  Annotator and Reviewer instructions. Verify both agents received identical codebook
  definitions and tag schemas.
- **Empty annotations (no spans tagged):** The Annotator may be too conservative. Switch
  from zero-shot to few-shot with 2-3 positive examples to calibrate detection threshold.
- **Gold standard format mismatch:** Ensure gold annotations use the same tokenization as
  the system output. Normalize whitespace and punctuation handling before computing
  binary vectors.
- **Truncated JSON responses:** For long texts, chunk input into segments of 200-500
  tokens to avoid response truncation. Reassemble annotations after all chunks complete.

## Limitations

- The dual-agent workflow doubles the token cost per annotation compared to single-pass
  approaches. For budget-constrained bulk annotation, consider running the Reviewer only
  on a sample to estimate quality.
- This approach works best for **span-level and token-level** annotation tasks. For
  document-level classification (e.g., "is this document positive or negative?"), the
  Reviewer adds less value since there is no span boundary to critique.
- The Reviewer cannot catch errors it doesn't know to look for. If the codebook is
  ambiguous or incomplete, both agents will reproduce the same systematic biases.
- Performance depends on the base model's linguistic knowledge. Highly specialized
  annotation tasks (e.g., phonological analysis, syntactic tree labeling) may still
  require fine-tuned models rather than prompt-based approaches.

## Reference

**Paper:** [LinguistAgent: A Reflective Multi-Model Platform for Automated Linguistic
Annotation](https://arxiv.org/abs/2602.05493v1) — Li, 2026. Key insight: a dual-agent
Annotator-Reviewer loop with structured reasoning fields consistently improves annotation
quality over single-pass LLM labeling across zero-shot, few-shot, and RAG paradigms.