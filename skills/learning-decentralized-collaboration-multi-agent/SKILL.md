---
name: "learning-decentralized-collaboration-multi-agent"
description: "Design and orchestrate decentralized multi-LLM collaboration systems using Multi-Agent Actor-Critic (MAAC) patterns from CoLLM-CC/DC. Decomposes complex tasks into role-specialized parallel agents with centralized or decentralized critic evaluation. Use when: 'set up multi-agent collaboration', 'build a decentralized agent pipeline', 'coordinate parallel LLM agents', 'design actor-critic multi-agent system', 'implement CoLLM-style agent collaboration', 'create role-decomposed agent workflows'."
---

# Decentralized Multi-Agent LLM Collaboration with Actor-Critic Coordination

This skill enables Claude to design, implement, and orchestrate **decentralized multi-LLM collaboration systems** based on the Multi-Agent Actor-Critic (MAAC) framework from *Learning Decentralized LLM Collaboration with Multi-Agent Actor Critic* (Liu et al., 2026). The core technique uses Centralized Training with Decentralized Execution (CTDE): agents run inference independently in parallel (each with its own policy/role), while a shared or per-agent critic provides stable training signal. Claude applies this pattern to decompose user tasks into role-specialized agents, define joint reward functions, choose between centralized (CoLLM-CC) or decentralized (CoLLM-DC) evaluation, and wire up the collaboration pipeline.

## When to Use

- When the user asks to **build a multi-agent system** where LLM agents collaborate on a shared task (writing, coding, analysis)
- When designing **parallel agent pipelines** where agents must work independently but contribute to a single output
- When the user wants to **decompose a complex task into specialized roles** (e.g., one agent plans, another implements, a third reviews)
- When choosing between **centralized vs. decentralized coordination** strategies for agent teams
- When setting up **MARL-based fine-tuning** of multiple LLMs to cooperate using the CoMLRL framework
- When the user needs to **reduce variance in multi-agent training** by replacing Monte Carlo rollouts with actor-critic methods
- When orchestrating agents for **long-horizon or sparse-reward tasks** where naive independent evaluation fails

## Key Technique

The paper identifies a fundamental problem with existing multi-agent LLM collaboration: most systems use predefined execution protocols that require a central coordinator at runtime. The MAAC framework instead trains agents to collaborate effectively during a centralized training phase, then discards the coordination scaffolding so agents execute fully in parallel at inference time. This is the CTDE principle applied to LLM collaboration.

