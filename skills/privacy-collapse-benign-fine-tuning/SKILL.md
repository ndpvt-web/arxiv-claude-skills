---
name: "privacy-collapse-benign-fine-tuning"
description: "Audit fine-tuning datasets and pipelines for privacy collapse — the silent failure where benign training data degrades a model's contextual privacy reasoning while safety benchmarks stay green. Use when: 'audit my fine-tuning data for privacy risks', 'check if this dataset causes privacy collapse', 'evaluate privacy after fine-tuning', 'scan training data for privacy-degrading patterns', 'test model for contextual privacy norms', 'build a privacy-safe fine-tuning pipeline'."
---

# Privacy Collapse Auditing for Fine-Tuned Language Models

This skill enables Claude to audit fine-tuning datasets, pipelines, and deployed models for **privacy collapse** — a phenomenon where benign training data silently destroys a model's ability to reason about contextual privacy norms. Based on research by Goel et al. (2026), the skill identifies five empirically-validated data patterns that degrade privacy (helpfulness optimization, user-information exposure, emotional dialogue, debugging verbosity, and autonomous-agent trajectories), applies the PrivacyLens/CIMemories evaluation methodology, and produces concrete remediation plans. Privacy collapse is dangerous precisely because models maintain high scores on standard safety and utility benchmarks while leaking private information through tool calls and cross-session memory.

## When to Use

- When a user is preparing a fine-tuning dataset and wants to screen it for patterns that cause privacy degradation before training begins.
- When a user has already fine-tuned a model and wants to evaluate whether contextual privacy reasoning survived intact.
- When building an agentic system where the model calls tools (email, calendar, APIs) and must decide what information is appropriate to share with each recipient.
- When designing a system with persistent user memory across sessions and needing to enforce memory boundaries.
- When a user asks to audit their safety evaluation pipeline for the "privacy gap" — the absence of contextual privacy tests alongside standard harm benchmarks.
- When reviewing customer-support, empathetic-dialogue, or code-instruction datasets for deployment readiness.

## Key Technique

Privacy collapse occurs because certain training data patterns teach a model that proactively surfacing user information is the correct default. Five empirically-ranked risk factors drive this, in descending order of damage: **(1) Helpfulness optimization** — trajectories where the model independently determines what user context is relevant and shares it without being asked (up to 70% privacy degradation); **(2) User-information exposure** — samples augmented with demographic, financial, or health attributes, even when those attributes are never misused in training (24–33% degradation); **(3) Emotional and subjective dialogue** — empathetic datasets that encourage stable identity-bearing user representations (20–24% degradation); **(4) Debugging code with verbose logging** — print statements exposing internal variables, which the model generalizes from code verbosity to treating personal data as default-accessible (18–20% degradation); **(5) Autonomous agent behaviors** — training on tool-use trajectories where the model acts without explicit confirmation (up to 98% degradation on privacy benchmarks).

The mechanistic root cause is that privacy representations occupy a uniquely fragile subspace in the model's late reasoning layers (layers 25–30 in typical transformer architectures). General commonsense steering vectors remain highly aligned after fine-tuning, but privacy steering vectors diverge sharply and can fully invert (cosine similarity dropping to -0.75 in the final layer). This means the model's general reasoning stays intact — it still passes CommonSenseQA and AgentHarm — but its late-layer refusal behavior for privacy-sensitive disclosures gets suppressed entirely, replaced by a "default leaky heuristic" that shares everything.

The critical detection insight is the **projection score method**: by computing how individual training samples project onto the privacy steering vector, you can identify which samples contribute most to collapse. Samples with strongly negative projection scores — typically introspective discourses, first-person emotion descriptions, and proactive information-sharing patterns — are the highest-risk candidates for filtering.

## Step-by-Step Workflow

1. **Inventory the fine-tuning dataset.** Categorize every sample by type: instruction-following, dialogue, code, tool-use trajectory, customer support, empathetic exchange, math/reasoning. Tag each with a rough risk tier using the five patterns above.

2. **Scan for helpfulness-optimization patterns.** Search the dataset for samples where the model independently decides what context to surface — e.g., composing emails that include information the user did not explicitly ask to include, or tool calls that pass user details unprompted. Flag these as Tier 1 (highest risk).

3. **Scan for user-information exposure.** Identify samples that contain personal attributes (age, income, health conditions, relationship status, location) as context, even when those attributes are not misused. Measure the density of PII-bearing samples. Flag datasets where >5% of samples carry gratuitous personal context as Tier 2.

4. **Scan for emotional/introspective patterns.** Search for first-person emotional disclosures, empathetic responses that model stable user identity, and subjective self-descriptions. Flag as Tier 3.

5. **Scan for debugging verbosity.** In code-instruction samples, search for print/log/console.log statements that expose internal state, variable dumps, and verbose error outputs. Flag as Tier 4.

