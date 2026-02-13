---
name: "multi-agent-causal-reasoning-system"
description: "Build multi-agent systems that discover causal rules from event sequences using specialized agents (causal discovery, contextual information, orchestrator). Use when asked to: 'find causal patterns in event logs', 'build a multi-agent diagnostic system', 'automate rule discovery from sequences', 'generate Boolean fault rules from data', 'create an agent pipeline for causal reasoning', 'detect error patterns in time-series events'."
---

# Multi-Agent Causal Reasoning for Automated Rule Discovery

This skill teaches Claude to implement the CAREP (Causal Automated Reasoning for Error Patterns) architecture: a three-agent system that discovers Boolean causal rules from high-dimensional event sequences. Instead of relying on a single LLM to guess rules from raw data, CAREP decomposes the problem into causal discovery, contextual enrichment, and orchestrated synthesis — achieving 3x higher precision than standalone LLM approaches on datasets with 29,000+ event types and 474 target patterns.

## When to Use

- When the user needs to discover which combinations of events cause specific outcomes (e.g., fault diagnosis, alert correlation, root cause analysis)
- When building a multi-agent system where each agent handles a distinct analytical capability (statistical, contextual, synthesis)
- When the user asks to generate Boolean rules (AND/OR/NOT combinations) from sequential event data
- When automating rule creation that domain experts currently do by hand — error patterns, alert rules, incident classification
- When the user has labeled event sequences and wants interpretable causal explanations, not just predictions
- When working with high-dimensional discrete event data where brute-force correlation is infeasible

## Key Technique

CAREP replaces manual rule engineering with a three-agent pipeline. The **Causal Discovery Agent** computes information-theoretic causal indicators between events and target labels using conditional mutual information: `I(Y, X | Z) = H(Y|Z) - H(Y|Z,X)`. If this value exceeds zero, event X is a statistical cause of outcome Y. The agent then computes the Average Causal Effect (ACE) to quantify direction and magnitude: positive ACE means an event excites the outcome (AND/OR candidate), negative ACE means it inhibits it (NOT candidate). An adaptive logistic threshold `τ(m) = (τ_max - τ_min) / (1 + exp(α(log m - log m₀))) + τ_min` adjusts sensitivity based on how many sequences carry each label, preventing rare patterns from being drowned out.

The **Contextual Information Agent** uses retrieval-augmented generation (RAG) to inject domain knowledge — event descriptions, known rules, metadata, and sample sequences — into the reasoning context. This grounds the statistical findings in real-world meaning. The **Orchestrator Agent** receives structured JSON from both agents containing frequency scores, ACE statistics, point-wise mutual information (PMI) between event pairs, and retrieved context. It synthesizes candidate Boolean rules with chain-of-thought reasoning traces, outputting multiple hypotheses ranked by confidence (HIGH/MEDIUM/LOW).

The critical insight is the separation of concerns: statistical causal discovery handles the combinatorial explosion that LLMs cannot (29,100 event types), while the LLM orchestrator handles the synthesis and explanation that statistical methods cannot. Neither agent alone matches the combined system.

## Step-by-Step Workflow

1. **Define the event vocabulary and target labels.** Enumerate all discrete event types (your "DTCs") and the outcomes you want rules for (your "error patterns"). Structure input data as labeled sequences: each sequence is a list of `(timestamp, event_type)` pairs with one or more binary outcome labels.

2. **Train or configure sequence models for entropy estimation.** Implement two prediction models — one for next-event prediction (`Tfx`) and one for label prediction (`Tfy`). These can be Transformer encoders or simpler models depending on scale. They provide the probability distributions needed for conditional entropy computation. Use a context window of ~15 events for stable estimates.

3. **Implement the Causal Discovery Agent.** For each target label `Y_j` and candidate event `X_i`, compute conditional mutual information via Monte Carlo sampling: generate N plausible contexts from `Tfx`, compute `H(Y_j|Z)` and `H(Y_j|Z, X_i)`, take the difference. Filter candidates using the adaptive threshold `θ_j = μ_j + γσ_j` where `γ` controls sensitivity. Compute ACE for surviving candidates to determine excitatory (>0) vs. inhibitory (<0) relationships.

