---
name: "unveiling-cognitive-compass-theory-of-mind-guided"
description: "Apply Theory-of-Mind (ToM) guided reasoning chains to multimodal emotion analysis tasks. Decomposes emotional reasoning into hierarchical cognitive levels—perception, understanding, and causal cognition—tracking mental states explicitly before reaching conclusions. Use when: 'analyze the emotions in this image/video', 'why does this person feel that way', 'build an emotion reasoning pipeline', 'detect sarcasm or humor in multimodal content', 'evaluate emotional understanding in my model', 'add ToM-based reasoning to my MLLM'."
---

# Theory-of-Mind Guided Multimodal Emotion Reasoning

This skill teaches Claude to apply the ToM-guided reasoning framework from the HitEmotion paper (ICLR 2026) to real coding tasks involving emotional understanding in multimodal systems. The core technique decomposes emotion reasoning into three hierarchical cognitive levels—perception, understanding, and causal cognition—requiring explicit mental state tracking at each level before producing a final answer. This replaces flat emotion classification with structured chains of thought that model beliefs, intentions, and second-order reasoning, producing more faithful and coherent rationales.

## When to Use

- When the user asks to build or improve an emotion recognition/analysis pipeline that handles images, video, audio, or text
- When designing prompts or reasoning chains for MLLMs that need to explain *why* someone feels an emotion, not just *what* emotion is present
- When implementing sarcasm, humor, or irony detection where surface signals contradict intended meaning
- When evaluating an MLLM's emotional reasoning depth and identifying where it breaks down
- When the user wants to add structured intermediate reasoning (think-then-answer) to a multimodal model
- When building training pipelines that use process-level supervision on reasoning chains (TMPO-style RL)
- When constructing benchmarks or evaluation suites that test emotional cognition at multiple difficulty tiers

## Key Technique

**Hierarchical Cognitive Decomposition.** The paper identifies three ascending levels of emotional cognition: (1) Emotion Perception and Recognition (EPR)—mapping observable cues like facial expressions, tone, or word choice to emotion labels; (2) Emotion Understanding and Analysis (EUA)—contextualizing emotions within relationships, targets, and communicative goals (e.g., aspect-based sentiment); (3) Emotion Cognition and Reasoning (ECR)—causal attribution, temporal dynamics, and second-order reasoning (e.g., "she said it sarcastically because she believes he already knows the answer"). Current MLLMs collapse at Level 3, where no task in the HitEmotion benchmark exceeds 60% average accuracy.

**ToM-Guided Reasoning Chain.** Instead of prompting a model to jump directly from input to emotion label, the technique inserts explicit mental-state tracking steps. The model first attributes first-order mental states from observable signals, then contextualizes them relationally, then performs causal/second-order reasoning. Outputs are structured as `<think>[reasoning chain]</think><answer>[conclusion]</answer>`, forcing the model to externalize its cognitive process before committing to an answer.

**TMPO (Theory of Mind Process Optimization).** A two-stage RL method: Stage 1 fine-tunes on ToM-aligned reasoning chains (SFT). Stage 2 uses Group Relative Policy Optimization (GRPO) with a composite reward: structure reward (correct reasoning steps present), content reward (answer correctness), process reward (ToM-specific reasoning markers like "belief," "intention," "perspective"), and consistency reward (no logical contradictions). Weights: structure=0.4, content=1.0, process=0.1, consistency=1.0. This turns emotion reasoning from an emergent capability into a systematically trained one.

## Step-by-Step Workflow

1. **Classify the task's cognitive level.** Determine whether the user's task requires Level 1 (label an emotion), Level 2 (understand emotion in context/toward a target), or Level 3 (explain causes, detect irony, track mental states). This determines how deep the reasoning chain needs to go.

2. **Identify all available modalities.** Inventory the input signals: text (dialogue, captions, subtitles), audio (tone, pitch, pacing), visual (facial expressions, body language, scene context), and video (temporal dynamics, gesture sequences). Map each modality to the cues it can provide.

3. **Extract first-order mental state attributions (Level 1).** For each modality, produce explicit perception statements: "The speaker's facial expression shows furrowed brows and tight lips (anger cues)," "The vocal tone is flat and monotone (possible sadness or sarcasm)," "The text contains the word 'great' in a context suggesting positive sentiment."

4. **Contextualize with relational/target-aware reasoning (Level 2).** Link the perceived emotions to specific entities, relationships, and communicative goals: "The speaker directs this statement at their manager," "The smile occurs immediately after receiving negative news, suggesting a social mask rather than genuine happiness." Resolve cross-modal conflicts here (e.g., positive words + negative tone).

