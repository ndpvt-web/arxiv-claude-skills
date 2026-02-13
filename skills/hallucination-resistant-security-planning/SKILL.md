---
name: "hallucination-resistant-security-planning"
description: >
  Generate reliable incident response and security recovery plans using a
  generate-check-refine loop with consistency-based abstention. Reduces
  hallucinated security actions by validating LLM outputs against system
  constraints and lookahead predictions before committing.
  Trigger phrases: "plan incident response", "generate recovery plan",
  "security response plan", "analyze security logs", "incident recovery",
  "hallucination-resistant planning"
---

# Hallucination-Resistant Security Planning

This skill enables Claude to produce reliable security incident response and recovery plans by applying a generate-check-refine loop derived from Hammar et al. (2026). Instead of generating a plan in one shot and hoping it is correct, Claude generates multiple candidate actions, checks their internal consistency via lookahead predictions, abstains when consistency is low, collects or simulates external feedback, and refines candidates through in-context learning. This approach controls hallucination risk to a tunable probability bound and reduces recovery action counts by up to 30% compared to single-pass generation.

## When to Use

- When the user provides security logs (Snort alerts, Wazuh alerts, SIEM output) and asks for an incident response plan
- When building automated incident response playbooks that must be reliable and auditable
- When the user asks to analyze intrusion detection system (IDS) alerts and recommend containment/recovery steps
- When generating security remediation plans where a hallucinated action (e.g., wrong firewall rule, wrong host isolated) could cause harm
- When the user wants to evaluate or rank multiple possible security responses to an incident
- When building agentic security workflows that need a principled abstention mechanism to avoid acting on uncertain outputs

## Key Technique: Generate-Check-Refine with Consistency-Based Abstention

The core insight is that LLM hallucinations in planning tasks can be detected by measuring the **internal consistency** of the model's own predictions. When asked to generate N candidate actions and predict the expected remaining recovery time after each, a hallucinating model produces widely dispersed predictions. A consistent model produces tight clusters.

The consistency score is computed as:

```
lambda = exp(-beta/N * sum((T_predicted_i - T_mean)^2))
```

where `beta > 0` controls sensitivity and `T_predicted_i` is the LLM's predicted remaining time after candidate action `i`. This yields a value between 0 (highly inconsistent, likely hallucinating) and 1 (highly consistent, likely reliable). When `lambda <= gamma` (the consistency threshold), the framework **abstains** from acting and instead collects external feedback -- from a digital twin, a sandbox, a security expert, or a simulation -- then refines candidates via in-context learning.

This design provides a formal guarantee: by calibrating `gamma` on a small reference dataset of known-good and known-bad actions, the hallucination probability can be bounded to any desired level `kappa`. The ICL refinement converges with regret bounded by `O(sqrt(|A| * K * ln K))` where `K` is the number of refinement iterations and `|A|` is the action space size. In practice, convergence occurs within 8 iterations.

## Step-by-Step Workflow

1. **Parse and structure the security logs.** Extract alert signatures, source/destination IPs, ports, timestamps, and severity from the raw log input. Group alerts by attack phase (reconnaissance, exploitation, lateral movement, exfiltration) when possible.

2. **Identify the incident type and scope.** From the parsed logs, determine the attack category (ransomware, DoS, web exploit, multi-stage intrusion, etc.), affected hosts, and compromised services. State this explicitly before generating any plan.

3. **Generate N >= 3 candidate next-actions.** For each recovery phase (Contain, Assess, Preserve, Evict, Harden, Restore -- following the MITRE D3FEND taxonomy), generate at least 3 distinct candidate actions. Each action must be specific and executable (e.g., "Block outbound traffic from 10.0.1.15 on port 443 at the perimeter firewall" not "Block malicious traffic").

4. **Produce lookahead predictions for each candidate.** For every candidate action, predict the expected number of remaining recovery steps if that action is taken. Record all N predictions.

5. **Compute the consistency score.** Calculate the variance of the lookahead predictions and apply the exponential decay function. Use `beta = 1.0` as a default. Report the score.

