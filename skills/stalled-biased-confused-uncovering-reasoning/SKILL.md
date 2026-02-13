---
name: "stalled-biased-confused-uncovering-reasoning"
description: "Systematic root cause analysis for cloud/distributed system failures using a 16-category reasoning failure taxonomy and multi-hop fault propagation tracing. Use when: 'diagnose this production incident', 'find the root cause of this outage', 'trace this failure through our microservices', 'why is this service failing', 'analyze these alerts and logs to find what broke', 'RCA for this distributed system issue'."
---

# Systematic Root Cause Analysis with Reasoning Failure Awareness

This skill equips Claude to perform rigorous root cause analysis (RCA) on cloud and distributed system failures by applying the structured diagnostic methodology and 16-category reasoning failure taxonomy from Riddell et al. (2026). Instead of naive pattern-matching on symptoms, Claude traces multi-hop fault propagation through service dependency graphs, systematically evaluates evidence across data modalities (metrics, logs, traces), and actively avoids the 16 catalogued reasoning failures (anchoring bias, stalling, confused provenance, etc.) that empirically predict incorrect diagnoses.

## When to Use

- When the user reports a production incident and needs to identify the root cause across multiple interacting services
- When alert fatigue makes it unclear which of many firing alerts points to the actual origin of failure
- When a failure has propagated through a dependency chain and the symptom appears far from the true cause (multi-hop fault propagation)
- When the user provides mixed telemetry (metrics, logs, traces) and needs structured triage
- When debugging microservice architectures where a downstream service failure masks an upstream root cause
- When the user wants to audit or improve an existing RCA process or post-mortem analysis
- When building or evaluating automated RCA pipelines and needs a reasoning quality framework

## Key Technique

**Multi-hop fault propagation tracing with reasoning failure guards.** Traditional RCA approaches fail on distributed systems because symptoms manifest far from root causes. A database connection pool exhaustion in Service A may surface as HTTP 503 errors in Service D, three hops away. This paper demonstrates that even capable LLMs systematically fail at multi-hop RCA due to 16 specific, predictable reasoning failures -- not due to lack of knowledge, but due to flawed reasoning processes.

The core insight is that **four reasoning failures are the strongest negative predictors of correct diagnosis**: anchoring bias (RF-13: fixating on the first anomaly seen), repetition/stalling (RF-12: cycling without progress), arbitrary evidence selection (RF-07: inconsistent triage heuristics), and failure to update beliefs (RF-09: not revising hypotheses when evidence contradicts them). By explicitly checking for these during analysis, diagnostic accuracy improves substantially.

**Data modality matters more than model choice.** The paper found that metrics are the most critical modality for fault localization (removing them degrades accuracy by 7-15%), logs are essential for fault type classification, and traces are frequently noisy -- excluding trace data often *improves* accuracy (up to +28% on path reconstruction). This counter-intuitive finding means effective RCA should weight evidence modalities deliberately rather than treating all telemetry equally.

## Step-by-Step Workflow

1. **Map the system topology.** Before examining any alerts, build a mental model of the service dependency graph. Identify all services, their instances, and directional dependencies (A calls B, B reads from C). This graph is the substrate for fault propagation tracing.

2. **Triage alerts by modality.** Separate incoming evidence into three channels: metrics anomalies (latency spikes, error rate increases, resource saturation), log signals (ERROR-level entries, unusual low-frequency templates), and trace anomalies (abnormal response times, failed spans). Prioritize metrics for localization, logs for type classification.

3. **Be skeptical of trace data.** Trace alerts are often voluminous and noisy. Do not let trace volume dominate analysis. If trace alerts point in a different direction than metrics and logs, weight traces lower. Explicitly note when trace data is being deprioritized and why.

4. **Generate multiple root cause hypotheses.** Formulate at least 3 candidate root causes before committing to investigation. For each hypothesis, state: (a) the suspected faulty entity, (b) the suspected fault type, and (c) the predicted propagation path through the dependency graph. Guard against RF-13 (anchoring bias) by deliberately considering alternatives to the first anomaly encountered.

5. **Trace propagation paths through the dependency graph.** For each hypothesis, walk the dependency graph from the suspected root cause forward, checking whether each hop has corresponding anomalous evidence. A valid propagation path must follow actual service dependencies -- do not skip hops or invent connections. Validate that the path is a valid walk in the known topology.

6. **Apply evidence sufficiency checks at each hop.** At every node in the propagation path, ask: "Is there independent evidence (metric, log, or trace) that this entity was anomalous during the incident window?" If not, the path is speculative. Guard against RF-08 (evidential insufficiency) by requiring at least one corroborating signal per hop.

