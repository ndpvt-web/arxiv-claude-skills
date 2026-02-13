---
name: "vln-pilot-vision-language-as-autonomous"
description: "Build VLLM-driven autonomous navigation agents that interpret natural language instructions and ground them in visual observations to produce discrete action commands. Use when: 'build a vision-language navigation agent', 'drone navigation with natural language', 'VLLM pilot for indoor robot', 'FSM-based autonomous navigation', 'language-grounded visual action planning', 'closed-loop vision-language control system'."
---

# VLN-Pilot: Vision-Language Model as Autonomous Navigation Agent

This skill enables Claude to architect and implement autonomous navigation systems where a large Vision-Language Model (VLLM) replaces a human operator. Based on the VLN-Pilot framework, the core technique is a three-module closed-loop pipeline: a simulator/robot producing visual observations, a Python controller managing state transitions via a finite-state machine (FSM), and a VLLM that receives structured prompts (system role + state-specific instructions + output schema) along with camera images to produce discrete motion commands and state transitions. This pattern generalizes beyond drones to any embodied agent that must follow natural language instructions in a visually-observed environment.

## When to Use

- When the user wants to build an autonomous robot or drone that follows natural language instructions using a vision-language model (GPT-4V, Gemini, etc.)
- When designing a closed-loop perception-action system where an LLM/VLLM makes high-level navigation decisions from camera images
- When implementing a finite-state machine for robot task execution where state transitions are determined by a language model analyzing visual input
- When creating a simulation-to-VLLM pipeline (Unity, Gazebo, AirSim, Habitat) that sends observations to a cloud-hosted model and parses structured action responses
- When the user needs a discrete action space definition for VLLM-controlled navigation (forward distances, rotation angles, lateral shifts)
- When building prompt templates that combine system role, topological maps, FSM state context, and JSON output schemas for embodied AI agents

## Key Technique

VLN-Pilot's central insight is that VLLMs can serve as "supervisory controllers" — they handle high-level semantic reasoning (which room am I in? where is the door? what does the instruction require next?) while a deterministic finite-state machine handles low-level execution constraints. This hybrid separates what the VLLM is good at (language grounding, visual recognition, common-sense reasoning) from what it is bad at (precise metric spatial reasoning, volumetric awareness). The FSM constrains the VLLM's action choices per state, preventing nonsensical commands.

The perception-action loop works as follows: each cycle, the simulator sends a frontal RGB image (base64-encoded), drone position/orientation, and collision status to a Python controller. The controller constructs a three-part prompt — (1) system role defining the pilot persona, input/output formats, and reasoning requirements; (2) state-specific instructions listing the current goal, policy rules, allowed movements, and valid state transitions; (3) a strict JSON output schema requiring room identification, a single motion command, the next FSM state, and a visual scene description. The VLLM returns a JSON response which the controller validates against state constraints before translating it into simulator commands.

A critical design decision is the **discrete, predefined action space**: forward movements at 10/25/50 cm, rotations at 15/45/90 degrees, and lateral shifts at 10 cm. This avoids asking the VLLM to produce continuous values (which it cannot do reliably) and instead frames navigation as a classification problem over a small set of motion primitives. The paper found that GPT-4.1 significantly outperformed Gemini-2.5-Flash, partly because GPT adopted tighter proximity thresholds for spatial decisions while Gemini's conservative centering behavior caused oscillatory re-alignment and step-limit exhaustion.

## Step-by-Step Workflow

1. **Define the discrete action space.** Enumerate all motion primitives the agent can execute as labeled commands (e.g., `A1: forward 10cm`, `B2: rotate right 45deg`, `D1: lateral left 10cm`, `E: no-op`). Group them by category (forward, rotate-left, rotate-right, lateral, null). Assign each a fixed duration and displacement.

2. **Design the finite-state machine.** Map the task into sequential states (e.g., `recognize_room → search_door → orient_to_door → traverse_door → search_object → reach_object → describe_object`). For each state, define: the goal, allowed action subset, valid successor states, and termination conditions. Store this as a JSON or Python dataclass structure.

