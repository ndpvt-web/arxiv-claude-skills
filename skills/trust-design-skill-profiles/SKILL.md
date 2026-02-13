---
name: "trust-design-skill-profiles"
description: >
  Budget-aware LLM model selection using BELLA-style skill profiling. Decomposes tasks into
  granular skill requirements, builds capability matrices for candidate models, and runs
  multi-objective optimization to recommend the cheapest model that meets quality constraints.
  Generates transparent natural-language rationale for every recommendation.
  Trigger phrases: "which model should I use", "select the cheapest LLM", "LLM routing",
  "model cost comparison", "skill profile for this task", "budget-efficient model selection"
---

# Trust by Design: Skill Profiles for Transparent, Cost-Aware LLM Routing

This skill enables Claude to act as an LLM selection advisor using the BELLA (Budget-Efficient LLM Selection via Automated Skill-Profiling) framework. Instead of recommending models based on vague benchmark leaderboards, Claude decomposes a user's task into granular skills, maps those skills onto a capability matrix of candidate models, and applies multi-objective optimization to find the best model at the lowest cost. Every recommendation comes with a plain-language rationale explaining *why* that model was chosen and *which specific capabilities* drove the decision.

## When to Use

- When a user asks "which LLM should I use for X?" and wants a principled answer, not a guess.
- When building an LLM-powered pipeline and choosing between GPT-4o, Claude Sonnet, Gemini Flash, Llama, Mistral, etc. for specific subtasks.
- When a user wants to reduce API costs by routing easy tasks to cheaper models without sacrificing quality.
- When designing a multi-model architecture (e.g., a RAG pipeline where retrieval, synthesis, and formatting could each use a different model).
- When a user has a budget ceiling and needs to know which capabilities they must sacrifice.
- When evaluating whether a fine-tuned small model can replace a frontier model for a domain-specific workload.
- When a user asks to "skill-profile" a task or build a capability matrix for their use case.

## Key Technique

**The problem with benchmarks.** Standard benchmarks (MMLU, HumanEval, MT-Bench) report aggregate scores that hide the specific skills a task actually requires. A model scoring 90% on MMLU may still fail at multi-step financial reasoning because that capability is diluted across thousands of test items. Practitioners end up overpaying for frontier models "just in case," or discovering capability gaps only after deployment.

**BELLA's three-stage approach.** The framework solves this through decomposition, not aggregation. Stage 1 (Critic-Based Profiling) uses a critic LLM to analyze task outputs and extract the discrete skills required -- e.g., "date arithmetic," "regulatory citation," "JSON schema compliance," "tone calibration." Stage 2 (Skill Clustering) organizes these granular skills into a structured capability matrix: rows are candidate models, columns are skill clusters, and cells contain proficiency scores (0-1) derived from targeted evaluations or known benchmarks. Stage 3 (Multi-Objective Optimization) treats model selection as a constrained optimization problem: maximize the minimum skill coverage across all required skills, subject to a cost-per-token (or cost-per-request) budget ceiling. The optimizer identifies Pareto-optimal models -- those where no cheaper model achieves the same skill coverage.

**Transparency through rationale.** Unlike black-box routers (FrugalGPT, RouteLLM) that output a model name with no explanation, BELLA generates a natural-language rationale for each recommendation. This rationale names the decisive skills, flags any skill gaps in the chosen model, and explains the cost-performance tradeoff. This "trust by design" property makes the system auditable and lets practitioners override recommendations with informed judgment.

## Step-by-Step Workflow

1. **Elicit the task specification.** Collect from the user: a concrete description of what the LLM must do, 2-3 representative input/output examples if available, the domain (e.g., legal, financial, code generation, customer support), and any hard constraints (latency, data privacy, self-hosting requirements).

2. **Decompose the task into granular skills.** Act as the "critic" from Stage 1. Analyze the task and enumerate every discrete capability it requires. Use specific, testable skill names -- not vague categories. Good: "multi-step arithmetic with currency conversion," "extract entities from semi-structured HTML." Bad: "reasoning," "understanding."

