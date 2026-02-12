---
name: "towards-agentic-intelligence-for"
description: |
  Design and implement agentic AI systems for scientific discovery, especially materials science. Applies the pipeline-centric framework from "Towards Agentic Intelligence for Materials Science" (arXiv:2602.00169v2) to build LLM agents that plan, act, and learn across a full discovery loop — integrating literature mining, simulation tools (DFT, MD), robotic lab control, and iterative hypothesis refinement.
  Trigger phrases:
  - "Build an agentic pipeline for materials discovery"
  - "Design an LLM agent that interfaces with DFT simulations"
  - "Create a closed-loop scientific discovery agent"
  - "Implement a materials science agent with tool use and memory"
  - "Set up an autonomous research agent with credit assignment"
  - "Build a pipeline from literature mining to experimental validation"
---

# Agentic Intelligence for Scientific Discovery

This skill enables Claude to architect and implement **agentic AI systems for scientific discovery** using the pipeline-centric framework from Zhang et al. (2026). Instead of building isolated ML models for single prediction tasks, you design end-to-end agent systems that span corpus curation, domain-adapted reasoning, tool orchestration (DFT, robotic labs, databases), and closed-loop experimental feedback — all aligned toward tangible discovery outcomes through credit assignment rather than proxy benchmarks.

## When to Use

- When the user wants to build an LLM-based agent that orchestrates scientific workflows (literature search, hypothesis generation, simulation, experiment, analysis)
- When designing a system that connects language models to external computational tools like DFT calculators, molecular dynamics engines, or materials databases (Materials Project, ICSD)
- When the user asks to implement a closed-loop discovery pipeline where experimental or simulation results feed back into the agent's planning
- When building a multi-step research automation system with persistent memory across experiments
- When the user needs to mine scientific literature at scale to extract structured property datasets (e.g., thermoelectric coefficients, elastic constants, crystal structures)
- When implementing constraint-aware material candidate generation that respects synthesis feasibility, safety, and cost
- When designing evaluation frameworks for scientific agents that go beyond accuracy metrics to measure real discovery value

## Key Technique: Pipeline-Centric Agentic Discovery

The core insight is treating scientific discovery as an **end-to-end optimizable pipeline** rather than a collection of independent models. The pipeline has five coupled stages: (1) corpus curation, (2) pretraining/domain adaptation, (3) instruction tuning, (4) goal-conditioned agent deployment, and (5) iterative feedback from real outcomes. Critically, downstream discovery success propagates credit backward through all upstream stages — a data curation decision that improves synthesis yield six months later is a measurable, attributable win. This contrasts with the dominant approach of optimizing each stage against proxy benchmarks in isolation.

The framework distinguishes **passive/reactive systems** (fine-tuned models answering single queries like "predict the band gap of X") from **agentic systems** that exhibit three capabilities: *autonomy* (generating hypotheses and planning multi-step experiments without constant human input), *memory* (maintaining context across an entire research campaign, connecting results from experiment N to the design of experiment N+12), and *tool use* (calling DFT calculators, querying databases, controlling robotic synthesis platforms, executing code). An agentic system pursues long-horizon goals — "find a thermoelectric material with ZT > 2 that is synthesizable below 800K" — not isolated predictions.

Safety and constraint-awareness are first-class design requirements, not afterthoughts. Agents must check for hazardous chemistries, verify synthesis feasibility before proposing experiments, and be robust to data poisoning. The pipeline treats verification (computational stability checks, phase diagram consistency) as mandatory gates between hypothesis and experiment.

## Step-by-Step Workflow

1. **Define the discovery objective as a goal specification.** Frame the target as a concrete, measurable outcome — not "predict properties" but "identify candidate compositions with target elastic modulus > X GPa, synthesizable via arc melting, with no toxic elements." This goal drives every downstream design choice.

2. **Design the corpus curation layer.** Identify and prioritize data sources: scientific literature (Elsevier, Springer, arXiv), structured databases (Materials Project, AFLOW, ICSD, OQMD), experimental logs, and safety datasheets. Implement extraction pipelines that produce structured records linking composition, process parameters, structure descriptors, and measured properties.

3. **Implement domain-adapted retrieval (RAG).** Build a retrieval-augmented generation layer over the curated corpus. Index documents with embeddings tuned on materials vocabulary. Design retrieval queries that pull relevant prior work, known phase diagrams, and synthesis protocols for any candidate the agent considers.

4. **Define the agent's tool interface.** Create typed function schemas for each external tool the agent can invoke:
   - `run_dft(structure, functional, k_points) -> energy, forces, band_structure`
   - `query_materials_db(formula, property) -> records`
   - `submit_synthesis(composition, method, parameters) -> experiment_id`
   - `retrieve_characterization(experiment_id) -> xrd_pattern, sem_images, properties`
   - `check_safety(composition, conditions) -> hazard_report`

