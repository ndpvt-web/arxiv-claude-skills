---
name: "thinking-frames-visual-context"
description: "Decompose complex visual reasoning and spatial planning tasks into frame-by-frame intermediate steps, using visual context as explicit control signals and scaling inference by generating more intermediate frames. Use when: 'plan a path through this grid', 'solve this spatial puzzle step by step', 'generate intermediate visual states', 'break visual reasoning into frames', 'scale visual inference with more steps', 'use visual context for spatial planning'."
---

# Thinking in Frames: Frame-Based Visual Reasoning and Test-Time Scaling

This skill teaches Claude to apply the "Thinking in Frames" paradigm from Li et al. (2026), where complex visual reasoning problems are decomposed into sequences of intermediate visual states (frames) that bridge an initial state to a solution. Instead of attempting to jump directly from problem to answer, Claude generates or describes a chain of incremental visual snapshots — each frame acting as an explicit reasoning step. The technique leverages two key amplifiers: **visual context** (providing geometric/spatial anchors in the initial state to constrain reasoning) and **test-time scaling** (allocating more intermediate frames to harder problems, analogous to chain-of-thought scaling for text).

## When to Use

- When a user asks to plan a path through a maze, grid, or graph and needs step-by-step spatial reasoning
- When solving spatial arrangement puzzles (tangrams, jigsaw, tiling, packing problems)
- When the user needs to describe or generate intermediate visual states between a start and goal configuration
- When building pipelines that use video/image generation models for planning rather than just media output
- When implementing test-time compute scaling for visual tasks (allocating more inference budget to harder instances)
- When a user wants to convert a spatial reasoning problem into a sequence of discrete visual transitions
- When designing prompts or systems that condition generative models on visual context (icons, shapes, reference images) instead of text-only descriptions

## Key Technique

**Frame-based reasoning** reformulates visual planning as sequence generation. Given an initial visual state v_0, a goal g, and constraints c, the system produces a trajectory V = {v_0, v_1, ..., v_T} where each transition v_t -> v_{t+1} represents one reasoning step. This is analogous to chain-of-thought prompting, but the "thoughts" are visual frames rather than text tokens. The critical insight is that spatial relationships, geometric transformations, and continuous movements are expressed far more faithfully as pixel-level frames than as text descriptions — text creates an information bottleneck for spatial reasoning (GPT-5.2 achieves only 12.5% exact match on maze navigation vs. 96%+ for frame-based approaches).

**Visual context as control** means embedding explicit geometric anchors into the initial frame rather than describing them textually. For maze navigation, placing the agent icon directly in the first frame lets the model learn a conditional policy P(Trajectory | Icon, Layout) that generalizes to unseen icons with near-zero degradation. For spatial puzzles, providing piece shapes at canonical orientations in a sidebar yields 68% accuracy vs. 0.8% when pieces must be inferred without visual reference. The hierarchy is strict: more visual context = better generalization. Pearson correlation >= 0.6 between visual consistency and task success.

**Visual test-time scaling** allocates more frames to harder problems. By increasing the frame budget from 81 to 121 frames, out-of-distribution maze performance improves by +16 percentage points. A scaling factor kappa (frames per discrete step) controls granularity — at kappa=9, emergent self-correction behavior appears where the model backtracks from wrong turns mid-sequence. This scaling helps sequential discrete planning but has diminishing returns for continuous high-change manipulation (geometric deformation accumulates over long windows).

## Step-by-Step Workflow

1. **Classify the problem type.** Determine whether the task involves sequential discrete planning (path finding, navigation, turn-based moves) or continuous spatial manipulation (rotation, translation, assembly). This determines whether test-time scaling will help (sequential: yes, continuous: limited).

2. **Define the state representation.** Encode the initial state as a structured visual description or image specification. For grids: cell contents, walls, start/goal markers. For puzzles: piece geometries, colors, target silhouette. Always prefer visual/geometric representations over text narratives for spatial content.

3. **Embed visual context into the initial state.** Place all control signals directly into the first frame rather than describing them in text. This means: render the agent icon at its starting position, display reference shapes in a sidebar, show the goal state or target outline. Visual context should preserve geometry (position, orientation, scale) that the reasoning chain needs to maintain.

