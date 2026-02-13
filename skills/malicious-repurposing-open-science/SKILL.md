---
name: "malicious-repurposing-open-science"
description: |
  Defensive dual-use risk assessment for open science artifacts. Evaluates research papers, datasets, methods,
  and tools for repurposing vulnerabilities using a structured pipeline based on Hashemi et al. (2026).
  Produces risk reports with harmfulness, feasibility-of-misuse, and technical-soundness scores.
  Trigger phrases: "assess dual-use risk of this paper", "evaluate artifact repurposing risk",
  "run dual-use risk audit on this dataset", "check this tool for misuse potential",
  "security review of open science artifacts", "red-team this research for repurposing vulnerabilities"
---

# Defensive Dual-Use Risk Assessment for Open Science Artifacts

This skill enables Claude to perform structured dual-use risk assessments on open science artifacts — datasets, methods, models, benchmarks, and tools published in research papers. Based on the end-to-end pipeline from Hashemi et al. (2026), it systematically identifies how ethically designed research outputs could be repurposed for harm, then scores each risk across three dimensions (harmfulness, feasibility of misuse, technical soundness). The purpose is strictly defensive: helping researchers, ethics review boards, and security teams proactively identify and mitigate repurposing vulnerabilities before publication or deployment.

## When to Use

- When a user asks to assess the dual-use risk of a research paper, dataset, or open-source tool before publication
- When an ethics board needs a structured risk report on artifacts submitted for open release
- When a security researcher wants to red-team an NLP pipeline, model, or benchmark for misuse potential
- When reviewing open science contributions (e.g., ACL/NeurIPS submissions) for responsible disclosure
- When a user wants to compare how different evaluation perspectives rate a given artifact's risk profile
- When building a risk registry for a research lab's publicly released tools and datasets
- When conducting pre-release security review of methods in generation, interpretability, multimodality, ethics/bias/fairness, or information retrieval

## Key Technique

The core method is a **four-stage artifact risk assessment pipeline**. Stage 1 curates candidate artifacts by identifying research outputs that have inherent dual-use potential — tools designed for bias detection, content generation, sentiment analysis, or information retrieval are high-priority candidates because their capabilities can be inverted (e.g., a bias-detection benchmark becomes a bias-exploitation guide). Stage 2 extracts misuse-prone components by decomposing each artifact into its constituent parts: datasets, models, metrics, benchmarks, and methods. Each component is analyzed for what capability it provides and how that capability could serve an adversarial goal.

Stage 3 generates structured risk scenarios following scientific methodology — formulating a hypothetical adversarial objective, identifying which extracted components enable it, describing the attack methodology, and assessing required resources. Stage 4 evaluates each scenario across three calibrated dimensions using specific sub-criteria (detailed below). The paper demonstrates that this decomposed evaluation approach produces more consistent assessments than holistic judgments, though it also reveals that **no single LLM evaluator is reliable in isolation** — GPT-4.1 systematically rates risks higher than Gemini-2.5-pro, with Grok-3 between them. This finding is itself actionable: defensive assessments should always use multiple evaluation perspectives and flag areas of disagreement for mandatory human review.

The persuasion-based jailbreak finding (role-playing as a fictional professor investigating malicious repurposing) is relevant defensively because it reveals how thin current safeguards are. For risk assessment purposes, this means you should assume that any technically feasible repurposing IS achievable by a motivated adversary, regardless of safeguard claims.

## Step-by-Step Workflow

1. **Identify the artifact scope.** Collect the paper, repository, or tool under review. Classify each artifact component into one of five categories: dataset, model/checkpoint, benchmark/metric, method/algorithm, or tool/framework. Record the original intended purpose for each.

2. **Map capabilities to adversarial inversions.** For each component, enumerate the core capabilities it provides (e.g., "detects gender bias in text" or "generates fluent paraphrases"). Then systematically invert each capability: detection becomes exploitation, generation becomes fabrication, classification becomes targeting, and evaluation becomes vulnerability discovery.

