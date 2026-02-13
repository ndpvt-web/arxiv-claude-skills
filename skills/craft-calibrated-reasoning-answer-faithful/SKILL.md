---
name: "craft-calibrated-reasoning-answer-faithful"
description: "Apply CRAFT (Calibrated Reasoning with Answer-Faithful Traces) for multi-hop question answering with verified reasoning chains. Use when: 'answer a complex question from multiple documents', 'build a faithful multi-hop QA pipeline', 'reason over retrieved passages with citations', 'verify reasoning chain faithfulness', 'structured RAG with auditable traces', 'multi-step question decomposition with evidence grounding'"
---

# CRAFT: Calibrated Reasoning with Answer-Faithful Traces

This skill enables Claude to perform multi-hop question answering over retrieved documents using structured, auditable reasoning traces inspired by the CRAFT framework. Instead of generating a flat chain-of-thought that may hallucinate or contradict its own evidence, Claude decomposes complex questions into sub-questions, declares which documents it will cite, reasons step-by-step with explicit source attribution, and then extracts a final answer — all within a structured XML trace that can be mechanically verified for faithfulness.

## When to Use

- When the user provides multiple documents/passages and asks a question requiring synthesis across them
- When building a RAG pipeline that needs auditable, citation-grounded reasoning
- When answering multi-hop questions (e.g., "Who directed the film that won Best Picture the year X was born?")
- When the user needs to distinguish supporting evidence from distractors in a retrieved document set
- When implementing a QA system where reasoning-answer consistency must be verifiable
- When the user asks to decompose a complex question into sub-questions and chain the answers together
- When building evaluation harnesses for faithfulness of LLM-generated reasoning

## Key Technique

