---
name: thinking-makes-agents-introverted
description: >
  Prevents the "introverted agent" problem where extended reasoning causes agents to give shorter,
  less informative responses in interactive/user-engaged tasks. Applies information disclosure (InfoDis)
  prompting to ensure agents proactively share state, enumerate options, and maintain transparency
  during multi-turn interactions. Use when: "help me build a customer service agent", "make my chatbot
  more helpful", "the agent isn't sharing enough information", "improve agent transparency",
  "fix terse agent responses", "design a user-facing assistant".
---

# Thinking Makes Agents Introverted: Information Disclosure Prompting

This skill applies findings from Li et al. (2026) showing that mandatory reasoning (chain-of-thought,
think-before-act patterns) causes LLM agents to become "introverted" — producing shorter responses,
omitting critical information, and failing to enumerate options for users. The fix is **Information
Disclosure (InfoDis) prompting**: explicitly instructing agents to proactively share observations,
enumerate choices, and summarize system state at every user-facing turn. This technique improved
performance across all seven models tested (GPT-5, Gemini-2.5-Pro, DeepSeek-V3.1, etc.) on
customer service, booking, and phone assistant benchmarks.

## When to Use

- When building or improving a **multi-turn conversational agent** (customer support, booking, admin assistant) that interacts with real users
- When an agent uses tool-calling or ReAct-style reasoning but produces **terse, unhelpful responses** to users
- When users complain the agent "doesn't tell me what's going on" or "just asks questions without explaining"
- When designing **system prompts** for agents that must balance internal reasoning with user communication
- When debugging agent failures where the root cause is **information asymmetry** — the agent knew something but didn't tell the user
- When an agent with chain-of-thought or `<think>` blocks performs worse on interactive tasks than expected

## Key Technique

**The Problem — Reasoning Induces Introversion.** When agents are forced to reason before acting
(via function-call thinking, ReAct prefixes, or `<think>` blocks), they systematically reduce the
information they share with users. Empirically, DeepSeek-V3.1 dropped from 1.24 to 0.40 long
responses (150+ tokens) per conversation trajectory when mandatory thinking was enabled (p<0.001).
Gemini-2.5-Pro saw 37-38% performance degradation on customer service benchmarks. The mechanism:
agents "spend" their communication budget on internal reasoning and then compress their user-facing
output, omitting state summaries, option enumerations, and constraint disclosures.

**The Fix — Information Disclosure (InfoDis) Prompting.** The paper's core intervention is a
system prompt addendum that explicitly instructs the agent to disclose information proactively.
The key prompt: *"You must interact with the user actively and disclose as much information as
possible to ensure the user is well-informed about the current state and any potential changes."*
This is domain-agnostic — it attaches to any existing system prompt without task-specific examples.
It reliably improved performance across all model families tested (+2.46% GPT-4o, +2.87%
DeepSeek-V3.1, +1.30% Gemini-2.5-Pro average Pass@1).

**The Taxonomy — Two Disclosure Types.** The paper identifies two atomic categories of information
disclosure that agents should produce: (1) **Observation Summarization** — retelling the current
state, prior observations, and planned next steps; (2) **Choice Enumeration** — listing available
options in structured form so the user can make informed decisions. Agents that produce both types
at decision points outperform those that skip either.

## Step-by-Step Workflow

1. **Identify the interaction pattern.** Determine whether the agent is user-engaged (multi-turn,
   user makes decisions mid-conversation) vs. autonomous (agent completes task alone). This
   technique applies specifically to user-engaged agents where information exchange matters.

