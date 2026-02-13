---
name: "pabu-progress-aware-belief-update"
description: "Apply Progress-Aware Belief Update (PABU) to build efficient LLM agents that track task progress and selectively retain context instead of passing full histories. Use when: 'build an agent with belief states', 'reduce agent context bloat', 'make my agent loop more efficient', 'implement progress-aware agent', 'selective context retention for agents', 'compact agent state management'."
---

# Progress-Aware Belief Update (PABU) for Efficient LLM Agents

This skill teaches you to implement the PABU framework from Jiang et al. (2026), which replaces the common pattern of feeding full action-observation histories to LLM agents with a compact **belief state** that explicitly tracks task progress and selectively retains only useful past interactions. The result is agents that complete tasks more reliably (81% vs 57% on AgentGym) while using 27% fewer steps, because they avoid redundant actions caused by irrelevant context.

## When to Use

- When building an agentic loop (tool-use, ReAct, function-calling) and the agent starts repeating actions or losing track of what it already tried
- When context windows fill up with stale observations and the agent degrades on long tasks
- When you want to add explicit progress tracking to a multi-step agent workflow
- When an agent's action history grows large and you need to compress it without losing critical information
- When implementing a task-execution agent that must decide which observations to remember and which to discard
- When refactoring a naive "append everything to messages" agent into a structured belief-state agent

## Key Technique

### The Problem with Full Histories
Standard LLM agents append every action and observation to a growing message list. This introduces task-irrelevant noise, causes the model to fixate on stale information, and leads to redundant retries of failed actions. Token costs scale linearly with trajectory length.

### The PABU Belief State
PABU replaces the full history with a structured belief state **b = [q, p, A_att, A_available, O_saved]** where:
- **q** is the original user query (always retained)
- **p** is a natural-language progress estimate (e.g., "Step 2 of 4: found the target file, now need to extract the relevant section")
- **A_att** is the set of actions attempted at the current progress stage (prevents retrying the same failed action)
- **A_available** is the set of currently available actions (extracted from the most recent observation)
- **O_saved** is a selectively retained subset of past observations deemed useful for future decisions

At each step, the agent performs three predictions before choosing an action: (1) should the previous observation be saved into O_saved? (2) what is the current progress estimate? (3) what action to take next. Progress acts as an approximately Markovian backbone — when progress advances, A_att resets because the agent has moved to a new stage. When progress is unchanged, A_att accumulates to prevent loops.

### Why It Works
Explicit progress prediction forces the agent to reason about *where it is* in the task before acting. Selective retention forces it to reason about *what matters* before storing context. Together, these prevent the two most common failure modes: repeating actions (tracked by A_att) and being distracted by irrelevant observations (filtered by the retention decision). The belief state stays compact regardless of trajectory length.

## Step-by-Step Workflow

### 1. Define the Belief State Schema
Create a data structure for the belief state with five fields: `query`, `progress`, `attempted_actions`, `available_actions`, and `saved_observations`. Use a typed dict, dataclass, or Pydantic model.

### 2. Initialize the Belief State
Set `query` to the user's task description. Set `progress` to a starting description like "Task received, no actions taken yet." Initialize `attempted_actions`, `available_actions`, and `saved_observations` as empty lists.

### 3. Implement the Retention Decision
Before each action step, present the agent with the most recent observation and ask: "Does this observation contain information that will be needed for future steps? Respond YES or NO with a one-line reason." If YES, append a concise summary of the observation to `saved_observations`. Always discard raw verbose output in favor of extracted facts.

### 4. Implement Progress Prediction
After the retention decision, prompt the agent to estimate current progress relative to the previous step. The output should be a short natural-language statement describing what stage the task is in and what remains. If progress has advanced from the previous estimate, clear `attempted_actions` (the agent has moved to a new stage).

### 5. Build the Action Prompt from Belief State
Construct the LLM prompt using ONLY the belief state fields — not the raw history. Format it as: query, then progress summary, then saved observations, then attempted actions at this stage, then available actions. This is the agent's entire context for deciding the next action.

