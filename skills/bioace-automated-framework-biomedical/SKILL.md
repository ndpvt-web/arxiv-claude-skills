---
name: "bioace-automated-framework-biomedical"
description: |
  Evaluate biomedical QA outputs using the BioACE nugget-based framework — assess answer completeness, correctness, precision, recall, and citation quality against ground-truth nuggets.
  Trigger phrases:
  - "evaluate biomedical answers"
  - "check citation quality for medical QA"
  - "nugget-based evaluation of RAG output"
  - "assess completeness and correctness of biomedical text"
  - "verify biomedical citations with NLI"
  - "score biomedical question answering output"
---

# BioACE: Automated Biomedical Answer and Citation Evaluation

This skill enables Claude to apply the BioACE evaluation framework to assess the quality of biomedical question-answering (QA) outputs. BioACE decomposes evaluation into two axes — **answer quality** (measured via nugget-based completeness, correctness, precision, and recall) and **citation quality** (measured via NLI-based and LLM-based entailment checking). When a user needs to evaluate whether a biomedical answer is factually complete, correctly grounded, and properly cited against source literature, this skill provides the structured methodology to do so.

## When to Use

- When the user asks to evaluate the output of a biomedical RAG pipeline against reference answers
- When building an automated evaluation harness for medical QA systems and needing nugget-based scoring
- When verifying whether citations in a generated biomedical answer actually support the claims they are attached to
- When comparing multiple biomedical QA models and needing structured completeness/correctness/precision/recall metrics
- When decomposing a gold-standard biomedical answer into atomic nuggets for systematic evaluation
- When implementing NLI-based citation verification for scientific literature references

## Key Technique

**Nugget-based answer evaluation.** BioACE decomposes reference (gold-standard) answers into discrete atomic facts called "nuggets" — each nugget is a single verifiable claim or piece of information (e.g., "Metformin is a first-line treatment for type 2 diabetes"). The system then checks each nugget against the generated answer to compute four metrics: **completeness** (proportion of reference nuggets covered), **correctness** (factual accuracy of matched content), **precision** (ratio of correct claims to total claims in the generated answer), and **recall** (proportion of required nuggets successfully captured). This nugget decomposition is what makes the evaluation granular rather than relying on coarse text-similarity scores like ROUGE or BERTScore.

**Citation entailment verification.** For each citation attached to a claim in the generated answer, BioACE checks whether the cited passage actually entails the claim. This is done using Natural Language Inference (NLI) models — specifically biomedical-domain NLI models like BioNLI and SciNLI, which outperform general-domain alternatives (e.g., RoBERTa-MNLI) on this task. The framework also supports LLM-based entailment as an alternative. A citation is scored as "supporting" only if the NLI model classifies the relationship between the cited passage and the claim as entailment (not contradiction or neutral).

**Why this matters.** Standard text-similarity metrics fail in the biomedical domain because a semantically similar but factually incorrect answer can score highly. BioACE's nugget decomposition catches partial answers and hallucinated content that surface-level metrics miss. The citation verification layer adds a second check: even if an answer is correct, unsupported or fabricated citations undermine trust.

## Step-by-Step Workflow

1. **Decompose the reference answer into nuggets.** Break the gold-standard answer into atomic, independently verifiable facts. Each nugget should be a single claim — e.g., "Drug X inhibits enzyme Y" rather than a compound sentence with multiple facts. Store nuggets as a JSON array.

2. **Decompose the generated answer into claim segments.** Parse the system-generated answer into individual claim segments, preserving any citation markers (e.g., `[1]`, `[PMID:12345]`) attached to each segment.

3. **Match generated claims to reference nuggets.** For each reference nugget, determine whether the generated answer contains a semantically equivalent claim. Use sentence embeddings (e.g., `all-MiniLM-L6-v2`, `all-mpnet-base-v2`, or `sup-simcse-roberta-large`) to compute similarity, with a threshold for matching (typically cosine similarity >= 0.75).

4. **Score answer completeness.** Calculate `completeness = number_of_matched_nuggets / total_reference_nuggets`. A score of 1.0 means every required fact is present in the generated answer.

5. **Score answer correctness.** For each matched claim, verify factual accuracy — check that the generated claim does not distort or contradict the nugget it matches. Flag claims that are semantically close but factually inverted (e.g., "increases risk" vs. "decreases risk"). Score `correctness = number_of_correct_matches / number_of_matched_nuggets`.

6. **Compute precision and recall.** Calculate `precision = number_of_correct_matches / total_generated_claims` and `recall = number_of_correct_matches / total_reference_nuggets`. These give a balanced view of over-generation (low precision) vs. under-coverage (low recall).

