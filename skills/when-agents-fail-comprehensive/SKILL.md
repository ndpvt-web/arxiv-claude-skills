---
name: "when-agents-fail-comprehensive"
description: "Diagnose and fix bugs in LLM agent systems using a research-backed taxonomy of 11 bug types, 9 root causes, and 12 observable effects across agent core, tools, planning, and memory components. Use when: 'my LangChain agent is crashing', 'debug this CrewAI workflow', 'why is my agent stuck in a loop', 'agent gives empty responses', 'fix this AutoGen agent error', 'agent ignores tools I defined'."
---

# When Agents Fail: Systematic Bug Diagnosis for LLM Agent Systems

This skill equips Claude with a structured diagnostic framework for identifying, classifying, and fixing bugs in LLM agent-based software. Derived from a study of 1,187 real bug reports across Stack Overflow, GitHub, and Hugging Face forums, it covers agents built with LangChain, LangGraph, CrewAI, AutoGen, LlamaIndex, Semantic Kernel, Hugging Face Transformers Agents, and custom implementations. Rather than guessing at fixes, Claude systematically narrows the bug by type, root cause, affected component, and observable effect to produce targeted solutions.

## When to Use

- When a user reports a crash, error, or unexpected behavior in an LLM agent system
- When an agent produces empty responses, partial output, or ignores available tools
- When an agent enters an infinite loop or hangs during multi-step reasoning
- When a user is getting parsing errors from LLM output that doesn't match expected formats
- When upgrading an agent framework breaks previously working code
- When an agent loses conversation context or memory between turns
- When reviewing or building agent code and the user wants to proactively catch common bug patterns
- When a user asks "why does my agent keep failing" without a clear error message

## Key Technique: Component-Aware Bug Taxonomy

LLM agents consist of four architectural components, each with distinct failure modes:

**Agent Core** (hosts 58% of all bugs): The orchestration layer that manages LLM calls, prompt construction, and response routing. Bugs here include logic errors in chain construction, incorrect argument formats passed to LLM APIs, and configuration mistakes in model parameters. When a user reports a crash or incorrect output, start diagnosis here.

**Tools** (second most affected): The external integrations an agent can invoke. Bugs manifest as tools being silently ignored, argument schema mismatches between the agent and tool definitions, or availability failures when external services go down. Tool bugs are distinctive because the agent often appears to work but produces wrong or incomplete results.

**Planning** (responsible for 66.6% of infinite loops): The reasoning layer that decomposes tasks into steps. When an agent gets stuck in a loop or produces incoherent multi-step plans, the planning component is the primary suspect. This is especially common in ReAct-style and multi-agent orchestration patterns.

**Memory** (causes 57.1% of stateless interaction bugs): The context management layer. When an agent "forgets" prior conversation, repeats itself, or loses track of intermediate results, memory configuration or implementation is typically at fault.

The taxonomy classifies every bug along three independent dimensions -- type (what went wrong), root cause (why), and effect (how it manifests) -- enabling precise diagnosis instead of trial-and-error debugging.

## Step-by-Step Diagnostic Workflow

1. **Identify the observable effect.** Map the user's reported symptom to one of: Crash, Incorrect Output, Empty Response, Output Dump (excessive unstructured text), Stateless Interaction, Partial Output, Tool Ignored, Slow Output, Warning, Hang, Indeterminate Loop, Resource Overuse, or Silent Fail. This immediately narrows the search space.

2. **Locate the affected component.** Based on the effect, determine whether the bug likely resides in Agent Core, Tools, Planning, or Memory. Use these heuristics:
   - Crash/Warning/Incorrect Output → start with Agent Core
   - Tool Ignored/Partial Output → check Tools component
   - Indeterminate Loop/Hang → investigate Planning
   - Stateless Interaction → examine Memory

3. **Classify the bug type.** Match the symptoms and component to one of the 11 bug types:
   - **Logic Bug** (22-30% of all bugs): Incorrect function use, missing code segments, flawed conditional logic
   - **Configuration Bug**: Wrong model parameters, temperature, max_tokens, or environment settings
   - **Initialization Bug**: Variables or client objects used before proper setup
   - **Argument Bug**: API called with wrong argument types, order, or format
   - **Parsing Bug**: LLM output doesn't match the parser's expected schema (JSON, XML, structured)
   - **Prompting Bug**: Missing system prompt components, contradictory instructions, or under-specified tool descriptions
   - **API Bug**: Dependency conflicts, version incompatibilities, uninstalled packages
   - **Reference Bug**: Importing deprecated or renamed modules after framework updates
   - **Availability Bug**: External LLM or service unreachable
   - **Model Bug**: Selected model lacks capability for the task (e.g., vision task on text-only model)
   - **Resource Limitation Bug**: Insufficient GPU memory, API rate limits, or credit exhaustion

