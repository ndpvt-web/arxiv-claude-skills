---
name: "projdevbench-benchmarking-ai-coding"
description: >
  Evaluate and improve AI-generated codebases using ProjDevBench's dual-scoring methodology:
  execution-based testing (80%) combined with LLM-assisted code review (20%). Applies structured
  evaluation across system architecture, functional correctness, and iterative refinement.
  Use when: "evaluate this codebase", "review my project architecture", "benchmark my code agent",
  "score this repository end-to-end", "assess code quality with execution tests",
  "run a ProjDevBench-style evaluation on this project".
---

# ProjDevBench: End-to-End Codebase Evaluation and Improvement

This skill enables Claude to evaluate complete codebases — not just individual files or functions — using the dual-scoring framework from ProjDevBench. Instead of only checking if code compiles and passes tests, this approach combines automated execution testing (weighted 80%) with structured code review (weighted 20%) to produce a holistic quality score. The technique is especially valuable for assessing AI-generated repositories, evaluating student projects, auditing system architecture decisions, and identifying the specific failure modes (wrong answers, time limit exceeded, memory leaks, architectural violations) that cause real-world project failures.

## When to Use

- When the user asks you to evaluate a complete codebase or repository, not just a single function
- When assessing whether an AI coding agent produced a correct and well-architected solution
- When reviewing a project that must meet a specification with both correctness and design quality
- When the user wants to understand *why* their code fails — distinguishing compilation errors, runtime errors, wrong answers, TLE, and memory issues
- When scoring or ranking multiple implementations of the same project specification
- When the user asks to improve a codebase iteratively based on structured diagnostic feedback
- When auditing for forbidden library usage, specification violations, or architectural shortcuts

## Key Technique

ProjDevBench's core insight is that **functional correctness alone is insufficient** to evaluate project-level code. The benchmark revealed a 27.38% overall acceptance rate across six state-of-the-art coding agents, with the dominant failure modes being: Wrong Answer (41.86%), Time Limit Exceeded (13.91%), and architectural constraint violations. Agents handle basic data structures but fail at complex system design, algorithmic optimization, and resource management.

The evaluation combines two complementary signals. **Execution-based testing** (80% weight) runs the code against an Online Judge with fine-grained diagnostic verdicts — not just pass/fail, but compilation error, runtime error, wrong answer, TLE, MLE, and memory leak categories. **Code review assessment** (20% weight) uses rule-based constraint checking plus LLM-assisted specification compliance review to catch issues invisible to test cases: forbidden library usage, hardcoded shortcuts, architectural violations, and specification non-compliance. The final score formula is: `0.8 * execution_score + 0.2 * code_review_score`.

The benchmark also evaluates **iterative refinement** — whether an agent can improve its solution when given diagnostic feedback. This maps directly to real development: the first implementation rarely passes all tests, and the ability to interpret error signals and systematically fix issues is what separates effective agents from ones that produce one-shot code and stop.

## Step-by-Step Workflow

1. **Parse the project specification into testable requirements.** Extract from the user's description (or README/spec document): functional requirements, input/output formats, resource constraints (time limits, memory limits), language/build constraints, and any explicitly forbidden approaches or libraries.

2. **Inventory the codebase structure.** Map out all source files, build configuration (CMakeLists.txt, Makefile, package.json, etc.), and project organization. Check that the build system is complete — ProjDevBench failures often start with missing build files or incorrect compilation targets.

3. **Run execution-based evaluation.** Build the project and run it against test cases. Classify each result into one of six diagnostic categories:
   - **CE** (Compilation Error): Code does not compile
   - **RE** (Runtime Error): Crashes, segfaults, uncaught exceptions
   - **WA** (Wrong Answer): Produces incorrect output
   - **TLE** (Time Limit Exceeded): Exceeds time constraints
   - **MLE** (Memory Limit Exceeded): Exceeds memory constraints
   - **ML** (Memory Leak): Detected via sanitizers or valgrind

4. **Compute the execution score.** For each test case, assign binary pass/fail. Weight test cases if the specification defines subtasks with different point values. Calculate: `execution_score = (weighted_passed / weighted_total) * 100`.

5. **Perform structured code review.** Evaluate the codebase against these dimensions:
   - **Specification compliance**: Does the code follow all stated constraints? Are forbidden libraries avoided? Does the output format match exactly?
   - **Architecture quality**: Is the code modular? Are responsibilities separated? Is the project structure logical for the problem domain?
   - **Resource efficiency**: Are algorithms appropriate for the input size? Are there obvious O(n^2) solutions where O(n log n) is required?
   - **Constraint adherence**: No hardcoded test answers, no external network calls, no bypassing intended difficulty.

