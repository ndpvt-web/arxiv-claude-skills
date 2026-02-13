---
name: "following-dragons-code-review-guided"
description: "Extract security-relevant signals from code review comments and translate them into fuzzer-guiding annotations using the EyeQ pipeline. Use when the user says 'guide fuzzing from code reviews', 'find dragons in review comments', 'annotate code for fuzzing', 'review-guided fuzzing', 'extract security signals from PRs', or 'instrument code for AFL++ from review discussions'."
---

This skill enables Claude to apply the EyeQ technique from "Following Dragons: Code Review-Guided Fuzzing" (Luu et al., 2026). EyeQ bridges the gap between developer intelligence expressed in code review discussions and automated fuzz testing. It extracts implicit security signals from review comments, classifies them against CWE-699 weakness categories, localizes the implicated code regions, and generates IJON annotations that steer AFL++ toward the program states developers flagged as fragile or security-critical. The technique found 40+ previously unknown bugs in the PHP interpreter, outperforming unguided fuzzing by 2.7x.

## When to Use

- When the user asks to analyze code review comments or PR discussions for security-relevant signals
- When the user wants to generate fuzzer annotations (IJON macros) from developer concerns expressed in reviews
- When the user has a codebase with AFL++ or libFuzzer and wants to prioritize fuzzing of developer-flagged risky regions
- When the user asks to classify code review comments against CWE categories
- When the user wants to map review-discussed concerns to specific functions and variables for targeted testing
- When the user asks to instrument C/C++ code with IJON annotations for annotation-aware fuzzing
- When the user is triaging a backlog of review comments to identify which warrant deeper security testing

## Key Technique

Standard fuzzers maximize code coverage but are blind to which paths developers consider most dangerous. Code reviewers routinely express security-relevant reasoning -- concerns about buffer boundaries, integer overflow, input validation gaps, resource exhaustion -- but this intelligence is trapped in natural language comments that testing tools ignore. EyeQ systematically harvests this intelligence and converts it into machine-consumable fuzzing guidance.

The core insight is a four-stage pipeline: (1) **Review Classification** uses a two-phase LLM approach -- coarse filtering identifies candidate CWE-699 categories, then fine-grained classification selects the most specific matching CWE. (2) **Code Localization** maps security-relevant comments to concrete functions via candidate ranking followed by diff-based verification against actual code changes. (3) **Code Instrumentation** identifies the specific variables or expressions within those functions that expose the security-critical state and wraps them in IJON annotation macros (`IJON_SET`, `IJON_MAX`, `IJON_MIN`). (4) **Fuzzing Execution** runs AFL++ with the IJON bitmap, treating annotations as first-class feedback signals that reward semantically meaningful executions even when coverage does not increase.

The annotation macros are lightweight and non-invasive. `IJON_SET(x)` reports a numeric program state to the fuzzer. `IJON_MAX(x)` and `IJON_MIN(x)` reward monotonic progress toward extreme values. `IJON_STATE()` captures discrete execution phases. All annotations are guarded with `#ifdef _USE_IJON` so they compile away in production builds. This means the technique requires no changes to program semantics or developer workflows.

## Step-by-Step Workflow

1. **Collect review comments.** Gather code review discussions from the target repository -- PR comments, inline review threads, commit-level discussions. Extract the comment text, the file path and diff context, and the reviewer's identity. Focus on comments that discuss behavior, correctness, or edge cases rather than style or formatting.

2. **Classify comments for security relevance (coarse pass).** For each comment, determine whether it expresses a security-relevant concern. Act as a senior application security engineer. Look for implicit signals: mentions of bounds, overflow, validation, allocation, parsing, trust boundaries, error handling, or resource limits. Map candidate comments to the CWE-699 taxonomy of 40 coding weakness categories. Prioritize recall -- it is better to flag a benign comment as potentially relevant than to miss a real signal.

3. **Refine CWE classification (fine-grained pass).** For comments that passed the coarse filter, select the most specific matching CWE entry. Prefer definition matches over keyword matches. For example, a comment about "large on-stack allocations pushing past the stack boundary" maps to CWE-1218 (Memory Buffer Errors) or CWE-787 (Out-of-Bounds Write), not merely "input validation." Output the CWE ID, the concrete phrase from the comment that triggered classification, and a confidence level.

