---
name: "constrained-process-maps-multi-agent"
description: "Build multi-agent workflows structured as constrained DAG process maps with Monte Carlo uncertainty estimation. Each agent occupies a specialized review role, with predefined escalation paths and terminal states (automated label or human review). Use when: 'build a multi-agent compliance pipeline', 'add uncertainty-aware escalation to my agent workflow', 'create a review chain with human fallback', 'design a DAG-based agent workflow with confidence thresholds', 'implement Monte Carlo sampling for agent decisions', 'build a multi-stage content moderation system'."
---

# Constrained Process Maps for Multi-Agent Workflows

This skill enables Claude to design and implement multi-agent systems structured as directed acyclic graphs (DAGs), where each node is a specialized LLM agent with a defined role (e.g., content reviewer, risk assessor, legal checker). Agents quantify their own epistemic uncertainty through Monte Carlo sampling -- running the same classification multiple times and measuring agreement -- then use predefined escalation edges to route uncertain cases to downstream specialists or human review. The framework is formalized as a finite-horizon MDP, guaranteeing termination and enabling principled accuracy-vs-cost trade-offs. Based on Joshi & Rudow (2026), this approach achieved up to 19% accuracy gains and 85x reduction in human review burden over single-agent baselines.

## When to Use

- When the user needs a multi-agent pipeline for classification, moderation, or compliance tasks with escalation logic
- When building content safety systems that must route uncertain cases to human reviewers
- When designing review workflows where different agents handle different regulatory or domain concerns (legal, risk, content)
- When the user wants to quantify LLM decision confidence without access to model logits or internal probabilities
- When implementing a multi-stage approval or labeling pipeline that must guarantee termination
- When the user asks to reduce human review workload by filtering high-confidence automated decisions

## Key Technique

**DAG-Structured Process Maps.** The workflow is a directed acyclic graph G = (S, E) where nodes are specialized agents and edges are allowable escalation paths. Terminal (absorbing) states are the final outcomes: a concrete label (e.g., "safe", "unsafe") or "human review". Because the graph is acyclic, every trajectory is guaranteed to terminate within the DAG's diameter -- no infinite loops, no runaway agent chains. Each agent has a narrowly-scoped system prompt (a Standard Operating Procedure) defining its review focus.

**Monte Carlo Uncertainty Estimation.** Instead of asking an LLM "how confident are you?" (which is unreliable), each agent generates `n` independent classification samples for the same input. The distribution of outputs (e.g., 8 "safe", 1 "unsafe", 1 "uncertain" out of n=10) empirically quantifies epistemic uncertainty without accessing model internals. A policy function (e.g., majority vote with a confidence threshold) maps this sample vector to a decision: finalize the label, or escalate to the next agent in the DAG.

**Two-Level Uncertainty Model.** Agent-level uncertainty is the Monte Carlo sample distribution. System-level uncertainty is captured by which terminal state the MDP reaches -- if the pipeline cannot resolve the case with sufficient confidence, it terminates in the human-review absorbing state. This separation lets you tune each agent's threshold independently and optimize the overall system for your target accuracy-vs-human-review trade-off.

## Step-by-Step Workflow

1. **Define the terminal states.** Enumerate the final outcomes your pipeline must produce: concrete labels (e.g., `safe`, `unsafe`, `approved`, `rejected`) plus `human_review` as the catch-all escalation sink.

2. **Identify agent roles and draw the DAG.** Map each review concern to a dedicated agent node. Define directed edges representing escalation paths. A typical three-tier structure: `Worker -> Triage -> {Risk Agent, Legal Agent} -> terminal states`. Verify the graph is acyclic and every path reaches a terminal state.

3. **Write scoped system prompts (SOPs) for each agent.** Each agent gets a narrow mandate. The Worker handles first-pass classification, the Triage agent routes based on uncertainty type, and specialist agents (Risk, Legal, etc.) apply domain-specific criteria. Prompts must instruct the agent to output one of the allowed labels (e.g., `safe`, `unsafe`, `uncertain`) followed by reasoning.

4. **Implement Monte Carlo sampling per agent.** For each input at each agent node, call the LLM `n` times independently to produce a sample vector `[a^(1), ..., a^(n)]`. Start with n=1 for speed, scale to n=3-5 for confidence, or n=25+ for fine-grained threshold tuning on critical subsets.

5. **Define the policy function for each agent.** Map the sample vector to a decision. The simplest policy is majority vote: if >50% of samples agree on a label, finalize it; otherwise escalate. For tighter control, set a confidence threshold (e.g., require 80% agreement to finalize). The threshold controls the accuracy-vs-escalation trade-off.

6. **Implement the transition logic.** Code the DAG traversal: start at the Worker, apply the agent's policy, follow the appropriate edge to the next node or terminal state. Pass the input and any accumulated context along escalation edges.

7. **Wire in the human review terminal state.** When the pipeline exhausts all agent nodes without reaching a confident label, route to human review. Log the full trajectory (which agents saw the case, their sample distributions, their reasoning) so the human reviewer has context.

