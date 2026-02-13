---
name: "benchmarking-zero-shot-few-shot-phishing"
description: "Detect phishing URLs using LLM zero-shot and few-shot prompting with structured classification prompts. Use when: 'classify this URL as phishing or legitimate', 'analyze URLs for phishing', 'build a phishing detection prompt', 'detect suspicious URLs with few-shot examples', 'benchmark phishing detection accuracy', 'zero-shot URL security classification'."
---

This skill enables Claude to classify URLs as phishing or legitimate using the unified zero-shot and few-shot prompting framework from Hasan & BusiReddyGari (2025). Rather than relying on feature-engineered ML pipelines or blocklists, this approach passes raw URL strings directly to an LLM with a structured prompt comprising an instruction, optional balanced examples, and a query — achieving up to 94% accuracy with just six few-shot examples. The technique is especially valuable when labeled training data is scarce or when new phishing campaigns emerge faster than traditional models can retrain.

## When to Use

- When a user asks to classify one or more URLs as phishing or legitimate without training a model
- When building a phishing URL detection pipeline that needs to work immediately with zero labeled data
- When the user wants to evaluate or compare LLM prompting strategies for security classification tasks
- When analyzing suspicious URLs in an incident response or threat hunting workflow
- When the user needs a structured prompt template for binary URL classification with constrained output
- When benchmarking detection accuracy across different prompting configurations (zero-shot vs few-shot, balanced vs imbalanced)

## Key Technique

The paper's core insight is that LLMs can perform phishing URL detection through **prompt-only inference** — no manual feature extraction, no fine-tuning, no URL parsing into domain age, path length, or entropy features. The URL string itself contains enough lexical and structural signal (misspelled brand names, excessive subdomains, suspicious TLDs, encoded characters, abnormal path depth) for an LLM to classify it when given the right prompt structure.

The unified prompting framework concatenates three components: an **Instruction** (I) that assigns a cybersecurity expert role and constrains output to binary labels, optional **Examples** (E) that provide balanced URL-label demonstrations, and a **Query** (Q) containing the target URL. Temperature is set to 0 for deterministic output, and max tokens is capped at 10 to force concise classification. Few-shot examples are balanced (equal phishing and legitimate) and disjoint from the evaluation set.

The critical finding is that **few-shot prompting with six balanced examples (3 phishing, 3 legitimate) substantially improves performance across all tested LLMs**, boosting F1 by 2-4 percentage points over zero-shot. However, the relationship between example count and performance is model-dependent: some models peak with just one example while others improve steadily up to nine. This means the optimal few-shot configuration must be empirically validated per model.

## Step-by-Step Workflow

1. **Collect target URLs** — Gather the URL(s) to classify. Preserve the full URL string including protocol, subdomains, path, query parameters, and fragments. Do not normalize, truncate, or strip components.

2. **Construct the system instruction** — Use a role-constrained prompt that assigns cybersecurity expertise and forces binary output:
   ```
   You are a cybersecurity expert. Respond only with 0 for phishing or 1 for legitimate.
   ```

3. **Select few-shot examples (if available)** — Choose 6 balanced examples: 3 known phishing URLs and 3 known legitimate URLs. Ensure examples are representative of common phishing patterns (typosquatting, subdomain abuse, path mimicry) and common legitimate patterns (well-known domains, standard paths). Keep examples disjoint from any URLs being classified.

4. **Format the few-shot examples block** — Structure each example as a URL-label pair with descriptive labels:
   ```
   URL: http://secure-bankofamerica.com.verify.xyz/login  Answer: 0 (phishing)
   URL: https://www.google.com/search?q=weather  Answer: 1 (legitimate)
   URL: http://paypa1-verify.com/update-info  Answer: 0 (phishing)
   URL: https://github.com/anthropics/claude-code  Answer: 1 (legitimate)
   URL: http://microsoft-365.account-verify.ru/signin  Answer: 0 (phishing)
   URL: https://stackoverflow.com/questions/12345  Answer: 1 (legitimate)
   ```

5. **Construct the query** — Format the target URL for classification:
   ```
   URL: {target_url}  Is this URL phishing or legitimate? Respond with 0 or 1.
   ```

6. **Set inference parameters** — Use temperature=0 for deterministic classification and limit max output tokens to 10 to prevent verbose explanations that complicate parsing.

7. **Parse the response** — Extract the binary label (0 or 1) from the model output. If the response is unparseable (contains neither 0 nor 1 as a standalone token), flag it for manual review rather than guessing.

8. **Compute evaluation metrics (for batch classification)** — When classifying a dataset, compute accuracy, precision, recall, F1 (macro-averaged), AUROC, and AUPRC using scikit-learn. Report all six metrics — accuracy alone is misleading on imbalanced data.

