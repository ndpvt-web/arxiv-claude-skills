---
name: "swe-agi-benchmarking-specification-driven-software"
description: "Build production-scale software strictly from formal specifications, RFCs, and standards documents using a scaffold-first, constraint-mapping workflow. Use when: 'implement this RFC', 'build a parser from this spec', 'construct a decoder from the standard', 'implement this protocol specification', 'build from this formal grammar', 'create a solver from these constraints'."
---

# Specification-Driven Software Construction

This skill enables Claude to build complete, production-scale software systems — parsers, interpreters, protocol implementations, binary decoders, SAT solvers — directly from formal specifications, RFCs, and authoritative standards. Based on the SWE-AGI benchmark methodology (Zhang et al., 2026), the core insight is that **specification fidelity, not general coding ability, determines success** in autonomous software construction. The workflow uses API scaffold design, explicit constraint mapping, and iterative specification-referenced validation to implement 1,000–10,000 lines of core logic that would otherwise take weeks of human engineering.

## When to Use

- When the user provides an RFC, formal specification, or standard document and asks to implement it (e.g., "implement RFC 9293 TCP state machine")
- When building a parser or interpreter from a formal grammar (BNF, EBNF, PEG)
- When implementing a binary format decoder from a specification (e.g., PNG, WASM, PDF)
- When constructing a protocol handler from an authoritative standard
- When building a constraint solver (SAT, SMT) from formal problem definitions
- When the user wants a clean-room implementation that must not rely on existing codebases, only on the specification itself
- When implementing any system where correctness is defined by adherence to a written standard, not by heuristic behavior

## Key Technique: Scaffold-First Specification Construction

The SWE-AGI research reveals that autonomous agents fail not because they misunderstand specifications, but because they fail to **simultaneously satisfy multiple interdependent constraints** during code generation. Constraint violations account for ~45% of failures, followed by type mismatches (25%) and missing edge cases (20%). Pure logic errors are only 10% — agents understand *what* to build but struggle with the combinatorial pressure of *building it all at once*.

The solution is a **scaffold-first, constraint-mapped, iteratively-validated** workflow. Before writing any implementation logic, define the complete type scaffold: all public types, module boundaries, function signatures, and error types derived directly from the specification. This scaffold acts as a structural contract that catches type mismatches early and decomposes the specification into independently implementable units.

The second critical insight from SWE-AGI is that **code reading, not writing, becomes the dominant bottleneck as codebases scale**. Agents that maintain persistent references back to the specification during error recovery outperform those that treat the spec as a one-time input by ~45%. Every validation failure must be mapped back to a specific specification clause, not debugged through generic reasoning.

## Step-by-Step Workflow

1. **Ingest and section the specification.** Read the full RFC, standard, or formal grammar. Break it into discrete numbered sections, each covering one functional requirement or constraint. Create a specification index mapping section numbers to the features they define.

2. **Extract all constraints into an explicit checklist.** Convert every "MUST", "SHALL", "REQUIRED" (and their negatives) from the specification into testable assertions. For formal grammars, enumerate every production rule. For binary formats, list every field, its offset, size, endianness, and valid value range. This checklist becomes your source of truth.

3. **Design the type scaffold from the specification structure.** Define all public types, enums, structs, and function signatures *before* writing any logic. Each type should map 1:1 to a concept in the specification. Define module boundaries that mirror the specification's own organizational structure (e.g., one module per RFC section, one type per grammar non-terminal).

4. **Define error types that reference specification clauses.** Create error variants that name the specific specification section they represent (e.g., `InvalidHeaderField { section: "4.3.2", detail: String }`). This forces error handling to be specification-aware, not generic.

5. **Write test cases directly from specification examples.** Before implementing logic, generate test cases from any examples, test vectors, or sample data in the specification. For RFCs, use the provided examples verbatim. For grammars, test both valid and invalid inputs at every production boundary. Aim for at least one test per constraint from step 2.

6. **Implement incrementally by specification section, easiest first.** Start with the simplest, least-dependent specification sections. Implement one section at a time, running the corresponding tests after each. Do not proceed to the next section until the current one passes all its tests.

7. **Cross-validate constraint interactions after each section.** After implementing a new section, re-run all previous tests. Specification constraints often interact (e.g., header parsing affects body decoding). When a previously passing test fails, map the failure to the specific constraint interaction and fix it before continuing.

8. **Re-consult the specification on every failure.** When a test fails, do not debug by reading only the implementation code. Go back to the specification, re-read the relevant section, and verify that the implementation matches clause-by-clause. The SWE-AGI finding is clear: agents that re-consult specs during error recovery achieve ~60% higher pass rates.

9. **Handle edge cases last, systematically.** After core logic passes, walk the constraint checklist from step 2 and verify edge cases: boundary values, optional fields, error conditions, and explicitly undefined behavior. Implement defensive handling for each.

