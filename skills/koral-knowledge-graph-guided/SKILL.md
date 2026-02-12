---
name: "koral-knowledge-graph-guided"
description: "Build Knowledge Graph-guided LLM reasoning pipelines for operational telemetry analysis. Combines a Literature KG (extracted from domain papers) with a Data KG (materialized from telemetry) to ground LLM outputs in structured evidence. Triggers: 'build a knowledge graph from telemetry', 'KG-guided LLM reasoning', 'analyze SSD or hardware telemetry with a knowledge graph', 'ground LLM analysis in structured evidence', 'knowledge graph pipeline for operational data', 'extract knowledge graph from research papers'"
---

KORAL enables Claude to build end-to-end Knowledge Graph-guided LLM reasoning systems that turn fragmented operational telemetry and domain literature into structured, queryable graphs, then use those graphs to constrain and ground LLM analysis. The core technique is a dual-KG architecture: a **Literature KG** extracted from research papers via taxonomy-aligned chain-of-thought prompting, and a **Data KG** materialized from time-series telemetry via rule-based summarization. Both graphs feed structured context into LLM prompts, producing evidence-backed Descriptive, Predictive, Prescriptive, and What-if analysis without requiring large training datasets or deep domain expertise from the operator.

## When to Use

- When the user wants to build a system that extracts structured knowledge graphs from research papers or technical documents using LLMs
- When the user needs to analyze hardware telemetry (SMART data, sensor readings, workload metrics) with LLM reasoning grounded in domain knowledge
- When the user asks to combine knowledge graphs with LLMs for explainable operational diagnostics
- When building a pipeline that must produce evidence-cited, hallucination-resistant analysis from an LLM
- When the user wants to implement the four analysis modes: descriptive (what happened), predictive (what will happen), prescriptive (what to do), or what-if (counterfactual scenarios)
- When the user needs to turn unstructured domain documents into RDF/Turtle knowledge graphs with provenance tracking

## Key Technique

KORAL's insight is that raw telemetry and raw LLM reasoning are each insufficient alone. Telemetry is fragmented across time windows and attribute families (media wear, interface errors, environmental conditions), while LLMs hallucinate when operating without structured domain constraints. The solution is a **dual Knowledge Graph** that acts as both a data organizer and a reasoning guardrail.

**Stage I: Literature KG Construction.** Research papers are parsed into clean text with section anchors. An LLM receives each document along with a domain taxonomy (a hierarchical vocabulary of concepts like `Temperature`, `IOPS`, `GarbageCollection`, `TailLatency`) and produces strict JSON containing entities, triples, evidence sentences, and confidence scores. Each extracted relation (e.g., `Temperature --degrades--> 99thPercentileLatency`) maps to the closest taxonomy path and carries provenance back to the source sentence. Outputs serialize to RDF Turtle format and merge into a global knowledge graph. This turns scattered literature into a queryable evidence store.

**Stage II: Data KG + KG-Guided Reasoning.** Telemetry (e.g., 30-day SMART attribute windows) passes through a rule base that computes robust summaries (median, p95, max), temporal features (trend slopes, Mann-Kendall statistics, change points), and data quality indicators (coverage, missingness, imputation flags). These materialize as typed KG nodes (AttributeFrame, WorkloadFrame, EnvFrame) connected by semantic group edges. At inference time, the system constructs a prompt from three elements: (1) the user query, (2) a compact Data KG summary with units and coverage, and (3) SPARQL-retrieved Literature KG snippets with citations. This structured context constrains the LLM to produce grounded, domain-aligned analysis across all four reasoning modes.

## Step-by-Step Workflow

1. **Define the domain taxonomy.** Create a `taxonomy.json` file containing a hierarchical vocabulary of domain concepts organized into semantic families (e.g., `media/wear`, `interface/errors`, `environment/thermal`). Each concept gets a canonical name, aliases, and a type designation (class vs. instance).

2. **Build the Literature KG extraction prompt.** Write a chain-of-thought prompt that instructs the LLM to: (a) identify entities matching taxonomy concepts, (b) extract directional triples with predicates like `degrades`, `mitigatedBy`, `correlatesWith`, (c) attach an evidence sentence and confidence score (0-1) to each triple, and (d) normalize synonyms to canonical taxonomy terms. Enforce strict JSON output schema.

