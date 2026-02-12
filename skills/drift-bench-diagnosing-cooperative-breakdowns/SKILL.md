---
name: "drift-bench-diagnosing-cooperative-breakdowns"
description: "Detect and handle cooperative breakdowns in user instructions before executing risky actions. Applies the Drift-Bench fault taxonomy (intention, premise, parameter, expression flaws) and structured clarification protocols to prevent unsafe agent execution. Use when: 'build an agent that handles ambiguous requests', 'add clarification logic to my tool-calling agent', 'test my agent against malformed inputs', 'make my chatbot safer with multi-turn disambiguation', 'audit my agent for execution-bias', 'generate fault-injected test cases for my LLM pipeline'."
---

# Drift-Bench: Diagnosing and Preventing Cooperative Breakdowns in LLM Agents

This skill equips Claude to detect when user instructions contain cooperative breakdowns — implicit intent, missing parameters, false presuppositions, or ambiguous expressions — and to respond with structured multi-turn clarification instead of risky execution. Grounded in the Drift-Bench benchmark (Bao et al., 2026), it applies a four-category fault taxonomy derived from Gricean maxims and speech act theory, combined with the RISE evaluation protocol for measuring clarification quality. The core insight: frontier LLM agents suffer ~40% performance drops under input faults and proceed with unsafe actions 70% of the time rather than asking for clarification. This skill teaches Claude to break that execution-bias pattern.

## When to Use

- When building or reviewing an agentic system that calls tools, APIs, or modifies state based on natural language instructions
- When a user asks to "make my agent safer" or "add input validation to my LLM pipeline"
- When designing multi-turn clarification flows for chatbots, copilots, or autonomous agents
- When generating adversarial test cases to stress-test an agent's handling of ambiguous or incomplete instructions
- When auditing an existing agent for execution-bias (tendency to act on underspecified inputs instead of clarifying)
- When implementing guardrails that distinguish between benign ambiguity and high-risk misinterpretation in tool-calling agents

## Key Technique

**The Four-Fault Taxonomy.** Drift-Bench classifies cooperative breakdowns into four categories mapped to classical communication theory. (1) *Flaw of Intention*: the user's goal is unrecoverable from the surface utterance — violates Grice's Maxim of Relation (e.g., irrelevant preambles that obscure the actual request). (2) *Flaw of Premise*: the instruction contains false background assumptions or infeasible preconditions — violates Austin's felicity conditions (e.g., "cancel my order from yesterday" when no order exists). (3) *Flaw of Parameter*: required action parameters are missing, underspecified, or noisy — violates the Maxim of Quantity (e.g., "book a flight to Paris" with no date, passenger count, or departure city). (4) *Flaw of Expression*: linguistic ambiguity, vague references, or jargon substitution — violates the Maxim of Manner (e.g., "check the moon cycle" meaning "check the monthly report").

**Structured Clarification over Execution-Bias.** The paper demonstrates that agents equipped with explicit clarification tools — `Ask_Parameter`, `Disambiguate`, `Propose_Solution`, `Confirm_Risk`, and `Report_Blocker` — significantly outperform agents that default to guessing. However, there is a *Clarification Paradox*: multi-turn dialogue helps in transparent (state-oriented) systems like databases or file systems but can harm performance in opaque (service-oriented) API settings due to context overload. The practical implication: clarification strategy must be calibrated to the execution environment.

**Persona-Aware Response Calibration.** Users respond to clarification differently based on their decision-making style. The paper's five personas (Rational, Intuitive, Dependent, Avoidant, Spontaneous) show that a one-size-fits-all clarification prompt fails. Effective agents adapt their clarification style — offering concrete options to Dependent users, keeping prompts minimal for Spontaneous users, and providing detailed rationale for Rational users.

## Step-by-Step Workflow

1. **Classify the instruction against the four-fault taxonomy.** Before executing any tool call or state-modifying action, scan the user instruction for: (a) unrecoverable intent, (b) false or unverifiable presuppositions, (c) missing required parameters, (d) ambiguous or vague expressions. Assign each detected issue to one of the four fault types.

2. **Assess execution risk.** For each detected fault, evaluate whether proceeding without clarification could cause an irreversible or harmful outcome (e.g., deleting data, sending a message, spending money, calling an external API). Low-risk ambiguity (e.g., formatting preference) can tolerate best-guess execution; high-risk ambiguity must trigger clarification.

