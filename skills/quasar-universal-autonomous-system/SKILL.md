---
name: "quasar-universal-autonomous-system"
description: "Build autonomous multi-scale scientific simulation pipelines using the QUASAR architecture: a Strategist-Operator-Evaluator agent trio with adaptive planning, hierarchical knowledge retrieval, and context-efficient memory. Triggers: 'build an autonomous simulation pipeline', 'orchestrate DFT and MD workflows', 'QUASAR-style agent system', 'multi-scale atomistic workflow', 'autonomous computational chemistry', 'scientific simulation agent framework'"
---

This skill enables Claude to design and implement autonomous multi-scale simulation orchestration systems following the QUASAR architecture (arXiv:2602.00185). QUASAR introduces a three-agent paradigm -- Strategist, Operator, Evaluator -- that replaces rigid tool-calling pipelines with reasoning-driven orchestration. The key innovation is that the LLM performs generalized scientific reasoning rather than executing predefined function chains, using double-pass adaptive planning, a four-level hierarchical knowledge retrieval protocol, and context-efficient memory compression to autonomously navigate complex computational workflows spanning density functional theory, molecular dynamics, Monte Carlo simulations, and machine learning potentials.

## When to Use

- When the user asks to build an autonomous agent system that orchestrates scientific simulations (DFT, MD, Monte Carlo, ML potentials)
- When designing a multi-step computational pipeline where tasks must chain across scales (e.g., quantum calculation parameterizes a classical force field for large-scale dynamics)
- When the user wants a Strategist-Operator-Evaluator architecture for any domain requiring planning, execution, and self-evaluation loops
- When building a system that needs hierarchical knowledge retrieval -- progressing from internal confidence to RAG to logical inference to external search
- When implementing checkpoint-based fault tolerance and interruption-aware resumption for long-running computational workflows
- When creating an auto-improvement loop where failed results trigger diagnostic replanning rather than hard failure

## Key Technique

QUASAR's architecture decomposes autonomous simulation into three coordinated agents. The **Strategist** performs double-pass planning: it generates an initial task decomposition, then a second pass explicitly checks for missing domain-specific requirements (e.g., equilibration steps, convergence criteria) before execution begins. Two user-tunable parameters -- **granularity** (checkpoint frequency) and **accuracy** (eco for rapid screening, pro for rigorous results) -- govern the plan's depth and computational cost tradeoff.

The **Operator** executes tasks using a four-level hierarchical knowledge-access protocol. Level 1: proceed autonomously when confident in simulation physics and syntax. Level 2: perform semantic similarity matching via RAG against documentation and code examples. Level 3: systematically explore provided example repositories, interpret filenames, and parse README files to infer appropriate simulation patterns -- this addresses the critical problem that simulation engines use highly specialized input syntaxes with sparse semantic context unsuitable for embedding-based retrieval alone. Level 4: fall back to external web search only as a last resort. This hierarchy dramatically reduces hallucinated configurations.

The **Evaluator** closes the loop by verifying task outputs, compressing completed task context (distilling valuable actions while discarding noise and failed attempts), and triggering auto-improvement cycles when results fall short. Upon workflow completion, both task-level and run-level summaries are persisted to the filesystem in Markdown, enabling incremental improvement across runs. A silent-failure detection mechanism checks intermediate outputs at configurable intervals, terminating stalled simulations and reconstructing input parameters rather than waiting for timeout.

## Step-by-Step Workflow

1. **Define the simulation objective and constraints.** Capture the user's goal (e.g., "screen photocatalysts for methyl orange degradation"), the target methods (DFT, MD, Monte Carlo, ML potentials), and accuracy/granularity preferences (eco vs. pro mode).

2. **Implement the Strategist with double-pass planning.** Build a planning module that generates an initial task graph from the objective, then runs a second validation pass that checks for missing domain requirements. Represent the plan as a DAG of tasks with explicit dependencies, convergence criteria, and checkpoint markers.

3. **Build the four-level hierarchical knowledge retrieval stack.** Implement retrieval in escalating order: (a) internal LLM confidence assessment, (b) semantic RAG over indexed documentation and input-file examples, (c) filesystem exploration of example repositories with filename interpretation and README parsing, (d) external web search fallback. Gate each level with a confidence threshold before escalating.

4. **Implement the Operator execution engine.** For each task in the plan, the Operator generates simulation input files (e.g., Quantum ESPRESSO input, LAMMPS scripts, RASPA input), invokes the appropriate computational engine, and monitors execution. Use ASE and pymatgen as universal structure-manipulation interfaces.

5. **Add persistent checkpointing after every task.** Serialize the full agent state -- conversation history, completed steps, intermediate results -- after each execution step. On restart, auto-inject an interruption-awareness prompt that guides the Operator to resume from the most recent intermediate state (e.g., a partially relaxed structure) rather than reinitializing.