3. **Run Stage I extraction over documents.** For each paper/report, parse text with section anchors, send it through the extraction prompt, validate outputs against the taxonomy (schema completeness, ontology type checks, evidence span alignment), and serialize accepted triples to per-document RDF Turtle files. Merge all files into a `global_knowledge_graph.ttl`.

4. **Define the telemetry rule base.** Create a `rule_base.json` mapping raw telemetry attributes to semantic groups, ideal directions (Low/High/Monitor), windowing parameters, and summarization functions. Specify which robust statistics (median, p95, max) and temporal features (trend slope, change points, exposure above thresholds) to compute for each attribute family.

5. **Materialize the Data KG from telemetry.** For each device/time window, apply the rule base to produce typed frames (AttributeFrame, WorkloadFrame, EnvFrame) with explicit DataQuality nodes tracking coverage and imputation. Connect frames via semantic group edges and window-scoped identifiers. Optionally serialize to RDF Turtle for audit.

6. **Implement SPARQL-based Literature KG retrieval.** Given an analysis query and the Data KG summary, extract key concepts (e.g., temperature, write amplification) and query the Literature KG for relevant subgraphs. Retrieve triples with their evidence sentences and confidence scores, ranked by relevance to the detected conditions.

7. **Compose the three-part LLM prompt.** Assemble: (a) the user's analysis question with the target reasoning mode (descriptive/predictive/prescriptive/what-if), (b) the compact Data KG summary preserving units, coverage, and temporal descriptors, and (c) the retrieved Literature KG snippets with source citations. Never pass raw telemetry directly.

8. **Generate and validate the analysis.** Send the composed prompt to the LLM. Parse the response into atomic claims. Verify each claim against the Data KG (checking units, coverage, window scope) and the Literature KG (checking directional consistency). Compute grounding metrics: Faithfulness Precision (supported claims / total claims) and Counterfactual Validity for what-if scenarios.

9. **Produce structured output with provenance.** Format the final analysis as JSON containing the reasoning text, cited evidence triples, confidence scores, and data quality caveats. For predictive mode, include probability scores and time horizons. For prescriptive mode, include recommended actions with mechanism-level citations.

10. **Support fleet-level aggregation.** For batch analysis, group devices into cohorts, run per-device analysis, then aggregate findings into fleet-level summaries with distribution statistics across the cohort. Track which Literature KG evidence applies across multiple devices.

## Concrete Examples

**Example 1: Building a Literature KG from SSD Research Papers**

User: "I have a folder of 20 SSD research papers as PDFs. Help me build a knowledge graph that captures how environmental factors affect SSD performance."

Approach:
1. Create `taxonomy.json` with relevant concept hierarchy:
```json
{
  "environment": {
    "thermal": ["Temperature", "InletTemp", "JunctionTemp"],
    "mechanical": ["Vibration", "Shock"],
    "atmospheric": ["Humidity", "Altitude"]
  },
  "performance": {
    "latency": ["TailLatency", "P99Latency", "MeanLatency"],
    "throughput": ["IOPS", "Bandwidth", "ReadThroughput", "WriteThroughput"]
  },
  "reliability": {
    "errors": ["UncorrectableErrors", "BitErrorRate", "CRCErrors"],
    "wear": ["WriteAmplification", "PECycles", "SpareBlocks"]
  }
}
```

2. Write the extraction prompt enforcing taxonomy alignment:
```
You are extracting structured knowledge from an SSD research paper.
For each causal or correlational finding, output a JSON triple:
{
  "subject": "<taxonomy_term>",
  "predicate": "degrades|improves|mitigatedBy|correlatesWith|causes",
  "object": "<taxonomy_term>",
  "evidence": "<exact sentence from paper>",
  "confidence": 0.0-1.0,
  "section": "<section where evidence appears>"
}
Map all mentions to the closest taxonomy path. Normalize synonyms
(e.g., "tail latency" -> "TailLatency", "p99" -> "P99Latency").
```

3. Process each paper, validate triples, serialize to Turtle:
```turtle
@prefix ssd: <http://example.org/ssd#> .
@prefix prov: <http://www.w3.org/ns/prov#> .

ssd:Temperature ssd:degrades ssd:P99Latency .
ssd:P99Latency ssd:evidence "Sustained temperatures above 45C increased
    p99 write latency by 23% (Section 4.2)" .
ssd:P99Latency ssd:confidence "0.85" .
ssd:P99Latency prov:wasDerivedFrom ssd:Paper_Chen2024 .
```

