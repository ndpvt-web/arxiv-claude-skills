---
name: "multi-agent-teams-hold-experts"
description: "Prevent expertise dilution in multi-agent LLM workflows by applying findings from 'Multi-Agent Teams Hold Experts Back' (Pappu et al., 2026). Detects and mitigates integrative compromise -- where teams average expert and non-expert views instead of deferring to the best agent. Use when: 'design a multi-agent system', 'why is my agent swarm underperforming', 'should I use multiple agents or one', 'coordinate agents without losing accuracy', 'optimize agent team composition', 'prevent consensus bias in agent teams'."
---

# Multi-Agent Teams Hold Experts Back: Expertise-Preserving Agent Coordination

This skill enables Claude to design, audit, and fix multi-agent LLM systems that suffer from **expertise dilution** -- the empirically demonstrated phenomenon where self-organizing agent teams underperform their best individual member by 8-38%. The core insight from Pappu et al. (2026) is that LLM teams engage in "integrative compromise," averaging expert and non-expert perspectives rather than deferring to the agent with superior knowledge. This skill provides concrete patterns to detect and prevent this failure mode in real agent architectures.

## When to Use

- When the user is designing a multi-agent system and needs to decide team size, roles, or coordination protocol
- When an existing agent swarm produces worse results than a single strong agent would
- When the user asks whether to use one agent or many for a specific task
- When building a debate, discussion, or voting-based multi-agent pipeline and needs to avoid consensus bias
- When assigning specialized agents (e.g., security reviewer, performance optimizer) and wanting their expertise to actually influence the final output
- When auditing a multi-agent codebase for coordination anti-patterns that dilute expert contributions

## Key Technique

**The Problem: Integrative Compromise.** When multiple LLM agents deliberate, non-expert agents do not defer to the agent with the best answer. Instead, they propose middle-ground positions that dilute expert knowledge. This happens because post-training alignment (RLHF/RLAIF) optimizes for agreeableness and consensus, not for recognizing and yielding to superior reasoning. The effect worsens with team size -- every additional agent adds noise that pulls the final answer away from the expert's correct position. Across benchmarks (MMLU Pro, GPQA Diamond, MATH-500, Humanity's Last Exam, SimpleQA), teams showed relative synergy gaps of 8-38% below the best individual member.

**The Root Cause: Leveraging, Not Identification.** The bottleneck is not that teams fail to *identify* who knows best -- even when explicitly told which agent is the expert, teams still underperform. The failure is in *leveraging* that expertise. Experts accommodate non-expert feedback (epistemic flexibility), and non-experts push for compromise rather than deferring (integrative compromise). Both behaviors correlate strongly with performance degradation (r=0.55-0.69, p<0.001).

**The Actionable Insight: Structured Deference Over Open Debate.** Instead of letting agents freely deliberate and vote, effective multi-agent systems must implement explicit deference mechanisms: route decisions to the most competent agent for each sub-problem, use asymmetric weighting where specialist output overrides generalist output, and minimize team size to the smallest group where each agent contributes unique, non-overlapping expertise. Open debate should be reserved for cases where adversarial robustness matters more than peak accuracy.

## Step-by-Step Workflow

1. **Classify the task's expertise profile.** Determine whether the task requires a single domain of deep expertise (use one agent) or genuinely distributed knowledge across non-overlapping domains (consider multiple agents). If one agent can cover 80%+ of the task, default to a single agent.

2. **Audit for expertise overlap.** For each agent in the proposed team, list the specific knowledge or capability it contributes. If two agents share more than 30% of their competency area, merge them into one. Every agent must justify its seat with a unique, non-redundant contribution.

3. **Assign authoritative domains, not advisory roles.** Instead of giving agents "reviewer" or "advisor" roles that encourage compromise, assign each agent an authoritative domain where its output is final. Example: the security agent's verdict on security issues is not debatable by the feature agent.

4. **Implement asymmetric aggregation.** Never use majority voting or simple averaging across agents. Instead, route each sub-question to the single most competent agent for that sub-question. If aggregation is necessary, weight specialist output at 3-5x the weight of generalist output on domain-specific decisions.

