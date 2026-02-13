---
name: "pamas-self-adaptive-multi-agent-system"
description: "Build hierarchical multi-agent systems that detect misinformation, anomalies, and deceptive content using perspective-aware aggregation. Agents are split into specialized Auditors, Coordinators, and a Decision-Maker to prevent information drowning. Use when: 'detect misinformation with agents', 'build a multi-agent fact checker', 'anomaly detection with LLM agents', 'hierarchical agent system for content verification', 'PAMAS framework', 'perspective-aware multi-agent system'."
---

# PAMAS: Self-Adaptive Multi-Agent System with Perspective Aggregation

This skill enables Claude to build hierarchical multi-agent systems based on the PAMAS framework for detecting misinformation, fake reviews, deceptive content, and anomalies. The core innovation is **perspective-aware aggregation**: instead of giving every agent the full input (which causes truthful signals to drown out sparse deceptive cues), each agent examines only a specialized feature subset. A three-tier hierarchy of Auditors, Coordinators, and a Decision-Maker then aggregates these partial perspectives while preserving minority anomaly signals that flat voting or debate systems would suppress.

## When to Use

- When the user asks to build a multi-agent system for misinformation or fake news detection
- When the user wants to classify content (reviews, posts, claims) as truthful or deceptive using LLM agents
- When building anomaly detection pipelines where the signal-to-noise ratio is low (sparse deceptive cues in mostly truthful data)
- When the user wants a hierarchical agent architecture with specialized roles rather than flat debate or voting
- When implementing a scalable agent framework that prunes redundant agents and routes inference adaptively
- When the user needs explainable, traceable decisions from a multi-agent system (each tier provides auditable rationale)

## Key Technique

**The Information-Drowning Problem.** In standard multi-agent systems, every agent sees the complete input. When content is mostly truthful with only subtle deceptive cues, agents converge on the dominant "truthful" pattern. Inter-agent communication (debate, voting) then amplifies this convergence, suppressing the weak anomaly signals that matter most for detection. This is analogous to how a committee of reviewers who all read the same report will tend to agree on the obvious conclusion, missing buried inconsistencies.

**Perspective-Aware Decomposition and Hierarchical Aggregation.** PAMAS solves this by decomposing each input into ~20 characteristic dimensions (statistical features like rating variance, frequency patterns, length distributions; and semantic features like sentiment polarity, emotional intensity, readability). Each Auditor agent receives a distinct profile restricting it to a small subset of these dimensions. This forced specialization means different Auditors literally cannot see the same dominant patterns, so minority anomaly cues get amplified rather than suppressed. Coordinators then aggregate Auditor outputs using weighted voting with evolving trust weights, preserving diversity across groups. The Decision-Maker at the top holds full context plus an evolving memory (confidence weights, experience patterns from past errors, and action logs), synthesizing Coordinator outputs with its own direct judgment.

**Self-Adaptive Topology and Routing.** To prevent the agent count from ballooning, PAMAS prunes agents whose contributions are redundant (high cosine similarity to peers, low marginal accuracy gain) and expands with complementary agents when coverage gaps exist. At inference time, confidence-guided routing activates only the top-2 most trusted subordinates first; if they agree, the decision finalizes without activating the full tree. This cuts token consumption dramatically while maintaining accuracy.

## Step-by-Step Workflow

1. **Decompose the input into feature dimensions.** Given content to verify (a social media post, product review, news claim), extract ~10-20 characteristic dimensions split into two categories: (a) statistical features (e.g., posting frequency, rating distribution, text length variance, metadata patterns) and (b) semantic features (e.g., sentiment polarity, emotional intensity, logical consistency, readability score, source credibility signals). Represent each dimension as a named key-value pair.

2. **Design Auditor profiles with complementary feature subsets.** Create 4-8 Auditor agents, each assigned a non-overlapping or minimally-overlapping subset of 2-4 feature dimensions. For example: Auditor-A gets {sentiment_polarity, emotional_intensity}, Auditor-B gets {rating_variance, frequency_pattern}, Auditor-C gets {readability, logical_consistency}. Each Auditor's system prompt restricts its analysis to ONLY its assigned dimensions.

3. **Run each Auditor on its restricted view.** For each Auditor, construct a prompt containing only its assigned feature values (not the full input). The Auditor outputs a binary decision (truthful/deceptive) plus a short reasoning string explaining which cues it flagged. Store the tuple `(decision, reasoning, auditor_id)`.

4. **Group Auditors under Coordinators.** Organize Auditors into 2-3 groups of 2-4 Auditors each. Each group is managed by a Coordinator agent. Grouping should maximize intra-group diversity (Auditors in the same group should cover different feature categories).

5. **Aggregate at the Coordinator level using weighted voting.** Each Coordinator collects decisions from its Auditors and applies: `d_C = indicator(sum(w_j * (2*d_j - 1)) > 0)` where `w_j` is the trust weight for Auditor j. Initialize weights uniformly. The Coordinator outputs its aggregated decision, the vote distribution, and a summary of the strongest anomaly cues raised by its Auditors.

