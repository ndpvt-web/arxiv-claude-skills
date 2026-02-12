---
name: "swe-agi-benchmarking-specification-driven-software"
description: "Build production-scale software systems from formal specifications, RFCs, and standards documents using specification-driven construction methodology. Triggers: 'implement this RFC', 'build a parser from the spec', 'implement this standard', 'construct from specification', 'build from RFC', 'specification-driven implementation'"
---

# Specification-Driven Software Construction

This skill enables Claude to build complete, production-scale software systems — parsers, interpreters, binary decoders, protocol implementations, and algorithmic solvers — strictly from authoritative specifications, RFCs, and standards documents. Drawing from the SWE-AGI benchmark methodology, it emphasizes long-horizon architectural reasoning, explicit planning before coding, disciplined code reading, and constraint satisfaction against formal requirements. The core insight: when building from specs, comprehending existing code and specifications is harder than writing new code — so invest disproportionate effort in reading and planning.

## When to Use

- When the user asks to implement a protocol, format, or standard from an RFC or specification document (e.g., "implement a JSON parser per RFC 8259")
- When building a parser, lexer, interpreter, or compiler from a formal grammar or language specification
- When implementing a binary decoder or encoder from a file format specification (e.g., PNG, WASM, PDF)
- When constructing an algorithmic system described by formal constraints (e.g., SAT solver, constraint solver)
- When the user provides a standards document and asks for a conforming implementation with a predefined API surface
- When implementing a system that must pass a comprehensive compliance test suite against a specification
- When refactoring or extending an existing codebase to conform to a new or updated specification

## Key Technique

**Specification-driven construction** differs fundamentally from typical code generation. Instead of inferring behavior from examples or writing code from vague descriptions, the agent must satisfy explicit constraints defined in authoritative documents. The SWE-AGI benchmark shows that this requires three capabilities most agents lack: (1) sustained comprehension of long, dense specification text, (2) architectural planning that maps spec requirements to code structure before writing begins, and (3) incremental validation against the spec throughout implementation rather than only at the end.

The critical finding from SWE-AGI is that **code reading — not writing — is the dominant bottleneck** as systems scale. Agents that re-read the same code sections repeatedly waste 60-75% of their token budget on comprehension rather than production. The remedy is to build a persistent mental model: read the spec once thoroughly, extract a structured requirements checklist, design the architecture against that checklist, then implement module by module with targeted re-reads only when needed. Agents that employ an explicit planning phase before coding outperform reactive approaches by 20-30%.

A second key insight is **constraint satisfaction over functional correctness**. Generating code that "works" is insufficient — the implementation must satisfy non-functional specification constraints including error handling semantics, edge-case behavior, encoding rules, and conformance to the exact API scaffold provided. This means every implementation decision must trace back to a specific clause in the specification.

## Step-by-Step Workflow

1. **Ingest and parse the specification document.** Read the full specification (RFC, grammar, format doc) end-to-end. Extract a structured requirements list: mandatory behaviors, optional features, error conditions, edge cases, and conformance levels (MUST/SHOULD/MAY per RFC 2119 if applicable). Do not skim — missing a single MUST-level requirement causes conformance failure.

2. **Identify the API scaffold and constraints.** If the user provides predefined interfaces, function signatures, or type definitions, catalog them exhaustively. These are non-negotiable boundaries. Map each API entry point to the specification sections it must satisfy. If no scaffold is provided, design one by identifying the natural module boundaries in the specification.

3. **Decompose into implementation modules.** Break the specification into 4-8 cohesive modules (e.g., for a parser: tokenizer, AST types, parser core, error recovery, serialization). Order them by dependency — foundational types first, then core logic, then integration layers. Each module should map to specific spec sections.

4. **Design the architecture before writing code.** For each module, write a brief design note: what data structures it uses, what spec constraints it satisfies, what interfaces it exposes to other modules. Identify cross-cutting concerns (error handling strategy, encoding/decoding conventions, state management). This planning phase is where most implementation quality is determined.

5. **Implement module by module, bottom-up.** Start with the lowest-dependency module (typically type definitions and utility functions). For each module: (a) re-read only the relevant spec sections, (b) implement the core logic, (c) handle all MUST-level edge cases from the spec, (d) write or run tests before moving to the next module. Do not implement the full system in one pass.

