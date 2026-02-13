---
name: "aorchestra-automating-sub-agent-creation"
description: "Dynamically create specialized sub-agents for complex multi-step tasks using the AOrchestra pattern: decompose goals, then spawn tailored (Instruction, Context, Tools, Model) executors on-the-fly. Use when: 'break this task into sub-agents', 'orchestrate agents for this problem', 'create a multi-agent workflow', 'delegate subtasks to specialized agents', 'build an agent pipeline for this', 'dynamically assign agents to subtasks'."
---

This skill enables Claude to apply the AOrchestra orchestration pattern from the paper "AOrchestra: Automating Sub-Agent Creation for Agentic Orchestration" (Ruan et al., 2026). Instead of using pre-defined agent roles, you dynamically decompose complex tasks and spawn specialized sub-agents on-the-fly by concretizing a four-tuple -- (Instruction, Context, Tools, Model) -- for each subtask. This produces better results than static multi-agent designs because each executor is purpose-built for its specific subtask, receives only relevant context (not the full history), and uses appropriately-scoped tools and model choices.

## When to Use

- When the user asks you to solve a complex, multi-step problem that spans different skill domains (e.g., "research this topic, write code based on the findings, then test it")
- When a task requires coordinating multiple agents or sub-processes with different capabilities
- When the user wants to build an automated pipeline that delegates work to specialized executors
- When solving long-horizon tasks where passing full context to every step would cause degradation
- When the user wants a cost-efficient agent workflow that routes simple subtasks to lighter models and complex ones to stronger models
- When designing a multi-agent system architecture and needs a framework-agnostic abstraction
- When a task has failed with a monolithic approach and needs decomposition into specialized sub-agents

## Key Technique

**The Four-Tuple Agent Abstraction.** AOrchestra models every agent as a tuple `(I, C, T, M)`: **Instruction** (the specific, actionable subtask directive), **Context** (curated findings from prior steps -- not the full history), **Tools** (the subset of capabilities needed), and **Model** (the LLM chosen based on subtask complexity). This abstraction is framework-agnostic -- it works whether you're spawning Claude Code agents via the Task tool, calling APIs, or orchestrating shell processes.

**Curated Context Over Full Context.** A critical finding from the paper: passing *all* accumulated history to sub-agents actually hurts performance (84% accuracy) compared to passing *no* context (86%). The winning strategy is **curated context** -- selectively injecting only the relevant findings and artifacts from previous steps (96% accuracy). This means the orchestrator must actively decide what each sub-agent needs to know, filtering out noise and distracting details from prior execution traces.

**The Orchestrator's Restricted Action Space.** The central orchestrator never interacts with the environment directly. Its only two actions are `Delegate(I, C, T, M)` and `Finish(answer)`. This forces clean separation between planning and execution. At each step, the orchestrator reviews subtask history, evaluates whether results are sufficient, and either delegates the next subtask or returns the final answer. This Review-Evaluate-Decide loop continues until the task is complete or the attempt budget is exhausted.

## Step-by-Step Workflow

1. **Analyze the top-level goal.** Parse the user's request to identify the end objective, constraints, and success criteria. Determine whether the task genuinely requires multi-agent decomposition or can be solved directly.

2. **Decompose into subtasks.** Break the goal into a sequence of concrete, actionable subtasks. Each subtask should be independently executable and have a clear deliverable. Order them by dependency -- identify which subtasks can run in parallel and which require prior results.

3. **For each subtask, concretize the four-tuple:**
   - **Instruction (I):** Write a specific, actionable directive. Bad: "Handle the data." Good: "Parse the CSV file at /data/sales.csv into a pandas DataFrame, identify columns with >10% missing values, and return their names with missing percentages."
   - **Context (C):** Select only the relevant findings from completed subtasks. Do NOT pass the full history. If subtask 3 needs the output of subtask 1 but not subtask 2, only include subtask 1's results.
   - **Tools (T):** Scope the tool set to what this subtask actually needs. A code-execution subtask gets Bash; a file-search subtask gets Grep and Glob; a research subtask gets WebFetch. Restrict tools to prevent distraction.
   - **Model (M):** Route simple subtasks (file reading, formatting, search) to faster/cheaper models (haiku). Route complex reasoning, code generation, or critical decisions to stronger models (opus/sonnet).

