---
name: "pcl-reasoner-v15-advancing-math-reasoning-offline"
description: "Implement offline reinforcement learning pipelines for LLM reasoning tasks — decoupling data collection from training for stability and efficiency. Use when: 'set up offline RL for math reasoning', 'implement offline RL instead of GRPO', 'build a two-stage SFT then RL pipeline', 'create binary reward RL training', 'stabilize RL training for LLMs', 'offline policy optimization for reasoning models'."
---

# Offline Reinforcement Learning for LLM Reasoning (PCL-Reasoner-V1.5)

This skill enables Claude to design and implement **offline reinforcement learning pipelines** that improve LLM reasoning capabilities — particularly mathematical reasoning — by decoupling inference (data collection) from policy training. The core insight from PCL-Reasoner-V1.5 is that collecting a fixed dataset of rollouts with binary reward labels, then training on that static dataset, avoids the instability, engineering complexity, and computational overhead of online RL methods like GRPO. This "infer once, train many" paradigm achieves state-of-the-art results (90.9% on AIME 2024) while being simpler to implement.

## When to Use

- When the user wants to add an RL stage after supervised fine-tuning to boost a model's reasoning ability
- When the user is experiencing training instability with online RL methods (GRPO, PPO) and needs a more stable alternative
- When the user needs to build a data pipeline that generates candidate solutions, labels them with binary rewards, and uses them for offline policy optimization
- When the user asks how to implement reward-weighted log-likelihood training for LLMs
- When the user wants to set up a two-stage training pipeline (SFT followed by RL) for a reasoning model
- When the user needs to design a verification-based reward system using an external LLM judge
- When the user is working with limited compute and wants an RL approach that reuses a single static dataset across multiple training experiments

## Key Technique

**Offline RL vs. Online RL for LLMs:** Standard online RL methods like GRPO operate in an iterative cycle — the current policy generates rollouts, those rollouts are scored, and the policy is updated, all in a tight loop. This creates feedback loops where the training distribution shifts continuously, causing reward collapse, divergence, and requiring complex orchestration frameworks for weight synchronization between inference and training workers. PCL-Reasoner-V1.5's offline approach instead performs a single inference pass with the SFT model to collect all candidate solutions upfront, labels them with binary rewards, and then trains on this fixed dataset. This decoupling eliminates the dynamic interplay that causes instability.

**The Loss Function:** The offline RL objective is `L(theta) = -sum_i R_i * pi_theta(y_i | x_i)`, where `pi_theta` uses the geometric mean of token log-probabilities: `pi_theta(y_i | x_i) = exp((1/|y_i|) * sum_t log p_theta(y_i,t | x_i, y_i,<t))`. The length normalization (dividing by `|y_i|`) is critical — it prevents the model from favoring short incorrect answers over long correct reasoning chains. Binary rewards `R=+1` (correct) and `R=-1` (incorrect) weight each sample, so the model simultaneously learns to increase probability of correct solutions and decrease probability of incorrect ones. Importance sampling is deliberately omitted, as it was found ineffective for LLM RL training despite the distributional mismatch between the data-generating policy (SFT model) and the training policy.

**Data Collection Strategy:** The paper generates 8 candidate answers per problem from a curated set of 6,068 difficult math questions using the SFT model, producing ~48K raw rollouts. An external verifier (Qwen3-32B) assigns binary correctness labels. Problems where all 8 candidates are unanimously correct or incorrect are excluded (no reward variance = no learning signal), yielding a final dataset of 30,215 samples (14,512 positive, 15,703 negative). This roughly balanced distribution is important for training stability.

## Step-by-Step Workflow

1. **Curate a challenging problem set.** Collect problems at the difficulty frontier of your SFT model — problems it sometimes solves and sometimes fails. Filter out trivially easy (always correct) and impossibly hard (always wrong) problems, as these provide no gradient signal.

2. **Generate multiple candidate solutions per problem.** Run inference with your SFT model using diverse sampling (temperature=0.6, top_k=40, top_p=0.95) to produce N candidates per problem (N=8 works well). Use chain-of-thought formatting with explicit `<think>` reasoning and `<answer>` delimiters.

3. **Label candidates with binary rewards.** Use a reliable external verifier — either a stronger LLM, a symbolic math checker, or unit tests for code — to assign R=+1 (correct) or R=-1 (incorrect) to each candidate. For math, verify the final boxed answer against ground truth.

4. **Filter for reward variance.** Remove all candidates from any problem where all N samples received the same reward. Keep only problems that have at least one correct and one incorrect candidate, ensuring the model sees contrastive signal.

5. **Implement the length-normalized reward-weighted loss.** Code the training objective:
   ```python
   # For each (prompt, response, reward) triple:
   log_probs = model.forward(prompt + response)  # per-token log probs
   avg_log_prob = log_probs.sum() / len(response_tokens)  # geometric mean
   loss = -reward * torch.exp(avg_log_prob)
   ```
   The length normalization (`/ len(response_tokens)`) is essential to avoid penalizing longer reasoning chains.

