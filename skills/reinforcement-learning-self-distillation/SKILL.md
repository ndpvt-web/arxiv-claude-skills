---
name: "reinforcement-learning-self-distillation"
description: "Implement Self-Distillation Policy Optimization (SDPO) for RL training loops that convert rich textual feedback into dense token-level learning signals without external reward models. Use when: 'implement SDPO training loop', 'self-distillation for RL', 'use error feedback in RL training', 'dense credit assignment from textual feedback', 'test-time self-distillation', 'RLVR with rich feedback'."
---

# Self-Distillation Policy Optimization (SDPO)

This skill enables Claude to implement Self-Distillation Policy Optimization, a reinforcement learning technique that converts rich textual feedback (runtime errors, test failures, judge evaluations) into dense token-level learning signals. Instead of relying on scalar pass/fail rewards like standard RLVR, SDPO treats the current model conditioned on feedback as a self-teacher and distills its corrected next-token predictions back into the base policy. This yields dramatically better sample efficiency (4-10x fewer generations) and final accuracy across code generation, scientific reasoning, and tool use tasks.

## When to Use

- When building an RL training loop for code generation where compiler/runtime errors provide textual feedback beyond pass/fail
- When implementing RLVR and you want to exploit rich environment signals (error messages, failing test cases, judge explanations)
- When you need dense per-token credit assignment instead of sparse per-sequence rewards
- When implementing test-time adaptation where a model refines itself on hard problems using its own feedback
- When training on tasks where some rollouts succeed and others fail on the same question, and you want failed attempts to learn from successful ones
- When standard GRPO or PPO shows slow convergence due to sparse reward signals

## Key Technique

**The Credit Assignment Problem in RLVR:** Standard methods like GRPO assign a single scalar advantage to every token in a generated sequence. If a 500-token code solution fails one test case due to an off-by-one error at token 312, every token gets the same negative signal. This is wasteful -- most tokens were correct.

**SDPO's Self-Teacher Mechanism:** SDPO splits the current model into two roles. The *student* `pi_theta(.|x)` generates attempts normally. The *self-teacher* `pi_theta(.|x, f, y)` is the same model but conditioned on the feedback `f` (e.g., "IndexError at line 12: list index out of range") and the original attempt `y`. Because LLMs can identify mistakes when shown error context, the teacher assigns higher probability to corrected tokens at the exact positions where errors occurred. The training loss minimizes the KL divergence between student and teacher distributions at every token position, creating `|y| * K` unique advantages per sequence (where K is the vocabulary approximation size) instead of one flat scalar.

**Why It Works Without External Models:** The key insight is that in-context learning is already a form of self-correction. When you append "Your code failed with: IndexError at line 12" to the context, the model's next-token probabilities shift to avoid that exact mistake. SDPO distills this shift into the weights via `L_SDPO = sum_t KL(pi_theta(.|x, y_<t) || stopgrad(pi_theta(.|x, f, y_<t)))`. The `stopgrad` prevents the teacher from collapsing toward the student. In environments with only binary rewards, successful rollouts from the same batch serve as implicit feedback for failed attempts.

## Step-by-Step Workflow

### Training Loop Implementation

1. **Sample rollouts from the student policy.** For each prompt `x` in the batch, generate `G` candidate responses `{y_1, ..., y_G}` using the current policy `pi_theta(.|x)`. Use temperature sampling (typically T=1.0) to ensure diversity.

2. **Collect environment feedback for each rollout.** Execute each response against the verifiable environment (run code against test cases, check math proofs, query a judge). Capture both the binary reward `r` AND the full textual feedback `f` (error messages, stack traces, failing test case outputs, judge rationale).

3. **Construct self-teacher inputs for failed rollouts.** For each failed response `y_i` with feedback `f_i`, build the teacher context as `[x, f_i, y_i]`. For environments with only scalar rewards, use a successful rollout `y_j` (where `r_j = 1`) from the same prompt as implicit feedback: teacher context becomes `[x, y_j, y_i]`.

