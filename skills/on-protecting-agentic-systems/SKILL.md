---
name: "on-protecting-agentic-systems"
description: "Watermark agentic AI systems to protect intellectual property against imitation attacks using the AGENTWM framework. Embeds verifiable signals into visible action trajectories by biasing semantically equivalent tool execution paths. Triggers: 'watermark my agent', 'protect agent IP', 'detect model theft from agent outputs', 'add watermarking to tool-calling agent', 'verify if an agent was stolen', 'agentic system IP protection'"
---

# AGENTWM: Watermarking Agentic Systems for IP Protection

This skill enables Claude to design and implement watermarking schemes for tool-calling agentic systems using the AGENTWM framework. The core technique exploits **semantic equivalence of action sequences** — identifying groups of functionally identical tool invocations (e.g., `Outlook.SendEmail` vs. `Gmail.SendEmail`) and subtly biasing which equivalent action the agent uses. This bias is statistically detectable by the IP owner but invisible to end users, enabling verification that a suspect model was trained on stolen outputs without needing access to internal reasoning traces or model weights.

## When to Use

- When a user is building a tool-calling agent (function-calling LLM) and wants to protect it against imitation/distillation attacks
- When a user asks how to detect if a competitor's agent was fine-tuned on their agent's outputs
- When designing an API-calling agent where multiple vendors provide equivalent functionality and the user wants to embed IP fingerprints
- When a user needs to attribute a stolen model back to a specific leaked user or API key
- When implementing grey-box watermarking where only the action trajectory (not internal chain-of-thought) is visible
- When evaluating whether an existing watermarking scheme is robust against adaptive adversaries who may try to remove the signal

## Key Technique

**Semantic Equivalence Exploitation.** AGENTWM identifies groups of actions that produce identical outcomes — called *equivalence classes*. Five scheme types define these classes: (1) **Vendor Replacement (VR)** — same function from different providers (e.g., `Gmail.SendEmail` vs. `Outlook.SendEmail`); (2) **Parameter Granularity Replacement (PGR)** — same tool accepting different input abstractions; (3) **Interface Aliasing (IA)** — versioned or regional endpoints for the same backend; (4) **Auxiliary Equivalence (AE)** — adding benign ancillary actions like confirmation fetches; (5) **Compositional Equivalence (CE)** — atomic vs. decomposed operations (e.g., `MoveFile` vs. `CopyFile + DeleteFile`).

**Logit-Biased Injection.** For each equivalence set ℰᵢ = {w₁, w₂, ..., wₖ}, AGENTWM boosts the probability of a designated target action wₜ using exponential scaling: `p̂(wₜ) = p(wₜ)·eᵟ / [p(wₜ)·eᵟ + Σₖ≠ₜ p(wₖ)]`. The bias strength δ controls detectability vs. stealth. Each user receives an N-bit UID that activates a unique subset of equivalence passes — if bit i is 1, pass Pᵢ₊₁ is activated. This creates a per-user fingerprint embedded in the action distribution, enabling both theft detection and attacker attribution.

