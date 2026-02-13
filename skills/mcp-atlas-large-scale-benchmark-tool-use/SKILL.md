---
name: "mcp-atlas-large-scale-benchmark-tool-use"
description: "Design and evaluate multi-server MCP tool-use benchmarks using claims-based scoring rubrics. Use when: 'benchmark my MCP agent', 'evaluate tool-use across servers', 'build a claims-based rubric for tool calls', 'test multi-step MCP workflows', 'score agent tool discovery', 'diagnose tool-use failures'."
---

# MCP-Atlas: Large-Scale Benchmark Design for Multi-Server Tool-Use Evaluation

This skill enables Claude to design, build, and evaluate rigorous benchmarks for MCP tool-use agents following the MCP-Atlas methodology. Instead of naive pass/fail testing with hand-picked tools, you apply claims-based scoring rubrics, distractor-augmented tool surfaces, and structured failure diagnostics to measure how well an agent discovers, parameterizes, and orchestrates tools across multiple real MCP servers. This is the standard for evaluating whether an AI agent can actually use tools in the wild.

## When to Use

- When building a benchmark or test suite that evaluates an agent's ability to use MCP tools across multiple servers
- When designing evaluation rubrics for multi-step tool-calling workflows (e.g., "fetch weather, geocode the location, then compute a route")
- When diagnosing why an agent fails at tool-use tasks -- distinguishing tool discovery errors from parameterization errors from task understanding gaps
- When writing natural-language evaluation prompts that deliberately avoid naming specific tools, forcing the agent to discover them
- When implementing partial-credit scoring for agentic workflows instead of binary pass/fail
- When setting up containerized, reproducible test harnesses for MCP server integration testing
- When selecting distractor tools to stress-test an agent's ability to pick the right tool from a noisy surface

## Key Technique

### Claims-Based Scoring Over Binary Pass/Fail

MCP-Atlas rejects binary scoring. Each task defines a list of **atomic, independently verifiable claims** grounded exclusively in tool outputs. A claim like "The distance between A and B is 142 km" is scored as fulfilled (1.0), partially fulfilled (0.5), or not fulfilled (0). Task coverage equals the sum of claim scores divided by total claims, with a pass threshold at coverage >= 0.75. This decouples scoring from the specific execution path -- an agent that takes a different route to the correct answer still receives full credit, while an agent that gets partial results receives proportional credit.

### Distractor-Augmented Tool Surfaces

Real agents face noisy tool catalogs, not curated minimal sets. Each MCP-Atlas task exposes 10-25 tools: 3-7 target tools needed for the solution plus 5-10 strategically selected **distractors** from the same servers or similar categories (e.g., `google-maps_distance_matrix` alongside `google-maps_geocode` when only one is needed). This forces genuine tool discovery rather than brute-force enumeration and measures discovery precision/recall as distinct metrics.

### Structured Failure Diagnostics

Beyond the final score, MCP-Atlas categorizes failures into four buckets: **Tool Usage** (56.7% of failures -- wrong tool selected, bad parameters, wrong sequencing), **Task Understanding** (30.3% -- premature stopping, missed subgoals), **Response Quality** (8.5% -- correct tool calls but bad synthesis), and **Logical Errors** (4.5% -- flawed conditional reasoning). These diagnostics reveal exactly where an agent's tool-use pipeline breaks down, enabling targeted improvement.

## Step-by-Step Workflow

### Designing a Benchmark

1. **Select MCP servers and enumerate their tools.** Catalog each server's available tools with full JSON schemas, parameter types, and return formats. Group servers into domain buckets (Basic, Analytics, Productivity, Financial, Coding) to enable domain-stratified analysis.

2. **Author natural-language task prompts that never name specific tools or servers.** Write single-turn prompts requiring 3-6 tool calls across 2+ servers. Example: instead of "Use brave_search to find X, then use weather_get_forecast for Y," write "What's the weekend weather forecast for the city that won the 2024 Champions League final?"

3. **Define the reference trajectory for each task.** Document the minimal sequence of tool invocations including inter-call dependencies (e.g., "output of call 1 feeds parameter of call 3"). This is for diagnostic analysis only -- it is not the only valid execution path.

4. **Write atomic claims for each task.** Break the expected final answer into independently verifiable factual claims. Each claim must be grounded in a specific tool output, not general knowledge. Aim for 3-8 claims per task scaling with complexity.

5. **Select distractor tools for each task's tool surface.** For every target tool, include 2-3 plausible distractors from the same server or domain (e.g., if the task needs `filesystem_read_file`, also expose `filesystem_list_directory` and `filesystem_search_files`). Target 10-25 total tools per task.

6. **Containerize each MCP server with sandboxed state.** Pin server versions, isolate filesystems, allow-list network egress, and restart containers per evaluation run to ensure clean, reproducible state.