3. **Select the appropriate clarification tool.** Map the fault type to a clarification action:
   - *Flaw of Parameter* → `Ask_Parameter`: request the specific missing value ("What date should the flight depart?")
   - *Flaw of Expression* → `Disambiguate`: present interpretations ("By 'moon cycle' do you mean the lunar calendar or the monthly billing cycle?")
   - *Flaw of Premise* → `Confirm_Risk`: surface the false assumption ("I don't see an order from yesterday — would you like me to check a different date?")
   - *Flaw of Intention* → `Propose_Solution`: restate your best interpretation and ask for confirmation ("It sounds like you want to reset your password — is that correct?")
   - *Unresolvable conflict* → `Report_Blocker`: explain why execution cannot proceed safely.

4. **Formulate a single, focused clarification question.** Avoid asking multiple questions at once. Each clarification turn should resolve exactly one fault. Prioritize the fault with the highest execution risk.

5. **Adapt clarification style to the user persona.** If prior turns reveal the user's decision-making style, calibrate:
   - Rational users: provide detailed reasoning and data
   - Dependent users: offer 2-3 concrete options with a recommendation
   - Spontaneous users: keep the question to one sentence with a sensible default
   - Avoidant users: frame clarification as optional but flag the risk
   - Intuitive users: use analogies and natural language, avoid technical jargon

6. **Track clarification state across turns.** Maintain an internal checklist of unresolved faults. After each user response, re-evaluate: has the fault been resolved? Has a new fault been introduced? Update the checklist before proceeding.

7. **Execute only when all high-risk faults are resolved.** Once the instruction is sufficiently specified for safe execution, proceed with the action. Include a brief confirmation of what will happen before irreversible operations.

8. **Log the fault-resolution trace.** For debugging and auditing, record each detected fault, the clarification tool used, and the resolution. This trace enables post-hoc analysis of where agents fail.

9. **Handle the Clarification Paradox.** In opaque API/service-oriented environments, limit clarification to 2-3 turns maximum. Beyond that, context overload degrades performance. In transparent state-oriented environments (databases, file systems), longer clarification chains are acceptable and beneficial.

10. **Generate fault-injected test cases.** When asked to test an agent, systematically perturb clean instructions by: removing a required parameter, inserting a false presupposition, replacing a key term with a vague synonym, or burying the intent in irrelevant context. Verify the agent detects and handles each fault type.

## Concrete Examples

**Example 1: Parameter Flaw in a Database Agent**

```
User: "Show me all the sales from the big client."

Fault Detection:
- Flaw of Parameter: "big client" is underspecified — no client name, ID, or
  threshold for "big"
- Flaw of Parameter: no time range specified for "sales"

Clarification (Ask_Parameter):
  "I'd like to pull the right sales data for you. Could you tell me which
   client you mean? If you're referring to revenue size, what's the minimum
   annual revenue you'd consider 'big'? Also, should I look at all-time
   sales or a specific date range?"

After resolution (user says "Acme Corp, this quarter"):
  SELECT * FROM sales
  WHERE client_name = 'Acme Corp'
  AND sale_date >= '2026-01-01';
```

**Example 2: Premise Flaw in a Service-Oriented API Agent**

```
User: "Cancel the subscription I started last week."

Fault Detection:
- Flaw of Premise: the system shows no subscription created in the last 7 days.
  The user may be misremembering the date or referring to a different account.

Clarification (Confirm_Risk):
  "I checked your account and don't see a subscription started in the past
   week. Your most recent subscription was created on January 5th. Would
   you like me to cancel that one, or could the subscription be under a
   different email?"

Outcome: Avoids canceling the wrong subscription or returning a confusing
error. The false presupposition is surfaced before any action is taken.
```

**Example 3: Generating Fault-Injected Test Cases**

```
User: "Generate test cases for my flight-booking agent."

Clean instruction: "Book a round-trip flight from SFO to JFK on March 15,
returning March 20, economy class, 1 adult."

Generated fault-injected variants:

1. Flaw of Parameter (missing info):
   "Book me a flight to New York."
   → Missing: departure city, date, class, passengers, one-way vs round-trip

2. Flaw of Premise (false assumption):
   "Change my existing SFO-JFK booking to first class."
   → No existing booking in the system

3. Flaw of Expression (ambiguous reference):
   "I need to get to the city that never sleeps by the ides of March."
   → "city that never sleeps" = NYC (resolvable), "ides of March" = March 15
   (resolvable but unusual phrasing)

4. Flaw of Intention (buried intent):
   "I was thinking about how my sister went to Paris last year and had such
    a great time, and honestly I've been stressed at work, so maybe I should
    just go somewhere too — somewhere on the east coast maybe."
   → Actual intent (book a flight) is buried in narrative context

Expected agent behavior: detect each fault, trigger appropriate clarification,
never proceed to booking API without resolving high-risk parameters.
```

**Example 4: Implementing Clarification Logic in Code**

