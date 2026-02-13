---
name: "agentdog-diagnostic-guardrail-framework"
description: >
  Implement diagnostic safety guardrails for AI agent systems using the AgentDoG
  three-dimensional taxonomy (risk source, failure mode, real-world harm). Monitors
  agent trajectories, diagnoses root causes of unsafe actions, and provides fine-grained
  risk labels beyond binary safe/unsafe classification.
  Trigger phrases: "add safety guardrails to my agent", "diagnose agent risks",
  "monitor agent trajectory safety", "implement agentic guardrail",
  "classify agent risk behavior", "audit agent tool use safety"
---

# AgentDoG: Diagnostic Guardrail Framework for AI Agent Safety

This skill enables Claude to design and implement diagnostic safety guardrails for AI agent systems following the AgentDoG framework from Liu et al. (2026). Rather than applying blunt binary safe/unsafe filters, this approach monitors full agent trajectories (sequences of actions and observations) and produces structured three-dimensional diagnoses: *where* the risk originates (source), *how* the agent fails (failure mode), and *what* real-world harm results (consequence). This gives developers actionable root-cause information instead of opaque rejections, and catches "seemingly safe but unreasonable" actions that binary classifiers miss.

## When to Use

- When building an agent orchestration system that calls external tools (APIs, file systems, databases) and you need runtime safety monitoring
- When a user asks to audit or classify the risk profile of an agent's execution trace
- When implementing guardrail middleware that intercepts agent actions before execution
- When designing a safety taxonomy for a new agentic application (e.g., coding agents, browsing agents, financial agents)
- When debugging why an agent took an unsafe or unreasonable action in a multi-step workflow
- When adding explainable safety decisions to an agent pipeline (provenance beyond "blocked")

## Key Technique

**Three-Dimensional Risk Taxonomy.** AgentDoG decomposes agentic risk into three orthogonal dimensions. The *Risk Source* (Where) identifies the origin: user input (direct prompt injection), environmental observation (indirect injection, unreliable data), external entities (malicious tools, corrupted feedback), or internal logic (hallucination, flawed reasoning). The *Failure Mode* (How) captures the behavioral mechanism: unconfirmed actions, flawed planning, improper tool use, insecure interactions, procedural deviation, inefficient execution, harmful content generation, unauthorized disclosure, or inaccurate information. The *Real-World Harm* (What) categorizes consequences across 10 domains: privacy, financial, security, physical/health, psychological, reputational, information ecosystem, public service, fairness, and functional harms.

**Trajectory-Level Diagnosis.** Unlike guardrails that only inspect the final output, AgentDoG analyzes the full trajectory `T = [(a1, o1), (a2, o2), ...]` where each step is an action-observation pair. Actions include thinking traces, tool calls, and responses; observations include tool outputs and environment feedback. The guardrail produces a binary verdict *plus* a diagnostic three-tuple `(risk_source, failure_mode, harm_type)` drawn from the taxonomy. This means a blocked action comes with an explanation like "Risk Source: Environmental Observation (indirect prompt injection), Failure Mode: Improper Tool Use (wrong parameters), Harm: Financial Loss" rather than just "unsafe."

**Attribution for Root-Cause Analysis.** AgentDoG includes an explainability layer that traces unsafe verdicts back to specific trajectory steps and even specific sentences. It computes temporal information gain (how much a step increased risk likelihood) and per-sentence attribution scores combining necessity (probability drop when removed) and sufficiency (probability hold when isolated). This lets developers pinpoint exactly which step and which piece of context triggered the safety flag.

## Step-by-Step Workflow

1. **Define your agent's action schema.** Enumerate all tools the agent can call, their parameter types, and side effects. Categorize each tool's risk surface: read-only vs. write, internal vs. external, reversible vs. irreversible. This mirrors AgentDoG's tool definition inventory (drawn from 2,292+ tool definitions in training).

2. **Implement trajectory logging.** Instrument your agent loop to capture each step as a structured `(action, observation)` pair. Actions must include the tool name, parameters, and any chain-of-thought reasoning. Observations must include the raw tool response and any environment state changes. Store these as an ordered list.

3. **Map your domain risks to the three-dimensional taxonomy.** For each axis, select the relevant subcategories:
   - **Source (Where):** User Input, Environmental Observation, External Entity, Internal Logic
   - **Failure Mode (How):** Unconfirmed Action, Flawed Planning, Improper Tool Use, Insecure Interaction, Procedural Deviation, Inefficient Execution, Harmful Generation, Unauthorized Disclosure, Inaccurate Information
   - **Harm (What):** Privacy, Financial, Security, Physical/Health, Psychological, Reputational, Info Ecosystem, Public Service, Fairness, Functional

