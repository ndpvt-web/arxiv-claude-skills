---
name: "perfguard-performance-aware-agent-visual"
description: >
  Performance-aware multi-tool orchestration for visual content generation pipelines.
  Implements PerfGuard's three mechanisms (PASM, APU, CAPO) to select, score, and
  schedule AI image/video tools based on measured capability boundaries instead of
  generic descriptions. Use when: "build a visual generation pipeline", "orchestrate
  multiple image tools", "select the best AI model for this image task",
  "score and rank generation tools", "adaptive tool selection for AIGC",
  "performance-aware tool routing".
---

# PerfGuard: Performance-Aware Agent for Visual Content Generation

This skill enables Claude to design and implement multi-tool orchestration systems for AI-generated visual content (AIGC) that select tools based on **measured performance boundaries** rather than static descriptions. Derived from the PerfGuard framework (ICLR 2026), the approach replaces the naive assumption that "all tools work equally well" with a structured scoring, ranking, and adaptive update system across three mechanisms: Performance-Aware Selection Modeling (PASM), Adaptive Preference Update (APU), and Capability-Aligned Planning Optimization (CAPO).

## When to Use

- When the user is building a pipeline that routes tasks to multiple image generation or editing models (e.g., FLUX, SDXL, DALL-E, Midjourney APIs)
- When the user wants to automatically pick the best tool for a specific visual subtask (e.g., "which model handles spatial relationships better?")
- When the user needs a scoring/evaluation system for comparing AI-generated images across quality dimensions
- When the user asks to build a self-improving agent that learns which tools work best over time
- When the user wants to decompose complex visual tasks into subtasks matched to tool strengths
- When the user is implementing fallback/retry logic for visual generation where output quality is uncertain

## Key Technique

**The core insight:** Generic tool descriptions ("generates high-quality images") are useless for selection. PerfGuard replaces them with multi-dimensional performance scores measured on fine-grained capability axes. For text-to-image tools, these axes are: *color accuracy*, *shape fidelity*, *texture realism*, *2D spatial*, *3D spatial*, *numeracy* (correct object count), and *non-spatial semantics*. For image editing tools: *addition*, *removal*, *replacement*, *attribute alteration*, *motion change*, *style transfer*, and *background change*. Each tool gets a score (0.0-1.0) per dimension, producing a capability fingerprint.

**Selection works by weighted dot product.** Given a task, the system identifies which dimensions matter (e.g., a request involving "three red cups arranged left to right" weights numeracy, color, and 2D-spatial heavily). The preference score for each candidate tool is `sum(tool.score[dim] * task.weight[dim])`. Tools are ranked by this score; the top-N candidates execute. Results are evaluated via a multi-dimensional quality assessment (object count, position, attributes, style, background, semantic alignment) scored at three levels (L1=1.0, L2=0.66, L3=0.33). If the composite score falls below a threshold (default 0.8), the system re-plans and retries with alternative tools.

