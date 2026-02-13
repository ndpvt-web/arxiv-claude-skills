---
name: "jaf-judge-agent-forest"
description: |
  Implement the Judge Agent Forest (JAF) pattern: evaluate and refine AI-generated outputs by judging cohorts of related query-response pairs together rather than in isolation, using peer exemplars to surface cross-instance patterns and inconsistencies.
  Use this skill when the user says:
  - "Evaluate these outputs together to find inconsistencies"
  - "Use a judge agent forest to refine these results"
  - "Cross-check these responses against each other"
  - "Set up cohort-based evaluation for my agent outputs"
  - "Find patterns across these related classifications"
  - "Build a judge that learns from peer examples"
---

# JAF: Judge Agent Forest

This skill enables Claude to apply the Judge Agent Forest framework — a technique where instead of evaluating each AI-generated response independently, a judge examines **cohorts** of related query-response pairs simultaneously. By seeing how a primary agent handled similar problems, the judge detects cross-instance inconsistencies, surfaces hidden patterns, and produces feedback that is far more calibrated than isolated evaluation. This turns the judge from a point-scorer into a holistic critic that reasons about an agent's behavior across its full output distribution.

## When to Use

- When the user has a batch of related items to classify, triage, or score and wants higher-quality evaluations than one-at-a-time review provides
- When a primary agent produces outputs that need systematic quality control (e.g., code reviews across a PR set, security finding triage, support ticket categorization)
- When the user asks to detect inconsistencies across a set of related AI-generated outputs (e.g., "why did the model flag this config but not that nearly identical one?")
- When building an evaluation harness for an LLM pipeline and wanting ensemble-style robustness without training multiple models
- When the user needs to triage cloud misconfigurations, vulnerability reports, or any domain where severity ratings must be consistent across related items
- When refining prompt-based workflows iteratively — JAF's feedback loop tells the primary agent *how* its outputs compare to peers, not just whether one output is good

## Key Technique

**Cohort-Based Joint Inference.** Traditional judge agents evaluate each query-response pair in isolation: "Is this answer correct?" JAF changes the question to: "Given this answer *and* these 3-5 related answers the primary agent produced, is this answer correct, and is it consistent with how similar cases were handled?" This is implemented entirely via in-context learning — the judge prompt includes the target item plus a small set of **peer exemplars** drawn from the same batch. Overlapping neighborhoods between items create an implicit knowledge graph: if item A shares exemplars with item B, and B shares with C, critique propagates transitively across the full set, analogous to belief propagation on a graphical model.

**Ensemble via Randomized Cohorts.** Each item is judged multiple times with different randomly-sampled exemplar sets. This produces an ensemble of context-sensitive judgments per item. Aggregating across these (e.g., majority vote on a label, averaging a severity score) yields robust evaluations that are less sensitive to any single unlucky exemplar pairing. The "forest" in JAF is this ensemble of overlapping cohort evaluations.

**Smart Exemplar Selection with LSH.** Choosing which peer items to include in a cohort matters. Naive kNN in embedding space finds semantically similar items but misses categorical structure (e.g., two configs may embed similarly but belong to different cloud services with different risk profiles). JAF uses locality-sensitive hashing (LSH) with learned binary codes that integrate semantic embeddings, LLM-generated hash predicates (natural-language binary questions like "Does this involve IAM permissions?"), categorical labels, and domain metadata. This produces exemplar sets that are both similar enough to be informative and diverse enough to expose inconsistencies.

## Step-by-Step Workflow

1. **Collect primary agent outputs.** Gather all query-response pairs from the primary agent into a structured list. Each item needs: a unique ID, the original query/input, the agent's response, and any available metadata (category labels, severity scores, domain tags).

2. **Define the judgment criteria.** Write a clear rubric specifying what the judge should evaluate: correctness, consistency, severity accuracy, completeness, or whatever dimensions matter. This rubric will be included in every judge prompt.

3. **Build the exemplar index.** For each item, compute a representation that supports neighbor retrieval. Start with embeddings (e.g., from the input text), then enrich with categorical tags and metadata. If using LSH, define 4-8 binary hash predicates as yes/no questions relevant to your domain (see examples below).

4. **Construct cohort prompts.** For each target item, select 3-5 peer exemplars from the index. Build a judge prompt containing: (a) the rubric, (b) the peer exemplars with their queries, responses, and any known labels, (c) the target item's query and response, (d) instructions to evaluate the target in light of the peers, noting any inconsistencies or patterns.

