---
name: "automated-structural-testing-llm-based"
description: "Write structural tests for LLM-based agents using trace-based assertions, mocked LLM responses, and the test automation pyramid. Use when the user says 'test my agent', 'write agent tests', 'mock LLM responses', 'add regression tests for my agent', 'structural testing for agents', or 'trace-based assertions'."
---

# Structural Testing for LLM-Based Agents

This skill enables Claude to write deep structural tests for LLM-based agents — not just end-to-end acceptance tests, but layered tests that capture agent trajectories as traces, mock LLM behavior for deterministic replay, and assert on internal tool invocations and intermediate outputs. The approach adapts the classical test automation pyramid (unit / integration / E2E) to agentic systems, enabling fast regression detection and root-cause analysis without expensive live LLM calls.

## When to Use

- When the user asks to write tests for an LLM agent or agentic workflow
- When an agent has tools (function calls) and the user needs to verify correct tool selection and parameterization
- When the user wants deterministic, repeatable agent tests that don't call a live LLM
- When debugging why an agent chose the wrong tool or produced an incorrect response — root-cause analysis via trace inspection
- When the user wants to adopt TDD for agent development: define expected traces first, then implement the agent
- When building a CI pipeline for an agentic application and needing fast, cost-free regression tests
- When the user needs to test multi-turn conversation flows with specific expected tool chains

## Key Technique

Traditional agent testing evaluates from the user's perspective: give it an input, check the final output. This is the top of the test pyramid — slow, expensive, and brittle. Structural testing operates at the lower layers. It captures **traces** — structured records of every internal operation (LLM invocations, tool calls, intermediate reasoning) — and makes them the foundation for assertions. Instead of asking "did the agent give a good answer?", you ask "did the agent call `get_weather` with `city='Amsterdam'` before composing its response?"

The method has three pillars. **Traces** (OpenTelemetry-compatible) record agent trajectories: which tools were invoked, with what inputs, what they returned, how many LLM turns occurred, token counts, and latency. **Mocking** replaces the live LLM with deterministic response sequences so that a test always follows the same execution path — no flakiness from model non-determinism. **Assertions** operate on the collected traces to verify structural properties: tool inclusion/exclusion, output content, invocation order, and custom validation functions.

This layered approach mirrors the software engineering test pyramid. At the base, **unit tests** mock the LLM and assert on individual tool invocations. In the middle, **integration tests** verify multi-step tool chains and conversation flows with mocked responses. At the top, **acceptance tests** use live LLMs with semantic similarity metrics. Most tests should be at the base — fast, cheap, deterministic — with fewer tests at each higher level.

## Step-by-Step Workflow

1. **Identify the agent's tool surface.** List every tool/function the agent can invoke, its parameters, and expected return types. This defines what structural assertions are possible.

2. **Define test cases as conversation turns.** For each scenario, write the sequence of user inputs that exercises a specific code path. Use `Case(user_inputs=["turn1", "turn2", ...])` to model multi-turn interactions.

3. **Create mock LLM responses.** For unit and integration tests, build a mock client that returns predetermined responses. Map each user input to a specific tool-use response or text response the LLM would produce, ensuring deterministic execution:
   ```python
   from generative_ai_toolkit.mock import MockBedrockConverse
   mock_client = MockBedrockConverse()
   agent = BedrockConverseAgent(
       model_id="test-model",
       bedrock_client=mock_client,
       system_prompt="You are a travel assistant."
   )
   ```

4. **Register tools on the agent.** Attach the same tool functions (or lightweight stubs) that the production agent uses, so the mock LLM's tool-use responses can be executed:
   ```python
   agent.register_tool(get_current_location)
   agent.register_tool(get_interesting_things_to_do)
   ```

5. **Run the test case and collect traces.** Execute the conversation and capture the trace log:
   ```python
   from generative_ai_toolkit.test import Case
   test_case = Case(user_inputs=["Find things to do near me within 30 minutes"])
   traces = test_case.run(agent)
   ```

6. **Write trace-based assertions.** Use the `Expect` API to assert on tool invocations, outputs, and text responses:
   ```python
   from generative_ai_toolkit.test import Expect
   Expect(traces).tool_invocations.to_include("get_current_location")
   Expect(traces).tool_invocations.to_include("get_interesting_things_to_do")
   Expect(traces).tool_invocations.to_not_include("start_navigation")
   Expect(traces).agent_text_response.to_include("suggest")
   ```

