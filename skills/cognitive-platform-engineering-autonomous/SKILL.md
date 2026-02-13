---
name: "cognitive-platform-engineering-autonomous"
description: "Build autonomous cloud operations using a four-plane cognitive architecture (Sensing, Reasoning, Orchestration, Experience) with Kubernetes, Terraform, OPA, and ML anomaly detection. Use when: 'set up self-healing Kubernetes infrastructure', 'add anomaly detection to my cloud platform', 'create OPA policies for autonomous remediation', 'build a cognitive operations pipeline', 'implement intent-based infrastructure management', 'add intelligent auto-scaling with feedback loops'."
---

# Cognitive Platform Engineering for Autonomous Cloud Operations

This skill enables Claude to design and implement autonomous cloud operations systems using the four-plane Cognitive Platform Engineering architecture from Punniyamoorthy et al. (2026). Instead of reactive, rule-based DevOps automation, this approach embeds sensing, reasoning, and autonomous action directly into the platform lifecycle through a continuous feedback loop. The result is infrastructure that detects anomalies via ML, evaluates responses through OPA policies, executes remediation via Kubernetes operators and Terraform, and surfaces decisions to humans through an experience layer -- reducing mean time to resolution, improving resource efficiency, and maintaining compliance without manual intervention.

## When to Use

- When the user wants to add **self-healing capabilities** to Kubernetes clusters (auto-remediation of pod failures, node pressure, resource exhaustion)
- When the user asks to build an **anomaly detection pipeline** for cloud infrastructure metrics (CPU spikes, memory leaks, latency degradation, error rate increases)
- When the user needs **OPA policies** that trigger automated infrastructure changes based on reasoning outputs rather than static thresholds
- When the user wants to implement **intent-based infrastructure** where they declare desired state (e.g., "99.9% availability, <200ms p95 latency") and the system self-adjusts
- When the user asks to create a **feedback loop** between monitoring, analysis, and remediation in their DevOps pipeline
- When the user wants to add **intelligent auto-scaling** that goes beyond CPU/memory thresholds by incorporating anomaly detection and predictive models
- When the user needs to build a **compliance-as-code** layer that continuously validates and corrects infrastructure drift

## Key Technique: Four-Plane Cognitive Architecture

The core insight is structuring autonomous operations into four cooperating planes rather than a flat automation pipeline. Traditional DevOps tools (Prometheus alerts -> PagerDuty -> human -> kubectl) create a serial chain with humans as the bottleneck. The cognitive architecture replaces this with parallel, feedback-driven planes:

**Sensing Plane** collects telemetry (metrics via Prometheus, logs via Fluentd/Loki, traces via Jaeger/OpenTelemetry) and normalizes it into a unified data model. The key innovation is not just collection but *correlation* -- linking a latency spike in service A to a memory pressure event on node B. **Reasoning Plane** applies ML models (isolation forests for anomaly detection, statistical deviation models for trend analysis) to the correlated telemetry, producing structured assessments: anomaly type, confidence score, probable root cause, and recommended action. This replaces static threshold alerts with contextual understanding. **Orchestration Plane** evaluates recommendations against OPA policies (business rules, compliance constraints, blast radius limits) and executes approved actions via Kubernetes operators or Terraform applies. Crucially, policies gate what the system *may* do autonomously vs. what requires human approval. **Experience Plane** presents decisions, rationale, and outcomes to operators, and feeds human corrections back into the Reasoning Plane for continuous improvement.

The continuous feedback loop is what distinguishes this from one-shot automation: every autonomous action's outcome is measured, and the system learns whether the action actually improved the situation or made it worse. This closes the loop between action and observation, enabling reinforcement-learning-style improvement over time.

## Step-by-Step Workflow

1. **Audit the existing infrastructure stack** -- Identify what telemetry sources exist (Prometheus, CloudWatch, Datadog), what IaC tool manages resources (Terraform, Pulumi, Helm), and what policy enforcement is in place. Map these to the four planes to find gaps.

