---
name: "mock-worlds-real-skills"
description: >
  Build synthetic agentic training environments with mock tools, underspecified tasks,
  and rubric-based evaluation. Use this skill when users ask to "create synthetic training
  data for tool-use agents", "build a mock environment for agent testing", "generate
  underspecified tasks with hidden context", "design rubric-based reward functions for
  agentic RL", "simulate tool APIs for agent rollouts", or "train small models to beat
  large ones on agentic tasks".
---

# Mock Worlds, Real Skills: Synthetic Agentic Training with SYNTHAGENT

This skill enables Claude to apply the SYNTHAGENT framework for building synthetic agentic training pipelines. The core idea: instead of relying on narrow open-source datasets or unstable real APIs, you jointly synthesize diverse tool-use tasks *and* simulate complete mock environments, then train small LLMs with rubric-based reinforcement learning. The key innovation is deliberate **information partitioning** -- splitting task context into what the agent sees (an underspecified instruction) and what only a simulated user knows (hidden context) -- which forces the agent to learn clarification-seeking behavior and genuine multi-step reasoning rather than shortcut memorization.

## When to Use

- When building a training pipeline for small agentic LLMs that need to match or exceed larger models on tool-use, search, or math tasks
- When generating synthetic training data for multi-turn tool-calling agents and existing datasets are too narrow or too easy
- When real-world APIs are too unstable, rate-limited, or expensive for large-scale RL rollouts and you need mock tool environments
- When designing reward functions for agentic RL that go beyond binary success/failure signals
- When you want agents to learn to ask clarifying questions instead of hallucinating missing information
- When prototyping a new tool ecosystem and need to test agent behavior before building real integrations

## Key Technique

SYNTHAGENT addresses two structural bottlenecks in agentic RL: (1) existing training data is narrow and trivially solvable, and (2) real APIs are unstable during large-scale rollout. The framework has three interlocking components.

**Synthetic Task & Tool Ecosystem Generation.** A strong teacher model generates tasks guided by diverse personas (e.g., a database administrator, a newsletter editor, a logistics planner). For each persona, the teacher infers a high-level workflow, constructs a task-specific virtual tool ecosystem with full API specifications, and adds task-level forbidden constraints (e.g., "do not call VoiceTranscriber on audio exceeding 90 seconds"). Crucially, each task gets its *own* dedicated tool suite, so the agent must learn tool-use procedures rather than memorize a fixed API catalog.

**Information Partitioning & Mock Execution.** The explicit task description is rewritten into an intentionally underspecified instruction. Critical details are moved to a hidden user context that the agent can only access by asking clarifying questions. During rollouts, an LLM-based user simulator answers agent queries from this hidden context, while a lightweight mock tool system maintains a finite mapping of (tool_call, response) pairs per task for deterministic, stable execution. This mock system stays small -- typically under 160 entries per task across 16 rollout trajectories -- because each task has its own tool suite with only a few calls needing consistency.

**Rubric-Based Composite Rewards.** Instead of a single binary reward, each task gets a structured rubric: `R(τ) = I(τ) * (N_subgoals(τ) + I_user_query(τ))`, where `I(τ)` is a binary penalty zeroing out reward on forbidden behavior violations, `N_subgoals(τ)` is the fraction of required workflow subgoals completed, and `I_user_query(τ)` is the fraction of required user interactions satisfied. This decomposition gives the RL optimizer a dense, interpretable signal.

## Step-by-Step Workflow

1. **Select diverse personas.** Draw from a persona pool (e.g., Persona Hub or your own domain-specific list). Each persona defines a distinct professional or personal context that shapes the task. Aim for at least 50 unique personas to ensure task diversity.

2. **Generate task workflows per persona.** For each persona, use a strong model to produce a high-level workflow describing how that person would accomplish a realistic goal. Example: a senior care coordinator's workflow might be "Collect voice update -> Transcribe -> Find literary quote -> Add photo with alt-text -> Format newsletter -> Generate audio narration."

3. **Synthesize a task-specific tool ecosystem.** For each workflow, generate a set of tool descriptions with full API specifications (name, parameters, return types, error conditions). Each task gets its own tool suite -- do not reuse a global tool catalog. Example tools for the newsletter task: `VoiceTranscriber(audio_url, max_duration)`, `LiteratureFinder(theme, genre)`, `ImageDescriber(image_url)`, `NewsletterFormatter(sections, template)`, `AudioNarrationGenerator(text, voice_style)`.

4. **Define forbidden constraints.** For each task, specify 1-3 behaviors the agent must never perform. These act as hard guardrails. Examples: "Do not call VoiceTranscriber on audio exceeding 90 seconds", "Do not reboot the database during active transactions", "Do not send the newsletter without user confirmation of the final draft."