5. **Perform causal and second-order reasoning (Level 3).** Model why the emotion exists and how it is meant to be interpreted: "She believes he already knows the project failed, so her cheerful tone is sarcastic—she intends him to feel the weight of the failure." Track belief states: what does person A believe about person B's knowledge/feelings?

6. **Structure the output as think-then-answer.** Wrap the reasoning chain in `<think>...</think>` tags and the final answer in `<answer>...</answer>` tags. The think block must contain explicit mental-state language (beliefs, intentions, perspectives) at each applicable level.

7. **Validate cross-modal consistency.** Check that the conclusion does not contradict evidence from any single modality without an explicit explanation. If text says "I'm fine" but audio/visual signals say otherwise, the reasoning chain must address the discrepancy.

8. **Implement reward signals for training pipelines (if building a training system).** Define four reward components: (a) structure reward—verify `<think>` and `<answer>` tags and correct step ordering; (b) content reward—compare final answer to ground truth; (c) process reward—count ToM keywords in the reasoning chain; (d) consistency reward—penalize contradictions between the reasoning and the answer. Use weight ratio 0.4:1.0:0.1:1.0.

9. **Evaluate at each cognitive level independently.** Do not report a single aggregate score. Break performance into EPR, EUA, and ECR tiers to identify exactly where the model's reasoning breaks down. This exposes capability breakpoints that aggregate metrics hide.

10. **Iterate on failure modes.** If the model fails at Level 3 but succeeds at Levels 1-2, add more causal reasoning examples to training data or strengthen the process reward. If it fails at Level 1, the issue is perceptual and requires better feature extraction, not deeper reasoning.

## Concrete Examples

**Example 1: Building a sarcasm detection prompt chain**

User: "I need to detect sarcasm in video clips where someone says something positive but means the opposite. How should I structure my reasoning?"

Approach:
1. Classify as Level 3 (ECR)—sarcasm requires second-order reasoning about speaker intent vs. literal meaning.
2. Identify modalities: text transcript, audio tone, facial expression from video frames.
3. Build the reasoning chain template:

```
<think>
[Level 1 - Perception]
- Text: The speaker says "Oh, what a wonderful surprise."
- Audio: Flat intonation, no pitch rise on "wonderful," slight elongation of "oh."
- Visual: Eye roll at 0:02, forced smile that does not engage orbicularis oculi.

[Level 2 - Contextual Understanding]
- The statement follows the speaker being assigned extra weekend work.
- The phrase "wonderful surprise" is directed at the manager who assigned it.
- Cross-modal conflict: positive lexical content vs. negative prosodic/visual cues.

[Level 3 - Causal/Second-Order Reasoning]
- The speaker believes the manager knows this is unwelcome.
- The speaker intends the positive words to be interpreted as the opposite.
- This is verbal irony: the speaker's communicative goal is to express displeasure
  while maintaining plausible deniability.
</think>
<answer>Sarcasm detected. The speaker uses verbal irony—positive words with negative
prosody and facial cues—to express displeasure about the work assignment.</answer>
```

**Example 2: Evaluating an MLLM's emotional reasoning depth**

User: "I have a multimodal model and want to benchmark how well it understands emotions at different depths. How do I set this up?"

Approach:
1. Create three test suites corresponding to the HitEmotion hierarchy:

```python
# Level 1 (EPR): Direct emotion classification
epr_tasks = {
    "facial_emotion_recognition": {
        "input": "image of a face",
        "expected": "emotion label (happy, sad, angry, ...)",
        "metric": "weighted_f1"
    },
    "sentiment_classification": {
        "input": "text + image",
        "expected": "positive/negative/neutral",
        "metric": "accuracy"
    }
}

# Level 2 (EUA): Target-aware emotion understanding
eua_tasks = {
    "aspect_sentiment": {
        "input": "review text + product image",
        "expected": "sentiment toward specific aspect (e.g., 'battery life: negative')",
        "metric": "f1_per_aspect"
    },
    "multimodal_humor_detection": {
        "input": "meme image + caption",
        "expected": "humorous / not humorous + explanation",
        "metric": "accuracy + rationale_quality"
    }
}

# Level 3 (ECR): Causal reasoning and second-order ToM
ecr_tasks = {
    "emotion_cause_extraction": {
        "input": "video clip + transcript",
        "expected": "cause span + causal chain",
        "metric": "span_f1 + chain_coherence"
    },
    "sarcasm_reasoning": {
        "input": "video + audio + text",
        "expected": "sarcasm label + speaker belief state + intended interpretation",
        "metric": "accuracy + faithfulness_score"
    }
}
```

2. Score each level independently. Report a per-level breakdown. Flag any Level 3 task scoring below 60% as a confirmed reasoning deficit.

**Example 3: Adding TMPO-style training to a multimodal model**

