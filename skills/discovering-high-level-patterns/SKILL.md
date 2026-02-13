---
name: "discovering-high-level-patterns"
description: "Extract high-level semantic patterns from fine-grained simulation or event logs using LM-guided program synthesis. Transforms raw numerical traces into annotated pattern timelines, then composes reward/query programs from natural language goals. Use when: 'analyze simulation logs for patterns', 'find high-level events in trace data', 'generate reward functions from goals', 'summarize physics simulation output', 'build pattern detectors for time-series logs', 'annotate event traces with semantic labels'."
---

# Discovering High-Level Patterns from Simulation Traces

This skill enables Claude to transform verbose, fine-grained simulation logs or event traces into compact, semantically meaningful pattern annotations -- then reason over those annotations using natural language. The core technique, from Memery & Subr (2026), synthesizes executable pattern detector programs that scan raw traces and output boolean activation timelines (e.g., "collision at t=3.2", "stable support from t=5 to t=8"). These annotated traces replace raw numerical data as context for downstream tasks like question answering, reward program generation, and planning -- dramatically improving scalability and LM reasoning accuracy over physics or event-driven domains.

## When to Use

- When the user has simulation output (physics engine logs, game replays, robotics telemetry) and wants to identify meaningful events or phases within it
- When building reward functions or fitness functions from natural language goal descriptions for RL or optimization
- When summarizing long time-series logs into human-readable event narratives
- When the user asks to "find patterns" or "detect events" in sequential state data (positions, velocities, contacts, statuses over time)
- When creating a domain-specific language (DSL) for composing temporal/logical queries over event traces
- When raw trace data is too large or too granular for direct LM context and needs coarse-grained abstraction

## Key Technique

**The scalability problem.** Feeding raw simulation traces directly to an LM fails because traces contain thousands of timesteps of numerical state (positions, velocities, contacts per object). This exceeds context limits and overwhelms the LM's ability to reason about physics. The paper's insight is to insert an intermediate representation: a pattern annotation matrix where each row is a timestep and each column is a named high-level pattern (e.g., "sliding_contact", "lever_launch", "global_rest"), with boolean or continuous activation values.

**Program synthesis for pattern detection.** Rather than hand-coding detectors, the method uses FunSearch-style evolutionary program synthesis. An LM proposes candidate Python functions that read trace data and return activation signals. These candidates are scored on a fitness function combining: (1) correlation between state-space distances and annotation distances across trajectories, (2) novelty relative to existing detectors, and (3) penalties for excessive length or computation time. Candidates are iteratively mutated, evaluated, and refined. Natural language descriptions from domain experts ("a lever launches a ball") seed the initial candidates, and additional patterns are self-discovered by analyzing LM reasoning traces from Q&A tasks.

**Reward program composition via DSL.** Once a pattern library exists, natural language goals like "Launch the green ball into the second bucket" are translated into structured reward programs using three predicate types: EVENT (checks pattern occurrence with parameters), temporal predicates (BEFORE, AFTER, DURING for ordering), and logical predicates (AND, OR, NOT for composition). Dense reward is computed by averaging the fraction of satisfied subclauses, enabling gradient-rich optimization signals instead of sparse binary rewards.

## Step-by-Step Workflow

1. **Structure the raw trace data.** Parse simulation output into a sequence of state snapshots `tau = [x_1, x_2, ..., x_T]` where each `x_t` is a dictionary of object IDs mapped to their properties (position, velocity, rotation, contact list, semantic labels). Normalize coordinates and timestamps.

2. **Define the pattern vocabulary.** Collect natural language descriptions of expected high-level patterns from the user or domain knowledge. Examples: "two objects collide", "object rests stably on surface", "object moves then stops", "lever-like launch". Each description becomes a candidate pattern label.

3. **Synthesize pattern detector programs.** For each pattern label, generate a Python function signature `def detect_pattern(trace: List[Dict]) -> List[float]` that returns per-timestep activation values in [0, 1]. Use the LM to propose an initial implementation based on the natural language description and trace schema. Include the trace data format in the prompt so the detector accesses correct fields.

4. **Evaluate and refine detectors.** Score each detector using: (a) correlation -- do traces that are similar in state-space also produce similar annotation vectors? (b) novelty -- does this detector capture information not already covered by existing detectors? (c) penalties for runtime and code length. Iteratively prompt the LM to mutate low-scoring detectors, supplying the previous code and its fitness breakdown. If execution fails, supply the error message and request repair.

