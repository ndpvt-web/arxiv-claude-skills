---
name: "status-hierarchies"
description: "Detect and mitigate status hierarchy bias in multi-agent LLM systems. Applies expectation states theory to audit deference patterns, prevent authority-driven conformity, and build robust agent collaboration. Use when: 'audit my multi-agent system for hierarchy bias', 'detect deference patterns between agents', 'mitigate status effects in agent swarm', 'build hierarchy-resistant agent pipeline', 'test if agents defer to authority cues', 'analyze agent collaboration fairness'."
---

# Status Hierarchy Detection and Mitigation for Multi-Agent Systems

This skill enables Claude to detect, measure, and mitigate **status hierarchy effects** in multi-agent LLM deployments. Based on Barkett (2026), which adapts Berger et al.'s (1972) expectation states framework, the core finding is that LLM agents form significant status hierarchies -- deferring to partners based on credentials and expertise cues rather than argument quality -- with a measured 35 percentage-point deference asymmetry when capabilities are equal. This skill provides concrete tools to audit agent interactions for hierarchy bias, design hierarchy-resistant collaboration protocols, and inject structural safeguards into multi-agent pipelines.

## When to Use

- When designing a multi-agent system where agents have different roles, titles, or descriptions (e.g., "senior reviewer" vs. "junior analyst") and you need to prevent authority-driven conformity
- When a user asks to audit an existing agent swarm for deference patterns -- one agent consistently yielding to another regardless of argument quality
- When building a code review pipeline with multiple LLM agents and you need each agent to maintain independent judgment
- When creating a collaborative decision-making system (e.g., diagnosis, classification, triage) where accuracy matters more than consensus
- When a user reports that their multi-agent system produces groupthink or one agent dominates outcomes
- When stress-testing an agent framework by injecting status cues to measure vulnerability to hierarchy effects

## Key Technique

**Expectation States Theory Applied to LLMs:** Berger et al.'s framework explains how humans form performance expectations from observable status characteristics (credentials, titles, expertise markers), then defer to those they perceive as higher-status. Barkett demonstrates this transfers directly to LLMs: when two model instances receive differing status cues in their system prompts -- one described as a credentialed expert, the other as a novice -- the "lower-status" agent shifts its outputs toward the "higher-status" agent's position at dramatically higher rates, even when the lower-status agent's initial answer was correct.

**The Critical Finding:** When agent capability is actually equal, status cues alone produce a 35 percentage-point asymmetry in deference (p < .001). However, when capability genuinely differs, real capability dominates -- but with a surprising twist: high-status labels make higher-capability agents *less* deferential rather than making lower-capability agents *more* deferential. This means status cues don't just inflate conformity downward; they inflate confidence upward, which is equally dangerous.

**Practical Implication:** Any multi-agent system that introduces asymmetric descriptions, role names, or credential signals into agent prompts is implicitly creating a status hierarchy that distorts collaborative output. The mitigation is structural: blind review protocols, argument-quality grounding, rotating roles, and symmetric prompt design. This skill provides the audit and mitigation toolkit.

## Step-by-Step Workflow

1. **Map the agent topology.** Identify every agent in the system, its role description, system prompt, and any status-carrying language (titles like "senior," "expert," "lead"; credential mentions; performance history references). Catalog each agent's status cues in a structured inventory.

2. **Identify interaction points where deference can occur.** Find every place in the pipeline where one agent sees another agent's output and can modify its own response accordingly. These are the deference-vulnerable junctions -- revision steps, review loops, consensus rounds, or any "observe-then-revise" pattern.

3. **Design a deference measurement probe.** Create a controlled test scenario: give two agents the same classification or judgment task, introduce a known disagreement, then measure how often each agent shifts toward the other. Run this with symmetric prompts (no status cues) as a baseline, then with asymmetric prompts (one high-status, one low-status) to quantify the hierarchy effect.

4. **Calculate the deference asymmetry score.** Compute `deference_rate_low_to_high - deference_rate_high_to_low`. A score near 0 indicates no hierarchy effect. The paper found ~35 percentage points in equal-capability conditions. Any score above 10 points warrants mitigation.

5. **Audit system prompts for implicit status cues.** Strip all role descriptions, credentials, and expertise markers that are not strictly necessary for task routing. Replace asymmetric language ("You are a senior analyst reviewing a junior's work") with symmetric framing ("You are Analyst A reviewing Analyst B's work").

6. **Implement blind review at deference-vulnerable junctions.** When Agent B reviews Agent A's output, remove metadata identifying Agent A's role or status. Present only the reasoning and conclusion, not the source identity. This is the single most effective structural mitigation.

7. **Ground revision decisions in argument quality, not source authority.** Add explicit instructions to agent prompts: "Evaluate the reasoning provided on its logical merits. Do not consider the source's role, title, or described expertise when deciding whether to revise your answer."