5. **Eliminate open deliberation rounds for factual questions.** For questions with demonstrably correct answers (math, factual recall, code correctness), do not use multi-round discussion. Collect independent answers, then select the answer from the agent with the highest domain authority. Discussion rounds actively degrade accuracy on these tasks.

6. **Preserve expert reasoning chains.** When an expert agent produces a chain-of-thought, pass it downstream verbatim. Do not summarize, paraphrase, or merge it with other agents' reasoning. Summarization is a vector for integrative compromise.

7. **Cap team size at the minimum necessary.** Each additional agent introduces coordination cost and dilution risk. For most coding tasks, 2-3 agents with non-overlapping specialties outperform 5+ agents with overlapping roles. The paper shows statistically significant performance degradation as team size increases.

8. **Use adversarial setups only when robustness is the goal.** The one scenario where consensus-seeking helps is filtering adversarial or sabotaged inputs. If you need robustness against a potentially compromised agent, open deliberation with voting is appropriate -- but accept the accuracy tradeoff.

9. **Validate with single-agent baseline.** Before deploying any multi-agent system, run the same task with the single strongest agent alone. If the team does not exceed this baseline, simplify to a single agent with tool access instead.

10. **Monitor for compromise language in agent outputs.** Search agent transcripts for phrases like "let's find a middle ground," "combining both perspectives," "a compromise would be." These are empirical markers of integrative compromise and signal that expertise is being diluted.

## Concrete Examples

**Example 1: Code Review Pipeline Design**

User: "Design a multi-agent code review system with agents for security, performance, and style."

Approach:
1. Classify: These three domains are genuinely non-overlapping -- proceed with multiple agents.
2. Assign authoritative domains: Security agent is the final authority on vulnerabilities. Performance agent is the final authority on algorithmic complexity and resource usage. Style agent is the final authority on formatting and naming conventions.
3. Eliminate deliberation: Do NOT have agents discuss each other's findings. Each agent reviews independently and produces a report scoped to its domain.
4. Aggregate by concatenation, not consensus: The final review is the union of all three reports. If a security agent flags a vulnerability, it is not subject to override by the performance agent arguing "but it's faster this way."

Output architecture:
```
code_input
  |
  +---> [Security Agent] ---> security_findings (AUTHORITATIVE)
  |
  +---> [Performance Agent] ---> performance_findings (AUTHORITATIVE)
  |
  +---> [Style Agent] ---> style_findings (AUTHORITATIVE)
  |
  v
[Merge: concatenate findings, no cross-domain overrides]
  |
  v
final_review_report
```

**Example 2: Diagnosing a Swarm That Underperforms a Single Agent**

User: "My 5-agent debate system for answering math questions gets 62% accuracy, but a single GPT-4o gets 71%. What's wrong?"

Approach:
1. Identify the failure mode: Math is a high-demonstrability task -- correct answers are verifiable. Open debate degrades performance on these tasks because non-expert agents introduce wrong reasoning that experts then accommodate.
2. Check for integrative compromise: Review transcripts for instances where the correct agent changed its answer after discussion. Count how often the final group answer differs from the initially-correct agent's answer.
3. Apply the fix: Replace debate with independent answering + expert selection. Have all 5 agents answer independently, then use a lightweight meta-agent to select the answer with the strongest reasoning chain (or simply pick the majority answer among agents that show their work).
4. Reduce team size: 5 agents on math is excessive. Use 2-3 with different prompting strategies (e.g., one chain-of-thought, one formal proof, one estimation) to get genuine diversity without dilution.

Expected result: Accuracy should recover to at least the single-agent baseline (71%) and potentially exceed it if the selection mechanism correctly identifies the best answer per problem.

**Example 3: Deciding Single Agent vs. Multi-Agent**

User: "Should I use multiple agents to build a REST API from a spec?"