4. **Spawn the sub-agent.** Use the Task tool with the appropriate `subagent_type` and pass the constructed instruction as the prompt. Include curated context inline in the prompt. Set `model` to match your M selection. Choose the subagent_type that matches the needed tools (e.g., `Bash` for command execution, `general-purpose` for multi-tool tasks, `Explore` for codebase search).

5. **Collect and summarize results.** When the sub-agent returns, extract the core findings, artifacts, and any errors. Summarize these into a structured format that can serve as curated context for downstream subtasks.

6. **Review-Evaluate-Decide loop.** After each sub-agent completes:
   - **Review:** What did the sub-agent accomplish? Were there errors or partial results?
   - **Evaluate:** Do the accumulated results satisfy the original goal?
   - **Decide:** If sufficient, proceed to Finish. If gaps remain, construct the next Delegate tuple. If a subtask failed, re-delegate with adjusted instruction, additional context about the failure, or a more capable model.

7. **Handle failures adaptively.** If a sub-agent fails or times out, don't blindly retry. Analyze the failure, adjust the tuple -- tighten the instruction, add error context, upgrade the model, or change the tool set -- then re-delegate.

8. **Synthesize the final answer.** Once all necessary subtasks are complete, combine their results into a coherent response to the user's original request. Reference specific sub-agent outputs rather than re-deriving information.

## Concrete Examples

**Example 1: Multi-step research and code generation**

```
User: "Find the top 3 most-starred Python testing frameworks on GitHub,
       then create a comparison table as a CSV file."

Orchestration plan:

Subtask 1 - Research:
  I: "Search GitHub for the top 3 most-starred Python testing frameworks.
      Return each framework's name, star count, and one-line description."
  C: (none -- first subtask)
  T: WebFetch, Bash (for gh CLI)
  M: haiku (straightforward search task)
  → subagent_type: general-purpose, model: haiku

Subtask 2 - Generate CSV:
  I: "Create a CSV file at /tmp/testing_frameworks.csv with columns:
      Name, Stars, Description. Populate with the provided data."
  C: "Research results: 1) pytest - 12.5k stars - Simple powerful testing
      2) unittest - (stdlib) - Built-in test framework
      3) robot - 9.8k stars - Generic automation framework"
  T: Write (file creation only)
  M: haiku (simple formatting task)
  → subagent_type: general-purpose, model: haiku

Result: CSV file created with accurate, sourced data.
```

**Example 2: Debugging a failing test suite**

```
User: "The tests in tests/api/ are failing. Fix them."

Orchestration plan:

Subtask 1 - Diagnose:
  I: "Run pytest tests/api/ -x --tb=short and report: which tests fail,
      the exact error messages, and the relevant source file paths."
  C: (none -- first subtask)
  T: Bash
  M: haiku (just running a command and parsing output)
  → subagent_type: Bash, model: haiku

Subtask 2 - Investigate root cause:
  I: "Read the failing test file and the source module it tests.
      Identify why test_create_user fails with 'KeyError: email'.
      Report the root cause and proposed fix."
  C: "pytest output: tests/api/test_users.py::test_create_user FAILED
      KeyError: 'email' in src/api/users.py:42"
  T: Read, Grep, Glob
  M: sonnet (requires code reasoning)
  → subagent_type: general-purpose, model: sonnet

Subtask 3 - Implement fix:
  I: "In src/api/users.py:42, the code accesses request.json['email']
      but the field was renamed to 'email_address' in the schema update.
      Fix the key access and update the test assertion to match."
  C: "Root cause: schema migration renamed 'email' to 'email_address'
      in UserCreate model. Source file: src/api/users.py line 42.
      Test file: tests/api/test_users.py line 18."
  T: Read, Edit
  M: sonnet (code modification requires precision)
  → subagent_type: general-purpose, model: sonnet

Subtask 4 - Verify:
  I: "Run pytest tests/api/ and confirm all tests pass. Report results."
  C: "Fix applied: changed request.json['email'] to
      request.json['email_address'] in src/api/users.py:42"
  T: Bash
  M: haiku (just running tests)
  → subagent_type: Bash, model: haiku

Result: Tests pass. User sees diagnosis, fix explanation, and green tests.
```

**Example 3: Cost-optimized document processing pipeline**

