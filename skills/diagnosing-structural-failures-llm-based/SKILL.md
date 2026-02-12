---
name: "diagnosing-structural-failures-llm-based"
description: "Diagnose and mitigate structural failures in LLM-based evidence extraction for meta-analysis and systematic reviews. Implements a progressive schema-constrained query framework to identify role reversals, binding drift, instance compression, and numeric misattribution. Use when: 'extract structured data from research papers', 'build a meta-analysis pipeline', 'diagnose why my LLM extraction is failing', 'evaluate evidence extraction quality', 'bind variables to effect sizes from papers', 'aggregate extracted study data reliably'."
---

# Diagnosing Structural Failures in LLM-Based Evidence Extraction

This skill enables Claude to build, evaluate, and debug LLM-based pipelines that extract structured evidence from scientific papers for meta-analysis and systematic reviews. It applies the diagnostic framework from Tan & D'Souza (2026), which decomposes extraction into a progression of schema-constrained queries at increasing relational complexity -- from isolated atoms (single fields) through multi-field binding (variable-role-method tuples) to corpus-level aggregation. By testing each level independently, you can pinpoint exactly where structural fidelity breaks down rather than treating extraction as a monolithic pass/fail.

## When to Use

- When building a pipeline to extract structured study records (sample size, variables, effect sizes, methods) from a corpus of scientific papers
- When an existing LLM extraction pipeline produces plausible-looking but statistically unreliable outputs, and you need to diagnose why
- When designing prompts or schemas for pulling relational data (IV/DV pairs, method-effect bindings) from research documents
- When evaluating whether an LLM can reliably bind roles, methods, and numeric values across multiple analyses within a single paper
- When aggregating extracted data downstream (computing means, filtering by thresholds) and results don't match manual spot-checks
- When choosing between per-document extraction vs. concatenated multi-document input for a review pipeline

## Key Technique

The core insight is that LLM extraction failures in meta-analysis are not entity recognition problems -- models can find variable names, numbers, and method labels reasonably well in isolation. The failures are **structural**: when a paper reports multiple analyses, the LLM conflates which variable plays which role in which analysis with which effect size. The paper defines four specific failure modes: **role reversal** (swapping independent and dependent variables despite explicit cues), **cross-analysis binding drift** (correctly extracting a variable but associating it with the wrong method or effect size from a different analysis in the same paper), **multi-instance compression** (losing entries when a results section is dense with many reported analyses), and **numeric misattribution** (pairing an effect size value with the wrong variable-method combination).

The diagnostic framework organizes extraction into two dimensions -- **semantic focus** (object-centric properties like sample size/location vs. method-centric properties like statistical methods/effect sizes) and **structural complexity** (L1 single atoms, L2 multi-field binding, C corpus-derived aggregation). This creates a grid of queries from trivial (extract the country) to extremely hard (extract the full 6-tuple: document ID, IV, DV, method, conditions, effect size). By measuring F1 at each cell, you identify the exact complexity threshold where your pipeline breaks. The paper shows that even state-of-the-art models achieve F1=0.00 on the highest-arity tuple extraction, and that per-document processing consistently outperforms concatenated multi-document input.

Downstream aggregation amplifies these failures catastrophically. A query like "compute the mean sample size across all papers" requires every upstream extraction to be both correct and complete -- a single missing document invalidates the statistic. The paper reports >60% failure rates on such derived queries even when underlying atom extraction has F1=0.66.

## Step-by-Step Workflow

1. **Define the meta-analytic schema.** Enumerate every field your review needs: document ID, population (P), geolocation (G), sample size (N), statistical method (A), independent variable (IV), dependent variable (DV), scale/unit (S/U), conditions (C), effect size (E). Assign each a data type (string, integer, float, enum) and document-attribution constraint (every value must trace to a specific source paper).

2. **Decompose extraction into a query ladder.** Build queries at four complexity levels:
   - **L1 atoms**: Extract each field independently ("Extract the sample size from this paper")
   - **L2 pairs/triples**: Bind related fields ("Extract each (IV, DV) pair with their scale")
   - **L2 full tuples**: Bind all method-centric fields ("Extract (IV, DV, method, conditions, effect size) for each analysis")
   - **C aggregation**: Derive corpus statistics ("Count papers with N > 100", "Compute mean effect size")

3. **Construct a gold-standard evaluation set.** Manually annotate 5-15 papers per domain with ground-truth values for every schema field. Store as structured JSON with document attribution. This is non-negotiable -- without gold data, you cannot distinguish extraction errors from annotation ambiguity.

