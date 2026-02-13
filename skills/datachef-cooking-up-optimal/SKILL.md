---
name: "datachef-cooking-up-optimal"
description: "Automate data recipe generation for LLM fine-tuning and adaptation. Generates executable data processing pipelines (filtering, synthesis, mixing, augmentation) that transform raw data sources into optimized training corpora for a target task. Trigger phrases: 'create a data recipe', 'optimize training data mix', 'build a data pipeline for fine-tuning', 'curate training data for task X', 'generate a data processing pipeline', 'mix datasets for domain adaptation'."
---

# DataChef: Automated Data Recipe Generation for LLM Adaptation

This skill enables Claude to generate **end-to-end data recipes** — executable Python pipelines that transform raw data sources into optimized training corpora for fine-tuning LLMs on specific target tasks. Based on the DataChef framework, the approach treats data curation as a structured optimization problem: given a target benchmark and a pool of available data sources, produce a complete pipeline of filtering, synthesis, mixing, and augmentation steps that maximizes downstream task performance. Instead of manual trial-and-error data curation, this skill applies principled recipe design with proxy-based evaluation to iterate toward high-quality training data.

## When to Use

- When the user wants to fine-tune or adapt an LLM to a specific domain (math, code, finance, medical, etc.) and needs to decide which datasets to use and how to process them
- When the user has multiple raw data sources and needs a pipeline to filter, clean, combine, and format them into a training corpus
- When the user asks "what data should I train on?" or "how do I mix these datasets for best results?"
- When the user needs to synthesize additional training examples to fill gaps in their existing data
- When the user wants to automate the data preparation step of an LLM training workflow
- When the user is iterating on data quality and needs a systematic approach rather than ad-hoc filtering

## Key Technique

**Data recipes as code.** A data recipe is a tuple `r = (g, d)` where `g` is an executable Python pipeline and `d` is the resulting training dataset. The pipeline composes operations from a pool: **filtering** (quality-based selection, deduplication, keyword extraction), **synthesis** (LLM-generated augmentation, format conversion), **mixing** (weighted combination of multiple sources), and **selection** (relevance-based subsetting). The key insight is that the entire recipe — not just individual steps — must be optimized holistically, because interactions between processing steps matter.

**Proxy reward for fast iteration.** Evaluating a recipe normally requires full fine-tuning, which is prohibitively expensive. DataChef uses a **Data Verifier** — a rubric-based evaluator that scores sampled instances from the generated dataset on a scale: Invalid/Incorrect (0), Task Mismatch (0.4), Pass (1.0). This proxy score correlates with downstream performance and enables evaluating dozens of candidate recipes without training a model for each one. Penalties are applied for execution failures (empty dataset) and format violations.

**Iterative refinement via RL.** The framework uses Group Relative Policy Optimization (GRPO) to learn recipe generation as a policy. Multiple candidate recipes are sampled per task, scored by the proxy verifier, and the policy is updated to favor higher-scoring recipes. This replaces the expert-in-the-loop iteration cycle with an automated search over recipe space. In practice, generating 32 candidates and selecting the best by proxy score matches or exceeds human-expert recipes.

## Step-by-Step Workflow

1. **Define the target task precisely.** Specify the downstream benchmark or evaluation criteria — e.g., "math reasoning evaluated on AIME-style problems" or "financial QA evaluated on OpenFinData." Include example inputs/outputs of the target format so the recipe can align data to it.

2. **Inventory available data sources.** List all accessible datasets with metadata: name, domain, size, format (JSON lines, CSV, parquet), and a brief description of content. Categorize each as raw-web, curated-academic, synthetic, or domain-specific.

3. **Analyze source-task relevance.** For each data source, estimate relevance to the target task. Use keyword overlap, domain match, and format compatibility. Rank sources by expected utility. Discard clearly irrelevant sources early.

4. **Design the filtering stage.** Write Python functions that remove low-quality instances: deduplication (MinHash or exact match), length filtering (remove trivially short or excessively long), quality heuristics (perplexity threshold, language detection), and content filtering (remove toxic/irrelevant text).

5. **Design the synthesis stage.** Identify gaps between available data and target task requirements. Write prompts and scripts that use an LLM to: (a) reformat existing data into the target task format, (b) generate new examples for underrepresented categories, (c) augment with chain-of-thought reasoning or step-by-step solutions where needed.

6. **Design the mixing stage.** Determine proportions for combining filtered/synthesized sources. Use a weighted sampling strategy — oversample high-relevance sources, undersample generic ones. Specify total target corpus size based on compute budget (typically 10K-500K examples for adaptation).

7. **Implement the recipe as an executable pipeline.** Write a single Python script that chains all stages: load sources, filter, synthesize, mix, validate output format, and write the final training dataset. Include error handling for missing files, format mismatches, and empty intermediate results.