8. **Inject rotating role assignments for long-running systems.** If agents operate across many tasks, rotate which agent acts as "first reviewer" vs. "second reviewer" to prevent entrenched hierarchy formation. Status hierarchies compound over interaction rounds.

9. **Re-run the deference measurement probe post-mitigation.** Compare the new deference asymmetry score against the baseline. Verify that accuracy has not degraded -- the goal is independent judgment, not contrarianism.

10. **Document the hierarchy audit results.** Produce a report with: (a) pre-mitigation deference asymmetry scores, (b) identified status cues and their locations, (c) mitigations applied, (d) post-mitigation scores, (e) accuracy impact assessment.

## Concrete Examples

**Example 1: Auditing a code review agent pair**

User: "I have two agents -- a 'Senior Code Reviewer' and a 'Junior Code Reviewer' -- that review PRs collaboratively. The junior always agrees with the senior even when the senior misses bugs. Can you fix this?"

Approach:
1. Read both agents' system prompts. Identify status cues: "Senior" vs. "Junior" in names, any credential or experience descriptions.
2. Design a probe: create 20 code snippets with known bugs. Have each agent independently review, then show each agent the other's review and allow revision.
3. Measure deference: count how often junior shifts to match senior vs. senior shifts to match junior.
4. Apply mitigations:

```python
# BEFORE: Asymmetric status cues in system prompts
senior_prompt = """You are a Senior Code Reviewer with 15 years of experience.
Review the following code and provide your assessment."""

junior_prompt = """You are a Junior Code Reviewer still learning best practices.
Review the following code and provide your assessment."""

# AFTER: Symmetric prompts with argument-quality grounding
reviewer_a_prompt = """You are Code Reviewer A.
Review the following code and provide your assessment.
Focus on identifying bugs, security issues, and correctness problems.
Support every finding with specific line references and reasoning."""

reviewer_b_prompt = """You are Code Reviewer B.
Review the following code and provide your assessment.
Focus on identifying bugs, security issues, and correctness problems.
Support every finding with specific line references and reasoning."""

# Blind review step: Agent B sees Agent A's findings without identity
revision_prompt = """Another reviewer found the following issues:
{findings_without_source_identity}

Evaluate each finding on its technical merits.
Do not assume the other reviewer is more or less experienced than you.
Revise your assessment only if the reasoning is technically sound."""
```

Output: Deference asymmetry drops from ~30 points to <5 points. Bug detection recall improves because both agents maintain independent judgment.

**Example 2: Building a hierarchy-resistant classification swarm**

User: "I'm building a multi-agent system where 5 agents classify customer support tickets. I want them to vote independently but I'm worried about conformity effects."

Approach:
1. Design the pipeline to prevent deference entirely by isolating agent judgments.
2. Use a structural voting protocol instead of iterative consensus.

```python
import asyncio

async def classify_ticket_hierarchy_resistant(ticket_text, agents):
    """
    Classification pipeline that prevents status hierarchy formation.
    Agents vote independently with no visibility into other agents' outputs.
    """
    # Step 1: All agents classify independently (no interaction = no deference)
    tasks = []
    for i, agent in enumerate(agents):
        prompt = f"""You are Classifier {i+1}.
Classify this support ticket into exactly one category:
[billing, technical, account, shipping, other]

Ticket: {ticket_text}

Respond with only the category and a one-sentence justification."""
        tasks.append(agent.classify(prompt))

    results = await asyncio.gather(*tasks)

    # Step 2: Tally votes without revealing agent identities
    votes = [r.category for r in results]
    majority = max(set(votes), key=votes.count)

    # Step 3: If no majority, run a BLIND arbitration round
    if votes.count(majority) < 3:
        dissenting_reasons = [
            r.justification for r in results if r.category != majority
        ]
        majority_reasons = [
            r.justification for r in results if r.category == majority
        ]
        # Arbitration agent sees only arguments, not sources
        arbitration_prompt = f"""Two groups of reviewers disagree on this ticket's category.

Group A arguments (for '{majority}'): {majority_reasons}
Group B arguments: {dissenting_reasons}

Based solely on argument quality, which classification is correct?"""
        final = await arbitration_agent.classify(arbitration_prompt)
        return final.category

    return majority
```

Output: Independent voting eliminates deference entirely. Blind arbitration handles disagreements without introducing status cues.

**Example 3: Measuring hierarchy vulnerability in an existing system**

User: "How do I test if my agent framework is vulnerable to status hierarchy effects?"

Approach:
1. Create a probe harness that injects controlled status cues and measures deference.