4. **Determine the root cause.** Trace the bug type to its underlying cause:
   - API Misuse → wrong context manager usage, invalid API arguments
   - Incorrect/Missing Parameters → omitted required args, wrong default values
   - Incorrect Data Format → input/output schema mismatch
   - Incorrect/Missing Control Flow → missing error handling, wrong branching logic
   - Incorrect Instruction → prompt engineering flaws, orchestration misconfiguration
   - API Limitation → unsupported feature, service downtime
   - Component Mismatch → incompatible LLM-tool-memory combination
   - Requirement Violation → dependency version conflict, incompatible library pairing
   - Environment/External → machine-specific or OS-level issues

5. **Check for framework-specific patterns.** Apply known framework tendencies:
   - LangChain: 51.5% of requirement violation bugs; check `langchain-core` vs `langchain-community` version alignment
   - CrewAI: Rising API misuse rate (66.6% of recent bugs); verify agent role/goal/backstory format
   - LangGraph: Graph construction errors; verify node and edge definitions
   - AutoGen: Multi-agent message routing issues; check speaker selection logic
   - LlamaIndex: Index construction and retrieval pipeline mismatches

6. **Read the actual error trace.** Before proposing fixes, read the full stack trace or error output. Map the traceback to the component and bug type identified above. Look for the deepest non-framework frame to find user code responsibility.

7. **Propose a targeted fix.** Based on the classified bug type and root cause, generate a specific code fix. Do not suggest generic "try reinstalling" unless the diagnosis points to Requirement Violation.

8. **Add a defensive check.** After fixing the immediate bug, add one targeted guard against recurrence: type validation for Argument Bugs, output schema enforcement for Parsing Bugs, retry logic for Availability Bugs, or dependency pinning for API Bugs.

9. **Verify the fix addresses the root cause, not just the symptom.** Confirm the fix targets the root cause category, not just the observable effect. A crash caused by a Parsing Bug needs output format correction, not a bare try/except.

## Concrete Examples

**Example 1: Agent crashes with `OutputParserException`**

User: "My LangChain agent crashes with OutputParserException when calling a tool."

Approach:
1. Observable effect: **Crash**
2. Component: **Agent Core** (parser lives in orchestration layer)
3. Bug type: **Parsing Bug** -- LLM output doesn't match expected format
4. Root cause: **Incorrect Data Format** -- model returns free text instead of structured action format
5. Framework check: LangChain's `AgentOutputParser` expects `Action: <tool>\nAction Input: <input>` format

Diagnosis and fix:
```python
# ROOT CAUSE: The prompt doesn't enforce output format strictly enough,
# or the model deviates from the expected ReAct format.

# FIX 1: Add output format instructions to the prompt
from langchain.agents import AgentExecutor, create_react_agent

# Ensure the prompt template includes explicit format instructions
prompt = hub.pull("hwchase17/react")  # Use a well-tested prompt

# FIX 2: Add an output-fixing parser as a fallback
from langchain.output_parsers import OutputFixingParser
from langchain_core.output_parsers import JsonOutputParser

base_parser = JsonOutputParser(pydantic_object=ActionSchema)
fixing_parser = OutputFixingParser.from_llm(parser=base_parser, llm=llm)

# FIX 3: Handle the parsing failure gracefully with handle_parsing_errors
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    handle_parsing_errors=True,  # Returns error to agent for self-correction
    max_iterations=5,
)
```

**Example 2: Agent stuck in infinite loop**

User: "My CrewAI agent keeps repeating the same steps and never finishes."

Approach:
1. Observable effect: **Indeterminate Loop**
2. Component: **Planning** (66.6% of loop bugs originate here)
3. Bug type: **Logic Bug** or **Prompting Bug**
4. Root cause: **Incorrect/Missing Control Flow** or **Incorrect Instruction**
5. Framework check: CrewAI agents need explicit goal completion criteria

Diagnosis and fix:
```python
# ROOT CAUSE: Agent lacks clear termination criteria in its goal,
# or the task description doesn't specify what "done" looks like.

# BEFORE (buggy):
researcher = Agent(
    role="Researcher",
    goal="Research the topic",  # Too vague -- no completion signal
    backstory="You are a research assistant.",
    llm=llm,
)

# AFTER (fixed):
researcher = Agent(
    role="Researcher",
    goal="Find 3 key facts about the topic and return them as a numbered list. "
         "Stop after compiling the list.",  # Explicit termination criterion
    backstory="You are a research assistant who delivers concise results.",
    llm=llm,
    max_iter=10,          # Hard iteration cap as safety net
    allow_delegation=False,  # Prevent circular delegation loops
)

# Also set max_iterations on the Crew itself:
crew = Crew(
    agents=[researcher],
    tasks=[research_task],
    max_rpm=10,  # Rate limit to prevent runaway API calls
)
```

