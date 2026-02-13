---
name: "can-post-training-transform-causal"
description: "Perform rigorous causal inference tasks using structured reasoning pipelines inspired by CauGym. Estimate treatment effects (ATE, CDE, ETT, NDE, NIE), compute probabilities of necessity/sufficiency, apply the backdoor criterion for deconfounding, and build causal DAGs from domain knowledge. Trigger phrases: 'estimate causal effect', 'what is the treatment effect of', 'causal reasoning over this data', 'apply backdoor adjustment', 'counterfactual analysis', 'build a causal DAG for this problem'."
---

# Causal Inference Reasoning with CauGym-Style Pipelines

This skill enables Claude to perform structured causal inference on user-provided data or domain problems. Based on the CauGym framework (Chen et al., 2026), it applies Pearl's three-level causal hierarchy — association, intervention, and counterfactual — through explicit step-by-step reasoning over structural causal models (SCMs). Rather than relying on intuition or correlation, this skill enforces a disciplined pipeline: construct a causal DAG, identify adjustment sets via the backdoor criterion, compute causal quantities symbolically, and present results with clear assumptions stated.

## When to Use

- When the user asks to estimate a treatment effect (e.g., "What is the causal effect of X on Y?")
- When the user provides observational data and wants to know if a relationship is causal or confounded
- When the user needs to apply the backdoor criterion to identify valid adjustment sets
- When the user asks counterfactual questions ("If the patient had not received treatment, would they have recovered?")
- When the user wants to compute specific causal quantities: ATE, CDE, ETT, NDE, NIE, PN, or PS
- When the user needs to determine whether available data is sufficient for causal identification or if conditions are lacking
- When the user asks to build or validate a causal DAG for a domain problem
- When the user wants to reason about mediation (direct vs. indirect effects)

## Key Technique

**The CauGym approach** demonstrates that structured causal reasoning follows a learnable pipeline. The core insight is that causal inference tasks decompose into: (1) constructing a directed acyclic graph (DAG) encoding causal assumptions, (2) applying identification rules (backdoor criterion, front-door criterion, do-calculus) to determine whether a causal quantity is estimable from observational data, and (3) computing the quantity using the appropriate adjustment formula. This structured decomposition is what allows a 14B model to outperform GPT-o3 on causal benchmarks — the reasoning is procedural, not intuitive.

**The seven core tasks** span Pearl's causal hierarchy. At the interventional level: ATE (Average Treatment Effect — population-wide effect of treatment), CDE (Controlled Direct Effect — effect while holding mediators fixed), ETT (Effect of Treatment on the Treated — effect on the subpopulation that was actually treated). At the counterfactual level: NDE (Natural Direct Effect — direct pathway only), NIE (Natural Indirect Effect — mediated pathway only), PN (Probability of Necessity — was the treatment necessary for the outcome?), PS (Probability of Sufficiency — would treatment be sufficient to produce the outcome?). Each has a specific formula and identification condition.

**Critical robustness principle:** A rigorous causal reasoner must detect when data is insufficient. CauGym trains models to output `LACK_CONDITION` when required conditional probabilities are missing, and to ignore irrelevant variables (redundant information). Always verify that the data provided is sufficient before computing a causal estimate.

## Step-by-Step Workflow

1. **Identify the causal question type.** Classify the user's request into one of: ATE, CDE, ETT, NDE, NIE, PN, PS, causal discovery, or deconfounding. If ambiguous, ask the user to clarify whether they want a population-level effect (ATE), a subgroup effect (ETT), or a counterfactual (PN/PS).

2. **Construct or validate the causal DAG.** From the user's domain description or provided graph, build a directed acyclic graph with named nodes (variables) and directed edges (causal relationships). Explicitly state every assumption: "We assume X causes Y directly, and Z is a common cause of both X and Y." If the user provides data but no DAG, propose one based on domain knowledge and ask for confirmation.

3. **Identify confounders using the backdoor criterion.** For the treatment-outcome pair (X, Y), find a set Z such that: (a) no variable in Z is a descendant of X, and (b) Z blocks every backdoor path from X to Y (paths with an arrow into X). State the identified adjustment set explicitly.

4. **Check data sufficiency.** Verify that the observational data contains all conditional probabilities required by the adjustment formula. If any P(Y|X, Z) term cannot be computed from the data, report `LACK_CONDITION` with a specific explanation of what is missing, rather than guessing or approximating.

