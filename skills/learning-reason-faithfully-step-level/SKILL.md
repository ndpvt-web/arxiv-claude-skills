---
name: "learning-reason-faithfully-step-level"
description: "Apply FaithRL's step-level faithfulness verification to multi-step reasoning tasks. Decomposes reasoning into individually verified steps, penalizes unsupported claims, and preserves valid partial derivations. Use when: 'verify each reasoning step', 'reduce hallucination in chain-of-thought', 'faithful step-by-step reasoning', 'audit my reasoning chain', 'step-level credit assignment', 'penalize unsupported reasoning steps'."
---

# FaithRL: Step-Level Faithfulness Maximization for Reasoning

This skill enables Claude to apply the FaithRL framework from Gui et al. (2026) to produce reasoning chains where every intermediate step is grounded in provided evidence. Instead of only checking whether a final answer is correct, this technique verifies each reasoning step independently, penalizes steps that introduce unsupported claims, and preserves credit for valid partial derivations even when the final answer fails. The result is reasoning that is both more trustworthy and more debuggable.

## When to Use

- When the user asks Claude to solve a multi-step reasoning problem and needs confidence that each step is justified (e.g., multi-hop QA, mathematical proofs, legal analysis)
- When building or reviewing a chain-of-thought pipeline and the user wants to detect and flag hallucinated intermediate steps
- When designing reward functions or evaluation rubrics for LLM reasoning outputs that go beyond binary correctness
- When the user asks to audit, verify, or score an existing reasoning chain step-by-step against source evidence
- When implementing RLVR (Reinforcement Learning with Verifiable Rewards) training pipelines and the user wants step-level rather than outcome-level supervision
- When the user is debugging why a model gets correct answers via spurious reasoning and wants to identify unfaithful steps

## Key Technique

**The problem with outcome-only evaluation.** Standard approaches evaluate reasoning by checking whether the final answer is correct. This creates a dangerous blind spot: a model can arrive at right answers through wrong reasoning (spurious correctness) or develop over-confidence by never being penalized for unsupported intermediate claims. FaithRL addresses this by formalizing a *faithfulness-maximization objective* where a reasoning trajectory is faithful if and only if the knowledge it uses matches exactly the evidence required by the query: `S_f = {trajectory | used_knowledge(trajectory) = required_evidence(query)}`.

**Geometric reward design.** FaithRL replaces flat reward signals with capability-dependent coefficients. Given a model's current correctness rate `y_0` and hallucination rate `x_0`, rewards are: `+y_0` for correct answers, `0` for false refusals, and `-x_0` for hallucinations. This naturally balances accuracy against safety proportional to the model's actual capability, preventing both over-confidence and over-conservativeness.

**Faithfulness-Aware Advantage Modulation (FAAM).** The core mechanism assigns credit at the step level using a verifier `V(step) = 1 if step uses only evidence-supported knowledge, else 0`. For positive-advantage trajectories (good outcomes), faithful steps get full reinforcement while unfaithful steps are discounted by factor `alpha`. For negative-advantage trajectories (bad outcomes), unfaithful steps take full penalty while faithful steps receive reduced penalty. This preserves valid partial reasoning even in failed trajectories, preventing the "throw the baby out with the bathwater" problem of outcome-level penalties.

## Step-by-Step Workflow

1. **Decompose the reasoning task into atomic steps.** Given a multi-step problem, break the required reasoning into a numbered sequence of discrete claims or deductions. Each step should make exactly one inferential move.

2. **Identify the evidence set for the query.** Enumerate all pieces of source evidence (facts, data, premises, context passages) that are provided or can be verified. Label this set `E(q)` — the required evidence for query `q`.

3. **For each reasoning step, extract the knowledge it uses.** Annotate what information each step relies on. Distinguish between: (a) knowledge drawn from the evidence set, (b) common/definitional knowledge, and (c) claims not grounded in any provided source.

4. **Apply the faithfulness verifier to each step.** Mark each step as *faithful* (`V=1`) if it uses only evidence-supported knowledge, or *unfaithful* (`V=0`) if it introduces unsupported claims. Be strict — a step that smuggles in an ungrounded assumption is unfaithful even if the assumption happens to be true.

5. **Score the trajectory using the geometric reward.** Classify the overall outcome as correct, false-refusal, or hallucination. Apply the geometric reward: reward correct answers proportionally to baseline correctness, penalize hallucinations proportionally to baseline hallucination rate.

