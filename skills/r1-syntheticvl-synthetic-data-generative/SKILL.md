---
name: "r1-syntheticvl-synthetic-data-generative"
description: "Synthesize high-quality multimodal training data using Collective Adversarial Data Synthesis (CADS). Implements a cyclic generate-judge-optimize pipeline where multiple models collectively produce diverse data, consensus-score it for quality, and adversarially optimize generation context toward harder samples. Use when: 'generate synthetic training data for my vision-language model', 'create challenging multimodal QA pairs', 'build a synthetic dataset with adversarial filtering', 'synthesize diverse math/science visual reasoning data', 'improve my MLLM with synthetic data', 'set up a data generation pipeline with quality control'."
---

# Collective Adversarial Data Synthesis (CADS) for Multimodal Training Data

This skill enables Claude to design and implement **Collective Adversarial Data Synthesis (CADS)** pipelines that autonomously generate high-quality, diverse, and challenging multimodal training data for MLLMs. The core innovation is a cyclic two-phase system: multiple models collectively generate candidate data, then collectively judge it via consensus scoring, and an adversarial optimization loop steers generation toward samples that expose model disagreements -- the most valuable training signal. This replaces naive single-model data generation with a self-improving pipeline that progressively produces harder, more informative samples.

## When to Use

- When the user wants to generate synthetic training data for a vision-language or multimodal model
- When the user needs to augment a small real dataset with diverse synthetic samples for fine-tuning
- When building a data generation pipeline that filters for quality and difficulty automatically
- When the user asks to create challenging visual QA pairs (math diagrams, charts, science figures)
- When implementing adversarial data selection to focus training on hard examples
- When the user wants to scale a seed dataset from hundreds to thousands of high-quality samples
- When designing a multi-model consensus system for data quality assessment

## Key Technique

**CADS** operates in two cyclic phases. In **CAD-Generate**, a collective of diverse models (e.g., GPT-4o, Gemini, DeepSeek-R1, Claude) each analyze a seed sample to extract its knowledge domain and underlying rationale, then independently apply one of four meta-strategies to synthesize new data: (1) Numerical & Parameter Variation -- same structure, different quantities; (2) Logic Reversion -- swap conditions and objectives; (3) Auxiliary Extension -- add reinforcing elements; (4) Isomorphic Scenario Transfer -- map the reasoning to a different visual domain. Each model also generates an explicit visual prompt specifying spatial layout, object attributes, and data values, which feeds an image generation model.

In **CAD-Judge**, the synthesized samples are evaluated by multiple MLLM judges who independently attempt to solve each question. A **consensus score** C counts how many judges arrive at the correct answer. Samples where C=0 are discarded (likely flawed), samples where C=K (unanimous) are kept as clean training data, and samples where 1 <= C < K are flagged as **adversarial candidates** -- solvable but challenging enough to cause disagreement among strong models. These adversarial candidates are the most valuable for driving model improvement.

The **Adversarial Context Optimization** mechanism closes the loop. It has two sub-steps: **Reflect** -- analyze patterns in where judges disagreed and why -- and **Optimize** -- inject those insights back into the generation context so the next round of CAD-Generate produces harder, more targeted samples. This cycle repeats (up to ~10 iterations per seed), progressively shifting the distribution toward high-value data.

## Step-by-Step Workflow

1. **Curate seed data.** Collect 50-500 seed samples, each consisting of either a multimodal sample (image + question + answer) or a textual description of a target task domain (e.g., "geometry problems involving triangle area calculation"). Seeds define the knowledge domains to explore.

2. **Configure the model collective.** Select 3-5 diverse generative models to serve as the collective (e.g., GPT-4o, Gemini-2.5-Flash, Claude Sonnet, DeepSeek-R1). Diversity in model architecture and training data is critical -- it ensures broad coverage of reasoning strategies and reduces blind spots.

3. **Implement the CAD-Generate phase.** For each seed, have every model in the collective execute three steps sequentially:
   - **Rationale Analysis**: Extract the knowledge domain (e.g., "Plane Geometry") and core reasoning skill (e.g., "applying the Pythagorean theorem to compute distance").
   - **Strategy Selection**: Randomly assign one of the four meta-strategies (Numerical Variation, Logic Reversion, Auxiliary Extension, Isomorphic Transfer) to encourage diversity.
   - **Visual Prompt + QA Generation**: Produce a detailed visual prompt (specifying spatial layout, colors, labels, data values) alongside the new question and ground-truth answer.

4. **Generate visual content.** Feed each visual prompt into an image generation model (e.g., a text-to-image model capable of diagrams, charts, and scientific figures). Verify the generated image actually contains the elements specified in the prompt before proceeding.

5. **Implement the CAD-Judge phase.** For each synthesized (image, question, answer) triplet, have K judge models (can overlap with generators) independently attempt to answer the question given only the image and question. Compute the consensus score C = number of judges whose answer matches the ground truth.

