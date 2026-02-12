---
name: "integrating-fine-grained-audio-visual-evidence"
description: >
  Build robust multimodal emotion reasoning pipelines using the SABER-LLM
  perceive-then-reason paradigm. Decomposes audio-visual analysis into structured
  evidence extraction (visual, acoustic, linguistic) before synthesizing causal
  emotional reasoning -- preventing unimodal dominance and hallucination.
  Trigger phrases: "analyze emotions in video", "multimodal emotion reasoning",
  "detect sarcasm in audio-visual content", "build emotion analysis pipeline",
  "perceive then reason over multimodal data", "structured evidence decomposition"
---

# Integrating Fine-Grained Audio-Visual Evidence for Multimodal Emotion Reasoning

This skill teaches Claude to build multimodal emotion analysis systems using the Structured Evidence Decomposition (SED) paradigm from SABER-LLM. Instead of letting a model fuse all modalities into a single prediction (which causes unimodal dominance and hallucination), SED enforces a strict **perceive-then-reason** pipeline: first extract visual evidence, then acoustic evidence conditioned on visual context, then synthesize causal reasoning grounded in both. This architecture is particularly effective for ambiguous or contradictory signals like sarcasm, where facial expression and vocal tone convey opposing sentiments.

## When to Use

- When building a video or audio-visual emotion analysis pipeline and the user needs structured, explainable outputs rather than flat labels
- When the user wants to detect sarcasm, irony, or mixed emotions where visual and acoustic signals conflict
- When designing annotation schemas for multimodal affective datasets (the six-dimensional schema applies broadly)
- When implementing preference optimization (DPO) for multimodal models that must maintain cross-modal consistency
- When a user asks to analyze a video clip's emotional content and needs grounded evidence for each modality
- When building LLM-based reasoning chains over video that must avoid hallucinating unsupported emotional states
- When creating evaluation benchmarks that test whether models can handle emotionally ambiguous or contradictory multimodal inputs

## Key Technique

### Structured Evidence Decomposition (SED)

Traditional multimodal emotion models fuse all inputs simultaneously, which causes **unimodal dominance** -- the model over-relies on whichever modality has the strongest statistical signal (usually text transcripts), ignoring subtle but critical cues in facial expression or vocal prosody. SABER-LLM solves this by decomposing the output generation into a strict causal chain:

**Y = [E_visual, E_acoustic, R]** where the probability factorizes as:

`P(Y|X) = P(E_v|X) * P(E_a|E_v, X) * P(R|E_v, E_a, X)`

The model must first articulate what it sees (micro-expressions, gaze, posture, scene context), then what it hears (prosody, pitch shifts, tonal intensity) conditioned on the visual context, and only then synthesize a causal emotional interpretation. This forces grounding: every claim in the final reasoning must trace back to specific observed evidence.

### Six-Dimensional Annotation Schema

SABER structures each annotation across six dimensions: (1) Video Description -- macro-scene context; (2) Facial Expression -- micro-expressions, gaze patterns; (3) Body Language -- posture, gestures, social signals; (4) Acoustic Features -- prosody, pitch, tonal intensity; (5) Speech Content -- transcripts, semantic meaning; (6) Comprehensive Reasoning -- causal synthesis of all evidence into emotional interpretation. This schema ensures no modality is skipped during annotation or inference.

### Consistency-Aware Direct Preference Optimization (CA-DPO)

After SFT training on the structured evidence format, CA-DPO addresses a remaining failure mode: the model may produce internally contradictory analyses (e.g., noting a "trembling voice" but concluding "confident delivery"). CA-DPO samples K candidate responses, scores them for logical consistency and evidence alignment, then forms preference pairs (consistent = preferred, contradictory = rejected) and optimizes via standard DPO loss. This stage specifically targets emotionally ambiguous or conflicting samples where cross-modal alignment is hardest.

## Step-by-Step Workflow

### Building a Multimodal Emotion Reasoning Pipeline

1. **Define the six-dimensional evidence schema** for your domain. Map each dimension to concrete extractors: use a face analysis model (e.g., OpenFace, MediaPipe) for facial expressions, a speech prosody extractor (e.g., OpenSMILE, librosa) for acoustic features, ASR (e.g., Whisper) for transcripts, and a vision model for scene description and body language.

2. **Build the evidence extraction layer** as independent modules. Each module outputs structured JSON with dimension name, timestamp ranges, specific observations, and confidence scores. Do NOT merge modalities at this stage -- the perceive-then-reason separation is the core architectural principle.