4. **Compute co-occurrence structure with PMI.** For all pairs of candidate events that passed the causal filter, calculate point-wise mutual information: `pmi(x_i, x_j) = log[p(x_i, x_j) / (p(x_i) * p(x_j))]`. High PMI between two excitatory causes suggests they should be grouped with OR (alternative causes); low PMI suggests AND (co-required causes).

5. **Build the Contextual Information Agent.** Create an embedding index over event descriptions and known rules using a text embedding model. Implement RAG retrieval that, given a target label, returns: (a) descriptions of candidate events, (b) similar known rules as reference patterns, (c) metadata like system identifiers or component names, (d) 5-10 sample sequences carrying the target label.

6. **Structure the orchestrator input as JSON.** Combine outputs from both agents into a single structured payload per target label:
   ```json
   {
     "target_label": "EP_042",
     "causal_candidates": [
       {"event": "DTC_1091", "ace": 0.72, "frequency": 0.85, "std": 0.04, "description": "..."},
       {"event": "DTC_3402", "ace": -0.31, "frequency": 0.60, "std": 0.08, "description": "..."}
     ],
     "pmi_pairs": [
       {"events": ["DTC_1091", "DTC_2205"], "pmi": 3.2},
       {"events": ["DTC_1091", "DTC_4410"], "pmi": 0.1}
     ],
     "context": {"known_similar_rules": [...], "sample_sequences": [...], "metadata": {...}}
   }
   ```

7. **Implement the Orchestrator Agent with chain-of-thought prompting.** Prompt the LLM to: (a) analyze excitatory vs. inhibitory candidates from ACE signs, (b) determine AND vs. OR groupings from PMI values, (c) construct Boolean expressions in the form `y = x1 & x5 & (x12 | x3) & !x10`, (d) generate 3-5 candidate rules ranked by confidence, (e) produce a natural-language reasoning trace for each rule explaining why each event was included.

8. **Validate candidate rules with truth-table evaluation.** For each generated Boolean rule, evaluate it symbolically (e.g., using SymPy) against the test sequences. Compute precision, recall, and F1 by comparing the rule's truth table against ground-truth labels. Use both structural metrics (do the rules contain the right events?) and semantic metrics (do the rules produce the right outputs?).

9. **Iterate with feedback.** If validation reveals low recall, decrease the threshold `γ` to admit more candidate causes. If precision is low, increase `γ` or add inhibitory (NOT) terms. Feed validation results back to the orchestrator for rule refinement.

10. **Output rules with interpretable traces.** Deliver each rule alongside its reasoning chain: which events were identified as causal and why, their ACE magnitudes, co-occurrence structure, and the contextual evidence that supports the rule. This transparency is essential for domain expert review.

## Concrete Examples

**Example 1: Server Alert Correlation**

User: "I have 50,000 alert sequences from our monitoring system with 800 unique alert types. Each sequence is labeled with one of 30 incident categories. Help me build a system that discovers which alert combinations indicate each incident type."

Approach:
1. Parse alert logs into `(timestamp, alert_type)` sequences with incident labels
2. Train a small Transformer on next-alert prediction to model sequence distributions
3. Implement the Causal Discovery Agent: for each incident category, compute CMI for all 800 alert types, apply adaptive threshold, compute ACE
4. Build a RAG index over alert descriptions from the runbook
5. For incident category "database_failover": the causal agent finds `DISK_LATENCY_HIGH` (ACE=0.65), `REPLICATION_LAG` (ACE=0.58), `CONNECTION_POOL_EXHAUSTED` (ACE=0.41), `HEALTH_CHECK_OK` (ACE=-0.30)
6. PMI between `DISK_LATENCY_HIGH` and `REPLICATION_LAG` is 4.1 (high co-occurrence → OR group)
7. Orchestrator synthesizes: `database_failover = (DISK_LATENCY_HIGH | REPLICATION_LAG) & CONNECTION_POOL_EXHAUSTED & !HEALTH_CHECK_OK`

Output:
```
Rule: database_failover = (DISK_LATENCY_HIGH | REPLICATION_LAG) & CONNECTION_POOL_EXHAUSTED & !HEALTH_CHECK_OK
Confidence: HIGH

Reasoning:
- DISK_LATENCY_HIGH and REPLICATION_LAG are alternative root causes (PMI=4.1, both excitatory)
- CONNECTION_POOL_EXHAUSTED co-occurs in 85% of incidents (ACE=0.41, required condition)
- HEALTH_CHECK_OK presence inhibits this pattern (ACE=-0.30, negation term)
- Validated on 2,400 held-out sequences: Precision=0.82, Recall=0.76, F1=0.79
```

