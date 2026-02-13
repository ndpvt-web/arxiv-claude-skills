---
name: "ecg-r1-protocol-guided-modality-agnostic-mllm"
description: "Build protocol-guided medical AI interpretation pipelines with structured diagnostic reasoning, modality-robust architectures, and evidence-grounded reward models. Triggers: 'build ECG interpretation pipeline', 'protocol-guided medical AI', 'implement diagnostic reasoning chain', 'medical MLLM with evidence rewards', 'modality dropout training', 'structured clinical interpretation system'"
---

# ECG-R1: Protocol-Guided, Modality-Agnostic Medical Interpretation Systems

This skill teaches Claude to build reliable medical signal interpretation systems using the ECG-R1 methodology: a three-part framework combining (1) protocol-guided instruction data generation that grounds AI outputs in measurable clinical features and quantitative thresholds, (2) interleaved modality dropout for robustness when input modalities are missing, and (3) reinforcement learning with diagnostic evidence rewards that penalize hallucination and reward clinically grounded reasoning. While the paper targets ECG interpretation, the architectural patterns generalize to any domain where structured diagnostic protocols exist and hallucination is dangerous.

## When to Use

- When the user asks to build an AI system that interprets medical signals (ECG, EEG, EMG) and needs clinically grounded outputs rather than plausible-sounding hallucinations
- When building a multimodal system that must remain robust when one input modality is missing (e.g., signal data available but image unavailable, or vice versa)
- When implementing a training pipeline that uses reinforcement learning with domain-specific evidence rewards rather than generic preference optimization
- When constructing structured instruction datasets from clinical protocols, diagnostic guidelines, or any domain with formal decision logic
- When evaluating medical AI outputs for hallucination by checking whether diagnostic claims are supported by measurable evidence in the input
- When designing a chain-of-thought reasoning system that follows a fixed diagnostic protocol with quantitative thresholds

## Key Technique

**Protocol-Guided Instruction Data Generation.** Instead of having a teacher model generate free-form interpretations (which propagates hallucinations), ECG-R1 encodes a 17-step clinical protocol into the data generation prompt. This protocol is organized into five phases: (1) Technical QA, Rate & Rhythm, (2) Axis, Conduction & Intervals, (3) Voltage & Hypertrophy, (4) Ischemia & Infarction, (5) Electrolytes & QT. Each phase specifies exact leads to examine, quantitative thresholds (e.g., PR interval 0.12-0.20s, ST elevation >=1mm in limb leads, SV1+RV6 >35mm for LVH), and differential exclusion logic. The generated corpus forces the model to cite specific lead measurements rather than making vague claims.

**Interleaved Modality Dropout.** During training, input modalities (ECG time-series signal, ECG image) are randomly dropped at rates from 25% to 100%, forcing the model to produce consistent interpretations regardless of which modality is available. This is not standard dropout on hidden units -- it operates at the input modality level, creating a model that gracefully degrades rather than failing when a modality is absent. At inference, the model accepts signal-only, image-only, or both.

**RL with Diagnostic Evidence Rewards.** Three reward signals train the model: (1) `KeyDiagnosticEvidenceORM` checks whether the model's chain-of-thought contains verbatim evidence tokens extracted from ground-truth interpretations across six clinical reasoning steps (max 4 words per finding, max 3 findings per category); (2) `FormatRewardORM` validates structural compliance (presence of `<think>` reasoning tags); (3) `DiagnosisAccuracyORM` computes Jaccard similarity between predicted and ground-truth diagnostic labels. This composite reward grounds the RL optimization in verifiable clinical evidence rather than stylistic fluency.

## Step-by-Step Workflow

1. **Define the diagnostic protocol as a structured schema.** Enumerate every phase of interpretation with explicit steps, the specific input features to examine at each step, and quantitative thresholds that determine normal vs. abnormal findings. Encode this as a prompt template or JSON schema -- not as free-form instructions.

    ```python
    PROTOCOL = {
        "phases": [
            {
                "name": "Technical QA, Rate & Rhythm",
                "steps": [
                    "Check baseline quality, calibration (10mm/mV), speed (25mm/s)",
                    "Calculate heart rate from RR intervals",
                    "Identify rhythm via P-wave morphology in leads II, V1"
                ],
                "thresholds": {"hr_normal": [60, 100], "pr_interval": [0.12, 0.20]}
            },
            # ... additional phases
        ]
    }
    ```

