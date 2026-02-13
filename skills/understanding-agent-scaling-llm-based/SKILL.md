---
name: "understanding-agent-scaling-llm-based"
description: "Design diversity-aware multi-agent systems that maximize performance with fewer agents. Uses information-theoretic K* effective channel analysis to replace brute-force agent scaling with principled heterogeneous configurations. Trigger phrases: 'optimize my agent swarm', 'why are my agents not improving with scale', 'design a diverse multi-agent system', 'K-star analysis for agents', 'reduce agent count without losing accuracy', 'heterogeneous agent configuration'"
---

# Diversity-Aware Multi-Agent System Design

This skill enables Claude to design, analyze, and optimize LLM-based multi-agent systems using the information-theoretic framework from Yang et al. (2026). Instead of naively adding more identical agents (which saturates quickly due to correlated outputs), this skill applies the principle that **2 diverse agents can match or exceed 16 homogeneous agents** by maximizing effective information channels through model, prompt, and tool heterogeneity.

## When to Use

- When the user is building a multi-agent system and wants to determine the right number and configuration of agents
- When an existing agent swarm shows diminishing returns as more agents are added
- When the user asks how to make agents "work together better" or "stop agreeing with each other"
- When designing voting, debate, or ensemble architectures and choosing how to differentiate agents
- When the user wants to reduce API costs by using fewer agents without losing quality
- When evaluating whether a multi-agent approach is actually outperforming a single-agent baseline
- When configuring agent personas, model mixtures, or tool assignments for a collaborative task

## Key Technique

### The Information Budget Ceiling

Every task has a finite amount of extractable information, quantified as the conditional entropy `H(Y|X)` -- the intrinsic uncertainty of the correct answer given the input. No multi-agent system, regardless of agent count, can exceed this ceiling. Performance is bounded by `I_MAS(n) <= H(Y|X)`, where `I_MAS(n)` is the mutual information extracted by n agents. The practical consequence: adding agents only helps if each new agent provides **complementary evidence** that reduces residual uncertainty.

### Why Homogeneous Scaling Fails

When agents share the same model, prompt, and tools, their outputs are highly correlated. The effective information contributed by agent n+1 is nearly zero if it just echoes agents 1 through n. Formally, performance improves as `1 - e^{-alpha*K}` where `alpha` is the complementarity rate (probability a new channel provides missing evidence) and `K` is the effective channel count. Homogeneous agents push K toward 1 regardless of n, so the exponent stays small and performance plateaus around 4 agents.

### K* -- Measuring Effective Diversity Without Labels

The K* metric (effective channel count) quantifies how many independent information directions the agents actually span, computed without ground-truth labels:

1. Embed each agent's output using a sentence transformer
2. Build the cosine-similarity Gram matrix G across all agent outputs
3. Trace-normalize: `rho = G / Tr(G)`
4. Compute eigenvalues `{lambda_j}` of rho
5. `K* = 2^H(rho)` where `H(rho) = -sum(lambda_j * log2(lambda_j))`

When K* is close to 1, agents are redundant. When K* approaches n, each agent contributes unique information. The product `alpha * K` governs total performance -- optimizing for higher K* with fewer agents is strictly better than scaling n with low K*.

## Step-by-Step Workflow

1. **Characterize the task type.** Determine whether the task is reasoning-heavy (math, logic, code -- where K* strongly predicts accuracy) or knowledge-heavy (trivia, common sense -- where K* correlation is weaker). This determines how aggressively to invest in diversity.

2. **Establish a single-agent baseline.** Run one agent on a representative sample. Measure accuracy. This is your floor -- any multi-agent system that doesn't beat this is wasting compute.

3. **Design diversity layers.** Apply heterogeneity at up to four levels, each additive:
   - **L1 (None):** Same model, same prompt -- baseline homogeneous
   - **L2 (Persona diversity):** Same model, distinct system prompts that enforce different reasoning strategies (e.g., "step-by-step verifier" vs. "pattern-matching estimator" vs. "formal proof writer")
   - **L3 (Model diversity):** Different base models (e.g., Claude + GPT-4o + Llama-3.1), same prompt
   - **L4 (Full diversity):** Different models AND distinct personas -- maximum effective channels

4. **Assign complementary personas.** For each agent, write a system prompt that enforces a distinct cognitive strategy. Good archetypes for reasoning tasks:
   - **Conservative Verifier:** "Check every step. Flag errors. Show your verification."
   - **Creative Explorer:** "Look for shortcuts, patterns, and unconventional approaches."
   - **Rigorous Formalist:** "Use precise notation. State assumptions. Prove each claim."
   - **Intuitive Estimator:** "Estimate the answer first, then verify. Flag suspiciously large/small values."
   - **Systematic Decomposer:** "Break the problem into independent subproblems. Solve each separately."

