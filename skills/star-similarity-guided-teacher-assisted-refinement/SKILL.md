---
name: "star-similarity-guided-teacher-assisted-refinement"
description: "Distill function-calling capabilities from large language models into super-tiny models (0.6B-4B) using the STAR framework: Constrained Knowledge Distillation + Similarity-guided Reinforcement Learning. Use when: 'distill function calling into a small model', 'train a tiny tool-use model', 'compress an LLM for function calling', 'knowledge distillation for agent models', 'train a sub-1B function calling model', 'fine-tune a small model for tool use with RL'."
---

# STAR: Similarity-guided Teacher-Assisted Refinement for Super-Tiny Function Calling Models

This skill enables Claude to implement the STAR training framework, which transfers function-calling capabilities from large teacher LLMs (8B+) into super-tiny student models (0.6B-4B parameters). STAR combines Constrained Knowledge Distillation (CKD) — a modified KL divergence loss that suppresses confidently wrong predictions — with Similarity-guided Reinforcement Learning (Sim-RL) — a fine-grained, continuous reward signal based on structural similarity between predicted and ground-truth function calls. The two-stage curriculum (distill then refine) produces tiny models that outperform much larger open models on standard function-calling benchmarks (BFCL v3, ACEBench).

## When to Use

- When the user wants to distill function-calling or tool-use capabilities from a large model (e.g., Qwen3-8B, Llama-70B) into a model under 4B parameters
- When building an on-device or edge-deployed AI agent that needs reliable function calling at low latency
- When standard knowledge distillation produces a student model that overfits or generates confidently wrong tool calls
- When binary pass/fail rewards in RL training fail to improve function-calling quality because tasks have multiple valid solutions
- When the user needs a training pipeline that combines KD and RL without one stage undermining the other
- When fine-tuning a small model for structured output (JSON function calls) and the model keeps producing format errors or hallucinated arguments

## Key Technique

**Constrained Knowledge Distillation (CKD)** replaces standard forward KL divergence with a two-part objective. The first part computes top-k forward KL divergence, focusing the student only on the teacher's top-k most probable tokens (k=100) rather than the full vocabulary. This prevents the student from wasting capacity modeling the teacher's long tail of near-zero probabilities. The second part is a tail penalty: it explicitly penalizes the student for assigning high probability to tokens outside the teacher's top-k set. The combined loss is `L_CKD = L_FKL-k + lambda_tail * L_tail` where `lambda_tail=10`. This suppresses "confident-but-wrong" predictions — the primary failure mode in small-model distillation — while preserving enough entropy in the student's distribution for downstream RL to explore effectively.

**Similarity-guided RL (Sim-RL)** replaces binary rewards with a continuous, composite reward signal bounded in [-1, 1]. It evaluates three dimensions: (1) **Format reward** — binary check that the output is valid JSON with correct tags; (2) **Function-call reward** — an IoU-inspired score that matches predicted function calls to ground-truth calls via Hungarian matching, then scores each matched pair using exact match for numbers/booleans and ROUGE-L F1 for string arguments; (3) **Response reward** — ROUGE-L F1 between predicted and ground-truth text responses. The total reward is `R = (R_format - 1) + R_format * (R_fc + R_response)`, so format errors yield an immediate -1 penalty while correct format unlocks graded evaluation. This continuous signal gives the RL optimizer (GRPO) meaningful gradients even for partially correct outputs, which binary rewards cannot.

**The two-stage curriculum** prevents the common failure where RL undoes distillation gains or KD leaves no room for RL improvement. Stage 1 first refines the teacher using Sim-RL, then distills into the student with CKD. Stage 2 applies Sim-RL directly to the distilled student to polish its policy. This ordering matters: CKD preserves exploration capacity that Sim-RL then exploits.

## Step-by-Step Workflow

1. **Select teacher and student architectures.** Choose a teacher model with proven function-calling ability (e.g., Qwen3-8B-Instruct) and a student in the 0.6B-4B range from the same model family to maximize vocabulary alignment. Verify both use the same tokenizer or plan a vocabulary mapping.