7. **Add negative and boundary assertions.** Verify the agent does NOT call tools it shouldn't, handles missing parameters gracefully, and respects tool ordering constraints.

8. **Organize tests into pyramid layers.** Place fast mocked tests in a `tests/unit/` directory, integration tests with multi-step chains in `tests/integration/`, and any live-LLM semantic tests in `tests/acceptance/`. Weight the count heavily toward unit tests.

9. **Integrate into CI.** Unit and integration tests (mocked) run on every commit with zero LLM cost. Acceptance tests run on a schedule or before releases.

10. **Capture traces as regression baselines.** When the agent behaves correctly, save the trace as a snapshot. Future test runs compare against this baseline to catch regressions from prompt changes, model upgrades, or tool modifications.

## Concrete Examples

**Example 1: Unit-testing tool selection for a travel agent**

User: "Write tests for my travel agent that verify it calls get_location before get_attractions."

Approach:
1. Mock the LLM to return tool-use responses in the expected order
2. Run a single-turn conversation
3. Assert on trace tool invocations

```python
import pytest
from generative_ai_toolkit.agent import BedrockConverseAgent
from generative_ai_toolkit.mock import MockBedrockConverse
from generative_ai_toolkit.test import Case, Expect

def get_current_location() -> str:
    """Gets the user's current GPS coordinates."""
    return "52.3676, 4.9041"

def get_attractions(location: str, max_drive_minutes: int) -> str:
    """Gets nearby attractions within drive time."""
    return "Rijksmuseum, Vondelpark, Anne Frank House"

@pytest.fixture
def travel_agent():
    agent = BedrockConverseAgent(
        model_id="test-model",
        bedrock_client=MockBedrockConverse(),
        system_prompt="You are a travel assistant. Always get location first.",
    )
    agent.register_tool(get_current_location)
    agent.register_tool(get_attractions)
    return agent

def test_tool_ordering(travel_agent):
    case = Case(user_inputs=["What can I do nearby within 20 minutes?"])
    traces = case.run(travel_agent)

    Expect(traces).tool_invocations.to_include("get_current_location")
    Expect(traces).tool_invocations.to_include("get_attractions")
    # Agent should NOT start navigation without user confirmation
    Expect(traces).tool_invocations.to_not_include("start_navigation")

def test_agent_asks_for_preferences(travel_agent):
    case = Case(user_inputs=["I want to do something fun"])
    traces = case.run(travel_agent)

    # Agent should ask clarifying questions, not jump to tool use
    Expect(traces).agent_text_response.to_include("prefer")
```

**Example 2: Multi-turn regression test with mocked responses**

User: "Add regression tests for my customer support agent's refund flow."

Approach:
1. Define a multi-turn conversation covering the full refund path
2. Mock deterministic LLM responses for each turn
3. Assert the agent calls the right tools in sequence and produces expected outputs

```python
def test_refund_flow_happy_path(support_agent):
    case = Case(user_inputs=[
        "I want a refund for order #12345",
        "Yes, the item was damaged",
        "Yes, please process the refund",
    ])
    traces = case.run(support_agent)

    # Verify correct tool chain
    Expect(traces).tool_invocations.to_include("lookup_order")
    Expect(traces).tool_invocations.to_include("check_refund_eligibility")
    Expect(traces).tool_invocations.to_include("process_refund")
    Expect(traces).tool_invocations.to_include("process_refund").with_output("approved")

    # Verify agent confirms with user
    Expect(traces).agent_text_response.to_include("refund has been processed")

def test_refund_flow_ineligible(support_agent):
    case = Case(user_inputs=[
        "Refund order #99999",
        "I just changed my mind",
    ])
    traces = case.run(support_agent)

    Expect(traces).tool_invocations.to_include("lookup_order")
    Expect(traces).tool_invocations.to_include("check_refund_eligibility")
    # Should NOT process refund for ineligible case
    Expect(traces).tool_invocations.to_not_include("process_refund")
```

**Example 3: Test-driven development for a new agent tool**

User: "I'm adding a schedule_meeting tool to my assistant. Help me write the tests first."

Approach:
1. Write failing tests that define expected behavior
2. User implements the tool to make tests pass