3. **Build the topological map.** Represent the environment as a node-edge graph where nodes are rooms/zones and edges are traversable connections (doors, hallways). This gives the VLLM spatial context without requiring metric coordinates. Serialize as JSON for prompt injection.

4. **Construct the three-part prompt template.** Create a system prompt establishing the VLLM as a drone/robot pilot, specifying that it receives a camera image and must output a single valid JSON action. Create state-specific prompt sections loaded dynamically based on the current FSM state. Define the JSON output schema with required fields: `current_location`, `action_command`, `next_state`, `scene_description`, and optionally `reasoning`.

5. **Implement the Python controller.** Build a loop that: (a) receives observations from the simulator/robot (RGB image, pose, collision flag), (b) base64-encodes the image and constructs the full prompt with current state context, (c) sends to the VLLM API (OpenAI, Google, etc.), (d) parses and validates the JSON response against the FSM's allowed actions for the current state, (e) translates the action command into simulator/robot movement, (f) updates the FSM state.

6. **Add validation and fallback logic.** Reject VLLM responses that specify disallowed actions for the current state. On parse failure or invalid action, retry the prompt (up to 2 retries) or execute the null action (`E: no-op`). Enforce a maximum step count (e.g., 50 steps) to prevent infinite loops.

7. **Implement collision handling.** When the simulator reports a collision, inject this into the next prompt cycle as additional context (e.g., `"collision_detected": true`). Optionally force a backward movement or rotation before resuming VLLM decision-making.

8. **Wire up the simulation environment.** Use Unity ML-Agents, AirSim, Habitat-Sim, or Gazebo to produce RGB observations and accept discrete motion commands. Communicate via Flask/FastAPI middleware or direct Python bindings. Ensure the observation-action cycle completes within acceptable latency for the application.

9. **Run evaluation with multiple spawn points and repetitions.** Test each navigation instruction from at least 3 starting positions with 5 repetitions each. Track success rate, average steps to completion, collision count, and step-limit failures. Compare VLLM providers to identify which handles spatial reasoning best.

10. **Iterate on prompt engineering.** Tune spatial language in prompts (e.g., define what "centered on the door" means with explicit pixel-region guidance). Add few-shot examples of correct action selections for ambiguous visual scenes. Adjust the action granularity if the VLLM consistently overshoots or undershoots targets.

## Concrete Examples

**Example 1: Building a VLLM-controlled drone navigation agent in Python**

User: "I want to build a system where GPT-4V controls a simulated drone to navigate between rooms based on natural language commands."

Approach:
1. Define the action space as a Python enum:
```python
from enum import Enum

class DroneAction(Enum):
    FORWARD_10CM = ("A1", {"dx": 0.10, "duration": 0.5})
    FORWARD_25CM = ("A2", {"dx": 0.25, "duration": 1.0})
    FORWARD_50CM = ("A3", {"dx": 0.50, "duration": 1.5})
    ROTATE_RIGHT_15 = ("B1", {"dyaw": -15, "duration": 0.3})
    ROTATE_RIGHT_45 = ("B2", {"dyaw": -45, "duration": 0.8})
    ROTATE_RIGHT_90 = ("B3", {"dyaw": -90, "duration": 1.2})
    ROTATE_LEFT_15 = ("C1", {"dyaw": 15, "duration": 0.3})
    ROTATE_LEFT_45 = ("C2", {"dyaw": 45, "duration": 0.8})
    ROTATE_LEFT_90 = ("C3", {"dyaw": 90, "duration": 1.2})
    LATERAL_LEFT = ("D1", {"dy": 0.10, "duration": 0.5})
    LATERAL_RIGHT = ("D2", {"dy": -0.10, "duration": 0.5})
    NO_OP = ("E", {"duration": 0.0})
```

