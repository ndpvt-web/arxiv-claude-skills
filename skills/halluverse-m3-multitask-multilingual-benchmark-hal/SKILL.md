---
name: "halluverse-m3-multitask-multilingual-benchmark-hal"
description: "Detect and classify hallucinations in LLM outputs across languages using the HalluVerse-M3 fine-grained taxonomy (entity-level, relation-level, sentence-level). Trigger phrases: 'check for hallucinations', 'detect hallucinations in this output', 'verify factual consistency', 'multilingual hallucination audit', 'classify hallucination type', 'hallucination detection benchmark'."
---

# HalluVerse-M3: Fine-Grained Multilingual Hallucination Detection

This skill enables Claude to systematically detect and classify hallucinations in LLM-generated text using the three-tier taxonomy from the HalluVerse-M3 benchmark. Rather than binary "hallucinated or not" judgments, Claude decomposes generated text into propositions and classifies each deviation as entity-level (wrong entities), relation-level (wrong predicates/attributes), or sentence-level (entirely fabricated claims). The method works across English, Arabic, Hindi, and Turkish, and applies to both question-answering outputs and dialogue/document summaries.

## When to Use

- When a user asks to verify whether an LLM-generated answer contains factual errors against a reference source
- When reviewing multilingual outputs (especially Arabic, Hindi, or Turkish) for factual consistency
- When auditing summaries of conversations or documents for fabricated or distorted claims
- When building or evaluating a hallucination detection pipeline and needing a structured classification scheme
- When the user wants to understand *what kind* of hallucination occurred, not just whether one exists
- When comparing hallucination rates across languages for the same source content

## Key Technique

HalluVerse-M3 decomposes text into **propositions** of the form `p = <relation; entity_1, ..., entity_k; attributes>`. A hallucination is detected by aligning propositions in the generated output against propositions in the reference text, then classifying the mismatch:

- **Entity-level hallucination**: The predicate and attributes match a reference proposition, but at least one entity is substituted. Example: "Berlin is the capital of France" when the reference says "Paris is the capital of France" -- the relation (capital-of) and attribute (France) are correct, but the entity (Berlin vs Paris) is wrong.
- **Relation-level hallucination**: The entities are correct, but the predicate or attributes are altered. Example: "Einstein invented the telephone" when the reference says "Einstein developed the theory of relativity" -- the entity (Einstein) is correct, but the relation (invented telephone vs developed theory) is wrong.
- **Sentence-level hallucination**: The entire proposition has no corresponding reference -- it is fabricated whole-cloth. Example: A summary claims "The participants also discussed budget allocations" when the source dialogue never mentions budgets at all.

The key insight from the benchmark is that these three types have sharply different detection difficulty. Entity hallucinations are easiest to catch (~96% accuracy for top models in English QA), relation hallucinations are moderate (~87%), and sentence-level hallucinations are hardest (~74%). Performance degrades further in lower-resource languages (Hindi accuracy drops 7-10 points below English) and on summarization tasks vs QA. This means detection strategies must be calibrated: entity checks can rely on named-entity matching, but sentence-level checks require holistic semantic entailment reasoning.

## Step-by-Step Workflow

1. **Collect the reference and generated texts.** Obtain the ground-truth source (reference answer, source document, or dialogue transcript) and the LLM-generated output to audit. If the text is multilingual, identify the language to calibrate expectations (English > Arabic/Turkish > Hindi in detection reliability).

2. **Decompose both texts into propositions.** Break each text into atomic factual claims in the form `<relation; entities; attributes>`. For QA, each sentence typically yields 1-3 propositions. For summaries, decompose each summary sentence independently.

3. **Align generated propositions to reference propositions.** For each proposition in the generated text, find the closest matching proposition in the reference. Use semantic similarity rather than exact string matching -- entities may be paraphrased or transliterated across languages.

4. **Classify unaligned or misaligned propositions.** For each generated proposition:
   - If it aligns to a reference proposition but entities differ -> **entity-level hallucination**
   - If it aligns to a reference proposition but the relation/attributes differ -> **relation-level hallucination**
   - If no reference proposition aligns at all -> **sentence-level hallucination**
   - If it fully aligns -> **faithful** (not hallucinated)

