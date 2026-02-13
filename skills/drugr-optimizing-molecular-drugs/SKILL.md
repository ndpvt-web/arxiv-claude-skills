---
name: "drugr-optimizing-molecular-drugs"
description: "Optimize molecular drug candidates using LLM-based explicit pharmacological reasoning over SMILES structures. Applies the DrugR three-phase pipeline (continual pretraining, reverse-data-engineered SFT, multi-granular RL) to improve ADMET properties while preserving binding affinity. Use when: 'optimize this drug molecule', 'improve ADMET properties of this compound', 'reason about molecular modifications', 'explain why this structural change improves absorption', 'build a molecule optimization pipeline', 'generate drug analogs with better toxicity profiles'."
---

# DrugR: LLM-Based Explicit Reasoning for Molecular Drug Optimization

This skill enables Claude to apply the DrugR framework for optimizing small-molecule drug candidates through explicit, step-by-step pharmacological reasoning. Rather than treating molecular optimization as a black-box generation task, DrugR introduces a structured reasoning chain that connects each structural edit (expressed in SMILES notation) to predicted changes in ADMET properties (Absorption, Distribution, Metabolism, Excretion, Toxicity), enabling interpretable and actionable optimization decisions. The approach combines domain-specific pretraining, reverse data engineering for supervised fine-tuning, and Pareto-aware multi-granular reinforcement learning.

## When to Use

- When the user provides a SMILES string and asks to improve its drug-likeness, toxicity profile, or other ADMET properties
- When building a pipeline to generate and evaluate molecular analogs with improved pharmacological profiles
- When the user needs interpretable reasoning for why a specific molecular modification improves or degrades a drug property
- When implementing a multi-objective optimization system that must balance structural similarity, binding affinity, and ADMET scores simultaneously
- When creating training data for molecular optimization models using reverse data engineering (pairing worse-to-better molecule pairs with LLM-generated rationales)
- When the user wants to evaluate candidate molecules against a panel of ADMET indicators (QED, logP, TPSA, CYP inhibition, hepatotoxicity, AMES toxicity, etc.)

## Key Technique

**The core insight of DrugR is that LLMs struggle with molecular optimization because the relationship between molecular structure (SMILES) and pharmacological properties is implicit and complex.** DrugR solves this by forcing explicit reasoning: before proposing a structural modification, the model must articulate which functional groups affect which properties, what trade-offs exist, and why the proposed edit is expected to improve the target metrics. This reasoning-first approach dramatically improves optimization quality compared to direct SMILES-to-SMILES generation.

**The training pipeline has three phases.** Phase 1 (Continual Pretraining) feeds the LLM ~610K samples mixing chemical QA, MoleculeNet benchmarks, and general text to build molecular literacy without catastrophic forgetting. Phase 2 (Supervised Fine-Tuning via Reverse Data Engineering) constructs training pairs by: collecting ~10K approved drugs from DrugBank, generating structurally similar candidates, scoring all molecules with ADMETLab, pairing worse-to-better molecules, then using a reasoning model (DeepSeek-R1) to annotate the transformation rationale. This yields ~4,855 high-quality (molecule, reasoning, optimized_molecule) triples. Phase 3 (Self-Balanced Multi-Granular RL) applies GRPO with separate reward heads for the reasoning segment (target-property F1, reasoning quality, reasoning richness) and the SMILES segment (overall optimization score, ECFP4 fingerprint similarity, binding energy), using Pareto-aware reweighting to prevent any single reward from dominating.

**The result is a model that achieves 89.5% relative improvement in overall ADMET optimization score while maintaining >0.64 fingerprint similarity and 95.84% binding affinity preservation** (<=−6 kcal/mol). Critically, each optimization comes with an explicit reasoning chain that a medicinal chemist can audit.

## Step-by-Step Workflow

1. **Parse the input molecule as a SMILES string.** Validate the SMILES using RDKit (`Chem.MolFromSmiles()`). If the user provides a common name or structure diagram, convert to SMILES first. Reject invalid SMILES immediately with a clear error.

2. **Profile the baseline ADMET properties.** Compute or retrieve key indicators for the input molecule: QED (quantitative estimate of drug-likeness), logP (lipophilicity), TPSA (topological polar surface area), molecular weight, number of hydrogen bond donors/acceptors, CYP inhibition predictions, hepatotoxicity risk, AMES mutagenicity, and Caco-2 permeability. Use ADMETLab 2.0/3.0 API or local predictors where available.

3. **Identify optimization targets.** Based on the user's request or the baseline profile, determine which ADMET properties need improvement. Explicitly state which properties are deficient and what acceptable ranges look like (e.g., logP between 1-3 for oral bioavailability, TPSA < 140 A^2 for membrane permeability).

4. **Generate explicit pharmacological reasoning.** Before proposing any structural change, write a step-by-step reasoning chain that: (a) identifies the substructures or functional groups responsible for the deficient property, (b) proposes a specific structural modification (e.g., replacing an aromatic nitro group to reduce mutagenicity, adding a hydroxyl for solubility), (c) predicts the expected effect on the target property AND potential side-effects on other properties, (d) assesses trade-offs.

