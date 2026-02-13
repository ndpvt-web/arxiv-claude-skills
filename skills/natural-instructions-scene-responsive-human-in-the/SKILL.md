---
name: "natural-instructions-scene-responsive-human-in-the"
description: >
  Build instruction-conditioned autonomous driving planners that accept free-form
  natural language from passengers and generate scene-aware trajectories using
  Vision-Language-Action (VLA) models. Based on the doScenes/OpenEMMA pipeline:
  multi-stage VLM reasoning (scene description, object identification, intent
  estimation) with injected human instructions for trajectory generation.

  Trigger phrases:
  - "Build an instruction-following driving planner"
  - "Add natural language control to an autonomous driving pipeline"
  - "Implement human-in-the-loop trajectory planning with VLA models"
  - "Create a scene-responsive motion planner that takes passenger instructions"
  - "Integrate language instructions into an end-to-end driving model"
  - "Condition trajectory prediction on free-form text input"
---

# Natural-Language Instruction-Conditioned Scene-Responsive Motion Planning

This skill enables Claude to help developers build autonomous driving planners
that accept free-form natural language instructions from passengers and produce
scene-aware trajectories. The core technique comes from adapting OpenEMMA (an
open-source MLLM-based end-to-end driving framework) with the doScenes dataset,
injecting passenger instructions into a three-stage VLM reasoning pipeline
(scene description, object identification, intent estimation) to generate
10-step speed-curvature trajectories. The key insight: instructions with
referentiality (pointing to specific scene objects like "follow the yellow car")
dramatically reduce trajectory error compared to generic commands.

## When to Use

- When the user is building an autonomous driving planner and wants to add
  natural language passenger input as a control signal
- When implementing a multi-stage VLM reasoning chain for robotics or vehicle
  planning (scene perception -> object identification -> intent -> action)
- When designing prompt templates that inject human instructions into an existing
  VLA model without architectural changes
- When evaluating instruction-conditioned trajectory planners using ADE metrics
  on nuScenes-derived datasets
- When classifying natural language instructions by referentiality type (static,
  dynamic, combined) to understand their effect on planner performance
- When building human-in-the-loop systems where free-form language must be
  grounded in visual scene context before generating physical actions

## Key Technique

The paper adapts OpenEMMA, which uses LLaVA-1.6-Mistral-7B as its VLM backbone,
to accept passenger-style natural language instructions. The original OpenEMMA
pipeline has three sequential reasoning stages: (1) **Scene Description** --
the VLM processes front-camera frames and generates text summaries of traffic
lights, vehicles, pedestrians, and lane markings; (2) **Object Identification**
-- the VLM lists 2-3 critical road users with locations and relevance; (3)
**Intent Estimation** -- the VLM determines the ego-vehicle's driving intent
(turn left, right, or go straight). After all three stages, the model outputs a
10-step speed-curvature trajectory.

The instruction injection is elegantly minimal: passenger text is prepended to
the scene-description prompt as `"The passenger says: '<instruction>'. Always
prioritize the passenger's instruction unless it is unsafe..."`. This lets the
VLM weigh the instruction against its own visual perception across all three
reasoning stages, without any architectural modification. The key finding is that
**instruction referentiality matters more than instruction length**. Dynamic
references ("follow the yellow car ahead") yield the lowest trajectory error
(ADE 2.764), while non-referential commands ("go straight") perform worst (ADE
3.397). Typical-length instructions (9-12 words) hit the sweet spot between
clarity and verbosity.

The most dramatic result: instruction conditioning prevented extreme baseline
failures (98.7% ADE reduction overall), particularly in stationary scenarios
where the uninstructed model would predict waypoints outside scene boundaries.
Instructions like "wait for pedestrians to cross" helped the VLM understand it
should remain stopped rather than hallucinating forward motion.

## Step-by-Step Workflow

1. **Set up the VLA model backend.** Install LLaVA-1.6-Mistral-7B (or
   equivalent open-source VLM) locally. Ensure GPU availability -- the model
   fully occupies an RTX 4090-class GPU during inference. Avoid proprietary API
   dependencies for reproducibility.