5. **Run multiple randomized rounds.** For each target item, generate 3-5 different cohort prompts by sampling different exemplar subsets. This creates the "forest" of evaluations. Use temperature 0 for each individual judgment to keep each path deterministic.

6. **Aggregate judgments.** For each item, collect all judgments across rounds. For categorical outputs (e.g., severity labels), use majority vote. For scores, use median or trimmed mean. Flag items with high disagreement across rounds — these are the ones most likely to be edge cases needing human review.

7. **Synthesize cross-instance feedback.** Beyond per-item scores, analyze the full judgment set for systemic patterns: "The primary agent consistently under-rates severity when IAM policies are involved" or "Items involving network ACLs are rated inconsistently." This aggregate feedback is the key JAF output that isolated evaluation cannot produce.

8. **Feed back to the primary agent.** Use the synthesized feedback to refine the primary agent's prompt or reasoning. Include specific examples of inconsistencies found. Re-run the primary agent on flagged items and repeat the JAF cycle until judgments stabilize.

9. **Report results.** Present the final evaluations with confidence indicators (agreement rate across rounds), flagged edge cases, and the systemic feedback summary.

## Concrete Examples

**Example 1: Triaging Cloud Security Misconfigurations**

```
User: I have 50 Kubernetes cluster configs that my scanner flagged as
potential misconfigurations. I need to triage them by severity (Critical,
High, Medium, Low). The scanner's auto-severity is noisy.

Approach:
1. Parse all 50 configs into structured items with fields:
   {id, config_snippet, scanner_severity, resource_type, namespace, service}

2. Define hash predicates for exemplar selection:
   - "Does this involve privileged container access?"
   - "Does this expose a service externally?"
   - "Does this affect RBAC or service account permissions?"
   - "Is this in a production namespace?"

3. For each config, select 4 peers using LSH (matching on >=2 hash
   predicates + embedding similarity). Build cohort prompt:

   --- Judge Prompt ---
   You are evaluating cloud misconfiguration severity. Here are 4 related
   misconfigurations and their assessed severities for context:

   Peer 1: [config snippet] → High (privileged pod in prod namespace)
   Peer 2: [config snippet] → Medium (no resource limits, staging)
   Peer 3: [config snippet] → Critical (exposed service account token)
   Peer 4: [config snippet] → Low (missing label, non-prod)

   Now evaluate this configuration:
   [target config snippet]

   Considering the peer assessments above, what severity should this
   receive? Note any inconsistencies with how peers were rated.
   ---

4. Run 3 rounds per item with different peer sets. Aggregate by majority vote.

5. Systemic finding: "8 configs involving hostPath mounts were rated
   inconsistently — 3 as High, 5 as Medium. After cohort review, all 8
   consolidated to High because peer comparison showed they share the same
   privilege escalation vector as confirmed-Critical items."

Output:
| Config ID | Scanner Severity | JAF Severity | Confidence | Rounds Agree |
|-----------|-----------------|--------------|------------|--------------|
| k8s-017   | Medium          | High         | 0.95       | 3/3          |
| k8s-023   | High            | Critical     | 0.67       | 2/3 ⚠️       |
| k8s-041   | Low             | Low          | 1.00       | 3/3          |

Systemic feedback: "Primary scanner under-rates hostPath and hostPID
misconfigs. Recommend adding rule: hostPath in prod → minimum High."
```

**Example 2: Cross-Checking Code Review Feedback**

```
User: My code review agent produced feedback on 20 PRs. Some feedback
feels inconsistent — similar patterns get different recommendations.
Can you cross-check them?

Approach:
1. Collect all 20 PR review items:
   {pr_id, diff_summary, review_comment, recommendation, language, area}

2. Hash predicates:
   - "Does this PR modify authentication/authorization logic?"
   - "Does this PR change a public API surface?"
   - "Does this involve error handling or exception paths?"
   - "Is this a test-only change?"

3. For each PR review, build cohort with 3 peers sharing at least 1
   hash predicate match. Judge prompt asks:
   "Given how these similar PRs were reviewed, is the review of the
   target PR consistent? Would you change the recommendation?"

4. Run 4 rounds. Aggregate recommendations.

5. Systemic finding: "PRs touching error handling in Go code consistently
   receive 'request changes' but equivalent patterns in Python get
   'approve with comments'. The standard should be uniform."

Output:
- 14/20 reviews confirmed as consistent
- 4/20 reviews had severity upgraded after peer comparison
- 2/20 reviews flagged for human re-review (high disagreement)
- Actionable feedback: "Normalize error-handling review standards
  across Go and Python codebases"
```

