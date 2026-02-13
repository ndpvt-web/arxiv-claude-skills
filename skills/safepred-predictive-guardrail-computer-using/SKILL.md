---
name: "safepred-predictive-guardrail-computer-using"
description: "Implement predictive safety guardrails for computer-using agents and automated pipelines using world-model-based risk prediction. Catches long-term risks that reactive checks miss by simulating future states before executing actions. Use when: 'add safety guardrails to my agent', 'predict risks before executing shell commands', 'build a predictive safety layer', 'guardrail my automation pipeline', 'prevent irreversible actions in my agent workflow', 'add a world model safety check to my CUA'."
---

# SafePred: Predictive Guardrails for Computer-Using Agents

This skill teaches Claude to implement **predictive safety guardrails** based on the SafePred framework. Instead of only checking whether the *current* action is dangerous (reactive), SafePred uses a world model to predict short-term and long-term consequences of candidate actions, scores them against explicit safety policies, and blocks or replans when predicted futures violate those policies. This catches delayed-harm scenarios--like modifying a system Python instead of using a virtualenv, or hardcoding credentials that later leak via version control--that reactive guardrails fundamentally cannot detect.

## When to Use

- When building or extending a computer-using agent (browser automation, OS automation, CLI agent) that needs safety constraints beyond simple action blocklists
- When the user asks to "add guardrails" or "safety checks" to an agent pipeline that executes shell commands, file operations, or API calls
- When designing an automation system where some actions appear safe locally but cause irreversible damage downstream (e.g., deleting logs that break future audits)
- When implementing a multi-step agent workflow that needs both step-level intervention and task-level replanning on risk detection
- When the user wants to prevent their agent from entering non-progressive loops (repeating the same failing action)
- When building a safety evaluation harness to test whether an agent's action sequences comply with a set of policies

## Key Technique

**The core insight:** State prediction is not the same as risk prediction. Predicting what *will happen* after an action is only useful once you evaluate that predicted state against *explicit safety policies*. SafePred separates these concerns into a three-phase pipeline: (1) structure safety policies from natural-language documents, (2) predict both immediate UI/system changes and long-horizon outcomes for each candidate action, and (3) convert policy violations into a risk score that drives step-level feedback or full task replanning.

**How the world model works:** Given the current system state `s_t`, a candidate action `a_t`, the task intent, active policies, recent trajectory (last 7 steps), and the execution plan, the world model produces two outputs. The *short-term prediction* `s_{t+1}` describes immediate changes (files created, UI elements altered, command output). The *long-term prediction* `z_{t:T}` is a natural-language abstraction of whether the action advances the task, introduces reversible or irreversible obstacles, or deviates from objectives. This avoids unreliable multi-step rollouts by grounding long-term assessment in policy-relevant attributes rather than exhaustive state enumeration.

**The decision loop:** Each candidate action's predicted states are evaluated against the policy set to identify violations `V_t` with explanations `e_t`. A scoring function converts violations into a risk score `r_t`. If `r_t <= threshold`, the action executes. If all candidates exceed the threshold, the system generates *risk guidance* (step-level reflection injected into the agent prompt) and *plan guidance* (task-level replanning that restructures the remaining action space). This creates a closed risk-to-decision loop rather than a simple pass/fail gate.

## Step-by-Step Workflow

1. **Define safety policies as structured data.** Parse your safety requirements (natural language docs, compliance rules, organizational standards) into a machine-readable policy set. Each policy should have an ID, a natural-language description, a severity level (low/medium/high/critical), and whether violations are reversible.

2. **Capture the execution context for each candidate action.** Build an input tuple containing: current system/UI state `s_t`, the candidate action `a_t`, the high-level task intent `I`, the policy set `P`, recent trajectory window `T_{t-k+1:t}` (last 5-7 steps), and the current execution plan `P_t`.

3. **Generate short-term state prediction.** Use an LLM (or fine-tuned model) as the world model to predict the immediate next state `s_{t+1}`--what files change, what commands output, what UI elements appear or disappear. Prompt it with the full context from step 2 and request a structured JSON delta of changes.

