---
name: "iterative-refinement-improves-compositional"
description: "Implement iterative critic-guided refinement loops for compositional image generation. Uses a VLM critic to progressively correct T2I outputs across multiple rounds, with action selection (continue/backtrack/restart/stop) and targeted sub-prompts. Trigger phrases: 'generate a complex image with multiple objects', 'iterative image refinement', 'compositional image generation pipeline', 'critic-in-the-loop image generation', 'build an image refinement agent', 'VLM-guided image editing loop'."
---

# Iterative Refinement for Compositional Image Generation

This skill teaches Claude to build **critic-in-the-loop pipelines** that iteratively refine text-to-image outputs for complex, multi-constraint prompts. Rather than generating once and hoping for the best, the technique uses a vision-language model (VLM) as a critic that scores the current image, identifies what's missing or wrong, and issues targeted editing instructions -- repeating until the image faithfully matches all compositional requirements. The method draws from chain-of-thought reasoning: decompose a hard generation problem into sequential corrections.

## When to Use

- When a user needs to generate images from prompts containing **multiple objects, spatial relations, and attributes** (e.g., "a red cube on top of a blue sphere next to a green cylinder on a wooden table")
- When building an **agentic image generation pipeline** that self-corrects using VLM feedback
- When implementing **test-time compute scaling** for image generation -- trading more inference steps for higher compositional accuracy
- When the user asks to integrate a verifier/critic loop into an existing T2I workflow
- When designing a system that must handle **ConceptMix-style** prompts with 5-7+ simultaneous constraints
- When a user wants to build a **multi-stream image generation agent** that allocates compute budget across parallel and iterative axes

## Key Technique

The core insight is that **iterative refinement with a critic consistently outperforms parallel sampling** at the same compute budget. Given budget B (total generation attempts), splitting it as B = T iterations x M parallel streams with T >> M yields better results than generating M = B independent samples and picking the best. For example, at B=16, using 8 iterative steps across 2 parallel streams beats 16 independent samples.

The pipeline has four components: (1) a **T2I generator** produces an initial image, (2) a **VLM verifier** scores each constraint in the prompt with binary yes/no checks (e.g., "Does the image contain a corgi?" / "Is the hill texture metallic?"), (3) a **VLM critic** receives the image, the prompt, verifier scores, and remaining budget, then selects an action and writes a targeted sub-prompt, and (4) an **image editor** applies the sub-prompt to modify the current image. The critic chooses from four actions: **CONTINUE** (edit the current best image), **BACKTRACK** (revert to the previous step and re-edit), **RESTART** (discard trajectory and regenerate from scratch with a refined prompt), or **STOP** (accept the current image).

What makes this work is the **structured critic prompt**: the VLM receives the full original prompt, the history of sub-prompts applied so far, per-concept binary scores, a cumulative mean score, and the remaining step budget. This context lets the critic make informed decisions about whether to keep editing, revert, or start over -- mimicking how a human artist would iteratively refine a composition.

## Step-by-Step Workflow

1. **Parse the compositional prompt into atomic constraints.** Break the user's complex prompt into individual verifiable elements: objects, attributes (color, size, texture), spatial relations (on top of, next to, behind), and counts. Each becomes a separate verification question.

2. **Initialize M parallel generation streams.** Generate M independent initial images from the full prompt using a T2I model (e.g., DALL-E 3, FLUX, Stable Diffusion 3, Gemini image generation). M=2 is a good default for budgets of 8-16.

3. **Score each stream with the VLM verifier.** For each generated image, ask the VLM a binary yes/no question per constraint (e.g., "Does this image contain a small pink screwdriver?"). Compute the mean score across all constraints to get an overall alignment score.

4. **Invoke the VLM critic with full context.** Send the critic the original prompt, the current image, all previous sub-prompts in this stream, the per-constraint verifier scores, the cumulative mean score, and the remaining iteration budget. The critic outputs an action (CONTINUE, BACKTRACK, RESTART, or STOP) and a targeted sub-prompt describing the specific edit needed.