9. **Analyze precision-recall trade-offs** — Few-shot prompting typically increases precision (fewer false positives) at a slight cost to recall (more missed phishing). Decide which trade-off fits the deployment context: high recall for user-facing filters, high precision for automated blocking.

10. **Iterate on example selection** — If performance is below target, experiment with different few-shot examples. Prioritize examples that represent the phishing patterns most common in your target domain (e.g., financial brand impersonation, OAuth phishing, URL shortener abuse).

## Concrete Examples

**Example 1: Zero-shot single URL classification**

User: "Is this URL phishing? http://amaz0n-prime.account-update.xyz/verify?id=8823"

Approach:
1. Construct system prompt: "You are a cybersecurity expert. Respond only with 0 for phishing or 1 for legitimate."
2. Construct query: "URL: http://amaz0n-prime.account-update.xyz/verify?id=8823  Is this URL phishing or legitimate? Respond with 0 or 1."
3. Set temperature=0, max_tokens=10.
4. Parse output.

Output:
```
Classification: 0 (phishing)

Indicators detected:
- Brand impersonation: "amaz0n" uses character substitution (0 for o)
- Suspicious domain: "account-update.xyz" is not an Amazon-owned domain
- Deceptive path: "/verify" mimics a legitimate account verification flow
- Query parameter: "id=8823" suggests phishing kit session tracking
```

**Example 2: Few-shot batch classification with evaluation**

User: "I have a CSV of 500 URLs. Build me a phishing detection pipeline using few-shot prompting and report accuracy metrics."

Approach:
1. Load the CSV and extract the URL column and ground-truth labels.
2. Select 6 balanced few-shot examples (3 phishing, 3 legitimate) from a held-out set, disjoint from the 500 evaluation URLs.
3. For each URL, construct the full prompt: system instruction + few-shot block + query.
4. Call the LLM API with temperature=0, max_tokens=10.
5. Parse binary predictions, flagging unparseable responses.
6. Compute metrics against ground truth.

Output:
```python
import pandas as pd
from sklearn.metrics import (accuracy_score, precision_score, recall_score,
                             f1_score, roc_auc_score, average_precision_score)

# Load data
df = pd.read_csv("urls.csv")  # columns: url, label (0=phishing, 1=legitimate)

# Hold out 6 examples for few-shot (3 per class), disjoint from eval set
examples_phishing = df[df.label == 0].sample(3, random_state=42)
examples_legit = df[df.label == 1].sample(3, random_state=42)
few_shot_ids = set(examples_phishing.index) | set(examples_legit.index)
eval_df = df.drop(index=few_shot_ids)

SYSTEM_PROMPT = ("You are a cybersecurity expert. "
                 "Respond only with 0 for phishing or 1 for legitimate.")

def build_few_shot_block(phishing_examples, legit_examples):
    lines = []
    for _, row in phishing_examples.iterrows():
        lines.append(f"URL: {row.url}  Answer: 0 (phishing)")
    for _, row in legit_examples.iterrows():
        lines.append(f"URL: {row.url}  Answer: 1 (legitimate)")
    return "\n".join(lines)

def classify_url(url, few_shot_block, client):
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": f"{few_shot_block}\n\nURL: {url}\n"
         "Is this URL phishing or legitimate? Respond with 0 or 1."}
    ]
    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=10,
        temperature=0,
        messages=messages
    )
    text = response.content[0].text.strip()
    if "0" in text:
        return 0
    elif "1" in text:
        return 1
    return None  # unparseable

few_shot_block = build_few_shot_block(examples_phishing, examples_legit)

# Classify all evaluation URLs
predictions = []
for _, row in eval_df.iterrows():
    pred = classify_url(row.url, few_shot_block, client)
    predictions.append(pred)

eval_df["pred"] = predictions
valid = eval_df.dropna(subset=["pred"])

# Report metrics
print(f"Accuracy:  {accuracy_score(valid.label, valid.pred):.4f}")
print(f"Precision: {precision_score(valid.label, valid.pred, average='macro'):.4f}")
print(f"Recall:    {recall_score(valid.label, valid.pred, average='macro'):.4f}")
print(f"F1:        {f1_score(valid.label, valid.pred, average='macro'):.4f}")
print(f"AUROC:     {roc_auc_score(valid.label, valid.pred):.4f}")
print(f"AUPRC:     {average_precision_score(valid.label, valid.pred):.4f}")
print(f"Unparseable: {eval_df.pred.isna().sum()} / {len(eval_df)}")
```

**Example 3: Comparing zero-shot vs few-shot for a custom URL list**