6. **Synthesize at the Decision-Maker level.** The Decision-Maker receives: (a) all Coordinator decisions with rationales, (b) the full original input for direct assessment, and (c) its evolving memory (past error patterns, trust weights for Coordinators). It produces the final verdict with a confidence score and a traceable explanation chain.

7. **Update memory on labeled feedback.** When ground truth is available, update: (a) confidence weights toward each subordinate using exponential moving average: `w[j] = (1-alpha)*w[j] + alpha*(reflection_score + long_term_accuracy)`, and (b) experience memory by storing self-reflections from misclassified cases only (correct cases do not update, reducing noise).

8. **Prune and expand the topology periodically.** After a batch of evaluations, compute each Auditor's marginal contribution: `Score_i = (accuracy_with_i - accuracy_without_i) - lambda * avg_cosine_similarity_to_peers`. Remove agents with negative scores. For expansion, evaluate candidate profiles: `Gain_i = delta_accuracy_with_i - gamma * avg_cosine_similarity_to_peers`. Add high-gain candidates.

9. **Enable confidence-guided routing for efficiency.** At inference time, the Decision-Maker activates only its top-2 trusted Coordinators. If they agree, finalize immediately. If they disagree, progressively activate additional Coordinators in descending trust order until a non-tie majority emerges. Apply the same logic at the Coordinator level toward its Auditors.

10. **Return the structured verdict.** Output a JSON object containing: `verdict` (truthful/deceptive), `confidence` (0-1), `explanation` (Decision-Maker rationale), `coordinator_summaries` (array of Coordinator-level reasoning), and `auditor_flags` (array of specific anomaly cues raised by individual Auditors).

## Concrete Examples

**Example 1: Fake Review Detection**

User: "Build a multi-agent system to detect fake product reviews. I have review text, star rating, reviewer history, and product metadata."

Approach:
1. Define feature dimensions: {sentiment_polarity, emotional_intensity} (semantic), {rating_deviation_from_mean, review_length_vs_product_avg, reviewer_posting_frequency, reviewer_rating_variance} (statistical), {readability_score, specificity_of_claims} (semantic).
2. Create 4 Auditors:
   - Auditor-A (Sentiment Specialist): sees sentiment_polarity, emotional_intensity
   - Auditor-B (Behavioral Analyst): sees reviewer_posting_frequency, reviewer_rating_variance
   - Auditor-C (Statistical Outlier): sees rating_deviation_from_mean, review_length_vs_product_avg
   - Auditor-D (Linguistic Analyst): sees readability_score, specificity_of_claims
3. Group under 2 Coordinators: Coordinator-1 manages {A, B}, Coordinator-2 manages {C, D}.
4. Decision-Maker sees full review text + Coordinator summaries.

Output:
```json
{
  "verdict": "deceptive",
  "confidence": 0.87,
  "explanation": "Coordinator-1 flagged emotional manipulation (Auditor-A: extreme positive sentiment inconsistent with 3-star rating) combined with suspicious posting pattern (Auditor-B: 12 reviews in 24 hours). Coordinator-2 noted statistical anomaly (Auditor-C: rating deviates 2.3 SD from product mean) but normal linguistic quality. The cross-coordinator disagreement on linguistic features was resolved by direct analysis showing template-like phrasing.",
  "coordinator_summaries": [
    {"coordinator": 1, "decision": "deceptive", "key_cue": "sentiment-rating mismatch + burst posting"},
    {"coordinator": 2, "decision": "deceptive", "key_cue": "statistical rating outlier"}
  ],
  "auditor_flags": [
    {"auditor": "A", "flag": "sentiment_polarity=0.92 but star_rating=3, mismatch > 0.5 threshold"},
    {"auditor": "B", "flag": "12 reviews in 24h, typical user posts 1.2/week"},
    {"auditor": "C", "flag": "rating deviation 2.3 SD from product mean"},
    {"auditor": "D", "flag": "no anomaly detected, readability normal"}
  ]
}
```

**Example 2: Political Misinformation Detection**

User: "I need to fact-check political claims from social media posts. I have the claim text, source account info, engagement metrics, and related fact-check articles."

Approach:
1. Feature dimensions: {claim_specificity, hedging_language, emotional_appeal} (semantic), {account_age, follower_ratio, posting_frequency} (statistical), {engagement_anomaly, share_to_like_ratio} (statistical), {consistency_with_factchecks, source_credibility_score} (semantic).
2. Create 5 Auditors across these dimension groups.
3. Group under 2 Coordinators: one for account/engagement signals, one for content/credibility signals.
4. Decision-Maker reconciles with full claim text and memory of past misinformation patterns.

Output:
```json
{
  "verdict": "likely_misinformation",
  "confidence": 0.79,
  "explanation": "Content analysis shows high emotional appeal with low specificity (classic misinformation pattern from experience memory). Account signals are borderline (new account but organic follower growth). Engagement pattern is anomalous (share:like ratio 4.2x normal). Two existing fact-checks contradict the central claim.",
  "routing_info": {
    "agents_activated": 4,
    "agents_total": 7,
    "tokens_saved_pct": 43
  }
}
```

