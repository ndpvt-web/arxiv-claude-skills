---
name: "draincode-stealthy-energy-consumption"
description: "Evaluate and defend RAG-based code generation systems against energy-drain attacks that poison retrieval contexts to inflate LLM output length, latency, and GPU energy consumption. Use when: 'audit my RAG pipeline for energy attacks', 'test code retrieval poisoning resilience', 'detect adversarial triggers in retrieved code', 'harden my code generation system against context poisoning', 'benchmark energy cost of retrieval-augmented code generation', 'simulate DrainCode-style attacks on my pipeline'."
---

# DrainCode: Defending Against Energy-Drain Attacks on RAG Code Generation

This skill enables Claude to audit, test, and harden retrieval-augmented code generation (RAG) systems against DrainCode-style adversarial attacks. DrainCode (Wang et al., 2026) demonstrated that an attacker can poison a retrieval corpus with adversarial trigger tokens embedded in code snippets, causing LLMs to suppress their end-of-sequence (EOS) token and produce outputs 2-10x longer than normal -- inflating GPU latency by up to 182% and energy consumption by up to 155%, while largely preserving functional correctness. This skill teaches how to detect such poisoning, build defenses, and stress-test RAG pipelines for computational-efficiency vulnerabilities.

## When to Use This Skill

- When building or auditing a RAG-based code generation system and you need to assess its resilience to retrieval poisoning
- When you notice anomalously long outputs from a code-generation LLM and suspect context contamination
- When designing a retrieval corpus ingestion pipeline and need to filter adversarial snippets
- When benchmarking the energy and latency profile of a code generation system under adversarial conditions
- When implementing detection layers (perplexity filters, classifier-based guards) for retrieved code contexts
- When red-teaming a code assistant that retrieves snippets from public repositories (e.g., HuggingFace datasets, GitHub)

## Key Technique

**The Attack Model.** DrainCode targets the standard RAG code-generation pipeline: a retriever (typically BM25 or embedding-based) fetches top-k code snippets from a corpus, concatenates them with the user's incomplete code as context, and feeds the combined prompt to an LLM for completion. The attacker poisons 1-3 snippets per query in the retrieval corpus by embedding short adversarial trigger token sequences within syntactically valid code blocks. These triggers are optimized via gradient-based search to minimize EOS probability across all generation positions (L1 loss) while maximizing hidden-state diversity to encourage varied, lengthy output (L2 nuclear-norm loss), subject to a KL-divergence constraint that keeps non-trigger output distributions close to clean baselines -- making the poisoning hard to detect by surface-level metrics.

**Why It Matters for Defense.** Existing defenses performed poorly against DrainCode: SVM classifiers on hidden representations achieved only 30-37% detection accuracy, perplexity-based filters scored 28-29%, and fine-tuned CodeBERT reached 51-62% but is limited to 512 tokens. This means production systems need layered, purpose-built defenses: output-length anomaly detection, token-budget enforcement, retrieval-time perplexity gating on full-length snippets, and energy-consumption monitoring. The attack's transferability across models (tested on DeepSeek-Coder-7B, CodeQwen-7B, Internlm2-7B, Llama3-8B) and prompting strategies makes defense-in-depth essential.

**The Defensive Opportunity.** Because triggers are gradient-optimized token sequences, they often contain unusual token co-occurrences within otherwise normal code. Statistical anomaly detection on token bigram/trigram distributions within retrieved snippets, combined with strict output-length budgets and energy-per-query monitoring, provides the most practical defense stack.

## Step-by-Step Workflow

### For Auditing a RAG Pipeline

1. **Map the retrieval architecture.** Identify the retriever type (BM25, dense embedding, hybrid), corpus source (public repos, curated datasets), number of retrieved snippets (k), and how context is concatenated with user queries. Document the exact prompt template used.

2. **Establish clean baselines.** Run 100+ representative code-generation queries against the unmodified pipeline. Record per-query metrics: output token count, wall-clock latency, GPU energy (via `nvidia-smi` or `codecarbon`), and functional correctness (pass@1 on unit tests). Compute mean, median, P95, and P99 for each metric.

3. **Craft probe snippets that simulate adversarial triggers.** Insert sequences of low-frequency tokens (e.g., rarely-used Unicode identifiers, unusual variable names with high token entropy) into otherwise valid code snippets. Place these at comment boundaries, inside docstrings, and between function definitions -- the positions DrainCode targets.

4. **Inject probe snippets into the retrieval corpus.** Add 1-3 poisoned snippets per test query to the corpus, ensuring they rank in the top-k results for target queries. Re-run the baseline query set and compare output length, latency, and energy against clean baselines.

5. **Detect anomalous outputs.** Flag any query where output length exceeds 2x the P95 baseline, latency exceeds 1.5x baseline, or energy exceeds 1.5x baseline. These thresholds correspond to the lower bound of DrainCode's demonstrated impact.

