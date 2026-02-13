---
name: "just-in-time-reinforcement-learning-continual"
description: "Implement JitRL-style continual learning for LLM agents: training-free policy optimization via experience memory, advantage estimation, and logit modulation. Use when asked to 'add experience memory to an agent', 'implement continual learning without fine-tuning', 'build a JitRL agent', 'optimize agent actions from past trajectories', 'add non-parametric RL to an LLM pipeline', or 'make my agent learn from its mistakes at inference time'."
---

# Just-In-Time Reinforcement Learning for LLM Agents

This skill enables Claude to implement JitRL, a training-free continual learning framework that makes LLM agents improve over time without gradient updates or fine-tuning. The core idea: maintain a non-parametric memory of past `(state, action, return)` experiences, retrieve relevant trajectories at inference time, estimate action advantages via k-NN, and apply an additive logit correction that is the provably optimal solution to the KL-constrained policy optimization objective. This gives agents the ability to learn from successes and failures across episodes while preserving the base LLM's capabilities.

## When to Use

- When building an LLM agent (web navigation, game-playing, tool-use) that must improve its decisions over repeated episodes without retraining
- When a user asks to "add memory" or "learning from experience" to an existing agentic pipeline
- When implementing test-time policy optimization that avoids catastrophic forgetting
- When the user wants to reduce API costs compared to fine-tuning approaches like WebRL or online RLHF
- When building an agent that operates in a repeatable environment (web tasks, CLI workflows, interactive fiction) where past trajectories are informative
- When adding exploration-exploitation tradeoffs to an LLM agent's action selection

## Key Technique

**The Problem.** Deployed LLMs have frozen weights. Traditional RL requires expensive gradient updates and risks catastrophic forgetting. JitRL solves the KL-constrained policy optimization objective `pi* = argmax (E[A(s,a)] - (1/beta) * D_KL(pi' || pi_theta))` in closed form, avoiding both issues.

**The Solution.** The closed-form optimal policy is `pi*(a|s) proportional to pi_theta(a|s) * exp(beta * A(s,a))`. In logit space, this reduces to a simple additive update: `z'(s,a) = z(s,a) + beta * A_hat(s,a)`, where `z` are the base LLM's output logits and `A_hat` is the estimated advantage. No gradients, no backpropagation -- just arithmetic on logits.

**Advantage Estimation.** Advantages come from a non-parametric experience memory storing `(state, action, discounted_return)` triplets. Given a current state, retrieve the top-k most similar past states using Jaccard similarity over tokenized representations. Estimate the state value `V_hat(s)` as the mean return of retrieved neighbors, and action value `Q_hat(s,a)` as the mean return of neighbors that took action `a`. The advantage is `A_hat(s,a) = Q_hat(s,a) - V_hat(s)`. For unseen actions, a small exploration bonus `alpha/|N(s)|` is added with probability `lambda` to encourage trying new strategies.

## Step-by-Step Workflow

1. **Define the state representation for your domain.** Encode environment observations into a canonical string form suitable for similarity matching. For web tasks, regularize URLs by replacing dynamic IDs with wildcards (e.g., `/user/edit/42` becomes `/user/edit/*`) and append recent action history. For CLI/tool-use agents, structure as `[State: key_nouns] [Action: recent_verbs]`. The representation must be stable enough that similar situations produce similar strings.

2. **Implement the experience memory store.** Create a persistent store (JSON on disk or a lightweight DB) holding `(state_repr, action_text, discounted_return)` triplets. Organize by episode so you can recompute returns when reward signals change. The store should support append, bulk retrieval, and optional size limits (LRU eviction or return-weighted pruning).

3. **Implement Jaccard similarity retrieval.** Tokenize state representations into n-gram sets (unigrams or bigrams). For a query state `s`, compute `J(s, s_i) = |T(s) intersect T(s_i)| / |T(s) union T(s_i)|` against all stored states. Return the top-k entries exceeding a similarity threshold (default k=10, threshold=0.8). For large memories, use MinHash or locality-sensitive hashing to avoid O(N) scans.

4. **Estimate action advantages from retrieved neighbors.** Compute `V_hat(s) = mean(G_i for i in N(s))` across all retrieved neighbors. For each candidate action `a`, compute `Q_hat(s,a) = mean(G_j for j in N(s) where action_j == a)` if the action has been seen before. For unseen actions, set `Q_hat(s,a) = V_hat(s) + alpha/|N(s)|` with probability `lambda` (exploration), otherwise `Q_hat(s,a) = V_hat(s)`. Then `A_hat(s,a) = Q_hat(s,a) - V_hat(s)`.

5. **Construct the augmented candidate action set.** Merge actions proposed by the base LLM with actions seen in retrieved trajectories. This ensures the agent considers both its default policy and historically successful actions. Deduplicate by normalizing action text.