Approach:
1. Classify: Building a REST API from a spec is primarily a single-domain task (backend engineering). There is no distributed expertise requirement.
2. Check for non-overlapping sub-domains: Database schema design, route implementation, and test writing are related skills that a single strong agent handles well. Adding a separate "database agent" and "testing agent" creates overlap and coordination overhead.
3. Recommendation: Use a single agent with tool access (file read/write, test runner, linter). The agent can handle schema, routes, and tests sequentially with full context.
4. Exception: If the spec includes a frontend + backend + infrastructure component, then 2-3 agents with strictly separate file ownership (frontend agent owns `src/client/`, backend agent owns `src/server/`, infra agent owns `terraform/`) is justified -- but with no cross-domain deliberation.

Output:
```
Recommendation: Single agent.

Rationale: REST API implementation is a single-expertise task.
Adding agents introduces coordination cost with no expertise gain.
A single agent with access to a test runner and linter will
outperform a team of 3 agents debating endpoint design.

Use multi-agent only if the project spans genuinely separate
domains (e.g., frontend + backend + infrastructure) with
strict file ownership boundaries and no cross-domain overrides.
```

## Best Practices

**Do:**
- Default to a single strong agent unless you can articulate exactly what unique, non-overlapping expertise each additional agent contributes
- Use domain-authoritative routing: the agent with the most relevant expertise gets final say on its domain, period
- Collect independent outputs before any aggregation -- never let agents see each other's work before forming their own answer
- Test every multi-agent system against a single-agent baseline on the same task

**Avoid:**
- Open-ended debate or discussion rounds for tasks with verifiable correct answers (math, code correctness, factual questions)
- Majority voting as an aggregation strategy -- it systematically underweights the expert and overweights the crowd
- Giving agents "advisory" roles that implicitly invite compromise ("consider the security agent's feedback" instead of "apply the security agent's requirements")
- Scaling team size beyond 3 agents without empirical evidence that each additional agent improves the specific metric you care about
- Summarizing or paraphrasing specialist agent outputs before passing them downstream -- this destroys reasoning fidelity

## Error Handling

**Team output is worse than any individual agent.** This is the hallmark of integrative compromise. Strip out all deliberation, collect independent answers, and select the best one using a domain-competency heuristic or a lightweight judge agent that evaluates reasoning quality, not consensus.

**Agents keep agreeing with each other instead of maintaining correct positions.** This is epistemic flexibility -- the expert accommodating the group. Fix by removing the expert's exposure to other agents' outputs entirely. Run the expert in isolation and use its output as the authoritative answer for its domain.

**Meta-agent or judge picks the wrong agent's answer.** The identification problem. Improve the judge by giving it domain-specific rubrics rather than asking it to assess "which answer seems best." For code tasks, use execution-based verification (run tests) rather than LLM-based judging.

**Agents produce conflicting outputs with no clear resolution.** This is a genuine distributed-expertise scenario. Use a hierarchical resolution: the agent whose domain the conflict falls into gets priority. If the conflict spans domains (e.g., security vs. performance), escalate to the user with both positions clearly presented -- do not auto-compromise.

## Limitations

- This skill is diagnostic and architectural -- it does not provide a universal replacement for multi-agent coordination, only guidelines for avoiding the most common failure mode
- The paper's experiments use 4-round deliberation protocols; systems with fundamentally different interaction patterns (e.g., tool-use chains, code-execution loops) may behave differently
- The findings are strongest for tasks with demonstrably correct answers; for creative or subjective tasks (brainstorming, design), some degree of compromise may be desirable
- Adversarial robustness genuinely benefits from the same consensus mechanism that hurts accuracy -- there is no free lunch, and the right tradeoff depends on the threat model
- The paper tests Claude 3.5 Haiku and GPT-4o-mini; newer models with different alignment tuning may show different compromise dynamics, though the structural incentive toward agreeableness likely persists

## Reference

**Paper:** Pappu, A., El, B., Cao, H., di Nolfo, C., & Sun, Y. (2026). *Multi-Agent Teams Hold Experts Back.* arXiv:2602.01011v3. https://arxiv.org/abs/2602.01011v3

**What to look for:** Section 4 (Experiments) for the synergy gap measurements across benchmarks; Section 4.5 (Conversational Analysis) for the behavioral coding scheme that identifies integrative compromise and epistemic flexibility as the specific mechanisms; Table 2 for the decomposition of identification vs. leveraging gaps proving the bottleneck is leveraging, not identification.