6. **Apply the abstention policy.** If `lambda <= gamma` (default `gamma = 0.9`), flag the action set as unreliable. Do NOT commit to any action. Instead, explicitly state: "Consistency score {lambda} is below threshold {gamma}. Abstaining and requesting validation." Then describe what external feedback would resolve the uncertainty (e.g., "Run candidate action X in a sandbox and observe whether service Y remains available").

7. **Incorporate feedback and refine.** When feedback is available (user provides it, or a tool returns results), append it to context and regenerate candidate actions. Recompute consistency. Repeat until `lambda > gamma` or maximum iterations (K=8) are reached.

8. **Select the best action.** From the consistent action set, choose the candidate with the lowest predicted remaining recovery time. Commit it to the plan and advance to the next recovery phase.

9. **Assemble the full recovery plan.** Structure the final plan using the six MITRE D3FEND phases: Contain, Assess, Preserve, Evict, Harden, Restore. Each phase should list concrete, ordered actions with the host, command, or configuration change specified.

10. **Report confidence and audit trail.** For each committed action, report the consistency score at the time of selection and the number of ICL refinement rounds required. Flag any actions where feedback was needed.

## Concrete Examples

**Example 1: Ransomware incident from Snort alerts**

User: "Here are Snort alerts from our Windows network. Generate a recovery plan."
```
[1:31033:6] MALWARE Win.Trojan.Cryptodefense {TCP} 147.32.84.165:1057 -> 222.88.205.195:443
[1:31033:6] MALWARE Win.Trojan.Cryptodefense {TCP} 147.32.84.165:1058 -> 91.121.42.33:443
[1:2016978:2] ET TROJAN CryptoDefense Ransomware Checkin {TCP} 147.32.84.165:1060 -> 194.150.168.95:80
```

Approach:
1. Parse: CryptoDefense ransomware on host 147.32.84.165, C2 communication to 3 external IPs over ports 443 and 80.
2. Identify: Ransomware incident, single infected host, active C2 channel.
3. Generate 3 candidate Contain actions:
   - A1: Isolate 147.32.84.165 from network via switch port shutdown
   - A2: Block outbound to 222.88.205.195, 91.121.42.33, 194.150.168.95 at firewall
   - A3: Quarantine host via EDR agent network isolation
4. Lookahead predictions: A1 -> 5 remaining steps, A2 -> 7 remaining steps, A3 -> 5 remaining steps
5. Consistency: variance = 0.89, lambda = exp(-1.0/3 * 0.89) = 0.74
6. lambda (0.74) <= gamma (0.9): **Abstain.** The dispersion suggests uncertainty about whether partial network blocks (A2) would suffice. Request feedback: "Verify whether host 147.32.84.165 runs any critical services that would be disrupted by full isolation."
7. User feedback: "It's a developer workstation, no critical services." Regenerate: all 3 candidates now predict 5 steps. New lambda = 0.99.
8. Select A1 (isolate via switch port) -- tied with A3 but more immediate.

Output:
```
## Incident Response Plan: CryptoDefense Ransomware
Affected host: 147.32.84.165 (developer workstation)

### 1. Contain [lambda=0.99, ICL rounds=1]
- Isolate 147.32.84.165 by disabling switch port on access switch
- Block outbound to C2 IPs (222.88.205.195, 91.121.42.33, 194.150.168.95) at perimeter firewall

### 2. Assess [lambda=0.95, ICL rounds=0]
- Image RAM of 147.32.84.165 before powering down
- Enumerate encrypted files and check for shadow copy availability

### 3. Preserve [lambda=0.97, ICL rounds=0]
- Create forensic disk image of 147.32.84.165
- Export Snort alert logs for time window 2024-03-15T08:00 to present

### 4. Evict [lambda=0.92, ICL rounds=0]
- Wipe and reimage 147.32.84.165 from known-good baseline
- Revoke all credentials used on this host

### 5. Harden [lambda=0.91, ICL rounds=1]
- Deploy application whitelisting on reimaged host
- Update Snort rules to alert on CryptoDefense variant signatures

### 6. Restore [lambda=0.98, ICL rounds=0]
- Restore user files from last known-clean backup
- Reconnect host to network and monitor for 48 hours

Total actions: 10 | Feedback rounds: 2 | Max hallucination risk: 2%
```

