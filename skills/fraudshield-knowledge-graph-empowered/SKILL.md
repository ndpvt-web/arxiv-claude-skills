---
name: "fraudshield-knowledge-graph-empowered"
description: "Detect and defend against fraudulent content in LLM inputs using knowledge-graph-augmented analysis. Builds a fraud tactic-keyword bipartite graph, scores associations by confidence, prunes ambiguities, and augments prompts with XML-tagged keywords plus evidence rationales. Use when: 'check this email for fraud', 'is this job posting a scam', 'analyze this contract for manipulation', 'detect phishing in this message', 'flag suspicious text in this document', 'add fraud detection to my LLM pipeline'."
---

# FraudShield: Knowledge Graph Empowered Fraud Defense for LLM Inputs

This skill enables Claude to detect and defend against fraudulent content embedded in text that will be processed by LLMs. It implements the FraudShield framework (WWW 2026), which constructs a weighted bipartite knowledge graph linking suspicious keywords to four canonical fraud tactics, prunes low-confidence and ambiguous associations, then augments the original input with XML-tagged keywords and supporting evidence. This structured augmentation guides any downstream LLM toward fraud-aware, interpretable responses rather than blindly trusting manipulative content.

## When to Use

- When the user asks to analyze an email, message, or document for scam indicators or fraudulent intent
- When building an automated pipeline (contract review, job application screening, customer support) that needs fraud resilience
- When the user wants to understand *why* specific text is suspicious, not just whether it is
- When reviewing job postings, service offers, relationship messages, or phishing attempts for red flags
- When implementing a defense layer that wraps LLM inputs with fraud-detection metadata before inference
- When the user asks to build a content moderation or trust-and-safety filter for text inputs

## Key Technique

FraudShield categorizes fraud manipulation into four canonical tactics: **Urgency Pressure** (time constraints, scarcity, fear-based imperatives), **Suspicious Information** (malformed URLs, unrealistic offers, geographic anomalies), **Sensitive Requests** (credential harvesting, disguised verification, data inconsistencies), and **Credibility Claims** (authority assertions, professional jargon, real-event name-dropping). Every piece of fraudulent text employs one or more of these tactics, and the framework's power comes from systematically mapping keywords to them.

The core innovation is a **weighted bipartite graph** connecting keyword clusters to fraud tactics. Keywords are extracted from the input and scored 0-10 for association strength with each tactic, along with a rationale explaining the association. Overlapping keywords are clustered (e.g., "jdfinance.cn" and "jdfinance" collapse into one cluster). Edges with confidence below a threshold (default tau=5) are pruned, and each keyword cluster is disambiguated to its single strongest tactic. This eliminates false positives on benign text while retaining high-signal associations.

The refined graph then drives **XML-tagged augmentation**: detected keywords are wrapped in tactic-labeled XML tags (e.g., `<Urgency Pressure>Act now</Urgency Pressure>`) and the strongest rationale per tactic is appended as evidence. The augmented input replaces the original before the LLM processes it. This approach is model-agnostic, interpretable (users see exactly which words triggered which tactic and why), and generalizes across fraud types including phishing, impersonation, fake job postings, fraudulent services, and romance scams.

## Step-by-Step Workflow

1. **Receive and normalize the input text.** Strip formatting artifacts but preserve URLs, email addresses, phone numbers, and domain names exactly as they appear -- these are high-signal features for Suspicious Information detection.

2. **Extract keyword-tactic-score-rationale quadruplets.** For each of the four fraud tactics (Urgency Pressure, Suspicious Information, Sensitive Requests, Credibility Claims), identify keywords in the text that associate with that tactic. Assign each keyword a confidence score from 0-10 and write a one-sentence rationale. Output structured data: `{tactic, keyword, score, rationale}`.

3. **Cluster overlapping keywords.** Group keywords where one is a substring of another (e.g., "verify your account" and "verify" merge into one cluster anchored by the longer phrase). This prevents double-counting and cleans up the association map.

4. **Construct the weighted bipartite graph.** Create nodes for each keyword cluster and each tactic. Add edges weighted by the average confidence score across all keywords in the cluster for that tactic.

