---
name: "tokenomics-quantifying-where-tokens"
description: "Analyze and optimize token consumption in LLM-based multi-agent software engineering workflows. Maps agent execution traces to SDLC stages (Design, Coding, Code Completion, Code Review, Testing, Documentation) and quantifies where tokens are spent. Use when: 'analyze token usage in my agent pipeline', 'where are tokens being wasted in my workflow', 'optimize my multi-agent system costs', 'profile my agentic coding pipeline', 'reduce LLM costs in my CI/CD agents', 'tokenomics analysis of my agent framework'."
---

# Tokenomics: Token Consumption Analysis for Agentic Software Engineering

This skill enables Claude to profile, analyze, and optimize token consumption in LLM-based multi-agent (LLM-MA) software engineering systems. Using the methodology from Salim et al. (2026), it maps agent execution traces to standardized SDLC stages and quantifies token distribution across input, output, and reasoning tokens — revealing that iterative code review consumes ~59% of all tokens while initial code generation uses only ~9%. This drives concrete optimization decisions: where to add caching, where to truncate context, and where human checkpoints prevent runaway costs.

## When to Use

- When the user wants to understand why their multi-agent coding pipeline is expensive and where tokens are going
- When profiling a ChatDev, MetaGPT, AutoGen, CrewAI, or custom agent framework to find cost hotspots
- When designing a new multi-agent workflow and wanting to predict token costs per development stage
- When the user asks to instrument an agent system with token tracking and reporting
- When optimizing an existing agentic pipeline by reducing redundant context passing between agents
- When comparing token efficiency across different agent architectures or LLM models for the same tasks
- When building dashboards or reports that break down LLM spend by software development activity

## Key Technique

The core method is **SDLC-stage token mapping**: rather than treating an agent pipeline as a black box with a single cost number, you decompose execution into six canonical stages — Design, Coding, Code Completion, Code Review, Testing, and Documentation — and measure input, output, and reasoning tokens independently at each stage. This reveals that cost is not uniformly distributed. The paper's analysis of 30 tasks on ChatDev with GPT-5 found Code Review alone consumes 59.4% of all tokens, while Design uses only 2.4%. This is because review involves iterative multi-turn dialogue where full code context is re-sent on every round.

The second key insight is the **input token dominance problem** (called the "communication tax"): input tokens average 53.9% of total consumption across all stages. This happens because agents pass full accumulated context to each other on every turn. The ratio varies dramatically by stage — Documentation is 80.2% input tokens (mostly reading existing code to summarize it), while Coding is only 6.9% input tokens (mostly generating new code). Understanding these per-stage ratios tells you exactly where context compression, summarization, or caching will have the highest ROI.

The actionable framework is: (1) instrument your agent traces to log token counts per LLM call with stage labels, (2) aggregate by SDLC stage, (3) compute input/output/reasoning breakdowns, (4) compare against the baseline distribution to identify anomalies, and (5) apply targeted optimizations to the highest-cost stages first.

## Step-by-Step Workflow

1. **Instrument agent traces.** Add logging around every LLM API call in the pipeline to capture: the phase/stage label, input token count, output token count, reasoning token count (if using a reasoning model), the prompt text, and the response text. Store traces as structured JSON (one record per API call).

2. **Map framework phases to SDLC stages.** Create a mapping from your framework's internal phase names to the six canonical stages: Design (requirements, architecture decisions), Coding (initial code generation), Code Completion (filling placeholders/stubs), Code Review (iterative review and modification loops), Testing (running tests, fixing bugs), Documentation (manuals, READMEs, environment docs). If a framework phase spans two stages, split it.

3. **Parse and aggregate token counts.** Write a script that reads the trace logs, groups API calls by their SDLC stage, and sums input, output, and reasoning tokens per stage. Calculate both absolute counts and percentages of total.

4. **Compute per-stage token type ratios.** For each stage, calculate the proportion of input vs. output vs. reasoning tokens. High input ratios (>60%) indicate context-passing overhead; high output ratios indicate generative work; high reasoning ratios indicate complex decision-making.

5. **Compare against the baseline distribution.** Use the empirical baseline from the paper as a reference point: Design ~2.4%, Coding ~8.6%, Code Completion ~26.8%, Code Review ~59.4%, Testing ~10.3%, Documentation ~20.1%. Significant deviations indicate either different task complexity or potential inefficiencies.

6. **Identify the top cost driver.** Rank stages by total token consumption. The highest-consuming stage is your primary optimization target. In most pipelines, this will be Code Review or similar iterative refinement loops.

7. **Analyze iteration depth in the top cost stage.** Count how many LLM round-trips occur in the most expensive stage. Each round-trip in Code Review re-sends the full conversation history and code, compounding input token costs. Log the number of review iterations per task.

