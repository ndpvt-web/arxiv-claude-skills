---
name: "llm-autodp-automatic-data-processing"
description: "Automatically generate and optimize data processing pipelines for LLM fine-tuning datasets using an agent-driven iterative strategy search. Trigger phrases: 'clean my training data', 'optimize dataset for fine-tuning', 'automate data processing pipeline', 'improve data quality for training', 'build a data cleaning strategy', 'process noisy fine-tuning data'"
---

# LLM-AutoDP: Automatic Data Processing for Fine-Tuning

This skill enables Claude to act as an autonomous data processing agent that generates, evaluates, and iteratively refines data cleaning and optimization pipelines for LLM fine-tuning datasets. Rather than applying a fixed cleaning recipe, Claude designs multi-step processing strategies composed of four operator families (Cleaning, Optimization, Generation, Selection), evaluates each strategy by its downstream fine-tuning impact, and converges on the best pipeline through feedback-driven refinement -- following the LLM-AutoDP framework from VLDB 2026.

## When to Use

- When the user has a QA, instruction-tuning, or chat dataset and wants to improve model performance by cleaning it before fine-tuning
- When the user asks to remove low-quality, duplicated, or noisy samples from a training corpus
- When the user needs a systematic approach to dataset curation rather than ad-hoc filtering
- When the user wants to build a reusable data processing pipeline for domain-specific data (medical, legal, financial)
- When the user wants to compare multiple cleaning strategies and pick the best one empirically
- When privacy constraints prevent manual inspection of raw data -- the pipeline must operate without a human reading individual samples

## Key Technique

LLM-AutoDP treats data processing as a combinatorial strategy search problem. The search space is defined by four operator teams: **Cleaning** (deduplication, HTML removal, special-character filtering, length constraints, n-gram repetition filtering), **Optimization** (rewriting questions, answers, or full QA pairs for fluency and accuracy), **Generation** (synthesizing missing questions, answers, or new QA pairs), and **Selection** (scoring samples via gradient information or model confidence and keeping only high-quality ones). A strategy is a specific ordered composition of 1-4 teams -- yielding 64 possible selection spaces. The agent generates K diverse candidate strategies per round, each is applied to the data, a model is fine-tuned, and a normalized improvement score `s = r_strategy - r_baseline` quantifies impact. The agent then receives all scores as in-context feedback and proposes refined strategies for the next round. Convergence typically occurs in 4-5 rounds.

Three acceleration techniques make this tractable on large datasets. **Distribution-Preserving Sampling (DPS)** reduces data to ~20% while preserving the ratio of clean-to-noisy samples by embedding-based iterative selection within each subset. **Processing Target Selection (PTS)** trains a binary classifier on a labeled subset to flag low-quality samples, then applies expensive processing only to the noisy subset. **Cache-and-Reuse (CRM)** stores intermediate outputs of each strategy; when a new strategy shares a prefix with a prior one, the cached result is loaded and only the suffix operations are executed, cutting redundant compute dramatically.

## Step-by-Step Workflow

1. **Profile the dataset.** Load the dataset and compute summary statistics: total samples, field names, average/min/max token lengths, language, duplicate rate (exact and near-duplicate via MinHash), special-character ratios, and class or topic distribution. Output a concise data profile report.

2. **Define the operator inventory.** Map available processing operations to the four teams:
   - *Cleaning*: dedup (MinHash LSH), HTML/tag stripping, special-char ratio filter (threshold configurable), token-length bounds, n-gram repetition filter
   - *Optimization*: rewrite question for clarity, rewrite answer for accuracy/fluency, rewrite full QA pair
   - *Generation*: generate missing question from answer, generate missing answer from question, generate new QA pairs from context
   - *Selection*: score samples by perplexity/gradient signal, keep top-k%

3. **Apply Distribution-Preserving Sampling.** Embed all samples using a sentence embedding model. Train or apply a binary quality classifier (PTS) to split data into clean/noisy subsets. Sample from each subset independently using cosine-similarity-maximizing iterative selection to build a representative ~20% subsample that preserves the clean/noisy ratio.

4. **Generate initial candidate strategies.** Propose 4 diverse strategies, each a different ordered composition of operator teams. Prefer smaller team compositions (1-2 teams) in round 1 to isolate the effect of each operator family. Format each strategy as an explicit ordered pipeline, e.g., `[Cleaning → Selection]` or `[Optimization → Cleaning → Selection]`.

5. **Execute each strategy on the subsample.** Apply the operators in the specified order. Use the Cache-and-Reuse mechanism: before running a strategy, check if a previously executed strategy shares a prefix; if so, load the cached intermediate dataset and execute only the remaining suffix operators.

