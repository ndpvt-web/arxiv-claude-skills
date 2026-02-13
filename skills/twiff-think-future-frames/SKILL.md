---
name: "twiff-think-future-frames"
description: |
  Apply the TwiFF "Think With Future Frames" technique for dynamic visual reasoning tasks.
  Implements iterative future-state generation interleaved with textual reasoning to solve
  temporal prediction, instruction-following, and motion-understanding problems.
  Trigger phrases:
  - "reason about what happens next in this video/scene"
  - "predict future frames from this image"
  - "build a visual chain-of-thought pipeline"
  - "implement iterative visual reasoning"
  - "dynamic visual QA system"
  - "temporal reasoning with generated frames"
---

# TwiFF: Think With Future Frames — Dynamic Visual Reasoning

This skill teaches Claude to apply the TwiFF technique from Liu et al. (2026) for building systems that reason about dynamic visual scenarios by iteratively generating hypothetical future frames and interleaving them with textual chain-of-thought reasoning. Instead of reasoning abstractly about "what happens next," the approach concretizes each reasoning step by producing a plausible future visual state, analyzing it, then using that analysis to guide the next generation step. This is directly applicable when building video understanding pipelines, visual QA systems for dynamic content, robotics planning interfaces, or any system that needs to reason about temporal visual sequences.

## When to Use

- When the user asks to build a system that predicts what happens next in a video or image sequence
- When implementing visual chain-of-thought (VCoT) reasoning that must handle temporal dynamics, not just static images
- When designing a pipeline that alternates between visual generation and textual analysis for multi-step reasoning
- When building evaluation benchmarks for dynamic visual QA (instruction-following, prediction, camera motion tasks)
- When the user needs to structure training data as interleaved text-and-image reasoning trajectories
- When integrating a video generation model with an image comprehension model into a unified reasoning system
- When implementing a prompt format that uses special tokens (`<rimage>`) to mark where generated visual content is inserted into a reasoning chain

## Key Technique

**Visual Chain-of-Thought for Dynamic Scenes.** Standard chain-of-thought prompting produces text-only reasoning steps. Visual CoT (VCoT) extends this by inserting images as intermediate reasoning artifacts. However, most VCoT methods operate on static scenes — they annotate or crop existing images. TwiFF's core insight is that for dynamic scenarios (video, robotics, physics), the most useful intermediate reasoning artifact is a *generated future frame* — a plausible visualization of what the scene looks like after an action or time step. The model literally "thinks" by imagining the future.

**Iterative Interleaved Generation.** The TwiFF architecture implements a loop: (1) receive an input frame and question, (2) generate textual reasoning about the current state, (3) produce a future frame depicting the next plausible state, (4) reason about that generated frame, (5) repeat until the answer emerges. The output format is `{thought_0}\n<rimage>\n{thought_1}\n<rimage>\n{thought_2}\n<ans>{answer}</ans>`, where each `<rimage>` token is replaced by a generated image. The number of iterations is controlled by a `MAX_ROUND` parameter, allowing variable-depth reasoning based on task complexity.

**Two-Stage Unification.** The model is trained in two stages. Stage 1 builds foundational visual comprehension and basic frame generation capability. Stage 2 jointly fine-tunes these abilities so that video generation and image understanding reinforce each other — the generator learns to produce frames that are maximally useful for reasoning, and the reasoner learns to extract information from generated (imperfect) frames. This synergy is what distinguishes TwiFF from pipeline approaches that simply chain separate models.

## Step-by-Step Workflow

1. **Define the task category.** Classify the dynamic reasoning problem as one of three types: *instruction* (understanding and following visual directions), *prediction* (forecasting future states), or *camera motion* (reasoning about viewpoint changes). This determines the prompt template and the kind of future frames to generate.

2. **Structure the input as frame + question.** Format the input following TwiFF's convention: `frame_1\n<image>\n{question}`. The `<image>` token is a placeholder for the actual input frame. If multiple input frames are available, index them (`frame_1`, `frame_2`, etc.) and reference specific frame indices.

3. **Design the interleaved reasoning format.** Define the output structure as alternating text reasoning and `<rimage>` tokens: `{reasoning_step_0}\n<rimage>\n{reasoning_step_1}\n<rimage>\n...\n<ans>{final_answer}</ans>`. Each `<rimage>` marks where a generated future frame is inserted. Plan for 2-4 reasoning rounds for typical tasks; use `MAX_ROUND` up to 8 for complex multi-step scenarios.

4. **Implement the generation loop.** Build the iterative pipeline: at each round, (a) pass the accumulated context (input frame, prior reasoning, prior generated frames) to the model, (b) decode the next text reasoning segment, (c) if an `<rimage>` token is produced, invoke the video generation component to produce the next future frame, (d) append the generated frame to the context and continue decoding.

