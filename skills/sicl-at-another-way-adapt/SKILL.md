---
name: "sicl-at-another-way-adapt"
description: "Adapt auditory LLMs to low-resource speech/audio tasks using Speech In-Context Learning Adaptation Training (SICL-AT). Strengthens a model's ability to learn from a few audio demonstrations at inference time by post-training on episodic ICL episodes from high-resource data. Use when: 'adapt speech model to new domain', 'few-shot audio classification', 'low-resource ASR adaptation', 'in-context learning for audio LLM', 'improve speech model with few examples', 'post-training recipe for audio tasks'."
---

# SICL-AT: Adapt Auditory LLMs to Low-Resource Tasks via In-Context Learning Training

This skill enables Claude to design and implement **Speech In-Context Learning Adaptation Training (SICL-AT)** pipelines that strengthen auditory LLMs' ability to adapt to new speech and audio tasks using only a handful of demonstrations at inference time. Instead of brittle direct fine-tuning on scarce labeled data, SICL-AT post-trains the model on episodic ICL episodes constructed from high-resource speech data (ASR, speech translation), teaching the model to condition its outputs on demonstration audio-label pairs. The resulting model generalizes this ICL capability to unseen domains including child speech recognition, audio understanding, and multilingual tasks.

## When to Use

- When the user needs to adapt an auditory LLM (e.g., Qwen2.5-Omni, MiMo-Audio) to a **low-resource speech or audio domain** where labeled data is scarce (< 1,000 samples)
- When direct fine-tuning on a small target dataset causes overfitting or degrades performance on other tasks
- When the user wants a model that can adapt at **inference time** by conditioning on a few in-domain audio demonstrations, without retraining per task
- When building a speech pipeline for **domain-shifted data** (e.g., children's speech, accented speech, noisy environments) where the training distribution doesn't match deployment
- When the user asks to improve few-shot audio classification, audio reasoning, or speech question-answering on an auditory LLM
- When designing a post-training recipe that improves ICL capability across multiple downstream audio tasks simultaneously

## Key Technique

**The Problem with Direct Fine-Tuning:** When labeled in-domain data is scarce or distribution-shifted, fine-tuning an auditory LLM directly on that data tends to overspecialize. For example, fine-tuning on a children's speech corpus (RSR) improves RSR performance but catastrophically degrades performance on another children's ASR benchmark (MyST), jumping WER from 14.25 to 29.47. The model memorizes the narrow training distribution instead of learning to generalize.

**SICL-AT's Solution — Episodic ICL Training:** SICL-AT reformulates post-training as episodic in-context learning. During training, each step samples a task, retrieves k demonstration pairs (audio + label) from a pool, concatenates them before a query, and trains the model to predict the query's label conditioned on those demonstrations. Only lightweight LoRA adapters (rank 8, alpha 32) are updated, preserving the base model's broad capabilities. The training data comes entirely from high-resource tasks (CommonVoice ASR, CoVoST2 speech translation), but the learned ICL capability transfers to unseen tasks like audio understanding and reasoning.

**Why It Works:** By training on the *meta-task* of "use these demonstrations to inform your prediction" rather than on any specific domain, SICL-AT teaches the model a generalizable adaptation mechanism. At inference time, you provide 1-8 task-relevant audio demonstrations, and the model uses them to calibrate its output — no gradient updates needed. This consistently outperforms direct fine-tuning: on children's ASR (RSR), SICL-AT achieves 16.95 WER vs. 31.09 for fine-tuning; on audio understanding (MMAU), it reaches 73.4% vs. 66.9% zero-shot.

## Step-by-Step Workflow

1. **Select the base auditory LLM.** Choose a multimodal model that accepts interleaved audio and text inputs (e.g., Qwen2.5-Omni, MiMo-Audio). Verify it supports multi-turn or concatenated audio inputs in a single context window — this is essential for ICL.

2. **Assemble high-resource training data as episodic pools.** Gather large labeled speech datasets that will serve as demonstration sources. Recommended: English ASR (CommonVoice, ~16K samples), Speech Translation (CoVoST2, ~37K samples across language pairs). Optionally include Speech QA (MMSU, ~5K samples) if targeting audio reasoning tasks. Split each into a query set and a demonstration pool.

3. **Implement demonstration retrieval via TICL.** For each query instance, retrieve k nearest-neighbor demonstrations from the pool using text-embedding similarity (encode transcriptions/labels, use cosine-KNN). This ensures demonstrations are semantically relevant to the query, not random. Use k=1 to 8 depending on context length budget.

4. **Format episodic training episodes.** Construct each training input as: `[demo_1_audio, demo_1_label, demo_2_audio, demo_2_label, ..., demo_k_audio, demo_k_label, query_audio]` with the training target being `query_label`. The concatenation order matters — demonstrations precede the query.

5. **Configure LoRA-only training.** Attach LoRA adapters (rank 8, alpha 32) to the LLM backbone. Freeze all other parameters including the audio encoder. This prevents catastrophic forgetting of base capabilities while learning ICL behavior.

6. **Train with multi-task episodic sampling.** At each training step: (a) sample a task index uniformly from available tasks, (b) sample a query from that task's query set, (c) retrieve k demonstrations via TICL, (d) compute cross-entropy loss on the query label conditioned on the full episode. Optimize only the LoRA parameters.

7. **Evaluate with progressive task configurations.** Train three checkpoints with increasing task diversity: SICL-AT1 (ASR only), SICL-AT2 (ASR + ST), SICL-AT3 (ASR + ST + SQA). Evaluate each on held-out low-resource tasks to find the best training mix. More diverse training helps audio reasoning but may slightly degrade pure ASR.

8. **Deploy with inference-time demonstration retrieval.** At inference, for each new audio query: retrieve k in-domain demonstrations from a small labeled pool (even 50-100 examples suffice), format them as the episode prefix, and run a single forward pass. No fine-tuning is needed at deployment.

9. **Measure with capped utterance-level metrics.** For ASR tasks, use utterance-level WER capped at 1.0 to prevent outlier utterances from dominating. For classification/reasoning tasks, use accuracy. Compare against zero-shot (no demos), Vanilla ICL (demos without SICL-AT training), and direct fine-tuning baselines.

10. **Iterate on demonstration selection strategy.** If performance plateaus, experiment with: (a) increasing k (more demos per episode), (b) diversifying the demonstration pool, (c) using leave-one-out for small evaluation sets to prevent data leakage, (d) mixing random and KNN-retrieved demos during training for robustness.

## Concrete Examples

**Example 1: Adapting ASR to Children's Speech**

```
User: I have a children's speech ASR model based on MiMo-Audio that performs
poorly (31% WER). I only have 200 labeled utterances of children's speech.
How do I adapt it without overfitting?

Approach:
1. Do NOT fine-tune directly on the 200 utterances — this will overspecialize.
2. Collect high-resource adult ASR data (CommonVoice English, ~16K samples).
3. Post-train with SICL-AT: create episodic ICL episodes from CommonVoice,
   using TICL retrieval with k=4 demonstrations per query.
4. Apply LoRA (rank 8, alpha 32) to MiMo-Audio's LLM layers only.
5. Train for the same number of steps as you would with direct fine-tuning.
6. At inference, retrieve 4 nearest demonstrations from your 200 children's
   speech samples for each test query using text-embedding KNN.

Output (expected improvement):
- Zero-shot baseline:           31.39 WER
- Direct fine-tune on 200 utt:  ~31.09 WER (marginal gain, degrades other domains)
- SICL-AT + 4 demos at test:    ~16.95 WER (46% relative reduction)
```

**Example 2: Few-Shot Audio Understanding/Reasoning**

```
User: I want to improve my auditory LLM on audio understanding tasks (MMAU
benchmark) but I have no labeled audio reasoning data for training. Can I
still improve performance?

Approach:
1. Use SICL-AT with ASR + Speech Translation + Speech QA training data
   (the SICL-AT3 configuration — none of these overlap with MMAU).
2. Format episodic training: for each step, sample one of three tasks,
   retrieve k=4 demonstrations, train the model to predict query labels
   conditioned on the demonstration prefix.
3. At MMAU evaluation time, provide k retrieved audio demonstrations
   from MMAU's few available examples (leave-one-out to avoid leakage).

Output (expected improvement):
- Zero-shot baseline:    66.90% accuracy
- Vanilla ICL (no training): ~68% accuracy
- SICL-AT3 + ICL demos:  73.40% accuracy (+6.5 points absolute)

Key insight: The ICL capability trained on unrelated speech tasks
(ASR, translation) transfers to audio reasoning.
```

**Example 3: Implementing the Training Loop in PyTorch**

```python
# Pseudocode for SICL-AT episodic training loop

from peft import LoraConfig, get_peft_model

# 1. Configure LoRA
lora_config = LoraConfig(r=8, lora_alpha=32, target_modules=["q_proj", "v_proj"])
model = get_peft_model(auditory_llm, lora_config)

# 2. Define task pools
tasks = {
    "asr": {"query_set": asr_queries, "demo_pool": asr_demos},
    "st":  {"query_set": st_queries,  "demo_pool": st_demos},
    "sqa": {"query_set": sqa_queries, "demo_pool": sqa_demos},
}

# 3. Episodic training
for step in range(num_steps):
    # Sample task
    task_name = random.choice(list(tasks.keys()))
    task = tasks[task_name]

    # Sample query
    query = random.choice(task["query_set"])

    # Retrieve k demonstrations via text-embedding KNN
    demos = retrieve_knn_demos(query, task["demo_pool"], k=4)

    # Build episode: [demo1_audio, demo1_label, ..., query_audio]
    input_sequence = []
    for demo in demos:
        input_sequence.extend([demo.audio, demo.label])
    input_sequence.append(query.audio)

    # Forward pass, loss on query.label only
    loss = model(input_sequence, target=query.label)
    loss.backward()
    optimizer.step()
```

## Best Practices

- **Do:** Use text-embedding KNN (TICL) for demonstration retrieval — semantically relevant demos consistently outperform random selection.
- **Do:** Start with SICL-AT1 (ASR-only training) and progressively add tasks. Monitor whether adding task diversity helps or hurts your specific target.
- **Do:** Cap utterance-level WER at 1.0 during evaluation to prevent outlier utterances from distorting aggregate metrics.
- **Do:** Use leave-one-out when the evaluation set is small — never include the query in its own demonstration pool.
- **Avoid:** Fine-tuning all model parameters. Only update LoRA adapters to prevent catastrophic forgetting of the base model's broad capabilities.
- **Avoid:** Using too many demonstrations (k > 8) — context window limits and diminishing returns make k=4 a strong default.
- **Avoid:** Assuming more training task diversity is always better. SICL-AT3 improves audio reasoning but slightly degrades pure ASR compared to SICL-AT1. Match your training mix to your target.

## Error Handling

- **Demonstrations degrade performance:** If adding ICL demos hurts a specific model, verify that the model architecture actually supports multi-audio-input contexts. Some auditory LLMs process only a single audio input per turn. SICL-AT requires interleaved multi-audio support.
- **Training divergence:** With LoRA rank 8 and alpha 32, the effective learning rate is amplified. If loss diverges, reduce learning rate or lower alpha. Monitor validation loss on a held-out split of the high-resource data.
- **Domain mismatch in demonstration retrieval:** If TICL retrieval returns irrelevant demos (e.g., due to domain gap between embeddings), fall back to random retrieval from the target domain pool. Random in-domain demos outperform KNN-retrieved out-of-domain ones.
- **Context window overflow:** With k=4 demos of long audio, the total input may exceed the model's context window. Truncate demonstrations to a maximum duration (e.g., 10 seconds each) or reduce k.

## Limitations

- **Requires multi-audio context support:** The base auditory LLM must be able to process multiple audio segments in a single input. Models limited to single-audio-per-turn cannot use this approach.
- **High-resource training data still needed:** SICL-AT is not zero-resource — it requires thousands of labeled samples from high-resource tasks (ASR, ST) for episodic training. The "low-resource" aspect applies only to the target task.
- **Inference cost scales with k:** Each demonstration adds audio encoding and context processing overhead. Inference latency increases linearly with the number of demonstrations.
- **Not all models benefit equally:** The paper shows consistent gains on MiMo-Audio and Qwen2.5-Omni, but models with weak baseline ICL capability may see smaller improvements.
- **No published training hyperparameters:** The paper does not report specific learning rates, batch sizes, or training schedules, so practitioners must tune these independently.

## Reference

**Paper:** [SICL-AT: Another way to adapt Auditory LLM to low-resource task](https://arxiv.org/abs/2601.18904v1) — Zheng et al., 2026. Look for Algorithm 1 (episodic training procedure), Table 2 (comprehensive results across tasks), and Section 4.3 (ablation across SICL-AT1/2/3 configurations showing how training task diversity affects downstream generalization).