3. **Implement the sequential evidence chain**. Wire the modules so visual evidence (scene + face + body) is extracted first, then acoustic evidence is extracted with visual context as conditioning input. This ordering matters: acoustic interpretation of a "rising pitch" depends on whether the visual context shows a question or an argument.

4. **Design structured prompts** that enforce the six-dimensional format. The prompt template should require the model to produce each evidence dimension as a labeled section before a final "Comprehensive Reasoning" section. Reject or re-prompt any output that skips dimensions or jumps directly to a conclusion.

5. **Build a hallucination verification layer** with two checks: (a) compare ASR-generated transcripts against any speech content claims using Word Error Rate thresholds, and (b) cross-validate visual descriptions against an independent vision model's output using semantic similarity scoring. Flag entries where WER > threshold or semantic similarity < threshold.

6. **Construct training data in the SED format**. For each video clip, annotate all six dimensions independently, then write the comprehensive reasoning section that explicitly references evidence from the other five dimensions. Use the causal language pattern: "Because [visual evidence] and [acoustic evidence], the speaker likely feels [emotion] due to [causal explanation]."

7. **Train with two-stage optimization**. Stage 1: SFT on the structured evidence format (2 epochs, batch 128, lr 1e-4) to teach the model the perceive-then-reason decomposition. Stage 2: CA-DPO (1 epoch, batch 64, lr 1e-5) sampling K=10 candidate responses at temperature 0.8, scoring for cross-modal consistency, and forming preference pairs.

8. **Evaluate with modality conflict tests**. Build a diagnostic test set with equal parts "consistent" samples (all modalities agree) and "inconsistent" samples (e.g., happy face + sad voice, sarcastic tone + literal words). Score models on both subsets separately to measure robustness to contradiction.

9. **Deploy with structured output parsing**. In production, parse the model's output into the six-dimensional JSON schema. If any dimension is missing or contains only generic filler ("normal expression"), flag the response for re-inference or human review.

## Concrete Examples

**Example 1: Building a Sarcasm Detection Pipeline**

User: "I need to detect sarcasm in customer service call recordings that include video. The model keeps predicting sentiment based only on the words and missing sarcastic tone."

Approach:
1. Diagnose the problem as unimodal dominance -- the model over-relies on text transcript sentiment
2. Implement SED evidence extraction with separate modules:
   - Visual: Extract facial action units (AU12 lip corner puller + AU6 cheek raiser = genuine smile vs. AU12 alone = social/sarcastic smile)
   - Acoustic: Extract pitch contour, speaking rate, and intensity; sarcasm often shows exaggerated pitch range with flat affect
   - Linguistic: Run ASR then flag surface-level positive phrases
3. Design the prompt to enforce sequential evidence:

```
Analyze the emotional content of this customer service interaction.

## Visual Evidence
Describe the customer's facial expressions, noting specific action units
and their temporal alignment with speech.

## Acoustic Evidence
Given the visual context above, describe vocal prosody: pitch contour,
speaking rate changes, intensity patterns, and any mismatches with
the facial expression.

## Speech Content
Transcribe and note the literal semantic content.

## Comprehensive Reasoning
Synthesize the evidence above. Specifically address any contradictions
between modalities (e.g., positive words + flat/exaggerated prosody +
controlled facial expression = potential sarcasm). Provide a causal
explanation for your emotional assessment.
```

Output structure:
```json
{
  "visual_evidence": {
    "facial_expression": "Lip corner pull without orbicularis oculi engagement (social smile, AU12 only). Slight eyebrow raise at 0:03.",
    "body_language": "Arms crossed, minimal gesture during positive statements."
  },
  "acoustic_evidence": {
    "prosody": "Exaggerated pitch range on 'wonderful service' (F0 peak 280Hz vs baseline 180Hz). Elongated vowels in 'reeeally helpful'.",
    "intensity": "Flat overall energy despite lexically emphatic content."
  },
  "speech_content": "Oh, that was really wonderful service. So helpful.",
  "reasoning": "Contradictory signals: lexically positive content paired with non-Duchenne smile, crossed arms, and exaggerated prosody suggest sarcasm. The pitch exaggeration on 'wonderful' exceeds typical sincere emphasis by 2x, consistent with ironic intent.",
  "emotion": "sarcasm (masking frustration)",
  "confidence": 0.87,
  "conflict_flag": true
}
```

**Example 2: Building a Six-Dimensional Annotation Pipeline for a Video Dataset**