5. **Build the planning module with chain-of-thought reasoning.** Implement a planner that decomposes the high-level goal into a directed acyclic graph of sub-tasks: literature review, candidate generation, computational screening, stability verification, synthesis planning, experimental validation. Each node specifies inputs, outputs, success criteria, and fallback actions.

6. **Implement persistent memory.** Design a structured memory store (experiment log, hypothesis tracker, dead-end registry) that the agent reads at the start of each planning cycle. Store not just results but *why* each experiment was run and what was learned from failures. Use this to prevent redundant experiments and enable long-horizon reasoning.

7. **Add constraint-aware filtering gates.** Before any candidate reaches simulation or experiment, run it through feasibility checks: thermodynamic stability (convex hull distance), synthesis accessibility (known routes for similar compositions), safety screening (toxicity, flammability, regulatory status), and cost estimation.

8. **Implement the feedback/credit-assignment loop.** After each experimental or simulation outcome, propagate learning backward: update the candidate ranker, refine retrieval query templates, flag data sources that led to false positives, and adjust the planner's heuristics. Log influence traces so you can answer "which upstream decision led to this outcome?"

9. **Build evaluation around discovery outcomes.** Define metrics that correlate with real discovery value: hit rate (fraction of synthesized candidates meeting the goal), cycle time (iterations to reach target), novelty (distance from known materials in feature space), and safety compliance rate. Avoid over-indexing on intermediate metrics like prediction RMSE.

10. **Deploy with human-in-the-loop checkpoints.** Insert approval gates before irreversible actions (synthesis, expensive computation). Provide the human reviewer with the agent's reasoning chain, confidence estimates, and risk assessment. Progressively expand agent autonomy as trust is established.

## Concrete Examples

**Example 1: Building a thermoelectric materials discovery agent**

```
User: Build me an agent that discovers new thermoelectric materials with
ZT > 1.5 by mining literature and screening with DFT.

Approach:
1. Define goal spec: ZT > 1.5 at 300-800K, earth-abundant elements only,
   no Pb/Cd/Tl, synthesizable via ball milling or spark plasma sintering.

2. Build literature extraction pipeline:
   - Scrape thermoelectric papers from corpus
   - Extract (composition, ZT, temperature, synthesis_method) tuples
   - Store in structured DB with provenance links

3. Implement agent with tool interface:

   tools = {
       "search_literature": {
           "params": {"query": "str", "filters": {"year_min": "int"}},
           "returns": "List[Paper]"
       },
       "run_dft_relaxation": {
           "params": {"cif_structure": "str", "functional": "PBE|HSE06"},
           "returns": {"energy_per_atom": "float", "converged": "bool"}
       },
       "compute_transport": {
           "params": {"band_structure": "BandStructure", "temperature": "float"},
           "returns": {"seebeck": "float", "conductivity": "float", "zt": "float"}
       },
       "check_stability": {
           "params": {"composition": "str"},
           "returns": {"e_above_hull": "float", "decomposition": "List[str]"}
       }
   }

4. Agent planning loop (pseudocode):

   memory = ExperimentLog()
   while not goal_met:
       candidates = agent.generate_hypotheses(memory, literature_rag)
       candidates = filter_constraints(candidates)  # stability, safety, cost
       for c in rank(candidates):
           dft_result = run_dft_relaxation(c.structure)
           if dft_result.e_above_hull < 0.05:  # thermodynamically accessible
               transport = compute_transport(dft_result.band_structure, T=600)
               memory.log(c, dft_result, transport)
               if transport.zt > 1.5:
                   flag_for_synthesis(c)
       agent.update_strategy(memory)  # credit assignment

Output: Ranked list of candidates with DFT-validated ZT predictions,
stability assessments, suggested synthesis routes, and full reasoning traces.
```

**Example 2: Closed-loop literature mining to structured dataset**

```
User: Extract all reported elastic constants from the literature on
high-entropy alloys and build a machine-readable dataset.

Approach:
1. Define extraction schema:
   {
     "composition": "str (e.g., CoCrFeMnNi)",
     "crystal_structure": "str (FCC/BCC/HCP)",
     "elastic_constants": {"C11": "float", "C12": "float", "C44": "float"},
     "bulk_modulus": "float (GPa)",
     "measurement_method": "str (DFT/ultrasonic/nanoindentation)",
     "temperature": "float (K)",
     "source_doi": "str"
   }

2. Build extraction agent with tools:
   - search_papers(query, date_range) -> paper_ids
   - extract_tables(paper_id) -> List[Table]
   - extract_text_values(paper_id, schema) -> List[Record]
   - validate_units(record) -> record_with_si_units
   - cross_reference_db(composition) -> known_values

3. Agent workflow:
   a. Search for "high-entropy alloy elastic" papers (expect ~2000 hits)
   b. For each paper, extract tables and inline values matching schema
   c. Normalize units to SI (GPa for moduli)
   d. Cross-reference against Materials Project for consistency checks
   e. Flag outliers (>3 sigma from DFT predictions) for human review
   f. Track extraction confidence per record

4. Credit assignment: After human review of flagged records, update
   extraction prompts and confidence thresholds for next batch.

Output: CSV/JSON dataset with ~5000 records, provenance DOIs,
confidence scores, and a quality report showing extraction precision/recall
estimated from the human-reviewed sample.
```

