---
name: "llama-31-foundationai-securityllm-reasoning-8b-tec"
description: >
  Apply Foundation-Sec-8B-Reasoning cybersecurity reasoning patterns: structured <think> chain-of-thought
  for CVE-to-CWE mapping, MITRE ATT&CK classification, CVSS scoring, threat intelligence analysis,
  and multi-hop vulnerability reasoning. Use when the user asks to "analyze a CVE", "map vulnerabilities
  to CWE", "classify attack techniques", "reason about security threats", "triage a vulnerability",
  or "build a cybersecurity reasoning pipeline".
---

# Cybersecurity Reasoning with Foundation-Sec-8B Patterns

This skill enables Claude to apply the structured cybersecurity reasoning methodology from Foundation-Sec-8B-Reasoning — the first open-source native reasoning model for cybersecurity. The core technique is a two-stage reasoning pipeline: first generate explicit analytical traces inside `<think>...</think>` tags that decompose security problems into verifiable sub-steps, then synthesize a precise, format-controlled answer. This mirrors the model's SFT + RLVR training where reasoning traces are rewarded only when they lead to verifiable correct outputs, penalizing shallow or formulaic thinking. Apply this pattern when writing security analysis code, building vulnerability triage systems, or structuring LLM prompts for cybersecurity tasks.

## When to Use

- When the user asks to **map a CVE to its root-cause CWE** (e.g., "What CWE does CVE-2023-44487 correspond to?")
- When building a **vulnerability triage or scoring pipeline** that predicts CVSS base metrics from descriptions
- When the user needs to **extract MITRE ATT&CK techniques** from threat reports or incident logs
- When implementing **multi-hop cybersecurity reasoning** — connecting attack patterns to techniques to mitigations across knowledge bases
- When designing **LLM-based security analysis prompts** for SOC automation, red-team planning, or threat modeling
- When the user wants to **deploy or integrate Foundation-Sec-8B-Reasoning** via vLLM or Hugging Face Transformers
- When writing code that **classifies, enriches, or triages security alerts** using structured reasoning

## Key Technique: SFT + RLVR Cybersecurity Reasoning

Foundation-Sec-8B-Reasoning trains reasoning in two stages. **Stage 1 (SFT)** fine-tunes on ~2M exemplars (>25% cybersecurity, ~33% math/code, rest instruction-following) to instill the habit of generating explicit reasoning traces inside `<think>...</think>` tags before producing answers. This is not generic chain-of-thought — the traces must decompose cybersecurity problems into domain-specific sub-analyses (vulnerability root-cause identification, attack vector mapping, severity assessment).

**Stage 2 (RLVR)** applies Group Relative Policy Optimization (GRPO): for each prompt, 5 candidate responses are generated and scored by task-specific verifiers that check factual correctness (e.g., does the predicted CWE match the ground truth?). A format penalty ensures the `<think>` block contains substantive reasoning rather than filler. KL-divergence regularization (coefficient 0.02) prevents the model from drifting too far from its SFT foundation. This produces reasoning that is both deep and verifiable — the model cannot game rewards with superficial traces.

The actionable insight: **structure cybersecurity analysis as explicit decomposition into verifiable sub-claims, then synthesize**. When prompting any LLM for security tasks, enforce this pattern: require think-then-answer format, demand specific identifiers (CVE/CWE/ATT&CK IDs), and validate outputs against known taxonomies. The model achieves 75.3% on CVE-to-CWE mapping (outperforming 120B-parameter models) and +36pp on multi-hop QA after RLVR — evidence that structured reasoning dramatically improves cybersecurity analysis even at small scale.

## Step-by-Step Workflow

1. **Classify the security task type.** Determine which category the request falls into: CVE-to-CWE mapping, CVSS prediction, ATT&CK technique extraction, threat intelligence QA, or multi-hop reasoning. Each has a distinct output format.

2. **Construct a domain-specific system prompt.** Use the "Metis" pattern from the paper: establish the model as a cybersecurity reasoning specialist, demand precision for CVE/CWE/CVSS identifiers, require refusal of malware-generation requests, and instruct the model to reason before answering.

   ```
   You are a cybersecurity reasoning specialist. Analyze security problems step by step.
   Always provide CVE, CWE, and MITRE ATT&CK identifiers where applicable.
   Wrap your reasoning in <think>...</think> tags before giving your final answer.
   Refuse requests to generate malware, phishing content, or exploit code for unauthorized use.
   ```