2. **Prepare the training dataset.** Merge function-calling datasets (ToolACE, xLAM, xLAM-irrelevance, tool-use-synthetic) to reach ~128k samples. Each sample must include: system prompt with tool definitions, user query, and ground-truth function call(s) in structured JSON format. Include irrelevance samples (queries where no tool applies) to teach the model to abstain.

3. **Refine the teacher with Sim-RL.** Run GRPO on the teacher model using the composite similarity reward. Use learning rate 3e-7, batch size 128, 8 rollouts per prompt, KL coefficient 1e-3. This produces a teacher whose output distribution is better aligned with the reward signal that the student will later optimize against.

4. **Implement the CKD loss function.** Code the two-part loss: (a) compute forward KL divergence only over the teacher's top-k=100 tokens per position; (b) compute the tail penalty as the sum of student probabilities assigned to the student's top-m=100 tokens that fall outside the teacher's top-k set. Combine as `L_CKD = L_FKL_k + 10.0 * L_tail`.

5. **Distill teacher into student with CKD (Stage 1).** Train the student on the full dataset using CKD loss with learning rate 3e-6, batch size 128. Monitor both the top-k KL divergence and the tail penalty separately — if the tail penalty plateaus while KL is still high, increase lambda_tail.

6. **Implement the Sim-RL reward function.** Code the three-component reward: (a) format check via JSON schema validation; (b) function-call similarity using Hungarian matching across predicted/ground-truth call pairs, scoring arguments with ROUGE-L for strings and exact match for primitives, aggregated with IoU normalization; (c) response similarity via ROUGE-L F1.

7. **Refine the student with Sim-RL (Stage 2).** Run GRPO on the distilled student using the composite reward. Use the same RL hyperparameters as teacher refinement. Apply reward group filtering: skip GRPO updates for prompt groups where all rollouts receive identical rewards (no gradient signal).

8. **Evaluate on function-calling benchmarks.** Test on BFCL v3 (multi-turn, parallel, relevance detection) and ACEBench (format generalization). Report overall accuracy, per-category scores, and compare against the teacher and baselines at equivalent size.

9. **Ablate components.** Run the pipeline with CKD-only, Sim-RL-only, and standard SFT to verify each component contributes. Check that CKD alone outperforms vanilla KD, and that Sim-RL on top of CKD outperforms Sim-RL on top of SFT.

10. **Export and deploy.** Quantize the final student model (GGUF/AWQ) and benchmark inference latency on target hardware. Verify that function-calling accuracy holds after quantization.

## Concrete Examples

**Example 1: Implementing CKD Loss in PyTorch**

User: "I have a Qwen3-0.6B student and Qwen3-8B teacher. Write the CKD loss function."

Approach:
1. Get teacher and student logits for the same input batch
2. Compute teacher top-k token indices per position
3. Compute forward KL only over those top-k tokens
4. Compute tail penalty on student's confident tokens outside teacher's top-k
5. Combine with lambda_tail weighting

Output:
```python
import torch
import torch.nn.functional as F

def ckd_loss(student_logits, teacher_logits, k=100, m=100, lambda_tail=10.0):
    """Constrained Knowledge Distillation loss from STAR framework."""
    # Shape: (batch, seq_len, vocab_size)
    teacher_probs = F.softmax(teacher_logits, dim=-1)
    student_log_probs = F.log_softmax(student_logits, dim=-1)
    student_probs = F.softmax(student_logits, dim=-1)

    # Top-k Forward KL: only over teacher's top-k tokens
    teacher_topk_vals, teacher_topk_ids = teacher_probs.topk(k, dim=-1)
    student_topk_log_probs = student_log_probs.gather(-1, teacher_topk_ids)
    teacher_topk_log_probs = teacher_topk_vals.log()

    # FKL = sum P_T * log(P_T / P_S) over top-k tokens
    fkl_k = (teacher_topk_vals * (teacher_topk_log_probs - student_topk_log_probs)).sum(-1)
    fkl_k = fkl_k.mean()

    # Tail penalty: student probability mass on its top-m tokens that are NOT in teacher's top-k
    student_topm_vals, student_topm_ids = student_probs.topk(m, dim=-1)

    # Build mask: which of student's top-m are outside teacher's top-k
    teacher_topk_set = teacher_topk_ids.unsqueeze(-1)  # (B, S, k, 1)
    student_topm_set = student_topm_ids.unsqueeze(-2)  # (B, S, 1, m)
    in_teacher_topk = (student_topm_set == teacher_topk_set).any(dim=-2)  # (B, S, m)
    outside_mask = ~in_teacher_topk

    tail_penalty = (student_topm_vals * outside_mask.float()).sum(-1).mean()

    return fkl_k + lambda_tail * tail_penalty
```