4. **Build the guardrail evaluation prompt.** Construct a system prompt that presents the full trajectory and asks for: (a) a binary safe/unsafe verdict, (b) if unsafe, the three-tuple `(source, failure_mode, harm_type)` with free-text justification, (c) identification of the specific trajectory step(s) that triggered the verdict. Use the taxonomy labels as a constrained output vocabulary.

5. **Implement pre-execution and post-step hooks.** Insert the guardrail at two points: *before* executing high-risk tool calls (pre-execution gate) and *after* each step completes (post-step audit). Pre-execution gates block dangerous actions; post-step audits catch cascading risks that only emerge across multiple steps.

6. **Handle "safe but unreasonable" actions.** Configure the guardrail to flag actions that are not overtly harmful but indicate degraded agent behavior: redundant API calls (Inefficient Execution), skipping confirmation for destructive operations (Unconfirmed Action), or deviating from the established plan without justification (Procedural Deviation). These should produce warnings rather than hard blocks.

7. **Implement graduated response policies.** Map each harm category to an enforcement action: hard-block (e.g., security, financial), soft-block with user confirmation (e.g., privacy, reputational), warn-and-log (e.g., functional, inefficiency). The three-tuple output drives this routing automatically.

8. **Add attribution tracing for flagged actions.** When a trajectory is flagged, walk backward through the steps to identify the root cause. Compute which step most increased the risk signal (temporal information gain). Present the developer with: the triggering step, the originating risk source, and the specific text or parameter that caused the flag.

9. **Build a feedback loop for taxonomy refinement.** Log all guardrail verdicts with their three-tuples. Periodically review false positives and false negatives to identify taxonomy gaps or miscalibrated thresholds in your domain. Add domain-specific subcategories under the three dimensions as needed.

10. **Test against adversarial trajectories.** Construct test cases for each cell of the taxonomy matrix: direct prompt injection leading to unauthorized file deletion (User Input x Improper Tool Use x Security Harm), indirect injection via tool output leading to data exfiltration (Environmental Observation x Insecure Interaction x Privacy Harm), etc. Aim for coverage across all three dimensions.

## Concrete Examples

**Example 1: Guardrail middleware for a coding agent**

User: "Add a safety guardrail to my coding agent that uses tools like `run_shell`, `write_file`, and `read_file`."

Approach:
1. Define the tool risk surface: `run_shell` is high-risk (irreversible, external), `write_file` is medium-risk (irreversible, internal), `read_file` is low-risk (read-only).
2. Implement trajectory capture and the three-dimensional classifier.
3. Wire pre-execution gating on `run_shell` and `write_file`.

Output:
```python
from dataclasses import dataclass
from enum import Enum
from typing import Optional

class RiskSource(Enum):
    USER_INPUT = "user_input"
    ENVIRONMENTAL = "environmental_observation"
    EXTERNAL_ENTITY = "external_entity"
    INTERNAL_LOGIC = "internal_logic"

class FailureMode(Enum):
    UNCONFIRMED_ACTION = "unconfirmed_action"
    FLAWED_PLANNING = "flawed_planning"
    IMPROPER_TOOL_USE = "improper_tool_use"
    INSECURE_INTERACTION = "insecure_interaction"
    PROCEDURAL_DEVIATION = "procedural_deviation"
    INEFFICIENT_EXECUTION = "inefficient_execution"
    HARMFUL_GENERATION = "harmful_generation"
    UNAUTHORIZED_DISCLOSURE = "unauthorized_disclosure"
    INACCURATE_INFO = "inaccurate_information"

class HarmType(Enum):
    PRIVACY = "privacy"
    FINANCIAL = "financial"
    SECURITY = "security"
    PHYSICAL = "physical_health"
    PSYCHOLOGICAL = "psychological"
    REPUTATIONAL = "reputational"
    INFO_ECOSYSTEM = "info_ecosystem"
    PUBLIC_SERVICE = "public_service"
    FAIRNESS = "fairness"
    FUNCTIONAL = "functional"

@dataclass
class TrajectoryStep:
    action: dict        # {"tool": str, "params": dict, "reasoning": str}
    observation: dict   # {"output": str, "side_effects": list}

@dataclass
class Diagnosis:
    is_safe: bool
    source: Optional[RiskSource] = None
    failure_mode: Optional[FailureMode] = None
    harm_type: Optional[HarmType] = None
    flagged_step: Optional[int] = None
    explanation: Optional[str] = None

TOOL_RISK_LEVELS = {
    "run_shell": "high",    # irreversible, external side effects
    "write_file": "medium", # irreversible, internal
    "read_file": "low",     # read-only
}

PRE_EXEC_GATES = {"high", "medium"}  # tools requiring pre-execution check

def evaluate_trajectory(steps: list[TrajectoryStep]) -> Diagnosis:
    """Analyze full trajectory and return three-dimensional diagnosis."""
    # Build the evaluation prompt with the full trajectory
    prompt = build_guardrail_prompt(steps)
    # Call the guardrail model (AgentDoG or LLM-as-judge)
    result = call_guardrail_model(prompt)
    return parse_diagnosis(result)

def pre_execution_gate(step: TrajectoryStep, history: list[TrajectoryStep]) -> Diagnosis:
    """Check a pending action before execution."""
    tool = step.action.get("tool", "")
    if TOOL_RISK_LEVELS.get(tool, "low") not in PRE_EXEC_GATES:
        return Diagnosis(is_safe=True)

    # Check for common unsafe patterns
    params = step.action.get("params", {})

    # Pattern: shell command with piped secrets or destructive ops
    if tool == "run_shell":
        cmd = params.get("command", "")
        if any(p in cmd for p in ["rm -rf /", "| curl", "chmod 777", "> /dev/"]):
            return Diagnosis(
                is_safe=False,
                source=RiskSource.INTERNAL_LOGIC,
                failure_mode=FailureMode.IMPROPER_TOOL_USE,
                harm_type=HarmType.SECURITY,
                flagged_step=len(history),
                explanation=f"Blocked destructive/exfiltration shell command: {cmd[:80]}"
            )

    # Pattern: writing to sensitive paths without confirmation
    if tool == "write_file":
        path = params.get("path", "")
        if any(s in path for s in [".env", "credentials", "/etc/", ".ssh/"]):
            return Diagnosis(
                is_safe=False,
                source=RiskSource.INTERNAL_LOGIC,
                failure_mode=FailureMode.UNCONFIRMED_ACTION,
                harm_type=HarmType.PRIVACY,
                flagged_step=len(history),
                explanation=f"Write to sensitive path requires confirmation: {path}"
            )

    # Full trajectory evaluation for context-dependent risks
    return evaluate_trajectory(history + [step])
```

