---
name: "alrm-agentic-robotic-manipulation"
description: >
  Build agentic LLM-driven robotic manipulation pipelines using the ALRM framework pattern:
  a ReAct-style reasoning loop with dual execution modes (Code-as-Policy for direct code generation,
  Tool-as-Policy for iterative tool-based execution). Generates modular robot control code with
  closed-loop planning, observation, and replanning.
  Trigger phrases: "robot manipulation agent", "agentic robot control", "ReAct robot planner",
  "code-as-policy generation", "tool-as-policy robot", "LLM robotic pipeline"
---

# ALRM: Agentic LLM for Robotic Manipulation

This skill enables Claude to build agentic robotic manipulation systems following the ALRM architecture from Santos et al. (2026). The core pattern decomposes natural language task instructions into a **Task Planner Agent** (ReAct-style reasoning loop that generates subtasks) and a **Task Executor Agent** (converts subtasks into robot actions via either direct code generation or iterative tool calls), connected by an observation feedback channel that enables closed-loop replanning. This is applicable to any robotics project that needs LLM-driven task decomposition, code generation for robot APIs, or adaptive execution with error recovery.

## When to Use

- When the user asks to build an LLM-driven robot control pipeline that decomposes natural language commands into executable robot actions
- When implementing a ReAct-style planning loop for robotic manipulation (think-act-observe-revise cycles)
- When generating Python code that calls robot control APIs (pick, place, move_to, grasp) from natural language instructions
- When designing a system where an LLM iteratively calls robot tool functions and adapts based on execution feedback
- When building a modular agent architecture that separates task planning from task execution in robotics
- When creating a benchmark or evaluation harness for robotic manipulation tasks with linguistically diverse instructions

## Key Technique

ALRM's insight is that robotic manipulation benefits from **separating planning from execution** in an agentic loop, rather than generating a monolithic action sequence. The **Task Planner Agent** uses a ReAct framework (Reason + Act) to decompose a user request like "sort the fruits into the bowl by color" into atomic subtasks such as "get positions of all objects," "pick up the lemon and place it in the bowl," etc. After each subtask executes, the planner receives an observation (e.g., "lemon successfully picked" or "object orange not found") and decides the next subtask or revises its plan. This continues until the task is fulfilled or a step limit is reached.

The framework offers two complementary execution modes. **Code-as-Policy (CaP)** has the executor LLM generate a complete Python script per subtask that calls predefined robot API functions. This is fast (fewer LLM calls) but brittle -- a single code error fails the entire subtask. **Tool-as-Policy (TaP)** instead uses the LLM's tool-calling capability to emit one function call per step, observe the result, and emit the next call. TaP is slower but more robust because small errors can be corrected mid-execution without replanning from scratch.

The predefined action set is deliberately small and composable: `pick`, `place`, `move_to`, `move_to_home_pos` for robot control; `get_objects`, `get_reference_names` for perception; `compute_grasp`, `get_pose` for pose computation. The LLM receives these as typed Python function signatures with descriptions, parameters, and return types, plus a one-shot pick-and-place example. This constrained action vocabulary prevents hallucinated robot commands while still covering multi-step manipulation tasks.

## Step-by-Step Workflow

1. **Define the robot action API as typed Python functions.** Create a module exposing 6-10 atomic actions (e.g., `pick(object_name)`, `place(object_name, target_pose)`, `get_objects() -> list[str]`, `get_pose(object_name) -> Pose`). Each function must have a docstring specifying parameters, return type, and failure modes. Keep actions atomic -- one gripper operation per function.

2. **Build the Task Planner Agent prompt.** Construct a system prompt that instructs the LLM to use ReAct-style reasoning: generate a `Thought:` (analyze current state and what to do next), then an `Action:` (emit exactly one subtask), then wait for an `Observation:` (feedback from execution). Include subtask templates: "Get the position of [object]", "Pick up [object] and place it [relation] [destination]", "Get the names of objects in the environment". Instruct the planner to generate only one subtask at a time and focus on one object per step.

3. **Build the Task Executor Agent prompt for CaP mode.** Provide the executor with all action function signatures, a one-shot code example for a pick-and-place task, and instructions to generate a self-contained Python script that calls the API functions to fulfill the subtask. The script should include error handling (try/except around each action call) and return a structured result dict.

4. **Build the Task Executor Agent prompt for TaP mode.** Instead of code generation, configure the executor to use LLM tool-calling. Register each robot action as a tool with its JSON schema. Instruct the executor to emit one tool call per step, observe the return value, and continue until the subtask is complete. Include best-practice templates for common subtask patterns.

