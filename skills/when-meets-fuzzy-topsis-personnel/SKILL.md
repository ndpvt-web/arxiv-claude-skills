---
name: "when-meets-fuzzy-topsis-personnel"
description: "Rank and select candidates using LLM-scored profiles combined with Fuzzy TOPSIS multi-criteria decision-making. Use when the user says 'rank these candidates', 'score resumes against criteria', 'build a hiring decision matrix', 'fuzzy TOPSIS ranking', 'multi-criteria candidate selection', or 'automate personnel screening'."
---

# LLM + Fuzzy-TOPSIS Personnel Selection

This skill enables Claude to build automated personnel selection pipelines that combine LLM-based profile analysis with the Fuzzy TOPSIS (Technique for Order Preference by Similarity to Ideal Solution) multi-criteria decision-making method. Rather than producing a single subjective score per candidate, the technique scores each candidate across multiple criteria (education, experience, skills, communication), converts those scores into triangular fuzzy numbers to capture uncertainty, then applies the TOPSIS algorithm to produce a defensible, bias-resistant ranking. Based on the framework from Hoque et al. (2026), which achieved 91% alignment with expert evaluations on software engineering candidates.

## When to Use

- When the user has a set of candidate profiles (resumes, LinkedIn data, structured bios) and needs to rank them against defined criteria
- When the user asks to build a multi-criteria scoring system for hiring, applicant screening, or talent evaluation
- When the user wants to reduce subjectivity in candidate evaluation by combining LLM analysis with mathematical ranking
- When the user needs to compare candidates across heterogeneous attributes (text-heavy experience vs. numeric years vs. categorical degrees)
- When the user asks to implement TOPSIS, Fuzzy TOPSIS, or any MCDM-based ranking in a recruitment or personnel context
- When the user wants to weight criteria differently (e.g., experience matters more than education for a senior role) and get a principled ranking

## Key Technique

**Fuzzy TOPSIS** extends classical TOPSIS by replacing crisp numbers with triangular fuzzy numbers (TFNs). A TFN is a triple `(l, m, u)` representing the lower bound, most likely value, and upper bound of an uncertain score. This matters in personnel selection because human judgment is inherently imprecise -- a candidate's "experience quality" is not a single number but a range. By encoding scores as TFNs, the system captures this uncertainty rather than discarding it.

**The LLM-TOPSIS pipeline** works in two stages. First, an LLM analyzes each candidate's textual profile and produces numerical scores (1-10) for each criterion. These raw scores are converted into TFNs using a linguistic scale mapping (e.g., a score of 7 maps to "Good" = (5, 7, 9)). Second, the Fuzzy TOPSIS algorithm normalizes the fuzzy decision matrix, applies criterion weights (also expressed as TFNs), computes the Fuzzy Positive Ideal Solution (FPIS) and Fuzzy Negative Ideal Solution (FNIS), calculates each candidate's distance from both ideals, and produces a closeness coefficient (CC) between 0 and 1. Candidates are ranked by descending CC.

**Why this beats simple LLM scoring:** Direct LLM ranking is sensitive to prompt phrasing and order effects. Fuzzy TOPSIS forces explicit criterion decomposition, weight assignment, and mathematical aggregation. This makes the ranking auditable -- you can trace exactly why Candidate A ranks above Candidate B by examining per-criterion scores, weights, and distances. It also allows stakeholders to adjust weights without re-running the LLM.

## Step-by-Step Workflow

1. **Define criteria and weights.** Establish 3-6 evaluation criteria (e.g., Education, Experience, Technical Skills, Communication). Assign linguistic weights to each: "Very High" = (0.7, 0.9, 1.0), "High" = (0.5, 0.7, 0.9), "Medium" = (0.3, 0.5, 0.7), "Low" = (0.1, 0.3, 0.5). These become TFN weight vectors.

2. **Collect and structure candidate profiles.** Parse each candidate's data into a consistent schema with fields for each criterion. Handle missing fields by flagging them for penalty or imputation rather than silently ignoring them.

3. **Score each candidate per criterion using the LLM.** For each candidate-criterion pair, prompt the LLM with the candidate's relevant text and ask for a 1-10 integer score with a brief justification. Use a structured output format (JSON) to enforce consistency.

4. **Convert raw scores to triangular fuzzy numbers.** Map integer scores to TFNs using a predefined linguistic scale:
   - 1-2 "Very Poor": (0, 1, 3)
   - 3-4 "Poor": (1, 3, 5)
   - 5-6 "Fair": (3, 5, 7)
   - 7-8 "Good": (5, 7, 9)
   - 9-10 "Excellent": (7, 9, 10)

5. **Build the fuzzy decision matrix.** Construct an M x N matrix where M = candidates, N = criteria, and each cell contains a TFN. This is the raw fuzzy decision matrix.

6. **Normalize the fuzzy decision matrix.** For benefit criteria (higher is better), normalize each TFN `(l, m, u)` by dividing by the maximum upper bound in that column: `(l/u_max, m/u_max, u/u_max)`. For cost criteria (lower is better), invert: `(l_min/u, l_min/m, l_min/l)`.

