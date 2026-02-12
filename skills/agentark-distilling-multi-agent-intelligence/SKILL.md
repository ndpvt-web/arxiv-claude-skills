---
name: "agentark-distilling-multi-agent-intelligence"
description: "Distill multi-agent debate reasoning into a single LLM's behavior. Apply AgentArk's three-tier distillation strategy to design training pipelines, self-correcting prompts, and process-aware reward systems. Use when: 'distill multi-agent into single model', 'reduce multi-agent inference cost', 'train self-correcting reasoning', 'build process reward model', 'agent distillation pipeline', 'single-agent with multi-agent quality'."
---

# AgentArk: Distilling Multi-Agent Intelligence into a Single LLM Agent

This skill enables Claude to apply the AgentArk framework for collapsing multi-agent debate systems into a single model that retains multi-agent-level reasoning, self-correction, and robustness. Rather than running N agents at inference time (with N-fold cost and error propagation risk), AgentArk shifts that computation to training time through three hierarchical distillation strategies: reasoning-enhanced fine-tuning, trajectory-based data augmentation, and process-aware distillation with step-level reward models. Claude uses this skill to architect distillation pipelines, design debate-to-training data converters, build process reward models, and structure GRPO-based RL fine-tuning for any reasoning task.

## When to Use

- When the user wants to replace a multi-agent debate system with a single model that preserves reasoning quality
- When the user asks how to reduce inference cost of a multi-agent pipeline without sacrificing accuracy
- When building a training pipeline that teaches a model to self-correct by learning from agent debate trajectories
- When designing a Process Reward Model (PRM) that scores intermediate reasoning steps, not just final answers
- When the user needs to generate diverse, correctness-filtered training data from multi-agent interactions
- When applying GRPO (Group Relative Policy Optimization) for reasoning-focused RL fine-tuning
- When the user wants a single agent to exhibit step decomposition, intermediate verification, and error localization behaviors typically only seen in multi-agent systems

## Key Technique

**The core insight**: Multi-agent debate works because agents critique each other's reasoning, catch errors, and converge on better answers through iterative refinement. AgentArk captures these dynamics in training data and reward signals so a single model internalizes the critique-and-correct loop. The distilled model generates, evaluates, and refines answers within a single forward pass -- mimicking how a human internalizes group reasoning after enough collaborative problem-solving.

**Three hierarchical strategies** address increasing levels of sophistication. **Reasoning-Enhanced SFT (R-SFT)** trains on final consensus answers paired with full reasoning traces, using a combined loss over both rationale quality and answer correctness. **Data Augmentation (DA)** extracts 1-3 diverse correct trajectories per problem from debate logs using a teacher LLM -- selecting paths that use distinct mathematical identities, logical heuristics, or assumptions -- forcing the student to learn multiple valid solution strategies rather than memorizing one. **Process-Aware Distillation (PAD)** treats it as an RL problem: a Process Reward Model trained with contrastive loss scores each reasoning step (not just the final answer), then GRPO optimizes the policy against these step-level rewards without requiring a separate value function.

**What matters most**: PRM capacity matters more than student model size. Reasoning quality in training data outweighs quantity -- high-signal trajectories from corrective debate rounds (where agents pivoted from wrong to right) transfer better than clean error-free paths. Excessive supervision can overwhelm small models, so data filtering is critical.

## Step-by-Step Workflow

1. **Run multi-agent debate to generate raw data.** Deploy N homogeneous agents (same LLM backbone) debating for K rounds on your task dataset. Each round: agents generate reasoning traces conditioned on the problem and peer responses. Use 3-5 agents and 2-3 rounds as a baseline. Collect full debate logs including all intermediate traces.

   ```yaml
   # Example debate config
   num_agents: 4
   num_rounds: 3
   temperature: 0.7
   max_tokens: 2048
   ```

2. **Filter for correctness and extract corrective trajectories.** Verify final consensus answers against ground truth. Prioritize trajectories where agents initially proposed incorrect steps but self-corrected after critique -- these "corrective trajectories" capture the core reasoning dynamics better than clean paths.

   ```python
   # Pseudocode for trajectory filtering
   for debate in debate_logs:
       if debate.final_answer == ground_truth:
           for trace in debate.agent_traces:
               if trace.had_correction:  # pivoted from wrong to right
                   priority_trajectories.append(trace)
               elif trace.all_correct:
                   standard_trajectories.append(trace)
   ```

