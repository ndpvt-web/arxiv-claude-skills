---
name: "smartoracle-agentic-approach"
description: "Agentic differential oracle for triaging cross-implementation discrepancies. Decomposes bug triage into specialized sub-agents (discrepancy finder, spec checker, false-positive critic, duplicate checker) that independently gather evidence and synthesize a verdict. Use when: 'triage differential test results', 'build a differential oracle', 'reduce false positives in cross-engine testing', 'agentic bug triage pipeline', 'analyze spec compliance differences', 'filter noise from differential fuzzing'."
---

# SmartOracle: Agentic Differential Oracle for Cross-Implementation Bug Triage

This skill enables Claude to build and apply an **agentic triage pipeline** for differential testing — where the same input is run across multiple implementations of a specification and behavioral differences must be classified as genuine bugs or benign noise. Instead of using a single monolithic prompt, the SmartOracle approach decomposes triage into specialized sub-agents that independently gather evidence from execution runs and specification documents, then synthesize a final REPORT/SKIP verdict. This architecture achieves higher recall, lower false-positive rates, and dramatically lower cost than sequential chain-of-thought baselines.

## When to Use

- When the user is **differential testing** multiple implementations of the same spec (e.g., JS engines, SQL databases, compilers, JSON parsers, crypto libraries) and needs to triage which output differences are real bugs vs. permitted variations.
- When the user asks to **build a test oracle** that classifies behavioral discrepancies against a specification document.
- When the user wants to **reduce false positives** from differential fuzzing campaigns that generate thousands of findings.
- When the user needs an **agentic pipeline** where sub-agents each handle one aspect of analysis (discrepancy extraction, spec lookup, duplicate detection, false-positive critique).
- When the user is working on **spec conformance testing** and needs to ground behavioral differences in authoritative specification language.
- When the user asks to **triage or prioritize bug reports** from automated testing tools that produce high volumes of noisy results.

## Key Technique

**Decomposed agentic triage replaces monolithic reasoning.** Traditional differential oracle construction is manual: an engineer inspects output differences, consults the specification, checks for duplicates, and decides whether to file a bug. A naive LLM approach sends all this context into one giant prompt — but this is slow, expensive, and loses accuracy as context grows. SmartOracle's core insight is that triage is naturally decomposable into independent evidence-gathering tasks that can be handled by specialized sub-agents, each with a focused prompt and limited scope.

**The architecture uses five sub-agents coordinated by an orchestrator.** (1) A **Discrepancy Finder** that normalizes behavioral differences across implementations, identifies the root divergence, and produces a minimal reproducing snippet. (2) A **Specification Checker** that queries the relevant spec sections to determine whether the observed behavior is mandated, permitted, or prohibited. (3) A **False-Positive Critic** that specifically looks for cases where differences fall within implementation-defined or undefined behavior in the spec. (4) A **Duplicate Checker** that compares current findings against previously-reported bugs using similarity matching. (5) A **Confidence Scorer** that assigns a quantitative confidence (0.0–1.0) based on evidence quality. The orchestrator sequences these agents, collects their outputs, and emits a final REPORT or SKIP verdict.

**Why this works better:** Each sub-agent uses a smaller, cheaper model (e.g., Gemini 2.5 Flash vs. Pro) with a tightly scoped prompt, reducing both token usage and hallucination surface. Independent evidence gathering means one agent's errors don't cascade. The original paper showed 4x faster analysis and 10x lower API cost versus a sequential single-model baseline, with recall improving from 0.68 to 0.84.

## Step-by-Step Workflow

1. **Define the specification and implementations under test.** Identify the authoritative specification document (e.g., ECMA-262 for JavaScript, SQL standard, RFC for a protocol) and the set of implementations to compare (minimum 2, ideally 3+). Cache or index the specification for fast section retrieval.

2. **Collect differential findings.** Run identical inputs across all implementations and capture stdout, stderr, and exit codes. Store results as structured records: `{input, impl_name, stdout, stderr, exit_code}`. Filter out cases where all implementations produce identical output or all fail identically.

3. **Cluster findings to reduce volume.** Group raw findings first by exit-code signature (which implementation crashed/succeeded), then apply TF-IDF encoding on output text with K-means clustering within each group. Select one representative finding per cluster to avoid redundant analysis.

4. **Deploy the Discrepancy Finder agent.** For each representative finding, invoke a sub-agent with a prompt scoped to: (a) extract and normalize the behavioral difference, (b) identify which implementation(s) diverge from the majority, (c) hypothesize a root cause, and (d) produce a minimal self-contained reproducing snippet. Output a structured discrepancy report.

5. **Deploy the Specification Checker agent.** Feed the discrepancy report to a sub-agent that queries the cached specification. The agent should retrieve the relevant spec sections (by keyword search or section index), quote the normative language, and classify the behavior as: MANDATED (spec requires this behavior), PERMITTED (spec allows variation), or PROHIBITED (spec forbids this behavior).

