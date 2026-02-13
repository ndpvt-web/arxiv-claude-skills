---
name: "diffuspeech-silent-thought-spoken"
description: |
  Apply the "Silent Thought, Spoken Answer" paradigm from DiffuSpeech to build speech-text
  generation pipelines where internal reasoning traces guide and improve speech output quality.
  Uses masked diffusion over unified discrete token vocabularies to jointly denoise text thinking
  and speech tokens with modality-specific masking schedules.
  Trigger phrases: "build a speech reasoning pipeline", "silent thought spoken answer",
  "diffusion speech generation with thinking", "speech QA with chain of thought",
  "joint text-speech diffusion model", "DiffuSpeech implementation"
---

# DiffuSpeech: Silent Thought, Spoken Answer

This skill enables Claude to implement the "Silent Thought, Spoken Answer" paradigm from DiffuSpeech — the first diffusion-based speech-text language model that jointly generates internal text reasoning and spoken responses through iterative masked denoising. Instead of autoregressive left-to-right generation (where errors in early tokens propagate irreversibly), this approach treats generation as iterative refinement: both thinking traces and speech tokens are denoised simultaneously, with the model committing to high-confidence predictions first and refining uncertain positions over multiple steps. The result is speech output that benefits from bidirectional context and explicit reasoning, achieving state-of-the-art speech-to-speech QA accuracy (+9 points over baselines) and TTS quality (6.2% WER).

## When to Use

