---
name: "cua-skill-develop-skills-computer"
description: "Build reusable, parameterized skill libraries for computer-using agents (CUAs). Decomposes GUI automation into Skill Cells (intent), Parameterized Execution Graphs (actions), and Skill Composition Graphs (chaining). Use when: 'build a skill library for desktop automation', 'create reusable GUI action primitives', 'design a computer-using agent with skill retrieval', 'structure browser automation as composable skills', 'add failure recovery to my GUI agent', 'make my automation agent learn from past failures'."
---

# CUA-Skill: Build Reusable Skill Libraries for Computer-Using Agents

This skill teaches Claude to design and implement structured, reusable skill libraries for autonomous agents that operate computer interfaces (browsers, desktops, web apps). Based on Microsoft's CUA-Skill framework, the core idea is to decompose GUI automation into three layers: a **Skill Cell** capturing user intent, a **Parameterized Execution Graph** encoding concrete GUI actions with placeholder arguments, and a **Skill Composition Graph** defining how skills chain together. This architecture replaces brittle, monolithic automation scripts with modular, retrieval-friendly skill units that an LLM planner can dynamically select, parameterize, and compose at runtime.

## When to Use

- When the user wants to build a desktop, browser, or GUI automation agent that needs reusable action primitives rather than one-off scripts
- When designing a skill/tool library for an LLM-based agent that interacts with application UIs (Playwright, Puppeteer, pyautogui, etc.)
- When the user asks to make GUI automation more robust by adding failure recovery and retry logic
- When structuring a large set of automation routines into a composable, searchable library
- When building an agent that must dynamically select which automation skill to run based on natural-language instructions and current screen state
- When the user wants to add memory-aware failure recovery so an agent stops repeating the same broken action sequence

## Key Technique

**The Three-Layer Skill Abstraction.** CUA-Skill decomposes every automation capability into three components. The **Skill Cell** is a metadata envelope capturing the skill's name, natural-language intent description, preconditions (what screen state must hold before execution), postconditions (what should be true after), and a typed argument schema with placeholders. The **Parameterized Execution Graph (PEG)** is an ordered sequence of concrete GUI actions (click, type, scroll, wait, assert) where each action references elements via locators and binds values from the argument schema. The PEG supports conditional branching so the same skill can handle variant UI states. The **Skill Composition Graph (SCG)** encodes how skills chain: which skills must complete before others, how outputs flow as inputs to downstream skills, and what alternative paths exist when a skill fails.

**Dynamic Retrieval and Argument Instantiation.** At runtime, an LLM planner receives the user's task and current UI state (screenshot or DOM), then retrieves the best-matching skill from the library using semantic similarity on intent descriptions. The planner instantiates the skill's argument placeholders using context from the task instruction and observed UI elements. This separation of skill definition from argument binding means skills are written once and reused across many tasks with different parameters.

**Memory-Aware Failure Recovery.** The agent maintains an execution log recording each skill invocation, its parameters, and whether it succeeded or failed. When a postcondition check fails, the agent consults this log to avoid retrying the exact same parameters. It can backtrack to a known-good state, try alternative argument values, or escalate to a different skill entirely. This persistent memory prevents infinite retry loops -- the most common failure mode in naive GUI agents.

## Step-by-Step Workflow

1. **Audit the target application's UI surface.** Enumerate the screens, dialogs, and workflows the agent must handle. Group related actions into functional clusters (e.g., "file management," "form submission," "navigation"). Each cluster becomes a candidate skill.

2. **Define Skill Cells for each capability.** For every skill, write a JSON/YAML metadata block containing: `name` (kebab-case identifier), `intent` (1-2 sentence natural-language description), `preconditions` (list of UI state assertions that must hold), `postconditions` (list of expected outcomes), and `args` (typed argument schema with descriptions and defaults).