5. **Select frame indices strategically.** When constructing training data, choose `recon_frames` (target frames to reconstruct) that are temporally informative — not adjacent frames, but frames that capture meaningful state transitions. For an 8-frame video, input frame index 1 and reconstruction targets at indices 3 and 6 is a good default spread.

6. **Build the training data in JSONL format.** Each sample should follow the structure:
   ```json
   {
     "video": "/path/to/video.mp4",
     "frames": [1],
     "recon_frames": [3, 6],
     "conversations": [
       {"from": "system", "value": "{task_prompt}"},
       {"from": "human", "value": "frame_1\n<image>\n{question}"},
       {"from": "gpt", "value": "{reasoning}\n<rimage>\n{reasoning}\n<rimage>\n<ans>{answer}</ans>"}
     ]
   }
   ```

7. **Implement two-stage training.** Stage 1: train on visual comprehension and basic frame reconstruction independently. Stage 2: jointly fine-tune so the generator and reasoner co-adapt. Use the training scripts in `train_scripts/Stage1.sh` as a template for multi-GPU distributed training.

8. **Run inference with controlled depth.** Execute inference with an explicit `max_round` parameter to control reasoning depth:
   ```bash
   python scripts/inference.py \
     --max_round 8 \
     --model_dir models/TwiFF-7B \
     --checkpoint_file model.safetensors \
     --QA_file output/demo.jsonl
   ```

9. **Evaluate with dual metrics.** Score outputs on two axes: (a) *reasoning trajectory plausibility* — are the intermediate generated frames and reasoning steps coherent and logically connected? (b) *answer correctness* — is the final answer right? Use GPT-based judgment (as in TwiFF-Bench) or human evaluation for trajectory quality.

10. **Iterate on failure cases.** When the model produces implausible future frames, diagnose whether the issue is generation quality (Stage 1 problem) or reasoning-generation misalignment (Stage 2 problem). Increase `MAX_ROUND` if answers are correct but reasoning is shallow; decrease it if the model hallucinates in later rounds.

## Concrete Examples

**Example 1: Building a Video Prediction QA Pipeline**

User: "I want to build a system that takes a single frame from a cooking video and answers 'What will the chef do next?'"

Approach:
1. Classify as a *prediction* task
2. Structure input: `frame_1\n<image>\nWhat will the chef do next?`
3. Design 3-round reasoning:
   - Round 1: Analyze current frame ("The chef is holding a knife above an onion on a cutting board")
   - Round 2: Generate future frame showing the cut, reason about it ("The chef has begun slicing the onion into rings")
   - Round 3: Generate frame showing completion, produce answer
4. Format output: `The chef is positioned with a knife above an onion.\n<rimage>\nThe onion is now being sliced into even rings.\n<rimage>\n<ans>The chef will dice the onion into rings and likely add them to the pan visible in the background.</ans>`

Output structure:
```
thought_0: "The chef is holding a knife positioned above a white onion on a wooden cutting board. A heated pan is visible to the right."
[generated_frame_1: future state showing knife mid-cut]
thought_1: "The chef has begun cutting the onion into uniform rings. The pan appears to have oil heating."
[generated_frame_2: future state showing diced onion being transferred]
answer: "The chef will slice the onion into rings and transfer them to the heated pan for sauteing."
```

**Example 2: Constructing a TwiFF-Style Training Dataset**

User: "I have 50,000 video clips of robotic arm movements. How do I format them for TwiFF training?"

Approach:
1. Extract 8 uniformly-spaced frames per video clip
2. Designate frame 1 as input, frames 3 and 6 as reconstruction targets
3. Write question-answer pairs for each clip (e.g., "What will the robot arm grasp next?")
4. Generate interleaved reasoning text that describes the state at each target frame
5. Output as JSONL

Output:
```jsonl
{
  "video": "clips/robot_grasp_00421.mp4",
  "frames": [1],
  "recon_frames": [3, 6],
  "conversations": [
    {"from": "system", "value": "You are a visual reasoning assistant. Analyze the scene and predict the robot's next actions by thinking through future states step by step."},
    {"from": "human", "value": "frame_1\n<image>\nWhat object will the robot arm pick up and where will it place it?"},
    {"from": "gpt", "value": "The robot arm is currently positioned above a red cube and a blue cylinder on a table.\n<rimage>\nThe gripper has closed around the red cube and is lifting it upward.\n<rimage>\n<ans>The robot arm will grasp the red cube and place it on the elevated platform to the right of the workspace.</ans>"}
  ]
}
```

**Example 3: Implementing the Evaluation Dual-Metric Scorer**

