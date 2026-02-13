---
name: "emoshift-lightweight-activation-steering"
description: "Implement lightweight activation steering for emotion-controllable speech synthesis. Adds learned steering vectors to LLM-based TTS hidden states to shift emotional expression without full fine-tuning. Use when: 'add emotion control to TTS', 'steer model activations for emotion', 'lightweight emotional speech synthesis', 'activation steering for expressive TTS', 'control emotion intensity in speech generation', 'EmoShift emotion-aware synthesis'."
---

# EmoShift: Lightweight Activation Steering for Emotion-Aware Speech Synthesis

This skill enables Claude to help users implement **activation steering** for emotion-controllable text-to-speech systems, following the EmoShift framework (ICASSP 2026). The core idea: instead of scaling fixed emotion embeddings or fully fine-tuning a TTS model, you learn a small steering vector per emotion that shifts hidden-state activations in the output embedding space. This achieves superior emotional expressiveness with only ~10M trainable parameters (~1/30 of full fine-tuning), while preserving speaker identity and naturalness.

## When to Use

- When the user wants to add controllable emotion to an LLM-based TTS system (e.g., CosyVoice, VALL-E, or similar autoregressive speech-token models) without full fine-tuning.
- When the user asks how to implement activation steering or representation engineering for audio/speech generation models.
- When the user needs parameter-efficient adaptation of a speech model to express specific emotions (happy, angry, sad, surprise, neutral).
- When the user wants runtime control over emotional intensity in synthesized speech (e.g., "make it 3x angrier").
- When the user is building an expressive dialogue system or virtual assistant and needs emotion modulation at inference time.
- When the user asks about steering vectors, latent-space emotion offsets, or lightweight emotion injection layers for generative models.

## Key Technique

**Activation steering** works by learning a direction in a model's hidden-state space that corresponds to a target behavior -- here, a specific emotion. EmoShift introduces an **EmoSteer layer**: a small learnable projection matrix W_e (dimensions d x d) per emotion category. Given hidden states **h** from the TTS backbone, the steering vector is computed as **v**_e = **h** @ W_e, and the steered hidden state becomes **h'** = **h** + epsilon * **v**_e, where epsilon is a small fixed scaling factor (default 0.001). This additive offset nudges the model's internal representations toward emotion-specific regions without destroying the learned speech quality or speaker characteristics.

The critical insight is that emotions manifest as *directional offsets* in latent space, not as absolute positions. By learning these offsets rather than fixed embeddings, the model captures emotion-specific characteristics that generalize across utterances and speakers. At inference time, you gain a free knob for **intensity control**: replace epsilon with alpha * epsilon where alpha >= 1. Setting alpha = 3 boosts emotion recognition accuracy from 74.3% to 75.9% without degrading naturalness.

The entire EmoSteer layer adds only ~10M parameters and trains in 5 epochs on as few as 300 utterances per emotion, using standard negative log-likelihood loss on ground-truth speech tokens. The base model weights stay frozen. This makes the approach practical for adding emotion control to any pretrained autoregressive TTS system.

## Step-by-Step Workflow

1. **Select or load the base TTS model.** Use a pretrained LLM-based TTS backbone (e.g., CosyVoice-300M-Instruct or similar autoregressive speech-token generator). Freeze all its parameters.

2. **Define emotion categories.** Establish the target emotion set (e.g., neutral, happy, angry, sad, surprise). Each emotion gets its own learnable projection matrix W_e of shape (d, d), where d is the hidden-state dimension of the backbone.

3. **Implement the EmoSteer layer.** Create a module that:
   - Takes the backbone's hidden states **h** (shape: [batch, seq_len, d]).
   - Selects the appropriate W_e based on the target emotion label.
   - Computes the steering vector: **v**_e = h @ W_e.
   - Returns steered states: **h'** = **h** + epsilon * **v**_e (epsilon = 0.001 default).

4. **Inject the EmoSteer layer into the model.** Place it at the output embedding layer of the autoregressive decoder -- after the final transformer block, before the token-prediction head. This is where emotion-specific activation offsets have the most direct effect on generated speech tokens.

5. **Prepare the training data.** Collect or use an emotional speech dataset (e.g., ESD) with parallel utterances across emotion categories. ~300 utterances per emotion is sufficient. Format input sequences as: [start_token, speaker_emb, emotion_prompt, text_emb, speech_tokens, end_token].

6. **Train with teacher forcing.** Freeze all backbone parameters. Train only the EmoSteer projection matrices W_e using:
   - Loss: negative log-likelihood on ground-truth speech tokens.
   - Learning rate: 1e-4.
   - Epochs: 5.
   - Standard AdamW optimizer.

