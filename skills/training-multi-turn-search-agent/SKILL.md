---
name: "training-multi-turn-search-agent"
description: "Build and train multi-turn search agents using BranPO (Branching Relative Policy Optimization) with contrastive dynamic branch sampling. Focuses on tail-divergence analysis and step-level contrastive supervision for long-horizon retrieval tasks. Use when: 'train a search agent', 'build a multi-turn retrieval pipeline', 'improve search agent with RL', 'implement BranPO training', 'contrastive trajectory sampling', 'optimize multi-hop QA agent'."
---

# Training Multi-Turn Search Agents via Contrastive Dynamic Branch Sampling (BranPO)

This skill enables Claude to design and implement training pipelines for multi-turn search agents using BranPO (Branching Relative Policy Optimization). BranPO is a value-free reinforcement learning method that provides step-level contrastive supervision without requiring dense reward signals. It exploits a key empirical insight -- **tail-divergence** -- where agent trajectories that succeed versus fail diverge primarily in their final search steps, not their early ones. By truncating trajectories near the tail and resampling alternative continuations, BranPO constructs contrastive prefix-suffix pairs that directly teach the model which late-stage decisions lead to correct answers.

## When to Use

- When the user asks to train or fine-tune an LLM-based search agent that performs multi-hop question answering (e.g., HotpotQA, MuSiQue, 2WikiMQA)
- When building a ReAct-style agent loop (reason-retrieve-synthesize) and the user wants to improve its retrieval strategy via RL
- When the user has trajectory rollout data from a search agent and wants to construct contrastive training pairs from it
- When implementing GRPO-based RL training and the user needs better credit assignment in long-horizon (4-16 turn) settings
- When the user wants to reduce training compute by focusing learning signal on critical decision points rather than full trajectories
- When debugging a search agent whose early retrieval steps are fine but whose late-stage synthesis or follow-up queries degrade

## Key Technique: BranPO with Tail-Divergence Exploitation

**The core insight**: In multi-turn search, most trajectories share nearly identical early steps. Whether the agent ultimately succeeds or fails depends on decisions in the final 1-3 turns -- the "tail." Standard trajectory-level RL (like GRPO) wastes gradient signal on shared prefixes and suffers from high variance when assigning credit across long sequences.

**BranPO's solution**: Instead of training on full trajectories, BranPO truncates each trajectory near its tail to create a shared prefix, then resamples multiple alternative suffixes (continuations) from that prefix. Only suffixes with *different outcomes* from the original are retained -- creating contrastive pairs. The gradient update then decomposes into: (1) a low-variance prefix update using averaged returns across branches, and (2) a suffix update that recovers a DPO-like (Direct Preference Optimization) objective, maximizing the likelihood gap between successful and failed continuations given the same prefix.

**Difficulty-aware branching** further optimizes compute: easy tasks (all rollouts correct) get minimal branching; hard tasks (low accuracy) trigger recursive multi-depth branching up to 3 levels deep. A **redundant step masking** mechanism identifies unnecessarily long correct trajectories by checking if shorter alternative paths reach the same answer, zeroing out advantages for wasteful actions.

## Step-by-Step Workflow

1. **Define the search environment**: Set up a retrieval backend (e.g., dense retriever like E5-base-v2 over a document corpus) with a fixed `top_k` (typically 3). Define the agent's action space as: emit a search query OR produce a final answer. Cap maximum turns (start with 4, extend to 8-16).

2. **Implement the ReAct agent loop**: Structure each turn as reason-then-act. The agent reads the question and retrieved context, decides whether to issue another search query or synthesize a final answer. Each trajectory is a sequence of (thought, action, observation) tuples.

3. **Collect initial rollouts**: For each training question, sample N trajectories (N=4 is effective) from the current policy. Record each trajectory's full token sequence and compute an outcome reward (e.g., F1 score against ground truth).

4. **Compute group accuracy**: For each question group, calculate the fraction of correct trajectories. This drives the difficulty-aware branching schedule:
   - `acc = 1.0` (all correct): single branch, single recursion depth
   - `acc < 1.0`: 2 branches per depth
   - `acc < 0.5` AND individual F1 < 0.8: recurse up to 3 depths backward from tail

5. **Truncate and resample (branch sampling)**: For each trajectory, truncate at the tail (last turn). Fix the prefix. Sample alternative continuations from the current policy conditioned on that prefix. Retain only branches where the outcome *differs* from the original trajectory -- these are the contrastive pairs.