4. **Compute teacher log-probabilities without gradient.** Run a forward pass through the same model with the feedback-augmented context under `stopgrad`. Extract top-K logits at each token position of the original response. Use K=10-50 depending on memory budget. This step requires no additional sampling -- just recomputing log-probs of the existing tokens.

5. **Compute student log-probabilities with gradient.** Run a standard forward pass for `pi_theta(.|x)` on the same responses, retaining the computation graph for backpropagation. Extract the corresponding top-K logits at matching positions.

6. **Calculate the SDPO loss.** For each token position `t`, compute the symmetric Jensen-Shannon divergence (preferred over raw KL for stability) between student and teacher top-K distributions. Sum across all positions and average across the batch. Optionally blend with standard GRPO loss for successful rollouts.

7. **Apply EMA regularization to the teacher.** Interpolate teacher parameters with the initial (pre-training) parameters using a small coefficient (alpha=0.01): `theta_teacher = (1 - alpha) * theta + alpha * theta_init`. This prevents the teacher from drifting too far and losing its corrective signal.

8. **Update policy via gradient descent.** Backpropagate through the SDPO loss and update `theta`. Use standard optimizer settings (AdamW, lr ~1e-6 for 7B models). The gradient acts as a logit-level policy gradient where advantages are automatically derived from student-teacher probability ratios.

9. **Repeat with fresh rollouts.** Sample new responses from the updated policy and iterate. SDPO typically converges in 4x fewer generations than GRPO because each update carries denser information.

### Test-Time Self-Distillation

10. **For hard problems at inference, apply SDPO iteratively.** Generate an attempt, collect feedback, perform a few gradient steps of self-distillation on the single example, then generate again. This compresses the interaction history into weights rather than extending the context window, achieving the same discovery rate as best-of-64 sampling with ~20 attempts.

## Concrete Examples

**Example 1: SDPO Training Loop for Code Generation**

User: "Implement an SDPO training step for a code generation model that uses compiler error feedback."

Approach:
1. Define the dual-role forward pass (student without feedback, teacher with feedback)
2. Implement top-K logit extraction and JSD loss
3. Wire up the training step with stopgrad on the teacher