4. **Run L1 atom extraction first and measure baseline.** Process each paper independently (not concatenated). Use explicit schema-constrained prompts that name the exact field and expected format. Compute precision, recall, and F1 per field against your gold set.

5. **Run L2 binding queries and compare against L1.** If L1 recall for variables is 0.70 but L2 (IV, DV) pair recall drops to 0.30, the gap is structural binding failure, not entity recognition. Check specifically for role reversal by comparing role-agnostic variable extraction (V) against role-constrained (IV, DV).

6. **Diagnose specific failure modes on L2 errors.** For each false positive tuple, classify the error:
   - *Role reversal*: IV and DV are swapped (the variable names are correct but roles are wrong)
   - *Binding drift*: Variables are from one analysis, method/effect from another
   - *Instance compression*: The paper reports N analyses but the model returns fewer
   - *Numeric misattribution*: The effect size value exists in the paper but belongs to a different analysis

7. **Test per-document vs. multi-document regimes.** Run the same queries with papers concatenated into a single prompt. Measure the F1 delta. Expect degradation, especially on L2 method-centric tasks. If global input drops M2 F1 from 0.22 to 0.05 (as in the paper), use per-document extraction exclusively.

8. **Evaluate downstream aggregation independently.** Run your C-level queries (counts, means, filters) on both gold data and extracted data. The gap reveals aggregation amplification. If upstream F1 is 0.66 but downstream count accuracy is 0.20, your pipeline is not reliable for corpus-level statistics.

9. **Implement structural mitigations.** Based on diagnosed failure modes:
   - For role reversal: Add explicit role-definition prompts with examples; use two-pass extraction (first extract variables, then assign roles)
   - For binding drift: Process one analysis at a time; use table-aware extraction that respects row boundaries
   - For instance compression: Ask the model to first count analyses, then extract each one individually
   - For numeric misattribution: Extract numbers with their surrounding context; verify effect sizes against reported methods

10. **Re-run the full query ladder after mitigations and compare.** Measure improvement at each complexity level. Iterate until L2 binding meets your reliability threshold or document that manual review is required for specific query levels.

## Concrete Examples

**Example 1: Diagnosing a failing medical review extraction pipeline**

User: "I'm extracting study data from 50 clinical papers for a meta-analysis on BMI and cardiovascular outcomes. My LLM pipeline gets the study countries and sample sizes mostly right, but the effect sizes in my summary table don't match the papers. How do I figure out what's going wrong?"

Approach:
1. Build the query ladder for your schema:
   - L1: Extract country, sample size, statistical method, IV, DV, effect size independently
   - L2: Extract (IV, DV) pairs; then (IV, DV, method, effect size) tuples
   - C: Compute pooled effect size across papers
2. Manually annotate 10 papers as gold standard
3. Run L1 queries -- expect decent F1 (0.50-0.65)
4. Run L2 tuple extraction -- this is where the break will appear
5. Classify errors on the L2 false positives

Output (diagnostic report):
```
## Extraction Diagnostic Report

### L1 Atom Extraction (per-document)
| Field       | Precision | Recall | F1   |
|-------------|-----------|--------|------|
| Country     | 0.91      | 0.87   | 0.89 |
| Sample Size | 0.78      | 0.72   | 0.75 |
| Method      | 0.83      | 0.69   | 0.75 |
| IV          | 0.71      | 0.65   | 0.68 |
| DV          | 0.74      | 0.63   | 0.68 |
| Effect Size | 0.80      | 0.58   | 0.67 |

### L2 Tuple Extraction (IV, DV, Method, Effect Size)
| Metric    | Score |
|-----------|-------|
| Precision | 0.32  |
| Recall    | 0.18  |
| F1        | 0.23  |

### Failure Mode Breakdown (on 47 false positive tuples)
- Role reversal (IV/DV swapped):     8 (17.0%)
- Cross-analysis binding drift:      11 (23.4%)
- Numeric misattribution:            14 (29.8%)
- Instance compression (missing):    9  (19.1%)
- Other (hallucinated values):       5  (10.6%)

### Diagnosis
Primary failure: Numeric misattribution. Effect size values are
real numbers from the papers but bound to the wrong IV/DV/method
combination. This is a binding problem, not a recognition problem.

### Recommended Mitigations
1. Switch to table-aware extraction (parse results tables separately)
2. Extract one analysis per prompt turn instead of all at once
3. Require the model to quote the sentence containing each effect size
```