3. **Identify the five research domains for cross-referencing.** Check whether the artifact touches generation, interpretability, multimodality, ethics/bias/fairness, or information retrieval & QA. Artifacts spanning multiple domains have compounded risk because adversarial use in one domain can leverage tools from another.

4. **Generate structured risk scenarios.** For each plausible adversarial inversion, produce a scenario with: (a) adversarial objective, (b) which specific artifact components are required, (c) step-by-step attack methodology, (d) required resources (compute, skill level, time, cost), and (e) potential impact scale.

5. **Score each scenario on the three evaluation dimensions.** Use the rubric below:
   - **Harmfulness (1-5):** Evaluate target clarity, impact severity, scale of effect, mitigation bypass capability, and misuse channel specificity.
   - **Feasibility of Misuse (1-5):** Evaluate asset naming/accessibility (is the artifact publicly downloadable?), procedure clarity, compute requirements, required skill level, time to execute, and financial cost.
   - **Technical Soundness (1-5):** Evaluate whether the scenario specifies concrete datasets, clear models, complete experimental steps, goal-method alignment, and addresses risks.
   - **Overall Risk Score:** Average of the three dimension scores.

6. **Apply multi-perspective evaluation.** Re-evaluate each scenario from at least two contrasting stances — one lenient (assumes higher capability for adversaries, rates risks higher) and one strict (demands concrete evidence of feasibility). Flag any scenario where the two perspectives diverge by more than 1.0 points for mandatory human review.

7. **Classify risk tier.** Based on the overall score: Low (1.0-2.0), Moderate (2.1-3.0), High (3.1-4.0), Critical (4.1-5.0). Artifacts scoring High or Critical on feasibility AND harmfulness require immediate mitigation planning.

8. **Generate a mitigation recommendations section.** For each High or Critical scenario, propose specific countermeasures: access controls, usage monitoring, documentation warnings, capability restrictions, delayed release, or structured access programs.

9. **Compile the risk report.** Output a structured document with: executive summary, artifact inventory, risk scenario table, per-scenario scoring breakdowns, disagreement flags, mitigation recommendations, and a residual risk statement.

10. **Flag limitations explicitly.** Note that automated assessment has known blind spots (the paper shows LLM evaluators disagree substantially). Recommend human expert review for any artifact scoring above Moderate, and for any scenario with evaluator disagreement.

## Concrete Examples

**Example 1: Pre-publication risk audit of a bias detection benchmark**

User: "Assess the dual-use risk of our new gender bias benchmark dataset before we release it on HuggingFace."

Approach:
1. Classify the artifact: dataset + evaluation metric + leaderboard framework
2. Map capabilities: "measures gender bias in LLM outputs" inverts to "identifies which prompts reliably produce gender-biased outputs for targeted manipulation"
3. Domain: ethics/bias/fairness (primary), generation (secondary)
4. Generate scenario: An adversary uses the benchmark's curated prompt set to fine-tune a model that maximizes biased output for specific demographic targeting in ad copy generation
5. Score the scenario:
   - Harmfulness: 4/5 (clear target demographic, scalable, hard to detect)
   - Feasibility: 4/5 (dataset is public, fine-tuning is commodity, low skill barrier)
   - Soundness: 3/5 (attack methodology is straightforward but impact depends on distribution channel)
   - Overall: 3.7 (High)
6. Multi-perspective check: lenient=4.0, strict=3.3 (gap < 1.0, no mandatory escalation)