6. **Filter and categorize results.** Apply the three-tier classification:
   - **Discard** (C=0): Likely has synthesis flaws, multimodal misalignment, or an incorrect ground-truth answer.
   - **Clean** (C=K): High-quality, unanimous agreement. Include in the training set.
   - **Adversarial** (1 <= C < K): Solvable but challenging. Flag for adversarial context optimization and include in training set.

7. **Run Adversarial Context Optimization.** Analyze the adversarial set to identify patterns: which reasoning steps tripped up judges? What visual elements caused confusion? Distill these into explicit insights (the "Reflect" step). Then inject these insights into the generation prompt context for the next iteration (the "Optimize" step), biasing generation toward similarly challenging territory.

8. **Iterate the cycle.** Repeat steps 3-7 for up to 10 iterations per seed. Each round's adversarial insights refine the next round's generation, progressively improving data difficulty and quality without manual intervention.

9. **Assemble the final dataset.** Aggregate all samples that passed filtering (C >= 1) across all seeds and iterations. Deduplicate by comparing question text similarity (e.g., cosine similarity on embeddings > 0.95 threshold). Target a balanced distribution across domains.

10. **Train with GRPO.** Fine-tune the target MLLM on the assembled dataset using Group Relative Policy Optimization with 8 rollout samples per question, a global batch size of 128, temperature 1.0, and learning rate 1e-6. This reinforcement-learning-based training specifically benefits from the adversarial difficulty gradient in the data.

## Concrete Examples

**Example 1: Building a Math Visual Reasoning Dataset from Seeds**

```
User: I have 200 geometry seed problems with diagrams. I want to expand
this to 4,000 synthetic training samples for fine-tuning Qwen2.5-VL-7B.

Approach:
1. Parse each seed into (image, question, answer, domain="Geometry").
2. Configure collective: GPT-4o, Gemini-2.5-Flash, Claude Sonnet, DeepSeek-R1.
3. For each seed, run 5 iterations of CAD-Generate:
   - Iteration 1: Each model applies a randomly assigned strategy.
     Seed: "Find the area of triangle ABC with vertices at (0,0), (4,0), (0,3)."
     -> Numerical Variation: "Find the area of triangle DEF with vertices at (1,2), (5,2), (1,7)."
     -> Logic Reversion: "Given a triangle with area 6, find the height if the base is 4."
     -> Isomorphic Transfer: "A park is shaped as a triangle on a city map [image]. Calculate its area."
   - Each output includes a visual prompt for image generation.
4. Generate images from visual prompts using the image model.
5. CAD-Judge: 4 judges solve each problem. Filter:
   - C=0 -> discard (~15% typical)
   - C=4 -> clean set (~50%)
   - C=1,2,3 -> adversarial set (~35%)
6. Adversarial Optimization: "Judges frequently disagreed on problems
   requiring coordinate transformation before area calculation.
   Next round: emphasize rotated/translated coordinate systems."
7. Repeat. After 5 iterations across 200 seeds x 4 models = ~4,000 candidates.
   After filtering: ~3,400 high-quality samples.
   Scale up iterations to reach 4,000 target.

Output: A JSON-lines dataset where each line contains:
{
  "image_path": "synth_00042.png",
  "question": "In the figure, triangle PQR has vertices P(2,1), Q(6,1), R(4,5). Find the area.",
  "answer": "8 square units",
  "domain": "Geometry",
  "consensus_score": 3,
  "iteration": 3,
  "strategy": "numerical_variation"
}
```

**Example 2: Implementing the Consensus Scoring Pipeline**

```
User: How do I implement the CAD-Judge consensus scoring in Python?

Approach:
1. Define the judge evaluation function.
2. Aggregate scores across judges.
3. Categorize each sample.

Output (Python implementation):

import asyncio
from dataclasses import dataclass

@dataclass
class SynthSample:
    image_path: str
    question: str
    ground_truth: str
    consensus_score: int = 0
    category: str = "pending"  # "clean", "adversarial", "discard"

async def judge_sample(sample: SynthSample, judge_models: list) -> SynthSample:
    """Have K judges independently answer, then compute consensus."""
    predictions = await asyncio.gather(*[
        model.answer(image=sample.image_path, question=sample.question)
        for model in judge_models
    ])
    # Consensus score: count matches with ground truth
    sample.consensus_score = sum(
        1 for pred in predictions
        if normalize_answer(pred) == normalize_answer(sample.ground_truth)
    )
    K = len(judge_models)
    if sample.consensus_score == 0:
        sample.category = "discard"
    elif sample.consensus_score == K:
        sample.category = "clean"
    else:
        sample.category = "adversarial"
    return sample

async def run_cad_judge(samples: list, judge_models: list) -> dict:
    """Run CAD-Judge on all samples, return categorized results."""
    judged = await asyncio.gather(*[
        judge_sample(s, judge_models) for s in samples
    ])
    return {
        "clean": [s for s in judged if s.category == "clean"],
        "adversarial": [s for s in judged if s.category == "adversarial"],
        "discard": [s for s in judged if s.category == "discard"],
    }
```

