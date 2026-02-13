---
name: "benchmarking-abap-code-generation"
description: "Generate syntactically correct and functional ABAP code using iterative compiler feedback loops. Applies the empirical methodology from Wallraven et al. (2026) to produce SAP ABAP classes that pass syntax checks and unit tests through up to 5 rounds of error-driven refinement. Trigger phrases: 'generate ABAP code', 'write ABAP class', 'fix ABAP syntax error', 'ABAP compiler feedback', 'SAP ABAP development', 'iterative ABAP correction'."
---

# Iterative Compiler-Feedback ABAP Code Generation

This skill enables Claude to generate high-quality SAP ABAP code by applying an iterative compiler feedback loop derived from the empirical benchmark study by Wallraven et al. The core insight: initial ABAP generation succeeds only ~24% of the time, but feeding compiler error messages back into subsequent generation attempts raises success rates to ~75% within 5 rounds. This skill encodes the prompt structure, error classification taxonomy, and iterative correction strategy that proved most effective across 180 benchmark tasks spanning string handling, list operations, mathematical calculations, logical conditions, and SAP database operations.

## When to Use

- When the user asks to generate an ABAP class, method, or function module from a natural language description
- When the user provides ABAP compiler errors (syntax errors, type mismatches, declaration errors) and asks for a corrected version
- When the user needs to translate a HumanEval-style algorithm problem into ABAP
- When the user wants to write ABAP unit tests or needs code that passes existing ABAP unit tests
- When the user is working with SAP internal tables, database operations, or business logic in ABAP
- When the user asks to convert code from another language (Python, Java, etc.) into ABAP
- When the user encounters SAP-specific issues like RETURNING vs EXPORTING parameter conflicts or Z-class naming conventions

## Key Technique: Iterative Compiler Feedback Loop

ABAP is a low-resource, proprietary language deeply embedded in the SAP ecosystem. LLMs have significantly less ABAP training data than languages like Python or JavaScript, which causes high initial error rates. The paper demonstrates that a structured feedback loop---where compiler diagnostics are fed back to the LLM for correction---dramatically improves output quality.

The workflow operates in up to 5 refinement rounds. In Round 0, the LLM generates ABAP code from the task description alone. If compilation or unit tests fail, the specific error message from the SAP server is appended to the prompt, and the LLM generates a corrected version. The empirical data shows the largest gains come in the first two rounds (initial ~24% jumps to ~42% after Round 1, ~53% after Round 2), with diminishing but still valuable returns through Round 5. The improvement curve has not fully flattened at 5 rounds, meaning additional iterations could yield further gains.

Error types fall into distinct categories that require different correction strategies: **class creation errors** (structural problems preventing the class from being created), **syntax errors** (type/conversion errors, declaration errors, lexical errors, structural issues), and **unit test failures** (code compiles but produces wrong results). The correction approach must differ: syntax errors benefit from targeted fixes to the specific line, while unit test failures often require rethinking the algorithm logic.

## Step-by-Step Workflow

1. **Constrain the generation environment.** Before writing any ABAP, establish the target system version (e.g., NetWeaver 7.57 / S/4HANA 2022), naming conventions (classes starting with `Z`), and parameter style (use `RETURNING` parameters, not `EXPORTING`). These constraints prevent an entire class of structural errors.

2. **Structure the output as a global ABAP class.** Generate a complete class with both `DEFINITION` and `IMPLEMENTATION` sections. Use a single public static method per class. This mirrors the format that SAP systems expect and avoids the most common class creation failures.

3. **Generate the initial ABAP code (Round 0).** Produce the code based solely on the task description. Use a low temperature (0.2) mentally---favor the most likely correct ABAP constructs over creative alternatives. Output only code, no explanations.

4. **Classify any compiler errors by phase and type.** When the user reports errors, determine: (a) Did the class fail to create? (b) Did it fail syntax checking? (c) Did it fail at unit test execution? Then sub-classify syntax errors as declaration errors, lexical errors, type/conversion errors, or structural errors. This classification drives the correction strategy.