5. **Partition information into instruction vs. hidden context.** Rewrite the explicit workflow into a vague, underspecified user request. Move critical details (preferences, constraints, specific values) into a hidden context document that only the user simulator can access. The instruction should have high conditional entropy -- the agent *cannot* solve it without asking questions. Example: "The checkout service is returning 500 errors again. Can you investigate and fix it?" (hidden: the root cause is a stale DB connection pool; rollback is required but only after verifying no active transactions).

6. **Build the mock tool system.** Implement a per-task finite mapping `M = {(tool_call_i, response_i)}`. Pre-seed it with the expected tool calls from the reference workflow. During rollouts, when an agent issues a call: check M for a semantically equivalent prior call; if found, return the cached response; if not, generate a plausible response and add it to M.

7. **Configure the user simulator.** Set up a lightweight LLM (can be small, e.g., 3B-8B parameters) with the hidden context as its system prompt. It answers agent questions faithfully from the hidden context. Since these are simple formatted Q&A interactions, this component does not need a powerful model.

8. **Construct the rubric.** For each task, extract subgoals from the reference workflow (each step = one subgoal). List required user interactions (questions the agent must ask before it can proceed). Combine with forbidden constraints into the composite reward: `R = I_forbidden * (frac_subgoals + frac_interactions)`.

9. **Run RL training with GRPO.** Use Group Relative Policy Optimization with rollout size 16, max response length ~13K tokens, max 16 turns per rollout. Train for 2 epochs. Mix ~11K synthetic tool-use tasks with a smaller set (~4K) of math/search reasoning instances rewritten with personas.

10. **Evaluate on held-out agentic benchmarks.** Test on multi-turn tool-use benchmarks (e.g., TAU-2, BFCL Multi-turn) and reasoning benchmarks (AIME, Olympiad). Compare against both the base model and larger baselines to verify that the small model has closed or exceeded the gap.

## Concrete Examples

**Example 1: Generating a synthetic task with mock tools**

User: "I need to create synthetic training data for a tool-use agent that handles IT incident response."

Approach:
1. Select persona: "On-call SRE at a mid-size fintech company"
2. Generate workflow: Check alerting dashboard -> Query log aggregator -> Identify failing service -> Check service dependencies -> Apply hotfix or rollback -> Verify recovery -> Update incident ticket
3. Synthesize tools:
```json
[
  {"name": "AlertDashboard.getActiveAlerts", "params": {"severity": "string", "service": "string?"}, "returns": "Alert[]"},
  {"name": "LogAggregator.search", "params": {"query": "string", "time_range": "string", "service": "string"}, "returns": "LogEntry[]"},
  {"name": "ServiceGraph.getDependencies", "params": {"service_name": "string"}, "returns": "Dependency[]"},
  {"name": "DeploymentManager.rollback", "params": {"service": "string", "to_version": "string"}, "returns": "DeployResult"},
  {"name": "IncidentTracker.update", "params": {"incident_id": "string", "status": "string", "notes": "string"}, "returns": "Ticket"}
]
```
4. Define forbidden constraints: "Do not rollback without first checking active transactions via LogAggregator", "Do not close the incident ticket before verifying recovery"
5. Write underspecified instruction: "The payments service is throwing errors. Please look into it."
6. Hidden context: "The root cause is a bad deploy of payments-service v2.3.1 pushed 20 minutes ago. The safe rollback target is v2.3.0. There are no active transactions currently. The incident ID is INC-4821."

Output rubric:
```yaml
forbidden:
  - "Called DeploymentManager.rollback before LogAggregator.search"
  - "Called IncidentTracker.update with status=resolved before verifying recovery"
subgoals:
  - "Retrieved active alerts for payments service"
  - "Searched logs to identify the failing version"
  - "Checked service dependencies"
  - "Rolled back to v2.3.0"
  - "Verified service recovery"
  - "Updated incident ticket INC-4821"
required_interactions:
  - "Asked user which service is affected or confirmed payments"
  - "Asked about or confirmed rollback target version"
  - "Asked whether active transactions need draining"
```

**Example 2: Building a mock tool environment for RL rollouts**

User: "I have 500 synthetic tasks. How do I build the mock tool system for stable rollouts?"

Approach:
1. For each task, initialize an empty mapping `M = {}`
2. Pre-seed M with reference trajectory tool calls and their expected responses
3. Implement the rollout loop:

```python
class MockToolSystem:
    def __init__(self, reference_calls: list[tuple[str, dict, Any]]):
        # Pre-seed with reference trajectory
        self.mapping = {}
        for tool_name, args, response in reference_calls:
            key = self._normalize(tool_name, args)
            self.mapping[key] = response

    def execute(self, tool_name: str, args: dict, generator_llm=None) -> Any:
        key = self._normalize(tool_name, args)
        # Check for semantically equivalent prior call
        match = self._find_match(key)
        if match:
            return self.mapping[match]
        # Generate new response and cache it
        response = generator_llm.generate_tool_response(tool_name, args)
        self.mapping[key] = response
        return response

    def _normalize(self, tool_name: str, args: dict) -> str:
        return f"{tool_name}::{json.dumps(args, sort_keys=True)}"

    def _find_match(self, key: str) -> str | None:
        if key in self.mapping:
            return key
        # Optional: fuzzy matching for semantically equivalent calls
        for existing_key in self.mapping:
            if self._semantically_equivalent(key, existing_key):
                return existing_key
        return None
```

