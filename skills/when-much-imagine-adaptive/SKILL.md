---
name: "when-much-imagine-adaptive"
description: "Adaptive test-time scaling framework that decides WHEN and HOW MUCH to invoke expensive generative steps (world models, tool calls, API queries) before committing compute. Based on the AVIC (Adaptive Visual Imagination with Confidence) paper. Use when: 'build an adaptive pipeline that skips unnecessary generation steps', 'add confidence gating before expensive API calls', 'implement adaptive test-time compute scaling', 'reduce wasted inference by checking sufficiency first', 'build a system that decides whether to call a model or skip it', 'implement selective tool invocation with confidence checks'."
---

# Adaptive Test-Time Scaling: Decide When and How Much to Invoke Expensive Compute

This skill teaches you to implement the AVIC (Adaptive Visual Imagination with Confidence) pattern from Yu et al. (2026): a three-phase pipeline where a system **assesses whether current evidence is sufficient**, **selectively invokes expensive generation/inference**, and **adaptively scales the amount of compute** based on confidence. The core insight is that indiscriminate invocation of expensive steps (world models, tool calls, API queries, LLM chains) often wastes compute and can *degrade* output quality by introducing misleading evidence. AVIC's gating mechanism reduced world-model calls by 40-60% while matching or exceeding fixed-budget baselines on spatial reasoning benchmarks.

## When to Use

- When building a pipeline that calls an expensive model (image generation, LLM reasoning chain, database query, web search) and you want to skip unnecessary calls
- When implementing a multi-step agent where some steps are conditionally useful and you need a principled way to gate them
- When designing a system that generates multiple candidate outputs (views, samples, tool results) and you need to decide how many to generate per query
- When optimizing inference cost by avoiding redundant or harmful augmentation steps
- When building retrieval-augmented generation and some queries don't need retrieval at all
- When creating an adaptive compute budget system that allocates more resources to harder inputs and less to easy ones

## Key Technique

**The problem**: Fixed-budget pipelines apply the same expensive steps to every input. The AVIC paper shows this has three failure modes: (1) wasting compute on inputs where current evidence already suffices, (2) generating too few augmentations for genuinely hard inputs, and (3) actively degrading performance when generated content introduces misleading artifacts. On the SAT-Real benchmark, indiscriminate imagination *decreased* accuracy on 15-20% of questions compared to no imagination at all.

**The solution**: AVIC inserts a **sufficiency gate** before any expensive step. The gate is a lightweight confidence assessment that asks: "Given what I already have, can I answer reliably?" If confidence is high, skip generation entirely. If confidence is low, invoke generation and **scale the budget adaptively** -- harder inputs get more generated samples. The confidence signal comes from the same MLLM that will consume the results, making it cheap to compute. On SAT-Real with GPT-4.1, this approach improved accuracy from 74.0% to 79.3% while using substantially fewer world-model calls than always-on baselines.

**The architecture has three phases**: (1) **Evidence Assessment** -- score the sufficiency of current input using a confidence probe, (2) **Selective Invocation** -- compare the confidence score against a threshold to decide whether to invoke the expensive step, (3) **Scaled Imagination** -- if invoked, determine the number of augmentations dynamically based on confidence deficit (lower confidence = more augmentations). Results from augmentations are aggregated via majority voting or weighted ensemble before the final answer.

## Step-by-Step Workflow

1. **Define the expensive step to gate.** Identify the costly operation in your pipeline: a world-model call, an LLM chain-of-thought, a tool invocation, an API query, or a retrieval step. This is what you will conditionally skip or scale.

2. **Implement the confidence probe.** Before the expensive step, query the primary model for a confidence score on the current input alone. Use an explicit prompt like: "Given only the current information, how confident are you in answering this question? Rate 0-100 and explain what additional information would help." Parse the numeric score.

3. **Set the sufficiency threshold.** Define a confidence threshold T (start with T=70 for most tasks). If the confidence score >= T, skip the expensive step entirely and proceed with the answer from current evidence. This threshold is calibrated on a small validation set.

4. **Implement the adaptive budget function.** When confidence < T, compute the generation budget as: `budget = min(MAX_BUDGET, ceil(BASE_BUDGET * (T - confidence) / T))`. This allocates more augmentations to lower-confidence inputs. Typical values: BASE_BUDGET=3, MAX_BUDGET=8.

5. **Invoke the expensive step with the computed budget.** Generate `budget` augmentations (novel views, retrieved documents, tool call variants, reasoning chains). Each augmentation should provide genuinely different evidence -- not redundant copies.

6. **Process each augmentation independently.** Run the primary model on each augmented input separately to produce candidate answers. This prevents cross-contamination between augmentations.

7. **Aggregate candidates via majority voting or weighted ensemble.** Count answer frequencies across all candidates (including the original non-augmented answer). Use majority vote for classification tasks or weighted average for numeric outputs. Weight by per-candidate confidence if available.