5. **Choose an aggregation architecture.** Use **majority voting** for tasks with discrete answers (classification, multiple-choice, yes/no). Use **multi-round debate** (3-4 rounds) for open-ended reasoning where agents can refine each other's arguments. Use **hub-and-spoke** (centralized) topology when one strong model should synthesize weaker agents' outputs.

6. **Right-size the agent count.** Start with 2-4 fully diverse agents (L4). Homogeneous systems plateau at ~4 agents; heterogeneous systems see diminishing returns around 8. Going beyond 8 agents rarely justifies the cost.

7. **Measure K* on a sample.** Embed agent outputs from a held-out sample using a sentence transformer. Compute the Gram matrix, eigenvalues, and effective rank. If K* < 2 with n agents, your agents are too similar -- increase diversity before increasing count.

8. **Analyze K* sub-metrics.** Compute K*_conditioned (diversity among correct answers vs. incorrect answers). A healthy system has high diversity in correct reasoning paths and fragmented/low-diversity incorrect paths. If incorrect answers are highly diverse, your aggregation may amplify noise.

9. **Iterate on the weakest diversity dimension.** If K* is low despite persona diversity, add model diversity. If model diversity doesn't help, the task may have low intrinsic uncertainty (H(Y|X) is small) and a single strong agent suffices.

10. **Deploy with cost monitoring.** Track accuracy per API dollar. The goal is maximum accuracy at minimum agent count. A well-configured 2-agent L4 system should match a 16-agent L1 system at 1/8 the cost.

## Concrete Examples

**Example 1: Optimizing a code review multi-agent system**

User: "I have 8 GPT-4o agents doing code review with majority vote but adding more agents stopped improving results around agent 5. How do I fix this?"

Approach:
1. Diagnose: 8 identical agents = L1 (no diversity). Outputs are highly correlated, K* likely near 1-2.
2. Reduce to 3 agents with L4 diversity:
   - Agent 1: Claude Opus with persona "security auditor -- focus on injection, auth, data exposure"
   - Agent 2: GPT-4o with persona "performance reviewer -- focus on algorithmic complexity, memory leaks, unnecessary allocations"
   - Agent 3: Llama-3.1-70B with persona "correctness verifier -- trace logic paths, check edge cases, verify invariants"
3. Aggregate with weighted vote (weight by model confidence or use a hub model to synthesize).

Output:
```
Configuration:
  agents: 3 (down from 8)
  diversity_level: L4 (model + persona)
  aggregation: weighted_vote

  agent_1:
    model: claude-opus-4-6
    persona: "You are a security-focused code reviewer. Analyze code for OWASP Top 10 vulnerabilities, authentication flaws, injection risks, and data exposure. Ignore style issues."

  agent_2:
    model: gpt-4o
    persona: "You are a performance-focused code reviewer. Analyze algorithmic complexity, memory allocation patterns, unnecessary copies, and potential bottlenecks. Ignore security unless it causes crashes."

  agent_3:
    model: llama-3.1-70b
    persona: "You are a correctness-focused code reviewer. Trace every logical branch, check boundary conditions, verify loop invariants, and test edge cases mentally. Ignore style and performance."

  expected_K_star: ~2.5-3.0 (vs ~1.3 for 8 identical agents)
```

**Example 2: Building a diverse research synthesis swarm**

User: "I want agents to research a topic and synthesize findings. How many agents and what configuration?"

Approach:
1. Research synthesis is reasoning-heavy -- K* is a strong predictor here.
2. Design 4 agents at L4 diversity with debate aggregation (3 rounds):
   - Agent 1 (Claude): "Methodological critic -- evaluate study design, sample sizes, statistical validity"
   - Agent 2 (GPT-4o): "Contrarian -- actively seek evidence against the emerging consensus, find counterexamples"
   - Agent 3 (Gemini): "Synthesizer -- find connections across sources, identify meta-patterns, build unified narratives"
   - Agent 4 (Llama-3.1-70B): "Domain specialist -- focus on technical accuracy, correct use of terminology, factual precision"
3. Use 3 rounds of debate where each agent sees and responds to others' outputs.

Output:
```
Architecture: debate (3 rounds, decentralized mesh)
Agents: 4 (L4 diversity)

Round 1: Each agent independently researches and produces findings
Round 2: Each agent critiques others' findings from their persona's lens
Round 3: Agents converge on final synthesis, flagging unresolved disagreements

Key: The contrarian agent prevents premature consensus.
The methodological critic catches weak evidence before it enters the synthesis.
Model diversity ensures different training data biases cancel out.
```