5. **Prune low-confidence edges.** Remove any edge with weight below the threshold tau=5. This eliminates weak, noise-driven associations that would cause false positives on legitimate text.

6. **Disambiguate multi-tactic keywords.** For each keyword cluster that still connects to multiple tactics, retain only the edge with the highest weight. Each keyword cluster maps to exactly one tactic after this step.

7. **Select evidence rationales.** For each tactic that survived pruning, select the rationale with the highest confidence score as the supporting evidence for that tactic.

8. **Augment the input with XML tags.** Replace each detected keyword in the original text with `<Tactic Name>keyword</Tactic Name>`. Append the evidence rationales as a structured block after the augmented text.

9. **Generate the fraud-aware response.** Pass the augmented input to the LLM (or produce the analysis directly). The XML tags and evidence guide the model to treat flagged regions with appropriate skepticism.

10. **Present interpretable results.** Return the original text with highlighted fraud indicators, a summary of detected tactics, the confidence-weighted keyword map, and the evidence rationales. This gives the user both the verdict and the reasoning chain.

## Concrete Examples

**Example 1: Phishing Email Analysis**

User: "Check this email for fraud: 'Dear valued customer, your account has been compromised. Click http://paypa1-secure.xyz/verify to verify your identity within 24 hours or your account will be permanently locked. PayPal Security Team.'"

Approach:
1. Extract keywords and score against four tactics:
   - "within 24 hours", "permanently locked" -> Urgency Pressure (score: 9, 8)
   - "http://paypa1-secure.xyz/verify" -> Suspicious Information (score: 10, rationale: domain uses digit-substitution "paypa1" mimicking "paypal" with a non-standard TLD)
   - "verify your identity" -> Sensitive Requests (score: 8, rationale: solicits identity verification through external link)
   - "PayPal Security Team" -> Credibility Claims (score: 7, rationale: impersonates authority of legitimate financial institution)
2. Cluster, prune (all above tau=5), disambiguate
3. Produce augmented text and evidence

Output:
```
FRAUD ANALYSIS RESULT
=====================
Risk Level: HIGH (4/4 tactics detected)

Augmented Text:
"Dear valued customer, your account has been compromised. Click
<Suspicious Information>http://paypa1-secure.xyz/verify</Suspicious Information>
to <Sensitive Requests>verify your identity</Sensitive Requests>
<Urgency Pressure>within 24 hours</Urgency Pressure> or your account will be
<Urgency Pressure>permanently locked</Urgency Pressure>.
<Credibility Claims>PayPal Security Team</Credibility Claims>."

Detected Tactics & Evidence:
1. Suspicious Information (confidence: 10/10)
   Keyword: "http://paypa1-secure.xyz/verify"
   Evidence: Domain uses digit-substitution ("paypa1" for "paypal") with
   non-standard .xyz TLD -- classic typosquatting pattern.

2. Urgency Pressure (confidence: 9/10)
   Keywords: "within 24 hours", "permanently locked"
   Evidence: Artificial time constraint combined with threat of permanent
   loss creates panic to bypass rational evaluation.

3. Sensitive Requests (confidence: 8/10)
   Keyword: "verify your identity"
   Evidence: Directs user to provide identity credentials through an
   external, suspicious link rather than in-app verification.

4. Credibility Claims (confidence: 7/10)
   Keyword: "PayPal Security Team"
   Evidence: Impersonates a legitimate financial institution's security
   department to establish false authority.
```

**Example 2: Fake Job Posting Detection**

User: "Is this job post legit? 'URGENT HIRING: Remote Data Entry, $85/hr, no experience needed. Send your SSN and bank details to hr@careers-g00gle.net for direct deposit setup. Google LLC.'"

Approach:
1. Extract and score keywords against tactics
2. Build graph, prune below tau=5, disambiguate
3. Tag and present findings

