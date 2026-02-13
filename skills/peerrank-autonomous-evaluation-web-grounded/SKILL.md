---
name: "peerrank-autonomous-evaluation-web-grounded"
description: "Implement PeerRank-style autonomous multi-model evaluation pipelines where LLMs symmetrically generate tasks, answer with web grounding, judge peers, and aggregate bias-controlled scores into rankings. Use when: 'evaluate multiple LLMs against each other', 'build an autonomous model benchmark', 'run peer review across AI models', 'detect bias in LLM judgments', 'rank models without human labels', 'set up web-grounded model evaluation'."
---

# PeerRank: Autonomous LLM Evaluation Through Web-Grounded, Bias-Controlled Peer Review

This skill enables Claude to build and orchestrate **PeerRank-style evaluation pipelines** — fully autonomous systems where multiple LLMs symmetrically act as task designers, respondents, and evaluators. Instead of relying on static benchmarks or human judges, each model generates questions, answers them with live web retrieval, scores peer responses, and the system aggregates bias-filtered peer assessments into stable rankings. This eliminates the need for gold references, human annotation, or single-judge bottlenecks.

## When to Use

- When the user wants to **compare multiple LLMs** (e.g., GPT-4, Claude, Gemini) without relying on static leaderboards
- When building an **automated evaluation harness** that stays current by grounding answers in live web data
- When the user needs to **detect and quantify bias** in model-as-judge setups (self-preference, name bias, position bias)
- When designing a **multi-agent evaluation framework** where no single model has privileged authority
- When the user asks to **validate model rankings** against established benchmarks like TruthfulQA or GSM8K
- When replacing a single LLM-as-judge with a **peer panel** to reduce systematic evaluation errors

## Key Technique

PeerRank's core insight is treating LLM evaluation as a **symmetric multi-agent process**. Every participating model plays all three roles: it generates evaluation questions within defined categories (factual knowledge, reasoning/logic, current events, creative/open-ended, practical how-to), it answers every question posed by every other model using optional web retrieval, and it independently scores every peer's response on a 1-10 rubric. This symmetry means no model has special authority — rankings emerge from the collective assessment.

**Web grounding** is category-scoped: only current-events questions trigger live web retrieval (via a uniform provider like Tavily or SerpAPI), while other categories test pure model capability. This prevents web access from masking reasoning deficiencies while ensuring factual currency is rewarded where it matters.

**Bias control** is the critical differentiator. PeerRank runs evaluations under three regimes — shuffle-only (randomize answer order, show identities), blind-only (hide identities, fixed order), and shuffle+blind (hide identities and randomize order). By comparing scores across regimes, the system quantifies self-bias (`delta_self = mean(self_ratings) - peer_score`), name bias (shuffle minus shuffle+blind), and position bias (blind minus shuffle+blind). The shuffle+blind baseline serves as the debiased ground truth. Final peer scores are computed as `P_j = E[s_{i!=j, q}]` — the mean of all peer-assigned scores excluding self-ratings — then sorted to produce rankings.

## Step-by-Step Workflow

### 1. Define the Model Pool and Categories

Enumerate all models to evaluate (minimum 4 for meaningful peer statistics, ideally 8-12). Define 5 evaluation categories: factual knowledge, reasoning/logic, current events, creative/open-ended, and practical how-to. Assign a question quota per model (the paper uses 35 per model, 7 per category).

### 2. Generate Evaluation Questions

Prompt each model to generate its quota of questions using structured JSON output. Each question must specify its category, difficulty level, and expected answer characteristics. Use a generation prompt like:

```
Generate {n} evaluation questions for category "{category}".
Output JSON: [{"question": str, "category": str, "difficulty": "easy"|"medium"|"hard"}]
Ensure questions are specific, unambiguous, and answerable. Avoid yes/no questions.
```

### 3. Compile the Question Bank

Aggregate all generated questions into a single pool. Do not filter or deduplicate — the endogenous distribution is part of the signal (a model that generates poor questions reveals its own limitations). Tag each question with its source model ID for later bias analysis.

### 4. Collect Web-Grounded Responses

Route every question to every model. For current-events questions only, prepend web retrieval context using a uniform search provider (Tavily, SerpAPI, or equivalent). All other categories receive no web context. Collect responses as structured JSON:

```
{"model_id": str, "question_id": str, "response": str, "web_sources_used": [str] | null}
```

### 5. Run Peer Evaluation Under Three Bias Regimes

For each question, have every non-author model evaluate every response. Run under three conditions:

- **Shuffle+Blind** (baseline): Randomize response order, strip model identities
- **Shuffle-only**: Randomize order, show model names
- **Blind-only**: Fixed order, strip identities

Evaluators output structured judgments:

```json
{"score": 8, "reason": "accurate with strong sourcing but minor omission on timeline", "flags": ["clear_correct"]}
```

Score rubric: 10 = correct, complete, well-justified; 7-9 = mostly correct, minor gaps; 4-6 = mixed correctness; 1-3 = mostly incorrect or hallucinated. Flags include: `hallucination`, `unsupported_specifics`, `evasive`, `incorrect`, `good_uncertainty`, `clear_correct`.

### 6. Compute Bias Metrics

For each model j, calculate:
- **Self-bias**: `delta_self_j = mean(scores j gave itself) - mean(scores peers gave j)`
- **Name bias**: `mean(shuffle_scores_j) - mean(shuffle_blind_scores_j)`
- **Position bias**: `mean(blind_scores_j) - mean(shuffle_blind_scores_j)`

Flag models or individual judgments where bias exceeds 1.0 points on the 10-point scale.

### 7. Aggregate Debiased Peer Scores

Compute the final peer score for model j using only shuffle+blind evaluations, excluding self-ratings:

```
P_j = (1 / (N-1)*Q) * sum over all i!=j, all questions q of s_{i,j,q}
```

Where N = number of models, Q = number of questions. Rank models by descending P_j.

### 8. Compute Elo Ratings for Validation

Convert pairwise score comparisons into Elo ratings (K-factor=32). For each question, the model with the higher peer score wins the pairwise match. Verify Spearman rank correlation between Elo and mean peer scores (target rho > 0.7).

### 9. Validate Against External Benchmarks (Optional)

If available, correlate peer scores with known benchmark accuracy (e.g., TruthfulQA, GSM8K) to confirm the pipeline produces meaningful rankings.

### 10. Generate the Evaluation Report

Output a structured report with: ranked model table, per-category breakdowns, bias analysis heatmap, confidence intervals, and flagged anomalies.

## Concrete Examples

**Example 1: Comparing 4 LLMs for a RAG Application**

```
User: I need to pick the best LLM for our retrieval-augmented generation pipeline.
      We're considering GPT-4o, Claude Sonnet, Gemini 1.5, and Llama 3.1 70B.
      Can you set up an automated evaluation?

Approach:
1. Define 5 categories weighted toward RAG relevance: factual knowledge (10 questions),
   reasoning/logic (7), current events (8), creative synthesis (5), practical how-to (5)
2. Script API calls for each model to generate its 35 questions (structured JSON)
3. Route all 140 questions (4 models * 35) to all 4 models with web retrieval
   enabled only for current-events questions via Tavily
4. Run peer evaluation: each model judges all 3 peers' answers per question
   under shuffle+blind regime (560 evaluations total per regime)
5. Compute P_j for each model, compute self-bias deltas
6. Generate ranking table with confidence intervals

Output:
| Rank | Model          | Peer Score (P_j) | Self-Bias | Factual | Reasoning | Current |
|------|----------------|-------------------|-----------|---------|-----------|---------|
| 1    | Claude Sonnet  | 8.12 +/- 0.23    | +0.31     | 8.4     | 8.3       | 7.8     |
| 2    | GPT-4o         | 7.89 +/- 0.19    | +0.45     | 8.1     | 7.9       | 7.6     |
| 3    | Gemini 1.5     | 7.54 +/- 0.28    | +0.22     | 7.3     | 7.2       | 8.1     |
| 4    | Llama 3.1 70B  | 6.91 +/- 0.35    | +0.67     | 7.0     | 6.8       | 6.9     |

Bias Alert: Llama 3.1 shows elevated self-bias (+0.67). Recommend inspecting
its evaluation prompts for systematic leniency patterns.
```

**Example 2: Building a Bias Audit for LLM-as-Judge**

```
User: We use GPT-4 as a judge in our eval pipeline. How can I check if it's biased?

Approach:
1. Collect 50 existing evaluation questions from the user's pipeline
2. Get responses from 3+ models (including GPT-4 itself)
3. Run GPT-4 evaluation under all three bias regimes:
   - Shuffle+Blind: anonymized, randomized order
   - Shuffle-only: names visible, randomized order
   - Blind-only: anonymized, fixed order
4. Compute name bias and position bias per scored response
5. Statistical test: paired t-test between regimes, report effect sizes

Output:
Bias Analysis for GPT-4 as Judge:
- Self-bias: +0.82 points (rates own answers higher than peers do)
- Name bias: +0.34 when evaluating "GPT-4" labeled responses vs anonymous
- Position bias: +0.21 toward first-position responses
- Recommendation: Use shuffle+blind regime in production. Consider replacing
  single-judge with 3-model peer panel to reduce variance by ~40%.
```

