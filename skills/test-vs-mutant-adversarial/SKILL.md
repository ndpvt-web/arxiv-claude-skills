---
name: "test-vs-mutant-adversarial"
description: >
  Adversarial test generation using two competing LLM agents: a Test Agent that writes unit tests
  and a Mutant Agent that creates code mutations to expose blind spots. The agents co-evolve through
  iterative rounds, producing test suites with high coverage AND strong fault detection.
  Trigger phrases: "generate robust tests", "adversarial test generation", "mutation testing",
  "find blind spots in my tests", "harden my test suite", "test vs mutant"
---

# Adversarial LLM Test Generation (AdverTest)

This skill enables Claude to generate robust unit test suites by running an adversarial loop between two reasoning roles: a **Test Agent (T)** that writes and refines tests, and a **Mutant Agent (M)** that creates single-line code mutations designed to survive the current test suite. By iterating this attack-and-defend cycle guided by coverage and mutation score feedback, the resulting tests catch corner cases and subtle bugs that conventional test generation misses. Based on the AdverTest framework (Chang et al., 2026), which improved fault detection by 8.56% over the best existing LLM-based methods.

## When to Use

- When the user asks to "generate robust tests" or "harden my test suite" for a function or class
- When the user wants mutation-testing-informed test generation without setting up external mutation tools
- When existing tests pass but the user suspects they miss corner cases or boundary conditions
- When the user asks to "find blind spots" or "weak spots" in their current test coverage
- When writing tests for complex logic (math, parsing, state machines, financial calculations) where subtle off-by-one or operator errors are likely
- When the user explicitly asks for "adversarial testing" or "test vs mutant" style generation

## Key Technique

**Adversarial co-evolution of tests and mutants.** Traditional test generation optimizes for coverage alone, which misses bugs that live in covered-but-poorly-asserted code paths. AdverTest adds a second optimization axis: mutation score. A Mutant Agent examines the source code and the current test suite's coverage map, then crafts single-line mutations (operator swaps, boundary shifts, logic inversions, constant changes) specifically targeting uncovered lines and lines where existing tests don't distinguish correct from incorrect behavior. The Test Agent then receives these surviving mutants as concrete adversarial examples and writes new tests that kill them.

**Single-line mutation with the Coupling Effect.** Each mutation modifies exactly one line of source code. This constraint, grounded in the coupling effect hypothesis from mutation testing research, means that tests capable of detecting simple single-line faults will also detect more complex multi-line faults. The LLM generates context-aware mutations rather than applying fixed mechanical operators, so it can produce semantically meaningful mutants that simulate plausible real bugs (e.g., replacing `Math.max` with `Math.min` near boundary conditions).

**Deferred evaluation for efficiency.** Rather than re-running the full test suite after every single new test, evaluation is batched at the end of each round. This avoids redundant computation while still giving both agents accurate feedback for the next iteration.

## Step-by-Step Workflow

1. **Analyze the source code under test.** Read the target function/class, identify its public API, input types, return types, dependencies, and edge-case-prone logic (conditionals, loops, arithmetic, null checks, boundary comparisons).

2. **Generate the initial test suite (Round 0).** Acting as the Test Agent, write a comprehensive set of unit tests covering the happy path, boundary values, error cases, and any obviously tricky logic. Use the project's existing test framework and conventions. Compile and validate that all tests pass against the original code.

3. **Perform coverage analysis.** Mentally trace (or run if tooling is available) which lines and branches the initial tests exercise. Identify the set of uncovered lines (Lu) and weakly-asserted branches.

4. **Generate adversarial mutants (Mutant Agent, Round 1).** For each uncovered or weakly-tested region, create single-line mutations that a plausible bug could introduce:
   - **Coverage-driven mutants:** Target uncovered lines by changing operators, constants, or return values on those lines.
   - **Mutation-driven mutants:** Target covered lines where current tests would NOT detect a change -- e.g., swap `>` to `>=`, change `+` to `-`, replace a constant with a neighboring value, invert a boolean condition.
   - Each mutant must compile, modify exactly one line, and represent a semantically meaningful change (no whitespace or comment changes).

5. **Identify surviving mutants.** For each mutant, determine whether the current test suite would catch it. A mutant "survives" if no existing test would produce a different result when run against the mutated code versus the original. List surviving mutants with their line numbers and the specific change made.

6. **Augment tests to kill surviving mutants (Test Agent, Round 1).** For each surviving mutant (or group of mutants on the same line), write one or more new tests that:
   - Pass on the original code
   - Would fail on the mutated version
   - Include clear assertions that distinguish correct from incorrect behavior at that specific code point

7. **Repeat the adversarial loop (Rounds 2-N).** Generate new mutants targeting the updated test suite's remaining blind spots, then write tests to kill those. Run 3-5 total rounds or until no new surviving mutants can be found.