```python
def test_schedule_meeting_requires_all_params(assistant_agent):
    """Agent should ask for missing info before calling schedule_meeting."""
    case = Case(user_inputs=["Schedule a meeting with Alice"])
    traces = case.run(assistant_agent)

    # Agent should NOT schedule without time — it should ask
    Expect(traces).tool_invocations.to_not_include("schedule_meeting")
    Expect(traces).agent_text_response.to_include("time")

def test_schedule_meeting_with_complete_info(assistant_agent):
    """Agent should call schedule_meeting when all info is provided."""
    case = Case(user_inputs=[
        "Schedule a meeting with Alice tomorrow at 2pm in Room B"
    ])
    traces = case.run(assistant_agent)

    Expect(traces).tool_invocations.to_include("schedule_meeting")

def test_schedule_meeting_conflict_handling(assistant_agent):
    """Agent should relay conflict info and suggest alternatives."""
    case = Case(user_inputs=[
        "Schedule a meeting with Alice tomorrow at 2pm",
        "How about 3pm instead?",
    ])
    traces = case.run(assistant_agent)

    Expect(traces).tool_invocations.to_include("schedule_meeting")
    Expect(traces).tool_invocations.to_include("check_availability")
```

## Best Practices

- **Do:** Start with mocked unit tests for each tool in isolation. Verify the agent selects the right tool for a given input before testing multi-tool chains.
- **Do:** Use `Permute` to test across multiple system prompts and model variants simultaneously, catching prompt fragility early:
  ```python
  agent_parameters={
      "system_prompt": Permute([prompt_v1, prompt_v2]),
      "model_id": Permute(["claude-sonnet", "claude-haiku"])
  }
  ```
- **Do:** Store passing trace snapshots as regression baselines. When you change a prompt or upgrade a model, diff the new traces against baselines.
- **Do:** Test negative paths — verify the agent does NOT call dangerous tools (e.g., `delete_account`) without explicit user confirmation.
- **Avoid:** Writing only acceptance-level tests with live LLMs. These are slow, expensive, and flaky due to model non-determinism. Reserve them for final validation.
- **Avoid:** Asserting on exact text matches for agent responses. LLM text varies; assert on tool invocations and use `to_include` for key phrases rather than `to_equal` for full responses.

## Error Handling

- **Mock response mismatch:** If the mock doesn't provide enough responses for the number of LLM turns the agent needs, the test will fail with an index error. Ensure your mock response sequence matches the expected conversation length.
- **Tool not registered:** If the mock LLM returns a tool-use block for a tool that isn't registered on the agent, the framework raises an error. Always register all tools (or stubs) before running test cases.
- **Flaky acceptance tests:** If live-LLM tests pass intermittently, move the assertion logic down to the integration layer with mocks. Only keep semantic similarity checks (cosine similarity, BLEU) at the acceptance layer.
- **Trace assertion failures:** When `Expect(traces).tool_invocations.to_include("X")` fails, inspect the full trace to see which tools were actually called. Print `[t.span_name for t in traces]` to debug.
- **Multi-turn state issues:** Call `agent.reset()` between test cases to clear conversation history, preventing state leakage between tests.

## Limitations

- The reference implementation (`generative-ai-toolkit`) targets Amazon Bedrock's Converse API. Adapting to other providers (OpenAI, Anthropic direct, local models) requires writing custom mock classes and trace collectors.
- Mocked tests verify that the agent follows a predetermined path — they cannot catch emergent misbehavior that only appears with real LLM reasoning. A balanced pyramid with some live-LLM tests is still necessary.
- Trace-based assertions work best for tool-using agents. Pure conversational agents without tool calls have fewer structural properties to assert on — semantic similarity metrics are more appropriate there.
- The `Permute` feature for cross-testing prompts and models produces a combinatorial explosion. Limit permutations to 2-3 dimensions to keep test suites manageable.
- Custom trace attributes require instrumenting your tool functions with `AgentContext.current().tracer` — this is a code change in production tools, not just test code.

## Reference

**Paper:** [Automated structural testing of LLM-based agents: methods, framework, and case studies](https://arxiv.org/abs/2601.18827v1) — Kohl et al., IEEE BigData 2025. Focus on Section III (methods: traces, mocking, assertions), Section IV (test automation pyramid adaptation), and Section V (case studies demonstrating faster root-cause analysis).

**Code:** [github.com/awslabs/generative-ai-toolkit](https://github.com/awslabs/generative-ai-toolkit) — Install with `pip install "generative-ai-toolkit[all]"`. See `examples/genai_toolkit_getting_started.ipynb` for a complete walkthrough.