```python
import torch
import torch.nn.functional as F

def sdpo_training_step(model, tokenizer, prompts, env, optimizer,
                       top_k=20, alpha=0.01, initial_params=None, G=8):
    """One SDPO training step with rich feedback from code execution."""
    model.train()
    total_loss = 0.0

    for prompt in prompts:
        # Step 1: Sample G rollouts from student
        input_ids = tokenizer.encode(prompt, return_tensors="pt").to(model.device)
        rollouts = []
        for _ in range(G):
            output = model.generate(input_ids, max_new_tokens=512,
                                     do_sample=True, temperature=1.0)
            rollouts.append(output[0, input_ids.shape[1]:])

        # Step 2: Execute and collect feedback
        results = []
        for rollout in rollouts:
            code = tokenizer.decode(rollout, skip_special_tokens=True)
            reward, feedback = env.execute(code)  # feedback = error msg or "all tests passed"
            results.append((rollout, reward, feedback))

        # Identify successful and failed rollouts
        successes = [(r, fb) for r, rw, fb in results if rw == 1]
        failures = [(r, fb) for r, rw, fb in results if rw == 0]

        if not failures:
            continue  # Nothing to distill

        for failed_tokens, error_feedback in failures:
            # Step 3: Build teacher context
            if error_feedback and error_feedback.strip():
                # Rich feedback available: use error message
                teacher_prefix = f"{prompt}\n[Feedback]: {error_feedback}\n[Original attempt]:"
            elif successes:
                # Scalar-only: use successful rollout as implicit feedback
                success_code = tokenizer.decode(successes[0][0], skip_special_tokens=True)
                teacher_prefix = f"{prompt}\n[Reference solution]:\n{success_code}\n[Original attempt]:"
            else:
                continue

            teacher_ids = tokenizer.encode(teacher_prefix, return_tensors="pt").to(model.device)
            student_ids = input_ids

            # Step 4: Teacher forward pass (no gradient)
            with torch.no_grad():
                teacher_context = torch.cat([teacher_ids, failed_tokens.unsqueeze(0)], dim=1)
                teacher_logits = model(teacher_context).logits
                # Extract logits at response token positions
                offset = teacher_ids.shape[1]
                teacher_response_logits = teacher_logits[:, offset-1:-1, :]  # shifted
                teacher_topk = torch.topk(teacher_response_logits, top_k, dim=-1)

            # Step 5: Student forward pass (with gradient)
            student_context = torch.cat([student_ids, failed_tokens.unsqueeze(0)], dim=1)
            student_logits = model(student_context).logits
            s_offset = student_ids.shape[1]
            student_response_logits = student_logits[:, s_offset-1:-1, :]

            # Step 6: Compute JSD loss over top-K tokens
            # Gather student logits at teacher's top-K indices
            student_at_topk = torch.gather(student_response_logits, -1, teacher_topk.indices)

            teacher_probs = F.softmax(teacher_topk.values, dim=-1)
            student_probs = F.softmax(student_at_topk, dim=-1)

            # Symmetric Jensen-Shannon Divergence
            m = 0.5 * (teacher_probs + student_probs)
            jsd = 0.5 * (F.kl_div(m.log(), teacher_probs, reduction='batchmean') +
                         F.kl_div(m.log(), student_probs, reduction='batchmean'))
            total_loss += jsd

    # Step 7: EMA regularization toward initial params
    if initial_params is not None:
        with torch.no_grad():
            for p, p_init in zip(model.parameters(), initial_params):
                p.data.mul_(1 - alpha).add_(p_init.data, alpha=alpha)

    # Step 8: Gradient update
    optimizer.zero_grad()
    total_loss.backward()
    optimizer.step()

    return total_loss.item()
```

**Example 2: Test-Time Self-Distillation for Hard Problems**

User: "I have a model that can't solve a hard competitive programming problem in one shot. Implement test-time SDPO to iteratively refine it."

Approach:
1. Generate attempt, execute, collect feedback
2. Perform a few gradient steps of self-distillation
3. Generate again from the updated model
4. Repeat until solved or budget exhausted

```python
def test_time_sdpo(model, tokenizer, problem, env, max_attempts=20,
                   sdpo_steps=3, lr=1e-5, top_k=10):
    """Test-time self-distillation: refine model on a single hard problem."""
    optimizer = torch.optim.AdamW(model.parameters(), lr=lr)
    initial_params = [p.clone() for p in model.parameters()]

    for attempt in range(max_attempts):
        # Generate attempt
        input_ids = tokenizer.encode(problem, return_tensors="pt").to(model.device)
        output = model.generate(input_ids, max_new_tokens=1024,
                                 do_sample=True, temperature=0.8)
        response_tokens = output[0, input_ids.shape[1]:]
        code = tokenizer.decode(response_tokens, skip_special_tokens=True)

        reward, feedback = env.execute(code)
        if reward == 1:
            print(f"Solved in {attempt + 1} attempts")
            return code

        # Self-distill from the feedback for a few steps
        for _ in range(sdpo_steps):
            loss = sdpo_single_example(
                model, tokenizer, problem, response_tokens,
                feedback, optimizer, top_k, initial_params
            )

        print(f"Attempt {attempt+1}: {feedback[:80]}... | loss={loss:.4f}")

    return None  # Budget exhausted
```

**Example 3: Integrating SDPO Into an Existing GRPO Pipeline**

User: "I already have GRPO training working. How do I add SDPO on top of it?"

Approach:
1. Keep the GRPO loss for successful rollouts (they still get positive scalar reward)
2. Add the SDPO self-distillation loss for failed rollouts that have textual feedback
3. Blend with a mixing coefficient