8. **Final test augmentation.** After the last mutant generation round, perform one final pass as the Test Agent to ensure the suite addresses the most recent mutants. The Test Agent always gets the last move.

9. **Validate and clean up.** Ensure all tests compile, pass against the original code, have clear names describing what they test, and include meaningful assertion messages. Remove any redundant tests that are strictly subsumed by others.

10. **Report the results.** Present the final test suite with a summary: number of adversarial rounds, total mutants generated, mutants killed, estimated mutation score, and the specific blind spots that were discovered and addressed.

## Concrete Examples

**Example 1: Hardening tests for a utility function**

User: "Generate robust tests for this function"
```java
public static int clamp(int value, int min, int max) {
    if (min > max) throw new IllegalArgumentException("min > max");
    return Math.max(min, Math.min(max, value));
}
```

Approach:

Round 0 -- Initial test suite (Test Agent):
```java
@Test void clampWithinRange() { assertEquals(5, clamp(5, 0, 10)); }
@Test void clampBelowMin() { assertEquals(0, clamp(-3, 0, 10)); }
@Test void clampAboveMax() { assertEquals(10, clamp(15, 0, 10)); }
@Test void clampAtMin() { assertEquals(0, clamp(0, 0, 10)); }
@Test void clampAtMax() { assertEquals(10, clamp(10, 0, 10)); }
@Test void clampInvalidRange() { assertThrows(IllegalArgumentException.class, () -> clamp(5, 10, 0)); }
```

Round 1 -- Mutant Agent generates surviving mutants:
```
Mutant M1: return Math.max(min, Math.min(max, value))  -->  return Math.min(min, Math.min(max, value))
  Line 3: swaps outer Math.max to Math.min. Survives because clampBelowMin asserts 0 which Math.min(0,-3)=-3 would break... wait, actually killed.

Mutant M2: return Math.max(min, Math.min(max, value))  -->  return Math.max(min, Math.max(max, value))
  Line 3: swaps inner Math.min to Math.max. SURVIVES -- clampAboveMax expects 10, Math.max(10,15)=15, killed.
  Actually killed. Let me find real survivors:

Mutant M3: if (min > max)  -->  if (min >= max)
  Line 2: changes > to >=. SURVIVES -- no test calls clamp with min==max.

Mutant M4: if (min > max)  -->  if (min < max)
  Line 2: inverts comparison. SURVIVES with edge cases only if min==max path not tested.
```

Round 1 -- Test Agent kills survivors:
```java
@Test void clampMinEqualsMax() { assertEquals(5, clamp(5, 5, 5)); }
@Test void clampMinEqualsMaxValueBelow() { assertEquals(3, clamp(1, 3, 3)); }
@Test void clampMinEqualsMaxValueAbove() { assertEquals(3, clamp(7, 3, 3)); }
```

These tests kill M3 (min>=max would throw for equal min/max) and M4 (min<max would throw for valid equal range).

---

**Example 2: Adversarial testing for a string parser**

User: "Find blind spots in my tests for this parser"
```python
def parse_key_value(line: str) -> tuple[str, str]:
    line = line.strip()
    if not line or line.startswith('#'):
        return None
    idx = line.index('=')
    key = line[:idx].strip()
    value = line[idx+1:].strip()
    if not key:
        raise ValueError("Empty key")
    return (key, value)
```

Existing tests:
```python
def test_basic(): assert parse_key_value("name=Alice") == ("name", "Alice")
def test_spaces(): assert parse_key_value(" host = localhost ") == ("host", "localhost")
def test_comment(): assert parse_key_value("# comment") is None
def test_empty(): assert parse_key_value("") is None
```

Adversarial analysis -- Mutant Agent finds blind spots:
```
M1: line.index('=') --> line.index(':')    # No test would detect if separator changed
    SURVIVES? No -- test_basic has '=' so index(':') raises ValueError. Killed.

M2: if not key --> if key                  # Inverts empty key check
    SURVIVES -- no test provides an empty key like "=value"

M3: line[:idx].strip() --> line[:idx]      # Removes strip() on key
    SURVIVES -- test_spaces has spaces around '=' but " host " != "host" would fail.
    Actually killed by test_spaces.

M4: line[idx+1:] --> line[idx:]            # Off-by-one includes '=' in value
    SURVIVES -- "name=Alice" would give "=Alice" not "Alice". Killed by test_basic.

M5: line.startswith('#') --> line.startswith('//')
    SURVIVES -- only test_comment uses '#', but mutation changes the recognized comment prefix.
    Actually test_comment would return ("#comment","") or similar, not None. So test_comment kills it?
    No: startswith('//') is False for "# comment", so `not line` is False, falls through to index('='), raises ValueError. Killed differently but still caught.

M6: value = line[idx+1:].strip() --> value = line[idx+1:]
    SURVIVES -- test_spaces asserts "localhost" but would get "localhost " with trailing space. Killed.

M7: return None (for empty/comment) --> return ("", "")
    SURVIVES -- tests check `is None` so this would be caught. Killed.

Real survivors after careful analysis:
M8: if not key --> pass (remove the check entirely)
    SURVIVES -- no test provides "=value" input with empty key.

M9: line = line.strip() --> line = line.lstrip()
    SURVIVES -- no test has trailing whitespace only (test_spaces has both sides, lstrip leaves trailing).
    Actually " host = localhost " lstripped = "host = localhost " then value = "localhost " after strip... wait, value has its own strip(). Key is "host". So this survives if only trailing whitespace on the line matters. Need a test where trailing whitespace on the line itself matters to the parsing structure.
```