**Example 3: Adversarial Context Optimization Prompt Design**

```
User: Show me how to structure the adversarial optimization prompt
that feeds insights back into the next generation round.

Approach:
1. Collect error patterns from adversarial samples.
2. Structure a reflection prompt.
3. Inject into generation context.

Output (prompt template):

=== REFLECT PHASE PROMPT ===
You are analyzing adversarial samples where judge models disagreed.

Adversarial samples from this round:
{adversarial_samples_summary}

Judge agreement patterns:
- Judges agreed on: {agreement_patterns}
- Judges disagreed on: {disagreement_patterns}
- Common error types: {error_analysis}

Task: Identify 3-5 specific reasoning challenges that caused
disagreement. For each, describe:
1. The reasoning skill being tested
2. Why it caused disagreement
3. How to generate more samples targeting this skill

=== OPTIMIZE PHASE: INJECT INTO GENERATION CONTEXT ===
Previous generation insights (use these to create harder problems):
{reflected_insights}

When generating new samples, prioritize:
- Problems requiring {challenging_skill_1}
- Visual scenarios where {confusing_element}
- Multi-step reasoning chains involving {hard_transition}

Apply strategy: {selected_meta_strategy}
Seed context: {seed_rationale}
Generate a new (visual_prompt, question, answer) triplet.
```

## Best Practices

- **Do:** Use at least 3-4 diverse models in the collective. The paper demonstrates that model diversity is the primary driver of data diversity -- a single model generating 4x the data produces worse results than 4 models generating 1x each.
- **Do:** Keep all four meta-strategies (Numerical Variation, Logic Reversion, Auxiliary Extension, Isomorphic Transfer) active and randomly assigned. Dropping any strategy measurably reduces downstream performance.
- **Do:** Treat adversarial samples (1 <= C < K) as the highest-value training data. These are the samples where models are "almost right" -- exactly where learning signal is richest.
- **Do:** Validate visual-semantic alignment manually on a random 5% sample each round. Image generation models can silently drop elements from the visual prompt.
- **Avoid:** Discarding adversarial samples or only keeping unanimous (C=K) data. The ablation study shows adversarial optimization contributes +2.6% accuracy on MathVista -- nearly as much as the entire CAD-Generate+CAD-Judge pipeline.
- **Avoid:** Running more than ~10 iterations per seed. Returns diminish as the generation context becomes overloaded with optimization constraints, and prompts grow unwieldy.

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| High discard rate (C=0 > 30%) | Image generation model ignoring visual prompt details | Add explicit verification step: re-read generated image with an MLLM and check if specified elements are present before judging |
| All samples unanimous (C=K) | Data is too easy; no adversarial signal | Increase strategy complexity; bias toward Logic Reversion and Isomorphic Transfer which produce harder samples |
| Low diversity across iterations | Generated questions converge to similar patterns | Rotate which model leads generation each iteration; increase temperature; inject "must differ from previous" constraints |
| Adversarial optimization diverges | Later iterations produce unsolvable problems | Cap difficulty: if C=0 rate exceeds 40% in any round, roll back to previous round's context and reduce optimization aggressiveness |
| Ground truth errors in generated data | Judge consensus is high but answer is wrong | Cross-validate ground truth with a separate symbolic solver (e.g., SymPy for math) before entering the judge phase |

## Limitations

- **Image generation quality bottleneck.** CADS is only as good as the image generation model. Complex diagrams with precise spatial relationships (e.g., overlapping geometric figures, multi-axis charts) frequently have visual errors that corrupt the training signal. Always verify a random subset.
- **API cost.** Running 4 generative models + 4 judge models for 10 iterations per seed is expensive. Budget approximately 8-10 API calls per final training sample after filtering losses.
- **Domain scope.** The paper validates on math, science, and chart understanding. The meta-strategies (especially Logic Reversion and Isomorphic Transfer) may not transfer cleanly to domains like medical imaging or satellite imagery where "isomorphic scenarios" are harder to define.
- **No guarantee of correctness.** Consensus scoring filters for consistency, not correctness. If all judges share the same blind spot, a wrong answer passes as "clean." Symbolic verification or human spot-checks remain necessary for safety-critical applications.
- **Requires strong seed data.** The pipeline amplifies and diversifies existing seeds. If seeds are narrow or low-quality, the generated data inherits those limitations despite adversarial optimization.

## Reference

**Paper:** [R1-SyntheticVL: Is Synthetic Data from Generative Models Ready for Multimodal Large Language Model?](https://arxiv.org/abs/2602.03300v1) -- Zhang et al., 2026. Focus on Section 3 (CADS method), Table 3 (ablation showing each component's contribution), and Table 4 (synthetic vs. real data comparison showing synthetic data can match or exceed real data performance).