**Example 2: Multi-stage intrusion from Wazuh alerts**

User: "Analyze these Wazuh alerts and tell me what happened and what to do."
```
rule.id:5710 srcip:203.0.113.50 - SSH brute force attempt on 10.0.2.10
rule.id:5712 srcip:203.0.113.50 - SSH authentication success on 10.0.2.10
rule.id:550 user:www-data - Unauthorized file modification /var/www/html/index.php
rule.id:510 srcip:10.0.2.10 dstip:10.0.2.20 - Lateral movement detected (SMB)
```

Approach:
1. Parse: SSH brute force from 203.0.113.50 succeeded on 10.0.2.10, followed by webshell deployment and lateral movement to 10.0.2.20.
2. Identify: Multi-stage intrusion (brute force -> access -> persistence -> lateral movement). Two hosts compromised.
3. Generate candidates per phase, compute consistency, abstain and refine where needed.

Output:
```
## Incident Analysis
Attack chain: SSH brute force (203.0.113.50) -> compromise 10.0.2.10 ->
webshell (/var/www/html/index.php) -> lateral movement to 10.0.2.20 via SMB

## Response Plan

### 1. Contain
- Block 203.0.113.50 at perimeter firewall [lambda=0.96]
- Isolate 10.0.2.10 and 10.0.2.20 to quarantine VLAN [lambda=0.93]
- Disable www-data SSH key and password authentication [lambda=0.97]

### 2. Assess
- Check 10.0.2.20 for indicators of compromise via SMB exploitation [lambda=0.88, ICL=1]
  (Abstained once: needed to verify whether SMB service on 10.0.2.20 was
   patched against EternalBlue before recommending specific remediation)
- Enumerate all files modified by www-data in last 72 hours [lambda=0.95]

### 3. Preserve
- Snapshot both VMs before remediation [lambda=0.99]
- Export Wazuh alerts and auth logs [lambda=0.98]

### 4. Evict
- Remove /var/www/html/index.php webshell [lambda=0.97]
- Audit and rotate all credentials on both hosts [lambda=0.94]
- Kill any reverse shells or persistence mechanisms [lambda=0.91]

### 5. Harden
- Enforce SSH key-only auth, disable password login [lambda=0.99]
- Implement fail2ban with 3-attempt lockout [lambda=0.96]
- Restrict SMB traffic between hosts to necessary shares only [lambda=0.92]

### 6. Restore
- Restore index.php from version control [lambda=0.99]
- Re-enable network connectivity with 72-hour enhanced monitoring [lambda=0.95]
```

**Example 3: Generating a playbook template with abstention hooks**

User: "Build me an automated incident response script that uses this consistency-checking approach."

Approach:
1. Generate a Python script skeleton with the generate-check-refine loop
2. Implement the consistency function
3. Wire abstention to a callback (Slack alert, ticket creation, or sandbox execution)

Output:
```python
import math
from typing import Callable

def consistency_score(predictions: list[float], beta: float = 1.0) -> float:
    """Compute exponential-decay consistency from lookahead predictions."""
    n = len(predictions)
    if n == 0:
        return 0.0
    mean = sum(predictions) / n
    variance = sum((p - mean) ** 2 for p in predictions) / n
    return math.exp(-beta * variance)

def generate_check_refine(
    generate_fn: Callable,       # returns (candidates, predictions)
    feedback_fn: Callable,       # collects external feedback for a candidate
    gamma: float = 0.9,
    max_icl_rounds: int = 8,
    beta: float = 1.0,
) -> dict:
    """Core loop: generate candidates, check consistency, refine if needed."""
    context = []
    for k in range(max_icl_rounds):
        candidates, predictions = generate_fn(context)
        score = consistency_score(predictions, beta)
        if score > gamma:
            best_idx = predictions.index(min(predictions))
            return {
                "action": candidates[best_idx],
                "consistency": score,
                "icl_rounds": k,
                "abstentions": k,
            }
        # Abstain: collect feedback on the would-be best action
        best_idx = predictions.index(min(predictions))
        feedback = feedback_fn(candidates[best_idx])
        context.append({
            "candidate": candidates[best_idx],
            "feedback": feedback,
        })
    # Max iterations reached -- return best available with warning
    candidates, predictions = generate_fn(context)
    best_idx = predictions.index(min(predictions))
    return {
        "action": candidates[best_idx],
        "consistency": consistency_score(predictions, beta),
        "icl_rounds": max_icl_rounds,
        "warning": "Max ICL rounds reached without meeting consistency threshold",
    }
```