**Example 3: Implementing a safety-aware synthesis planning agent**

```
User: Build an agent that proposes synthesis routes for novel perovskite
compositions and checks them for safety before sending to the robotic lab.

Approach:
1. Agent receives target: CsPb0.5Sn0.5I3 perovskite thin film

2. Planning decomposition:
   - Retrieve known synthesis routes for similar perovskites
   - Adapt route for target composition
   - Run safety checks on all precursors and intermediates
   - Verify phase stability via DFT
   - Generate robotic lab protocol

3. Safety gate implementation:

   def safety_check(protocol):
       for chemical in protocol.precursors + protocol.solvents:
           hazard = lookup_ghs(chemical)
           if hazard.contains("acute_toxicity_cat1"):
               return BLOCK, f"{chemical}: acute toxicity"
           if hazard.contains("flammable") and protocol.temp > chemical.flash_point:
               return BLOCK, f"{chemical} above flash point at {protocol.temp}C"
       if any(is_lead_compound(c) for c in protocol.precursors):
           return WARN, "Lead-containing: require fume hood + PPE protocol"
       return PASS, "No critical hazards identified"

4. Agent checks safety BEFORE generating the lab protocol.
   If blocked, it proposes lead-free alternatives (Sn, Ge substitution)
   and re-plans. If warned, it adds required safety measures to the protocol.

Output: Executable robotic lab protocol with safety annotations,
alternative compositions if original is hazardous, and full reasoning trace.
```

## Best Practices

- **Do:** Frame discovery goals as concrete, measurable specifications with explicit constraints (target property ranges, excluded elements, synthesis method restrictions). Vague goals produce vague agents.
- **Do:** Log every agent decision with its reasoning chain and the evidence it consulted. This enables credit assignment and makes failures debuggable.
- **Do:** Treat safety checks as hard gates, not soft suggestions. Block execution paths that involve unscreened hazards rather than appending warnings.
- **Do:** Build memory that captures *negative results* — which candidates failed and why. This is often more valuable than positive results for pruning the search space.
- **Avoid:** Optimizing agent components against proxy benchmarks (prediction accuracy, perplexity) without verifying correlation to actual discovery outcomes.
- **Avoid:** Deploying agents with unrestricted tool access. Every tool call should be typed, validated, and logged. Sandbox simulation calls; require human approval for physical experiments.

## Error Handling

- **DFT convergence failure:** Log the failed structure, reduce k-point density or switch functional, retry. After 3 failures, flag the candidate as computationally intractable and deprioritize.
- **Literature extraction hallucination:** Cross-validate extracted values against database entries or other papers reporting the same material. Flag discrepancies >20% for human review. Never propagate unvalidated extracted data into the screening pipeline.
- **Tool timeout or unavailability:** Implement circuit breakers — if a tool fails N times, pause that branch of the plan and continue with other candidates. Queue retries with exponential backoff.
- **Contradictory data sources:** When literature and database values conflict, log both with provenance, weight by measurement method quality (single-crystal > polycrystal > DFT-predicted), and flag for human arbitration.
- **Agent proposes infeasible synthesis:** Catch via constraint gates. If the agent repeatedly proposes blocked routes, inspect its retrieval context — it may be drawing from outdated or irrelevant literature. Update RAG filters.

## Limitations

- This framework assumes access to computational infrastructure (DFT codes, HPC clusters) and optionally robotic labs. Without these backends, the agent reduces to a literature mining and hypothesis generation system — still useful, but not closed-loop.
- Credit assignment across the full pipeline is theoretically motivated but practically difficult to implement rigorously. In most real deployments, feedback loops will be approximate (expert judgment, heuristic scoring) rather than formal influence functions.
- The agent cannot replace domain expertise for interpreting ambiguous experimental results (e.g., mixed-phase XRD patterns, unexpected defect formation). Human-in-the-loop checkpoints remain essential.
- Literature extraction quality depends heavily on access to full-text papers. Open-access coverage varies by subfield; paywalled content creates systematic gaps in the agent's knowledge.
- The framework is designed around materials science but the agentic architecture generalizes to other scientific domains (drug discovery, catalysis, energy storage) with domain-specific tool swaps.

## Reference

**Paper:** Zhang, H., Li, Y., Huang, W., Hou, Z., & Song, Y. (2026). "Towards Agentic Intelligence for Materials Science." arXiv:2602.00169v2.
**What to look for:** Section 3 for the pipeline-centric architecture and credit assignment framework; Section 5 for the passive-vs-agentic contrast and the autonomy/memory/tool-use capability model; Section 4 for concrete tool integration patterns with DFT and robotic labs; Section 6 for the safety-aware agent design principles.