2. **Structure the three-stage reasoning prompt chain.** Create three sequential
   prompt templates:
   - Stage 1 (Scene Description): "Describe the driving scene in the image.
     Focus on traffic signals, nearby vehicles, pedestrians, and lane markings."
   - Stage 2 (Object Identification): "List 2-3 critical road users. For each,
     state their position relative to ego-vehicle and why they matter."
   - Stage 3 (Intent Estimation): "Based on scene context, determine the
     ego-vehicle's intent: turn left, turn right, or proceed straight."

3. **Inject passenger instructions into Stage 1.** Prepend the instruction to
   the scene-description prompt using this template:
   ```
   The passenger says: '{instruction}'.
   Always prioritize the passenger's instruction unless it is unsafe or
   physically impossible given the current scene.
   ```
   Do NOT create a separate fourth stage -- the instruction must flow through
   all three reasoning stages as accumulated context.

4. **Encode ego-vehicle state as prompt context.** Include historical ego-state
   (speed, heading, position) as structured text in the prompt so the VLM can
   reason about current dynamics alongside the visual frame and instruction.

5. **Generate 10-step speed-curvature trajectories.** Configure the VLM output
   format to produce pairs of (speed, curvature) values for each of the 10
   future timesteps. Define the output schema explicitly in the prompt:
   ```
   Output exactly 10 waypoints as (speed_m_s, curvature_1_m) pairs:
   [(s1, c1), (s2, c2), ..., (s10, c10)]
   ```

6. **Convert speed-curvature to x,y coordinates.** Transform the trajectory
   from curvature-velocity space to Cartesian coordinates for evaluation and
   downstream use:
   ```python
   def curvature_velocity_to_xy(trajectory, dt=0.5):
       x, y, theta = 0.0, 0.0, 0.0
       points = [(x, y)]
       for speed, curvature in trajectory:
           ds = speed * dt
           dtheta = curvature * ds
           theta += dtheta
           x += ds * math.cos(theta)
           y += ds * math.sin(theta)
           points.append((x, y))
       return points
   ```

7. **Classify instruction referentiality.** Tag each instruction as one of four
   types to predict expected planner performance:
   - **Non-referential**: Generic ("go straight", "speed up") -- worst ADE
   - **Static-only**: Fixed landmarks ("turn right at the intersection")
   - **Dynamic-only**: Moving objects ("follow the yellow car") -- best ADE
   - **Static+Dynamic**: Combined ("pass the bus near the traffic light")

8. **Implement the safety gate.** Before executing any trajectory, validate that
   the instruction does not conflict with safety constraints. If the VLM's scene
   description identifies hazards contradicting the instruction, default to the
   safest feasible trajectory.

9. **Evaluate using Average Displacement Error (ADE).** Compute the mean L2
   distance between predicted and ground-truth positions across all timesteps:
   ```python
   def compute_ade(predicted, ground_truth):
       assert len(predicted) == len(ground_truth)
       return sum(
           math.sqrt((p[0]-g[0])**2 + (p[1]-g[1])**2)
           for p, g in zip(predicted, ground_truth)
       ) / len(predicted)
   ```

10. **Filter outliers for fair comparison.** Apply 97.5th-percentile ADE
    filtering when comparing instructed vs. uninstructed baselines. Without
    filtering, the massive reduction in catastrophic failures (stationary scene
    hallucinations) dominates the metric and obscures the nuanced instruction
    benefit (~5.1% improvement on filtered data).

## Concrete Examples

**Example 1: Building the prompt injection for a driving planner**

User: "I have an OpenEMMA-style driving model. How do I add passenger
instruction support?"

Approach:
1. Locate the scene-description prompt template in the pipeline.
2. Add an instruction injection slot before the scene analysis directive.
3. Chain the instruction context through all downstream reasoning stages.