7. **Implement the claims-based scorer.** For each agent response, evaluate every claim as fulfilled (1.0), partially fulfilled (0.5), or not fulfilled (0). Compute coverage = sum(claim_scores) / len(claims). Pass if coverage >= 0.75.

8. **Capture diagnostic telemetry on every tool call.** Log: which tools were attempted (discovery precision/recall), parameter correctness (schema-aware comparison), error rates, recovery rates (did the agent retry after an error?), and total call count including duplicates (efficiency).

9. **Run evaluation across the full task set and compute stratified metrics.** Report pass rate and mean coverage per domain bucket. Break down failures into the four diagnostic categories (Tool Usage, Task Understanding, Response Quality, Logical Errors).

10. **Iterate on failure hotspots.** Use the diagnostic breakdown to target improvements: if Tool Usage dominates, improve tool descriptions or discovery mechanisms; if Task Understanding dominates, improve prompt decomposition; if Response Quality dominates, improve answer synthesis.

## Concrete Examples

**Example 1: Designing a Cross-Server Benchmark Task**

User: "Help me create an MCP benchmark task that tests whether an agent can combine search and geolocation tools."

Approach:
1. Write a natural prompt: "Find the headquarters address of the company that developed PyTorch, then tell me the current weather there and the driving distance from MIT."
2. Identify required tools: `brave_search` (find Meta HQ), `google-maps_geocode` (resolve address), `weather_get_forecast` (current conditions), `google-maps_distance_matrix` (driving distance).
3. Define reference trajectory with dependencies: search -> geocode(search.result.address) -> weather(geocode.lat, geocode.lng) + distance(origin="MIT", dest=geocode.address).
4. Write atomic claims:
   - "Meta (formerly Facebook) developed PyTorch"
   - "Meta headquarters is at 1 Hacker Way, Menlo Park, CA"
   - "Current temperature at Menlo Park is reported" (value grounded in tool output)
   - "Driving distance from MIT to Menlo Park is reported in miles or km"
5. Add distractors: `brave_search_images`, `google-maps_elevation`, `google-maps_place_details`, `weather_get_alerts`.

Output task schema:
```json
{
  "task_id": "geo-weather-001",
  "prompt": "Find the headquarters address of the company that developed PyTorch, then tell me the current weather there and the driving distance from MIT.",
  "tool_surface": [
    "brave_search", "brave_search_images",
    "google-maps_geocode", "google-maps_elevation", "google-maps_place_details", "google-maps_distance_matrix",
    "weather_get_forecast", "weather_get_alerts"
  ],
  "target_tools": ["brave_search", "google-maps_geocode", "weather_get_forecast", "google-maps_distance_matrix"],
  "reference_trajectory": [
    {"tool": "brave_search", "depends_on": []},
    {"tool": "google-maps_geocode", "depends_on": ["brave_search"]},
    {"tool": "weather_get_forecast", "depends_on": ["google-maps_geocode"]},
    {"tool": "google-maps_distance_matrix", "depends_on": ["google-maps_geocode"]}
  ],
  "claims": [
    "Meta (formerly Facebook) developed PyTorch",
    "Meta headquarters is located at 1 Hacker Way, Menlo Park, CA",
    "Current weather conditions at Menlo Park are reported with temperature",
    "Driving distance from MIT campus to Meta HQ is reported"
  ],
  "pass_threshold": 0.75
}
```

**Example 2: Implementing a Claims-Based Scorer**

User: "Write a scoring function for my MCP benchmark that gives partial credit."

Approach:
1. Accept agent's final text response and the task's claims list.
2. For each claim, use an LLM judge to classify as fulfilled/partial/not fulfilled.
3. Compute coverage score and compare against threshold.

```python
from dataclasses import dataclass

@dataclass
class ClaimResult:
    claim: str
    score: float  # 1.0, 0.5, or 0.0
    evidence: str  # excerpt from agent response supporting the score

def score_claims(agent_response: str, claims: list[str], judge_fn) -> dict:
    """Score an agent response against a list of atomic claims.

    Args:
        agent_response: The agent's final text answer.
        claims: List of atomic, verifiable claim strings.
        judge_fn: Callable(claim, response) -> ClaimResult
    """
    results = [judge_fn(claim, agent_response) for claim in claims]
    coverage = sum(r.score for r in results) / len(results)
    return {
        "coverage": coverage,
        "passed": coverage >= 0.75,
        "claim_results": results,
        "fulfilled": sum(1 for r in results if r.score == 1.0),
        "partial": sum(1 for r in results if r.score == 0.5),
        "missed": sum(1 for r in results if r.score == 0.0),
    }
```

**Example 3: Diagnosing Agent Failures from Tool Call Logs**

User: "My agent only passes 40% of benchmark tasks. Help me figure out why."

Approach:
1. Parse tool call logs to extract attempted tools, parameters, errors, and retries.
2. Classify each failed task into the four failure categories.
3. Compute distribution and identify the dominant failure mode.