8. **Evaluate with proxy scoring.** Sample 50-100 instances from the output dataset. Score each against the target task rubric: Does it match the expected format? Is the content correct? Is it relevant to the target domain? Compute the average pass rate.

9. **Iterate: generate multiple recipe variants.** Vary the recipe parameters — different filtering thresholds, different synthesis prompts, different mixing ratios — to produce 4-8 candidate recipes. Score each with the proxy evaluator and select the top performer.

10. **Validate end-to-end.** Fine-tune on the selected recipe's output dataset and evaluate on the target benchmark. Compare against a naive baseline (unprocessed data concatenation) to confirm the recipe adds value.

## Concrete Examples

**Example 1: Math Domain Adaptation**

User: "I want to fine-tune Llama-3-8B for math competition problems. I have access to MATH dataset, GSM8K, and some scraped competition problems from AoPS. How should I prepare the training data?"

Approach:
1. Define target: AMC/AIME-style problems requiring multi-step reasoning, evaluated by exact answer match.
2. Inventory sources: MATH (12.5K problems, diverse difficulty), GSM8K (8.5K grade-school problems), AoPS scrape (~20K raw HTML posts).
3. Relevance: MATH = high, AoPS = high but noisy, GSM8K = moderate (too easy but useful for format).
4. Filter: Deduplicate across all three sources. For AoPS: extract problem-solution pairs from HTML, discard posts without clear answers, filter by math-competition keywords.
5. Synthesize: Use an LLM to rewrite GSM8K solutions with step-by-step reasoning chains. Generate harder variants of MATH problems by increasing parameter values.
6. Mix: 50% MATH (filtered), 30% AoPS (cleaned), 15% synthesized hard variants, 5% GSM8K (reformatted).
7. Pipeline output: ~25K examples in chat format with problem/solution pairs.

Output recipe (pipeline sketch):
```python
import json
from datasets import load_dataset, concatenate_datasets

def load_and_filter_math():
    ds = load_dataset("hendrycks/math", split="train")
    # Keep levels 3-5 for competition relevance
    return ds.filter(lambda x: x["level"] in ["Level 3", "Level 4", "Level 5"])

def clean_aops(raw_path):
    with open(raw_path) as f:
        posts = json.load(f)
    cleaned = []
    for post in posts:
        problem, solution = extract_problem_solution(post["html"])
        if problem and solution and len(solution) > 50:
            cleaned.append({"problem": problem, "solution": solution})
    return deduplicate(cleaned, threshold=0.85)

def synthesize_hard_variants(math_ds, llm_client, n=3000):
    prompts = [format_augmentation_prompt(ex) for ex in math_ds.select(range(1000))]
    return [llm_client.generate(p) for p in prompts]

def build_recipe():
    math_filtered = load_and_filter_math()          # ~6K
    aops_cleaned = clean_aops("aops_scrape.json")    # ~8K
    variants = synthesize_hard_variants(math_filtered, client)  # ~3K
    gsm_reformatted = reformat_gsm8k()               # ~1.5K

    combined = weighted_sample([
        (math_filtered, 0.50),
        (aops_cleaned, 0.30),
        (variants, 0.15),
        (gsm_reformatted, 0.05),
    ], total=25000)
    return format_as_chat(combined)
```

**Example 2: Financial QA Adaptation**

User: "I need to adapt a base model for financial question answering. I have SEC filings, financial news articles, and the FiQA dataset. Create a data recipe."

Approach:
1. Define target: financial QA — given a question about finance, produce an accurate grounded answer.
2. Inventory: SEC filings (raw text, ~100K documents), financial news (50K articles), FiQA (6.6K QA pairs).
3. Relevance: FiQA = high (direct QA), SEC filings = medium (need QA extraction), news = medium (need reformatting).
4. Filter SEC filings: Extract tables and key sections (risk factors, MD&A). Remove boilerplate. Filter news by financial-keyword density.
5. Synthesize: Use an LLM to generate QA pairs from SEC filing paragraphs and news articles — "Given this passage, generate a question and answer."
6. Mix: 40% FiQA original, 35% SEC-derived QA, 25% news-derived QA.

Output recipe (pipeline sketch):
```python
def build_finance_recipe():
    fiqa = load_dataset("fiqa", split="train")

    sec_passages = extract_sec_sections("sec_filings/",
                                         sections=["risk_factors", "mda"])
    sec_passages = filter_by_length(sec_passages, min_len=100, max_len=2000)
    sec_qa = synthesize_qa_pairs(sec_passages, client,
                                  prompt="Generate a factual QA pair from this SEC filing excerpt.")
    sec_qa = filter_by_quality(sec_qa, min_answer_len=20)

    news_passages = load_news("financial_news.jsonl")
    news_passages = filter_by_keywords(news_passages,
                                        keywords=["revenue", "earnings", "market", "stock"])
    news_qa = synthesize_qa_pairs(news_passages, client,
                                   prompt="Generate a financial analysis question from this article.")

    combined = weighted_sample([
        (fiqa, 0.40),
        (sec_qa, 0.35),
        (news_qa, 0.25),
    ], total=15000)
    return format_as_instruction(combined)
```