5. **Build the pattern annotation matrix.** Run all accepted detectors over each trace to produce a matrix of shape `(T, num_patterns)`. Each cell is the activation level of pattern `j` at timestep `t`. This matrix replaces the raw trace as the primary representation.

6. **Compose reward or query programs from goals.** When the user provides a natural language goal, translate it into a DSL expression combining EVENT, temporal (BEFORE, AFTER), and logical (AND, OR, NOT) predicates over the pattern library. Include object-specific parameters where needed. Compute dense reward as the fraction of satisfied subclauses.

7. **Validate reward programs against known outcomes.** Test the composed reward program on traces with known success/failure labels. Check that successful traces score higher than failed ones. Adjust predicate thresholds or rewrite subclauses if ranking is incorrect.

8. **Generate natural language summaries.** Convert the annotation matrix into a textual timeline: "t=0.0-1.2: ball slides along ramp (sliding_contact active). t=1.3: ball collides with block (rigid_collision active). t=1.4-3.0: block at rest on platform (stable_support active)." Use this for Q&A, planning context, or human review.

9. **Iterate the pattern library.** Analyze cases where reward programs or summaries fail. Identify missing patterns by examining raw trace segments that lack any annotation. Synthesize new detectors for these gaps and re-annotate.

## Concrete Examples

**Example 1: Detecting events in a physics simulation log**

```
User: I have a log from a 2D physics engine with timestamped positions and
velocities for 5 objects. Find the key physical events.

Approach:
1. Parse the log into structured snapshots:
   trace = [
     {"t": 0.0, "objects": {"ball": {"pos": [10,50], "vel": [5,-2]}, "platform": {"pos": [30,10], "vel": [0,0]}, ...}},
     {"t": 0.1, "objects": {"ball": {"pos": [10.5,49.8], "vel": [5,-2.1]}, ...}},
     ...
   ]

2. Define candidate patterns from domain knowledge:
   - "collision": two objects' positions converge to within contact distance
     and relative velocity reverses sign
   - "rest_state": an object's velocity magnitude stays below threshold
     for N consecutive steps
   - "free_fall": object has downward-increasing velocity with no contacts

3. Synthesize detector for "collision":
   def detect_collision(trace, obj_a, obj_b, contact_dist=2.0):
       activations = []
       for t in range(1, len(trace)):
           dist = euclidean(trace[t][obj_a]["pos"], trace[t][obj_b]["pos"])
           prev_dist = euclidean(trace[t-1][obj_a]["pos"], trace[t-1][obj_b]["pos"])
           closing = prev_dist > dist
           contact = dist < contact_dist
           activations.append(1.0 if (contact and closing) else 0.0)
       return activations

4. Run all detectors, produce annotation matrix, then summarize:

Output:
   t=0.0-0.8: ball in free_fall toward platform
   t=0.9:     ball collides with platform (collision detected)
   t=1.0-2.5: ball at rest on platform (rest_state active)
   t=2.6:     ball collides with wedge (collision detected)
   t=2.7-3.5: ball in free_fall off edge
```

**Example 2: Generating a reward function from a natural language goal**

```
User: Write a reward function for the goal "Stack the red block on top of
the blue block" given my simulation trace format.

Approach:
1. Identify required patterns from goal semantics:
   - "contact": red block touches blue block
   - "above": red block center-y > blue block center-y
   - "stable_support": both blocks have near-zero velocity for sustained period

2. Compose DSL reward program:
   reward = AND(
     EVENT("contact", {"obj_a": "red_block", "obj_b": "blue_block"}),
     EVENT("above", {"top": "red_block", "bottom": "blue_block"}),
     AFTER("contact", "stable_support",
           {"objects": ["red_block", "blue_block"], "min_duration": 0.5})
   )

3. Implement dense scoring:
   def compute_reward(trace, reward_program):
       subclauses = reward_program.subclauses
       scores = [evaluate_clause(trace, c) for c in subclauses]
       return sum(scores) / len(scores)  # dense: fraction of satisfied subclauses

Output:
   def reward_fn(trace):
       contact = any(detect_contact(trace, "red_block", "blue_block"))
       above = any(detect_above(trace, "red_block", "blue_block"))
       stable = detect_stable_support(trace, ["red_block", "blue_block"],
                                       min_duration=0.5)
       stable_after_contact = stable and contact_before_stable(trace)
       return (float(contact) + float(above) + float(stable_after_contact)) / 3.0
```

**Example 3: Summarizing a robotics telemetry trace for Q&A**