5. **Propose the optimized SMILES.** Apply the reasoned modification to produce a new SMILES string. Validate it with RDKit. Compute ECFP4 fingerprint similarity (Tanimoto) to the original--target >= 0.6 to preserve the pharmacophore.

6. **Evaluate the optimized molecule.** Recompute all ADMET indicators on the new SMILES. Compare against the baseline. If binding affinity is critical, run docking estimation (e.g., AutoDock Vina or a surrogate model) to verify the modification preserves target engagement (<= −6 kcal/mol).

7. **Apply multi-objective Pareto assessment.** Check whether the new molecule Pareto-dominates the original across the targeted properties without regressing on others. If trade-offs exist, explicitly report them. If the molecule is dominated, iterate with a different modification strategy.

8. **Iterate with self-balancing.** If the first optimization attempt doesn't satisfy all constraints, revisit the reasoning chain. Prioritize the most deficient property but apply diminishing weight to properties that are already in-range (the Pareto-aware reweighting principle from DrugR's RL phase).

9. **Present results with full reasoning transparency.** Output the original SMILES, optimized SMILES, a property comparison table, the structural modification description, and the complete reasoning chain. Flag any properties that worsened.

10. **Generate multiple candidates if requested.** Repeat steps 4-8 with different modification strategies (e.g., bioisosteric replacement, scaffold hopping, side-chain variation) to provide a ranked set of candidates.

## Concrete Examples

**Example 1: Reducing toxicity while maintaining efficacy**

```
User: This compound has good binding affinity but fails AMES mutagenicity.
      Optimize it: CC1=CC(=O)C2=C(C1)[N+](=O)C=C2N

Approach:
1. Parse and validate SMILES -> valid molecule (nitro-substituted aminoquinone)
2. Baseline ADMET profile:
   - AMES mutagenicity: POSITIVE (toxic) -- primary concern
   - QED: 0.38 (low drug-likeness)
   - logP: 1.2 (acceptable)
   - Binding affinity: -8.2 kcal/mol (good)

3. Explicit reasoning chain:
   "The aromatic nitro group [N+](=O) is a known structural alert for
   mutagenicity. Nitro groups undergo nitroreduction to form reactive
   nitroso and hydroxylamine intermediates that damage DNA. Replacing
   the nitro group with a cyano (-C#N) group preserves the electron-
   withdrawing character needed for binding but eliminates the
   mutagenic liability. The amino group at position 2 should be
   retained as it contributes to target binding via hydrogen bonding.
   Trade-off: cyano substitution may slightly increase logP (+0.3)
   but this remains within acceptable range."

4. Proposed optimized SMILES: CC1=CC(=O)C2=C(C1)C(=CC2N)C#N
5. Evaluation:
   - AMES mutagenicity: NEGATIVE (resolved)
   - QED: 0.52 (improved)
   - logP: 1.5 (still acceptable)
   - Fingerprint similarity: 0.72 (above 0.6 threshold)
   - Binding affinity: -7.6 kcal/mol (preserved)

Output: Property comparison table + reasoning chain + optimized SMILES
```

**Example 2: Building a reverse data engineering pipeline**

```
User: I want to create training data for a molecular optimization model
      using the DrugR reverse data engineering approach.

Approach:
1. Collect approved drugs from DrugBank (~10K molecules with SMILES)
2. For each drug, generate 5-10 structurally similar molecules using
   constrained molecular generation (e.g., fragment-based enumeration
   or matched molecular pair transforms), maintaining Tanimoto > 0.5
3. Score ALL molecules (originals + generated) with ADMETLab across
   23 ADMET indicators
4. For each property, pair molecules where one is better:
   (worse_mol, better_mol, target_properties)
5. Filter to pairs where the better molecule improves at least one
   target property without catastrophically degrading others
6. Use an LLM (e.g., DeepSeek-R1 or Claude) to annotate each pair
   with a reasoning chain explaining the structural difference and
   why it leads to the property improvement
7. Format as training triples:
   (input_smiles, reasoning_chain, optimized_smiles)
8. Split 85/10/5 train/dev/test

Output structure:
{
  "input": "CC(=O)Oc1ccccc1C(=O)O",
  "reasoning": "The ester linkage at position 1 is susceptible to rapid
    hydrolysis by plasma esterases, limiting oral bioavailability.
    Replacing the acetyl ester with a methyl ether preserves the
    oxygen's role in the hydrogen-bond network while eliminating
    the hydrolytically labile bond. This should improve metabolic
    stability (HLM t1/2) without significantly altering logP...",
  "output": "COc1ccccc1C(=O)O",
  "properties_improved": ["metabolic_stability", "oral_bioavailability"]
}
```

**Example 3: Multi-objective optimization with Pareto assessment**

