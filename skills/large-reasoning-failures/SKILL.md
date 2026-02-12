---
name: "large-reasoning-failures"
description: "Detect, diagnose, and mitigate LLM reasoning failures in code and AI pipelines using the taxonomy from Song et al. (TMLR 2026). Use when: 'audit my LLM pipeline for reasoning failures', 'why is my model getting this wrong', 'add guardrails for LLM reasoning', 'check for reversal curse or compositional failures', 'harden my prompt against reasoning bugs', 'debug flaky LLM outputs'."
---

# LLM Reasoning Failure Detection and Mitigation

This skill equips Claude to systematically identify, classify, and fix reasoning failures in LLM-powered systems using the comprehensive taxonomy from Song, Han & Goodman (TMLR 2026). Rather than ad-hoc debugging, it applies a structured framework that distinguishes fundamental architectural failures, application-specific limitations, and robustness issues across informal (intuitive), formal (logical), and embodied reasoning — then selects targeted mitigation strategies for each failure class.

## When to Use

- When a user reports inconsistent or incorrect LLM outputs and needs root-cause analysis
- When building or reviewing prompt chains, agent systems, or RAG pipelines that involve multi-step reasoning
- When the user asks to audit an LLM application for known failure modes before deployment
- When debugging why an LLM fails on simple tasks it "should" get right (counting, reversal, basic logic)
- When designing evaluation suites or adversarial tests for LLM-based features
- When hardening prompts against order bias, framing effects, or compositional breakdowns
- When reviewing multi-agent system designs for communication and coordination failure risks

## Key Technique

The paper's core contribution is a **two-axis taxonomy** for reasoning failures. The first axis classifies the *type* of reasoning: informal (cognitive biases, theory of mind, social reasoning), formal (logical inference, arithmetic, compositional reasoning), or embodied (spatial, physical). The second axis classifies the *nature* of the failure: fundamental (architectural — affects all tasks), application-specific (domain-bound), or robustness (inconsistent across minor input variations). This cross-classification turns vague "the model is wrong" complaints into precise diagnoses with known mitigations.

The actionable insight is that most LLM reasoning bugs map to a small set of well-studied failure patterns, each with distinct root causes and targeted fixes. For example, the **reversal curse** (trained on "A is B" but failing on "B is A") stems from unidirectional training objectives and is mitigated by data augmentation with reversed formulations. **Compositional reasoning failures** (solving each step but failing the chain) trace to shallow planning and respond to Chain-of-Thought decomposition or graph-structured reasoning. **Order bias** in multiple-choice contexts comes from causal masking in Transformers and is countered by shuffling and majority-vote ensembles.

The practical power is in the **mitigation strategy taxonomy**: prompting fixes (CoT, step-by-step decomposition), data-centric fixes (augmentation, debiasing), in-processing fixes (adversarial training), post-processing fixes (output filtering, verification agents), architectural fixes (neuro-symbolic augmentation), and system-design fixes (inspector agents, structured communication protocols). Matching the right fix category to the diagnosed failure class avoids wasted effort.

## Step-by-Step Workflow

1. **Collect failure evidence.** Gather the specific inputs, expected outputs, and actual LLM outputs that demonstrate the problem. Note whether failures are consistent or sporadic across minor input variations.

2. **Classify the reasoning type.** Determine whether the task involves informal reasoning (intuition, social understanding, bias-sensitive decisions), formal reasoning (logic, math, code, compositional inference), or embodied reasoning (spatial, physical, navigation). A single task may span multiple types.

3. **Classify the failure nature.** Determine whether the failure is fundamental (appears across many tasks and prompts — e.g., counting errors, reversal curse), application-specific (only in this domain — e.g., legal syllogism failures), or a robustness issue (correct on one phrasing, wrong on a trivially rephrased version).

4. **Match to a known failure pattern.** Cross-reference the evidence against these high-frequency patterns:
   - **Reversal curse**: Model knows "A is B" but fails "B is A"
   - **Compositional breakdown**: Solves sub-problems individually, fails the chain
   - **Order/anchoring bias**: Output changes when option order or initial framing changes
   - **Counting/arithmetic errors**: Fails character-level or digit-level operations
   - **Theory of mind failure**: Cannot track beliefs of different agents/users
   - **Confirmation bias**: Latches onto the first plausible answer, ignores contradictions
   - **Framing effect**: Different wording of equivalent problems yields different answers
   - **Working memory overload**: Degrades as context length or variable count increases

5. **Identify root causes.** Map the failure pattern to its architectural or training root cause:
   - Tokenization artifacts (counting, character-level tasks)
   - Causal masking / unidirectional attention (reversal curse, order bias)
   - Next-token prediction mismatch with multi-step planning (compositional failures)
   - Training data bias inheritance (cognitive biases, framing effects)
   - RLHF surface-level alignment (moral reasoning inconsistencies)

