---
name: "socratic-geo-synthetic-data-generation"
description: "Generate high-quality synthetic training data through multi-agent feedback loops where a Teacher agent creates parameterized problems, a Solver agent exposes weaknesses through failure, and failures drive targeted augmentation. Use when: 'generate synthetic training data', 'create geometric reasoning dataset', 'build a multi-agent data pipeline', 'expand seed examples into a full dataset', 'couple data generation with model learning', 'failure-driven data augmentation'."
---

# Socratic-Geo: Synthetic Data Generation via Multi-Agent Interaction

This skill enables Claude to build **failure-driven synthetic data generation pipelines** based on the Socratic-Geo framework. Instead of randomly generating training examples and filtering for quality, you design a closed-loop system where a Teacher agent creates parameterized problems as executable code, a Solver agent attempts them, and Solver failures directly trigger targeted augmentation -- ensuring every new example addresses a real weakness. This approach turns 108 seed problems into thousands of high-quality training pairs using a fraction of the data that passive methods require.

## When to Use

- When the user needs to **expand a small seed dataset** into a large, high-quality training set without manual annotation
- When building a **synthetic data pipeline for geometric reasoning**, math problem generation, or any domain where problems can be expressed as parameterized code
- When the user wants to implement **multi-agent data generation** where agents collaborate in a Teacher-Solver feedback loop
- When creating training data that **targets specific model weaknesses** rather than randomly sampling the problem space
- When the user asks to generate **image-text pairs, code-instruction pairs, or structured problem-solution pairs** with automated quality control
- When building a **curriculum learning pipeline** that progressively increases difficulty based on learner performance

## Key Technique

The Socratic-Geo framework replaces the standard "generate-then-filter" paradigm with a **dynamically coupled synthesis loop**. Three agents collaborate: a **Teacher** that creates problems as parameterized Python scripts (e.g., matplotlib/tikz code that draws geometric figures with varying parameters), a **Solver** that attempts to solve them using reinforcement learning (GRPO -- Group Relative Policy Optimization), and a **Generator** that learns to produce visual outputs from natural-language instructions derived from the code. The critical insight is that the Solver's failures are not discarded -- they are the primary signal driving the Teacher's next round of problem creation.

The Teacher enforces data quality through two reflection mechanisms. **Reflect** requires the Teacher to solve its own invented problem; only if the Teacher's answer matches the reference solution is the problem admitted to the training curriculum. **RePI (Reflective Problem Inspection)** validates that the generated code executes successfully and that the visual output aligns with the textual description -- catching rendering errors, degenerate geometries, and label mismatches before they pollute the dataset. Together, these filters achieve what the paper calls "image-text pair purity."

The expansion follows a staged curriculum: Stage 1 starts with seed problems plus first-round augmentations (~0.4k), Stage 2 adds augmentations from Stage 1 failures (~1.0k), and Stage 3 reaches ~2.5k problems. At each stage, the Solver trains on the current curriculum, its failures trigger the Teacher, and the cycle repeats. This means every problem in the final dataset exists because a model actually struggled with something -- there is no wasted data.

## Step-by-Step Workflow

1. **Define the seed problem set.** Collect a small number of hand-crafted examples (even as few as ~100) that cover the core categories of your domain. Each seed should be a parameterized script that produces a problem instance (image + question + answer) when executed with specific parameter values.

2. **Implement the Teacher agent.** Build a code-generation agent that takes a seed problem (or a failed problem) and produces a modified Python script. The Teacher varies geometric parameters (angles, lengths, positions), adds or removes constraints, and changes the question type. The output is always executable code, not raw text.

3. **Implement the Reflect quality gate.** After the Teacher generates a new problem, have it solve the problem independently. Compare the Teacher's solution against the reference answer embedded in the code. Reject any problem where the Teacher cannot reproduce the correct answer -- this catches ambiguous, unsolvable, or incorrectly parameterized problems.

4. **Implement the RePI visual validation gate.** Execute the generated Python script and verify: (a) the code runs without errors, (b) the rendered output matches the textual description (labels are placed correctly, geometric elements are visible and non-degenerate), and (c) no visual artifacts corrupt the training signal. Use programmatic checks (bounding box validation, label overlap detection) rather than manual inspection.

5. **Implement the Solver agent.** Build a model (or use an existing LLM) that attempts to solve each problem in the curriculum. For each problem, generate k solution attempts (k=8 is a good default). Score each attempt with a binary reward: correct or incorrect. Compute group-relative advantages: `A(i) = (R_i - mean(R)) / std(R)`.

6. **Identify failure cases.** After the Solver processes the current curriculum, collect all problems where the Solver failed across all k attempts. These failures represent genuine reasoning gaps -- the exact problems the model cannot yet handle.

7. **Trigger targeted augmentation.** For each failure, pass the failed problem back to the Teacher with a diagnostic prompt: "The Solver failed on this problem. Analyze the likely reasoning gap (missed geometric property, violated constraint, unfamiliar configuration) and generate a new problem that explicitly exercises this gap." The Teacher modifies the underlying code to construct a targeted variant.

