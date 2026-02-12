---
name: "training-multi-turn-search-agent"
description: "Train multi-turn search and retrieval agents using Contrastive Dynamic Branch Sampling (BranPO). Applies tail-divergence analysis and contrastive suffix learning to improve RL training of agents that iteratively search, retrieve, and reason. Use when: 'train a search agent with RL', 'improve multi-turn retrieval agent', 'implement BranPO training', 'build a ReAct search agent trainer', 'contrastive RL for tool-using agents', 'optimize multi-hop QA agent training'."
---

# Training Multi-Turn Search Agents via Contrastive Dynamic Branch Sampling (BranPO)

This skill enables Claude to implement and apply **Branching Relative Policy Optimization (BranPO)** — a reinforcement learning method for training multi-turn search agents that iteratively query, retrieve, and synthesize information. BranPO exploits a key empirical finding: in multi-turn search trajectories, performance diverges mainly at **tail decisions** (the final 1-2 search steps), not early ones. Instead of training on entire trajectories with sparse outcome rewards, BranPO truncates trajectories near the tail, resamples alternative continuations, and applies contrastive supervision between successful and failed suffixes sharing the same prefix. This yields sharper credit assignment, better sample efficiency, and 2-5 F1 point improvements over standard GRPO on multi-hop QA benchmarks.

## When to Use

- When the user wants to train or fine-tune an LLM-based search agent that performs multi-turn retrieval (e.g., a ReAct agent calling a search API)
- When implementing RL training for any agent with long-horizon, sparse-reward trajectories and tool use
- When the user asks to improve a multi-hop question answering system that iteratively searches for evidence
- When building a training pipeline for agents that alternate between thinking and acting (ReAct, tool-calling agents)
- When the user wants to reduce training variance in trajectory-level RL by focusing on critical decision points
- When debugging why an RL-trained search agent makes poor decisions at the final retrieval step despite good early behavior

## Key Technique

**The Tail-Divergence Insight.** Empirical analysis of search agents on HotpotQA and 2WikiMultihopQA reveals that early search steps are largely similar across successful and failed trajectories. Performance diverges at the tail — the last 1-2 actions determine success or failure. This means standard trajectory-level rewards (GRPO, PPO) waste gradient signal on early steps that are already near-optimal, while under-weighting the critical late decisions.

**BranPO's Three Mechanisms.** (1) **Dynamic Branch Sampling (DBS):** Rather than rolling out complete independent trajectories, BranPO identifies a shared prefix up to a branching point near the tail, then resamples multiple alternative continuations (suffixes). This constructs contrastive pairs — a successful suffix C+ and a failed suffix C- sharing the same prefix B. The gradient pushes the model toward C+ and away from C-: `nabla L_suffix ~ nabla log pi(C+|B) - nabla log pi(C-|B)`. (2) **Difficulty-Aware Budgeting:** Easy questions (all rollouts correct) get minimal branching (1 sample at the final step). Hard questions (accuracy < 0.5) get recursive branching with 3 depths and 2 samples per level. Medium questions get 2 depths. This allocates compute where learning signal exists. (3) **Redundant Step Masking (RSM):** If branching from a penultimate step produces a correct shorter trajectory, the extra steps in the original are masked (zero gradient), suppressing the agent's tendency toward unnecessary verification searches.

**Training Pipeline.** The full pipeline uses a two-stage curriculum: Stage 1 trains with a 4-turn interaction limit for 40 optimization steps; Stage 2 extends to 8 turns for another 40 steps. An SFT cold-start on 10K correct trajectories precedes RL. The method works with Qwen2.5-7B and Qwen3-4B, achieving 55.7 average F1 vs 54.0 for GRPO and 53.9 for Tree-GRPO on multi-hop benchmarks.

## Step-by-Step Workflow

1. **Define the search agent's action space.** Implement a ReAct-style agent with three action types: `Search[query]` (issues a retrieval call), `Think[reasoning]` (internal chain-of-thought), and `Finish[answer]` (terminal action). Each action is a variable-length token sequence. The environment returns search results as observations after each `Search` action.

2. **Collect initial trajectories via SFT cold-start.** Roll out the base model on your training questions. Filter for correct trajectories (matching ground-truth answers). Fine-tune the base model on ~10K correct trajectories for 1 epoch. This gives the policy enough competence to produce meaningful rollouts in RL.

3. **Roll out N trajectories per question from the current policy.** For each training question, sample N (typically 8-16) complete trajectories with the current policy. Record the full action sequence `(a1, a2, ..., aT)` and the final scalar reward `R(tau)` (F1 score against ground truth).

4. **Classify question difficulty from rollout outcomes.** Compute group accuracy across the N rollouts: if all correct, label "easy"; if accuracy >= 0.5 and F1 >= 0.8, label "medium"; otherwise label "hard". This determines branching budget in the next step.