4. **Generate long-term outcome prediction.** In the same or a follow-up call, produce a natural-language assessment `z_{t:T}` of downstream consequences: does this action advance the task, create technical debt, introduce security exposure, or cause irreversible system changes?

5. **Evaluate predicted states against policies.** Pass `s_{t+1}`, `z_{t:T}`, and the policy set to a risk evaluator. For each policy, determine whether the predicted outcome violates it. Collect the set of violated policies `V_t` with explanations `e_t` for each violation.

6. **Compute the risk score.** Apply a scoring function `r_t = phi(V_t)` that aggregates violation severities. A simple approach: sum severity weights (low=1, medium=3, high=7, critical=15) across all violated policies.

7. **Apply the threshold decision.** If `r_t <= tau` (your configured threshold), allow the action. If `r_t > tau`, block it and proceed to corrective intervention.

8. **Generate step-level risk guidance.** When an action is blocked, produce a reflection prompt that explains *which* policies were violated, *why* the predicted outcome is unsafe, and *what alternative approaches* the agent should consider. Inject this as context into the agent's next action generation.

9. **Trigger task-level replanning when needed.** If step-level guidance fails to produce a safe alternative after N retries (typically 2-3), escalate to plan guidance: restructure the remaining execution plan to route around the unsafe region entirely.

10. **Log all predictions, scores, and decisions.** Record every `(action, prediction, violations, score, decision)` tuple for auditability and for future fine-tuning of the world model.

## Concrete Examples

**Example 1: Preventing irreversible system Python modification**

```
User: My CLI agent needs to install numpy for a data task. Add a safety check.

Approach:
1. Define policies:
   - P1: "Do not modify the system Python environment directly" (severity: high)
   - P2: "Use virtual environments for package installation" (severity: high)
   - P3: "Avoid sudo for package management unless explicitly authorized" (severity: critical)

2. Agent proposes action: `sudo pip install numpy`

3. World model short-term prediction:
   {
     "state_delta": "numpy installed in /usr/lib/python3/dist-packages",
     "side_effects": ["system pip upgraded", "system packages potentially overwritten"]
   }

4. World model long-term prediction:
   "Installing into the system Python will conflict with OS package manager.
    Future apt upgrades may break. Reversal requires full Python reinstall."

5. Risk evaluation:
   - P1 violated: modifies system Python (severity 7)
   - P2 violated: no virtualenv used (severity 7)
   - P3 violated: uses sudo (severity 15)
   - Total risk score: 29 (threshold: 10) -> BLOCKED

6. Risk guidance injected into agent:
   "Action blocked: modifies system Python directly. Create a virtual environment
    first with `python3 -m venv .venv && source .venv/bin/activate`, then install
    numpy without sudo."

7. Agent regenerates: `python3 -m venv .venv && source .venv/bin/activate && pip install numpy`
   Risk score: 0 -> ALLOWED
```

**Example 2: Catching credential hardcoding in a deployment script**

```
User: Build a guardrail for my DevOps agent that writes deployment configs.

Approach:
1. Define policies:
   - P1: "Never hardcode secrets, passwords, or API keys in files" (severity: critical)
   - P2: "Use environment variables or secret managers for credentials" (severity: high)
   - P3: "Do not commit .env files or credentials to version control" (severity: critical)

2. Agent proposes action: write to deploy.sh with content including
   `export DB_PASSWORD="hunter2"`

3. Short-term prediction:
   {
     "state_delta": "deploy.sh created with plaintext password on line 12",
     "file_tracked": true
   }

4. Long-term prediction:
   "Password will be committed to git history. Even if later removed, it persists
    in git log. Any developer with repo access gains database credentials."

5. Risk evaluation:
   - P1 violated: hardcoded password (severity 15)
   - Total risk score: 15 -> BLOCKED

6. Risk guidance: "Use `$DB_PASSWORD` referencing an environment variable set
   via your CI/CD secret store, or read from a .env file listed in .gitignore."

7. Agent regenerates deploy.sh using `${DB_PASSWORD:?Required}` syntax.
   Risk score: 0 -> ALLOWED
```

**Example 3: Detecting non-progressive agent loops**

