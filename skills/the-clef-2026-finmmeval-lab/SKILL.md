---
name: "the-clef-2026-finmmeval-lab"
description: "Build multilingual, multimodal financial AI evaluation pipelines using the FinMMEval framework. Covers financial exam QA, cross-lingual document reasoning, and trading decision systems. Use when the user says: 'evaluate a financial LLM', 'build a financial QA benchmark', 'multilingual financial reasoning pipeline', 'financial decision-making agent', 'cross-lingual finance evaluation', 'financial exam question answering system'."
---

# FinMMEval: Multilingual & Multimodal Financial AI Evaluation

This skill enables Claude to build evaluation pipelines and data-processing systems for financial AI, following the FinMMEval framework from CLEF 2026. The framework decomposes financial AI evaluation into three tiers: domain knowledge testing via exam QA across six languages, cross-lingual document reasoning over SEC filings paired with multilingual news, and live trading decision-making with rationale generation. Claude can apply this structure to build benchmarks, implement evaluation harnesses, construct multilingual financial datasets, and create agent pipelines that reason over mixed financial modalities.

## When to Use

- When the user asks to evaluate or benchmark a financial LLM on multilingual data
- When building a financial question-answering system that handles multiple languages (English, Chinese, Arabic, Hindi, Spanish, Greek, Japanese)
- When constructing a pipeline to extract answers from SEC filings (10-K/10-Q) combined with multilingual news articles
- When implementing a trading decision agent that outputs Buy/Hold/Sell with justification from price data and news
- When designing a financial exam QA system covering CFA, CPA, EFPA, or similar professional certifications
- When the user needs to evaluate financial reasoning across difficulty tiers (factual vs. expert-level multi-document reasoning)
- When building evaluation metrics for financial tasks: accuracy, ROUGE-1, BLEURT, Sharpe ratio, cumulative return

## Key Technique

FinMMEval structures financial AI evaluation as a three-tier hierarchy moving from **knowledge** to **reasoning** to **action**. Task 1 (Financial Exam QA) tests foundational domain knowledge through multiple-choice questions drawn from professional certification exams in six languages -- CFA (English, 600q), CPA (Chinese, 300q), EFPA (Spanish, 230q), GRFinQA (Greek, 268q), BBF (Hindi, 500-1000q from 25+ Indian exams), and SAHM (Arabic, 873q). This establishes whether a model possesses the prerequisite financial vocabulary and conceptual grounding.

Task 2 (PolyFiQA) escalates to cross-lingual reasoning: given an English SEC filing excerpt and multilingual news articles (in English, Chinese, Japanese, Spanish, or Greek), the model must produce evidence-grounded answers (max 100 words) to questions posed in the target language. The dataset has two difficulty tiers -- Easy (factual/numerical trend extraction) and Expert (multi-document synthesis for investment strategy questions) -- with 172 instances per tier. Inter-annotator agreement exceeds 89%, and evaluation uses ROUGE-1 as the primary metric with BLEURT and factual consistency as secondaries.

Task 3 (Financial Decision Making) closes the loop by requiring actionable output. Given a market context consisting of historical daily prices, contemporaneous news, and momentum labels (bullish/neutral/bearish), the model must output a discrete trading action (Buy/Hold/Sell) plus a concise rationale (max 50 words) citing evidence. This is evaluated on real financial performance metrics: cumulative return, Sharpe ratio, maximum drawdown, and volatility. The task currently covers BTC and TSLA with daily submission cadence, preventing forward-looking bias.

## Step-by-Step Workflow

1. **Identify the evaluation tier.** Determine which of the three tasks applies: exam QA (knowledge testing), cross-lingual QA (document reasoning), or decision-making (action generation). If building a full benchmark, implement all three in sequence.

2. **Define language scope.** Select target languages from the supported set. Task 1 covers English, Chinese, Arabic, Hindi, Spanish, Greek. Task 2 covers English, Chinese, Japanese, Spanish, Greek. Task 3 is English-only. Participants may evaluate on any subset.

