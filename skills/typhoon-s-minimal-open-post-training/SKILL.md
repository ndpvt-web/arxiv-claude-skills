---
name: "typhoon-s-minimal-open-post-training"
description: |
  Design and implement minimal post-training pipelines for sovereign LLMs using the Typhoon-S recipe:
  SFT + on-policy distillation + InK-GRPO reinforcement fine-tuning.
  Trigger phrases:
  - "post-train a base model into an instruction-tuned model"
  - "set up sovereign LLM fine-tuning with minimal data"
  - "configure InK-GRPO training for domain-specific reasoning"
  - "build an on-policy distillation pipeline from a teacher model"
  - "fine-tune an LLM for legal reasoning in a low-resource language"
  - "create a minimal post-training recipe without preference tuning"
---

# Typhoon-S: Minimal Open Post-Training for Sovereign LLMs

This skill enables Claude to design, configure, and implement minimal post-training pipelines that transform base language models into high-quality instruction-tuned assistants — particularly for sovereign or low-resource language settings. It applies the Typhoon-S three-stage recipe (SFT, on-policy distillation, InK-GRPO) to produce capable models without massive instruction corpora or complex RLHF pipelines.

## When to Use

- When the user wants to post-train a base LLM into a general-purpose instruction-following assistant with limited compute (4-8 GPUs)
- When building a domain-specific model (legal, medical, cultural) for a non-English language while preserving general capabilities
- When configuring on-policy distillation from a larger teacher model to a smaller student model using GKD
- When implementing InK-GRPO — reinforcement fine-tuning that injects domain knowledge via a next-token prediction loss alongside policy optimization
- When designing a data pipeline that generates verifiable instruction-following data (AutoIF) for a target language
- When the user needs to avoid catastrophic forgetting during domain-specific reinforcement fine-tuning
- When setting up agentic RFT with tool use (search + read) for retrieval-augmented reasoning tasks

## Key Technique

**The core insight of Typhoon-S is that a carefully staged three-phase post-training recipe can match or exceed models trained with far more data and compute.** The three stages are: (1) supervised fine-tuning on ~340k samples to teach instruction-following, (2) on-policy distillation using Generalized Knowledge Distillation (GKD) where the student generates responses and the teacher provides soft token-level supervision, and (3) small-scale reinforcement fine-tuning using InK-GRPO to inject domain-specific knowledge without degrading general capabilities.

**InK-GRPO is the paper's primary algorithmic contribution.** Standard GRPO optimizes a policy using reward-weighted likelihood, but domain-specific RL training risks catastrophic forgetting. InK-GRPO augments the GRPO loss with a stochastic cross-entropy (next-token prediction) loss: `L = L_GRPO + lambda * b * L_CE`, where `b ~ Bernoulli(rho)`. This means with probability `rho` (default 0.6), the model also trains on a standard language modeling objective over domain-specific text, anchoring its knowledge while the RL objective improves reasoning. The stochastic activation prevents the CE loss from dominating and allows the RL signal to drive behavioral change.

**The on-policy distillation stage uses full-logits GKD rather than top-K approximation.** At each step, with probability 0.25 the student generates its own response (on-policy), otherwise a reference response is used. The teacher then provides full next-token probability distributions, and the student minimizes forward KL divergence against the teacher. This is more robust than top-K distillation and handles distribution mismatch better. Dynamic model swapping with FSDP CPU offloading makes this feasible on 4-8 H100 GPUs.

## Step-by-Step Workflow

1. **Prepare the SFT data mixture (~340k samples).** Combine a general instruction dataset (e.g., Tulu 3 SFT, ~200k), a tool-use dataset (e.g., Toucan Tool, ~100k), and target-language instruction data generated via AutoIF (~40k). Ensure format consistency with chat templates including system, user, and assistant roles.

2. **Run supervised fine-tuning on the base model.** Use AdamW optimizer with learning rate `2e-5`, batch size 32, for 2 epochs. Enable sequence packing up to 16,384 tokens to maximize throughput. This stage teaches the model basic instruction-following and chat behavior.

3. **Prepare the on-policy distillation (OPD) data mixture (~160k samples).** Curate a subset: ~100k general instructions, ~40k target-language AutoIF data, ~20k tool-use examples. This is a filtered, higher-quality subset of the SFT data.

4. **Run on-policy distillation using GKD.** Configure the student-data fraction (lambda) to 0.25, meaning 25% of training uses student-generated responses. Set learning rate to `1e-6`, run for 1 epoch. Use full-logits forward KL divergence as the objective. Deploy FSDP CPU offloading to fit both student and teacher models in GPU memory.

5. **Prepare domain-specific RFT data.** For the target domain (e.g., legal reasoning), collect question-answer pairs with supporting contexts. For agentic settings, set up a vector database (e.g., FAISS IVF-SQ8 with a small embedding model) and define tool schemas for `search` (returns top-3 documents) and `read` (returns full document content).

