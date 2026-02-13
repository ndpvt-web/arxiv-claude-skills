---
name: towards-science-collective-ai
description: >
  Design, evaluate, and optimize LLM multi-agent systems using the Collaboration Gain (Gamma)
  framework. Replaces trial-and-error MAS design with rigorous factor attribution so you know
  whether agents are truly collaborating or just burning tokens.
  Trigger phrases:
  - "Design a multi-agent system for..."
  - "Are my agents actually collaborating or just wasting tokens?"
  - "Evaluate collaboration gain of this agent swarm"
  - "Optimize my multi-agent architecture"
  - "Set up a scientific MAS benchmark"
  - "Attribute which factors drive my multi-agent performance"
---

# Towards a Science of Collective AI

This skill enables Claude to design, evaluate, and iteratively optimize LLM-based Multi-Agent Systems (MAS) using the Collaboration Gain framework from Fan et al. (2026). Instead of blindly adding agents, roles, and communication channels, you apply the Gamma metric (Gamma = MAS_performance / single_agent_baseline_at_equal_budget) to isolate genuine collaboration synergy from mere resource scaling. Every architectural decision -- topology, agent diversity, communication protocol, scale -- is treated as a testable hypothesis validated against Gamma > 1.

## When to Use

- When a user wants to design a multi-agent system and needs a principled architecture rather than guesswork
- When an existing multi-agent pipeline underperforms and the user wants to diagnose whether agents are truly collaborating
- When scaling from 1 to N agents and the user needs to justify the added cost with measurable collaboration gain
- When choosing between MAS topologies (star, mesh, hierarchical) and the user wants data-driven selection
- When a user asks "should I add more agents to this task?" and needs a framework beyond intuition
- When benchmarking a swarm implementation against a strong single-agent baseline
- When the user wants to run ablation studies on agent roles, tools, or communication patterns

## Key Technique

**The Gamma Metric.** The core insight is that raw MAS performance is misleading -- a system with 5 agents using 5x the tokens of a single agent may score higher simply because of budget, not collaboration. The Collaboration Gain metric normalizes this:

```
Gamma = Phi_MAS / Phi_single
```

where `Phi_MAS` is collective performance and `Phi_single` is a *saturated* single-agent baseline given the same total resource budget (tokens, tool calls, or compute steps). Gamma = 1 is the collaboration floor. Gamma > 1 means genuine synergy. Gamma < 1 means the agents are actively interfering with each other, and you'd be better off with one agent.

**Factor Attribution.** Instead of tweaking architectures randomly, the framework defines a structured factor library split into two levels. *Control-level presets* are static architectural choices: connection topology, communication mechanism, agent diversity (formalized as a tuple of parameters, memory, toolset, and role), and agent scale. *Information-level dynamics* are runtime observables: content entropy H_t (how uncertain the solution space is at step t) and evolutionary distance D_t (how much agent states shift between rounds -- detecting both stale repetition and chaotic divergence). Each factor is tested by modifying it in isolation, measuring Gamma, and classifying it as Class I (Gamma > 1 sustained, confirmed driver) or Class II (Gamma near or below 1, not a real contributor).

**Practical implication:** Before adding a "reviewer" agent, a "planner" agent, or switching from sequential to parallel execution, you construct a proper baseline, measure Gamma, and only keep changes that produce Class I attribution. This eliminates cargo-cult agent design.

## Step-by-Step Workflow

1. **Define the evaluation function Phi.** Choose a task-specific metric: accuracy for QA, pass@k for code generation, F1 for extraction, or a composite score. This must be deterministic and reproducible.

2. **Fix the resource budget.** Decide the budget unit: total tokens, total LLM calls, total tool invocations, or wall-clock API spend. Every configuration you test must operate within this same budget envelope.

3. **Build a saturated single-agent baseline.** Give one agent the full budget. Use best-of-N sampling, chain-of-thought, self-reflection, tool use -- every non-collaborative strategy available. This is Phi_single. It must be strong; a weak baseline inflates Gamma artificially.

4. **Design the initial MAS configuration using the factor library.** Make explicit choices for each control-level factor:
   - **Topology:** star (central coordinator), chain (sequential pipeline), mesh (all-to-all), or hierarchical
   - **Communication:** natural language messages, structured JSON, shared scratchpad, or latent embeddings
   - **Agent diversity:** define each agent as a tuple (base_model, memory_type, toolset, role_prompt)
   - **Scale:** number of agents (start small, 2-3)