6. **Evaluate strategies via downstream proxy.** For each processed subsample, fine-tune a small proxy model (or compute a proxy metric like perplexity drop on a held-out validation set). Compute the normalized improvement score `s_k = r_k - r_baseline` for each strategy.

7. **Refine strategies with feedback.** Present all strategy-score pairs as in-context feedback. Analyze which operator teams and orderings contributed most. Generate the next round of K candidate strategies by: combining top-performing teams, reordering operators, adjusting operator parameters (e.g., stricter dedup threshold), or adding/removing teams.

8. **Iterate until convergence.** Repeat steps 5-7 for up to 5 rounds. Convergence is reached when the best score improvement between consecutive rounds falls below a threshold (e.g., <1% relative gain). Return the best-performing strategy.

9. **Apply the winning strategy to the full dataset.** Run the final pipeline on 100% of the data (PTS ensures only noisy samples undergo expensive processing). Output the processed dataset and a summary report of operations applied, samples removed/modified, and expected quality improvement.

10. **Generate the reusable pipeline specification.** Export the final strategy as a declarative config (JSON or YAML) listing each operator, its parameters, and execution order, so the pipeline can be rerun on future data without repeating the search.

## Concrete Examples

**Example 1: Cleaning a Medical QA Dataset for Fine-Tuning**

User: "I have 50,000 medical QA pairs scraped from forums. Many are duplicates, some have HTML artifacts, and answer quality varies wildly. Help me build a cleaning pipeline before fine-tuning Llama on it."

Approach:
1. Profile the dataset: find 12% exact duplicates, 8% near-duplicates, 15% of answers under 10 tokens, 5% contain HTML tags.
2. Define operators: MinHash dedup (threshold 0.8), HTML stripping, token-length filter (min 20 tokens for answers), QA-pair optimization (rewrite low-quality answers), perplexity-based selection (keep top 80%).
3. Apply DPS to create a 10,000-sample subsample preserving noisy/clean ratio.
4. Round 1 strategies:
   - S1: `[Cleaning]` -- dedup + HTML strip + length filter
   - S2: `[Cleaning → Optimization]` -- clean then rewrite poor answers
   - S3: `[Optimization → Selection]` -- rewrite then keep best
   - S4: `[Cleaning → Optimization → Selection]` -- full pipeline
5. Evaluate each on proxy fine-tune. Scores: S1=+3.2, S2=+5.8, S3=+4.1, S4=+6.5.
6. Round 2: refine S4 variants (adjust dedup threshold, add n-gram filter, try rewriting questions too). Best score reaches +7.9.
7. Apply winning pipeline to full 50k dataset. Output: 38,200 processed samples.

Output:
```yaml
pipeline:
  name: "medical-qa-v1"
  steps:
    - operator: dedup_minhash
      params: { threshold: 0.75, num_perm: 128 }
    - operator: strip_html
    - operator: filter_token_length
      params: { min_answer_tokens: 20 }
    - operator: filter_ngram_repetition
      params: { n: 5, max_ratio: 0.3 }
    - operator: optimize_qa_pair
      params: { target: "answer", criteria: "medical accuracy and fluency" }
    - operator: select_by_perplexity
      params: { keep_ratio: 0.85 }
  result:
    input_samples: 50000
    output_samples: 38200
    estimated_win_rate_vs_unprocessed: "~82%"
```

**Example 2: Iterative Strategy Search for Legal Instruction Data**

User: "I'm fine-tuning a model on 20,000 legal consultation dialogues. How do I find the best data processing strategy?"

Approach:
1. Profile: multi-turn dialogues, average 450 tokens, 3% duplicates, inconsistent formatting across sources.
2. Apply PTS: train binary classifier on 500 manually labeled samples to separate clean/noisy. Result: 35% flagged as noisy.
3. Subsample via DPS: 4,000 samples (20%), preserving 65/35 clean/noisy split.
4. Run 4-round iterative search:
   - Round 1: Test each operator team in isolation. Cleaning scores +2.1, Optimization +3.7, Generation +1.2, Selection +2.9.
   - Round 2: Combine top performers. `[Optimization → Selection]` scores +5.3, `[Optimization → Cleaning → Selection]` scores +5.1.
   - Round 3: Tune Optimization parameters (rewrite both Q and A, not just A). Score rises to +6.0.
   - Round 4: Marginal gain (<0.5%). Converged.
5. Apply `[Optimization(Q+A) → Selection(top 85%)]` to all 7,000 noisy samples. Clean samples pass through unchanged.

Output:
```
Strategy search complete in 4 rounds.
Winning pipeline: Optimization(Q+A) → Selection(keep_top=85%)
  - 13,000 clean samples: passed through (no processing)
  - 7,000 noisy samples: 5,950 retained after optimization + selection
  - Final dataset: 18,950 samples
  - Proxy win rate vs unprocessed: 78%
  - Search cost: 4 rounds x 4 strategies = 16 evaluations
  - Cache hits: 6 (37.5% compute saved via CRM)
```

