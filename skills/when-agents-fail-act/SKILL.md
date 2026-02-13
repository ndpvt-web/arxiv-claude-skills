---
name: "when-agents-fail-act"
description: >
  Diagnose and fix tool invocation failures in multi-agent LLM systems using a 12-category error taxonomy
  covering tool initialization, parameter handling, execution, and result interpretation.
  Use when: "debug agent tool calls", "why is my agent failing to call tools",
  "diagnose tool use errors", "improve agent reliability", "audit multi-agent tool invocations",
  "evaluate tool-use reliability across models".
---

# Diagnostic Framework for Tool Invocation Reliability in Multi-Agent LLM Systems

This skill enables Claude to systematically diagnose, classify, and fix failures in LLM-based agent tool invocations using the 12-category error taxonomy from Huang, Malwe & Wang (2026). Rather than treating tool-call failures as opaque errors, this framework decomposes them into four distinct phases—initialization, parameter handling, execution, and result interpretation—each with specific failure modes and targeted fixes. Apply this to debug agent pipelines, evaluate model reliability for tool use, and set production readiness thresholds.

## When to Use

- When an LLM agent fails to invoke a tool and you need to diagnose whether the failure is in initialization, parameters, execution, or result parsing
- When building or reviewing a multi-agent system and you need to audit tool-call reliability across the pipeline
- When choosing between models (open-weight vs proprietary) for a tool-calling agent and need reliability benchmarks
- When a smaller or local model is failing at tool use and you need to identify if tool initialization is the bottleneck
- When deploying agents on edge or commodity hardware and you need to validate accuracy-latency trade-offs
- When writing automated tests for agent tool invocation and need a systematic error taxonomy to cover
- When triaging production incidents where agents silently fail to act on user requests

## Key Technique

The framework's core insight is that tool invocation is not a single atomic action—it is a four-phase pipeline, and failures cluster differently depending on model size, quantization, and hardware. The four phases are:

**Phase 1 — Tool Initialization:** The agent must recognize that a tool call is needed, select the correct tool from its available set, and format the invocation intent. This is where smaller models fail most often—they either skip the tool call entirely (acting as if no tool exists) or hallucinate a nonexistent tool name. **Phase 2 — Parameter Handling:** The agent must extract, validate, and correctly format all required parameters. Common failures include type mismatches (passing a string where an integer is expected), missing required fields, and hallucinated parameter names. **Phase 3 — Execution:** The tool runs but encounters runtime errors—API timeouts, permission failures, malformed payloads that passed validation but fail at the endpoint. **Phase 4 — Result Interpretation:** The agent receives the tool's response but misparses it, ignores errors in the response, hallucinates content not present in the result, or fails to chain the output into the next step.

The 12-category taxonomy distributes three specific error types across each phase, creating a structured diagnostic matrix. The actionable finding is that reliability is not uniform: tool initialization failures dominate for models under ~14B parameters, while larger models (32B+) match proprietary model performance. For production systems, this means you can diagnose the *phase* of failure first, then apply targeted mitigations rather than blindly switching models or adding retries.

## Step-by-Step Diagnostic Workflow

1. **Reproduce the failure deterministically.** Capture the exact prompt, tool definitions (JSON schema), model identifier, and system prompt. Run the same input at temperature 0 at least 3 times to distinguish stochastic from deterministic failures.

2. **Classify the failure phase.** Inspect the raw model output (before any framework parsing). Determine which phase failed:
   - *No tool call emitted at all* → Phase 1 (Initialization)
   - *Tool call present but malformed parameters* → Phase 2 (Parameter Handling)
   - *Tool call well-formed but execution returns error* → Phase 3 (Execution)
   - *Tool executed successfully but agent misuses the result* → Phase 4 (Result Interpretation)

3. **Apply the error sub-category.** Within each phase, identify the specific failure mode:
   - **Phase 1:** (a) Tool omission—agent responds in plain text ignoring available tools; (b) Wrong tool selection—agent picks a different tool than intended; (c) Hallucinated tool—agent invokes a tool not in its schema
   - **Phase 2:** (a) Missing required parameters; (b) Type/format mismatch (e.g., `"42"` instead of `42`); (c) Hallucinated parameters—agent adds fields not in the schema
   - **Phase 3:** (a) Timeout or rate limit; (b) Authentication/permission error; (c) Payload rejected by downstream API
   - **Phase 4:** (a) Ignored error response—agent proceeds as if call succeeded; (b) Hallucinated result—agent fabricates data not in the response; (c) Incomplete extraction—agent uses only part of the result, dropping critical fields

4. **Check against known model-specific patterns.** Consult the reliability profile:
   - Models <7B: Expect >15% Phase 1 failures (tool omission). Mitigate with explicit "You MUST use a tool" system prompt reinforcement and few-shot tool-call examples.
   - Models 7B-14B: Phase 1 drops significantly (~96.6% success at 14B). Phase 2 errors become the primary concern. Mitigate with stricter schema validation and parameter examples.
   - Models 32B+: Match proprietary model performance (~99-100% success). Failures are predominantly Phase 3 (infrastructure) or Phase 4 (complex multi-step chains).