3. **Construct the dataset schema.** For Task 1: `{question: str, options: [A1, A2, A3, A4], answer: str, language: str, exam_source: str}`. For Task 2: `{report: str, news: {lang: str, text: str}[], question: str, question_lang: str, answer: str, difficulty: "easy"|"expert"}`. For Task 3: `{date: str, asset: str, price: float, news: str[], momentum: "bullish"|"neutral"|"bearish", action: "buy"|"hold"|"sell", rationale: str}`.

4. **Implement data ingestion for each modality.** Parse SEC filings (10-K/10-Q) into structured sections. Ingest multilingual news with language tags. For Task 3, build a time-series loader that pairs daily prices with contemporaneous news, enforcing no future data leakage (strict `date <= current_date` filtering).

5. **Build the evaluation harness per task.** Task 1: compute accuracy as `correct / total` per language and overall. Task 2: compute ROUGE-1 between generated and reference answers, plus BLEURT scores and factual consistency checks. Task 3: simulate a portfolio from the action sequence and compute cumulative return, Sharpe ratio, max drawdown, daily volatility, and annualized volatility.

6. **Implement the cross-lingual QA pipeline (Task 2).** Accept a question in language L, retrieve the relevant English filing section and news articles in language L, run the model to generate an evidence-grounded answer (capped at 100 words), and score against the reference.

7. **Implement the decision agent (Task 3).** Build a daily loop: load the market context up to day t, feed price history + news + momentum label to the model, extract the discrete action and rationale, record the decision, and advance to t+1. Enforce the 50-word rationale limit.

8. **Add quality controls.** Validate exam questions against known answer keys. For PolyFiQA, verify inter-annotator agreement thresholds (target > 89%). For decision-making, verify no lookahead bias by asserting all input timestamps precede the decision timestamp.

9. **Generate evaluation reports.** Produce per-language breakdowns for Tasks 1 and 2. For Task 3, produce equity curves, drawdown charts, and risk-adjusted return tables. Aggregate cross-task scores to assess the full knowledge-reasoning-action pipeline.

10. **Iterate on failure modes.** Analyze which languages or question types cause the largest accuracy drops. For Task 2, check if errors cluster in the Easy or Expert tier. For Task 3, examine whether losses correlate with specific momentum regimes or news sentiment misreadings.

## Concrete Examples

**Example 1: Building a Financial Exam QA Evaluation Pipeline**

User: "I have a financial LLM and want to evaluate it on CFA-style multiple-choice questions across multiple languages."

Approach:
1. Define the dataset schema for exam QA with fields for question text, four options, correct answer, language code, and exam source.
2. Load or construct question sets per language (e.g., CFA English, CPA Chinese, EFPA Spanish).
3. Run the model on each question, extracting the selected option.
4. Compute per-language accuracy and overall accuracy.

```python
import json
from collections import defaultdict

def evaluate_exam_qa(model, dataset_path: str) -> dict:
    with open(dataset_path) as f:
        questions = json.load(f)

    results = defaultdict(lambda: {"correct": 0, "total": 0})

    for q in questions:
        prompt = (
            f"Question ({q['exam_source']}, {q['language']}):\n"
            f"{q['question']}\n"
            f"A) {q['options'][0]}\nB) {q['options'][1]}\n"
            f"C) {q['options'][2]}\nD) {q['options'][3]}\n"
            f"Answer with only the letter (A, B, C, or D)."
        )
        prediction = model.generate(prompt).strip().upper()
        lang = q["language"]
        results[lang]["total"] += 1
        if prediction == q["answer"]:
            results[lang]["correct"] += 1

    summary = {}
    for lang, counts in results.items():
        summary[lang] = {
            "accuracy": counts["correct"] / counts["total"],
            "correct": counts["correct"],
            "total": counts["total"],
        }
    summary["overall"] = {
        "accuracy": sum(r["correct"] for r in results.values())
                  / sum(r["total"] for r in results.values())
    }
    return summary
```