6. **Apply FAAM to assign step-level credit.** For each step, compute its modulation weight:
   - If the trajectory has positive advantage (correct/good): faithful steps get weight `1.0`, unfaithful steps get weight `alpha` (e.g., `0.1`)
   - If the trajectory has negative advantage (wrong/hallucinated): unfaithful steps get weight `1.0`, faithful steps get weight `alpha`

7. **Preserve valid partial derivations.** When a reasoning chain fails at step N but steps 1 through N-1 are faithful, explicitly mark those early steps as *preserved* — they should not be penalized. Report them as reusable building blocks.

8. **Generate the faithfulness report.** Produce a structured output showing: each step's content, its faithfulness verdict, its modulation weight, the overall trajectory classification, and the aggregate faithfulness score (fraction of faithful steps).

9. **Suggest targeted corrections.** For each unfaithful step, identify what evidence is missing or what unsupported claim was introduced, and propose a grounded alternative that uses only the evidence set.

10. **Iterate if needed.** Re-run the reasoning with corrections applied. Verify that the new chain achieves both correctness and full faithfulness.

## Concrete Examples

**Example 1: Multi-hop question answering with evidence audit**

```
User: Given these facts:
  (1) The Eiffel Tower is in Paris.
  (2) Paris is the capital of France.
  (3) France is in Europe.
  Answer: "What continent is the Eiffel Tower on?" and verify each reasoning step.

Approach:
1. Evidence set E(q) = {fact1, fact2, fact3}
2. Decompose reasoning:
   Step 1: The Eiffel Tower is in Paris. [uses fact1] → V=1 (faithful)
   Step 2: Paris is in France. [uses fact2] → V=1 (faithful)
   Step 3: France is in Europe. [uses fact3] → V=1 (faithful)
   Step 4: Therefore, the Eiffel Tower is in Europe. [logical chain from steps 1-3] → V=1

Output:
  Answer: Europe
  Faithfulness: 4/4 steps faithful (100%)
  Trajectory: CORRECT
  No unsupported claims detected.
```

**Example 2: Detecting an unfaithful step in math reasoning**

```
User: Review this solution for errors in reasoning faithfulness:
  Problem: "If x + 3 = 7, what is x^2?"
  Step 1: x + 3 = 7, so x = 4.
  Step 2: 4 is a positive even number, so x^2 will be even.
  Step 3: x^2 = 4^2 = 16.
  Answer: 16

Approach:
1. Evidence set: {equation x + 3 = 7}
2. Step-level verification:
   Step 1: x = 7 - 3 = 4. [Direct algebra from premise] → V=1 (faithful)
   Step 2: "4 is positive even, so x^2 is even." [True but UNSUPPORTED —
           the problem doesn't ask about parity, and this claim doesn't
           derive from the evidence set. It introduces unnecessary
           ungrounded reasoning.] → V=0 (unfaithful)
   Step 3: x^2 = 16. [Direct computation from Step 1] → V=1 (faithful)

Output:
  Answer: 16 (CORRECT)
  Faithfulness: 2/3 steps faithful (67%)
  Trajectory: Correct but contains spurious reasoning
  FAAM weights (positive advantage):
    Step 1: weight=1.0 (faithful, reinforce)
    Step 2: weight=0.1 (unfaithful, discount)
    Step 3: weight=1.0 (faithful, reinforce)
  Recommendation: Remove Step 2 — it adds no derivational value and
  introduces an ungrounded observation.
```

**Example 3: Preserving valid partial derivations in a failed chain**