**Example 2: Manufacturing Defect Classification**

User: "We have sensor event logs from an assembly line. Events are things like TEMP_SPIKE, VIBRATION_ANOMALY, PRESSURE_DROP, etc. (200 event types). We label sequences with defect types. Can you automate our defect classification rules?"

Approach:
1. Structure sensor events as sequences with defect-type labels
2. Causal Discovery Agent identifies that for defect type "weld_crack": `TEMP_SPIKE` (ACE=0.81), `VIBRATION_ANOMALY` (ACE=0.55), `COOLING_RATE_FAST` (ACE=0.38) are excitatory; `PREHEAT_COMPLETE` (ACE=-0.45) is inhibitory
3. PMI analysis: `TEMP_SPIKE` and `COOLING_RATE_FAST` have PMI=0.3 (independent → AND); `VIBRATION_ANOMALY` and `TEMP_SPIKE` have PMI=3.8 (correlated → OR group)
4. Contextual agent retrieves: "TEMP_SPIKE indicates localized overheating at weld joint, often paired with rapid cooling defects"
5. Orchestrator produces candidate rules with explanations

Output:
```
Rule 1 (HIGH): weld_crack = (TEMP_SPIKE | VIBRATION_ANOMALY) & COOLING_RATE_FAST & !PREHEAT_COMPLETE
Rule 2 (MEDIUM): weld_crack = TEMP_SPIKE & COOLING_RATE_FAST & !PREHEAT_COMPLETE
Rule 3 (LOW): weld_crack = TEMP_SPIKE & (COOLING_RATE_FAST | VIBRATION_ANOMALY)
```

**Example 3: Multi-Agent System Architecture in Python**

User: "Show me how to structure the three-agent system in code."

```python
from dataclasses import dataclass

@dataclass
class CausalCandidate:
    event: str
    ace: float          # Average Causal Effect: >0 excitatory, <0 inhibitory
    frequency: float    # How often this relationship appears
    std: float          # Standard deviation across folds

@dataclass
class PMIPair:
    events: tuple[str, str]
    pmi: float          # High PMI → OR group, Low PMI → AND group

class CausalDiscoveryAgent:
    """Computes conditional mutual information and ACE for event-label pairs."""

    def __init__(self, event_model, label_model, gamma=1.0):
        self.event_model = event_model  # Tfx: next-event predictor
        self.label_model = label_model  # Tfy: label predictor
        self.gamma = gamma

    def compute_cmi(self, sequences, target_label, candidate_event, n_samples=100):
        """I(Y, X | Z) = H(Y|Z) - H(Y|Z, X) via Monte Carlo."""
        h_y_given_z = self._conditional_entropy(sequences, target_label)
        h_y_given_zx = self._conditional_entropy(sequences, target_label, given=candidate_event)
        return h_y_given_z - h_y_given_zx

    def compute_ace(self, sequences, target_label, candidate_event):
        """ACE = E_Z[P(Y|X,Z) - P(Y|Z)]. Positive=excitatory, negative=inhibitory."""
        # Implementation uses label_model predictions with/without candidate event
        ...

    def discover(self, sequences, target_label) -> list[CausalCandidate]:
        """Run full causal discovery for one target label."""
        candidates = []
        threshold = self._adaptive_threshold(target_label, sequences)
        for event in self.event_vocabulary:
            cmi = self.compute_cmi(sequences, target_label, event)
            if cmi > threshold:
                ace = self.compute_ace(sequences, target_label, event)
                candidates.append(CausalCandidate(event=event, ace=ace, ...))
        return candidates

class ContextualInformationAgent:
    """RAG-based agent that retrieves domain knowledge for candidate events."""

    def __init__(self, embedding_model, knowledge_base):
        self.embedding_model = embedding_model
        self.knowledge_base = knowledge_base

    def retrieve_context(self, target_label, candidates, sample_sequences):
        return {
            "descriptions": self._get_descriptions(candidates),
            "similar_known_rules": self._retrieve_similar_rules(target_label),
            "sample_sequences": sample_sequences[:10],
            "metadata": self._get_metadata(target_label),
        }

class OrchestratorAgent:
    """Synthesizes Boolean rules from causal and contextual inputs."""

    def __init__(self, llm_client):
        self.llm_client = llm_client

    def synthesize_rules(self, candidates, pmi_pairs, context) -> list[dict]:
        prompt = self._build_prompt(candidates, pmi_pairs, context)
        response = self.llm_client.generate(prompt)
        return self._parse_rules(response)  # Returns rules + reasoning traces

# Pipeline orchestration
def run_carep_pipeline(sequences, target_label):
    causal_agent = CausalDiscoveryAgent(event_model, label_model)
    context_agent = ContextualInformationAgent(embedder, knowledge_base)
    orchestrator = OrchestratorAgent(llm_client)

    candidates = causal_agent.discover(sequences, target_label)
    pmi_pairs = compute_pmi_pairs(candidates, sequences)
    context = context_agent.retrieve_context(target_label, candidates, sequences)
    rules = orchestrator.synthesize_rules(candidates, pmi_pairs, context)
    return rules
```