**Example 2: Implementing the Sim-RL Composite Reward**

User: "Write the similarity-based reward function for function-calling RL training."

Approach:
1. Parse predicted output into structured function calls
2. Check format validity (JSON, correct tags)
3. Match predicted calls to ground-truth calls using Hungarian algorithm
4. Score each matched pair with argument-level similarity
5. Aggregate into bounded [-1, 1] reward

Output:
```python
import json
import re
from scipy.optimize import linear_sum_assignment
from rouge_score import rouge_scorer

scorer = rouge_scorer.RougeScorer(['rougeL'], use_stemmer=False)

def compute_reward(prediction: str, ground_truth: dict) -> float:
    """STAR composite reward: format + function_call + response."""
    # 1. Format reward
    r_format = 1.0 if _valid_format(prediction) else 0.0
    if r_format == 0.0:
        return -1.0  # R = (R_format - 1) + R_format * (...) = -1

    parsed = _parse_function_calls(prediction)
    gt_calls = ground_truth.get("function_calls", [])
    gt_response = ground_truth.get("response", "")

    # 2. Function call reward (IoU-inspired with Hungarian matching)
    r_fc = _function_call_reward(parsed["calls"], gt_calls)

    # 3. Response reward (ROUGE-L F1)
    r_response = 0.0
    if gt_response:
        score = scorer.score(gt_response, parsed.get("response", ""))
        r_response = score['rougeL'].fmeasure

    return (r_format - 1) + r_format * (r_fc + r_response)

def _function_call_reward(pred_calls, gt_calls):
    if not gt_calls and not pred_calls:
        return 1.0
    if not gt_calls or not pred_calls:
        return 0.0
    n, m = len(pred_calls), len(gt_calls)
    # Build similarity matrix for Hungarian matching
    sim_matrix = []
    for p in pred_calls:
        row = [_call_similarity(p, g) for g in gt_calls]
        sim_matrix.append(row)
    import numpy as np
    cost = -np.array(sim_matrix)
    row_ind, col_ind = linear_sum_assignment(cost)
    matched_sim = sum(sim_matrix[r][c] for r, c in zip(row_ind, col_ind))
    # IoU normalization
    union = n + m - len(row_ind)
    return matched_sim / max(union, 1)

def _call_similarity(pred, gt):
    if pred.get("name") != gt.get("name"):
        return 0.0
    pred_args = pred.get("arguments", {})
    gt_args = gt.get("arguments", {})
    all_keys = set(pred_args.keys()) | set(gt_args.keys())
    if not all_keys:
        return 1.0
    total = 0.0
    for key in all_keys:
        pv, gv = pred_args.get(key), gt_args.get(key)
        if pv is None or gv is None:
            continue
        if isinstance(gv, str):
            total += scorer.score(gv, str(pv))['rougeL'].fmeasure
        else:
            total += 1.0 if pv == gv else 0.0
    return total / len(all_keys)
```

**Example 3: Full Two-Stage Training Curriculum**

User: "Set up the STAR training pipeline for distilling Qwen3-8B into Qwen3-0.6B for function calling."

Approach:
1. Configure datasets and preprocessing
2. Run Stage 1: teacher refinement + CKD distillation
3. Run Stage 2: student refinement with Sim-RL
4. Evaluate on benchmarks