6. **Implement retrieval-time filtering.** Add a perplexity gate: compute per-token perplexity of each retrieved snippet using a small language model (e.g., a 1B-parameter model). Reject snippets whose perplexity exceeds 2 standard deviations above the corpus mean. Also compute token bigram entropy and flag statistical outliers.

7. **Enforce output-length budgets.** Set a hard `max_new_tokens` limit at 2x the P95 clean output length for your task distribution. This directly caps the energy amplification factor regardless of context poisoning.

8. **Deploy runtime energy monitoring.** Instrument the inference server to log GPU energy per query (using `codecarbon`, `pyJoules`, or direct NVML calls). Set alerts when rolling-average energy exceeds 1.3x the clean baseline over a 5-minute window.

9. **Validate defenses under attack simulation.** Re-run the probe-snippet injection from step 4 with all defenses active. Confirm that flagged queries are caught, output lengths are capped, and energy stays within budget. Measure any impact on clean-query performance (false positive rate, latency overhead of filtering).

10. **Document the threat model and residual risk.** Record which attack vectors are mitigated (corpus poisoning, prompt injection), which are partially mitigated (white-box trigger optimization against your specific model), and which require ongoing monitoring (novel trigger patterns, model updates changing vulnerability surface).

## Concrete Examples

**Example 1: Auditing a Code Completion Service**

User: "I run a RAG code completion service using BM25 retrieval over a public Python snippet corpus and DeepSeek-Coder-7B. How do I test if it's vulnerable to energy-drain attacks?"

Approach:
1. Instrument the inference endpoint with `codecarbon` to measure energy per query
2. Collect baseline metrics over 200 representative Python completion queries
3. Create 50 poisoned snippets by inserting high-entropy token sequences into valid Python functions:

```python
# Probe snippet example: adversarial tokens embedded in a valid function
def calculate_total(items):
    # xtq_7kz mf_2rp vbn_9wl  <-- unusual token sequence simulating trigger
    total = 0
    for item in items:
        total += item.price * item.quantity
    return total
```

4. Inject these into the BM25 index targeting queries about list/dict operations
5. Compare output lengths and energy: if mean output tokens jump from ~300 to ~800+, the system is vulnerable
6. Apply fix: add `max_new_tokens=600` (2x P95 baseline) and perplexity filtering on retrieved snippets

Output:
```
Baseline: mean=298 tokens, P95=412 tokens, energy=0.42 Wh/query
With probes: mean=847 tokens, P95=1,203 tokens, energy=0.89 Wh/query
Verdict: VULNERABLE (2.8x output inflation, 2.1x energy increase)
After mitigation: mean=305 tokens, P95=420 tokens, energy=0.44 Wh/query
False positive rate on clean queries: 0.8%
```

**Example 2: Building a Retrieval Filter**

User: "Write a retrieval-time filter that screens out potentially poisoned code snippets before they reach the LLM."

Approach:
1. Load a small reference model for perplexity scoring
2. Compute corpus-wide perplexity statistics
3. Filter each retrieved snippet before prompt assembly

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer
import numpy as np

class RetrievalPoisonFilter:
    def __init__(self, model_name="microsoft/phi-2", threshold_std=2.0):
        self.tokenizer = AutoTokenizer.from_pretrained(model_name)
        self.model = AutoModelForCausalLM.from_pretrained(
            model_name, torch_dtype=torch.float16, device_map="auto"
        )
        self.threshold_std = threshold_std
        self.corpus_stats = None  # calibrated during setup

    def calibrate(self, clean_snippets: list[str], sample_size=500):
        """Compute perplexity distribution over known-clean snippets."""
        perplexities = []
        for snippet in clean_snippets[:sample_size]:
            ppl = self._compute_perplexity(snippet)
            perplexities.append(ppl)
        self.corpus_stats = {
            "mean": np.mean(perplexities),
            "std": np.std(perplexities),
        }

    def is_suspicious(self, snippet: str) -> bool:
        """Return True if snippet perplexity is anomalously high."""
        if self.corpus_stats is None:
            raise RuntimeError("Call calibrate() first")
        ppl = self._compute_perplexity(snippet)
        threshold = (
            self.corpus_stats["mean"]
            + self.threshold_std * self.corpus_stats["std"]
        )
        return ppl > threshold

    def filter_retrievals(self, snippets: list[str]) -> list[str]:
        """Remove suspicious snippets from retrieval results."""
        return [s for s in snippets if not self.is_suspicious(s)]

    def _compute_perplexity(self, text: str) -> float:
        inputs = self.tokenizer(text, return_tensors="pt", truncation=True,
                                max_length=2048).to(self.model.device)
        with torch.no_grad():
            outputs = self.model(**inputs, labels=inputs["input_ids"])
        return torch.exp(outputs.loss).item()