3. **Build the Parameterized Execution Graph (PEG).** For each skill, define the ordered sequence of GUI actions. Each action specifies: `action_type` (click, type, scroll, select, wait, assert), `locator` (CSS selector, XPath, accessibility label, or coordinates), `value` (literal or `{{arg_name}}` placeholder), and optional `condition` (branch predicate based on screen state). Keep PEGs as short as possible -- a skill should do one coherent thing.

4. **Design the Skill Composition Graph (SCG).** Define edges between skills: `depends_on` (skill B requires skill A to complete first), `data_flow` (skill A's output field maps to skill B's input arg), and `fallback` (if skill A fails, try skill C instead). Store this as an adjacency list or DAG definition.

5. **Implement a skill registry with semantic search.** Store all Skill Cells in a searchable index. At minimum, embed each skill's `intent` field using a text embedding model and support nearest-neighbor retrieval. For smaller libraries (<100 skills), keyword matching on intent + preconditions is sufficient.

6. **Build the planner loop.** The LLM planner takes as input: (a) the user's task instruction, (b) the current UI state (screenshot description or DOM snapshot), (c) the execution history so far. It outputs: the skill to invoke and concrete argument values. Prompt the planner to reason about which preconditions are currently satisfied to select the right skill.

7. **Implement argument instantiation.** Parse the selected skill's `args` schema, then fill each placeholder from the planner's output. Validate types and required fields before execution. Reject and re-prompt the planner if instantiation fails.

8. **Execute the PEG with postcondition checking.** Run each action in the PEG sequentially, substituting `{{arg_name}}` placeholders with instantiated values. After the final action, evaluate all postconditions. If any postcondition fails, mark the execution as failed and log the full context.

9. **Apply memory-aware failure recovery.** On failure, append `{skill_name, args, failure_reason, ui_state_snapshot}` to the failure log. Before the planner selects the next action, inject recent failures into its context with the instruction: "The following attempts failed -- do not repeat them. Choose a different approach or different parameters." If the same skill has failed 2+ times, force the planner to select an alternative skill or escalate.

10. **Iterate and expand the skill library.** After initial deployment, review execution logs to find: (a) tasks where no skill matched (gaps to fill), (b) skills that frequently fail (need refinement), (c) recurring multi-skill sequences that should become a single composite skill. Add new skills to the registry and update composition edges.

## Concrete Examples

**Example 1: Building a browser automation skill library**

User: "I'm building a Playwright-based agent that helps users manage their email. Create a skill library structure for common Gmail actions."

Approach:
1. Identify core Gmail capabilities: compose email, reply, search, archive, label, attach file, delete.
2. Define each as a Skill Cell with parameterized arguments.
3. Wire them together via a Composition Graph.

Output structure:
```yaml
# skills/compose-email.yaml
name: compose-email
intent: "Compose and send a new email to specified recipients with subject and body"
preconditions:
  - "Gmail inbox or any Gmail page is loaded"
postconditions:
  - "Sent confirmation banner is visible OR draft is saved"
args:
  to:
    type: string
    required: true
    description: "Recipient email address(es), comma-separated"
  subject:
    type: string
    required: true
    description: "Email subject line"
  body:
    type: string
    required: true
    description: "Email body text"
  send:
    type: boolean
    default: true
    description: "Whether to send immediately or save as draft"

peg:
  - action: click
    locator: "[gh='cm'] div.T-I.T-I-KE"  # Compose button
    wait_after: 1000
  - action: type
    locator: "input[aria-label='To recipients']"
    value: "{{to}}"
  - action: type
    locator: "input[name='subjectbox']"
    value: "{{subject}}"
  - action: type
    locator: "div[aria-label='Message Body']"
    value: "{{body}}"
  - action: click
    condition: "{{send}} == true"
    locator: "div[aria-label='Send']"
  - action: assert
    condition: "{{send}} == true"
    locator: "span.bAq"  # "Message sent" banner
    timeout: 5000
```

