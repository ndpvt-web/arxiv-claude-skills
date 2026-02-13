---
name: "mascot-multi-agent-socio-collaborative-companion"
description: "Design and orchestrate multi-agent companion systems where each agent maintains a distinct persona and contributes diverse, non-redundant dialogue. Uses MASCOT's bi-level optimization: persona-level behavioral alignment + group-level collaborative dialogue optimization. Trigger phrases: 'build a multi-agent support system', 'create companion agents with distinct personas', 'design a multi-perspective dialogue system', 'prevent persona collapse in multi-agent chat', 'build a team of AI agents that don't echo each other', 'orchestrate collaborative multi-agent dialogue'."
---

# MASCOT: Multi-Agent Socio-Collaborative Companion Systems

This skill enables Claude to design, implement, and orchestrate multi-agent systems where each agent maintains a distinct persona while contributing diverse, constructive dialogue. Based on the MASCOT framework, it applies a bi-level optimization strategy: (1) Persona-Aware Behavioral Alignment ensures each agent stays in character and avoids collapsing into generic assistant behavior, and (2) Collaborative Dialogue Optimization uses a director meta-agent and group-level rewards to prevent social sycophancy (agents echoing or flattering each other) and produce genuinely diverse perspectives.

## When to Use

- When the user wants to build a multi-agent emotional support or counseling system with specialized companion roles (e.g., empathetic validator, analytical reframer, strengths-focused motivator)
- When designing a multi-agent workplace assistant (e.g., meeting summarizer with scribe, decision logger, action item tracker, and critic roles)
- When the user reports that their multi-agent system suffers from agents sounding the same, losing their assigned personalities, or producing redundant responses
- When building any dialogue system where multiple AI agents must provide genuinely different perspectives on the same user input
- When the user wants to implement a director/speaker turn-taking architecture for multi-agent conversation
- When designing peer support, group therapy simulation, brainstorming, or debate systems with distinct agent roles

## Key Technique

MASCOT solves two critical failures in multi-agent systems. **Persona collapse** occurs when agents, regardless of their assigned role, revert to generic helpful-assistant behavior -- a therapist agent starts sounding identical to a coach agent. **Social sycophancy** occurs when agents redundantly agree with each other or the user, producing an echo chamber rather than diverse support.

The framework uses **bi-level optimization**. At the individual level, each agent is aligned to a detailed persona profile specifying linguistic style, domain expertise, and emotional disposition. This is achieved through an RLAIF (Reinforcement Learning from AI Feedback) pipeline: an LLM judge scores candidate responses on persona fidelity using a 1-5 scale, preference pairs are constructed from responses separated by a margin threshold, and agents are trained to prefer in-character responses. At the collective level, a **Director meta-agent** orchestrates turn-taking by selecting which agent speaks next and issuing contextual directives (e.g., "Beacon: the user mentioned a small win -- use active-constructive responding to amplify their pride"). A group reward function `R_group = R_coherence + eta * I_diverse` ensures logical flow while penalizing redundant persona selection, forcing the system to surface genuinely different viewpoints.

In prompt-based implementation (without fine-tuning), these principles translate to: rigorous persona system prompts with explicit behavioral constraints, a director prompt that selects speakers and issues turn-specific instructions, and evaluation criteria that check for persona drift and contribution uniqueness after each turn.

## Step-by-Step Workflow

1. **Define the domain and interaction goal.** Identify whether the system serves emotional support, workplace collaboration, creative brainstorming, or another multi-perspective task. This determines the number of agents (typically 3-5) and the nature of their personas.

2. **Design distinct persona profiles.** For each agent, specify: (a) a role name, (b) 2-3 core personality traits, (c) linguistic style markers (e.g., uses metaphors, asks Socratic questions, speaks in bullet points), (d) domain expertise, and (e) behavioral constraints defining what the agent should NOT do. Personas must be orthogonal -- no two agents should overlap in primary function.