5. **Identify branching points via tail-divergence analysis.** For each question, find the step index where successful and failed trajectories diverge. In practice, this is near the final 1-2 actions. The branching point `b` is the last step index where the majority of rollouts still share the same prefix (same search queries issued, same context accumulated).

6. **Resample contrastive suffixes from branching points.** Truncate trajectories at the branching point to extract the shared prefix B. From that state, resample alternative continuations using the current policy. For hard questions, apply recursive branching: branch at depth 1, then branch again from the new suffixes at depth 2 and 3 (2 samples per level). Retain only suffix pairs where outcomes differ (one correct, one incorrect).

7. **Compute two-level advantages.** For each trajectory, compute: (a) **Base advantage** over the full trajectory: `A_base = (r - mean(r)) / std(r)` across all rollouts of that question. (b) **Branch advantage** over the suffix only: `A_branch = (r_branch - mean(r_branch)) / std(r_branch)` across all resampled continuations from the same prefix. Apply base advantage to prefix tokens and branch advantage to suffix tokens.

8. **Apply redundant step masking.** For each trajectory with extra verification steps after a correct intermediate answer, branch from the penultimate step. If a shorter continuation also reaches the correct answer, zero out the advantage for the redundant trailing steps in the original trajectory.

9. **Update the policy with the contrastive objective.** Use the combined prefix + suffix advantages in a PPO-style update with clipped surrogate objective. Key hyperparameters: learning rate 5e-6, KL coefficient 0.001, batch size 256, clip range 0.2. Run for 40 optimization steps per stage.

10. **Extend the horizon in Stage 2.** After Stage 1 converges (4-turn limit), increase the interaction limit to 8 turns and repeat steps 3-9 for another 40 steps. Evaluate on held-out sets with up to 16 turns to test generalization to longer horizons.

## Concrete Examples

**Example 1: Training a Multi-Hop QA Search Agent**

```
User: I want to train a Qwen-based agent to answer multi-hop questions
by searching Wikipedia. The agent should learn when to search again vs.
when to give a final answer.

Approach:
1. Set up a ReAct environment with a Wikipedia search API. The agent
   alternates Think/Search/Finish actions. Reward = F1 of final answer
   vs. ground truth.

2. SFT cold-start: Roll out Qwen2.5-7B on 5K HotpotQA training
   questions. Filter ~10K correct trajectories. Fine-tune 1 epoch
   (lr=2e-5, batch=32).

3. Stage 1 RL (4-turn limit):
   - For each of 2K training questions, roll out 8 trajectories
   - Classify: Q1 has 6/8 correct -> medium, Q2 has 2/8 -> hard
   - Q1 (medium): Branch at step 3 (of 4), resample 2 depths x 2
     samples = 4 alternative suffixes. Keep pairs where one succeeds
     and one fails.
   - Q2 (hard): Branch at step 2, recursive 3 depths x 2 samples =
     8 alternative suffixes.
   - Compute A_base for prefix tokens, A_branch for suffix tokens
   - PPO update with lr=5e-6, 40 steps

4. Stage 2 RL (8-turn limit): Same process, extended horizon.

Output: Agent achieves ~69 F1 on 2WikiMQA (vs ~64 for vanilla GRPO),
with particularly strong gains on 3+ hop questions where tail decisions
matter most.
```

**Example 2: Implementing the Branching Logic in Code**

```
User: Show me how to implement the dynamic branch sampling step.

Approach:
1. After collecting rollouts, group by question ID
2. Find the divergence point by comparing action sequences

Python pseudocode:

def find_branch_point(trajectories):
    """Find the last step where majority of trajectories agree."""
    max_len = max(len(t.actions) for t in trajectories)
    for step in range(max_len - 1, -1, -1):
        actions_at_step = [t.actions[step] for t in trajectories
                          if len(t.actions) > step]
        most_common = Counter(actions_at_step).most_common(1)[0]
        if most_common[1] / len(actions_at_step) > 0.6:
            return step + 1  # Branch AFTER this shared step
    return 0

def sample_contrastive_suffixes(policy, prefix_state, n_samples,
                                 depth, max_depth, difficulty):
    """Recursively sample alternative continuations."""
    suffixes = []
    for _ in range(n_samples):
        suffix = policy.rollout_from(prefix_state)
        suffixes.append(suffix)

    if depth < max_depth and difficulty != "easy":
        for s in suffixes:
            sub_branch = len(s.actions) - 1
            sub_state = s.state_at(sub_branch)
            suffixes.extend(
                sample_contrastive_suffixes(
                    policy, sub_state, n_samples,
                    depth + 1, max_depth, difficulty
                )
            )
    return suffixes

def build_contrastive_pairs(suffixes, prefix):
    """Pair successful and failed continuations."""
    pos = [s for s in suffixes if s.reward > 0.5]
    neg = [s for s in suffixes if s.reward <= 0.5]
    pairs = []
    for p in pos:
        for n in neg:
            pairs.append((prefix, p, n))
    return pairs

Output: These functions slot into the RL training loop between rollout
collection and advantage computation.
```