6. **Implement the Evaluator with context compression.** After each task completes, the Evaluator verifies outputs against expected criteria (convergence, physical plausibility), then compresses the task context into a structured summary. Discard failed attempts and noise; retain only actionable outcomes and parameter choices.

7. **Build the silent-failure detection mechanism.** Configure a periodic check-in (default: 15-minute intervals) where the Operator evaluates intermediate outputs. If a simulation is not converging (e.g., DFT energy oscillating without decreasing), terminate the run and reconstruct input parameters with adjusted settings.

8. **Wire the auto-improvement loop.** When the Evaluator determines results are suboptimal, switch the Strategist from generative to diagnostic mode. It critically analyzes previous outputs, identifies the failure mode (e.g., wrong functional choice), and proposes targeted improvements -- not a full replan, but a surgical correction to escape local optima.

9. **Persist run-level summaries in Markdown.** On workflow completion, write a structured summary containing: objective, methods used, key parameters, results, and lessons learned. Store alongside the checkpoint data so subsequent runs can load prior context for incremental improvement.

10. **Expose granularity and accuracy as user-configurable parameters.** Granularity controls task decomposition depth and checkpoint frequency. Accuracy selects between eco mode (fast screening with approximate methods) and pro mode (publication-quality calculations with hybrid functionals and tight convergence).

## Concrete Examples

**Example 1: Photocatalyst Screening Pipeline**

User: "Build an autonomous pipeline that screens La-doped ATaO3 perovskites (A = Li, Na, K) for photocatalytic degradation of methyl orange under UV light."

Approach:
1. Strategist generates plan: (a) build doped crystal structures, (b) relax geometries with DFT, (c) compute band structures and align band edges to redox potentials, (d) assess doping impact on optical gap, (e) evaluate defect formation energies, (f) rank candidates.
2. Second-pass review catches missing steps: adds k-point convergence test and Hubbard U parameter selection for Ta d-orbitals.
3. Operator executes each task using Quantum ESPRESSO via ASE interface. Uses Level 1 knowledge for structure generation, Level 2 RAG for pseudopotential selection, Level 3 example-file parsing for DFT+U input syntax.
4. Evaluator compresses each completed DFT run into: structure, total energy, band gap, band edge positions.
5. Auto-improvement triggers when LiTaO3 band gap looks anomalous -- Strategist switches from PBE+U to HSE hybrid functional for verification.

Output:
```
Screening Summary (quasar-run-2026-02-13)
=========================================
Candidate Rankings for MeO Photodegradation (UV):
1. NaTaO3:La (5%) -- Band gap: 3.8 eV, edges straddle MeO/MeO- redox
2. KTaO3:La (5%)  -- Band gap: 3.6 eV, VBM slightly misaligned
3. LiTaO3:La (5%) -- Band gap: 4.1 eV, too wide for efficient UV absorption

Method: PBE+U (U=4.0 eV on Ta-5d), verified with HSE06 for NaTaO3
Convergence: ecutwfc=60 Ry, k-mesh 6x6x6 (tested at 4x4x4, 8x8x8)
```

**Example 2: Multi-Scale Force Field Parameterization**

User: "Chain a DFT calculation on a small water box into a MACE ML potential, then run a large-scale LAMMPS MD simulation to predict density at 298K and 1 bar."

Approach:
1. Strategist plans three-phase workflow: (a) DFT energy/force calculations on 64-molecule water box snapshots, (b) train MACE potential on DFT data, (c) NPT equilibration of 4096-molecule box with LAMMPS using the trained potential.
2. Operator runs Quantum ESPRESSO for 50 uncorrelated snapshots, extracts energies and forces via ASE.
3. Operator trains MACE model, validates on held-out 10 snapshots (force MAE < 50 meV/A threshold).
4. Operator generates LAMMPS input for NPT at 298K/1bar, 2ns equilibration + 1ns production.
5. Silent-failure check at 15 min: verifies density is converging toward ~1.0 g/cm3, not diverging.
6. Evaluator compresses: final density, RDF comparison to experiment, force-field validation metrics.

Output:
```
Multi-Scale Workflow Complete
=============================
Phase 1: DFT (QE/PBE-D3) -- 50 snapshots, 64 H2O molecules
Phase 2: MACE training -- Force MAE: 32 meV/A (threshold: 50)
Phase 3: LAMMPS NPT -- 4096 molecules, 3 ns total
Result: rho = 0.997 +/- 0.003 g/cm3 (expt: 0.997 g/cm3)
Checkpoint: ~/.quasar/runs/water-density-2026/checkpoint.json
```

**Example 3: Agent Framework for Non-Simulation Domains**

User: "I want to adapt the QUASAR Strategist-Operator-Evaluator pattern for an autonomous data analysis pipeline that processes CSV datasets."

