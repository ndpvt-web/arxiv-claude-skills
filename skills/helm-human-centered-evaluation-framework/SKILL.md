---
name: "helm-human-centered-evaluation-framework"
description: "Evaluate LLM-powered recommender systems across five human-centered dimensions: Intent Alignment, Explanation Quality, Interaction Naturalness, Trust & Transparency, and Fairness & Diversity. Use when: 'evaluate my recommendation system', 'audit my LLM recommender for bias', 'score my chatbot recommendations', 'measure explanation quality of my recommender', 'check fairness of my recommendation engine', 'run a human-centered evaluation on my rec system'."
---

# HELM: Human-Centered Evaluation for LLM-Powered Recommenders

This skill enables Claude to apply the HELM evaluation framework to systematically assess LLM-powered recommender systems beyond accuracy metrics. Instead of only measuring hit rates and NDCG, HELM evaluates five dimensions that determine real-world user experience: whether the system understands user intent, generates faithful explanations, interacts naturally, communicates transparently, and recommends fairly. Claude can use this to design evaluation rubrics, write automated metric pipelines, audit recommendation outputs, and produce diagnostic reports that reveal trade-offs invisible to traditional metrics.

## When to Use

- When the user asks to evaluate or audit an LLM-based recommendation system (e-commerce, content, restaurants, etc.)
- When building a recommendation system and needing to design evaluation criteria beyond accuracy
- When a user wants to detect popularity bias or fairness issues in LLM-generated recommendations
- When assessing whether LLM recommendation explanations are faithful (not hallucinated)
- When comparing multiple recommender approaches (e.g., GPT-4 vs fine-tuned model vs collaborative filtering)
- When designing a human evaluation protocol for recommendation quality with expert raters
- When the user wants to measure consistency of a conversational recommender across paraphrased queries

## Key Technique

HELM decomposes evaluation into five dimensions, each with four measurable sub-constructs scored on 5-point Likert scales. The dimensions are: **Intent Alignment** (does the system understand what the user actually wants, including unstated preferences?), **Explanation Quality** (are explanations informative, personalized, faithful to actual item attributes, and actionable?), **Interaction Naturalness** (is the conversation coherent, fluent, appropriately verbose, and adaptive?), **Trust & Transparency** (does the system communicate uncertainty, stay consistent, cite sources, and acknowledge limitations?), and **Fairness & Diversity** (does it avoid popularity bias, treat demographics equitably, and maintain catalog diversity?).

The critical insight is the **accuracy-fairness-experience triangle**: LLMs like GPT-4 excel at user experience (explanation quality 4.21/5, naturalness 4.35/5) but introduce significant popularity bias (Gini 0.73 vs 0.58 for collaborative filtering). Traditional systems achieve accuracy and fairness but poor interaction quality. No system dominates all dimensions, so HELM computes an overall **Human-Centered Score (HCS)** using the geometric mean across dimensions: `HCS = (S1 * S2 * S3 * S4 * S5)^(1/5)`. The geometric mean prevents a high score in one dimension from masking failure in another — a system scoring 5.0 on naturalness but 1.0 on fairness gets HCS 2.24, not 3.4.

HELM also introduces automated verification pipelines: **faithfulness checking** (extract claimed item attributes via NER, verify against metadata — GPT-4 only achieves 81.7% accuracy despite high subjective ratings), **consistency testing** (paraphrase queries via back-translation, measure cosine similarity of response embeddings), and **fairness metrics** (Gini coefficient on recommendation frequency, catalog coverage, intra-list diversity). These automated checks catch problems that even expert raters miss.

## Step-by-Step Workflow

1. **Define the evaluation scope.** Identify the recommender system under test, its domain (e.g., movies, books, products, restaurants), the item catalog with metadata (attributes like genre, author, price, location), and the user profiles or interaction histories available.

2. **Generate evaluation scenarios across five categories.** Create test cases covering: cold-start (new user states preferences in natural language), preference refinement (multi-turn dialogues where the user iterates), contextual requests (situation-dependent like "dinner for a date tonight"), exploratory browsing (open-ended discovery), and comparison requests (asking the system to explain trade-offs between options). Aim for 30+ scenarios per category for statistical power.

3. **Collect recommendation outputs.** Run each scenario through the recommender system(s), capturing the full conversation transcript, the recommended items, and any generated explanations. Store structured outputs: `{scenario_id, user_profile, conversation_turns[], recommended_items[], explanations[]}`.