6. **Validate each module against the specification incrementally.** After implementing each module, trace through the relevant spec clauses and verify each is satisfied. Run any available tests. Fix conformance issues immediately — do not accumulate spec debt across modules.

7. **Integrate modules and test end-to-end.** Wire modules together following the architecture plan. Run the full test suite. For any failures, trace the failure back to a specific spec clause to determine whether the issue is a misunderstanding of the spec or a code bug.

8. **Audit for specification completeness.** Walk through the requirements checklist from step 1. Verify every MUST-level requirement has a corresponding implementation. Check that SHOULD-level requirements are implemented where feasible. Document any intentional omissions of MAY-level features.

9. **Optimize code reading efficiency.** When debugging or extending, avoid re-reading entire files. Instead, maintain awareness of module boundaries and jump directly to the relevant section. Use targeted searches rather than broad exploration. This discipline prevents the token-budget waste that SWE-AGI identifies as the primary scaling bottleneck.

10. **Document spec-to-code traceability.** For complex implementations, add brief comments citing the specific spec section a code block satisfies (e.g., `// RFC 8259 Section 7: Strings`). This aids future maintenance and conformance auditing.

## Concrete Examples

**Example 1: Implementing a JSON parser from RFC 8259**

```
User: Implement a complete JSON parser in Python following RFC 8259.
       It should handle all value types, unicode escapes, and produce
       clear error messages for malformed input.

Approach:
1. Read RFC 8259 fully. Extract requirements:
   - MUST support: object, array, number, string, true, false, null
   - MUST handle: UTF-8 encoding, \uXXXX escapes, surrogate pairs
   - MUST reject: trailing commas, single quotes, unquoted keys
   - Number format: no leading zeros, optional fraction/exponent

2. Design API scaffold:
   - parse(text: str) -> JsonValue (top-level entry)
   - JsonValue = dict | list | str | int | float | bool | None
   - JsonParseError with position and descriptive message

3. Module decomposition:
   - Module A: Scanner/tokenizer (string -> token stream)
   - Module B: String parser (handle escapes, unicode, surrogate pairs)
   - Module C: Number parser (RFC 8259 Section 6 grammar)
   - Module D: Recursive descent parser (tokens -> JsonValue)
   - Module E: Error reporting with line/column tracking

4. Implement bottom-up: A -> B -> C -> D -> E
   Each module tested against spec edge cases before proceeding.

Output: A conforming parser that passes RFC 8259 compliance tests,
        handles surrogate pairs correctly, rejects non-conforming
        input with clear positional error messages.
```

**Example 2: Building a WebAssembly binary decoder from the WASM spec**

```
User: Build a WASM binary module decoder that reads .wasm files and
       extracts the module structure per the WebAssembly Core Specification.

Approach:
1. Parse WASM Core Spec Binary Format section. Extract requirements:
   - Magic number \0asm, version 1
   - Section types: Type, Import, Function, Table, Memory, Global,
     Export, Start, Element, Code, Data (each with specific encoding)
   - LEB128 unsigned/signed integer encoding
   - Validation rules for section ordering

2. API scaffold:
   - decode(bytes) -> WasmModule
   - WasmModule contains typed section lists
   - DecodeError with byte offset

3. Module decomposition:
   - Module A: LEB128 decoder (unsigned + signed variants)
   - Module B: Type section parser (function signatures)
   - Module C: Import/Export section parsers
   - Module D: Code section parser (function bodies, locals)
   - Module E: Module-level decoder (magic, version, section dispatch)

4. Implement A first (foundational), then B-D (independent sections
   can be parallelized), then E (integration).

Output: A decoder that reads any valid .wasm binary, produces a
        structured module representation, and rejects malformed
        binaries with byte-offset error messages per spec.
```

**Example 3: Implementing a SAT solver from formal definition**