4. **Determine the frame budget.** For sequential planning: allocate kappa frames per discrete step, starting with kappa=7. For N expected steps, total frames = N * kappa. If the problem is out-of-distribution (larger grid, longer path than training examples), increase kappa to 9-11. For continuous manipulation: keep frame count moderate (under 100) to avoid geometric drift.

5. **Generate the frame sequence.** Produce each intermediate frame as an incremental change from the previous one. Each frame should show exactly one atomic action: one grid step, one piece movement, one rotation. Describe or render the full visual state at each step — do not skip to the end or summarize multiple transitions.

6. **Validate visual consistency across frames.** Check that persistent elements (walls, obstacles, piece shapes, colors) remain unchanged between frames. Flag any frame where geometry deforms, objects disappear, or spatial relationships violate constraints. Visual consistency correlates directly with solution correctness.

7. **Apply self-correction when detecting errors.** If a frame reveals the trajectory has entered a dead end or placed a piece incorrectly, generate corrective frames that backtrack. Do not restart from scratch — the model should demonstrate within-sequence error recovery (this emerges naturally at sufficient frame budgets).

8. **Extract the solution from the frame sequence.** Convert the visual trajectory into the requested output format: a list of actions (UP, RIGHT, DOWN...), coordinate sequences, transformation parameters, or a final assembled state. Verify the extracted solution against the original constraints.

9. **Scale up if the initial attempt fails.** If the solution is incorrect or incomplete, increase the frame budget by 50% and regenerate. For maze-like tasks, this consistently improves out-of-distribution performance. For manipulation tasks, instead increase visual context (add reference orientations, gridlines, or alignment guides).

10. **Report both the solution and the reasoning trace.** Present the final answer alongside a summary of the frame-by-frame reasoning, highlighting any backtracking or correction points. This makes the reasoning auditable.

## Concrete Examples

**Example 1: Grid Path Planning**

User: "Find the shortest path through this 7x7 maze from (0,0) to (6,6). Walls at: (1,0), (1,1), (1,2), (3,3), (3,4), (5,5). Show your reasoning step by step."

Approach:
1. Classify: sequential discrete planning on a grid — frame-based scaling applies
2. Render initial state as a grid with walls marked, agent at (0,0), goal at (6,6)
3. Set kappa=9 (7x7 is spatially OOD relative to typical training distributions)
4. Generate frames, one per grid step, showing agent position advancing

Output:
```
Initial State (Frame 0):
  Agent: (0,0) | Goal: (6,6) | Walls: {(1,0),(1,1),(1,2),(3,3),(3,4),(5,5)}

Frame 1: Agent moves to (0,1) — RIGHT. Open path, no wall.
Frame 2: Agent moves to (0,2) — RIGHT. Open path.
Frame 3: Agent moves to (0,3) — RIGHT. (1,2) wall is below, doesn't block.
  [... intermediate frames omitted for brevity ...]
Frame 9: Agent reaches (3,2). Cannot go to (3,3) — wall. Detour DOWN.
Frame 10: Agent moves to (4,2) — DOWN. Bypassing wall cluster.
  [... self-correction: frames 9-11 show rerouting around obstacle ...]
Frame 16: Agent arrives at (6,6). Goal reached.

Solution: RIGHT x3, DOWN x3, RIGHT x3, DOWN x3, RIGHT x3, DOWN — 16 steps
Path: (0,0)->(0,1)->(0,2)->(0,3)->(1,3)->(2,3)->(2,4)->(2,5)->(2,6)->
      (3,6)->(4,6)->(4,5)->(5,4)->(6,4)->(6,5)->(6,6)
```

**Example 2: Tangram-Style Spatial Assembly**

User: "I have 4 colored triangles and 1 square. Arrange them to fill a large square target. Describe the placement step by step."

Approach:
1. Classify: continuous spatial manipulation — limit frame count, maximize visual context
2. Define pieces with exact geometry (vertices, colors) and target silhouette
3. Embed visual context: show all pieces at canonical orientations in a sidebar next to the empty target
4. Generate frames placing one piece at a time, easiest fits first