```
User: "Process all PDF invoices in /data/invoices/, extract totals,
       and flag any with amounts over $10,000."

Orchestration plan:

Subtask 1 - Enumerate files:
  I: "List all PDF files in /data/invoices/ and return their paths."
  C: (none)
  T: Glob
  M: haiku (trivial file listing)
  → subagent_type: Explore, model: haiku

Subtask 2 - Extract totals (per-file, parallelizable):
  I: "Read the PDF at {path}. Extract the invoice total amount.
      Return: {filename, total_amount, currency}."
  C: (none -- independent per file)
  T: Read (PDF support)
  M: haiku (structured extraction from clear documents)
  → subagent_type: general-purpose, model: haiku
  Note: Launch multiple agents in parallel for throughput.

Subtask 3 - Analyze and flag:
  I: "Given the extracted invoice data, identify all invoices with
      total > $10,000. Format as a markdown table with columns:
      File, Amount, Flagged."
  C: "Extracted data: [{invoice_001.pdf, $4,500, USD}, ...]"
  T: (none -- pure reasoning)
  M: haiku (simple filtering and formatting)
  → subagent_type: general-purpose, model: haiku

Result: Flagged invoices table delivered to user.
```

## Best Practices

**Do:**
- Write subtask instructions as specific, actionable directives with concrete deliverables. Include file paths, function names, expected output formats.
- Curate context aggressively -- pass only findings relevant to the current subtask. Summaries beat raw logs.
- Route by complexity: use `haiku` for search, file operations, and formatting; use `sonnet` or `opus` for code reasoning, debugging, and architectural decisions.
- Track subtask results in your todo list so you can reference them when constructing context for downstream subtasks.
- Parallelize independent subtasks by launching multiple Task tool calls in a single message.
- Include failure context when re-delegating: what went wrong, what was tried, what to do differently.

**Avoid:**
- Passing full conversation history or all prior sub-agent outputs as context to every sub-agent. This degrades performance.
- Using a single powerful model for all subtasks. Simple tasks waste resources on opus; route them to haiku.
- Writing vague instructions like "look into the issue" or "handle the data." Every instruction should specify what to do, where to do it, and what to return.
- Over-decomposing simple tasks. If a task can be done in one focused agent call, don't split it into five subtasks for the sake of the pattern.
- Letting the orchestrator (you) directly execute subtask work. Maintain the separation: you plan and delegate, sub-agents execute.
- Ignoring sub-agent failures. Always analyze errors and adapt the tuple before re-delegating.

## Error Handling

- **Sub-agent timeout or max-steps exceeded:** The subtask was too broad. Break it into smaller pieces with tighter instructions and re-delegate.
- **Wrong or partial results:** Add the error details to the context for a follow-up subtask. Consider upgrading the model for the retry.
- **Tool access failures:** Verify the tool exists and is appropriate. Switch subagent_type if the current one lacks the needed tool (e.g., `Bash` agent can't use `Edit`).
- **Context overload in downstream agents:** Your curated context is too verbose. Summarize more aggressively -- extract only the 2-3 key findings needed.
- **Budget exhaustion:** If you've used many attempts without progress, step back and re-analyze the decomposition. The subtask boundaries may be wrong.
- **Circular dependencies between subtasks:** Re-order the decomposition. If two subtasks depend on each other, merge them into one sub-agent call with both objectives.

## Limitations

- **Overhead on simple tasks.** The decompose-delegate-collect cycle adds latency. For tasks solvable in a single focused step, skip this pattern entirely.
- **Orchestrator reasoning quality.** The quality of decomposition and context curation depends on the orchestrator's own reasoning. Poor decomposition leads to wasted sub-agent calls.
- **No shared mutable state.** Sub-agents are isolated. If subtask B needs to modify a file that subtask A created, you must serialize them and pass context explicitly. There is no shared memory.
- **Model routing is heuristic.** There is no formal complexity metric for choosing models. The orchestrator estimates difficulty, which can misroute. When in doubt, use a stronger model for the first attempt.
- **Token cost scales with decomposition depth.** Each sub-agent call consumes tokens for its own context window. Deep decomposition trees can be expensive even with cheap models.

## Reference

- **Paper:** [AOrchestra: Automating Sub-Agent Creation for Agentic Orchestration](https://arxiv.org/abs/2602.03786v2) -- Focus on Section 3 (the four-tuple abstraction and orchestrator design), Section 4.3 (context curation ablation showing curated > full > none), and Table 2 (benchmark results demonstrating 16.28% relative improvement).
- **Code:** [github.com/FoundationAgents/AOrchestra](https://github.com/FoundationAgents/AOrchestra) -- See `aorchestra/tools/delegate.py` for tuple construction and `aorchestra/main_agent.py` for the Review-Evaluate-Decide loop.