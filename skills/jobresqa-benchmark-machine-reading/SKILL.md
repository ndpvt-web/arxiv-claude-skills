---
name: "jobresqa-benchmark-machine-reading"
description: "Build and evaluate multilingual machine reading comprehension systems for HR documents (resumes and job descriptions). Implements the JobResQA pipeline: tiered QA generation, cross-document reasoning, placeholder-based bias testing, TEaR translation, and G-Eval LLM-as-judge scoring. Use when: 'evaluate resume parsing accuracy', 'build HR question answering', 'test multilingual resume understanding', 'check bias in resume screening', 'cross-document QA on resumes and JDs', 'benchmark LLM on HR tasks'."
---

This skill enables Claude to build, evaluate, and deploy multilingual machine reading comprehension (MRC) systems for HR documents — resumes and job descriptions — using the methodology from the JobResQA benchmark. It covers the full pipeline: generating tiered QA pairs at three complexity levels, de-identifying and synthesizing realistic HR documents with controllable demographic placeholders, translating benchmarks across languages with human-in-the-loop quality assurance, and evaluating answers using a structured LLM-as-judge rubric. The core insight is that HR document understanding requires cross-document reasoning (linking resume content to JD requirements) and that placeholder-controlled attributes enable systematic bias detection in LLM outputs.

## When to Use

- When the user asks to build a QA system that answers questions about resumes, CVs, or job descriptions
- When evaluating whether an LLM can extract and reason about candidate qualifications against job requirements
- When testing for demographic bias in resume screening or candidate evaluation systems
- When building a multilingual HR document processing pipeline (especially EN, ES, IT, DE, ZH)
- When creating synthetic but realistic HR test data that preserves privacy through de-identification
- When the user needs to benchmark LLM reading comprehension on structured professional documents
- When implementing an LLM-as-judge evaluation framework for open-ended QA tasks
- When translating QA benchmarks across languages while maintaining annotation quality

## Key Technique

**Tiered Cross-Document QA.** JobResQA defines three complexity levels for HR document comprehension. *Basic* questions (26.5%) target single-passage factual extraction from one document — e.g., "What is the candidate's highest degree?" *Intermediate* questions (36.7%) require reasoning across multiple sections of both the resume and JD — e.g., "Does the candidate's work experience align with the required years of experience?" *Complex* questions (36.8%) demand inference, external knowledge integration, and cross-document synthesis — e.g., "How transferable are the candidate's bioinformatics skills to the document management role described in the JD?" This tiered structure exposes where LLMs fail: most models handle Basic extraction but degrade sharply on Complex reasoning, especially in non-English languages.

**Placeholder-Based Bias Control.** Instead of hardcoding demographic details, all personally identifiable information is replaced with typed placeholders: `[NAME]`, `[LASTNAME]`, `[BIRTHPLACE]`, `[NATIONALITY]` (demographic), `[UNIVERSITY]`, `[INSTITUTION]` (educational prestige), `[COMPANY]`, `[POSITION]` (professional), and `[ADDRESS]`, `[CITY]`, `[ZIPCODE]` (socioeconomic). By systematically substituting different values into these slots, you can measure whether an LLM's answer quality or tone changes based on candidate demographics — revealing bias without confounding variables.

**G-Eval LLM-as-Judge Scoring.** Answers are evaluated by a judge LLM on a 0.0-1.0 continuous scale using a structured rubric: the judge checks semantic equivalence between the model's answer and a human reference, verifies grounding in source documents (specific quotes required), confirms language consistency, and penalizes hallucinated content. Scores map to: 0.0-0.3 (factually incorrect), 0.3-0.6 (mostly incorrect), 0.6-0.9 (correct but missing minor details), 0.9-1.0 (fully correct). This replaces brittle exact-match metrics with semantically aware evaluation.

## Step-by-Step Workflow

1. **Collect and pair HR documents.** Gather resume-JD pairs by computing cosine similarity between job titles using a multilingual encoder (e.g., `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`). Select top matches where title similarity > 0.7 and industry sectors align. Manually review the top-k pairs for coherence.

2. **De-identify with typed placeholders.** Run named entity recognition on resumes to extract names, emails, phones, dates, and addresses. Replace each with its typed placeholder: `[NAME]`, `[EMAIL]`, `[PHONE]`, `[DATE_START]`, `[DATE_END]`. For JDs, use rule-based extraction for company names (`[COMPANY]`), products (`[PRODUCT]`), and branch locations (`[CITY]`). Maintain a placeholder registry mapping each placeholder type to its bias category (demographic, socioeconomic, educational, professional).

3. **Generate synthetic documents.** Feed de-identified documents to an LLM (temperature 0.7, top_p 1.0) with instructions to: (a) preserve all placeholders verbatim, (b) rephrase job titles and shift dates while maintaining chronological order, (c) replace gendered language with gender-neutral alternatives, and (d) maintain comparable document length and formatting. Validate outputs by checking placeholder preservation and structural coherence.