5. **Implement phase-specific mitigations.** Apply targeted fixes rather than generic retries:
   - Phase 1 → Add explicit tool-use instructions in system prompt; reduce tool count to minimize selection confusion; add few-shot examples of correct tool invocations
   - Phase 2 → Add parameter descriptions with examples in the tool schema; use `enum` constraints where possible; validate parameters before execution
   - Phase 3 → Add retry with exponential backoff; implement circuit breakers; validate payloads client-side before sending
   - Phase 4 → Add explicit result-parsing instructions in the prompt; use structured output mode; validate agent's interpretation against the raw tool response

6. **Build a deterministic test suite.** Create test cases covering each of the 12 error categories. For each tool in your system, write at least one test per failure mode:
   - A prompt that should trigger tool use (tests Phase 1)
   - A prompt requiring specific parameter values (tests Phase 2)
   - A mock that returns an error response (tests Phase 3 + 4)

7. **Run the test suite across candidate models.** Execute all tests at temperature 0, record pass/fail per category, and compute per-phase success rates. Use the paper's threshold: **>96% overall success rate** as the minimum for production deployment on non-critical paths; **>99%** for critical paths.

8. **Profile latency alongside accuracy.** Record p50 and p95 latency per tool call. The framework shows that 14B models achieve 7.3s average latency on commodity hardware—use this as a baseline for edge deployment feasibility.

9. **Set up continuous reliability monitoring.** In production, log every tool invocation with its phase outcome (success, or which phase/category failed). Alert when any single error category exceeds its baseline rate by >2x.

10. **Iterate model selection based on data.** If Phase 1 errors dominate, upgrade the model. If Phase 2 errors dominate, improve your tool schemas. If Phase 3 errors dominate, fix your infrastructure. If Phase 4 errors dominate, improve your result-parsing prompts.

## Concrete Examples

**Example 1: Diagnosing why a local agent ignores tools**

```
User: "My Ollama agent running qwen2.5:7b just responds in plain text
when I ask it to search the database. It never calls the search tool."

Approach:
1. Classify: Agent produces text without any tool call → Phase 1 failure,
   sub-category (a) Tool omission
2. Check model profile: 7B model → high Phase 1 failure rate expected
3. Inspect system prompt: Confirm tool definitions are provided in the
   correct format (JSON schema with name, description, parameters)
4. Apply Phase 1 mitigations:
   - Add to system prompt: "You MUST use the search_database tool to
     answer questions about data. NEVER answer from memory."
   - Add a few-shot example showing the correct tool call format
   - Reduce available tools from 12 to the 3 most relevant
5. Re-test at temperature 0 with 5 runs

Output:
Diagnosis: Phase 1 / Tool Omission
Root cause: 7B model struggles with tool initialization when presented
with many tools and no few-shot examples.
Fix: Added explicit tool-use instruction + few-shot example + reduced
tool count from 12 to 3. Success rate improved from ~60% to ~92%.
Recommendation: For >96% reliability, upgrade to qwen2.5:14b.
```

**Example 2: Auditing a multi-agent pipeline before production**

```
User: "We have a 5-agent pipeline using Claude 3.5 Sonnet for customer
support. Each agent calls 2-3 tools. How do I evaluate reliability
before going live?"

Approach:
1. Inventory all tools across all 5 agents (expect 10-15 total)
2. For each tool, create test cases covering all 4 phases:
   - Phase 1: Prompt that should trigger the tool (verify it fires)
   - Phase 2: Prompt requiring edge-case parameters (empty strings,
     Unicode, large numbers, optional fields)
   - Phase 3: Mock returning HTTP 500, timeout, auth error
   - Phase 4: Mock returning valid-but-tricky responses (empty arrays,
     null fields, paginated results)
3. Run the full suite: 15 tools x 12 categories = 180 test cases
4. Execute each test 3x at temperature 0 = 540 runs
5. Score per-phase and per-agent success rates

Output:
| Agent        | Phase 1 | Phase 2 | Phase 3 | Phase 4 | Overall |
|-------------|---------|---------|---------|---------|---------|
| Router      | 100%    | 98.3%   | 100%    | 96.7%   | 98.8%   |
| Lookup      | 100%    | 100%    | 94.4%   | 100%    | 98.6%   |
| Resolver    | 100%    | 96.7%   | 100%    | 93.3%   | 97.5%   |
| Escalation  | 100%    | 100%    | 100%    | 100%    | 100%    |
| Summary     | 100%    | 100%    | 100%    | 96.7%   | 99.2%   |

Findings: All agents exceed 96% threshold. Resolver has Phase 4
weakness (misparses multi-item responses). Fix: add explicit parsing
instructions for array responses in Resolver's system prompt.
```