2. Define the FSM:
```python
FSM_CONFIG = {
    "recognize_room": {
        "goal": "Identify the current room from visual cues",
        "allowed_actions": ["E"],
        "valid_transitions": ["search_door", "search_object"],
    },
    "search_door": {
        "goal": "Rotate to locate a door leading toward the target room",
        "allowed_actions": ["B1","B2","B3","C1","C2","C3","E"],
        "valid_transitions": ["orient_to_door"],
    },
    "orient_to_door": {
        "goal": "Align the drone to face the door center",
        "allowed_actions": ["B1","C1","D1","D2","A1","E"],
        "valid_transitions": ["traverse_door"],
    },
    "traverse_door": {
        "goal": "Fly through the doorway into the next room",
        "allowed_actions": ["A1","A2","A3","B1","C1","E"],
        "valid_transitions": ["recognize_room"],
    },
    # ... additional states for object search/reach
}
```

3. Build the prompt and call the VLLM:
```python
import base64, json, openai

def build_prompt(state, topo_map, prev_action):
    system = (
        "You are an autonomous indoor drone pilot. You receive a frontal camera "
        "image and must output exactly one JSON object with keys: current_location, "
        "action_command, next_state, scene_description. Output ONLY valid JSON."
    )
    state_cfg = FSM_CONFIG[state]
    user = (
        f"Current FSM state: {state}\n"
        f"Goal: {state_cfg['goal']}\n"
        f"Allowed actions: {state_cfg['allowed_actions']}\n"
        f"Valid next states: {state_cfg['valid_transitions']}\n"
        f"Topological map: {json.dumps(topo_map)}\n"
        f"Previous action: {prev_action}\n"
        f"Instruction: Go to the bedroom and find the lamp."
    )
    return system, user

def get_vllm_action(image_bytes, state, topo_map, prev_action):
    system, user = build_prompt(state, topo_map, prev_action)
    b64_img = base64.b64encode(image_bytes).decode()
    response = openai.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": system},
            {"role": "user", "content": [
                {"type": "text", "text": user},
                {"type": "image_url", "image_url": {
                    "url": f"data:image/jpeg;base64,{b64_img}"
                }},
            ]},
        ],
        response_format={"type": "json_object"},
    )
    return json.loads(response.choices[0].message.content)
```

**Example 2: Adding collision recovery and step-limit enforcement**

User: "The drone keeps getting stuck in doorframes. How do I handle collisions?"

Approach:
1. Detect collision from simulator feedback
2. Inject collision context into the next VLLM prompt
3. Optionally force a recovery action before resuming

```python
MAX_STEPS = 50
COLLISION_RECOVERY_ACTIONS = ["A1"]  # small backward nudge handled by sim

def navigation_loop(sim, topo_map, instruction):
    state = "recognize_room"
    prev_action = "E"
    for step in range(MAX_STEPS):
        obs = sim.get_observation()  # {image, position, yaw, collision}

        if obs["collision"]:
            # Force a small backward movement, then add context
            sim.execute_action("retreat_10cm")
            obs = sim.get_observation()

        result = get_vllm_action(
            obs["image"], state, topo_map, prev_action,
            extra_context={"collision_just_occurred": obs["collision"]}
        )

        # Validate action against FSM
        if result["action_command"] not in FSM_CONFIG[state]["allowed_actions"]:
            result["action_command"] = "E"  # fallback to no-op

        sim.execute_action(result["action_command"])
        prev_action = result["action_command"]

        if result["next_state"] in FSM_CONFIG[state]["valid_transitions"]:
            state = result["next_state"]

        if state in ("stay_in_room", "describe_object"):
            return {"success": True, "steps": step + 1}

    return {"success": False, "reason": "step_limit_exceeded"}
```

**Example 3: Prompt template for a specific FSM state**

User: "Show me what the full prompt looks like for the 'orient_to_door' state."