4. **Create tiered QA pairs.** For each resume-JD pair, generate questions at three levels. *Basic*: write questions answerable from a single section of one document (e.g., "What programming languages does the candidate list?"). *Intermediate*: write questions requiring cross-section reasoning within or across documents (e.g., "Does the candidate meet the minimum experience requirement?"). *Complex*: write questions requiring inference or external domain knowledge (e.g., "How relevant is the candidate's research background to the role's data analysis requirements?"). Each QA entry must include: `question`, `short_answer` (1-2 sentences), `explanation` (detailed justification with document quotes), and `complexity_level`.

5. **Translate with the TEaR pipeline (if multilingual).** (a) Zero-shot translate at paragraph level, instructing the LLM to preserve all placeholders and formatting. (b) Have native speakers annotate MQM errors on a 10% sample across seven categories (Terminology, Accuracy, Linguistic, Style, Locale, Design, Hallucination) at four severity levels (Critical, Major, Minor, Neutral). (c) Use annotated errors as few-shot examples to estimate errors across the full dataset. (d) Feed estimated errors back to the LLM for refinement. (e) Post-process: normalize placeholder translations via dictionaries, enforce verb tense consistency (present for current roles, past for prior), and apply gender-inclusive notation (e.g., "el/la candidato/a").

6. **Run QA inference.** Prompt the evaluation target LLM as "an expert hiring assistant professional" with the full resume and JD as context. Instruct it to produce both a concise `short_answer` and a detailed `explanation` grounded in source document quotes. Require the response language to match the question language. Run across all QA pairs and all target languages.

7. **Evaluate with G-Eval LLM-as-judge.** For each model answer, prompt the judge LLM (temperature 0.7, top_p 0.9) with: the question, the reference short_answer and explanation, and the model's output. The judge scores on the 0.0-1.0 rubric, checking: factual correctness, semantic equivalence to reference (ignoring stylistic differences), grounding in source documents, absence of hallucination, and language consistency. Aggregate scores by complexity level and language.

8. **Run bias analysis.** Select a set of placeholder value profiles representing different demographic groups (e.g., vary `[NAME]` and `[NATIONALITY]` across genders and ethnicities). Substitute values into the same document templates. Re-run QA inference and scoring. Compare score distributions across profiles using statistical tests (e.g., Kruskal-Wallis) to detect significant performance disparities.

9. **Analyze and report results.** Break down G-Eval scores by: language, complexity level, model size, and model family. Flag languages where mean score drops below 0.5 (the "substantial degradation" threshold observed for German and Chinese in smaller models). Identify complexity levels where models fail systematically. Report bias test results with effect sizes.

## Concrete Examples

**Example 1: Building an HR QA evaluation benchmark**

User: "I have 50 resume-JD pairs. Help me create a QA benchmark to test our LLM's ability to screen candidates."

Approach:
1. Load and parse the 50 pairs, extracting structured sections (education, experience, skills from resumes; requirements, responsibilities from JDs).
2. De-identify using NER + regex: replace all PII with typed placeholders. Store the mapping for potential bias testing later.
3. For each pair, generate ~5-6 QA entries across complexity tiers:
   - Basic: "What is the candidate's most recent job title?" / "Which degree does the JD require?"
   - Intermediate: "Does the candidate's listed Python experience meet the JD's 3-year minimum?"
   - Complex: "Given the candidate's transition from academia to industry, how well do their research publications align with the role's emphasis on applied ML?"
4. Format as TSV with columns: `example_id`, `resume_id`, `resume`, `jd_id`, `jd`, `question`, `short_answer`, `explanation`, `complexity_level`, `language`.

Output:
```tsv
example_id	resume_id	resume	jd_id	jd	question	short_answer	explanation	complexity_level	language
qa_001	r_01	"[NAME] ... 5 years at [COMPANY]..."	jd_01	"Senior Engineer ... requires 3+ years..."	Does the candidate meet the experience requirement?	Yes, the candidate exceeds it by 2 years.	The resume states 5 years at [COMPANY] as a software engineer. The JD requires "3+ years of relevant engineering experience." 5 > 3, so the requirement is met with margin.	intermediate	en
```

**Example 2: Detecting bias in an LLM resume screener**

User: "We're worried our LLM evaluates candidates differently based on names. Can you test for this?"

Approach:
1. Take existing placeholder-based resume-JD-QA triples.
2. Create demographic profiles by substituting placeholder values:
   - Profile A: `[NAME]=James Smith`, `[NATIONALITY]=American`, `[UNIVERSITY]=MIT`
   - Profile B: `[NAME]=Wei Zhang`, `[NATIONALITY]=Chinese`, `[UNIVERSITY]=Tsinghua University`
   - Profile C: `[NAME]=Fatima Al-Hassan`, `[NATIONALITY]=Jordanian`, `[UNIVERSITY]=University of Jordan`
3. Run identical QA inference for all profiles. Collect G-Eval scores.
4. Compare distributions:

Output:
```
Bias Analysis Results (n=581 QA pairs x 3 profiles)
─────────────────────────────────────────────────────
Profile          Mean G-Eval   Std    p-value (vs. A)
Profile A (James)    0.72      0.24   —
Profile B (Wei)      0.71      0.25   0.43 (not sig.)
Profile C (Fatima)   0.68      0.27   0.03 (sig. *)

* Significant at p<0.05. Investigate Complex-level questions
  where Profile C scores 0.61 vs 0.71 for Profile A.
  Potential bias in cross-document inference tasks.
```

**Example 3: Evaluating multilingual HR comprehension**

User: "We need to check if our model handles German and Chinese resumes as well as English ones."

Approach:
1. Load the JobResQA benchmark TSVs for EN, DE, and ZH from `data/jobresqa.{en,de,zh}.tsv`.
2. Run QA inference on all three languages using the same model and prompt template (translated to target language).
3. Score with G-Eval judge, using language-matched reference answers.
4. Break down by complexity level:

Output:
```
Model: Llama 3.3 70B — G-Eval Scores by Language & Complexity
──────────────────────────────────────────────────────────────
             Basic    Intermediate    Complex    Overall
English      0.82       0.74          0.63       0.73
German       0.61       0.48          0.32       0.47
Chinese      0.63       0.50          0.31       0.48

FINDING: German and Chinese degrade 25+ points overall.
Complex cross-document reasoning is the primary failure mode,
dropping to ~0.31 vs 0.63 in English. Recommend: (1) add
German/Chinese HR training data, (2) use retrieval-augmented
approach for cross-document questions, (3) test with larger
multilingual models.
```

## Best Practices

- **Do:** Always include typed placeholders in synthetic HR data rather than fake names — this enables downstream bias testing and prevents accidental PII leakage in benchmarks.
- **Do:** Generate QA pairs at all three complexity levels in roughly the 25/37/37 distribution. Skewing toward Basic questions gives misleadingly high benchmark scores.
- **Do:** Use the structured G-Eval rubric (0.0-1.0 continuous scale with defined bands) rather than binary correct/incorrect — HR QA answers are rarely fully right or wrong.
- **Do:** Require source document quotes in explanations. This grounds answers and makes hallucination detection straightforward during evaluation.
- **Avoid:** Evaluating only in English. The paper shows 25+ point G-Eval drops in German and Chinese even for 70B models. Always test the languages your system will serve.
- **Avoid:** Using exact-match or ROUGE metrics for HR QA evaluation. These metrics fail on paraphrased but semantically correct answers and miss hallucinated but lexically similar responses.
- **Avoid:** Hardcoding demographic details in test data. Once names and nationalities are baked in, you cannot run controlled bias studies without regenerating the entire dataset.

## Error Handling

- **Placeholder corruption during synthesis:** After generating synthetic documents, validate that every placeholder in the original appears in the output. If placeholders are dropped or malformed, reject and regenerate that document.
- **Language mismatch in QA output:** If the model responds in a different language than the question, flag the response and score it 0.0. This is a known failure mode for smaller multilingual models.
- **Judge score instability:** G-Eval with temperature 0.7 introduces variance. Run the judge 3 times per answer and take the mean. If standard deviation across runs exceeds 0.2, flag for manual review.
- **Translation placeholder leakage:** During TEaR translation, LLMs sometimes translate placeholder tokens (e.g., `[NAME]` becomes `[NOMBRE]`). Post-process with a placeholder normalization dictionary mapping translated placeholders back to canonical English forms.
- **Empty or refusal responses:** Some models refuse to answer HR questions citing ethical concerns. Log these as `score=0.0` with a `refusal` flag rather than excluding them, as refusal rate is itself a meaningful metric.

## Limitations

- The benchmark contains 581 QA pairs across 105 document pairs — sufficient for evaluation but not for fine-tuning. Use it as a test set, not a training set.
- Complex questions requiring external domain knowledge (e.g., skill transferability across industries) have inherently subjective ground truth. G-Eval scores in the 0.6-0.9 range may reflect legitimate answer variation rather than errors.
- The five languages covered (EN, ES, IT, DE, ZH) do not include Arabic, Hindi, Japanese, Korean, or other high-demand HR markets. Extending to new languages requires the full TEaR pipeline with native speaker annotators.
- Placeholder-based bias testing controls for name/nationality effects but cannot capture subtler biases encoded in writing style, document formatting, or vocabulary choices.
- The LLM-as-judge approach inherits the judge model's own biases. Cross-validate with human evaluation on a sample (recommended: 10-15% of QA pairs) when deploying in production.

## Reference

**Paper:** [JobResQA: A Benchmark for LLM Machine Reading Comprehension on Multilingual Resumes and JDs](https://arxiv.org/abs/2601.23183v1) (Carrino et al., 2026). Key sections: Section 3 for the data generation pipeline, Section 4 for TEaR translation methodology, Section 5 for G-Eval evaluation protocol, and Table 5 for cross-lingual performance breakdowns.

**Code & Data:** [github.com/Avature/jobresqa-benchmark](https://github.com/Avature/jobresqa-benchmark) — TSV datasets in `data/`, evaluation scripts in `scripts/run_eval_qa.py`, and prompt templates in `resources/prompts/`.