```

4. Integrate into the RAG pipeline between retrieval and prompt assembly

**Example 3: Energy Monitoring Dashboard**

User: "Set up energy-per-query monitoring for my code generation API to detect ongoing attacks."

Approach:
1. Wrap the inference endpoint with energy measurement
2. Maintain a rolling window for anomaly detection
3. Alert when energy exceeds threshold

```python
from codecarbon import EmissionsTracker
from collections import deque
import statistics

class EnergyAnomalyDetector:
    def __init__(self, window_size=100, alert_multiplier=1.5):
        self.window = deque(maxlen=window_size)
        self.alert_multiplier = alert_multiplier
        self.baseline_median = None

    def calibrate(self, baseline_energies: list[float]):
        self.baseline_median = statistics.median(baseline_energies)

    def record_and_check(self, energy_kwh: float) -> dict:
        self.window.append(energy_kwh)
        rolling_median = statistics.median(self.window)
        is_anomalous = (
            self.baseline_median is not None
            and rolling_median > self.baseline_median * self.alert_multiplier
        )
        return {
            "energy_kwh": energy_kwh,
            "rolling_median": rolling_median,
            "baseline_median": self.baseline_median,
            "alert": is_anomalous,
        }

# Usage in inference endpoint:
def generate_with_monitoring(query, retrieved_context, model, detector):
    tracker = EmissionsTracker(log_level="error")
    tracker.start()
    output = model.generate(retrieved_context + query)
    emissions = tracker.stop()
    result = detector.record_and_check(emissions)
    if result["alert"]:
        log_security_event("energy_anomaly", result)
    return output
```

## Best Practices

- **Do:** Set explicit `max_new_tokens` limits on all code generation endpoints. This is the single most effective mitigation -- it directly caps the amplification factor regardless of trigger sophistication.
- **Do:** Calibrate perplexity filters on your actual corpus, not generic benchmarks. Token distributions vary significantly between Python web code, C++ systems code, and data science notebooks.
- **Do:** Monitor output length distributions in production with rolling percentile tracking. A sudden shift in the P95 output length is an early indicator of corpus poisoning.
- **Do:** Validate retrieved snippets with multiple signals (perplexity, token bigram entropy, code parsability) rather than relying on any single detector. The paper showed each individual defense achieves under 62% accuracy.
- **Avoid:** Relying solely on CodeBERT-based classifiers for detection -- their 512-token limit means most real code snippets are truncated, losing trigger tokens that may appear later in the snippet.
- **Avoid:** Assuming functional correctness implies safety. DrainCode maintains near-baseline pass@1 scores (only 1-2 percentage point drops) while dramatically inflating compute costs.

## Error Handling

- **Perplexity filter rejects too many clean snippets (high false-positive rate):** Loosen the threshold from 2.0 to 2.5 standard deviations, or switch to a percentile-based threshold (e.g., reject above P99 of the calibration set). Re-calibrate on a larger, more representative sample of clean snippets.
- **Energy monitoring triggers false alerts during legitimate long-generation tasks:** Segment monitoring by query type (single-line completion vs. full-function generation) and maintain separate baselines for each. Use query-type-aware thresholds.
- **Retrieved snippets fail to parse after filtering:** Ensure the filter returns the original unmodified snippets that pass screening. Never modify snippet content during filtering -- only accept or reject.
- **Calibration data is stale after corpus updates:** Re-run calibration whenever more than 10% of the retrieval corpus changes. Automate this as part of the corpus ingestion pipeline.

## Limitations

- **White-box triggers are model-specific.** Triggers optimized against DeepSeek-Coder-7B may not transfer fully to GPT-4 or Claude. However, the paper showed meaningful cross-model transfer, so defense should not assume model-specificity provides safety.
- **Perplexity filtering adds latency.** Running a separate model for per-snippet scoring adds 10-50ms per retrieved snippet. For latency-sensitive applications, consider async pre-computation or caching perplexity scores at indexing time.
- **Adaptive attackers.** A sophisticated attacker aware of perplexity filtering can add a perplexity constraint to their optimization objective, producing triggers that evade perplexity-based detection. Defense must remain layered.
- **Closed-source models.** The original attack requires gradient access (white-box). Against closed-source APIs, the attack is harder to mount but not impossible -- transfer attacks from open-source surrogates remain viable.
- **Energy measurement granularity.** Per-query GPU energy measurement via NVML or codecarbon has noise at short timescales. Reliable anomaly detection requires aggregation over windows of 50+ queries.

## Reference

Wang, Y., Wu, J., Jiang, T., Liu, M., & Chen, J. (2026). *DRAINCODE: Stealthy Energy Consumption Attacks on Retrieval-Augmented Code Generation via Context Poisoning.* arXiv:2601.20615v3. [https://arxiv.org/abs/2601.20615v3](https://arxiv.org/abs/2601.20615v3)

Key sections to study: Section 3 (attack formulation with EOS loss + nuclear-norm diversity loss + KL constraint), Section 4.2 (per-model results showing 118-226% output inflation), and Section 5 (defense evaluation showing all tested detectors below 62% accuracy).