10. **Final conformance sweep.** Run the complete test suite. For any remaining failures, trace each to its specification clause, fix, and document which clause it satisfies. The implementation is complete when every item in the constraint checklist from step 2 has a corresponding passing test.

## Concrete Examples

**Example 1: Implementing a JSON parser from RFC 8259**

```
User: Implement a JSON parser strictly following RFC 8259.

Approach:
1. Section the spec: structural characters (section 2), values (section 3),
   objects (section 4), arrays (section 5), numbers (section 6),
   strings (section 7).

2. Extract constraints:
   - Strings MUST use UTF-8 encoding
   - Numbers MUST NOT have leading zeros (except 0 itself)
   - Structural whitespace is only: SP, HT, LF, CR
   - Unicode escape: \uXXXX with surrogate pair handling
   - Max nesting: implementation-defined but must handle reasonable depth
   ... (enumerate all 30+ MUST/SHALL constraints)

3. Design scaffold:
   ```
   enum JsonValue {
     Null
     Bool(Bool)
     Number(Float64)
     Str(String)
     Array(List[JsonValue])
     Object(Map[String, JsonValue])
   }

   enum ParseError {
     UnexpectedChar { pos: Int, expected: String, section: String }
     InvalidNumber { pos: Int, detail: String, section: String }
     InvalidString { pos: Int, detail: String, section: String }
     UnexpectedEOF { section: String }
   }

   fn parse(input: String) -> Result[JsonValue, ParseError]
   fn parse_value(input: String, pos: Int) -> Result[(JsonValue, Int), ParseError]
   fn parse_string(input: String, pos: Int) -> Result[(String, Int), ParseError]
   fn parse_number(input: String, pos: Int) -> Result[(Float64, Int), ParseError]
   fn parse_array(input: String, pos: Int) -> Result[(List[JsonValue], Int), ParseError]
   fn parse_object(input: String, pos: Int) -> Result[(Map[String, JsonValue], Int), ParseError]
   fn skip_whitespace(input: String, pos: Int) -> Int
   ```

4. Write tests from RFC examples:
   - `parse("null")` -> Ok(Null)
   - `parse("[1, \"two\", true]")` -> Ok(Array(...))
   - `parse("{\"key\": 42}")` -> Ok(Object(...))
   - `parse("01")` -> Err(InvalidNumber, section "6")
   - `parse("\"\\uD800\\uDC00\"")` -> Ok(Str(...)) // surrogate pair

5. Implement section by section: whitespace -> null/bool ->
   numbers -> strings -> arrays -> objects.

Output: A fully conformant RFC 8259 parser with every MUST constraint
tested and every error referencing its specification section.
```

**Example 2: Building a WASM binary decoder from the WebAssembly spec**

```
User: Build a WebAssembly binary format decoder per the WASM Core Spec section 5.

Approach:
1. Section the spec: module structure (5.1), types (5.2), imports (5.3),
   functions (5.4), tables (5.5), memories (5.6), globals (5.7),
   exports (5.8), code (5.9), data (5.10), custom sections.

2. Extract constraints:
   - Magic number: 0x00 0x61 0x73 0x6D
   - Version: 0x01 0x00 0x00 0x00
   - LEB128 unsigned/signed integer encoding
   - Section IDs: 0-12, custom=0, type=1, import=2, ...
   - Sections MUST appear in order (except custom sections)
   - Each section: id (1 byte) + size (u32 LEB128) + contents
   ... (enumerate 100+ constraints from spec)

3. Design scaffold:
   ```
   // Types mirror spec section 2.3
   struct Module { types, funcs, tables, mems, globals, ... }
   enum ValType { I32, I64, F32, F64 }
   struct FuncType { params: List[ValType], results: List[ValType] }

   // Decoder mirrors spec section 5
   struct Decoder { bytes: Bytes, pos: Int }
   fn decode_module(bytes: Bytes) -> Result[Module, DecodeError]
   fn decode_section(d: Decoder) -> Result[Section, DecodeError]
   fn decode_leb128_u32(d: Decoder) -> Result[Int, DecodeError]
   fn decode_leb128_s32(d: Decoder) -> Result[Int, DecodeError]
   // ... one function per spec subsection
   ```

4. Tests from spec test suite: use known .wasm binaries,
   validate each section independently, test malformed inputs.

5. Implement: magic/version -> LEB128 -> type section ->
   import section -> ... -> code section (most complex).

Output: A streaming WASM decoder where each DecodeError references
the specific spec section violated, enabling precise diagnostics.
```

**Example 3: Implementing a SAT solver from DIMACS CNF specification**