2. **Generate protocol-guided instruction data.** Use a capable LLM as a teacher, but constrain its generation with the protocol. The prompt must require lead-by-lead examination, citation of specific measurements, and differential exclusion (stating what was ruled out and why). Output should be structured narrative paragraphs, not bullet lists, to train natural clinical language.

3. **Extract diagnostic evidence tokens from ground-truth interpretations.** For each training example, apply a six-step clinical framework to extract short verbatim phrases (max 4 words each, max 3 per category) that constitute the key diagnostic evidence. Store these as substring-match targets for the reward model. Preserve exact phrasing including negatives like "no ST elevation."

    ```json
    {
        "evidence": {
            "rhythm": ["sinus rhythm", "regular rate"],
            "conduction": ["normal PR interval"],
            "ischemia": ["no ST elevation", "no T-wave inversion"],
            "diagnosis": ["normal sinus rhythm"]
        }
    }
    ```

4. **Build the modality-decoupled encoder architecture.** Implement separate encoding paths for each input modality (e.g., a 1D convolutional or transformer encoder for time-series signals, a vision encoder for images). Feed encoded representations as tokens into the LLM backbone via projection layers. The encoders must be independently operational.

5. **Implement interleaved modality dropout in the training loop.** During each training step, randomly mask entire modality inputs with configurable dropout rates. Start with lower rates (25%) and increase. The key constraint: the model must produce the same diagnostic output regardless of which modality combination is presented.

    ```python
    def apply_modality_dropout(signal_tokens, image_tokens, dropout_rate=0.5):
        if random.random() < dropout_rate:
            if random.random() < 0.5:
                signal_tokens = torch.zeros_like(signal_tokens)
            else:
                image_tokens = torch.zeros_like(image_tokens)
        return signal_tokens, image_tokens
    ```

6. **Implement the three-component reward model for RL.** Code three reward functions: (a) `KeyDiagnosticEvidenceORM` -- substring-match the model's reasoning against extracted evidence tokens across all six clinical categories; (b) `FormatRewardORM` -- verify structural compliance with the expected reasoning format; (c) `DiagnosisAccuracyORM` -- compute Jaccard similarity between predicted and ground-truth diagnostic label sets.

    ```python
    def evidence_reward(generated_text, evidence_tokens):
        """Reward based on verbatim evidence presence in reasoning."""
        score = 0
        for category, tokens in evidence_tokens.items():
            for token in tokens:
                if token.lower() in generated_text.lower():
                    score += 1
        return score / max(sum(len(v) for v in evidence_tokens.values()), 1)

    def diagnosis_accuracy_reward(predicted_labels, ground_truth_labels):
        """Jaccard similarity between diagnostic label sets."""
        pred = set(predicted_labels)
        gt = set(ground_truth_labels)
        if not pred and not gt:
            return 1.0
        return len(pred & gt) / len(pred | gt)
    ```

7. **Run supervised fine-tuning (SFT) on the protocol-guided corpus.** Train the multimodal LLM on the generated instruction data with modality dropout active. Use the protocol-structured data to establish the model's baseline reasoning pattern before RL refinement.

8. **Run RL training with the composite reward.** Use DAPO (or similar policy optimization) with rollout-based generation. The model generates candidate interpretations, scores them against the three reward components, and updates toward higher-evidence, higher-accuracy outputs. Weight evidence rewards heavily to combat hallucination.

9. **Evaluate with grounded interpretation metrics.** Do not rely solely on text similarity (BLEU, ROUGE). Evaluate whether each diagnostic claim in the output is supported by a specific measurable finding cited in the reasoning. Use an LLM-as-judge or substring matching against evidence tokens to compute a "grounding rate."

10. **Test modality robustness explicitly.** Run inference with signal-only, image-only, and both-modalities configurations. The diagnostic accuracy should degrade gracefully (not catastrophically) when a modality is absent. Report per-modality accuracy gaps.