7. **Test hypotheses against each other, not in isolation.** Compare the evidence supporting each hypothesis side-by-side. Which hypothesis explains the most observed anomalies? Which has the fewest unsupported hops? Guard against RF-09 (failure to update beliefs) by explicitly abandoning hypotheses that are contradicted by evidence.

8. **Check for confused provenance (RF-03).** Verify that you are not attributing the root cause to the component that *observed* or *reported* the failure rather than the component that *caused* it. The monitoring service that fires an alert is not the root cause -- the upstream dependency that triggered the anomaly is.

9. **Classify the fault type using log evidence.** Once the root cause location is identified, use log content to classify the fault type (e.g., network partition, resource exhaustion, configuration error, deployment regression, dependency timeout). Metrics tell you *where*; logs tell you *what*.

10. **Produce a structured diagnosis.** Output: (a) root cause entity and instance, (b) fault type classification, (c) propagation path with evidence at each hop, (d) confidence level, and (e) explicitly list which reasoning failure guards were applied and what alternative hypotheses were considered and rejected.

## Concrete Examples

**Example 1: Multi-hop microservice failure**

User: "Our checkout service is returning 503 errors. Here are the alerts: checkout-service latency p99 jumped to 12s, payment-service error rate at 45%, database connection pool at 100% on postgres-primary, checkout-service logs show 'upstream timeout', payment-service logs show 'connection refused to postgres-primary'."

Approach:
1. Map topology: checkout-service -> payment-service -> postgres-primary
2. Triage by modality:
   - Metrics: checkout latency spike, payment error rate, DB connection pool saturated
   - Logs: checkout "upstream timeout" (symptom), payment "connection refused" (closer to cause)
3. Generate hypotheses:
   - H1: postgres-primary connection pool exhaustion (root) -> payment-service connection failures -> checkout-service timeouts
   - H2: payment-service bug causing connection leak -> postgres-primary pool exhaustion -> checkout-service timeouts
   - H3: checkout-service itself is the root cause (traffic spike)