User: "How do I evaluate my TwiFF-style model's outputs on both reasoning quality and answer correctness?"

Approach:
1. Collect model outputs in `model_response_unscore.jsonl` format (list of alternating text segments and image paths)
2. For answer correctness: extract text within `<ans>...</ans>` tags, compare against ground truth using GPT-based semantic matching
3. For trajectory plausibility: send the full interleaved reasoning (text + generated image descriptions) to a judge LLM with a rubric assessing logical coherence, temporal consistency, and visual plausibility
4. Report both scores separately — a model can get the right answer with bad reasoning or vice versa

Output evaluation script pattern:
```python
def evaluate_twiff_output(response, ground_truth):
    # Extract answer
    answer = extract_between_tags(response, "<ans>", "</ans>")

    # Extract reasoning trajectory
    segments = response.split("<rimage>")
    reasoning_steps = [s.strip() for s in segments if s.strip() and "<ans>" not in s]

    # Score answer correctness (0-1)
    answer_score = judge_answer_correctness(answer, ground_truth)

    # Score trajectory plausibility (0-1)
    trajectory_score = judge_trajectory_plausibility(reasoning_steps)

    return {"answer": answer_score, "trajectory": trajectory_score}
```

## Best Practices

- **Do:** Use temporally spread frame indices for reconstruction targets (e.g., frames 3 and 6 out of 8) rather than consecutive frames — this forces the model to reason about meaningful state changes, not trivial interpolations.
- **Do:** Start with `MAX_ROUND=4` for most tasks and increase only if reasoning depth is insufficient. More rounds means more opportunities for error accumulation.
- **Do:** Evaluate trajectory plausibility separately from answer correctness. A model that gets the right answer through implausible reasoning is fragile and will fail on harder problems.
- **Do:** Include the system prompt in training data to steer reasoning style per task category (instruction vs. prediction vs. camera motion).
- **Avoid:** Generating future frames for purely static reasoning tasks (e.g., "What color is the car?"). TwiFF is designed for temporal dynamics — use standard VCoT or direct QA for static scenes.
- **Avoid:** Using more than 8 reasoning rounds in practice. The original experiments show diminishing returns and increased hallucination beyond this depth.

## Error Handling

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| Generated frames are blurry or incoherent | Stage 1 video generation undertrained | Increase Stage 1 training duration; verify video generation backbone quality independently |
| Reasoning text contradicts generated frames | Stage 2 co-adaptation insufficient | Extend Stage 2 joint fine-tuning; add explicit consistency loss |
| Model always generates max rounds | `MAX_ROUND` too high or model hasn't learned to terminate early | Reduce `MAX_ROUND`; add early-stopping examples in training data with `<ans>` before max rounds |
| Answer correct but reasoning vacuous | Model shortcutting through the reasoning steps | Increase weight on trajectory plausibility in evaluation; add harder training examples that require genuine multi-step reasoning |
| CUDA OOM during inference | Generated frames consuming too much memory | Process frames at lower resolution; reduce batch size; use flash attention (`flash_attn==2.5.8`) |

## Limitations

- **Requires video generation capability.** The full TwiFF pipeline depends on a video/image generation model to produce future frames. Text-only LLMs cannot implement the visual generation component — they can only simulate the reasoning format with text descriptions of hypothetical frames.
- **Static scenes gain nothing.** If the visual content has no temporal dynamics (e.g., document images, static diagrams), this technique adds complexity without benefit. Use standard VCoT or direct reasoning instead.
- **Hallucination compounds.** Each generated future frame is imperfect. Errors in early frames propagate through later reasoning rounds. For high-stakes applications, validate generated frames against physical constraints.
- **Compute intensive.** Generating frames at each reasoning step is significantly more expensive than text-only CoT. Budget 3-5x the inference cost of a text-only approach.
- **Training data requires video.** The JSONL format expects video files with extractable frames. Adapting this to non-video domains (e.g., physics simulations, game states) requires custom frame extraction.

## Reference

**Paper:** Liu et al., "TwiFF (Think With Future Frames): A Large-Scale Dataset for Dynamic Visual Reasoning," arXiv:2602.10675v1, 2026. [https://arxiv.org/abs/2602.10675v1](https://arxiv.org/abs/2602.10675v1)

**Code & Data:** [https://github.com/LiuJunhua02/TwiFF](https://github.com/LiuJunhua02/TwiFF)

**What to look for:** Section 3 (TwiFF-2.7M dataset construction and the interleaved reasoning format), Section 4 (the two-stage training procedure and unified architecture), and Table 2 (benchmark comparisons showing TwiFF's improvement over text-only CoT and static VCoT baselines).