CRAFT addresses three failure modes in RAG-based multi-hop QA: **reasoning collapse** (where complex multi-hop chains degrade under noisy retrieval), **reasoning-answer inconsistency** (where the model produces a correct answer but its stated reasoning doesn't actually support it), and **loss of format control** (where free-form chain-of-thought drifts from the required structure).

The core insight is a **structured XML trace** with four components: `<plan>` (sub-question decomposition), `<gold_docs>` (pre-declared evidence boundary), `<reason>` (step-by-step reasoning with document citations), and `<answer>` (final extracted answer). This structure isn't decorative — each component enables a specific **faithfulness audit**. The plan-to-reason check verifies reasoning follows the declared sub-questions. The gold_docs-to-reason check ensures all citations fall within the declared evidence set. The reason-to-answer check confirms the answer is entailed by the reasoning. The grounding check verifies each claim is supported by the cited document text.

In the original paper, these audits serve as reward signals for GRPO-based reinforcement learning. For inference-time application, we use them as **self-verification steps**: generate the trace, then audit each dimension, and regenerate or flag components that fail consistency checks.

## Step-by-Step Workflow

1. **Receive the question and document set.** Collect the user's question `q` and retrieved documents `D = {d_1, d_2, ..., d_K}`. Number each document with an index for citation tracking.

2. **Decompose into sub-questions.** Break the multi-hop question into an ordered sequence of sub-questions, each addressing one reasoning hop. Output these in a `<plan>` block. For a 2-hop question, this typically yields 2 sub-questions; for bridge-type questions, the answer to sub-question 1 feeds into sub-question 2.

3. **Declare the evidence boundary.** Before reasoning, scan the documents and pre-declare which document indices contain supporting evidence in a `<gold_docs>` block. This forces explicit commitment to an evidence set before reasoning begins, preventing post-hoc rationalization.

4. **Reason step-by-step with citations.** In a `<reason>` block, address each sub-question in order. For each reasoning step, cite the specific document index (e.g., `[doc 3]`) and quote or paraphrase the relevant passage. Chain the intermediate conclusions: the output of step N becomes input context for step N+1.

5. **Extract the final answer.** In an `<answer>` block, state the final answer derived from the reasoning chain. This must be a direct consequence of the last reasoning step.

6. **Audit: Plan-to-Reason alignment.** Verify that every sub-question from the plan is addressed in the reasoning, in order, and no reasoning steps exist that don't correspond to a planned sub-question.

7. **Audit: Citation-to-Evidence grounding.** Verify that every document index cited in `<reason>` appears in the `<gold_docs>` set, and that at least one citation exists. Flag any citation to a document not in the declared evidence set.

8. **Audit: Reason-to-Answer entailment.** Verify the final answer in `<answer>` logically follows from the conclusion of the reasoning chain. If the answer contradicts or is unsupported by the reasoning, flag it.

9. **Audit: Claim-to-Source verification.** For each factual claim in `<reason>`, verify it is actually stated or supported in the cited document text. Flag any claim that the cited document doesn't support.

10. **Report results with confidence.** Present the answer along with the structured trace. If any audit fails, either regenerate the failing component or explicitly disclose the inconsistency to the user.

## Concrete Examples

**Example 1: Bridge-type multi-hop question**

User: Given these documents, answer: "What is the capital of the country where the creator of Python was born?"
- Doc 1: "Python is a programming language created by Guido van Rossum."
- Doc 2: "Guido van Rossum was born in Haarlem, Netherlands, on January 31, 1956."
- Doc 3: "The capital of the Netherlands is Amsterdam. The government seat is The Hague."
- Doc 4: "Haarlem is a city in North Holland, known for its medieval architecture."
- Doc 5: "Ruby was created by Yukihiro Matsumoto in Japan."

Approach:
```xml
<plan>
Subquestion 1: Who created Python?
Subquestion 2: In which country was the creator of Python born?
Subquestion 3: What is the capital of that country?
</plan>

<gold_docs>[1, 2, 3]</gold_docs>

<reason>
From document [1]: Python was created by Guido van Rossum.
From document [2]: Guido van Rossum was born in Haarlem, Netherlands.
Therefore the country is the Netherlands.
From document [3]: The capital of the Netherlands is Amsterdam.
</reason>

<answer>Amsterdam</answer>
```

Audit results:
- Plan→Reason: PASS (all 3 sub-questions addressed in order)
- Gold_docs→Reason: PASS (citations [1],[2],[3] all within declared set)
- Reason→Answer: PASS ("Amsterdam" follows from reasoning chain)
- Grounding: PASS (each claim verified against cited document text)

**Example 2: Comparison-type question with distractors**

User: From these passages, determine: "Are the directors of Inception and Interstellar the same person?"
- Doc 1: "Inception (2010) was directed by Christopher Nolan and stars Leonardo DiCaprio."
- Doc 2: "The Dark Knight (2008) was directed by Christopher Nolan."
- Doc 3: "Interstellar (2014) was directed by Christopher Nolan and features Matthew McConaughey."
- Doc 4: "Arrival (2016) was directed by Denis Villeneuve."

Approach:
```xml
<plan>
Subquestion 1: Who directed Inception?
Subquestion 2: Who directed Interstellar?
Subquestion 3: Are they the same person?
</plan>

<gold_docs>[1, 3]</gold_docs>

<reason>
From document [1]: Inception was directed by Christopher Nolan.
From document [3]: Interstellar was directed by Christopher Nolan.
Both films were directed by the same person, Christopher Nolan.
</reason>

<answer>Yes, both Inception and Interstellar were directed by Christopher Nolan.</answer>
```

Audit results:
- Plan→Reason: PASS
- Gold_docs→Reason: PASS (docs [2] and [4] correctly excluded as distractors)
- Reason→Answer: PASS
- Grounding: PASS

**Example 3: Audit failure detection and recovery**

User: "Which award did the university attended by the author of Harry Potter win in 2020?"
- Doc 1: "J.K. Rowling wrote the Harry Potter series."
- Doc 2: "J.K. Rowling studied at the University of Exeter."
- Doc 3: "The University of Edinburgh awarded Rowling an honorary degree."
- Doc 4: "The University of Exeter won a Queen's Anniversary Prize in 2015."

Initial (flawed) trace:
```xml
<plan>
Subquestion 1: Who authored Harry Potter?
Subquestion 2: Which university did they attend?
Subquestion 3: What award did that university win in 2020?
</plan>

<gold_docs>[1, 2, 4]</gold_docs>

<reason>
From document [1]: J.K. Rowling authored Harry Potter.
From document [2]: Rowling attended the University of Exeter.
From document [4]: The University of Exeter won a Queen's Anniversary Prize.
</reason>

<answer>Queen's Anniversary Prize in 2020</answer>
```

Audit results:
- Plan→Reason: PASS
- Gold_docs→Reason: PASS
- Reason→Answer: **FAIL** — Document [4] states the prize was in 2015, not 2020. The answer claims 2020 but the reasoning evidence says 2015.
- Grounding: **FAIL** — The claim "won in 2020" is not supported by the cited document.

Recovery: Disclose the inconsistency. The documents do not contain evidence of an award in 2020. Report: "Based on the available documents, I cannot find an award won by the University of Exeter in 2020. Document [4] mentions a Queen's Anniversary Prize in 2015. The question may reference information not present in the provided documents."

## Best Practices

**Do:**
- Always number documents explicitly so citations are unambiguous and mechanically verifiable.
- Declare `<gold_docs>` *before* writing `<reason>` — this prevents post-hoc cherry-picking of evidence.
- Run all four audit checks even when the answer seems obviously correct. Reasoning-answer inconsistency is most dangerous when the answer happens to be right.
- When a multi-hop question has more than 3 hops, consider whether intermediate sub-answers need their own mini-audit before feeding into the next hop.

**Avoid:**
- Do not cite a document in reasoning that wasn't declared in `<gold_docs>`. If you discover relevant evidence mid-reasoning, update the declaration first.
- Do not skip the plan step for "simple" questions — even 2-hop questions benefit from explicit decomposition since it prevents reasoning collapse.
- Do not generate the answer first and then backfill reasoning to justify it. The trace order (plan → evidence → reason → answer) is load-bearing, not cosmetic.
- Do not treat distractors as benign — noisy retrieved documents are the primary cause of reasoning-answer inconsistency. Explicitly excluding them in `<gold_docs>` is the defense.

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| **Reasoning collapse** (incoherent mid-chain) | Plan→Reason audit fails: reasoning steps don't follow sub-question order | Regenerate `<reason>` block, following `<plan>` step by step |
| **Hallucinated citation** | Citation index not in `<gold_docs>` or references non-existent document | Remove the claim; check if answer still holds without it |
| **Answer not entailed** | Reason→Answer audit fails | Re-derive answer strictly from final reasoning step; if no valid answer follows, report "insufficient evidence" |
| **Unsupported claim** | Grounding audit fails for a specific claim | Remove or correct the claim; re-check downstream reasoning that depended on it |
| **Format violation** | XML tags missing or malformed | Regenerate the full trace; structured format is required for auditability |
| **No evidence found** | `<gold_docs>` is empty after scanning all documents | Report that the question cannot be answered from the provided documents |

## Limitations

- **Static document set only.** CRAFT operates on a pre-retrieved set of documents. It does not perform iterative retrieval (fetching new documents based on intermediate reasoning). For questions requiring information discovery, pair this with an iterative retrieval step upstream.
- **Faithfulness audits are heuristic at inference time.** Without the trained reward model from the paper, the four-dimension audit relies on Claude's own judgment, which may miss subtle inconsistencies it would also make during generation.
- **Scales poorly beyond ~5 hops.** The structured trace becomes unwieldy for very long reasoning chains. For 5+ hop questions, consider hierarchical decomposition (sub-plans within sub-plans).
- **Requires numbered, accessible documents.** The citation mechanism assumes documents are discrete, indexed units. It doesn't work well with a single long document or unstructured context.
- **Cannot fix retrieval failures.** If the retrieval step missed a critical document, no amount of faithful reasoning over the remaining set will produce the correct answer. The framework will correctly report "insufficient evidence," but the user still doesn't get their answer.

## Reference

**Paper:** [CRAFT: Calibrated Reasoning with Answer-Faithful Traces via Reinforcement Learning for Multi-Hop Question Answering](https://arxiv.org/abs/2602.01348v1) (Liu et al., 2026)

**Key takeaway:** The four-component XML trace (plan, gold_docs, reason, answer) combined with four-dimension faithfulness auditing (plan→reason, citation→evidence, reason→answer, claim→source) provides a mechanically verifiable structure that catches reasoning-answer inconsistencies — the most insidious failure mode where the answer is correct but the reasoning is fabricated.