## Concrete Examples

**Example 1: Building a protocol-guided ECG interpretation training pipeline**

User: "I need to build a training data generation pipeline for an ECG interpretation model that doesn't hallucinate. The model should cite specific lead findings."

Approach:
1. Define the 5-phase, 17-step ECG interpretation protocol as a structured prompt template
2. For each ECG record in the training set, extract computed features (heart rate, intervals, axis, ST deviations) from the signal processing pipeline
3. Pass the features + protocol template to a teacher LLM with instructions to produce narrative interpretation grounded in the specific measurements
4. Validate each generated interpretation: reject any that contain diagnostic claims without citing a specific lead and measurement
5. Extract evidence tokens (max 4 words, max 3 per category) for RL reward computation

Output structure:
```python
# Protocol-guided generation prompt (abbreviated)
SYSTEM_PROMPT = """You are a cardiologist writing an ECG interpretation.
Follow this exact protocol:

Phase 1 - Technical QA, Rate & Rhythm:
- State calibration and speed settings
- Calculate heart rate from RR interval (normal: 60-100 bpm)
- Identify rhythm by examining P-waves in leads II and V1

Phase 2 - Axis, Conduction & Intervals:
- Determine axis from leads I, II, aVF (normal: -30 to +90 degrees)
- Measure PR interval (normal: 0.12-0.20s; >0.20s = first-degree AV block)
- Measure QRS duration (normal: <0.12s; >=0.12s = bundle branch block)

[... phases 3-5 ...]

Rules:
- Every diagnostic claim MUST cite the specific lead(s) and measurement
- State what was ruled out and why (differential exclusion)
- Write in narrative paragraphs, not bullet points
- If a finding is normal, state it explicitly with the measured value
"""
```

**Example 2: Implementing modality dropout for a multimodal medical AI**

User: "My medical imaging model fails completely when the CT scan is missing and only the lab report is available. How do I make it robust to missing modalities?"

Approach:
1. Implement separate encoders for each modality (CT image encoder, lab report text encoder)
2. Add modality dropout to the training loop that randomly zeros out one modality's encoded representation
3. Use a curriculum: start at 25% dropout, increase to 75% as training progresses
4. Add a consistency loss: when both modalities are present, the output should match what the model produces with either modality alone

```python
class ModalityDropoutTrainer:
    def __init__(self, model, dropout_schedule):
        self.model = model
        self.dropout_schedule = dropout_schedule  # e.g., {0: 0.25, 5000: 0.5, 10000: 0.75}

    def get_dropout_rate(self, step):
        rate = 0.25
        for threshold, new_rate in sorted(self.dropout_schedule.items()):
            if step >= threshold:
                rate = new_rate
        return rate

    def train_step(self, ct_features, lab_features, labels, step):
        rate = self.get_dropout_rate(step)

        # Randomly drop one modality
        if random.random() < rate:
            choice = random.choice(["ct", "lab"])
            if choice == "ct":
                ct_features = torch.zeros_like(ct_features)
            else:
                lab_features = torch.zeros_like(lab_features)

        output = self.model(ct_features, lab_features)
        loss = self.compute_loss(output, labels)
        return loss
```

**Example 3: Building an evidence-grounded reward model for medical RL**

User: "I'm fine-tuning a medical report generation model with RLHF but it keeps producing fluent-sounding but factually wrong reports. How do I fix the reward?"

Approach:
1. Replace generic preference rewards with domain-specific evidence rewards
2. Extract key diagnostic evidence from ground-truth reports as short verbatim phrases
3. Implement three reward components: evidence presence, format compliance, diagnostic label accuracy
4. Weight evidence rewards highest to prioritize factual grounding over fluency

