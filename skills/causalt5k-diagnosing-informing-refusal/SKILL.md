---
name: "causalt5k-diagnosing-informing-refusal"
description: "Diagnose and correct causal reasoning failures in LLM outputs using the CausalT5K framework. Detects rung collapse (answering causal questions with mere correlations), sycophantic drift (abandoning correct answers under pressure), and generates Wise Refusals that specify missing evidence. Trigger phrases: 'diagnose causal reasoning', 'check for rung collapse', 'detect sycophancy in reasoning', 'wise refusal analysis', 'causal trap detection', 'audit causal claims in this output'"
---

This skill enables Claude to apply the CausalT5K diagnostic framework to real-world causal reasoning tasks. It operationalizes Pearl's Ladder of Causation (association, intervention, counterfactual) as a practical audit tool: classifying causal claims by rung, detecting when reasoning collapses to a lower rung than the question demands, identifying 10 specific causal trap families (selection bias, Simpson's paradox, reverse causation, etc.), and producing structured Wise Refusals that name the trap, specify the missing evidence, and explicitly decline unwarranted endorsement.

## When to Use

- When a user asks you to evaluate whether a causal claim is justified by the evidence presented (e.g., "Does this data prove X causes Y?")
- When reviewing code, data pipelines, or analytics that draw causal conclusions from observational data
- When a user presents a study, A/B test, or experiment and asks whether the causal interpretation is valid
- When building or reviewing LLM evaluation harnesses that test causal reasoning quality
- When a user pushes back on your answer with social or epistemic pressure and you need to audit whether your revised answer constitutes sycophantic drift
- When generating counterfactual explanations (e.g., "What would have happened if we had deployed version B?") and needing to verify structural soundness
- When designing prompts or evaluation rubrics that must distinguish correlation from causation

## Key Technique