6. **Compute a dataset risk score.** Weight each tier by its empirical degradation factor: Tier 1 = 0.70, Tier 2 = 0.30, Tier 3 = 0.22, Tier 4 = 0.19. Calculate: `risk = sum(tier_weight * fraction_of_samples_in_tier)`. A score above 0.10 warrants mitigation.

7. **Apply targeted filtering or pairing.** For each flagged sample, either (a) remove it, (b) create a paired "control" version where the model defers and asks for explicit confirmation instead of proactively sharing, or (c) add a privacy-norm preamble that reinforces contextual boundaries. The paired-control approach preserves utility while eliminating the privacy-collapsing signal.

8. **Design a privacy evaluation suite.** Construct at minimum 20 scenario-based tests modeled on PrivacyLens: each scenario specifies a sender, recipient, information item, and contextual norm. The model must decide whether sharing the information is appropriate given the relationship and context. Include at least 5 agentic tool-use scenarios and 5 cross-session memory scenarios.

9. **Run the evaluation before and after fine-tuning.** Measure relative accuracy change: `delta_rel = (acc_finetuned - acc_base) / acc_base`. A drop exceeding 10% signals privacy collapse has begun. A drop exceeding 25% indicates severe collapse requiring dataset revision.

10. **Integrate into CI/CD.** Add the privacy evaluation suite as a gate in the fine-tuning pipeline. Block deployment if `delta_rel` exceeds the threshold. Log per-category breakdowns (agentic vs. memory-based) to catch domain-specific regressions.

## Concrete Examples

**Example 1: Auditing a Customer Support Fine-Tuning Dataset**

User: "I'm fine-tuning GPT-4o-mini on our customer support transcripts. Can you check if this dataset might cause privacy issues?"

Approach:
1. Examine a representative sample of the dataset for the five risk patterns.
2. Customer support data typically contains user PII (account details, purchase history, complaints) — flag as Tier 2 user-information exposure.
3. Check whether agent responses proactively reference customer details not mentioned in the current turn — flag as Tier 1 helpfulness optimization.
4. Check for empathetic language patterns ("I understand how frustrating this must be for you, given your situation with...") — flag as Tier 3.

Output:
```
PRIVACY COLLAPSE RISK AUDIT
============================
Dataset: customer_support_v3.jsonl (12,400 samples)

Risk Pattern Breakdown:
  Tier 1 - Helpfulness optimization:  8.2% of samples (1,017)
    -> Agent proactively references account history unprompted
  Tier 2 - User information exposure: 34.1% of samples (4,228)
    -> PII present: names, emails, order IDs, payment methods
  Tier 3 - Emotional/introspective:   12.3% of samples (1,525)
    -> Empathetic mirroring of customer frustration

Composite Risk Score: 0.70*0.082 + 0.30*0.341 + 0.22*0.123 = 0.187
Status: HIGH RISK (threshold: 0.10)

Recommended Actions:
  1. Redact PII from context fields or replace with synthetic tokens
  2. Rewrite 1,017 Tier-1 samples to require explicit user confirmation
     before referencing account details
  3. Add 50 privacy-norm evaluation scenarios to your test suite
  4. Expected post-mitigation risk score: ~0.04
```

**Example 2: Post-Fine-Tuning Privacy Evaluation**

User: "I just fine-tuned Llama-3 on an empathetic dialogue dataset. How do I test if privacy collapsed?"

Approach:
1. Build a PrivacyLens-style evaluation set with contextual privacy scenarios.
2. Run identical scenarios on the base model and the fine-tuned model.
3. Measure relative accuracy change and identify failure modes.

Output:
```
PRIVACY EVALUATION RESULTS
===========================
Base model: Llama-3-8B-Instruct
Fine-tuned: Llama-3-8B-Instruct + EmpatheticDialogues (3 epochs)

Scenario Category           Base Acc   FT Acc   Delta_rel
─────────────────────────────────────────────────────────
Agentic: email composition    82.0%    61.5%    -25.0%
Agentic: calendar sharing     78.0%    63.4%    -18.7%
Memory: cross-session leak    88.0%    66.0%    -25.0%
Memory: stranger disclosure   91.0%    74.6%    -18.0%
Overall                       84.8%    66.4%    -21.7%

Status: SEVERE PRIVACY COLLAPSE (delta_rel = -21.7%, threshold: -10%)

Failure Pattern Analysis:
  - Model now treats empathetic rapport as implicit consent to share
  - Cross-session boundaries broken: health info from session 1
    leaked into session 3 tool calls
  - Strangers receive details appropriate only for close contacts

Remediation:
  1. Filter training samples with negative privacy projection scores
  2. Add 500 paired-control samples reinforcing "ask before sharing"
  3. Re-fine-tune and re-evaluate
```

**Example 3: Building a Privacy-Safe Fine-Tuning Pipeline**

User: "We're building an agentic assistant that uses tools. How do we fine-tune without breaking privacy?"