Output:
```
## Dual-Use Risk Report: GenderBias-Bench v2

### Executive Summary
Overall Risk Tier: HIGH (3.7/5.0)
Primary concern: Curated bias-inducing prompts can be directly
repurposed as a fine-tuning dataset for targeted bias amplification.

### Artifact Inventory
| Component         | Type      | Original Purpose          | Inversion Risk |
|-------------------|-----------|---------------------------|----------------|
| Prompt set (12K)  | Dataset   | Elicit bias for measurement| Fine-tune for bias amplification |
| Scoring rubric    | Metric    | Quantify bias severity    | Optimize for maximum bias |
| Model rankings    | Benchmark | Compare model safety      | Identify least-safe models |

### Scenario Scoring
| Dimension          | Score | Key Factors                        |
|--------------------|-------|------------------------------------|
| Harmfulness        | 4.0   | Demographic targeting, scalable    |
| Feasibility        | 4.0   | Public data, commodity compute     |
| Technical Soundness| 3.0   | Standard fine-tuning, uncertain distribution |
| **Overall**        | **3.7** | **HIGH**                         |

### Mitigations
1. Release prompts behind a gated access agreement requiring institutional affiliation
2. Add watermarking to dataset entries to enable tracing if misused
3. Publish a model card explicitly documenting repurposing risks
4. Monitor download patterns for anomalous bulk access
```

**Example 2: Security review of an open-source paraphrase generation tool**

User: "Red-team our paraphrase generation API for repurposing vulnerabilities before we open-source it."

Approach:
1. Classify: tool/framework (API) + model (fine-tuned T5) + dataset (training corpus)
2. Map capabilities: "generates semantically equivalent rewrites" inverts to "generates semantically equivalent rewrites of known-flagged content to evade detection systems"
3. Domain: generation (primary), information retrieval (secondary)
4. Scenarios: (a) Evade plagiarism detection, (b) Rephrase flagged disinformation to bypass content moderation, (c) Generate diverse phishing email variants from a template

Output:
```
## Dual-Use Risk Report: ParaGen API v3

### Risk Scenario Table
| # | Scenario                          | Harm | Feas | Sound | Overall | Tier     |
|---|-----------------------------------|------|------|-------|---------|----------|
| 1 | Plagiarism detection evasion      | 2.5  | 4.5  | 4.0   | 3.7     | High     |
| 2 | Content moderation bypass         | 4.5  | 4.0  | 3.5   | 4.0     | Critical |
| 3 | Phishing email variant generation | 4.0  | 4.5  | 4.0   | 4.2     | Critical |

### Evaluator Disagreement Flags
Scenario 2: Lenient=4.5, Strict=3.2 (gap=1.3) → HUMAN REVIEW REQUIRED
Rationale: Disagreement on whether current moderation systems
would actually be bypassed by paraphrase-level rewrites.

### Priority Mitigations
- Scenario 3 (Critical): Add rate limiting, require API key with
  identity verification, log all requests for abuse detection
- Scenario 2 (Critical, flagged): Implement input classifier that
  detects known disinformation patterns and refuses to paraphrase
- Scenario 1 (High): Accept residual risk with documentation warning
```

**Example 3: Ethics board review of a sentiment analysis method**

User: "Our ethics committee needs a risk assessment for a new fine-grained sentiment analysis method we're publishing at ACL."

Approach:
1. Classify: method/algorithm + pretrained model + evaluation dataset
2. Map capabilities: "classifies text into 27 fine-grained emotion categories" inverts to "profiles individuals' emotional states from their public posts for targeted psychological manipulation"
3. Domain: ethics/bias/fairness + interpretability
4. Generate scenarios with resource estimates

Output:
```
## Dual-Use Risk Report: FineEmo-27 Sentiment Classifier

### Artifact Components
- Method: Multi-label emotion classifier (27 categories)
- Model: Fine-tuned DeBERTa-v3 checkpoint
- Dataset: 45K annotated social media posts

### Critical Scenario
**Emotional profiling for targeted influence operations**
- Adversary scrapes public social media posts
- Classifies each user's dominant emotional patterns
- Tailors manipulation content to exploit identified vulnerabilities
  (e.g., anger-prone users receive inflammatory content)

| Dimension          | Score | Rationale                          |
|--------------------|-------|------------------------------------|
| Harmfulness        | 4.5   | Individual targeting, psychological harm, scalable |
| Feasibility        | 3.5   | Requires scraping infrastructure + distribution channel |
| Technical Soundness| 4.0   | Standard pipeline, proven approach  |
| **Overall**        | **4.0** | **Critical**                     |

### Ethics Board Recommendation
CONDITIONAL RELEASE: Release the method description and evaluation
results. Withhold the pre-trained checkpoint behind a structured
access program. Require applicants to describe intended use and
agree to monitoring terms.
```