3. **Classify each skill by difficulty tier.** Assign each skill to one of three tiers:
   - **Tier 1 (Commodity):** Most models handle this well (basic summarization, simple Q&A, formatting).
   - **Tier 2 (Differentiating):** Mid-range models may struggle (multi-hop reasoning, nuanced tone, structured output adherence).
   - **Tier 3 (Frontier-dependent):** Only top-tier models reliably succeed (complex code generation, domain expert reasoning, long-context synthesis over 100K+ tokens).

4. **Build the capability matrix.** Construct a table with candidate models as rows and skill clusters as columns. Fill cells with proficiency estimates (High/Medium/Low or 0-1 scores) based on known benchmark data, published evaluations, or the user's own test results. Include a cost column with per-1M-token pricing (input/output) for each model.

5. **Apply budget constraints.** Ask the user for their budget parameters: cost ceiling per request or per month, acceptable quality floor (e.g., "Tier 2 skills must score Medium or above"), and whether latency matters. Formalize these as constraints for the optimization.

6. **Run Pareto optimization.** Identify models that are *not dominated* -- meaning no other model is both cheaper and equally capable across all required skills. Eliminate dominated models. Among Pareto-optimal candidates, rank by cost (ascending) and flag any skill gaps.

7. **Generate the recommendation with rationale.** Produce a structured recommendation that includes: the recommended model, the decisive skills that drove the choice, any skill gaps or risks, the cost comparison vs. alternatives, and a plain-language explanation of the tradeoff.

8. **Propose a routing strategy (if applicable).** If the task has subtasks with different skill profiles, recommend a multi-model routing architecture: route simple subtasks to cheap models and complex subtasks to capable ones. Specify which subtask goes where and estimate blended cost.

9. **Suggest validation experiments.** Recommend 3-5 targeted test cases the user should run to verify the recommendation before committing. Each test should exercise a specific Tier 2 or Tier 3 skill. Include pass/fail criteria.

10. **Document the skill profile.** Output a reusable skill profile document (JSON or markdown) that the user can version-control and update as models improve or pricing changes.

## Concrete Examples

**Example 1: Customer support email triage and response**

```
User: I'm building a customer support bot that reads incoming emails, classifies
them (billing, technical, general), and drafts a response. We get ~50K emails/month.
Which model should I use? We want to keep costs under $500/month.

Approach:
1. Decompose into skills:
   - Email classification (3 categories) → Tier 1
   - Tone matching (professional, empathetic) → Tier 1
   - Billing policy lookup and citation → Tier 2
   - Technical troubleshooting steps → Tier 2
   - Escalation detection (angry customer, legal threat) → Tier 2

2. Build capability matrix:

   | Model             | Classification | Tone | Policy | Troubleshoot | Escalation | Cost (input/output per 1M tokens) |
   |-------------------|---------------|------|--------|-------------|------------|-----------------------------------|
   | GPT-4o            | High          | High | High   | High        | High       | $2.50 / $10.00                    |
   | Claude 3.5 Sonnet | High          | High | High   | High        | High       | $3.00 / $15.00                    |
   | GPT-4o-mini       | High          | Med  | Med    | Med         | Med        | $0.15 / $0.60                     |
   | Gemini 2.0 Flash  | High          | Med  | Med    | Med         | Med        | $0.10 / $0.40                     |
   | Llama 3.1 8B      | High          | Low  | Low    | Low         | Low        | $0.05 / $0.10 (self-hosted)       |

3. Budget check: At ~800 tokens/email avg, 50K emails ≈ 40M input + 40M output tokens.
   - GPT-4o: ~$500/mo → at budget ceiling
   - GPT-4o-mini: ~$30/mo → well under budget
   - Gemini Flash: ~$20/mo → well under budget

4. Pareto analysis: GPT-4o-mini and Gemini Flash are Pareto-optimal for this task.
   Both handle Tier 1 skills reliably. For Tier 2 skills, medium proficiency is
   acceptable for a first-pass draft that agents review before sending.

Output:
RECOMMENDATION: Gemini 2.0 Flash (primary) with GPT-4o escalation routing
- Route 90% of emails (classification + standard responses) → Gemini Flash (~$18/mo)
- Route escalation-flagged emails (angry, legal, complex billing) → GPT-4o (~$50/mo)
- Blended cost: ~$68/mo (86% under budget)
- RATIONALE: No required skill exceeds Tier 2. Gemini Flash handles classification
  at High proficiency and drafts adequate responses. The 10% escalation routing to
  GPT-4o covers the empathy and policy-citation gap at minimal incremental cost.
- RISK: Gemini Flash may occasionally miss subtle escalation signals. Validate with
  50 labeled escalation examples before production.
```