```
User: This reasoning chain reached a wrong answer. Which steps are salvageable?
  Question: "What is the GDP per capita of the country where Toyota is headquartered?"
  Context: "Toyota is headquartered in Toyota City, Japan. Japan's GDP is $4.2T.
            Japan's population is 125 million."
  Step 1: Toyota is headquartered in Japan. [from context]
  Step 2: Japan's GDP is $4.2 trillion. [from context]
  Step 3: Japan's population is about 100 million. [CONTRADICTS context: 125M]
  Step 4: GDP per capita = $4.2T / 100M = $42,000.
  Answer: $42,000

Approach:
1. Evidence: {HQ=Japan, GDP=$4.2T, population=125M}
2. Verification:
   Step 1: V=1 (faithful — matches context)
   Step 2: V=1 (faithful — matches context)
   Step 3: V=0 (unfaithful — context says 125M, step claims ~100M)
   Step 4: V=0 (unfaithful — built on unfaithful Step 3)

Output:
  Answer: $42,000 (INCORRECT — should be $33,600)
  Trajectory: HALLUCINATION
  FAAM weights (negative advantage):
    Step 1: weight=0.1 (faithful in failed chain — reduced penalty, PRESERVED)
    Step 2: weight=0.1 (faithful in failed chain — reduced penalty, PRESERVED)
    Step 3: weight=1.0 (unfaithful — full penalty)
    Step 4: weight=1.0 (unfaithful — full penalty)
  Salvageable steps: Steps 1-2 are correctly grounded.
  Correction: Replace Step 3 with "Japan's population is 125 million"
              then Step 4 becomes $4.2T / 125M = $33,600.
```

## Best Practices

- **Do:** Verify each step independently against the evidence set before assessing the chain as a whole. A correct final answer does not validate intermediate steps.
- **Do:** Use the geometric reward calibration — scale penalties to the model's or system's current capability level rather than using fixed reward magnitudes. This prevents over-correction.
- **Do:** Always preserve faithful prefix steps in failed chains. Discarding valid partial work wastes signal and makes debugging harder.
- **Do:** Distinguish between knowledge from the evidence set, common definitional knowledge, and truly unsupported claims. Not all external knowledge is hallucination — calibrate the verifier to the task's evidence scope.
- **Avoid:** Treating faithfulness as binary for the whole chain. The entire point of FAAM is step-granular credit; collapsing to chain-level loses the key advantage.
- **Avoid:** Setting `alpha=0` (completely ignoring unfaithful steps in positive trajectories or faithful steps in negative trajectories). A small `alpha` like 0.1 maintains gradient signal for all steps while still strongly differentiating faithful from unfaithful.
- **Avoid:** Applying this framework to single-step or trivial reasoning where decomposition adds no value. The overhead of step-level verification is only justified when reasoning is genuinely multi-step.

## Error Handling

- **Ambiguous step boundaries:** If reasoning is written as prose without clear steps, decompose it into atomic inferential moves before verification. Each sentence that introduces a new claim or deduction becomes a step.
- **Missing evidence set:** If the user doesn't provide explicit evidence, define the evidence scope upfront (e.g., "the provided passage," "the database schema," "standard mathematical axioms") and document what counts as grounded.
- **Verifier disagreement:** When it's unclear whether a step is faithful, flag it as `V=uncertain` and explain the ambiguity rather than forcing a binary verdict. Let the user decide the evidence boundary.
- **Cascading unfaithfulness:** If an early step is unfaithful and later steps depend on it, mark all dependent steps as unfaithful too, but note which ones would be faithful *if* the upstream step were corrected.

## Limitations

- Step-level faithfulness verification requires a well-defined evidence set. For open-ended creative or exploratory tasks where "evidence" is not clearly bounded, this framework adds friction without clear benefit.
- The geometric reward design assumes measurable baseline correctness and hallucination rates. For novel tasks without historical performance data, fall back to uniform reward weighting.
- FAAM assumes a reliable step-level verifier. If verification itself is noisy or unreliable, the modulation weights may misattribute credit. In practice, verification quality degrades for domain-specific or highly technical reasoning where grounding is hard to assess.
- This approach optimizes for faithfulness to *provided* evidence. It does not verify whether the evidence itself is true — garbage in, faithfully reasoned garbage out.

## Reference

**Paper:** Gui, R., Li, Y., Qu, X., Liu, Z., & Cheng, Y. (2026). "Learning to Reason Faithfully through Step-Level Faithfulness Maximization." arXiv:2602.03507v1. [https://arxiv.org/abs/2602.03507v1](https://arxiv.org/abs/2602.03507v1)

**What to look for:** Section 4 for the faithfulness-maximization objective and Theorem 4.1 (over-confidence mitigation proof); Section 4.2 for the geometric reward derivation; Section 4.3 for the FAAM mechanism and modulation formula; Table 1 for benchmark results showing hallucination reduction with maintained correctness.

**Code:** [https://github.com/aintdoin/FaithRL](https://github.com/aintdoin/FaithRL)