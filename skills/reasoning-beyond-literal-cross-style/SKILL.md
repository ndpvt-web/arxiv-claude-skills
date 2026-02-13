---
name: "reasoning-beyond-literal-cross-style"
description: "Detect and interpret figurative language (sarcasm, humor, offense, metaphor) in multimodal image-text content using a structured five-step reasoning chain. Triggers: 'detect sarcasm in this meme', 'is this image sarcastic', 'analyze figurative meaning', 'interpret this humor/metaphor', 'classify meme sentiment', 'explain why this meme is funny/offensive'"
---

# Cross-Style Multimodal Reasoning for Figurative Language

This skill enables Claude to interpret figurative language in multimodal (image + text) content by applying a structured five-step reasoning chain derived from the CrossStyle-MMR framework. Instead of making snap classification judgments, Claude decomposes figurative language analysis into explicit steps: describing visual content, interpreting the caption, detecting mismatches between modalities, inferring communicative intent, and arriving at a justified label. This approach transfers across figurative styles -- reasoning trained on sarcasm improves humor detection and vice versa -- enabling robust cross-style generalization from a single unified pipeline.

## When to Use

- When the user provides a meme, social media post, or image-text pair and asks whether it is sarcastic, humorous, offensive, or metaphorical
- When building a content moderation pipeline that must distinguish literal from figurative intent in user-generated image-text content
- When the user asks to classify or label figurative language style in multimodal datasets (e.g., MMSD2.0, Memotion, MultiMET)
- When implementing a reasoning-trace-based VLM fine-tuning pipeline for figurative language tasks using teacher-student distillation
- When the user needs to understand *why* a piece of content is sarcastic/humorous rather than just getting a binary label
- When building a cross-style training pipeline where reasoning learned on one figurative style (e.g., sarcasm) bootstraps performance on another (e.g., humor)

## Key Technique

The core insight is that figurative language understanding improves dramatically when models produce structured reasoning traces before classification. The framework uses a **five-step reasoning schema**: (1) describe what the image shows, (2) interpret what the caption says at face value, (3) detect mismatches between visual and textual meaning, (4) infer the speaker's actual communicative intent, and (5) predict the figurative label. This schema forces explicit incongruity detection -- the mechanism by which sarcasm inverts meaning, humor exploits absurdity, offense weaponizes context, and metaphor maps abstract concepts onto concrete imagery.

The pipeline has three stages. **Stage 1 (Distillation):** A large teacher VLM (e.g., LLaMA3.2-90B-Vision-Instruct) generates chain-of-thought reasoning traces under zero-shot prompting using the five-step schema. Traces are automatically filtered to retain only those with valid reasoning paths and correct final labels. **Stage 2 (Student Training):** A smaller student VLM (e.g., Qwen2.5-VL-3B) is fine-tuned via supervised learning on the distilled traces, then further refined with Reinforcement Learning with Verifiable Rewards (RLVR/GRPO) where the reward signal combines accuracy and format adherence. **Stage 3 (Cross-Style Unification):** Joint SFT followed by GRPO across all four figurative styles produces a single generalized model that outperforms much larger models.

The critical finding for practitioners: **cross-style transfer is real and measurable**. SFT on sarcasm CoTs followed by RLVR on humor yields ~10% accuracy improvement over training on humor alone. Sarcasm and humor transfer most readily to each other; metaphor and offense share structural similarities. This means you can bootstrap a new figurative style classifier from an existing one with minimal target-style data.

## Step-by-Step Workflow

1. **Receive the multimodal input.** Identify the image and accompanying text (caption, tweet, meme text overlay). If the user provides only text, note that full cross-modal reasoning requires an image -- fall back to text-only figurative analysis with a caveat.

2. **Step 1 -- Image Description.** Describe the visual content objectively. Note objects, people, expressions, settings, visual tone, and any text rendered in the image. Do not interpret intent yet. Wrap this in the reasoning trace.

3. **Step 2 -- Caption Interpretation.** Parse the textual caption at face value. Identify its literal meaning, tone markers (punctuation, capitalization, emoji), and any named entities or cultural references.

4. **Step 3 -- Mismatch Detection.** Compare the visual content against the textual meaning. Identify specific incongruities: does the image contradict the caption? Does the visual tone clash with the textual sentiment? Is there exaggeration, understatement, or absurdity in the pairing? This is the critical step -- figurative language lives in the gap between expressed and intended meaning.

