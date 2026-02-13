---
name: "marble-multi-agent-reasoning-bioinformatics"
description: "Iteratively refine bioinformatics and ML models using MARBLE's multi-agent debate framework with role-specialized agents, literature-aware reference selection, and performance-grounded memory. Use when the user says 'refine my model architecture', 'iteratively improve this bioinformatics pipeline', 'debate-driven model optimization', 'use multi-agent reasoning to improve my ML model', 'MARBLE-style refinement', or 'autonomous model evolution'."
---

# MARBLE: Multi-Agent Reasoning for Bioinformatics Learning and Evolution

This skill enables Claude to apply the MARBLE framework — a structured multi-agent debate system for iteratively refining ML model architectures. MARBLE uses role-specialized agents (Research Group, Critic, Model Principal, Implement Architect, Code Expert, and Validators) that propose, debate, and implement architectural modifications grounded in literature and empirical performance. Rather than making one-shot suggestions, MARBLE runs a closed refinement loop: retrieve relevant papers, debate architectural changes, implement the winner, evaluate, and update a performance-grounded memory that steers future iterations away from failed approaches and toward proven gains.

## When to Use

- When the user wants to iteratively improve a bioinformatics or ML model architecture (e.g., spatial transcriptomics, drug-target interaction, drug response prediction)
- When the user asks for literature-informed architectural suggestions with structured critique before implementation
- When the user needs an autonomous refinement loop that tracks what worked and what failed across multiple iterations
- When the user wants to avoid regression during model evolution by using checkpoint-based gating
- When the user asks to set up a multi-agent debate workflow for model design decisions
- When the user has a working baseline model and wants systematic, evidence-based improvements rather than ad-hoc tuning

## Key Technique

MARBLE's core insight is that reliable model improvement requires **structured disagreement** before implementation. A single agent proposing changes tends toward repetitive or unstable modifications. MARBLE instead orchestrates a pipeline of eight role-specialized agents across four modules: Paper Selection, Ideation (debate), Execution, and Evolving Memory Update. The Research Group proposes modifications drawn from retrieved literature, the Critic challenges feasibility and compatibility, and the Model Principal ranks candidates by expected impact. Only the winning proposal proceeds to implementation by the Code Expert, with validation gates at each step.

The **reference selection** mechanism balances exploitation and exploration. Papers are scored by a hybrid of embedding similarity (BGE-M3 against the current architecture) and a batch-aggregated reward signal computed every 10 iterations from success/failure counts. A domain-weight modulation ensures the system doesn't over-exploit familiar territory: 2 high-domain papers (exploitation), 1 mid-domain, and 2 low-domain papers (exploration) are selected per cycle. This prevents the refinement loop from converging prematurely on a local optimum.

The **performance-grounded memory** is what separates MARBLE from one-shot LLM code generation. After each iteration, the system records the references used, the architectural modification applied, and the quantitative outcome. Every 10 iterations, empirical performance signals suppress ineffective references and promote promising directions. The historically best-performing checkpoint serves as the foundation for all subsequent modifications, so regressions are structurally prevented rather than merely detected. Ablation studies show that removing evolving memory causes the largest performance drop (NPG from 0.231 to 0.144), confirming that institutional memory across iterations is the framework's most valuable component.

## Step-by-Step Workflow

1. **Establish baseline.** Run the user's current model on their evaluation dataset. Record the baseline metric (accuracy, AUC, MSE, etc.) and save the model checkpoint. This becomes iteration 0 in the memory ledger.

2. **Retrieve candidate references.** Search for relevant papers using domain-specific and architecture-specific keywords (e.g., "graph attention spatial transcriptomics", "transformer drug binding affinity"). Filter to high-quality sources. Score each paper with hybrid similarity: `S(i) = S_embedding(i) + 0.1 * R_batch(b)`, where the reward term is zero for the first batch of 10 iterations.

3. **Select a balanced reference set.** Pick 5 papers using domain-weight modulation: 2 from high-similarity (exploitation, wd=0.9), 1 from mid-similarity (wd=0.5), 2 from low-similarity (exploration, wa=0.9). This ensures both incremental refinements and novel architectural ideas enter the debate.

4. **Run the Ideation debate.** Simulate three agent roles:
   - **Research Group**: Propose 2-3 architectural modifications grounded in the selected papers, citing specific techniques (e.g., "replace GCN layers with GAT from Paper #3 to capture spatial attention").
   - **Critic**: Challenge each proposal on compatibility with the existing architecture, implementation complexity, and risk of regression. Cite specific failure modes.
   - **Model Principal**: Rank proposals by expected performance impact given the memory of prior successes/failures, and select the winning modification.

5. **Document the implementation plan.** The Implement Architect role converts the winning proposal into a concrete technical blueprint: which files change, which layers/modules are added or replaced, what hyperparameters need tuning, and what the expected code diff looks like. The Plan Validator checks feasibility before coding begins.

