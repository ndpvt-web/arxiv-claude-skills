---
name: "teaching-evaluating-reason-about"
description: "Apply knowledge-augmented reasoning distillation for polymer design tasks. Builds structured Chain-of-Thought pipelines grounded in polymer property knowledge bases. Triggers: 'polymer property prediction', 'polymer design constraints', 'SMILES polymer reasoning', 'polymer Tg prediction', 'knowledge-augmented CoT for materials', 'polymer benchmark evaluation'"
---

# Knowledge-Augmented Reasoning for Polymer Design

This skill enables Claude to tackle polymer design tasks — property prediction, structure generation, constraint satisfaction, and comparative ranking — using the knowledge-augmented reasoning distillation method from PolyBench (arXiv:2601.16312). Instead of relying on memorized facts, the approach grounds every reasoning step in a structured polymer profile (SMILES, experimental properties, RDKit-computed descriptors) and decomposes complex multi-property tasks into explicit Chain-of-Thought traces that mirror how subject-matter experts reason about polymer design.

## When to Use

- When the user asks to predict a polymer property (Tg, Tm, density, tensile strength, etc.) given a SMILES string or polymer name
- When designing or screening polymers that must satisfy multiple constraints simultaneously (e.g., "find a polyester with Tg > 80 C and good solvent resistance")
- When comparing or ranking candidate polymers across several properties
- When generating valid polymer SMILES structures that meet specific property targets
- When building a structured reasoning pipeline for materials informatics or polymer data analysis
- When creating training data or evaluation benchmarks for polymer-aware language models
- When explaining structure-property relationships in polymers with mechanistic reasoning

## Key Technique: Knowledge-Augmented Reasoning Distillation

The core insight from PolyBench is that LLMs fail at polymer design not because they lack reasoning ability, but because they lack grounding in polymer-specific evidence. The solution is a three-stage pipeline:

**Stage 1 — Knowledge Injection.** Before reasoning begins, augment each query with a standardized polymer profile. This profile aggregates the polymer's structural representation (IUPAC name, SMILES), experimentally reported properties (Tg, Tm, tensile strength, density, solubility parameters), and RDKit-computed structural attributes (molecular weight, LogP, H-bond donors/acceptors, rotatable bonds, topological polar surface area, aromatic ring count). This grounding eliminates hallucination by making the model explain provided evidence rather than generate facts from memory.

**Stage 2 — Structured CoT Decomposition.** The task is decomposed into explicit reasoning steps: (1) identify the key structural features and constraints, (2) retrieve relevant property data from the profile, (3) apply domain reasoning (e.g., how aromatic backbone rigidity raises Tg), (4) perform any quantitative comparisons or calculations, (5) state the conclusion with justification. This mirrors how polymer scientists solve multi-property design problems and is particularly effective for compositional tasks requiring trade-off reasoning.

**Stage 3 — Automated Verification.** Each generated reasoning chain is fact-checked against ground truth property values and structural data. Chains with incorrect intermediate reasoning are filtered out. This achieves ~80% accuracy in identifying correct reasoning chains and ensures training signal quality.

## Step-by-Step Workflow

1. **Parse the polymer query** into its components: identify target property/properties, any structural constraints (polymer class, monomer type, backbone features), and desired output format (numeric prediction, ranking, structure generation, or explanation).

2. **Build the polymer profile** for each polymer involved. Assemble: (a) SMILES string or repeat unit notation, (b) IUPAC or common name, (c) known experimental properties from the user's data or standard databases (PolyInfo, Polymer Genome), (d) RDKit-computed descriptors — compute molecular weight, LogP, number of H-bond donors/acceptors, rotatable bond count, topological polar surface area, and aromatic ring count from the SMILES.

3. **Classify the task complexity** using the PolyBench taxonomy:
   - **Structural Understanding**: parsing SMILES syntax, identifying functional groups, repeat units
   - **Conceptual Knowledge**: definitions, polymer class identification, basic principles
   - **Property Prediction**: single structure-to-property estimation
   - **Property Comparison & Ranking**: multi-polymer screening with property constraints
   - **Advanced Property Reasoning**: mechanistic trade-off analysis across multiple properties
   - **Synthesis & Design**: monomer selection, reaction pathway, constrained structure generation

4. **Inject knowledge into the prompt** by prepending the polymer profile(s) as structured context. Format as a clear block containing all known properties and computed descriptors so every reasoning step can reference concrete data.