3. **Choose your distillation tier based on compute budget and target quality.**
   - **R-SFT** (simplest): Fine-tune on (reasoning_trace, answer) pairs with combined loss. Good baseline, lowest compute.
   - **DA** (moderate): Use a teacher LLM to extract 1-3 diverse correct reasoning paths per problem from debate logs. Train student on augmented dataset.
   - **PAD** (strongest): Train a PRM on step-level correctness labels, then run GRPO to optimize the student policy against PRM rewards.

4. **For DA: Run correctness-first diverse extraction.** Prompt a high-capacity teacher to parse each debate log and extract reasoning paths that are (a) correct (lead strictly to ground truth) and (b) diverse (use different strategies). Deduplicate by approach, not by surface text.

   ```
   Teacher prompt: "Given this multi-agent debate log, extract 1-3 reasoning
   trajectories that arrive at the correct answer '{gt}' using DISTINCT
   logical approaches. Each trajectory must be self-contained and verifiable.
   Label each with the core strategy it employs."
   ```

5. **For PAD: Train the Process Reward Model in two stages.** Stage I: Freeze the LLM backbone, train only a reward head on step-level correctness labels using contrastive loss. This aligns features without catastrophic forgetting. Stage II: Unfreeze the backbone for end-to-end fine-tuning specialized in detecting logical fallacies at each reasoning step.

   ```bash
   # PRM training
   python prm/finetune2.py \
     --model_name_or_path <base_model> \
     --train_data_path labeled_steps.jsonl \
     --output_dir prm_checkpoint \
     --learning_rate 1e-4 \
     --per_device_train_batch_size 64 \
     --bf16
   ```

6. **For PAD: Run GRPO optimization.** Sample N completions per prompt (N >= 8), score each with the PRM, compute group-relative advantages, and update the policy. Use RLOO (Reinforce Leave-One-Out) advantage estimation for variance reduction.

   ```bash
   python -m openrlhf.cli.train_grpo \
     --pretrain <sft_model> \
     --reward_pretrain <prm_checkpoint> \
     --n_samples_per_prompt 8 \
     --advantage_estimator rloo \
     --reward_mode PRMVR \
     --micro_rollout_batch_size 4
   ```

7. **Evaluate distilled models on held-out reasoning tasks.** Measure not just final-answer accuracy but reasoning quality: step decomposition clarity, self-correction frequency, and coherence across multi-step chains. Compare against the original multi-agent system and a vanilla SFT baseline.

8. **Iterate on data quality, not quantity.** If the distilled model underperforms, improve trajectory filtering before adding more data. Check that corrective trajectories outnumber clean ones. Verify the PRM correctly assigns low scores to plausible-but-wrong intermediate steps.

## Concrete Examples

**Example 1: Distilling a math reasoning debate system into a single model**

User: "We have a 4-agent debate system for math word problems that gets 85% accuracy but costs 4x inference. How do I distill it into one model?"

Approach:
1. Collect debate logs from all 4 agents across your training set (e.g., GSM8K, MATH)
2. Filter to debates where final consensus matches ground truth
3. Extract corrective trajectories where agents caught each other's arithmetic or logic errors
4. Apply DA strategy: use GPT-4 as teacher to extract 2-3 diverse solution paths per problem
5. Fine-tune your target model (e.g., Qwen3-8B) on the augmented dataset
6. If accuracy gap remains >3%, upgrade to PAD: train a PRM on step-labeled data, run GRPO

Output structure:
```
training_data/
  debate_logs/          # Raw multi-agent outputs
  filtered_correct/     # Correctness-verified trajectories
  augmented/            # Teacher-extracted diverse paths
  step_labels/          # Per-step correctness labels for PRM
models/
  rsft_baseline/        # Stage 1: R-SFT model
  da_model/             # Stage 2: DA-trained model
  prm_checkpoint/       # Process Reward Model
  pad_final/            # Stage 3: GRPO-optimized model
```

**Example 2: Building a self-correcting code review agent**

User: "I want a single model that reviews code like a team of reviewers would -- catching bugs, style issues, and logic errors in one pass."

Approach:
1. Set up 3-agent debate where each agent reviews code from a different angle (correctness, performance, maintainability)
2. Run debates on a corpus of PRs with known issues
3. Collect trajectories where Agent B caught a bug Agent A missed, or Agent C refined Agent B's suggestion
4. Extract corrective patterns: "Initially missed null check -> peer pointed out edge case -> revised to include guard clause"
5. Fine-tune with R-SFT on (code_diff, multi_perspective_review) pairs
6. For higher quality, apply PAD with a PRM trained to score review completeness at each review step

