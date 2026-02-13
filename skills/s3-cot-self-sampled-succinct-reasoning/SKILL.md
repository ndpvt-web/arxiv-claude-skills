---
name: "s3-cot-self-sampled-succinct-reasoning"
description: "Apply dual-cognitive reasoning (System 1 fast / System 2 slow) to compress verbose chain-of-thought into succinct, efficient reasoning traces while preserving accuracy. Use when: 'make this reasoning shorter', 'compress chain of thought', 'fast thinking mode', 'succinct reasoning', 'reduce reasoning verbosity', 'System 1 thinking for this problem'."
---

# S3-CoT: Self-Sampled Succinct Reasoning

This skill enables Claude to apply a dual-cognitive reasoning framework inspired by the S3-CoT paper. Instead of always producing long, verbose chain-of-thought reasoning, Claude can dynamically switch between **System 2** (slow, deliberate, step-by-step reasoning) and **System 1** (fast, compressed, intuition-like reasoning) depending on problem difficulty. The core insight is that most reasoning chains contain substantial redundancy -- by progressively compressing reasoning while verifying answer consistency, you can produce traces that are 20-40% shorter without accuracy loss.

## When to Use

- When the user asks to "think faster" or "be more concise in reasoning" about a problem
- When building LLM pipelines that need efficient CoT and the user wants to reduce token costs
- When implementing self-consistency filtering to select high-quality reasoning samples from LLM outputs
- When designing training data generation pipelines that produce variable-length reasoning traces
- When the user wants to implement progressive curriculum training for reasoning compression
- When building dual-mode reasoning systems that adaptively choose verbose or succinct thinking
- When optimizing inference cost by compressing reasoning traces in production LLM applications
- When the user asks Claude to solve math, logic, or medical reasoning problems efficiently

## Key Technique

**Activation Steering for Variable-Length Reasoning.** The S3-CoT method discovers that LLMs encode a "variable-length direction" (VL-D) in their hidden states that controls reasoning verbosity. By computing the difference-in-means between activations produced by "think briefly" vs. "think step by step" prompts, you get a steering vector. Adding this vector (scaled by strength alpha) to hidden states at specific layers produces shorter reasoning without retraining. The key insight for practical use: you don't need activation-level access to apply the principle. Prompt-level steering with explicit compression instructions, combined with self-consistency verification, achieves analogous results.

**Progressive Compression Curriculum.** Rather than jumping straight to maximally compressed reasoning, the method uses a curriculum that starts with near-original-length traces (Length-Ratio 0.9-1.0) and progressively includes shorter variants down to 0.0-1.0. This prevents the "collapse" that happens when you force extreme compression too early. Each stage maintains a roughly uniform distribution across compression levels. For practical application, this means you should compress reasoning incrementally -- first trim obvious redundancy, then eliminate repetitive verification steps, then compress multi-step arithmetic into single leaps.

**Self-Evolution Without Gold Answers.** The most practically useful finding: you can filter reasoning traces purely by self-consistency. Generate multiple reasoning variants at different compression levels for the same problem. If the answer stays consistent across short and long variants, the compressed version is reliable. This eliminates the need for ground-truth labels and works across domains (math, medicine, general reasoning).

## Step-by-Step Workflow

1. **Classify problem difficulty.** Assess whether the problem requires System 2 (multi-step derivation, novel problem structure, high stakes) or can use System 1 (familiar pattern, single-hop reasoning, routine calculation). Default to System 2 for unfamiliar domains.

2. **Generate a full System 2 reasoning trace first.** Solve the problem with complete step-by-step reasoning, showing all intermediate steps. This is your "anchor" solution that establishes correctness.

3. **Identify redundant reasoning segments.** Scan the full trace for: (a) restating the problem, (b) trivial arithmetic shown in full, (c) self-verification loops that repeat conclusions, (d) hedge phrases and meta-commentary ("Let me think about this..."), (e) steps that state obvious implications.

4. **Apply progressive compression at Level 1 (Length-Ratio ~0.7).** Remove meta-commentary and problem restatement. Combine trivial sequential steps. Keep all substantive logical transitions. Verify the compressed trace still reaches the same answer.

5. **Apply progressive compression at Level 2 (Length-Ratio ~0.4).** Collapse multi-step arithmetic into results. Remove intermediate verification. Retain only the critical reasoning pivots -- the steps where the solution direction changes or a key insight is applied. Verify answer consistency.