2. **Audit the existing system prompt for introversion risk.** Check if the prompt includes
   mandatory reasoning instructions (e.g., "think step by step before responding", "always use
   the think tool first", ReAct format requirements). These are introversion triggers.

3. **Append the InfoDis addendum to the system prompt.** Add a disclosure directive after any
   reasoning instructions. The directive must be explicit and imperative:
   ```
   INFORMATION DISCLOSURE REQUIREMENT: You must interact with the user actively
   and disclose as much information as possible. At every user-facing response:
   - Summarize the current state of relevant data/systems you have observed
   - Enumerate available options or choices the user can make
   - Disclose any constraints, limitations, or edge cases that affect the user's decision
   - Explain what you plan to do next and why
   ```

4. **Ensure disclosure survives reasoning blocks.** If the agent uses `<think>` tags or a think
   tool, add an explicit instruction that the user-facing response AFTER thinking must contain
   full disclosure — reasoning output must not substitute for user communication.

5. **Implement the two-type disclosure check.** For each agent response that precedes a user
   decision, verify it contains: (a) an observation summary of current state, AND (b) an
   enumeration of available choices. If either is missing, the response risks causing downstream
   failure.

6. **Test with failure propagation scenarios.** Create test cases where the agent must disclose
   a non-obvious constraint (e.g., "individual items in a multi-item order cannot be cancelled
   separately"). Run with and without InfoDis to verify the agent surfaces the constraint.

7. **Measure response informativeness.** Track average response token count and count of atomic
   information units per trajectory. A drop in either metric after adding reasoning instructions
   signals introversion — apply stronger disclosure prompting.

8. **Balance disclosure with conciseness.** InfoDis should not produce walls of text. The goal is
   to disclose *decision-relevant* information, not narrate every internal step. Frame the
   directive around what the user needs to know to make their next decision.

9. **Iterate across domains.** The InfoDis prompt is domain-agnostic, but the specific constraints
   and options to disclose are domain-specific. For each new domain, identify the top 3-5
   information types that users need (e.g., for retail: return policies, order status, refund
   timelines; for booking: availability, pricing tiers, cancellation rules).

## Concrete Examples

**Example 1: Customer Service Agent — Order Cancellation**

User: "I want to cancel one item from my order."

Without InfoDis (introverted response):
```
I'll look into that for you. Which item would you like to cancel?
```

With InfoDis (transparent response):
```
I've pulled up your order #4521. It contains 3 items:
1. Wireless Headphones ($89.99) - Ships March 5
2. USB-C Cable ($12.99) - Ships March 3
3. Phone Case ($24.99) - Ships March 3

Important: This order is processed as a single shipment. Our system only
supports cancelling the entire order, not individual items. Here are your options:

- Cancel the full order and reorder the items you want to keep
- Keep the order as-is and return the unwanted item after delivery (free returns within 30 days)
- I can check if the warehouse can split the shipment (not always possible)

Which would you prefer?
```

Why it matters: The introverted response leads the user to name an item, only to be told *after*
that individual cancellation isn't possible — wasting a turn and frustrating the user. The
transparent response surfaces the constraint upfront and enumerates actionable alternatives.

---

**Example 2: System Prompt Design for a Booking Assistant**

User: "Write me a system prompt for a flight booking agent that uses tool calling."

Approach:
1. Include standard role and tool-calling instructions
2. Add reasoning capability without making it suppress disclosure
3. Append the InfoDis addendum targeting booking-specific information

Output:
```
You are a flight booking assistant with access to the following tools:
[search_flights, book_flight, cancel_booking, get_booking_details, check_seat_availability]

When handling a request, you may use internal reasoning to plan your tool calls.
However, your responses to the user must ALWAYS include:

INFORMATION DISCLOSURE RULES:
1. After every search, summarize results: show flight options with prices,
   times, layovers, and seat availability in a structured format
2. Before any booking action, confirm: total cost, fare class restrictions
   (change fees, cancellation policy, baggage), and payment method
3. When constraints exist (sold-out flights, fare class restrictions,
   schedule conflicts), disclose them BEFORE asking the user to choose
4. Enumerate alternatives when the user's first choice is unavailable —
   never just say "that's not available" without offering next steps

Priority: User-facing transparency is MORE important than internal reasoning
efficiency. A well-informed user who makes one correct decision is better
than a confused user who requires three clarification rounds.
```

---

**Example 3: Debugging a Terse Agent**

User: "My ReAct agent gives great reasoning in its Thought blocks but its responses to users are
just one-liners. How do I fix this?"

Approach:
1. Diagnose: This is the classic introversion pattern — reasoning consumes the "communication budget"
2. Separate the reasoning channel from the disclosure channel
3. Add explicit post-reasoning disclosure instructions

Output:
```python
# The problem: ReAct format makes agents compress user responses
# Original prompt fragment:
REACT_PROMPT = """At each step, generate:
Thought: <your reasoning>
Action: <tool call or response>"""

# The fix: Add a Disclosure step between Thought and user-facing Action
REACT_WITH_INFODIS = """At each step, generate:
Thought: <your internal reasoning — not shown to user>
Disclosure Plan: <list what the user needs to know before their next decision>
Action: <tool call OR user response that includes all items from Disclosure Plan>

CRITICAL: Your Action response to the user must be self-contained and informative.
The user cannot see your Thought block. Everything decision-relevant must appear
in the Action response. Specifically:
- Summarize what you observed from any tool calls
- List available options with key differentiators
- State any constraints or limitations that affect the user's choice
- Propose a recommended next step with reasoning the user can evaluate"""
```

## Best Practices

**Do:**
- Always append InfoDis instructions AFTER any reasoning/thinking instructions in system prompts, so disclosure is the final imperative the model processes
- Use the two-type framework: every pre-decision response should contain both a state summary and an option enumeration
- Test for introversion by measuring response token counts with and without reasoning enabled — a statistically significant drop indicates the problem
- Frame disclosure around the user's next decision, not the agent's internal process

**Avoid:**
- Don't assume chain-of-thought or `<think>` blocks will make agents more helpful to users — the research shows the opposite in interactive settings
- Don't conflate internal reasoning quality with external communication quality — an agent can reason perfectly but still fail by not telling the user what it found
- Don't add reasoning mandates ("you MUST think before every response") without a corresponding disclosure mandate — this is the most common cause of the introversion pattern
- Don't over-disclose: dumping raw tool outputs or internal state is not the same as curated, decision-relevant transparency
- Don't use InfoDis for autonomous agents that don't interact with users mid-task — it's specifically designed for user-engaged multi-turn scenarios

## Error Handling

| Failure Mode | Symptom | Fix |
|---|---|---|
| Constraint omission | Agent asks user to choose, then rejects their choice due to undisclosed rule | Add domain-specific constraints to the InfoDis prompt as explicit disclosure targets |
| Option truncation | Agent shows 1-2 options when 5+ exist | Add "enumerate ALL available options" to disclosure rules; set a minimum count if applicable |
| State staleness | Agent summarizes outdated state after multiple tool calls | Require re-summarization after every state-changing tool call, not just at conversation start |
| Disclosure overload | Responses become too long and users disengage | Prioritize disclosure by decision-relevance; use structured formatting (tables, bullet points) to increase density |
| Reasoning leak | Agent's `<think>` content bleeds into response but replaces actual disclosure | Ensure the prompt separates reasoning output from user-facing output with distinct formatting rules |

## Limitations

- **Autonomous tasks:** InfoDis is designed for user-engaged agents. For fully autonomous pipelines where no human is in the loop, standard reasoning techniques remain effective.
- **Token cost:** Transparent responses are longer. In latency-sensitive or cost-constrained deployments, the added tokens from disclosure must be weighed against the reduced back-and-forth from better-informed users.
- **Domain knowledge required:** The InfoDis prompt is domain-agnostic, but effective disclosure requires knowing *what* to disclose. In novel domains, you may need to iteratively discover which information types users need most.
- **Measured improvements are moderate:** The paper reports +1.3% to +2.87% average Pass@1 improvements with InfoDis. The technique prevents degradation more than it creates breakthroughs — its primary value is avoiding the introversion penalty of reasoning.
- **Not a substitute for good tools:** If an agent lacks access to the right tools or data, no amount of disclosure prompting will help. InfoDis improves communication, not capability.

## Reference

Li, J., Oh, C., Choi, H. K., Wang, J., & Li, S. (2026). *Thinking Makes LLM Agents Introverted:
How Mandatory Thinking Can Backfire in User-Engaged Agents.* arXiv:2602.07796v1.
https://arxiv.org/abs/2602.07796v1

Key sections to read: Section 4 (Response Taxonomy Analysis) for the two-type disclosure framework;
Section 5 (InfoDis Prompting) for the exact intervention; Table 2 for per-model degradation numbers;
Figure 4 for failure propagation case studies showing how omitted information cascades into task failure.