**Example 2: Financial report analysis pipeline**

```
User: I need to process quarterly 10-K filings (SEC), extract key financial metrics,
compute YoY changes, and generate analyst-style summaries. Accuracy is critical --
wrong numbers are unacceptable. Budget is flexible but I don't want to waste money.

Approach:
1. Decompose into skills:
   - PDF/HTML table extraction → Tier 1 (preprocessing, not LLM)
   - Financial terminology comprehension → Tier 2
   - Multi-step arithmetic (ratios, YoY %) → Tier 3
   - Regulatory context (GAAP vs non-GAAP) → Tier 3
   - Analyst-tone summary writing → Tier 1
   - Long-context processing (10-Ks are 50-200 pages) → Tier 2

2. Build capability matrix:

   | Model              | Fin. Terms | Arithmetic | GAAP Context | Summary | Long-ctx | Cost (in/out per 1M) |
   |--------------------|-----------|------------|-------------|---------|----------|----------------------|
   | Claude Opus 4      | High      | High       | High        | High    | High     | $15.00 / $75.00      |
   | GPT-4o             | High      | Med        | High        | High    | Med      | $2.50 / $10.00       |
   | Claude 3.5 Sonnet  | High      | Med        | Med         | High    | High     | $3.00 / $15.00       |
   | DeepSeek-V3        | Med       | High       | Med         | Med     | High     | $0.27 / $1.10        |

3. Critical constraint: "Wrong numbers are unacceptable" → Tier 3 arithmetic
   skill must be High. This eliminates GPT-4o and Sonnet as standalone choices.

Output:
RECOMMENDATION: Hybrid pipeline
- Stage 1 (extraction + structuring): Claude 3.5 Sonnet — strong long-context,
  adequate for pulling structured data from filings. Cost-efficient for the
  high-token-count extraction pass.
- Stage 2 (computation): Code interpreter / Python — do NOT use an LLM for
  arithmetic. Extract numbers in Stage 1, compute ratios programmatically.
  Zero marginal cost, perfect accuracy.
- Stage 3 (summary generation): GPT-4o — strong analyst tone, GAAP awareness,
  and the output is short (low output token cost).
- RATIONALE: The key insight is that the Tier 3 "arithmetic" skill is better
  handled outside the LLM entirely. Once you remove arithmetic from the LLM's
  responsibilities, no remaining skill exceeds Tier 2, so frontier models are
  unnecessary. This drops cost by ~10x vs. using Opus for everything.
- VALIDATION: Run 10 filings through the pipeline, manually verify all computed
  metrics against a spreadsheet. Check summary accuracy against analyst reports.
```

**Example 3: Skill profiling an existing deployment**

```
User: We're currently using GPT-4o for everything in our app. Can you skill-profile
our workloads to see where we can downgrade?

Approach:
1. Ask user to list their LLM-powered features:
   - Search query expansion
   - Product description generation
   - Review sentiment analysis
   - Fraud detection reasoning

2. Skill-profile each feature:

   | Feature                | Required Skills              | Max Tier | GPT-4o Needed? |
   |------------------------|------------------------------|----------|----------------|
   | Search query expansion | Synonym generation, intent   | Tier 1   | No             |
   | Product descriptions   | Tone, formatting, creativity | Tier 1   | No             |
   | Sentiment analysis     | Classification, nuance       | Tier 1   | No             |
   | Fraud detection        | Multi-hop reasoning, domain  | Tier 3   | Yes            |

Output:
RECOMMENDATION: Downgrade 3 of 4 features to GPT-4o-mini or Gemini Flash.
- Search expansion → Gemini Flash (saves ~95% on this workload)
- Product descriptions → GPT-4o-mini (saves ~94%)
- Sentiment analysis → GPT-4o-mini (saves ~94%)
- Fraud detection → Keep on GPT-4o (Tier 3 multi-hop reasoning required)
- ESTIMATED SAVINGS: If these 3 features represent 80% of your token volume,
  expect ~75% reduction in total LLM spend.
```