6. **Configure InK-GRPO training.** Set `lambda=0.1` (CE loss weight), `rho=0.6` (Bernoulli activation probability), learning rate `1e-6`, temperature 0.7. Use DAPO-style decoupled clipping with `clip_high=0.24, clip_low=0.20`. The reward function should weight accuracy at 0.9 and format compliance at 0.1.

7. **Define the reward function.** For accuracy: use an LLM-as-judge scoring on a 0-2 scale (incorrect=0, partial=1, full=2), normalized to [0,1]. For format: check for proper `<thinking></thinking>` tag placement (binary 0/1). For agentic settings, use accuracy-only rewards since tool-call structure provides implicit reasoning scaffolding.

8. **Run InK-GRPO training.** Generate rollouts at temperature 0.7, compute rewards, and update with the combined GRPO + stochastic CE loss. Mask tool outputs during gradient computation in agentic settings. Apply overlong reward shaping to penalize excessively verbose responses. Target ~1 day on 4xH100 for a 4B model.

9. **Evaluate for catastrophic forgetting.** Track an unweighted mean across general benchmarks (chat quality, instruction-following, knowledge, math, code, agentic tasks). Performance should remain within ~2 points of the pre-RFT baseline. If degradation exceeds this, reduce lambda or rho.

10. **Evaluate sovereign capabilities.** Test on domain-specific benchmarks (e.g., NitiBench for Thai legal reasoning, MIRAGE-Bench for multilingual RAG). Compare against both the pre-RFT checkpoint and strong proprietary baselines to validate domain improvement.

## Concrete Examples

**Example 1: Setting up SFT + OPD for a sovereign base model**

User: "I have a Qwen3-8B base model adapted for Vietnamese. I want to turn it into an instruction-following assistant. I have 8 H100 GPUs."

Approach:
1. Assemble SFT data: 200k general instructions (Tulu 3), 100k tool-use samples, 40k Vietnamese AutoIF data
2. Configure SFT training script:

```bash
# train_sft.sh
python -m trl.scripts.sft \
  --model_name_or_path ./qwen3-8b-vietnamese-base \
  --dataset_path ./data/sft_340k.jsonl \
  --output_dir ./checkpoints/sft \
  --learning_rate 2e-5 \
  --per_device_train_batch_size 4 \
  --gradient_accumulation_steps 1 \
  --num_train_epochs 2 \
  --max_seq_length 16384 \
  --packing true \
  --bf16 true
```

3. Configure OPD with GKD using a 30B teacher:

```bash
# train_opd.sh
python -m trl.scripts.distill \
  --student_model_path ./checkpoints/sft \
  --teacher_model_name_or_path Qwen/Qwen3-30B-A3B-Instruct \
  --dataset_path ./data/opd_160k.jsonl \
  --output_dir ./checkpoints/opd \
  --learning_rate 1e-6 \
  --student_data_fraction 0.25 \
  --num_train_epochs 1 \
  --distillation_objective forward_kl \
  --full_logits true \
  --fsdp_cpu_offload true
```

Output: An instruction-tuned 8B model with strong general chat, tool use, and Vietnamese language capabilities.

**Example 2: Adding legal reasoning via InK-GRPO**

User: "My instruction-tuned model is ready but scores poorly on Thai legal benchmarks. I want to improve legal reasoning without hurting general performance."

Approach:
1. Prepare legal QA pairs with reference answers and supporting statute texts
2. Build a FAISS index over the legal corpus using a small embedding model
3. Configure InK-GRPO with agentic tool use:

```python
# ink_grpo_config.py
config = {
    "model_path": "./checkpoints/opd",
    "output_dir": "./checkpoints/ink-grpo-legal",

    # InK-GRPO specific
    "ink_lambda": 0.1,          # CE loss weight
    "ink_rho": 0.6,             # Bernoulli activation probability
    "ce_data_path": "./data/legal_corpus.jsonl",  # Domain text for CE loss

    # GRPO settings
    "learning_rate": 1e-6,
    "temperature": 0.7,
    "clip_high": 0.24,
    "clip_low": 0.20,

    # Reward
    "reward_accuracy_weight": 0.9,
    "reward_format_weight": 0.1,
    "reward_judge_model": "gpt-4o",

    # Agentic tool use
    "tools": ["search", "read"],
    "retriever_model": "Qwen/Qwen3-Embedding-0.6B",
    "faiss_index_type": "IVF_SQ8",
    "search_top_k": 3,
    "mask_tool_outputs": True,
}
```

4. Monitor general benchmarks alongside legal accuracy during training

Output: Legal reasoning improves by ~4% on NitiBench-style benchmarks while general capabilities remain within 2 points of baseline.

**Example 3: Generating target-language instruction data with AutoIF**

User: "I need to create Thai instruction-following data for the SFT stage. How do I use AutoIF?"

Approach:
1. Source Thai prompts from translated WildChat English data and existing Thai instruction sets
2. Generate responses using a strong model (e.g., Qwen3-235B) with code-verifiable constraints:

```python
# autoif_pipeline.py
# Stage 1: Define verifiable constraints using few-shot examples
constraint_prompt = """
Given this instruction, define a testable constraint.

Instruction: "Write a summary of Thai labor law in 3 paragraphs"
Constraint (Python):
def verify(response):
    paragraphs = response.strip().split('\\n\\n')
    return len(paragraphs) == 3

Instruction: "{user_instruction}"
Constraint (Python):
"""

# Stage 2: Generate responses and filter via constraint verification
# Reject responses scoring below 7 on quality scale

# Stage 3: Augment by randomly:
# - Translating constraints between English and Thai
# - Placing constraints in system message vs user prompt
# This improves cross-lingual alignment and robustness
```

3. Target ~40k verified samples for the Thai-specific portion of SFT data

Output: 40k high-quality, constraint-verified Thai instruction-response pairs ready for the SFT mixture.

## Best Practices

- **Do** use full-logits distillation rather than top-K approximation during OPD — it is more robust to distribution mismatch and avoids mode-dropping artifacts.
- **Do** keep InK-GRPO's stochastic CE activation probability (rho) between 0.5-0.7. Too low and you lose the knowledge anchoring effect; too high and the CE loss dominates the RL signal.
- **Do** mask tool outputs during gradient computation in agentic RFT — training on tool outputs teaches the model to mimic retrieval artifacts rather than reason about them.
- **Do** track an unweighted mean across diverse general benchmarks (chat, math, code, knowledge) to detect catastrophic forgetting early.
- **Avoid** scaling instruction data beyond necessity. The paper shows 340k SFT + 160k OPD samples are sufficient — doubling data volume does not proportionally improve results.
- **Avoid** complex multi-stage preference tuning (DPO/RLHF) when InK-GRPO with task-specific rewards achieves comparable results with less infrastructure.
- **Avoid** using format rewards in agentic settings. Tool-call structure already provides reasoning scaffolding, and format rewards can conflict with natural agentic behavior.

## Error Handling

- **Catastrophic forgetting during RFT**: If general benchmark scores drop more than 2 points, reduce `ink_lambda` (try 0.05) or `ink_rho` (try 0.4). Alternatively, increase the proportion of general-domain text in the CE loss corpus.
- **OPD memory issues with large teacher**: Use FSDP CPU offloading. If still insufficient, switch to a MoE teacher (e.g., Qwen3-30B-A3B which activates only 3B parameters) or reduce the student-data fraction to lower the number of concurrent forward passes.
- **Reward hacking in InK-GRPO**: If the model learns to game the reward function (e.g., producing superficially correct but vacuous answers), add overlong reward shaping to penalize verbose responses and tighten the accuracy reward to require exact match on key terms.
- **AutoIF constraint verification failures**: If too many generated responses fail constraint checks, simplify constraints to structural properties (length, format, keyword presence) rather than semantic correctness. Semantic filtering should use a separate quality judge.
- **Low on-policy sample quality during OPD**: If the student generates poor responses early in training, reduce the student-data fraction from 0.25 to 0.1 for the first 20% of training, then anneal back to 0.25.

## Limitations

- **Compute floor**: Even the minimal recipe requires 4xH100 GPUs for ~1 day for RFT on a 4B model. For 8B+ models, expect 8 GPUs and longer training. This is academic-scale, not consumer-scale.
- **Teacher model dependency**: On-policy distillation requires a strong teacher (30B+ parameters recommended). If no suitable teacher exists for your language, the OPD stage may underperform and you may need to rely more heavily on SFT data quality.
- **Reward model quality**: InK-GRPO's effectiveness depends on reward signal quality. For domains without clear ground truth (creative writing, open-ended reasoning), LLM-as-judge rewards may be noisy and unreliable.
- **Language transfer assumptions**: The AutoIF pipeline assumes a strong multilingual base model that can follow constraints in the target language. For truly low-resource languages with poor base model coverage, the constraint verification step may fail frequently.
- **Single-domain RFT**: The paper demonstrates InK-GRPO on legal reasoning. Applying it simultaneously to multiple domains (legal + medical + financial) is unexplored and may require separate RFT runs or careful data balancing.

## Reference

**Paper**: [Typhoon-S: Minimal Open Post-Training for Sovereign Large Language Models](https://arxiv.org/abs/2601.18129v1) — Look for Section 3 (InK-GRPO loss formulation), Section 2.2 (GKD on-policy distillation procedure), and Table 3/8/11 (benchmark results demonstrating minimal forgetting).
**Code**: [github.com/scb-10x/typhoon-s](https://github.com/scb-10x/typhoon-s) — Training scripts in `trl/` (SFT + OPD) and `verl/` (InK-GRPO).
**Data & Models**: [HuggingFace: typhoon-ai/typhoon-s](https://huggingface.co/collections/typhoon-ai/typhoon-s) — 516k post-training dataset and 30.1k sovereign capability dataset.