- When building a speech-to-speech QA system that needs to reason before answering (e.g., math questions, factual queries answered via voice)
- When implementing a masked diffusion framework that unifies discrete text tokens and quantized speech codes under a single denoising process
- When designing a training pipeline with modality-specific masking schedules for joint text-speech generation
- When constructing a dataset pairing spoken QA samples with explicit text reasoning traces (like DiffuSpeech's ThinkingTalk dataset)
- When replacing an autoregressive speech LM with a diffusion-based alternative to avoid cascading token errors
- When implementing confidence-based iterative unmasking for discrete sequence generation

## Key Technique

**Masked Diffusion over Unified Tokens.** DiffuSpeech treats both text and speech as discrete tokens in a single vocabulary (126,973 tokens: 126,464 text + 500 HuBERT speech codes + 9 special tokens). The forward diffusion process corrupts sequences by independently masking each token with probability γ(t) = cos(π/2 · (1−t)), progressing from no masking (t=0) to full masking (t=1). The reverse process — a transformer (LLaDA backbone, 8B params) — learns to predict original tokens from corrupted versions. Crucially, during training, cross-entropy loss is applied only to masked target positions, while condition tokens (user input) remain unmasked. This selective masking means the model learns to reconstruct the response given the query, not to memorize inputs.

**Joint Reasoning + Speech Generation.** The sequence format is `[task_token, user_speech, thinking_text, reply_speech]`. During iterative denoising (default 64 steps), text reasoning and speech tokens inform each other bidirectionally — unlike autoregressive models where text reasoning must fully precede speech generation. At each denoising step i, the model predicts all masked positions, then unmasks the top-k_i most confident predictions (k_i = ⌈n·(T−i+1)/T⌉, a linear schedule). This confidence-first unmasking lets the model lock in structural decisions (word choice, prosody contours) early while refining fine details later. Ablations show thinking traces alone add +13.4 accuracy points, and diffusion outperforms autoregressive variants by converging faster (10K steps for ASR, 15K for TTS) to lower error rates (7.1% vs 11.4% ASR WER; 10.5% vs 14.7% TTS WER).

**Why This Matters for Implementation.** The core engineering insight is that speech prosody and semantic content are entangled — you need global context to produce natural speech. Autoregressive models commit to early tokens without seeing the full response. Diffusion lets the model "see" the whole utterance during generation, much like how humans think before speaking. This translates directly into implementation choices: use masked diffusion (not autoregressive) when output quality depends on global coherence, and pair it with explicit reasoning traces when the task requires multi-step logic.

## Step-by-Step Workflow

1. **Define the unified token vocabulary.** Combine your text tokenizer vocabulary with discrete speech codes. Use a speech encoder (HuBERT-base or similar) with k-means quantization to produce ~500 discrete speech codes. Add special tokens: `[MASK]`, `[S2S]`, `[S2T]`, `[T2S]`, `[T2T]`, `[BOS]`, `[EOS]`, `[SEP]`, `[PAD]`. Map all tokens to a single embedding space.

2. **Implement the masked diffusion forward process.** For a sequence of length n, at noise level t ∈ [0,1], independently replace each token with `[MASK]` with probability γ(t) = cos(π/2 · (1−t)). Apply masking only to target tokens (thinking + reply), never to condition tokens (task token + user input). This selective masking is critical — it teaches the model conditional generation, not unconditional reconstruction.

3. **Build the denoising transformer backbone.** Use a standard transformer (LLaDA-style) that takes the masked sequence and noise level t as input, and predicts logits over the full vocabulary for every masked position. The training loss is cross-entropy averaged over masked positions only: `L = −(1/|masked|) Σ log p(x_orig | x_masked, t)`.

4. **Implement modality-specific masking schedules.** Apply the cosine schedule γ(t) = cos(π/2 · (1−t)) but allow different noise sampling distributions per modality during training. Speech tokens may benefit from higher masking rates early in training to force the model to learn global speech structure before fine details.

5. **Structure training in two stages.** Stage 1 (speech-text alignment): Train on TTS, ASR, and text LM tasks in a 0.4/0.4/0.2 ratio using large-scale data (LibriHeavy, VoxPopuli, CommonVoice for speech; RefinedWeb for text). Stage 2 (instruction following): Fine-tune on speech-to-speech, speech-to-text, and text-to-text tasks using the ThinkingTalk-style dataset with reasoning traces.

6. **Construct the reasoning-paired speech QA dataset.** Take existing text QA datasets and rewrite them with an LLM (e.g., Qwen-32B) into three components: user question, assistant reasoning trace, and assistant spoken reply. Filter with an LLM judge across correctness, oral suitability, reasoning quality, and naturalness dimensions. Synthesize audio using a high-quality TTS system (diverse speakers for questions, consistent voice for answers).

7. **Implement confidence-based iterative unmasking for inference.** At inference, start with all target positions masked. For T=64 denoising steps, at step i: (a) run the model to get logits for all masked positions, (b) compute confidence as max softmax probability per position, (c) unmask the top k_i = ⌈n·(T−i+1)/T⌉ most confident positions by replacing `[MASK]` with predicted tokens, (d) feed the partially unmasked sequence into the next step. After T steps, all positions are unmasked.

8. **Decode speech tokens back to audio.** Pass the generated discrete speech codes through a frozen vocoder (HiFi-GAN or similar) to reconstruct the waveform. The text reasoning trace is available as a byproduct — log it for debugging, display it as a transcript, or use it for quality verification.

9. **Evaluate with appropriate metrics.** Measure speech quality with WER (target: <10% on clean benchmarks), QA accuracy with exact-match or LLM-judge scoring on spoken answers, and language understanding preservation with MMLU or similar benchmarks on the text pathway.

10. **Iterate on masking schedule and step count.** Ablate the number of denoising steps (32, 64, 128) against quality and latency. Fewer steps trade quality for speed. Adjust the confidence threshold or unmasking schedule shape (linear vs. cosine) based on your quality-latency requirements.

## Concrete Examples

**Example 1: Building the Masked Diffusion Training Loop**

User: "Implement the core masked diffusion training step for a unified text-speech model."

Approach:
1. Sample a batch of sequences with format `[task, user_input, thinking, reply]`
2. Sample noise levels t ~ Uniform(0,1) per sequence
3. Create masks by replacing target tokens with `[MASK]` at rate γ(t)
4. Forward through transformer, compute CE loss on masked positions only

Output:
```python
import torch
import torch.nn.functional as F
import math

def cosine_masking_rate(t):
    """γ(t) = cos(π/2 · (1-t)), maps t∈[0,1] to masking probability."""
    return torch.cos(math.pi / 2 * (1 - t))

def create_masked_input(tokens, t, condition_mask, mask_token_id):
    """Apply masking to target positions only.

    Args:
        tokens: (B, L) original token IDs
        t: (B,) noise levels per sample
        condition_mask: (B, L) bool, True for positions that should never be masked
        mask_token_id: int, ID of [MASK] token
    """
    gamma = cosine_masking_rate(t)  # (B,)
    # Per-token masking probability, zero for condition positions
    mask_prob = gamma.unsqueeze(1).expand_as(tokens)  # (B, L)
    mask_prob = mask_prob * (~condition_mask).float()
    # Sample which tokens to mask
    should_mask = torch.bernoulli(mask_prob).bool()
    masked_tokens = tokens.clone()
    masked_tokens[should_mask] = mask_token_id
    return masked_tokens, should_mask

def diffusion_training_step(model, tokens, condition_mask, mask_token_id):
    """One training step of masked diffusion."""
    B = tokens.shape[0]
    # Sample noise levels uniformly
    t = torch.rand(B, device=tokens.device)
    # Mask target tokens
    masked_tokens, was_masked = create_masked_input(
        tokens, t, condition_mask, mask_token_id
    )
    # Predict original tokens from masked input
    logits = model(masked_tokens, t)  # (B, L, V)
    # Loss only on masked positions
    loss = F.cross_entropy(
        logits[was_masked], tokens[was_masked], reduction='mean'
    )
    return loss
```

**Example 2: Implementing Confidence-Based Iterative Unmasking at Inference**

User: "Write the inference loop that iteratively unmasks tokens using confidence scores."

Approach:
1. Initialize all target positions as `[MASK]`
2. At each step, predict logits, score confidence, unmask top-k
3. Use linear schedule for k to ensure all positions filled by step T

Output:
```python
import torch
import math

def diffusion_inference(model, condition_tokens, num_target_positions, T=64,
                        mask_token_id=0, device='cuda'):
    """Iterative confidence-based unmasking inference.

    Args:
        model: trained denoising transformer
        condition_tokens: (1, L_cond) token IDs for task + user input
        num_target_positions: int, number of positions to generate
        T: int, number of denoising steps
        mask_token_id: int, ID of [MASK] token
    """
    B = 1
    n = num_target_positions
    # Start fully masked
    target = torch.full((B, n), mask_token_id, dtype=torch.long, device=device)
    sequence = torch.cat([condition_tokens, target], dim=1)
    target_start = condition_tokens.shape[1]

    is_masked = torch.ones(B, n, dtype=torch.bool, device=device)

    for i in range(T):
        # Noise level decreases from 1 to 0
        t = torch.tensor([1.0 - i / T], device=device)

        logits = model(sequence, t)  # (B, L_total, V)
        target_logits = logits[:, target_start:, :]  # (B, n, V)

        # Confidence = max probability per position
        probs = torch.softmax(target_logits, dim=-1)
        confidence, predicted = probs.max(dim=-1)  # (B, n)

        # Only consider still-masked positions
        confidence[~is_masked] = -1.0

        # Number of positions to unmask this step
        k_i = math.ceil(n * (T - i) / T) - math.ceil(n * (T - i - 1) / T)
        k_i = max(k_i, 1)

        if is_masked.sum() <= k_i:
            # Unmask everything remaining
            unmask = is_masked.clone()
        else:
            # Top-k most confident masked positions
            _, topk_idx = confidence.topk(k_i, dim=1)
            unmask = torch.zeros_like(is_masked)
            unmask.scatter_(1, topk_idx, True)
            unmask &= is_masked

        # Commit predictions at unmasked positions
        target[unmask] = predicted[unmask]
        is_masked[unmask] = False

        # Update full sequence
        sequence = torch.cat([condition_tokens, target], dim=1)

    return target  # (B, n) generated token IDs
```

**Example 3: Building the ThinkingTalk-Style Dataset Pipeline**

User: "Create a pipeline that converts text QA pairs into speech QA samples with reasoning traces."

Approach:
1. Rewrite text QA pairs into spoken format with explicit reasoning
2. Filter with quality criteria
3. Synthesize speech for both questions and answers

Output:
```python
# Step 1: Rewrite QA pairs with reasoning traces
REWRITE_PROMPT = """Rewrite this QA pair for a spoken conversation.
Produce three components:
- user_question: Natural spoken question (conversational, no written-only artifacts)
- reasoning: Step-by-step thinking the assistant does internally (2-4 sentences)
- spoken_reply: Natural spoken answer (concise, clear for audio)

Original Q: {question}
Original A: {answer}

Output as JSON with keys: user_question, reasoning, spoken_reply"""

# Step 2: Quality filtering with LLM judge
JUDGE_PROMPT = """Rate this spoken QA sample on a 1-5 scale for each criterion:
1. Correctness: Is the reasoning and answer factually correct?
2. Oral suitability: Would this sound natural when spoken aloud?
3. Reasoning quality: Does the thinking trace logically lead to the answer?
4. Conciseness: Is the spoken reply appropriately brief?

Sample: {sample}
Output JSON: {{"correctness": N, "oral": N, "reasoning": N, "conciseness": N}}"""

QUALITY_THRESHOLD = {"correctness": 4, "oral": 3, "reasoning": 3, "conciseness": 3}

# Step 3: Dataset assembly
def build_thinking_talk_dataset(qa_pairs, llm_client, tts_engine):
    """Convert text QA to speech QA with reasoning traces."""
    samples = []
    for qa in qa_pairs:
        # Rewrite
        rewritten = llm_client.generate(
            REWRITE_PROMPT.format(question=qa["q"], answer=qa["a"])
        )
        # Judge
        scores = llm_client.generate(
            JUDGE_PROMPT.format(sample=rewritten)
        )
        if all(scores[k] >= QUALITY_THRESHOLD[k] for k in QUALITY_THRESHOLD):
            # Synthesize audio (diverse speakers for questions)
            speaker_id = hash(qa["q"]) % 4  # rotate 4 speakers
            q_audio = tts_engine.synthesize(rewritten["user_question"], speaker=speaker_id)
            a_audio = tts_engine.synthesize(rewritten["spoken_reply"], speaker=0)
            samples.append({
                "question_audio": q_audio,
                "thinking_text": rewritten["reasoning"],
                "answer_audio": a_audio,
                "question_text": rewritten["user_question"],
                "answer_text": rewritten["spoken_reply"],
            })
    return samples
```

## Best Practices

- **Do:** Apply selective masking — never mask condition/input tokens during training. The model must learn conditional generation, not sequence memorization. This is the single most common implementation mistake.
- **Do:** Use confidence-based unmasking with a linear schedule during inference. Greedy unmasking (all-at-once) degrades quality significantly; the iterative approach lets high-confidence structural tokens anchor the generation.
- **Do:** Include explicit text reasoning traces in the training data even if your final output is speech-only. Ablations show +13.4 accuracy points from thinking traces. The text trace acts as an internal scaffold.
- **Do:** Train in two stages — alignment first (TTS/ASR/LM), then instruction following. Skipping stage 1 leads to poor speech-text alignment and high WER.
- **Avoid:** Using fewer than 32 denoising steps at inference. Quality degrades sharply below this threshold. 64 steps is the recommended default; 128 shows diminishing returns.
- **Avoid:** Autoregressive generation when the task requires global coherence across the output (e.g., matching prosody to semantic content). Diffusion's bidirectional context is the core advantage — if your output is short and sequential, autoregressive may be simpler and sufficient.

## Error Handling

- **Unmasking stalls (no confident predictions):** If max confidence across all masked positions drops below 0.1, the model is uncertain about the entire remaining sequence. Fall back to sampling from the predicted distribution rather than argmax, or reduce the number of positions unmasked per step.
- **Modality mismatch in generated tokens:** If the model outputs text tokens in speech positions or vice versa, enforce vocabulary constraints by masking logits: zero out text token logits for speech positions and speech token logits for text positions before softmax.
- **Vocoder artifacts from out-of-distribution codes:** If generated speech codes produce distorted audio, apply a post-hoc filter: reject sequences where any speech code appears fewer than N times in the training distribution, or re-run denoising with those positions re-masked.
- **Reasoning trace is incoherent but speech is fine:** This indicates the model hasn't learned strong text-speech coupling. Increase the weight of text positions in the loss, or add an auxiliary text-only reasoning loss during stage 2 training.

## Limitations

- Requires a pretrained speech encoder (HuBERT) and vocoder (HiFi-GAN) — this approach does not learn end-to-end waveform generation. Audio quality is bounded by vocoder quality.
- The 64-step iterative inference is inherently slower than single-pass autoregressive generation for the same sequence length. Not suitable for real-time streaming applications without step-count reduction techniques.
- Discrete speech codes (500 codes from HuBERT) lose fine acoustic detail compared to continuous representations. Speaker identity, emotion, and subtle prosody may be degraded.
- The ThinkingTalk dataset (26K samples, 319 hours) is English-only. Multilingual extension requires language-specific speech encoders and TTS systems for data construction.
- The "silent thought" paradigm adds computational overhead for tasks that don't require reasoning (e.g., simple TTS). For pure synthesis tasks, the thinking trace pathway is unnecessary weight.

## Reference

[DiffuSpeech: Silent Thought, Spoken Answer via Unified Speech-Text Diffusion](https://arxiv.org/abs/2601.22889v1) — Look for: Section 3 (masked diffusion framework and modality-specific masking), Section 4 (ThinkingTalk dataset construction), Table 1 (S2S QA accuracy comparisons), Table 2 (TTS WER results), and Section 5.3 (ablations on thinking traces and diffusion vs. autoregressive).