---
name: "dynamic-role-assignment-multi-agent"
description: "Dynamically assign specialized roles to multiple AI agents via a meta-debate protocol (proposal + peer review) before running the actual task. Use when: 'set up a multi-agent debate', 'assign agent roles dynamically', 'which model should handle which role', 'run a meta-debate to pick agents', 'optimize role assignment for my agent swarm', 'capability-aware agent selection'."
---

# Dynamic Role Assignment for Multi-Agent Debate

This skill implements the Dynamic Role Assignment framework from Zhang et al. (2026), which runs a **Meta-Debate** before any multi-agent task to determine which agent (or model) is best suited for each role. Instead of statically assigning every role to the same model or randomly picking, agents first generate role-specific proposals, then peer-review each other's proposals using scored criteria. The highest-scoring agent for each role wins the assignment. Applied on top of existing debate systems, this approach outperforms uniform assignment by up to 74.8% and random assignment by up to 29.7%.

## When to Use

- When orchestrating a multi-agent workflow where different roles require different strengths (e.g., code reviewer vs. implementer vs. test writer)
- When you have access to multiple models or agent configurations and need to decide which handles which subtask
- When building a debate-style reasoning system with roles like Affirmative, Negative, and Judge
- When a user asks to "set up agents for a complex problem" and you need to allocate responsibilities
- When running a multi-agent code review, architecture discussion, or adversarial red-team exercise
- When you want to avoid the default pattern of assigning every role to the same model

## Key Technique

The core insight is that **role assignment should be treated as an optimization problem, not an assumption**. Most multi-agent systems assign every role to the same model (uniform) or pick randomly. Dynamic Role Assignment adds a lightweight preliminary round -- the Meta-Debate -- that empirically tests each candidate agent's fitness for each role before committing.

The Meta-Debate has two stages. In the **Proposal Stage**, every candidate agent generates a response for every role, given the actual task. For example, if there are 3 roles and 3 candidate models, this produces 9 proposals. In the **Peer Review Stage**, each agent evaluates all proposals for a given role against automatically generated, role-specific criteria (e.g., "Accuracy 1-5", "Technical Depth 1-5", "Argumentative Strength 1-5"). Scores are averaged across all evaluators, and the agent with the highest mean score for each role wins that assignment.

This works because different models genuinely excel at different things: one may be better at structured argumentation (Affirmative role), while another excels at finding flaws (Negative role) or synthesizing conclusions (Judge role). The Meta-Debate surfaces these differences cheaply before committing to a full multi-round debate.

## Step-by-Step Workflow

1. **Define the role set.** List every distinct role the task requires. For a debate: Affirmative, Negative, Judge. For code tasks: Architect, Implementer, Reviewer, Tester. Each role needs a one-paragraph description of its responsibilities and success criteria.

2. **Define the candidate agent set.** List every available agent or model configuration. These can be different models (GPT-4o, Claude, Gemini), different prompt strategies (Chain-of-Thought, Step-Back, Program-of-Thoughts), or different tool configurations. You need at least as many candidates as roles.

3. **Generate role-specific evaluation criteria.** For each role, produce 3-5 scored criteria (1-5 scale) that define what "good" looks like. Use the task context to make criteria specific. For a Reviewer role on a security audit: "Vulnerability Detection Accuracy (1-5)", "False Positive Rate (1-5, lower is better)", "Explanation Clarity (1-5)".

4. **Run the Proposal Stage.** For every (role, candidate) pair, prompt the candidate with the actual task and the role description. Collect the response as a proposal. Template: `"You are acting as [ROLE]. [ROLE_DESCRIPTION]. Here is the task: [TASK]. Provide your response demonstrating how you would fulfill this role."` Store all proposals as `P[role][candidate]`.