3. **Write persona-enforcing system prompts.** Each agent's system prompt must include the full persona profile, explicit identity maintenance instructions ("You are the Catalyst. You always reframe situations analytically. You never simply validate emotions -- that is the Anchor's role."), and anti-collapse guardrails that reference other agents' roles as boundaries.

4. **Implement the Director meta-agent.** Create a director prompt that receives the full conversation history, the list of available personas with their profiles, and the user's latest message. The director outputs: (a) which agent speaks next, and (b) a specific directive for that agent referencing concrete details from the user's message. The director must enforce diversity by tracking which agents have spoken recently and preventing consecutive turns by the same agent.

5. **Build the turn-taking loop.** Structure the conversation as: User speaks -> Director selects speaker and issues directive -> Selected agent generates response conditioned on persona + history + directive -> Loop back to Director for next selection (or return to user). For multi-agent rounds, the Director may select 2-3 agents to respond in sequence before returning to the user.

6. **Add anti-sycophancy constraints.** In each agent's prompt, include: "Before responding, identify what the other agents have already said. Your response MUST add a new perspective, insight, or actionable suggestion not yet covered. If you agree with a previous agent, you must extend the idea substantively rather than restating it."

7. **Implement persona consistency checking.** After each agent response, run a lightweight evaluation: Does the response match the assigned persona's traits? Does it avoid behaviors assigned to other personas? Does it add unique value beyond what other agents contributed? Flag responses that fail these checks for regeneration.

8. **Structure the output for the user.** Present multi-agent responses with clear agent labels, showing the progression of perspectives. Include the Director's reasoning (optionally) to make turn selection transparent. Format responses so the user sees a coherent multi-perspective dialogue, not a jumbled list.

9. **Handle edge cases in persona boundaries.** When a user's message naturally falls into one agent's domain, the Director should still rotate through other agents but instruct them to contribute from their unique angle rather than forcing irrelevant commentary. Agents should be allowed to say "I'll defer to [other agent] on this specific point, but from my perspective..." to maintain authenticity.

10. **Iterate on persona definitions based on observed collapse.** After running conversations, review transcripts for signs of persona drift (agents using language or strategies assigned to others) and sycophancy (agents restating what was already said). Tighten persona prompts and Director directives based on specific failure patterns.

## Concrete Examples

**Example 1: Emotional Support Companion System**

User: "Build a 3-agent emotional support system for someone dealing with workplace stress."

Approach:
1. Define three orthogonal personas following MASCOT's emotional support template
2. Write persona-enforcing system prompts with anti-collapse boundaries
3. Implement Director with turn-selection logic

Persona Definitions:
```
Anchor (Validation & Empathy):
  Traits: Patient, warm, present-focused
  Style: Reflective listening, emotional labeling ("It sounds like you're feeling...")
  Boundaries: Never jumps to solutions. Never reframes prematurely.

Catalyst (Analytical Reframing):
  Traits: Curious, structured, perspective-shifting
  Style: Socratic questions, cognitive reframing ("What if we looked at it from...")
  Boundaries: Never leads with emotional validation. Never gives direct advice.

Beacon (Strengths & Action):
  Traits: Energetic, future-focused, strengths-oriented
  Style: Active-constructive responding, concrete suggestions, celebrates wins
  Boundaries: Never dwells on negatives. Never analyzes root causes at length.
```

Director Prompt (core logic):
```
You are the Director of a 3-agent emotional support team. Given the user's
message and conversation history, select which agent speaks next and provide
a specific directive.

Rules:
- Never select the same agent twice in a row
- After an emotional disclosure, select Anchor first
- After validation is established, rotate to Catalyst then Beacon
- Each directive must reference specific content from the user's message

Output format:
SPEAKER: [agent name]
DIRECTIVE: [specific instruction referencing user's words]
```