**Statistical Verification.** Detection uses Jensen-Shannon Divergence (JSD) between the suspect model's empirical action distribution and the expected watermarked distribution. A pass Pᵢ is detected if `JSD(D'ᵢ, D̂ᵢ) < θⱼ` (low divergence means the suspect matches the watermark pattern). A model is flagged as stolen if the number of detected passes exceeds threshold θₙ (recommended: 3). Cosine similarity between the detected binary pass vector and stored user fingerprints enables attribution to a specific leaked user.

## Step-by-Step Workflow

1. **Inventory the agent's tool set.** Enumerate every tool/API the agent can invoke. For each tool, record its name, parameters, return type, and semantic purpose. Target domains with 20+ tools for meaningful watermark capacity.

2. **Mine semantic equivalence classes.** For each tool, search for alternatives using the five scheme types (VR, PGR, IA, AE, CE). Embed tool descriptions using a text embedding model, compute pairwise cosine similarity, and shortlist pairs above a similarity threshold (e.g., 0.85). This is the "Generator" phase.

3. **Validate equivalence rigorously.** For each candidate pair, run both alternatives in a sandboxed environment against diverse test inputs. Compare outputs and side effects. Only pairs that produce identical results across all test cases become registered watermark passes. This is the "Verifier" phase — expect roughly 50% of candidates to survive validation.

4. **Define the watermark pass registry.** Assign each validated equivalence class a pass ID. Record the equivalence set ℰᵢ and designate target action wₜ for each. Example registry entry: `P₁₂: {Gmail.SendEmail → target, Outlook.SendEmail} with δ=2.0`.

5. **Implement the UID-to-pass mapping.** Generate a random N-bit UID for each API user. Map bit positions to pass IDs — bit i=1 activates pass Pᵢ₊₁. Constrain Hamming weight to [5, 20] active passes per user to balance detectability and stealth.

6. **Inject watermarks at inference time.** Intercept the agent's action selection step. When the agent invokes a tool matching an active pass's equivalence set, apply the logit bias formula to boost the target action's probability by factor eᵟ while renormalizing alternatives. This modifies which equivalent tool gets called without changing the outcome.

7. **Build the verification dataset.** Maintain a private dataset of prompts that reliably trigger tool invocations within registered equivalence classes. This dataset must share the distribution of the original fine-tuning data but remain secret.

8. **Run verification against suspect models.** Query the suspect model with the verification dataset. Collect action trajectories (only visible tool calls needed — no access to reasoning traces or weights). For each pass, compute the empirical action distribution and calculate JSD against the expected watermarked distribution.

9. **Apply the detection decision rule.** Count passes where `JSD < θⱼ` (recommended θⱼ ∈ {0.010, 0.015}). If the count exceeds θₙ=3, classify the model as stolen. For attribution, compute cosine similarity between the detected pass vector and all stored user fingerprints — the highest match identifies the source of the leak.

10. **Monitor and maintain.** Periodically re-validate equivalence classes as APIs evolve. Rotate pass assignments for new users. Track detection metrics (false positive rate, detection power) across model updates.

## Concrete Examples

**Example 1: Watermarking an Enterprise Email/Calendar Agent**

User: "I'm building a multi-tool agent that handles email, calendar, and file operations using Microsoft 365 and Google Workspace APIs. How do I watermark it against model theft?"

Approach:
1. Inventory tools: `Gmail.Send`, `Outlook.Send`, `GCalendar.CreateEvent`, `OutlookCalendar.CreateEvent`, `GDrive.MoveFile`, `GDrive.CopyFile`, `GDrive.DeleteFile`, etc.
2. Mine equivalence classes:
   - VR Pass: `{Gmail.Send, Outlook.Send}` — both send email identically
   - VR Pass: `{GCalendar.CreateEvent, OutlookCalendar.CreateEvent}`
   - CE Pass: `{GDrive.MoveFile}` ≡ `{GDrive.CopyFile + GDrive.DeleteFile}`
   - AE Pass: `{BookMeeting}` ≡ `{BookMeeting + GetConfirmation}`
3. Validate each pair by sending test emails/events through both paths and confirming identical outcomes.
4. Register 12 passes. Assign UIDs: User A gets UID `101100010100` (6 active passes).
5. At inference, when the agent would call `Gmail.Send`, apply logit bias δ=2.0 to boost `Outlook.Send` for User A (if pass 1 is active).

Output — Pass Registry (JSON):
```json
{
  "passes": [
    {"id": "P1", "equivalence_set": ["Gmail.Send", "Outlook.Send"], "target": "Outlook.Send", "delta": 2.0, "type": "VR"},
    {"id": "P2", "equivalence_set": ["GCalendar.CreateEvent", "OutlookCalendar.CreateEvent"], "target": "OutlookCalendar.CreateEvent", "delta": 2.0, "type": "VR"},
    {"id": "P3", "equivalence_set": [["GDrive.MoveFile"], ["GDrive.CopyFile", "GDrive.DeleteFile"]], "target": 1, "delta": 1.5, "type": "CE"}
  ],
  "users": {
    "user_A": {"uid": "101100010100", "active_passes": ["P1", "P2", "P4", "P5", "P8", "P10"]}
  }
}
```

**Example 2: Detecting a Stolen Agent**

User: "A competitor released an agent that behaves suspiciously like ours. How do I verify if it was trained on our outputs?"

Approach:
1. Prepare 500 verification prompts from the private dataset that trigger tool calls in registered equivalence classes.
2. Query the suspect agent. Record only the visible action trajectories (tool names and parameters — no reasoning traces needed).
3. For each pass Pᵢ, count how often each equivalent action appears across all trajectories.
4. Compute JSD between observed distribution and expected watermarked distribution for each pass.
5. Apply decision rule.

Output — Verification Report:
```
Pass Detection Results (θⱼ = 0.015):
  P1  (Gmail/Outlook):     JSD = 0.008  ✓ DETECTED
  P2  (GCal/OutlookCal):   JSD = 0.011  ✓ DETECTED
  P3  (Move/Copy+Delete):  JSD = 0.042  ✗ not detected
  P4  (SendSMS vendors):   JSD = 0.006  ✓ DETECTED
  P5  (API v1/v2):         JSD = 0.009  ✓ DETECTED
  ...
Detected passes: 8 / 12  (θₙ = 3 → WATERMARK CONFIRMED)

Attribution (cosine similarity):
  user_A: 0.94  ← MATCH
  user_B: 0.31
  user_C: 0.18