**Example 2: Building a new extraction pipeline with structural safeguards**

User: "I need to build a pipeline to extract study characteristics from agricultural science papers for a systematic review on drought stress and crop yield. How should I structure the extraction to avoid common LLM failures?"

Approach:
1. Define schema: `{doc_id, population, geolocation, sample_size, IV, DV, method, conditions, effect_size, scale, unit}`
2. Implement the query ladder as separate extraction stages
3. Build in per-level validation

Output (pipeline architecture):
```python
QUERY_LADDER = {
    # Level 1: Atom extraction (run first, validate before proceeding)
    "L1_object": [
        {"field": "geolocation", "prompt": "Extract the country where this study was conducted. Return ISO country name only."},
        {"field": "sample_size", "prompt": "Extract the primary sample size (N) reported. Return integer only."},
        {"field": "population", "prompt": "Extract the study population or organism. Return a short noun phrase."},
    ],
    "L1_method": [
        {"field": "methods", "prompt": "List each distinct statistical method used (e.g., Pearson correlation, linear regression, ANOVA). One per line."},
        {"field": "variables_agnostic", "prompt": "List all study variables mentioned, regardless of role. One per line."},
        {"field": "IV", "prompt": "List variables explicitly used as independent/predictor variables. One per line."},
        {"field": "DV", "prompt": "List variables explicitly used as dependent/outcome variables. One per line."},
    ],

    # Level 2: Binding extraction (run per-analysis, not per-paper)
    "L2_pairs": [
        {"fields": ["IV", "DV"], "prompt": "For each reported analysis, extract the (independent variable, dependent variable) pair. Format: IV | DV, one pair per line."},
        {"fields": ["IV", "DV", "scale", "unit"], "prompt": "For each variable pair, also extract scale (nominal/ordinal/continuous) and unit of measurement."},
    ],
    "L2_full_tuple": [
        {"fields": ["IV", "DV", "method", "conditions", "effect_size"],
         "prompt": "For each individual analysis reported in this paper, extract a row: IV | DV | statistical method | conditions | effect size value. One row per analysis. Do NOT combine analyses."},
    ],

    # Level C: Aggregation (run on extracted data, not on papers)
    "C_derived": [
        {"operation": "count", "filter": "sample_size > 100"},
        {"operation": "mean", "field": "sample_size"},
        {"operation": "filter", "condition": "effect_size > 0.7"},
    ],
}

# Key structural safeguard: two-pass role assignment
def extract_with_role_validation(paper_text, llm_client):
    # Pass 1: role-agnostic variable list
    all_vars = llm_client.extract("L1_method", "variables_agnostic", paper_text)
    # Pass 2: role-assigned extraction
    ivs = llm_client.extract("L1_method", "IV", paper_text)
    dvs = llm_client.extract("L1_method", "DV", paper_text)
    # Validate: every IV and DV should appear in all_vars
    orphan_ivs = [v for v in ivs if v not in all_vars]
    orphan_dvs = [v for v in dvs if v not in all_vars]
    if orphan_ivs or orphan_dvs:
        flag_for_review(paper_text, orphan_ivs, orphan_dvs)
    # Validate: check for role reversal (same var as both IV and DV)
    role_conflicts = set(ivs) & set(dvs)
    if role_conflicts:
        flag_role_ambiguity(paper_text, role_conflicts)
    return ivs, dvs

# Key structural safeguard: per-analysis decomposition
def extract_tuples_safely(paper_text, llm_client):
    # Step 1: count analyses
    n_analyses = llm_client.query(
        "How many distinct statistical analyses (each with its own effect size) are reported? Return integer."
    , paper_text)
    # Step 2: extract each analysis individually
    tuples = []
    for i in range(int(n_analyses)):
        t = llm_client.query(
            f"Extract ONLY analysis #{i+1} (in order of appearance): IV | DV | method | conditions | effect size."
        , paper_text)
        tuples.append(t)
    return tuples
```

**Example 3: Evaluating per-document vs. multi-document input**

User: "Should I send all 11 papers to the LLM at once or process them one at a time?"

Approach:
1. Run the same L1 and L2 queries under both regimes on your gold set
2. Compare F1 at each query level
3. Calculate the degradation delta

