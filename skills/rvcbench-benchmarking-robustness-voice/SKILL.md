---
name: "rvcbench-benchmarking-robustness-voice"
description: "Benchmark and harden voice cloning systems against real-world degradation using the RVCBench framework. Evaluates VC model robustness across input noise, compression, cross-lingual scenarios, and adversarial perturbations. Triggers: 'benchmark voice cloning robustness', 'evaluate TTS model robustness', 'test voice cloning against noise', 'set up RVCBench pipeline', 'harden voice cloning system', 'compare VC model robustness'"
---

This skill enables Claude to deploy and operate the RVCBench benchmarking framework for systematically evaluating the robustness of modern voice cloning (VC) models. RVCBench tests 10 robustness dimensions across four perturbation categories -- input variation, generation challenges, output post-processing, and adversarial perturbations -- against 11 representative VC architectures including XTTS, CosyVoice, F5-TTS, SPARK, StyleTTS2, and others. Claude can use this skill to set up the benchmark, configure evaluation runs, interpret robustness metrics (SIM-O, WER, PESQ, UTMOS), and recommend hardening strategies based on empirically identified failure modes.

## When to Use

- When the user wants to evaluate how a voice cloning or TTS model handles noisy, compressed, or degraded reference audio
- When comparing multiple VC models (e.g., CosyVoice vs. F5-TTS vs. XTTS) on robustness criteria before selecting one for production
- When setting up a CI/CD quality gate that tests synthesized speech under realistic deployment conditions (codec compression, background noise, sample rate conversion)
- When the user needs to apply audio protection methods (SafeSpeech, Enkidu, Gaussian noise, spectral perturbations) and measure their effect on cloning fidelity
- When investigating cross-lingual or long-context stability of a voice cloning pipeline
- When the user asks to reproduce or extend the RVCBench evaluation from the paper on custom data or new VC models

## Key Technique

RVCBench introduces a four-category robustness taxonomy that covers the full voice cloning pipeline. **Input variations** test how reference audio degradation (background noise at varying SNR, codec compression, sample rate conversion, reverberation) affects speaker similarity and transcription accuracy. **Generation challenges** probe model stability under long-context synthesis and cross-lingual prompts where the target language differs from the reference. **Output post-processing** applies real-world transformations (MP3 compression, bandwidth limiting, spectral filtering) to generated audio and measures quality retention. **Adversarial perturbations** use both passive noise injection and proactive protection methods (SafeSpeech, Enkidu, AntiFake) to stress-test identity preservation.

The benchmark pipeline operates in three stages: (1) baseline generation from clean references, (2) systematic perturbation application at controlled intensity levels, and (3) metric evaluation comparing perturbed outputs against the clean baseline. The Hydra-based configuration system lets you compose experiments by mixing protection methods, VC models, datasets, and denoising strategies through YAML overrides. Each run produces structured `metrics.json` output containing SIM-O (speaker similarity to original), WER (word error rate via Whisper), PESQ (perceptual speech quality), and UTMOS (neural MOS predictor) scores.

The key empirical finding is that **diffusion-based models generally outperform autoregressive codec-token language models on robustness**, and that content preservation (WER) degrades faster than speaker identity (SIM-O) under most perturbations. CosyVoice and SPARK show the strongest overall resilience, while models like GLM-4-Voice and VibeVoice are significantly more sensitive to acoustic modifications and noise. This knowledge directly informs model selection and hardening priorities.

## Step-by-Step Workflow

1. **Clone and install RVCBench.** Set up the Python environment with PyTorch, torchaudio, and hydra-core, then clone the repository and install dependencies:
   ```bash
   git clone https://github.com/Nanboy-Ronan/RVCBench.git
   cd RVCBench
   pip install -r requirements.txt
   ```

2. **Download model checkpoints and dataset manifests.** Retrieve the required VC model weights and the LibriTTS/VCTK evaluation splits. For SafeSpeech protection, run its dedicated download script:
   ```bash
   python src/protection/safespeech/original_code/download_models.py
   ```