5. **Decompose the reasoning** into an explicit CoT chain:
   - State the premise (what is being asked and what constraints apply)
   - List the relevant data points from the polymer profile
   - Apply domain reasoning (structure-property relationships, known trends)
   - Perform quantitative comparisons if multiple polymers or thresholds are involved
   - State the conclusion with explicit justification tied to the data

6. **For structure generation tasks**, produce candidate SMILES strings and validate them: check chemical validity (parseable by RDKit), verify the structure matches the target polymer class, and confirm computed properties align with specified constraints.

7. **For ranking/comparison tasks**, create a structured comparison table with each polymer as a row and each property as a column. Apply the constraints as filters, then rank remaining candidates by the optimization target.

8. **Verify the output** against known data: cross-check predicted properties against experimental ranges, validate SMILES strings parse correctly, and confirm the reasoning chain is internally consistent (no step contradicts the data in the profile).

9. **Report results** with the full reasoning chain visible, so the user can audit each step. Include confidence qualifiers where property estimates extrapolate beyond known data ranges.

## Concrete Examples

**Example 1: Property Prediction with Knowledge Augmentation**

```
User: Predict the glass transition temperature (Tg) of polycarbonate
      given its repeat unit SMILES: *OC(=O)Oc1ccc(C(C)(C)c2ccc(O*)cc2)cc1

Approach:
1. Parse the SMILES to identify structural features:
   - Bisphenol A backbone (two phenyl rings connected by C(CH3)2)
   - Carbonate linkage (OC(=O)O)
   - Aromatic content: high (two aromatic rings per repeat unit)

2. Build the polymer profile via RDKit on the repeat unit:
   - Molecular weight of repeat unit: ~254.3 g/mol
   - Aromatic ring count: 2
   - Rotatable bonds: 4 (limited by rigid aromatic backbone)
   - H-bond acceptors: 3 (carbonate oxygens)
   - LogP: ~3.1 (moderately hydrophobic)

3. Apply structure-property reasoning:
   - High aromatic content restricts backbone mobility → raises Tg
   - The C(CH3)2 bridging group adds steric bulk → further restricts rotation
   - Carbonate linkage provides some flexibility but is offset by aromatics
   - Known trend: aromatic polycarbonates have Tg in the 140-160 C range

4. Conclusion with justification:
   Predicted Tg ~ 147 C. The rigid bisphenol A backbone with two aromatic
   rings and the sterically bulky isopropylidene bridge severely restrict
   chain mobility. This is consistent with experimental values of 145-150 C
   for BPA-polycarbonate.

Output:
  Property: Glass Transition Temperature (Tg)
  Prediction: ~147 C
  Confidence: High (well-characterized polymer, strong structural rationale)
  Key factors: aromatic backbone rigidity, steric bulk of C(CH3)2 bridge
```

**Example 2: Multi-Constraint Polymer Screening**

```
User: Find candidate polymers from this list that satisfy:
      Tg > 100 C, tensile strength > 60 MPa, and soluble in chloroform.
      Candidates: polyethylene, polystyrene, polycarbonate, nylon-6,6, PMMA

Approach:
1. Build profiles for each polymer with known properties:

   | Polymer        | Tg (C) | Tensile (MPa) | CHCl3 Soluble |
   |---------------|--------|----------------|---------------|
   | Polyethylene   | -120   | 25-45          | No            |
   | Polystyrene    | 100    | 40-60          | Yes           |
   | Polycarbonate  | 147    | 55-75          | Yes           |
   | Nylon-6,6      | 47     | 70-85          | No            |
   | PMMA           | 105    | 50-77          | Yes           |

2. Apply constraints as sequential filters:
   - Tg > 100 C → eliminates PE (-120), nylon-6,6 (47), borderline PS (100)
   - Tensile > 60 MPa → from remaining {PC, PMMA}: PC (55-75, upper range
     meets it), PMMA (50-77, upper range meets it)
   - CHCl3 soluble → both PC and PMMA pass

3. Rank by combined performance:
   - Polycarbonate: Tg=147, tensile up to 75 MPa — strong on both
   - PMMA: Tg=105, tensile up to 77 MPa — meets thresholds but marginal Tg

Output:
  Recommended: Polycarbonate (best combined margin on all constraints)
  Alternative: PMMA (meets all constraints but with less margin on Tg)
  Eliminated: PE (fails Tg, tensile, solubility), PS (borderline Tg, low
  tensile), Nylon-6,6 (fails Tg, insoluble in CHCl3)
```

**Example 3: Structure Generation with Constraint Satisfaction**

