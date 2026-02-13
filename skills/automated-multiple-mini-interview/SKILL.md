---
name: "automated-multiple-mini-interview"
description: "Multi-agent framework for scoring subjective, open-ended responses (interviews, essays, reflections) using transcript refinement + criterion-specific parallel scoring with calibrated few-shot examples. Use when: 'score these interview responses', 'evaluate candidate answers', 'grade these essays on a rubric', 'assess soft skills from text', 'build an automated scoring pipeline', 'rate open-ended responses against criteria'."
---

# Automated Multi-Agent Scoring for Subjective Responses

This skill enables Claude to evaluate subjective, open-ended text responses (interview transcripts, essays, reflective writing, short answers) using a multi-agent decomposition strategy from Huynh et al. (2026). Instead of asking a single LLM pass to score everything at once, the technique splits evaluation into (1) a preprocessing/refinement stage that cleans and distills the response, then (2) parallel criterion-specific scoring agents, each calibrated with three strategically chosen examples (low/medium/high). This structured decomposition nearly doubles scoring agreement with human experts compared to fine-tuned baselines (QWK 0.62 vs 0.32) and generalizes across domains without retraining.

## When to Use

- When the user wants to score or grade open-ended text responses against a rubric with multiple criteria (empathy, communication, ethical reasoning, argumentation, etc.)
- When building an automated evaluation pipeline for interviews, admissions essays, reflective journals, or short-answer assessments
- When a single-pass LLM scoring attempt produces inconsistent or criterion-conflated results
- When the user has a rubric with distinct scoring dimensions and wants per-criterion reliability
- When evaluating transcripts from spoken interviews (including noisy ASR output)
- When the user wants to replicate human panel scoring at scale without fine-tuning a model

## Key Technique

**Why single-pass scoring fails on subjective tasks.** When an LLM scores a response on multiple criteria simultaneously, cross-criterion interference occurs: the model's judgment on one dimension (e.g., empathy) bleeds into its score for another (e.g., ethical reasoning). Fine-tuning on rationale-based methods also struggles because subjective qualities like empathy are expressed through implicit narrative signals rather than explicit keywords. The model learns surface patterns rather than the underlying construct.

**The multi-agent decomposition.** The framework uses a two-stage pipeline. First, a Refiner agent preprocesses the raw text: correcting transcription errors, stripping filler words and conversational preambles, removing redundancies, and distilling the core substantive content without adding anything new. Second, the refined text is distributed to N independent Scorer agents, one per rubric criterion. Each scorer receives only its own criterion definition, the scoring scale descriptors, and three calibrated exemplars selected at the 5th, 50th, and 95th percentiles for that specific criterion. This isolation prevents cross-contamination between criteria.

**Why 3-shot calibration matters.** The paper found that exactly three examples (low/medium/high) optimally anchor the scoring scale. Fewer examples leave the scale ambiguous; more examples (4+) cause recency bias where the model over-indexes on the last example's score. The exemplars must be criterion-specific -- generic "good/bad" examples degrade performance significantly. This makes calibration data selection the single most important design decision in the pipeline.

## Step-by-Step Workflow

1. **Define the rubric.** For each criterion to be scored, write a clear criterion definition and a scale descriptor (e.g., 1-7 Likert, 1-5 holistic, or letter grades). Each criterion must be independently scorable -- if two criteria always co-vary, merge them.

2. **Select calibration exemplars.** For each criterion, choose exactly three example responses that represent low (roughly 5th percentile), medium (50th percentile), and high (95th percentile) performance on that specific criterion. Label each with its score. These must be real responses scored by humans, not synthetic examples.

3. **Build the Refiner prompt.** Instruct the refiner to: (a) correct minor transcription or grammatical errors, (b) remove filler words, self-introductions, and question repetitions, (c) eliminate redundant statements, (d) preserve all substantive content without adding, reordering, or interpreting. Output a concise cleaned version.

4. **Run the Refiner agent.** Pass the raw response through the refiner and capture the cleaned transcript. If the input is already clean prose (e.g., a written essay), this step can be simplified to just removing redundancies.