2. **Implement the Sensing Plane** -- Deploy or configure Prometheus with ServiceMonitor CRDs for metrics, set up structured logging with correlation IDs, and configure OpenTelemetry collectors. Write a telemetry normalizer that emits a unified event schema:
   ```yaml
   # Unified telemetry event schema
   apiVersion: cognitive.platform/v1
   kind: TelemetryEvent
   metadata:
     source: prometheus
     correlationId: "abc-123"
   spec:
     metric: container_memory_usage_bytes
     value: 1.8Gi
     threshold: 2Gi
     node: worker-3
     namespace: production
     timestamp: "2026-01-24T10:30:00Z"
   ```

3. **Build the Reasoning Plane** -- Implement an anomaly detection service that consumes normalized telemetry. Use isolation forests for multivariate anomaly detection and EWMA (Exponentially Weighted Moving Average) for trend detection. Output structured assessments:
   ```json
   {
     "anomalyId": "anom-456",
     "type": "memory_pressure",
     "confidence": 0.87,
     "affectedResources": ["pod/api-server-7b9c", "node/worker-3"],
     "probableCause": "memory_leak_in_container",
     "recommendedAction": "restart_pod",
     "severity": "warning"
   }
   ```

4. **Define OPA policies for the Orchestration Plane** -- Write Rego policies that gate autonomous actions based on confidence thresholds, blast radius, time-of-day, and compliance requirements:
   ```rego
   package cognitive.orchestration

   default allow_autonomous_action = false

   allow_autonomous_action {
     input.assessment.confidence > 0.80
     input.assessment.severity != "critical"
     blast_radius_acceptable
     within_change_window
   }

   blast_radius_acceptable {
     count(input.assessment.affectedResources) <= 3
   }

   within_change_window {
     hour := time.clock(time.now_ns())[0]
     hour >= 6
     hour <= 22
   }
   ```

5. **Build Kubernetes operators for autonomous remediation** -- Create a custom controller that watches for approved assessments and executes remediation actions (pod restarts, HPA adjustments, node cordoning, rollbacks). Use controller-runtime or Kopf (Python):
   ```python
   # Kopf-based remediation operator (simplified)
   import kopf
   import kubernetes

   @kopf.on.create('cognitive.platform', 'v1', 'remediationorders')
   def handle_remediation(spec, **kwargs):
       action = spec.get('action')
       target = spec.get('target')
       if action == 'restart_pod':
           api = kubernetes.client.CoreV1Api()
           api.delete_namespaced_pod(target['name'], target['namespace'])
       elif action == 'scale_hpa':
           # Adjust HPA min/max replicas
           ...
   ```

6. **Wire Terraform for infrastructure-level remediation** -- For actions that go beyond Kubernetes (scaling node groups, adjusting cloud resources), generate Terraform variable overrides and trigger applies through a CI pipeline or Atlantis:
   ```hcl
   # Auto-generated by Orchestration Plane
   variable "node_group_desired_size" {
     default = 5  # Adjusted from 3 based on assessment anom-789
   }
   ```

7. **Implement the feedback loop** -- After every autonomous action, measure the outcome against the original anomaly signal. Store action-outcome pairs for the Reasoning Plane to learn from. Create a CRD that tracks remediation effectiveness:
   ```yaml
   apiVersion: cognitive.platform/v1
   kind: RemediationOutcome
   spec:
     remediationId: "rem-101"
     anomalyId: "anom-456"
     actionTaken: "restart_pod"
     outcomeMetrics:
       anomalyResolved: true
       timeToResolution: "45s"
       sideEffects: []
     feedback: "effective"
   ```

8. **Build the Experience Plane dashboard** -- Create a lightweight UI or CLI tool that shows: active anomalies, pending and executed remediations, policy decisions (allowed/denied with reasons), and feedback loop metrics. Expose this via a Kubernetes service or integrate into Grafana.

