---
name: "mitigating-conversational-inertia-multi-turn"
description: "Detect and break conversational inertia in multi-turn agent interactions — where an LLM repeats its own prior actions as implicit few-shot examples instead of exploring alternatives. Apply Context Preference Learning and context management to improve agentic task performance. Use when: 'my agent keeps repeating the same action', 'break out of action loops', 'agent is stuck in a loop', 'improve multi-turn agent exploration', 'balance exploration and exploitation in agent', 'reduce imitation bias in tool-calling agent'."
---

# Mitigating Conversational Inertia in Multi-Turn Agents

This skill teaches Claude to detect and counteract **conversational inertia** — the phenomenon where LLMs in multi-turn agentic loops develop strong diagonal attention to their own previous responses, causing them to repeat the same (often failing) actions instead of exploring alternatives. The technique, from Wan et al. (2026), provides concrete context management strategies that reduce imitation bias and improve task completion rates by 5-10% across diverse agent environments without requiring environment reward signals.

## When to Use

- When building or debugging a multi-turn agent (tool-calling loop, ReAct, etc.) that gets stuck repeating the same failed action
- When an agentic workflow plateaus because the model mimics its prior outputs instead of trying new approaches
- When designing context windows for long-running agent sessions (e.g., web browsing, code debugging, game playing)
- When a user reports their agent "loops" or "gets stuck" after many turns
- When optimizing the exploration/exploitation tradeoff in an agent that uses its conversation history as implicit few-shot examples
- When implementing context management (truncation, windowing, periodic clearing) for multi-turn tool-use agents

## Key Technique

**The Problem — Diagonal Attention as Imitation Bias.** In multi-turn agent conversations, the model's attention mechanism develops a diagonal pattern: the i-th token of the current response disproportionately attends to the i-th position of the previous assistant response. This means the model literally copies the structure and content of its last action. The longer the conversation history, the stronger this effect — creating a tension where more context provides richer environmental feedback (exploitation) but also amplifies repetitive behavior (undermines exploration).

**The Insight — Context Length Controls Inertia.** For identical environment states, actions generated with longer conversation contexts exhibit measurably stronger inertia (more repetition) than actions generated with shorter contexts. This asymmetry lets you construct preference pairs without any environment reward: the short-context response is preferred (more exploratory) and the long-context response is rejected (more inertial). This is the basis of **Context Preference Learning (CPL)**, which uses Direct Preference Optimization (DPO) to train the model to favor low-inertia outputs.

**The Inference Strategy — Context Management.** Even without fine-tuning, you can reduce inertia at inference time through three context management strategies: (1) **Window Context** — retain only the W most recent conversation rounds (W=6 is optimal); (2) **Clip Context** — periodically clear the context when it exceeds H rounds, retaining only the L most recent rounds (H=12, L=1); (3) **Long Context** — full history, useful when exploitation matters more than exploration. Clip Context achieves the best balance, reducing diagonal attention by ~14% while providing 2-7x prefill speedup.

## Step-by-Step Workflow

1. **Diagnose inertia.** Review the agent's conversation history for repeated actions. Look for the signature pattern: the agent takes the same action (or structurally similar action) 2+ times in a row despite receiving feedback that it failed. Flag sequences like `action A -> fail -> action A -> fail -> action A`.