Output:
```
Visual Context Setup:
  Target: 200x200 square outline
  Pieces (sidebar, canonical orientation):
    - Red triangle: right-isosceles, legs=200
    - Blue triangle: right-isosceles, legs=200
    - Green triangle: right-isosceles, legs=100
    - Yellow triangle: right-isosceles, legs=100
    - White square: 100x100

Frame 0: Empty target with all pieces in sidebar.
Frame 1: Place Red triangle — vertices at (0,0), (200,0), (0,200).
         Fills bottom-left diagonal half.
Frame 2: Place Blue triangle — vertices at (200,0), (200,200), (100,100).
         Fills upper-right region. Gap remains: 100x100 square + two small triangles.
Frame 3: Place White square — center at (100,150), filling the square gap.
Frame 4: Place Green triangle — vertices at (0,200), (100,200), (100,100).
Frame 5: Place Yellow triangle — vertices at (100,100), (100,200), (0,200).
         >> Overlap detected with Green triangle! Backtracking...
Frame 5 (corrected): Yellow triangle at (100,100), (200,200), (100,200).
         All pieces placed. No overlap. Target filled.

Final placement (JSON):
{
  "red_triangle":    {"vertices": [[0,0],[200,0],[0,200]]},
  "blue_triangle":   {"vertices": [[200,0],[200,200],[100,100]]},
  "white_square":    {"center": [50,150], "size": 100},
  "green_triangle":  {"vertices": [[0,200],[100,200],[0,100]]},
  "yellow_triangle": {"vertices": [[100,100],[200,200],[100,200]]}
}
```

**Example 3: Implementing a Frame-Based Planning Pipeline**

User: "Build a Python pipeline that uses a video generation model for maze solving with test-time scaling."

Approach:
1. Structure the pipeline around the three pillars: visual context injection, frame generation, and scaling
2. Implement configurable frame budget with kappa parameter
3. Include visual consistency validation between frames

Output:
```python
import dataclasses
from typing import List, Tuple

@dataclasses.dataclass
class MazeState:
    grid: List[List[int]]  # 0=open, 1=wall
    agent_pos: Tuple[int, int]
    goal_pos: Tuple[int, int]
    agent_icon: str  # path to visual icon — visual context anchor

@dataclasses.dataclass
class Frame:
    step: int
    state: MazeState
    action: str  # UP/DOWN/LEFT/RIGHT or BACKTRACK

def compute_frame_budget(grid_size: int, path_length_estimate: int,
                         is_ood: bool = False) -> Tuple[int, int]:
    """Determine kappa and total frames via test-time scaling."""
    kappa = 7  # default frames per discrete step
    if is_ood:
        kappa = 9  # scale up for out-of-distribution
    if grid_size > 6:
        kappa = 11  # further scale for large grids
    total_frames = path_length_estimate * kappa
    return kappa, min(total_frames, 200)  # architectural ceiling

def render_initial_frame(maze: MazeState) -> "Image":
    """Embed visual context: agent icon at start, goal marker, walls rendered."""
    # Render grid with walls as dark cells, open as light
    # Place agent_icon image at agent_pos (VISUAL context, not text label)
    # Mark goal_pos with distinct visual indicator
    ...

def generate_reasoning_frames(maze: MazeState, kappa: int,
                              model: "VideoGenModel") -> List[Frame]:
    """Generate frame sequence using video model as planning policy."""
    initial_image = render_initial_frame(maze)
    prompt = f"Navigate from start to goal, showing one step per {kappa} frames."
    video = model.generate(
        first_frame=initial_image,
        prompt=prompt,
        num_frames=compute_frame_budget(
            len(maze.grid), estimate_path_length(maze))[1]
    )
    return extract_frames_and_actions(video, kappa)

def validate_visual_consistency(frames: List[Frame]) -> List[int]:
    """Flag frames where walls/grid structure changed (consistency check)."""
    errors = []
    reference_grid = frames[0].state.grid
    for i, frame in enumerate(frames[1:], 1):
        if frame.state.grid != reference_grid:
            errors.append(i)
    return errors

def solve_with_scaling(maze: MazeState, model: "VideoGenModel",
                       max_retries: int = 3) -> List[str]:
    """Full pipeline: generate, validate, scale up on failure."""
    kappa, budget = compute_frame_budget(
        len(maze.grid), estimate_path_length(maze),
        is_ood=(len(maze.grid) > 6))
    for attempt in range(max_retries):
        frames = generate_reasoning_frames(maze, kappa, model)
        errors = validate_visual_consistency(frames)
        if not errors and frames[-1].state.agent_pos == maze.goal_pos:
            return [f.action for f in frames if f.action != "IDLE"]
        kappa = min(kappa + 2, 13)  # scale up
    raise RuntimeError("Failed after scaling retries")
```