**Example 3: Implementing a Continuous Evaluation Pipeline**

```
User: Set up a weekly automated model evaluation that stays current.

Approach:
1. Create a cron-triggered pipeline with these stages:
   a. question_gen.py — Each model generates 7 questions per category (35 total)
   b. web_answer.py — All models answer; current-events questions get fresh
      Tavily search results prepended as context
   c. peer_eval.py — Shuffle+blind evaluation pass, structured JSON output
   d. aggregate.py — Compute P_j, Elo, bias deltas, week-over-week trends
   e. report.py — Generate markdown report with ranking changes highlighted
2. Store all raw evaluations in append-only JSONL for longitudinal analysis
3. Alert on ranking changes > 2 positions or bias spikes > 1.0

Pipeline structure:
  weekly_eval/
    config.yaml          # model list, API keys, category weights
    question_gen.py      # structured question generation per model
    web_answer.py        # web-grounded response collection
    peer_eval.py         # multi-regime peer scoring
    aggregate.py         # P_j computation, Elo, bias metrics
    report.py            # markdown + JSON report generation
    data/                # append-only JSONL storage
    reports/             # weekly output reports
```

## Best Practices

- **Do:** Always use the shuffle+blind regime as your primary scoring baseline — it is the only condition that controls for both identity and position bias simultaneously.
- **Do:** Exclude self-ratings from aggregation. Self-bias is a measurement signal, not a scoring input. Compute `P_j` only from peer scores where `i != j`.
- **Do:** Use structured JSON output for all pipeline stages (question generation, response collection, evaluation). This makes bias analysis and aggregation programmatic.
- **Do:** Apply web grounding selectively — only for questions that genuinely require current information. Blanket web access masks reasoning and knowledge differences.
- **Avoid:** Using fewer than 4 models. PeerRank's statistical power depends on multiple independent evaluators; with 2-3 models, individual biases dominate.
- **Avoid:** Filtering or deduplicating generated questions before the evaluation run. The quality of questions a model generates is itself diagnostic information. Filter only for malformed outputs.

## Error Handling

- **API failures during response collection**: Retry with exponential backoff. Log missing responses as null — the aggregation formula handles partial data by adjusting the denominator.
- **Malformed evaluator output**: If a model returns scores outside 1-10 or invalid JSON, discard that single judgment. Do not discard the entire question. Log the failure rate per model as a reliability metric.
- **Score collapse** (all models score ~same): Increase question difficulty, add more categories, or check that the evaluation prompt demands fine-grained scoring. A rubric that allows "7 for anything decent" produces uninformative rankings.
- **Bias exceeding scoring range**: If self-bias > 2.0 on a 10-point scale, the model's evaluations are unreliable. Exclude it as a judge (but keep it as a respondent) and report the exclusion.
- **Web retrieval failures**: If the search API fails for current-events questions, mark those questions as "ungrounded" and exclude them from the current-events category score. Do not fall back to ungrounded answers — that would give retrieval-failed models an unfair advantage on web-dependent questions.

## Limitations

- **Cost scales quadratically**: With N models and Q questions, you need ~N*N*Q API calls for full peer evaluation. At N=12, Q=420, this is ~55,000+ calls per run. Budget accordingly or sample.
- **Models may game structured evaluation**: If models recognize they are in an evaluation pipeline, behavior may diverge from production use. Mitigate by varying prompt templates across runs.
- **Web grounding introduces provider dependency**: Different search APIs return different results. Rankings for current-events questions are only comparable within a single search provider per run.
- **Peer scores are relative, not absolute**: PeerRank tells you model A is better than model B in this pool. Adding or removing a model changes everyone's scores. Do not compare P_j across different evaluation runs with different model pools.
- **Creative/open-ended categories are inherently noisier**: Peer agreement is lower for subjective tasks. Weight these categories lower if you need high-confidence rankings, or report them separately.

## Reference

**Paper**: [PeerRank: Autonomous LLM Evaluation Through Web-Grounded, Bias-Controlled Peer Review](https://arxiv.org/abs/2602.02589v1) (Margalit et al., 2026)

Key implementation details to reference: Section on bias regimes (shuffle/blind/shuffle+blind), the peer score formula `P_j = E[s_{i!=j, q}]`, the evaluation rubric (1-10 with flags), and the Elo validation methodology (K=32, 254K matches, Spearman rho=0.755).