5. **Run the Peer Review Stage.** For each role, send all proposals to every evaluator agent with the scoring criteria. Template: `"Evaluate each candidate's suitability for the [ROLE] role. Score each proposal on the following criteria (1-5 each): [CRITERIA_LIST]. Return scores as JSON."` Collect scores as `S[role][candidate][evaluator]`.

6. **Aggregate scores and assign roles.** For each role, compute the mean score across all evaluators for each candidate: `mean_score[role][candidate] = avg(S[role][candidate][*])`. Assign each role to `argmax(mean_score[role])`. If a candidate wins multiple roles, assign the role where their margin of victory is largest, then re-assign the other role to the next-best candidate.

7. **Handle conflicts.** If one agent is the best candidate for multiple roles, use a greedy assignment: sort all (role, candidate) pairs by score descending, assign greedily ensuring each candidate fills at most one role (unless candidates outnumber roles or reuse is acceptable).

8. **Execute the main task.** Run the actual multi-agent debate or workflow with the optimized role assignments. The Meta-Debate is now complete; proceed with the assigned configuration.

9. **Log the assignment rationale.** Record which agent was assigned to which role and their scores. This provides an audit trail and helps calibrate future assignments.

10. **Skip the Meta-Debate when unnecessary.** For trivial tasks where all candidates perform equally, or when only one candidate is available, bypass the meta-debate to save tokens. A heuristic: if the task is estimated to be very easy or very hard (all candidates will succeed or fail equally), skip.

## Concrete Examples

**Example 1: Multi-Agent Code Review**

```
User: I have a PR with security-sensitive changes. Set up agents to review it thoroughly.

Approach:
1. Define roles:
   - Security Reviewer: Focus on vulnerabilities, injection risks, auth flaws
   - Logic Reviewer: Focus on correctness, edge cases, algorithmic issues
   - Style/Maintainability Reviewer: Focus on readability, patterns, tech debt

2. Define candidates (3 agent configurations):
   - Agent A: Claude with security-focused system prompt
   - Agent B: Claude with chain-of-thought reasoning prompt
   - Agent C: Claude with code-quality checklist prompt

3. Generate criteria for each role:
   Security Reviewer: Vulnerability Detection (1-5), OWASP Coverage (1-5), False Positive Rate (1-5)
   Logic Reviewer: Edge Case Coverage (1-5), Reasoning Depth (1-5), Correctness (1-5)
   Style Reviewer: Actionability of Suggestions (1-5), Pattern Recognition (1-5)

4. Run proposals: Each agent reviews a representative diff chunk in each role.

5. Peer review: Agents score each other's proposals.

6. Result:
   Security Reviewer -> Agent A (avg score 4.2)
   Logic Reviewer    -> Agent B (avg score 4.5)
   Style Reviewer    -> Agent C (avg score 3.9)

7. Execute the full code review with these assignments.
```

**Example 2: Architectural Decision Debate**

```
User: We need to decide between microservices vs monolith for our new service.
      Run a structured debate with multiple perspectives.

Approach:
1. Define roles:
   - Advocate (argues FOR microservices)
   - Critic (argues AGAINST / for monolith)
   - Judge (synthesizes arguments, makes recommendation)

2. Define candidates:
   - Agent A: Claude with "experienced distributed systems architect" persona
   - Agent B: Claude with "pragmatic startup CTO" persona
   - Agent C: Claude with "technical evaluator" persona

3. Meta-Debate proposal stage:
   Each agent writes a short argument for each role given the user's context.

4. Peer review scores (averaged):
   Advocate role:  Agent A=4.3, Agent B=3.8, Agent C=3.5
   Critic role:    Agent A=3.6, Agent B=4.4, Agent C=3.9
   Judge role:     Agent A=3.4, Agent B=3.7, Agent C=4.6

5. Assignments: A->Advocate, B->Critic, C->Judge

6. Run 3-round debate:
   Round 1: Advocate presents case, Critic rebuts
   Round 2: Both refine arguments addressing counterpoints
   Round 3: Judge synthesizes and delivers recommendation

Output: A structured recommendation document with the Judge's verdict,
key arguments from both sides, and confidence level.
```