8. **Qualify new problems through Reflect + RePI.** Run both quality gates on every Teacher-generated augmentation. Only problems that pass both checks enter the next stage of the curriculum. This dual filter is critical -- removing it degrades final performance significantly.

9. **Train the Solver on the expanded curriculum.** Use GRPO (or DPO/PPO as appropriate) to train the Solver on the new curriculum. The Solver never sees Teacher solutions -- it learns only from binary reward signals indicating correctness.

10. **Iterate stages.** Repeat steps 6-9 for 2-3 stages, progressively expanding the curriculum. Optionally, train a Generator model on accumulated (image, code, instruction) triplets using supervised fine-tuning to distill the programmatic drawing logic into a neural model.

## Concrete Examples

**Example 1: Expanding a Geometry Problem Set**

```
User: I have 50 hand-written geometry problems as Python scripts that draw
figures with matplotlib and include questions + answers. I need to expand this
to 500+ high-quality training pairs for fine-tuning a vision-language model.

Approach:
1. Organize the 50 seeds by category (triangle properties, circle theorems,
   angle relationships, coordinate geometry).
2. For each seed, build a Teacher prompt:
   "Given this geometry script, create a variant by:
    - Changing the triangle from acute to obtuse
    - Adding an altitude and asking about its length
    - Varying numeric parameters within valid ranges
    Output a complete, executable Python script."
3. Run Reflect: have the Teacher solve each new problem. Discard any where
   the Teacher's answer differs from the embedded reference.
4. Run RePI: execute each script, check that the figure renders without
   overlapping labels or degenerate shapes (e.g., collinear triangle vertices).
5. Deploy a Solver (e.g., GPT-4o or a fine-tuned model) on all qualifying
   problems. Collect failures.
6. Feed failures back to Teacher: "The Solver could not determine the length
   of the altitude in an obtuse triangle. Generate 3 variants that practice
   altitude computation in obtuse, right, and acute triangles."
7. Repeat for 2 stages until reaching 500+ validated pairs.

Output structure per example:
{
  "image": "problem_127.png",
  "code": "# Python script that generates problem_127.png\nimport matplotlib...",
  "question": "In triangle ABC, if angle B = 120° and AB = 5, find the length of altitude from A to BC.",
  "answer": "5 * sin(60°) = 5√3/2 ≈ 4.33",
  "category": "triangle_altitude",
  "stage": 2,
  "origin": "failure_augmentation_from_problem_043"
}
```

**Example 2: Building a Multi-Agent Code Pipeline**

```
User: Help me implement the Socratic-Geo multi-agent loop in Python.

Approach:
1. Define the agent interfaces:

   class Teacher:
       def generate_variant(self, seed_script: str, diagnostic: str = None) -> str:
           """Returns a modified Python script."""
       def reflect_solve(self, script: str) -> tuple[str, bool]:
           """Solves the problem; returns (answer, matches_reference)."""
       def repi_validate(self, script: str) -> tuple[bool, str]:
           """Executes script, checks visual validity; returns (valid, reason)."""

   class Solver:
       def attempt(self, image_path: str, question: str, k: int = 8) -> list[dict]:
           """Returns k attempts, each with {answer, correct: bool}."""
       def get_failures(self, curriculum: list) -> list:
           """Returns problems where all k attempts failed."""

2. Implement the core loop:

   curriculum = load_seeds("seeds/")
   for stage in range(3):
       solver_results = solver.attempt_all(curriculum)
       failures = solver.get_failures(solver_results)
       new_problems = []
       for problem in failures:
           diagnostic = analyze_failure(problem, solver_results[problem.id])
           for _ in range(3):  # 3 variants per failure
               variant_script = teacher.generate_variant(problem.script, diagnostic)
               answer, matches = teacher.reflect_solve(variant_script)
               if not matches:
                   continue
               valid, reason = teacher.repi_validate(variant_script)
               if not valid:
                   continue
               new_problems.append(build_example(variant_script, answer))
       curriculum.extend(new_problems)
       solver.train(curriculum)  # GRPO training on expanded set

3. The diagnostic prompt for failure analysis:
   "The Solver attempted this geometry problem 8 times and failed every time.
    Problem: {question}
    Solver attempts: {attempts_summary}
    Identify the geometric property or reasoning step the Solver likely missed.
    Then modify the Python code to create a problem that isolates this skill."
```

**Example 3: Quality-Gating Synthetic Math Problems**