2. **Classify the failure mode.** Determine whether the repetition is due to (a) genuinely limited action space (only one valid action exists), (b) insufficient environment feedback (the agent doesn't know it failed), or (c) conversational inertia (the agent is copying its prior output). Inertia is the cause when the agent has alternative actions available and receives clear failure signals but still repeats.

3. **Apply Window Context management.** Truncate the conversation history to the most recent W=6 interaction rounds (user/assistant pairs) before each new generation. This is the simplest intervention and works well for most agentic loops. Implement by slicing the messages array before calling the LLM.

4. **Apply Clip Context for long-running agents.** For agents running 20+ turns, implement periodic context clearing: when the conversation exceeds H=12 rounds, trim to only the L=1 most recent round plus the system prompt. This is more aggressive than windowing but breaks inertia more effectively in long sessions.

5. **Preserve critical information across clips.** When clipping context, extract and re-inject essential state into the system prompt or a summary message: current environment state, key findings so far, and actions already tried. This prevents losing exploitation value while breaking the inertia pattern.

6. **Inject exploration directives after detecting loops.** When you detect 2+ repeated actions, insert an explicit instruction: "Your previous approach of [action] failed [N] times. List 3 alternative actions before choosing one." This breaks the diagonal attention pattern by forcing divergent token generation.

7. **Diversify action framing.** When constructing the agent prompt, vary the format of action descriptions across turns. If prior turns used `{"action": "click", "target": "button"}`, switch to a natural language format or reorder fields. Format variation disrupts the positional copying that drives diagonal attention.

8. **Implement a repetition detector.** Add a programmatic check in your agent loop: if the last N actions (N >= 2) are identical or have cosine similarity > 0.9, trigger an intervention (context clip, exploration directive, or temperature increase from 0.0 to 0.3-0.5 for one turn).

9. **Balance exploration and exploitation by phase.** Use Long Context (full history) during early turns when the agent is gathering information. Switch to Window or Clip Context after turn 8-12 when the agent should be acting on what it's learned. This mirrors the explore-then-exploit pattern from reinforcement learning.

10. **Validate the fix.** After applying context management, measure: (a) action diversity — count unique actions per N turns, (b) task completion rate, (c) turns-to-completion. Effective inertia mitigation increases action diversity while maintaining or improving completion rate.

## Concrete Examples

**Example 1: Web browsing agent stuck clicking the same button**

User: My web agent keeps clicking "Show More" on a search results page instead of clicking on actual results. It's been doing this for 15 turns.

Approach:
1. Identify the inertia pattern: the agent's conversation history contains 15 rounds of `click("Show More")` -> page update -> `click("Show More")`
2. The diagonal attention is causing the model to copy `click("Show More")` from its prior response
3. Apply Clip Context: trim to system prompt + last 1 round + a summary of what's been loaded

Implementation:
```python
def manage_context(messages, max_rounds=12, clip_to=1):
    """Clip context to break conversational inertia."""
    # Count assistant/user round pairs
    rounds = []
    current_round = []
    for msg in messages[1:]:  # skip system prompt
        current_round.append(msg)
        if msg["role"] == "assistant":
            rounds.append(current_round)
            current_round = []

    if len(rounds) > max_rounds:
        # Keep system prompt + summary + last clip_to rounds
        summary = {
            "role": "user",
            "content": (
                f"[Context refreshed. Prior {len(rounds)} rounds trimmed. "
                f"Key state: You are on a search results page with 45 results loaded. "
                f"You have already clicked 'Show More' {len(rounds)} times. "
                f"Try a DIFFERENT action — click on a specific result link.]"
            )
        }
        clipped = [messages[0], summary]
        for r in rounds[-clip_to:]:
            clipped.extend(r)
        return clipped
    return messages
```

Output: After clipping, the agent generates `click("Result: Best hiking boots 2026")` instead of repeating `click("Show More")`.

**Example 2: Code debugging agent repeating the same fix**

User: My coding agent keeps adding the same try/except block even though the error is a missing import, not an exception handling issue.

Approach:
1. Detect the loop: agent has attempted `try/except` 3 times with identical code
2. Apply the repetition detector + exploration directive

Implementation:
```python
def detect_and_break_inertia(messages, agent_actions):
    """Detect repeated actions and inject exploration directive."""
    # Check last 3 actions for repetition
    recent = agent_actions[-3:]
    if len(recent) >= 2 and len(set(recent)) == 1:
        # Inertia detected — inject exploration directive
        intervention = {
            "role": "user",
            "content": (
                f"STOP: You have attempted '{recent[0]}' {len(recent)} times "
                f"and it has not resolved the error. This approach is not working. "
                f"Before your next action, list 3 fundamentally different approaches "
                f"to fix this error, then choose the most promising one."
            )
        }
        # Also apply window context (keep last 3 rounds only)
        system = messages[0]
        recent_msgs = []
        count = 0
        for msg in reversed(messages[1:]):
            recent_msgs.insert(0, msg)
            if msg["role"] == "assistant":
                count += 1
            if count >= 3:
                break
        return [system, intervention] + recent_msgs
    return messages
```

Output: The agent responds with "Alternative approaches: (1) Add `import pandas` to the top of the file, (2) Check if the module is installed with pip, (3) Verify the module name spelling" — then correctly adds the missing import.

**Example 3: Designing context management for a ReAct agent**

User: I'm building a ReAct agent for data analysis. How should I manage context to prevent it from getting stuck?

Approach:
1. Implement a phased context strategy
2. Early turns: full context for information gathering
3. Mid/late turns: window or clip context for action diversity

Implementation:
```python
class InertiaAwareAgent:
    def __init__(self, llm, tools):
        self.llm = llm
        self.tools = tools
        self.messages = []
        self.action_history = []
        self.turn = 0

    def get_context(self):
        """Phase-aware context management."""
        self.turn += 1

        # Phase 1 (turns 1-8): Full context for exploration/info gathering
        if self.turn <= 8:
            return self.messages

        # Phase 2 (turns 9+): Window context to prevent inertia
        system = self.messages[0]
        window_size = 6  # optimal from paper
        rounds = self._extract_rounds(self.messages[1:])

        if len(rounds) <= window_size:
            return self.messages

        # Build summary of early findings
        summary = self._summarize_early_rounds(rounds[:-window_size])
        windowed = [system, {"role": "user", "content": summary}]
        for r in rounds[-window_size:]:
            windowed.extend(r)
        return windowed

    def _check_inertia(self):
        """Detect action repetition."""
        if len(self.action_history) >= 3:
            last_3 = self.action_history[-3:]
            if last_3[0] == last_3[1] == last_3[2]:
                return True
        return False

    def step(self, observation):
        self.messages.append({"role": "user", "content": observation})
        context = self.get_context()

        if self._check_inertia():
            context.insert(1, {
                "role": "user",
                "content": "You are repeating yourself. Try a different approach."
            })

        response = self.llm(context)
        self.messages.append({"role": "assistant", "content": response})
        self.action_history.append(self._extract_action(response))
        return response
```

## Best Practices

**Do:**
- Implement a repetition detector as a standard component in any multi-turn agent loop — it costs almost nothing and catches inertia early
- Use Clip Context (H=12, L=1) as the default for agents running more than 15 turns; it outperforms both full context and simple windowing
- Preserve environment state information when clipping context by injecting a summary message after the system prompt
- Increase temperature slightly (0.0 -> 0.3) for one turn when inertia is detected, then return to the base temperature

**Avoid:**
- Do not simply increase temperature globally — this degrades exploitation quality on all turns, not just inertial ones; intervene surgically
- Do not assume longer context is always better for agents — the paper demonstrates that full context amplifies inertia and *reduces* performance compared to windowed context in 6/8 environments
- Do not clip context so aggressively that the agent loses track of the task goal — always retain the system prompt and a state summary
- Do not ignore the problem and rely on max-turn limits — an inertial agent wastes all remaining turns on repeated actions, burning tokens and time

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| Agent still loops after windowing | Window size too large (W > 8) | Reduce to W=4-6 or switch to Clip Context |
| Agent loses track of goal after clipping | Essential state not preserved | Add a structured summary message after each clip |
| Action diversity too high (random behavior) | Over-aggressive inertia breaking | Increase window size or reduce temperature; some repetition is valid exploitation |
| Agent performance drops on early turns | Premature context restriction | Use phased approach: full context early, restricted context after turn 8-12 |
| Repetition detector false positives | Legitimately repeated action (e.g., scrolling a long page) | Add action-specific allowlists; some actions are correctly repeated |

## Limitations

- **Context management is a heuristic, not a trained solution.** Without fine-tuning via Context Preference Learning (DPO), inference-time strategies reduce but do not eliminate inertia. They are the practical option when you cannot fine-tune the underlying model.
- **State summarization quality matters.** If the summary injected after a context clip is poor, the agent loses critical information. This is itself an LLM generation step that can fail.
- **Not all repetition is inertia.** In environments with limited action spaces or where the correct strategy genuinely involves repeating an action (e.g., retrying a network request), the repetition detector will false-positive. Allowlists or domain-specific logic are needed.
- **Optimal parameters are environment-dependent.** W=6, H=12, L=1 are good defaults from the paper's experiments across 8 environments, but specific tasks may need tuning.
- **The technique addresses action-level repetition, not reasoning-level stagnation.** An agent that generates different actions but follows the same flawed reasoning strategy requires higher-level intervention (e.g., reflection, planning resets).

## Reference

Wan, Y., Cao, Z., Zhang, Z., Zeng, Z., & Shen, S. (2026). *Mitigating Conversational Inertia in Multi-Turn Agents.* arXiv:2602.03664v2. [https://arxiv.org/abs/2602.03664v2](https://arxiv.org/abs/2602.03664v2)

Key sections to read: Section 3 (diagonal attention analysis proving inertia exists), Section 4 (Context Preference Learning algorithm and DPO formulation), Section 5.3 (context management strategies with optimal hyperparameters W=6, H=12, L=1), Appendix C (maze environment case study showing 10.6x improvement with Clip Context over Window Context under bad initialization).