6. **Score the code review.** Assign a score from 0-100 based on the severity of violations found. Full marks if no violations; deduct proportionally for each category of violation (specification non-compliance is most severe, minor style issues least).

7. **Compute the combined score.** Apply the formula: `final_score = 0.8 * execution_score + 0.2 * code_review_score`. Report the score with a breakdown showing both components.

8. **Generate diagnostic feedback.** For each failure, produce actionable feedback: which test cases failed and why, which specification constraints were violated, and specific code locations that need attention. Prioritize by impact — fix CE before WA, fix WA before TLE.

9. **Support iterative refinement.** If the user wants to improve the code, guide them through fixes in priority order: compilation issues first, then correctness bugs, then performance optimization, then architectural improvements. Re-evaluate after each round of changes.

10. **Track improvement across iterations.** Maintain a record of scores across attempts to verify that fixes are actually improving the codebase and not introducing regressions.

## Concrete Examples

**Example 1: Evaluating a student's B+ Tree key-value store**

User: "Evaluate my key-value database implementation against the spec. It should support GET, PUT, DELETE with persistence to disk, using a B+ tree index."

Approach:
1. Read the specification and identify requirements: CRUD operations, disk persistence, B+ tree indexing, specific time/memory limits
2. Inventory the codebase: check for `CMakeLists.txt`, source files, header organization
3. Build the project and run against provided test data
4. Classify results:
   - 45/60 test cases pass (WA on edge cases with duplicate keys)
   - 8 cases TLE on bulk insert (O(n) tree traversal instead of O(log n))
   - 7 cases RE on delete operations (null pointer when deleting from leaf)
5. Code review: finds `#include <map>` used as a fallback — violates the "implement your own B+ tree" constraint

Output:
```
## Evaluation Results

### Execution Score: 75.0 / 100
- Passed: 45/60 test cases
- Wrong Answer: 7 (duplicate key handling in PUT)
- Time Limit Exceeded: 8 (bulk insert performance)
- Runtime Error: 7 (null dereference in DELETE from leaf nodes)

### Code Review Score: 60.0 / 100
- VIOLATION: std::map used as fallback index (forbidden per spec) [-30]
- VIOLATION: No disk persistence for index nodes (in-memory only) [-10]
- Architecture: Clean header/source separation [OK]

### Final Score: 72.0 (0.8 * 75.0 + 0.2 * 60.0)

### Priority Fixes:
1. Fix null pointer in BPlusTree::deleteFromLeaf() — check for empty node before accessing children[0]
2. Handle duplicate keys in PUT by updating existing value instead of inserting
3. Replace std::map fallback with actual B+ tree traversal for lookups
4. Optimize bulk insert — current tree rebalancing is O(n) per insert; batch and build bottom-up
```

**Example 2: Scoring a game server/client implementation**

User: "I built a Minesweeper server and client in C++. The server manages game state, the client sends moves over sockets. Can you evaluate it?"

Approach:
1. Parse spec: server must handle concurrent clients, validate moves, detect win/loss, communicate over TCP
2. Build both executables from CMakeLists.txt
3. Run integration test suite simulating multiple game sessions
4. Classify: 30/40 pass, 6 WA (mine count mismatch on reveal), 4 TLE (blocking I/O on concurrent clients)
5. Code review: architecture uses single-threaded blocking accept() — won't scale to concurrent clients per spec

Output:
```
## Evaluation Results

### Execution Score: 75.0 / 100
- Passed: 30/40 integration tests
- Wrong Answer: 6 (mine adjacency count off-by-one in reveal logic)
- TLE: 4 (server blocks on single client, starving others)

### Code Review Score: 55.0 / 100
- ISSUE: Single-threaded blocking server — spec requires concurrent client support [-25]
- ISSUE: No graceful shutdown on SIGINT [-10]
- ISSUE: Client hardcodes localhost:8080 instead of reading from config [-10]
- Architecture: Clean protocol separation between game logic and networking [OK]

### Final Score: 71.0 (0.8 * 75.0 + 0.2 * 55.0)

### Priority Fixes:
1. Fix adjacency counting: off-by-one in grid boundary check at board.cpp:87
2. Switch server to non-blocking I/O (epoll/select) or spawn a thread per client
3. Make host/port configurable via command-line arguments
4. Add SIGINT handler for clean resource teardown
```

**Example 3: Iterative refinement of an interpreter**

User: "My Python interpreter in C++ passes 60% of tests. Help me get it to 100%."