## Best Practices

- **Do:** Separate statistical causal discovery from LLM synthesis. The causal agent handles the combinatorial search over thousands of event types that LLMs hallucinate on; the LLM handles the synthesis and explanation that statistics cannot.
- **Do:** Use ACE sign to distinguish AND/OR/NOT. Positive ACE events are excitatory (AND or OR candidates), negative ACE events are inhibitory (NOT candidates). Use PMI between excitatory pairs to decide AND vs OR: high PMI means the events tend to co-occur as alternatives (OR), low PMI means they are independently required (AND).
- **Do:** Generate multiple candidate rules per target (3-5) with confidence levels. Domain experts review the top candidates rather than trusting a single output.
- **Do:** Include reasoning traces with every rule. Each trace should cite the ACE magnitude, frequency, and contextual evidence for every included event.
- **Avoid:** Sending raw event sequences directly to an LLM for rule discovery. With thousands of event types, LLMs achieve ~0.25 precision vs. ~0.78 with the CAREP pipeline. The statistical pre-filtering is essential.
- **Avoid:** Using a fixed threshold across all target labels. Labels with few supporting sequences need lower thresholds (more permissive), while common labels need higher thresholds. The adaptive logistic decay function handles this automatically.

## Error Handling

- **Sparse target labels (few sequences):** The adaptive threshold `τ(m)` approaches `τ_max` for rare labels, which can filter out all candidates. Lower `γ` or set a minimum candidate count (e.g., at least 3 candidates pass through).
- **No causal candidates found:** If CMI is near zero for all events, the target label may not have deterministic event causes. Fall back to the contextual agent alone with explicit low-confidence markers.
- **Contradictory PMI signals:** If two events have high PMI but different ACE signs (one excitatory, one inhibitory), they likely represent a confounded relationship. Flag these for manual review rather than forcing them into a rule.
- **Orchestrator hallucination:** The LLM may invent event names not in the candidate list. Always validate that every event in a generated rule exists in the causal discovery output. Post-process rules with a strict allowlist check.
- **Symbolic validation failure:** If SymPy cannot parse a generated Boolean expression, retry with a stricter output format prompt or use regex-based rule parsing as a fallback.

## Limitations

- Requires labeled sequential data — unlabeled event logs need a separate labeling step before this pipeline applies.
- The causal discovery agent needs trained sequence models (Transformers or equivalent), which requires substantial training data (~100+ sequences per target label).
- Boolean rules are inherently limited: they cannot express temporal ordering constraints (event A must occur before event B), continuous thresholds, or probabilistic relationships.
- The system discovers correlational-causal rules from observational data, not interventional causality. Confounders shared across event types can produce spurious rules.
- Scales to ~30,000 event types with the information-theoretic approach, but computational cost grows linearly with vocabulary size times the number of target labels.

## Reference

**Paper:** "Multi-Agent Causal Reasoning System for Error Pattern Rule Automation in Vehicles" — Math, Lorenz, Oelsner, Lienhart (2026). [arXiv:2602.01155v2](https://arxiv.org/abs/2602.01155v2)

Look for: Section 3 (CAREP architecture with the three-agent decomposition), Section 3.2 (one-shot causal discovery via conditional mutual information), Section 3.3 (adaptive thresholding and ACE computation), Section 4 (structural and semantic evaluation protocols), and Figure 4 (orchestrator JSON input structure).