6. **Apply redundant step masking**: For correct trajectories that exceed the group's average length, branch from the penultimate step. If the alternative reaches the correct answer in fewer steps, mask the original's extra steps by zeroing their advantage weights.

7. **Compute advantages**: Calculate two sets of normalized advantages:
   - **Base advantage** (prefix): `A_base = (r_base - mean(r_base)) / std(r_base)` where `r_base` averages returns across all branches for that trajectory
   - **Branch advantage** (suffix): `A_branch = (r_branch - mean(r_branch)) / std(r_branch)` computed within each branching group

8. **Apply policy gradient update**: Use GRPO-style token-level probability ratio clipping for the prefix portion. For the suffix portion, the gradient naturally becomes `nabla log pi(C+|B) - nabla log pi(C-|B)`, maximizing the gap between good and bad continuations sharing the same prefix.

9. **Iterate with curriculum**: Train for 40 steps at 4-turn limit, then extend to 8-turn limit for another 40 steps. Evaluate at 16-turn limit to test generalization to longer horizons.

10. **Evaluate on held-out benchmarks**: Measure both F1 (token-level) and LasJ/exact match on multi-hop QA datasets. Compare against GRPO, Tree-GRPO, and other baselines to verify that tail-focused training improves late-stage decision quality.

## Concrete Examples

**Example 1: Training a HotpotQA search agent**

User: "I have a Qwen3-4B model and want to train it as a multi-turn search agent on HotpotQA using RL. How should I structure the training?"

Approach:
1. Set up a Wikipedia retrieval index using E5-base-v2 embeddings over the Dec 2018 dump, returning top-3 passages per query
2. Implement a ReAct loop: the model alternates between generating a search query and reading retrieved passages, up to 4 turns
3. Cold-start with SFT: run basic GRPO for 80 iterations, filter the ~10K correct trajectories, and fine-tune for 1 epoch
4. Switch to BranPO: sample 4 trajectories per question, compute group accuracy, apply tail truncation and contrastive resampling
5. Train for 40 steps, then extend the turn limit to 8 and train for 40 more steps

Output:
```python
# Pseudocode for BranPO training loop
for step in range(num_train_steps):
    for question_group in batch:
        # Initial rollout
        trajectories = [sample_trajectory(policy, question, max_turns=4) for _ in range(N)]
        rewards = [compute_f1(traj.answer, ground_truth) for traj in trajectories]
        acc = mean([r > 0.5 for r in rewards])

        # Difficulty-aware branching config
        num_branches = 1 if acc == 1.0 else 2
        max_depth = 3 if (acc < 0.5) else (2 if acc < 1.0 else 1)

        contrastive_pairs = []
        for traj, r in zip(trajectories, rewards):
            for depth in range(1, max_depth + 1):
                prefix = traj.truncate_at(len(traj) - depth)
                for _ in range(num_branches):
                    suffix = policy.sample_continuation(prefix)
                    r_branch = compute_f1(suffix.answer, ground_truth)
                    if (r_branch > 0.5) != (r > 0.5):  # contrastive
                        contrastive_pairs.append((prefix, suffix, r_branch, r))

        # Compute advantages and update
        base_advantages = group_normalize([avg_branch_return(t) for t in trajectories])
        branch_advantages = group_normalize([r for _, _, r, _ in contrastive_pairs])
        policy.update(trajectories, contrastive_pairs, base_advantages, branch_advantages)
```

**Example 2: Diagnosing a search agent that fails on multi-hop questions**

User: "My search agent gets the first retrieval right but then asks bad follow-up queries and hallucinates in its final answer. How do I fix this?"

Approach:
1. Collect 100+ trajectory rollouts and annotate them with per-turn correctness
2. Verify the tail-divergence pattern: compare successful vs failed trajectories and confirm they share early prefixes but diverge in the last 1-2 turns
3. Implement contrastive branch sampling at the divergence point -- truncate right before the bad follow-up query and resample alternatives
4. Retain only pairs where one continuation succeeds and the other fails
5. Train with the BranPO objective, which directly penalizes the bad suffix and rewards the good one given the same prefix context

Output:
```
Diagnostic findings:
- 87% of trajectories share identical first 2 search queries
- Divergence occurs at turn 3-4 in 91% of failure cases
- Common failure: agent issues a vague reformulation instead of a targeted follow-up
- BranPO training target: truncate at turn 2, resample turn 3-4 continuations
- Expected improvement: +4-6 F1 points based on paper benchmarks
```

**Example 3: Implementing redundant step masking**