6. **Apply the logit modulation.** For each candidate action, compute updated logits: `z'(a) = z(a) + beta * A_hat(s,a)`, where `z(a)` is the base LLM's log-probability for action `a` and `beta` controls exploitation strength (start with beta=1.0, tune in [0.5, 5.0]). Pass updated logits through softmax and sample the action. If you lack direct logit access (e.g., API-only models), approximate by constructing a weighted prompt that emphasizes high-advantage actions and de-emphasizes low-advantage ones.

7. **Execute the chosen action and collect rewards.** Run the selected action in the environment. Record the state-action pair. At episode boundaries (task success/failure or step limit), use an LLM evaluator to assign step-wise reward scores `r_t` for each action taken.

8. **Compute discounted returns and update memory.** After an episode ends, compute `G_t = sum(gamma^(u-t) * r_u for u in [t, T])` with gamma=0.95 for each timestep. Store all `(state_t, action_t, G_t)` triplets into the experience memory. Persist to disk.

9. **Iterate across episodes.** On each new episode, the retrieval pool grows, advantage estimates become more accurate, and the agent's policy improves. Monitor cumulative reward per episode to verify learning. Expect meaningful improvement within 5-20 episodes for moderately complex tasks.

10. **Tune hyperparameters based on observed behavior.** If the agent is too conservative (always repeating known-good actions), lower `beta` or raise `lambda`. If it wastes time on bad exploration, raise `beta` or the similarity threshold. If retrieval is too sparse, lower the threshold or switch to bigram tokenization.

## Concrete Examples

**Example 1: Web Navigation Agent with Experience Memory**

User: "Build a web agent that learns from past browsing sessions to get better at filling out forms on our internal tool."

Approach:
1. Define state as the regularized current URL + visible form field names + last 3 actions taken.
2. Store experience as JSON files under `memory/{task_type}/{timestamp}.json`, each entry containing `{state, action, return}`.
3. On each new task, tokenize the current state, retrieve top-10 similar past states via Jaccard similarity.
4. Compute advantage for each candidate action (click, type, select) based on returns of retrieved neighbors.
5. Modulate the LLM's action probabilities: `z'(a) = z(a) + 1.5 * A_hat(s,a)`.
6. After task completion, score each step (1.0 for correct fills, -0.5 for errors, 0.0 for neutral navigation), compute discounted returns with gamma=0.95, and store to memory.

Output structure:
```python
# experience_memory.py
class ExperienceMemory:
    def __init__(self, memory_dir="memory/", gamma=0.95):
        self.entries = []  # List of (state_repr, action, G)
        self.memory_dir = memory_dir

    def retrieve(self, query_state, top_k=10, threshold=0.8):
        query_tokens = set(tokenize(query_state))
        scored = []
        for state_repr, action, G in self.entries:
            entry_tokens = set(tokenize(state_repr))
            jaccard = len(query_tokens & entry_tokens) / len(query_tokens | entry_tokens)
            if jaccard >= threshold:
                scored.append((jaccard, state_repr, action, G))
        scored.sort(reverse=True)
        return scored[:top_k]

    def estimate_advantage(self, query_state, candidate_actions, alpha=0.1, lam=0.3):
        neighbors = self.retrieve(query_state)
        if not neighbors:
            return {a: 0.0 for a in candidate_actions}
        V_hat = sum(G for _, _, _, G in neighbors) / len(neighbors)
        advantages = {}
        for a in candidate_actions:
            matching = [G for _, _, act, G in neighbors if act == a]
            if matching:
                Q_hat = sum(matching) / len(matching)
            elif random.random() < lam:
                Q_hat = V_hat + alpha / len(neighbors)
            else:
                Q_hat = V_hat
            advantages[a] = Q_hat - V_hat
        return advantages

    def update(self, trajectory, step_rewards):
        """trajectory: list of (state, action), step_rewards: list of float"""
        T = len(trajectory)
        for t in range(T):
            G_t = sum(self.gamma**(u - t) * step_rewards[u] for u in range(t, T))
            state, action = trajectory[t]
            self.entries.append((state, action, G_t))
        self.save()
```

**Example 2: CLI Automation Agent That Improves Over Deployments**

User: "I have a DevOps agent that runs deployment scripts. It sometimes picks suboptimal rollback strategies. Make it learn from past incidents."

Approach:
1. State representation: `[Service: {name}] [Error: {error_type}] [Metrics: {cpu/mem/latency_bucket}] [Recent: {last_2_actions}]`.
2. Actions: "rollback-canary", "restart-pods", "scale-up", "revert-config", "escalate-to-human".
3. After each incident resolution, score the outcome: 1.0 for fast resolution, 0.5 for slow resolution, -1.0 for escalation or extended downtime.
4. On new incidents, retrieve similar past incidents, compute advantage per action, and present the agent with a reranked action list.

Output:
```python
def select_action(llm, state, memory, beta=2.0):
    # Get base LLM action proposals with log-probs
    candidates = llm.propose_actions(state)  # {action: log_prob}

    # Get historically successful actions from memory
    historical_actions = memory.get_historical_actions(state)
    for a in historical_actions:
        if a not in candidates:
            candidates[a] = llm.score_action(state, a)  # base log-prob

    # Compute advantages
    advantages = memory.estimate_advantage(state, list(candidates.keys()))

    # Apply JitRL logit modulation
    modulated = {a: candidates[a] + beta * advantages[a] for a in candidates}

    # Softmax and sample
    probs = softmax(modulated)
    return sample(probs)