### 6. Generate and Execute the Action
The agent selects an action from `available_actions` (or generates a free-form action if the environment allows it). Before executing, check if the action is already in `attempted_actions` for the current progress stage. If it is, force the agent to choose a different action or escalate.

### 7. Record the Action and Observation
Add the chosen action to `attempted_actions`. Execute it and capture the observation. Update `available_actions` from the new observation if applicable.

### 8. Loop Until Completion or Budget
Repeat steps 3-7. Terminate when the agent's progress prediction indicates task completion, when a final-answer action is generated, or when a step budget is exhausted.

### 9. Handle Budget Exhaustion Gracefully
If the step budget runs out, return the best partial result along with the current belief state so the user (or an outer agent) can inspect what was accomplished and what remains.

## Concrete Examples

**Example 1: File Search and Extraction Agent**

```
User: "Find all TODO comments across the codebase and generate a summary report."

Belief State Evolution:

Step 0 (init):
  query: "Find all TODO comments and generate summary report"
  progress: "Task received. Need to search codebase for TODOs."
  attempted_actions: []
  saved_observations: []

Step 1:
  action: grep -r "TODO" --include="*.py" src/
  observation: [180 lines of grep output across 12 files]
  retention_decision: YES — "Contains all TODO locations and text"
  saved_observations: ["12 files contain TODOs: auth.py(3), api.py(5), ..."]
  progress: "Found TODO locations. Now need to categorize and summarize."
  attempted_actions: [] (progress advanced, so reset)

Step 2:
  action: read auth.py lines 45-60, 112-115, 200-205
  observation: [TODO context from auth.py]
  retention_decision: YES — "Auth TODOs relate to session expiry and token refresh"
  saved_observations: ["12 files contain TODOs...", "auth.py: 3 TODOs about session/token handling"]
  progress: "Categorized 1/12 files. Continue reading remaining files."
  attempted_actions: [read auth.py]

... (continues through files, retaining summaries, discarding raw content)

Step 8:
  progress: "All 12 files categorized. Ready to generate report."
  action: generate_report(saved_observations)
  output: Structured markdown report with TODOs grouped by category
```

**Example 2: Multi-Step API Debugging Agent**

```
User: "The /users endpoint returns 500. Find and fix the bug."

Belief State Evolution:

Step 0 (init):
  query: "Fix 500 error on /users endpoint"
  progress: "Need to reproduce and diagnose the 500 error."
  attempted_actions: []
  saved_observations: []

Step 1:
  action: curl localhost:3000/users
  observation: "500 Internal Server Error: Cannot read property 'map' of undefined"
  retention_decision: YES — "Error message identifies the root cause type"
  saved_observations: ["GET /users returns: Cannot read property 'map' of undefined"]
  progress: "Reproduced error. 'map' called on undefined — likely a null DB result."

Step 2:
  action: read routes/users.js
  observation: [40 lines of route handler code]
  retention_decision: YES — "Line 12: const users = await db.query(...); Line 14: return users.map(...)"
  saved_observations: ["...error msg...", "users.js:12 queries DB, :14 calls .map() on result without null check"]
  progress: "Found bug: db.query can return undefined when table is empty. Need to add null guard."

Step 3:
  action: edit routes/users.js line 14 — add "const users = result || [];"
  observation: "File saved successfully"
  retention_decision: NO — "Save confirmation has no future value"
  progress: "Fix applied. Need to verify."

Step 4:
  action: curl localhost:3000/users
  observation: "200 OK: []"
  retention_decision: NO — "Verification complete, no future use"
  progress: "Bug fixed and verified. Task complete."
  → return final answer
```

**Example 3: Preventing Redundant Actions**