5. **Step 4 -- Intent Inference.** Based on detected mismatches, infer the speaker's actual communicative intent. For sarcasm: the intended meaning is opposite to the literal text. For humor: the mismatch is designed to amuse. For offense: the pairing targets a group or individual. For metaphor: the image maps an abstract concept onto a concrete visual.

6. **Step 5 -- Label Prediction with Justification.** Output the figurative style classification (sarcastic/not-sarcastic, humorous/not-humorous, offensive/not-offensive, metaphorical/literal) along with a confidence assessment grounded in the reasoning chain.

7. **Format the reasoning trace.** Enclose the full five-step reasoning in `<think>...</think>` tags and the final classification in `<answer>...</answer>` tags for downstream parseability.

8. **Apply cross-style awareness.** If the content exhibits multiple figurative styles (e.g., sarcastic *and* humorous), classify each independently using the same reasoning chain, noting shared incongruity signals.

9. **Handle ambiguous cases explicitly.** If the mismatch detection step yields weak or contested signals, flag the content as ambiguous rather than forcing a binary label. Report which aspects push toward figurative vs. literal interpretation.

10. **When building a training pipeline:** Generate distillation traces from a large teacher model using the five-step schema prompt, filter for correctness, fine-tune a smaller student model via SFT (5 epochs, lr=2e-4, cosine schedule), then apply GRPO (2 epochs, lr=1e-5, 8 rollouts) with a reward combining accuracy + format adherence.

## Concrete Examples

**Example 1: Sarcasm Detection in a Meme**

User: "Is this meme sarcastic? The image shows a person smiling while their car is on fire. The caption says 'What a great start to my Monday!'"

Approach:
```
<think>
Step 1 (Image Description): The image shows a person standing next to a car
engulfed in flames. The person is smiling or grinning. The setting appears to
be a parking lot during daytime.

Step 2 (Caption Interpretation): The caption "What a great start to my Monday!"
literally expresses enthusiasm and positivity about the beginning of the week.
The exclamation mark reinforces an upbeat tone.

Step 3 (Mismatch Detection): Strong incongruity detected. The visual content
depicts a clearly negative event (car fire) paired with positive/enthusiastic
text. The person's smile despite the disaster amplifies the contradiction. This
is a textbook sentiment inversion pattern.

Step 4 (Intent Inference): The speaker's actual intent is the opposite of the
literal text. They are expressing frustration, exasperation, or resignation
about a terrible start to their week. The smile is performative -- a coping
mechanism that reinforces the sarcastic framing.

Step 5 (Label): Sarcastic. High confidence. The image-text incongruity follows
a classic sarcasm pattern: positive language applied to an unambiguously
negative situation, with visual cues (forced smile) supporting ironic intent.
</think>

<answer>Sarcastic</answer>
```

**Example 2: Humor vs. Literal in an Image-Text Pair**

User: "Classify this: Image shows a dog sitting at a desk wearing glasses with a laptop open. Caption: 'When the code compiles on the first try.'"

Approach:
```
<think>
Step 1 (Image Description): A dog is seated at an office desk, wearing
reading glasses, positioned in front of an open laptop. The dog's posture
mimics a human office worker. The setting is a typical home office.

Step 2 (Caption Interpretation): The caption references a software development
scenario -- code compiling successfully on the first attempt. This is widely
understood among programmers as a rare and surprising event.

Step 3 (Mismatch Detection): The incongruity operates on two levels. First,
anthropomorphism: a dog performing human work is inherently absurd. Second,
the shared cultural knowledge that first-try compilation is unusual creates
a "too good to be true" tension. The dog's composed, professional demeanor
contrasts with the implied improbability.

Step 4 (Intent Inference): The creator intends humor through relatable
exaggeration. The message is: the feeling of code compiling on the first try
is so rare and satisfying that it warrants an absurd reaction image. No
sarcasm is present -- the sentiment is genuinely positive, amplified through
absurdist visual framing.

Step 5 (Label): Humorous (not sarcastic). The content uses absurdist
anthropomorphism paired with programmer in-group cultural reference to
generate humor. No sentiment inversion detected.
</think>

<answer>Humorous</answer>
```

**Example 3: Building a Cross-Style Training Pipeline**

User: "I want to fine-tune a small VLM to detect sarcasm and humor in memes. How should I structure the training pipeline?"

Approach:
1. Collect image-text pairs from MMSD2.0 (sarcasm, ~25K pairs) and Memotion (humor, ~9.8K pairs)
2. Generate reasoning traces using a large teacher model with this prompt template:

```
Given the image and caption below, analyze whether this content is [sarcastic/humorous].
Provide your reasoning in five steps:
1. Describe what the image shows
2. Interpret the caption's literal meaning
3. Identify any mismatches between image and text
4. Infer the speaker's actual intent
5. Provide your final classification

Format your response with reasoning inside <think></think> and your
final answer inside <answer></answer>.

Caption: "{caption}"
```

3. Filter generated traces: keep only those where the predicted label matches ground truth and the trace contains all five reasoning steps
4. Fine-tune student VLM via SFT on combined sarcasm + humor traces (5 epochs, lr=2e-4, cosine annealing)
5. Apply GRPO refinement (2 epochs, lr=1e-5, 8 rollouts per sample) with reward = 0.5 * accuracy + 0.5 * format_adherence
6. Evaluate cross-style transfer: the joint model should outperform style-specific models on both tasks

## Best Practices

- **Do:** Always complete all five reasoning steps before classification. Skipping mismatch detection (Step 3) causes the most significant accuracy drops -- it is the core mechanism that distinguishes figurative from literal content.
- **Do:** Treat cross-style transfer as a resource multiplier. If you have strong sarcasm training data but limited humor data, train on sarcasm first and transfer -- sarcasm and humor share the strongest bidirectional transfer (~10% accuracy gain).
- **Do:** Filter distilled reasoning traces aggressively. Only retain traces where the teacher model's final label matches ground truth AND all five schema steps are present. Low-quality traces degrade student performance.
- **Do:** Use the `<think>...</think>` and `<answer>...</answer>` tag format consistently. This enables automated evaluation, reward computation during RLVR, and downstream parsing.
- **Avoid:** Classifying figurative language from text alone when an image is available. The paper shows that cross-modal incongruity is the primary signal -- text-only analysis misses visual amplification and inversion of meaning.
- **Avoid:** Training on a single figurative style when multi-style data is available. Joint training across all four styles (sarcasm, humor, offense, metaphor) consistently outperforms single-style specialists, even on the specialist's own benchmark.

## Error Handling

- **Ambiguous incongruity:** When the mismatch between image and text is subtle or culturally dependent, explicitly state the ambiguity in the reasoning trace rather than forcing a confident label. Flag cultural context that may be required for accurate interpretation.
- **Missing image context:** If the image is low-resolution, cropped, or unavailable, note this limitation in Step 1 and reduce confidence in the final classification. Text-only figurative detection is feasible but less reliable.
- **Multi-style overlap:** Content can be simultaneously sarcastic and humorous, or offensive and metaphorical. When multiple styles are detected, classify each independently and note the overlap rather than choosing one.
- **Teacher trace quality:** During pipeline construction, if the teacher model generates reasoning traces that reach the correct label via flawed reasoning (right answer, wrong path), discard them. Format-only filtering is insufficient -- validate that Step 3 (mismatch detection) actually identifies a real incongruity.
- **Cultural and temporal context:** Figurative language is heavily context-dependent. Memes reference current events, slang, and subcultures. When context is missing, state this explicitly rather than hallucinating cultural references.

## Limitations

- The five-step schema is optimized for the four tested figurative styles (sarcasm, humor, offense, metaphor). Other figurative forms like irony-without-inversion, allegory, or hyperbole may require schema adaptation.
- Cross-modal reasoning assumes the image and text have a meaningful semantic relationship. Random or template-based image-text pairings (common in low-effort memes) may not exhibit the incongruity patterns the framework relies on.
- The framework was validated on English-language datasets. Figurative language is deeply cultural and linguistic -- transfer to other languages requires new training data and potentially modified reasoning schemas.
- Offense detection shows the lowest accuracy (59.51%) across all four styles, suggesting that offensive content requires contextual and normative reasoning beyond incongruity detection.
- The teacher-student distillation requires access to a large VLM (90B+ parameters) for trace generation, which may be a resource constraint for some teams.

## Reference

**Paper:** [Reasoning Beyond Literal: Cross-style Multimodal Reasoning for Figurative Language Understanding](https://arxiv.org/abs/2601.17197v1) (Cheshmi et al., 2026). Look for: the five-step reasoning schema definition (Section 3), cross-style transfer matrix showing which style pairs transfer best (Section 5), and the GRPO reward function design combining accuracy with format adherence.

**Code:** [github.com/scheshmi/CrossStyle-MMR](https://github.com/scheshmi/CrossStyle-MMR)