Output -- Modified prompt template:
```python
SCENE_DESCRIPTION_TEMPLATE = """
{instruction_block}

You are an autonomous vehicle's perception system. Analyze the front-camera
image and describe:
1. Traffic signal states visible ahead
2. Vehicles within 50m and their trajectories
3. Pedestrians and cyclists near the planned path
4. Lane markings and road geometry

Provide a structured scene summary.
"""

def build_instruction_block(instruction: str | None) -> str:
    if instruction is None:
        return ""
    return (
        f"The passenger says: '{instruction}'. "
        "Always prioritize the passenger's instruction unless it is unsafe "
        "or physically impossible given the current scene."
    )

def run_pipeline(image, ego_state, instruction=None):
    ctx = {}
    # Stage 1: Scene description with injected instruction
    ctx["scene"] = vlm.generate(
        SCENE_DESCRIPTION_TEMPLATE.format(
            instruction_block=build_instruction_block(instruction)
        ),
        image=image
    )
    # Stage 2: Object identification (instruction context carries forward)
    ctx["objects"] = vlm.generate(
        OBJECT_ID_TEMPLATE.format(**ctx),
        image=image
    )
    # Stage 3: Intent estimation
    ctx["intent"] = vlm.generate(
        INTENT_TEMPLATE.format(**ctx),
        image=image
    )
    # Stage 4: Trajectory generation
    trajectory = vlm.generate(
        TRAJECTORY_TEMPLATE.format(**ctx, ego_state=ego_state),
        image=image
    )
    return parse_trajectory(trajectory)
```

**Example 2: Classifying instructions by referentiality for evaluation**

User: "I have a dataset of passenger instructions. How do I categorize them to
predict planner accuracy?"

Approach:
1. Define referentiality categories and heuristic classifiers.
2. Tag each instruction and report distribution.
3. Use categories to stratify ADE evaluation.

Output:
```python
import re

DYNAMIC_PATTERNS = [
    r"\b(follow|behind|ahead of|next to|pass)\b.*\b(car|truck|bus|bike|pedestrian|vehicle)\b",
    r"\b(car|truck|bus|bike|vehicle)\b.*\b(in front|to the left|to the right)\b",
]
STATIC_PATTERNS = [
    r"\b(at the|near the|past the|before the|after the)\b.*\b(intersection|light|sign|stop|crosswalk|building|corner)\b",
]

def classify_referentiality(instruction: str) -> str:
    text = instruction.lower()
    has_dynamic = any(re.search(p, text) for p in DYNAMIC_PATTERNS)
    has_static = any(re.search(p, text) for p in STATIC_PATTERNS)
    if has_dynamic and has_static:
        return "static+dynamic"   # ADE ~2.783
    if has_dynamic:
        return "dynamic"          # ADE ~2.764 (best)
    if has_static:
        return "static"           # ADE ~3.027
    return "non-referential"      # ADE ~3.397 (worst)

# Stratify evaluation
from collections import defaultdict
results = defaultdict(list)
for scene in dataset:
    cat = classify_referentiality(scene.instruction)
    ade = compute_ade(model.predict(scene), scene.ground_truth)
    results[cat].append(ade)

for cat, ades in sorted(results.items()):
    print(f"{cat:20s}  mean ADE: {sum(ades)/len(ades):.3f}  n={len(ades)}")
```

**Example 3: Handling the stationary-scene failure mode**

User: "My planner hallucinates forward motion when the car should be stopped.
How do I fix this with instructions?"

Approach:
1. Detect stationary ground-truth scenes (near-zero ego speed).
2. For these scenes, ensure instruction injection explicitly signals waiting.
3. Add a post-processing clamp for low-confidence stationary predictions.

Output:
```python
def detect_stationary(ego_state, speed_threshold=0.5):
    """Check if ego-vehicle is effectively stopped."""
    return abs(ego_state.speed_mps) < speed_threshold

def augment_instruction_for_stationary(instruction, ego_state, scene_desc):
    """If scene suggests stopped state, reinforce in instruction context."""
    if detect_stationary(ego_state):
        stationary_cues = [
            "red light", "stop sign", "pedestrian crossing",
            "waiting", "stopped", "yield"
        ]
        if any(cue in scene_desc.lower() for cue in stationary_cues):
            return (
                f"{instruction} Note: the vehicle is currently stopped. "
                "Do not predict forward motion unless the scene clearly "
                "indicates it is safe to proceed."
            )
    return instruction

def clamp_stationary_trajectory(trajectory, ego_state, speed_threshold=0.5):
    """Post-process: if ego is stopped and predicted speeds are low,
    clamp to zero to prevent hallucinated drift."""
    if not detect_stationary(ego_state):
        return trajectory
    avg_pred_speed = sum(s for s, _ in trajectory) / len(trajectory)
    if avg_pred_speed < speed_threshold * 2:
        return [(0.0, 0.0)] * len(trajectory)
    return trajectory
```

