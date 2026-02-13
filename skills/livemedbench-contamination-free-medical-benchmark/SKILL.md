---
name: "livemedbench-contamination-free-medical-benchmark"
description: "Build contamination-free LLM evaluation pipelines with multi-agent data curation and automated rubric-based scoring. Uses LiveMedBench's three-agent curation framework and bipolar rubric evaluation to assess LLM outputs against granular, case-specific criteria. Trigger phrases: 'build a contamination-free benchmark', 'evaluate LLM with rubrics', 'curate clinical test data', 'automated rubric evaluation pipeline', 'detect data contamination in LLMs', 'multi-agent data curation framework'."
---

# Contamination-Free LLM Benchmarking with Multi-Agent Curation and Rubric Evaluation

This skill enables Claude to build evaluation pipelines for LLMs that resist data contamination and produce clinically meaningful scores. It implements two core systems from the LiveMedBench paper: (1) a **Multi-Agent Clinical Curation Framework** that transforms noisy real-world data into structured, validated test cases using a Screener-Validator-Controller agent chain, and (2) an **Automated Rubric-based Evaluation Framework** that decomposes expert responses into weighted, bipolar criteria and scores model outputs against them — achieving far stronger alignment with human experts than LLM-as-a-Judge approaches.

## When to Use

- When the user needs to build a benchmark pipeline that harvests fresh data on a recurring schedule and enforces temporal separation from model training cutoffs
- When evaluating LLM outputs on open-ended tasks where ROUGE/BLEU are inadequate and LLM-as-a-Judge introduces length bias or misses safety hazards
- When building a multi-agent data processing pipeline that filters, structures, and validates raw text (forum posts, case reports, support tickets) into evaluation-ready cases
- When the user wants to detect data contamination by comparing model performance on pre-cutoff vs. post-cutoff test subsets
- When creating rubric-based grading systems that decompose gold-standard answers into weighted binary criteria with both positive (reward) and negative (penalty) polarity
- When building automated QA pipelines for medical, legal, or other domain-specific content that requires evidence-based validation

## Key Technique