4. **Localize implicated functions.** Identify which functions in the codebase are implicated by each security-relevant comment. Use two steps: (a) rank candidate functions by relevance to the review discussion, assigning HIGH/MEDIUM/LOW confidence, and (b) verify the localization by grounding it in the actual diff -- confirm the function appears in the changeset the review addresses. Do not invent function names; only select from functions present in the codebase.

5. **Identify annotation targets within functions.** For each localized function, identify the specific variables, expressions, or control-flow points that expose the security-critical state the reviewer was concerned about. Look for: parsed input values, buffer size calculations, loop bounds, allocation sizes, pointer arithmetic, return codes from validation functions, or state machine transitions. Select the top five candidates, providing verbatim code substrings as anchors.

6. **Generate IJON annotations.** For each annotation target, choose the appropriate IJON macro:
   - `IJON_SET(expr)` for values the fuzzer should explore diversely (e.g., parsed configuration values, hash table sizes)
   - `IJON_MAX(expr)` for values where pushing toward large extremes may trigger bugs (e.g., allocation sizes, recursion depths)
   - `IJON_MIN(expr)` for values where pushing toward zero or negative may trigger bugs (e.g., remaining buffer space, reference counts)
   - `IJON_STATE()` for capturing discrete phases in multi-step operations (e.g., state machine transitions in protocol parsers)

   Wrap all annotations in `#ifdef _USE_IJON` / `#endif` guards.

7. **Insert annotations into source code.** Place each annotation immediately after the statement that computes or assigns the target expression. Ensure the annotation does not alter control flow or data flow. Add a trailing comment explaining the rationale (e.g., `// TRACKS EXTREME VALUE`, `// MONITORS ALLOCATION BOUND`).

8. **Configure and execute the fuzzer.** Compile the instrumented code with AFL++ using the IJON-enabled bitmap. Set up the fuzzing harness targeting the entry points that exercise the annotated functions. Run the fuzzer with a time budget appropriate to the codebase size (24 hours is a reasonable starting point for medium-sized projects).

9. **Triage crashes against review context.** When the fuzzer produces crashes, cross-reference the crashing inputs against the original review comments. Determine whether the crash validates the reviewer's concern. Classify bugs by CWE and severity. Check whether the bug reproduces on the latest version of the codebase.

10. **Report findings with provenance.** For each confirmed bug, document: the original review comment that surfaced the concern, the CWE classification, the localized function, the annotation that guided discovery, the crashing input, and the root cause analysis. This provenance chain connects developer intuition to concrete vulnerability evidence.

## Concrete Examples

**Example 1: PHP Fiber Stack Overflow**

User: "This PR review comment says 'how should fibers guard against stack exhaustion?' and there's concern about 'large on-stack allocations pushing the stack pointer far beyond the stack boundary.' Help me generate fuzzing annotations."

Approach:
1. Classify: CWE-1218 (Memory Buffer Errors) -- reviewer explicitly discusses stack boundary violations
2. Localize: `OnUpdateFiberStackSize()` -- the function that parses user-provided INI values for fiber stack size
3. Identify target: `EG(fiber_stack_size)` -- the variable holding the parsed configuration value
4. Generate annotation:

```c
static ZEND_INI_MH(OnUpdateFiberStackSize) {
    if (new_value) {
        zend_long tmp = zend_ini_parse_quantity_warn(new_value, name);
        if (tmp < 0) {
            return FAILURE;
        }
        EG(fiber_stack_size) = tmp;
#ifdef _USE_IJON
        IJON_SET(EG(fiber_stack_size)); // TRACKS EXTREME VALUE
#endif
    }
    return SUCCESS;
}
```

Output: The fuzzer discovers that input `"9690x-D"` passes the negative check but produces a value triggering AddressSanitizer stack-overflow when a Fiber is started.

---

**Example 2: Hash Table Resource Exhaustion**

User: "A reviewer wrote 'this doesn't limit the number of hash collisions -- an attacker could craft keys to degrade to O(n) lookup.' I want to fuzz this."

Approach:
1. Classify: CWE-400 (Uncontrolled Resource Consumption) -- reviewer describes algorithmic complexity attack
2. Localize: `zend_hash_do_resize()` and `_zend_hash_add_or_update_i()` -- functions managing hash table growth
3. Identify targets: `ht->nNumUsed` (element count), `ht->nTableMask` (table size), collision chain length
4. Generate annotations:

```c
// In _zend_hash_add_or_update_i():
#ifdef _USE_IJON
    IJON_MAX(ht->nNumUsed);           // REWARD HIGH ELEMENT COUNTS
    IJON_MAX(chain_len);              // REWARD LONG COLLISION CHAINS
#endif
```

Output: Fuzzer steers toward inputs producing deep collision chains, uncovering a denial-of-service condition at scale.

---

**Example 3: Scanning a PR for Security Signals**

User: "Here are 15 review comments from our latest PR. Which ones are security-relevant and what should I fuzz?"

Approach:
1. Coarse classification pass over all 15 comments, flagging those mentioning bounds, allocation, parsing, trust, validation
2. Fine-grained CWE mapping for flagged comments
3. Output a ranked table:

```
| # | Comment Excerpt                              | CWE        | Confidence | Target Function         |
|---|----------------------------------------------|------------|------------|-------------------------|
| 3 | "no check on array index after realloc"      | CWE-787    | HIGH       | process_input_buffer()  |
| 7 | "what if the length field is negative?"       | CWE-190    | HIGH       | parse_header()          |
| 11| "race between free and callback invocation"  | CWE-416    | MEDIUM     | event_dispatch()        |
```

Then generate IJON annotations for each HIGH-confidence entry.

## Best Practices

- **Do:** Ground every CWE classification in a concrete phrase from the review comment. Quote the exact words that triggered the classification. This prevents hallucinated security signals.
- **Do:** Use the two-phase classification approach (coarse then fine-grained). Single-pass classification misses subtle signals and over-classifies benign comments.
- **Do:** Verify function localization against the actual diff. Reviewers often discuss concerns broadly, but the annotation must target the specific function in the changeset.
- **Do:** Prefer `IJON_SET` for configuration/input parsing values and `IJON_MAX` for size/count variables. The choice of macro determines how the fuzzer steers.
- **Avoid:** Annotating variables that are constant or immediately validated -- the fuzzer cannot meaningfully explore their state space.
- **Avoid:** Inserting annotations that change control flow (e.g., inside conditional expressions or as macro arguments that expand to statements with side effects).
- **Avoid:** Over-annotating. More than 5-7 annotations per function dilutes the fuzzer's attention. Prioritize the variables most directly tied to the reviewer's concern.

## Error Handling

- **Comment has no security relevance:** If the coarse classifier finds no CWE-699 match, skip the comment. Do not force-fit non-security concerns (style, performance, naming) into vulnerability categories.
- **Cannot localize to a function:** If the review comment discusses architecture or design rather than specific code, note it as "not localizable" and suggest manual review instead of annotation.
- **Annotation target is optimized away:** If the compiler eliminates the annotated expression, wrap the annotation in a `volatile` read or use `__attribute__((used))` to prevent dead-code elimination.
- **Fuzzer produces no crashes after extended run:** Re-examine whether the annotation exposes the right state. Consider whether the vulnerability requires multi-step setup that the harness does not provide. Add `IJON_STATE()` annotations at prerequisite checkpoints.
- **False positive crashes:** Cross-reference crashes against the review context. If the crash is in unrelated code, the annotation may be steering the fuzzer off-target. Narrow the annotation or add constraints.

## Limitations

- Requires an annotation-aware fuzzer (AFL++ with IJON support). Standard AFL or libFuzzer without modification cannot consume the annotations.
- Most effective on C/C++ codebases where IJON macros can be inserted. Applying to managed languages (Java, Python, Go) requires equivalent instrumentation frameworks, which may not exist.
- The technique depends on code reviews containing security-relevant discussion. Repositories with minimal or style-only reviews will yield few actionable signals.
- LLM-based classification achieves ~75% localization accuracy. Manual verification of high-confidence annotations is recommended before committing long fuzzing campaigns.
- Does not replace coverage-guided fuzzing -- it complements it. Run both annotated and unannotated fuzzing campaigns to cover both developer-flagged and unexpected paths.
- Review comments in languages other than English may reduce classification accuracy.

## Reference

**Paper:** "Following Dragons: Code Review-Guided Fuzzing" -- Luu, Pasdar, Charoenwet, Murray, Cohney (2026). arXiv:2602.10487v1. https://arxiv.org/abs/2602.10487v1

Look for: Figures 3-7 (the exact LLM prompt templates for each pipeline stage), Listing 4 (the annotated PHP fiber example), and Table III (the CWE classification accuracy breakdown by category).