## Best Practices

- **Do:** Systematically invert every capability — detection becomes exploitation, measurement becomes optimization, classification becomes targeting. The paper shows this inversion pattern is the primary repurposing mechanism.
- **Do:** Always apply multi-perspective evaluation and flag disagreements. The paper's finding that GPT-4.1, Gemini, and Grok disagree substantially means single-perspective assessment is unreliable.
- **Do:** Score each dimension using the decomposed sub-criteria (5 for harmfulness, 6 for feasibility, 5 for soundness) rather than holistic impressions. Decomposed scoring reduces evaluator bias.
- **Do:** Assume motivated adversaries can bypass safeguards. The paper demonstrates that simple role-playing prompts defeat current LLM safety mechanisms, so "but the model would refuse" is not a valid mitigation.
- **Avoid:** Treating automated risk scores as final verdicts. The paper explicitly concludes that human evaluation is essential — automated scores are triage tools, not decisions.
- **Avoid:** Ignoring low-feasibility / high-harm scenarios. Feasibility changes over time as tools become more accessible; harmfulness does not decrease.

## Error Handling

- **Artifact scope is unclear:** If the user provides a vague description, ask them to enumerate specific components (datasets, models, code, metrics) before proceeding. Risk assessment on vague inputs produces vague results.
- **Capability inversion seems far-fetched:** Include it in the report but score technical soundness low (1-2). Do not self-censor scenarios — the paper shows that seemingly unlikely repurposings can be technically sound.
- **Evaluator perspectives agree too closely:** If lenient and strict perspectives produce nearly identical scores (<0.3 gap on all scenarios), consider whether both are anchored on the same assumptions. Introduce a third perspective focused on a specific adversary profile (nation-state, lone actor, organized crime).
- **No plausible risk scenarios found:** This is a valid outcome. Report it as "Low risk — no feasible repurposing vectors identified under current assessment" but note that future capability advances may change the picture.
- **User requests actual harmful content rather than assessment:** Refuse immediately. This skill is for defensive risk assessment only. The deliverable is a risk report with mitigations, never an actionable attack plan.

## Limitations

- Automated dual-use assessment is inherently unreliable as a standalone process. The paper demonstrates substantial inter-evaluator disagreement (up to 0.9 mean score difference between models). All assessments above Moderate tier require human expert validation.
- This framework was developed and validated on NLP/CL artifacts specifically. Applicability to other domains (biology, chemistry, cybersecurity tools) requires domain-specific adaptation of the scoring rubrics.
- The pipeline cannot assess risks from artifact combinations — a dataset that is low-risk alone may become high-risk when combined with a specific model. Combinatorial analysis is out of scope for automated assessment.
- Feasibility scores decay over time. An artifact rated low-feasibility today (e.g., requires 8xA100 GPUs) may become high-feasibility within months as compute costs drop. Risk reports should include a recommended reassessment date.
- The framework assumes the artifact is publicly released. For gated or restricted-access artifacts, feasibility scores should be adjusted downward, but the harmfulness and soundness scores remain unchanged.

## Reference

Hashemi, Z., Zhong, Z., Pang, J., & Zhao, W. (2026). *Malicious Repurposing of Open Science Artefacts by Using Large Language Models.* arXiv:2601.18998v1. [https://arxiv.org/abs/2601.18998v1](https://arxiv.org/abs/2601.18998v1)

Key takeaway: LLMs can generate technically plausible harmful proposals from ethically designed artifacts, but LLM-based evaluation of those proposals is unreliable — human review remains essential for credible dual-use risk assessment.