8. **Run calibration experiments.** Test the pipeline on a labeled dataset across different values of `n` and different confidence thresholds. Measure accuracy, false positive/negative rates, human review volume, and latency. Use 5+ independent trials to estimate confidence intervals.

9. **Tune thresholds per agent.** Agents at different positions in the DAG may need different confidence levels. A first-pass Worker can be aggressive (low threshold, high throughput), while a Legal agent near the end of the pipeline should be conservative (high threshold, low risk of error).

10. **Deploy with monitoring.** Track per-agent uncertainty distributions, escalation rates, and terminal-state frequencies in production. Drift in these metrics signals model degradation or distribution shift in inputs.

## Concrete Examples

**Example 1: Content Safety Moderation Pipeline**

User: "Build a multi-agent content moderation system that classifies user posts as safe or unsafe, with human review fallback for uncertain cases."

Approach:
1. Define terminal states: `safe`, `unsafe`, `human_review`
2. Design DAG: `ContentReviewer -> Triage -> {RiskAssessor, PolicyChecker} -> terminal states`
3. Implement each agent with scoped prompts and Monte Carlo sampling

```python
import asyncio
import json
from collections import Counter

# --- DAG Definition ---
PROCESS_MAP = {
    "content_reviewer": {
        "escalates_to": "triage",
        "can_finalize": ["safe", "unsafe"],
    },
    "triage": {
        "escalates_to": ["risk_assessor", "policy_checker"],
        "can_finalize": [],  # triage only routes, never finalizes
    },
    "risk_assessor": {
        "escalates_to": "human_review",
        "can_finalize": ["safe", "unsafe"],
    },
    "policy_checker": {
        "escalates_to": "human_review",
        "can_finalize": ["safe", "unsafe"],
    },
}

AGENT_PROMPTS = {
    "content_reviewer": (
        "You are a content safety reviewer. Classify the following user post "
        "with a single word: 'safe', 'unsafe', or 'uncertain'. "
        "Then provide a comma and your reasoning."
    ),
    "risk_assessor": (
        "You are a risk assessment specialist. Evaluate whether this content "
        "poses risks of harm, misinformation, or bias. Classify as 'safe', "
        "'unsafe', or 'uncertain', followed by a comma and reasoning."
    ),
    "policy_checker": (
        "You are a platform policy compliance checker. Evaluate whether this "
        "content violates terms of service or community guidelines. Classify "
        "as 'safe', 'unsafe', or 'uncertain', followed by a comma and reasoning."
    ),
}

async def monte_carlo_sample(llm_client, prompt: str, input_text: str, n: int) -> list[str]:
    """Generate n independent classification samples."""
    tasks = [
        llm_client.complete(f"{prompt}\n\nContent: {input_text}")
        for _ in range(n)
    ]
    responses = await asyncio.gather(*tasks)
    labels = []
    for r in responses:
        first_word = r.strip().split(",")[0].strip().lower()
        if first_word in ("safe", "unsafe", "uncertain"):
            labels.append(first_word)
        else:
            labels.append("uncertain")
    return labels

def apply_policy(samples: list[str], threshold: float = 0.67) -> str:
    """Majority vote with confidence threshold. Returns label or 'escalate'."""
    counts = Counter(samples)
    total = len(samples)
    for label in ["safe", "unsafe"]:
        if counts.get(label, 0) / total >= threshold:
            return label
    return "escalate"

async def run_pipeline(llm_client, input_text: str, n: int = 3) -> dict:
    """Traverse the DAG from content_reviewer to a terminal state."""
    trajectory = []
    current_agent = "content_reviewer"

    while current_agent not in ("safe", "unsafe", "human_review"):
        config = PROCESS_MAP[current_agent]
        prompt = AGENT_PROMPTS.get(current_agent)

        if prompt is None:
            # Triage node: route to first specialist
            current_agent = config["escalates_to"][0]
            continue

        samples = await monte_carlo_sample(llm_client, prompt, input_text, n)
        decision = apply_policy(samples, threshold=0.67)
        trajectory.append({
            "agent": current_agent,
            "samples": samples,
            "decision": decision,
        })

        if decision == "escalate":
            next_node = config["escalates_to"]
            current_agent = next_node if isinstance(next_node, str) else next_node[0]
        else:
            current_agent = decision  # terminal state

    return {"label": current_agent, "trajectory": trajectory}
```

Output: Each input gets a label (`safe`, `unsafe`, or `human_review`) plus a full trajectory showing which agents processed it and their sample distributions.

---

**Example 2: Document Compliance Review**

User: "I need a three-stage review pipeline for loan applications: initial screening, risk assessment, and regulatory compliance. Uncertain cases go to a human underwriter."

Approach:
1. Terminal states: `approved`, `rejected`, `human_underwriter`
2. DAG: `Screener -> RiskAnalyst -> RegulatoryReviewer -> human_underwriter`
3. Each agent samples n=5 times with domain-specific SOPs

