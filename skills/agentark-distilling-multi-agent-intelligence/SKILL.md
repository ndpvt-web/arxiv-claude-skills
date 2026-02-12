---
name: "agentark-distilling-multi-agent-intelligence"
description: "Distill multi-agent debate dynamics into a single LLM through hierarchical training strategies. Use when: 'set up multi-agent distillation pipeline', 'distill debate into single model', 'train process reward model for reasoning', 'run GRPO fine-tuning on reasoning traces', 'generate diverse reasoning trajectories from agent debate', 'build AgentArk distillation workflow'."
---

# AgentArk: Distilling Multi-Agent Intelligence into a Single LLM Agent

This skill enables Claude to help users implement the AgentArk distillation framework — a pipeline that runs multi-agent debates to generate diverse reasoning trajectories, then distills those collective dynamics into a single model's weights through three hierarchical strategies: reasoning-enhanced supervised fine-tuning (R-SFT), trajectory-based data augmentation (DA), and process-aware distillation (PAD) with a trained process reward model and GRPO reinforcement learning. The result is a single LLM that reasons with the quality of a multi-agent system at the inference cost of one model.

## When to Use

- When the user wants to improve a single model's reasoning by leveraging multi-agent debate data, rather than deploying expensive multi-agent systems at inference time
- When setting up a training pipeline that generates reasoning trajectories from LLM debates and fine-tunes a student model on them
- When the user needs to train a Process Reward Model (PRM) that scores intermediate reasoning steps, not just final answers
- When implementing GRPO (Group Relative Policy Optimization) using a PRM as the reward signal for reasoning tasks
- When the user asks to generate diverse, correctness-filtered training data from multi-agent interactions on math, QA, or medical reasoning datasets
- When configuring the AgentArk repo (inference.py, label.py, prm/finetune2.py, openrlhf GRPO) for an end-to-end distillation run

## Key Technique

**Core insight:** Multi-agent debate systems (where N agents argue over a problem for K rounds) produce high-quality reasoning, but cost N*K inference calls per problem. AgentArk shifts this cost from inference to training by collecting debate trajectories once, then distilling the collective intelligence into a single model's weights. The distilled model learns to self-correct and reason from multiple angles without needing peer agents at test time.

**Three hierarchical strategies, each building on the last:**

1. **R-SFT (Reasoning-Enhanced SFT):** Fine-tunes the student on `(problem, reasoning_trace, answer)` triples extracted from successful debate trajectories. The loss has two terms — a reasoning loss over intermediate tokens and an answer loss over the final prediction — forcing the model to learn *how* to reason, not just *what* to answer.

2. **DA (Data Augmentation via Correctness-First Diverse Extraction):** Uses a teacher LLM to extract k=1..3 structurally distinct trajectories per problem from debate logs, filtered for correctness against ground truth. The student trains on all k trajectories per problem, internalizing multiple valid solution paths rather than memorizing one.

3. **PAD (Process-Aware Distillation):** The most powerful strategy. First trains a PRM (Process Reward Model) initialized from the student's weights in two stages — frozen backbone (reward head only) then full fine-tuning. Then runs GRPO: samples G reasoning outputs from the student, scores each with the PRM, computes normalized advantages, and optimizes a clipped surrogate objective with KL penalty against a reference policy. This teaches the student to self-evaluate reasoning steps, not just produce them.

**Results:** PAD achieves +4.8% average accuracy improvement over a single agent baseline across math (MATH, GSM8K), medical (MedMCQA), and multi-hop QA (HotpotQA) tasks, approaching vanilla multi-agent performance at a fraction of the inference cost. Cross-family distillation (e.g., debate data from Qwen used to train LLaMA) yields larger, more consistent gains than same-family.

## Step-by-Step Workflow

### 1. Set up the environment and clone AgentArk

```bash
git clone https://github.com/AIFrontierLab/AgentArk.git
cd AgentArk
pip install -r requirements.txt
# Key deps: transformers, vllm, flash-attn, deepspeed, trl, torch, datasets, wandb
# Requires Python 3.10+, CUDA 12.5, 40GB+ GPU
```

### 2. Configure the multi-agent debate method

Choose a debate protocol by editing `methods/<method_name>/configs/config_main.yaml`. For LLM Debate:

```yaml
num_agents: 3      # 3-5 agents recommended; small models saturate at 5
num_rounds: 2      # 2-3 rounds; diminishing returns beyond 3
```

For DyLAN (dynamic LLM-agent network with listwise ranking):

```yaml
num_agents: 4
num_rounds: 3
activation: "listwise"
```

### 3. Run multi-agent inference to generate debate logs