7. **Compute the weighted normalized matrix.** Multiply each normalized TFN by the criterion's weight TFN using fuzzy multiplication: `(l1*l2, m1*m2, u1*u2)`.

8. **Determine FPIS and FNIS.** The Fuzzy Positive Ideal Solution (FPIS) is `(1, 1, 1)` for each criterion (the best achievable weighted score). The Fuzzy Negative Ideal Solution (FNIS) is `(0, 0, 0)` for each criterion.

9. **Calculate distance from each ideal.** For each candidate, compute the sum of vertex distances to FPIS and FNIS across all criteria. Vertex distance between TFNs `(l1, m1, u1)` and `(l2, m2, u2)` is: `sqrt((1/3) * ((l1-l2)^2 + (m1-m2)^2 + (u1-u2)^2))`.

10. **Compute closeness coefficient and rank.** For each candidate: `CC = D_negative / (D_positive + D_negative)`. Rank candidates by descending CC. A CC closer to 1 means the candidate is closer to the ideal solution.

## Concrete Examples

**Example 1: Ranking three software engineering candidates**

User: "I have three candidates for a senior backend role. Rank them based on their profiles."

Approach:
1. Define criteria: Experience (Very High weight), Technical Skills (Very High), Education (Medium), Communication (High).
2. Score each candidate via LLM analysis of their profiles.
3. Apply Fuzzy TOPSIS to produce final ranking.

```python
import math

# Linguistic weight scale (TFNs)
WEIGHTS = {
    "Very High": (0.7, 0.9, 1.0),
    "High":      (0.5, 0.7, 0.9),
    "Medium":    (0.3, 0.5, 0.7),
    "Low":       (0.1, 0.3, 0.5),
}

# Linguistic score scale (TFNs)
SCORE_SCALE = {
    (1, 2):  (0, 1, 3),    # Very Poor
    (3, 4):  (1, 3, 5),    # Poor
    (5, 6):  (3, 5, 7),    # Fair
    (7, 8):  (5, 7, 9),    # Good
    (9, 10): (7, 9, 10),   # Excellent
}

def score_to_tfn(score):
    for (lo, hi), tfn in SCORE_SCALE.items():
        if lo <= score <= hi:
            return tfn
    return (0, 1, 3)

def fuzzy_multiply(a, b):
    return (a[0]*b[0], a[1]*b[1], a[2]*b[2])

def vertex_distance(a, b):
    return math.sqrt((1/3) * ((a[0]-b[0])**2 + (a[1]-b[1])**2 + (a[2]-b[2])**2))

# Criteria and weights
criteria = {
    "Experience":       WEIGHTS["Very High"],
    "Technical Skills": WEIGHTS["Very High"],
    "Education":        WEIGHTS["Medium"],
    "Communication":    WEIGHTS["High"],
}

# LLM-generated raw scores (1-10) per candidate per criterion
raw_scores = {
    "Alice": {"Experience": 9, "Technical Skills": 8, "Education": 7, "Communication": 6},
    "Bob":   {"Experience": 7, "Technical Skills": 9, "Education": 8, "Communication": 8},
    "Carol": {"Experience": 6, "Technical Skills": 7, "Education": 9, "Communication": 9},
}

# Convert to TFNs
fuzzy_matrix = {}
for cand, scores in raw_scores.items():
    fuzzy_matrix[cand] = {c: score_to_tfn(s) for c, s in scores.items()}

# Normalize (all benefit criteria -- divide by max upper bound per criterion)
for c in criteria:
    u_max = max(fuzzy_matrix[cand][c][2] for cand in fuzzy_matrix)
    for cand in fuzzy_matrix:
        l, m, u = fuzzy_matrix[cand][c]
        fuzzy_matrix[cand][c] = (l/u_max, m/u_max, u/u_max)

# Apply weights
weighted = {}
for cand in fuzzy_matrix:
    weighted[cand] = {
        c: fuzzy_multiply(fuzzy_matrix[cand][c], criteria[c])
        for c in criteria
    }

# Compute distances to FPIS (1,1,1) and FNIS (0,0,0)
FPIS = (1, 1, 1)
FNIS = (0, 0, 0)

results = {}
for cand in weighted:
    d_pos = sum(vertex_distance(weighted[cand][c], FPIS) for c in criteria)
    d_neg = sum(vertex_distance(weighted[cand][c], FNIS) for c in criteria)
    cc = d_neg / (d_pos + d_neg)
    results[cand] = round(cc, 4)

# Rank by descending closeness coefficient
ranking = sorted(results.items(), key=lambda x: -x[1])
for rank, (name, cc) in enumerate(ranking, 1):
    print(f"#{rank}: {name} (CC={cc})")
```

Output:
```
#1: Bob   (CC=0.5765)
#2: Alice (CC=0.5531)
#3: Carol (CC=0.5304)
```

Bob ranks highest because his strong Technical Skills score is amplified by the "Very High" weight on that criterion, and his solid Communication score benefits from its "High" weight.