```
User: I'm generating math problems but getting too many invalid ones.
How do I implement the Reflect + RePI quality gates?

Approach:
1. Reflect gate implementation:
   def reflect_gate(teacher_llm, generated_script: str) -> bool:
       # Extract the reference answer from the script
       ref_answer = extract_answer(generated_script)
       # Have the Teacher solve it from scratch (image + question only)
       teacher_answer = teacher_llm.solve(
           image=execute_and_render(generated_script),
           question=extract_question(generated_script)
       )
       return answers_match(teacher_answer, ref_answer, tolerance=0.01)

2. RePI gate implementation:
   def repi_gate(generated_script: str) -> tuple[bool, str]:
       try:
           exec(generated_script)  # Must execute without errors
       except Exception as e:
           return False, f"Execution error: {e}"
       image = load_rendered_image()
       # Check for degenerate geometry
       if detect_collinear_points(image):
           return False, "Degenerate: collinear points"
       # Check label placement
       if detect_overlapping_labels(image):
           return False, "Labels overlap"
       # Check text-image alignment
       described_elements = parse_description(generated_script)
       visible_elements = detect_geometric_elements(image)
       if not all(e in visible_elements for e in described_elements):
           return False, "Missing visual elements"
       return True, "Valid"

3. Apply both gates in sequence (Reflect first since it's cheaper):
   problems_admitted = 0
   for script in candidate_scripts:
       if not reflect_gate(teacher, script):
           log("Rejected by Reflect: unsolvable or ambiguous")
           continue
       valid, reason = repi_gate(script)
       if not valid:
           log(f"Rejected by RePI: {reason}")
           continue
       curriculum.append(script)
       problems_admitted += 1
```

## Best Practices

- **Do:** Always implement both quality gates (Reflect and RePI). The paper shows that removing either one degrades final model performance. Reflect catches logical errors; RePI catches visual/rendering errors.
- **Do:** Use the Solver's actual failure distribution to drive augmentation priority. Problems that fail across all k attempts are more valuable augmentation targets than borderline cases.
- **Do:** Keep problems as executable code (Python scripts with parameterized geometry), not as static images. This enables systematic variation, reproducibility, and automated validation.
- **Do:** Stage the curriculum expansion (3 stages works well). Training the Solver on each stage before collecting failures for the next stage ensures augmentation targets real gaps, not random noise.
- **Avoid:** Generating problems randomly and filtering post-hoc. The core insight of this framework is that generation should be *driven by* learning failures, not decoupled from them.
- **Avoid:** Letting the Solver see the Teacher's reference solutions during training. The Solver must learn from binary reward signals only (correct/incorrect), which prevents shortcut learning and improves generalization.
- **Avoid:** Skipping the RePI visual validation step even when problems "look fine" programmatically. Degenerate geometries (near-zero angles, overlapping points, off-screen labels) are common failure modes that silently corrupt training.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Teacher generates unsolvable problems | Parameter ranges produce degenerate geometry | Add explicit parameter constraints (e.g., all triangle angles > 5 degrees) and tighten RePI checks |
| Reflect gate rejects too many problems (>60%) | Teacher prompt is too creative, producing ambiguous questions | Constrain the Teacher to modify parameters and add elements, not rewrite the question format |
| RePI validation fails on rendering | matplotlib/tikz version differences, font issues | Pin rendering dependencies; use headless rendering with fixed DPI and figure size |
| Solver never fails (no augmentation triggers) | Seed problems are too easy or Solver is too strong | Add harder seed categories or reduce Solver capacity to expose genuine reasoning gaps |
| Curriculum grows but Solver performance plateaus | New problems are too similar to existing ones | Add diversity constraints to the Teacher: require different geometric configurations, not just parameter variations |
| Code execution timeout during RePI | Generated script contains infinite loops or extremely complex drawings | Set a strict execution timeout (5-10 seconds) and reject scripts that exceed it |

## Limitations

- **Domain specificity.** The parameterized-code approach works best for domains where problems can be expressed as executable scripts (geometry, plots, diagrams, circuit layouts). It is less natural for free-form text or photographic data.
- **Teacher quality ceiling.** The Teacher agent must be capable enough to generate valid problems and solve them correctly. If the Teacher itself has reasoning gaps, Reflect will pass flawed problems. Use the strongest available model for the Teacher role.
- **Computational cost.** Each augmentation round requires executing code, rendering images, running the Teacher's Reflect check, and training the Solver. For large seed sets, this can be expensive even though it is far more efficient than random generation.
- **Diversity saturation.** After several stages, the Teacher may struggle to generate genuinely novel problems from the same failure patterns. Monitor diversity metrics (problem category distribution, parameter entropy) and introduce new seed categories when saturation is detected.
- **Single-domain validation.** The paper validates on geometric reasoning specifically. Adapting to other domains (chemistry diagrams, UML generation, architectural layouts) requires re-designing the parameterization scheme and quality gates.

## Reference

**Paper:** [Socratic-Geo: Synthetic Data Generation and Geometric Reasoning via Multi-Agent Interaction](https://arxiv.org/abs/2602.03414v1) (Jiao et al., 2026)

Key sections to study: Algorithm 1 (the core Teacher-Solver loop), Section 3.2 (Reflect and RePI quality gates), Section 3.3 (GRPO-based Solver training), and the ablation study showing the impact of removing each quality gate on final benchmark performance.