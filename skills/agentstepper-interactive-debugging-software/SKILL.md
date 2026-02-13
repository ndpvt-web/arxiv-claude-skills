---
name: "agentstepper-interactive-debugging-software"
description: "Interactive debugging of LLM-powered software development agents using structured trajectory analysis, stepwise execution, and live editing of prompts/tool calls. Use when: 'debug my agent', 'why did the agent do that', 'trace agent execution', 'step through agent actions', 'inspect agent trajectory', 'agent is producing wrong output'."
---

# AgentStepper: Interactive Debugging of Software Development Agents

This skill enables Claude to apply the AgentStepper debugging methodology to LLM-powered software development agents. Rather than treating agent execution as an opaque black box, this technique decomposes agent trajectories into structured conversations between the LLM, the agent program, and tools -- then provides debugging primitives (breakpoints, stepping, live editing) adapted from conventional debuggers but operating at agent-action-level abstraction. The core insight: debugging agents shares deep similarities with debugging software programs, but requires reasoning at higher abstractions corresponding to meaningful agent actions rather than lines of code.

## When to Use

- When a user says an AI coding agent (SWE-Agent, Aider, Claude Code, Devin, etc.) produced incorrect output and wants to understand why
- When the user needs to trace through an agent's decision-making process step by step to find where it went wrong
- When debugging a custom agent framework and the user wants to add inspection/breakpoint capabilities
- When the user wants to compare successive LLM prompts in an agent loop to find prompt drift or context corruption
- When building or instrumenting an agent pipeline and needs to intercept and edit tool calls or LLM responses mid-execution
- When analyzing agent logs or trajectories to identify the specific action cycle where a bug was introduced
- When a user asks "why did the agent edit the wrong file" or "where did the agent lose track of the task"

## Key Technique

AgentStepper models agent execution as **interleaved structured conversations** rather than flat linear logs. Every agent operates in a repeating 4-step cycle: (1) prompt the LLM, (2) receive the LLM response, (3) invoke a tool, (4) capture the tool output. By decomposing raw execution traces into these four message types and displaying them chronologically in parallel columns (agent-to-LLM and agent-to-tools), the trajectory becomes navigable. Each cycle represents one "step" -- the atomic unit of debugging.

The second key innovation is **repository-level change tracking**. Every tool invocation that modifies code triggers a git commit with an LLM-generated summary. This produces a commit-history-style view of the agent's modifications, allowing developers to see exactly which action introduced a problematic code change. Combined with a message inspector that can diff successive prompts, this makes it possible to pinpoint both *what* changed and *why* the agent decided to change it.

The third innovation is **live editing at breakpoints**. When execution is paused, developers can modify the prompt before it reaches the LLM, simulate an LLM response entirely, alter tool invocation arguments, or edit tool outputs before the agent processes them. This enables controlled experimentation: "what would happen if the agent received different search results?" or "what if I correct the LLM's flawed reasoning before it acts?"

## Step-by-Step Workflow

1. **Capture the raw trajectory.** Collect the full execution log of the agent run -- every LLM prompt, LLM response, tool invocation (name + arguments), and tool output. If logs are unavailable, reconstruct from available artifacts (shell history, git history, chat transcripts).

2. **Decompose into action cycles.** Parse the trajectory into repeating 4-step cycles: `prompt -> response -> tool_call -> tool_output`. Label each event with its cycle number and type. If a cycle is incomplete (e.g., the agent made consecutive LLM calls without tool use), mark the deviation.

3. **Generate one-line summaries for each event.** For each message in the trajectory, produce a concise summary: what the prompt asked, what the LLM decided, which tool was called with what intent, and what the tool returned. Use the following summary templates:
   - Prompt: "Asked LLM to [action] regarding [target]"
   - Response: "LLM decided to [action] because [reasoning]"
   - Tool call: "Invoked [tool_name] on [target] with [key_args]"
   - Tool output: "Returned [result_summary], [success/failure]"

4. **Build a structured conversation view.** Arrange events in two parallel streams -- left column for agent-LLM interactions, right column for agent-tool interactions -- ordered chronologically. This reveals the causal chain: prompt led to decision led to action led to result led to next prompt.

5. **Track repository-level changes.** For each tool invocation that modified files, capture the diff (or reconstruct it from git history). Associate each diff with its action cycle and summary. Present these as a commit-history view showing progressive code modifications.

6. **Identify the divergence point.** Walk through the structured trajectory to find the first cycle where the agent's behavior deviates from the correct path. Check: Did the prompt contain incorrect context? Did the LLM misinterpret the prompt? Did the tool call use wrong arguments? Did the tool return unexpected output?