## Best Practices

- **Do:** Always generate at least 3 candidate actions per decision point. Fewer candidates make the consistency score unreliable due to low sample size.
- **Do:** Report the consistency score alongside every committed action so the user has an auditable confidence trail.
- **Do:** Use the MITRE D3FEND taxonomy (Contain, Assess, Preserve, Evict, Harden, Restore) to structure response plans. This ensures completeness and aligns with industry standards.
- **Do:** Calibrate gamma on the user's environment if possible. A gamma of 0.9 is a good default, but environments with higher base variance in recovery times may need adjustment.
- **Avoid:** Committing to actions when consistency is below threshold just to produce output faster. The entire value of this framework is in principled abstention.
- **Avoid:** Treating the consistency score as ground truth about action quality. It measures self-consistency of the LLM's predictions, not objective correctness. High consistency with wrong assumptions still produces wrong plans.
- **Avoid:** Skipping the feedback loop. When abstaining, always explain what feedback would resolve the uncertainty rather than just saying "low confidence."

## Error Handling

- **All candidates have identical predictions (variance = 0, lambda = 1.0):** This can indicate the model is pattern-matching rather than reasoning. Cross-check by asking the model to justify why each candidate leads to the same outcome. If justifications are vague or identical, treat as suspicious and request external validation.
- **Consistency never exceeds threshold after max iterations:** Report the best available action with a clear warning. Recommend the user validate manually before executing. Do not silently lower the threshold.
- **Logs are ambiguous or incomplete:** State explicitly what information is missing (e.g., "Cannot determine if lateral movement occurred -- no east-west traffic logs available"). Generate plans conditional on the most likely scenario but flag assumptions.
- **Action space is too large:** When many candidate actions are possible, cluster them by category (network, host, credential, application) and run the consistency check per category to keep predictions tractable.

## Limitations

- The consistency score measures **self-consistency**, not **correctness**. A model can be consistently wrong. This framework reduces but does not eliminate hallucination risk.
- Lookahead predictions require the LLM to estimate remaining recovery steps, which is itself a prediction that can be inaccurate. The technique works best when the model has been exposed to similar incident types.
- The formal guarantees (hallucination risk bound, ICL regret bound) assume access to a calibration dataset of known-good and known-bad actions. Without calibration, the threshold gamma = 0.9 is a heuristic.
- Digital twin or sandbox feedback is assumed available. In environments without simulation capabilities, human expert feedback is the fallback, which introduces latency.
- The framework was evaluated on IDS alert logs (Snort, Wazuh). Applying it to other security domains (cloud misconfiguration, supply chain, insider threat) requires adaptation of the action taxonomy and log parsing.

## Reference

Hammar, K., Alpcan, T., & Lupu, E. (2026). *Hallucination-Resistant Security Planning with a Large Language Model.* IEEE/IFIP Network Operations and Management Symposium (NOMS 2026). arXiv: [2602.05279](https://arxiv.org/abs/2602.05279v1). Focus on Algorithm 1 (the generate-check-refine loop), Proposition 1 (hallucination risk control via threshold calibration), and Section IV (incident response experiments across CTU-Malware-2014, CIC-IDS-2017, AIT-IDS-V2-2022, and CSLE-IDS-2024 datasets).