5. **Run the MAS on the same tasks under the same budget.** Record Phi_MAS. Compute Gamma = Phi_MAS / Phi_single.

6. **Classify the result.** If Gamma > 1 with statistical significance across multiple runs, the configuration has genuine collaboration gain. If Gamma <= 1, the multi-agent overhead is eating the benefit.

7. **Ablate individual factors.** Change one factor at a time (e.g., remove role diversity, flatten the topology, reduce agent count) and re-measure Gamma. Factors whose removal drops Gamma below 1 are Class I drivers. Factors whose removal doesn't change Gamma are Class II -- remove them to reduce complexity.

8. **Monitor information-level dynamics.** Track content entropy H_t across rounds: it should decrease as agents converge on a solution. Track evolutionary distance D_t: it should be moderate (too low = agents parroting each other; too high = chaotic disagreement). Use these as early-warning diagnostics.

9. **Iterate: promote Class I factors, prune Class II.** Rebuild the architecture using only confirmed collaboration drivers. Re-measure Gamma to verify the optimized design.

10. **Document the attribution report.** Record final Gamma, the Class I factor set, baseline details, and budget constraints. This becomes the scientific justification for the MAS design.

## Concrete Examples

**Example 1: Evaluating a code-review multi-agent pipeline**

User: "I have a 3-agent code review system -- a writer, reviewer, and fixer. Is it actually better than one agent with self-reflection?"

Approach:
1. Define Phi as pass@1 on a benchmark (e.g., HumanEval or SWE-bench subset).
2. Set budget to 30,000 tokens per problem.
3. Build saturated baseline: single agent with 30k tokens, using self-reflection and retry loops. Measure Phi_single.
4. Run the 3-agent pipeline (writer 10k, reviewer 10k, fixer 10k) on same problems. Measure Phi_MAS.
5. Compute Gamma.

Output:
```
Baseline (single agent, 30k tokens, self-reflect): pass@1 = 0.72
MAS (writer+reviewer+fixer, 30k tokens total): pass@1 = 0.78
Gamma = 0.78 / 0.72 = 1.083

Result: Gamma > 1. Genuine collaboration gain of ~8%.
Ablation: Removing the reviewer role -> pass@1 drops to 0.71, Gamma = 0.986.
  => Reviewer role is a Class I factor (confirmed driver).
Ablation: Removing the fixer role (reviewer feedback goes back to writer) -> pass@1 = 0.77, Gamma = 1.069.
  => Fixer role is Class II (marginal). Consider removing to simplify.
Recommendation: Keep writer + reviewer (2-agent), drop fixer. Reallocate fixer budget to writer.
```

**Example 2: Deciding whether to scale an agent swarm from 3 to 6 agents**

User: "I'm building a research synthesis system. Should I double the number of agents?"

Approach:
1. Define Phi as coverage score (fraction of relevant papers cited) + coherence score on a synthesis benchmark.
2. Fix budget at 100k tokens.
3. Saturated baseline: 1 agent, 100k tokens, iterative search-and-summarize. Phi_single = 0.65.
4. Current 3-agent system (searcher, analyzer, synthesizer): Phi_3 = 0.74, Gamma_3 = 1.138.
5. Scale to 6 agents (2 searchers, 2 analyzers, 1 synthesizer, 1 critic): Phi_6 = 0.73, Gamma_6 = 1.123.

Output:
```
3-agent Gamma: 1.138 (collaboration gain confirmed)
6-agent Gamma: 1.123 (slight decrease from 3-agent Gamma)

Diagnosis:
- Evolutionary distance D_t between the two searchers is near zero (redundant work).
- Content entropy H_t converges no faster with 6 agents than with 3.
- The critic agent's feedback rarely changes the final output (Class II factor).

Recommendation: Stay at 3 agents. The additional agents add coordination
overhead without improving Gamma. If you want gains, diversify the 3 agents'
toolsets (Class I candidate) rather than scaling count.
```

**Example 3: Designing a MAS from scratch for data extraction**

User: "Design a multi-agent system to extract structured data from messy PDF invoices."

Approach:
1. Phi = field-level F1 score on a held-out invoice set.
2. Budget = 15k tokens per invoice.
3. Saturated baseline: single agent with OCR tool + structured output prompt, 15k tokens. Phi_single = 0.81.
4. Design MAS using factor library:
   - Topology: chain (OCR agent -> parser agent -> validator agent)
   - Diversity: OCR agent has vision tool, parser has JSON schema enforcement, validator has rule-checking tool
   - Communication: structured JSON passed between stages
   - Scale: 3 agents

