---
name: "cord-bridging-audio-text-reasoning"
description: "Implement CORD (Cross-modal On-policy Distillation) to bridge modality gaps in multimodal AI systems. Applies weighted self-distillation where a strong modality (e.g., text) teaches a weaker modality (e.g., audio/image) within the same model using on-policy rollouts, importance-weighted token alignment, and judge-based sequence rewards. Triggers: 'bridge audio-text gap', 'cross-modal distillation', 'align audio reasoning with text', 'multimodal self-distillation', 'CORD training pipeline', 'fix modality degradation in LALM'"
---

# CORD: Bridging Modality Gaps via Weighted On-Policy Cross-Modal Distillation

This skill enables Claude to design and implement **cross-modal self-distillation training pipelines** based on the CORD framework. CORD addresses a critical problem in multimodal AI: when a language model is extended to process a new modality (audio, images, video), its reasoning ability through that modality degrades compared to text, even when the semantic content is identical. CORD fixes this by using the model's own text-conditioned reasoning as an internal teacher to align the weaker modality's outputs, combining token-level importance-weighted KL divergence with sequence-level judge-based GRPO rewards -- all on-policy, meaning supervision follows the model's actual inference states rather than a detached teacher's trajectories.

## When to Use

- When a user is building or fine-tuning a Large Audio Language Model (LALM) and observes that audio-conditioned responses are worse than text-conditioned responses for identical queries
- When implementing a training pipeline that aligns a weaker input modality (audio, image, video) with a stronger one (text) inside a single unified model
- When the user wants to apply knowledge distillation without an external teacher model -- using the same model's text pathway as the teacher
- When designing on-policy reinforcement learning for multimodal models, specifically combining token-level divergence minimization with sequence-level reward optimization
- When the user needs a data-efficient alignment method (CORD achieves results with only 80k synthetic paired samples)
- When debugging or reducing "modality gap" -- the measurable performance drop when a model processes audio vs. equivalent text

## Key Technique