Output:
```
FRAUD ANALYSIS RESULT
=====================
Risk Level: HIGH (4/4 tactics detected)

Detected Tactics & Evidence:
1. Suspicious Information (confidence: 10/10)
   Keywords: "$85/hr, no experience needed", "hr@careers-g00gle.net"
   Evidence: Unrealistic compensation for zero-qualification role; email
   domain uses digit substitution ("g00gle") mimicking Google.

2. Sensitive Requests (confidence: 9/10)
   Keywords: "SSN", "bank details"
   Evidence: Requests Social Security Number and banking information
   before any hiring process -- legitimate employers never collect
   these during initial application.

3. Urgency Pressure (confidence: 8/10)
   Keyword: "URGENT HIRING"
   Evidence: Creates false urgency to discourage due diligence.

4. Credibility Claims (confidence: 7/10)
   Keyword: "Google LLC"
   Evidence: Impersonates a major tech company but uses a non-Google
   email domain for communication.

Recommendation: Do not respond. Report the posting to the platform.
```

**Example 3: Building a Fraud Defense Pipeline**

User: "I'm building a contract review tool. How do I add FraudShield-style defense to my LLM pipeline?"

Approach:
1. Define the four-tactic taxonomy as a reusable schema
2. Implement keyword extraction as a pre-processing step
3. Build the bipartite graph with pruning logic
4. Output augmented prompts for the downstream LLM

Output:
```python
# FraudShield defense layer for LLM pipelines

FRAUD_TACTICS = [
    "Urgency Pressure",
    "Suspicious Information",
    "Sensitive Requests",
    "Credibility Claims",
]

def extract_fraud_keywords(text: str, llm_client) -> list[dict]:
    """Stage 1: Extract keyword-tactic associations with confidence scores."""
    prompt = f"""Analyze the following text for fraud indicators.
For each of the four tactics ({', '.join(FRAUD_TACTICS)}), identify
keywords that associate with that tactic. For each keyword, provide:
- keyword: the exact text span
- tactic: which fraud tactic it maps to
- score: confidence 0-10 that this keyword indicates this tactic
- rationale: one sentence explaining why

Return as JSON array. Text to analyze:
{text}"""
    return llm_client.parse_json(llm_client.complete(prompt))

def cluster_keywords(associations: list[dict]) -> dict[str, list[dict]]:
    """Stage 2a: Cluster overlapping keywords (substring merging)."""
    clusters = {}
    sorted_kws = sorted(associations, key=lambda x: len(x["keyword"]), reverse=True)
    for assoc in sorted_kws:
        merged = False
        for anchor in clusters:
            if assoc["keyword"] in anchor or anchor in assoc["keyword"]:
                clusters[anchor].append(assoc)
                merged = True
                break
        if not merged:
            clusters[assoc["keyword"]] = [assoc]
    return clusters

def build_and_prune_graph(clusters: dict, tau: float = 5.0) -> list[dict]:
    """Stage 2b-2c: Build bipartite graph, prune, disambiguate."""
    refined = []
    for anchor, assocs in clusters.items():
        # Group by tactic, compute average score
        tactic_scores = {}
        for a in assocs:
            tactic_scores.setdefault(a["tactic"], []).append(a["score"])
        tactic_avgs = {t: sum(s)/len(s) for t, s in tactic_scores.items()}
        # Prune below threshold
        tactic_avgs = {t: s for t, s in tactic_avgs.items() if s >= tau}
        if not tactic_avgs:
            continue
        # Disambiguate: keep only highest-scoring tactic
        best_tactic = max(tactic_avgs, key=tactic_avgs.get)
        best_rationale = max(
            [a for a in assocs if a["tactic"] == best_tactic],
            key=lambda x: x["score"]
        )["rationale"]
        refined.append({
            "keyword": anchor,
            "tactic": best_tactic,
            "score": tactic_avgs[best_tactic],
            "rationale": best_rationale,
        })
    return refined

def augment_input(text: str, refined: list[dict]) -> str:
    """Stage 3-4: XML-tag keywords and append evidence."""
    augmented = text
    # Sort by keyword length descending to avoid partial replacements
    for item in sorted(refined, key=lambda x: len(x["keyword"]), reverse=True):
        tag = item["tactic"]
        augmented = augmented.replace(
            item["keyword"],
            f"<{tag}>{item['keyword']}</{tag}>"
        )
    # Append evidence block
    evidence = "\n\nFRAUD EVIDENCE:\n"
    for item in refined:
        evidence += f"- {item['tactic']} ({item['score']:.1f}/10): {item['rationale']}\n"
    return augmented + evidence

def fraudshield_defend(text: str, llm_client, tau: float = 5.0) -> str:
    """Full FraudShield pipeline: extract -> cluster -> prune -> augment."""
    associations = extract_fraud_keywords(text, llm_client)
    clusters = cluster_keywords(associations)
    refined = build_and_prune_graph(clusters, tau)
    if not refined:
        return text  # No fraud signals detected; pass through unchanged
    return augment_input(text, refined)
```