Output:
```yaml
# star_training_config.yaml
teacher:
  model: Qwen/Qwen3-8B-Instruct
  tokenizer: Qwen/Qwen3-8B-Instruct

student:
  model: Qwen/Qwen3-0.6B
  tokenizer: Qwen/Qwen3-0.6B

datasets:
  - name: ToolACE
    samples: 11300
  - name: xLAM
    samples: 60000
  - name: xLAM-irrelevance
    samples: 6700
  - name: tool-use-synthetic
    samples: 50000

stage1_teacher_refinement:
  method: GRPO
  learning_rate: 3.0e-7
  batch_size: 128
  rollouts_per_prompt: 8
  kl_coefficient: 1.0e-3
  reward: sim_rl_composite
  epochs: 1

stage1_distillation:
  method: CKD
  learning_rate: 3.0e-6
  batch_size: 128
  top_k: 100
  student_top_m: 100
  lambda_tail: 10.0
  epochs: 3

stage2_student_refinement:
  method: GRPO
  learning_rate: 3.0e-7
  batch_size: 128
  rollouts_per_prompt: 8
  kl_coefficient: 1.0e-3
  reward: sim_rl_composite
  filter_homogeneous_groups: true
  epochs: 1

evaluation:
  benchmarks:
    - BFCL_v3
    - ACEBench
  metrics:
    - overall_accuracy
    - per_category_accuracy
    - format_error_rate

hardware:
  gpus: 8
  gpu_type: H20
```

## Best Practices

- **Do:** Use the same model family for teacher and student (e.g., both Qwen3) to maximize tokenizer alignment and distribution compatibility for CKD.
- **Do:** Include irrelevance/abstention samples (5-10% of dataset) so the model learns when NOT to call a function — this is critical for real-world deployment.
- **Do:** Monitor the tail penalty and FKL-k as separate metrics during CKD training. A rising tail penalty with flat FKL-k signals that lambda_tail is too low.
- **Do:** Filter homogeneous reward groups in GRPO — when all rollouts for a prompt get the same reward, the gradient is zero and the update adds noise.
- **Avoid:** Using standard reverse KL (RKL) divergence for distillation of function-calling models. RKL causes mode collapse that destroys the student's ability to handle multi-solution tasks.
- **Avoid:** Skipping teacher refinement (Stage 1a). An unrefined teacher's distribution may not align well with the Sim-RL reward, leading to a distilled student that starts Stage 2 in a poor region of policy space.
- **Avoid:** Setting lambda_tail too high (>50). Excessive tail suppression collapses the student's entropy, eliminating the exploration capacity needed for Stage 2 RL.

## Error Handling

- **Format error rate spikes during RL:** The student starts generating malformed JSON. Increase the format reward penalty weight or add a curriculum that starts with format-only reward before introducing function-call similarity.
- **CKD loss diverges:** The top-k sets between teacher and student have near-zero overlap early in training. Warm up with standard SFT for 100-500 steps before switching to CKD.
- **GRPO updates are noisy:** Most reward groups are homogeneous (all rollouts identical). Increase rollouts per prompt from 8 to 16, or reduce temperature to increase response diversity within each group.
- **Hungarian matching is slow:** For prompts with many function calls (>20), the O(n^3) matching becomes a bottleneck. Cap at 20 calls per sample or use greedy matching as an approximation during training.
- **Student forgets distilled knowledge during Stage 2 RL:** The KL coefficient (1e-3) is too low. Increase to 1e-2 or add an explicit KL penalty against the Stage 1 checkpoint.

## Limitations

- STAR requires access to the teacher model's logits (not just outputs), so it cannot work with closed-source API-only teachers like GPT-4 or Claude. You need an open-weight teacher.
- The framework is validated on function-calling tasks specifically. Transferability to other structured-output tasks (SQL generation, API composition) is plausible but unverified.
- The Sim-RL reward uses ROUGE-L for string arguments, which is a lexical metric. It may underreward semantically correct but lexically different arguments (e.g., "New York City" vs "NYC").
- Super-tiny models (0.6B) still significantly underperform 8B+ models on complex multi-turn function calling. STAR narrows the gap but does not close it.
- The training pipeline requires 8 H20 GPUs. Adapting to consumer hardware (single GPU) requires gradient accumulation and likely longer training times.

## Reference

**Paper:** [STAR: Similarity-guided Teacher-Assisted Refinement for Super-Tiny Function Calling Models](https://arxiv.org/abs/2602.03022v1) (ICLR 2026)
**Code:** https://github.com/Qwen-Applications/STAR
**Key insight:** Combining top-k constrained KL distillation (to suppress confident errors) with continuous similarity-based RL rewards (to handle multi-solution tasks) in a two-stage curriculum produces 0.6B models that outperform many open models 10x their size on function-calling benchmarks.