```
User: "Install the project dependencies and run the test suite."

Step 1:
  action: npm install
  observation: "npm ERR! ERESOLVE: peer dependency conflict with react@18"
  retention_decision: YES — "Dependency conflict details needed for resolution"
  progress: "Install failed due to peer dependency conflict."
  attempted_actions: [npm install]

Step 2:
  agent considers "npm install" again but A_att blocks it
  action: npm install --legacy-peer-deps
  observation: "added 847 packages in 12s"
  retention_decision: NO — "Success confirmation, no future value"
  progress: "Dependencies installed. Now run tests."
  attempted_actions: [] (progress advanced, reset)

Step 3:
  action: npm test
  observation: "Tests: 42 passed, 3 failed..."
  retention_decision: YES — "Failed test names needed for reporting"
  progress: "Tests complete. 3 failures to report."
  → return summary with failing test details
```

## Best Practices

**Do:**
- Keep `saved_observations` as concise extracted facts, not raw output. Summarize a 200-line grep result into "12 files, 47 matches, concentrated in src/api/"
- Reset `attempted_actions` whenever progress genuinely advances — this is what allows the agent to retry actions that previously failed in a different context
- Make progress descriptions specific and stage-oriented: "Identified root cause in users.js:14" is better than "Making progress"
- Include the retention reasoning in your prompt so the LLM explicitly justifies what it keeps and discards

**Avoid:**
- Saving every observation "just in case" — this defeats the purpose and recreates the full-history problem
- Using progress as a simple percentage (e.g., "50%") — natural-language descriptions carry more information for the LLM's next decision
- Skipping the duplicate-action check against `attempted_actions` — this is the primary mechanism that prevents infinite loops
- Putting raw tool output into `saved_observations` without summarization — extract the 2-3 facts that matter

## Error Handling

| Problem | Cause | Resolution |
|---------|-------|------------|
| Agent loops on the same action | `attempted_actions` check not enforced | Before executing, verify the action is not in `attempted_actions`. If blocked, prompt the agent to explain why it wants to retry and force an alternative |
| `saved_observations` grows unbounded | Retention decision is too permissive | Cap at 8-10 entries. When full, ask the agent to merge or evict the least relevant entry before adding a new one |
| Progress never advances | Agent is stuck or task is harder than expected | After N steps at the same progress stage, inject a meta-prompt: "You have attempted N actions without advancing. Reassess your approach." |
| Retention discards critical info | Retention decision is too aggressive | For high-stakes tasks, default to retaining and let the agent evict later. Better to over-retain than lose a key fact |
| Belief state prompt exceeds context | Too many saved observations accumulated | Trigger a compaction step: ask the agent to consolidate all saved observations into a single summary paragraph |

## Limitations

- **Training-dependent optimization**: The full PABU paper trains a model to predict retention and progress jointly. When using this as a prompting strategy (without fine-tuning), retention and progress quality depend on the base model's instruction-following ability.
- **Simple tasks don't benefit**: For 1-3 step tasks, the overhead of retention decisions and progress tracking adds complexity without payoff. Use PABU only when trajectories typically exceed 4-5 steps.
- **Progress prediction can stall**: If the agent cannot articulate what "progress" means for an unusual task, the Markovian backbone breaks down. Provide task-specific progress stage examples in the system prompt when possible.
- **Not a replacement for better tools**: If the agent loops because the right tool doesn't exist, belief state management won't help. PABU optimizes *how* the agent uses its tools, not *which* tools are available.
- **Retention is lossy**: Once an observation is discarded, it cannot be recovered. For tasks where any detail might matter later (e.g., forensic analysis), full-history approaches may be safer.

## Reference

Jiang, H., Ge, L., Cai, H., & Song, R. (2026). *PABU: Progress-Aware Belief Update for Efficient LLM Agents*. arXiv:2602.09138v1. [https://arxiv.org/abs/2602.09138v1](https://arxiv.org/abs/2602.09138v1)

Key sections to read: Section 3 for the formal belief state definition (b = [q, p, A_att, A_available, O_saved]), Section 3.2 for the training objective that jointly supervises retention/progress/action, and Section 4 for ablation results showing both components are necessary.