```python
LOAN_DAG = {
    "screener": {
        "sop": "Check basic eligibility: income, credit score, employment status.",
        "escalates_to": "risk_analyst",
        "labels": ["approved", "rejected", "uncertain"],
        "threshold": 0.8,  # high confidence to auto-approve at first pass
    },
    "risk_analyst": {
        "sop": "Assess default risk, debt-to-income ratio, collateral value.",
        "escalates_to": "regulatory_reviewer",
        "labels": ["approved", "rejected", "uncertain"],
        "threshold": 0.6,
    },
    "regulatory_reviewer": {
        "sop": "Verify compliance with lending regulations, fair lending laws, TILA.",
        "escalates_to": "human_underwriter",
        "labels": ["approved", "rejected", "uncertain"],
        "threshold": 0.6,
    },
}
```

Each agent runs 5 independent LLM calls. The Screener uses a high threshold (0.8) because auto-approving at the first stage carries the most risk. Later agents use lower thresholds because cases reaching them have already been flagged as marginal.

---

**Example 3: Tuning n and Thresholds**

User: "My pipeline sends too many cases to human review. How do I reduce that?"

Approach:
1. Run calibration with labeled test data across n=1, 3, 5, 10
2. For each n, sweep confidence thresholds from 0.5 to 0.9
3. Plot accuracy vs. human review rate, pick the operating point

```
n=1:  accuracy=88.0%, human_review=0.2%   (fast, aggressive)
n=3:  accuracy=84.8%, human_review=2.3%   (balanced)
n=5:  accuracy=84.3%, human_review=3.8%   (conservative)
n=25: accuracy=86.1%, human_review=1.1%   (fine-grained thresholds)
```

Key insight: n=1 with a simple pass/escalate policy often outperforms higher n values on both accuracy and human review volume, because it avoids the "regression to uncertainty" that occurs when Monte Carlo samples introduce noise at moderate n. Use higher n only when you need fine-grained threshold control for specific agents.

## Best Practices

- **Do:** Keep agent SOPs narrowly scoped. A Risk agent should only evaluate risk, not also check legal compliance. Narrow scope improves Monte Carlo sample consistency.
- **Do:** Start with n=1 (single sample per agent) as your baseline, then increase n only if you need finer threshold control. The paper found n=1 often gives the best accuracy-to-cost ratio.
- **Do:** Make the DAG explicit and acyclic. Draw it out. Verify every path terminates. Never allow an edge from a downstream agent back to an upstream one.
- **Do:** Log the full trajectory (agent, samples, decision) for every input. This audit trail is essential for debugging, calibration, and regulatory compliance.
- **Avoid:** Asking the LLM to self-report confidence ("on a scale of 1-10, how sure are you?"). Monte Carlo sampling over independent calls is more reliable than introspective confidence.
- **Avoid:** Using a single monolithic agent with a long prompt that tries to handle all review stages. The paper showed this baseline underperforms a structured DAG by up to 19% accuracy.

## Error Handling

- **All samples return "uncertain":** The pipeline escalates to the next agent. If all agents return uncertain, the case correctly reaches human review. This is working as designed, not an error.
- **LLM returns an unparseable label:** Default to "uncertain" for that sample. This biases toward escalation, which is the safe failure mode.
- **Timeout or API failure on one sample:** Reduce the effective n for that agent call. If n drops below a minimum (e.g., 1), retry or escalate immediately.
- **Threshold miscalibration:** If human review volume is unexpectedly high or low, re-run calibration on a fresh labeled sample. Agent thresholds may need adjustment after model updates or input distribution shifts.
- **DAG has a cycle:** The pipeline will not terminate. Validate the graph structure at initialization by running a topological sort; reject any graph where topological sort fails.

## Limitations

- **Requires labeled calibration data.** You need ground-truth labels to tune thresholds and measure accuracy. Without calibration data, threshold selection is guesswork.
- **Latency scales with n and DAG depth.** At n=5 with three agent tiers, a single input requires up to 15 LLM calls in the worst case. Parallel sampling within each agent helps, but sequential DAG traversal is inherently serial.
- **Assumes decomposable review criteria.** The approach works when review concerns (content, risk, legal) are separable into independent agent roles. Tightly coupled criteria that require joint reasoning may not decompose cleanly.
- **Static policy only.** The majority-vote policy is fixed at deployment time. The paper does not address online policy learning or adaptive thresholds that respond to streaming input distributions.
- **Not suited for generative tasks.** This framework is designed for classification/labeling pipelines. It does not apply to open-ended generation, summarization, or creative tasks where "safe/unsafe/uncertain" labels are not meaningful.

## Reference

Joshi, A. & Rudow, M. (2026). *Constrained Process Maps for Multi-Agent Generative AI Workflows.* arXiv:2602.02034v1. Key sections: Section 3 (MDP formalization and DAG structure), Section 4 (Monte Carlo uncertainty estimation), Section 5 (compliance case study with self-harm detection results).