```bash
python inference.py \
  --method_name llm_debate \
  --test_dataset_name MATH \
  --model_name Qwen/Qwen2.5-7B-Instruct \
  --model_temperature 0.5 \
  --model_max_tokens 4096 \
  --use_vllm \
  --tensor_parallel_size 2
```

This produces debate logs with per-agent reasoning traces `{r_1, ..., r_n}` and consensus answers for each problem.

### 4. Label solutions for correctness and extract trajectories

```bash
python label.py --input_file <debate_output> --dataset_name MATH
```

Output format per problem:

```json
{
  "query": "Find the value of x such that ...",
  "gt": "42",
  "solutions": [
    {"id": 1, "text": "Step 1: ... Step 2: ... Answer: 42", "is_correct": true},
    {"id": 2, "text": "Step 1: ... Answer: 37", "is_correct": false}
  ]
}
```

Filter to keep only trajectories where agents reached the correct ground-truth answer. Prioritize "corrective trajectories" — cases where an agent initially erred but self-corrected after peer critique.

### 5. Choose and execute a distillation strategy

**For R-SFT:** Format data as `(x, r, y*)` triples. Fine-tune with combined reasoning + answer loss:

```
L_SFT = -E[L_reasoning + L_answer]
L_reasoning = sum(log p(r_t | r_<t, x))   # learn to reason
L_answer = log p(y* | r, x)                # ground in correct answer
```

**For DA:** Use a teacher LLM to extract k diverse trajectories per problem from debate logs. Each must be correct (leads to y\*) and structurally distinct. Train on all k per problem:

```
L_Aug = -(1/k) * sum_i sum_t log p(y_t | y_<t, r_i, x)
```

**For PAD (recommended):** Proceed to steps 6-8.

### 6. Train the Process Reward Model (PRM)

Initialize PRM from student model weights. Stage I — freeze backbone, train reward head only:

```bash
python prm/finetune2.py \
  --model_name_or_path <student_model> \
  --fix_llm \
  --bf16 \
  --gradient_checkpointing \
  --output_dir ./prm_stage1
```

Stage II — unfreeze backbone for full specialization:

```bash
python prm/finetune2.py \
  --model_name_or_path ./prm_stage1 \
  --bf16 \
  --gradient_checkpointing \
  --output_dir ./prm_stage2
```

The PRM learns to assign step-level correctness scores z_t in {0, 1} to each intermediate reasoning step, using contrastive loss that compares steps within episodes.

### 7. Run GRPO reinforcement learning with PRM rewards

```bash
python -m openrlhf.cli.train_grpo \
  --pretrain <student_model> \
  --reward_model ./prm_stage2 \
  --reward_mode PRMVR \
  --n_samples_per_prompt 8 \
  --advantage_estimator rloo \
  --bf16
```

GRPO samples G=8 reasoning outputs per problem, scores them with the PRM, computes normalized advantages `A_i = (R(o_i) - mean) / std`, and optimizes a clipped surrogate with KL penalty against the reference (pre-training) policy.

### 8. Evaluate the distilled model

```bash
# Math tasks (exact match)
python -m eval.math_eval --input_file <results> --dataset_name MATH

# QA tasks (ROUGE, BERTScore, F1)
python -m eval.short_answer_eval \
  --model_name_or_path <distilled_model> \
  --dataset_name HotpotQA
```

Compare against: single-agent baseline, vanilla multi-agent system, and each distillation strategy to validate gains.

## Concrete Examples

**Example 1: Distilling math reasoning from Qwen debate into LLaMA**

```
User: I want to improve my LLaMA-3-8B's math reasoning without using
      multi-agent inference. Can we distill from Qwen debate traces?

Approach:
1. Run 3-agent, 2-round LLM Debate using Qwen-2.5-7B-Instruct on
   MATH + GSM8K datasets via inference.py
2. Label solutions with label.py, filter for correct consensus answers
3. Extract 3 diverse trajectories per problem using a teacher LLM
   (correctness-first diverse extraction)
4. Train PRM initialized from LLaMA-3-8B weights (Stage I frozen,
   Stage II unfrozen)
5. Run GRPO on LLaMA-3-8B with PRM rewards, n_samples_per_prompt=8

Output:
- Cross-family distillation yields larger gains than same-family
- Expect ~4-6% accuracy improvement on MATH over baseline LLaMA-3-8B
- Single-model inference cost (no multi-agent overhead at test time)
```

**Example 2: Training a PRM for step-level reasoning verification**