```
User: I have a 10,000-step trace from a robotic arm simulation. Summarize
it so I can ask questions about what happened.

Approach:
1. Define patterns relevant to manipulation:
   - "grasp": gripper width decreasing while object within gripper envelope
   - "lift": grasped object's z-position increasing
   - "place": grasped object's z-position decreasing then gripper opens
   - "transport": object moving laterally while grasped
   - "idle": all joint velocities below threshold

2. Run detectors across 10,000 steps, producing annotation matrix (10000 x 5).

3. Compress into phase summary by merging consecutive identical activations:

Output:
   Phase 1 (steps 0-1200):    idle -- arm stationary, no object contact
   Phase 2 (steps 1201-1450): grasp -- gripper closes on red cube
   Phase 3 (steps 1451-2800): lift + transport -- red cube lifted and moved
   Phase 4 (steps 2801-3100): place -- red cube lowered onto table B
   Phase 5 (steps 3101-3400): idle -- arm returns to home position
   Phase 6 (steps 3401-3650): grasp -- gripper closes on blue cylinder
   ...

Now answerable: "When was the red cube placed?" -> Phase 4, steps 2801-3100
                 "How many objects were grasped?" -> 2 (red cube, blue cylinder)
```

## Best Practices

- **Do:** Start with a small set of 5-8 manually-described patterns seeded by domain knowledge before attempting automatic discovery. The paper found library size plateaus around 12 useful patterns.
- **Do:** Score detectors on correlation (annotation distances should track state-space distances) and novelty (new detectors should capture information not already covered). Both are needed to avoid redundant or vacuous patterns.
- **Do:** Use dense reward (fraction of satisfied subclauses) rather than binary reward when composing reward programs. The paper shows 75% success rate at 250 optimization samples with dense reward vs. 50% with binary.
- **Do:** Include the exact trace data schema (field names, units, coordinate systems) in every synthesis prompt so generated detectors access correct attributes.
- **Avoid:** Feeding raw multi-thousand-step traces directly as LM context. This is the antipattern the entire method exists to solve.
- **Avoid:** Over-engineering detectors with complex ML models. Simple Python functions operating on positions, velocities, and contacts are more interpretable, debuggable, and composable.
- **Avoid:** Skipping the iterative repair step. Initial synthesized programs frequently have bugs. Supply the error message and previous code, then prompt for a fix -- the paper uses this loop as a core part of the pipeline.

## Error Handling

| Problem | Solution |
|---|---|
| Synthesized detector throws runtime errors | Feed the traceback and detector code back to the LM with a repair prompt. Iterate up to 3 times before discarding the candidate. |
| Detector always returns 0 or always returns 1 | Check activation variance across traces. Discard degenerate detectors and re-synthesize with a more specific natural language description or adjusted thresholds. |
| Reward program ranks failed traces higher than successful ones | Validate against labeled trace pairs. Decompose the reward into subclauses and test each independently to isolate the faulty predicate. |
| Pattern library misses important events | Examine raw trace segments where no pattern is active. Ask the user to describe what is happening in those segments, then synthesize a new detector for the gap. |
| Annotation matrix too large for context | Compress by merging consecutive timesteps with identical activations into phase summaries with start/end timestamps. |

## Limitations

- **Domain-specific bootstrapping required.** The pattern vocabulary must be seeded with domain-relevant descriptions. A physics simulation and a network traffic log need entirely different pattern sets -- there is no universal pattern library.
- **Detector quality depends on trace schema.** If the simulation log lacks key fields (e.g., contact forces, object types), certain patterns cannot be detected regardless of synthesis quality.
- **Continuous/soft patterns are harder.** The method works best for discrete events (collisions, grasps) and struggles with gradual phenomena like "slowly heating up" or "gradually losing stability" without careful threshold tuning.
- **Not a substitute for actual physics simulation.** The method annotates traces from an existing simulator -- it does not predict physics. It improves LM reasoning *about* simulations, not the simulations themselves.
- **Scalability ceiling.** While far more scalable than raw traces, very long simulations (millions of steps) with many objects still produce large annotation matrices that may need additional compression.

## Reference

Memery, S. & Subr, K. (2026). *Discovering High Level Patterns from Simulation Traces*. arXiv:2602.10009v1. [https://arxiv.org/abs/2602.10009v1](https://arxiv.org/abs/2602.10009v1)

Key sections to consult: Algorithm 1 (DiscoverPatternDetectors) for the synthesis loop, Algorithm 2 (Evaluate) for the fitness function components (correlation, novelty, penalties), and Section 5 for reward program DSL syntax and dense scoring.