3. **Select or create a Hydra configuration.** Configs follow the naming pattern `[protection_method]_on_[dataset].yaml`. For a clean baseline evaluation, use `ozspeech_ots.yaml`. For protection benchmarking, use configs like `safespeech_on_libritts.yaml` or `enkidu_on_libritts.yaml`. Override parameters on the command line:
   ```bash
   python run_vc.py --config-name ozspeech_ots \
     vc.max_samples=50 \
     dataset.root_path=/path/to/libritts
   ```

4. **Run baseline voice cloning** on clean reference audio to establish unperturbed performance scores using `run_vc.py`. This generates audio and computes SIM-O, WER, PESQ, and UTMOS baselines.

5. **Apply perturbations and re-evaluate.** Use `run_protect.py` to apply protection/perturbation methods to reference audio, then `run_vc_protect.py` to clone from the perturbed references. Compare the resulting `metrics.json` against the clean baseline.

6. **Optionally run denoising recovery.** Use `run_denoiser.py` to apply denoising to protected audio before cloning, measuring whether post-processing can circumvent protections:
   ```bash
   python run_denoiser.py --config-name safespeech_on_libritts
   ```

7. **Aggregate and compare results.** Parse `results/<run_name>/<timestamp>/metrics.json` files across runs. Compute delta-SIM-O and delta-WER to quantify robustness degradation per perturbation type and per model.

8. **Interpret results using the robustness taxonomy.** Map each metric degradation to its perturbation category (input/generation/output/adversarial) to identify which pipeline stage is the weakest link for your chosen VC model.

9. **Recommend hardening strategies.** Based on identified failure modes, suggest targeted mitigations: audio preprocessing (denoising, normalization) for input-sensitive models, prompt engineering for generation-challenged models, or post-processing-aware fine-tuning for output-fragile models.

10. **Integrate into CI/CD.** Set acceptance thresholds (e.g., SIM-O > 0.75, WER < 15%) and run the benchmark as a regression test whenever the VC model or preprocessing pipeline changes.

## Concrete Examples

**Example 1: Comparing VC models for production deployment**

User: "I need to pick between CosyVoice and XTTS for a voice cloning feature that will receive phone-quality audio. Which handles noisy input better?"

Approach:
1. Set up RVCBench with both models configured
2. Run clean baseline for both: `python run_vc.py --config-name ozspeech_ots vc.model=cosyvoice` and `vc.model=xtts`
3. Apply phone-quality degradation (8kHz sample rate, codec compression, background noise at 15dB SNR) via the protection pipeline
4. Run `run_vc_protect.py` for both models against the degraded references
5. Compare delta-SIM-O and delta-WER between clean and degraded conditions

Output:
```json
{
  "model": "cosyvoice",
  "clean": {"sim_o": 0.82, "wer": 8.3, "pesq": 3.4, "utmos": 4.1},
  "phone_degraded": {"sim_o": 0.74, "wer": 12.1, "pesq": 2.8, "utmos": 3.5},
  "delta_sim_o": -0.08,
  "delta_wer": +3.8
}
{
  "model": "xtts",
  "clean": {"sim_o": 0.79, "wer": 9.1, "pesq": 3.2, "utmos": 3.9},
  "phone_degraded": {"sim_o": 0.63, "wer": 18.7, "pesq": 2.2, "utmos": 2.9},
  "delta_sim_o": -0.16,
  "delta_wer": +9.6
}
```
Recommendation: CosyVoice retains speaker identity and content accuracy far better under phone-quality degradation. XTTS loses nearly twice as much speaker similarity and WER nearly doubles.

**Example 2: Testing audio protection effectiveness**

User: "I want to protect my voice samples from unauthorized cloning. How effective is SafeSpeech?"

Approach:
1. Run clean cloning baseline across target VC models: `python run_vc.py --config-name ozspeech_ots`
2. Apply SafeSpeech protection: `python run_protect.py --config-name safespeech_on_libritts`
3. Attempt cloning from protected audio: `python run_vc_protect.py --config-name safespeech_on_libritts`
4. Test denoising bypass: `python run_denoiser.py --config-name safespeech_on_libritts`
5. Compare SIM-O scores across clean, protected, and denoised-then-cloned conditions