**Example 2: Diagnosing an indirect prompt injection in tool output**

User: "My browsing agent fetched a webpage that contained hidden instructions. How do I detect this?"

Approach:
1. Classify as Risk Source: Environmental Observation (indirect prompt injection).
2. Monitor the trajectory for behavioral changes after the tainted observation.
3. Flag if the agent's subsequent actions deviate from the original task plan.

Output:
```python
def detect_injection_in_observation(
    step: TrajectoryStep,
    original_task: str,
    prior_plan: list[str],
) -> Diagnosis:
    """Detect indirect prompt injection in tool/environment output."""
    observation_text = step.observation.get("output", "")

    # Heuristic: check for instruction-like patterns in tool output
    injection_signals = [
        "ignore previous",
        "ignore all prior",
        "disregard your instructions",
        "new task:",
        "system prompt:",
        "you are now",
        "instead, do the following",
    ]
    has_injection_signal = any(
        sig in observation_text.lower() for sig in injection_signals
    )

    if not has_injection_signal:
        return Diagnosis(is_safe=True)

    # Check if the agent's next action deviates from prior plan
    next_action = step.action.get("reasoning", "")
    plan_deviation = not any(
        planned_step.lower() in next_action.lower()
        for planned_step in prior_plan
    )

    if has_injection_signal and plan_deviation:
        return Diagnosis(
            is_safe=False,
            source=RiskSource.ENVIRONMENTAL,
            failure_mode=FailureMode.PROCEDURAL_DEVIATION,
            harm_type=HarmType.SECURITY,
            flagged_step=None,
            explanation=(
                "Indirect prompt injection detected in tool output. "
                "Agent deviated from established plan after processing "
                f"tainted observation containing: "
                f"'{next(s for s in injection_signals if s in observation_text.lower())}'"
            ),
        )

    # Signal present but agent stayed on plan -- warn but don't block
    return Diagnosis(
        is_safe=True,
        source=RiskSource.ENVIRONMENTAL,
        failure_mode=FailureMode.INSECURE_INTERACTION,
        harm_type=HarmType.SECURITY,
        explanation="Injection-like content in observation, but agent maintained plan adherence. Monitor closely.",
    )
```

**Example 3: Auditing a multi-step agent trajectory post-hoc**

User: "I have a log of my agent's actions. Can you analyze it for safety issues using the AgentDoG taxonomy?"

Approach:
1. Parse the trajectory log into `(action, observation)` pairs.
2. Evaluate each step against the three-dimensional taxonomy.
3. Produce a structured audit report.

