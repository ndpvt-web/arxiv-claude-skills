---
name: "deepasmr-llm-based-zero-shot-asmr"
description: "Build zero-shot ASMR speech generation systems using a two-stage LLM + flow-matching pipeline that separates speaking style from speaker timbre via discrete token factorization. Use when: 'generate ASMR speech from normal voice', 'build whispery TTS pipeline', 'zero-shot voice style transfer system', 'implement speech style factorization with discrete tokens', 'create ASMR voice cloning from read speech', 'design two-stage LLM speech synthesis architecture'."
---

# DeepASMR: Zero-Shot ASMR Speech Generation via LLM + Flow-Matching Pipeline

This skill enables Claude to design, implement, and debug zero-shot ASMR speech generation systems based on the DeepASMR architecture. The core technique uses discrete speech tokens (trained with ASR objectives and Finite Scalar Quantization) to achieve a soft factorization of speaking style from speaker timbre, then reconstructs full audio through a two-stage pipeline: an autoregressive LLM that encodes content and style into token sequences, followed by a conditional flow-matching Transformer decoder that reconstructs speaker-specific mel-spectrograms. This allows generating high-fidelity ASMR whisper speech for any speaker given only a short clip of their normal read-style voice — no whispered training data from the target speaker required.

## When to Use

- When the user wants to build a TTS system that can generate ASMR/whisper speech from a normal voice reference clip
- When implementing zero-shot voice style transfer where style (e.g., whisper, soft-spoken) must transfer independently of speaker identity
- When designing a two-stage speech synthesis pipeline with an LLM token predictor and a flow-matching acoustic decoder
- When building a speech tokenization system that needs to disentangle style attributes from speaker timbre
- When constructing a training pipeline for expressive speech generation using large-scale curated ASMR data
- When evaluating generated speech quality using a multi-axis protocol (objective metrics, LLM-based scoring, unvoiced ratio analysis)
- When the user needs to implement similarity-based prompt retrieval (Virtual Pool) for cross-style voice cloning

## Key Technique

**Discrete Token Factorization.** DeepASMR's central insight is that S3 tokens — discrete speech representations extracted by a tokenizer trained with an ASR (Automatic Speech Recognition) objective using Finite Scalar Quantization (FSQ) — naturally create a style-dominant embedding space. In this space, ASMR and normal speech form distinct clusters separated by a hyperplane, while speaker identity persists as a weaker, residual signal within each cluster. Concretely, a speaker classifier trained on S3 hidden states achieves 86.4% accuracy (vs. 2.8% random baseline), confirming timbre information is present but subordinate to style. This soft factorization is what makes the two-stage architecture viable: the LLM handles the style-dominant content tokens, while the acoustic decoder recovers the fine-grained timbre from the speaker's reference audio.

**Two-Stage Pipeline.** Stage 1 uses a decoder-only LLM (initialized from Qwen2.5-0.5B) to autoregressively predict target S3 semantic tokens conditioned on text input and a style task prompt. Stage 2 uses a flow-matching Transformer decoder (24 layers, 16 attention heads, 1024-dim embeddings with U-Net skip connections) conditioned on the concatenated token sequence `[z_spk, z_pred]` and the speaker prompt's mel-spectrogram features. The flow-matching loss is `L_CFM = E[||v_t(x_t, z) - (x_1 - (1 - sigma_min) * x_0)||^2]`, where x_1 is the ground-truth mel and x_0 is Gaussian noise. A HiFi-GAN vocoder converts the output mel-spectrogram to a waveform. For cross-style generation (normal voice in, ASMR out), the system uses cosine-similarity retrieval over WeSpeaker embeddings to find the closest ASMR speaker in a Virtual Pool, providing the style prompt while the original speaker's mel features supply timbre.

**Training Regimen.** Stage 1 pre-trains the LLM on 200K hours of general speech data for 250K steps, then fine-tunes on DeepASMR-DB (674.5 hours, 35 speakers, 40K+ samples across English and Mandarin). Stage 2 trains the acoustic model for 40 epochs at learning rate 1e-5 with Adam optimizer, using 23K frames per batch with gradient accumulation of 2 on 8x A100 GPUs.

## Step-by-Step Workflow