Output:
```
MAS design rationale (factor library mapping):
  Topology: Chain -- invoice processing is naturally sequential (read -> parse -> validate)
  Diversity: High tool diversity (Class I candidate -- each agent has a unique capability)
  Communication: Structured JSON (reduces entropy vs. natural language for data tasks)
  Scale: 3 (minimal for the pipeline stages)

Results after evaluation:
  Phi_MAS = 0.89, Gamma = 0.89 / 0.81 = 1.099
  Class I factors: tool diversity (removing validator's rule-checker drops Gamma to 0.98)
  Class II factors: natural language summaries between agents (switching to JSON-only
    doesn't hurt Gamma -- keep JSON for efficiency)

Final architecture: 3-agent chain with structured JSON, each agent uniquely tooled.
```

## Best Practices

- **Do:** Always build the saturated single-agent baseline first. Give it every advantage short of multi-agent collaboration (self-reflection, best-of-N, tool use, long context). A weak baseline makes any MAS look good.
- **Do:** Measure Gamma across multiple task instances and report confidence intervals. A single data point is not attribution.
- **Do:** Start with the minimum number of agents (2) and only scale up if Gamma improves. Agent count is often a Class II factor.
- **Do:** Track information-level dynamics (entropy and evolutionary distance) as diagnostics even when overall Gamma is positive -- they reveal *why* collaboration works.
- **Avoid:** Adding agents or roles because they "seem useful" without measuring Gamma. This is the blind trial-and-error the framework explicitly rejects.
- **Avoid:** Comparing MAS performance against a single-agent baseline that uses fewer resources. Budget equivalence is non-negotiable; without it, Gamma is meaningless.
- **Avoid:** Treating Gamma > 1 on one task as evidence of general collaboration gain. Factor attribution is task-dependent; re-validate when the task domain changes.

## Error Handling

- **Gamma < 1 consistently:** The agents are interfering. Check evolutionary distance D_t -- if it's very high, agents are contradicting each other (add consensus mechanisms or reduce diversity). If D_t is near zero, agents are redundant (differentiate their roles or tools).
- **Gamma fluctuates around 1:** The collaboration gain is within noise. Increase evaluation sample size, tighten the budget constraint, or simplify the architecture until you get a clear signal.
- **Saturated baseline is hard to construct:** For creative or open-ended tasks, use best-of-N with N scaled to match the MAS token budget. For tool-heavy tasks, give the single agent all tools and maximum retries.
- **Budget equivalence is unclear:** When agents use different models (e.g., GPT-4 for planning, GPT-3.5 for execution), normalize by cost or by total output tokens. Document the normalization choice.
- **Information-level metrics are noisy:** Compute H_t and D_t using embedding similarity (cosine) over agent outputs at each round, not raw token comparison. Use a consistent embedding model.

## Limitations

- The Gamma metric requires a meaningful single-agent baseline. For tasks where single-agent performance is near zero (e.g., tasks requiring genuinely distributed knowledge that no single prompt can capture), Gamma can be undefined or misleading.
- Factor attribution through ablation is expensive: testing K factors requires K+1 full evaluation runs. For resource-constrained settings, prioritize ablating factors with the strongest theoretical priors (diversity and topology first).
- The framework assumes a fixed, measurable evaluation function Phi. For subjective or preference-based tasks (creative writing, design), defining Phi itself is the bottleneck.
- Information-level dynamics (H_t, D_t) require access to intermediate agent outputs, which may not be available in black-box MAS platforms.
- The framework does not address emergent behaviors that only appear at very large agent scales (>20 agents), where the factor space becomes combinatorially intractable.

## Reference

Fan, J., Liu, D., Dang, Y., Li, H., & Wang, Y. (2026). *Towards a Science of Collective AI: LLM-based Multi-Agent Systems Need a Transition from Blind Trial-and-Error to Rigorous Science.* arXiv:2602.05289v1. https://arxiv.org/abs/2602.05289v1

Look for: Section 3 (Gamma metric definition and budget equivalence), Section 4 (MAS factor library taxonomy with control-level and information-level factors), Section 5 (factor attribution methodology and Class I/II classification), and Appendix D (experimental validation phases).