7. **Generate speech with emotion control.** At inference, run the autoregressive decoder with the EmoSteer layer active. The emotion is selected by choosing the corresponding W_e. The model generates discrete speech tokens which are decoded to waveforms via the backbone's vocoder (e.g., flow-matching decoder).

8. **Tune emotional intensity at inference.** Multiply the steering factor by a gain alpha: **h'** = **h** + alpha * epsilon * **v**_e. Use alpha = 1.0 for default intensity, alpha = 2-3 for stronger expression, alpha < 1 for subtler emotion. No retraining needed.

9. **Evaluate quality.** Measure emotion recognition accuracy (feed synthesized audio through an SER model), speaker similarity (cosine similarity of speaker embeddings), and naturalness (MOS listening tests or UTMOS automatic scoring).

10. **Iterate on the steering factor.** If emotions bleed into each other or naturalness degrades, reduce epsilon or alpha. If emotion is too subtle, increase alpha. The separation between emotion vectors in latent space can be visualized via t-SNE/PCA to diagnose issues.

## Concrete Examples

**Example 1: Adding an EmoSteer layer to a PyTorch TTS model**

User: "I have a pretrained autoregressive TTS model with hidden dim 1024. How do I add emotion steering?"

Approach:
1. Define the EmoSteer module with per-emotion projection matrices.
2. Hook it into the model's forward pass at the output embedding layer.
3. Freeze backbone, train only EmoSteer.

Output:
```python
import torch
import torch.nn as nn

class EmoSteerLayer(nn.Module):
    """Lightweight activation steering for emotion control.
    Learns a projection matrix per emotion that computes a steering
    vector added to hidden states with a small scaling factor.
    """
    def __init__(self, hidden_dim: int, emotions: list[str], epsilon: float = 0.001):
        super().__init__()
        self.epsilon = epsilon
        self.emotions = emotions
        self.emotion_to_idx = {e: i for i, e in enumerate(emotions)}
        # One projection matrix per emotion: W_e of shape (d, d)
        self.projections = nn.ParameterList([
            nn.Parameter(torch.randn(hidden_dim, hidden_dim) * 0.01)
            for _ in emotions
        ])

    def forward(self, h: torch.Tensor, emotion: str, alpha: float = 1.0) -> torch.Tensor:
        """
        Args:
            h: Hidden states [batch, seq_len, hidden_dim]
            emotion: Target emotion name
            alpha: Intensity gain (1.0 = default, >1 = stronger)
        Returns:
            Steered hidden states of same shape
        """
        idx = self.emotion_to_idx[emotion]
        W_e = self.projections[idx]
        v_e = h @ W_e                          # steering vector
        return h + alpha * self.epsilon * v_e   # additive steering

# Usage:
emotions = ["neutral", "happy", "angry", "sad", "surprise"]
emosteer = EmoSteerLayer(hidden_dim=1024, emotions=emotions)

# Trainable params: 5 * 1024 * 1024 = ~5.2M (plus any bias terms)
print(f"EmoSteer params: {sum(p.numel() for p in emosteer.parameters()):,}")
```

**Example 2: Integrating EmoSteer into a CosyVoice-style forward pass**

User: "How do I inject the steering layer into my existing model without modifying the base code?"

Approach:
1. Use a forward hook on the target layer.
2. The hook applies the EmoSteer offset before the prediction head.

Output:
```python
class EmoShiftWrapper:
    """Wraps a frozen TTS model with emotion steering via hooks."""

    def __init__(self, model, hidden_dim, emotions):
        self.model = model
        self.emosteer = EmoSteerLayer(hidden_dim, emotions)
        self.current_emotion = "neutral"
        self.current_alpha = 1.0
        self._hook = None

    def activate(self, target_layer_name: str):
        """Register a forward hook on the output embedding layer."""
        target = dict(self.model.named_modules())[target_layer_name]

        def hook_fn(module, input, output):
            return self.emosteer(output, self.current_emotion, self.current_alpha)

        self._hook = target.register_forward_hook(hook_fn)

    def set_emotion(self, emotion: str, alpha: float = 1.0):
        self.current_emotion = emotion
        self.current_alpha = alpha

    def deactivate(self):
        if self._hook:
            self._hook.remove()

# Usage:
wrapper = EmoShiftWrapper(tts_model, hidden_dim=1024,
                          emotions=["neutral", "happy", "angry", "sad", "surprise"])
wrapper.activate("decoder.output_projection")
wrapper.set_emotion("angry", alpha=2.5)
# Now tts_model.generate(...) produces angry speech
```