6. **Apply progressive compression at Level 3 (Length-Ratio ~0.2).** Produce a System 1 trace: state the key insight, apply it directly, give the answer. This should read like an expert's intuitive solution. Verify answer consistency.

7. **Self-consistency check across compression levels.** Compare answers from all compression levels. If they agree, the most compressed version that the context requires is safe to use. If they disagree, fall back to the longest consistent trace.

8. **Select the appropriate output.** For routine problems with confirmed self-consistency, output the Level 2 or Level 3 trace. For complex or high-stakes problems, output Level 1. Always note the compression level used so the user can request more detail if needed.

9. **When building training pipelines, generate the full spectrum.** For each problem, produce traces at 4-5 compression levels. Filter by answer correctness (if gold labels exist) or self-consistency (if not). Package as dual-cognitive training data with System 1 and System 2 prompt prefixes.

10. **Implement curriculum ordering for fine-tuning.** If training a model on compressed reasoning data, start SFT with the least-compressed samples (Length-Ratio 0.9-1.0), then progressively include shorter samples in 0.1 decrements per training phase. Use LoRA (r=8, alpha=16) for parameter-efficient training.

## Concrete Examples

**Example 1: Math Problem -- Progressive Compression**

User: Solve this efficiently: "A store sells apples at $2 each and oranges at $3 each. If Maria bought 15 fruits for $36, how many apples did she buy?"

System 2 (Full trace):
```
Let me define variables. Let a = number of apples, o = number of oranges.
We know: a + o = 15 (total fruits)
And: 2a + 3o = 36 (total cost)
From the first equation: o = 15 - a
Substituting into the second: 2a + 3(15 - a) = 36
Expanding: 2a + 45 - 3a = 36
Simplifying: -a + 45 = 36
Therefore: -a = -9
So: a = 9
Verification: 9 apples ($18) + 6 oranges ($18) = $36. Total fruits: 15. Correct.
Maria bought 9 apples.
```

System 1 (Compressed trace):
```
Let a = apples. System: a + o = 15, 2a + 3o = 36.
Substituting: 2a + 3(15-a) = 36 -> -a = -9 -> a = 9.
Maria bought 9 apples.
```

Self-consistency: Both traces yield 9. Compressed version is reliable.
Token reduction: ~60% fewer tokens.

**Example 2: Building a Self-Consistency Filter for LLM Outputs**

User: I'm generating CoT training data from my model. How do I filter for quality without gold labels?

Approach:
1. For each problem, generate N reasoning traces at different verbosity levels using prompt variation:
   - Prompt A (verbose): "Think step by step carefully and show all work."
   - Prompt B (medium): "Solve this, showing key steps only."
   - Prompt C (brief): "Solve this as briefly as possible."
2. Extract the final answer from each trace.
3. Apply majority voting: if >= 2 of 3 traces agree on the answer, retain all consistent traces.
4. Discard problems where traces disagree -- these indicate difficulty beyond the model's reliable capability at compressed lengths.

```python
def self_consistency_filter(problems, model, prompts):
    """Filter reasoning traces by cross-compression consistency."""
    filtered_data = []
    for problem in problems:
        traces = {}
        answers = {}
        for level, prompt_template in prompts.items():
            prompt = prompt_template.format(problem=problem)
            trace = model.generate(prompt)
            traces[level] = trace
            answers[level] = extract_answer(trace)

        # Check consistency across compression levels
        answer_values = list(answers.values())
        majority_answer = max(set(answer_values), key=answer_values.count)
        agreement = answer_values.count(majority_answer) / len(answer_values)

        if agreement >= 0.66:  # At least 2/3 agree
            for level, trace in traces.items():
                if answers[level] == majority_answer:
                    filtered_data.append({
                        "problem": problem,
                        "trace": trace,
                        "compression_level": level,
                        "system_prompt": "brief" if level == "brief" else "detailed"
                    })
    return filtered_data
```

Retention rates from the paper: 83-99% of samples pass this filter with near-perfect accuracy on retained data.

**Example 3: Dual-Cognitive System Prompts for Production**

User: How do I set up a dual-mode reasoning API that lets callers choose fast vs. thorough?

Approach:
1. Define two system prompts matching the paper's dual-cognitive design.
2. Route based on caller preference or automatic difficulty assessment.