5. **Apply language-specific scrutiny.** For Arabic, watch for entity transliteration errors that may look like hallucinations but are translation artifacts. For Hindi, apply extra scrutiny since detection accuracy is lowest -- flag uncertain cases explicitly. For Turkish, attend to agglutinative morphology that can change entity boundaries.

6. **Apply task-specific scrutiny.** For QA outputs, focus on entity-level checks since answers are typically short and entity-dense. For summarization, prioritize sentence-level checks since summaries are more prone to inserting unsupported claims.

7. **Rate confidence per finding.** Assign a confidence level (high/medium/low) to each detected hallucination. Entity-level detections are typically high-confidence; sentence-level detections in non-English languages are typically lower-confidence.

8. **Produce a structured report.** Output a per-proposition table with: the generated text span, the hallucination type (entity/relation/sentence/faithful), the conflicting reference span (if any), and the confidence level.

9. **Suggest corrections.** For entity and relation hallucinations, propose the correct proposition from the reference. For sentence-level hallucinations, recommend deletion of the unsupported claim.

## Concrete Examples

**Example 1: QA hallucination detection (English)**

User: "Check this answer for hallucinations. Reference: 'The Eiffel Tower was completed in 1889 and stands 330 meters tall in Paris, France.' Generated answer: 'The Eiffel Tower was completed in 1887 and stands 330 meters tall in Lyon, France.'"

Approach:
1. Decompose reference: P1=<completed; Eiffel Tower; 1889>, P2=<stands; Eiffel Tower; 330m>, P3=<located-in; Eiffel Tower; Paris, France>
2. Decompose generated: G1=<completed; Eiffel Tower; 1887>, G2=<stands; Eiffel Tower; 330m>, G3=<located-in; Eiffel Tower; Lyon, France>
3. Align and classify:
   - G1 vs P1: relation matches, entity matches, but attribute differs (1887 vs 1889) -> **relation-level hallucination**
   - G2 vs P2: full match -> **faithful**
   - G3 vs P3: relation matches, attributes match, but entity differs (Lyon vs Paris) -> **entity-level hallucination**

Output:
```
| Generated Span                    | Type       | Reference Conflict         | Confidence |
|-----------------------------------|------------|----------------------------|------------|
| "completed in 1887"               | relation   | Reference says 1889        | high       |
| "stands 330 meters tall"          | faithful   | --                         | --         |
| "in Lyon, France"                 | entity     | Reference says Paris       | high       |
```

**Example 2: Dialogue summarization audit (Arabic/English)**

User: "Verify this summary against the source dialogue. Source dialogue covers a customer calling about a delayed shipment, agent confirms 3-day delay, offers $10 credit. Summary: 'The customer called about a delayed shipment. The agent confirmed a 5-day delay and offered a $20 refund. The customer also inquired about upgrading their subscription plan.'"

Approach:
1. Decompose summary into propositions: S1=<called-about; customer; delayed shipment>, S2=<confirmed; agent; 5-day delay>, S3=<offered; agent; $20 refund>, S4=<inquired-about; customer; subscription upgrade>
2. Align against source:
   - S1 aligns to source (customer called about delay) -> **faithful**
   - S2 aligns but attribute differs (5-day vs 3-day) -> **relation-level hallucination**
   - S3 aligns but entities differ ($20 refund vs $10 credit) -> **entity-level hallucination**
   - S4 has no corresponding source proposition -> **sentence-level hallucination**

Output:
```
| Summary Claim                          | Type       | Issue                              | Confidence |
|----------------------------------------|------------|------------------------------------|------------|
| "customer called about delayed shipment"| faithful  | --                                 | --         |
| "confirmed a 5-day delay"              | relation   | Source says 3-day delay            | high       |
| "offered a $20 refund"                 | entity     | Source says $10 credit             | high       |
| "inquired about upgrading subscription"| sentence   | Not mentioned in source dialogue   | high       |
```

**Example 3: Multilingual comparison (Hindi)**

