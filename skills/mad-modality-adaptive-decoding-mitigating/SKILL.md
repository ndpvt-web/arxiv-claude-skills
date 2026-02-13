---
name: "mad-modality-adaptive-decoding-mitigating"
description: "Implement Modality-Adaptive Decoding (MAD) to suppress cross-modal hallucinations in multimodal LLMs. Uses self-assessment queries and weighted contrastive decoding branches to let models focus on task-relevant modalities. Trigger phrases: 'reduce multimodal hallucination', 'cross-modal interference', 'modality-adaptive decoding', 'contrastive decoding for multimodal', 'suppress audio-visual hallucination', 'MAD decoding pipeline'"
---

# Modality-Adaptive Decoding (MAD) for Mitigating Cross-Modal Hallucinations

This skill enables Claude to implement and integrate the MAD (Modality-Adaptive Decoding) technique into multimodal LLM inference pipelines. MAD is a training-free method that eliminates cross-modal hallucinations -- where one input modality (e.g., audio) inappropriately biases generation about another (e.g., video) -- by first querying the model to self-assess which modalities are relevant to a given task, then using those modality probabilities to adaptively weight four contrastive decoding branches during token generation.

## When to Use

- When building or modifying an audio-visual language model inference pipeline that suffers from one modality overriding another (e.g., a model describing sounds that match the video scene but aren't actually present in the audio)
- When implementing contrastive decoding for any multimodal LLM and needing adaptive per-query modality weighting instead of fixed static weights
- When the user asks to reduce hallucinations in VideoLLaMA2-AV, Qwen2.5-Omni, or similar audio-visual language models without retraining
- When integrating a modality self-assessment step into an existing MLLM generation loop to dynamically route attention
- When evaluating multimodal models on cross-modal hallucination benchmarks (CMM, AVHBench) and needing a decoding-time intervention
- When the user wants to build a modality-aware inference wrapper around any multimodal model that processes two or more input streams

## Key Technique

**The Problem.** Standard multimodal LLMs process all input modalities simultaneously, which causes cross-modal hallucination: a model watching a video of a dog park with birds chirping in the audio might hallucinate "the dogs are barking" because the visual content biases the audio-related generation. This happens because the model lacks explicit modality-interaction control during decoding.

**Self-Assessment Query.** MAD's first innovation is a self-assessment step. Before generating the actual answer, the model is prompted with: *"To answer this question, which modality is needed (audio, video, or both)?"* The logits for the tokens `audio`, `video`, and `both` are extracted and softmax-normalized into weights `[w_a, w_v, w_av]`. This exploits the model's own internal understanding of task requirements -- no external classifier needed.

**Weighted Four-Branch Contrastive Decoding.** MAD then runs four parallel decoding branches at each token step, each comparing a "full modality" configuration against a "degraded modality" configuration: (1) full AV vs. degraded-video+audio, (2) full AV vs. video+degraded-audio, (3) video-only vs. both-degraded, (4) audio-only vs. both-degraded. The contrastive delta from each branch is scaled by `gamma * w_m` (where gamma is a fixed base strength, typically 2.5, and `w_m` is the self-assessed modality weight), then added to the base logits. This means audio-focused questions amplify audio-discriminative signals while suppressing video interference, and vice versa.

## Step-by-Step Workflow

1. **Identify the multimodal model architecture.** Determine whether the target model (e.g., Qwen2.5-Omni, VideoLLaMA2-AV) exposes per-token logits and supports selective modality masking or degradation at inference time. Check if audio and video encoders can be independently bypassed.

2. **Implement the modality self-assessment query.** Before the main generation pass, construct a prompt that appends the question: `"To answer this question, which modality is needed (audio, video, or both)?"` Feed the full multimodal input (video frames `X_v`, audio features `X_a`, text question `X_q`) along with this query `X_m` through the model. Extract the raw logits at the first generated token position for the tokens corresponding to `"audio"`, `"video"`, and `"both"`.

3. **Compute adaptive modality weights.** Apply softmax over the three extracted logits to produce normalized weights:
   ```python
   import torch.nn.functional as F
   z = torch.tensor([z_both, z_video, z_audio])
   w_av, w_v, w_a = F.softmax(z, dim=0).tolist()
   ```

4. **Set up the four modality-branch forward passes.** For each token generation step, prepare four input configurations:
   - Branch 1 (Visual CD, audio present): Full `(X_v, X_a)` vs. degraded `(X_v_degraded, X_a)`
   - Branch 2 (Audio CD, visual present): Full `(X_v, X_a)` vs. `(X_v, X_a_degraded)`
   - Branch 3 (Visual CD, audio absent): `(X_v, -)` vs. `(X_v_degraded, -)`
   - Branch 4 (Audio CD, visual absent): `(-, X_a)` vs. `(-, X_a_degraded)`

   "Degraded" means replacing the modality input with noise, zeros, or a learned null token depending on the model's architecture.

5. **Compute the MAD logits at each decoding step.** For each token position `t`, compute:
   ```python
   # Base logits from full multimodal input
   logits_base = model(X_v, X_a, X_q, y_<t)

   # Contrastive deltas for each branch
   delta_1 = logits_full_av - logits_degraded_v_with_a
   delta_2 = logits_full_av - logits_v_with_degraded_a
   delta_3 = logits_v_only - logits_both_degraded
   delta_4 = logits_a_only - logits_both_degraded

   # Weighted MAD logits
   logits_mad = logits_base + gamma * (
       w_av * (delta_1 + delta_2) +
       w_v * delta_3 +
       w_a * delta_4
   )
   ```

6. **Select the next token using the MAD logits.** Apply the standard sampling strategy (argmax, top-k, nucleus) to `logits_mad` instead of the raw model logits. Append the selected token and continue until EOS.

7. **Tune the gamma parameter.** Start with `gamma = 2.5` (paper's optimal on 100-sample ablation over [0.5, 3.0] in 0.5 steps). If output becomes degenerate or overly conservative, reduce gamma. If hallucinations persist, increase it.

8. **Validate with modality-specific test cases.** Test with queries that specifically target one modality (e.g., "What sound is playing?" for audio, "What color is the object?" for video) and verify the self-assessed weights `[w_av, w_v, w_a]` correctly reflect the expected modality relevance.

9. **Optimize inference throughput.** The four-branch approach requires ~4x forward passes per token. Batch the branch computations into a single forward pass where possible using input stacking. Cache KV states shared across branches to reduce redundant computation.

10. **Integrate into the serving pipeline.** Wrap the MAD logic as a custom `generate()` method or a logits processor hook that intercepts the model's token selection. Expose `gamma` as a configurable parameter in the API or CLI.

## Concrete Examples

**Example 1: Adding MAD to a Qwen2.5-Omni inference script**

User: "I'm getting hallucinations where my Qwen2.5-Omni model describes sounds from the video scene instead of the actual audio track. Help me add MAD decoding."

Approach:
1. Locate the model's generation call (typically `model.generate()` or a custom loop).
2. Add a pre-generation self-assessment query pass.
3. Implement the four-branch logits processor.

Output (logits processor skeleton):
```python
class MADLogitsProcessor:
    def __init__(self, model, video_input, audio_input, gamma=2.5):
        self.model = model
        self.video = video_input
        self.audio = audio_input
        self.degraded_video = torch.zeros_like(video_input)
        self.degraded_audio = torch.zeros_like(audio_input)
        self.gamma = gamma
        self.weights = self._assess_modality_weights()

    def _assess_modality_weights(self):
        """Query the model to self-assess modality relevance."""
        query = "To answer this question, which modality is needed (audio, video, or both)?"
        logits = self.model.get_next_token_logits(
            video=self.video, audio=self.audio, text=query
        )
        token_ids = [
            tokenizer.encode("both")[0],
            tokenizer.encode("video")[0],
            tokenizer.encode("audio")[0],
        ]
        z = logits[0, token_ids]
        return F.softmax(z, dim=0)  # [w_av, w_v, w_a]

    def __call__(self, input_ids, scores):
        """Apply MAD contrastive adjustment to logits."""
        w_av, w_v, w_a = self.weights

        # Full AV logits (scores already computed by model)
        logits_full = scores

        # Branch forward passes for contrastive deltas
        logits_degraded_v_a = self.model.forward(self.degraded_video, self.audio, input_ids)
        logits_v_degraded_a = self.model.forward(self.video, self.degraded_audio, input_ids)
        logits_v_only = self.model.forward(self.video, None, input_ids)
        logits_a_only = self.model.forward(None, self.audio, input_ids)
        logits_none = self.model.forward(self.degraded_video, self.degraded_audio, input_ids)

        delta_1 = logits_full - logits_degraded_v_a
        delta_2 = logits_full - logits_v_degraded_a
        delta_3 = logits_v_only - logits_none
        delta_4 = logits_a_only - logits_none

        mad_logits = logits_full + self.gamma * (
            w_av * (delta_1 + delta_2) + w_v * delta_3 + w_a * delta_4
        )
        return mad_logits
```

**Example 2: Evaluating MAD on AVHBench**

User: "I want to benchmark my audio-visual model on AVHBench with and without MAD to measure hallucination reduction."

Approach:
1. Run baseline inference on AVHBench with standard decoding.
2. Run MAD-enhanced inference with `gamma=2.5`.
3. Compare accuracy on cross-modal hallucination questions.

Output (evaluation harness):
```python
results = {"baseline": [], "mad": []}

for sample in avhbench_dataset:
    video, audio, question, ground_truth = sample

    # Baseline
    baseline_answer = model.generate(video=video, audio=audio, prompt=question)
    results["baseline"].append(baseline_answer == ground_truth)

    # MAD
    processor = MADLogitsProcessor(model, video, audio, gamma=2.5)
    mad_answer = model.generate(
        video=video, audio=audio, prompt=question,
        logits_processor=[processor]
    )
    results["mad"].append(mad_answer == ground_truth)

baseline_acc = sum(results["baseline"]) / len(results["baseline"])
mad_acc = sum(results["mad"]) / len(results["mad"])
print(f"Baseline: {baseline_acc:.1%}, MAD: {mad_acc:.1%}, Delta: {mad_acc - baseline_acc:+.1%}")
```

**Example 3: Diagnosing which modality the model over-relies on**

User: "My model keeps describing video content when asked about audio. How can I verify this is a cross-modal hallucination issue?"

Approach:
1. Run the self-assessment query on a batch of audio-only questions.
2. Check whether `w_v` is incorrectly high for audio-targeted questions.
3. If weights are correct but hallucinations persist, the issue is in generation, not assessment -- MAD is the right fix.

Output (diagnostic script):
```python
audio_questions = load_audio_only_questions()
weight_log = []

for q in audio_questions:
    weights = assess_modality_weights(model, video, audio, q)
    weight_log.append({"question": q, "w_av": weights[0], "w_v": weights[1], "w_a": weights[2]})
    print(f"Q: {q[:60]}... | w_av={weights[0]:.2f} w_v={weights[1]:.2f} w_a={weights[2]:.2f}")

avg_w_v = sum(w["w_v"] for w in weight_log) / len(weight_log)
print(f"\nAvg video weight for audio questions: {avg_w_v:.2f}")
if avg_w_v > 0.3:
    print("WARNING: Model over-attributes relevance to video for audio tasks.")
    print("Self-assessment may need calibration, but MAD can still help via contrastive suppression.")
```

## Best Practices

- **Do:** Always run the self-assessment query with the exact user question appended, not a generic placeholder. The weight quality depends on seeing the actual task.
- **Do:** Start with `gamma=2.5` and tune on a small held-out set (50-100 examples). The optimal value is task-dependent but the paper found 2.5 robust across benchmarks.
- **Do:** Cache the self-assessed weights per question -- they are computed once and reused for all tokens in the response.
- **Do:** Use the same tokenizer to resolve the token IDs for "audio", "video", and "both" -- different tokenizers may split these words differently.
- **Avoid:** Setting gamma too high (>3.0), which causes the contrastive term to dominate and produces degenerate or repetitive text.
- **Avoid:** Applying MAD to unimodal models or tasks where only one modality is ever present. MAD is specifically designed for cross-modal interference in multimodal inputs.
- **Avoid:** Degrading modalities by simply removing tokens from the sequence without replacing them. Use zero vectors or learned null embeddings to maintain positional alignment.

## Error Handling

- **Degenerate output (repetition, empty responses):** Reduce gamma by 0.5 increments. If the contrastive signal is too strong, it collapses the output distribution.
- **Self-assessment returns uniform weights:** The model may not distinguish modality relevance for ambiguous questions. Fall back to equal weights `[0.33, 0.33, 0.33]` which reduces MAD to standard contrastive decoding.
- **Token ID mismatch for modality words:** Some tokenizers split "audio" or "video" into subword pieces. Use the first subword token's logit, or sum logits across all subword tokens for that word.
- **OOM with four-branch forward passes:** Batch branches as pairs, run sequentially with gradient checkpointing disabled, or use 4-bit quantization (`--load-4bit`). Share KV cache across branches that have identical prefix inputs.
- **Weights don't correlate with expected modality:** Verify the self-assessment prompt is formatted exactly as the model expects (chat template, system prompt). Malformed prompts produce unreliable logits.

## Limitations

- **Inference cost:** MAD requires ~4x forward passes per token plus the one-time self-assessment query. This is significant for real-time applications. KV caching mitigates but does not eliminate this overhead.
- **Model compatibility:** Currently validated only on VideoLLaMA2-AV and Qwen2.5-Omni. Applying to models with fused multimodal encoders (where modalities cannot be independently masked) requires architectural adaptation.
- **Two-modality assumption:** The four-branch formulation is designed for audio+video. Extending to three or more modalities (e.g., audio+video+text+depth) requires combinatorial expansion of branches.
- **Self-assessment reliability:** The quality of modality weights depends on the model's ability to introspect. Weaker models may produce unreliable self-assessments, reducing MAD's effectiveness.
- **Not a fix for unimodal hallucinations:** MAD targets cross-modal interference specifically. Hallucinations caused by insufficient knowledge or reasoning errors within a single modality are unaffected.

## Reference

- **Paper:** [MAD: Modality-Adaptive Decoding for Mitigating Cross-Modal Hallucinations in Multimodal Large Language Models](https://arxiv.org/abs/2601.21181v1) -- Focus on Section 3 (Method) for the full four-branch contrastive formula and Algorithm 1 pseudocode, and Section 4.3 for gamma ablation results.
- **Code:** [https://github.com/top-yun/MAD](https://github.com/top-yun/MAD) -- Reference implementations for Qwen2.5-Omni and VideoLLaMA2-AV.