Output:
```json
{
  "en": {"accuracy": 0.72, "correct": 432, "total": 600},
  "zh": {"accuracy": 0.65, "correct": 195, "total": 300},
  "es": {"accuracy": 0.61, "correct": 140, "total": 230},
  "overall": {"accuracy": 0.68}
}
```

**Example 2: Cross-Lingual Financial Document QA (PolyFiQA)**

User: "Build a pipeline that answers financial questions in Japanese using English SEC filings and Japanese news articles."

Approach:
1. Ingest the English 10-K filing and segment it into sections (revenue, cash flow, risk factors).
2. Load Japanese news articles tagged with relevance scores.
3. For each question in Japanese, retrieve the most relevant filing section and news article.
4. Generate an evidence-grounded answer in Japanese (max 100 words).
5. Evaluate with ROUGE-1 against reference answers.

```python
from rouge_score import rouge_scorer

def polyfiqa_pipeline(model, filing_text: str, news_articles: list[dict],
                      question: str, lang: str) -> str:
    # Retrieve relevant context
    context_prompt = (
        f"SEC Filing Excerpt:\n{filing_text[:3000]}\n\n"
        f"News ({lang}):\n"
        + "\n".join(a["text"][:500] for a in news_articles[:3])
        + f"\n\nQuestion ({lang}): {question}\n\n"
        f"Answer in {lang} using evidence from the documents above. "
        f"Maximum 100 words."
    )
    answer = model.generate(context_prompt)
    # Enforce word limit
    words = answer.split()
    return " ".join(words[:100])

def evaluate_polyfiqa(predictions: list[str], references: list[str]) -> dict:
    scorer = rouge_scorer.RougeScorer(["rouge1"], use_stemmer=False)
    scores = [scorer.score(ref, pred)["rouge1"].fmeasure
              for pred, ref in zip(predictions, references)]
    return {"rouge1_mean": sum(scores) / len(scores)}
```

Output:
```json
{
  "rouge1_mean": 0.43,
  "easy_tier_rouge1": 0.51,
  "expert_tier_rouge1": 0.35
}
```

**Example 3: Financial Trading Decision Agent**

User: "Create a daily trading decision agent for BTC that outputs Buy/Hold/Sell with rationale."

Approach:
1. Build a time-series data loader that provides price history and news up to each day.
2. Feed the context to the model with the momentum label.
3. Parse the discrete action and rationale from the output.
4. Simulate the portfolio and compute performance metrics.

```python
import numpy as np

def trading_agent_step(model, date: str, prices: list[float],
                       news: list[str], momentum: str) -> dict:
    recent_prices = prices[-30:]  # Last 30 days
    prompt = (
        f"Date: {date}\n"
        f"Asset: BTC\n"
        f"Recent prices (last 30 days): {recent_prices}\n"
        f"Current price: {recent_prices[-1]}\n"
        f"Momentum: {momentum}\n"
        f"Recent news:\n" + "\n".join(f"- {n}" for n in news[-5:]) +
        f"\n\nDecision: Output exactly one of: BUY, HOLD, SELL\n"
        f"Rationale: Justify in 50 words or fewer, citing evidence.\n"
        f"Format: ACTION: <action>\\nRATIONALE: <text>"
    )
    response = model.generate(prompt)
    action = parse_action(response)  # Extract BUY/HOLD/SELL
    rationale = parse_rationale(response)  # Extract justification
    return {"date": date, "action": action, "rationale": rationale}

def compute_portfolio_metrics(actions: list[dict], prices: list[float]) -> dict:
    # Simulate: BUY = +1 position, SELL = -1, HOLD = 0 change
    returns = np.diff(prices) / prices[:-1]
    daily_returns = []
    position = 0
    for i, act in enumerate(actions[:-1]):
        if act["action"] == "BUY": position = 1
        elif act["action"] == "SELL": position = 0
        daily_returns.append(position * returns[i])
    dr = np.array(daily_returns)
    cumulative_return = float(np.prod(1 + dr) - 1)
    sharpe = float(np.mean(dr) / (np.std(dr) + 1e-8) * np.sqrt(365))
    max_dd = float(np.min(np.minimum.accumulate(np.cumprod(1 + dr)) - np.cumprod(1 + dr)))
    return {
        "cumulative_return": cumulative_return,
        "sharpe_ratio": sharpe,
        "max_drawdown": max_dd,
        "annualized_volatility": float(np.std(dr) * np.sqrt(365)),
    }
```