Approach:
1. Map the three-agent architecture: Strategist plans the analysis (EDA, feature engineering, modeling), Operator executes pandas/sklearn code, Evaluator validates outputs and compresses context.
2. Implement double-pass planning: first pass generates analysis steps, second pass checks for missing data validation, outlier handling, and train/test split.
3. Implement hierarchical knowledge retrieval adapted for data science: Level 1 (confident in pandas syntax), Level 2 (RAG over sklearn docs), Level 3 (explore example notebooks in repo), Level 4 (web search for domain-specific techniques).
4. Add checkpoint after each analysis phase; Evaluator compresses completed phases into summary statistics rather than full DataFrames.
5. Auto-improvement loop: if model performance is poor, Strategist enters diagnostic mode and proposes feature engineering changes.

Output:
```python
# Strategist-Operator-Evaluator pipeline scaffold
class Strategist:
    def plan(self, objective: str, data_schema: dict) -> TaskDAG:
        """Double-pass: generate plan, then validate completeness."""
        initial = self._generate_plan(objective, data_schema)
        return self._validate_and_patch(initial)

class Operator:
    retrieval_levels = [
        InternalConfidence,   # L1: proceed if confident
        SemanticRAG,          # L2: search indexed docs
        RepoExplorer,         # L3: parse example files
        WebSearchFallback,    # L4: external search
    ]
    def execute(self, task: Task) -> TaskResult:
        knowledge = self._retrieve(task, self.retrieval_levels)
        return self._run_with_checkpointing(task, knowledge)

class Evaluator:
    def verify_and_compress(self, result: TaskResult) -> Summary:
        """Validate output, compress context, trigger improvement if needed."""
        if not self._meets_criteria(result):
            return self._trigger_diagnostic_replan(result)
        return self._compress(result)
```

## Best Practices

- **Do** implement the double-pass planning validation. The second pass catches domain-specific requirements (equilibration, convergence tests, charge balancing) that LLMs routinely omit in initial plans.
- **Do** use the four-level knowledge hierarchy strictly in order. Skipping to web search introduces hallucinated parameters; filesystem exploration of example inputs (Level 3) is often more reliable than semantic RAG for specialized input syntaxes.
- **Do** compress task context aggressively after Evaluator verification. Token expansion from accumulated conversation history degrades LLM focus -- retain only parameter choices and key results, not debug traces.
- **Do** implement the interruption-awareness prompt on checkpoint recovery. Without it, the system restarts from scratch rather than resuming from partially converged states.
- **Avoid** hard-coding tool schemas or agent hierarchies. QUASAR's power comes from prompt-based extensibility -- new simulation engines are added via documentation and examples, not code-level integration.
- **Avoid** treating auto-improvement as full replanning. Diagnostic mode should propose targeted corrections (switch functional, adjust convergence threshold) not regenerate the entire workflow.

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| DFT non-convergence | Silent-failure check: energy oscillating after N SCF cycles | Operator reduces mixing parameter, increases ecutwfc, or switches to different diagonalization algorithm |
| Simulation divergence | Periodic density/temperature check exceeds physical bounds | Terminate run, reduce timestep or adjust thermostat coupling |
| Wrong method selection | Evaluator finds results contradict known benchmarks | Auto-improvement loop: Strategist enters diagnostic mode, switches method (e.g., PBE+U to HSE) |
| Checkpoint corruption | Checksum validation on reload | Fall back to previous valid checkpoint; if none, restart from last Evaluator-compressed summary |
| Knowledge retrieval miss | All four levels exhausted without confident answer | Return explicit uncertainty to Strategist; flag task for human review rather than guessing |
| Token context overflow | Context length approaching model limit | Force Evaluator compression of oldest task contexts; summarize and discard raw conversation |

## Limitations

- The three-agent architecture adds latency compared to direct tool-calling for simple, well-defined single-step calculations. For routine tasks with known parameters, a direct script is faster.
- Hierarchical knowledge retrieval depends on the quality of indexed documentation and example repositories. Poorly documented simulation codes will degrade Level 2 and Level 3 retrieval, forcing frequent fallback to unreliable web search.
- Auto-improvement loops can cycle indefinitely if the Evaluator's success criteria are too strict or the method space is too large. Implement a maximum iteration count and escalate to human review.
- Context compression is lossy by design. If a downstream task needs details from an earlier compressed step, relevant information may have been discarded. Persist full logs alongside compressed summaries for manual recovery.
- The architecture assumes access to actual simulation engines (Quantum ESPRESSO, LAMMPS, RASPA, MACE). Without installed computational backends, the system can only generate input files, not execute and validate them.

## Reference

**Paper:** Yang & Evans, "QUASAR: A Universal Autonomous System for Atomistic Simulation and a Benchmark of Its Capabilities" (arXiv:2602.00185v1, 2026). Focus on Section 2 (architecture: Strategist/Operator/Evaluator roles), Section 2.2 (hierarchical knowledge retrieval protocol), and Section 3 (three-tiered benchmark results demonstrating generalized reasoning over task-specific automation).