```python
SYSTEM_PROMPTS = {
    "system1": (
        "Provide the briefest reasoning process possible. "
        "State only the critical insight and final computation. "
        "Put final answer within \\boxed{}"
    ),
    "system2": (
        "Reason step by step, showing all intermediate work. "
        "Verify your answer before concluding. "
        "Put final answer within \\boxed{}"
    ),
}

def classify_difficulty(problem: str) -> str:
    """Heuristic difficulty classifier for routing."""
    indicators_hard = ["prove", "optimize", "minimum number", "at most", "if and only if"]
    indicators_easy = ["calculate", "compute", "how many", "what is", "solve"]
    hard_score = sum(1 for i in indicators_hard if i in problem.lower())
    easy_score = sum(1 for i in indicators_easy if i in problem.lower())
    return "system2" if hard_score > easy_score else "system1"

def dual_cognitive_inference(problem: str, mode: str = "auto"):
    if mode == "auto":
        mode = classify_difficulty(problem)
    system_prompt = SYSTEM_PROMPTS[mode]
    response = llm.generate(system_prompt=system_prompt, user_prompt=problem)
    # Fallback: if System 1 answer seems uncertain, retry with System 2
    if mode == "system1" and response.confidence < 0.8:
        response = llm.generate(
            system_prompt=SYSTEM_PROMPTS["system2"], user_prompt=problem
        )
    return response
```

Token savings: System 1 uses 20-40% fewer tokens on routine problems while maintaining accuracy.

## Best Practices

- **Do:** Always generate the full reasoning first, then compress. Compression without a correctness anchor leads to plausible-sounding wrong answers.
- **Do:** Use self-consistency across compression levels as your primary quality signal. If the short and long answers agree, the short answer is almost certainly correct (99.8%+ accuracy per the paper).
- **Do:** Apply progressive compression in stages rather than jumping to maximum compression. Each stage should remove one category of redundancy.
- **Do:** Maintain both System 1 and System 2 capabilities. Train or prompt for both modes so the model can fall back to verbose reasoning on hard problems.
- **Avoid:** Compressing reasoning for problems the model hasn't seen before or domains where it lacks confidence. Compression amplifies existing capability -- it cannot create understanding that isn't there.
- **Avoid:** Using only the shortest traces for training. The paper shows this causes performance collapse. You need the full spectrum of compression levels in your training data.
- **Avoid:** Skipping the self-consistency verification step in production. A 2-second consistency check prevents confidently wrong compressed answers.

## Error Handling

- **Inconsistent answers across compression levels:** Fall back to the full System 2 trace. Log the problem as "compression-sensitive" for later analysis. These problems often involve subtle multi-step dependencies where intermediate steps carry essential information.
- **Generation collapse (output becomes incoherent at high compression):** Reduce compression strength. The paper found that alpha values beyond -0.5 for general LLMs cause collapse. In prompt-based compression, this means your "be brief" instruction is too aggressive -- ask for "key steps only" instead of "one sentence."
- **Domain transfer failure:** The method transfers well from math to medicine (per paper results), but verify self-consistency when applying to a new domain. If consistency rates drop below 80%, the compression curriculum needs domain-specific calibration.
- **Curriculum training divergence:** If loss spikes during progressive compression training, slow down the curriculum. Extend the number of steps at each Length-Ratio bracket before introducing shorter samples.

## Limitations

- Compression quality depends on the base model's underlying capability. If the model doesn't truly understand the problem, compressed reasoning will be confidently wrong rather than helpfully brief.
- The technique works best for well-structured problems (math, logic, medical QA) where "correct answer" is verifiable. Open-ended generation tasks (creative writing, design) don't have a clean self-consistency signal.
- Self-consistency filtering discards 1-17% of problems. For critical applications requiring 100% coverage, you need a fallback to full-length reasoning for filtered-out problems.
- The progressive curriculum requires multiple training passes, increasing total training time even though per-pass compute is modest (2x A100 vs 8x H100 for RL baselines).
- Prompt-based compression (without activation steering) is an approximation. It captures the qualitative behavior but may not achieve the same fine-grained length control as hidden-state intervention.

## Reference

**Paper:** [S3-CoT: Self-Sampled Succinct Reasoning Enables Efficient Chain-of-Thought LLMs](https://arxiv.org/abs/2602.01982v1) (Du et al., 2026). Look for: Section 3 (VL-D identification and activation steering), Section 4 (progressive compression curriculum), and Table 1 (benchmark results showing 20-40% token reduction with stable accuracy).

**Code & Models:** [github.com/DYR1/S3-CoT](https://github.com/DYR1/S3-CoT) -- includes sampling code, evaluation scripts, and HuggingFace model checkpoints under `yrdu/`.