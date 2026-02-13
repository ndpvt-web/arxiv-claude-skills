---
name: "tracellm-leveraging-prompt-engineering"
description: "Establish and verify traceability links between software artifacts (requirements, design docs, test cases, regulations) using TraceLLM's prompt engineering framework. Trigger phrases: 'trace requirements to code', 'check traceability between artifacts', 'link requirements to test cases', 'verify requirement coverage', 'map design elements to requirements', 'trace link analysis'"
---

# TraceLLM: Requirements Traceability via Structured Prompt Engineering

This skill enables Claude to establish, verify, and analyze traceability links between software development artifacts — requirements, design documents, test cases, source code, and regulatory standards. It applies the TraceLLM framework's systematic prompt engineering methodology: assigning an expert role, injecting domain context, declaring artifact types, constraining the relationship query with precise language ("directly fulfills"), and enforcing binary output format. The technique consistently outperforms traditional information retrieval methods and fine-tuned models on traceability benchmarks across aerospace, healthcare, and general software domains.

## When to Use

- When the user asks to trace requirements to design elements, test cases, or code modules
- When verifying whether a set of test cases covers all specified requirements
- When mapping regulatory or compliance standards to implementation artifacts
- When performing impact analysis — identifying which artifacts are affected by a requirement change
- When auditing traceability matrices for completeness or correctness
- When the user provides two sets of software artifacts and asks "which ones are related?"
- When building or validating a Requirements Traceability Matrix (RTM)

## Key Technique

TraceLLM's core insight is that traceability performance depends critically on **prompt structure**, not just model capability. The framework builds prompts from four composable components: (1) a **role assignment** establishing expertise ("You are an expert in software traceability"), (2) **domain context** anchoring the system type ("artifacts from a healthcare system"), (3) **artifact type declarations** clarifying what is being compared ("(1) is a high-level requirement and (2) is a test case"), and (4) a **constrained relationship query** with precise semantics ("Does (2) directly fulfill (1)?"). The word "directly" proved critical — it improved precision significantly by discouraging transitive or indirect link inference.

For few-shot scenarios, the framework uses **label-aware diversity-based demonstration selection**. This means choosing example artifact pairs that (a) maintain balanced representation of positive and negative trace links (label-aware), and (b) maximize semantic diversity across the selected examples using embedding-based cosine dissimilarity (diversity-based). This combination prevents the model from biasing toward always predicting links exist (a common failure mode) while exposing it to the broadest range of artifact relationships. Demonstrations are embedded using sentence transformers and selected via greedy dissimilarity maximization.

The framework was validated through rigorous iterative refinement (7 prompt iterations, each changing a single variable) on a held-out validation set before final evaluation. Key findings: adding domain context and expert roles improved recall substantially; adding chain-of-thought reasoning ("explain your reasoning first") actually hurt performance by introducing noise into the binary classification. Temperature is set to 0.0 for deterministic outputs.

## Step-by-Step Workflow

1. **Identify the artifact types.** Classify each set of artifacts the user provides — requirements (high-level, low-level), design elements, test cases, source code units, regulatory clauses. This classification drives the prompt's artifact type declarations.

2. **Determine the domain context.** Identify the system domain (aerospace, healthcare, finance, embedded systems, web application, etc.). This gets injected into the prompt as domain anchoring context.

3. **Define the traceability relationship.** Clarify what "linked" means for this pair of artifact types. For requirements-to-design: "directly fulfills." For requirements-to-test-cases: "directly verifies." For requirements-to-regulations: "directly addresses." Use the word "directly" to prevent transitive inference.

4. **Structure each artifact pair for evaluation.** Format source and target artifacts as numbered items:
   ```
   (1) [SOURCE_ARTIFACT_TEXT]
   (2) [TARGET_ARTIFACT_TEXT]
   ```

