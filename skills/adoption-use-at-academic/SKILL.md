---
name: "adoption-use-at-academic"
description: >
  Build institutional LLM platforms that integrate with existing data systems (EHR, CRM, ERP)
  using the ChatEHR pattern: model-agnostic automations, interactive UI, hallucination monitoring,
  and value assessment frameworks. Use when asked to "build an LLM platform for our organization",
  "create automations that combine prompts with live data", "design a model-agnostic AI system",
  "set up LLM monitoring and evaluation", "build a ChatEHR-style integration", or
  "create a value assessment framework for AI deployments".
---

# Institutional LLM Platform Builder (ChatEHR Pattern)

This skill enables Claude to design and build institutional LLM platforms following the ChatEHR architecture from Stanford Medicine. The core pattern separates **automations** (static prompt + data combinations for fixed tasks) from **interactive use** (free-form LLM queries over organizational data), wraps both in a model-agnostic execution layer, and adds continuous monitoring for hallucinations, inaccuracies, and business value. This approach eliminates "workflow friction" by pulling data directly from source systems rather than requiring manual entry, making LLM use an organizational capability rather than a standalone tool.

## When to Use

- When the user asks to build an LLM-powered platform that integrates with existing organizational data systems (EHR, CRM, ERP, data warehouses)
- When designing a system that needs to support both fixed automated tasks and free-form interactive LLM queries over the same data
- When the user needs a model-agnostic architecture that can route different tasks to different LLMs based on task requirements
- When building monitoring infrastructure for LLM outputs including hallucination detection and accuracy tracking
- When creating a value assessment framework to quantify ROI of LLM deployments across cost savings, time savings, and revenue impact
- When the user wants to move from ad-hoc LLM tool usage to a governed, institution-wide AI platform
- When designing prompt-and-data pipelines that combine structured and unstructured data for LLM consumption

## Key Technique

The ChatEHR pattern solves a fundamental problem: standalone LLM tools create "workflow friction" because users must manually copy-paste data into prompts. The solution is a **data integration layer** that automatically assembles relevant records (spanning years of history) into LLM-ready context, paired with two interaction modes. **Automations** are pre-built combinations of a specific prompt template, a data query, and a target LLM -- they run a fixed task (e.g., pre-visit chart review, eligibility screening) without user prompt engineering. **Interactive mode** lets trained users write free-form queries against the full data timeline through a UI embedded in the source system.

The model-agnostic design is critical. Different tasks have different requirements: some need long-context models for multi-year timelines, some need faster/cheaper models for simple extraction, some need specific safety profiles. The platform routes each automation to the most appropriate LLM rather than forcing one model for all tasks. This also provides vendor independence -- when a better model appears, you swap it in without changing the automation logic.

Monitoring goes beyond pre-deployment benchmarks. The paper found that static benchmarks were insufficient for real-world evaluation, requiring **continuous production monitoring** that tracks hallucination rates (fabricated information), inaccuracy rates (misrepresented real information), and task-specific quality metrics. Combined with a **value assessment framework** that quantifies savings across three dimensions (cost reduction, time savings, revenue growth), this enables data-driven prioritization of which automations to build next.

## Step-by-Step Workflow

1. **Map the data landscape.** Inventory all data sources the platform needs to access. For each source, document the data types (structured records, free-text notes, images, time-series), access patterns (API, database query, file export), data volume per entity, and time span. Design a unified data access layer that can assemble a complete entity timeline from multiple sources.

2. **Design the automation schema.** Define each automation as a declarative configuration containing: (a) a prompt template with variable placeholders, (b) a data query specification that defines what data to pull and how to structure it, (c) the target LLM model ID, (d) output format specification, and (e) evaluation criteria. Store these as versioned configuration files (YAML or JSON).

3. **Build the data assembly pipeline.** Implement data fetchers for each source system. Create a timeline assembler that merges records from multiple sources into a chronologically ordered, LLM-ready context document. Apply context window management: prioritize recent data, summarize older data, and truncate intelligently when the timeline exceeds the model's context limit.

4. **Implement the model-agnostic execution layer.** Build an abstraction over multiple LLM providers (OpenAI, Anthropic, Google, open-source). Each automation's config specifies which model to use. The execution layer handles prompt assembly (template + data), API calls with retries, response parsing, and structured output extraction. Use a provider interface pattern so new models can be added without changing automation logic.

5. **Create the interactive UI.** Build a query interface embedded in (or linked from) the source system. The UI should: display the assembled entity timeline for context, accept free-form prompts, let users select which data segments to include, show LLM responses with source attribution, and log every session for monitoring. Require user training before granting access.