6. **Implement the modification.** The Code Expert role writes or modifies the model code according to the blueprint. Keep changes minimal and targeted — one architectural modification per iteration, not a full rewrite.

7. **Validate the code.** The Code Validator role checks syntactic correctness, configuration consistency, and that the modification matches the blueprint. If validation fails, allow up to 10 retry attempts. The Code Expert can contest false-positive validation errors (the rebuttal mechanism).

8. **Execute and evaluate.** Run the modified model on the same evaluation dataset as the baseline. Record the metric. Compare against the historically best checkpoint, not just the previous iteration.

9. **Update the evolving memory.** Log the iteration entry: references used, modification applied, metric achieved, success/failure flag. If the new metric exceeds the historical best, update the best checkpoint. Every 10 iterations, recompute batch-aggregated rewards for all references to adjust future selection scores.

10. **Decide: continue or converge.** If the patience counter (e.g., 10 consecutive non-improving iterations) is exhausted, stop and return the best checkpoint. Otherwise, loop back to step 2 with updated memory and reference scores.

## Concrete Examples

**Example 1: Improving a spatial transcriptomics model**

```
User: I have a STAGATE model for spatial domain segmentation on the DLPFC
dataset. It gets ARI=0.45. Help me iteratively improve it using the
MARBLE approach.

Approach:
1. Record baseline: STAGATE on DLPFC, ARI=0.45, save checkpoint.
2. Retrieve papers: search "spatial transcriptomics graph attention
   domain segmentation" on PMC/OpenReview. Score ~200 candidates.
3. Select 5 references: 2 high-similarity (e.g., SpaGCN, GraphST),
   1 mid (e.g., a general graph transformer paper),
   2 low (e.g., a contrastive learning method from vision domain).
4. Debate:
   - Research Group proposes: (a) Add adaptive edge weighting from
     GraphST, (b) Replace linear decoder with MLP decoder per SpaGCN,
     (c) Add contrastive spatial loss from the vision paper.
   - Critic challenges: (c) requires negative sampling which may not
     transfer well to sparse spatial graphs. (a) is low-risk, high
     compatibility.
   - Model Principal selects (a): adaptive edge weighting, citing
     compatibility with existing GAT layers and low regression risk.
5. Implement: Add learnable edge weight parameters to the spatial
   graph construction in model.py, lines 45-60.
6. Evaluate: ARI improves to 0.48. Update memory, new best checkpoint.
7. Next iteration: memory suppresses GraphST-style approaches (already
   applied), promotes unexplored directions.

Output after 5 iterations:
  Iteration 0: ARI=0.450 (baseline)
  Iteration 1: ARI=0.480 (adaptive edge weights)
  Iteration 2: ARI=0.472 (MLP decoder — regression, reverted)
  Iteration 3: ARI=0.495 (attention-based spatial aggregation)
  Iteration 4: ARI=0.510 (layer normalization + residual connections)
  Iteration 5: ARI=0.508 (contrastive loss — no gain, kept iter 4)
  Best: ARI=0.510 at iteration 4, NPG=+0.060
```

**Example 2: Drug-target interaction prediction refinement**

```
User: My DeepDTA model for drug-target binding affinity prediction
has CI=0.878 on the Davis dataset. I want to systematically improve
it. Use multi-agent debate to pick the best modifications.

Approach:
1. Baseline: DeepDTA on Davis, CI=0.878.
2. Retrieve papers on "drug-target interaction deep learning binding
   affinity CNN transformer".
3. Select 5 references with balanced domain weights.
4. Debate round 1:
   - Research Group proposes: (a) Replace 1D-CNN with BiLSTM for
     protein sequences, (b) Add molecular graph encoder alongside
     SMILES CNN, (c) Cross-attention between drug and target
     representations.
   - Critic: (b) requires graph featurization pipeline — high
     implementation overhead. (c) is modular and additive.
   - Model Principal selects (c): cross-attention fusion layer.
5. Implement cross-attention between drug CNN output and protein CNN
   output before the fully connected prediction head.
6. Evaluate: CI=0.885. Record success, update memory.
7. Continue iterations with memory-informed reference selection.

Memory ledger after iteration 1:
  | Iter | Modification          | CI    | Status  |
  |------|-----------------------|-------|---------|
  | 0    | Baseline DeepDTA      | 0.878 | —       |
  | 1    | Cross-attention fusion| 0.885 | Success |
```

**Example 3: Setting up the debate for a custom model**