6. **Deploy the False-Positive Critic agent.** A dedicated adversarial agent reviews the discrepancy report and spec findings specifically looking for reasons to SKIP: undefined behavior clauses, implementation-defined behavior, host-environment dependencies, or locale-specific variations. This agent's job is to find reasons the difference is NOT a bug.

7. **Deploy the Duplicate Checker agent.** Compare the current finding's discrepancy summary against a stored database of previously-reported bugs using top-k similarity matching (k=10). If a match exceeds the similarity threshold, flag as duplicate with a reference to the existing report.

8. **Score confidence.** A scoring agent evaluates evidence quality: Was the spec section found and unambiguous? Did the reproducer confirm the discrepancy? Is the root cause well-understood? Assign a confidence score from 0.0 to 1.0.

9. **Synthesize the final verdict.** The orchestrator collects all sub-agent outputs and applies decision logic: REPORT if the spec checker says PROHIBITED, the false-positive critic found no exemption, the duplicate checker found no match, and confidence exceeds threshold (e.g., 0.7). Otherwise SKIP, with a reason code.

10. **Emit structured output.** Produce a triage report for each finding containing: verdict (REPORT/SKIP), confidence score, discrepancy summary, relevant spec sections quoted, reproducing snippet, and reasoning chain. Batch these into a prioritized list sorted by confidence.

## Concrete Examples

**Example 1: Triaging JavaScript engine differential test results**

```
User: I ran the same JS test suite across V8, SpiderMonkey, and JavaScriptCore.
      I have 200 cases where outputs differ. Help me triage which are real bugs.

Approach:
1. Parse the 200 differential findings into structured records.
2. Cluster by exit-code pattern (e.g., "V8 crashes, others succeed") then
   by TF-IDF similarity on stderr text. Reduce to ~30 representative clusters.
3. For each cluster representative, run the sub-agent pipeline:
   - Discrepancy Finder: "V8 throws TypeError on line 3 for
     `Reflect.construct(Date, [], undefined)`, while SM and JSC return a
     Date object."
   - Spec Checker: Queries ECMA-262 §26.1.2 (Reflect.construct) —
     "If newTarget is undefined, throw a TypeError" — V8 is correct.
   - False-Positive Critic: Confirms SM and JSC are non-conformant, not V8.
   - Duplicate Checker: No match in known bugs database.
   - Confidence: 0.92 (spec language is unambiguous, reproducer confirmed).
4. Verdict: REPORT — SpiderMonkey and JSC violate ECMA-262 §26.1.2.

Output:
┌─────────┬────────┬──────┬──────────────────────────────────────┐
│ Finding │ Verdict│ Conf │ Summary                              │
├─────────┼────────┼──────┼──────────────────────────────────────┤
│ CLU-007 │ REPORT │ 0.92 │ SM/JSC: Reflect.construct missing    │
│         │        │      │ TypeError for undefined newTarget     │
│ CLU-012 │ SKIP   │ 0.85 │ Locale-dependent toLocaleString diff │
│ CLU-019 │ REPORT │ 0.78 │ GraalJS: incorrect Array.from with   │
│         │        │      │ proxy handler (§23.1.2.1 violation)  │
│ CLU-031 │ SKIP   │ 0.90 │ Duplicate of CLU-007 (same root)     │
└─────────┴────────┴──────┴──────────────────────────────────────┘
```

**Example 2: Building a differential oracle for SQL database testing**

```
User: I'm differential testing PostgreSQL, MySQL, and SQLite with
      randomly generated SQL queries. How do I build an oracle to
      classify result differences?

Approach:
1. Define the reference specification: ISO SQL:2016 standard, plus each
   engine's documented extensions/deviations.
2. Structure findings as {query, pg_result, mysql_result, sqlite_result}.
3. Build the sub-agent pipeline adapted for SQL:
   - Discrepancy Finder: Normalize result sets (sort rows, cast types)
     before comparing. Extract the minimal query that triggers divergence.
   - Spec Checker: Query the SQL standard for the relevant clause
     (e.g., NULL handling in GROUP BY, implicit type coercion rules).
   - False-Positive Critic: Check each engine's documented deviations
     from the standard (e.g., MySQL's non-standard GROUP BY behavior
     is documented and intentional, not a bug).
   - Duplicate Checker: Match against known engine quirks databases.
4. Handle SQL-specific noise sources:
   - Non-deterministic ORDER BY (no ORDER BY = any order is valid)
   - Floating-point precision differences (within epsilon = SKIP)
   - Engine-specific type coercion (documented deviation = SKIP)

Output:
Verdict: REPORT — SQLite returns NULL for `NULLIF(0, 0)` while
PostgreSQL and MySQL correctly return NULL. Wait — all return NULL.
Revised: SKIP — All engines agree. Initial diff was caused by
trailing whitespace in output capture. Reproducer confirms identical
behavior.
```

**Example 3: Applying the pattern to compiler differential testing**