5. **Build criterion-specific Scorer prompts.** For each criterion, construct a prompt containing: (a) a system role establishing the evaluator persona, (b) the criterion definition and scale descriptors, (c) the three calibrated exemplars with their scores, (d) the cleaned candidate response, (e) an instruction to output only the numeric score for this criterion.

6. **Run all Scorer agents in parallel.** Execute each criterion scorer independently. The key principle is isolation -- no scorer sees other criteria or other scorers' outputs. This prevents anchoring and cross-criterion interference.

7. **Collect and aggregate scores.** Gather the per-criterion scores into a structured output (JSON object, table row, etc.). Compute any composite scores required by the rubric (sum, weighted average, etc.).

8. **Validate output ranges.** Confirm each score falls within the valid scale range. If a scorer returns an out-of-range value or non-numeric output, re-run that specific scorer with a stricter output format instruction.

9. **Optionally generate rationales.** If explanations are needed, run a separate rationale-generation pass after scoring is complete. Do NOT ask scorers to generate rationales during scoring -- the paper found this degrades scoring accuracy.

10. **Report results.** Present per-criterion scores, composite scores, and (if requested) rationales in a structured format. Flag any criteria where the score is at a scale boundary (1 or max), as these may warrant human review.

## Concrete Examples

**Example 1: Scoring Interview Responses for Empathy and Communication**

```
User: I have 50 interview transcripts where candidates respond to the scenario
"Your close friend tells you they just received a serious medical diagnosis."
Score each on empathy (1-7) and communication clarity (1-7). Here are my
calibration examples for each criterion.

Approach:
1. Define two criteria with scale descriptors:
   - Empathy: 1="No acknowledgment of emotion" ... 7="Deep, specific emotional
     attunement with validation"
   - Communication: 1="Incoherent or off-topic" ... 7="Clear, structured,
     purposeful response"

2. For each criterion, accept the user's 3 calibration examples (low/med/high).

3. Build refiner prompt:
   "You are a transcript preprocessor. Clean this interview transcript by:
   - Correcting transcription errors
   - Removing filler words (um, uh, like, you know)
   - Removing interviewer questions and candidate self-introductions
   - Eliminating repeated points
   Output ONLY the cleaned candidate response. Do not add interpretation."

4. For each of 50 transcripts:
   a. Run Refiner → cleaned text
   b. Run Empathy Scorer (criterion def + 3 examples + cleaned text) → score
   c. Run Communication Scorer (criterion def + 3 examples + cleaned text) → score
   d. Collect scores into results table

Output:
| Candidate | Empathy (1-7) | Communication (1-7) | Composite |
|-----------|---------------|---------------------|-----------|
| C001      | 5             | 6                   | 5.5       |
| C002      | 3             | 4                   | 3.5       |
| C003      | 6             | 5                   | 5.5       |
| ...       | ...           | ...                 | ...       |
```

**Example 2: Essay Scoring on an Existing Rubric (ASAP-style)**

```
User: Grade these student essays on a 0-3 scale for "content accuracy"
and "argument quality". I have example essays scored by teachers.

Approach:
1. Since essays are written text, simplify the Refiner to just remove
   redundant sentences and off-topic tangents.

2. Build Content Accuracy Scorer prompt:
   "You are an expert essay evaluator. Score ONLY content accuracy.
   Scale: 0=Factually wrong, 1=Some correct facts but major gaps,
   2=Mostly accurate with minor errors, 3=Fully accurate and complete.

   Example A (Score: 1): [low-percentile essay text]
   Example B (Score: 2): [mid-percentile essay text]
   Example C (Score: 3): [high-percentile essay text]

   Student essay: [cleaned essay]
   Output ONLY the integer score (0-3):"

3. Build Argument Quality Scorer with its own 3 examples and criterion def.

4. Run both scorers per essay, aggregate.

Output:
{"essay_id": "E042", "content_accuracy": 2, "argument_quality": 3, "total": 5}
```

**Example 3: Building a Reusable Scoring Pipeline in Code**