```python
class MedicalEvidenceRewardModel:
    def __init__(self, evidence_weight=0.5, format_weight=0.1, accuracy_weight=0.4):
        self.weights = {
            "evidence": evidence_weight,
            "format": format_weight,
            "accuracy": accuracy_weight
        }

    def compute_reward(self, generated_text, evidence_tokens, gt_labels):
        # 1. Evidence reward: are key findings mentioned verbatim?
        evidence_score = self._evidence_match(generated_text, evidence_tokens)

        # 2. Format reward: does output follow required structure?
        format_score = 1.0 if "<think>" in generated_text else 0.0

        # 3. Diagnosis accuracy: Jaccard similarity of label sets
        pred_labels = self._extract_labels(generated_text)
        accuracy_score = self._jaccard(pred_labels, gt_labels)

        return (self.weights["evidence"] * evidence_score +
                self.weights["format"] * format_score +
                self.weights["accuracy"] * accuracy_score)

    def _evidence_match(self, text, evidence_tokens):
        total, matched = 0, 0
        for category, tokens in evidence_tokens.items():
            for token in tokens:
                total += 1
                if token.lower() in text.lower():
                    matched += 1
        return matched / max(total, 1)

    def _jaccard(self, pred, gt):
        pred, gt = set(pred), set(gt)
        if not pred and not gt:
            return 1.0
        return len(pred & gt) / len(pred | gt)
```

## Best Practices

- **Do:** Encode domain protocols as explicit, machine-readable schemas with quantitative thresholds -- never rely on the model to "know" clinical criteria from pretraining
- **Do:** Extract evidence tokens as short verbatim phrases (max 4 words) for substring matching in reward models; longer phrases cause brittle matching
- **Do:** Preserve negative findings ("no ST elevation") in evidence extraction -- the absence of pathology is diagnostic evidence too
- **Do:** Test modality robustness explicitly at evaluation time across all combinations (modality A only, modality B only, both) and report the accuracy gap
- **Avoid:** Using free-form teacher model generation without protocol constraints -- this is the primary source of hallucinated medical AI outputs
- **Avoid:** Applying standard dropout (on hidden units) as a substitute for modality-level dropout -- they solve different problems; modality dropout teaches the model to function without an entire input stream
- **Avoid:** Rewarding only final diagnostic accuracy without evidence grounding -- a model can guess the right diagnosis for the wrong reasons, which is dangerous in deployment

## Error Handling

- **Protocol violations in generated data:** Validate each generated training example against the protocol schema. Reject examples where diagnostic claims lack specific lead/measurement citations. Automate this with regex or LLM-based validation.
- **Evidence token extraction failures:** When ground-truth interpretations use non-standard phrasing, the 4-word max constraint may fragment meaningful findings. Maintain a synonym mapping for common clinical terms but never alter the source text for reward matching.
- **Modality dropout causing training instability:** If loss spikes when dropout rate increases, use a slower curriculum schedule. Monitor per-modality performance during training to ensure neither modality path is collapsing.
- **Reward hacking in RL:** The model may learn to insert evidence tokens without genuine reasoning. Mitigate by requiring evidence tokens to appear within the `<think>` reasoning section, not just the final output, and by using the Jaccard label accuracy reward as a complementary signal.

## Limitations

- The protocol-guided approach requires a well-defined, step-by-step diagnostic protocol for the target domain. Domains without formalized diagnostic criteria (e.g., subjective symptom assessment) cannot directly use this method.
- Interleaved modality dropout assumes modalities are partially redundant. If modalities carry entirely non-overlapping information, dropping one will necessarily degrade performance and the model cannot compensate.
- Evidence-based substring matching rewards are fragile to paraphrasing. The 4-word max constraint mitigates this but does not eliminate it. Consider embedding-based similarity as a fallback for domains with high linguistic variability.
- The three-stage pipeline (SFT then rollout then DAPO) requires substantial compute. For smaller teams, the protocol-guided SFT alone provides significant hallucination reduction without the RL stage.

## Reference

**Paper:** [ECG-R1: Protocol-Guided and Modality-Agnostic MLLM for Reliable ECG Interpretation](https://arxiv.org/abs/2602.04279v1) -- Look for the 17-step clinical protocol in Appendix, the modality dropout rates in the training configuration, and the three-component reward formulation (KeyDiagnosticEvidenceORM, FormatRewardORM, DiagnosisAccuracyORM) in the RL section. Code: [github.com/PKUDigitalHealth/ECG-R1](https://github.com/PKUDigitalHealth/ECG-R1).