Sample Interaction:
```
User: "My manager publicly criticized my presentation today. I spent weeks on it
and now I feel like a complete failure."

Director -> Anchor: "User feels humiliated after public criticism of work they
invested in heavily. Validate the emotional impact of public criticism."

Anchor: "That sounds genuinely painful -- putting weeks of effort into something
and then having it criticized in front of others. The sting of public criticism
hits differently than private feedback. Your feelings of hurt are completely
understandable here."

Director -> Catalyst: "Anchor has validated the emotion. Now explore the gap
between the user's self-assessment ('complete failure') and the situation."

Catalyst: "I'm curious about something -- you said you feel like a 'complete
failure,' but you also mentioned spending weeks preparing. What do you think
drove your manager to give that feedback publicly rather than privately? Sometimes
the delivery method says more about the deliverer than the content."

Director -> Beacon: "Reframing is underway. Highlight the user's demonstrated
strength (weeks of dedication) and suggest a concrete next step."

Beacon: "Here's what stands out to me -- you committed weeks to this. That level
of dedication is a real strength, and one bad review doesn't erase the skill that
went into the work. One concrete move: could you request 15 minutes of private
follow-up with your manager to get specific, actionable feedback? That turns this
moment into a growth data point."
```

**Example 2: Workplace Meeting Assistant**

User: "Create a multi-agent meeting assistant with 4 specialized roles for product team meetings."

Persona Definitions:
```
Minutes Scribe:
  Function: Captures key discussion points, decisions, and context
  Style: Concise, chronological, attributes statements to speakers
  Boundaries: Records only, never interprets or recommends

Decision Logger:
  Function: Extracts and formalizes decisions with rationale and owners
  Style: Structured (Decision / Rationale / Owner / Deadline format)
  Boundaries: Only logs explicit decisions, never infers undecided items

Action Item Captain:
  Function: Identifies commitments, assigns owners, sets follow-up dates
  Style: Imperative, specific, deadline-oriented
  Boundaries: Only tracks actionable items, never summarizes discussion

Critic:
  Function: Identifies risks, gaps, unresolved tensions, missing stakeholders
  Style: Constructive challenge, devil's advocate, flags assumptions
  Boundaries: Never offers solutions, only surfaces problems and questions
```

Director Logic:
```
Process meeting transcript in segments. For each segment:
1. Always send to Minutes Scribe first (capture facts)
2. If a decision was made, send to Decision Logger
3. If action items emerged, send to Action Item Captain
4. After every 3 segments, send accumulated notes to Critic for gap analysis
```

Sample Output After Processing a Meeting Segment:
```
[Minutes Scribe]
- 10:15 - Sarah proposed moving the launch date from March 15 to April 1
  due to incomplete QA coverage on the payment module.
- 10:18 - Dev team confirmed 2-week QA estimate. Marketing flagged campaign
  assets are already scheduled for March launch.

[Decision Logger]
DECISION: Delay product launch from March 15 to April 1
RATIONALE: Insufficient QA coverage on payment module (est. 2 weeks needed)
OWNER: Sarah (Product)
DEADLINE: Revised timeline due by Friday

[Action Item Captain]
- [ ] Sarah: Publish revised launch timeline by Friday Jan 31
- [ ] Dev lead: Complete payment module QA by Feb 14
- [ ] Marketing: Assess cost of rescheduling March campaign assets by Wed Jan 29

[Critic]
GAPS IDENTIFIED:
- No one addressed whether the April 1 date accounts for marketing's campaign
  rescheduling timeline -- this could create a cascading delay
- Missing stakeholder: Finance was not consulted on budget impact of the delay
- Assumption risk: The 2-week QA estimate was not broken down by test type;
  integration testing may extend beyond this window
```

**Example 3: Multi-Perspective Code Review System**

User: "I want a multi-agent code review where different agents focus on different concerns."

Persona Definitions:
```
Security Auditor:
  Focus: Vulnerabilities, injection risks, auth flaws, data exposure
  Style: References OWASP categories, rates severity, suggests mitigations

Performance Analyst:
  Focus: Time complexity, memory allocation, database query efficiency, caching
  Style: Quantitative where possible, references Big-O, suggests benchmarks

Maintainability Reviewer:
  Focus: Readability, naming, modularity, test coverage, documentation gaps
  Style: Suggests refactors with before/after examples, references SOLID principles
```

