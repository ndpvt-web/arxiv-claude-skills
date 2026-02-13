---
name: "on-use-support-conduction"
description: "LLM-assisted systematic literature review and mapping study pipeline. Automates screening, data extraction, and classification of research papers while maintaining human-in-the-loop verification. Use when the user says 'systematic review', 'literature mapping', 'screen these papers', 'extract data from papers', 'classify research articles', or 'survey the literature on'."
---

# LLM-Assisted Systematic Mapping Study Pipeline

This skill enables Claude to act as a research assistant for conducting systematic mapping studies and systematic literature reviews (SLRs). Based on the methodology reported by Barros et al. (2026), it implements a structured pipeline where the LLM handles repetitive screening, data extraction, and classification tasks while the human researcher retains all methodological decisions and final judgments. The core insight is that LLMs deliver the most value on structured, objective tasks with clear criteria — not on interpretive analysis — and that every LLM output must be verified against original sources before acceptance.

## When to Use

- When the user has a collection of paper titles/abstracts and needs to screen them against inclusion/exclusion criteria
- When the user wants to extract structured metadata (authors, methods, findings, categories) from a set of research papers
- When the user asks to classify papers by research type, domain, methodology, or any taxonomy
- When the user is planning a systematic mapping study or SLR and needs help structuring the protocol
- When the user has a CSV/BibTeX of search results and wants to triage them for relevance
- When the user needs to synthesize or tabulate findings across multiple studies
- When the user asks to build a research landscape or "map" of a topic area

## Key Technique

The paper's central contribution is a phased human-LLM collaboration model for systematic mapping studies. Rather than replacing the researcher, the LLM serves as a high-throughput assistant for the three most time-consuming phases: **screening** (deciding which papers to include), **data extraction** (pulling structured information from each paper), and **classification** (categorizing papers along predefined dimensions). The human researcher designs the protocol, defines all criteria, makes final decisions, and verifies every LLM output.

The critical methodological finding is that prompt quality dominates outcome quality. Effective prompts must include: explicit inclusion/exclusion criteria with definitions, the exact output format expected (structured tables or JSON), few-shot examples of correct classifications, and constraints specifying what to exclude. Vague or underspecified prompts produce inconsistent results that cost more time to fix than they save. The authors recommend iterative prompt refinement — start with a small sample, evaluate LLM outputs against manual results, tighten the prompt, and repeat until agreement stabilizes before scaling to the full dataset.

A non-negotiable principle is **cross-verification**: every LLM-generated screening decision, extracted data point, or classification must be checked against the original source document. Hallucinations are especially dangerous in data extraction, where the LLM may fabricate plausible-sounding methodological details or findings that do not appear in the paper. The recommended mitigation is to ask the LLM to cite specific passages supporting its outputs, then verify those citations exist.

## Step-by-Step Workflow

1. **Define the review protocol.** Collect the user's research questions, scope, inclusion criteria (IC), exclusion criteria (EC), target databases, and search strings. Structure these into a formal protocol document with unambiguous definitions for every criterion.

2. **Ingest the candidate paper list.** Parse the user's input (BibTeX, CSV, JSON, or plain text) into a structured format with fields: `id`, `title`, `authors`, `year`, `venue`, `abstract`, `doi`. Normalize and deduplicate entries.

3. **Build a screening prompt template.** Construct a prompt that includes: (a) the research question, (b) each IC and EC with definitions and boundary examples, (c) the instruction to output a JSON object with `{id, decision: "include"|"exclude"|"uncertain", reasoning: "..."}`, and (d) 2-3 few-shot examples showing correct include/exclude decisions with reasoning.

4. **Screen papers in batches.** Process papers in batches of 5-10 (not one-by-one, not hundreds at once). For each batch, pass titles and abstracts through the screening prompt. Flag all "uncertain" papers for human review. Output a summary table of decisions with counts.