```yaml
# composition/email-with-attachment.yaml
name: email-with-attachment
skills:
  - id: compose
    skill: compose-email
    args:
      to: "{{to}}"
      subject: "{{subject}}"
      body: "{{body}}"
      send: false  # Don't send yet
  - id: attach
    skill: attach-file
    depends_on: [compose]
    args:
      file_path: "{{attachment_path}}"
  - id: send
    skill: send-draft
    depends_on: [attach]
    fallback: save-draft  # If send fails, at least save
```

**Example 2: Adding failure recovery to an existing automation agent**

User: "My Puppeteer agent keeps retrying the same failed login action in a loop. How do I add CUA-Skill style failure recovery?"

Approach:
1. Introduce an execution log that tracks attempts.
2. Inject failure context into the planner prompt.
3. Implement backoff and alternative-skill selection.

Output:
```typescript
interface ExecutionRecord {
  skillName: string;
  args: Record<string, unknown>;
  timestamp: number;
  success: boolean;
  failureReason?: string;
  uiStateSnapshot?: string;
}

class FailureMemory {
  private log: ExecutionRecord[] = [];
  private readonly MAX_RETRIES = 2;

  record(entry: ExecutionRecord): void {
    this.log.push(entry);
  }

  getFailuresForSkill(skillName: string): ExecutionRecord[] {
    return this.log.filter(e => e.skillName === skillName && !e.success);
  }

  shouldBlockSkill(skillName: string, args: Record<string, unknown>): boolean {
    const failures = this.getFailuresForSkill(skillName);
    const sameArgFailures = failures.filter(
      f => JSON.stringify(f.args) === JSON.stringify(args)
    );
    return sameArgFailures.length >= this.MAX_RETRIES;
  }

  buildPlannerContext(): string {
    const recentFailures = this.log
      .filter(e => !e.success)
      .slice(-5);
    if (recentFailures.length === 0) return "";
    return [
      "## Recent Failures (do NOT repeat these exact approaches):",
      ...recentFailures.map(f =>
        `- Skill "${f.skillName}" with args ${JSON.stringify(f.args)}: ${f.failureReason}`
      ),
      "Choose different parameters or a different skill entirely."
    ].join("\n");
  }
}

// In the planner loop:
async function planAndExecute(task: string, memory: FailureMemory) {
  const uiState = await captureScreenState();
  const failureContext = memory.buildPlannerContext();

  const plannerResponse = await llm.complete({
    prompt: `Task: ${task}\nUI State: ${uiState}\n${failureContext}\nSelect a skill and provide arguments.`
  });

  const { skillName, args } = parsePlannerOutput(plannerResponse);

  if (memory.shouldBlockSkill(skillName, args)) {
    // Force re-planning with explicit block
    return planAndExecute(task, memory); // Planner sees updated context
  }

  const result = await executeSkill(skillName, args);
  memory.record({ skillName, args, timestamp: Date.now(), ...result });
  return result;
}
```

**Example 3: Designing a skill registry with retrieval**

User: "I have 50 automation skills defined as YAML files. How do I build a retrieval system so my agent can find the right skill?"

Approach:
1. Load and index all skill intent descriptions.
2. Implement semantic + keyword hybrid search.
3. Filter candidates by precondition matching.

Output:
```python
import json
from pathlib import Path
from dataclasses import dataclass
from sentence_transformers import SentenceTransformer
import numpy as np

@dataclass
class SkillCell:
    name: str
    intent: str
    preconditions: list[str]
    postconditions: list[str]
    args: dict
    peg: list[dict]

class SkillRegistry:
    def __init__(self, skills_dir: str):
        self.model = SentenceTransformer("all-MiniLM-L6-v2")
        self.skills: list[SkillCell] = self._load_skills(skills_dir)
        self.embeddings = self.model.encode(
            [s.intent for s in self.skills]
        )

    def retrieve(self, query: str, ui_state: dict, top_k: int = 5) -> list[SkillCell]:
        # Semantic retrieval on intent
        q_emb = self.model.encode([query])
        scores = np.dot(self.embeddings, q_emb.T).flatten()
        top_indices = np.argsort(scores)[::-1][:top_k * 2]
        candidates = [self.skills[i] for i in top_indices]

        # Filter by precondition satisfaction
        viable = []
        for skill in candidates:
            if self._preconditions_met(skill.preconditions, ui_state):
                viable.append(skill)
            if len(viable) >= top_k:
                break
        return viable

    def _preconditions_met(self, preconditions: list[str], ui_state: dict) -> bool:
        # Check each precondition against current UI state
        for pre in preconditions:
            if not self._evaluate_condition(pre, ui_state):
                return False
        return True
```