User: "I have an LLM generating answers in Hindi. How do I evaluate hallucination rates compared to English?"

Approach:
1. Prepare parallel test instances -- same questions with reference answers in both English and Hindi
2. Run the LLM on both language versions
3. Apply the proposition-alignment workflow to both outputs
4. Expect Hindi detection accuracy to be 7-10% lower than English based on HalluVerse-M3 findings
5. For Hindi specifically: flag cases where entity boundaries are ambiguous due to script or morphology, and use back-translation to English as a secondary verification step
6. Report per-language, per-hallucination-type accuracy breakdown

Output:
```
| Language | Entity Acc | Relation Acc | Sentence Acc | Overall |
|----------|-----------|-------------|-------------|---------|
| English  | 96%       | 87%         | 74%         | 86%     |
| Hindi    | 88%       | 79%         | 65%         | 77%     |
```
Note: Hindi sentence-level detection is the weakest link. Consider adding a back-translation verification step for Hindi outputs to catch fabricated claims.

## Best Practices

- **Do:** Always decompose text into atomic propositions before judging. Checking whole paragraphs for hallucinations misses fine-grained errors.
- **Do:** Treat the three hallucination types as a hierarchy of difficulty. Entity checks are fast and reliable; sentence-level checks require careful semantic reasoning.
- **Do:** Calibrate confidence by language. English detections are more reliable than Hindi or Turkish detections. Flag low-resource language findings as needing human review.
- **Do:** For summarization tasks, pay special attention to sentence-level hallucinations -- summaries frequently insert plausible but unsupported claims.
- **Avoid:** Binary "hallucinated / not hallucinated" judgments. The type of hallucination matters for downstream correction.
- **Avoid:** Relying on string-matching for entity comparison across languages. Transliterated names and morphological variants require semantic comparison.
- **Avoid:** Assuming QA-tuned hallucination detectors transfer to summarization. HalluVerse-M3 shows a consistent 8-10% accuracy gap between the two tasks.

## Error Handling

- **Ambiguous proposition alignment**: When a generated proposition partially matches multiple reference propositions, align to the closest semantic match and flag the ambiguity in the report. Do not force a single alignment if the match is genuinely unclear.
- **Translation artifacts vs. real hallucinations**: In multilingual settings, transliteration differences (e.g., "Mumbai" vs "Bombay") are not hallucinations. Cross-reference entity identity before classifying as entity-level.
- **Compounded hallucinations**: A single generated sentence may contain both entity and relation errors. Decompose into separate propositions and classify each independently rather than assigning one label to the whole sentence.
- **Missing reference text**: If no ground-truth reference is available, the proposition-alignment method cannot be applied. Fall back to self-consistency checks or knowledge-grounded verification instead, and note that confidence will be lower.

## Limitations

- This method requires a reference text to align against. It cannot detect hallucinations in open-ended generation without ground truth.
- The three-type taxonomy (entity, relation, sentence) does not cover all hallucination modes -- e.g., temporal reasoning errors or logical contradictions within the generated text itself are not explicitly modeled.
- Detection accuracy is inherently lower for Hindi and Turkish. Results in these languages should be treated as indicative, not definitive.
- Summarization hallucination detection remains materially harder than QA detection even with this framework. Sentence-level hallucinations in long summaries can still evade structured detection.
- The proposition decomposition step is itself imperfect -- complex sentences with nested clauses may not decompose cleanly, introducing classification noise.

## Reference

**Paper**: [HalluVerse-M3: A Multitask Multilingual Benchmark for Hallucination in LLMs](https://arxiv.org/abs/2602.06920v1) (Abdaljalil et al., 2026). Look for: the formal proposition-based hallucination taxonomy (Section 3), the controlled hallucination injection methodology, and the per-language per-task accuracy breakdowns in the evaluation tables.

**Dataset**: [sabdalja/HalluVerse-M3 on HuggingFace](https://huggingface.co/datasets/sabdalja/HalluVerse-M3) -- 4,038 instances across 4 languages and 2 tasks with fine-grained hallucination type labels.