User: "Compare zero-shot and few-shot phishing detection on my URL list and show which is better."

Approach:
1. Run the full evaluation pipeline twice: once without examples (zero-shot), once with 6 balanced examples (few-shot).
2. Use identical system instruction and query format for both.
3. Compare all six metrics side by side.

Output:
```
| Metric    | Zero-shot | Few-shot (6 examples) | Delta  |
|-----------|-----------|-----------------------|--------|
| Accuracy  | 0.8760    | 0.9250                | +0.049 |
| Precision | 0.8780    | 0.9030                | +0.025 |
| Recall    | 0.8730    | 0.9530                | +0.080 |
| F1        | 0.8760    | 0.9270                | +0.051 |
| AUROC     | 0.8760    | 0.9250                | +0.049 |
| AUPRC     | 0.9070    | 0.9400                | +0.033 |

Recommendation: Few-shot prompting with 6 balanced examples improves
all metrics. The largest gain is in recall (+8%), meaning fewer phishing
URLs are missed. Use few-shot for production deployments.
```

## Best Practices

- **Do:** Always use balanced few-shot examples (equal phishing and legitimate). Imbalanced examples bias the model toward the majority class.
- **Do:** Set temperature to 0 and max tokens to 10. This forces deterministic, concise output and prevents the model from hedging or explaining instead of classifying.
- **Do:** Use macro-averaged metrics (not micro) so each class contributes equally to the score, especially important when class distributions are imbalanced.
- **Do:** Keep few-shot examples disjoint from evaluation data. Leaking test URLs into examples inflates metrics.
- **Avoid:** Stripping URL components before classification. The full URL string — including scheme, subdomains, path, query parameters, and fragments — contains classification signal. Do not normalize away features.
- **Avoid:** Using more than 9 few-shot examples without validation. The paper shows some models degrade with too many examples. Start with 6 and test 1, 3, and 9 to find the optimum.
- **Avoid:** Relying on accuracy alone for imbalanced datasets. At 1% phishing prevalence, a model that always predicts "legitimate" achieves 99% accuracy but catches zero threats. Always report F1, AUROC, and AUPRC.

## Error Handling

- **Unparseable responses:** If the model returns text that contains neither "0" nor "1" as a standalone token, mark the prediction as null and exclude it from metric computation. Report the unparseable rate — if it exceeds 2%, the prompt format may need adjustment.
- **Rate limiting:** When classifying large batches (1000+ URLs), implement exponential backoff and batch the requests. Consider using the Anthropic Batch API for cost efficiency.
- **Example contamination:** If few-shot examples accidentally overlap with evaluation URLs, remove them. Even one leaked example can inflate metrics on small datasets.
- **Encoding issues:** Some phishing URLs contain unicode homoglyphs (e.g., Cyrillic "а" vs Latin "a"), punycode (xn--), or percent-encoded characters. Pass these as-is — the LLM handles them natively through its tokenizer.
- **Model refusals:** Some LLMs may refuse to classify URLs they perceive as harmful content. If this occurs, adjust the system prompt to emphasize the defensive cybersecurity context: "You are performing authorized security analysis to protect users from phishing threats."

## Limitations

- **No real-time page content analysis.** This technique classifies URL strings only. It cannot detect phishing pages hosted at legitimate-looking URLs (e.g., compromised WordPress sites) that require page content or screenshot analysis.
- **Cost at scale.** Each URL requires a full LLM API call. For millions of URLs, this is orders of magnitude more expensive than a trained classifier or rule-based system. Best used as a second-stage filter after cheap pre-filters.
- **Label freshness.** Few-shot examples reflect phishing patterns at the time of selection. As phishing tactics evolve (new TLDs, new brand targets, new URL obfuscation techniques), examples must be refreshed.
- **No calibrated confidence scores.** The binary 0/1 output does not provide a probability. AUROC/AUPRC in the paper treat predictions as hard labels. For risk-scored pipelines, this approach needs augmentation (e.g., prompting for a confidence level).
- **Imbalanced data degradation.** At extreme imbalance (1% phishing), F1 drops to 0.53-0.66 even with few-shot prompting. The technique works best on moderately balanced or pre-filtered URL streams.

## Reference

Hasan, N. & BusiReddyGari, P. (2025). *Benchmarking Large Language Models for Zero-shot and Few-shot Phishing URL Detection.* arXiv:2602.02641. NeurIPS 2025 LAW Workshop. [https://arxiv.org/abs/2602.02641v1](https://arxiv.org/abs/2602.02641v1) — See Tables 1-3 for per-model performance under balanced and imbalanced settings, and Section 4 for the unified prompting framework specification.