6. **Configure training hyperparameters.** Use AdamW with beta1=0.9, beta2=0.95, weight_decay=0.1. Start with a low learning rate (1e-6) decaying to 1e-7 via cosine schedule. Use a global batch size of 128. Train for approximately 800 steps — monitor loss convergence rather than fixing epochs.

7. **Do NOT apply importance sampling correction.** Even though the training data was generated by the SFT policy (not the current RL policy), skip importance weighting. The paper empirically shows it hurts rather than helps in LLM RL settings.

8. **Evaluate with pass@1 on held-out benchmarks.** Use greedy or low-temperature sampling on evaluation sets. Track both accuracy and average response length — expect RL to increase response length (the model learns to reason more extensively).

9. **Iterate by regenerating data if needed.** If a single offline RL pass plateaus, you can regenerate the candidate dataset using the improved RL model and repeat — but each round is a clean, decoupled cycle, not a continuous online loop.

## Concrete Examples

**Example 1: Building an Offline RL Training Script for Math Reasoning**

User: "I have a Qwen2.5-7B model fine-tuned on math data. I want to add an RL stage to improve its AIME-level performance. How should I set this up?"

Approach:
1. Identify a set of competition math problems at the model's difficulty frontier (~1000-6000 problems)
2. Generate 8 candidate chain-of-thought solutions per problem with temperature=0.6
3. Verify each solution's final answer against ground truth using a math checker
4. Filter to keep only problems with mixed correct/incorrect candidates
5. Implement the offline RL training loop with length-normalized reward-weighted loss

Output — data collection script:
```python
import json
from vllm import LLM, SamplingParams

# Step 1: Load SFT model and problems
model = LLM("path/to/sft-model", tensor_parallel_size=4)
problems = json.load(open("math_problems.json"))

# Step 2: Generate candidates with diverse sampling
sampling = SamplingParams(
    temperature=0.6, top_k=40, top_p=0.95,
    max_tokens=32768, n=8  # 8 candidates per problem
)

SYSTEM = (
    "You are a helpful assistant. Think step-by-step inside <think></think> tags, "
    "then give your final answer inside <answer></answer> tags using \\boxed{}."
)

results = []
for p in problems:
    prompt = f"<|im_start|>system\n{SYSTEM}<|im_end|>\n<|im_start|>user\n{p['question']}<|im_end|>\n<|im_start|>assistant\n"
    outputs = model.generate([prompt], sampling)[0]
    for out in outputs.outputs:
        extracted = extract_boxed_answer(out.text)
        reward = 1 if extracted == p["answer"] else -1
        results.append({
            "prompt": prompt, "response": out.text,
            "reward": reward, "problem_id": p["id"]
        })

# Step 3: Filter for reward variance
from collections import defaultdict
by_problem = defaultdict(list)
for r in results:
    by_problem[r["problem_id"]].append(r)

filtered = []
for pid, samples in by_problem.items():
    rewards = {s["reward"] for s in samples}
    if len(rewards) > 1:  # has both correct and incorrect
        filtered.extend(samples)

json.dump(filtered, open("offline_rl_data.json", "w"))
print(f"Kept {len(filtered)} samples from {len(by_problem)} problems")
```

Output — training loop:
```python
import torch
from torch.utils.data import DataLoader
from transformers import AutoModelForCausalLM, AutoTokenizer

model = AutoModelForCausalLM.from_pretrained("path/to/sft-model", torch_dtype=torch.float16)
tokenizer = AutoTokenizer.from_pretrained("path/to/sft-model")
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-6, betas=(0.9, 0.95), weight_decay=0.1)

dataset = load_offline_rl_data("offline_rl_data.json")
loader = DataLoader(dataset, batch_size=128, shuffle=True)

for step, batch in enumerate(loader):
    input_ids = batch["input_ids"]       # prompt + response tokens
    labels = batch["labels"]             # -100 for prompt, token ids for response
    rewards = batch["rewards"]           # +1 or -1

    outputs = model(input_ids=input_ids, labels=labels)
    per_token_logprobs = -outputs.loss  # need per-token, not averaged

    # Compute length-normalized log probability per sample
    response_mask = (labels != -100).float()
    response_lengths = response_mask.sum(dim=1)
    avg_logprob = (per_token_logprobs * response_mask).sum(dim=1) / response_lengths

    # Reward-weighted loss (no importance sampling)
    policy_value = torch.exp(avg_logprob)
    loss = -(rewards * policy_value).mean()

    loss.backward()
    optimizer.step()
    optimizer.zero_grad()

    if step >= 800:
        break
```

**Example 2: Designing a Verification-Based Reward System**

User: "I'm training a code reasoning model. How do I set up the binary reward labeling for offline RL?"

Approach:
1. Replace math answer checking with unit test execution
2. Design the reward assignment to handle partial correctness
3. Ensure the verifier is reliable to avoid reward noise