9. **Add escalation paths** -- For actions blocked by OPA policies (low confidence, critical severity, outside change window), generate structured alerts with full context (anomaly assessment, recommended action, policy denial reason) so humans can make informed decisions quickly.

10. **Validate end-to-end with chaos engineering** -- Use Litmus or Chaos Mesh to inject failures (pod kills, CPU stress, network partitions) and verify that the four-plane loop detects, reasons about, remediates, and learns from each scenario.

## Concrete Examples

**Example 1: Self-healing memory leak detection**

User: "Set up autonomous detection and remediation for memory leaks in our production Kubernetes cluster."

Approach:
1. Deploy Prometheus with `container_memory_usage_bytes` and `container_memory_working_set_bytes` scraping at 15s intervals
2. Create an anomaly detection job that fits an isolation forest on per-pod memory time series, flagging pods whose growth rate deviates >2 standard deviations from peers in the same deployment
3. Write an OPA policy: allow autonomous pod restart if confidence > 0.85, pod is not a singleton, and deployment has readiness probes configured
4. Deploy a Kopf operator that watches `RemediationOrder` CRDs and executes rolling restart of the affected deployment
5. Record outcome: did memory usage stabilize within 5 minutes post-restart?

Output structure:
```
sensing/
  prometheus-servicemonitor.yaml    # Metrics collection for memory
  telemetry-normalizer-deployment.yaml
reasoning/
  anomaly-detector/
    model.py                        # Isolation forest on memory time series
    assessment-publisher.py         # Emits TelemetryAssessment CRDs
orchestration/
  opa-policies/
    memory-remediation.rego         # Confidence, blast radius, readiness gates
  remediation-operator/
    handler.py                      # Kopf operator for pod restarts
  crds/
    remediationorder-crd.yaml
    remediationoutcome-crd.yaml
experience/
  grafana-dashboard.json            # Anomaly + remediation visibility
```

**Example 2: Intent-based auto-scaling with OPA guardrails**

User: "I want to declare that my API should maintain p95 latency under 200ms and the system should auto-adjust replicas and resources."

Approach:
1. Define an intent CRD:
   ```yaml
   apiVersion: cognitive.platform/v1
   kind: OperationalIntent
   metadata:
     name: api-latency-intent
   spec:
     target: deployment/api-server
     objectives:
       - metric: http_request_duration_seconds_p95
         threshold: 0.2
         operator: "<="
     constraints:
       maxReplicas: 20
       maxCpuPerPod: "2000m"
       maxMonthlyCost: 5000
   ```
2. Sensing Plane monitors `http_request_duration_seconds` histogram and computes p95
3. Reasoning Plane compares current p95 against intent, predicts whether scaling replicas or increasing CPU limits is more likely to reduce latency (based on historical action-outcome data)
4. OPA policy enforces constraints (max replicas, max CPU, cost ceiling)
5. Orchestration Plane patches the HPA or VPA accordingly
6. Feedback loop measures whether p95 improved after the change

Output: The system autonomously scales from 3 to 7 replicas when p95 crosses 180ms, then records that this action reduced p95 to 120ms, reinforcing "scale replicas" as the preferred action for latency-driven anomalies.

**Example 3: Compliance drift detection and auto-correction**

User: "Add continuous compliance checking that automatically fixes Terraform drift for our security policies."

Approach:
1. Write OPA policies encoding security requirements (e.g., no public S3 buckets, encrypted EBS volumes, VPC flow logs enabled)
2. Sensing Plane runs `terraform plan` on a schedule, capturing drift as structured events
3. Reasoning Plane classifies drift severity: cosmetic (tag changes) vs. security-critical (public access enabled)
4. Orchestration Plane auto-applies corrections for security-critical drift if confidence > 0.90, queues cosmetic drift for batch correction
5. Experience Plane generates a compliance report showing: drifts detected, auto-corrected, and pending human review