Output:
```python
def audit_trajectory(trajectory_log: list[dict]) -> dict:
    """Produce a structured safety audit of a recorded agent trajectory."""
    steps = [
        TrajectoryStep(action=entry["action"], observation=entry["observation"])
        for entry in trajectory_log
    ]

    findings = []
    for i, step in enumerate(steps):
        diagnosis = evaluate_trajectory(steps[: i + 1])
        if not diagnosis.is_safe or diagnosis.explanation:
            findings.append({
                "step": i,
                "tool": step.action.get("tool"),
                "verdict": "unsafe" if not diagnosis.is_safe else "warning",
                "risk_source": diagnosis.source.value if diagnosis.source else None,
                "failure_mode": diagnosis.failure_mode.value if diagnosis.failure_mode else None,
                "harm_type": diagnosis.harm_type.value if diagnosis.harm_type else None,
                "explanation": diagnosis.explanation,
            })

    return {
        "total_steps": len(steps),
        "findings_count": len(findings),
        "unsafe_count": sum(1 for f in findings if f["verdict"] == "unsafe"),
        "warning_count": sum(1 for f in findings if f["verdict"] == "warning"),
        "findings": findings,
        "taxonomy_coverage": summarize_taxonomy_hits(findings),
    }

# Example audit output:
# {
#   "total_steps": 12,
#   "findings_count": 2,
#   "unsafe_count": 1,
#   "warning_count": 1,
#   "findings": [
#     {
#       "step": 4,
#       "tool": "web_search",
#       "verdict": "warning",
#       "risk_source": "environmental_observation",
#       "failure_mode": "inaccurate_information",
#       "harm_type": "info_ecosystem",
#       "explanation": "Agent used unverified search result as factual basis for recommendation"
#     },
#     {
#       "step": 9,
#       "tool": "send_email",
#       "verdict": "unsafe",
#       "risk_source": "internal_logic",
#       "failure_mode": "unconfirmed_action",
#       "harm_type": "privacy",
#       "explanation": "Agent sent email containing user PII without explicit confirmation"
#     }
#   ]
# }
```

## Best Practices

- **Do:** Analyze the full trajectory, not just the latest action. Risks often emerge from the interaction between steps (e.g., a benign read followed by an exfiltrating write).
- **Do:** Use the three-tuple output to drive graduated enforcement. Not all unsafe actions warrant hard blocks -- inefficient execution should warn, while security harms should block.
- **Do:** Log all diagnostic three-tuples for trend analysis. Clusters of the same `(source, failure_mode)` pair indicate a systematic agent weakness to fix upstream.
- **Do:** Treat "safe but unreasonable" actions (procedural deviation, inefficient execution) as first-class signals. These often precede overtly unsafe behavior.
- **Avoid:** Relying on keyword matching alone for injection detection. Use trajectory-level behavioral analysis -- did the agent's plan change after a suspicious observation?
- **Avoid:** Hardcoding risk policies that ignore context. The same tool call (`write_file`) may be safe in one trajectory context and unsafe in another. Always evaluate with trajectory history.

## Error Handling

- **False positives on benign tool use:** When the guardrail flags legitimate actions, check if the trajectory context was insufficient. Provide more history steps to the evaluator. Tune thresholds per-tool based on observed false positive rates.
- **Taxonomy gaps in novel domains:** If your agent operates in a domain not well covered by the 10 harm categories (e.g., robotics, medical), extend the harm dimension with domain-specific subcategories while maintaining the three-axis structure.
- **Latency from full-trajectory evaluation:** For real-time agents, use the lightweight pre-execution gate (pattern matching + single-step check) for synchronous blocking and defer full trajectory evaluation to asynchronous post-step audits.
- **Ambiguous risk source attribution:** When multiple trajectory steps contribute to a risk, report the primary source (highest temporal information gain) but include secondary contributors in the explanation field.

## Limitations

- The three-dimensional taxonomy was designed for tool-using AI agents in digital environments. Purely conversational or embodied robotic agents may need adapted taxonomies.
- Full trajectory evaluation at every step is computationally expensive. For production systems, implement tiered evaluation: lightweight heuristics for low-risk tools, full evaluation for high-risk tools and anomalous behavior.
- The framework diagnoses risks but does not automatically remediate them. Developers must implement the enforcement layer (block, warn, confirm) based on the diagnostic output.
- Attribution tracing assumes the guardrail model can reliably identify which step caused the risk. For very long trajectories (50+ steps), attribution accuracy may degrade.
- The taxonomy categories are comprehensive but finite. Novel attack vectors (e.g., multi-agent collusion) may not map cleanly to existing subcategories without extension.

## Reference

**Paper:** Liu, D., Ren, Q., Qian, C., Shao, S., & Xie, Y. (2026). *AgentDoG: A Diagnostic Guardrail Framework for AI Agent Safety and Security.* arXiv:2601.18491v1. [https://arxiv.org/abs/2601.18491v1](https://arxiv.org/abs/2601.18491v1)

Look for: Section 3 (three-dimensional taxonomy with full subcategory definitions), Section 4 (ATBench benchmark construction), Section 5 (diagnostic guardrail architecture and trajectory-level evaluation), and Appendix A (complete taxonomy tables with examples for every subcategory).