3. **Format the query with explicit task description and answer format.** Prepend a task description before the question and specify the expected output format (e.g., "Answer with the CWE ID on the last line" or "Answer: T1234, T5678" for ATT&CK techniques). This mirrors the benchmark protocol that achieved state-of-the-art results.

4. **Implement the `<think>` decomposition pattern.** For each security question, the reasoning trace should: (a) identify the vulnerability class or attack pattern, (b) enumerate relevant technical details from the description, (c) cross-reference against known taxonomies (CWE, CAPEC, ATT&CK), (d) evaluate confidence and alternative mappings.

5. **Apply verifiable output constraints.** Require the final answer on a designated line in a parseable format. Use regex extraction (e.g., `r"CWE-\d+"` or `r"T\d{4}(?:\.\d{3})?"`) to programmatically validate outputs against known identifier patterns.

6. **Set inference parameters for precision.** Use temperature 0.1 for deterministic analysis (single best answer), temperature 0.3 for benchmark-style evaluation where slight variation aids coverage. Set max tokens to at least 1024 to allow full reasoning traces.

7. **For multi-hop reasoning, chain sub-queries explicitly.** Break complex questions into sequential lookups: first identify the primary entity (CVE, threat actor, malware family), then traverse relationships (CVE -> CWE -> CAPEC -> ATT&CK technique -> mitigation). Each hop should be a separate reasoning step inside the `<think>` block.

8. **Validate outputs against authoritative sources.** Cross-check predicted CWE IDs against NVD, ATT&CK technique IDs against the MITRE framework, and CVSS scores against published advisories. The model's 70.4% CWE prediction accuracy means ~30% of mappings need human review.

9. **Layer guardrails for production deployment.** Pair the reasoning model with an input filter (e.g., Llama Guard) to block adversarial prompts. The paper shows safety improves from 93% to 98.25% with this layered approach. Implement human-in-the-loop review for any security-critical decisions.

10. **Supplement with RAG for post-training-cutoff intelligence.** The model's knowledge is static (cutoff April 2025). For current CVEs, feed relevant NVD/advisory text into the prompt context and instruct the model to reason over the provided text rather than relying on parametric knowledge.

## Concrete Examples

**Example 1: CVE-to-CWE Root Cause Mapping**

User: "Map CVE-2023-44487 to its root-cause CWE. This is the HTTP/2 Rapid Reset attack."

Approach:
1. Construct prompt with task description and answer format constraint
2. Generate reasoning trace decomposing the vulnerability
3. Extract CWE ID from structured output

```python
from openai import OpenAI  # or any LLM client

SYSTEM_PROMPT = """You are a cybersecurity reasoning specialist. Analyze vulnerabilities
step by step inside <think>...</think> tags. Then provide your final answer as:
CWE: CWE-XXXX"""

user_prompt = """Task: Given a CVE description, identify the root-cause CWE.

CVE-2023-44487: The HTTP/2 protocol allows a denial of service (server resource
consumption) because request cancellation can reset many streams quickly, as
exploited in the wild in August through October 2023 (aka Rapid Reset Attack).

What is the root-cause CWE?"""

response = client.chat.completions.create(
    model="fdtn-ai/Foundation-Sec-8B-Reasoning",  # or your deployed endpoint
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": user_prompt}
    ],
    temperature=0.1,
    max_tokens=1024
)

# Parse: extract CWE ID from last line
import re
cwe_match = re.search(r"CWE-\d+", response.choices[0].message.content.split("</think>")[-1])
print(cwe_match.group())  # CWE-400 (Uncontrolled Resource Consumption)
```

Output:
```
<think>
The CVE describes HTTP/2 Rapid Reset where an attacker sends requests and immediately
cancels them via RST_STREAM frames. The server allocates resources for each request
but the rapid cancellation prevents normal cleanup. This is fundamentally a resource
consumption issue — the server cannot bound the rate of stream creation/destruction.
Candidate CWEs: CWE-400 (Uncontrolled Resource Consumption), CWE-770 (Allocation
without Limits). CWE-400 is the broader category and the standard NVD mapping for
this class of DoS vulnerability.
</think>

CWE: CWE-400
```