6. **Implement hallucination and accuracy monitoring.** For each LLM generation, build a verification pipeline: (a) extract factual claims from the output, (b) cross-reference claims against source data, (c) flag claims not grounded in provided data (hallucinations) and claims that misrepresent source data (inaccuracies), (d) compute per-generation metrics. Target baselines: track hallucinations/generation and inaccuracies/generation over time.

7. **Build the value assessment framework.** For each automation or interactive use pattern, quantify impact across three axes: **cost savings** (labor hours replaced x hourly rate), **time savings** (reduction in task completion time x volume), **revenue impact** (faster throughput, reduced errors, new capabilities). Create a scoring model that ranks potential new automations by expected value to prioritize development.

8. **Deploy with governance controls.** Implement role-based access (trained users only for interactive mode, specific roles for each automation), audit logging for all LLM interactions, prompt and response archiving for compliance, model version tracking, and a kill switch per automation. Establish a review process for new automations before production deployment.

9. **Set up continuous evaluation dashboards.** Build monitoring dashboards showing: automation run counts and error rates, interactive session volumes and user adoption curves, hallucination and inaccuracy rates over time (with alerting thresholds), model cost per automation, and value delivered per automation. Use these to detect degradation when models are updated.

10. **Iterate on the automation portfolio.** Use the value assessment framework to identify the next highest-value automation. Follow the pattern: prototype with interactive mode, observe common query patterns, formalize the most valuable patterns into new automations, evaluate, and deploy.

## Concrete Examples

**Example 1: Building a model-agnostic automation system for customer support**

User: "We have a CRM with years of customer interaction history. Build a system where we can create automations that summarize customer history before calls and also let agents query the data interactively."

Approach:
1. Design the data access layer for the CRM (tickets, calls, emails, purchase history)
2. Create the automation config schema
3. Build two initial automations: pre-call summary and account health assessment
4. Build the interactive query UI for agents

Output (automation config):
```yaml
# automations/pre-call-summary.yaml
name: pre_call_summary
version: "1.2"
description: Generate a concise customer summary before support calls
trigger: manual | scheduled_before_call
data_query:
  sources:
    - type: crm_tickets
      filter: "customer_id = {{customer_id}}"
      time_range: "last_2_years"
      fields: [subject, status, resolution, created_at, priority]
    - type: crm_emails
      filter: "customer_id = {{customer_id}}"
      time_range: "last_6_months"
      fields: [subject, body_summary, sentiment, date]
    - type: purchases
      filter: "customer_id = {{customer_id}}"
      time_range: "all"
      fields: [product, amount, date, renewal_date]
  assembly:
    format: chronological_timeline
    max_tokens: 12000
    overflow_strategy: summarize_oldest
model:
  provider: anthropic
  model_id: claude-sonnet-4-20250514
  reason: "Good balance of speed and quality for summarization"
prompt_template: |
  You are a customer support assistant. Given the following customer timeline,
  generate a pre-call briefing covering:
  1. Customer tier and tenure
  2. Recent issues (last 90 days) and their resolution status
  3. Unresolved pain points or recurring themes
  4. Upcoming renewals or at-risk indicators
  5. Recommended talking points

  CUSTOMER TIMELINE:
  {{assembled_timeline}}

  Respond in structured markdown. Only include facts present in the data above.
output:
  format: markdown
  max_length: 800
evaluation:
  check_grounding: true
  required_sections: [recent_issues, renewals, talking_points]
```

```python
# execution/engine.py - Model-agnostic execution layer
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Any
import yaml

@dataclass
class AutomationResult:
    output: str
    model_used: str
    tokens_in: int
    tokens_out: int
    latency_ms: float
    grounding_score: float | None = None

class LLMProvider(ABC):
    @abstractmethod
    async def generate(self, prompt: str, config: dict) -> dict:
        pass

class ProviderRegistry:
    def __init__(self):
        self._providers: dict[str, LLMProvider] = {}

    def register(self, name: str, provider: LLMProvider):
        self._providers[name] = provider

    def get(self, name: str) -> LLMProvider:
        if name not in self._providers:
            raise ValueError(f"Unknown provider: {name}")
        return self._providers[name]

class AutomationEngine:
    def __init__(self, registry: ProviderRegistry, data_assembler, monitor):
        self.registry = registry
        self.data_assembler = data_assembler
        self.monitor = monitor

    async def run(self, automation_path: str, params: dict) -> AutomationResult:
        config = yaml.safe_load(open(automation_path))

        # Step 1: Assemble data from source systems
        timeline = await self.data_assembler.assemble(
            config["data_query"], params
        )

        # Step 2: Fill prompt template
        prompt = config["prompt_template"].replace(
            "{{assembled_timeline}}", timeline
        )
        for key, value in params.items():
            prompt = prompt.replace(f"{{{{{key}}}}}", str(value))

        # Step 3: Route to the configured model
        provider = self.registry.get(config["model"]["provider"])
        response = await provider.generate(prompt, config["model"])

        # Step 4: Monitor output quality
        grounding_score = None
        if config.get("evaluation", {}).get("check_grounding"):
            grounding_score = await self.monitor.check_grounding(
                response["text"], timeline
            )

        result = AutomationResult(
            output=response["text"],
            model_used=config["model"]["model_id"],
            tokens_in=response["usage"]["input"],
            tokens_out=response["usage"]["output"],
            latency_ms=response["latency_ms"],
            grounding_score=grounding_score,
        )

        # Step 5: Log for continuous monitoring
        await self.monitor.log_run(config["name"], result, params)
        return result
```