**Example 3: Privacy-Preserving Pipeline for Healthcare Data**

User: "We have patient discharge summaries that need processing before fine-tuning, but no one on the team should read individual records. Can you design a pipeline that works without human review of the data?"

Approach:
1. Operate in black-box mode: compute only aggregate statistics (distributions, token-length histograms, duplicate rates) -- never display individual samples.
2. Train PTS classifier on synthetic/public medical data labeled for quality, then apply it to the private dataset to flag noisy samples.
3. Use DPS on embeddings (embeddings are dense vectors, not readable text) to create a subsample.
4. Run strategy search using only aggregate metrics as feedback (validation perplexity, downstream task accuracy) -- no sample-level inspection.
5. Export pipeline config. All processing operators (dedup, length filter, optimization) execute programmatically without human review.

Output:
```
Privacy-preserving pipeline generated.
Mode: black-box (no raw data exposed to operators or logs)
Aggregate stats used: token-length distribution, duplicate rate, quality-score histogram
Pipeline: [Cleaning(dedup+length) → Optimization(answer_rewrite) → Selection(top_80%)]
Samples processed: 15,000 → 11,400
No individual records were displayed or logged during strategy search.
```

## Best Practices

- **Do:** Start with isolated operator teams in round 1 to understand the independent contribution of each before combining them. This prevents wasting early rounds on complex strategies where failures are hard to diagnose.
- **Do:** Use the Cache-and-Reuse mechanism aggressively. Structure candidate strategies so they share prefixes (e.g., all start with `Cleaning →`) to maximize cache hits across rounds.
- **Do:** Preserve the clean/noisy sample ratio when subsampling. Naive random sampling can over- or under-represent noisy samples, leading to misleading strategy evaluations.
- **Do:** Export the final pipeline as a reusable config so it can be applied to future batches of similar data without re-running the search.
- **Avoid:** Running optimization/generation operators on already-clean samples. Use PTS to target only noisy data -- this cuts cost and prevents degrading good samples.
- **Avoid:** Exceeding 5 refinement rounds. Empirically, gains plateau by round 4-5. Further rounds add compute cost with diminishing returns.

## Error Handling

- **Strategy produces worse results than baseline (negative score):** This is expected for some candidates. Ensure at least one conservative strategy (e.g., dedup-only) is included per round as a safety anchor. Never apply a negative-scoring strategy to the full dataset.
- **PTS classifier has low confidence on many samples:** The clean/noisy boundary may be ambiguous. Fall back to processing all samples rather than risking misclassification. Alternatively, use a confidence threshold and only skip processing for high-confidence clean samples.
- **Cache misses on every strategy:** Strategies may be too diverse. In the next round, explicitly instruct the search to generate strategies sharing a common prefix with the current best performer.
- **Subsample evaluation doesn't correlate with full-dataset results:** DPS may have failed to capture distributional structure. Increase subsample size from 20% to 40%, or verify the embedding model produces meaningful representations for the domain.
- **Fine-tuning proxy is too expensive per round:** Replace full fine-tuning with a cheaper proxy: perplexity evaluation on the validation set, or few-shot evaluation using the base model. This reduces per-strategy cost at some accuracy tradeoff.

## Limitations

- The framework assumes a held-out validation set exists to evaluate strategies. Without one, feedback signals are unreliable.
- Strategy search requires multiple fine-tuning runs (or proxy evaluations), which can be expensive for large models. Best suited for small-to-medium proxy models during search, then applying the winning pipeline before training the full-size model.
- The four operator teams (Cleaning, Optimization, Generation, Selection) cover common QA/instruction-tuning scenarios but may not capture all domain-specific needs (e.g., code datasets need AST-aware processing, multilingual data needs language-specific handling).
- Privacy guarantees are procedural (no human views data), not formal (no differential privacy). For regulated domains, additional privacy mechanisms should be layered on top.
- The binary quality classifier (PTS) requires some labeled examples. In domains where labeling is itself expensive or sensitive, bootstrapping the classifier may be a bottleneck.

## Reference

**Paper:** [LLM-AutoDP: Automatic Data Processing via LLM Agents for Model Fine-tuning](https://arxiv.org/abs/2601.20375v1) (VLDB 2026)
**Key insight:** Treating data processing as an iterative strategy search over composable operator teams -- where an LLM agent proposes, evaluates, and refines pipelines using only aggregate feedback -- converges to high-quality cleaning pipelines in 4-5 rounds with 80%+ win rates, while three acceleration techniques (DPS, PTS, CRM) cut search time by 10x.