**Example 3: Implementing the Adaptive Topology**

User: "My PAMAS system has 8 Auditors but inference is slow. How do I prune redundant ones?"

Approach:
1. Collect decision vectors from all Auditors over a validation set of 50+ labeled examples.
2. Compute pairwise cosine similarity between Auditor decision vectors.
3. For each Auditor i under Coordinator C, compute:
   ```
   Score_i = (Acc(group_with_i) - Acc(group_without_i)) - 0.3 * mean(cosine_sim(i, peers))
   ```
4. Rank by Score. Remove any Auditor with Score < 0.
5. After pruning, check if any feature dimensions are now uncovered. If so, create a new candidate Auditor for those dimensions and compute Gain before adding.

Output:
```
Pruning results:
  Auditor-3: Score = -0.02 (high similarity to Auditor-1, marginal accuracy gain 0.1%) -> REMOVED
  Auditor-7: Score = -0.05 (zero marginal accuracy contribution) -> REMOVED
  Remaining: 6 Auditors, accuracy unchanged (96.2%), inference cost reduced 25%

Expansion check:
  Uncovered dimension: engagement_velocity (was shared between Auditor-3 and Auditor-5)
  Candidate Auditor-9 (engagement_velocity + temporal_pattern): Gain = 0.08 -> ADDED
  Final: 7 Auditors, accuracy 96.4%, net cost reduction 12%
```

## Best Practices

- **Do:** Assign each Auditor a genuinely distinct feature subset. The entire framework's value depends on forced perspective diversity. Overlapping subsets recreate the information-drowning problem.
- **Do:** Initialize Coordinator trust weights uniformly and let them evolve from labeled feedback. Premature weighting biases the system before it has evidence.
- **Do:** Update experience memory only on misclassified examples. Correct cases add noise; errors are where the system learns its blind spots.
- **Do:** Use confidence-guided routing in production to reduce cost. Full hierarchy traversal is only needed during training/evaluation phases.
- **Avoid:** Giving any Auditor access to the full input. The moment an Auditor sees everything, it reverts to the dominant-pattern convergence problem.
- **Avoid:** Using simple majority voting without trust weights at the Coordinator level. Uniform voting loses the adaptive benefit of learning which Auditors are reliable for which content types.

## Error Handling

- **All Auditors agree "truthful" but ground truth is deceptive:** This indicates the feature decomposition missed the relevant deceptive dimension. Add the missed dimension as a new Auditor profile and retrain trust weights. Check if the Decision-Maker's direct full-context analysis also missed it (if so, the cue may be genuinely novel).
- **Coordinators deadlock (equal weighted votes):** Activate all remaining Auditors under that Coordinator (disable routing). If still tied, escalate the full Auditor reasoning to the Decision-Maker for direct resolution.
- **Confidence-guided routing keeps activating all agents:** Trust weights may have converged to near-uniform values. Re-evaluate with a larger labeled batch, or manually inspect whether two Coordinators are covering near-identical perspectives (a sign that topology pruning is needed).
- **Memory bloat in Decision-Maker:** Cap experience memory at 100-200 entries. Use LLM-based summarization to distill older entries into general patterns before evicting them.

## Limitations

- **Requires meaningful feature decomposition.** If the content domain lacks clearly separable dimensions (e.g., very short text with no metadata), the perspective-splitting advantage diminishes. The framework works best when inputs are feature-rich.
- **Labeled data needed for adaptation.** Trust weight evolution, topology pruning, and experience memory all depend on ground-truth feedback. In zero-shot scenarios, the system still works via initial uniform weights but cannot self-optimize.
- **Token cost scales with hierarchy depth.** Even with routing optimizations, a 3-tier hierarchy with 8+ Auditors requires multiple LLM calls per input. For high-throughput, low-latency applications, consider reducing to a 2-tier architecture (Auditors + Decision-Maker).
- **Domain-specific feature engineering.** The ~20 characteristic dimensions must be defined per domain (reviews vs. news vs. social posts). There is no universal feature set; each deployment requires domain analysis.
- **Not suited for real-time single-item classification** at sub-second latency due to multi-agent overhead. Best for batch processing or scenarios where accuracy outweighs speed.

## Reference

- **Paper:** [PAMAS: Self-Adaptive Multi-Agent System with Perspective Aggregation for Misinformation Detection](https://arxiv.org/abs/2602.03158v1) (Wang et al., 2026)
- **Key insight to look for:** Section 3's formalization of the information-drowning problem and how hierarchical perspective assignment with weighted aggregation preserves minority anomaly signals that flat MAS architectures suppress. Table 1 shows 97.15% accuracy on Amazon, 96.28% on DeRev2018, and 96.43% on PolitiFact, outperforming DyLAN and Layer baselines by 2-4 points while using fewer tokens.