5. **Apply targeted corrections based on error category.**
   - *Declaration errors* (most common): Check for undeclared variables, incorrect type names, or missing DATA statements. ABAP requires explicit declarations.
   - *Lexical errors*: Fix string literals (use single quotes `'...'`), check for invalid characters, ensure correct use of periods as statement terminators.
   - *Type/conversion errors*: Verify compatible types in assignments and method parameters. ABAP is strongly typed---implicit conversions that work in Python will fail here.
   - *Structural errors*: Ensure matching ENDMETHOD/ENDCLASS, correct section ordering (PUBLIC/PROTECTED/PRIVATE), and proper method signature syntax.

6. **Preserve working portions of the code.** When correcting, change only what the error message indicates is broken. Do not rewrite the entire class---this risks introducing new errors in previously correct sections.

7. **For unit test failures, re-examine the algorithm.** If the code compiles but tests fail, the logic is wrong, not the syntax. Re-read the task requirements, check edge cases (empty inputs, boundary values, type overflows), and verify that ABAP-specific behavior (0-based vs 1-based indexing, string operations) matches expectations.

8. **Iterate up to 5 rounds.** Each round should incorporate the latest error message. If the same error persists after 2 rounds, try a fundamentally different approach rather than making incremental fixes to the same structure.

9. **Validate ABAP-specific constructs.** Before finalizing, check: internal table operations use correct syntax (`APPEND`, `READ TABLE`, `LOOP AT`), SELECT statements use proper Open SQL syntax, numeric literals use periods as decimal separators, and method signatures match the expected interface.

10. **Output the final class as a single self-contained code block.** Include the complete `CLASS ... DEFINITION` and `CLASS ... IMPLEMENTATION` so the user can directly paste it into their SAP system.

## Concrete Examples

**Example 1: String Reversal Task**

User: Write an ABAP class that reverses a string. The class should be named ZCL_STRING_REVERSE with a static method REVERSE that takes a string and returns the reversed version.

Approach:
1. Generate a Z-prefixed global class with RETURNING parameter
2. Use ABAP string processing (STRLEN, substring access)
3. Structure as DEFINITION + IMPLEMENTATION

Output (Round 0):
```abap
CLASS zcl_string_reverse DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    CLASS-METHODS reverse
      IMPORTING iv_input TYPE string
      RETURNING VALUE(rv_result) TYPE string.
ENDCLASS.

CLASS zcl_string_reverse IMPLEMENTATION.
  METHOD reverse.
    DATA(lv_len) = strlen( iv_input ).
    DATA(lv_index) = lv_len - 1.
    WHILE lv_index >= 0.
      rv_result = rv_result && iv_input+lv_index(1).
      lv_index = lv_index - 1.
    ENDWHILE.
  ENDMETHOD.
ENDCLASS.
```

**Example 2: Iterative Correction from Compiler Error**

User: I tried your ABAP code and got this error: `"LV_RESULT" is not declared. Use a DATA statement.`

Approach:
1. Classify error: syntax error, sub-type declaration error
2. Identify the undeclared variable---ABAP inline declarations with `DATA(...)` require certain system versions; explicit DATA statement is safer
3. Fix only the declaration, preserve the rest

Corrected output (Round 1):
```abap
METHOD calculate_sum.
  DATA lv_result TYPE i.
  DATA lv_index TYPE i.
  lv_index = 0.
  WHILE lv_index < lines( it_numbers ).
    lv_index = lv_index + 1.
    READ TABLE it_numbers INDEX lv_index INTO DATA(lv_num).
    lv_result = lv_result + lv_num.
  ENDWHILE.
  rv_sum = lv_result.
ENDMETHOD.
```

**Example 3: SAP Database Operation Task**

User: Write an ABAP class that reads all entries from table SFLIGHT where the carrier ID is 'LH' and returns the total price as a decimal.

Approach:
1. This is a SAP-specific database task (one of the 16 practical scenarios in the benchmark)
2. Use Open SQL SELECT with WHERE clause
3. Use ABAP aggregate function or loop-based summation
4. Declare proper types matching SFLIGHT table structure

Output:
```abap
CLASS zcl_flight_price DEFINITION PUBLIC FINAL CREATE PUBLIC.
  PUBLIC SECTION.
    CLASS-METHODS get_total_price
      IMPORTING iv_carrid TYPE s_carr_id
      RETURNING VALUE(rv_total) TYPE s_price.
ENDCLASS.

CLASS zcl_flight_price IMPLEMENTATION.
  METHOD get_total_price.
    SELECT SUM( price ) FROM sflight
      INTO rv_total
      WHERE carrid = iv_carrid.
  ENDMETHOD.
ENDCLASS.
```