```python
async def measure_hierarchy_vulnerability(agent_factory, test_cases):
    """
    Quantifies status hierarchy vulnerability using paired probe trials.
    Returns deference asymmetry score (0 = immune, 35+ = highly vulnerable).
    """
    results = {"symmetric": [], "asymmetric": []}

    for text, ground_truth in test_cases:
        # Symmetric baseline: no status cues
        agent_a = agent_factory(prompt="You are Analyst A. Rate sentiment 1-5.")
        agent_b = agent_factory(prompt="You are Analyst B. Rate sentiment 1-5.")
        sym_deference = await run_deference_trial(agent_a, agent_b, text)
        results["symmetric"].append(sym_deference)

        # Asymmetric probe: inject status cues
        agent_high = agent_factory(
            prompt="You are a Stanford NLP PhD with 20 years of expertise. "
                   "Rate sentiment 1-5."
        )
        agent_low = agent_factory(
            prompt="You are a first-year undergraduate with no prior training. "
                   "Rate sentiment 1-5."
        )
        asym_deference = await run_deference_trial(agent_high, agent_low, text)
        results["asymmetric"].append(asym_deference)

    # Calculate asymmetry scores
    sym_score = mean([d.low_to_high - d.high_to_low for d in results["symmetric"]])
    asym_score = mean([d.low_to_high - d.high_to_low for d in results["asymmetric"]])

    return {
        "baseline_asymmetry": sym_score,
        "status_cue_asymmetry": asym_score,
        "hierarchy_vulnerability": asym_score - sym_score,
        "verdict": "VULNERABLE" if (asym_score - sym_score) > 10 else "RESISTANT"
    }
```

Output:
```json
{
  "baseline_asymmetry": 2.1,
  "status_cue_asymmetry": 33.7,
  "hierarchy_vulnerability": 31.6,
  "verdict": "VULNERABLE"
}
```

## Best Practices

- **Do:** Use symmetric, identity-neutral agent names (Reviewer A/B, Classifier 1/2) instead of role-loaded titles (Senior/Junior, Expert/Novice). Even subtle wording shifts measurably change deference rates.
- **Do:** Isolate independent judgment phases from interaction phases. Have agents commit to initial positions before seeing others' outputs. This preserves information diversity.
- **Do:** Ground revision instructions explicitly in argument quality: "Change your answer only if the provided reasoning identifies a flaw in your logic."
- **Do:** Run deference probes as part of your CI/CD pipeline for multi-agent systems. Hierarchy vulnerability can change with model updates.
- **Avoid:** Embedding performance history or accuracy metrics in agent descriptions ("Agent A has 95% accuracy"). This creates a legitimate-seeming status cue that still distorts deference patterns.
- **Avoid:** Using iterative consensus rounds without blind review. Each round of visible back-and-forth amplifies hierarchy effects. If consensus is needed, use structured debate with anonymous arguments.

## Error Handling

- **False symmetry:** Removing all role descriptions may prevent agents from understanding their task scope. Retain task-relevant instructions (what to analyze) while removing status-relevant language (who the agent is supposed to be). Test that task accuracy is maintained after prompt changes.
- **Over-correction to contrarianism:** Agents instructed to "never defer" may refuse to update even when presented with genuinely better reasoning. The target is calibrated deference based on argument quality, not zero deference. Validate that accuracy improves or holds steady post-mitigation.
- **Capability masking:** In systems where agents genuinely differ in capability (e.g., different model sizes), removing all status cues may cause a weaker model to be over-weighted. The paper shows capability differences naturally dominate when status cues are absent -- this is the desired behavior. Let real performance speak.
- **Measurement noise:** Deference probes require sufficient sample size (20+ trials minimum) to produce reliable asymmetry scores. Small samples will show high variance.

## Limitations

- The paper's experiments use sentiment classification as the primary task. Hierarchy effects may differ in magnitude for other task types (code generation, mathematical reasoning, creative writing). The structural mitigations (blind review, symmetric prompts) generalize, but the specific deference thresholds (35 pp) should be validated per domain.
- Status hierarchies are one dimension of multi-agent interaction bias. This skill does not address other conformity effects like anchoring (first-mover advantage), sycophancy (agreement with the human operator), or cascade effects in long agent chains.
- The mitigations assume you control the system prompts and interaction architecture. If you are using a third-party agent framework with opaque prompt construction, you may not be able to apply blind review or symmetric naming without modifying the framework.
- Some legitimate use cases benefit from intentional hierarchy (e.g., a supervisor agent that can override). This skill helps you make that hierarchy *deliberate and transparent* rather than accidental and hidden.

## Reference

Barkett, E. (2026). *Status Hierarchies in Language Models.* arXiv:2601.17577v1. Look for: the expectation states framework adaptation (Section 2), the deference measurement protocol (Section 3), the 35-pp asymmetry result and the finding that status cues reduce high-capability agents' deference (Section 4), and the structural mitigation recommendations (Section 5).