6. **Select mitigation strategy by risk tier.**
   - *Low-risk*: Prompt engineering — add CoT, decompose into explicit sub-steps, shuffle options, request self-verification
   - *Medium-risk*: Prompt engineering + verification layer — use a second LLM call or rule-based checker to validate reasoning steps
   - *High-risk*: System-level — inspector agents, ensemble voting, neuro-symbolic fallbacks for arithmetic, structured communication protocols for multi-agent systems

7. **Implement the fix in code.** Apply the selected mitigation directly to the prompt template, pipeline code, or system architecture. This means editing prompt strings, adding verification functions, restructuring agent communication, or integrating external tools.

8. **Build a regression test.** Create a minimal test set that captures the original failure plus perturbation variants (rephrased, reordered, scaled-up). Include both the failing case and known-good cases to detect regressions.

9. **Validate robustness.** Run the fix against the perturbation test set. Check that accuracy is consistent across minor variations — not just correct on the original input. Flag any remaining inconsistencies as residual robustness issues.

10. **Document the failure class and fix.** Add a code comment or doc entry referencing the specific failure pattern (e.g., "Mitigates reversal curse per Song et al. 2026, Section 4.1") so future maintainers understand why the mitigation exists.

## Concrete Examples

**Example 1: Debugging a RAG pipeline that fails on reverse lookups**

User: "My RAG system can answer 'What language did Guido van Rossum create?' but fails on 'Who created Python?' — why?"

Approach:
1. Classify: Formal reasoning, logic in natural language
2. Match pattern: **Reversal curse** — the retrieval + LLM pipeline encodes "Guido -> Python" but not "Python -> Guido"
3. Root cause: Unidirectional encoding in both the embedding model and the LLM
4. Mitigation (prompt-level): Rewrite the retrieval query to also search for reversed formulations; add a prompt instruction: "Consider both directions of any stated relationship"
5. Mitigation (data-level): Augment the knowledge base with bidirectional formulations of key facts

Output:
```python
# Before: single-direction retrieval
query = user_question  # "Who created Python?"

# After: bidirectional retrieval augmentation (mitigates reversal curse)
def augment_query_bidirectional(question: str) -> list[str]:
    """Generate reversed formulations to counter reversal curse.
    See: Song et al. 2026, Section 4.1 - Reversal Curse."""
    return [
        question,
        rephrase_as_reverse(question),  # "What did Python's creator do?" etc.
    ]

results = []
for q in augment_query_bidirectional(user_question):
    results.extend(retriever.search(q, top_k=3))
results = deduplicate_and_rerank(results)
```

**Example 2: Fixing inconsistent multi-choice evaluation scores**

User: "My LLM-as-judge gives different scores depending on which answer is listed first in the prompt. How do I fix this?"

Approach:
1. Classify: Informal reasoning, cognitive bias
2. Match pattern: **Order bias / anchoring bias** — Transformer causal masking creates positional preference
3. Root cause: Attention distribution skews toward earlier tokens; RLHF training reinforces position-dependent preferences
4. Mitigation: Shuffle-and-vote — evaluate each permutation and take majority vote

Output:
```python
import itertools
import collections

def evaluate_with_order_debiasing(
    question: str,
    answers: list[str],
    judge_llm,
    num_shuffles: int = 3,
) -> str:
    """Mitigate order bias by evaluating multiple permutations.
    See: Song et al. 2026, Section 3.1 - Cognitive Biases."""
    votes = collections.Counter()

    # Sample random permutations (or all if len(answers) is small)
    perms = list(itertools.permutations(range(len(answers))))
    selected = random.sample(perms, min(num_shuffles, len(perms)))

    for perm in selected:
        shuffled = [answers[i] for i in perm]
        prompt = format_judge_prompt(question, shuffled)
        choice_idx = judge_llm(prompt)  # returns index in shuffled list
        original_idx = perm[choice_idx]  # map back to original order
        votes[original_idx] += 1

    return answers[votes.most_common(1)[0][0]]
```

**Example 3: Hardening a multi-agent coding system against coordination failures**

User: "My multi-agent pipeline has a planner agent and a coder agent, but the coder keeps misinterpreting the planner's instructions on complex tasks."

Approach:
1. Classify: Informal reasoning (explicit social / multi-agent), plus formal reasoning (compositional)
2. Match patterns: **Theory of mind failure** (coder doesn't model planner's intent), **compositional breakdown** (long plans lose coherence), **working memory overload** (too much context)
3. Root cause: Next-token prediction lacks explicit reasoning depth for jointly conditioned objectives; no verification loop
4. Mitigation: Add structured communication protocol with verification phase and inspector agent