## Best Practices

- **Do:** Keep skills atomic -- one skill should accomplish one coherent user-visible outcome (e.g., "send email," not "manage email"). Compose complex workflows from multiple atomic skills via the SCG.
- **Do:** Write intent descriptions in plain language as if explaining the action to a non-technical user. This improves retrieval accuracy when the planner searches for skills.
- **Do:** Include explicit postcondition assertions (element visible, URL changed, text appeared) so failures are detected immediately rather than cascading silently.
- **Do:** Log every execution attempt with full context (skill name, args, UI state, outcome). This log is the foundation for failure recovery and skill improvement.
- **Avoid:** Hardcoding pixel coordinates or brittle selectors in PEGs. Use semantic locators (aria-labels, data-testid, role attributes) that survive UI layout changes.
- **Avoid:** Creating skills that depend on timing (sleep/wait with fixed durations). Use explicit wait-for-condition assertions instead. Fixed waits are the top cause of flaky automation.
- **Avoid:** Allowing the planner to retry the same skill with identical arguments more than twice. After two failures with the same parameters, force alternative skill selection or escalation.

## Error Handling

| Failure Mode | Detection | Recovery Strategy |
|---|---|---|
| Skill not found | Retrieval returns empty or low-confidence matches | Fall back to raw LLM planning without skill library; log the gap for future skill creation |
| Precondition not met | UI state check fails before PEG execution | Navigate to required state first (compose a "navigate-to" skill), or select a different skill whose preconditions match |
| Action element not found | Locator timeout during PEG execution | Retry with fallback locators if defined; capture screenshot and ask planner to identify the element |
| Postcondition not met | Assert fails after PEG completes | Log failure with full context; planner re-selects with failure memory injected |
| Argument instantiation fails | Type validation or required field missing | Re-prompt the planner with the argument schema and validation error message |
| Infinite retry loop | Same skill+args attempted 3+ times | Hard block that skill+args combination; force alternative approach or surface error to user |

## Limitations

- **Requires upfront skill authoring.** The library must be manually built before the agent can use it. There is no automatic skill generation from demonstrations in this framework -- skills are carefully engineered, not learned.
- **GUI-specific skills are fragile across versions.** A skill written for Gmail's current UI will break when Google changes their markup. Plan for ongoing maintenance of locators and action sequences.
- **Semantic retrieval has a ceiling.** When multiple skills have similar intent descriptions (e.g., "search contacts" vs. "find contact"), retrieval can return the wrong skill. Precondition filtering helps but doesn't fully solve ambiguity.
- **Not suitable for highly dynamic UIs.** Applications with procedurally generated or user-customized interfaces (e.g., drag-and-drop dashboards) resist parameterized skills because element locators are unpredictable.
- **Composition graphs require manual wiring.** The SCG must be authored by a developer who understands inter-skill dependencies. Incorrect dependency edges cause silent failures in multi-skill workflows.

## Reference

**Paper:** [CUA-Skill: Develop Skills for Computer Using Agent](https://arxiv.org/abs/2601.21123v2) -- Chen et al., 2026. Focus on Section 3 (Skill Cell / PEG / SCG architecture), Section 4 (Agent planning loop and failure recovery), and the WindowsAgentArena evaluation for concrete performance data. Project page: https://microsoft.github.io/cua_skill/