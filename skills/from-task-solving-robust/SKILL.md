---
name: "from-task-solving-robust"
description: "Build LLM agent workflows that stay robust under partial observability, noisy signals, shifting environments, and internal state drift. Applies the four-stressor robustness framework from Pezeshkpour & Hruschka (2026) to real automation pipelines. Use when: 'make this agent more robust', 'handle unreliable API responses', 'add fallback logic to my pipeline', 'my agent breaks when the environment changes', 'add verification steps to my workflow', 'build a fault-tolerant automation'."
---

# Robust Real-World Adaptation for LLM Agent Workflows

This skill teaches Claude to design and harden LLM agent systems against the four deployment stressors identified in Pezeshkpour & Hruschka (2026): **partial observability** (incomplete state), **dynamic environments** (conditions shift mid-execution), **noisy signals** (unreliable feedback from tools/APIs), and **dynamic agent state** (the agent's own capabilities degrade or change). The core insight is that task-solving ability alone does not predict deployment robustness -- agents must explicitly budget for verification, adapt strategies when conditions shift, and infer unstated objectives from context rather than assuming a clean interface.

## When to Use

- When building a multi-step agent pipeline that calls external APIs, scrapers, or tools that may fail or return stale data
- When the user says "my agent works in testing but breaks in production" -- the gap between clean-interface and deployment-like conditions
- When designing automation that must handle partial failures gracefully (e.g., some API calls succeed while others timeout)
- When adding retry, verification, or fallback logic to an existing agent workflow
- When the user asks to "stress-test" or "harden" an agent system
- When building a pipeline where the environment changes during execution (e.g., rate limits kick in, data sources update, credentials rotate)
- When an agent must infer what "success" means from context rather than a single explicit metric

## Key Technique: Four-Stressor Robustness Framework

The paper benchmarks five state-of-the-art LLMs in a grid game called WildGrid that deliberately violates the "clean interface" assumption most agent benchmarks rely on. The game has simple rules (collect three keys, reach the exit) but introduces four stressors that mirror real deployment conditions:

1. **Partial observability**: The agent sees only a local window, not the full state. Hidden rules govern tile behavior based on context the agent hasn't observed. In real systems, this maps to incomplete API responses, missing database fields, or undocumented service behavior. Agents must experiment, form hypotheses, and act conservatively under ambiguity.

2. **Dynamic environments with distribution shift**: At fixed intervals, a latent "weather" variable changes regimes -- altering action reliability and resource costs. Hazards spread. Teleport events relocate the agent. This mirrors production systems where rate limits change, dependencies update, or infrastructure shifts. The key finding: agents that front-load information gathering (Scan/Measure early) dramatically outperform those that dive into action immediately. Robust agents transition from an exploration phase to an exploitation phase, while fragile agents use myopic trial-and-error throughout.

3. **Noisy signals and costly verification**: Observations are corrupted with per-cell noise. Actions fail stochastically (movement "slip"). Verification costs energy -- Scan expands visibility temporarily, Measure reveals hidden structure, but both drain the resource that also powers actions. This is the verification budget problem: you can't check everything, so you must decide *what* is worth verifying. The paper shows moderate noise can paradoxically improve performance by forcing agents into more cautious strategies.

4. **Dynamic agent state (capability drift)**: Mid-episode "drift events" change the agent's action reliability and cost profiles without notification. The agent must detect degradation from outcome patterns and recalibrate. In production, this maps to model degradation, token budget exhaustion, or infrastructure slowdowns that change what the agent can reliably do.

## Step-by-Step Workflow

When asked to build or harden an agent workflow, apply these steps:

1. **Audit the interface assumptions.** List every external dependency (APIs, databases, file systems, other services) and classify each as reliable, intermittent, or unknown. For each, identify what information the agent receives vs. what it assumes -- this is your partial observability surface.

2. **Map the four stressors to the specific system.** For each dependency and step in the pipeline, ask: (a) Can the agent observe the full state? (b) Can conditions change mid-execution? (c) Can signals be noisy, stale, or corrupted? (d) Can the agent's own capabilities degrade (rate limits, token budgets, credential expiry)?

3. **Design an explicit verification budget.** Not every step needs verification. Prioritize verification for: actions with high penalty on failure, actions whose preconditions depend on stale or partial information, and actions taken after a detected environment shift. Implement verification as a callable check (not just a retry) that confirms the expected postcondition before proceeding.

4. **Implement phased execution: explore-then-exploit.** Structure the agent to front-load information gathering before committing to costly actions. In practice: probe API health, validate schema assumptions, check rate limit headers, and confirm data freshness *before* launching the main workflow. Budget 15-25% of total compute/calls for this reconnaissance phase.

5. **Add change detection with replanning triggers.** Monitor key signals for distribution shift: response latencies, error rates, schema changes, unexpected null fields. When a shift is detected, pause the current plan, re-evaluate assumptions, and replan from the current state rather than retrying the failed step blindly.

6. **Implement graduated fallback, not binary retry.** Design a fallback hierarchy: (a) retry with backoff, (b) retry with degraded parameters (smaller batch, simpler query), (c) switch to alternative data source or method, (d) return partial results with explicit uncertainty markers, (e) escalate to human with a structured summary of what was tried and what failed.

7. **Add capability drift detection.** Track the agent's own success rate over a rolling window. If action reliability drops (e.g., API calls that used to succeed now fail 30% of the time), trigger recalibration: reduce concurrency, increase verification frequency, or switch to more conservative action selection.

8. **Infer implicit objectives from context.** Real tasks have unstated goals beyond the explicit request. The paper shows agents naturally trade off completion, efficiency, and penalty avoidance. Make these trade-offs explicit in the agent prompt: "Your job is NOT just to finish, but to act robustly under uncertainty. Minimize penalties. Prefer partial correct results over complete but unreliable ones."

9. **Test under stressor combinations, not just individual failures.** Single-stressor testing misses interaction effects. Test with: (a) one stressor at a time, (b) pairs of stressors, (c) all stressors simultaneously. The paper finds non-monotonic and model-specific sensitivities -- what helps under one stressor can hurt under another.

10. **Log decisions, not just outcomes.** Record why the agent chose each action, what it verified, what it assumed, and what triggered any fallback. This decision log is essential for debugging robustness failures that don't reproduce under clean conditions.

## Concrete Examples

**Example 1: Hardening a data ingestion pipeline**

User: "My agent pulls data from three APIs, transforms it, and loads it into a database. It works fine in dev but keeps failing in production."

Approach:
1. Audit each API: identify response time variability, rate limits, schema stability, and authentication token lifetimes
2. Map stressors: API-1 has intermittent 502s (noisy signals), API-2 changes schema monthly (dynamic environment), API-3 has strict rate limits that vary by time of day (dynamic agent state -- effective capability changes)
3. Add health probes before the main run: `GET /health` for each API, check rate limit headers, validate a sample response against expected schema
4. Implement verification after each transform step: row count checks, null-rate thresholds, schema conformance
5. Add graduated fallback: retry 502s with exponential backoff (max 3), fall back to cached data if API-2 schema changed, throttle API-3 calls dynamically based on remaining rate limit quota

Output structure:
```python
class RobustPipeline:
    def run(self):
        # Phase 1: Reconnaissance (explore)
        health = self.probe_all_sources()
        if not health.all_ok:
            self.replan(health.degraded_sources)

        # Phase 2: Execute with verification
        for source in self.sources:
            data = self.fetch_with_fallback(source)
            if not self.verify_postcondition(data, source.expected_schema):
                data = self.fallback_strategy(source)

        # Phase 3: Transform with drift detection
        result = self.transform(data)
        if self.drift_detected(result):
            self.log_decision("Drift detected, recalibrating")
            result = self.transform_conservative(data)

        self.load(result)
```

**Example 2: Adding robustness to a web scraping agent**

User: "Build me a scraper that collects pricing data from competitor sites. Some sites change layout frequently."

Approach:
1. Classify each target site by stressor profile: layout stability (dynamic environment), anti-bot measures (noisy signals / dynamic agent state), data completeness (partial observability)
2. Front-load reconnaissance: fetch each page, detect layout fingerprint, compare against last known structure before parsing
3. Implement selector fallback chains: primary CSS selector -> secondary XPath -> regex extraction -> mark as "unverified" with raw HTML snippet
4. Add change detection: if >20% of expected fields return null, flag the site as "shifted" and skip rather than ingest bad data
5. Track per-site success rates; if a site drops below 70% extraction rate over 5 runs, alert and pause that source

Output structure:
```python
def scrape_with_robustness(site):
    # Reconnaissance: detect current layout regime
    page = fetch(site.url)
    layout_fingerprint = detect_layout(page)

    if layout_fingerprint != site.last_known_layout:
        log_decision(f"Layout shift detected for {site.name}")
        selectors = discover_selectors(page, site.target_fields)
    else:
        selectors = site.primary_selectors

    # Extract with graduated fallback
    results = {}
    for field in site.target_fields:
        for strategy in selectors[field].fallback_chain:
            value = strategy.extract(page)
            if value is not None:
                results[field] = VerifiedValue(value, strategy.confidence)
                break
        else:
            results[field] = UnverifiedValue(null, reason="all_strategies_failed")

    # Capability drift: track extraction success rate
    site.update_success_rate(results)
    if site.success_rate < 0.7:
        alert(f"{site.name} extraction degraded, pausing")

    return results
```

**Example 3: Making a deployment automation agent robust**

User: "My CI/CD agent sometimes deploys broken builds because health checks pass initially but the service degrades after 30 seconds."

Approach:
1. This is a noisy signals + dynamic environment problem: health checks give false positives (noise), and the service state changes post-deploy (environment shift)
2. Replace single-point health check with phased verification: check at t=0, t=30s, t=60s, t=120s with increasing confidence thresholds
3. Add capability drift detection: monitor error rates and p99 latency in the first 5 minutes post-deploy
4. Implement graduated rollback: if t=30s check shows >5% error rate, hold traffic at canary percentage; if t=60s still degraded, roll back automatically

Output:
```yaml
deploy_verification:
  phase_1:  # t=0, reconnaissance
    check: health_endpoint
    threshold: 200_ok
    action_on_fail: abort_deploy
  phase_2:  # t=30s, post-stabilization
    check: [health_endpoint, error_rate, p99_latency]
    thresholds: {error_rate: "<2%", p99: "<500ms"}
    action_on_fail: hold_canary
  phase_3:  # t=60s, confirmation
    check: [error_rate, p99_latency, business_metrics]
    thresholds: {error_rate: "<1%", p99: "<300ms"}
    action_on_fail: rollback
  phase_4:  # t=120s, full promotion
    check: all_metrics_stable_for_60s
    action_on_pass: promote_to_100_percent
    action_on_fail: rollback_with_alert
```

## Best Practices

- **Do:** Front-load information gathering before committing to irreversible actions. The paper shows agents that spend 15-25% of their budget on reconnaissance consistently outperform those that act immediately.
- **Do:** Make verification a budgeted, deliberate decision -- not an afterthought. Decide what to verify based on penalty severity, not convenience.
- **Do:** Design for phased execution (explore then exploit) rather than single-pass pipelines. Structure code so the exploration phase feeds into a plan that the exploitation phase executes.
- **Do:** Include the robustness prompt framing in agent system messages: "Your job is NOT just to finish, but to act robustly under uncertainty. The world may be underspecified and change over time. Observations may be noisy. Actions have costs."
- **Avoid:** Blind retries without diagnosis. Retrying a failed action without checking whether conditions have changed is the single most common fragility pattern.
- **Avoid:** Assuming rankings are stable across conditions. A strategy that works under noise may fail under environment shift. Test under each stressor independently and in combination.
- **Avoid:** Over-verifying. The paper shows that excessive sensing drains the budget needed for actual execution. Verify high-stakes actions; trust low-stakes ones.

## Error Handling

| Failure Mode | Detection | Response |
|---|---|---|
| Partial observability gap | Unexpected null fields, missing data, undocumented behavior | Log the gap, attempt discovery probe, fall back to conservative defaults |
| Environment shift mid-execution | Schema changes, new error codes, latency spikes, changed rate limits | Pause current plan, re-probe affected dependencies, replan from current state |
| Noisy/corrupted signal | Inconsistent responses across retries, values outside expected range | Cross-validate with secondary source, apply confidence threshold before acting |
| Capability drift | Rolling success rate drop, increased latency in agent's own actions | Reduce concurrency, increase verification frequency, switch to conservative mode |
| Compounding stressors | Multiple simultaneous anomalies | Return partial results with explicit uncertainty markers, escalate to human |

## Limitations

- This framework adds overhead. For simple, well-characterized, stable environments (internal microservice with SLA guarantees, local file operations), the full four-stressor treatment is unnecessary.
- Verification budgets require calibration. The right ratio of reconnaissance to execution depends on the penalty structure of the specific domain -- there is no universal 15-25% rule.
- Change detection assumes some baseline of "normal" to compare against. For greenfield deployments with no historical data, drift detection must rely on specification-based checks rather than statistical monitoring.
- The framework addresses operational robustness, not correctness. An agent can be robust to all four stressors and still produce wrong results if the underlying logic is flawed.
- Multi-stressor interactions are hard to predict. The paper finds non-monotonic effects (moderate noise can help). Testing is the only reliable way to discover interaction effects in a specific system.

## Reference

**Paper:** Pezeshkpour, P. & Hruschka, E. (2026). "From Task Solving to Robust Real-World Adaptation in LLM Agents." arXiv:2602.02760v1. https://arxiv.org/abs/2602.02760v1

**What to look for:** Section 3 for the WildGrid environment design and four-stressor definitions; Section 5 for ablation results showing non-monotonic stressor effects and model-specific failure modes; the system prompt structure in Section 4 for the robustness-oriented agent framing.