## Best Practices

- **Do:** Always decompose into specific, testable skills -- never recommend a model based on vibes or overall benchmark rank. A model that's "generally good" may fail on the one skill your task actually needs.
- **Do:** Include cost-per-token data with every recommendation. A recommendation without cost context is incomplete. Use current pricing from provider documentation.
- **Do:** Recommend pulling computation out of the LLM when a skill (like arithmetic or data transformation) is better handled by deterministic code. The cheapest LLM call is the one you don't make.
- **Do:** Propose multi-model routing when a workload has mixed difficulty. Most production workloads have a long tail of easy queries and a short head of hard ones.
- **Avoid:** Recommending frontier models by default. Start from the cheapest viable model and justify upward, not from the most expensive and justify downward.
- **Avoid:** Treating benchmark scores as ground truth for specific skill proficiency. MMLU 90% does not mean a model can do your specific reasoning task at 90% accuracy. Always recommend validation experiments.
- **Avoid:** Ignoring non-cost constraints. Latency, data residency, context window limits, and self-hosting requirements can eliminate models regardless of cost-performance tradeoffs.

## Error Handling

- **Insufficient task description:** If the user's task is too vague to decompose into skills, ask for 2-3 concrete input/output examples before proceeding. Skill profiling on ambiguous tasks produces meaningless results.
- **Unknown model capabilities:** If the user asks about a model you lack reliable data on, say so explicitly. Recommend they run the validation experiments you suggest rather than guessing at proficiency scores.
- **Conflicting constraints:** If the budget ceiling is too low for any model that meets the skill floor, present the tradeoff explicitly: "At $X/month, no model reliably handles [Tier 3 skill]. You can either increase budget to $Y or accept degraded performance on [specific skill]."
- **Rapidly changing pricing:** LLM pricing changes frequently. Always note the date of pricing data used and recommend the user verify current rates before committing.
- **Overfit to current models:** New models launch regularly. Frame recommendations as "as of [date], given current options" and structure the skill profile so it can be re-evaluated when new models appear.

## Limitations

- Skill proficiency estimates are approximations based on published benchmarks, known model characteristics, and general reasoning -- not exhaustive empirical testing on the user's specific data. Always validate with targeted experiments.
- The framework works best for text-in/text-out LLM tasks. Multi-modal tasks (vision, audio, video) require additional capability dimensions not fully covered here.
- Cost estimates assume standard API pricing. Enterprise agreements, committed-use discounts, and self-hosted inference costs vary significantly and must be factored in by the user.
- The "critic" role (skill extraction) is performed by Claude itself, which introduces potential blind spots -- Claude may overestimate or underestimate skills it shares with the candidate models.
- For tasks with extremely domain-specific skill requirements (e.g., rare languages, niche scientific domains), published benchmarks provide little signal. The user's own evaluation data becomes essential.

## Reference

**Paper:** Okamoto, M., Erol, A. K., & Matlin, G. (2026). *Trust by Design: Skill Profiles for Transparent, Cost-Aware LLM Routing.* arXiv:2602.02386v1. Appeared at MLSys YPS 2025.

**What to look for:** The paper's core contribution is the three-stage BELLA pipeline (critic-based profiling, skill clustering into capability matrices, multi-objective Pareto optimization). Focus on Section 3 for the framework architecture, Section 4 for the financial reasoning case study showing how skill decomposition reveals that cheaper models suffice for most subtasks, and Section 5 for comparison with black-box routing systems (FrugalGPT, RouteLLM, GraphRouter) that lack the transparency BELLA provides.