Tests to kill survivors:
```python
def test_empty_key():
    with pytest.raises(ValueError, match="Empty key"):
        parse_key_value("=value")

def test_multiple_equals():
    assert parse_key_value("a=b=c") == ("a", "b=c")

def test_no_equals():
    with pytest.raises(ValueError):
        parse_key_value("no_equals_here")

def test_whitespace_only():
    assert parse_key_value("   ") is None
```

---

**Example 3: Quick adversarial pass on existing tests**

User: "Are there blind spots in my tests for this function?"

In this case, skip initial test generation (Round 0) and go directly to the Mutant Agent role. Systematically apply single-line mutations to each line of the source, evaluate which ones the existing tests would catch, and report the surviving mutants as blind spots with suggested tests to close them.

## Best Practices

- **Do:** Generate mutants that simulate plausible real bugs. Swap comparison operators (`>` to `>=`), change arithmetic (`+` to `-`), alter constants by +/-1, invert boolean conditions, swap similar method calls (`max`/`min`, `ceil`/`floor`).
- **Do:** Group mutants by the source line they modify. This helps the Test Agent write focused tests for each code region.
- **Do:** Give the Test Agent the last move. After the final mutant round, always generate one more set of tests so the suite addresses the latest adversarial inputs.
- **Do:** Keep tests readable. Each adversarial test should have a clear name that describes the bug scenario it catches (e.g., `test_boundary_when_width_equals_left_block`).
- **Avoid:** Generating trivial or equivalent mutants like changing whitespace, comments, variable names, or logging statements. Every mutant must change observable behavior.
- **Avoid:** Running more than 5 adversarial rounds. Diminishing returns set in quickly; 3-5 rounds typically surface the important blind spots.

## Error Handling

- **Mutant appears equivalent (no test can distinguish it from original):** Some mutations produce functionally identical code. If after two attempts no distinguishing test can be written, mark the mutant as equivalent and move on. Do not inflate the surviving mutant count with equivalents.
- **Generated test fails on original code:** This means the test has an incorrect assertion or the code has a real bug. Investigate which: if the code is correct, fix the test assertion; if the code is wrong, flag it to the user as a potential real defect discovered through adversarial analysis.
- **Mutant doesn't compile:** Discard it. Only count mutants that produce valid, compilable code. The single-line constraint helps minimize this.
- **No surviving mutants found:** This is a good outcome. Report the mutation score as high/complete and note which mutation categories were attempted.

## Limitations

- **Language-agnostic but Java-optimized:** The original paper validated on Java (Defects4J). The adversarial loop applies to any language, but the mutation patterns are most naturally expressed for statically typed languages with clear line boundaries.
- **Requires understanding of the code under test:** The Mutant Agent needs to reason about what the code does to generate meaningful mutations. Obfuscated, deeply nested, or extremely long functions reduce the quality of generated mutants.
- **No actual execution:** When Claude performs this mentally (without running the code), there's a risk of incorrectly classifying mutants as surviving or killed. For high-stakes scenarios, suggest the user run the generated tests and report back.
- **Equivalent mutant problem:** Some mutations produce semantically identical programs. This is an undecidable problem in general; Claude may occasionally generate or waste rounds on equivalent mutants.
- **Not a replacement for integration/E2E tests:** This technique strengthens unit-level fault detection. It does not address system-level interaction bugs, concurrency issues, or environment-specific failures.

## Reference

**Paper:** [Test vs Mutant: Adversarial LLM Agents for Robust Unit Test Generation](https://arxiv.org/abs/2602.08146v2) (Chang et al., 2026). Look for Algorithm 2 (the adversarial loop), the dual coverage-driven and mutation-driven augmentation strategies, and the case study showing how surviving mutants guided discovery of a real Defects4J bug through boundary condition testing.