**Example 3: Diagnosing a Poorly-Performing Search Agent**

```
User: My search agent keeps doing unnecessary extra searches after it
already has the answer. How do I fix this with BranPO?

Approach:
1. This is the exact problem Redundant Step Masking (RSM) addresses.
2. During training, for each trajectory with T steps:
   - Branch from step T-1 (penultimate)
   - If a 1-step continuation from T-1 produces the correct answer,
     the original step T is redundant
   - Zero out the advantage for step T's tokens
3. Implementation:

def apply_redundant_step_masking(trajectory, policy):
    """Mask advantages for unnecessary trailing actions."""
    masks = [1.0] * len(trajectory.actions)
    for t in range(len(trajectory.actions) - 1, 0, -1):
        state = trajectory.state_at(t - 1)
        short_suffix = policy.rollout_from(state, max_steps=1)
        if short_suffix.is_correct and trajectory.is_correct:
            masks[t] = 0.0  # This step was unnecessary
        else:
            break  # Stop — earlier steps were needed
    return masks

Output: After applying RSM during training, the agent learns to issue
Finish[answer] as soon as it has sufficient evidence, reducing average
trajectory length by 1-2 steps while maintaining accuracy.
```

## Best Practices

- **Do:** Start with an SFT cold-start before RL. Without it, rollouts are too noisy for meaningful branching — the policy needs baseline competence to produce trajectories worth contrasting.
- **Do:** Scale branching budget by difficulty. Spending recursive branching compute on easy questions (already all-correct) wastes resources. Reserve deep branching for hard questions where learning signal exists.
- **Do:** Use the two-stage curriculum (short horizon first, then extend). Training directly at 8 turns converges slower than building up from 4 turns.
- **Do:** Keep only contrastive pairs where outcomes genuinely differ. Pairs of two successes or two failures provide no directional gradient signal.
- **Avoid:** Applying dense step-level rewards as a substitute. BranPO's strength is that it requires only sparse outcome rewards — adding heuristic intermediate rewards introduces bias and often hurts.
- **Avoid:** Branching too early in the trajectory. The tail-divergence finding shows early steps are largely shared; branching at step 1 or 2 wastes compute regenerating near-identical prefixes.

## Error Handling

- **All rollouts succeed (no contrastive signal):** Skip branching for these questions. Mark as "easy" and apply only base-level advantage with minimal branching (1 sample at last step). If too many questions are easy, increase task difficulty or reduce the turn limit.
- **All rollouts fail (no positive examples):** Increase the sampling budget (more rollouts per question). If still failing, the SFT cold-start was insufficient — go back and fine-tune on more correct trajectories before RL.
- **Branching point detection fails (no shared prefix):** This happens when trajectories diverge from step 1. Fall back to standard GRPO for these questions. This indicates the question may require fundamentally different search strategies, not tail-level refinement.
- **Redundant step masking is too aggressive:** If the agent becomes overly terse and misses answers, reduce RSM scope — only mask the very last step rather than recursively checking multiple trailing steps.
- **KL divergence spikes:** The contrastive suffix loss can push the policy far from the reference. Monitor KL and reduce the learning rate or increase the KL coefficient (default 0.001) if divergence exceeds 0.1.

## Limitations

- **Requires multiple rollouts per question.** BranPO needs 8-16 trajectories per training example plus resampled branches. This is 3-5x the compute of standard GRPO. Not practical for extremely large training sets without significant GPU resources.
- **Assumes tail-divergence holds.** The method works best when early steps are largely correct and errors concentrate at the end. For tasks where errors occur uniformly throughout the trajectory (e.g., complex code generation), the branching strategy may not help.
- **Single-turn tasks see no benefit.** BranPO is designed for multi-turn agents with 3+ interaction steps. For single-turn generation or simple retrieval, standard RLHF/DPO is more appropriate.
- **Reward must be binary or near-binary.** The contrastive pairing requires clear positive/negative labels. Tasks with continuous, ambiguous rewards (e.g., open-ended generation quality) make it hard to form meaningful contrastive pairs.
- **Tested only on Qwen models at 4B-7B scale.** Generalization to other architectures and larger scales is plausible but not empirically validated in the paper.

## Reference

[Training Multi-Turn Search Agent via Contrastive Dynamic Branch Sampling](https://arxiv.org/abs/2602.03719v1) — Zhao et al., 2026. Focus on Section 3 (BranPO algorithm), Section 3.3 (difficulty-aware sampling), and Table 1 (main results showing 2-5 F1 improvement over GRPO baselines on multi-hop QA).