User: "Some of my agent's correct trajectories take 6 turns when the answer could be found in 3. How do I penalize unnecessary steps?"

Approach:
1. For each correct trajectory longer than the group average, identify the penultimate step
2. Branch from that step: resample an alternative continuation
3. If the alternative reaches the correct answer with fewer remaining steps, mark the original's extra steps as redundant
4. Set `mask[redundant_steps] = 0` so their advantage contributions are zeroed out during training
5. This teaches the agent to be concise without penalizing correctness

Output:
```python
def apply_redundant_step_masking(trajectory, group_avg_length, policy, ground_truth):
    if not trajectory.is_correct or len(trajectory) <= group_avg_length:
        return np.ones(len(trajectory))  # no masking

    mask = np.ones(len(trajectory))
    prefix = trajectory.truncate_at(len(trajectory) - 2)  # penultimate step
    alt_suffix = policy.sample_continuation(prefix)

    if (compute_f1(alt_suffix.answer, ground_truth) > 0.5
            and len(alt_suffix) < len(trajectory) - len(prefix)):
        # Original took more steps than necessary
        mask[len(prefix):] = 0  # zero out redundant tail steps

    return mask
```

## Best Practices

- **Do**: Start with a short turn limit (4 turns) and progressively extend (8, then 16). The paper shows this curriculum approach outperforms training at full length from the start.
- **Do**: Use group normalization for advantages -- normalize within each question group rather than globally. This prevents easy questions from dominating the gradient.
- **Do**: Retain only *contrastive* branches (where outcome differs from original). Non-contrastive branches add compute cost without learning signal.
- **Do**: Run a cold-start SFT phase before RL. Filtering correct trajectories from initial GRPO rollouts and fine-tuning creates a stronger starting point.
- **Avoid**: Branching from early trajectory positions. The tail-divergence finding shows early steps carry little discriminative signal -- branch near the end.
- **Avoid**: Uniform branching across all tasks. Difficulty-aware allocation (more branches for harder tasks, fewer for easy ones) is critical for compute efficiency.
- **Avoid**: Training with dense per-step rewards. BranPO's strength is providing step-level supervision from outcome-only rewards via the contrastive structure -- adding noisy intermediate rewards can hurt.

## Error Handling

- **All rollouts succeed (acc=1.0)**: Minimal branching is still applied (depth=1, 1 branch) to check for redundant steps. If no contrastive pairs are found, skip branch training for that group and use only base advantages.
- **All rollouts fail (acc=0.0)**: Maximum branching is triggered (depth=3, 2 branches). If resampled branches also all fail, the question may be too hard for the current policy -- log it and consider filtering it from this training iteration.
- **Degenerate contrastive pairs**: If branch sampling consistently produces only same-outcome suffixes, the truncation point may be too late. Move the truncation 1-2 steps earlier to find the actual divergence point.
- **Advantage normalization instability**: When a group has very few trajectories (< 3), standard deviation can be near-zero. Add a small epsilon (1e-8) to the denominator or skip normalization for that group.
- **Retrieval backend failures**: If the retrieval index returns empty results, the agent's observation is empty. Ensure the training loop handles empty observations gracefully rather than crashing -- the agent should learn to reformulate its query.

## Limitations

- BranPO assumes the tail-divergence pattern holds -- that early steps are largely shared across success/failure trajectories. In domains where early decisions are critical (e.g., irreversible actions), the method's focus on tail branching may miss important learning signal.
- The method requires multiple rollouts per question (N=4 baseline + branches), which demands significant inference compute even if total training steps are reduced.
- Contrastive pair quality depends on the current policy being "close" to producing both good and bad continuations. If the policy is too weak (everything fails) or too strong (everything succeeds), contrastive signal vanishes.
- The approach is validated primarily on knowledge-intensive QA with Wikipedia retrieval. Transfer to other search domains (code search, web APIs, database queries) is plausible but unverified.
- Redundant step masking uses a binary mask, which can be aggressive -- it zeros out all extra steps even if some were partially useful. A softer weighting scheme may be more appropriate in some settings.

## Reference

**Paper**: [Training Multi-Turn Search Agent via Contrastive Dynamic Branch Sampling](https://arxiv.org/abs/2602.03719v1) (Zhao et al., 2026)
**Code**: [github.com/YubaoZhao/BranPO](https://github.com/YubaoZhao/BranPO)
**Key takeaway**: Look for the tail-divergence analysis (Section 3) showing that success/failure trajectories share early prefixes, and Algorithm 1 detailing the full BranPO procedure with difficulty-aware branching schedules.