User: "I have 50K video clips of therapy sessions and need to annotate them for emotion research. How should I structure the annotations?"

Approach:
1. Implement the SABER six-dimensional schema adapted for therapeutic context
2. Build automated pre-annotation pipeline, then human review layer

```python
from dataclasses import dataclass, field
from enum import Enum

class ModalityConfidence(Enum):
    HIGH = "high"        # Clear, unambiguous signal
    MODERATE = "moderate" # Detectable but subtle
    LOW = "low"          # Ambiguous or partially occluded
    ABSENT = "absent"    # Modality not available

@dataclass
class SABERAnnotation:
    clip_id: str
    timestamp_range: tuple[float, float]

    # Dimension 1: Scene context
    video_description: str  # "Therapy office, patient seated across from therapist, warm lighting"

    # Dimension 2: Facial expression
    facial_expression: str  # "Brow furrow (AU4) onset at 2.1s, lip press (AU24) sustained 2.1-3.4s"
    facial_confidence: ModalityConfidence = ModalityConfidence.HIGH

    # Dimension 3: Body language
    body_language: str  # "Self-soothing gesture (hand to neck) at 2.3s, gaze aversion at 2.5s"
    body_confidence: ModalityConfidence = ModalityConfidence.HIGH

    # Dimension 4: Acoustic features
    acoustic_features: str  # "Voice pitch drops 40Hz at 'I don't know', speaking rate slows by 30%"
    acoustic_confidence: ModalityConfidence = ModalityConfidence.HIGH

    # Dimension 5: Speech content
    speech_content: str  # Verbatim transcript
    asr_wer: float = 0.0  # Word error rate from verification

    # Dimension 6: Comprehensive reasoning
    comprehensive_reasoning: str  # Causal synthesis referencing dimensions 1-5
    cross_modal_consistency: float = 1.0  # 0-1 score of modality agreement

    # Metadata
    modality_conflict: bool = False  # True if dimensions contradict
    conflict_description: str = ""   # "Verbal denial of distress contradicts acoustic and facial cues"

def build_verification_pipeline(annotation: SABERAnnotation) -> list[str]:
    """Two-step hallucination verification per SABER protocol."""
    issues = []
    # Step 1: ASR consistency check
    if annotation.asr_wer > 0.15:
        issues.append(f"Speech content WER {annotation.asr_wer:.2f} exceeds threshold")
    # Step 2: Cross-validate visual claims against independent vision model
    if annotation.cross_modal_consistency < 0.6:
        issues.append(f"Cross-modal consistency {annotation.cross_modal_consistency:.2f} is low")
    # Step 3: Check reasoning references evidence
    for dim in ["facial", "acoustic", "body", "speech"]:
        if dim not in annotation.comprehensive_reasoning.lower():
            issues.append(f"Reasoning does not reference {dim} evidence")
    return issues
```

**Example 3: Implementing CA-DPO for Cross-Modal Consistency**

User: "My multimodal emotion model sometimes says 'the speaker sounds anxious' but then concludes 'they appear relaxed and confident'. How do I fix these contradictions?"

Approach:
1. This is exactly the internal inconsistency that CA-DPO targets
2. Build a preference pair generation pipeline:

```python
import json
from typing import NamedTuple

class PreferencePair(NamedTuple):
    prompt: str
    chosen: str    # Internally consistent response
    rejected: str  # Internally contradictory response

def generate_preference_pairs(
    model, clips: list, k: int = 10, temperature: float = 0.8
) -> list[PreferencePair]:
    """
    CA-DPO preference pair construction:
    1. For each clip, sample K candidate responses
    2. Score each for internal logical consistency
    3. Pair highest-scored (chosen) with lowest-scored (rejected)
    """
    pairs = []
    for clip in clips:
        prompt = format_sed_prompt(clip)
        candidates = [model.generate(prompt, temperature=temperature) for _ in range(k)]

        scored = []
        for candidate in candidates:
            evidence = extract_evidence_sections(candidate)
            reasoning = extract_reasoning_section(candidate)
            # Score: does reasoning align with stated evidence?
            consistency = score_evidence_reasoning_alignment(evidence, reasoning)
            # Score: do modality-specific claims contradict each other?
            cross_modal = score_cross_modal_consistency(evidence)
            scored.append((candidate, consistency * 0.6 + cross_modal * 0.4))

        scored.sort(key=lambda x: x[1], reverse=True)
        if scored[0][1] - scored[-1][1] > 0.2:  # Meaningful quality gap
            pairs.append(PreferencePair(
                prompt=prompt,
                chosen=scored[0][0],
                rejected=scored[-1][0]
            ))
    return pairs

def score_cross_modal_consistency(evidence: dict) -> float:
    """
    Check for contradictions between modality-specific evidence claims.
    Examples of contradictions to detect:
    - Visual says 'tense jaw' but acoustic says 'relaxed, flowing speech'
    - Facial says 'genuine smile' but body says 'defensive, closed posture'
    """
    # Implementation: embed each evidence section, compute pairwise
    # sentiment/arousal alignment, flag large disagreements
    ...
```