```
User: Optimize this molecule for both permeability AND reduced CYP3A4
      inhibition: O=C(NC1CCCCC1)c1cc2ccccc2[nH]1

Approach:
1. Baseline: Caco-2 permeability = -5.8 (poor), CYP3A4 inhibition = YES
2. Reasoning chain:
   "The indole-2-carboxamide scaffold presents two issues: (a) the
   large planar aromatic surface increases CYP3A4 binding, and (b)
   the NH on the indole limits passive permeability via desolvation
   penalty. Strategy: N-methylate the indole nitrogen to reduce
   H-bond donor count (improving permeability) and introduce a
   methyl at C-5 to disrupt planarity (reducing CYP3A4 fitting).
   Risk: N-methylation increases logP by ~0.5; monitor for
   solubility impact."
3. Propose: O=C(NC1CCCCC1)c1cc2cc(C)ccc2n1C
4. Evaluate both properties + check Pareto dominance
5. If Pareto-dominant: accept. If trade-off: report explicitly.

Output:
| Property          | Original | Optimized | Status    |
|-------------------|----------|-----------|-----------|
| Caco-2 perm       | -5.8     | -5.1      | Improved  |
| CYP3A4 inhibition | YES      | NO        | Improved  |
| logP              | 2.1      | 2.6       | Acceptable|
| Fingerprint sim   | --       | 0.74      | Pass      |
| Binding affinity   | -7.8     | -7.3      | Preserved |
Verdict: Pareto-dominant. Both targets improved, no regression.
```

## Best Practices

- **Do:** Always validate SMILES with RDKit before and after optimization. Invalid SMILES waste all downstream computation.
- **Do:** Write the reasoning chain BEFORE generating the optimized molecule. This mirrors DrugR's explicit reasoning architecture and catches flawed logic early.
- **Do:** Compute fingerprint similarity (ECFP4/Tanimoto) for every proposed modification. Reject candidates below 0.4 similarity--they've likely changed the pharmacophore.
- **Do:** Report ALL property changes, not just the target. A molecule with improved solubility but new hepatotoxicity is not an improvement.
- **Avoid:** Proposing modifications without a mechanistic rationale. "I changed the methyl to ethyl" is insufficient--explain WHY (e.g., "to increase metabolic stability by blocking CYP2D6-mediated N-demethylation").
- **Avoid:** Optimizing more than 2-3 properties simultaneously without Pareto analysis. Multi-objective optimization without trade-off tracking leads to hallucinated improvements.

## Error Handling

- **Invalid SMILES input:** Validate immediately with `Chem.MolFromSmiles()`. If None, report the specific parsing failure and ask the user to verify the structure.
- **RDKit sanitization failures on proposed molecule:** The structural modification produced a chemically invalid molecule. Revert the modification and try an alternative approach (e.g., different substitution position).
- **Fingerprint similarity below threshold:** The optimization was too aggressive. Constrain modifications to single-point changes (one functional group swap at a time) and re-attempt.
- **ADMET prediction service unavailable:** Fall back to RDKit descriptor calculations (QED, logP, TPSA, HBD/HBA, MW) which can be computed locally. Note that CYP inhibition and hepatotoxicity predictions require external models.
- **Binding affinity regression:** If docking scores worsen by > 1 kcal/mol, the modification likely disrupts a key pharmacophoric interaction. Analyze the binding pose to identify which interaction was lost and constrain the next modification to preserve it.
- **Conflicting multi-objective rewards:** When improving one property necessarily degrades another (e.g., permeability vs. solubility), explicitly present the Pareto frontier and let the user choose the trade-off point.

## Limitations

- **Claude cannot run RDKit or docking simulations natively.** The reasoning and SMILES generation steps work well, but quantitative ADMET scoring and fingerprint similarity calculations require the user to have RDKit, ADMETLab, or equivalent tools installed. Claude can generate the code to run these evaluations.
- **SMILES generation by LLMs is imperfect.** Complex structural modifications may produce invalid SMILES. Always validate programmatically--never trust LLM-generated SMILES without RDKit parsing.
- **No access to DrugR model weights directly.** This skill applies DrugR's reasoning methodology through Claude's general chemical knowledge. For production-grade molecular optimization, use the actual DrugR checkpoints from the authors' repository.
- **Binding affinity estimates are approximate.** Without running actual molecular docking (AutoDock Vina, Glide, etc.), binding energy predictions are qualitative. Flag this uncertainty in outputs.
- **Limited to small-molecule drugs.** DrugR's SMILES-based approach does not apply to biologics (antibodies, peptides > ~50 residues, nucleic acid therapeutics).
- **ADMET predictions have model-dependent accuracy.** Different prediction tools (ADMETLab, pkCSM, SwissADME) can give conflicting results. Cross-validate critical predictions with multiple tools.

## Reference

- **Paper:** [DrugR: Optimizing Molecular Drugs through LLM-based Explicit Reasoning](https://arxiv.org/abs/2602.08213v1) -- Focus on Section 3 (Methodology) for the three-phase training pipeline and Section 4 (Experiments) for the 23-indicator ADMET evaluation protocol.
- **Code:** [https://github.com/Haoranliu-lab/DrugR-main](https://github.com/Haoranliu-lab/DrugR-main) -- Model checkpoints, training data, and evaluation scripts.