Output: A single model that generates reviews structured as:
```
## Correctness
- Line 42: Potential null dereference when `user.profile` is undefined [HIGH]

## Performance
- Line 78: N+1 query inside loop -- batch fetch outside [MEDIUM]

## Self-correction
- Initially considered line 55 safe, but on reflection the type cast
  could fail for union types. Recommend explicit type guard.
```

**Example 3: Reducing a multi-agent summarization pipeline**

User: "Our 3-agent summarize-critique-refine pipeline produces great summaries but is too slow for production."

Approach:
1. Collect (document, agent1_draft, agent2_critique, agent3_refined_summary) tuples
2. Focus on cases where the critique meaningfully improved the summary (changed factual claims, fixed omissions)
3. For R-SFT: train on (document -> refined_summary) with the full critique chain as the reasoning trace
4. For DA: extract diverse summarization strategies (extractive key points vs. abstractive narrative vs. structured outline)
5. The distilled model learns to internally generate-critique-refine, producing first-pass summaries at refined quality

## Best Practices

**Do:**
- Prioritize corrective trajectories (wrong-to-right pivots) over clean trajectories -- they carry the highest learning signal
- Use a two-stage PRM training curriculum (frozen backbone then full fine-tune) to prevent catastrophic forgetting
- Start with R-SFT as a baseline before investing in DA or PAD -- sometimes simple fine-tuning on consensus outputs is sufficient
- Keep group size >= 4 in GRPO sampling for stable advantage estimation
- Validate PRM quality independently before using it for policy optimization -- a bad PRM will teach bad reasoning

**Avoid:**
- Flooding small models with excessive training data -- capacity-limited models degrade with too much supervision; filter aggressively
- Using only error-free debate paths -- clean trajectories teach the answer but not the self-correction behavior
- Skipping the feature-alignment stage in PRM training -- training the full model end-to-end from scratch produces unstable reward signals
- Assuming more debate agents or rounds always helps -- diminishing returns set in quickly, especially for smaller student models
- Evaluating only final-answer accuracy -- measure reasoning coherence, step decomposition, and self-correction frequency to detect quality regressions

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| Distilled model parrots debate format instead of reasoning | Training data includes raw debate markup (e.g., "Agent 1 says...") | Strip agent identity markers; keep only reasoning content |
| PRM assigns uniform scores | Stage I training was skipped or too short | Retrain with frozen backbone for at least 1 epoch before unfreezing |
| GRPO training diverges | Group size too small or learning rate too high | Increase n_samples_per_prompt to 8+, reduce LR by 2-5x |
| DA model memorizes one solution path | Teacher extracted insufficiently diverse trajectories | Tighten diversity criteria in teacher prompt; require distinct core strategies |
| Distilled model worse than vanilla SFT | Debate data quality is low (many incorrect consensuses) | Improve correctness filtering; increase debate rounds from 2 to 3 |
| Self-correction is superficial ("Let me reconsider... same answer") | Training data lacks genuine corrective pivots | Oversample trajectories with measurable reasoning changes between rounds |

## Limitations

- **Requires access to multi-agent debate data**: You need to run the multi-agent system first to generate training data. This is a one-time cost but requires the infrastructure to run N agents.
- **PRM training needs step-level labels**: PAD requires per-step correctness annotations, which are expensive to produce for non-math domains where correctness is subjective.
- **Ceiling is the multi-agent system's quality**: The distilled model cannot exceed the reasoning quality of the debate system it learned from. If the multi-agent system makes systematic errors, the distilled model inherits them.
- **Domain transfer is limited**: A model distilled on math debate data won't automatically self-correct on code review tasks. Distillation is task-family specific.
- **Small models hit capacity walls**: Models under ~7B parameters show diminishing returns from PAD -- the additional process-aware signal overwhelms their capacity. R-SFT or DA may be the practical ceiling for small models.

## Reference

**Paper**: [AgentArk: Distilling Multi-Agent Intelligence into a Single LLM Agent](https://arxiv.org/abs/2602.03955v1) (Luo et al., 2026). Look for: Table 2 comparing R-SFT/DA/PAD across model sizes, Section 4 on PRM training curriculum, and the ablation on corrective vs. clean trajectory selection.

**Code**: [github.com/AIFrontierLab/AgentArk](https://github.com/AIFrontierLab/AgentArk) -- contains debate configs, PRM training scripts, GRPO integration with OpenRLHF, and evaluation harness.