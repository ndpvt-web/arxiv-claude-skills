---
name: "standardizing-longitudinal-radiology-report"
description: "Build LLM-based pipelines that automatically detect and classify longitudinal (temporal) changes in radiology reports. Use when the user mentions 'radiology report annotation', 'longitudinal report evaluation', 'temporal change detection in medical text', 'disease progression extraction', 'radiology NLP pipeline', or 'benchmark radiology report generation'."
---

# Standardizing Longitudinal Radiology Report Evaluation

This skill enables Claude to build automated annotation pipelines that detect longitudinal (temporal) information in radiology reports and classify disease progression across sequential examinations. The core technique is a two-stage LLM pipeline — first identifying sentences that compare current findings to prior studies, then extracting structured disease progression labels (improved / no change / worsened / unmentioned) — replacing brittle rule-based and manual-lexicon approaches with prompt-driven LLM classification that achieves 11.3% and 5.3% higher F1-scores on detection and tracking tasks respectively.

## When to Use

- When the user asks to build a pipeline that processes radiology reports and extracts temporal changes or comparisons to prior exams
- When the user needs to annotate a large corpus of medical reports (e.g., MIMIC-CXR) with structured longitudinal labels for benchmarking
- When the user wants to evaluate radiology report generation models on their ability to capture disease progression
- When the user asks to classify medical text sentences as longitudinal vs. cross-sectional
- When the user needs to extract per-disease progression status (improved, stable, worsened) from free-text clinical narratives
- When the user is building QA or NLP tools over sequential medical imaging reports

## Key Technique

The pipeline operates in two sequential stages. **Stage 1 (Longitudinal Sentence Detection)** takes each sentence from a radiology report and classifies it as either longitudinal (containing a comparison to a prior study) or cross-sectional (describing only current findings). This is a binary classification task where the LLM returns a structured label (`1` for longitudinal, `0` for cross-sectional). Sentences like "Pleural effusion has decreased compared to the prior exam" are longitudinal; "No acute cardiopulmonary abnormality" is cross-sectional.

**Stage 2 (Disease Progression Extraction)** processes only the sentences flagged as longitudinal and maps each to a curated vocabulary of ~50 radiological findings (atelectasis, pleural effusion, pneumonia, cardiomegaly, edema, pneumothorax, etc.), assigning each finding one of four progression labels: **improved** (finding resolved or decreased), **no change** (stable), **worsened** (increased severity or new development), or **unmentioned** (finding not discussed, used for hallucination detection in generated reports). This two-stage approach avoids running expensive disease-level extraction on irrelevant sentences, and the structured output enables direct F1-score comparison between ground-truth and generated reports.

The critical insight is that medium-scale LLMs (Qwen2.5-32B at ~32B parameters) outperform both larger models and rule-based systems for this task. Larger models (70B+) showed higher recall on sentence detection but lower precision on progression classification due to over-generation. The 32B model hit the optimal cost-accuracy-speed tradeoff at ~2 seconds per query, making corpus-scale annotation (95K+ reports) practical.

## Step-by-Step Workflow

1. **Define the disease vocabulary.** Create a curated list of radiological findings relevant to the domain — typically 30-50 terms for chest X-ray reports (e.g., `atelectasis`, `pleural_effusion`, `pneumonia`, `cardiomegaly`, `pulmonary_edema`, `pneumothorax`, `consolidation`, `lung_opacity`, `support_devices`). Store this as a JSON or YAML config so it can be updated per domain.

2. **Preprocess reports into sentences.** Split each radiology report's FINDINGS and IMPRESSION sections into individual sentences using a sentence tokenizer (e.g., `nltk.sent_tokenize` or `spacy`). Strip section headers and formatting artifacts. Preserve the report-level ID and sentence index for reassembly.

3. **Build the Stage 1 prompt (Longitudinal Sentence Detection).** Construct a few-shot prompt that defines the task, provides 3-5 positive and negative examples, and instructs the LLM to return a structured binary label. The prompt should emphasize that longitudinal sentences explicitly or implicitly compare current findings to prior studies.

4. **Run Stage 1 classification over all sentences.** Send each sentence (or batches of sentences) through the LLM with the Stage 1 prompt. Parse the structured output to partition sentences into longitudinal and cross-sectional sets.

5. **Build the Stage 2 prompt (Disease Progression Extraction).** For longitudinal sentences only, construct a prompt that provides the disease vocabulary and asks the LLM to return, for each mentioned finding, the progression label (`improved`, `no_change`, `worsened`). Use a structured output format (JSON preferred) to enforce schema compliance.

6. **Run Stage 2 extraction.** Process each longitudinal sentence through the LLM with the Stage 2 prompt. Parse the JSON output into a per-report, per-disease progression matrix.

7. **Aggregate per-report annotations.** Merge sentence-level extractions into a report-level structure: for each report, produce a dictionary mapping each disease to its progression status, with `unmentioned` as the default for diseases not referenced.