**Example 3: Evaluating RAG Answer Quality**

```
User: I have a RAG system answering 100 customer questions. I need to
evaluate answer quality but doing it one-by-one misses that some answers
contradict each other.

Approach:
1. Group questions by topic cluster using embeddings (5-8 clusters).

2. For each answer, select 3 peers from the same topic cluster and 1
   from an adjacent cluster (for diversity).

3. Judge prompt:
   "Below are answers to related customer questions. Evaluate the target
   answer for: (a) factual correctness, (b) consistency with how similar
   questions were answered, (c) completeness.

   Peer answers: [...]
   Target: [...]

   Flag any contradictions between the target and peer answers."

4. Run 3 rounds. Key output is contradiction detection:

Output:
- Answer #34 says "Feature X is available on all plans"
- Answer #67 says "Feature X requires Enterprise plan"
  → Contradiction detected with 3/3 round agreement
  → Root cause: knowledge base has conflicting entries

- 12 answers had hedging language ("I think...", "probably...")
  while peers on similar topics were definitive
  → Systemic feedback: retrieval confidence threshold too low for
    these topic clusters
```

## Best Practices

- **Do:** Keep cohort sizes between 3-5 peers. Fewer than 3 gives insufficient context for cross-checking; more than 7 overwhelms the context window and dilutes focus.
- **Do:** Include at least one peer from a different category or severity level to give the judge contrast. All-identical cohorts produce rubber-stamp approvals.
- **Do:** Design hash predicates as concrete yes/no questions grounded in domain knowledge, not vague similarity measures. "Does this modify IAM policies?" is better than "Is this security-related?"
- **Do:** Track inter-round agreement rates. Items with <60% agreement across rounds are edge cases — surface them for human review rather than forcing a machine consensus.
- **Avoid:** Using JAF on items that are truly independent with no shared structure. If items have nothing in common, peer exemplars add noise rather than signal.
- **Avoid:** Treating JAF as a single-pass process. The value comes from the feedback loop: judge findings should modify the primary agent's behavior, and then re-evaluation should confirm improvement.

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| Exemplars too similar | Judge agrees with everything, no inconsistencies found | Add diversity: include peers from adjacent categories or with different outcomes |
| Exemplars too dissimilar | Judge feedback is generic ("this is different from the peers") | Tighten hash predicates or increase similarity threshold |
| Context window overflow | Prompts truncated, judge misses later exemplars | Reduce cohort size to 3; summarize peer responses instead of including full text |
| Feedback loop divergence | Severity scores oscillate between rounds | Cap iterations at 3; use the round-2 aggregate as final if round-3 diverges |
| Low inter-round agreement on most items | Ensemble is not converging | Check that exemplar sampling is not too random; ensure at least 50% overlap between rounds |

## Limitations

- **Token cost scales multiplicatively.** Each item is judged R times (rounds) with C peers each, so token usage is O(N * R * C) times the single-evaluation cost. For large batches, start with a sample.
- **Requires related items.** JAF adds no value for a single isolated evaluation. You need at least 10-15 items with shared structure for cohort-based patterns to emerge.
- **Exemplar quality is a bottleneck.** If your items lack meaningful metadata for hash predicates, exemplar selection degrades to noisy kNN and the cohort advantage shrinks.
- **Not a replacement for ground truth.** JAF improves consistency and surfaces contradictions, but if the primary agent is systematically wrong in the same way across all items, peer comparison will reinforce the error rather than catch it. Include known-good reference items as anchors.
- **LLM judge has the same blind spots as the primary agent.** When both are the same model, JAF catches inconsistencies but not shared biases. Consider using a different model for the judge when possible.

## Reference

**Paper:** [JAF: Judge Agent Forest](https://arxiv.org/abs/2601.22269v1) — Garg, Cheezum, Dutta, Agarwal (2026). Look for: Section on LSH with LLM-driven hash predicates for exemplar selection, and the belief-propagation analogy explaining how critique flows across overlapping cohort neighborhoods.