**Example 2: Hallucination monitoring pipeline**

User: "We need to monitor our LLM outputs for hallucinations and inaccuracies in production. Build a monitoring system."

Approach:
1. Define hallucination (fabricated info) vs. inaccuracy (distorted real info)
2. Build claim extraction and verification pipeline
3. Create alerting and dashboarding

Output:
```python
# monitoring/hallucination_detector.py
from dataclasses import dataclass

@dataclass
class VerificationResult:
    claim: str
    status: str  # "grounded" | "hallucination" | "inaccuracy" | "unverifiable"
    source_evidence: str | None
    confidence: float

@dataclass
class GenerationAudit:
    generation_id: str
    total_claims: int
    hallucinations: int
    inaccuracies: int
    grounded: int
    unverifiable: int
    hallucination_rate: float
    inaccuracy_rate: float

class HallucinationMonitor:
    def __init__(self, claim_extractor, source_searcher, verifier):
        self.claim_extractor = claim_extractor
        self.source_searcher = source_searcher
        self.verifier = verifier

    async def audit_generation(
        self, generation_id: str, llm_output: str, source_data: str
    ) -> GenerationAudit:
        # Extract factual claims from the LLM output
        claims = await self.claim_extractor.extract(llm_output)

        results = []
        for claim in claims:
            # Search source data for evidence supporting/contradicting the claim
            evidence = await self.source_searcher.find_evidence(
                claim, source_data
            )

            if not evidence:
                # Claim not found in source data at all = hallucination
                results.append(VerificationResult(
                    claim=claim, status="hallucination",
                    source_evidence=None, confidence=0.85
                ))
            else:
                # Evidence found -- check if claim accurately represents it
                verification = await self.verifier.verify(claim, evidence)
                results.append(verification)

        hallucinations = sum(1 for r in results if r.status == "hallucination")
        inaccuracies = sum(1 for r in results if r.status == "inaccuracy")

        return GenerationAudit(
            generation_id=generation_id,
            total_claims=len(results),
            hallucinations=hallucinations,
            inaccuracies=inaccuracies,
            grounded=sum(1 for r in results if r.status == "grounded"),
            unverifiable=sum(1 for r in results if r.status == "unverifiable"),
            hallucination_rate=hallucinations / max(len(results), 1),
            inaccuracy_rate=inaccuracies / max(len(results), 1),
        )
```

**Example 3: Value assessment framework for prioritizing new automations**

User: "We have 12 potential LLM automations to build. Help us prioritize using a value assessment framework."

Approach:
1. Define the three-axis value model (cost savings, time savings, revenue)
2. Build a scoring and ranking system
3. Generate a prioritized roadmap

Output:
```python
# value/assessment.py
from dataclasses import dataclass

@dataclass
class ValueEstimate:
    automation_name: str
    # Cost savings: labor hours eliminated * hourly cost
    hours_saved_per_run: float
    runs_per_month: int
    hourly_labor_cost: float
    # Time savings: reduction in turnaround time
    current_turnaround_hours: float
    projected_turnaround_hours: float
    time_sensitive_volume: int  # tasks where speed matters
    # Revenue impact
    revenue_per_faster_unit: float  # revenue gained from faster processing
    error_reduction_pct: float  # fewer errors = less rework/refunds
    current_error_cost_monthly: float
    # Build cost
    estimated_build_effort_days: int
    monthly_llm_api_cost: float

    @property
    def annual_cost_savings(self) -> float:
        return self.hours_saved_per_run * self.runs_per_month * 12 * self.hourly_labor_cost

    @property
    def annual_time_value(self) -> float:
        hours_freed = (
            (self.current_turnaround_hours - self.projected_turnaround_hours)
            * self.time_sensitive_volume * 12
        )
        return hours_freed * self.hourly_labor_cost * 0.5  # conservative multiplier

    @property
    def annual_revenue_impact(self) -> float:
        speed_revenue = self.revenue_per_faster_unit * self.time_sensitive_volume * 12
        error_savings = self.current_error_cost_monthly * self.error_reduction_pct * 12
        return speed_revenue + error_savings

    @property
    def total_annual_value(self) -> float:
        return self.annual_cost_savings + self.annual_time_value + self.annual_revenue_impact

    @property
    def annual_cost(self) -> float:
        return self.monthly_llm_api_cost * 12

    @property
    def net_annual_value(self) -> float:
        return self.total_annual_value - self.annual_cost

    @property
    def roi_ratio(self) -> float:
        build_cost = self.estimated_build_effort_days * 8 * self.hourly_labor_cost
        total_cost = build_cost + self.annual_cost
        return self.total_annual_value / max(total_cost, 1)

def prioritize(candidates: list[ValueEstimate]) -> list[dict]:
    ranked = sorted(candidates, key=lambda c: c.roi_ratio, reverse=True)
    return [
        {
            "rank": i + 1,
            "name": c.automation_name,
            "annual_value": f"${c.net_annual_value:,.0f}",
            "roi": f"{c.roi_ratio:.1f}x",
            "cost_savings": f"${c.annual_cost_savings:,.0f}",
            "time_savings": f"${c.annual_time_value:,.0f}",
            "revenue_impact": f"${c.annual_revenue_impact:,.0f}",
            "build_days": c.estimated_build_effort_days,
        }
        for i, c in enumerate(ranked)
    ]
```