8. **Validate with gold-standard data.** If ground-truth annotations exist (or a manual sample is created), compute per-class precision, recall, and F1-score for both Stage 1 (binary) and Stage 2 (multi-class per disease). Use micro-averaged F1 as the primary metric.

9. **Benchmark report generation models.** To evaluate a generation model, run the same two-stage pipeline on both ground-truth and generated reports, then compare the per-disease progression matrices using F1-score. This directly measures whether the model captures longitudinal clinical information, unlike surface-level metrics (BLEU, ROUGE).

10. **Persist annotations in a structured format.** Save the annotated dataset as JSONL with fields: `report_id`, `sentence_index`, `sentence_text`, `is_longitudinal`, `disease_progressions` (dict). This enables downstream filtering, benchmarking, and model training.

## Concrete Examples

**Example 1: Annotating a single radiology report**

User: "I have a chest X-ray report and I want to extract which diseases improved, worsened, or stayed the same compared to the prior exam."

Approach:
1. Split the report into sentences
2. Classify each sentence as longitudinal or cross-sectional
3. Extract disease progression from longitudinal sentences

```python
# Input report (FINDINGS section)
report = """
Heart size is mildly enlarged, stable compared to prior exam.
There is improved aeration at the left lung base with decreased
atelectasis. Small bilateral pleural effusions persist unchanged.
No pneumothorax. Endotracheal tube is in satisfactory position.
"""

# After Stage 1 (sentence detection):
annotations = [
    {"sentence": "Heart size is mildly enlarged, stable compared to prior exam.",
     "is_longitudinal": True},
    {"sentence": "There is improved aeration at the left lung base with decreased atelectasis.",
     "is_longitudinal": True},
    {"sentence": "Small bilateral pleural effusions persist unchanged.",
     "is_longitudinal": True},
    {"sentence": "No pneumothorax.",
     "is_longitudinal": False},
    {"sentence": "Endotracheal tube is in satisfactory position.",
     "is_longitudinal": False},
]

# After Stage 2 (disease progression extraction):
progression = {
    "cardiomegaly": "no_change",
    "atelectasis": "improved",
    "pleural_effusion": "no_change",
    "pneumothorax": "unmentioned",  # negated, not a longitudinal comparison
}
```

**Example 2: Building the full annotation pipeline script**

User: "I need to annotate 10,000 MIMIC-CXR reports with longitudinal labels using a local LLM. Build me the pipeline."

Approach:
1. Set up batch processing with rate limiting
2. Implement both prompt stages
3. Output structured JSONL

```python
import json
from pathlib import Path

DISEASE_VOCAB = [
    "atelectasis", "cardiomegaly", "consolidation", "edema",
    "enlarged_cardiomediastinum", "fracture", "lung_lesion",
    "lung_opacity", "pleural_effusion", "pleural_other",
    "pneumonia", "pneumothorax", "support_devices"
]

STAGE1_PROMPT = """You are a radiology NLP specialist. Determine whether
the following sentence from a chest X-ray report contains longitudinal
information — i.e., it compares the current finding to a prior study.

Return ONLY a JSON object: {"is_longitudinal": true} or {"is_longitudinal": false}

Examples:
- "Cardiac silhouette is stable." -> {"is_longitudinal": true}
- "No acute cardiopulmonary process." -> {"is_longitudinal": false}
- "Pleural effusion has increased since prior exam." -> {"is_longitudinal": true}
- "Lungs are clear." -> {"is_longitudinal": false}

Sentence: {sentence}"""

STAGE2_PROMPT = """You are a radiology NLP specialist. Given a sentence that
contains longitudinal information from a chest X-ray report, extract which
diseases are mentioned and their progression status.

Disease vocabulary: {vocab}
Progression labels: improved, no_change, worsened

Return ONLY a JSON object mapping disease names to progression labels.
Only include diseases explicitly mentioned in the sentence.

Sentence: {sentence}"""


def annotate_report(report_text: str, llm_client) -> dict:
    sentences = split_into_sentences(report_text)
    results = []
    for sent in sentences:
        # Stage 1
        s1_resp = llm_client.query(STAGE1_PROMPT.format(sentence=sent))
        s1 = json.loads(s1_resp)
        entry = {"sentence": sent, "is_longitudinal": s1["is_longitudinal"]}
        # Stage 2 (only if longitudinal)
        if s1["is_longitudinal"]:
            s2_resp = llm_client.query(STAGE2_PROMPT.format(
                sentence=sent, vocab=", ".join(DISEASE_VOCAB)))
            entry["progressions"] = json.loads(s2_resp)
        results.append(entry)

    # Aggregate report-level progression
    report_progression = {d: "unmentioned" for d in DISEASE_VOCAB}
    for entry in results:
        for disease, status in entry.get("progressions", {}).items():
            report_progression[disease] = status
    return {"sentences": results, "report_progression": report_progression}
```

**Example 3: Benchmarking a report generation model**