## Best Practices

- **Do:** Always extract evidence for each modality independently before reasoning. The sequential factorization `P(E_v) -> P(E_a|E_v) -> P(R|E_v,E_a)` is the core contribution -- skipping it collapses back to unimodal dominance.
- **Do:** Include a `modality_conflict` flag in your output schema. Contradictions between modalities are not errors -- they are the most informative signals (sarcasm, suppressed emotion, social masking). Surface them explicitly.
- **Do:** Verify evidence claims against independent extractors. Use ASR with WER checks for speech content and independent vision models for visual descriptions. This two-step verification catches hallucinated observations.
- **Do:** Condition acoustic analysis on visual context. The same pitch rise means different things in a heated argument versus a calm question -- the visual scene disambiguates.
- **Avoid:** Letting the model skip directly to an emotion label without producing per-modality evidence sections. If the structured output is missing dimensions, reject and re-prompt.
- **Avoid:** Training only on emotionally unambiguous samples. The CA-DPO stage specifically requires a balanced mix of consistent (all modalities agree) and inconsistent (modalities conflict) samples to build robustness.
- **Avoid:** Using a single aggregate confidence score. Report confidence per modality so downstream systems can weight evidence appropriately when some channels are degraded (e.g., occluded face, noisy audio).

## Error Handling

| Failure Mode | Detection | Mitigation |
|---|---|---|
| Missing modality (no audio, face occluded) | Confidence score = `absent` for dimension | Mark dimension as unavailable; do not fabricate evidence. Adjust reasoning to note reduced confidence. |
| ASR transcript diverges from speech claims | WER check exceeds 0.15 | Re-run ASR; if persistent, flag speech content as unreliable and weight acoustic features higher. |
| Model skips evidence dimensions | Parse structured output; check for missing sections | Re-prompt with explicit instruction to address skipped dimension, or return partial result with gap noted. |
| Contradictory reasoning (evidence says X, conclusion says Y) | Cross-modal consistency scorer flags < 0.6 | Route to CA-DPO-style re-ranking: sample multiple completions, select most internally consistent. |
| Unimodal dominance detected | One modality's evidence is copy-pasted into reasoning while others are ignored | Add explicit prompt instruction: "Your reasoning MUST reference specific observations from at least 3 of the 5 evidence dimensions." |

## Limitations

- **Requires multimodal input.** The perceive-then-reason decomposition assumes both audio and visual channels are available. For text-only or audio-only emotion analysis, the sequential conditioning chain degenerates and simpler approaches suffice.
- **Computationally expensive at inference.** Generating structured evidence across six dimensions before reasoning produces significantly longer outputs than direct classification. Budget 3-5x the token count of a flat prediction.
- **Dependent on upstream extractor quality.** If face detection, ASR, or prosody extraction is poor (low-light video, heavy background noise, non-standard accents), the evidence layer will contain noise that propagates into reasoning.
- **Cultural and contextual bias.** Facial action unit interpretation and prosodic norms vary across cultures. A schema trained on one cultural context may misinterpret expression norms from another. Always validate against culturally appropriate ground truth.
- **Not suitable for real-time streaming.** The sequential evidence chain and verification steps introduce latency incompatible with live interaction. For real-time needs, consider a simplified two-dimension (visual + acoustic) variant without the full verification pipeline.

## Reference

**Paper:** [Integrating Fine-Grained Audio-Visual Evidence for Robust Multimodal Emotion Reasoning](https://arxiv.org/abs/2601.18321v2) (Zhao, Tian, Xie, 2026)

Look for: The structured evidence decomposition formulation (Section 3.2), the six-dimensional annotation schema (Section 3.1), and the CA-DPO preference pair construction process (Section 3.3). The SABER-Test diagnostic benchmark design (Section 4) provides a template for building your own modality-conflict evaluation sets.