8. **Apply targeted optimizations.** Based on findings, implement one or more: (a) context window compression — summarize prior turns instead of passing full history, (b) iteration caps — limit review rounds to 3-5 with human escalation, (c) diff-only passing — send only changed code sections rather than full files, (d) human-in-the-loop checkpoints before expensive iterative stages.

9. **Generate a cost report.** Produce a table or visualization showing token distribution by stage with input/output/reasoning breakdown, estimated cost in dollars (using the model's per-token pricing), and comparison to baseline percentages.

10. **Re-profile after optimization.** Run the same tasks with optimizations applied and compare the new token distribution against the pre-optimization profile to measure actual savings.

## Concrete Examples

**Example 1: Profiling a ChatDev-style pipeline**

User: "I'm running a multi-agent coding system and my API costs are way higher than expected. Can you help me figure out where the tokens are going?"

Approach:
1. Ask the user for their agent execution logs or help them add instrumentation
2. Parse the logs to extract per-call token counts with phase labels
3. Map phases to the six SDLC stages
4. Aggregate and produce a breakdown

Output:
```
Token Consumption Report — 30 Task Average
═══════════════════════════════════════════════════════════════
Stage            | Input   | Output  | Reasoning | Total  | %
─────────────────|─────────|─────────|───────────|────────|──────
Design           |   1,842 |     110 |     1,098 |  3,050 |  2.4%
Coding           |     760 |   6,380 |     3,860 | 11,000 |  8.6%
Code Completion  |  16,310 |  14,262 |     3,591 | 34,163 | 26.8%
Code Review      |  38,917 |  18,702 |  18,096   | 75,715 | 59.4%
Testing          |   7,984 |   2,722 |     2,419 | 13,125 | 10.3%
Documentation    |  20,563 |   2,129 |     2,948 | 25,640 | 20.1%
─────────────────|─────────|─────────|───────────|────────|──────
TOTAL            |  86,376 |  44,305 |    32,012 |162,693 |
Token type share |  53.1%  |  27.2%  |    19.7%  |        |

⚠ Code Review consumes 59.4% of all tokens.
⚠ Input tokens are 51.4% of Code Review — context re-sending overhead.
→ Recommendation: Cap review iterations at 3 rounds, use diff-only context.
→ Estimated savings: 25-40% of total token budget.
```

**Example 2: Building a token tracking decorator for a Python agent system**

User: "I want to add token tracking to my LangGraph agent pipeline. Help me instrument it."

Approach:
1. Create a decorator/wrapper that intercepts LLM calls
2. Log token counts with SDLC stage metadata
3. Provide aggregation utilities

Output:
```python
import json
import time
from dataclasses import dataclass, field, asdict
from collections import defaultdict
from typing import Literal

SDLCStage = Literal[
    "design", "coding", "code_completion",
    "code_review", "testing", "documentation"
]

@dataclass
class TokenTrace:
    stage: SDLCStage
    input_tokens: int
    output_tokens: int
    reasoning_tokens: int = 0
    timestamp: float = field(default_factory=time.time)

class TokenProfiler:
    def __init__(self):
        self.traces: list[TokenTrace] = []

    def record(self, stage: SDLCStage, response) -> None:
        usage = response.usage
        self.traces.append(TokenTrace(
            stage=stage,
            input_tokens=usage.prompt_tokens,
            output_tokens=usage.completion_tokens,
            reasoning_tokens=getattr(usage, "reasoning_tokens", 0),
        ))

    def report(self) -> dict:
        by_stage = defaultdict(lambda: {"input": 0, "output": 0, "reasoning": 0})
        for t in self.traces:
            by_stage[t.stage]["input"] += t.input_tokens
            by_stage[t.stage]["output"] += t.output_tokens
            by_stage[t.stage]["reasoning"] += t.reasoning_tokens
        total = sum(
            s["input"] + s["output"] + s["reasoning"]
            for s in by_stage.values()
        )
        return {
            stage: {
                **counts,
                "total": counts["input"] + counts["output"] + counts["reasoning"],
                "pct": round(
                    (counts["input"] + counts["output"] + counts["reasoning"])
                    / total * 100, 1
                ) if total > 0 else 0,
            }
            for stage, counts in by_stage.items()
        }

    def save(self, path: str) -> None:
        with open(path, "w") as f:
            json.dump([asdict(t) for t in self.traces], f, indent=2)

# Usage in an agent pipeline:
profiler = TokenProfiler()

# During code review stage:
response = client.chat.completions.create(model="gpt-4o", messages=messages)
profiler.record("code_review", response)

# At the end:
print(json.dumps(profiler.report(), indent=2))
profiler.save("token_traces.json")
```

**Example 3: Optimizing an expensive code review loop**

User: "My agent's code review loop is eating 70% of my token budget. How do I cut it down?"

Approach:
1. Analyze the review loop structure — count iterations and context growth per round
2. Implement diff-only context passing
3. Add an iteration cap with graceful fallback

Output:
```
Analysis of Code Review Token Growth:
  Round 1: 12,400 tokens (base code + initial review prompt)
  Round 2: 24,100 tokens (full history + code + round 1 feedback)
  Round 3: 35,800 tokens (full history grows linearly)
  Round 4: 47,200 tokens (diminishing review value, rising cost)
  Round 5: 58,500 tokens (marginal improvements only)

Problem: Each round re-sends the ENTIRE conversation history.
Context grows ~12K tokens per round while review value diminishes.

Optimizations applied:
1. Diff-only context: Send only changed lines + 5 lines of surrounding
   context instead of full files → 60% reduction in per-round input tokens
2. Iteration cap at 3 rounds: After 3 rounds, accept code or escalate
   to human review → eliminates rounds 4-5 entirely
3. Summarize prior rounds: Replace full chat history with a structured
   summary of outstanding issues → 40% reduction in history tokens

Expected savings: ~55% reduction in Code Review token consumption,
translating to ~33% reduction in total pipeline cost.
```

## Best Practices

- **Do:** Instrument every LLM call — even "cheap" stages like Design. Small stages provide baseline comparisons that make anomalies in expensive stages visible.
- **Do:** Track input, output, and reasoning tokens separately. A stage that is 80% input tokens (like Documentation) needs context compression, while a stage that is 58% output tokens (like Coding) is already efficient — the model is producing, not re-reading.
- **Do:** Measure iteration counts in review and testing loops. Token cost in iterative stages grows linearly (or worse) with round count. Capping iterations is often the single highest-ROI optimization.
- **Do:** Use the six-stage SDLC mapping even if your framework has different internal phase names. Standardized stages let you compare across frameworks and against published baselines.
- **Avoid:** Optimizing the cheapest stages first. Design at 2.4% of tokens is not worth engineering effort. Focus on Code Review (59.4%) and Code Completion (26.8%) where savings compound.
- **Avoid:** Assuming all token types cost the same. Many APIs charge differently for input vs. output vs. reasoning tokens. Weight your analysis by actual per-token pricing, not just raw counts.

## Error Handling

- **Missing token counts in API responses:** Some API wrappers strip usage metadata. Ensure you access the raw API response or configure your client to return usage stats (e.g., `stream_options={"include_usage": True}` for OpenAI streaming).
- **Unmappable phases:** If your framework has phases that don't cleanly fit one SDLC stage, create a mapping table and document the decision. Partial mappings (splitting one framework phase across two stages) are acceptable if you log the split rationale.
- **Non-reasoning models:** If the model doesn't produce reasoning tokens, set reasoning to 0 and work with input/output only. The input dominance finding (53.9%) still applies and is the primary optimization lever.
- **Low task counts:** The paper uses 30 tasks; with fewer than 10, per-stage percentages will have high variance. Report standard deviations alongside means and avoid drawing conclusions from small samples.
- **Phases that don't always execute:** Code Completion ran in only 6/30 tasks and Testing in 12/30. When computing averages, use only tasks where the phase actually executed, not the full task count.

## Limitations

- The baseline percentages (59.4% Code Review, etc.) were measured on ChatDev with GPT-5 on 30 tasks. Different frameworks, models, or task types will produce different distributions — treat the numbers as a reference, not a universal law.
- The methodology profiles token *quantity* but not token *quality*. A stage might use many tokens productively (generating correct code) or wastefully (re-reading unchanged context). Quality assessment requires separate evaluation.
- Reasoning tokens are only available from models that expose them (e.g., OpenAI's o-series, GPT-5). For models without reasoning token reporting, you lose one dimension of the analysis.
- The framework assumes sequential SDLC stages. Highly concurrent or non-linear agent workflows (e.g., parallel code generation and testing) require adapted aggregation logic.
- Cost optimization recommendations may reduce token usage at the expense of output quality. Always validate that optimized pipelines produce equivalent results before deploying.

## Reference

**Paper:** Salim, M., Latendresse, J., Khatoonabadi, S.H., & Shihab, E. (2026). "Tokenomics: Quantifying Where Tokens Are Used in Agentic Software Engineering." arXiv:2601.14470v1. [https://arxiv.org/abs/2601.14470v1](https://arxiv.org/abs/2601.14470v1)

**Key takeaway:** The primary cost of agentic software engineering is not code generation — it is iterative refinement. Code Review consumes 59.4% of tokens, and input tokens (context re-sending) account for 53.9% of all consumption. Optimize review loops and context passing first.