## Best Practices

- **Do:** Always embed spatial constraints visually in the initial frame rather than describing them in text. Visual context yields 85x better accuracy than text-only descriptions for spatial manipulation tasks.
- **Do:** Increase frame budget (kappa) for problems that exceed the training distribution — larger grids, longer paths, more pieces. Start with kappa=7 and increment by 2 on failure.
- **Do:** Check visual consistency between every pair of adjacent frames. Persistent elements (walls, shapes, colors) must not change. Consistency correlates with correctness (Pearson r >= 0.6).
- **Do:** Allow self-correction within the frame sequence rather than restarting from scratch. Backtracking frames are a feature, not a bug — they emerge at kappa >= 9.
- **Avoid:** Describing continuous spatial transformations (rotation angles, sub-pixel translations) in text. Text creates an information bottleneck that caps accuracy at 12-58% where visual approaches reach 96%+.
- **Avoid:** Using extremely long frame sequences (>200 frames) for continuous manipulation tasks. Geometric deformation accumulates and degrades visual fidelity. Keep under 100 frames for high-visual-change tasks.

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| Path dead-ends | Agent reaches wall with no valid moves | Increase kappa to enable self-correction frames; backtracking emerges at kappa >= 9 |
| Visual drift | Walls/pieces change shape across frames | Re-anchor visual context every N frames by re-rendering reference elements |
| Geometric deformation | Piece shapes distort over long sequences | Reduce total frame count; split into sub-sequences with fresh visual context at each segment |
| Text-visual mismatch | Text prompt contradicts initial frame content | Remove conflicting text descriptions; let the visual context dominate spatial control |
| OOD failure | Model fails on grid sizes or path lengths never seen in training | Apply test-time scaling: increase frames by 50%, verify improvement, cap at architectural limit |
| Piece overlap | Two pieces placed in same region during assembly | Add explicit overlap check after each frame; trigger backtrack frames on detection |

## Limitations

- **Sequential planning only benefits from scaling.** Test-time frame scaling improves maze-like discrete tasks but shows diminishing returns for continuous manipulation (tangram-style). High visual change accumulates deformation over long frame windows.
- **Architectural frame ceiling.** Most video generation models have positional embedding limits around 200 frames. Beyond this, you must segment the problem into sub-trajectories with fresh visual context at each boundary.
- **Model-dependent.** The technique requires a video generation model fine-tuned (even lightly via LoRA) on domain-specific trajectories. Pure zero-shot video generation without task-specific adaptation performs poorly.
- **Not a replacement for combinatorial search.** Frame-based reasoning is a learned heuristic policy, not an exhaustive search. It will miss optimal solutions on adversarial maze layouts or NP-hard spatial configurations.
- **Visual context quality matters.** The technique degrades sharply when visual anchors are ambiguous, low-resolution, or absent. A tangram puzzle with no shape reference sidebar drops from 68% to 0.8% accuracy.

## Reference

Li, C., Wang, Z., Li, J., Xu, Y., & Zhou, H. (2026). *Thinking in Frames: How Visual Context and Test-Time Scaling Empower Video Reasoning.* arXiv:2601.21037v1. [https://arxiv.org/abs/2601.21037v1](https://arxiv.org/abs/2601.21037v1)

Look for: Section 3 (frame-based formulation and visual context hierarchy), Section 4.2 (test-time scaling law and kappa analysis), Table 1 (maze navigation results across distribution shifts), Table 2 (tangram performance by visual context level), and Figure 3 (self-correction emergence visualization).