**The core insight of CORD** is that standard training of audio language models creates a persistent acoustic-semantic gap: the model understands text well but struggles with audio even when the content is semantically identical. Previous approaches used off-policy supervision (training along the teacher's output trajectories), which causes distribution mismatch -- the model never learns to correct errors that occur along its own audio inference paths. CORD solves this by performing *on-policy* self-distillation: it generates audio-conditioned rollouts, then at each decoding step compares the audio distribution against the text distribution given the *same prefix the audio modality actually produced*. This means corrections target real error patterns rather than hypothetical ones.

**Multi-granularity alignment** operates at two levels simultaneously. At the **token level**, CORD applies reverse KL divergence (which focuses on high-probability regions of the text teacher) with dual importance weighting: (1) KL-based weighting that identifies the top-K tokens with the highest cross-modal divergence and upweights them by factor alpha, since analysis shows KL is extremely skewed with the 80th percentile at only 0.23 while a few critical tokens carry large divergence; (2) positional decay weighting that emphasizes early tokens more heavily, because early misalignments trigger cascading errors throughout the sequence. At the **sequence level**, a judge model evaluates whether the complete audio-generated answer semantically matches what text would produce, and this binary reward is optimized via Group Relative Policy Optimization (GRPO) across groups of N=4 sampled trajectories.

**Data efficiency** is a key advantage. CORD uses only 80k synthetic paired samples (text instructions converted to audio via TTS), yet the learned alignment generalizes far beyond the training domain -- math-only training data improves performance on general knowledge benchmarks like MMSU and OpenBookQA, indicating CORD learns a fundamental modality-bridging capability rather than task-specific patterns.

## Step-by-Step Workflow

1. **Prepare semantic-equivalent paired data.** For each training instance, create a (x^a, x^t) pair where x^a is the audio version and x^t is the text version of the same instruction. Use a high-quality TTS model (e.g., Kokoro) to synthesize audio from text prompts. Target 80k pairs minimum; choose prompts with step-by-step reasoning traces (math problems work well as a universal alignment signal).

2. **Configure the unified model architecture.** Start with a pretrained LALM (e.g., Qwen2-Audio-7B-Instruct or Step-Audio2-mini). The model must be able to accept both audio and text inputs and share the same LLM backbone for generation. No external teacher model is needed -- the text pathway of the same model serves as teacher.

3. **Implement the on-policy audio rollout generator.** For each training pair, feed x^a to the model and sample a complete response: `y ~ p_theta(.|x^a)` using temperature 1.0 for the token-level objective and 1.5 for GRPO sequence sampling. Store the full token sequence and per-step logits.

4. **Compute token-level reverse KL divergence at each step.** For each position t in the rollout, condition both modalities on the *same* prefix y_{<t} (the one actually generated by audio) and compute: `D_t = KL(p_theta(.|y_{<t}, x^a) || p_theta(.|y_{<t}, x^t))`. This is the reverse KL, which mode-seeks toward the text distribution's high-probability regions.

5. **Apply importance-aware dual weighting.** Compute two weight factors per token: (a) **KL importance weight** `w_t^KL`: rank all tokens by D_t, assign weight alpha=2 to the top-K=20 tokens, weight 1 to the rest; (b) **Positional decay weight** `w_t^pos = beta - (beta-1)*(t-1)/(T-1)` with beta=2, linearly decaying from 2.0 at t=1 to 1.0 at t=T. Multiply them: `w_t = w_t^KL * w_t^pos`. Compute the token-level loss: `L_tok = E_y[sum(w_t * D_t)]`.

6. **Generate grouped trajectories for sequence-level GRPO.** For each audio input, sample N=4 complete responses. For each response, also generate the text-conditioned reference answer from x^t.

7. **Evaluate sequences with a judge model.** Use a capable LLM judge (with 99%+ self-evaluation accuracy) to produce binary rewards: `r_seq(y; x^a, x^t) = J(y, y_hat) in {0, 1}`, where J checks whether the audio output semantically matches the text output at the answer level.

8. **Compute GRPO advantages and sequence loss.** For each group of N responses, compute relative advantage: `A^(i) = r_seq^(i) - mean(r_seq)`. Apply the policy gradient: `L_seq = -E[(1/N) * sum(A^(i) * log p_theta(y^(i)|x^a))]`.

9. **Combine losses and update.** Sum the objectives: `L_CORD = L_tok + L_seq`. Backpropagate through the model with learning rate 3e-5. Both losses share the same audio rollout, making the pipeline computationally efficient.

10. **Validate on held-out cross-modal benchmarks.** Measure performance on both audio-conditioned and text-conditioned inputs for the same questions. Compute the modality gap (text accuracy minus audio accuracy). Target gap reduction of 40%+ across benchmarks while verifying no degradation on modality-specific tasks (music understanding, sound classification, speech recognition).

## Concrete Examples

**Example 1: Implementing the token-level weighted KL loss in PyTorch**

User: "I'm training a Qwen2-Audio model and audio reasoning is much worse than text. Help me implement CORD's token-level distillation loss."

Approach:
1. Identify the model's forward pass to obtain logits for both audio and text conditioning
2. Implement reverse KL with stop-gradient on the text (teacher) distribution
3. Add the dual importance weighting mechanism

Output:
```python
import torch
import torch.nn.functional as F

def cord_token_loss(
    audio_logits: torch.Tensor,   # (B, T, V) - logits from audio-conditioned path
    text_logits: torch.Tensor,    # (B, T, V) - logits from text-conditioned path (same prefix)
    alpha: float = 2.0,           # KL importance upweight factor
    beta: float = 2.0,            # positional decay start weight
    top_k: int = 20,              # number of high-KL tokens to upweight
) -> torch.Tensor:
    B, T, V = audio_logits.shape

    # Compute per-token distributions
    audio_log_probs = F.log_softmax(audio_logits, dim=-1)
    text_probs = F.softmax(text_logits.detach(), dim=-1)  # stop gradient on teacher
    text_log_probs = F.log_softmax(text_logits.detach(), dim=-1)

    # Reverse KL: KL(audio || text) = sum(audio * (log_audio - log_text))
    per_token_kl = (audio_log_probs.exp() * (audio_log_probs - text_log_probs)).sum(dim=-1)  # (B, T)

    # KL-based importance weighting: upweight top-K divergent tokens
    _, top_indices = per_token_kl.topk(top_k, dim=-1)  # (B, top_k)
    w_kl = torch.ones_like(per_token_kl)
    w_kl.scatter_(1, top_indices, alpha)

    # Positional decay weighting: early tokens get higher weight
    positions = torch.arange(T, device=audio_logits.device).float()
    w_pos = beta - (beta - 1.0) * positions / max(T - 1, 1)  # linear decay from beta to 1.0
    w_pos = w_pos.unsqueeze(0).expand(B, -1)

    # Combined weight
    w = w_kl * w_pos

    # Weighted token-level loss
    loss = (w * per_token_kl).sum(dim=-1).mean()
    return loss
```

**Example 2: Full CORD training loop with both token and sequence losses**

User: "Show me how to structure the complete CORD training loop combining both alignment levels."

Approach:
1. Structure the on-policy rollout generation
2. Combine token-level KL loss with sequence-level GRPO
3. Handle the judge evaluation step

Output:
```python
def cord_training_step(model, audio_input, text_input, judge_fn, n_samples=4):
    """Single CORD training step combining token-level and sequence-level alignment."""

    # === Token-level alignment (on-policy) ===
    # Generate audio rollout
    with torch.no_grad():
        audio_rollout = model.generate(audio_input, temperature=1.0, do_sample=True)

    # Forward pass: get logits for audio and text conditioned on SAME prefix
    audio_logits = model(audio_input, prefix=audio_rollout).logits
    text_logits = model(text_input, prefix=audio_rollout).logits  # same prefix, different modality

    L_tok = cord_token_loss(audio_logits, text_logits, alpha=2.0, beta=2.0, top_k=20)

    # === Sequence-level alignment (GRPO) ===
    # Sample N trajectories from audio-conditioned policy
    trajectories = []
    for _ in range(n_samples):
        with torch.no_grad():
            y = model.generate(audio_input, temperature=1.5, do_sample=True)
            trajectories.append(y)

    # Get text reference answer
    with torch.no_grad():
        text_ref = model.generate(text_input, temperature=0.0)

    # Judge evaluation: binary reward per trajectory
    rewards = torch.tensor([
        judge_fn(traj, text_ref) for traj in trajectories  # returns 0 or 1
    ], dtype=torch.float32)

    # GRPO advantage: relative within group
    advantages = rewards - rewards.mean()

    # Sequence-level policy gradient
    L_seq = torch.tensor(0.0)
    for traj, adv in zip(trajectories, advantages):
        log_prob = model.log_prob(traj, audio_input)
        L_seq -= adv * log_prob
    L_seq /= n_samples

    # Combined CORD objective
    L_cord = L_tok + L_seq
    return L_cord
```

**Example 3: Diagnosing the modality gap before applying CORD**

User: "How do I measure the audio-text reasoning gap to know if CORD would help my model?"

Approach:
1. Create a paired evaluation set with identical semantic content in both modalities
2. Run inference in both modes and compute the performance delta
3. Analyze where divergence concentrates

Output:
```python
def measure_modality_gap(model, eval_pairs, judge_fn):
    """Measure the audio-text performance gap across a paired evaluation set.
    Each pair is (audio_input, text_input, ground_truth_answer)."""
    audio_correct, text_correct = 0, 0
    token_kl_by_position = []

    for audio_in, text_in, gt_answer in eval_pairs:
        # Text-conditioned inference
        text_output = model.generate(text_in, temperature=0.0)
        text_score = judge_fn(text_output, gt_answer)
        text_correct += text_score

        # Audio-conditioned inference
        audio_output = model.generate(audio_in, temperature=0.0)
        audio_score = judge_fn(audio_output, gt_answer)
        audio_correct += audio_score

        # Per-position KL analysis (diagnostic)
        a_logits = model(audio_in, prefix=audio_output).logits
        t_logits = model(text_in, prefix=audio_output).logits
        kl_per_pos = compute_kl_per_token(a_logits, t_logits)  # (T,)
        token_kl_by_position.append(kl_per_pos)

    n = len(eval_pairs)
    text_acc = text_correct / n
    audio_acc = audio_correct / n
    gap = text_acc - audio_acc

    print(f"Text accuracy:  {text_acc:.2%}")
    print(f"Audio accuracy: {audio_acc:.2%}")
    print(f"Modality gap:   {gap:.2%}")
    print(f"CORD recommended: {'Yes' if gap > 0.03 else 'Gap is small, may not need CORD'}")

    # Show where KL concentrates (expect: early tokens, reasoning words)
    avg_kl = torch.stack(token_kl_by_position).mean(dim=0)
    print(f"Mean KL by position (first 10): {avg_kl[:10].tolist()}")
    return gap
```

## Best Practices

- **Do:** Use reverse KL divergence (not forward KL). Reverse KL makes the audio distribution mode-seek toward the text distribution's peaks, which is critical for reasoning tasks where you want the audio path to hit the same high-confidence decisions as text.
- **Do:** Weight early tokens more heavily. CORD's analysis shows that early-stage misalignments cascade into complete reasoning failures. The positional decay (beta=2 linearly decaying to 1) is a well-tested default.
- **Do:** Keep the top-K selection for KL importance weighting at K=20 across different model sizes. The paper found this value robust; the KL distribution is highly skewed (80th percentile ~0.23) so most tokens contribute near-zero gradient without this filtering.
- **Do:** Use the same model for both teacher (text) and student (audio) pathways. This is self-distillation -- no external teacher needed, and it ensures the text reference represents what *this specific model* can achieve.
- **Avoid:** Off-policy distillation (training on the teacher's output trajectories). This causes distribution mismatch -- the model never learns to recover from errors it actually makes during audio inference. Always generate audio rollouts first, then compute divergence along those rollouts.
- **Avoid:** Uniform token weighting. Without importance weighting, gradients are diluted by the vast majority of tokens that already have near-zero cross-modal divergence, making training inefficient.
- **Avoid:** Training only on domain-specific data and expecting narrow improvements. CORD's alignment generalizes: math-only training data improved general knowledge benchmarks (MMSU, OpenBookQA), so choose training data with rich reasoning traces rather than task-matched data.

## Error Handling

- **Judge model produces noisy rewards:** If the judge accuracy is below ~95%, sequence-level GRPO will be unstable. Validate judge reliability on a held-out set before training. Fall back to rule-based answer matching (exact match, regex extraction) for structured outputs like math.
- **KL divergence explodes at certain tokens:** Clip per-token KL to a reasonable maximum (e.g., 10.0) to prevent gradient spikes. This typically happens at tokens where the audio path is highly uncertain but text is confident.
- **Audio rollouts are degenerate (repetition, empty):** Increase sampling temperature or add a minimum length constraint. Degenerate rollouts provide no useful gradient signal for either loss term.
- **No measurable modality gap on evaluation:** If text and audio accuracy are already within 2-3%, CORD will provide minimal benefit. The technique specifically targets the modality gap, not general model quality.
- **Training divergence when combining L_tok and L_seq:** If the two losses have very different magnitudes, add a balancing coefficient. The paper uses equal weighting (1:1) but this assumes normalized loss scales.
- **TTS artifacts in synthetic training data:** Poor TTS quality introduces noise that the model cannot distinguish from the modality gap itself. Use a high-quality TTS model and validate that a human can understand the synthesized audio clearly.

## Limitations

- CORD requires the model to have a shared backbone for both modalities. It does not apply to systems with entirely separate audio and text models that are combined only at the output level.
- The technique addresses the *gap* between modalities but does not improve the text-conditioned upper bound. If the base LLM's reasoning is weak, CORD cannot compensate.
- Judge-based sequence rewards introduce a dependency on judge quality. For domains where semantic equivalence is hard to assess (creative writing, subjective opinions), the sequence-level component may be unreliable.
- Training requires paired data where audio and text carry identical semantic content. For tasks where audio contains unique information not in text (speaker emotion, acoustic scene), CORD's alignment objective could suppress modality-specific signals.
- The 80k sample efficiency was demonstrated on 7B-parameter models. Larger models may need proportionally more data, though the paper did not test this.

## Reference

**Paper:** [CORD: Bridging the Audio-Text Reasoning Gap via Weighted On-policy Cross-modal Distillation](https://arxiv.org/abs/2601.16547v1) (Hu et al., 2026)

**Key takeaway:** Look at Section 3 for the full mathematical formulation of the dual-weighted token loss and GRPO sequence loss, and Section 4.3 for ablation studies showing each component's contribution (importance weighting alone gives +5.8 avg improvement, adding positional weighting gives +7.2, full CORD gives +9.1).