5. **Compose the TraceLLM prompt.** Assemble the four components in order: role + domain + artifact types + relationship query + output constraint. Use this template:
   ```
   You are an expert in software traceability. You are given two artifacts
   from a [DOMAIN] system. (1) is a [SOURCE_TYPE] and (2) is a [TARGET_TYPE].
   Does (2) directly [RELATIONSHIP_VERB] (1)? Answer with only 'Yes' or 'No'.
   ```

6. **If few-shot examples are available, select demonstrations using label-aware diversity sampling.** Pick an equal number of positive (linked) and negative (unlinked) example pairs. Among candidates for each label, choose those that are most semantically dissimilar from each other to maximize coverage of the artifact space.

7. **Evaluate each candidate pair.** For each source-target combination, run the composed prompt and collect the binary verdict. Process pairs systematically — all targets for one source before moving to the next.

8. **Aggregate results into a traceability matrix.** Build a matrix with sources as rows and targets as columns. Mark cells as linked (Yes) or unlinked (No). Flag any source artifacts with zero links as potential coverage gaps.

9. **Assess confidence and flag uncertain links.** If running multiple passes (recommended for critical systems), repeat each pair evaluation 3-5 times. Flag pairs where verdicts are inconsistent across runs — these are candidates for human review.

10. **Present results for human validation.** Output the traceability matrix, highlight coverage gaps, and list uncertain links. TraceLLM is designed for semi-automated workflows where a human analyst reviews and validates the candidate links.

## Concrete Examples

**Example 1: Tracing requirements to test cases**

User: "I have 3 system requirements and 4 test cases. Which test cases trace to which requirements?"

Approach:
1. Classify artifacts: source = system requirements, target = test cases
2. Identify domain from content (e.g., embedded medical device)
3. For each of the 12 requirement-test pairs, compose and evaluate:

```
You are an expert in software traceability. You are given two artifacts
from a medical device system. (1) is a system requirement and (2) is a
test case. Does (2) directly verify (1)? Answer with only 'Yes' or 'No'.

(1) The system shall maintain patient heart rate readings within ±2 BPM accuracy.
(2) Test Case TC-003: Simulate heart rate input of 72 BPM via sensor mock.
    Verify displayed value is between 70 and 74 BPM. Expected: PASS.
```

Output:
```
Traceability Matrix:
              TC-001  TC-002  TC-003  TC-004
REQ-001        No      No      Yes     No
REQ-002        Yes     No      No      Yes
REQ-003        No      Yes     No      No

Coverage: All requirements have at least one linked test case.
No orphan test cases detected.
```

**Example 2: Mapping regulatory clauses to design elements**

User: "Check if our design document covers all the FDA 21 CFR Part 11 clauses we need to comply with."

Approach:
1. Classify: source = regulatory clauses, target = design elements
2. Domain: healthcare / FDA-regulated software
3. Relationship verb: "directly addresses"
4. Evaluate each regulation-design pair:

```
You are an expert in software traceability. You are given two artifacts
from an FDA-regulated healthcare system. (1) is a regulatory requirement
and (2) is a design element. Does (2) directly address (1)?
Answer with only 'Yes' or 'No'.

(1) 21 CFR 11.10(d): Limiting system access to authorized individuals.
(2) Design Element DE-Auth-03: Role-based access control module implementing
    user authentication via OAuth 2.0 with session timeout after 15 minutes
    of inactivity.
```

Output:
```
Regulatory Coverage Analysis:
  21 CFR 11.10(a) - Validation:        DE-Val-01, DE-Val-02     [COVERED]
  21 CFR 11.10(d) - Access Control:    DE-Auth-03               [COVERED]
  21 CFR 11.10(e) - Audit Trail:       (none)                   [GAP]
  21 CFR 11.10(g) - Authority Checks:  DE-Auth-03, DE-Auth-05   [COVERED]

WARNING: 1 regulatory clause has no traced design element.
  -> 21 CFR 11.10(e) requires human review and potential design updates.
```

**Example 3: Impact analysis for a requirement change**