Output:
```
## Input Regime Comparison (11 agricultural science papers)

| Query Level          | Per-Doc F1 | Global F1 | Delta   |
|----------------------|-----------|-----------|---------|
| L1 Object (G,N,P)   | 0.71      | 0.63      | -0.08   |
| L1 Method (A,IV,DV)  | 0.58      | 0.31      | -0.27   |
| L2 Pairs (IV,DV)     | 0.39      | 0.14      | -0.25   |
| L2 Full Tuple        | 0.08      | 0.00      | -0.08   |
| C Aggregation        | 0.15      | 0.05      | -0.10   |

Recommendation: Use per-document extraction. The global regime
causes a 25-point F1 drop on method-centric binding tasks due
to cross-document binding drift. The token savings of a single
call do not justify the structural degradation.
```

## Best Practices

- **Do:** Always process papers individually, then aggregate extracted data in a separate non-LLM step. Per-document extraction consistently outperforms concatenated input by 10-25 F1 points on binding tasks.
- **Do:** Run the full query ladder (L1 -> L2 -> C) on a gold-annotated subset before scaling. The L1-to-L2 F1 drop is your binding reliability signal -- if it exceeds 0.30, your pipeline needs structural mitigations before production use.
- **Do:** Use two-pass extraction for role assignment: first extract variables role-agnostically, then assign IV/DV roles in a separate prompt. This catches role reversals that account for ~15% of binding errors.
- **Do:** Require the model to quote source text alongside extracted values. This creates an auditable chain that catches numeric misattribution.
- **Avoid:** Treating extraction as a single monolithic prompt that asks for everything at once. Dense multi-field prompts trigger instance compression and binding drift.
- **Avoid:** Trusting corpus-level aggregation (means, counts, filtered sets) without first validating per-document completeness. A single missing document can invalidate a computed statistic. Always verify extraction count matches expected document count before aggregating.

## Error Handling

- **Role reversal detected (IV/DV overlap):** Flag papers where the same variable appears as both IV and DV. Re-extract with explicit methodological context ("In analysis X, which variable was manipulated/measured?"). If ambiguity persists, escalate to manual review.
- **Tuple count mismatch:** If the model extracts fewer tuples than analyses reported in a paper, use the count-then-extract pattern: first ask the model to count distinct analyses, then extract each by ordinal position.
- **Effect size format inconsistency:** Normalize effect size types (r, OR, beta, d, eta-squared) to labeled values `{type: "r", value: 0.45}` rather than bare numbers. This prevents cross-metric aggregation errors.
- **Semantic matching failures in evaluation:** When comparing extracted values to gold standard, use character-level similarity with a 0.95 threshold. Below that threshold, use an LLM judge to confirm semantic equivalence (handles paraphrased variable names).
- **Aggregation cascade failure:** If a C-level query returns implausible results, trace back through L2 and L1 outputs for the contributing documents. The failure is almost always upstream binding, not arithmetic.

## Limitations

- The diagnostic framework requires a manually curated gold-standard corpus (5-15 papers minimum per domain). There is no shortcut -- without ground truth, you cannot distinguish extraction errors from legitimate ambiguity.
- Full meta-analytic tuple extraction (6+ fields bound together) achieves near-zero F1 even with state-of-the-art models. For high-arity tuples, treat LLM extraction as a first-pass filter that still requires human verification.
- The framework is designed for empirical research papers with quantitative results sections. It does not apply to qualitative reviews, theoretical papers, or papers without reported effect sizes.
- Domain-specific terminology and reporting conventions cause significant performance variance (F1 ranges from 0.08 to 0.47 across domains). A pipeline tuned for medical studies may fail on agricultural or social science papers without re-calibration.
- The mitigations (per-analysis decomposition, two-pass role extraction) increase API cost linearly with the number of analyses per paper. For papers with 30+ reported analyses, this becomes expensive.

## Reference

Tan, Z. & D'Souza, J. (2026). *Diagnosing Structural Failures in LLM-Based Evidence Extraction for Meta-Analysis.* IRCDL 2026. [arXiv:2602.10881](https://arxiv.org/abs/2602.10881v1) | [Code](https://github.com/zhiyintan/LLM-Meta-Analysis)

Key takeaway: LLM extraction failures in meta-analysis are structural (role reversal, binding drift, instance compression, numeric misattribution), not entity recognition failures. Diagnose by measuring F1 at each complexity level of a query ladder, and mitigate by decomposing extraction into per-analysis, per-field passes with explicit role assignment.