5. **Write the identification formula.** Express the causal quantity in terms of observational distributions using the appropriate formula:
   - ATE: `P(Y=1|do(X=1)) - P(Y=1|do(X=0))` = `Sum_z [P(Y=1|X=1,Z=z) - P(Y=1|X=0,Z=z)] * P(Z=z)`
   - ETT: `P(Y_{X=1}=1|X=1) - P(Y_{X=0}=1|X=1)` using backdoor adjustment conditioned on treated
   - NDE/NIE: Mediation formulas decomposing total effect into direct and indirect pathways
   - PN: `P(Y_{X=0}=0|X=1,Y=1)` — requires both observational and interventional data or monotonicity
   - PS: `P(Y_{X=1}=1|X=0,Y=0)` — dual of necessity

6. **Compute the numerical result.** Plug in the provided probabilities or data summaries. Show each arithmetic step. For binary variables, enumerate all values of the adjustment set. For continuous variables, describe the integral or suggest an estimation method (e.g., inverse probability weighting, regression adjustment).

7. **Interpret the result in domain terms.** Translate the numerical answer back into the user's domain language: "The average causal effect of the drug on recovery is 0.15, meaning treatment increases recovery probability by 15 percentage points after adjusting for age and severity."

8. **State assumptions and limitations.** Explicitly list: (a) causal sufficiency (no unmeasured confounders), (b) positivity (all subgroups have nonzero probability of treatment), (c) consistency (well-defined interventions), and any domain-specific assumptions encoded in the DAG.

9. **Validate robustness.** If the user provides enough data, suggest sensitivity analyses: What if there is an unmeasured confounder? How large would its effect need to be to nullify the result? Reference E-values or bounds where appropriate.

10. **Generate code if requested.** Produce Python code using `dowhy`, `pgmpy`, or `networkx` for DAG construction, identification, and estimation. Include the full pipeline from DAG specification to effect estimation.

## Concrete Examples

**Example 1: Estimating ATE with confounding**

User: "I have data on whether patients took a drug (D), their age group (A: young/old), and recovery (R). I think age affects both drug-taking and recovery. What is the causal effect of the drug on recovery? Here are the probabilities: P(R=1|D=1,A=young)=0.8, P(R=1|D=0,A=young)=0.6, P(R=1|D=1,A=old)=0.5, P(R=1|D=0,A=old)=0.3, P(A=young)=0.4, P(A=old)=0.6."

Approach:
1. Identify task: ATE of D on R.
2. DAG: A -> D, A -> R, D -> R. Age is a confounder.
3. Backdoor criterion: Z = {A} satisfies the criterion (A is not a descendant of D, and A blocks the backdoor path D <- A -> R).
4. Data sufficiency: All required conditional probabilities are provided. Proceed.
5. Formula: ATE = Sum_a [P(R=1|D=1,A=a) - P(R=1|D=0,A=a)] * P(A=a)
6. Compute: ATE = (0.8 - 0.6)(0.4) + (0.5 - 0.3)(0.6) = 0.08 + 0.12 = 0.20

Output:
```
Causal DAG: Age -> Drug, Age -> Recovery, Drug -> Recovery
Adjustment set: {Age}
ATE = [P(R=1|D=1,A=young) - P(R=1|D=0,A=young)] * P(A=young)
    + [P(R=1|D=1,A=old) - P(R=1|D=0,A=old)] * P(A=old)
    = (0.8 - 0.6)(0.4) + (0.5 - 0.3)(0.6)
    = 0.08 + 0.12
    = 0.20

Interpretation: The drug increases recovery probability by 20 percentage
points on average, after adjusting for age confounding.

Assumptions: No unmeasured confounders beyond age, positivity holds
(both age groups can receive or not receive the drug).
```

**Example 2: Detecting insufficient data**

User: "Does smoking (S) cause cancer (C)? I know P(C=1|S=1)=0.4 and P(C=1|S=0)=0.1. There might be a genetic factor (G) that affects both."

Approach:
1. Identify task: ATE of S on C with potential confounder G.
2. DAG: G -> S, G -> C, S -> C.
3. Backdoor criterion: Z = {G} is needed.
4. Data sufficiency check: We need P(C|S,G) and P(G), but only P(C|S) is provided.

Output:
```
Causal DAG: Genetics -> Smoking, Genetics -> Cancer, Smoking -> Cancer
Required adjustment set: {Genetics}

LACK_CONDITION: Cannot compute the causal effect.
Missing: P(C=1|S=1,G=g), P(C=1|S=0,G=g), and P(G=g) for each
value of the genetic factor.

The naive difference P(C=1|S=1) - P(C=1|S=0) = 0.3 is a biased
associational estimate that conflates the causal effect of smoking
with the confounding influence of genetics.

To proceed, you need either:
(a) Data on P(C|S,G) and P(G), or
(b) A randomized experiment removing the S<-G confounding path, or
(c) An instrument variable that affects S but not C except through S.
```

**Example 3: Probability of Necessity (counterfactual)**