```
User: Implement a DPLL-based SAT solver that reads DIMACS CNF format
       and returns satisfying assignments or UNSAT.

Approach:
1. Parse DIMACS CNF specification and DPLL algorithm definition:
   - Input format: p cnf <vars> <clauses>, then clause lines
   - DPLL: unit propagation, pure literal elimination, branching
   - Output: SAT + assignment, or UNSAT

2. API scaffold:
   - solve(cnf: str) -> Result (SAT with model, or UNSAT)
   - Internal: Clause, Literal, Assignment types

3. Module decomposition:
   - Module A: DIMACS parser (text -> clause list)
   - Module B: Data structures (watched literals, assignment trail)
   - Module C: Unit propagation engine
   - Module D: DPLL search with backtracking
   - Module E: Solution formatter and validator

4. Implement A -> B -> C -> D -> E. Test C extensively with
   hand-crafted unit propagation scenarios before integration.

Output: A correct DPLL solver handling the full DIMACS format,
        with unit propagation and pure literal elimination,
        validated against standard SAT competition benchmarks.
```

## Best Practices

**Do:**
- Read the entire specification before writing any code. Partial spec reading is the top cause of conformance failures.
- Build a requirements checklist with MUST/SHOULD/MAY categorization and check items off during implementation.
- Implement and test one module at a time. Incremental validation catches spec misunderstandings early when they are cheap to fix.
- Design your architecture to mirror the specification's structure — if the spec has sections for "Strings," "Numbers," and "Arrays," your code should have corresponding modules.
- When a test fails, trace the failure to the specific spec clause before attempting a fix. The spec is the source of truth, not your intuition.

**Avoid:**
- Do not generate the entire implementation in a single pass. Monolithic generation leads to compounding spec violations that are expensive to untangle.
- Do not re-read large files repeatedly to "refresh" context. Build a module map during the first read and use targeted lookups thereafter.
- Do not invent behavior for cases the spec does not cover — flag them as undefined and ask the user for guidance, or follow the spec's stated default handling.
- Do not skip edge cases documented in the spec (surrogate pairs, integer overflow, malformed input). These are where conformance tests focus.
- Do not optimize prematurely. Get a conforming implementation first, then optimize with the spec's performance constraints in mind.

## Error Handling

**Specification ambiguity:** When the spec is unclear or contradictory, flag the ambiguity explicitly. Quote the conflicting clauses. Propose the most conservative interpretation (reject rather than accept ambiguous input) and note it for the user.

**Test failures with no obvious spec violation:** Re-read the specific spec section governing the failing case. Often the issue is a subtle encoding rule or edge case buried in a "Note" or "Implementation Consideration" section. Check for off-by-one errors in index-based specs.

**Scaling difficulties (large specs, many modules):** If the specification exceeds what can be held in context, create an explicit index mapping spec sections to code modules. Work section by section rather than trying to hold the full spec in working memory.

**API scaffold mismatches:** If the provided API surface conflicts with what the specification naturally requires, do not silently work around it. Raise the conflict with the user — the scaffold may need adjustment, or the spec may permit a different decomposition.

**Integration failures:** When individually-correct modules fail together, the issue is almost always at module boundaries. Check that data flowing between modules matches the types and invariants both sides expect. Trace a single input end-to-end through the module chain.

## Limitations

- Specifications that rely heavily on diagrams, visual formats, or non-textual content may not be fully accessible for extraction. Request the user provide a text summary of visual requirements.
- Extremely large specifications (500+ pages) exceed practical context limits. The skill works best with focused specification sections or standards under ~100 pages.
- Performance-critical implementations (real-time systems, high-throughput codecs) may need optimization passes that go beyond what specification conformance alone can guide.
- Specifications with extensive cross-references to other standards (e.g., a protocol spec referencing multiple subsidiary RFCs) require iterative deepening — not all dependencies can be resolved in a single pass.
- This approach assumes the specification is authoritative and correct. Buggy or outdated specs produce conforming but incorrect implementations.

## Reference

**Paper:** [SWE-AGI: Benchmarking Specification-Driven Software Construction with MoonBit in the Era of Autonomous Agents](https://arxiv.org/abs/2602.09447v2) — Zhang et al., 2026. Key takeaway: code reading dominates token budgets at scale (60-75%); explicit planning phases before coding improve success rates by 20-30%; incremental module-by-module validation is essential for specification conformance in systems requiring 1,000-10,000 lines of implementation.