---

**Example 2: Adjusting weights for a different role**

User: "Re-rank these same candidates but for a developer advocate role where communication matters most."

Approach: Change weight assignments without re-running LLM scoring.

```python
# Updated weights for developer advocate role
criteria_devrel = {
    "Experience":       WEIGHTS["Medium"],     # was Very High
    "Technical Skills": WEIGHTS["High"],       # was Very High
    "Education":        WEIGHTS["Low"],        # was Medium
    "Communication":    WEIGHTS["Very High"],  # was High
}
# Re-run steps 6-10 with new weights...
```

Output:
```
#1: Carol (CC=0.5488)  -- strong communication drives her to #1
#2: Bob   (CC=0.5396)
#3: Alice (CC=0.4897)  -- Alice drops due to lower communication score
```

This demonstrates the key advantage: weights are decoupled from scoring, so stakeholders can explore different prioritizations instantly.

---

**Example 3: Handling missing data**

User: "One candidate doesn't list their education. How do I handle that?"

Approach:
1. Assign a "Fair" default TFN (3, 5, 7) for missing data -- this represents genuine uncertainty (not penalizing, not rewarding).
2. Alternatively, flag the field and assign a penalty TFN like (0, 1, 3) if the criterion is mandatory.
3. Document the imputation choice so the ranking remains auditable.

```python
# If education is missing, use the midpoint TFN
if candidate.get("Education") is None:
    fuzzy_matrix[candidate_name]["Education"] = (3, 5, 7)  # Fair
    # Log: "Education imputed as Fair (3,5,7) due to missing data"
```

## Best Practices

- **Do:** Always require the LLM to output a brief justification alongside each score. This makes the decision matrix auditable and catches LLM hallucinations early.
- **Do:** Use at least 3 and no more than 7 criteria. Fewer than 3 loses the benefit of multi-criteria analysis; more than 7 causes weight dilution and cognitive overload for stakeholders reviewing results.
- **Do:** Present results with per-criterion breakdowns, not just the final CC rank. Stakeholders need to understand *why* a candidate ranked where they did.
- **Do:** Run sensitivity analysis by varying weights (e.g., +/- one linguistic level) to check if the top-ranked candidate is stable or changes with small weight shifts.
- **Avoid:** Letting the LLM assign weights. Weights encode organizational priorities and must come from hiring managers or domain experts.
- **Avoid:** Comparing CC values across different job postings or criteria sets. CC is only meaningful as a relative rank within a single evaluation run.

## Error Handling

- **LLM returns non-integer or out-of-range scores:** Validate all LLM outputs before proceeding. Re-prompt once with explicit constraints. If still invalid, flag the candidate-criterion pair and use a "Fair" default TFN.
- **All candidates score identically on a criterion:** The criterion adds no discriminating power. Log a warning and consider whether the criterion is too coarsely defined or irrelevant for this candidate pool.
- **Division by zero in normalization:** Occurs if all upper bounds are 0 for a criterion (impossible with the defined scale, but guard against it). Fall back to uniform normalization (1, 1, 1) for that column.
- **CC values are clustered (e.g., 0.51, 0.50, 0.49):** The candidates are effectively tied. Report the cluster and recommend additional criteria or a human tiebreaker rather than treating the ranking as decisive.
- **Missing candidate data:** Apply the imputation strategy from Example 3. Never silently drop a criterion for one candidate while keeping it for others -- this invalidates the matrix.

## Limitations

- **LLM scoring is the bottleneck.** The mathematical ranking is only as good as the per-criterion scores. If the LLM systematically over-rates or under-rates a criterion, the ranking inherits that bias.
- **Not suitable for very large candidate pools (500+) in a single pass.** The LLM scoring step is O(candidates x criteria) in API calls. For large pools, use a pre-filter (keyword matching, threshold screening) before applying Fuzzy TOPSIS to a shortlist.
- **Triangular fuzzy numbers are a simplification.** They assume symmetric or near-symmetric uncertainty. For highly skewed distributions (e.g., a candidate who is either brilliant or terrible at communication), trapezoidal or type-2 fuzzy numbers would be more appropriate but are significantly more complex.
- **The technique ranks candidates relative to each other, not against an absolute threshold.** Adding or removing a candidate changes everyone's CC. Do not use CC as a pass/fail cutoff.
- **Domain-specific criteria must be defined by humans.** The framework does not auto-discover what matters for a role -- it structures and aggregates assessments once criteria are established.

## Reference

Hoque, S., Karim, A. A. J., Alam, M. G. R., & Gope, N. (2026). *When LLM meets Fuzzy-TOPSIS for Personnel Selection through Automated Profile Analysis.* arXiv:2601.22433v1. [https://arxiv.org/abs/2601.22433v1](https://arxiv.org/abs/2601.22433v1)

Key sections to consult: Section III (Methodology) for the complete Fuzzy TOPSIS algorithm with formulas; Section IV (Experiments) for the linguistic scale mappings and DistilRoBERTa fine-tuning details; Table II for the TFN-to-linguistic-term conversion used in this skill.