User: "A patient took the drug (D=1) and recovered (R=1). Would they have NOT recovered if they hadn't taken the drug? Assume monotonicity (the drug never hurts). P(R=1|do(D=1))=0.7, P(R=1|do(D=0))=0.3."

Approach:
1. Identify task: PN — Probability of Necessity, P(R_{D=0}=0 | D=1, R=1).
2. Under monotonicity, PN = [P(R=1|do(D=1)) - P(R=1|do(D=0))] / P(R=1|do(D=1)).
3. Compute: PN = (0.7 - 0.3) / 0.7 = 0.571.

Output:
```
Task: Probability of Necessity
Question: Given the patient took the drug and recovered,
what is the probability they would NOT have recovered without it?

Under monotonicity assumption (drug never prevents recovery):
PN = [P(R=1|do(D=1)) - P(R=1|do(D=0))] / P(R=1|do(D=1))
   = (0.7 - 0.3) / 0.7
   = 0.571

Interpretation: There is a 57.1% probability that this patient's
recovery was causally due to the drug — i.e., they would not have
recovered without it.

Assumption: Monotonicity (no individual for whom the drug prevents
recovery). If monotonicity is violated, PN has wider bounds.
```

## Best Practices

**Do:**
- Always draw the causal DAG explicitly before computing anything, even if it seems obvious. The DAG is the source of all identification results.
- State the backdoor adjustment set and verify it satisfies both conditions (no descendants of treatment, blocks all backdoor paths).
- Report `LACK_CONDITION` honestly when data is insufficient rather than making unstated assumptions to fill gaps.
- Distinguish between associational quantities (correlations from data) and causal quantities (effects under intervention) in all explanations.
- Show the full algebraic computation with intermediate steps so the user can verify each substitution.

**Avoid:**
- Never conflate correlation with causation — P(Y|X) is not P(Y|do(X)) unless there is no confounding.
- Never adjust for descendants of treatment (collider bias / M-bias). This is the most common DAG error.
- Never compute NDE/NIE without identifying a valid mediator set; these require cross-world assumptions that must be stated.
- Never present a single point estimate without noting the key assumptions required for it to be valid (causal sufficiency, positivity, consistency).

## Error Handling

- **Cyclic graph provided:** If the user's proposed causal structure contains cycles, explain that causal DAGs must be acyclic. Suggest temporal ordering or equilibrium models as alternatives.
- **Collider conditioning:** If the user conditions on a collider (common effect of treatment and outcome), warn that this introduces bias and explain the collider principle.
- **Ambiguous treatment/outcome:** If the user's question doesn't clearly specify which variable is the treatment and which is the outcome, ask for clarification before proceeding.
- **Continuous variables:** If variables are continuous, note that binary formulas don't apply directly. Suggest regression-based approaches (e.g., linear SCMs, inverse probability weighting) and offer to generate `dowhy` or `statsmodels` code.
- **Missing DAG knowledge:** If the user cannot specify a DAG, suggest constraint-based discovery (PC algorithm) or score-based methods (GES) as a preliminary step, noting that discovery from observational data alone has fundamental limits (Markov equivalence classes).

## Limitations

- This reasoning pipeline assumes the causal DAG is correctly specified. If the true causal structure differs from what is assumed, all downstream estimates may be wrong. Causal inference from observational data fundamentally requires untestable assumptions.
- Probabilities of necessity and sufficiency (PN, PS) require either experimental data or strong assumptions like monotonicity. Without these, only bounds (not point estimates) are identifiable.
- The pipeline works best with discrete/binary variables and complete conditional probability tables. For high-dimensional continuous data, estimation requires additional statistical methods beyond what symbolic reasoning provides.
- This skill does not replace domain expertise. The DAG must be justified by subject-matter knowledge, not by data alone. Always involve domain experts in DAG construction for consequential decisions.
- Sensitivity to unmeasured confounding is not automatically quantified. For critical applications, pair this analysis with formal sensitivity analysis methods (E-values, Rosenbaum bounds).

## Reference

Chen, J., Chen, S., & Lu, C. (2026). *Can Post-Training Transform LLMs into Causal Reasoners?* arXiv:2602.06337. [https://arxiv.org/abs/2602.06337](https://arxiv.org/abs/2602.06337)

Key takeaway: Structured decomposition of causal tasks into DAG construction, identification via backdoor/front-door criteria, and formula-based computation enables reliable causal reasoning. The CauGym framework and its seven tasks (ATE, CDE, ETT, NDE, NIE, PN, PS) provide the taxonomy. Code and GRPO-trained model at [https://github.com/OpenCausaLab/CauGym](https://github.com/OpenCausaLab/CauGym).