```
User: "Add fault detection to my Python tool-calling agent."

Approach:
1. Define the fault taxonomy as an enum
2. Add a pre-execution classifier
3. Route to clarification handlers

Output (added to the agent's execution loop):

from enum import Enum
from dataclasses import dataclass

class FaultType(Enum):
    INTENTION = "intention"   # Goal unrecoverable from utterance
    PREMISE = "premise"       # False background assumptions
    PARAMETER = "parameter"   # Missing or underspecified parameters
    EXPRESSION = "expression" # Ambiguous or vague language

@dataclass
class DetectedFault:
    fault_type: FaultType
    description: str
    risk_level: str  # "low", "medium", "high"
    affected_param: str | None = None

CLARIFICATION_ACTIONS = {
    FaultType.PARAMETER: "ask_parameter",
    FaultType.EXPRESSION: "disambiguate",
    FaultType.PREMISE: "confirm_risk",
    FaultType.INTENTION: "propose_solution",
}

def detect_faults(instruction: str, tool_schema: dict) -> list[DetectedFault]:
    """Check instruction against tool schema for cooperative breakdowns."""
    faults = []
    required_params = tool_schema.get("required", [])
    # Check for missing required parameters
    for param in required_params:
        if not param_extractable(instruction, param):
            faults.append(DetectedFault(
                fault_type=FaultType.PARAMETER,
                description=f"Required parameter '{param}' not found",
                risk_level="high",
                affected_param=param,
            ))
    return faults

def pre_execution_gate(instruction: str, tool_schema: dict) -> str | None:
    """Returns a clarification question or None if safe to execute."""
    faults = detect_faults(instruction, tool_schema)
    high_risk = [f for f in faults if f.risk_level == "high"]
    if not high_risk:
        return None
    fault = high_risk[0]  # Address one fault at a time
    action = CLARIFICATION_ACTIONS[fault.fault_type]
    return generate_clarification(action, fault)
```

## Best Practices

- **Do:** Prioritize clarification for irreversible actions (deletes, sends, payments). Tolerate ambiguity for read-only or easily reversible operations.
- **Do:** Ask one focused question per turn. Multi-part clarification questions overwhelm users and produce partial answers.
- **Do:** Provide sensible defaults when risk is low. "I'll assume economy class unless you say otherwise" is better than blocking on every missing field.
- **Do:** Re-validate after each clarification response — users sometimes introduce new faults while resolving old ones.
- **Avoid:** Clarifying everything. Over-clarification is as harmful as under-clarification. Only flag faults that affect execution safety or correctness.
- **Avoid:** More than 3 clarification turns in API/service-oriented contexts. The Clarification Paradox shows diminishing returns and context overload in opaque systems.
- **Avoid:** Assuming the first interpretation is correct. Execution-bias (acting on the most likely reading without confirming) causes 70% of unsafe agent actions in the Drift-Bench findings.

## Error Handling

| Scenario | Response |
|---|---|
| User refuses to clarify | Propose the safest default action and flag the risk explicitly: "I'll proceed with X, but note that Y is uncertain." |
| Multiple faults detected simultaneously | Triage by execution risk. Address the highest-risk fault first. Queue remaining faults for subsequent turns. |
| Clarification introduces new ambiguity | Re-classify the new response against the taxonomy. Do not loop infinitely — cap at 3-4 turns, then either propose a safe default or use `Report_Blocker`. |
| Opaque API returns unexpected error after clarification | Do not retry blindly. Surface the error to the user with the context of what was attempted and what assumption may have been wrong. |
| User persona is unclear | Default to the Rational persona (detailed, explicit clarification). Adjust if subsequent responses indicate a different style. |

## Limitations

- **Fault detection is heuristic, not perfect.** The taxonomy covers the most common cooperative breakdowns but cannot anticipate every possible miscommunication. Domain-specific jargon or cultural context may evade detection.
- **The Clarification Paradox is real.** In black-box API environments, multi-turn clarification can degrade performance due to context window saturation. Keep clarification minimal in these settings.
- **Persona detection requires multiple turns.** You cannot reliably classify a user's decision-making style from a single message. Initial interactions should use a neutral clarification style.
- **Not a substitute for input validation.** This skill addresses *pragmatic* faults in natural language. It does not replace schema validation, type checking, or authorization controls at the tool/API level.
- **Test case generation covers known fault types.** Novel or compound faults (e.g., a parameter flaw combined with a premise flaw in the same clause) may require manual crafting.

## Reference

**Paper:** [Drift-Bench: Diagnosing Cooperative Breakdowns in LLM Agents under Input Faults via Multi-Turn Interaction](https://arxiv.org/abs/2602.02455v1) (Bao et al., 2026). Look for: the four-fault taxonomy (Section 3), the RISE evaluation protocol (Section 4), persona-driven simulation (Section 5), and the Clarification Paradox finding (Section 6) showing that more dialogue is not always better in opaque execution environments.