4. Trace paths:
   - H1: postgres-primary (pool at 100%) -> payment-service (connection refused in logs, 45% errors) -> checkout-service (upstream timeout, latency spike). All hops have evidence. Valid path.
   - H2: payment-service (error rate) -> postgres-primary (pool full). But no log evidence of a leak in payment-service. Weaker.
   - H3: checkout-service shows no originating error, only upstream timeout. Contradicted. Reject (RF-03 guard: don't blame the reporter).
5. Apply RF-09: H1 has the strongest evidence chain. H2 is possible but unsupported. H3 is contradicted.

Output:
```
Root Cause: postgres-primary — connection pool exhaustion
Fault Type: resource exhaustion (connection pool saturation)
Propagation Path: postgres-primary -> payment-service -> checkout-service
Evidence Chain:
  - postgres-primary: connection pool metric at 100% capacity
  - payment-service: logs show "connection refused to postgres-primary", error rate 45%
  - checkout-service: logs show "upstream timeout", p99 latency 12s
Confidence: High — all hops have corroborating evidence from independent modalities
Reasoning Guards Applied: RF-03 (rejected checkout as root cause — it's the reporter),
  RF-13 (considered 3 hypotheses before selecting), RF-09 (abandoned H3 on contradicting evidence)
```

**Example 2: Misleading trace data**

User: "Service mesh is showing tons of trace errors between frontend and API gateway, but our SLOs are actually being violated on the recommendation-service. Metrics show recommendation-service CPU at 98%, API gateway looks healthy in metrics. Traces show 200+ failed spans on frontend->API gateway path."

Approach:
1. Map topology: frontend -> API gateway -> recommendation-service (+ other backends)
2. Triage by modality — note the conflict:
   - Traces: 200+ failed spans on frontend->API gateway (high volume, noisy)
   - Metrics: recommendation-service CPU at 98% (clear resource signal), API gateway healthy
   - SLO: recommendation-service is the SLO violator
3. Apply trace skepticism: trace volume on frontend->API gateway is high but metrics show API gateway is healthy. This is likely noise or retries, not the root cause.
4. Hypotheses:
   - H1: recommendation-service CPU saturation is root cause, trace noise on frontend->gateway is a red herring
   - H2: API gateway issue causing both trace errors and downstream pressure
5. H2 is contradicted by healthy API gateway metrics. Reject per RF-09.

Output:
```
Root Cause: recommendation-service — CPU saturation
Fault Type: resource exhaustion (compute)
Propagation Path: recommendation-service (CPU 98%) -> SLO violations
Note: 200+ trace errors on frontend->API gateway path are noise/retries, not causal.
  API gateway metrics are healthy, contradicting trace signal.
Confidence: High
Reasoning Guards Applied: Trace skepticism (deprioritized voluminous trace data
  contradicted by metrics), RF-07 (did not let trace volume dominate triage)
```

**Example 3: Auditing an existing post-mortem**

User: "Review our post-mortem. It says the root cause was the frontend service because that's where customers saw errors. The frontend calls auth-service which calls user-db. Auth-service logs showed 'connection timeout to user-db' and user-db had disk I/O at 100%."

Approach:
1. Check for RF-03 (confused provenance): the post-mortem blames frontend, which is the component that *reported* the error to users, not the component that *caused* it.
2. Trace the dependency chain: frontend -> auth-service -> user-db
3. Evidence chain: user-db (disk I/O 100%) -> auth-service (connection timeout) -> frontend (user-visible errors)
4. The root cause is user-db disk I/O saturation, not frontend.

Output:
```
Post-mortem issue identified: RF-03 (Confused Provenance)
The post-mortem attributes root cause to frontend because that's where users
experienced errors. However, frontend is the symptom reporter, not the cause.

Corrected diagnosis:
  Root Cause: user-db — disk I/O saturation (100%)
  Propagation: user-db -> auth-service (connection timeout in logs) -> frontend (user-visible errors)

Recommendation: Revise post-mortem to identify user-db as root cause.
  Investigate what caused disk I/O saturation (query regression, missing index, storage capacity).
```

## Best Practices

- **Do:** Always map the service dependency graph before analyzing alerts. You cannot trace fault propagation without knowing the topology.
- **Do:** Generate at least 3 hypotheses before investigating any single one. This is the primary defense against anchoring bias (RF-13), the strongest predictor of incorrect RCA.
- **Do:** Weight metrics highest for localization and logs highest for fault type classification. This modality prioritization is empirically validated.
- **Do:** Explicitly state which hypotheses were considered and rejected, and why. This makes the reasoning auditable and guards against RF-09 (failure to update beliefs).
- **Avoid:** Letting trace data volume dominate your analysis. High trace error counts are often noise from retries and cascading timeouts, not root cause indicators.
- **Avoid:** Blaming the component closest to the user (RF-03). The service that reports the error is almost never the root cause in multi-hop propagation.
- **Avoid:** Circular reasoning or restating the same evidence in different words (RF-11, RF-12). If you catch yourself repeating analysis without new evidence, stop and seek a different data source or hypothesis.
- **Avoid:** Treating a single anomalous metric as sufficient evidence. Require corroboration across modalities or hops (RF-08 guard).

## Error Handling

- **Incomplete topology information:** If the service dependency graph is unknown or partial, state this explicitly and note that propagation path confidence is reduced. Ask the user for architecture diagrams or service mesh configuration.
- **Missing modality:** If only one data type is available (e.g., only logs, no metrics), acknowledge the limitation. Metrics-only analysis can localize but not classify. Logs-only analysis can classify but may mislocalize.
- **Contradictory evidence across modalities:** Do not ignore contradictions. Flag them explicitly. Contradictions often indicate either noisy data (especially traces) or a more complex multi-root-cause scenario.
- **Too many simultaneous anomalies:** If more than ~10 entities are anomalous, the incident may have multiple independent root causes or you may be looking at cascading symptoms. Focus on entities with the earliest anomaly timestamps and work forward.
- **Agent stalling detected:** If you find yourself repeating the same analysis step or cycling between the same two hypotheses, this is RF-12 (repetition/stalling). Force a pivot: examine a previously unconsidered entity, request a different data modality, or escalate to the user.

## Limitations

- This approach assumes a single root cause per incident. Multi-root-cause incidents (e.g., two independent failures coinciding) require parallel investigation tracks and are not well-handled by linear propagation tracing.
- The taxonomy was validated on microservice architectures (OnlineBoutique, MicroSS). Failure patterns may differ for monolithic systems, serverless architectures, or edge computing.
- Effective RCA requires the user to provide or describe the service dependency topology. Without topology, multi-hop tracing degrades to per-entity anomaly ranking.
- The paper found that even the best models achieved only 36-45% top-1 accuracy on localization in simpler scenarios, dropping to ~10% on complex ones. This skill improves reasoning discipline but does not guarantee correct diagnosis -- always validate conclusions with system owners.
- Temporal precision matters. If alert timestamps are coarse or unsynchronized across services, propagation ordering may be unreliable.

## Reference

Riddell, E., Riddell, J., Sun, G., Antkiewicz, M., & Czarnecki, K. (2026). *Stalled, Biased, and Confused: Uncovering Reasoning Failures in LLMs for Cloud-Based Root Cause Analysis.* arXiv:2601.22208v1. FORGE 2026.

Key takeaway: Table 3 contains the complete 16-failure taxonomy (RF-01 through RF-16) with definitions and examples. Tables 5-7 show which failures predict incorrect diagnosis. Section 5.3 quantifies modality impact.