**Example 4: Unit Test Failure Correction**

User: The code compiles fine but the unit test fails. Expected output for input `[3, 1, 2]` is `[1, 2, 3]` but I'm getting `[3, 2, 1]`.

Approach:
1. Classify: unit test failure (not a syntax issue)
2. The algorithm is sorting in descending instead of ascending order
3. Fix the comparison logic in the sort routine, not the structure
4. Re-verify with the failing test case mentally before outputting

Correction: Reverse the comparison operator in the sorting condition from `>` to `<` (or swap the ascending/descending flag), keeping all declarations and class structure intact.

## Best Practices

- **Do:** Always include both the `DEFINITION` and `IMPLEMENTATION` sections---partial code causes class creation failures, the most catastrophic error type.
- **Do:** Use `RETURNING VALUE(...)` for method output parameters unless the user explicitly requires `EXPORTING`. This is the modern ABAP convention and avoids a common structural error.
- **Do:** Prefer explicit `DATA` declarations over inline `DATA(...)` when targeting older NetWeaver systems. Ask the user about their system version if uncertain.
- **Do:** Use the exact error message text to guide corrections. The SAP compiler produces precise diagnostics---match corrections to the specific error code and line.
- **Avoid:** Rewriting the entire class when only one line has an error. This is the most common mistake in iterative correction and often introduces regressions.
- **Avoid:** Using Python/Java idioms that don't exist in ABAP (list comprehensions, lambda expressions, ternary operators). ABAP has its own control flow and data manipulation syntax.
- **Avoid:** Generating explanatory comments or markdown around the code when the user needs compilable output. SAP systems reject anything outside valid ABAP syntax.

## Error Handling

**Class creation failures (most severe):** The entire class structure is malformed. Common causes: missing ENDCLASS, wrong section ordering, invalid class name. Response: regenerate the full class skeleton from scratch rather than patching.

**Syntax errors after successful class creation:** The class structure is valid but individual statements are wrong. Use the compiler's line number and error code to make surgical fixes. The most frequent sub-types:
- Type/conversion errors (55% of syntax errors for top models): Fix type mismatches in assignments and parameters
- Declaration errors (43% for some models): Add missing DATA statements or correct type names
- Lexical errors: Fix string literal quoting, decimal separator usage, or invalid identifiers

**Unit test failures with clean compilation:** The hardest category to fix because the error is logical, not syntactic. Re-read the task specification carefully, check boundary conditions, and verify that ABAP's specific behavior (1-based table indexing, string offset handling) matches expectations.

**Persistent errors after 2+ rounds:** If the same error recurs, the approach is fundamentally flawed. Rewrite the method body using a different algorithm rather than continuing to patch the same code.

## Limitations

- ABAP is a low-resource language in LLM training data. Complex or obscure ABAP constructs (ABAP CDS views, RAP business objects, BAdI implementations) may produce unreliable output even with iterative correction.
- The iterative feedback loop requires access to an actual SAP system compiler. Without real compiler output, Claude must simulate error detection, which is less reliable than actual compilation.
- The benchmark's 75% success rate ceiling (after 5 rounds) means roughly 1 in 4 tasks will not be solved. Tasks involving cross-system integration, complex ALV reports, or deep SAP framework knowledge are the most likely to fail.
- The technique works best for self-contained algorithmic tasks and single-method classes. Multi-class architectures, event-driven SAP programs (reports, dynpros), and enhancement framework code are outside the validated scope.
- SAP system versions vary significantly. Code valid on S/4HANA 2022 may fail on older ECC 6.0 systems due to missing language features (inline declarations, string templates, mesh types).

## Reference

Wallraven, S., Köhne, T., Westenberger, H., & Moser, A. (2026). *Benchmarking Large Language Models for ABAP Code Generation: An Empirical Study on Iterative Improvement by Compiler Feedback.* arXiv:2601.15188v1. Key takeaway: structured compiler feedback loops with up to 5 iterations raise ABAP code generation success from ~24% to ~75%, with the largest gains in the first two rounds.