```
User: I have debate trajectories with labeled correct/incorrect steps.
      How do I train a PRM that scores reasoning quality?

Approach:
1. Initialize PRM from student model checkpoint to reuse its
   representations
2. Stage I: Freeze all layers except final layer + reward head.
   Train reward head on step-level correctness labels z_t in {0,1}
   using contrastive loss (compare steps within same episode)
3. Stage II: Unfreeze entire backbone. Fine-tune end-to-end so the
   model develops specialized attention patterns for fallacy detection
4. Validate PRM by checking correlation between PRM scores and actual
   step correctness on held-out data

Output:
- PRM that assigns scalar reward per reasoning step
- Use as reward signal for GRPO: advantages = (R(o_i) - mean) / std
- PRM capacity matters more than student size — invest in PRM quality
```

**Example 3: Quick R-SFT baseline without RL infrastructure**

```
User: I don't have RL training infrastructure yet. What's the simplest
      AgentArk strategy I can run with standard SFT tooling?

Approach:
1. Run multi-agent debate (3 agents, 2 rounds) on your target dataset
2. Label and filter for correct trajectories
3. Format as standard SFT data: (problem, reasoning_trace, answer)
4. Fine-tune with dual loss: reasoning loss on intermediate tokens +
   answer loss on final prediction
5. Use any SFT framework (transformers Trainer, axolotl, etc.)

Output:
- R-SFT gives moderate gains (lower than PAD but no RL needed)
- Particularly effective for smaller models (0.6B-1.7B)
- Good stepping stone before investing in full PAD pipeline
```

## Best Practices

**Do:**
- Use cross-family distillation (e.g., Qwen debate -> LLaMA student) for larger, more consistent gains than same-family
- Prioritize corrective trajectories — cases where agents initially erred but self-corrected after peer critique — as training data
- Invest in PRM quality over student size; a high-capacity PRM enables strong improvements even for small (0.6B) students
- Use the two-stage PRM curriculum (frozen then unfrozen) to prevent catastrophic forgetting of pre-trained features
- Start with R-SFT as a baseline, then layer on DA and PAD incrementally to measure marginal gains

**Avoid:**
- Using more than 5 debate agents for small student models (0.6B) — they saturate quickly; larger models benefit modestly up to 20 agents
- Training only on final answers without intermediate reasoning traces — the reasoning loss is critical for generalization
- Skipping correctness filtering of debate trajectories — incorrect traces will teach the student wrong reasoning patterns
- Expecting large gains on tasks requiring factual recall (e.g., MedMCQA) — AgentArk primarily improves reasoning process, not knowledge retrieval

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| PRM assigns near-uniform scores | Frozen-stage training too short | Increase Stage I epochs; verify labels are balanced |
| GRPO training diverges | KL penalty too low or advantage variance too high | Increase beta (KL weight); reduce n_samples_per_prompt |
| Debate logs have low consensus rate | Too few rounds or agents disagree fundamentally | Increase rounds to 3; use temperature 0.3-0.5 for more focused generation |
| DA trajectories lack diversity | Teacher LLM extracting similar paths | Explicitly prompt for distinct mathematical identities/logical heuristics in extraction |
| R-SFT overfits to training set | Small dataset or reasoning traces too homogeneous | Add DA-style augmentation; increase debate dataset size |
| Out-of-memory during GRPO | G samples * sequence length exceeds VRAM | Reduce n_samples_per_prompt to 4; enable gradient checkpointing and bf16 |

## Limitations

- **Compute-intensive data generation:** Multi-agent debate must run once to produce training data. For large datasets, this is a significant upfront cost (though amortized across all future inferences).
- **Reasoning over knowledge:** Distillation primarily improves reasoning process quality. Tasks dominated by factual recall (medical QA, trivia) see smaller gains.
- **PRM training complexity:** PAD requires a separate PRM training pipeline and RL infrastructure (GRPO), which is more complex to set up than standard SFT.
- **Debate quality ceiling:** The distilled model cannot exceed the reasoning quality of the multi-agent debate that generated its training data. Garbage in, garbage out.
- **Task transfer:** Models distilled on math reasoning transfer well to other reasoning tasks but may not generalize to fundamentally different domains (e.g., creative writing) without domain-specific debate data.

## Reference

**Paper:** [AgentArk: Distilling Multi-Agent Intelligence into a Single LLM Agent](https://arxiv.org/abs/2602.03955v1) — Luo et al., 2026. Focus on Section 3 (the three distillation strategies), Section 4 (PRM and GRPO formulations), and Tables 1-3 (comparative results across strategies, model families, and scales).

**Code:** [github.com/AIFrontierLab/AgentArk](https://github.com/AIFrontierLab/AgentArk) — Contains inference.py (debate generation), label.py (correctness labeling), prm/ (PRM training), and openrlhf integration (GRPO).