4. **Score Intent Alignment (4 sub-metrics).** For each scenario, rate on 1-5 scales: Explicit Intent Satisfaction (did it address stated requirements?), Implicit Intent Recognition (did it pick up on unstated preferences from context?), Intent Clarification Quality (did it ask good follow-up questions when ambiguous?), Goal Completion Support (did it help the user achieve their underlying objective?).

5. **Score Explanation Quality (4 sub-metrics + automated faithfulness).** Rate: Informativeness (decision-relevant detail), Personalization (tailored to this user), Faithfulness (no hallucinated attributes), Actionability (helps user decide). Then run automated faithfulness verification: extract all factual claims from explanations using NER, verify each claim against the item metadata catalog, compute `faithfulness_score = verified_claims / total_claims`.

6. **Score Interaction Naturalness (4 sub-metrics).** Rate: Dialogue Coherence (logical connection between turns), Language Fluency (grammar and naturalness), Appropriate Verbosity (response length fits the query), Conversational Adaptability (tone matches user's style). Supplement with automated coherence: compute semantic similarity between adjacent conversation turns using sentence embeddings.

7. **Score Trust & Transparency (4 sub-metrics + automated consistency).** Rate: Uncertainty Communication (does it express confidence levels?), Behavioral Consistency (same answer for same question?), Source Attribution (cites why it recommends), Limitation Acknowledgment (admits when it can't help). Run automated consistency test: paraphrase each query via back-translation (e.g., English-German-English), send paraphrases to the system, compute `consistency = mean(cosine_similarity(original_response_embedding, paraphrase_response_embedding))`.

8. **Compute Fairness & Diversity metrics (automated).** Calculate: Gini coefficient on item recommendation frequencies (0 = perfectly equal, 1 = one item gets all recommendations; healthy target < 0.60), Catalog Coverage@K (proportion of catalog items appearing in top-K recommendations across all users), Intra-List Diversity (average pairwise dissimilarity of items within each recommendation list using item feature vectors). Check for demographic parity by running identical scenarios with different user demographic signals and comparing output quality.

9. **Calculate the Human-Centered Score.** Normalize each dimension score to 0-1 range, then compute `HCS = (S_intent * S_explanation * S_interaction * S_trust * S_fairness)^(1/5)`. Report the HCS alongside the per-dimension breakdown and flag any dimension below 3.0/5.0 as a critical gap.

10. **Generate a diagnostic report with actionable recommendations.** Map each low-scoring sub-construct to a specific system improvement. For example: low faithfulness -> add retrieval-augmented generation with verified item metadata; high Gini -> add popularity debiasing in the ranking layer; low consistency -> add response caching or constrained decoding for factual queries.

## Concrete Examples

**Example 1: Evaluating a movie recommendation chatbot**

User: "I built a GPT-4-based movie recommender chatbot. Can you help me evaluate it properly beyond just accuracy?"

Approach:
1. Design 50 evaluation scenarios: 10 cold-start ("I like mind-bending sci-fi like Inception"), 15 refinement ("Actually, something less action-heavy"), 10 contextual ("Movie for a rainy Sunday afternoon with my partner"), 8 exploratory ("Surprise me with something I'd never pick myself"), 7 comparison ("Should I watch Oppenheimer or Killers of the Flower Moon?")
2. Collect chatbot outputs for all 50 scenarios
3. Score each dimension and run automated checks

Output:
```
HELM Evaluation Report — Movie Recommender v2.1
================================================

Dimension Scores (1-5 scale):
  Intent Alignment:        4.1  [EIS: 4.3, IIR: 3.8, ICQ: 4.0, GCS: 4.2]
  Explanation Quality:     3.9  [INF: 4.2, PER: 3.7, FAI: 3.4*, ACT: 4.1]
  Interaction Naturalness: 4.4  [COH: 4.5, FLU: 4.6, VER: 4.2, ADA: 4.1]
  Trust & Transparency:    3.2  [UNC: 2.8*, CON: 3.6, ATR: 3.1, LIM: 3.3]
  Fairness & Diversity:    2.9* [Gini: 0.71, Coverage: 14%, ILD: 0.39]

Human-Centered Score (HCS): 3.66 / 5.0

* = Critical gaps requiring attention

Automated Verification:
  Explanation Faithfulness: 79.3% (21% of claims unverifiable or incorrect)
  Response Consistency:     0.82 (paraphrase test, 50 query pairs)

Key Findings:
  1. CRITICAL: Fairness score below 3.0. Gini 0.71 indicates severe
     popularity bias — 80% of recommendations come from top 5% of catalog.
     Action: Add MMR-based re-ranking or popularity-penalized scoring.

  2. WARNING: Faithfulness at 79.3%. Common hallucinations: fabricated
     director credits, incorrect release years, invented plot details.
     Action: Ground explanations via RAG with verified movie metadata DB.

  3. WARNING: Uncertainty communication weak (2.8). System never hedges
     or expresses low confidence, even for obscure genres.
     Action: Add calibrated confidence signals in system prompt.
```

**Example 2: Writing an automated faithfulness checker**

User: "I need code to verify that my recommender's explanations don't hallucinate item attributes."

Approach:
1. Build a claim extraction pipeline using NER/attribute parsing
2. Verify each claim against the item metadata catalog
3. Compute faithfulness score

Output:
```python
import json
import re
from dataclasses import dataclass

@dataclass
class Claim:
    item_id: str
    attribute: str
    claimed_value: str
    status: str  # "verified", "incorrect", "unverifiable"

def extract_claims(explanation: str, item_ids: list[str]) -> list[dict]:
    """Extract factual claims from a recommendation explanation.
    Uses pattern matching for common attribute types.
    For production, replace with an LLM-based extractor."""
    patterns = {
        "genre": r"(?:genre|category|type)\s*(?:is|:)\s*([^,.]+)",
        "director": r"(?:directed by|director)\s*([^,.]+)",
        "author": r"(?:written by|author|by)\s+([^,.]+)",
        "year": r"(?:released in|from)\s*(\d{4})",
        "rating": r"(?:rated|rating)\s*([\d.]+)",
        "price": r"\$(\d+(?:\.\d{2})?)",
    }
    claims = []
    for attr, pattern in patterns.items():
        matches = re.findall(pattern, explanation, re.IGNORECASE)
        for match in matches:
            claims.append({"attribute": attr, "claimed_value": match.strip()})
    return claims

def verify_claims(
    claims: list[dict], item_metadata: dict
) -> list[Claim]:
    """Verify extracted claims against ground-truth item metadata."""
    results = []
    for claim in claims:
        attr = claim["attribute"]
        claimed = claim["claimed_value"].lower()
        if attr in item_metadata:
            actual = str(item_metadata[attr]).lower()
            if claimed in actual or actual in claimed:
                status = "verified"
            else:
                status = "incorrect"
        else:
            status = "unverifiable"
        results.append(Claim(
            item_id=item_metadata.get("id", "unknown"),
            attribute=attr,
            claimed_value=claim["claimed_value"],
            status=status,
        ))
    return results

def compute_faithfulness(results: list[Claim]) -> float:
    """Faithfulness = verified / (verified + incorrect).
    Unverifiable claims are excluded from the denominator."""
    verifiable = [r for r in results if r.status != "unverifiable"]
    if not verifiable:
        return 1.0
    verified = sum(1 for r in verifiable if r.status == "verified")
    return verified / len(verifiable)
```

**Example 3: Comparing two recommender systems**

User: "I'm deciding between a fine-tuned LLaMA model and a prompted GPT-4 approach for my book recommender. How do I compare them properly?"

Approach:
1. Create a shared scenario bank (same 100 scenarios for both systems)
2. Run both systems on all scenarios
3. Score both on all five HELM dimensions
4. Compute HCS and generate a comparison matrix

Output:
```
HELM Comparative Report — Book Recommender Candidates
======================================================

                        GPT-4 Prompted   LLaMA-3.1 Fine-tuned
Intent Alignment            4.2              3.7
Explanation Quality         4.3              3.8
Interaction Naturalness     4.4              3.5
Trust & Transparency        3.8              3.4
Fairness & Diversity        3.0              3.5
-------------------------------------------------------
HCS (geometric mean)        3.91             3.57

Automated Metrics:
  Hit Rate@10               0.29             0.34
  Faithfulness              81.2%            76.8%
  Consistency               0.84             0.91
  Gini Coefficient          0.72             0.65
  Coverage@100              13.1%            19.8%
  Intra-List Diversity      0.41             0.48

Trade-off Analysis:
  GPT-4 wins on user experience (naturalness +0.9, explanation +0.5)
  but loses on fairness (Gini +0.07 worse) and accuracy (Hit Rate -0.05).

  LLaMA wins on fairness, diversity, consistency, and raw accuracy
  but produces less natural, less explanatory interactions.

Recommendation:
  If user experience is primary: GPT-4 with post-hoc fairness re-ranking.
  If fairness/cost is primary: LLaMA with explanation quality fine-tuning.
  Hybrid option: Use LLaMA for candidate generation + GPT-4 for
  explanation generation on the final ranked list.
```

## Best Practices

- **Do:** Always report per-dimension scores alongside HCS. A single aggregate score hides the accuracy-fairness-experience trade-off that is the most important diagnostic finding.
- **Do:** Run automated faithfulness checks even when using expert raters. The paper found that expert ratings overestimate faithfulness — GPT-4 received 4.02/5 from experts but only 81.7% automated accuracy. Humans are bad at catching plausible-sounding hallucinations.
- **Do:** Use the geometric mean for HCS, not the arithmetic mean. The geometric mean penalizes systems that collapse on any single dimension, which is the correct behavior (a system with zero fairness should not score well overall).
- **Do:** Test consistency with paraphrased queries. Back-translate queries through another language (e.g., English-German-English) to generate natural paraphrases, then measure response similarity.
- **Avoid:** Evaluating only on accuracy metrics (Hit Rate, NDCG). The paper shows that NCF+Template achieves comparable Hit Rate@10 (0.312) to GPT-4 (0.298) but scores dramatically lower on HCS (3.08 vs 3.91), proving accuracy alone is misleading.
- **Avoid:** Assuming LLM-powered systems are automatically fairer. All LLM recommenders in the study showed higher popularity bias (Gini 0.61-0.73) than traditional collaborative filtering (0.58). LLMs amplify popularity bias from training data.

## Error Handling

- **Insufficient item metadata for faithfulness verification:** If the item catalog lacks structured attributes, faithfulness checking degrades. Fall back to LLM-as-judge verification (prompt a separate LLM to fact-check claims against item descriptions), but note this introduces its own error rate. Flag the reduced reliability in the report.
- **Too few evaluation scenarios for statistical significance:** Below 30 scenarios per category, sub-metric scores will have wide confidence intervals. Compute and report 95% confidence intervals on all scores. If intervals overlap between systems, do not claim a winner on that dimension.
- **Low inter-rater agreement:** If using multiple human raters, compute Fleiss' kappa or ICC before trusting scores. If agreement falls below 0.70, the rubric needs clarification or raters need recalibration. The HELM authors achieved ICC 0.78-0.87 after a 2-hour training session.
- **Domain-specific scoring difficulty:** The paper found restaurants harder to evaluate than movies (HCS 3.64 vs 4.12 for GPT-4) due to location/time dependencies. For real-time-dependent domains, include freshness and context-relevance as additional sub-metrics under Intent Alignment.

## Limitations

- HELM requires structured item metadata for faithfulness verification. Systems recommending items without a clean attribute catalog (e.g., user-generated content) cannot use automated faithfulness checking.
- The five dimensions and 20 sub-constructs are designed for recommendation systems specifically. Applying HELM to general-purpose chatbots or non-recommendation LLM tasks would require adapting the rubric.
- Expert evaluation is expensive. The original study used 12 domain experts across 847 scenarios. For rapid iteration, rely on the automated metrics (faithfulness, consistency, Gini, coverage, ILD) and reserve full expert evaluation for major releases.
- The geometric mean HCS penalizes dimension collapse but assumes all five dimensions are equally important. In practice, some applications may weight user experience over fairness or vice versa. Consider reporting a weighted HCS alongside the unweighted version when stakeholders have explicit priorities.
- HELM does not measure long-term user satisfaction, retention, or behavioral change. It evaluates snapshot quality of individual recommendation sessions, not longitudinal engagement.

## Reference

**Paper:** Mehta, S. (2026). "HELM: A Human-Centered Evaluation Framework for LLM-Powered Recommender Systems." arXiv:2601.19197v1. https://arxiv.org/abs/2601.19197v1

Look for: Table 2 (per-dimension scores by model and domain), Table 3 (automated fairness metrics), the faithfulness verification algorithm in Section 4.3, and the accuracy-fairness-experience triangle analysis in Section 5.2.