```python
def classify_failure(task, agent_trace) -> str:
    """Classify a failed task into one of four diagnostic categories."""
    target_tools = set(task["target_tools"])
    called_tools = {call["tool"] for call in agent_trace["calls"]}
    errors = [c for c in agent_trace["calls"] if c.get("error")]
    retries = [c for c in agent_trace["calls"] if c.get("is_retry")]

    # Tool Usage: wrong tools selected, bad params, or sequencing errors
    if not target_tools.issubset(called_tools):
        return "tool_usage"  # missed required tools
    if any(c.get("param_error") for c in agent_trace["calls"]):
        return "tool_usage"  # parameterization failures

    # Task Understanding: agent stopped early or missed subgoals
    if agent_trace["stopped_early"] or agent_trace["claims_attempted"] < len(task["claims"]):
        return "task_understanding"

    # Response Quality: tools called correctly but answer synthesis failed
    if agent_trace["all_tools_succeeded"] and not agent_trace["passed"]:
        return "response_quality"

    return "logical_error"  # conditional reasoning failure

def compute_failure_distribution(failures: list[dict]) -> dict:
    counts = {"tool_usage": 0, "task_understanding": 0, "response_quality": 0, "logical_error": 0}
    for f in failures:
        counts[f["category"]] += 1
    total = len(failures)
    return {k: {"count": v, "pct": round(100 * v / total, 1)} for k, v in counts.items()}
```

If Tool Usage dominates (>50%), focus on improving tool descriptions, adding few-shot examples of correct tool selection, or improving schema-aware parameter generation. If Task Understanding dominates (>30%), work on prompt decomposition and subgoal tracking.

## Best Practices

- **Do:** Write claims that are atomic and independently verifiable. "The temperature in Paris is 18C and the humidity is 65%" should be two separate claims, not one.
- **Do:** Include distractors from the same server as target tools. A distractor from an unrelated domain (e.g., a Slack tool when testing weather) is too easy to filter out.
- **Do:** Design prompts with implicit multi-hop dependencies where the output of one tool feeds the input of another. This tests genuine orchestration, not parallel independent lookups.
- **Do:** Pin MCP server versions and restart containers between runs. State leakage across tasks invalidates results.
- **Avoid:** Naming specific tools or servers in the task prompt. "Use brave_search to find..." bypasses tool discovery entirely.
- **Avoid:** Using a single binary pass/fail metric. Claims-based partial credit reveals whether an agent is close to solving a task or completely off track, which is critical for iterative improvement.
- **Avoid:** Setting the tool surface to only the exact required tools. Without distractors, you measure tool invocation, not tool discovery.

## Error Handling

- **Flaky MCP servers:** Real APIs have intermittent failures. Implement retry logic in the harness (not the agent) and flag tasks where server errors prevented evaluation. Exclude these from scoring rather than penalizing the agent.
- **Claims ambiguity:** If a claim can be satisfied by semantically equivalent but textually different answers (e.g., "142 km" vs "142 kilometers" vs "88.3 miles"), use a judge model with explicit normalization instructions. The MCP-Atlas paper found Gemini 2.5 Pro achieved 78% agreement with human annotators -- validate your judge against human labels on a sample.
- **Tool schema changes:** When MCP servers update their APIs, tasks referencing old parameter names will fail at the harness level, not the agent level. Version-pin servers and maintain a compatibility matrix.
- **Agent exceeds tool surface:** If the agent attempts a tool not in the task's exposed set, the harness should return a clear "tool not available" error (not a cryptic failure) so the agent can recover.

## Limitations

- **Single-turn only:** MCP-Atlas tasks are single-turn prompts. This does not evaluate dynamic replanning, multi-turn clarification, or interactive debugging -- all critical real-world agent capabilities.
- **Read-only operations:** The benchmark excludes write-based workflows (creating files, sending messages, modifying databases). Agents that excel at read-only orchestration may still fail at stateful mutations.
- **No latency or cost budgets:** The benchmark does not enforce time limits or API call budgets. An agent making 50 redundant calls to arrive at the right answer receives full credit, which misrepresents production readiness.
- **Judge model dependency:** Claims scoring relies on an LLM judge. For domains requiring precise numerical answers or domain-specific formatting, supplement with deterministic validators.
- **Scale ceiling:** 36 servers and 220 tools is substantial but still a fraction of the MCP ecosystem. Results may not generalize to highly specialized or niche tool domains.

## Reference

**Paper:** [MCP-Atlas: A Large-Scale Benchmark for Tool-Use Competency with Real MCP Servers](https://arxiv.org/abs/2602.00933v1) (Bandi et al., 2026). Focus on Section 3 (Task Schema & Claims Rubric), Section 4 (Distractor Selection & Tool Surface Design), and Table 3 (Failure Diagnostic Breakdown) for the most actionable implementation guidance.