**Two architectures are proposed.** CoLLM-CC uses a **centralized critic** — a single value function `V_phi(h_t)` that conditions on the *joint* history of all agents — to compute temporal-difference advantages `delta_t = r_t + gamma * V(h_{t+1}) - V(h_t)`. Because the critic sees all agents' states, it produces low-variance gradient estimates even with a single sample. CoLLM-DC uses **decentralized critics** — each agent `i` has its own value function `V_phi_i(h_{i,t})` that only sees that agent's local history. This is simpler to deploy but suffers from non-stationarity (other agents' changing policies look like environment noise to each local critic).

**The practical takeaway:** Use CoLLM-CC (centralized evaluation) for long-horizon tasks, sparse rewards, or when agents must tightly coordinate. Use CoLLM-DC (decentralized evaluation) for short-horizon, dense-reward tasks where independent assessment is sufficient. Monte Carlo approaches (no critic at all) work only when you can afford many rollout samples and the task horizon is short. Role decomposition through distinct prompts — not algorithmic mechanisms — drives specialization.

## Step-by-Step Workflow

1. **Analyze the task and determine if multi-agent decomposition is warranted.** Check whether the task has separable sub-components (e.g., planning vs. implementation, summarization vs. detail expansion, code logic vs. test writing). Single-focus tasks don't benefit from multi-agent collaboration.

2. **Define agent roles through distinct system prompts.** Each agent gets a role-specific prompt that constrains its contribution. Follow the paper's pattern: for writing, assign "high-level summarizer" and "detailed analyst"; for coding, assign "core logic architect" and "utility implementer"; for analysis, assign "researcher" and "synthesizer." Roles should be complementary, not overlapping.

3. **Choose the critic architecture based on task characteristics.**
   - **CoLLM-CC (centralized critic):** Use when the task is long-horizon (multiple rounds of refinement), has sparse rewards (only final output quality matters), or requires tight coordination between agents.
   - **CoLLM-DC (decentralized critics):** Use when the task is short-horizon (single-pass generation), has dense rewards (each agent's output is independently evaluable), or agents operate on clearly separable sub-tasks.
   - **No critic (Monte Carlo):** Use only for quick prototyping or when sample budget is unlimited.

4. **Define the joint reward function.** All agents share the same reward signal. Compose it from measurable metrics relevant to the domain:
   - *Writing:* weighted sum of coverage balance (length ratio), style consistency (Jaccard similarity), logical coherence (transition word frequency)
   - *Coding:* pass@k on test cases, with static analysis (AST parsing) and dynamic testing (sandbox execution) feedback
   - *Analysis:* factual accuracy, completeness, consistency between agent outputs

5. **Implement the collaboration loop.** Structure agent interaction as turns `t = 0, ..., H-1`:
   - Each agent generates its response `a_{i,t}` from its policy given its local history `h_{i,t}`
   - Collect all agent outputs into a joint action `a_t`
   - Compute the reward `r_t` from the joint output using the reward function
   - Update each agent's history with new observations (feedback from environment or other agents' outputs)
   - Store transition tuples `(h_t, a_t, r_t, h_{t+1})` in a replay buffer

6. **Implement the evaluation/critic module.** For CoLLM-CC: concatenate all agents' KV-cache histories into a joint representation and pass through the critic network. For CoLLM-DC: each agent's critic receives only that agent's history. Compute TD advantages: `delta_t = r_t + gamma * V(h_{t+1}) - V(h_t)`.

7. **Wire up the training loop with off-policy corrections.** Sample minibatches from the replay buffer. Compute importance sampling ratios `rho_{i,t} = pi_theta(a_{i,t}|h_{i,t}) / pi_old(a_{i,t}|h_{i,t})` to correct for the behavior policy drift. Update critic by minimizing TD loss, then update each actor using the policy gradient weighted by advantages and importance ratios.

8. **Set up decentralized execution.** At inference, discard all critics. Each agent runs independently using only its policy `pi_theta_i` and local KV cache. Agents execute in parallel — no central coordinator needed. Aggregate outputs according to the task structure (concatenation, merging, voting).

9. **Validate the pipeline end-to-end.** Run the full collaboration on representative inputs. Check that: (a) each agent contributes its specialized role, (b) the joint output outperforms single-agent baselines, (c) agents don't duplicate each other's work, (d) the reward signal differentiates good from bad collaboration.

10. **Iterate on role prompts and reward weights.** The most impactful lever is role prompt design. If agents produce redundant outputs, sharpen role boundaries. If the joint output lacks coherence, increase the weight on consistency metrics in the reward function.

## Concrete Examples

**Example 1: Collaborative Code Generation Pipeline**

User: "Set up a two-agent system where one agent writes the core algorithm and another writes tests and utilities."

Approach:
1. Define Agent A (Core Logic) with prompt: "You are the primary code architect. Write the main algorithm, data structures, and core functions. Do not write tests or utility helpers."
2. Define Agent B (Test & Utility) with prompt: "You are the testing and utility specialist. Given the core code from your partner, write comprehensive unit tests and any helper utilities needed. Do not rewrite core logic."
3. Select CoLLM-CC architecture — coding tasks have sparse rewards (pass/fail on tests) and require tight coordination.
4. Define reward: `r = pass_rate(test_suite) * utilization_score(agent_b_uses_agent_a_code)`
5. Run collaboration: Agent A generates core code -> Agent B receives it and writes tests/utilities -> execute test suite -> compute joint reward.

Output structure:
```python
# Agent A output: core_algorithm.py
def merge_sort(arr: list[int]) -> list[int]:
    if len(arr) <= 1:
        return arr
    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])
    return _merge(left, right)

def _merge(left, right):
    result, i, j = [], 0, 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i]); i += 1
        else:
            result.append(right[j]); j += 1
    return result + left[i:] + right[j:]

# Agent B output: test_core.py + utils.py
import pytest
from core_algorithm import merge_sort

def random_array(n, lo=0, hi=1000):
    import random; return [random.randint(lo, hi) for _ in range(n)]

@pytest.mark.parametrize("inp,expected", [
    ([], []),
    ([1], [1]),
    ([3,1,2], [1,2,3]),
    ([5,5,5], [5,5,5]),
])
def test_merge_sort_basic(inp, expected):
    assert merge_sort(inp) == expected

def test_merge_sort_large():
    arr = random_array(10000)
    assert merge_sort(arr) == sorted(arr)
```

Reward computation: 5/5 tests pass -> `pass_rate = 1.0`; Agent B imports and exercises Agent A's functions -> `utilization = 1.0`; joint reward `r = 1.0`.

---

**Example 2: Collaborative Document Summarization**

User: "Build a two-agent summarization system for long research papers — one agent captures the big picture, the other captures technical details."

Approach:
1. Define Agent A (High-Level Summarizer): "Produce a 3-sentence executive summary covering the paper's motivation, key contribution, and main result. Avoid technical details."
2. Define Agent B (Detail Summarizer): "Produce a detailed summary covering methodology, experimental setup, and specific numerical results. Avoid restating the high-level motivation."
3. Select CoLLM-DC architecture — summarization is short-horizon (single pass) with dense rewards (each summary is independently evaluable for coverage).
4. Define reward: `r = 0.4 * coverage_balance + 0.3 * style_consistency + 0.3 * coherence`
   - `coverage_balance`: ratio of summary lengths should be within [0.5, 2.0]
   - `style_consistency`: Jaccard similarity of writing style tokens
   - `coherence`: presence of logical connectors between the two summaries when concatenated
5. Each agent independently generates its summary. Concatenate with a section break. Evaluate joint reward.

Output structure:
```markdown
## Executive Summary (Agent A)
This paper addresses the challenge of training multiple LLMs to collaborate
without a central coordinator. The key contribution is two actor-critic
methods (CoLLM-CC and CoLLM-DC) that enable parallel decentralized
execution after centralized training. Results show CoLLM-CC significantly
outperforms baselines on long-horizon tasks.

## Technical Details (Agent B)
The authors implement CTDE using TD(0) advantages with importance sampling
for off-policy correction. On MBPP coding benchmarks, CoLLM-CC achieves
78.2% pass@1 vs. 71.4% for Monte Carlo baselines. On Minecraft HouseBuild,
CoLLM-CC reaches 0.82 IoU while CoLLM-DC plateaus at 0.64. Training uses
replay buffers with minibatch size 64 over 50 epochs.
```

---

**Example 3: Choosing the Right Architecture**

User: "I have a multi-step data pipeline: extract data, transform it, validate it, then load it. Should I use centralized or decentralized critics?"

Approach:
1. Assess task characteristics:
   - Horizon: H=4 steps (extract -> transform -> validate -> load) — **long-horizon** relative to single-pass tasks
   - Reward density: final success is only known after load completes — **sparse reward**
   - Agent coupling: transform depends on extract's output format, validate depends on transform's schema — **tight coupling**
2. All three indicators point to **CoLLM-CC (centralized critic)**.
3. Rationale: With sparse rewards, decentralized critics can't assign credit to individual steps. A centralized critic seeing all agents' states can learn that a bad extract format causes downstream failures, even though the immediate extract step produced output.

Output:
```
Architecture Decision: CoLLM-CC (Centralized Critic)

Reasoning:
- Long horizon (4 sequential steps) -> MC methods need K^4 samples
- Sparse reward (success only at pipeline end) -> DC critics can't converge
- Tight agent coupling (each step depends on prior) -> CC captures cross-agent dependencies

Implementation:
- Single critic V_phi(h_extract, h_transform, h_validate, h_load)
- Joint history concatenates all agent KV caches
- TD advantage at each step: delta_t = r_t + gamma * V(h_{t+1}) - V(h_t)
- At inference: each pipeline stage runs independently, no critic needed
```

## Best Practices

**Do:**
- Define agent roles through explicit, non-overlapping system prompts — role specialization is the primary mechanism for effective collaboration
- Use CoLLM-CC for any task where the final reward depends on how agents' outputs compose together, not just individual quality
- Include a `coverage_balance` or `utilization` term in the reward to prevent one agent from dominating while another idles
- Store transitions in a replay buffer and use off-policy corrections (importance sampling) to get more training signal per sample
- Start with 2 agents and scale up only if the task decomposition clearly warrants it — coordination overhead grows quadratically

**Avoid:**
- Don't use CoLLM-DC for sparse-reward or long-horizon tasks — the paper shows it fails to converge in these settings
- Don't rely on Monte Carlo methods when the task horizon exceeds 2-3 steps — sample complexity grows exponentially as K^H
- Don't let agents see each other's raw outputs during decentralized execution — this violates the decentralization principle and creates hidden dependencies
- Don't design overlapping roles — if both agents can do the same sub-task, they will produce redundant output and the reward signal becomes noisy
- Don't skip the reward function design — vague or single-metric rewards lead to mode collapse where agents converge to a degenerate collaboration pattern

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| Non-stationarity in CoLLM-DC | Critic loss oscillates, never converges | Switch to CoLLM-CC or freeze one agent's policy while training the other |
| Agent role collapse | Both agents produce near-identical outputs | Sharpen role prompts with explicit exclusion constraints ("Do NOT write X") |
| Reward hacking | High reward but low actual quality | Add a secondary metric (e.g., human eval proxy) or clip reward range |
| Importance ratio explosion | Policy gradient updates become unstable | Clip importance ratios to [1-epsilon, 1+epsilon] as in PPO; reduce replay buffer age |
| Sparse reward starvation | No learning signal for early-stage agents | Add intermediate shaped rewards at each step or use the centralized critic to bootstrap value estimates |
| KV cache memory overflow | OOM when concatenating joint histories for CC | Truncate older history turns or use a sliding window over the last N turns |

## Limitations

- **Requires a well-defined reward function.** If you can't quantify what "good collaboration" looks like for your task, the framework provides no benefit over prompting.
- **Training overhead is substantial.** MAAC requires multiple rollouts, replay buffers, and critic network training. For one-off tasks, direct prompting or chain-of-thought is more practical.
- **Two agents is the sweet spot.** The paper only evaluates 2-agent collaboration. Scaling to 3+ agents introduces combinatorial complexity in joint history representation and reward attribution.
- **Role decomposition is manual.** The framework does not automatically discover optimal role assignments — you must design them based on domain knowledge.
- **CoLLM-CC joint history scales linearly with agent count.** For many agents with long contexts, the centralized critic's input becomes prohibitively large.
- **Not suited for adversarial or competitive settings.** The framework assumes fully cooperative agents with shared rewards.

## Reference

Liu, S., Chen, T., Amiri, R., & Amato, C. (2026). *Learning Decentralized LLM Collaboration with Multi-Agent Actor Critic.* arXiv:2601.21972v2. Code: [github.com/OpenMLRL/CoMLRL](https://github.com/OpenMLRL/CoMLRL/releases/tag/v1.3.2). Key insight: Centralized critics enable stable training of decentralized LLM agents by providing low-variance advantage estimates, with the critic discarded at inference time for fully parallel execution.