User: "I want to fine-tune my vision-language model so it produces better emotion reasoning chains. How do I set up the reward function?"

Approach:
1. Define the composite reward:

```python
import re

def compute_reward(generated: str, ground_truth: str, task_type: str) -> float:
    # Structure reward (mu_1 = 0.4): verify reasoning format
    has_think = bool(re.search(r"<think>.*?</think>", generated, re.DOTALL))
    has_answer = bool(re.search(r"<answer>.*?</answer>", generated, re.DOTALL))
    r_structure = 1.0 if (has_think and has_answer) else 0.0

    # Content reward (mu_2 = 1.0): task-specific correctness
    answer_text = extract_answer_tag(generated)
    r_content = compute_task_metric(answer_text, ground_truth, task_type)

    # Process reward (mu_3 = 0.1): ToM keyword density in reasoning
    think_text = extract_think_tag(generated)
    tom_keywords = ["belief", "believes", "intention", "intends", "perspective",
                    "assumes", "expects", "knows", "thinks", "feels"]
    keyword_count = sum(1 for kw in tom_keywords if kw in think_text.lower())
    r_process = min(keyword_count / 5.0, 1.0)  # normalize, cap at 1.0

    # Consistency reward (mu_4 = 1.0): answer must not contradict reasoning
    r_consistency = check_reasoning_answer_alignment(think_text, answer_text)

    return 0.4 * r_structure + 1.0 * r_content + 0.1 * r_process + 1.0 * r_consistency
```

2. Use GRPO: sample N=8 candidate outputs per input, score each, compute normalized advantages, and optimize the policy against the reference model with KL penalty (beta=0.04).

## Best Practices

- **Do:** Always externalize mental states explicitly in the reasoning chain. Write out "Person A believes X" rather than jumping to "Person A is sarcastic." The intermediate states are what make the reasoning faithful and auditable.
- **Do:** Resolve cross-modal conflicts in the reasoning chain rather than silently ignoring one modality. If text is positive but tone is negative, say so and explain which signal dominates and why.
- **Do:** Evaluate at each cognitive level separately. A model scoring 90% on emotion recognition (Level 1) may score 40% on emotion cause reasoning (Level 3). Aggregate scores hide this.
- **Do:** Use the `<think>...</think><answer>...</answer>` structure consistently. It enables both better training signal extraction and cleaner evaluation.
- **Avoid:** Skipping Level 2 to jump from perception (Level 1) to causal reasoning (Level 3). The contextual grounding step prevents hallucinated causal chains.
- **Avoid:** Using ToM keyword counting as a primary reward signal. The process reward weight is intentionally low (0.1) because keyword presence is a weak proxy for actual reasoning quality. Over-weighting it leads to keyword stuffing without substance.

## Error Handling

- **Cross-modal contradiction without resolution:** If the reasoning chain identifies conflicting signals across modalities but fails to resolve them, flag the output as low-confidence and request human review. Do not default to the text modality—it is the most easily manipulated (sarcasm, lies, social performance).
- **Empty think blocks:** If the model produces `<think></think><answer>label</answer>`, the structure reward should be 0 (not 1). Presence of tags without content is a degenerate solution.
- **Level misclassification:** If a Level 3 task is treated as Level 1 (direct classification without reasoning), the output will appear correct on easy cases but fail on adversarial ones. Always verify the task's cognitive level before choosing the chain depth.
- **Reward hacking in TMPO:** Models may learn to insert ToM keywords without genuine reasoning. Mitigate by keeping process reward weight low and relying primarily on content + consistency rewards.

## Limitations

- The ToM-guided chain adds significant token overhead. For high-throughput systems doing simple sentiment classification (Level 1), the full three-level chain is unnecessary overhead—use it only for Level 2+ tasks.
- The technique assumes access to multimodal inputs. For text-only emotion analysis, Levels 1 and 2 still apply but Level 3 reasoning about visual/audio cues is not possible.
- TMPO training requires ground-truth reasoning chains, which are expensive to produce (the paper used LLM generation + human correction). For custom domains, expect significant annotation effort.
- Second-order ToM reasoning ("A believes that B believes X") is inherently speculative. The framework makes this speculation explicit and auditable, but it does not guarantee correctness.
- Cultural context heavily influences emotion expression and interpretation. The framework does not include a cultural calibration step—practitioners working across cultures should add one.

## Reference

**Paper:** "Unveiling the Cognitive Compass: Theory-of-Mind-Guided Multimodal Emotion Reasoning" (Luo et al., ICLR 2026) — [arXiv:2602.00971](https://arxiv.org/abs/2602.00971v1). Focus on Section 3 (ToM-guided reasoning chain design), Section 4 (TMPO reward formulation), and Table 2 (per-level benchmark results showing Level 3 deficits).