**Pearl's Ladder as Diagnostic Infrastructure.** The core insight of CausalT5K is that causal reasoning failures are not random -- they cluster into diagnosable pathologies mapped to three rungs of Pearl's causal hierarchy. Rung 1 (Association) asks "What do I observe?" Rung 2 (Intervention) asks "What happens if I act?" Rung 3 (Counterfactual) asks "What would have happened if things had been different?" Rung collapse occurs when a model answers a Rung 2 or 3 question using only Rung 1 evidence -- for example, citing a correlation to justify an interventional recommendation. The framework provides a taxonomy of 10 "Wolf" traps (invalid designs: selection bias, survivorship bias, confounding, Simpson's paradox, reverse causation, post hoc fallacy, ecological fallacy, base rate neglect, healthy user bias, regression to mean) and 8 "Sheep" designs (valid: RCTs, natural experiments, instrumental variables, difference-in-differences, regression discontinuity, ablation studies, mechanism + dose gradients, lottery assignment).

**Two-Axis Decomposition Reveals Hidden Failures.** Aggregate accuracy masks dangerous asymmetries. CausalT5K decomposes performance into Utility (sensitivity: correctly endorsing valid causal claims) and Safety (specificity: correctly rejecting invalid traps). A model with 96% Safety but 40% Utility rejects 60% of legitimate causal claims -- a skepticism trap invisible to single-number metrics. Similarly, the Detection-Correction Gap (48-55% across model families) shows that models frequently detect a causal trap but fail to follow through with refusal, producing a diagnosis without a conclusion.

**The Wise Refusal Protocol.** Rather than binary accept/reject, the framework mandates a three-step response for underdetermined claims: (1) Classify the trap family, (2) State the pivotal question -- the specific missing information that would resolve the ambiguity, (3) Explicitly refuse to endorse the claim. This transforms refusal from evasion into actionable guidance.

## Step-by-Step Workflow

1. **Identify the causal rung demanded by the question.** Determine whether the user's query is associational ("Is X correlated with Y?"), interventional ("Will doing X cause Y?"), or counterfactual ("Would Y have been different if X hadn't happened?"). Tag the question with its rung.

2. **Identify the causal rung of the available evidence.** Examine the data, study design, or reasoning provided. Classify it: observational correlation (Rung 1), controlled experiment or quasi-experiment (Rung 2), or structural causal model with specified invariants (Rung 3).

3. **Check for rung collapse.** Compare the rung of the question to the rung of the evidence. If the question demands Rung 2 but the evidence is Rung 1, flag this as rung collapse. Name the specific gap (e.g., "This is an interventional question but the evidence is purely associational -- no controlled manipulation was performed").

4. **Scan for Wolf traps in the evidence.** Systematically check the 10 trap families: selection bias, survivorship bias, healthy user bias, regression to mean, ecological fallacy, base rate neglect, confounding, Simpson's paradox, reverse causation, post hoc fallacy. For each plausible trap, note which variables or design features trigger it.

5. **Check for Sheep validity if evidence looks sound.** If no traps are detected, verify against the 8 valid design patterns (RCT, natural experiment, IV, DiD, RDD, ablation, mechanism + dose gradient, lottery assignment). Confirm the design satisfies its own assumptions.

6. **Compute Utility vs. Safety posture.** Ask: Am I at risk of over-rejection (high Safety, low Utility -- dismissing valid claims)? Or under-rejection (high Utility, low Safety -- endorsing flawed claims)? Calibrate accordingly.

7. **If the claim is underdetermined, execute the Wise Refusal protocol.** (a) Name the trap family or evidential gap. (b) State the pivotal question -- the specific data, design feature, or analysis that would resolve the ambiguity. (c) Explicitly decline to endorse the causal claim.

8. **If the claim is justified, endorse with explicit reasoning.** State which valid design pattern applies, confirm rung alignment, and note any residual caveats.

9. **Resist sycophantic drift under pushback.** If the user challenges a refusal with social pressure ("I really think X causes Y") or epistemic pressure ("Rethink, I suspect you're wrong"), re-run steps 1-6 rather than capitulating. Only change the answer if the user provides new evidence or a valid structural argument, not mere disagreement.

10. **Structure the output.** Deliver results in the diagnostic format: Rung Classification, Trap/Design Identification, Evidence Assessment, Verdict (Endorse / Wise Refusal / Reject), and Pivotal Question if applicable.

## Concrete Examples

**Example 1: Detecting rung collapse in a business analytics claim**

User: "Our data shows that users who enable push notifications have 40% higher retention. Should we force-enable notifications for all users?"

Approach:
1. Question rung: Rung 2 (interventional -- "What happens if we act?")
2. Evidence rung: Rung 1 (observational correlation between notification setting and retention)
3. Rung collapse detected: Rung 2 question answered with Rung 1 evidence
4. Wolf trap scan: Healthy user bias (engaged users both enable notifications AND retain), reverse causation (retention causes notification use, not vice versa), confounding (power users differ on many dimensions)

Output:
```
## Causal Diagnosis

**Rung Classification:** Question is Rung 2 (interventional); evidence is Rung 1 (associational).
**Rung Collapse:** Yes. The 40% retention difference is observational -- no intervention was performed.

**Trap Identification:**
- Healthy user bias: Users who enable notifications are likely more engaged to begin with.
- Reverse causation: Higher retention may drive notification adoption, not the other way around.
- Confounding: Engaged users differ from disengaged users on many unmeasured dimensions.

**Verdict: Wise Refusal**
The available evidence does not support the interventional claim that forcing notifications will increase retention.

**Pivotal Question:** Run a randomized experiment where a random subset of new users has notifications force-enabled, then compare 30-day retention between the treatment and control groups. This eliminates self-selection and isolates the causal effect of the notification intervention itself.
```

**Example 2: Validating a well-designed A/B test**

User: "We ran an A/B test with random assignment on 50,000 users. The treatment group (new checkout flow) had a 3.2% higher conversion rate (p=0.003, 95% CI [1.1%, 5.3%]). Is this causal?"

Approach:
1. Question rung: Rung 2 (interventional)
2. Evidence rung: Rung 2 (randomized controlled experiment)
3. No rung collapse -- rungs are aligned
4. Wolf trap scan: Random assignment eliminates confounding and selection bias by design; sample size is sufficient; no survivorship bias indicated
5. Sheep design match: RCT pattern confirmed

Output:
```
## Causal Diagnosis

**Rung Classification:** Question and evidence are both Rung 2 (interventional). No rung collapse.

**Design Validation:** Randomized controlled trial (Sheep design).
- Random assignment addresses confounding and selection bias.
- N=50,000 provides adequate power.
- Statistical significance (p=0.003) with meaningful CI [1.1%, 5.3%].

**Verdict: Endorse**
The causal claim that the new checkout flow increases conversion by approximately 3.2% is supported by the experimental design.

**Caveats:** Verify there was no differential attrition between groups (would reintroduce selection bias). Confirm the randomization unit matches the analysis unit (no clustering effects). External validity depends on whether the test population is representative of the target deployment population.
```

**Example 3: Wise Refusal with counterfactual reasoning (Rung 3)**

User: "If we hadn't migrated to microservices last year, our outage rate would be the same as before."

Approach:
1. Question rung: Rung 3 (counterfactual -- "What would have happened?")
2. Evidence: Implicit comparison of pre/post migration outage rates (Rung 1)
3. Rung collapse: Rung 3 question with Rung 1 evidence
4. Wolf traps: Post hoc fallacy (timing coincidence), confounding (other infrastructure changes occurred simultaneously)
5. Counterfactual error check: Potential fatalism (assuming migration was the only causal path to current outage rates) or hallucination (inventing mechanisms without structural model)

Output:
```
## Causal Diagnosis

**Rung Classification:** Question is Rung 3 (counterfactual); evidence is Rung 1 (temporal association).
**Rung Collapse:** Yes -- two rungs of collapse.

**Trap Identification:**
- Post hoc fallacy: Outage rate changes may coincide with migration without being caused by it.
- Confounding: Traffic growth, team changes, monitoring improvements, and other infrastructure changes all occurred in the same period.

**Verdict: Wise Refusal**
The counterfactual claim cannot be evaluated from pre/post comparison alone.

**Pivotal Question:** To evaluate this counterfactual, specify (1) the structural causal model linking microservices architecture to outage rate, (2) which variables are held invariant in the counterfactual world (traffic, team size, monitoring), and (3) whether any comparable systems remained on the monolith during the same period (natural experiment). Without these, the counterfactual is underdetermined.
```

## Best Practices

- **Do:** Always tag the causal rung of both the question and the evidence before making any judgment. This single step catches the majority of reasoning failures.
- **Do:** Name specific trap families rather than giving vague warnings like "correlation isn't causation." Saying "this is survivorship bias because we only observe companies that succeeded" is actionable; a generic caveat is not.
- **Do:** State the pivotal question in every Wise Refusal. The goal is to tell the user exactly what evidence would resolve the ambiguity, turning a refusal into a research directive.
- **Do:** Separately assess Utility and Safety when reviewing your own reasoning. Ask: "Am I rejecting too many valid claims (skepticism trap)?" and "Am I endorsing too many flawed claims (credulity trap)?"
- **Avoid:** Collapsing to Rung 1 reasoning when the question demands Rung 2 or 3. If someone asks "What would happen if we do X?" do not answer with "X is correlated with Y."
- **Avoid:** Abandoning a correct refusal under social pressure. Re-evaluate the evidence structure, not the user's tone. Only revise if new evidence or a valid structural argument is presented.
- **Avoid:** Over-hedging on well-designed experiments. When an RCT with proper randomization shows a clear effect, endorse it. Excessive caveats on valid designs erode Utility without improving Safety.

## Error Handling

- **Ambiguous rung classification:** When the user's question can be interpreted at multiple rungs (e.g., "Does X affect Y?" could be associational or interventional), ask for clarification: "Are you asking whether X and Y are correlated in your data, or whether intervening on X would change Y?"
- **Multiple simultaneous traps:** When several Wolf traps are plausible, list all of them ranked by likelihood. State which trap is most damaging to the causal claim and which pivotal question would address the most traps simultaneously.
- **Insufficient context to diagnose:** If the user provides a claim without the underlying evidence or study design, request the specific details needed: sample selection method, control group, randomization procedure, potential confounders, and time ordering of variables.
- **Detection-Correction Gap in your own reasoning:** If you identify a trap but feel pressure to still endorse the claim, this is the gap the paper documents at 48-55% across all model families. Force yourself through step 3 of the Wise Refusal protocol -- the explicit refusal.

## Limitations

- This framework is designed for evaluating causal claims against evidence. It does not generate causal models from scratch or perform statistical analysis on raw data.
- The Wolf/Sheep taxonomy covers the 18 most common case types. Exotic causal fallacies (e.g., Berkson's paradox in specialized contexts, interference effects in network experiments) may require domain-specific extensions.
- The Wise Refusal protocol works best when the user provides enough context about the study design or data source. Vague claims without supporting evidence cannot be fully diagnosed -- only flagged as underdetermined.
- Counterfactual evaluation (Rung 3) requires an explicit or inferable structural causal model. Without one, the framework can identify the absence but cannot construct the model for the user.
- This diagnostic approach audits reasoning quality but does not replace domain expertise. The pivotal question may require subject-matter knowledge to answer.

## Reference

**Paper:** Geng, Ouyang, Wu, Barretto, Hayes. "CausalT5K: Diagnosing and Informing Refusal for Trustworthy Causal Reasoning of Skepticism, Sycophancy, Detection-Correction, and Rung Collapse." arXiv:2602.08939v1 (2026). Look for: the Sheep/Wolf taxonomy (Table 2), Four-Quadrant Control Landscape (Table 8), Detection-Correction Gap measurements (Table 10), and Wise Refusal scoring rubric (Section 4.3).