**Example 3: Choosing a model for edge deployment**

```
User: "We need to run an agent on a Jetson Orin (16GB RAM) that calls
3 tools. Which model should we use?"

Approach:
1. Hardware constraint: 16GB RAM limits to models ≤14B (quantized)
2. Candidate models: qwen2.5:7b (Q4), qwen2.5:14b (Q4), functionary-small
3. Create test suite: 3 tools x 12 error categories = 36 tests
4. Run on target hardware, recording accuracy and latency
5. Apply framework thresholds

Output:
| Model           | Success Rate | Avg Latency | Phase 1 Errors | Fits 16GB |
|----------------|-------------|-------------|----------------|-----------|
| qwen2.5:7b-q4  | 88.9%       | 3.1s        | 8.3%           | Yes       |
| qwen2.5:14b-q4 | 96.6%       | 7.3s        | 1.1%           | Yes       |
| functionary-sm  | 93.2%       | 5.8s        | 4.2%           | Yes       |

Recommendation: qwen2.5:14b-q4. Meets the 96% production threshold.
The 7.3s latency is acceptable for non-real-time workflows. If latency
is critical (<4s), use qwen2.5:7b with Phase 1 mitigations and accept
the reliability trade-off, or add a retry layer.
```

## Best Practices

- **Do:** Always classify failures by phase before attempting fixes. A Phase 1 failure (model never calls the tool) requires fundamentally different mitigation than a Phase 2 failure (wrong parameters).
- **Do:** Test at temperature 0 with multiple runs. The framework uses deterministic testing to distinguish model limitations from stochastic variance.
- **Do:** Keep tool schemas minimal—fewer tools with clear descriptions reduce Phase 1 errors. The data shows tool selection confusion increases with tool count, especially for smaller models.
- **Do:** Include concrete parameter examples in tool descriptions. This directly reduces Phase 2 (parameter handling) failures across all model sizes.
- **Avoid:** Generic retry-on-failure without diagnosis. Retrying a Phase 1 failure (tool omission) with the same prompt will produce the same result at temperature 0. You must change the prompt or model.
- **Avoid:** Assuming proprietary models are immune. Even GPT-4 and Claude 3.5 show Phase 4 (result interpretation) failures on complex multi-step tool chains. Always test.
- **Avoid:** Over-indexing on overall success rate. A model at 97% overall but with 15% Phase 4 failures on a critical tool is worse than a model at 95% overall with evenly distributed errors.

## Error Handling

| Situation | Response |
|-----------|----------|
| Agent never produces tool calls despite correct schema | Phase 1 / Tool Omission: Strengthen system prompt, add few-shot examples, reduce tool count. If persistent, the model is too small—upgrade. |
| Agent calls the wrong tool consistently | Phase 1 / Wrong Tool Selection: Tool descriptions are ambiguous. Rewrite descriptions to be mutually exclusive. Rename tools for clarity. |
| Agent invents parameter names not in the schema | Phase 2 / Hallucinated Parameters: Add `additionalProperties: false` to JSON schema. Add few-shot examples showing exact parameter names. |
| Tool executes but agent says "I couldn't find anything" when data was returned | Phase 4 / Ignored Result: The agent failed to parse the response. Add explicit instructions: "After calling the tool, report the exact values returned." |
| Agent fabricates data that wasn't in the tool response | Phase 4 / Hallucinated Result: Add a verification step—instruct the agent to quote directly from the tool response. Log and compare raw responses vs agent output. |
| Intermittent failures (works sometimes, fails sometimes) | Temperature > 0 or non-deterministic infrastructure. Fix temperature to 0 for diagnosis. If still intermittent, check for race conditions in multi-agent orchestration. |

## Limitations

- The taxonomy is derived from 1,980 test instances across a specific set of models (Qwen2.5 series, Functionary, GPT-4, Claude 3.5/3.7). Newer models may introduce failure modes not captured in the 12 categories.
- The framework focuses on single tool calls. Complex scenarios involving parallel tool calls, tool-call chaining with branching logic, or tools that return streaming results may require extended classification.
- Reliability thresholds (96%, 99%) are derived from the paper's SME-centric deployment context. High-stakes domains (medical, financial) may require higher thresholds and additional validation layers.
- Edge hardware benchmarks (7.3s latency on 14B model) depend on specific hardware configurations. Your latency will vary with GPU, quantization method, batch size, and concurrent load.
- The framework evaluates procedural reliability (did the tool get called correctly?) but does not assess semantic correctness (did the agent call the *right* tool for the user's actual intent?).

## Reference

Huang, D., Malwe, G., & Wang, Z. (2026). "When Agents Fail to Act: A Diagnostic Framework for Tool Invocation Reliability in Multi-Agent LLM Systems." *ICAIBD 2026*. [arXiv:2601.16280](https://arxiv.org/abs/2601.16280) — Read for the full 12-category error taxonomy, per-model reliability benchmarks, and edge hardware performance profiles.