**Example 3: Quick Recipe Evaluation**

User: "I already built a training dataset of 50K coding examples. How do I evaluate whether it's good before spending GPU hours on training?"

Approach:
1. Define target benchmark (e.g., HumanEval, LiveCodeBench).
2. Sample 100 instances from the dataset.
3. Apply rubric-based proxy scoring.

Output evaluation script:
```python
def proxy_evaluate(dataset_path, target_task="code_generation", n_samples=100):
    dataset = load_jsonl(dataset_path)
    samples = random.sample(dataset, min(n_samples, len(dataset)))

    scores = []
    for instance in samples:
        # Rubric: format correctness, task relevance, content quality
        fmt_ok = check_format(instance, expected_keys=["instruction", "response"])
        relevant = check_relevance(instance, target_task, client)  # LLM judge
        quality = check_quality(instance, client)  # LLM judge: correct, coherent

        if not fmt_ok:
            scores.append(0.0)
        elif not relevant:
            scores.append(0.4)
        else:
            scores.append(1.0 if quality else 0.0)

    avg_score = sum(scores) / len(scores)
    print(f"Proxy score: {avg_score:.2f} (target: >0.75)")
    print(f"Format pass: {sum(1 for s in scores if s > 0)/len(scores):.0%}")
    print(f"Full pass: {sum(1 for s in scores if s == 1.0)/len(scores):.0%}")
    return avg_score
```

## Best Practices

- **Do:** Start with the target task format and work backward — define what a perfect training example looks like before designing the pipeline.
- **Do:** Generate multiple recipe variants (at least 4-8) with different mixing ratios and filtering thresholds, then select the best by proxy score. The best-of-N strategy is central to DataChef's success.
- **Do:** Apply quality filtering aggressively early in the pipeline. A smaller, cleaner dataset almost always outperforms a larger, noisier one for domain adaptation.
- **Do:** Use LLM-based synthesis to convert existing data into the target format rather than collecting new data from scratch — reformatting is cheaper and often more effective.
- **Avoid:** Skipping the proxy evaluation step and jumping straight to full fine-tuning. Proxy scoring catches bad recipes (empty outputs, format mismatches, irrelevant data) at near-zero cost.
- **Avoid:** Mixing too many heterogeneous sources without explicit relevance filtering. Irrelevant data dilutes the signal and hurts adaptation performance — more data is not always better.

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| Empty dataset after filtering | Pipeline produces 0 or very few examples | Relax filtering thresholds; check that filters aren't compounding to exclude everything |
| Format mismatch | Proxy scorer flags high "invalid format" rate | Add a format validation step after each pipeline stage; align output schema to training framework expectations |
| Low relevance score | Proxy reports >30% "task mismatch" | Re-examine source selection; add keyword or embedding-based relevance filtering before synthesis |
| Synthesis producing hallucinated content | Generated QA pairs contain fabricated facts | Ground synthesis prompts with source passages; add a verification step that checks answers against source text |
| Deduplication too aggressive | Useful near-duplicates removed (e.g., same problem, different solution paths) | Use higher similarity threshold (0.9+) or deduplicate only on input, not on input-output pairs |
| Recipe execution fails | Python errors in the pipeline script | Test each stage independently before composing; add try/except with logging around each transform |

## Limitations

- **No substitute for actual training evaluation.** Proxy scores correlate with downstream performance but are not perfect predictors. For high-stakes deployments, always validate the final recipe with a real training run.
- **Depends on LLM access for synthesis.** The synthesis stage requires API calls to a capable LLM, which adds cost and latency. For very large datasets, batch processing or local models may be necessary.
- **Recipe quality bounded by source pool.** If no relevant data sources exist for the target domain, no amount of filtering or mixing will produce good training data. The approach works best when there are several partially-relevant sources to combine.
- **Proxy evaluation requires target task examples.** You need at least a few examples of the target format to build the scoring rubric. Cold-start on entirely novel tasks is harder.
- **Not designed for pretraining.** This approach targets task-specific adaptation (fine-tuning on 10K-500K examples), not large-scale pretraining data curation where different dynamics apply.

## Reference

**Paper:** [DataChef: Cooking Up Optimal Data Recipes for LLM Adaptation via Reinforcement Learning](https://arxiv.org/abs/2602.11089v1) — Chen et al., 2026. Look for: Section 3 (recipe formulation and Data Verifier rubric), Section 4 (GRPO training with proxy rewards), and Appendix C (case study recipes for math and finance domains).