Output:
```json
{
  "cumulative_return": 0.124,
  "sharpe_ratio": 1.82,
  "max_drawdown": -0.067,
  "annualized_volatility": 0.248
}
```

## Best Practices

- **Do:** Enforce strict temporal ordering in Task 3 -- never include future prices or news in the model's context. Assert `all(input_date <= decision_date)` in your data loader.
- **Do:** Evaluate per-language separately before aggregating. A high overall score can mask catastrophic failure in low-resource languages like Hindi or Greek.
- **Do:** Cap answer length (100 words for PolyFiQA, 50 words for decision rationale). Verbose outputs dilute ROUGE scores and obscure reasoning quality.
- **Do:** Use professional-grade financial question sources (CFA, CPA exam banks) rather than synthetic questions. The benchmark's value comes from authentic domain expertise.
- **Avoid:** Translating English questions to other languages as a shortcut for multilingual evaluation. PolyFiQA uses native-language news paired with English filings -- the cross-lingual challenge is intentional.
- **Avoid:** Evaluating Task 3 on historical data where the model may have seen the outcomes in training. Use a live or post-training cutoff period to prevent data contamination.

## Error Handling

- **Language detection mismatch:** If the model answers in the wrong language, flag it as an automatic zero for that instance. Add a language-detection check (e.g., `langdetect`) on outputs before scoring.
- **Malformed action output (Task 3):** If the model doesn't produce a clean BUY/HOLD/SELL token, default to HOLD and log the failure. Track malformed output rate as a secondary metric.
- **Exceeding word limits:** Truncate at the word limit before scoring. Do not penalize further -- the truncated output is the scored output.
- **Missing news articles:** If multilingual news is unavailable for a specific date/language, fall back to the English filing only and flag the instance. This tests the model's ability to reason from partial context.
- **SEC filing parsing failures:** 10-K/10-Q filings have inconsistent formatting. Use section-header regex patterns (`Item 1`, `Item 7`, etc.) and fall back to sliding-window chunking if headers aren't detected.

## Limitations

- The framework currently covers only six languages for exam QA and five for PolyFiQA. Financial systems operating in other languages (e.g., Portuguese, Korean, German) need custom dataset construction.
- Task 3 is limited to two assets (BTC, TSLA). Generalizing to broader portfolios, options, or fixed-income instruments requires extending the data schema and evaluation metrics.
- The exam QA task is text-only multiple-choice. It does not test visual reasoning over charts, tables in filings, or scanned documents -- future FinMMEval editions plan to add these modalities.
- ROUGE-1 as the primary metric for PolyFiQA is a rough proxy for answer quality. Factual consistency and financial accuracy may diverge from lexical overlap, especially across languages with different morphology.
- The decision-making task uses a simple position model (fully in or fully out). Real trading involves position sizing, risk limits, and transaction costs not captured here.

## Reference

**Paper:** Xie et al., "The CLEF-2026 FinMMEval Lab: Multilingual and Multimodal Evaluation of Financial AI Systems" (arXiv:2602.10886v1, 2026). Read for: the three-tier evaluation hierarchy (knowledge/reasoning/action), per-language dataset specifications, and the live-submission anti-lookahead protocol for financial decision evaluation.