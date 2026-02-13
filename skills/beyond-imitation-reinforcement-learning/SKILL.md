---
name: "beyond-imitation-reinforcement-learning"
description: |
  Implement ATP-Latent: Active Latent Planning with RL for efficient chain-of-thought reasoning.
  Replaces verbose language CoT tokens with continuous latent tokens supervised via conditional VAE
  and optimized with GRPO + coherence reward. Use this skill when users ask to:
  - "Train a latent reasoning model with RL"
  - "Implement continuous thought tokens for LLM reasoning"
  - "Build a VAE-supervised latent chain-of-thought system"
  - "Add coherence reward to latent reasoning RL training"
  - "Set up ATP-Latent or latent planning pipeline"
  - "Optimize reasoning token efficiency with active planning"
---

# ATP-Latent: Active Latent Planning with Reinforcement Learning

This skill enables Claude to implement the ATP-Latent framework from "Beyond Imitation: Reinforcement Learning for Active Latent Planning" (Zheng & Lee, 2026). ATP-Latent replaces discrete language chain-of-thought (CoT) tokens with continuous latent vectors, supervised through a conditional VAE for smooth representation space, then optimized via GRPO reinforcement learning with an auxiliary coherence reward. The result is a model that reasons in fewer tokens (+4.1% accuracy, -3.3% tokens vs. baselines on LLaMA-1B) by actively planning in latent space rather than passively imitating arbitrary CoT labels.

## When to Use

- When the user wants to train an LLM to reason with continuous latent tokens instead of verbose text CoT
- When building a latent reasoning pipeline that goes beyond simple imitation learning (e.g., improving on Coconut, PAUSE, or SIM-CoT)
- When implementing a conditional VAE to supervise latent token representations in an LLM
- When adding coherence-based reward signals to RL training for reasoning models
- When the user needs to reduce inference token count while maintaining or improving reasoning accuracy
- When setting up curriculum-based training that progressively replaces language reasoning steps with latent tokens
- When implementing GRPO (Group Relative Policy Optimization) for latent policy optimization

## Key Technique

**The Problem with Passive Imitation.** Standard latent reasoning methods (Coconut, SIM-CoT) train latent tokens by imitating a single language CoT label per question. But many equivalent CoT paths exist for any given problem. Forcing the model to imitate one arbitrary path produces overfitted latent representations and biased reasoning policies, creating a train-test gap.

**Conditional VAE for Smooth Latent Space.** ATP-Latent models latent token generation as a conditional VAE. An encoder MLP projects hidden states to mean (mu) and log-variance (logvar), then samples via the reparameterization trick: `z = mu + std * eps`. A separate decoder LLM reconstructs language tokens from latent vectors. The KL divergence term `D_KL(N(mu, sigma^2) || N(0, I))` is weighted with a small beta (1e-3/d) to prioritize reasoning capacity over reconstruction fidelity. The total SFT loss combines encoder loss, decoder reconstruction loss, stop-head loss, and KL regularization: `L_SFT = L_Enc + L_Dec + L_Stop + beta * L_KL`.

**RL with Coherence Reward.** After SFT, ATP-Latent runs GRPO to actively optimize the latent reasoning policy. The key innovation is an auxiliary coherence reward `R_Coh` that decodes each latent token back to language via the VAE decoder, then checks whether equation right-hand sides from step t appear in subsequent steps' left-hand sides or the final answer. The combined reward is `f(L, A) = (1 + 1 * R_Coh(L)) * R_Correct(A) + 5 * R_Format(A)`, guiding RL toward logically consistent latent reasoning chains without requiring ground-truth CoT supervision.

## Step-by-Step Workflow

1. **Prepare the base model and tokenizer.** Load a causal LM (e.g., LLaMA-1B-Instruct via `AutoModelForCausalLM`). Add three special tokens to the tokenizer: `<|start-latent|>`, `<|end-latent|>`, and `<|latent|>`. Resize the model's token embeddings to accommodate them.