User: "I have ground-truth reports and model-generated reports. How do I evaluate if the model captures longitudinal information correctly?"

Approach:
1. Run the annotation pipeline on both ground-truth and generated reports
2. Compare per-disease progression labels
3. Compute class-level and micro-averaged F1

```python
from sklearn.metrics import classification_report

def evaluate_longitudinal(gt_reports, gen_reports, annotator):
    all_gt_labels, all_gen_labels = [], []
    for gt, gen in zip(gt_reports, gen_reports):
        gt_ann = annotator(gt)["report_progression"]
        gen_ann = annotator(gen)["report_progression"]
        for disease in DISEASE_VOCAB:
            all_gt_labels.append(gt_ann[disease])
            all_gen_labels.append(gen_ann[disease])

    print(classification_report(
        all_gt_labels, all_gen_labels,
        labels=["improved", "no_change", "worsened", "unmentioned"],
        digits=3
    ))

# Output:
#                precision  recall  f1-score  support
#     improved      0.412   0.389    0.400      312
#    no_change      0.634   0.701    0.666     1847
#     worsened      0.298   0.256    0.275      198
#  unmentioned      0.951   0.943    0.947     8643
#    micro avg      0.891   0.891    0.891    11000
```

## Best Practices

- **Do:** Use few-shot prompting with 3-5 diverse examples per stage — include edge cases like implicit comparisons ("persistent effusion") and negated findings ("no new infiltrate")
- **Do:** Enforce structured JSON output via system prompts or constrained decoding to prevent free-text drift that breaks downstream parsing
- **Do:** Use a curated, domain-specific disease vocabulary rather than letting the LLM invent terms — this ensures consistent label spaces across ground-truth and generated reports
- **Do:** Process Stage 1 before Stage 2 to avoid wasting inference on cross-sectional sentences, which typically constitute 60-70% of report content
- **Avoid:** Using models larger than necessary — 32B-parameter models match or exceed 70B models on this task while running 3-4x faster; test your target model before committing to corpus-scale runs
- **Avoid:** Treating `unmentioned` and `improved`/`no_change`/`worsened` as balanced classes — the class distribution is heavily skewed toward `unmentioned`, so always report per-class metrics, not just accuracy

## Error Handling

| Issue | Cause | Fix |
|-------|-------|-----|
| LLM returns free text instead of JSON | Prompt not constraining output format | Add "Return ONLY valid JSON" instruction; use JSON mode if the API supports it; add a regex-based fallback parser |
| Disease name not in vocabulary | LLM uses a synonym (e.g., "heart enlargement" vs. "cardiomegaly") | Post-process with a synonym mapping dictionary; normalize all disease names to canonical forms |
| Sentence splitter breaks mid-finding | Medical abbreviations confuse tokenizer (e.g., "Dr.", "approx.") | Use a medical-domain sentence splitter or add abbreviation exceptions to NLTK's Punkt tokenizer |
| Stage 2 assigns contradictory labels | Same disease appears in multiple sentences with different statuses | Implement a priority resolution rule: `worsened` > `no_change` > `improved` > `unmentioned`, or flag for manual review |
| Batch processing fails midway | API timeout or rate limit on large corpus | Implement checkpoint-resume: save progress per report_id in JSONL; skip already-annotated reports on restart |

## Limitations

- **Domain specificity.** The disease vocabulary and prompt examples are tuned for chest X-ray reports. Adapting to other modalities (MRI, CT, ultrasound) or body regions requires rebuilding the vocabulary and re-validating with a new gold-standard sample.
- **Implicit temporal references.** Sentences like "chronic cardiomegaly" imply stability without explicit comparison — LLMs may miss these or misclassify them. Accuracy on implicit longitudinal sentences is consistently lower than on explicit ones.
- **No severity grading.** The pipeline classifies direction of change (better/same/worse) but does not quantify magnitude. "Slightly improved" and "markedly improved" both map to `improved`.
- **Single-report scope.** The pipeline annotates individual reports; it does not align findings across a patient's full longitudinal series. Cross-report entity linking is a separate unsolved problem.
- **LLM hallucination in Stage 2.** The model may extract diseases not actually mentioned in the sentence. Always validate Stage 2 output against the original sentence text, especially for low-frequency findings.
- **Language and dataset bias.** Validated only on English-language MIMIC-CXR reports from a single US institution. Performance on reports from other languages, institutions, or documentation styles is unknown.

## Reference

Wang, X., Figueredo, G., Li, R., & Chen, X. (2026). *Standardizing Longitudinal Radiology Report Evaluation via Large Language Model Annotation.* arXiv:2601.16753v1. https://arxiv.org/abs/2601.16753v1

Key takeaway: A two-stage LLM pipeline (sentence detection then disease progression extraction) using a ~32B parameter model outperforms both larger LLMs and rule-based lexicon methods for annotating temporal changes in radiology reports, enabling standardized benchmarking of report generation models.