**Example 3: Reducing costs on a classification pipeline**

User: "I'm running 16 identical Claude Haiku agents with majority vote for sentiment classification. It's expensive. Can I do better?"

Approach:
1. 16 homogeneous agents is extreme L1 -- performance likely plateaued at agent 4-5.
2. Sentiment classification has relatively low intrinsic uncertainty (H(Y|X) is small for clear-sentiment text).
3. Replace with 3 agents at L3 diversity (model diversity, same prompt):
   - Claude Haiku
   - GPT-4o-mini
   - Gemma-2-9B
4. Use simple majority vote -- no debate needed for classification.

Output:
```
Before: 16x Claude Haiku, majority vote
  cost: 16 API calls per item
  accuracy: ~89.2% (plateaued at agent 5)

After: 3 diverse models, majority vote
  cost: 3 API calls per item (81% reduction)
  accuracy: ~89.8% (higher due to decorrelated errors)

Why it works: Different models make different mistakes.
Majority vote on uncorrelated errors is far more powerful
than majority vote on correlated errors. K* jumps from
~1.2 (16 homogeneous) to ~2.7 (3 heterogeneous).
```

## Best Practices

**Do:**
- Start with the minimum viable agent count (2-3) and add diversity before adding agents
- Use personas that enforce genuinely different **reasoning strategies**, not just different phrasings of the same approach
- Measure K* on a held-out sample before deploying -- if K* < 1.5 with multiple agents, redesign before scaling
- Combine model diversity (L3) with persona diversity (L2) for maximum effect (L4)
- Use debate aggregation for open-ended tasks and voting for discrete-answer tasks
- Monitor the ratio of K*_correct to K*_incorrect -- you want diverse correct paths and fragmented incorrect ones

**Avoid:**
- Adding more agents of the same type to "improve reliability" -- this is the homogeneous scaling trap
- Using generic personas like "you are a helpful assistant" vs "you are a very helpful assistant" -- these produce nearly identical outputs (K* stays near 1)
- Scaling beyond 8 agents without first confirming K* is actually increasing
- Applying this framework to tasks with very low intrinsic uncertainty -- if a single agent already achieves >95% accuracy, multi-agent overhead isn't justified
- Using debate for simple classification tasks -- the interaction overhead doesn't pay off when voting suffices

## Error Handling

**K* stays near 1 despite different personas:** The personas are too similar in practice. Rewrite them to enforce structurally different reasoning approaches (e.g., forward chaining vs. backward chaining, formal vs. intuitive). Test by checking if agents actually produce different intermediate steps, not just different final phrasings.

**Accuracy drops with heterogeneous agents:** One or more agents may be too weak for the task. Check individual agent accuracy -- an agent that's wrong more than it's right will poison the vote. Either remove it or weight its vote lower. Ensure correct-path diversity (K*_correct) exceeds incorrect-path diversity (K*_incorrect).

**Debate converges to wrong answer:** The aggregation is amplifying a shared bias across models. Add a "contrarian" agent persona explicitly instructed to argue against the majority. Or switch to voting (which preserves independent judgments) when debate causes herding.

**Diminishing returns appear at 3-4 agents even with diversity:** The task may have low intrinsic uncertainty H(Y|X). Check single-agent accuracy -- if it's already above 90%, the ceiling is close and multi-agent gains will be marginal regardless of configuration.

## Limitations

- K* requires embedding agent outputs with a sentence transformer, which adds inference cost and may not capture all dimensions of semantic diversity (e.g., reasoning structure vs. surface language)
- The framework assumes agents operate on the same input -- it doesn't directly address task decomposition or agent specialization on different subtasks
- K* correlates strongly with accuracy on reasoning tasks (math, logic, code) but weakly on knowledge-retrieval tasks (trivia, commonsense) where the bottleneck is training data coverage, not reasoning diversity
- The theoretical bounds assume conditional independence between effective channels, which is an approximation -- real agent outputs have complex dependency structures
- Persona engineering remains somewhat artisanal; there is no automated method to generate maximally diverse personas for an arbitrary task

## Reference

Yang, Y., Qu, C., Wen, M., Shi, L., & Wen, Y. (2026). *Understanding Agent Scaling in LLM-Based Multi-Agent Systems via Diversity.* arXiv:2602.03794v1. [https://arxiv.org/abs/2602.03794v1](https://arxiv.org/abs/2602.03794v1)

Look for: Theorem 3.2 (finite information budget), Definition 4.5 (K* effective channel count), Table 2 (L1-L4 diversity layer comparison), and Section 5.3 (design guidelines). Code: [https://github.com/SafeRL-Lab/Agent-Scaling](https://github.com/SafeRL-Lab/Agent-Scaling)