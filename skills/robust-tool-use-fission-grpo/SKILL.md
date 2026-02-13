---
name: "robust-tool-use-fission-grpo"
description: |
  Build fault-tolerant multi-turn tool-use agents that recover from execution errors instead of
  degenerating into repetitive invalid retries. Implements the Fission-GRPO pattern: when a tool
  call fails, the agent captures diagnostic feedback, fissions the failed trajectory into a
  corrective training instance, and resamples recovery actions on-policy.

  Trigger phrases:
  - "Make my agent recover from tool errors"
  - "Build a robust tool-use pipeline with error recovery"
  - "Implement Fission-GRPO for tool calling"
  - "Add self-correction to my LLM agent's tool calls"
  - "Handle tool execution failures gracefully in my agent"
  - "Train an agent to learn from its own tool-call mistakes"
---

# Robust Tool Use via Fission-GRPO

This skill enables Claude to build LLM-based agents and pipelines that **recover intelligently from tool execution errors** rather than blindly retrying the same failed call. The core idea comes from *Fission-GRPO* (Zhang et al., 2026): when a tool call fails, instead of discarding the trajectory or assigning a sparse negative reward, the system *fissions* the failed trajectory into a new training instance by appending diagnostic error feedback and then resampling corrective actions from the model's own policy. This converts every failure into a learning opportunity that is distribution-matched to the model's actual error modes.

## When to Use

- When building a **multi-turn agent** that calls external APIs, databases, or shell commands and needs to handle errors without human intervention.
- When an existing agent **loops on the same invalid tool call** after receiving an error, instead of adapting its parameters or switching strategy.
- When designing a **training pipeline** (SFT or RL) for a tool-calling model and you want to incorporate error-recovery supervision.
- When implementing a **retry/fallback harness** around LLM tool calls that should learn from past failures within the same session.
- When you need to build an **Error Simulator** that generates realistic diagnostic feedback for tool-call failures without executing real APIs.
- When evaluating agent robustness on benchmarks like **BFCL v4 Multi-Turn** and want to improve error recovery rates.

## Key Technique

**The Problem.** Standard RL (e.g., GRPO) treats tool-call errors as sparse negative rewards: the model learns *what not to do* but gets no signal about *how to recover*. Pre-collected error-correction datasets help, but suffer from distribution mismatch -- the synthetic errors don't match the specific mistakes the model actually makes during exploration. Smaller models (7-8B) are especially prone to degenerate looping: after a failed `get_weather(city="NYC")`, they re-emit the identical call dozens of times rather than trying `get_weather(location="New York City")`.

**The Solution: Fission-GRPO.** The framework operates in three stages per training batch. *Stage 1 (Explore)*: Sample G trajectories per query using the current policy and compute composite rewards (format compliance, functional correctness, efficiency). Apply standard GRPO updates. *Stage 2 (Fission)*: Filter trajectories where the correctness score falls below a threshold or format validation fails. For each failed trajectory, generate diagnostic feedback using a fine-tuned Error Simulator (a separate model trained on ~2K error-log instances that produces concise error strings *without revealing the correct answer*). Store the augmented context (original trajectory + error feedback) in a LIFO buffer. *Stage 3 (Recover)*: For each buffered error context, sample G' fresh recovery rollouts from the current policy and optimize them with the same GRPO objective. This creates a corrective training batch that is always on-policy and matched to the model's actual failure distribution.

**Why It Works.** The fission mechanism converts every failed trajectory into corrective supervision *within* the RL loop, so the model practices recovering from its own mistakes rather than from hypothetical ones. The Error Simulator prevents target leakage by never exposing the ground-truth tool call in its feedback -- it only describes *what went wrong* (e.g., "Parameter 'date' expects ISO-8601 format, received Unix timestamp"), forcing the model to reason about the fix. On BFCL v4 Multi-Turn, this yields a 5.7% absolute improvement in error recovery rate and a 4% overall accuracy gain over vanilla GRPO.

## Step-by-Step Workflow

1. **Define the tool schema and error taxonomy.** Enumerate every tool the agent can call, its parameter types, and the categories of errors it can produce: format errors (wrong types, missing required params), semantic errors (valid syntax but incorrect intent), and runtime errors (timeouts, rate limits, auth failures).

2. **Build the Error Simulator.** Fine-tune a model (or write deterministic rules for format errors) that takes as input: (a) the tool schema, (b) the conversation history, (c) the ground-truth expected call, and (d) the actual failed call -- and outputs a concise diagnostic string. The diagnostic must describe the failure mode *without* revealing the correct answer. For format errors, use deterministic messages like `"TypeError: 'date' expects string in YYYY-MM-DD format, got integer 1706745600"`. For semantic errors, use the simulator to produce messages like `"No results: 'search_flights' requires airport IATA codes, not city names"`.