5. **Execute the selected action.** If CONTINUE: pass the current image and sub-prompt to an image editor model. If BACKTRACK: revert to the previous iteration's image and apply the new sub-prompt. If RESTART: regenerate from scratch using the original prompt augmented with the sub-prompt. If STOP: mark this stream as complete.

6. **Re-score the edited image.** Run the verifier again on the new image to measure improvement. Track the best image and score seen across the entire trajectory.

7. **Repeat steps 4-6 for T iterations per stream.** Continue until all streams reach STOP, hit the iteration budget T, or achieve a perfect verifier score.

8. **Select the best image across all streams.** Compare the highest-scoring images from each parallel stream and return the overall best.

9. **Return the final image with a refinement trace.** Provide the user with the final image plus a log of each refinement step (sub-prompts applied, scores at each stage) so the process is transparent and debuggable.

## Concrete Examples

**Example 1: Multi-object scene generation pipeline**

```
User: Build me a Python pipeline that generates an image of "a tiny pink
screwdriver resting on a large golden anvil, next to a small turquoise
hourglass, on a rustic wooden workbench" and iteratively refines it.

Approach:
1. Parse constraints:
   - Object: pink screwdriver (tiny)
   - Object: golden anvil (large)
   - Object: turquoise hourglass (small)
   - Object: wooden workbench (rustic)
   - Relation: screwdriver ON anvil
   - Relation: hourglass NEXT TO anvil
   - All objects ON workbench

2. Generate initial image via API (e.g., OpenAI DALL-E 3 or fal.ai FLUX)

3. Verify with VLM (e.g., Gemini Flash or GPT-4o):
   - "Does this image contain a tiny pink screwdriver?" -> Yes
   - "Is there a large golden anvil?" -> Yes
   - "Is the screwdriver resting on the anvil?" -> No  <-- violation
   - "Is there a small turquoise hourglass?" -> No     <-- violation
   Score: 4/7 = 0.57

4. Critic decides: CONTINUE
   Sub-prompt: "Edit the image so the pink screwdriver is clearly
   resting on top of the golden anvil, and add a small turquoise
   hourglass next to the anvil on the workbench."

5. Apply edit via image editing model, re-score -> 6/7 = 0.86

6. Critic decides: CONTINUE
   Sub-prompt: "Make the hourglass clearly turquoise-colored."

7. Apply edit, re-score -> 7/7 = 1.0. Critic selects STOP.

Output: Final image + refinement log showing 3 iterations,
score progression [0.57, 0.86, 1.0].
```

**Example 2: Implementing the critic prompt template**

```
User: How should I structure the VLM critic prompt for this pipeline?

Approach: Build a structured prompt with all necessary context.

Output -- Critic system prompt template:

SYSTEM: You are an image composition critic. You will receive:
- A complex image generation prompt
- The current generated image
- A history of editing sub-prompts applied so far
- Per-constraint verification scores (binary yes/no)
- A cumulative mean alignment score
- The remaining editing budget

Your job:
1. Analyze which constraints are NOT yet satisfied
2. Prioritize the most impactful missing element
3. Select one action: CONTINUE, BACKTRACK, RESTART, or STOP
4. Write a concise, specific editing sub-prompt

Rules:
- Choose STOP only when all constraints score yes or budget is 1
- Choose BACKTRACK if the last edit made things worse
- Choose RESTART if cumulative score dropped below 0.3
- Choose CONTINUE in all other cases
- Sub-prompts must reference specific objects, colors, and positions

USER:
Original prompt: "{full_prompt}"
Edit history: {list_of_previous_sub_prompts}
Constraint scores: {per_constraint_yes_no_dict}
Mean score: {mean_score} ({n_satisfied}/{n_total} constraints met)
Remaining budget: {remaining_steps} steps

Respond in JSON:
{
  "analysis": "brief assessment of what's wrong",
  "action": "CONTINUE|BACKTRACK|RESTART|STOP",
  "sub_prompt": "specific editing instruction"
}
```

**Example 3: Budget allocation strategy**