1. **Set up the speech tokenizer.** Implement or integrate an S3-style tokenizer trained with ASR objectives and Finite Scalar Quantization (FSQ). The tokenizer must produce discrete tokens that capture phonetic content and speaking style while leaving fine timbre detail to downstream reconstruction. If starting from scratch, train on a large ASR corpus; otherwise, adapt an existing speech codec (e.g., SpeechTokenizer, CosyVoice's tokenizer) with FSQ quantization.

2. **Prepare the ASMR training corpus.** Curate paired normal/ASMR speech data following DeepASMR-DB's four-stage pipeline: (a) filter source videos by ASMR topic keywords, (b) apply quality control to reject noisy or music-heavy segments, (c) acquire and segment audio clips, (d) transcribe using a fast transcription API. Target at least several hundred hours across multiple speakers. Store as `{speaker_id, style_label, transcript, audio_path}` records.

3. **Build the LLM content-style encoder (Stage 1).** Initialize a decoder-only Transformer (Qwen2.5-0.5B or similar small LLM). The input sequence consists of text tokens followed by a style task token (normal or ASMR), and the model autoregressively predicts target S3 semantic tokens. Pre-train on large-scale general speech data (200K+ hours), then fine-tune on the ASMR corpus for 10 epochs at learning rate 1e-4.

4. **Build the flow-matching acoustic decoder (Stage 2).** Implement a Transformer encoder with 24 layers, 16 attention heads, 1024-dim embeddings, and U-Net-style skip connections. The decoder is conditioned on: (a) the concatenated semantic token sequence `[z_spk, z_pred]` from Stage 1, and (b) fine-grained mel-spectrogram features extracted from the speaker's reference audio. Use the conditional flow-matching loss `L_CFM` with Gaussian noise as x_0 and ground-truth mel as x_1. Train for 40 epochs at learning rate 1e-5 with Adam.

5. **Implement the Virtual Pool for cross-style inference.** Build a retrieval index of ASMR speaker embeddings using WeSpeaker. At inference time, when the input is a normal-speech reference clip: (a) extract the speaker embedding, (b) find the closest ASMR speaker via cosine similarity, (c) use the matched ASMR speaker's audio as the style prompt while preserving the original speaker's mel features for timbre conditioning.

6. **Wire the inference pipeline.** Chain the components: text input + style tag -> LLM predicts S3 tokens -> flow-matching decoder generates mel-spectrogram conditioned on tokens + speaker mel features -> HiFi-GAN vocoder produces the waveform. For cross-style (normal->ASMR), insert the Virtual Pool retrieval step before the LLM stage.

7. **Add iterative refinement (optional).** For higher quality, implement 2-3 iterative inference passes where each pass feeds the previous output back as input, progressively refining ASMR characteristics.

8. **Implement the evaluation protocol.** Measure: (a) WER/CER for intelligibility via an ASR system, (b) speaker similarity (SIM) using WavLM-Large cosine similarity between generated and reference audio, (c) LLM-based style score on a [-1, +1] scale (-1 = normal, +1 = ASMR) using a prompted language model, (d) Global Unvoiced Ratio (R_UV) — the percentage of unvoiced frames — as an ASMR-specific acoustic metric (higher R_UV indicates more whisper-like quality).

9. **Tune and validate across speaker/style combinations.** Test four configurations: ASMR->ASMR (intra-style), Normal->ASMR (cross-style, the key use case), ASMR->Normal, and Normal->Normal. Verify that cross-style Normal->ASMR achieves style scores above +0.5 and speaker SIM above 0.35.

10. **Package for deployment.** Export the LLM, acoustic decoder, vocoder, and Virtual Pool index as separate modules. The LLM and acoustic decoder can run independently, enabling flexible serving (e.g., batch token prediction on CPU, GPU-accelerated mel decoding). Provide a simple API: `generate_asmr(text, speaker_reference_audio, style="asmr") -> wav`.

## Concrete Examples

**Example 1: Building a Normal-to-ASMR Voice Cloning API**

User: "I want to build an API that takes a 10-second clip of someone's normal speaking voice and generates ASMR-style speech of any text in their voice."

Approach:
1. Set up the S3 tokenizer with FSQ quantization to extract discrete tokens from audio
2. Load the pre-trained LLM (Qwen2.5-0.5B backbone) fine-tuned on ASMR data
3. Load the flow-matching decoder (24-layer Transformer with U-Net skips)
4. Build a Virtual Pool index from your ASMR speaker corpus using WeSpeaker embeddings
5. At request time: extract speaker embedding from the 10s clip, retrieve closest ASMR voice from the pool, run LLM to predict ASMR S3 tokens, decode to mel via flow-matching conditioned on original speaker's mel features, vocode with HiFi-GAN

Output (Python pseudocode):
```python
class DeepASMRService:
    def __init__(self, llm_path, decoder_path, vocoder_path, pool_path):
        self.tokenizer = S3Tokenizer.load("s3_fsq_checkpoint")
        self.llm = AutoregressiveLLM.load(llm_path)  # Qwen2.5-0.5B fine-tuned
        self.decoder = FlowMatchingDecoder.load(decoder_path)  # 24L Transformer
        self.vocoder = HiFiGAN.load(vocoder_path)
        self.virtual_pool = VirtualPool.load(pool_path)  # WeSpeaker index

    def generate(self, text: str, speaker_ref: np.ndarray, style: str = "asmr"):
        # Extract speaker mel features for timbre conditioning
        speaker_mel = extract_mel(speaker_ref)
        speaker_emb = self.virtual_pool.extract_embedding(speaker_ref)

        # Cross-style: retrieve closest ASMR voice for style prompt
        if style == "asmr":
            style_prompt = self.virtual_pool.retrieve_closest(speaker_emb)
            z_spk = self.tokenizer.encode(style_prompt)
        else:
            z_spk = self.tokenizer.encode(speaker_ref)

        # Stage 1: LLM predicts semantic tokens
        z_pred = self.llm.generate(text, style_tag=style, prompt_tokens=z_spk)

        # Stage 2: Flow-matching decodes to mel
        z_concat = torch.cat([z_spk, z_pred], dim=-1)
        mel = self.decoder.sample(z_concat, speaker_mel, steps=32)

        # Vocoder
        waveform = self.vocoder(mel)
        return waveform
```

**Example 2: Implementing the Flow-Matching Decoder**

User: "How do I implement the flow-matching acoustic decoder from DeepASMR?"

Approach:
1. Build a 24-layer Transformer encoder with 16 heads and 1024-dim embeddings
2. Add U-Net-style skip connections (layer i connects to layer 24-i)
3. Condition on concatenated semantic tokens via cross-attention and speaker mel via FiLM or concatenation
4. Implement the conditional flow-matching training loop with the CFM loss

Output (PyTorch skeleton):
```python
class FlowMatchingDecoder(nn.Module):
    def __init__(self, d_model=1024, nhead=16, num_layers=24):
        super().__init__()
        self.layers = nn.ModuleList([
            TransformerBlock(d_model, nhead) for _ in range(num_layers)
        ])
        # U-Net skip connections: layer i <-> layer (num_layers - 1 - i)
        self.skip_projs = nn.ModuleList([
            nn.Linear(d_model * 2, d_model)
            for _ in range(num_layers // 2)
        ])
        self.time_embed = SinusoidalPositionEmbedding(d_model)
        self.token_proj = nn.Linear(token_dim, d_model)
        self.mel_proj = nn.Linear(mel_dim, d_model)

    def forward(self, x_t, t, z_concat, speaker_mel):
        t_emb = self.time_embed(t)
        cond = self.token_proj(z_concat)
        spk = self.mel_proj(speaker_mel)

        h = x_t + t_emb
        skip_cache = []
        mid = len(self.layers) // 2

        for i, layer in enumerate(self.layers):
            if i < mid:
                skip_cache.append(h)
            elif i >= mid:
                skip_idx = len(self.layers) - 1 - i
                h = self.skip_projs[skip_idx](torch.cat([h, skip_cache.pop()], -1))
            h = layer(h, context=cond, speaker=spk)
        return h  # predicted velocity v_t

def cfm_loss(model, x_1, z_concat, speaker_mel, sigma_min=1e-4):
    """Conditional flow-matching loss."""
    B = x_1.shape[0]
    t = torch.rand(B, 1, 1, device=x_1.device)
    x_0 = torch.randn_like(x_1)
    x_t = (1 - (1 - sigma_min) * t) * x_0 + t * x_1
    target = x_1 - (1 - sigma_min) * x_0
    v_pred = model(x_t, t.squeeze(), z_concat, speaker_mel)
    return F.mse_loss(v_pred, target)
```

**Example 3: Building the Evaluation Pipeline**

User: "How do I evaluate whether my ASMR generation system is working correctly?"

Approach:
1. Measure intelligibility: run generated audio through an ASR model (Whisper), compute WER/CER against input text
2. Measure speaker similarity: extract WavLM-Large embeddings from generated and reference audio, compute cosine similarity (target: > 0.35 for cross-style)
3. Measure style fidelity: prompt an LLM with audio features to score on [-1, +1] scale (target: > +0.5 for ASMR)
4. Measure acoustic ASMR quality: compute Global Unvoiced Ratio (R_UV) — the fraction of frames classified as unvoiced by a pitch tracker (ASMR speech has significantly higher R_UV than normal speech)

Output:
```python
def evaluate_asmr_generation(generated_wav, reference_wav, input_text):
    results = {}

    # 1. Intelligibility (WER)
    transcript = whisper_model.transcribe(generated_wav)
    results["wer"] = compute_wer(input_text, transcript)

    # 2. Speaker similarity
    gen_emb = wavlm_large.extract(generated_wav)
    ref_emb = wavlm_large.extract(reference_wav)
    results["speaker_sim"] = cosine_similarity(gen_emb, ref_emb)

    # 3. Unvoiced ratio (ASMR-specific acoustic measure)
    f0 = pitch_tracker.extract(generated_wav)
    results["unvoiced_ratio"] = (f0 == 0).sum() / len(f0)

    # 4. LLM-based style score
    mel_features = extract_mel_summary(generated_wav)
    results["style_score"] = llm_judge.score(
        mel_features,
        prompt="Rate this speech on a scale from -1 (normal) to +1 (ASMR)"
    )

    # Thresholds for Normal->ASMR cross-style generation
    assert results["wer"] < 0.10, "Intelligibility too low"
    assert results["speaker_sim"] > 0.35, "Speaker identity not preserved"
    assert results["style_score"] > 0.5, "Insufficient ASMR quality"
    assert results["unvoiced_ratio"] > 0.4, "Not enough unvoiced frames for ASMR"

    return results
```

## Best Practices

- **Do:** Use a tokenizer trained with ASR objectives (not a generic audio codec like EnCodec) for the discrete token stage — the ASR training objective is what creates the style-dominant factorization that makes the architecture work.
- **Do:** Pre-train the LLM on large-scale general speech data (100K+ hours) before fine-tuning on ASMR data. The 250K-step pre-training on 200K hours is critical for the LLM to learn robust token prediction.
- **Do:** Keep the two stages cleanly separated — the LLM should only predict discrete tokens, and the acoustic decoder should handle all continuous signal reconstruction. Mixing these responsibilities degrades both style control and audio quality.
- **Do:** Implement the Virtual Pool retrieval for cross-style inference. Without it, the model has no ASMR style reference when the input speaker only provides normal speech.
- **Avoid:** Training the flow-matching decoder with too high a learning rate. The paper uses 1e-5 for the acoustic model — higher rates destabilize the flow-matching convergence.
- **Avoid:** Using speaker embeddings alone for timbre conditioning in the decoder. The system requires fine-grained mel-spectrogram features from the reference audio, not just a single speaker embedding vector. The mel features carry the spectral detail needed for faithful timbre reconstruction.

## Error Handling

| Problem | Likely Cause | Fix |
|---|---|---|
| Generated ASMR sounds like normal speech | Virtual Pool retrieval returned a poor match, or style tag not properly conditioned | Check cosine similarity of retrieval; ensure style tag is correctly prepended to LLM input |
| Speaker identity lost in cross-style output | Mel conditioning too weak relative to style tokens | Increase weight of speaker mel features in the decoder's conditioning; verify mel extraction pipeline |
| High WER in generated speech | LLM generating incorrect semantic tokens | Check LLM fine-tuning convergence; verify S3 tokenizer alignment with training data |
| Metallic or buzzy audio artifacts | Flow-matching decoder under-trained or vocoder mismatch | Train decoder for more epochs (target 40); ensure HiFi-GAN is trained on matching mel configuration |
| Unvoiced ratio too low (not whispery enough) | Insufficient ASMR data in training set or tokenizer not capturing unvoiced characteristics | Augment training data with more whisper/ASMR samples; verify tokenizer preserves voicing distinction |

## Limitations

- **No public model weights.** As of the paper's publication, DeepASMR's trained checkpoints are not publicly released, so this architecture must be trained from scratch or adapted from available components (Qwen2.5 for LLM, custom flow-matching decoder).
- **Data-hungry.** The system requires hundreds of hours of curated ASMR data (DeepASMR-DB is 674.5 hours) and 200K hours of general speech for pre-training. Smaller datasets will produce noticeably worse results.
- **Cross-style quality ceiling.** Normal-to-ASMR (cross-style) generation achieves lower speaker similarity (~0.41 SIM) than intra-style generation, because the Virtual Pool retrieval introduces a timbre gap between the matched ASMR speaker and the target speaker.
- **Language coverage.** The system is validated only on English and Mandarin Chinese. Other languages would require additional ASMR training data and tokenizer adaptation.
- **Compute requirements.** Training requires 8x A100 GPUs. Inference is lighter but still requires GPU for the flow-matching decoder's iterative sampling.
- **Narrow style scope.** The architecture is designed specifically for ASMR (whisper/soft-spoken) style transfer. It does not generalize to arbitrary expressive styles (e.g., shouting, singing) without retraining.

## Reference

**Paper:** [DeepASMR: LLM-Based Zero-Shot ASMR Speech Generation for Anyone of Any Voice](https://arxiv.org/abs/2601.15596v1) (Zhang et al., 2026). Look for: Section 3 on discrete token factorization analysis, Section 4 on the two-stage architecture, and Section 5 on the DeepASMR-DB corpus curation pipeline. Demo samples at https://vivian556123.github.io/deepasmr-demo/.