Output:
```python
# Before: direct hand-off
plan = planner_agent.generate_plan(task)
code = coder_agent.implement(plan)  # misinterprets on complex tasks

# After: structured protocol with verification (mitigates multi-agent
# coordination failures per Song et al. 2026, Section 3.3)
plan = planner_agent.generate_plan(task)

# Step 1: Coder summarizes understanding before coding
understanding = coder_agent.summarize_plan(plan)

# Step 2: Inspector verifies alignment
alignment = inspector_agent.check_alignment(
    original_plan=plan,
    coder_understanding=understanding,
)
if not alignment.is_aligned:
    plan = planner_agent.clarify(plan, alignment.discrepancies)
    # Loop until aligned (with max retries)

# Step 3: Implement with step-by-step decomposition
for step in plan.decomposed_steps:
    code = coder_agent.implement_step(step)
    verification = inspector_agent.verify_step(step, code)
    if not verification.passed:
        code = coder_agent.fix(code, verification.feedback)
```

## Best Practices

**Do:**
- Always classify failures along both axes (reasoning type + failure nature) before choosing a fix — the same symptom can have different root causes
- Test with perturbation variants, not just the original failing input — a fix that works on one phrasing but breaks on another is a robustness issue, not a real fix
- Use the minimum mitigation tier that addresses the failure — prompt-level CoT before adding verification agents, verification agents before architectural changes
- Add explicit decomposition (Chain-of-Thought, step-by-step) for any task requiring more than one reasoning hop

**Avoid:**
- Assuming scaling alone will fix reasoning failures — the reversal curse persists across model sizes, counting errors appear even in frontier reasoning models
- Applying a single mitigation strategy uniformly — order bias requires shuffling, compositional failures require decomposition, arithmetic errors require tool use; these are distinct fixes
- Ignoring robustness testing after a fix — many mitigations improve average accuracy but leave inconsistency intact
- Over-relying on RLHF or alignment tuning to fix logical reasoning — it addresses surface-level alignment but can amplify biases and inconsistencies

## Error Handling

| Symptom | Likely Failure Class | First Diagnostic Step |
|---------|---------------------|----------------------|
| Correct on simple inputs, wrong on composed versions | Compositional breakdown | Test each sub-problem independently; if all pass, the composition logic is the failure point |
| Correct on one phrasing, wrong on rephrased equivalent | Robustness / framing effect | Run 5+ rephrasings and check consistency rate |
| Consistently wrong on reverse lookups | Reversal curse | Test both "A is B" and "B is A" formulations explicitly |
| Numeric answers off by small amounts | Arithmetic / counting failure | Test with smaller numbers first to isolate tokenization vs. algorithm issues |
| Multi-agent system produces incoherent outputs | Coordination failure + working memory | Check if each agent performs correctly in isolation; if yes, the communication protocol is the failure point |
| Model "hallucinates" confidence on wrong answers | Confirmation bias / calibration failure | Add explicit "what could be wrong with this answer?" self-critique step |

When a failure doesn't map cleanly to one pattern, it is likely a compound failure — multiple patterns interacting. Decompose the task into smaller units and diagnose each independently.

## Limitations

- This framework is diagnostic and strategic, not a silver bullet. Some fundamental failures (e.g., tokenization-induced counting errors) require architectural changes that prompt engineering cannot fully address.
- The taxonomy is based on research through early 2026. Novel failure modes in future architectures may not fit existing categories.
- Robustness mitigations (shuffling, ensembling) increase latency and cost linearly with the number of variants tested. For latency-sensitive applications, use targeted mitigations rather than blanket approaches.
- Embodied reasoning failures are largely out of scope for text-only coding tasks. The framework is most actionable for informal and formal reasoning in software contexts.
- The mitigation effectiveness varies by model. A fix validated on one LLM may not transfer to another — always re-test after model swaps.

## Reference

**Paper:** Song, P., Han, P., & Goodman, N. (2026). "Large Language Model Reasoning Failures." *Transactions on Machine Learning Research (TMLR)*, with Survey Certification. [arXiv:2602.06176](https://arxiv.org/abs/2602.06176v1)

**Repository:** [github.com/Peiyang-Song/Awesome-LLM-Reasoning-Failures](https://github.com/Peiyang-Song/Awesome-LLM-Reasoning-Failures) — curated collection of 200+ papers on LLM reasoning failures, organized by the taxonomy above. Start with the README for a quick-reference map of failure types to relevant papers.

**Key sections to consult:** Section 3 (Informal Reasoning Failures) for bias and social reasoning issues; Section 4 (Formal Reasoning Failures) for logic, arithmetic, and compositional breakdowns; Section 5 (Embodied Reasoning) for spatial/physical tasks; Section 6 (Mitigation Taxonomy) for the strategy selection framework.