```
User: Help me build a Python pipeline that implements this multi-agent
scoring framework for our admissions process.

Approach:
1. Create a config schema for rubric definitions:
   {
     "criteria": [
       {
         "name": "empathy",
         "definition": "Ability to recognize and respond to emotional cues...",
         "scale": {"min": 1, "max": 7},
         "exemplars": {
           "low":  {"text": "...", "score": 2},
           "mid":  {"text": "...", "score": 4},
           "high": {"text": "...", "score": 6}
         }
       }
     ]
   }

2. Implement the Refiner as a function calling the LLM with the cleaning prompt.

3. Implement each Scorer as an async function that takes (criterion_config,
   cleaned_text) and returns a score. Use asyncio.gather() to run all
   criteria scorers in parallel per candidate.

4. Add output validation: retry if score is out of range or non-numeric.

5. Aggregate into a DataFrame with per-criterion and composite columns.

Output: A Python module with classes RefinerAgent, ScorerAgent,
and ScoringPipeline, plus a CLI that takes a rubric config JSON
and a folder of transcripts, and outputs a scored CSV.
```

## Best Practices

- **Do:** Select calibration exemplars from real human-scored data at distinct percentiles (5th, 50th, 95th) for each criterion independently. The exemplars are the most impactful component of the framework.
- **Do:** Keep each Scorer agent strictly isolated to a single criterion. Never include multiple criteria in one prompt -- this is the core architectural principle that prevents cross-contamination.
- **Do:** Use exactly 3 few-shot examples per criterion. The paper demonstrates that 4+ examples cause recency bias, and fewer than 3 leave the scale poorly anchored.
- **Do:** Strip rationale generation from the scoring pass. If you need explanations, generate them in a separate post-hoc step after all scores are finalized.
- **Avoid:** Using generic "good/bad" examples across all criteria. Each criterion needs its own exemplars because a response can score high on empathy but low on structure.
- **Avoid:** Asking the model to score and explain simultaneously. The paper shows this degrades scoring accuracy as the model optimizes for plausible-sounding rationales rather than calibrated scores.

## Error Handling

- **Non-numeric output:** If a scorer returns text instead of a number, re-prompt with a stricter format: "Reply with ONLY a single integer between {min} and {max}. No other text."
- **Out-of-range scores:** Clamp or re-run. Clamping is acceptable for minor violations (e.g., 8 on a 1-7 scale); re-run for major violations (e.g., 0 on a 1-7 scale).
- **Refiner adds content:** If the refiner introduces interpretations or new material not in the original, re-run with an explicit negative instruction: "Do NOT add any content, interpretation, or inference. Only remove and clean."
- **Low agreement on specific criteria:** If one criterion consistently produces poor inter-rater agreement, the criterion definition is likely ambiguous. Tighten the scale descriptors and select more discriminating exemplars.
- **Noisy transcripts:** For heavily degraded ASR output, consider a two-pass refinement: first a literal error-correction pass, then the content-distillation pass.

## Limitations

- **Requires human-scored calibration data.** The framework cannot bootstrap from zero -- you need at least 3 human-scored examples per criterion to function. Without real exemplars, the scale anchoring fails.
- **Ceiling near human expert agreement.** The method approaches but does not exceed human inter-rater reliability. For criteria where even human experts disagree substantially, automated scoring will similarly struggle.
- **Criterion definitions must be precise.** Vague criteria like "overall quality" produce poor results. The framework works best when each criterion targets a specific, definable construct.
- **Scale sensitivity.** The 3-shot calibration works well for 5-7 point scales. Very fine-grained scales (e.g., 1-100) may need more exemplars or a different approach, violating the 3-shot optimum.
- **Cost scales linearly with criteria count.** Each criterion requires a separate LLM call per response, so scoring 10 criteria across 1000 responses means 10,000+ API calls plus 1,000 refiner calls.
- **Not suitable for factual/objective scoring.** If a question has a clear correct answer, use direct evaluation. This framework is designed for subjective, construct-dependent assessment.

## Reference

Huynh, R., Guerin, F., & Callwood, A. (2026). *Automated Multiple Mini Interview (MMI) Scoring.* arXiv:2602.02360v1. [https://arxiv.org/abs/2602.02360v1](https://arxiv.org/abs/2602.02360v1)

Key insight: Structured prompt decomposition (refine-then-score with isolated criterion agents and percentile-calibrated 3-shot examples) nearly doubles scoring agreement over fine-tuned models on subjective assessment tasks, and generalizes across domains without retraining.