```

**Example 3: API-Only Approximation (No Logit Access)**

User: "I'm using GPT-4o through the API and can't access logits. Can I still use JitRL?"

Approach:
1. Retrieve similar past experiences and compute advantages as usual.
2. Instead of modulating logits directly, construct a prompt injection that encodes advantage information:
   - Prepend a "learned strategy" section listing high-advantage actions with their estimated returns.
   - Frame low-advantage actions as "previously unsuccessful approaches to avoid."
3. Use the LLM's in-context learning to approximate the logit shift.

Output:
```python
def build_jitrl_prompt(base_prompt, state, memory):
    neighbors = memory.retrieve(state)
    if not neighbors:
        return base_prompt

    advantages = memory.estimate_advantage(state, memory.get_all_actions(state))
    sorted_actions = sorted(advantages.items(), key=lambda x: -x[1])

    strategy_section = "Based on past experience in similar situations:\n"
    for action, adv in sorted_actions[:3]:
        if adv > 0:
            strategy_section += f"- PREFER: '{action}' (historically effective, advantage={adv:.2f})\n"
    for action, adv in sorted_actions[-2:]:
        if adv < 0:
            strategy_section += f"- AVOID: '{action}' (historically poor, advantage={adv:.2f})\n"

    return f"{strategy_section}\n{base_prompt}"
```

## Best Practices

- **Do:** Normalize state representations aggressively. Replace dynamic IDs, timestamps, and session tokens with wildcards. Similarity matching degrades fast when states contain ephemeral noise.
- **Do:** Start with a small beta (1.0) and increase gradually. Over-exploitation early (high beta) locks the agent into suboptimal strategies before the memory has enough coverage.
- **Do:** Use an LLM evaluator for step-wise rewards rather than only sparse episode-level signals. Intermediate rewards produce much more informative advantage estimates, especially in long-horizon tasks.
- **Do:** Persist memory across sessions. The entire value of JitRL comes from accumulating experience. Losing memory resets the agent to base LLM performance.
- **Avoid:** Applying JitRL when the environment is highly non-stationary (e.g., the task distribution changes completely between episodes). The advantage estimates from old trajectories will be misleading.
- **Avoid:** Setting the similarity threshold too low. Retrieving dissimilar states injects noise into advantage estimates. If retrieval returns fewer than 3 neighbors, fall back to the base LLM policy rather than using noisy estimates.

## Error Handling

| Issue | Cause | Fix |
|-------|-------|-----|
| Advantages are all zero | Memory is empty or threshold too high | Lower `retrieval_threshold` to 0.6, or fall back to base LLM until memory has 20+ entries |
| Agent loops on the same action | High beta with limited action diversity in memory | Increase `lambda` (exploration probability) to 0.5, or decrease `beta` |
| Retrieval is too slow | Memory grown beyond 10K entries with brute-force Jaccard | Switch to MinHash LSH indexing or prune low-return entries periodically |
| Step-wise rewards are unreliable | LLM evaluator hallucinating reward scores | Add calibration: normalize rewards to [-1, 1], use majority vote across 3 evaluator calls |
| Memory diverges from current environment | Environment changed (new UI, new API version) | Add a recency weight `w = decay^(episode_age)` to retrieved returns, or flush old entries |

## Limitations

- **Requires repeated episodes in similar environments.** JitRL is not useful for one-shot tasks or completely novel environments where no prior trajectory data exists.
- **Logit access needed for exact method.** The provably optimal logit modulation requires access to output log-probabilities. API-only models (without `logprobs` parameter) must use the prompt-based approximation, which is weaker.
- **Memory scales linearly.** Without indexing, retrieval cost grows with memory size. For production systems with 100K+ entries, approximate nearest-neighbor methods are necessary.
- **Sensitive to state representation quality.** Poor state encoding (too verbose, too sparse, or including irrelevant information) degrades Jaccard similarity and makes retrieval unreliable.
- **Does not handle partial observability well.** If the true environment state is hidden, the observed state may map to very different optimal actions, causing high-variance advantage estimates.

## Reference

**Paper:** [Just-In-Time Reinforcement Learning: Continual Learning in LLM Agents Without Gradient Updates](https://arxiv.org/abs/2601.18510v1) (Li et al., 2026). Key sections: Section 4 for the closed-form derivation of `z' = z + beta * A_hat`, Section 4.2 for the k-NN advantage estimation procedure, and the appendix theorems for convergence guarantees. **Code:** [github.com/liushiliushi/JitRL](https://github.com/liushiliushi/JitRL).