7. **Extract citation-claim pairs.** For each citation in the generated answer, extract the (claim_text, cited_passage_text) pair. If the cited passage must be retrieved (e.g., from PubMed via PMID), fetch it first.

8. **Run NLI-based citation verification.** For each (claim, cited_passage) pair, classify the relationship using a biomedical NLI model. Prefer BioNLI or SciNLI over general-domain models. Label each citation as `entailment` (supports), `contradiction` (refutes), or `neutral` (irrelevant).

9. **Compute citation scores.** Calculate `citation_precision = entailment_citations / total_citations` and optionally `citation_recall = claims_with_at_least_one_supporting_citation / total_claims_requiring_citation`.

10. **Produce a structured evaluation report.** Aggregate all scores into a single report with per-question and overall metrics, including specific nuggets missed, incorrect claims, and unsupported citations.

## Concrete Examples

**Example 1: Evaluating a RAG system's answer about metformin**

User: "I have a biomedical QA system that answered the question 'What are the side effects of metformin?' — evaluate its output against the reference answer."

Approach:
1. Decompose the reference answer into nuggets:
```json
{
  "question": "What are the side effects of metformin?",
  "reference_nuggets": [
    "Gastrointestinal symptoms are the most common side effects of metformin",
    "Metformin can cause lactic acidosis in rare cases",
    "Metformin may lead to vitamin B12 deficiency with long-term use",
    "Diarrhea, nausea, and abdominal pain are frequent GI side effects",
    "Lactic acidosis risk increases with renal impairment"
  ]
}
```

2. Parse the generated answer into claims:
```json
{
  "generated_claims": [
    {"text": "Metformin commonly causes gastrointestinal issues such as diarrhea and nausea", "citations": ["PMID:19246357"]},
    {"text": "Lactic acidosis is a serious but rare complication", "citations": ["PMID:20536313"]},
    {"text": "Metformin can cause weight gain in some patients", "citations": []}
  ]
}
```

3. Compute scores:

```
Answer Evaluation:
  Matched nuggets: 3/5 (nuggets 1, 2, 4 matched; 3, 5 missed)
  Correct matches: 2/3 (claim about weight gain is incorrect — metformin typically causes weight loss)
  Completeness: 0.60
  Correctness:  0.67
  Precision:    0.67 (2 correct out of 3 generated claims)
  Recall:       0.40 (2 correct matches out of 5 reference nuggets)

Citation Evaluation:
  PMID:19246357 → claim "GI issues": entailment ✓
  PMID:20536313 → claim "lactic acidosis": entailment ✓
  Claim "weight gain": no citation provided ✗
  Citation precision: 2/2 = 1.00
  Citation coverage: 2/3 claims cited = 0.67
```

**Example 2: Building a citation verification module**

User: "Help me write a Python function that checks whether PubMed citations actually support the claims they're attached to in a biomedical QA output."

Approach:
1. Structure the function around NLI-based entailment classification.
2. Use a biomedical NLI model (BioNLI preferred).

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch

class CitationVerifier:
    def __init__(self, model_name="mosaicnlp/BioNLI"):
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = AutoModelForSequenceClassification.from_pretrained(model_name)
        self.labels = ["entailment", "neutral", "contradiction"]

    def verify(self, claim: str, cited_passage: str) -> dict:
        """Check if a cited passage entails the claim."""
        inputs = self.tokenizer(
            cited_passage, claim,
            return_tensors="pt", truncation=True, max_length=512
        )
        with torch.no_grad():
            logits = self.model(**inputs).logits
        probs = torch.softmax(logits, dim=-1)[0]
        pred_label = self.labels[probs.argmax().item()]
        return {
            "label": pred_label,
            "confidence": probs.max().item(),
            "supports_claim": pred_label == "entailment"
        }

    def evaluate_citations(self, claim_citation_pairs: list[dict]) -> dict:
        """Evaluate all citation-claim pairs and return aggregate scores."""
        results = []
        for pair in claim_citation_pairs:
            result = self.verify(pair["claim"], pair["cited_passage"])
            result["claim"] = pair["claim"]
            results.append(result)
        supporting = sum(1 for r in results if r["supports_claim"])
        return {
            "per_citation": results,
            "citation_precision": supporting / len(results) if results else 0,
            "total_citations": len(results),
            "supporting_citations": supporting
        }
```

**Example 3: Implementing nugget-based completeness scoring**

User: "Write code to score how complete a generated biomedical answer is against reference nuggets."

```python
from sentence_transformers import SentenceTransformer
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np