**Multi-Agent Curation (Screener → Validator → Controller):** Raw data (e.g., forum threads) passes through three specialized agents. The **Screener** standardizes each case into a SOAP-structured triplet: patient narrative entities (LN), primary query (Q), and actionable advice (LA). It forwards only complete cases where all three components are non-empty. The **Validator** scores each triplet on three dimensions — query clinical validity (binary, per Ely's taxonomy), narrative information sufficiency (fraction of required clinical items present), and evidence alignment (checking each advice element against retrieved medical evidence, scored 1.0/0.5/0.0). Cases must exceed thresholds on both sufficiency (>0.5) and alignment (>0.5). The **Controller** performs a final audit to verify every element in LN and LA is explicitly grounded in the original source text, preventing synthetic contamination.

**Automated Rubric Evaluation:** Rather than asking an LLM to holistically judge a response, this framework generates case-specific rubrics in three steps. First, **Theme-Guided Fact Extraction** filters the gold-standard answer to retain only facts relevant to assigned evaluation themes (e.g., Safety, Context Awareness). Second, **Bipolar Criterion Formulation** converts each fact into a binary criterion — positive criteria reward correct inclusions, negative criteria penalize hallucinations or contradictions. Third, **Axis Assignment & Weighting** labels each criterion with an evaluation axis (Accuracy, Completeness, Communication Quality, Context Awareness, Safety) and assigns a signed weight in [-10, 10]. The final score is computed as the sum of satisfied criterion weights divided by the maximum positive achievable score, clipped to [0, 1]. This achieves Macro F1 of 0.76 against human ratings, compared to 0.26 correlation for naive LLM-as-a-Judge.

**Contamination Detection:** By enforcing strict temporal separation — only including cases published after a known date — the pipeline can empirically quantify contamination. Models are evaluated on the full dataset vs. a post-cutoff subset. Performance degradation on post-cutoff cases signals contamination. In the paper, 84% of 38 models showed measurable drops, confirming that static benchmarks routinely produce inflated scores.

## Step-by-Step Workflow

### Part A: Data Curation Pipeline

1. **Define source connectors and temporal boundaries.** Write scrapers or API integrations for your data sources (forums, case repositories, ticket systems). Configure a cutoff date filter — only ingest records published after a chosen date to guarantee temporal separation from model training data. Apply basic pre-filters: language detection, minimum text length, non-duplicate check, domain keyword matching (e.g., ICD/ICF codebooks for medical data).

2. **Implement the Screener Agent.** Build an LLM-powered agent that takes raw text and extracts a structured triplet using the SOAP framework:
   - `LN` (Narrative Entities): patient demographics, symptoms, history, lab results
   - `Q` (Primary Query): the clinical question being asked
   - `LA` (Actionable Advice): the recommended actions from verified respondents

   Apply the forward condition: only pass cases where all three components are non-empty.

3. **Implement the Validator Agent.** Score each structured triplet on three dimensions:
   - `S_cc` (Query Validity): Binary — does Q represent a clinically meaningful chief complaint?
   - `S_inf` (Narrative Sufficiency): Fraction of required clinical information items present in LN. Use a domain-specific checklist (e.g., HPI elements for medical cases).
   - `S_align` (Evidence Alignment): For each advice element in LA, retrieve relevant evidence using a domain retriever (e.g., MedCPT, or a RAG pipeline over guidelines), then score alignment as 1.0 (supported), 0.5 (partially supported), or 0.0 (unsupported). Compute mean.

   Reject cases where `S_inf <= 0.5` or `S_align <= 0.5`.

4. **Implement the Controller Agent.** Run a final grounding audit: verify every entity in LN and every advice element in LA can be traced back to explicit text in the original source. Flag and reject any case containing synthesized or hallucinated content not present in the raw data.

5. **Store curated cases with metadata.** Persist each validated case as a JSON record including: source URL, publication date, structured triplet (LN, Q, LA), validation scores, and specialty/language tags. Index by date for temporal slicing.

### Part B: Rubric-Based Evaluation Pipeline

6. **Generate rubrics from gold-standard answers.** For each curated case, decompose LA into evaluation criteria using three sub-steps:
   - **Theme-Guided Fact Extraction**: Assign the case to one or more evaluation themes (e.g., Safety, Context Awareness, Response Depth). Filter LA to retain only theme-relevant facts.
   - **Bipolar Criterion Formulation**: Convert each fact into a binary criterion. Create both positive criteria ("Model mentions drug interaction with patient's existing medication") and negative criteria ("Model recommends contraindicated treatment given patient's renal status").
   - **Axis Assignment & Weighting**: Tag each criterion with an axis (Accuracy, Completeness, Communication, Context, Safety) and assign a signed weight `w_j` in [-10, 10]. Safety violations should carry heavy negative weights.

7. **Implement the rubric scorer.** For each model response, evaluate every criterion as met (1) or unmet (0) using an LLM grader. Compute the final score:
   ```
   score = clip( sum(w_j * indicator(criterion_j met)) / sum(w_k for all w_k > 0), 0, 1 )
   ```
   The denominator is the maximum positive achievable score, making the result a normalized percentage.

8. **Run contamination analysis.** Evaluate each model on the full dataset and on the post-cutoff subset separately. Compute the performance delta. A statistically significant drop on post-cutoff cases indicates the model has been exposed to pre-cutoff test data during training.

9. **Run error taxonomy analysis.** For the bottom-scoring cases per model, classify failures into root-cause categories:
   - **Contextual Neglect (CNIF)**: Model has the knowledge but fails to tailor it to patient-specific constraints
   - **Guideline Overgeneralization (GOPR)**: Model applies general guidelines without case-specific adaptation
   - **Hallucination (MHME)**: Model fabricates medical information
   - **Knowledge Gap (KGOC)**: Model lacks the required factual knowledge

   Compute category distributions to identify whether failures are knowledge-based or reasoning-based.

10. **Schedule recurring pipeline runs.** Automate weekly (or chosen cadence) data harvesting, curation, rubric generation, and evaluation. Append new cases to the benchmark. Track model performance over time to detect score inflation from contamination.

## Concrete Examples

**Example 1: Building a Medical QA Evaluation Pipeline**

User: "I need to evaluate GPT-4 and Claude on real clinical questions. Current benchmarks are stale and I suspect data contamination. Build me a pipeline."

Approach:
1. Set up scrapers for iCliniq and DXY forums, filtering posts from January 2025 onward
2. Implement the three-agent curation chain:

```python
# screener.py
from pydantic import BaseModel
from typing import Optional

class ClinicalTriplet(BaseModel):
    narrative_entities: dict  # LN: demographics, symptoms, history, labs
    primary_query: str        # Q: the clinical question
    actionable_advice: list[str]  # LA: physician recommendations

class ScreenerAgent:
    def __init__(self, llm_client):
        self.llm = llm_client

    def extract_triplet(self, raw_thread: str) -> Optional[ClinicalTriplet]:
        prompt = f"""Extract a structured clinical case from this forum thread
        using the SOAP framework. Return:
        - narrative_entities: patient demographics, symptoms, history, labs
        - primary_query: the main clinical question
        - actionable_advice: list of physician recommendations

        Thread: {raw_thread}"""

        triplet = self.llm.structured_output(prompt, schema=ClinicalTriplet)

        # Forward condition: all components must be non-empty
        if (triplet.narrative_entities
            and triplet.primary_query
            and triplet.actionable_advice):
            return triplet
        return None
```

3. Implement the validator with evidence retrieval:

```python
# validator.py
class ValidatorAgent:
    def __init__(self, llm_client, retriever):
        self.llm = llm_client
        self.retriever = retriever  # e.g., MedCPT or PubMed RAG

    def validate(self, triplet: ClinicalTriplet) -> dict:
        # Query validity (binary)
        s_cc = self.llm.classify(
            f"Is this a clinically meaningful chief complaint per Ely's taxonomy? "
            f"Query: {triplet.primary_query}",
            labels=["valid", "invalid"]
        ) == "valid"

        # Narrative sufficiency (fraction of required items present)
        required_items = ["demographics", "chief_complaint", "history",
                         "medications", "allergies"]
        present = sum(1 for item in required_items
                     if item in triplet.narrative_entities)
        s_inf = present / len(required_items)

        # Evidence alignment (mean score across advice elements)
        alignment_scores = []
        for advice in triplet.actionable_advice:
            evidence = self.retriever.search(advice, top_k=3)
            score = self.llm.score_alignment(advice, evidence)  # 1.0/0.5/0.0
            alignment_scores.append(score)
        s_align = sum(alignment_scores) / len(alignment_scores)

        return {
            "valid": s_cc and s_inf > 0.5 and s_align > 0.5,
            "scores": {"s_cc": s_cc, "s_inf": s_inf, "s_align": s_align}
        }
```

4. Generate rubrics and score model outputs:

```python
# rubric_scorer.py
class RubricCriterion(BaseModel):
    text: str           # e.g., "Mentions renal dose adjustment"
    polarity: str       # "positive" or "negative"
    axis: str           # Accuracy|Completeness|Communication|Context|Safety
    weight: float       # -10 to 10

def score_response(response: str, criteria: list[RubricCriterion],
                   grader_llm) -> float:
    max_positive = sum(c.weight for c in criteria if c.weight > 0)
    if max_positive == 0:
        return 0.0

    total = 0.0
    for criterion in criteria:
        met = grader_llm.evaluate(
            f"Does this response satisfy the criterion?\n"
            f"Criterion: {criterion.text}\n"
            f"Response: {response}",
            labels=["yes", "no"]
        ) == "yes"
        if met:
            total += criterion.weight

    return max(0.0, min(1.0, total / max_positive))
```

Output: A pipeline that weekly ingests fresh clinical cases, validates them through three agents, generates weighted rubrics, and scores models — producing per-axis breakdowns and contamination delta reports.

---

**Example 2: Detecting Data Contamination in a Custom Benchmark**

User: "I have a benchmark of 500 coding questions. I suspect GPT-4 has seen some of them. How do I detect contamination?"

Approach:
1. Establish temporal metadata for each question (date created/published)
2. Identify the training data cutoff for each model under evaluation
3. Split the benchmark into pre-cutoff and post-cutoff subsets
4. Evaluate models on both subsets independently

```python
# contamination_detector.py
from datetime import date

def detect_contamination(
    benchmark: list[dict],  # each has "question", "answer", "publish_date"
    model_cutoff: date,
    evaluate_fn  # function(question) -> score
) -> dict:
    pre_cutoff = [q for q in benchmark if q["publish_date"] <= model_cutoff]
    post_cutoff = [q for q in benchmark if q["publish_date"] > model_cutoff]

    pre_scores = [evaluate_fn(q) for q in pre_cutoff]
    post_scores = [evaluate_fn(q) for q in post_cutoff]

    pre_mean = sum(pre_scores) / len(pre_scores) if pre_scores else 0
    post_mean = sum(post_scores) / len(post_scores) if post_scores else 0
    delta = pre_mean - post_mean

    # Statistical significance test
    from scipy.stats import mannwhitneyu
    stat, p_value = mannwhitneyu(pre_scores, post_scores, alternative="greater")

    return {
        "pre_cutoff_mean": pre_mean,
        "post_cutoff_mean": post_mean,
        "delta": delta,
        "p_value": p_value,
        "contamination_likely": p_value < 0.05 and delta > 0.02
    }
```

Output: A report showing per-model performance deltas, p-values, and a contamination flag. In the paper, 84% of models showed statistically significant degradation on post-cutoff cases.

---

**Example 3: Building Rubrics for Non-Medical Domain Evaluation**

User: "I want to evaluate LLM-generated legal advice using rubrics instead of vibes-based judging."

Approach:
1. Adapt the rubric generation pipeline to legal domain:

```python
# Generate bipolar criteria from a gold-standard legal response
RUBRIC_GENERATION_PROMPT = """
Given this expert legal response to a client question, generate evaluation criteria.

Expert Response: {gold_answer}
Case Context: {case_context}

For each substantive point in the response, create:
1. A POSITIVE criterion (reward if the model includes this point correctly)
2. A NEGATIVE criterion (penalize if the model contradicts this or hallucinates)

Assign each criterion:
- An axis: Accuracy | Completeness | Communication | Context_Awareness | Safety
- A weight from -10 to 10 (safety violations get -8 to -10, minor omissions get 1-3)

Output as JSON array of criteria.
"""
```

2. Weight safety-critical criteria heavily (e.g., recommending an action that violates statute of limitations: weight -10)
3. Score using the normalized rubric formula with both polarities

Output: Per-case rubric with 5-10 weighted criteria, producing scores on [0,1] that correlate strongly with expert ratings and penalize dangerous advice regardless of response length.

## Best Practices

- **Do:** Use bipolar criteria (both positive and negative). Positive-only rubrics miss hallucinations and safety violations entirely. The paper found LLM-as-a-Judge exhibits length bias and fails to penalize safety hazards — negative criteria fix this.
- **Do:** Enforce temporal separation rigorously. Every test case must have a verifiable publication date, and evaluation subsets must be sliced by model training cutoff dates. Without this, contamination is undetectable.
- **Do:** Use the three-agent chain (Screener → Validator → Controller) sequentially, not in parallel. Each agent catches different failure modes — the Controller specifically prevents synthetic contamination that the Validator might miss.
- **Do:** Weight safety criteria with large negative values (-8 to -10). The paper's axis analysis shows Safety errors are among the most consequential but are systematically under-penalized by holistic judging methods.
- **Avoid:** Using a single LLM-as-a-Judge for holistic scoring. The paper demonstrates this approach achieves only 0.26 Pearson correlation with human ratings vs. 0.54 for rubric-based evaluation.
- **Avoid:** Static benchmarks without a refresh cadence. The paper shows 84% of models exhibit inflated scores on stale test data. Schedule weekly or monthly data harvesting at minimum.

## Error Handling

- **Screener produces incomplete triplets:** Expect 40-60% of raw posts to fail the forward condition (missing query, no actionable advice, or insufficient narrative). This is normal — the filtering is intentionally aggressive. Log rejection reasons and monitor rejection rates per source to detect source quality degradation.
- **Validator evidence retrieval fails:** If the retriever returns no relevant evidence for an advice element, default `s_align` for that element to 0.0 (not 0.5). Absence of evidence for a medical claim should be treated as a validation failure.
- **Rubric criteria are ambiguous:** If the grader LLM cannot determine whether a criterion is met, default to "not met" for positive criteria and "not met" for negative criteria (conservative scoring). Log ambiguous cases for human review.
- **Post-cutoff subset is too small:** Contamination analysis requires sufficient statistical power. If fewer than 50 post-cutoff cases are available, do not draw contamination conclusions — wait for more data to accumulate.
- **Score distribution is bimodal:** If models cluster at 0 and 1 rather than showing a gradient, criteria weights may be too extreme. Re-calibrate by reducing maximum weight magnitude and adding more fine-grained criteria.

## Limitations

- The curation pipeline requires domain-specific resources (medical taxonomies, evidence retrievers, specialty checklists). Adapting to a new domain requires rebuilding these components — the agent architecture transfers, but the knowledge base does not.
- Rubric generation depends on high-quality gold-standard answers. If reference answers are mediocre, the generated criteria will encode mediocrity. The system cannot exceed the quality ceiling of its reference data.
- The three-agent curation chain requires three sequential LLM calls per case, making it expensive at scale. Budget approximately 3x the cost of single-pass processing.
- Temporal separation detects contamination empirically but cannot prove it definitively. Performance drops on post-cutoff data could also reflect genuine domain shift or increased difficulty in newer cases.
- The rubric scorer still uses an LLM to evaluate binary criteria, introducing its own error rate. The paper reports Macro F1 of 0.76 vs. human inter-rater F1 of 0.89 — a meaningful gap remains.

## Reference

**Paper:** [LiveMedBench: A Contamination-Free Medical Benchmark for LLMs with Automated Rubric Evaluation](https://arxiv.org/abs/2602.10367v1) — Yan et al., 2026. Focus on Section 3 (Multi-Agent Clinical Curation Framework) for the three-agent pipeline design, and Section 4 (Automated Rubric-based Evaluation) for the bipolar criterion generation and weighted scoring formula.