**Example 3: Training loop for the EmoSteer layer**

User: "Show me how to train only the steering vectors while keeping the TTS model frozen."

Output:
```python
from torch.optim import AdamW

# Freeze the entire base model
for param in tts_model.parameters():
    param.requires_grad = False

# Only the EmoSteer projections are trainable
optimizer = AdamW(wrapper.emosteer.parameters(), lr=1e-4)

for epoch in range(5):
    for batch in emotion_dataloader:
        text_tokens = batch["text_tokens"].cuda()
        speech_tokens = batch["speech_tokens"].cuda()
        emotion_label = batch["emotion"]       # e.g., "happy"
        speaker_emb = batch["speaker_emb"].cuda()

        wrapper.set_emotion(emotion_label)

        # Teacher-forced forward pass through the backbone
        logits = tts_model(text_tokens, speaker_emb, speech_tokens[:, :-1])
        # logits pass through the hooked layer, getting steered

        loss = nn.functional.cross_entropy(
            logits.view(-1, logits.size(-1)),
            speech_tokens[:, 1:].reshape(-1)
        )

        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

    print(f"Epoch {epoch+1}/5 -- loss: {loss.item():.4f}")
```

## Best Practices

- **Do:** Keep epsilon small (0.001) during training. The steering offset should be a subtle nudge, not a large perturbation. Large offsets destroy speech quality.
- **Do:** Freeze all backbone parameters. The entire point is parameter efficiency -- training the backbone defeats the purpose and risks catastrophic forgetting.
- **Do:** Use alpha for intensity control at inference only. It is a free knob that requires no retraining. Start with alpha = 1.0 and adjust up to 3.0.
- **Do:** Initialize projection matrices with small random values (std = 0.01). Large initial values cause training instability.
- **Avoid:** Sharing a single projection matrix across emotions. Each emotion needs its own W_e to capture distinct latent directions.
- **Avoid:** Placing the EmoSteer layer too early in the network (e.g., after the first transformer block). It is most effective at the output embedding layer where representations directly influence token predictions.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| Generated speech has no audible emotion | epsilon or alpha too small; hook on wrong layer | Increase alpha to 2-3; verify the hook is on the output embedding layer |
| Speech quality degrades severely | epsilon too large; backbone not frozen | Reduce epsilon; confirm `requires_grad=False` on all backbone params |
| Emotions bleed into each other (angry sounds sad) | Insufficient training data; too few epochs | Add more per-emotion utterances (300+ recommended); train for 5 full epochs |
| Speaker identity changes | Steering vector too large; wrong injection point | Reduce alpha; ensure steering is applied after speaker conditioning |
| Training loss doesn't decrease | Learning rate too low; gradient not flowing through hook | Try lr=5e-4; verify the hook returns the modified tensor (not in-place) |
| CUDA out of memory during training | Projection matrix d x d is large for big hidden dims | Use a low-rank factorization: W_e = A_e @ B_e where A is (d, r) and B is (r, d) with r << d |

## Limitations

- **Emotion vocabulary is fixed at training time.** You cannot steer toward an emotion that has no learned W_e. Adding a new emotion requires training a new projection matrix (though existing ones stay frozen).
- **Depends on a capable base TTS model.** If the backbone cannot produce expressive speech at all, steering its hidden states will not create emotion from nothing. The base model must have latent emotion capacity.
- **Evaluated primarily on English (ESD dataset).** Cross-lingual emotion transfer is not validated. Tonal languages may require different epsilon tuning.
- **Intensity control via alpha is unbounded.** Very high alpha values (>5) will produce artifacts. There is no learned upper bound -- users must tune this empirically.
- **Single-emotion per utterance.** The framework steers toward one emotion per generation. Mixed emotions within a single utterance (e.g., starting sad and ending angry) require segment-level steering, which is not covered.

## Reference

**Paper:** [EmoShift: Lightweight Activation Steering for Enhanced Emotion-Aware Speech Synthesis](https://arxiv.org/abs/2601.22873v1) (ICASSP 2026)
**Key takeaway:** Emotions are directional offsets in TTS latent space. A small per-emotion projection matrix (h @ W_e), added to hidden states with a scaling factor of 0.001, achieves better emotion control than full fine-tuning at 1/30 the parameter cost. Look for Section 3 (EmoSteer layer formulation) and Table 1 (comparison against baselines).