Output:
```json
{
  "system": "You are an autonomous indoor drone pilot operating in a GPS-denied environment. You receive one frontal RGB camera image per cycle. You must output a single JSON object. Do not include any text outside the JSON. Required keys: current_location (string), action_command (string from allowed set), next_state (string from valid transitions), scene_description (one sentence describing what you see).",
  "state_context": "FSM State: orient_to_door. Goal: Align the drone so the target doorway is centered in the camera frame. The door should occupy the central third of the image horizontally. If the door is to the left, use C1 (rotate left 15 deg). If to the right, use B1 (rotate right 15 deg). If nearly centered but offset, use D1/D2 for lateral adjustment. If centered, transition to traverse_door. Allowed actions: B1, C1, D1, D2, A1, E. Valid next states: traverse_door.",
  "topological_map": {"nodes": ["living_room", "bedroom", "bathroom"], "edges": [["living_room","bedroom"],["living_room","bathroom"]]},
  "previous_action": "C2",
  "instruction": "Navigate to the bedroom."
}
```

## Best Practices

- **Do:** Use a discrete, enumerated action space. VLLMs cannot reliably produce continuous numeric values for velocity or angle — frame every decision as a classification over labeled commands.
- **Do:** Constrain available actions per FSM state. This prevents the VLLM from outputting semantically valid but contextually wrong commands (e.g., "forward 50cm" while still searching for a door by rotating).
- **Do:** Include the previous action and collision status in every prompt. This gives the VLLM temporal context to avoid repeating failed movements or oscillating.
- **Do:** Enforce JSON output with schema validation. Use `response_format={"type": "json_object"}` with OpenAI or equivalent structured output modes. Parse and validate every field before execution.
- **Avoid:** Asking the VLLM to estimate distances or judge whether the robot physically fits through a gap. VLLMs lack volumetric spatial awareness — handle clearance checks with sensor data or predefined thresholds.
- **Avoid:** Sending both front and rear camera images in the same prompt unless necessary. Each additional image increases latency and token cost. The frontal image carries the primary decision signal.

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| VLLM returns invalid JSON | JSON parse exception | Retry up to 2 times, then execute no-op (`E`) |
| Action not in allowed set for current state | FSM validation check | Replace with no-op, log warning |
| Step limit exceeded (50 steps) | Counter in main loop | Terminate episode, report failure with last state |
| VLLM API timeout or rate limit | HTTP error / timeout | Exponential backoff with 3 retries, then pause episode |
| Collision detected | Simulator collision flag | Execute retreat action, inject collision context into next prompt |
| VLLM hallucinates a room not in topological map | Validate `current_location` against map nodes | Override with last known valid location |
| State transition not in valid set | FSM transition validation | Keep current state, log the attempted invalid transition |

## Limitations

- **No volumetric awareness.** VLLMs perceive 2D images and cannot judge whether the robot's physical body fits through a narrow passage. Supplement with depth sensors or predefined clearance thresholds for real-world deployment.
- **Latency.** Each perception-action cycle requires a round-trip API call (typically 1-5 seconds with cloud VLLMs). This rules out real-time reactive flight and limits the approach to slow, deliberate navigation.
- **Cost at scale.** Each step sends an image + prompt to a commercial API. A 50-step episode with GPT-4o costs roughly $0.50-1.00. Long-running or high-frequency applications need budget controls.
- **Spatial reasoning gaps.** VLLMs interpret "centered" and "close to" inconsistently. GPT models tend toward aggressive proximity thresholds while Gemini models are overly conservative, causing oscillation. Explicit pixel-region guidance in prompts partially mitigates this.
- **Single-camera limitation.** With only a frontal camera, the agent has no awareness of obstacles behind or beside it. The FSM design must account for blind spots.
- **Simulation-to-real gap.** The framework is validated only in photorealistic simulation (Unity). Real-world transfer requires handling lighting variation, motion blur, and physical dynamics not present in simulation.

## Reference

[VLN-Pilot: Large Vision-Language Model as an Autonomous Indoor Drone Operator](https://arxiv.org/abs/2602.05552v1) — Dominguez-Dager et al., 2026. Focus on Section 3 (system architecture and FSM design), Section 4 (prompt engineering strategy and action space), and Section 5 (experimental comparison of GPT-4.1 vs Gemini-2.5-Flash on spatial reasoning tasks).