Approach:
1. Run full test suite, categorize all failures by diagnostic type
2. Group failures by language feature: string operations (8 failures), nested functions/closures (5), exception handling (4), list comprehensions (3)
3. Prioritize: string operations are simplest to fix and unblock the most tests
4. Guide fix for string slicing (off-by-one on negative indices), re-run
5. Score improves from 60% to 72% — string fixes also resolved 4 exception-handling tests that depended on string formatting
6. Next iteration: tackle closure variable capture (requires changing from copy to reference semantics in environment)
7. After 3 rounds: 60% -> 72% -> 88% -> 95%

Output:
```
## Iteration Tracking

| Round | Execution | Code Review | Final  | Changes Made                          |
|-------|-----------|-------------|--------|---------------------------------------|
| 0     | 60.0      | 80.0        | 64.0   | Initial submission                    |
| 1     | 72.0      | 80.0        | 73.6   | Fixed string slicing, negative index  |
| 2     | 88.0      | 85.0        | 87.4   | Fixed closure capture, scoping rules  |
| 3     | 95.0      | 90.0        | 94.0   | Fixed list comprehension, GC edge case|

Remaining failures: 2 TLE on deeply nested recursion (stack depth > 10000)
Recommendation: Implement tail-call optimization or increase stack limit
```

## Best Practices

- **Do:** Always classify failures into diagnostic categories (CE/RE/WA/TLE/MLE/ML) before attempting fixes. The category determines the fix strategy — a WA needs logic debugging, a TLE needs algorithm redesign, an RE needs defensive coding.
- **Do:** Check the build system first. A missing CMakeLists.txt or incorrect target name causes 100% failure regardless of code quality. ProjDevBench data shows compilation issues are the most common first-submission failure.
- **Do:** Verify specification constraints during code review even when all tests pass. Hardcoded answers or forbidden library usage can score 100% on execution but 0% on code review.
- **Do:** Weight execution testing more heavily (80%) than code review (20%) to match real-world priorities — working code that's ugly beats beautiful code that's broken.
- **Avoid:** Treating all test failures equally. A solution that passes 18/20 with 2 TLE is architecturally sound but needs optimization; a solution that passes 10/20 with 10 WA has fundamental logic errors. These require completely different remediation strategies.
- **Avoid:** Skipping iterative evaluation. The first round identifies the easy wins; subsequent rounds expose deeper issues that were masked by earlier failures. Always re-run the full suite after fixes.

## Error Handling

- **Build failures**: If the project won't compile, check for missing dependencies, incorrect compiler flags, or platform-specific code. Report the exact compiler error and suggest the fix before proceeding to any other evaluation.
- **No test cases available**: If the user has no test suite, generate representative test cases from the specification — cover normal cases, edge cases (empty input, maximum size), and adversarial cases (malformed input if applicable).
- **Ambiguous specification**: When the spec doesn't define behavior for an edge case, flag it as "undefined behavior" in the report rather than penalizing either interpretation. Note the ambiguity for the user to resolve.
- **Environment mismatch**: ProjDevBench runs in Docker containers. If evaluating locally, differences in compiler version, OS, or available libraries can cause false failures. Document the evaluation environment.
- **Flaky tests**: If a test passes intermittently, classify it separately. Flakiness usually indicates race conditions (in concurrent code) or uninitialized memory — both are real bugs worth reporting.

## Limitations

- This evaluation framework is designed for **complete, buildable projects**, not code snippets or single functions. For function-level evaluation, standard unit testing is more appropriate.
- The 80/20 weighting between execution and code review is calibrated for programming contest-style problems. For production codebases, you may want to increase the code review weight to account for maintainability, security, and documentation.
- The diagnostic categories (CE/RE/WA/TLE/MLE/ML) assume compiled languages with clear resource limits. For interpreted languages or web applications, adapt the categories (e.g., replace TLE with response time thresholds).
- Code review by LLM can miss subtle specification violations that require deep domain expertise. For high-stakes evaluations, combine LLM review with human expert review.
- The benchmark's 20 problems span 8 categories but are biased toward systems programming in C++. The methodology generalizes, but the specific problem set does not cover web development, mobile, or ML pipeline tasks.

## Reference

**Paper:** [ProjDevBench: Benchmarking AI Coding Agents on End-to-End Project Development](https://arxiv.org/abs/2602.01655v2) — Lu et al., 2026. Look for: Table 2 (per-problem acceptance rates), Section 4.2 (failure mode analysis showing WA at 41.86% and TLE at 13.91%), and the code review rubric in Section 3.3. The benchmark repository with all 20 problems is at [github.com/zsworld6/projdevbench](https://github.com/zsworld6/projdevbench).