```
User: Build a DPLL-based SAT solver that reads DIMACS CNF format.

Approach:
1. Section the spec: DIMACS CNF format (header line, clause lines,
   comment lines), DPLL algorithm definition.

2. Extract constraints:
   - Header: "p cnf <variables> <clauses>"
   - Clauses: space-separated integers, 0-terminated
   - Negative integers = negated variables
   - Comments: lines starting with 'c'
   - Output: SATISFIABLE + assignment, or UNSATISFIABLE

3. Design scaffold:
   ```
   struct Formula { num_vars: Int, clauses: List[Clause] }
   type Clause = List[Literal]
   struct Literal { var_id: Int, negated: Bool }
   enum SatResult { Sat(Map[Int, Bool]), Unsat }

   fn parse_dimacs(input: String) -> Result[Formula, ParseError]
   fn solve(formula: Formula) -> SatResult
   fn unit_propagate(formula: Formula, assignment: Assignment) -> PropResult
   fn choose_variable(formula: Formula, assignment: Assignment) -> Int
   fn dpll(formula: Formula, assignment: Assignment) -> SatResult
   ```

4. Tests: standard SAT competition benchmarks, known SAT/UNSAT instances,
   edge cases (empty clause = UNSAT, no clauses = SAT, single variable).

5. Implement: DIMACS parser -> unit propagation -> pure literal
   elimination -> DPLL backtracking -> variable selection heuristic.

Output: A correct DPLL solver that parses standard DIMACS CNF,
with every decision point traceable to the algorithm specification.
```

## Best Practices

**Do:**
- Extract every "MUST", "SHALL", "REQUIRED" into a numbered constraint checklist before writing any code. This is the single highest-leverage step — agents with explicit constraint lists show ~35% higher success rates.
- Design the full type scaffold before any implementation. Types that mirror specification concepts catch structural errors at compile time, reducing constraint violations by ~50%.
- Re-read the specification clause when a test fails, rather than only inspecting the code. Specification re-consultation during debugging is the strongest predictor of success.
- Implement and test one specification section at a time, in dependency order. Never implement two interacting sections simultaneously.

**Avoid:**
- Do not rely on general programming knowledge to fill gaps in the specification. If the spec is silent on a behavior, flag it explicitly rather than guessing from prior experience with similar systems.
- Do not treat the specification as a one-time input that you read once and then set aside. Keep it as a persistent reference throughout the entire implementation.
- Do not attempt to implement the entire system in one pass. Multi-constraint satisfaction in a single generation step is where agents fail most often (~45% of all errors).
- Do not write generic error messages. Every error should trace to a specification section, making failures diagnosable against the standard.

## Error Handling

| Failure Mode | Frequency | Recovery Strategy |
|---|---|---|
| **Constraint violation** (satisfies some but not all spec requirements) | ~45% | Re-read the violated spec section. List all constraints in that section. Check implementation against each one individually. The violation is usually in a constraint interaction, not a missing feature. |
| **Type mismatch** (code violates scaffold signatures) | ~25% | Return to the scaffold. The type is derived from the spec, so a mismatch means the implementation has drifted from the specification's data model. Re-derive the type from the spec and adjust. |
| **Missing edge cases** (specification corner cases unhandled) | ~20% | Walk the constraint checklist for the relevant section. Specifications often define edge cases in notes, appendices, or "special case" paragraphs. Search for "unless", "except", "if ... then" in the spec text. |
| **Logic errors** (algorithm wrong despite correct structure) | ~10% | Trace through a failing test case by hand against the specification's algorithm description. Compare each step of the spec's pseudocode to the implementation. |

## Limitations

- **Ambiguous specifications**: This workflow assumes a well-written, authoritative specification. If the standard is internally inconsistent, incomplete, or relies heavily on implicit knowledge, the constraint extraction step will produce gaps that compound during implementation.
- **Performance optimization**: Specification-driven construction prioritizes correctness, not performance. The resulting implementation will be correct per the spec but may need a separate optimization pass (which is outside the spec-fidelity workflow).
- **Specifications requiring external context**: Some RFCs reference other RFCs or assume familiarity with related standards. If the user provides only one document but the implementation requires knowledge from referenced standards, flag the missing references explicitly rather than inferring behavior.
- **Very large specifications**: For specs over ~100 pages (e.g., full HTTP/2, TLS 1.3), the constraint checklist from step 2 can become unwieldy. In these cases, subset the specification to the user's required scope before beginning.
- **Language-specific idioms**: The scaffold design assumes a typed language. For dynamically typed languages, the scaffold step produces module/function signatures rather than type definitions, and provides less compile-time protection against constraint violations.

## Reference

**Paper**: Zhang et al., "SWE-AGI: Benchmarking Specification-Driven Software Construction with MoonBit in the Era of Autonomous Agents" (arXiv:2602.09447v2, 2026).
**Key takeaway**: Specification fidelity — not general coding ability — determines success in autonomous software construction. The critical technique is explicit constraint mapping + iterative spec-referenced validation. Code reading (re-consulting the spec) outweighs code writing as the dominant factor in success at scale.
**Repository**: https://github.com/moonbitlang/SWE-AGI