2. **Implement the LatentPolicy head.** Create an MLP that takes hidden states from the base LM and projects them to `mu` and `logvar` vectors of the same hidden dimension. During training, sample latent tokens via reparameterization: `z = mu + exp(0.5 * logvar) * epsilon`. During inference, use `mu` directly (deterministic mode).

3. **Implement the STOPPolicy head.** Build a binary classifier (2-layer MLP with GELU) on top of each latent token's hidden state. This predicts whether to continue generating latent tokens or stop, ensuring uniform information density across the latent reasoning chain.

4. **Build the VAE decoder.** Initialize a separate (smaller) LLM as the decoder. It takes latent token embeddings as input and reconstructs the original language CoT text via cross-entropy loss. This decoder is used during SFT training and for computing coherence reward during RL -- it is not needed at inference time.

5. **Run curriculum-based SFT training.** Train across K stages (typically K=10, 2 epochs per stage). In stage k, remove the first k language CoT steps and replace them with `c * k` latent tokens (where c=2 tokens per step). Optimize the combined loss: `L_SFT = L_Enc + L_Dec + L_Stop + beta * L_KL` with beta=0.001, lr=1e-4, batch_size=32.

6. **Implement the coherence reward function.** For each sampled latent reasoning trajectory during RL: (a) decode all latent tokens back to language using the VAE decoder, (b) extract equations from the decoded text via regex, (c) for each equation, check if its RHS appears in a subsequent step's LHS or the final answer, (d) compute `R_Coh = matching_count / total_latent_tokens`.

7. **Implement GRPO training.** For each question, sample a group of G=8 latent reasoning trajectories. Compute per-trajectory rewards using: `f(L, A) = (1 + R_Coh(L)) * R_Correct(A) + 0.5 * R_Format(A)`. Normalize advantages within the group. Compute policy ratios from the Gaussian latent distribution: `pi(l_t | q, l_{1:t-1}) proportional to exp(-0.5/sigma^2 * ||l_hat_t - l_t||^2)`. Apply clipped surrogate loss with epsilon=0.2.

8. **Configure RL hyperparameters.** Use lr=1e-6, batch_size=16, latent_sigma=0.001, reward weights: accuracy=1.0, format=0.5, reasoning=0.1. Initialize from the best SFT checkpoint (typically checkpoint_15). Train for 1 epoch with 3 stages.

9. **Evaluate on benchmarks.** Test on GSM8K, MultiArith, SVAMP, and GSM-hard. Measure both accuracy and average latent tokens used. Expected results on LLaMA-1B: ~42.3% GSM8K, ~94.4% MultiArith, ~44.2% SVAMP with ~8.4 average tokens.

10. **Deploy for inference.** At inference time, strip the VAE decoder entirely. The model generates latent tokens autoregressively using the LatentPolicy head (deterministic mu), stops when the STOPPolicy predicts termination, then generates the answer in language tokens. No CoT text is produced -- only the final answer.

## Concrete Examples

**Example 1: Setting up the VAE-supervised latent token pipeline**

User: "I want to implement the ATP-Latent VAE supervision for latent reasoning tokens on my LLaMA model."

Approach:
1. Define the LatentPolicy and STOPPolicy modules in a `projector.py`:
```python
import torch
import torch.nn as nn

class LatentPolicy(nn.Module):
    def __init__(self, feature_size, intermediate_size, deterministic=False):
        super().__init__()
        self.deterministic = deterministic
        self.fc1 = nn.Linear(feature_size, intermediate_size)
        self.act = nn.GELU()
        self.mean = nn.Linear(intermediate_size, feature_size)
        self.log_var = nn.Linear(intermediate_size, feature_size)

    def forward(self, x):
        h = self.act(self.fc1(x))
        mu = self.mean(h)
        logvar = self.log_var(h)
        if self.deterministic:
            return mu, mu, logvar
        std = torch.exp(0.5 * logvar)
        eps = torch.randn_like(std)
        z = mu + std * eps
        return z, mu, logvar

class STOPPolicy(nn.Module):
    def __init__(self, feature_size, intermediate_size):
        super().__init__()
        self.fc1 = nn.Linear(feature_size, intermediate_size)
        self.act = nn.GELU()
        self.fc2 = nn.Linear(intermediate_size, 2)  # continue vs stop

    def forward(self, x):
        return self.fc2(self.act(self.fc1(x)))
```