```
User: I have a budget of 12 API calls. How should I split between
parallel streams and iterative refinement?

Approach: Apply the paper's finding that higher iterative allocation
wins on complex prompts.

Output:
For compositional prompts with 5+ constraints:
  - 2 parallel streams x 6 iterations each (recommended)
  - This beats 12 parallel samples with best-of-N selection

For simpler prompts with 2-3 constraints:
  - 3 parallel streams x 4 iterations each
  - Or even 4 streams x 3 iterations

General rule: allocate at least 60-70% of budget to iteration depth.
The paper found I=8, P=2 optimal at B=16, which is 80% iterative.

Implementation:
  streams = 2
  max_iters = budget // streams  # 6
  for stream in range(streams):
      img = generate(prompt)
      for step in range(max_iters):
          score = verify(img, constraints)
          if score == 1.0: break
          action, sub_prompt = critic(img, prompt, score, max_iters - step)
          img = apply_action(action, sub_prompt, img, history)
      candidates.append((img, score))
  return max(candidates, key=lambda x: x[1])
```

## Best Practices

- **Do:** Decompose prompts into atomic, independently verifiable constraints before starting the loop. Each constraint should map to a single yes/no question the VLM can answer reliably.
- **Do:** Include the full edit history and remaining budget in every critic call. The critic makes better action choices when it knows what's been tried and how much budget remains.
- **Do:** Track the best image seen across the entire trajectory, not just the latest one. Edits can sometimes degrade quality, so always keep the high-water mark.
- **Do:** Use a lightweight VLM for in-loop verification (e.g., Gemini Flash) and reserve expensive models (GPT-4o, Gemini Pro) for final evaluation only.
- **Avoid:** Skipping the BACKTRACK action. Without it, the pipeline can get stuck refining a fundamentally broken image instead of reverting to a better checkpoint.
- **Avoid:** Using overly vague sub-prompts like "improve the image." Sub-prompts must name specific objects, attributes, and spatial corrections to be effective.
- **Avoid:** Allocating all budget to parallel streams. The paper shows this is consistently worse than iterative refinement for prompts with 4+ constraints.

## Error Handling

- **VLM hallucination in verification:** The verifier may incorrectly report a constraint as satisfied. Mitigate by asking verification questions in multiple phrasings and requiring consensus, or by using a separate VLM for verification vs. criticism.
- **Editor ignoring instructions:** Image editing models sometimes fail to execute sub-prompts faithfully. If the score doesn't improve after an edit, the critic should select BACKTRACK to revert and try a differently-worded sub-prompt.
- **Score oscillation:** If the score keeps alternating between two values, the critic is likely fighting a tradeoff where fixing one constraint breaks another. Trigger RESTART with a combined sub-prompt addressing both constraints simultaneously.
- **Budget exhaustion without convergence:** If all streams exhaust their budget without reaching a perfect score, return the best image found and report which constraints remain unsatisfied so the user can decide whether to extend the budget or accept the result.
- **API rate limits:** Wrap each API call in retry logic with exponential backoff. The iterative nature means a single failed call shouldn't crash the pipeline -- just retry that step.

## Limitations

- **Depends on VLM accuracy:** The entire pipeline is bottlenecked by the critic's ability to correctly identify what's wrong. If the VLM can't distinguish "on top of" from "next to," no amount of iteration will fix spatial relations.
- **Image editing fidelity:** Current image editors often introduce artifacts or ignore instructions, especially for fine-grained spatial edits. The method works best with high-quality editors (GPT-Image, Gemini image editing).
- **Latency:** Each iteration requires a full VLM call + an image generation/edit call. For real-time applications, the sequential nature of iteration is a bottleneck. Parallel streams help but add cost.
- **Diminishing returns beyond ~8 iterations:** The paper shows most gains come in the first 4-6 steps. Beyond 8 iterations, improvements plateau while costs grow linearly.
- **Not suited for style/aesthetic refinement:** The binary constraint verification works well for compositional correctness (objects, attributes, relations) but poorly for subjective qualities like "beautiful," "cinematic," or "moody lighting."

## Reference

**Paper:** [Iterative Refinement Improves Compositional Image Generation](https://arxiv.org/abs/2601.15286v1) (Jaiswal et al., 2026). Focus on Section 3 (Method) for the algorithm and critic prompt design, Section 4.3 for budget allocation ablations, and Appendix for the full system prompts used with the VLM critic.