## Best Practices

- **Do:** Always score keywords against all four tactics before pruning. A keyword may appear benign under one tactic but score highly under another. The bipartite graph structure prevents premature filtering.
- **Do:** Preserve exact original text spans when extracting keywords. Fraud indicators often depend on precise character sequences (e.g., "paypa1" vs "paypal", "g00gle" vs "google"). Normalizing or lowercasing before extraction destroys signal.
- **Do:** Set tau conservatively (5/10 default) when analyzing general-purpose text. For high-stakes domains like financial contracts, lower tau to 3-4 to increase recall at the cost of more flagged benign text.
- **Do:** Present evidence rationales alongside highlighted keywords. The interpretability of FraudShield is its differentiator over black-box classifiers -- always surface the "why."
- **Avoid:** Skipping the disambiguation step. Without it, a keyword like "verify" may be tagged under both Sensitive Requests and Urgency Pressure, creating noise and confusing downstream models.
- **Avoid:** Applying FraudShield to text the user explicitly authored themselves. The framework is designed for analyzing *incoming* content (emails, job postings, contracts from third parties), not for second-guessing the user's own writing.

## Error Handling

- **No keywords extracted:** If the extraction step returns zero associations, the text is likely benign. Pass it through unchanged rather than forcing false positives. Report "No fraud indicators detected" to the user.
- **All edges pruned:** If every keyword-tactic edge falls below tau, the text contains only weak, ambiguous signals. Report the findings with a "low confidence" qualifier rather than a definitive fraud verdict.
- **Keyword overlap collisions:** If XML-tag insertion produces malformed nesting (a tagged keyword contains another tagged keyword), resolve by keeping only the longer keyword's tag. Sort replacements by keyword length descending to prevent this.
- **Adversarial evasion:** Sophisticated fraud may use homoglyphs, zero-width characters, or heavy paraphrasing to evade keyword matching. When the text "feels" manipulative but extraction yields low scores, flag this as a potential evasion attempt and recommend manual review.
- **Ambiguous tau tuning:** If users report too many false positives, raise tau. If fraud slips through, lower it. Provide the raw scores so users can make informed threshold decisions.

## Limitations

- The four-tactic taxonomy (Urgency Pressure, Suspicious Information, Sensitive Requests, Credibility Claims) covers the most common fraud patterns but may miss novel social engineering techniques that don't fit these categories (e.g., long-form emotional manipulation without urgency).
- Keyword-based detection is inherently bypassable by sufficiently creative paraphrasing. FraudShield reduces but does not eliminate this risk through the clustering and confidence-scoring layers.
- The framework is designed for text-based fraud. It does not analyze images, voice, video, or multi-modal content.
- Performance depends on the quality of the keyword extraction LLM call. If the extraction model itself is weak or manipulated, downstream steps inherit those errors.
- The approach adds latency (one extra LLM call for extraction plus graph processing) which may matter in real-time, high-throughput pipelines. Consider caching the knowledge graph for repeated similar inputs.

## Reference

**Paper:** [FraudShield: Knowledge Graph Empowered Defense for LLMs against Fraud Attacks](https://arxiv.org/abs/2601.22485v1) (WWW 2026) -- See Section 3 for the full four-stage pipeline (extraction, graph construction/pruning, evidence selection, XML augmentation) and Section 4 for evaluation across five fraud types with four LLMs.