5. **Validate screening results on a calibration set.** Before scaling, run the screening prompt on a sample of ~20 papers that the user has already manually classified. Compute agreement (Cohen's kappa or simple percent agreement). If agreement is below 0.8, refine the prompt — tighten definitions, add more examples, clarify edge cases — and re-run until the threshold is met.

6. **Extract data from included papers.** For each included paper, construct an extraction prompt specifying the exact fields to extract (e.g., `research_type`, `methodology`, `sample_size`, `main_findings`, `limitations`, `LLM_model_used`). Request output as a JSON object. Provide key excerpts or the abstract rather than relying on the LLM's training data — never ask the LLM to recall paper content from memory.

7. **Classify papers along mapping dimensions.** Define the classification taxonomy (e.g., research type facet: empirical/conceptual/experience report; domain facet: testing/requirements/maintenance). Build a classification prompt with category definitions and exemplars. Apply it to each paper's extracted data.

8. **Cross-verify all outputs.** For every extracted field and classification, generate a verification checklist. Flag any output where the LLM could not cite a specific passage. Mark these for mandatory human review. Report verification coverage (% of fields verified) to the user.

9. **Synthesize results into mapping tables.** Aggregate classifications into frequency tables, bubble plots, or heatmaps showing the distribution of studies across dimensions (year x topic, method x domain, etc.). Highlight gaps in the literature.

10. **Document the process.** Record every prompt version used, the calibration results, agreement metrics, number of manual corrections, and any protocol deviations. This transparency is essential for reproducibility.

## Concrete Examples

**Example 1: Screening a batch of papers**

User: "I have 150 papers from a Scopus search on 'LLMs in software testing'. Here are my inclusion criteria: (IC1) must discuss using LLMs for test generation or test oracle, (IC2) must be a peer-reviewed paper. Exclusion: (EC1) short papers under 4 pages, (EC2) papers not in English. Screen these first 5 for me."

Approach:
1. Parse the 5 paper entries (title + abstract provided by user)
2. Build a screening prompt with IC1, IC2, EC1, EC2 defined explicitly
3. Process each paper, outputting structured decisions

Output:
```json
[
  {"id": 1, "title": "TestGen-LLM: ...", "decision": "include", "reasoning": "Directly addresses LLM-based test generation (IC1), published at ICSE (IC2), 10 pages, English."},
  {"id": 2, "title": "A Survey of AI...", "decision": "exclude", "reasoning": "Discusses AI broadly but does not specifically address LLMs for test generation or oracles. Fails IC1."},
  {"id": 3, "title": "ChatGPT para pruebas...", "decision": "exclude", "reasoning": "Paper is in Spanish. Fails EC2."},
  {"id": 4, "title": "LLM Oracles for...", "decision": "include", "reasoning": "Proposes LLM-based test oracles (IC1), ISSTA 2025 full paper (IC2), 12 pages, English."},
  {"id": 5, "title": "Brief Note on...", "decision": "uncertain", "reasoning": "Abstract mentions LLM test generation but is a 2-page workshop abstract. Meets IC1 but borderline on EC1. Flagged for human review."}
]
```
Summary: 2 included, 2 excluded, 1 uncertain (requires human decision on page count threshold).

---

**Example 2: Structured data extraction**

User: "For each of these 3 included papers, extract: research type, LLM model used, SE activity, evaluation method, and main finding."

Approach:
1. Confirm the user can provide abstracts or key excerpts (do not rely on LLM memory of papers)
2. Build an extraction prompt with field definitions and output format
3. Extract data from each paper's provided text

Output:
```markdown
| Paper | Research Type | LLM Model | SE Activity | Evaluation | Main Finding |
|-------|--------------|-----------|-------------|------------|--------------|
| TestGen-LLM | Empirical | GPT-4 | Unit test generation | Mutation score on 5 OSS projects | 32% improvement over EvoSuite baseline |
| LLM Oracles | Empirical | Claude 3 | Test oracle | Precision/recall on Defects4J | 89% precision, 74% recall for oracle assertions |
| AutoSpec | Conceptual | GPT-3.5 | Requirements elicitation | Case study, 2 industry projects | Reduced specification time by 45% but introduced 12% false requirements |

**Verification flags**: AutoSpec "45%" and "12%" figures need human verification against original paper sections 4.2-4.3.
```

---

**Example 3: Calibrating a screening prompt**

User: "I manually screened 20 papers. Here are my decisions. Can you check how well your screening prompt agrees?"

Approach:
1. Run the screening prompt on all 20 papers
2. Compare LLM decisions against human ground truth
3. Compute agreement metrics and identify disagreement patterns

Output:
```
Calibration Results:
- Agreement: 17/20 (85%)
- Cohen's Kappa: 0.78
- Disagreements:
  - Paper #7: LLM said "include", human said "exclude"
    Reason: LLM interpreted "software analysis" as meeting IC1 (testing).
    Fix: Add to prompt "IC1 clarification: general software analysis tools
    that do not specifically target test artifacts are excluded."
  - Paper #12: LLM said "exclude", human said "include"
    Reason: Abstract was ambiguous; key testing context only in keywords.
    Fix: Add instruction "Also consider author keywords when abstract is
    ambiguous about SE activity."
  - Paper #19: LLM said "uncertain", human said "include"
    Reason: Borderline page count (4 pages exactly).
    Fix: Clarify "EC1: strictly fewer than 4 pages."

Recommendation: Apply these 3 prompt fixes and re-run calibration.
Target kappa >= 0.80 before scaling to full dataset.
```

## Best Practices

**Do:**
- Always provide the LLM with the actual text (abstract, excerpt) rather than asking it to recall paper content from its training data. Memory-based extraction is the primary source of hallucinations.
- Use few-shot examples in every prompt — 2-3 worked examples of correct screening decisions or extraction outputs dramatically improve consistency.
- Batch papers in groups of 5-10 for screening. Single-paper processing is inefficient; large batches cause quality degradation as context grows.
- Run a calibration round on 15-20 pre-classified papers before scaling. Measure agreement quantitatively (kappa >= 0.8) and refine the prompt based on specific disagreement patterns.
- Version-control your prompts. Record which prompt version produced which outputs so results are reproducible and auditable.

**Avoid:**
- Never accept LLM screening decisions or extracted data without cross-verification against the source document. Even high-confidence outputs can be hallucinated.
- Never use the LLM for interpretive synthesis (e.g., "what does the literature overall suggest?") without first completing human-verified extraction. Synthesis on unverified data compounds errors.
- Never skip prompt calibration to save time. Uncalibrated prompts on 500 papers will create more rework than the calibration would have cost.
- Never ask the LLM to make final inclusion/exclusion decisions on uncertain cases. These require human judgment and should always be flagged for researcher review.

## Error Handling

**Hallucinated data points:** When the LLM reports a specific number, method, or finding that seems too precise or convenient, flag it with `[VERIFY]` and instruct the user to check the original paper. Ask the LLM to quote the exact sentence supporting the claim — if it cannot, the data point is likely fabricated.

**Inconsistent classifications:** If the same paper receives different classifications across runs, the taxonomy definitions are ambiguous. Add boundary examples to the classification prompt (e.g., "A paper that proposes a tool AND evaluates it is 'Empirical', not 'Conceptual'").

**Context window overflow:** For papers with long full texts, extract data section-by-section (methods, results, discussion) rather than feeding the entire paper. Summarize each section's extraction, then merge.

**Low calibration agreement:** If kappa stays below 0.7 after 3 prompt iterations, the inclusion criteria themselves may be ambiguous. Recommend the user revisit and tighten the protocol definitions before continuing LLM-assisted screening.

**Batch size degradation:** If screening quality drops in larger batches, reduce batch size. Quality typically degrades above 10-15 papers per prompt due to attention dilution.

## Limitations

- This approach is designed for **structured, criteria-based tasks** (screening, extraction, classification). It does not replace human judgment for qualitative synthesis, theory building, or critical appraisal of study quality.
- The LLM cannot access papers behind paywalls. The user must provide the text content (abstracts, excerpts, or full text) as input.
- Prompt engineering effort is front-loaded and substantial. For small reviews (under 50 papers), manual processing may be faster than designing, calibrating, and verifying LLM-assisted processing.
- Hallucination risk is never zero. Even with cross-verification protocols, subtle fabrications (e.g., slightly altered numbers) can slip through if the verifier is fatigued or unfamiliar with the domain.
- Reproducibility depends on the LLM's temperature and version. Results may vary across sessions unless temperature is set to 0 and the model version is documented.

## Reference

Barros, C. F., Kalinowski, M., Kassab, M., & Graciano Neto, V. V. (2026). *On the Use of a Large Language Model to Support the Conduction of a Systematic Mapping Study: A Brief Report from a Practitioner's View.* arXiv:2602.10147v1. [https://arxiv.org/abs/2602.10147v1](https://arxiv.org/abs/2602.10147v1)

Key takeaway: LLMs achieve the highest value in systematic reviews when deployed as **structured assistants** for screening/extraction with calibrated prompts and mandatory human verification — not as autonomous reviewers.