8. **Implement a contradiction detector.** If augmented candidates disagree strongly with the original answer (e.g., vote split is near 50/50), flag the input as ambiguous rather than forcing a low-confidence answer. This prevents the "misleading imagination" failure mode.

9. **Log and monitor skip rates and accuracy.** Track what percentage of inputs skip the expensive step, and compare accuracy between skipped and augmented subsets. A healthy system skips 30-60% of inputs with equal or higher accuracy on the skipped subset.

10. **Calibrate thresholds on a validation set.** Run the full pipeline on held-out data with varying T values. Plot accuracy vs. compute cost (number of expensive calls). Choose the T that maximizes accuracy per unit cost on your specific task.

## Concrete Examples

**Example 1: Adaptive RAG -- Skip Retrieval When the LLM Already Knows**

User: "Build a QA pipeline that only retrieves documents when the model isn't confident enough to answer directly."

Approach:
1. Define the pipeline: User question -> Confidence check -> (optional) Retrieval -> Answer generation
2. Implement the confidence probe:
```python
def assess_sufficiency(question: str, llm) -> tuple[float, str]:
    """Ask the LLM if it can answer without retrieval."""
    prompt = f"""Question: {question}

Without searching any external sources, can you answer this question?
Rate your confidence from 0 to 100.
Format: CONFIDENCE: <number>
REASONING: <why you are or aren't confident>
ANSWER: <your best answer or "need more information">"""
    response = llm.generate(prompt)
    confidence = parse_confidence(response)  # extract numeric score
    preliminary_answer = parse_answer(response)
    return confidence, preliminary_answer
```

3. Implement the gated pipeline:
```python
SUFFICIENCY_THRESHOLD = 70
BASE_RETRIEVAL_DOCS = 3
MAX_RETRIEVAL_DOCS = 10

def adaptive_qa(question: str, llm, retriever) -> str:
    confidence, preliminary = assess_sufficiency(question, llm)

    if confidence >= SUFFICIENCY_THRESHOLD:
        # Skip retrieval entirely -- current knowledge suffices
        return preliminary

    # Scale retrieval budget by confidence deficit
    budget = min(MAX_RETRIEVAL_DOCS,
                 ceil(BASE_RETRIEVAL_DOCS * (SUFFICIENCY_THRESHOLD - confidence)
                      / SUFFICIENCY_THRESHOLD))

    docs = retriever.search(question, top_k=budget)
    augmented_prompt = build_rag_prompt(question, docs)
    augmented_answer = llm.generate(augmented_prompt)

    # Compare: if augmented answer contradicts preliminary with similar
    # confidence, flag for human review
    if contradicts(preliminary, augmented_answer) and confidence > 40:
        return flag_for_review(question, preliminary, augmented_answer)

    return augmented_answer
```

Output: A pipeline that skips retrieval for ~40% of factual questions the model already knows, retrieves 3 docs for moderately uncertain questions, and retrieves up to 10 docs for highly uncertain ones.

---

**Example 2: Adaptive Multi-Tool Agent -- Gate Expensive Tool Calls**

User: "My agent calls a code execution sandbox for every coding question, even trivial ones. Make it skip the sandbox when it's confident in the answer."

Approach:
1. Before each sandbox call, insert a sufficiency check
2. Use the LLM's own confidence as the gate signal
3. Scale sandbox iterations for harder problems

```python
class AdaptiveCodeAgent:
    def __init__(self, llm, sandbox, threshold=75, max_iterations=5):
        self.llm = llm
        self.sandbox = sandbox
        self.threshold = threshold
        self.max_iterations = max_iterations

    def solve(self, problem: str) -> dict:
        # Phase 1: Evidence Assessment
        assessment = self.llm.generate(f"""Problem: {problem}

Can you solve this without running code? Rate confidence 0-100.
CONFIDENCE: <number>
SOLUTION: <your answer>
NEEDS_EXECUTION: <what you'd want to test if you could run code>""")

        confidence = parse_confidence(assessment)
        solution = parse_solution(assessment)

        if confidence >= self.threshold:
            return {"answer": solution, "method": "direct", "sandbox_calls": 0}

        # Phase 2: Scaled Imagination
        iterations = min(self.max_iterations,
                         max(1, ceil(3 * (self.threshold - confidence)
                                     / self.threshold)))

        results = []
        for i in range(iterations):
            exec_result = self.sandbox.run(solution if i == 0
                                           else self.refine(problem, results))
            results.append(exec_result)
            if exec_result.all_tests_pass:
                break  # Early exit on success

        # Phase 3: Aggregation
        final = self.aggregate(problem, solution, results)
        return {"answer": final, "method": "sandbox",
                "sandbox_calls": len(results)}
```

Output: Agent that directly answers "What does `len([1,2,3])` return?" without sandbox (confidence 95), uses 1 sandbox call for moderate problems, and up to 5 iterations for complex algorithmic challenges.