2. Wrap the base LM with VAE-style forward pass:
```python
# During SFT forward pass for each latent position t:
hidden = base_model.get_hidden_state(input_ids, past_kv)
z, mu, logvar = latent_head(hidden[:, -1, :])
stop_logits = stop_head(hidden[:, -1, :])

# KL loss
kl = 0.5 * (mu**2 + logvar.exp() - 1 - logvar)
kl_loss = kl.mean()

# Feed z back as next input embedding
next_input = z.unsqueeze(1)  # [batch, 1, hidden_dim]
```

3. Train the decoder on reconstructing language CoT from latent tokens.

Output: A model that produces smooth, VAE-regularized latent representations instead of point estimates, enabling better RL optimization in the next stage.

---

**Example 2: Implementing the coherence reward for RL**

User: "How do I compute the coherence reward for ATP-Latent's RL training?"

Approach:
1. Decode each latent token trajectory back to language using the frozen VAE decoder
2. Extract mathematical equations from decoded text using regex
3. Check logical consistency across steps

```python
import re

def compute_coherence_reward(decoded_steps, final_answer):
    """
    decoded_steps: list of strings, each decoded from a latent token
    final_answer: string of the model's final answer
    """
    equations = []
    for step in decoded_steps:
        # Extract equations like "x = 5 + 3" -> lhs="x", rhs="5 + 3"
        matches = re.findall(r'([\w\s]+?)\s*=\s*(.+?)(?:\n|$)', step)
        for lhs, rhs in matches:
            equations.append((lhs.strip(), rhs.strip()))

    if not equations:
        return 0.0

    coherent_count = 0
    for i, (lhs_i, rhs_i) in enumerate(equations):
        # Check if RHS appears in any subsequent LHS or final answer
        for j in range(i + 1, len(equations)):
            if rhs_i in equations[j][0]:
                coherent_count += 1
                break
        else:
            if rhs_i in final_answer:
                coherent_count += 1

    return coherent_count / len(decoded_steps)
```

4. Combine with accuracy and format rewards:
```python
def combined_reward(latent_tokens, answer, ground_truth, decoded_steps):
    r_correct = 1.0 if extract_number(answer) == extract_number(ground_truth) else 0.0
    r_format = 1.0 if "\\boxed{" in answer else 0.0
    r_coh = compute_coherence_reward(decoded_steps, answer)
    return (1 + r_coh) * r_correct + 0.5 * r_format
```

Output: A reward signal that is non-zero only when the answer is correct, but amplified when the latent reasoning chain is internally coherent -- guiding GRPO toward logically structured latent plans.

---

**Example 3: Full training pipeline configuration**

User: "Set up the complete ATP-Latent training pipeline from SFT to RL."

Approach:
1. Create the SFT config (`ATP-Latent-SFT.yaml`):
```yaml
model_id: "meta-llama/Llama-3.2-1B-Instruct"
mode: coconutgpt_same_word_embedding_len_vae
training_method: full
lr: 1e-4
batch_size: 32
epochs: 25
epochs_per_stage: 2
max_n_latents: 10
c_thought: 2
kl_factor: 0.001
explain_mode: v1_aug
coconut: true
```

2. Run SFT with curriculum learning:
```bash
torchrun --nproc_per_node=4 run.py args/ATP-Latent-SFT.yaml
```

3. Create the RL config (`ATP-Latent-RL.yaml`), pointing to best SFT checkpoint:
```yaml
model_id: "meta-llama/Llama-3.2-1B-Instruct"
checkpoint: "results/sft_run/checkpoint_15"
mode: coconut_rl_end_vae
training_method: only_base_causallm
lr: 1e-6
batch_size: 16
clip_eps: 0.2
group_size: 8
reward_accuracy: 1.0
reward_format: 0.5
reward_reasoning: 0.1
latent_sigma: 0.001
max_n_latents: 10
```

4. Run RL:
```bash
torchrun --nproc_per_node=4 run.py args/ATP-Latent-RL.yaml
```