## Best Practices

- **Do:** Inject instructions into the earliest reasoning stage so the VLM
  carries the intent through scene description, object identification, and
  intent estimation -- not just at the trajectory generation step.
- **Do:** Encourage users to write instructions in the 9-12 word range with
  dynamic scene references (e.g., "follow the white SUV then turn right at the
  light") for best trajectory alignment.
- **Do:** Use open-source VLMs (LLaVA, etc.) for reproducibility and to avoid
  proprietary API latency in safety-critical real-time systems.
- **Do:** Always evaluate with and without 97.5th-percentile outlier filtering
  to distinguish catastrophic failure prevention from genuine instruction benefit.
- **Avoid:** Creating a separate instruction processing stage -- the power of
  this technique is that a single prompt injection propagates through the
  existing chain-of-thought reasoning without architectural changes.
- **Avoid:** Treating all instructions as equally actionable. The doScenes
  dataset found only 36% of annotations contained actionable instructions.
  Build a classifier to detect and gracefully handle non-actionable input
  ("nice weather today").

## Error Handling

- **VLM produces waypoints outside scene boundaries:** This is the primary
  failure mode for uninstructed baselines. If predicted coordinates exceed
  reasonable bounds (e.g., >100m from ego in a 5-second horizon), clamp or
  reject the trajectory and re-query with an explicit "remain in lane" constraint.
- **Instruction conflicts with safety:** When the VLM's scene description
  identifies pedestrians in the planned path but the instruction says "go
  straight", the safety gate must override. Log the conflict for review.
- **Ambiguous referentiality:** If an instruction references an object the VLM
  cannot identify in the scene ("turn after the blue building" but no building
  is visible), fall back to non-referential behavior and log a confidence warning.
- **GPU memory exhaustion:** LLaVA-1.6-Mistral-7B fully occupies an RTX 4090.
  If running multi-frame evaluation, process clips sequentially rather than
  batching. Budget ~20 minutes per clip for full three-stage reasoning.
- **Non-actionable instructions:** Detect instructions that describe conditions
  rather than actions ("the light is red") and convert them to actionable form
  ("stop at the red light") or skip instruction injection entirely.

## Limitations

- The doScenes annotations are **retrospective** (written after observing the
  driving trajectory), not real-time passenger speech. Real-world instructions
  will be noisier, more ambiguous, and potentially mid-maneuver.
- ADE measures trajectory proximity, not true instruction comprehension. A model
  could achieve low ADE by ignoring the instruction and predicting a common
  trajectory that happens to match.
- The technique was validated on a single VLM (LLaVA-1.6-Mistral-7B) and a
  single dataset (nuScenes + doScenes). Generalization to other models, sensor
  configurations, or geographies is unproven.
- Only front-camera views are used. Instructions referencing objects behind or
  beside the vehicle ("the car on my left") cannot be grounded visually.
- The authors describe this as "a demonstration of feasibility rather than a
  definitive measure of optimal instruction-conditioned planning performance."
  Production deployment would require extensive additional safety validation.

## Reference

**Paper:** [Natural Language Instructions for Scene-Responsive Human-in-the-Loop
Motion Planning in Autonomous Driving using Vision-Language-Action Models](https://arxiv.org/abs/2602.04184v1)
(Martinez-Sanchez, Roy, Greer, 2026). Look for: the three-stage prompt chain
architecture, the instruction injection template, referentiality classification
taxonomy, and the stationary-scene failure mode analysis (Figure 3).