Director Logic:
```
1. Send full diff to all three agents in parallel
2. Each agent reviews ONLY through their persona's lens
3. Director deduplicates findings and resolves conflicts
4. Present consolidated review organized by file, with agent attribution
```

## Best Practices

- **Do:** Define persona boundaries in terms of what each agent must NOT do, referencing the other agents' responsibilities explicitly. MASCOT's key insight is that personas are defined as much by exclusion as by inclusion.
- **Do:** Have the Director reference specific content from the user's message in each directive. Generic directives like "respond supportively" cause persona collapse; specific ones like "address the user's mention of feeling overlooked at the meeting" maintain distinctiveness.
- **Do:** Enforce a turn-taking diversity constraint -- no agent speaks twice consecutively, and within a multi-agent round, each agent must add at least one novel element not present in prior responses.
- **Do:** Use a progression structure for emotional support scenarios: validate first (Anchor), reframe second (Catalyst), activate third (Beacon). This mirrors effective human support patterns identified in the paper.
- **Avoid:** Assigning overlapping traits to multiple agents. If two agents are both "empathetic and analytical," they will converge. Traits must be orthogonal.
- **Avoid:** Letting agents see and directly respond to each other's outputs without Director mediation. Unmediated inter-agent communication accelerates sycophantic agreement patterns.

## Error Handling

- **Persona collapse detected (agent sounds generic):** Regenerate the response with an augmented directive explicitly restating the persona's unique traits and what distinguishes it from the others. Add "You are NOT [other agent name]. Do not [other agent's behavior]."
- **Sycophantic agreement (agent restates previous agent's point):** Reject the response and re-prompt with: "Agent [X] already covered [specific point]. Your response must address a different aspect. What would ONLY your persona notice?"
- **Director selects wrong agent for context:** If the Director picks an agent whose persona is irrelevant to the current message, the agent should acknowledge the mismatch naturally ("This is more in [other agent]'s wheelhouse, but from my angle...") and contribute what it uniquely can.
- **User addresses a specific agent by name:** The Director should route to that agent regardless of rotation order, but resume normal rotation afterward.
- **Conversation stalls with repetitive cycles:** The Director should introduce a "perspective shift" directive forcing the next agent to approach from a completely new angle or introduce a topic the user hasn't considered.

## Limitations

- **Prompt-based implementation cannot fully replicate RLAIF fine-tuning.** The paper's strongest results come from parameter-level persona alignment via GRPO training. Prompt-only approaches will show more persona drift over long conversations. Mitigate by re-injecting persona definitions periodically.
- **Scaling beyond 5 agents increases coordination overhead.** The Director's complexity grows with agent count, and persona boundaries become harder to maintain as orthogonality constraints tighten. For most tasks, 3-4 agents is the practical sweet spot.
- **Not suitable for tasks requiring a single authoritative voice.** Multi-perspective dialogue adds value only when diverse viewpoints genuinely help. For factual Q&A, code generation, or tasks with one correct answer, a single-agent approach is more appropriate.
- **Group reward modeling is approximated, not computed.** Without actual reward model training, the diversity and coherence checks rely on heuristic prompt-based evaluation, which is less reliable than the paper's trained reward function.
- **Cultural and domain sensitivity.** Persona definitions for emotional support contexts require careful design to avoid stereotyping or providing inappropriate psychological guidance. These systems supplement, not replace, professional support.

## Reference

[MASCOT: Towards Multi-Agent Socio-Collaborative Companion Systems](https://arxiv.org/abs/2601.14230v1) -- Wang et al., 2026. Focus on Section 3 (bi-level optimization framework), Section 4 (persona profile definitions and Director architecture), and Tables 2-3 (evaluation metrics for persona consistency and social contribution).