Output: A `global_knowledge_graph.ttl` with 200+ triples, queryable via SPARQL, each traced to a specific paper and sentence.

---

**Example 2: KG-Guided Telemetry Analysis for SSD Health**

User: "I have 30 days of SMART data for 500 SSDs. Help me build a pipeline that diagnoses drives at risk of failure with evidence-grounded explanations."

Approach:
1. Define the rule base for SMART attribute summarization:
```json
{
  "reallocated_sector_count": {
    "group": "media/wear",
    "ideal_direction": "Low",
    "summarize": ["median", "max", "trend_slope", "change_points"],
    "threshold_alert": 100
  },
  "temperature_celsius": {
    "group": "environment/thermal",
    "ideal_direction": "Monitor",
    "summarize": ["median", "p95", "max", "exposure_above_45C_hours"],
    "threshold_alert": 50
  }
}
```

2. Materialize the Data KG per drive:
```python
def build_data_kg(smart_df, rule_base):
    frames = {}
    for attr, rules in rule_base.items():
        series = smart_df[attr].dropna()
        frame = {
            "type": "AttributeFrame",
            "attribute": attr,
            "group": rules["group"],
            "window": f"{smart_df['date'].min()}/{smart_df['date'].max()}",
            "coverage": f"{series.count()}/{len(smart_df)} days",
            "median": float(series.median()),
            "p95": float(series.quantile(0.95)),
            "max": float(series.max()),
            "trend_slope": compute_trend(series),
            "change_points": detect_change_points(series),
        }
        frames[attr] = frame
    return frames
```

3. Query the Literature KG for relevant evidence, compose the prompt, and generate analysis:
```
LLM Prompt (assembled):
---
TASK: Predictive analysis for drive SN-4821
DATA KG SUMMARY:
- reallocated_sector_count: median=12, max=89, trend_slope=+2.1/day,
  change_point at day 18 (coverage: 30/30 days)
- temperature_celsius: median=41C, p95=48C, exposure_above_45C=142hrs
  (coverage: 28/30 days, 2 days imputed)
LITERATURE EVIDENCE:
- "Reallocated sector growth >1.5/day correlates with 78% failure
   within 60 days (Chen et al., confidence: 0.82)"
- "Sustained thermal exposure above 45C accelerates NAND wear by
   15-20% (Park et al., confidence: 0.88)"
QUESTION: What is the failure risk for this drive and why?
---
```

Output:
```json
{
  "reasoning_mode": "predictive",
  "failure_probability": 0.74,
  "time_horizon_days": 45,
  "explanation": "Drive SN-4821 shows accelerating reallocated sector growth (slope +2.1/day, exceeding the 1.5/day risk threshold from Chen et al.) with a change point at day 18 suggesting onset of media degradation. Concurrent thermal stress (142 hours above 45C) is expected to accelerate NAND wear by 15-20% (Park et al.), compounding the failure risk.",
  "evidence": [
    {"source": "data_kg", "claim": "trend_slope=+2.1/day", "verified": true},
    {"source": "literature_kg", "claim": "threshold 1.5/day", "paper": "Chen2024", "confidence": 0.82},
    {"source": "data_kg", "claim": "exposure_above_45C=142hrs", "verified": true}
  ],
  "grounding_score": 1.0,
  "prescriptive_action": "Schedule proactive data migration within 30 days. Investigate rack-level cooling for thermal mitigation."
}
```

---

**Example 3: What-If Counterfactual Analysis**

User: "What would happen to our fleet's tail latency if we reduced data center inlet temperature by 5C?"

Approach:
1. Retrieve current fleet Data KG summaries (temperature exposure, P99 latency distributions)
2. Query the Literature KG for temperature-latency associations:
   ```sparql
   SELECT ?evidence ?confidence WHERE {
     ssd:Temperature ssd:degrades ssd:P99Latency .
     ssd:P99Latency ssd:evidence ?evidence .
     ssd:P99Latency ssd:confidence ?confidence .
   }
   ```
3. Compose what-if prompt with current state + retrieved directional evidence
4. Validate that the LLM's counterfactual claims align with Literature KG direction