7. **Inspect prompt evolution.** Compare successive prompts (cycle N vs cycle N+1) to detect context corruption, lost information, or hallucinated state. Diff the prompts to see exactly what changed between cycles.

8. **Propose a fix via live-edit simulation.** For the identified divergence point, describe what edit to the prompt, LLM response, tool arguments, or tool output would have corrected the trajectory. This is the "what if" analysis: "If the prompt at cycle 3 had included the correct file path, the agent would have edited the right file."

9. **Validate the fix propagates.** Trace forward from the proposed edit to confirm it would resolve downstream issues, not just the immediate symptom. An incorrect tool output in cycle 2 may cause cascading failures through cycles 3-7.

10. **Report findings with actionable recommendations.** Present: (a) the specific cycle and event type where the bug originated, (b) the root cause (prompt issue, LLM reasoning failure, tool misconfiguration, or unexpected tool output), (c) the proposed fix, and (d) how to prevent recurrence (better prompts, tool validation, guardrails).

## Concrete Examples

**Example 1: Agent edits the wrong file**

User: "I ran SWE-Agent on issue #142 (fix the login timeout bug) but it edited `auth/session.py` instead of `auth/timeout.py`. Why?"

Approach:
1. Collect the agent's full trajectory from its log output
2. Decompose into action cycles:
   - Cycle 1: Prompt asks LLM to understand the issue -> LLM identifies timeout-related code -> tool `find_file` searches for "timeout" -> returns `auth/timeout.py` and `auth/session.py`
   - Cycle 2: Prompt asks LLM which file to edit -> LLM says "session.py handles the session timeout logic" -> tool `open_file` opens `auth/session.py`
   - Cycle 3: Prompt with file contents asks LLM for a fix -> LLM generates patch -> tool `edit_file` applies patch to `auth/session.py`
3. Identify divergence: Cycle 2, LLM response. The LLM conflated "session timeout" with "login timeout" -- both files appeared in search results, and the LLM picked the wrong one based on keyword similarity
4. Inspect prompt at Cycle 2: The prompt included both filenames but did not include file content previews to disambiguate

Output:
```
TRAJECTORY ANALYSIS
===================
Divergence Point: Cycle 2, Event: LLM Response
Root Cause: LLM selected auth/session.py over auth/timeout.py due to
  ambiguous context -- the prompt listed filenames without content previews.
  The LLM associated "login timeout" with "session" (plausible but incorrect).

Proposed Fix: At Cycle 2, the prompt should include the first 20 lines of
  each candidate file so the LLM can see that auth/timeout.py contains the
  LOGIN_TIMEOUT_SECONDS constant and the relevant retry logic.

Prevention: Add a file-preview step to the agent's file-selection pipeline.
  Before asking the LLM to choose a file, show it content snippets, not just
  filenames.
```

**Example 2: Agent enters an infinite tool-call loop**

User: "My custom repair agent keeps calling `run_tests` over and over without making progress. It's stuck in a loop."

Approach:
1. Capture the last 10 action cycles from the agent log
2. Decompose and summarize:
   - Cycle 5: Prompt with failing test -> LLM proposes fix -> `edit_file` applies patch -> `run_tests` returns 3 failures
   - Cycle 6: Prompt with 3 failures -> LLM proposes different fix -> `edit_file` applies patch -> `run_tests` returns 3 failures (same tests)
   - Cycle 7: Prompt with 3 failures -> LLM proposes fix identical to Cycle 5 -> `edit_file` applies (reverts to Cycle 5 state) -> `run_tests` returns 3 failures
   - Cycles 8-14: Oscillation between Cycle 5 and Cycle 6 patches
3. Diff successive prompts: The prompt at each cycle includes the current test output but does NOT include the history of previous attempts
4. Root cause: Without memory of past attempts, the LLM rediscovers the same two candidate fixes and oscillates

Output:
```
TRAJECTORY ANALYSIS
===================
Divergence Point: Cycle 6, Event: Prompt Construction
Root Cause: The agent's prompt template does not accumulate history of
  previous repair attempts. The LLM sees only the current test failures,
  causing it to oscillate between two candidate fixes indefinitely.

Code Changes (commit-style view):
  Cycle 5: +    timeout = max(timeout, MIN_TIMEOUT)    # Fix A
  Cycle 6: +    timeout = self.config.get('timeout')    # Fix B
  Cycle 7: +    timeout = max(timeout, MIN_TIMEOUT)    # Fix A (repeated)

Proposed Fix: Modify the prompt template to include a "Previous Attempts"
  section listing past patches and their test results. This gives the LLM
  the context to try genuinely new approaches.

Prevention: Add loop detection -- if the same tool output appears 3+ times,
  inject a meta-prompt: "Previous approaches have failed. Analyze why and
  try a fundamentally different strategy."
```