4. Keep M per-task (not shared across tasks) to avoid cross-contamination
5. Typical size: <160 entries per task with 16 rollout trajectories and ~10 tool calls each

**Example 3: Designing a rubric-based reward for a complex task**

User: "How do I write a reward function that handles subgoals, user interactions, and forbidden behaviors?"

Approach:
1. Define the three components separately, then combine:

```python
def compute_reward(trajectory, rubric):
    # Component 1: Forbidden behavior check (binary gate)
    for action in trajectory.actions:
        for forbidden in rubric.forbidden:
            if forbidden.matches(action):
                return 0.0  # Hard zero -- entire trajectory gets no reward

    # Component 2: Subgoal completion fraction
    completed = sum(1 for sg in rubric.subgoals if sg.satisfied_by(trajectory))
    subgoal_score = completed / len(rubric.subgoals)

    # Component 3: Required user interaction fraction
    asked = sum(1 for q in rubric.required_interactions if q.asked_in(trajectory))
    interaction_score = asked / len(rubric.required_interactions)

    # Composite reward
    return subgoal_score + interaction_score  # Range [0, 2], normalized by GRPO
```

2. The forbidden gate is multiplicative -- one violation zeros everything, giving a strong gradient signal against unsafe behavior
3. Subgoals and interactions are additive, providing dense partial-credit signal even when the agent only completes part of the task

## Best Practices

- **Do:** Generate a unique tool suite per task. This prevents the agent from memorizing API names and forces learning of transferable tool-use procedures.
- **Do:** Make instructions genuinely underspecified. If the agent can solve the task without asking any questions, the information gap is too small and RL training will suffer from gradient degeneracy.
- **Do:** Keep the mock tool mapping per-task and small. Cross-task sharing introduces spurious correlations. Under 160 entries per task is typical and sufficient.
- **Do:** Include forbidden constraints on every task. The binary penalty gate is the most effective component for teaching safety-aware behavior.
- **Avoid:** Using a single global tool catalog across all tasks. This leads to memorization rather than generalization.
- **Avoid:** Making hidden context too complex for the user simulator. The simulator should handle straightforward Q&A -- if it needs chain-of-thought reasoning to answer, simplify the context.
- **Avoid:** Setting max response length too short. Agentic rollouts with multi-turn tool calling and user interaction need at least 10K-13K tokens to complete realistic tasks.

## Error Handling

- **Agent never asks clarifying questions:** The information gap is insufficient. Increase underspecification by moving more critical parameters into hidden context. Verify that the instruction truly cannot be solved without the hidden information.
- **Mock tool responses are inconsistent across rollouts:** Ensure the mapping lookup uses normalized keys (sorted args, canonical tool names). Add semantic equivalence checking for slight parameter variations.
- **Reward signal is too sparse (most trajectories score 0):** Check that forbidden constraints are not too aggressive. Start with 1 forbidden behavior per task and scale up. Also verify subgoals are decomposed finely enough to give partial credit.
- **Training diverges or reward collapses:** Reduce the clipping ratio (start at 0.2), ensure rollout batch size is large enough (16+), and confirm the mix of synthetic tool-use and reasoning tasks is balanced (~70/30 split).
- **User simulator gives hallucinated answers:** Constrain the simulator prompt to only answer from the provided hidden context. Add an explicit instruction: "If the agent asks something not covered in the context, say you don't know."

## Limitations

- Requires a strong teacher model (70B+ parameters recommended) for initial task and tool synthesis. The quality ceiling of synthetic data is bounded by the teacher's capability.
- Mock tool environments cannot capture real-world API quirks like rate limiting, partial failures, or eventual consistency. Agents trained purely on mocks may need fine-tuning on real APIs for production deployment.
- Information partitioning assumes tasks where clarification-seeking is natural. For tasks with complete, unambiguous specifications (e.g., pure computation), this approach adds no value.
- Rubric construction requires domain knowledge to define meaningful subgoals and forbidden behaviors. Poorly designed rubrics can reward trivial behavior or penalize valid approaches.
- The approach is validated on models up to 14B parameters. Scaling behavior to 70B+ models is untested and may show diminishing returns.

## Reference

[Mock Worlds, Real Skills: Building Small Agentic Language Models with Synthetic Tasks, Simulated Environments, and Rubric-Based Rewards](https://arxiv.org/abs/2601.22511v1) -- Look for Section 3 (SYNTHAGENT framework details), Table 6 (concrete task examples with hidden context), Table 7 (rubric examples), and the ablation study showing that removing the information gap eliminates performance gains.