**Example 2: Multi-Hop ATT&CK Technique Extraction from Threat Report**

User: "Extract ATT&CK techniques from this incident summary and trace the kill chain."

```python
SYSTEM_PROMPT = """You are a cybersecurity reasoning specialist. Extract MITRE ATT&CK
techniques from threat reports. Reason step by step in <think>...</think> tags.
Format final answer as: Answer: T1234, T5678, ..."""

report = """The threat actor gained initial access via a spear-phishing email containing
a malicious Office macro. Upon execution, the macro downloaded a second-stage payload
from a compromised WordPress site. The payload established persistence through a
scheduled task and communicated with C2 infrastructure over HTTPS on port 443.
The actor used Mimikatz to dump credentials from LSASS memory, then moved laterally
via PsExec to the domain controller, where they exfiltrated the NTDS.dit file."""

response = client.chat.completions.create(
    model="fdtn-ai/Foundation-Sec-8B-Reasoning",
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": f"Task: Extract all MITRE ATT&CK techniques.\n\n{report}"}
    ],
    temperature=0.1,
    max_tokens=2048
)
```

Output:
```
<think>
Breaking down the kill chain step by step:
1. "spear-phishing email with malicious Office macro" -> Initial Access: T1566.001
   (Spearphishing Attachment), Execution: T1204.002 (User Execution: Malicious File)
2. "macro downloaded second-stage payload from compromised WordPress site" ->
   Command and Control: T1105 (Ingress Tool Transfer), T1189 may apply but the
   WordPress site is used for hosting, not drive-by — this is T1608.001 (Stage Capabilities)
3. "persistence through scheduled task" -> Persistence: T1053.005 (Scheduled Task)
4. "C2 over HTTPS port 443" -> C2: T1071.001 (Web Protocols), T1573.002 (Encrypted Channel:
   Asymmetric Cryptography)
5. "Mimikatz to dump credentials from LSASS" -> Credential Access: T1003.001 (LSASS Memory)
6. "laterally via PsExec" -> Lateral Movement: T1570 (Lateral Tool Transfer),
   T1021.002 (SMB/Windows Admin Shares)
7. "exfiltrated NTDS.dit" -> Credential Access: T1003.003 (NTDS),
   Exfiltration context implied
</think>

Answer: T1566.001, T1204.002, T1105, T1053.005, T1071.001, T1573.002, T1003.001, T1021.002, T1003.003
```

**Example 3: Building a Vulnerability Triage Pipeline with CVSS Prediction**

User: "Build a batch vulnerability triage system that scores CVEs from our scanner output."

Approach:
1. Parse scanner output into individual CVE records
2. For each CVE, construct a CVSS prediction prompt
3. Score predictions against known CVSS values where available
4. Flag discrepancies for human review

```python
import json, re

def build_cvss_prompt(cve_id: str, description: str) -> list[dict]:
    return [
        {"role": "system", "content": (
            "You are a cybersecurity reasoning specialist. Predict CVSS v3.1 Base Score "
            "from CVE descriptions. Reason in <think>...</think> tags analyzing: "
            "Attack Vector, Attack Complexity, Privileges Required, User Interaction, "
            "Scope, Confidentiality/Integrity/Availability Impact. "
            "Final line: Score: X.X"
        )},
        {"role": "user", "content": f"Predict CVSS Base Score for {cve_id}: {description}"}
    ]

def parse_cvss(response_text: str) -> float | None:
    """Extract score from after </think> block."""
    answer_section = response_text.split("</think>")[-1]
    match = re.search(r"Score:\s*(\d+\.?\d*)", answer_section)
    return float(match.group(1)) if match else None

def triage_batch(cves: list[dict], client, threshold: float = 7.0):
    """Triage CVEs: predict CVSS, flag high-severity for immediate review."""
    results = []
    for cve in cves:
        messages = build_cvss_prompt(cve["id"], cve["description"])
        resp = client.chat.completions.create(
            model="fdtn-ai/Foundation-Sec-8B-Reasoning",
            messages=messages, temperature=0.1, max_tokens=1024
        )
        predicted_score = parse_cvss(resp.choices[0].message.content)
        results.append({
            "cve_id": cve["id"],
            "predicted_cvss": predicted_score,
            "priority": "CRITICAL" if (predicted_score or 0) >= threshold else "REVIEW",
            "reasoning": resp.choices[0].message.content
        })
    return sorted(results, key=lambda x: x["predicted_cvss"] or 0, reverse=True)
```