Output:
```
What-if Analysis: Reducing inlet temperature by 5C
- Literature KG evidence indicates temperature reduction from 48C to 43C
  reduces p99 write latency by 12-18% (Park et al., conf: 0.88)
- Fleet Data KG shows 34% of drives currently exceed 45C p95 threshold
- Expected impact: Fleet-wide p99 latency improvement of 8-12%,
  concentrated in thermally stressed rack positions
- Counterfactual Validity: 1.0 (direction consistent with all 3 retrieved
  evidence triples)
```

## Best Practices

- **Do:** Always attach evidence sentences and confidence scores to every extracted KG triple. Provenance is what makes KG-guided reasoning trustworthy.
- **Do:** Use a domain taxonomy to normalize entity names before KG construction. Synonym proliferation (e.g., "tail latency" vs "p99" vs "99th percentile") fragments the graph and reduces retrieval accuracy.
- **Do:** Include explicit data quality nodes (coverage, missingness, imputation flags) in the Data KG. The LLM must know when data is incomplete to avoid overconfident claims.
- **Do:** Compute Faithfulness Precision (supported claims / total claims) on every LLM output to measure grounding quality systematically.
- **Avoid:** Passing raw telemetry time series directly to the LLM. Always summarize through the rule base into compact, typed frames with units and coverage.
- **Avoid:** Merging contradictory Literature KG triples by averaging confidence. Keep both with distinct provenance and let the retrieval layer surface the disagreement to the LLM for reasoned handling.

## Error Handling

- **Taxonomy miss during extraction:** When the LLM proposes an entity not in the taxonomy, route it to a `concept_proposals` queue for human review rather than silently dropping it. Log the evidence sentence for later taxonomy expansion.
- **Low coverage in Data KG:** If a telemetry attribute has <50% coverage in the observation window, annotate the corresponding frame with a `low_confidence` flag and instruct the LLM prompt to caveat any claims involving that attribute.
- **SPARQL retrieval returns empty:** If no Literature KG triples match the detected conditions, fall back to broader taxonomy parent queries (e.g., from `InletTemperature` to `Temperature`). If still empty, the LLM should explicitly state "no literature evidence available" rather than hallucinating support.
- **Grounding score below threshold:** If Faithfulness Precision drops below 0.7, regenerate with a more constrained prompt that explicitly lists the available Data KG frames and instructs the LLM to only make claims verifiable against them.
- **Conflicting evidence triples:** When the Literature KG contains opposing directional claims (e.g., "temperature improves X" vs "temperature degrades X"), surface both to the LLM with their respective confidence scores and source papers, and instruct it to reason about the discrepancy.

## Limitations

- The Literature KG quality depends entirely on the extraction LLM's accuracy. Complex multi-hop causal chains in papers may be missed or oversimplified into single triples. Human review of high-stakes triples is recommended.
- The rule base for Data KG summarization requires initial domain expertise to define attribute groups, ideal directions, and meaningful thresholds. This is a one-time cost but is not fully automated.
- SPARQL retrieval uses keyword-level concept matching against the taxonomy. It does not perform semantic similarity search, so queries using vocabulary outside the taxonomy may miss relevant triples.
- What-if analysis relies on directional associations from literature, not causal models. It cannot quantify counterfactual outcomes with statistical precision -- it provides evidence-supported directional estimates.
- The framework is designed for telemetry-rich domains (storage, hardware, infrastructure). Applying it to domains without structured telemetry streams requires adapting the Data KG materialization layer.
- Fleet-level aggregation assumes drives within a cohort share comparable operating conditions. Highly heterogeneous fleets may need sub-cohort stratification before meaningful fleet-level analysis.

## Reference

**Paper:** [KORAL: Knowledge Graph Guided LLM Reasoning for SSD Operational Analysis](https://arxiv.org/abs/2602.10246v1) (Akewar et al., 2026). Look for: the dual-KG architecture (Section 3), the taxonomy-aligned extraction prompt design (Section 3.1), the rule-based telemetry summarization into typed frames (Section 3.2), and the Faithfulness Precision grounding metric (Section 4).

**Code:** [github.com/Damrl-lab/KORAL](https://github.com/Damrl-lab/KORAL) -- reference implementation with Stage I (literature KG pipeline), Stage II (operational analysis), taxonomy definitions, and evaluation scripts.