---

**Example 3: Adaptive Image Generation Pipeline -- Skip Unnecessary Augmentations**

User: "I'm building a visual QA system that generates alternate camera angles. Some questions don't need extra views. Make it adaptive."

Approach:
1. Probe the VLM for spatial confidence on the original image
2. Only generate novel views when spatial reasoning is genuinely ambiguous
3. Scale the number of views by difficulty

```python
def adaptive_visual_qa(image, question, vlm, world_model):
    # Phase 1: Assess if current view suffices
    probe = vlm.query(image, f"""{question}
Can you answer this from the current viewpoint alone?
CONFIDENCE: <0-100>
ANSWER: <your answer>
WHAT_VIEW_WOULD_HELP: <describe a viewpoint that would help, or "none needed">""")

    confidence = parse_confidence(probe)

    if confidence >= 80:
        return parse_answer(probe)  # Current view is sufficient

    # Phase 2: Generate novel views scaled to difficulty
    n_views = min(6, max(1, ceil(4 * (80 - confidence) / 80)))
    viewpoints = plan_viewpoints(probe.what_view_would_help, n_views)
    novel_images = [world_model.generate(image, vp) for vp in viewpoints]

    # Phase 3: Aggregate across views
    answers = [vlm.query(img, question) for img in [image] + novel_images]
    return majority_vote(answers)
```

Output: Questions like "What color is the front of the car?" answered from the original view (confidence 90, zero generation calls). Questions like "How many windows are on the back wall?" trigger 3-4 novel views from behind.

## Best Practices

- **Do:** Start with a generous sufficiency threshold (70-80) and tighten based on validation data. Being too aggressive with skipping is worse than being too conservative.
- **Do:** Include the original (non-augmented) answer in the aggregation vote. It provides a useful baseline signal even when augmentation is triggered.
- **Do:** Log per-input confidence, skip/invoke decisions, and final accuracy. This data is essential for threshold calibration and debugging.
- **Do:** Use early exit within scaled imagination -- if the first augmentation produces a high-confidence answer, skip remaining budget.
- **Avoid:** Using binary yes/no gates. Continuous confidence scores enable proportional budget scaling, which is the key advantage over fixed strategies.
- **Avoid:** Setting MAX_BUDGET too high. The paper shows diminishing returns beyond 6-8 augmentations, and quality can degrade past that point due to accumulated noise.
- **Avoid:** Treating the confidence probe as ground truth. LLM confidence is a heuristic signal, not a calibrated probability. Always validate skip decisions against actual accuracy on held-out data.

## Error Handling

- **Confidence parsing fails**: Default to invoking the expensive step with BASE_BUDGET. Never skip on parse failure.
- **World model / tool returns garbage**: Implement a quality check on augmented outputs before including them in aggregation. Discard augmentations that are clearly corrupted (blank images, error messages, nonsensical text).
- **Vote ties in aggregation**: Fall back to the highest-confidence individual answer rather than random tie-breaking.
- **Threshold too aggressive (high skip rate, low accuracy on skipped)**: Lower the threshold by 10 points and re-evaluate. Monitor the accuracy gap between skipped and augmented subsets.
- **All augmentations disagree with original**: This signals a fundamentally ambiguous input. Return the original answer with a low-confidence flag rather than trusting noisy augmentations.

## Limitations

- **Confidence calibration**: LLMs are notoriously miscalibrated -- they may report high confidence on wrong answers and low confidence on correct ones. The threshold must be tuned per model and per task.
- **Overhead of the probe**: The sufficiency check itself costs one LLM call. For extremely cheap downstream steps, the probe overhead may not justify the savings.
- **Not suitable for safety-critical paths**: If every input *must* receive augmentation for safety or compliance reasons, adaptive skipping is inappropriate.
- **Domain-specific tuning required**: The optimal threshold, budget function, and aggregation strategy vary significantly across tasks. There is no universal configuration.
- **Limited to tasks with measurable confidence signals**: Works best when the model can articulate uncertainty. Tasks where uncertainty is opaque (e.g., subtle visual artifacts) may not benefit.

## Reference

**Paper**: Yu, Zhang, Wang, Yoon & Yao. "When and How Much to Imagine: Adaptive Test-Time Scaling with World Models for Visual Spatial Reasoning." arXiv:2602.08236v1 (2026). [https://arxiv.org/abs/2602.08236v1](https://arxiv.org/abs/2602.08236v1)

**Key takeaway**: The paper's Figure 3 and Table 2 demonstrate that 30-50% of spatial reasoning questions can be answered without any world-model augmentation, and that scaling imagination proportionally to confidence deficit achieves better accuracy-efficiency tradeoffs than any fixed budget. The three-phase pattern (assess, gate, scale) generalizes beyond visual reasoning to any pipeline with expensive conditional steps.