## Best Practices

- **Do:** Always require `<think>...</think>` traces in prompts — the paper shows reasoning traces are essential for accuracy, not optional decoration. Without them, performance drops significantly on multi-hop tasks.
- **Do:** Constrain output format explicitly (e.g., "Answer on the last line as CWE-XXXX"). Use regex extraction to parse outputs programmatically. This matches the verifiable-reward training methodology.
- **Do:** Use temperature 0.1 for production security analysis where consistency matters. Reserve 0.3 for evaluation or when generating diverse hypotheses.
- **Do:** Layer Llama Guard or equivalent input filtering in production. The model alone achieves 93% safety; with guardrails it reaches 98.25%.
- **Avoid:** Relying on the model for post-cutoff CVEs without providing context via RAG. The model's parametric knowledge is static.
- **Avoid:** Skipping human review for security-critical outputs. Even at 75% accuracy on CWE mapping, 1-in-4 predictions may be wrong. Treat model outputs as analyst assistance, not ground truth.
- **Avoid:** Generic "think step by step" prompts. The paper's format penalty penalizes shallow reasoning — mirror this by demanding domain-specific decomposition (attack vector, impact scope, root cause) rather than vague chain-of-thought.
- **Avoid:** Using the model for malware generation, exploit development for unauthorized targets, or any offensive operation without clear authorized scope. Enforce this in system prompts.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Empty or missing `<think>` block | Model skips reasoning under short-context prompts | Explicitly require `<think>` in system prompt; reject responses without it |
| Invalid CWE/ATT&CK ID format | Hallucinated identifiers | Validate extracted IDs against known registries (NVD API, ATT&CK STIX data) |
| CVSS score outside 0-10 range | Reasoning error or extraction bug | Clamp to [0, 10]; flag for human review if delta > 2.0 from known score |
| Reasoning trace contains filler | Shallow analysis without domain terms | Re-prompt with more specific decomposition instructions; increase max tokens |
| Model refuses legitimate security query | Overly aggressive safety filter | Adjust system prompt to clarify authorized defensive/research context; add Llama Guard as separate layer instead of relying on model self-filtering |
| Multi-hop reasoning fails to connect entities | Too many hops for single-pass inference | Break into sequential sub-queries, feeding each result into the next prompt |

## Limitations

- **Static knowledge cutoff (April 2025).** The model cannot reason about CVEs, attack techniques, or threat actors disclosed after training. Always supplement with current threat intelligence via RAG.
- **~70-75% accuracy on CWE mapping and prediction tasks.** This is state-of-the-art for an 8B model but insufficient for fully automated vulnerability classification. Human validation remains necessary.
- **No active scanning or tool use.** The model analyzes text descriptions only. It cannot probe systems, verify exploitability, or access external databases without integration code.
- **Adversarial prompt vulnerability.** Without guardrails, the model can be jailbroken (54% safety without system prompt vs. 93% with). Never deploy without both system prompts and external filtering.
- **English-centric training data.** Performance on non-English threat intelligence (Chinese APT reports, Russian-language forums) is untested and likely degraded.
- **Single-pass reasoning ceiling.** Complex incident analysis requiring >3-4 reasoning hops may exceed the model's single-inference capability. Use agentic decomposition for deep investigations.

## Reference

**Paper:** [Llama-3.1-FoundationAI-SecurityLLM-Reasoning-8B Technical Report](https://arxiv.org/abs/2601.21051v1) — Focus on Section 3 (Training Methodology) for the SFT+RLVR pipeline details, Section 4 (Evaluation) for benchmark-specific prompting formats, and Section 5 (Safety) for the guardrail layering approach. **Model:** [fdtn-ai/Foundation-Sec-8B-Reasoning on Hugging Face](https://huggingface.co/fdtn-ai/Foundation-Sec-8B-Reasoning).