**Example 3: Instrumenting an existing agent with debugging hooks**

User: "I have a Python agent framework. How do I add AgentStepper-style debugging to it?"

Approach:
1. Identify the four instrumentation points in the agent's main loop
2. Add event capture at each point using a minimal API

Output:
```python
# Minimal instrumentation -- 7 API calls, ~40 lines of code

from agentstepper import Debugger

dbg = Debugger(repo_path="./workspace")  # Tracks code changes via git

class MyAgent:
    def run(self, task):
        dbg.start_run(task_description=task)

        while not self.is_done():
            # 1. Before sending prompt to LLM
            prompt = self.build_prompt()
            prompt = dbg.before_llm_query(prompt)  # Breakpoint + live edit

            # 2. After receiving LLM response
            response = self.llm.complete(prompt)
            response = dbg.after_llm_response(response)  # Inspect + override

            # 3. Before invoking tool
            tool_name, tool_args = self.parse_tool_call(response)
            tool_name, tool_args = dbg.before_tool_call(tool_name, tool_args)

            # 4. After tool execution
            tool_output = self.execute_tool(tool_name, tool_args)
            tool_output = dbg.after_tool_output(tool_output)  # Inspect + edit

            self.update_state(tool_output)

        dbg.end_run()
```

Each `dbg.*` call: (a) records the event to the trajectory, (b) checks for breakpoints, (c) pauses if in stepping mode, and (d) allows live editing of the value before returning it. The debugger auto-commits file changes after each tool call.

## Best Practices

- **Do** decompose trajectories into the 4-event cycle (prompt, response, tool call, tool output) even when analyzing agents you did not build -- this structure is universal across agent frameworks
- **Do** diff successive prompts to catch context drift, where important information silently disappears from the context window across cycles
- **Do** track code changes at the repository level (git diffs per action cycle) rather than only inspecting tool call arguments -- the actual file state is ground truth
- **Do** generate concise one-line summaries for each event to create a scannable overview before diving into details
- **Avoid** reading raw agent logs linearly from start to finish -- use the structured conversation view to jump to the relevant cycle first
- **Avoid** assuming the bug is in the LLM response -- check all four event types. Tool misconfigurations and prompt construction errors are equally common and easier to fix

## Error Handling

| Problem | Diagnosis | Resolution |
|---------|-----------|------------|
| Incomplete trajectory logs | Agent crashed mid-cycle or logging was partial | Reconstruct from git history and shell history; mark missing events as `[unrecorded]` |
| No clear divergence point | Agent gradually drifted rather than making one wrong turn | Compare the trajectory against an ideal trajectory for the task; look for cumulative prompt degradation |
| Trajectory too long to analyze manually | Agent ran 50+ cycles | Start from the end (first visible symptom) and trace backward through the commit-history view to find when the problematic code change was introduced |
| Tool outputs are opaque (binary data, long outputs) | Cannot meaningfully summarize | Truncate tool outputs to first/last 50 lines; focus on exit codes and error messages as diagnostic signals |
| Multiple interleaved bugs | Agent made several independent mistakes | Isolate each bug by identifying which action cycles it affects; debug the earliest one first since later bugs may be cascading consequences |

## Limitations

- This technique works best for agents with a clear prompt-response-tool-output loop. Agents with complex multi-threaded or parallel tool execution are harder to decompose into clean cycles.
- Live editing (modifying prompts/responses mid-execution) requires either the agent framework to support interception hooks or a replay mechanism. Post-hoc analysis of logs can identify bugs but cannot test fixes interactively.
- LLM-generated summaries of trajectory events can themselves be inaccurate. When precision matters, always inspect the full message content, not just the summary.
- The technique does not help with bugs caused by the agent's high-level strategy or planning -- it is best at pinpointing execution-level mistakes within a given strategy.
- Repository-level change tracking assumes the agent modifies files on disk. Agents that operate purely through API calls (e.g., database modifications, HTTP requests) need adapted change-capture mechanisms.

## Reference

[AgentStepper: Interactive Debugging of Software Development Agents](https://arxiv.org/abs/2602.06593v1) -- Hutter & Pradel, 2026. Key sections: the 4-event action cycle model (Section 3), the 7-function instrumentation API (Table 1), the structured conversation view design (Section 4), and the user study showing bug identification success jumping from 17% to 60% with the debugger (Section 5).