**Example 3: Agent ignores defined tools**

User: "I defined a search tool but my agent never uses it, it just makes up answers."

Approach:
1. Observable effect: **Tool Ignored**
2. Component: **Tools** (tool definition or binding issue)
3. Bug type: **Prompting Bug** or **Configuration Bug**
4. Root cause: **Incorrect Instruction** (tool description unclear) or **Component Mismatch** (tool not bound to agent)

Diagnosis and fix:
```python
# ROOT CAUSE 1: Tool description doesn't clearly tell the agent WHEN to use it.
# The LLM decides tool usage based on the description string.

# BEFORE (buggy):
@tool
def search(query: str) -> str:
    """Search tool."""  # Too vague -- agent doesn't know when to invoke
    return search_api(query)

# AFTER (fixed):
@tool
def search(query: str) -> str:
    """Search the web for current information. USE THIS TOOL whenever the user
    asks about recent events, facts you're unsure about, or anything requiring
    up-to-date data. Input should be a search query string."""
    return search_api(query)

# ROOT CAUSE 2: Tool not actually bound to the agent.
# Verify tools list is passed correctly:
agent = initialize_agent(
    tools=[search],       # Confirm tool is in this list
    llm=llm,
    agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
    verbose=True,         # Enable to confirm tool availability in logs
)
```

## Best Practices

- **Do:** Pin framework dependency versions in `pyproject.toml` or `requirements.txt`. Requirement Violation is the top root cause on Stack Overflow (15%), and LangChain alone accounts for 51.5% of these bugs.
- **Do:** Write explicit, action-oriented tool descriptions that tell the agent *when* to use each tool, not just *what* it does. Tool Ignored bugs stem from vague descriptions.
- **Do:** Set `max_iterations` or equivalent caps on all agent loops. Planning-component bugs cause 66.6% of infinite loops, and a hard cap is the cheapest safety net.
- **Do:** Use `verbose=True` or equivalent logging during development. Most agent bugs are non-deterministic -- logs capture the reasoning trace needed for diagnosis.
- **Avoid:** Wrapping agent execution in broad `try/except Exception` blocks. This converts Crash effects into Silent Fail effects, making diagnosis harder. Catch specific exceptions.
- **Avoid:** Assuming a framework upgrade is backward-compatible. Reference Bugs from deprecated imports spike after major releases. Check the migration guide and changelog before upgrading.

## Error Handling

| Symptom | Likely Bug Type | First Check |
|---|---|---|
| `ImportError` or `ModuleNotFoundError` | Reference Bug | Module was renamed or moved in recent framework version |
| `ValidationError` from Pydantic | Argument Bug | API signature changed; check parameter types and names |
| `OutputParserException` | Parsing Bug | LLM output format doesn't match parser expectations |
| Agent returns `None` or empty string | Prompting Bug | System prompt missing output format instructions |
| `RateLimitError` or `429` | Availability/Resource Bug | Add retry with exponential backoff; check quota |
| Agent calls wrong tool repeatedly | Logic Bug in Planning | Review agent prompt for ambiguous tool selection criteria |
| `ContextWindowExceeded` | Resource Limitation Bug | Implement conversation summarization or trim message history |
| Agent works locally but fails in CI | Configuration Bug | Check environment variables, API keys, and model availability |

## Limitations

- This taxonomy is derived from bugs reported through early 2025. As agent frameworks mature, new bug categories may emerge (particularly around multi-agent coordination and long-running workflows).
- The framework-specific patterns are most reliable for LangChain and CrewAI, which had the largest representation in the dataset. Less common frameworks may have distinct failure modes not fully captured.
- Non-deterministic bugs (where the same code sometimes works and sometimes fails) are inherently harder to classify. The LLM's stochastic output means a Parsing Bug may only trigger on certain runs.
- This diagnostic framework addresses software bugs in agent *implementation*. It does not cover prompt quality issues that lead to poor but technically correct agent behavior (e.g., the agent works but gives unhelpful answers).
- Bugs in the underlying LLM itself (hallucination, instruction-following failures) fall outside this taxonomy unless they manifest through a specific agent component mismatch.

## Reference

**Paper:** Islam, N., Ayon, R.S., Thomas, D.G., Ahmed, S., & Wardat, M. (2026). "When Agents Fail: A Comprehensive Study of Bugs in LLM Agents with Automated Labeling." arXiv:2601.15232v1. https://arxiv.org/abs/2601.15232v1

**What to look for:** Tables 2-4 contain the complete bug type, root cause, and effect taxonomies with definitions. Figures 5-8 show component-level and framework-level distributions. Section 5 details the BugReAct automated labeling architecture for teams wanting to build their own bug classification pipelines.