```
User: I'm building a drug response prediction model. Before I code
anything, help me run a MARBLE-style debate on whether to use a
graph neural network or a transformer for the cell line encoder.

Approach:
1. Skip to Ideation — no baseline code yet, this is a design debate.
2. Retrieve 5 papers: 2 on GNN cell line modeling, 1 hybrid, 2 on
   transformer genomic encoders.
3. Debate:
   - Research Group (GNN advocate): GNNs capture gene-gene interaction
     topology directly. Papers show GNNs outperform MLPs by 8-12% on
     GDSC when pathway graphs are available.
   - Research Group (Transformer advocate): Transformers handle
     variable-length gene expression profiles without predefined graph
     structure. Self-attention learns implicit interactions.
   - Critic: GNN requires curated pathway graph — adds data dependency.
     Transformer requires more data to learn interactions from scratch.
     What is the user's dataset size?
   - Model Principal: For datasets >5000 samples, recommend transformer.
     For smaller datasets with available pathway graphs, recommend GNN.
     Ask user for dataset specifics before deciding.
4. Output the structured recommendation with cited trade-offs.

Output:
  DEBATE SUMMARY
  ─────────────
  Option A: GNN cell line encoder
    Pro: Encodes known biological topology, strong on small datasets
    Con: Requires curated pathway graph, rigid to new gene sets
    Cited: [Paper 1], [Paper 2]

  Option B: Transformer cell line encoder
    Pro: No graph dependency, learns implicit interactions, scalable
    Con: Needs >5K samples to outperform GNN, higher compute cost
    Cited: [Paper 4], [Paper 5]

  RECOMMENDATION: Depends on dataset size. Please specify sample count
  and whether pathway graph data is available.
```

## Best Practices

**Do:**
- Make exactly one architectural modification per iteration. Multiple changes confound which modification caused the performance change.
- Always compare against the historically best checkpoint, not just the previous iteration. This prevents accepting a sequence of small regressions.
- Balance reference selection between exploitation (similar papers) and exploration (cross-domain papers). The 2-1-2 split (high-mid-low domain similarity) is the empirically validated default.
- Record both successes and failures in the memory ledger. Failed approaches are as informative as successful ones for steering future iterations.
- Set a patience parameter (e.g., 10 non-improving iterations) to prevent infinite loops.

**Avoid:**
- Do not skip the Critic role. Ablation shows removing debate drops net performance gain by ~15%. Unchallenged proposals lead to incompatible or redundant modifications.
- Do not ignore the memory when selecting references. Without batch-aggregated rewards, the system repeatedly selects the same ineffective papers.
- Do not rewrite the entire model in one iteration. MARBLE's stability comes from incremental, targeted changes with regression gating.
- Do not treat all references equally. Domain-weight modulation matters — pure exploitation converges prematurely, pure exploration wastes iterations on irrelevant techniques.

## Error Handling

- **Code validation failure after 10 retries**: Revert to the last valid checkpoint and log the modification as infeasible in memory. The Critic should flag this pattern in future debates if similar approaches are proposed.
- **Performance regression**: Do not update the best checkpoint. Log the modification with a failure flag. The batch reward signal will suppress the associated references in future cycles.
- **No relevant papers found**: Broaden search keywords by removing architecture-specific terms (keep only domain terms). If still empty, use the existing memory ledger to re-rank previously successful reference categories.
- **Debate deadlock (agents cannot reach consensus)**: Default to the lowest-risk proposal — the one the Critic rated as most compatible with the existing architecture. Log the deadlock for the Model Principal to weigh in future rounds.
- **Docker/execution environment failure**: This is an infrastructure issue, not a model issue. Retry execution up to 10 times. If persistent, check resource constraints and container configuration before attributing failure to the architectural modification.

## Limitations

- MARBLE is designed for **iterative refinement of an existing baseline**, not for building models from scratch. You need a working model and evaluation pipeline before the loop begins.
- The framework assumes **quantitative evaluation metrics** are available. Qualitative or subjective tasks (e.g., visualization quality) cannot drive the performance-grounded memory.
- Reference retrieval depends on **accessible literature databases**. If the domain lacks published papers (very novel tasks), the debate quality degrades because proposals lack grounding.
- Each iteration requires a **full training and evaluation cycle**, which can be computationally expensive. MARBLE is most practical when individual training runs are under a few hours.
- The multi-agent debate is a **structured simulation** — Claude plays all agent roles sequentially. True multi-agent parallelism (as in the original LangGraph implementation) is not replicated, but the structured role separation still provides the critical benefit of adversarial critique.
- Performance gains follow **diminishing returns**. The original paper shows most improvement in the first 10-20 iterations, with convergence typically within 50 iterations.

## Reference

**Paper**: [MARBLE: Multi-Agent Reasoning for Bioinformatics Learning and Evolution](https://arxiv.org/abs/2601.14349v1) (Kim et al., 2026)

Key sections to consult: Section 2 (Methods) for the four-module architecture and agent role definitions; Table 4 for ablation results showing evolving memory and debate are the two most critical components; Equation 1 for the hybrid reference scoring formula; Figure 2 for the full system diagram.

**Code**: [github.com/PRISM-DGU/MARBLE](https://github.com/PRISM-DGU/MARBLE) — LangGraph-based implementation with Docker execution environments.