```
User: My browser agent keeps clicking the same button. How do I catch this?

Approach:
1. Define policy:
   - P1: "Agent must not repeat the same action more than 3 times without
     varying approach" (severity: medium)

2. Trajectory window shows last 5 actions:
   ["click #edit-btn", "click #edit-btn", "click #edit-btn", "click #edit-btn"]

3. Long-term prediction:
   "Action has been attempted 4 times with no state change. Repeating will not
    advance the task. Agent is in a non-progressive loop."

4. Risk evaluation:
   - P1 violated (severity 3)
   - Score: 3 -> but combined with loop detection heuristic -> escalate to
     PLAN GUIDANCE

5. Plan guidance: "The edit button is non-responsive. Replan: try keyboard
   shortcut Ctrl+E, or navigate to the edit page via URL, or check if
   authentication is required first."
```

## Best Practices

- **Do:** Keep policies explicit and machine-readable. Vague policies like "be safe" produce unreliable risk scores. Write policies as falsifiable statements: "Do not delete files outside the project directory."
- **Do:** Use the trajectory window (last 5-7 steps) as context for predictions. Actions that seem safe in isolation may be dangerous given recent history (e.g., a second `rm -rf` after a failed first attempt).
- **Do:** Separate short-term and long-term predictions. Short-term catches immediate dangers; long-term catches delayed consequences. Both are necessary.
- **Do:** Tune the risk threshold `tau` per environment. Development environments can tolerate higher thresholds than production systems.
- **Avoid:** Treating the world model as infallible. It predicts *likely* outcomes, not certain ones. Use conservative thresholds and always log predictions for review.
- **Avoid:** Blocking without guidance. A guardrail that only says "no" without suggesting alternatives forces the agent into a dead end. Always pair blocks with constructive risk guidance.
- **Avoid:** Running predictions on every trivial action (e.g., reading a file). Use a lightweight pre-filter to skip obviously safe actions and reserve world-model calls for state-changing operations.

## Error Handling

- **World model returns low-confidence prediction:** Fall back to conservative blocking and surface the uncertainty to the user. Log the action for human review.
- **All candidate actions are blocked and replanning also fails:** Halt execution and escalate to the user with the full risk assessment. Never silently loop on blocked actions.
- **Policy set is empty or misconfigured:** Refuse to run without policies. A guardrail with no policies provides false safety assurance. Require at least one policy before enabling the pipeline.
- **Trajectory window is corrupted or missing:** Proceed with short-term prediction only but flag that long-term prediction reliability is degraded. Increase the risk threshold temporarily.
- **Latency constraints:** If world-model inference is too slow for interactive use, cache predictions for identical (state, action) pairs and use the lightweight pre-filter aggressively.

## Limitations

- **World model accuracy depends on the underlying LLM.** Smaller models (8B parameters) achieve ~85% prediction accuracy vs. ~95% for frontier models. The safety-utility tradeoff is real: more conservative thresholds reduce risk but also reduce task completion.
- **Cannot predict truly novel failure modes.** The world model extrapolates from training data. Unprecedented system configurations or exotic attack vectors may not be predicted accurately.
- **Adds latency to every action.** Each candidate action requires an LLM inference call for prediction + evaluation. For latency-sensitive applications, batch predictions or use a fine-tuned small model (the paper's SafePred-8B was trained in 36 GPU-hours on 1,575 samples).
- **Policy quality is the ceiling.** If your safety policies don't cover a risk category, the guardrail won't catch it regardless of prediction quality. Policy authoring requires domain expertise.
- **Not a substitute for sandboxing.** Predictive guardrails reduce risk but don't eliminate it. Always combine with execution sandboxes, permission boundaries, and human-in-the-loop escalation for critical operations.

## Reference

**Paper:** [SafePred: A Predictive Guardrail for Computer-Using Agents via World Models](https://arxiv.org/abs/2602.01725v1) (Chen et al., 2026). Key sections: Section 3 for the three-phase architecture, Algorithm 1 for the complete decision loop pseudocode, Section 4.2 for the world model input/output format, and Table 2 for benchmark results showing 97.6% safety compliance with 21.4% utility improvement over reactive baselines.