```
User: Generate a polyester SMILES structure with predicted Tg > 80 C
      and thermal stability above 300 C.

Approach:
1. Identify design principles:
   - Polyester = ester linkages in backbone (-C(=O)O-)
   - Tg > 80 C requires backbone rigidity → aromatic diols/diacids
   - Thermal stability > 300 C requires aromatic backbone, no aliphatic
     weak links

2. Design candidate: Poly(ethylene naphthalate) — PEN
   - SMILES: *OC(=O)c1cc2cc(C(=O)O*)ccc2cc1
   - Naphthalene diacid provides extended aromatic system
   - Known properties: Tg ~ 120 C, thermal stability ~ 380 C

3. Validate via RDKit:
   - SMILES parses correctly ✓
   - Aromatic ring count: 2 (fused naphthalene) ✓
   - Ester linkages present ✓
   - Molecular weight of repeat unit: ~242 g/mol

4. Verify against constraints:
   - Tg ~ 120 C > 80 C ✓
   - Thermal stability ~ 380 C > 300 C ✓
   - Polyester class ✓

Output:
  Generated structure: Poly(ethylene naphthalate)
  SMILES: *OC(=O)c1cc2cc(C(=O)O*)ccc2cc1
  Predicted Tg: ~120 C (exceeds 80 C threshold)
  Predicted thermal stability: ~380 C (exceeds 300 C threshold)
  Rationale: Fused naphthalene ring system provides both chain rigidity
  (high Tg) and thermal stability through aromatic resonance stabilization.
```

## Best Practices

- **Do:** Always build the polymer profile first before reasoning. Ground every claim in concrete structural data or known property values — never assert properties from memory alone.
- **Do:** Use RDKit-computed descriptors (MW, LogP, TPSA, rotatable bonds, aromatic rings, H-bond donors/acceptors) as quantitative anchors for structure-property reasoning.
- **Do:** Decompose multi-constraint problems into sequential filters. Check each constraint independently, then rank survivors by margin of compliance.
- **Do:** Validate generated SMILES strings for chemical correctness — a structure that doesn't parse is useless regardless of how good the reasoning was.
- **Avoid:** Predicting exact numeric property values without stating uncertainty. Use ranges and confidence qualifiers (e.g., "Tg ~ 120-130 C based on structural analogy to PET").
- **Avoid:** Treating SMILES as opaque tokens. Parse them into structural features (functional groups, ring systems, chain flexibility) that connect to property trends.
- **Avoid:** Skipping the knowledge injection step. Reasoning without a grounded polymer profile is the primary failure mode identified in the paper.

## Error Handling

- **Invalid SMILES input**: If the user provides a SMILES string that doesn't parse, attempt common corrections (missing closure digits, mismatched parentheses) and report what was fixed. If unfixable, ask the user to verify the structure.
- **Unknown polymer properties**: When experimental data is unavailable, state this explicitly. Use structural analogy to similar known polymers and flag the prediction as low-confidence.
- **Contradictory constraints**: If the user's constraints are physically incompatible (e.g., "high crystallinity AND amorphous Tg behavior"), explain the trade-off and suggest relaxing one constraint.
- **Ambiguous polymer names**: Many polymers have multiple common names. Confirm the specific structure via SMILES before reasoning about properties.
- **Out-of-distribution polymers**: For novel or exotic polymer architectures (dendrimers, star polymers, complex copolymers), acknowledge that property predictions extrapolate beyond typical training data and reliability decreases.

## Limitations

- Property predictions are qualitative-to-semi-quantitative. For engineering-grade numeric predictions, molecular simulation or experimental measurement is required.
- The structured CoT approach excels at compositional reasoning (multi-property trade-offs, ranking) but shows limited improvement for tasks requiring precise quantitative targets, as noted in the paper.
- SMILES notation for polymers is inherently ambiguous for complex architectures (branched, crosslinked, block copolymers). The approach works best for linear homopolymers and simple copolymers.
- Knowledge augmentation depends on the quality and coverage of the polymer property database. Rare or newly synthesized polymers may lack sufficient grounding data.
- This approach does not replace computational chemistry tools (DFT, MD simulation) for high-accuracy property prediction — it provides rapid, reasoned screening and design guidance.

## Reference

**Paper**: "Teaching and Evaluating LLMs to Reason About Polymer Design Related Tasks" — arXiv:2601.16312v1 (2026). Key contribution: the knowledge-augmented reasoning distillation pipeline (knowledge injection → structured CoT → automated verification) and the PolyBench benchmark with six task categories spanning structural understanding through synthesis design.