3. **Implement the trajectory collector with error detection.** Wrap the agent's tool-call loop so that after each execution, the response is classified as: success (proceed), format error (deterministic feedback), or semantic error (invoke Error Simulator). Record the full trajectory including the error feedback.

4. **Implement the fission mechanism.** When a trajectory fails, create a new training instance by concatenating: [original prompt + conversation history + failed tool call + diagnostic feedback]. This becomes the prompt for recovery rollout sampling. Store these fissioned instances in a LIFO buffer (most recent errors are most relevant to current policy).

5. **Sample recovery rollouts on-policy.** For each fissioned instance in the buffer, sample G' (typically 8) new continuations from the *current* policy. These recovery attempts are the model's fresh tries at fixing its own mistake, given the error feedback.

6. **Compute composite rewards for all trajectories.** Score each trajectory (both original and recovery) using three components: (a) **Format compliance** R_fmt in {0, 1} -- does the output parse as a valid tool call? (b) **Functional correctness** R_corr in [0, 2] -- does the function name match (1.0) plus F1 overlap of parameters (up to 1.0)? (c) **Efficiency** R_len in [0, 1] -- penalize unnecessarily long or short responses via a piecewise Gaussian centered on reference length.

7. **Apply GRPO with group-normalized advantages.** Compute advantages within each group of G (or G') trajectories relative to each other. This means recovery trajectories compete against other recovery attempts for the same error, not against originally-successful trajectories. Update the policy using the clipped surrogate objective with KL regularization.

8. **Anneal reward weights over training.** Start with high weight on format compliance (to learn valid syntax first), then gradually shift weight toward functional correctness and efficiency. This curriculum prevents the model from gaming format rewards while producing incorrect calls.

9. **Deploy with runtime error-recovery loop.** At inference time, implement a retry harness: if a tool call fails, append the actual error message to the conversation and let the model generate a corrective call. Limit retries to 2-3 attempts to prevent infinite loops. The Fission-GRPO-trained model will have learned to interpret error feedback and adjust its calls accordingly.

10. **Evaluate on held-out error scenarios.** Measure: (a) overall task accuracy, (b) error recovery rate (fraction of initially-failed tasks that succeed after recovery), and (c) degenerate loop rate (fraction of errors that produce identical re-invocations). Target: recovery rate >60%, loop rate <10%.

## Concrete Examples

**Example 1: Building a retry harness for an API-calling agent**

User: "My LangChain agent calls a weather API but keeps retrying the same malformed request when it gets a 400 error. Help me fix this."

Approach:
1. Inspect the agent's tool-calling code and identify that errors are caught but the error message is not fed back to the LLM.
2. Modify the tool executor to return structured error feedback to the model's conversation context.
3. Implement a fission-style retry loop that appends diagnostic information and asks the model to generate a corrected call.

```python
# retry_harness.py — Fission-style error recovery for tool-calling agents

from typing import Callable, Any
import json

MAX_RETRIES = 3

def execute_with_recovery(
    llm: Callable,
    tool_fn: Callable,
    tool_schema: dict,
    messages: list[dict],
    max_retries: int = MAX_RETRIES,
) -> dict:
    """Execute a tool call with Fission-GRPO-inspired error recovery.

    Instead of blindly retrying, appends diagnostic feedback from the
    execution error and asks the LLM to generate a corrective call.
    """
    for attempt in range(max_retries + 1):
        # Ask LLM to produce a tool call
        response = llm(messages)
        tool_call = parse_tool_call(response)

        if tool_call is None:
            # Format error: the LLM didn't produce a valid tool call
            messages.append({
                "role": "system",
                "content": (
                    f"Your response could not be parsed as a valid tool call. "
                    f"Expected format: {json.dumps(tool_schema['example'])}. "
                    f"Please try again with correct JSON syntax."
                ),
            })
            continue

        try:
            result = tool_fn(**tool_call["parameters"])
            return {"success": True, "result": result, "attempts": attempt + 1}
        except Exception as e:
            # Fission: append diagnostic feedback, NOT the raw traceback
            diagnostic = build_diagnostic(tool_call, e, tool_schema)
            messages.append({
                "role": "tool",
                "content": f"Error: {diagnostic}",
                "tool_call_id": tool_call.get("id", ""),
            })

    return {"success": False, "error": "Max retries exceeded", "attempts": max_retries + 1}


def build_diagnostic(tool_call: dict, error: Exception, schema: dict) -> str:
    """Generate concise diagnostic feedback without revealing the answer."""
    error_type = type(error).__name__
    params = tool_call.get("parameters", {})
    expected_types = schema.get("parameters", {})

    diagnostics = []
    for key, value in params.items():
        if key in expected_types:
            expected = expected_types[key].get("type", "unknown")
            actual = type(value).__name__
            if expected == "string" and not isinstance(value, str):
                diagnostics.append(
                    f"Parameter '{key}' expects {expected}, got {actual}: {value!r}"
                )
            enum_vals = expected_types[key].get("enum")
            if enum_vals and value not in enum_vals:
                diagnostics.append(
                    f"Parameter '{key}' must be one of {enum_vals}, got {value!r}"
                )

    if not diagnostics:
        diagnostics.append(f"{error_type}: {str(error)[:200]}")

    return "; ".join(diagnostics)
```

**Example 2: Training an error-recovery-aware model with GRPO**

User: "I'm fine-tuning a Qwen3-8B model for tool calling with GRPO. How do I add error recovery training using the Fission approach?"

Approach:
1. Modify the GRPO training loop to detect failed trajectories.
2. Build an Error Simulator (or use rule-based feedback for format errors).
3. Fission failed trajectories and resample recovery rollouts.

```python
# fission_grpo_training.py — Pseudocode for the Fission-GRPO training loop

def fission_grpo_step(policy, queries, tools, error_simulator, config):
    """One training step of Fission-GRPO."""
    G = config.num_rollouts          # e.g., 8
    G_prime = config.num_fission     # e.g., 8
    delta_corr = config.corr_threshold  # e.g., 1.0
    fission_buffer = []

    # === Stage 1: Standard Exploration ===
    all_trajectories = []
    for query in queries:
        trajectories = [policy.sample(query, tools) for _ in range(G)]
        for traj in trajectories:
            traj.reward = compute_reward(traj, config)
        all_trajectories.extend(trajectories)

    # Standard GRPO update on exploration batch
    grpo_update(policy, all_trajectories, config)

    # === Stage 2: Error Identification & Fission ===
    for traj in all_trajectories:
        if traj.reward.r_corr < delta_corr or traj.reward.r_fmt == 0:
            # Generate diagnostic feedback
            if traj.reward.r_fmt == 0:
                feedback = deterministic_format_error(traj.last_call, tools)
            else:
                feedback = error_simulator.generate(
                    system_prompt=traj.system,
                    history=traj.messages,
                    ground_truth=traj.expected_call,
                    failed_call=traj.last_call,
                )
            # Create fissioned instance
            fissioned = traj.messages + [
                {"role": "tool", "content": f"Error: {feedback}"}
            ]
            fission_buffer.append(fissioned)

    # === Stage 3: Corrective Recovery ===
    recovery_trajectories = []
    for fissioned_context in fission_buffer:
        recoveries = [
            policy.sample_continuation(fissioned_context, tools)
            for _ in range(G_prime)
        ]
        for rec in recoveries:
            rec.reward = compute_reward(rec, config)
        recovery_trajectories.extend(recoveries)

    # GRPO update on recovery batch (advantages computed within each group)
    grpo_update(policy, recovery_trajectories, config)


def compute_reward(traj, config):
    """Composite reward: format + correctness + efficiency."""
    r_fmt = 1.0 if is_valid_tool_call(traj.last_call) else 0.0
    r_corr = (
        function_name_match(traj.last_call, traj.expected) * 1.0
        + param_f1_score(traj.last_call, traj.expected) * 1.0
    )
    r_len = efficiency_score(traj, config.ref_length)
    w = config.get_weights(config.current_step)  # annealed weights
    return w.fmt * r_fmt + w.corr * r_corr + w.len * r_len
```

**Example 3: Adding an Error Simulator to an existing agent framework**

User: "I need to test my agent against realistic API errors without hitting real endpoints."

Approach:
1. Catalog the error modes of each tool (type mismatches, missing params, invalid values, rate limits).
2. Build a simulator that produces diagnostic strings matching real error patterns.
3. Integrate it as a mock execution layer for testing and training.

```python
# error_simulator.py — Rule-based Error Simulator for common tool failures

ERROR_TEMPLATES = {
    "type_mismatch": "Parameter '{param}' expects {expected_type}, received {actual_type}: {value!r}",
    "missing_required": "Missing required parameter: '{param}'. Required parameters: {required}",
    "invalid_enum": "Invalid value for '{param}': {value!r}. Must be one of: {allowed}",
    "rate_limit": "Rate limit exceeded. Retry after {retry_after}s. Consider batching requests.",
    "not_found": "Resource not found: {resource}. Verify the identifier and try again.",
    "auth_failure": "Authentication failed. Token may be expired or lack scope '{scope}'.",
}

def simulate_error(tool_schema: dict, tool_call: dict) -> str | None:
    """Return a diagnostic error string if the call would fail, else None."""
    params = tool_call.get("parameters", {})
    schema_params = tool_schema.get("parameters", {})

    # Check missing required params
    required = [k for k, v in schema_params.items() if v.get("required")]
    for r in required:
        if r not in params:
            return ERROR_TEMPLATES["missing_required"].format(
                param=r, required=required
            )

    # Check type mismatches
    for key, value in params.items():
        if key not in schema_params:
            continue
        expected = schema_params[key]["type"]
        actual = type(value).__name__
        type_map = {"string": str, "integer": int, "number": (int, float), "boolean": bool}
        if expected in type_map and not isinstance(value, type_map[expected]):
            return ERROR_TEMPLATES["type_mismatch"].format(
                param=key, expected_type=expected, actual_type=actual, value=value
            )

        # Check enum constraints
        if "enum" in schema_params[key] and value not in schema_params[key]["enum"]:
            return ERROR_TEMPLATES["invalid_enum"].format(
                param=key, value=value, allowed=schema_params[key]["enum"]
            )

    return None  # Call appears valid
```

## Best Practices

**Do:**
- Always feed the **actual error message** back into the LLM's conversation context. The model cannot self-correct if it never sees what went wrong.
- Make diagnostics **specific but not answer-revealing**. Say "Parameter 'date' expects ISO-8601 format" not "The correct call is `get_events(date='2026-01-22')`". This forces the model to reason.
- Use a **LIFO buffer** for fissioned instances so the model trains on its most recent (most policy-relevant) errors first.
- **Anneal reward weights** from format-heavy to correctness-heavy. Early training should nail valid syntax; later training optimizes for semantic correctness.
- Set a **hard retry limit** (2-3 attempts) at inference time. Fission-GRPO-trained models recover quickly; unlimited retries signal a deeper problem.
- **Separate format errors from semantic errors** in your error taxonomy. Format errors get deterministic feedback; semantic errors need the Error Simulator.

**Avoid:**
- Do not include the ground-truth tool call in error feedback -- this causes target leakage and the model learns to parrot rather than reason.
- Do not treat all errors as equivalent negative reward. A call with the right function name but wrong parameter format is much closer to correct than a completely wrong function.
- Do not mix recovery trajectories with fresh-exploration trajectories in the same GRPO advantage group. Recovery attempts should compete against other recovery attempts for the same error.
- Do not use a static error dataset for training. The whole point of fission is that errors are generated on-policy, matching the model's current failure distribution.

## Error Handling

| Failure Mode | Symptom | Resolution |
|---|---|---|
| **Degenerate looping** | Model emits identical call 3+ times after error | Check that error feedback is actually appended to context; increase diagnostic specificity |
| **Target leakage** | Model copies answer from error message | Audit Error Simulator outputs; ensure they describe the *problem*, not the *solution* |
| **Format reward hacking** | Model produces valid JSON that calls the wrong function | Decrease format reward weight; increase correctness weight earlier in training |
| **Recovery over-confidence** | Model "fixes" correct calls unnecessarily | Only trigger fission when correctness score is below threshold; don't fission partial successes |
| **Buffer staleness** | Old fissioned instances no longer match current policy | Use LIFO ordering and cap buffer size (e.g., 2x batch size); flush after each epoch |

## Limitations

- **Requires ground-truth tool calls** for the Error Simulator training data and for computing correctness rewards. Purely open-ended tool use without reference solutions cannot use the full framework.
- **Error Simulator quality is critical.** If the simulator produces vague or misleading feedback, recovery rollouts will not improve. Budget effort for simulator fine-tuning or comprehensive rule coverage.
- **Computational overhead.** Fission doubles the number of rollouts per training step (G exploration + G' recovery per failed trajectory). On the paper's setup, this requires 8x H800 GPUs.
- **Single-turn recovery only.** The current framework fissions at the point of failure and resamples one corrective step. Multi-hop recovery chains (error -> partial fix -> second error -> final fix) are not explicitly modeled.
- **Benchmark-specific tuning.** The reward function (function name match + parameter F1) is designed for structured tool-call benchmarks like BFCL. Adapting to free-form tool use (e.g., code execution) requires redesigning the reward signal.

## Reference

Zhang, Z., Zhao, F., Wang, R., Wang, Z., & Liang, B. (2026). *Robust Tool Use via Fission-GRPO: Learning to Recover from Execution Errors.* arXiv:2601.15625v1. [https://arxiv.org/abs/2601.15625v1](https://arxiv.org/abs/2601.15625v1)

Look for: Section 3 (the three-stage algorithm), Figure 2 (fission mechanism diagram), Table 1 (BFCL v4 results), and the Error Simulator training procedure in Section 3.2.