Output:
```python
import subprocess
import tempfile

def verify_code_solution(problem: dict, generated_code: str) -> int:
    """Binary reward via test execution. Returns +1 or -1."""
    with tempfile.NamedTemporaryFile(mode='w', suffix='.py', delete=False) as f:
        # Write solution + test cases
        f.write(generated_code + "\n\n")
        f.write(problem["test_code"])
        f.flush()

        try:
            result = subprocess.run(
                ["python", f.name], capture_output=True, timeout=30
            )
            return 1 if result.returncode == 0 else -1
        except subprocess.TimeoutExpired:
            return -1

# For LLM-as-judge verification (when tests aren't available):
def verify_with_llm_judge(question: str, candidate: str, reference: str) -> int:
    """Use a stronger model as binary correctness judge."""
    judge_prompt = f"""Does the following solution correctly answer the question?
Question: {question}
Candidate answer: {candidate}
Reference answer: {reference}
Reply with exactly YES or NO."""

    response = call_judge_model(judge_prompt)  # e.g., GPT-4, Qwen3-32B
    return 1 if "YES" in response.upper() else -1
```

**Example 3: Diagnosing and Fixing Unstable Online RL by Switching to Offline**

User: "My GRPO training keeps diverging after ~200 steps. Loss spikes and reward collapses. What should I do?"

Approach:
1. Diagnose the root cause: online RL's feedback loop creates distribution shift
2. Propose switching to offline RL as a structural fix
3. Provide migration steps

Output:
```
The instability is likely caused by the online RL feedback loop:
- As the policy updates, it generates different rollouts
- These rollouts shift the reward distribution
- The shifted rewards cause overcorrection, leading to collapse

Migration to offline RL:

1. Take your current best checkpoint (before divergence)
2. Run a single large-scale inference pass to collect rollouts:
   - Use the same training prompts
   - Generate 8 samples per prompt with temperature=0.6
3. Label all rollouts with your existing reward function
4. Filter out prompts with no reward variance
5. Train on this FIXED dataset with the reward-weighted loss
6. Benefits you'll see immediately:
   - No more loss spikes (dataset doesn't change during training)
   - Reproducible runs (same data = same training dynamics)
   - Faster iteration (change hyperparams without re-collecting data)
```

## Best Practices

- **Do** use length normalization in the policy value computation. Without it, the model learns to produce shorter outputs regardless of correctness, destroying reasoning quality.
- **Do** filter out problems with zero reward variance (all-correct or all-incorrect). These contribute no learning signal and add noise.
- **Do** keep the positive/negative sample ratio roughly balanced (~48%/52% in the paper). Heavy imbalance biases training toward either always accepting or always rejecting.
- **Do** use a very low learning rate (1e-6 range) for the RL stage. The SFT model is already competent; RL is a refinement, not a retraining.
- **Avoid** importance sampling corrections. Despite the theoretical motivation (data was generated by a different policy), empirical evidence shows it hurts LLM RL training.
- **Avoid** training for too many steps on the fixed dataset. Overfitting to the offline data degrades generalization. Monitor validation accuracy and stop early if it plateaus.

## Error Handling

- **All candidates correct or incorrect for a problem:** Skip the problem entirely — it's either too easy or too hard for your current model. This is expected for ~30-50% of problems.
- **Reward noise from unreliable verification:** If your verifier has >5% error rate, the RL signal becomes noisy. Use a stronger verifier model, or require consensus from multiple verification methods before assigning rewards.
- **Loss doesn't decrease:** Check that rewards are correctly signed (+1/-1, not 0/1). Verify that the length normalization divides by response length, not total sequence length. Confirm the learning rate isn't too low to produce any gradient.
- **Model generates shorter responses after RL:** Length normalization may be missing or incorrectly applied. Also check that negative rewards aren't dominating the batch — the model learns to minimize output to minimize loss on negatives.
- **Out-of-memory during data collection:** Generate candidates in batches rather than all at once. The offline paradigm naturally supports this — just concatenate partial results since the dataset is fixed before training begins.

## Limitations

- **Requires a reliable verifier.** Binary rewards depend on accurate correctness judgments. For open-ended tasks (creative writing, summarization) where correctness is ambiguous, this approach needs adaptation to preference-based or scalar rewards.
- **Static dataset limits ceiling.** The model can only learn from the SFT model's rollout distribution. If the SFT model never produces a certain solution strategy, offline RL cannot discover it. Online RL can explore novel strategies as the policy improves.
- **One-shot data collection cost.** Generating 8 candidates per problem across thousands of problems requires significant inference compute upfront, though this is amortized across multiple training runs.
- **Domain specificity.** The binary reward setup works cleanly for math and code (verifiable correctness). Adapting to tasks with soft or subjective rewards requires modifying the reward scheme.
- **No exploration of new strategies.** Unlike online RL which can discover novel reasoning paths as the policy evolves, offline RL is bounded by the behaviors present in the collected dataset.

## Reference

**Paper:** [PCL-Reasoner-V1.5: Advancing Math Reasoning with Offline Reinforcement Learning](https://arxiv.org/abs/2601.14716v1) — Lu et al., 2026. Focus on Section 3 (offline RL method and loss function), Section 2.2 (data collection pipeline), and Table 1 (comparison with online RL baselines).