**The system learns from outcomes.** APU compares the predicted tool ranking against the actual quality ranking from execution. When discrepancies occur (predicted #1 actually performed worst), preference scores are updated. This closes the loop: the system gets better at tool selection over time without manual re-tuning. CAPO feeds these updated scores back into the planner, biasing subtask decomposition toward operations where strong tools exist rather than generating subtasks no tool can reliably handle.

## Step-by-Step Workflow

1. **Define capability dimensions for your tool domain.** For text-to-image: `[color, shape, texture, 2d_spatial, 3d_spatial, numeracy, non_spatial]`. For image editing: `[addition, removal, replacement, attribute_alter, motion_change, style_transfer, background_change]`. For video or other domains, define analogous axes.

2. **Build a tool registry with performance scores.** For each tool, populate a score dictionary across all dimensions. Initialize from benchmarks or manual evaluation. Store as JSON:
   ```json
   {
     "flux": {"color": 0.74, "shape": 0.57, "texture": 0.69, "2d_spatial": 0.63, "numeracy": 0.52, "non_spatial": 0.92},
     "sd3":  {"color": 0.81, "shape": 0.59, "texture": 0.73, "2d_spatial": 0.61, "numeracy": 0.48, "non_spatial": 0.91}
   }
   ```

3. **Implement the task analyzer (Analyst role).** Parse user requests to extract: the task type (generation vs. editing), target objects, spatial relationships, style requirements, and reference images. Output a structured goal representation.

4. **Implement dimension weight extraction.** From the analyzed task, compute a weight vector over capability dimensions. A task mentioning "three objects arranged in a grid" gets high weights on `numeracy` and `2d_spatial`. A style-transfer edit gets high weight on `style_transfer`. Use an LLM to classify or use keyword/rule-based mapping.

5. **Compute preference-weighted tool rankings.** For each candidate tool, calculate: `score = sum(tool_scores[dim] * task_weights[dim] for dim in dimensions)`. Sort tools descending by score. Select top-N (typically 2-3) for parallel execution.

6. **Execute and evaluate results.** Run selected tools. Score each output on evaluation dimensions: object category/count correctness, positional accuracy, attribute binding, style match, background fidelity, overall semantic alignment. Use three-tier scoring (L1=1.0 fully correct, L2=0.66 partially correct, L3=0.33 present but wrong). Composite score = `(mean(non_semantic_dims) + semantic_score) / 2`.

7. **Apply the quality threshold and retry loop.** If the best output scores below 0.8, feed the evaluation back to the planner to generate corrective subtasks (e.g., "fix object count" or "adjust color balance"). Re-execute with the next-ranked tools. Limit to 5 iterations.

8. **Update preference scores via APU.** After execution, compare predicted ranking to actual quality ranking. For each dimension where a tool underperformed expectations, decrease its score; where it overperformed, increase it. Use exponential moving average: `new_score = alpha * observed + (1-alpha) * old_score` with alpha ~0.1-0.3.

9. **Store experiences for retrieval.** Log each completed task as an experience record: `{tool_name, subtask, pre_tool, conditions, scores}`. Use CLIP embeddings or text similarity to retrieve relevant past experiences when planning new tasks.

10. **Feed updated preferences into planning (CAPO).** When decomposing future tasks, the planner should consult updated tool scores to avoid generating subtasks that fall into weak capability regions. If no tool scores above 0.5 on a needed dimension, restructure the plan to work around that limitation.

## Concrete Examples

**Example 1: Building a multi-model image generation router**

User: "I have access to FLUX, SDXL, and DALL-E APIs. Build a system that automatically picks the best model for each image generation request."

Approach:
1. Define the tool registry with initial scores per capability dimension
2. Build a FastAPI service with a `/generate` endpoint
3. Implement request analysis to extract dimension weights
4. Compute ranked tool selection per request
5. Execute with the top tool, evaluate output, update scores

Output structure:
```python
# tool_registry.py
TOOL_SCORES = {
    "flux": {
        "color": 0.74, "shape": 0.57, "texture": 0.69,
        "2d_spatial": 0.63, "3d_spatial": 0.45, "numeracy": 0.52,
        "non_spatial": 0.92
    },
    "sdxl": {
        "color": 0.81, "shape": 0.59, "texture": 0.73,
        "2d_spatial": 0.61, "3d_spatial": 0.50, "numeracy": 0.48,
        "non_spatial": 0.91
    },
    "dalle": {
        "color": 0.78, "shape": 0.65, "texture": 0.71,
        "2d_spatial": 0.58, "3d_spatial": 0.55, "numeracy": 0.60,
        "non_spatial": 0.88
    }
}

# selector.py
def select_tool(task_weights: dict[str, float], registry: dict) -> list[tuple[str, float]]:
    """Rank tools by preference-weighted score."""
    rankings = []
    for tool_name, scores in registry.items():
        total = sum(scores.get(dim, 0) * w for dim, w in task_weights.items())
        rankings.append((tool_name, total))
    return sorted(rankings, key=lambda x: x[1], reverse=True)

# analyzer.py
def extract_task_weights(prompt: str) -> dict[str, float]:
    """Map prompt characteristics to dimension weights."""
    weights = {dim: 0.1 for dim in DIMENSIONS}  # baseline
    if any(w in prompt.lower() for w in ["left", "right", "above", "below", "between"]):
        weights["2d_spatial"] = 0.8
    if any(w in prompt.lower() for w in ["three", "four", "five", "several", "many"]):
        weights["numeracy"] = 0.8
    if any(w in prompt.lower() for w in ["red", "blue", "green", "golden", "colorful"]):
        weights["color"] = 0.6
    # normalize
    total = sum(weights.values())
    return {k: v / total for k, v in weights.items()}
```

**Example 2: Self-improving image editing pipeline with fallback**

User: "Build an editing pipeline that tries multiple editing models and learns which works best for different edit types."

Approach:
1. Define editing-specific dimensions: addition, removal, replacement, attribute_alter, style_transfer, background_change
2. Implement the edit classifier to determine edit type from instruction
3. Select top-2 tools, execute both, evaluate with multi-dimensional scoring
4. Update tool preferences based on observed quality
5. Persist updated scores to disk for future sessions

Output structure:
```python
# evaluator.py
EVAL_DIMENSIONS = [
    "object_category", "object_count", "positional_accuracy",
    "attribute_binding", "style_match", "background_fidelity", "semantic_alignment"
]
TIER_SCORES = {"L1": 1.0, "L2": 0.66, "L3": 0.33}

def evaluate_output(original_prompt: str, image_path: str, llm_client) -> dict:
    """Score generated image across quality dimensions using VLM."""
    scores = {}
    for dim in EVAL_DIMENSIONS:
        response = llm_client.evaluate(
            image=image_path, prompt=original_prompt,
            question=f"Rate {dim}: L1 (fully correct), L2 (partially), L3 (present but wrong)"
        )
        scores[dim] = TIER_SCORES.get(response.tier, 0.33)

    non_semantic = [v for k, v in scores.items() if k != "semantic_alignment"]
    composite = (sum(non_semantic) / len(non_semantic) + scores["semantic_alignment"]) / 2
    return {"dimensions": scores, "composite": composite}

# updater.py
def update_preferences(tool_name: str, predicted_rank: int, actual_rank: int,
                       registry: dict, alpha: float = 0.2) -> dict:
    """APU: adjust scores when predicted vs actual rankings diverge."""
    if predicted_rank != actual_rank:
        adjustment = alpha * (predicted_rank - actual_rank) / max(predicted_rank, actual_rank)
        for dim in registry[tool_name]:
            registry[tool_name][dim] = max(0, min(1,
                registry[tool_name][dim] + adjustment))
    return registry
```

**Example 3: Capability-aligned task decomposition**

User: "I want to generate a complex scene: 'A medieval castle on a cliff at sunset with three knights on horseback in the foreground.' Break this into subtasks matched to the best available tools."

Approach:
1. Analyze the prompt for required capabilities: 3d_spatial (cliff perspective), numeracy (three knights), texture (medieval materials), color (sunset lighting)
2. Check tool registry for weak dimensions -- if no tool scores > 0.5 on numeracy, restructure to avoid relying on numeracy
3. Decompose into capability-aligned subtasks

Output:
```
Plan (CAPO-aligned):
  Subtask 1: Generate base scene "medieval castle on cliff at sunset"
    -> Best tool: SDXL (high color: 0.81, texture: 0.73)
    -> Avoids numeracy by not including knights yet

  Subtask 2: Generate single knight on horseback as reference
    -> Best tool: FLUX (high non_spatial: 0.92)
    -> Generates one high-quality knight to use as reference

  Subtask 3: Composite three knights into foreground using layout-guided generation
    -> Best tool: Layout_to_Image (uses bounding boxes for placement)
    -> Handles numeracy via explicit spatial layout rather than relying on model counting

  Subtask 4: Harmonize lighting and style across composited image
    -> Best tool: UltraEdit (style_transfer: 0.78)
    -> Ensures consistent sunset lighting across all elements

Quality gate: Evaluate composite score after each subtask. Threshold: 0.8.
Retry budget: up to 5 rounds with fallback to next-ranked tools.
```

## Best Practices

- **Do:** Benchmark tools empirically before populating the score registry. Run each tool on 50-100 diverse prompts spanning all capability dimensions. PerfGuard's value comes from accurate scores, not guesses.
- **Do:** Use parallel execution of top-N tools when latency budget allows. Comparing actual outputs is far more reliable than relying on scores alone, especially early in deployment.
- **Do:** Keep the APU learning rate (alpha) conservative (0.1-0.2). Aggressive updates cause score oscillation when evaluation is noisy.
- **Do:** Store experience records with CLIP embeddings of the prompt for fast semantic retrieval. Past task outcomes are the highest-signal input for planning.
- **Avoid:** Using the same weight vector for all tasks. The entire point of PASM is that different tasks stress different dimensions. A uniform weighting collapses back to a generic "best overall model" selection.
- **Avoid:** Skipping the evaluation step. Without measured output quality, APU cannot update preferences and the system cannot self-improve. Even a rough VLM-based evaluation (GPT-4V, Gemini) is better than none.

## Error Handling

- **Tool execution failure (timeout, OOM, API error):** Assign a composite score of 0.0, fall through to the next-ranked tool immediately. Log the failure mode in the experience record.
- **All top-N tools score below threshold:** After exhausting the retry budget (5 rounds), return the best result achieved with a quality warning. Do not loop indefinitely.
- **Evaluation model disagrees with human judgment:** Periodically calibrate VLM evaluation scores against human ratings. If the evaluator is consistently wrong on a dimension, add dimension-specific calibration offsets.
- **Score drift from APU:** Implement score bounds (never below 0.05, never above 0.99) and add periodic score decay toward benchmark baselines to prevent runaway drift.
- **New tool added to registry:** Initialize scores at the population mean across all dimensions, then run a quick benchmark pass (20-30 prompts) to establish a real fingerprint before using in production routing.

## Limitations

- **Requires a vision-language model for evaluation.** The multi-dimensional scoring system depends on a capable VLM (GPT-4V-class or better) to assess generated images. Without it, APU cannot function.
- **Initial scores need manual effort.** The system is only as good as its initial benchmarks. Cold-start with uniform scores will produce poor tool selection until enough APU iterations accumulate.
- **Dimension taxonomy is domain-specific.** The text-to-image and image-editing dimensions from the paper may not transfer directly to video generation, 3D, or audio. You must define appropriate axes for your domain.
- **Does not model tool interactions.** PASM scores tools independently. If two tools combined produce better results than either alone (e.g., generate + refine), this must be captured in the experience/planning layer, not in individual tool scores.
- **Evaluation latency adds overhead.** Running VLM evaluation after every generation step increases end-to-end latency. For real-time applications, consider batch evaluation or sampling strategies.

## Reference

- **Paper:** [PerfGuard: A Performance-Aware Agent for Visual Content Generation](https://arxiv.org/abs/2601.22571v1) (ICLR 2026)
- **Code:** [github.com/FelixChan9527/PerfGuard](https://github.com/FelixChan9527/PerfGuard)
- **What to look for:** Section 3 details PASM scoring dimensions and the preference-weighted selection formula; Section 4 covers the APU update mechanism with predicted-vs-actual rank comparison; Section 5 describes CAPO's planner integration. The experience replay system using CLIP embeddings is in Section 3.3.