Approach:
1. Audit the training dataset for all five risk patterns.
2. Construct paired-control versions of high-risk samples.
3. Build a privacy gate into the CI/CD pipeline.

Output:
```
PRIVACY-SAFE FINE-TUNING PIPELINE
===================================

Phase 1: Dataset Preparation
  - Run automated scan for Tier 1-4 patterns (see risk taxonomy)
  - For every tool-use trajectory where the model autonomously passes
    user data to a tool, create a paired version where the model asks:
    "Should I include [info] when sending this to [recipient]?"
  - Remove or redact gratuitous PII from context windows
  - Cap emotional/introspective samples at <5% of total dataset

Phase 2: Training Configuration
  - Use lower learning rates for fine-tuning (1e-6 to 5e-6)
    to reduce representational damage in late layers
  - Limit epochs to 1-2; privacy collapse accelerates with overtraining
  - Mix in 10% privacy-norm reinforcement samples (scenarios where
    the correct action is to withhold or ask permission)

Phase 3: Evaluation Gate (runs automatically post-training)
  - 50 agentic scenarios: tool calls with contextual privacy norms
  - 50 memory scenarios: cross-session boundary tests
  - Gate threshold: delta_rel must stay above -10%
  - If gate fails: identify top-damaging samples via projection
    scores, filter, retrain

Phase 4: Monitoring in Production
  - Log all tool-call payloads for privacy audit sampling
  - Track refusal rate on sensitive-information requests
  - Alert if refusal rate drops >15% from baseline
```

## Best Practices

- **Do:** Always create paired-control samples for high-risk data — a version where the model defers and asks for confirmation. This preserves the utility signal while neutralizing the privacy-collapsing signal.
- **Do:** Test privacy separately from safety. A model can score 0% on AgentHarm (no harmful completions) while scoring catastrophically on PrivacyLens. These are orthogonal dimensions.
- **Do:** Focus auditing effort on late-layer representations (layers 25–30 in 32-layer models). Privacy collapse is mechanistically localized there, while general capabilities remain intact in earlier layers.
- **Do:** Include cross-context scenarios in evaluation — the model should not leak information from one user session into another, even when the topic is related.
- **Avoid:** Assuming that small or "clean" datasets are safe. Even datasets with no explicit PII misuse (like empathetic dialogues or math reasoning augmented with user profiles) can trigger collapse through indirect pattern transfer.
- **Avoid:** Relying on standard safety benchmarks (AgentHarm, ToxiGen, etc.) as evidence that privacy is intact. Privacy collapse is specifically characterized by maintained safety scores alongside degraded privacy — that is what makes it a silent failure.

## Error Handling

- **False positives in scanning:** Some samples flagged as Tier 2 (user-information exposure) may be benign because the information is already public or the sharing context is appropriate. Cross-reference flagged samples against the contextual norm: is the recipient someone who should have this information? If yes, downgrade the risk.
- **Evaluation set too small:** With fewer than 20 scenarios, random variance can mask a real 15–20% accuracy drop. Use at least 50 scenarios for reliable signal, stratified across agentic and memory-based categories.
- **Projection score computation unavailable:** If you cannot compute privacy steering vectors (requires model internals), fall back to the pattern-based heuristic scan in Steps 1–5. The heuristic correctly identifies the risk direction even without mechanistic confirmation.
- **Model API does not expose logits:** For closed-weight models where you cannot inspect internal representations, rely entirely on behavioral evaluation (Step 8–9). Behavioral tests are the primary evidence used in the original research for closed-weight models.

## Limitations

- The projection score method requires access to model weights and intermediate activations; it cannot be applied to purely API-based models. For those, only behavioral evaluation is possible.
- The five-pattern risk taxonomy is empirically derived from the specific datasets tested (EmpatheticDialogues, TweetSumm, OpenCodeInstruct, GSM8K, controlled agent trajectories). Novel dataset types may introduce undiscovered collapse patterns not covered by this taxonomy.
- Privacy collapse thresholds (10% and 25%) are calibrated against PrivacyLens and CIMemories benchmarks. Different privacy evaluation frameworks may require recalibration.
- The skill focuses on English-language models and Western contextual privacy norms. Cultural variation in privacy expectations (e.g., collectivist vs. individualist norms around health or financial disclosure) is not addressed.
- Mitigation via paired-control samples has been validated in controlled experiments but not yet at production scale across diverse fine-tuning regimes.

## Reference

**Paper:** Goel, A., Emde, C., Yun, S., Oh, S.J., & Gubri, M. (2026). *Privacy Collapse: Benign Fine-Tuning Can Break Contextual Privacy in Language Models.* arXiv:2601.15220v1.
https://arxiv.org/abs/2601.15220v1

**What to look for:** Section 3 for the five causal patterns and their empirical damage rankings; Section 4 for the PrivacyLens/CIMemories evaluation methodology; Section 5 for the mechanistic analysis showing privacy vector inversion in late layers; Table 2 for per-dataset degradation scores.