```python
def combined_grpo_sdpo_loss(model, batch, grpo_weight=0.5, sdpo_weight=0.5):
    """Hybrid loss: GRPO for reward shaping + SDPO for dense credit assignment."""
    grpo_loss = compute_grpo_loss(model, batch)  # existing GRPO implementation

    # Only apply SDPO to failed rollouts with feedback
    failed_with_feedback = [
        (prompt, response, feedback)
        for prompt, response, reward, feedback in batch
        if reward == 0 and feedback is not None
    ]

    if failed_with_feedback:
        sdpo_loss = compute_sdpo_loss(model, failed_with_feedback)
        return grpo_weight * grpo_loss + sdpo_weight * sdpo_loss

    return grpo_loss
```

## Best Practices

- **Do:** Always apply `stopgrad` to the teacher's forward pass. Without it, the teacher collapses toward the student and ignores the feedback signal entirely.
- **Do:** Use symmetric Jensen-Shannon divergence instead of raw KL divergence. The paper found JSD significantly more stable during training.
- **Do:** Use EMA regularization toward initial parameters (alpha=0.01). Unregularized teachers cause training divergence.
- **Do:** Use successful rollouts as implicit feedback when the environment only returns scalar rewards. Pair each failed attempt with a passing solution from the same batch.
- **Avoid:** Applying SDPO to very small models (< 3B parameters). The self-teacher mechanism depends on emergent in-context learning ability, which weak models lack. The paper found SDPO underperforms GRPO on 1.5B models.
- **Avoid:** Using full vocabulary logits for the distillation loss. Top-K approximation (K=10-50) is essential for memory efficiency and performs comparably to full-vocabulary distillation.
- **Avoid:** Skipping the feedback quality check. Garbage or uninformative feedback (e.g., "Wrong answer" with no details) provides little self-teacher signal -- fall back to GRPO for those samples.

## Error Handling

- **Teacher-student divergence explosion:** If loss spikes, increase EMA alpha (pull teacher closer to initial params) or reduce learning rate. Switching from KL to JSD usually resolves this.
- **No successful rollouts in batch:** When all G rollouts fail for a prompt, you cannot construct implicit feedback from successes. Either skip that prompt or use the textual error feedback alone. Consider increasing G or lowering task difficulty.
- **Feedback is too long for context:** Truncate feedback to the most diagnostic portion (first error, first failing test case). The teacher only needs enough signal to identify the mistake location, not the full stack trace.
- **Memory pressure from dual forward passes:** The teacher pass is under `no_grad` and shares weights, so it adds ~50% memory overhead (activations only). Use gradient checkpointing on the student pass and reduce batch size if needed.
- **Degraded performance on easy tasks:** SDPO's overhead is unnecessary when the base model already achieves high pass rates. Monitor per-prompt pass rates and disable SDPO for prompts where >80% of rollouts succeed.

## Limitations

- Requires verifiable environments that can execute and evaluate generated outputs. Cannot be applied to open-ended generation tasks without a programmatic judge.
- The self-teacher quality is bounded by the model's in-context learning ability. If the model cannot interpret feedback to identify its mistakes, the distillation signal is noise.
- Models below ~3B parameters show diminished or negative returns from SDPO, as they lack sufficient in-context correction capability.
- Test-time SDPO modifies model weights, making it unsuitable for serving multiple users simultaneously without per-request model copies or adapter isolation.
- The technique assumes feedback is causally informative about the error. Generic rejection feedback ("incorrect") provides no more signal than scalar rewards.

## Reference

[Reinforcement Learning via Self-Distillation](https://arxiv.org/abs/2601.20802v1) -- Hubotter et al., 2026. Key sections: Section 2 for the SDPO loss derivation and Proposition 2.1 (gradient as logit-level policy gradient), Section 3 for the implicit feedback mechanism using successful rollouts, Section 5 for test-time self-distillation results showing 3x efficiency over best-of-k sampling.