User: "REQ-042 is changing. What design elements and test cases are affected?"

Approach:
1. Retrieve the text of REQ-042 and all design elements + test cases
2. Run TraceLLM pair evaluation for REQ-042 against every design element and test case
3. Report all linked artifacts as the impact set:

```
Impact Analysis for REQ-042 Change:

Affected Design Elements (3):
  - DE-107: Data encryption module (directly fulfills REQ-042)
  - DE-112: Key management service (directly fulfills REQ-042)
  - DE-115: Secure transport layer config (directly fulfills REQ-042)

Affected Test Cases (2):
  - TC-089: Encryption at-rest validation (directly verifies REQ-042)
  - TC-091: TLS handshake verification (directly verifies REQ-042)

Recommended Action: Review and potentially update all 5 artifacts above.
```

## Best Practices

- **Do:** Always include the word "directly" in the relationship query. This single word significantly reduces false positives from transitive or tangential relationships.
- **Do:** Set temperature to 0 (or as low as possible) for deterministic, reproducible trace link verdicts.
- **Do:** Constrain the output format to "Yes" or "No" only. Asking for explanations or confidence scores degrades classification accuracy.
- **Do:** When using few-shot examples, balance positive and negative link examples equally. Unbalanced demonstrations bias the model toward over- or under-predicting links.
- **Avoid:** Asking the model to "reason step by step" or "explain your thinking" before answering. The TraceLLM experiments showed chain-of-thought reasoning decreased recall for traceability classification.
- **Avoid:** Using generic prompts without domain context or artifact type declarations. The role + domain + artifact type components each independently improve performance.
- **Avoid:** Evaluating all pairs in a single massive prompt. Process one artifact pair per evaluation to maintain classification quality.

## Error Handling

- **Ambiguous artifact types:** If the user provides artifacts without clear type labels, ask them to clarify whether each set represents requirements, design elements, test cases, code, or regulations before proceeding.
- **Non-binary responses:** If the model returns anything other than "Yes" or "No" (e.g., "Partially" or "Maybe"), treat it as "No" for the matrix but flag the pair for human review with the full response text.
- **Inconsistent repeated runs:** If running multiple passes and a pair yields mixed results (e.g., 3 Yes, 2 No out of 5 runs), report it as "Uncertain" and include the vote distribution.
- **Large artifact sets:** For N sources and M targets, there are N*M pairs to evaluate. For sets larger than ~50x50, recommend the user prioritize by subsystem or process in batches. Report progress as you go.
- **Missing domain knowledge:** If the domain is unclear or highly specialized, ask the user for a one-sentence domain description to embed in the prompt rather than guessing.

## Limitations

- **Binary classification only.** TraceLLM determines whether a link exists or not — it does not quantify link strength, link type (satisfies, refines, derives-from), or partial fulfillment.
- **Pairwise evaluation cost.** The approach evaluates each source-target pair individually. For large artifact sets (hundreds of items), this creates O(N*M) evaluations which can be slow and expensive.
- **No structural reasoning.** The technique operates on textual content only. It cannot leverage dependency graphs, call trees, or architectural diagrams to infer links.
- **Domain-sensitive.** Performance varies across domains. The prompts should be adapted for each domain; a generic prompt will underperform a domain-contextualized one.
- **Semi-automated by design.** TraceLLM is explicitly intended to produce candidate links for human review, not to replace human judgment on traceability decisions in safety-critical systems.
- **Indirect links missed.** The "directly" qualifier intentionally excludes indirect or transitive relationships. If the user needs transitive traceability, a multi-hop approach is required.

## Reference

**Paper:** [TraceLLM: Leveraging Large Language Models with Prompt Engineering for Enhanced Requirements Traceability](https://arxiv.org/abs/2602.01253v1) — Alturayeif, Ahmad, Hassine (2026). Focus on Section 3 (methodology) for the prompt template structure, Table 3 for iterative refinement results, and Section 5 for demonstration selection strategy comparisons.