Conclusion: Suspect model trained on outputs leaked by user_A.
```

**Example 3: Implementing the Logit Bias Interceptor**

User: "Show me how to implement the action-level watermark injection in Python."

```python
import math
from typing import Dict, List, Optional

class AgentWMInterceptor:
    def __init__(self, pass_registry: Dict, user_uid: str, delta: float = 2.0):
        self.delta = delta
        self.active_passes = self._resolve_passes(pass_registry, user_uid)

    def _resolve_passes(self, registry, uid):
        """Map UID bits to active passes."""
        active = {}
        for i, bit in enumerate(uid):
            if bit == '1' and i < len(registry["passes"]):
                p = registry["passes"][i]
                active[frozenset(p["equivalence_set"])] = p["target"]
        return active

    def intercept(self, action_probs: Dict[str, float]) -> Dict[str, float]:
        """Apply logit bias to action probabilities at the equivalence set level."""
        action_set = frozenset(action_probs.keys())
        for equiv_set, target in self.active_passes.items():
            if equiv_set.issubset(action_set):
                return self._apply_bias(action_probs, equiv_set, target)
        return action_probs

    def _apply_bias(self, probs: Dict[str, float], equiv_set, target) -> Dict[str, float]:
        """Boost target action probability via exponential scaling."""
        biased = dict(probs)
        target_p = probs[target] * math.exp(self.delta)
        others_sum = sum(probs[a] for a in equiv_set if a != target)
        normalizer = target_p + others_sum
        for action in equiv_set:
            if action == target:
                biased[action] = target_p / normalizer
            else:
                biased[action] = probs[action] / normalizer
        return biased
```

## Best Practices

- **Do:** Validate every equivalence pair with automated execution testing in a sandbox — semantic similarity alone is insufficient. The paper found roughly half of embedding-similar pairs fail execution validation.
- **Do:** Use at least 20 watermark passes per domain to provide enough capacity for unique user fingerprints and robust detection even if some passes degrade over time.
- **Do:** Set the bias strength δ between 1.5 and 2.5 — lower values reduce detectability, higher values risk perceptible behavior changes.
- **Do:** Constrain UID Hamming weight to [5, 20] active passes per user to balance attribution precision against stealth.
- **Avoid:** Watermarking actions where equivalence is approximate rather than exact — any functional difference between alternatives will degrade agent performance and alert users.
- **Avoid:** Using a single JSD threshold — test across multiple θⱼ values ({0.005, 0.010, 0.015, 0.050}) and report detection at each to understand sensitivity.

## Error Handling

- **Equivalence class breaks after API update:** If a vendor deprecates an endpoint, the corresponding pass becomes unusable. Monitor API changelogs and maintain a pass health dashboard. Retire broken passes and replace them in the registry.
- **Insufficient tool diversity:** If the agent uses fewer than 10 distinct tools, there may not be enough equivalence classes for reliable watermarking. Consider adding interface aliases (IA scheme) or auxiliary operations (AE scheme) to artificially expand the pass count.
- **Low detection power:** If JSD values are clustered near the threshold, increase the verification dataset size (500+ prompts recommended) or raise δ to strengthen the signal. Alternatively, lower θₙ from 3 to 2 (trading some false-positive risk for sensitivity).
- **Adaptive adversary strips watermarks:** If an adversary randomizes equivalent actions uniformly, watermark signal degrades but the adversary's model also loses the performance benefits of learned action preferences. The paper shows watermark removal causes measurable utility degradation.
- **Attribution collision:** Two users with similar UIDs may have high cosine similarity. Ensure UID generation uses cryptographically random bits and enforce minimum Hamming distance between assigned UIDs.

## Limitations

- Requires the agent to have **multiple functionally equivalent tools** — agents with narrow, non-redundant toolsets cannot be meaningfully watermarked with this approach.
- Only protects against **imitation attacks via output distillation** — does not protect against weight theft, prompt extraction, or architecture cloning.
- Verification requires a **private dataset** of prompts that trigger equivalence-class tool calls. If this dataset leaks, the adversary can specifically avoid watermarked paths.
- The grey-box assumption (visible action trajectories) must hold — if the suspect system further obscures its tool calls, verification becomes impossible.
- Watermark capacity scales with the number of validated equivalence passes. Domains with fewer than ~15 passes may not support enough unique user fingerprints for reliable attribution at scale.

## Reference

**Paper:** [On Protecting Agentic Systems' Intellectual Property via Watermarking](https://arxiv.org/abs/2602.08401v1) — Wang et al., 2026. Look for Section 3 (framework design with the five equivalence scheme types), Equation 1 (logit bias formula), and Section 5 (the automated Generator-Verifier pipeline producing 101 validated passes across three domains).