```
User: I'm fuzzing GCC and Clang with Csmith-generated C programs.
      Hundreds of cases produce different outputs. Help me triage.

Approach:
1. Reference spec: ISO C17 standard (N2176).
2. Cluster findings by: optimization level where divergence appears,
   warning/error category, and output value differences.
3. Sub-agent pipeline for each cluster:
   - Discrepancy Finder: Identify the specific construct causing
     divergence (e.g., signed integer overflow, sequence point
     violation, uninitialized variable read).
   - Spec Checker: Query C17 standard — is this undefined behavior (UB),
     unspecified behavior, or implementation-defined?
   - False-Positive Critic: If the program invokes UB, SKIP — both
     compilers are correct regardless of output. Use static analysis
     tools (UBSan, -fsanitize=undefined) to confirm UB presence.
   - Confidence: Lower confidence when UB detection is uncertain.
4. Only REPORT cases where: (a) the program is well-defined per C17,
   (b) one compiler produces incorrect output, and (c) the reproducer
   is minimal and confirmed.

Output:
Finding GCC-CLG-0042: REPORT (conf: 0.88)
  GCC -O2 miscompiles strict-aliasing case that is valid under
  C17 §6.5¶7 — union-based type punning. Clang produces correct
  output at all optimization levels. Minimal reproducer: 12 lines.
```

## Best Practices

- **Do:** Use at least 3 implementations when possible. Two implementations make it ambiguous which one is wrong; three or more enable majority-vote heuristics to identify the outlier.
- **Do:** Cache and index the specification document for fast retrieval. Sub-agents need to query specific sections, not read the entire spec each time.
- **Do:** Run the False-Positive Critic as a dedicated adversarial agent, not as a step in the main reasoning chain. Separation prevents confirmation bias.
- **Do:** Cluster findings before triaging. Analyzing 30 cluster representatives is tractable; analyzing 10,000 raw findings is not.
- **Avoid:** Sending all engine outputs plus the entire specification into a single prompt. This monolithic approach costs 10x more and produces lower recall than the decomposed pipeline.
- **Avoid:** Trusting a REPORT verdict without a confirmed minimal reproducer. Always verify the discrepancy is reproducible before filing.
- **Avoid:** Hardcoding noise patterns for a specific specification version. When the spec evolves, the Specification Checker agent should reference the updated document rather than relying on static rules.

## Error Handling

- **Specification section not found:** If the Spec Checker cannot locate a relevant section, lower confidence to ≤0.5 and flag the finding for manual review. Do not guess at spec intent.
- **Reproducer fails to trigger discrepancy:** If the minimal snippet does not reproduce the difference, mark as SKIP with reason "non-reproducible" — the original difference may have been caused by environment state, timing, or test harness artifacts.
- **Sub-agent disagreement:** If the Spec Checker says PROHIBITED but the False-Positive Critic finds an exemption clause, escalate to manual review rather than auto-deciding. Include both agents' reasoning in the output.
- **Duplicate checker false match:** When similarity matching returns a near-match but the root cause differs, the orchestrator should compare the specific spec sections cited. Same section = likely duplicate; different section = new finding.
- **Rate limiting or API failures:** The pipeline should be resilient to individual agent failures. If one sub-agent call fails, retry with exponential backoff. If a non-critical agent (e.g., Duplicate Checker) is unavailable, proceed without it and note the gap.

## Limitations

- **Specification coverage:** The approach works best when an authoritative, machine-queryable specification exists. For systems with informal or incomplete specs (e.g., browser DOM behavior beyond the standard), the Spec Checker agent has less to ground its reasoning in, and false-positive rates increase.
- **Undefined/unspecified behavior:** Specifications that designate large areas as undefined or implementation-defined (C, C++) produce many findings that are technically correct for all implementations. The pipeline will correctly SKIP these, but the high skip rate can mask real bugs hiding among them.
- **Novel bug patterns:** Sub-agents are prompted based on known triage patterns. Truly novel classes of bugs — especially those requiring deep semantic understanding of multiple interacting spec sections — may receive low confidence scores and require manual review.
- **Spec evolution lag:** When a specification is updated (e.g., new ECMAScript edition), the cached spec must be refreshed and agents may need prompt adjustments for new language features or changed semantics.
- **Cost at scale:** While 10x cheaper than monolithic approaches, the pipeline still incurs per-finding API costs. For fuzzing campaigns producing millions of findings, the clustering step is critical — without effective pre-filtering, costs become prohibitive.

## Reference

**Paper:** [SmartOracle — An Agentic Approach to Mitigate Noise in Differential Oracles](https://arxiv.org/abs/2601.15074v1) (Srinivasan, Menzies, D'Amorim, 2026). Look for: the five sub-agent architecture (§3), the orchestrator decision logic, the comparison showing 4x speed / 10x cost improvement over sequential baselines (§5), and the active fuzzing campaign that found confirmed bugs in V8, JavaScriptCore, and GraalJS (§6).