def score_completeness(
    reference_nuggets: list[str],
    generated_answer: str,
    similarity_threshold: float = 0.75,
    model_name: str = "all-mpnet-base-v2"
) -> dict:
    """Score answer completeness using nugget matching."""
    model = SentenceTransformer(model_name)

    # Split generated answer into sentences as candidate claims
    import re
    generated_claims = [s.strip() for s in re.split(r'[.!?]+', generated_answer) if s.strip()]

    nugget_embeddings = model.encode(reference_nuggets)
    claim_embeddings = model.encode(generated_claims)

    sim_matrix = cosine_similarity(nugget_embeddings, claim_embeddings)

    matched_nuggets = []
    unmatched_nuggets = []
    for i, nugget in enumerate(reference_nuggets):
        max_sim = sim_matrix[i].max()
        best_match_idx = sim_matrix[i].argmax()
        if max_sim >= similarity_threshold:
            matched_nuggets.append({
                "nugget": nugget,
                "matched_claim": generated_claims[best_match_idx],
                "similarity": float(max_sim)
            })
        else:
            unmatched_nuggets.append({"nugget": nugget, "best_similarity": float(max_sim)})

    completeness = len(matched_nuggets) / len(reference_nuggets) if reference_nuggets else 0
    return {
        "completeness": completeness,
        "matched": matched_nuggets,
        "missed": unmatched_nuggets,
        "total_nuggets": len(reference_nuggets),
        "matched_count": len(matched_nuggets)
    }
```

## Best Practices

- **Do:** Decompose reference answers into fine-grained nuggets — each nugget should contain exactly one verifiable fact. Compound nuggets dilute evaluation precision.
- **Do:** Use domain-specific NLI models (BioNLI, SciNLI) for citation verification rather than general-domain models. The paper shows these correlate significantly better with human judgments in biomedical contexts.
- **Do:** Separate answer evaluation from citation evaluation. A factually correct answer with fabricated citations is a different failure mode than an incomplete but well-cited answer.
- **Do:** Use sentence-transformer embeddings (`all-mpnet-base-v2` or `sup-simcse-roberta-large`) for nugget matching — they outperform token-overlap metrics like ROUGE for semantic matching.
- **Avoid:** Using ROUGE, BLEU, or BERTScore alone for biomedical QA evaluation. These surface-level metrics miss factual errors and cannot distinguish correct from hallucinated content.
- **Avoid:** Setting the nugget-matching similarity threshold too low (< 0.6). Low thresholds produce false matches where a vaguely related sentence counts as covering a nugget it doesn't actually address.

## Error Handling

- **Missing citations:** If a generated answer makes claims without citations, flag these as "uncited claims" rather than failing. Calculate citation coverage separately.
- **Unresolvable PMIDs:** If a cited PMID cannot be retrieved from PubMed, mark the citation as "unverifiable" rather than "unsupporting." Report these separately.
- **Ambiguous nugget matches:** When a generated claim matches multiple nuggets with similar scores, use the highest-scoring match and mark the others as potentially matched. Consider using a bipartite matching algorithm (Hungarian method) to find optimal one-to-one assignments.
- **Model loading failures:** If a BioNLI model is unavailable, fall back to `roberta-large-mnli` with a warning that biomedical accuracy may be degraded.
- **Long passages exceeding token limits:** Truncate cited passages to 512 tokens (the NLI model limit). For longer passages, split into paragraphs and check entailment against each, taking the maximum entailment score.

## Limitations

- **Nugget creation requires domain expertise.** The framework evaluates against reference nuggets, but creating high-quality nuggets for a new question set still requires biomedical knowledge. This skill helps automate the *scoring* but not the *nugget authoring* step.
- **NLI models have a 512-token input limit.** Very long cited passages or claims must be truncated or chunked, which can lose context.
- **Citation verification assumes passage availability.** If the cited paper is behind a paywall or not indexed in PubMed, the passage cannot be retrieved for NLI checking.
- **Semantic matching is not logical reasoning.** Sentence embeddings can match paraphrases but may miss logical entailments (e.g., "X inhibits Y" does not match "Y activity is reduced by X" at high thresholds). NLI models partially address this but are not perfect.
- **The framework does not evaluate fluency or coherence.** BioACE focuses on factual coverage and citation grounding, not readability or organization of the answer.

## Reference

- **Paper:** [BioACE: An Automated Framework for Biomedical Answer and Citation Evaluations](https://arxiv.org/abs/2602.04982v2) — Gupta, Bartels, Demner-Fushman (2026). Focus on Sections 3-5 for the nugget-based evaluation methodology and NLI citation verification results.
- **Code:** [github.com/deepaknlp/BioACE](https://github.com/deepaknlp/BioACE) — Reference implementation with `answer_eval.py` and `citation_eval.py`.
- **Dataset:** [osf.io/ydbzq](https://osf.io/ydbzq/) — Ground-truth nuggets and baseline outputs for benchmarking.