Output:
```
Compliance Report - 2026-01-24
Drifts Detected: 4
  - s3://data-bucket: PublicAccessBlock disabled [CRITICAL] -> Auto-corrected
  - ebs/vol-abc: Encryption disabled [CRITICAL] -> Auto-corrected
  - ec2/i-xyz: Tag "owner" missing [LOW] -> Queued for batch
  - rds/prod-db: Backup retention 5d vs policy 7d [MEDIUM] -> Queued for review
Auto-corrections applied: 2
Pending human review: 2
Compliance score: 94% -> 98%
```

## Best Practices

- **Do:** Start with high-confidence, low-blast-radius autonomous actions (restarting a single pod in a deployment with many replicas) and expand scope incrementally as the feedback loop validates effectiveness.
- **Do:** Always include OPA policies as a mandatory gate between reasoning and action. Never allow the Reasoning Plane to directly execute changes -- the policy layer is the safety mechanism.
- **Do:** Store every action-outcome pair. The feedback loop is the core differentiator; without it, you have rule-based automation with extra steps.
- **Do:** Design remediation actions to be idempotent. The system may re-trigger the same action if the anomaly persists.
- **Avoid:** Allowing autonomous actions on stateful workloads (databases, message queues) without explicit human-in-the-loop policies. The blast radius is too high.
- **Avoid:** Training anomaly detection models on insufficient data. Require at least 2 weeks of baseline telemetry before enabling autonomous remediation to prevent false positives from triggering destructive actions.

## Error Handling

- **False positive anomalies:** If the feedback loop records that an autonomous action did not resolve the anomaly (or made it worse), automatically increase the confidence threshold for that anomaly type and escalate to humans. Implement a circuit breaker: after 2 consecutive ineffective actions on the same resource, halt autonomous remediation and alert.
- **OPA policy conflicts:** When multiple policies produce contradictory decisions, deny the action and log the conflict for human resolution. Never fail-open on policy evaluation errors.
- **Terraform apply failures:** If an auto-generated Terraform change fails to apply, capture the error output, attach it to the RemediationOutcome CRD with `feedback: "failed"`, and create a structured alert. Do not retry automatically -- infrastructure changes that fail may indicate a deeper problem.
- **Reasoning Plane unavailability:** If the ML service is down, fall back to static threshold-based alerting (degrade gracefully to traditional monitoring). Never skip the policy evaluation layer even in degraded mode.

## Limitations

- This approach requires a mature observability stack as a prerequisite. If the cluster lacks Prometheus, structured logging, or tracing, the Sensing Plane has nothing to work with -- deploy observability first.
- ML-based anomaly detection needs stable baseline data. During major deployments, migrations, or architecture changes, the models will produce unreliable assessments. Implement a "learning mode" that observes but does not act during volatile periods.
- The four-plane architecture adds operational complexity. For small teams running a handful of services, the overhead of maintaining CRDs, operators, OPA policies, and ML models may exceed the benefit of autonomous operations.
- Reinforcement learning for action selection (choosing between restart, scale, rollback) is promising but not production-proven at scale. Start with deterministic decision trees for action selection and introduce RL experimentally.
- Multi-cloud or hybrid environments multiply the Sensing and Orchestration Plane complexity significantly. The prototype in the paper targets a single Kubernetes cluster with Terraform-managed cloud resources.

## Reference

Punniyamoorthy, V., Saksena, N., Sankiti, S.R., Chockalingam, N., & Kirubakaran, A.M. (2026). *Cognitive Platform Engineering for Autonomous Cloud Operations.* arXiv:2601.17542v1. Key sections: the four-plane reference architecture diagram (Section 3), OPA policy integration patterns (Section 4), and MTTR/resource efficiency results from the Kubernetes prototype (Section 5).