5. **Implement the observation feedback channel.** After the executor completes a subtask (or fails), generate a natural language observation summarizing the result: e.g., "Successfully picked the lemon from position (0.3, 0.1, 0.05)" or "Failed: object 'orange' not found in environment." Pass this observation back to the planner's conversation history so it can reason about next steps.

6. **Implement the outer ReAct loop with termination conditions.** The planner loop runs until: (a) the planner emits a "Task Complete" signal, (b) the step count exceeds a configurable maximum (default: 15), or (c) three consecutive failures trigger an abort. Log every thought-action-observation triple for debugging.

7. **Wire up the API server layer.** Expose robot actions through a RESTful API (e.g., FastAPI endpoints like `/pick`, `/place`, `/get_objects`) that the generated code or tool calls invoke. This decouples the LLM agents from the robot middleware (ROS/MoveIt/simulator) and makes the system testable without hardware.

8. **Add linguistic variation handling.** When parsing user instructions, account for four paraphrase categories: lexical (synonyms like "grab" for "pick"), syntactical (reordered clauses), semantic (informal phrasing like "toss it over there"), and high-level reasoning (implicit instructions like "clean up" meaning "put everything in the bin"). The planner prompt should include examples of these variations.

9. **Build evaluation harness.** Structure test tasks as `environment x task x instruction_variant` triples. For each environment (e.g., kitchen, warehouse, tabletop), define canonical tasks with ground-truth action sequences, then generate 4-6 linguistic paraphrases per task. Score by comparing executed actions against ground truth, checking parameter format validity and task completion.

10. **Select execution mode based on model capability.** Use CaP for capable models (70B+ parameters or frontier APIs) where speed matters and tasks are well-defined. Use TaP for smaller models or tasks requiring error recovery. For production, default to TaP with a CaP fast-path for simple single-object tasks.

## Concrete Examples

**Example 1: Building a CaP executor for a pick-and-place robot**

User: "I have a WidowX 250s robot arm with ROS. I want an LLM agent that takes natural language commands like 'put the red block on the blue plate' and generates Python code to control the arm."

Approach:
1. Define the action API module with typed functions wrapping the REST endpoints
2. Build the planner and CaP executor system prompts
3. Wire the ReAct loop connecting planner to executor to API

Output (action API):
```python
# robot_actions.py
from dataclasses import dataclass
from typing import Optional
import requests

API_BASE = "http://localhost:8000"

@dataclass
class Pose:
    x: float; y: float; z: float
    roll: float = 0.0; pitch: float = 0.0; yaw: float = 0.0

def get_objects() -> list[str]:
    """Return names of all objects detected in the workspace."""
    return requests.get(f"{API_BASE}/get_objects").json()["objects"]

def get_pose(object_name: str) -> Optional[Pose]:
    """Return the 6-DOF pose of the named object, or None if not found."""
    resp = requests.get(f"{API_BASE}/get_pose", params={"name": object_name})
    return Pose(**resp.json()) if resp.status_code == 200 else None

def pick(object_name: str) -> dict:
    """Close gripper on the named object. Returns {"success": bool, "message": str}."""
    return requests.post(f"{API_BASE}/pick", json={"name": object_name}).json()

def place(object_name: str, target_pose: Pose) -> dict:
    """Place the held object at target_pose. Returns {"success": bool, "message": str}."""
    return requests.post(f"{API_BASE}/place", json={"name": object_name, "pose": vars(target_pose)}).json()

def move_to_home_pos() -> dict:
    """Return the arm to its home position."""
    return requests.post(f"{API_BASE}/move_to_home_pos").json()
```

Output (CaP executor generating code for a subtask):
```python
# Generated by CaP executor for subtask: "Pick up red block and place it on blue plate"
from robot_actions import get_pose, pick, place

blue_plate_pose = get_pose("blue plate")
assert blue_plate_pose is not None, "blue plate not found"

result = pick("red block")
assert result["success"], f"Pick failed: {result['message']}"

blue_plate_pose.z += 0.05  # offset above surface
result = place("red block", blue_plate_pose)
assert result["success"], f"Place failed: {result['message']}"
```

**Example 2: TaP executor with iterative tool calls and error recovery**

User: "I want the robot agent to recover from individual failures instead of failing the whole task."

Approach:
1. Register each robot action as an LLM tool with JSON schema
2. Configure the executor to emit one tool call per turn, observe, then continue
3. On failure, the executor retries or reports back to the planner for replanning

Output (TaP conversation trace):
```
System: You are a robot task executor. Call one tool at a time. Observe
        the result before calling the next tool.
Subtask: Pick up red block and place it on blue plate