## Best Practices

- **Do:** Define automations as declarative configs (YAML/JSON) separate from execution logic. This makes them versionable, reviewable, and auditable without touching code.
- **Do:** Route different tasks to different models. Summarization, extraction, classification, and generation have different performance/cost profiles. A single-model approach leaves value on the table.
- **Do:** Track hallucination and inaccuracy rates as separate metrics. A hallucination (fabricated fact) has different risk than an inaccuracy (distorted real fact), and they require different mitigation strategies.
- **Do:** Require user training before granting interactive mode access. Free-form prompting over sensitive data needs users who understand what the system can and cannot do.
- **Avoid:** Relying solely on pre-deployment benchmarks for production monitoring. Model behavior shifts with real-world data distributions, and benchmark performance does not predict production quality.
- **Avoid:** Building the platform around a single LLM vendor. Vendor lock-in removes your ability to optimize cost, quality, and latency per task. The provider abstraction layer is not optional -- it is architectural.
- **Avoid:** Quantifying value only as cost savings. Time savings and revenue impact often dwarf direct cost reduction. The three-axis framework prevents undervaluing high-impact automations that don't directly eliminate headcount.

## Error Handling

- **Data assembly failures:** When a source system is unreachable or returns incomplete data, fail the automation explicitly rather than running the LLM on partial context. Log which sources failed and alert operators. For interactive mode, display which data segments are missing so users can judge whether to proceed.
- **Context window overflow:** When the assembled timeline exceeds the model's context limit, apply the configured overflow strategy (summarize oldest, drop lowest-priority sources, or reject with a message). Never silently truncate -- the user or automation must know what data was excluded.
- **Model provider outages:** The model-agnostic layer should support fallback configurations: if the primary model is unavailable, route to a designated backup model. Log the fallback for monitoring since quality metrics may differ between models.
- **High hallucination rates:** Set alerting thresholds on hallucination rate per automation. If an automation exceeds its threshold (e.g., > 1.0 hallucinations/generation), auto-pause it and notify the team. Do not let degraded automations continue silently.
- **Value estimate drift:** Re-evaluate value assessments quarterly. Usage volumes, labor costs, and model pricing change. An automation that was high-value at launch may no longer justify its API costs.

## Limitations

- This pattern requires **direct programmatic access** to source data systems. If your organization's data is locked behind manual-only interfaces or restrictive APIs, the data assembly layer becomes the bottleneck.
- The hallucination monitoring pipeline itself uses LLMs for claim extraction and verification, introducing a **recursive trust problem**. High-stakes domains need human-in-the-loop verification alongside automated monitoring.
- The value assessment framework produces **estimates, not measurements**. Actual savings depend on user adoption, workflow changes, and organizational factors that are difficult to predict. Treat initial estimates as hypotheses to validate after deployment.
- Model-agnostic design adds **operational complexity**. Each model has different prompt formats, token limits, pricing, and failure modes. The abstraction layer must be actively maintained as providers change their APIs.
- This approach is designed for **institutional deployment** with governance infrastructure. Small teams or individual developers are better served by direct API integration without the automation/monitoring overhead.

## Reference

- **Paper:** [Adoption and Use of LLMs at an Academic Medical Center](https://arxiv.org/abs/2602.00074v1) (Shah et al., 2026)
- **Key insight:** The separation of automations (fixed prompt + data + model configs) from interactive use, combined with model-agnostic routing and continuous hallucination monitoring, transforms LLM use from a standalone tool into a governed institutional capability. Look for: the automation architecture pattern, the evaluation methodology that replaced benchmarks, and the three-axis value framework.