**Example 3: Multi-Model Reasoning on a Hard Math Problem**

```
User: Solve this competition math problem using multiple reasoning strategies.

Approach:
1. Define roles (from DMAD framework):
   - Chain-of-Thought Reasoner: Step-by-step logical derivation
   - Step-Back Prompter: Abstract the problem, then solve
   - Program-of-Thoughts: Write code to compute the answer

2. Candidates: 3 different model endpoints or temperature settings.

3. Each candidate generates a proposal for each role on the actual problem.

4. Peer review evaluates:
   - Accuracy of intermediate steps (1-5)
   - Completeness of reasoning (1-5)
   - Final answer correctness (1-5, verified where possible)

5. Assign roles based on scores. Run the full debate where each
   role-holder presents their solution, they critique each other,
   and a final answer is synthesized.

Output: The agreed-upon answer with a confidence indicator based on
whether all three approaches converged.
```

## Best Practices

**Do:**
- Write role descriptions that are specific to the task, not generic. "Review this PR for SQL injection, XSS, and auth bypass" beats "review for security issues."
- Use at least 3 evaluation criteria per role, each on a 1-5 scale, to get discriminative scores.
- Include the actual task content in proposals so agents demonstrate real capability, not hypothetical fitness.
- Log all scores and assignments so you can debug poor outcomes and refine criteria over time.

**Avoid:**
- Running Meta-Debate when you only have one candidate agent -- it wastes tokens with no benefit.
- Using vague criteria like "Overall Quality (1-5)" -- this produces flat scores that don't differentiate candidates.
- Letting a single evaluator score proposals -- averaging across multiple evaluators reduces bias and noise.
- Assigning the same agent to all roles after Meta-Debate just because it scored highest everywhere -- consider diversity of perspective as a tiebreaker.

## Error Handling

- **Tie scores:** When two candidates score identically for a role, break ties by preferring the candidate whose score variance across evaluators is lower (more consistent performance).
- **All scores are low:** If no candidate scores above 2.5/5.0 for a role, the role description or criteria may be misaligned. Revise them and re-run, or fall back to uniform assignment for that role.
- **Evaluator gaming:** If an agent consistently scores itself highest (self-promotion bias), exclude self-evaluations from the aggregation. Compute `mean_score` using only other agents' reviews.
- **Token budget exceeded:** The Meta-Debate generates `|roles| x |candidates|` proposals plus `|roles| x |candidates| x |evaluators|` evaluations. If this exceeds budget, reduce candidates or use a fast model for proposals and reserve the full model for evaluation only.
- **Role conflict (one agent wins multiple roles):** Use the greedy assignment described in Step 7. Alternatively, allow role reuse if the agent pool is smaller than the role set.

## Limitations

- The Meta-Debate adds a fixed overhead of proposal generation + peer review before any actual work begins. For simple tasks, this overhead is not justified.
- Effectiveness is bounded by the capabilities of the available agents. If all candidates are weak at a role, dynamic assignment cannot conjure competence from nowhere.
- The framework assumes roles are well-defined and independent. For tasks where roles are fluid or overlapping, static role definitions may be too rigid.
- Peer review quality depends on evaluator capability. If evaluators cannot distinguish good from bad proposals, scores become noise.
- Currently best suited for discrete role assignment. Continuous collaboration patterns (pair programming, mob review) need adaptation.

## Reference

Zhang, M., Kim, J., Xiang, S., Gao, J., & Cao, C. (2026). *Dynamic Role Assignment for Multi-Agent Debate*. arXiv:2601.17152v1. https://arxiv.org/abs/2601.17152v1

Key takeaway: Section 3 details the Meta-Debate algorithm (proposal + peer review), and Tables 1-3 show consistent gains over uniform and random assignment across GPQA, MathVision, and RealWorldQA benchmarks.