Output:
```
Protection effectiveness (SIM-O reduction from clean baseline):
  SafeSpeech on CosyVoice:  0.82 -> 0.41 (50% reduction) -- strong protection
  SafeSpeech on F5-TTS:     0.78 -> 0.52 (33% reduction) -- moderate protection
  SafeSpeech on XTTS:       0.79 -> 0.38 (52% reduction) -- strong protection

After denoising recovery attempt:
  CosyVoice:  0.41 -> 0.58 (partial recovery)
  F5-TTS:     0.52 -> 0.64 (significant recovery)
  XTTS:       0.38 -> 0.51 (partial recovery)
```

**Example 3: Adding a custom VC model to the benchmark**

User: "I fine-tuned a custom TTS model and want to benchmark its robustness with RVCBench."

Approach:
1. Create a new adversary module under `src/adversary/` implementing the model's inference API with the standard RVCBench interface (accepts reference audio path and text prompt, returns generated audio tensor)
2. Add a Hydra config under `configs/model/custom_model.yaml` specifying checkpoint path, sample rate, and model-specific parameters
3. Create a top-level experiment config `configs/custom_on_libritts.yaml` composing the new model with the LibriTTS dataset
4. Run the full benchmark suite:
   ```bash
   python run_vc.py --config-name custom_on_libritts
   python run_protect.py --config-name custom_on_libritts
   python run_vc_protect.py --config-name custom_on_libritts
   ```
5. Compare `metrics.json` against published baselines for the 11 built-in models

## Best Practices

- **Do:** Always establish a clean baseline before running perturbation experiments. Robustness is measured as delta from baseline, not absolute score.
- **Do:** Test at multiple perturbation intensities (e.g., SNR at 5dB, 10dB, 15dB, 20dB) to map the degradation curve rather than testing a single point.
- **Do:** Evaluate both speaker similarity (SIM-O) and content accuracy (WER) together. A model can preserve voice identity while garbling words, or vice versa.
- **Do:** Use the Hydra override system for experiment management rather than editing YAML files directly. This keeps configurations reproducible and composable.
- **Avoid:** Drawing conclusions from small sample sizes. Use at least 50+ utterances per condition (`vc.max_samples=50` minimum, full benchmark uses 14,370).
- **Avoid:** Assuming robustness to one perturbation type implies robustness to others. The paper shows that input noise robustness does not predict post-processing robustness.

## Error Handling

- **CUDA out of memory:** Reduce `vc.max_samples` or run with `vc.batch_size=1`. Some models (GLM-4-Voice, HiggsAudio) require significant VRAM.
- **Missing model checkpoints:** The benchmark will fail silently or produce zeros if checkpoints are not downloaded. Verify all paths in `configs/model/` before running.
- **Sample rate mismatch:** Different VC models expect different sample rates (16kHz, 22.05kHz, 24kHz). The pipeline handles resampling, but custom models must declare their expected rate in the config.
- **Hydra config composition errors:** If overrides conflict, Hydra throws a `ConfigCompositionException`. Use `--cfg job` to inspect the resolved config before running.
- **Empty metrics.json:** Usually indicates the dataset path is wrong or the manifest file is missing. Check `dataset.root_path` and verify the data directory contains the expected WAV files.

## Limitations

- The benchmark evaluates English-centric robustness primarily (LibriTTS, VCTK). Cross-lingual findings are limited to the languages represented in the evaluation set.
- RVCBench measures robustness of inference only -- it does not evaluate training-time robustness or fine-tuning stability.
- Adversarial perturbation methods (SafeSpeech, Enkidu) represent the state of the art as of early 2026. Newer adaptive attacks or defenses may not be captured.
- The benchmark does not test real-time streaming robustness, only offline batch processing. Latency-sensitive deployments need additional testing.
- Speaker similarity (SIM-O) relies on a specific speaker verification model; results may differ with alternative verification backends.

## Reference

**Paper:** [RVCBench: Benchmarking the Robustness of Voice Cloning Across Modern Audio Generation Models](https://arxiv.org/abs/2602.00443v1) (Liao et al., 2026). Look for Table 2-4 for per-model robustness scores across all 10 tasks, and Section 4 for the detailed analysis of failure modes by perturbation category.

**Code:** [github.com/Nanboy-Ronan/RVCBench](https://github.com/Nanboy-Ronan/RVCBench) -- Hydra-based pipeline with configs for all 11 models and 7 protection methods.