Output: A two-stage pipeline producing a model that reasons in ~8.4 latent tokens on average, achieving 47.7% average accuracy across GSM8K, MultiArith, SVAMP, and GSM-hard on LLaMA-1B.

## Best Practices

- **Do:** Use a small KL weight (beta=0.001 or 1e-3/d). The latent space should be smooth enough for RL exploration but not so regularized that it collapses to uninformative representations.
- **Do:** Initialize RL from a well-converged SFT checkpoint (around epoch 15 out of 25). Starting RL too early leads to unstable training; too late leads to overfitted representations that resist RL optimization.
- **Do:** Freeze the VAE decoder during RL training. The decoder is only used to compute coherence reward -- backpropagating through it would corrupt the reward signal.
- **Do:** Use curriculum learning for SFT with 2 epochs per stage, progressively replacing more language CoT steps with latent tokens. Jumping directly to full latent replacement fails.
- **Avoid:** Using large group sizes (G>16) in GRPO. G=8 provides sufficient variance for advantage estimation without excessive compute. The latent Gaussian policy already provides smooth gradients.
- **Avoid:** Setting the coherence reward weight too high. The reward structure `(1 + R_Coh) * R_Correct` ensures coherence only matters when the answer is correct -- don't add an independent coherence term that rewards coherent but wrong reasoning.

## Error Handling

- **KL collapse (all latent tokens map to the same point):** Reduce beta or use KL annealing -- start with beta=0, linearly increase to target over first 3 curriculum stages.
- **Stop head fires too early (model generates 0-1 latent tokens):** Increase the stop-head loss weight or pad latent sequences to a fixed length during early SFT stages (`pad_latent_to_max: true`).
- **RL reward is always 0 (model never gets correct answers):** The SFT checkpoint may be too weak. Verify SFT accuracy is at least 25% on the validation set before starting RL. If not, extend SFT training or adjust curriculum pacing.
- **Coherence reward is noisy (decoded text is garbled):** The VAE decoder may be undertrained. Ensure decoder reconstruction loss converges during SFT. Use `explain_mode: v1_aug` for augmented supervision.
- **GRPO gradients explode:** Clip gradients and ensure `latent_sigma` is not too small. With sigma=0.001, the policy ratio can become extreme for latent vectors far from the mean. Consider sigma=0.01 for early RL stages.

## Limitations

- **Scale:** Validated only on LLaMA-1B. Scaling to 7B+ models requires significant GPU resources and hyperparameter re-tuning -- the KL factor and curriculum pacing may not transfer directly.
- **Domain:** Tested primarily on math reasoning (GSM8K, MultiArith, SVAMP). The coherence reward relies on equation-structure regex and does not generalize to free-form reasoning tasks (e.g., commonsense QA, code generation) without a domain-specific reward function.
- **Decoder overhead:** The VAE decoder adds training-time compute (roughly 30-50% overhead during SFT). It is stripped at inference, but RL training requires it for coherence reward computation.
- **Interpretability trade-off:** Latent tokens are not human-readable. Debugging requires decoding via the VAE decoder, which only approximates the original reasoning. The decoded text may not perfectly reflect what the model "thinks."
- **Coherence reward is math-specific.** The RHS-in-LHS consistency check assumes step-by-step algebraic reasoning. For other domains, you must design a domain-appropriate coherence metric.

## Reference

**Paper:** "Beyond Imitation: Reinforcement Learning for Active Latent Planning" (Zheng & Lee, 2026) -- [arXiv:2601.21598](https://arxiv.org/abs/2601.21598v1)
Look for: Section 3 (CVAE formulation and KL weighting), Section 4 (coherence reward definition and GRPO adaptation), Table 1 (main results), and Appendix A (curriculum schedule details).

**Code:** [github.com/zz1358m/ATP-Latent-master](https://github.com/zz1358m/ATP-Latent-master) -- Key files: `methods/sim_cot_rl_vae_end.py` (RL+VAE model), `methods/sim_cot_vae_end.py` (SFT+VAE model), `modules/projector.py` (LatentPolicy/STOPPolicy), `modules/grpo.py` (GRPO loss).