---
name: "large-reasoning-failures"
description: "Detect and mitigate known LLM reasoning failures during code generation, review, and problem-solving. Applies the taxonomy from Song et al. (TMLR 2026) to catch compositional reasoning breakdowns, reversal curse errors, order bias, counting mistakes, and robustness fragility before they reach production. Use when: 'check my logic for reasoning failures', 'review this code for LLM-style bugs', 'harden this prompt against reasoning failures', 'why is the LLM getting this wrong', 'audit reasoning chain', 'stress-test this solution for robustness'."
---

# LLM Reasoning Failure Detection and Mitigation

This skill equips Claude with a structured methodology for identifying, diagnosing, and mitigating the known categories of LLM reasoning failures catalogued by Song, Han & Goodman (TMLR 2026). Rather than hoping reasoning is correct, this skill applies a systematic audit framework that checks generated code, logic chains, and problem solutions against the taxonomy of fundamental failures (compositional reasoning breakdown, reversal curse, working memory limits), application-specific failures (counting errors, arithmetic drift, theory-of-mind gaps in multi-agent code), and robustness issues (order sensitivity, framing effects, anchoring bias). The result is more reliable outputs with explicit reasoning verification at each step.

## When to Use

- When reviewing AI-generated code or logic for subtle reasoning bugs that compilers won't catch
- When a multi-step solution combines several facts or conditions and you need to verify compositional integrity
- When debugging why a prompt or chain-of-thought produces inconsistent or wrong answers
- When building prompts that must be robust to reordering, rephrasing, or minor input variation
- When writing code that involves counting, arithmetic, or sequential logic where LLMs are known to fail
- When designing multi-agent systems and need to guard against communication and long-horizon planning breakdowns
- When hardening test suites against the specific failure modes LLMs introduce (off-by-one from counting errors, reversed conditionals, order-dependent logic)

## Key Technique

The paper introduces a three-axis classification of reasoning failures: (1) **Fundamental failures** intrinsic to transformer architecture — working memory limits, inhibitory control weakness, the reversal curse (trained on "A is B" but failing "B is A"), and compositional reasoning breakdown where individual steps succeed but their combination fails. (2) **Application-specific failures** in domains like counting, arithmetic, theory of mind, moral consistency, and multi-agent coordination. (3) **Robustness failures** where logically equivalent inputs (reordered options, rephrased premises, renamed variables) produce different outputs.

The actionable insight is that these failures are **predictable and patterned**, not random. The reversal curse stems from unidirectional training — so any code that checks a bidirectional relationship (e.g., "if A contains B" vs "if B is contained in A") is a high-risk site. Compositional failures worsen with hop count — so any chain of 3+ dependent conditions needs explicit verification. Counting fails due to tokenization artifacts — so any generated code that counts characters, items, or iterations should be verified with concrete test cases. Order bias means the first option in any enumeration gets disproportionate weight — so generated switch/case logic and priority lists need scrutiny.

Mitigation follows a detect-then-fix pattern: identify which failure category applies, apply the corresponding countermeasure (decompose compositional chains, verify bidirectional relationships, test with reordered inputs, use explicit loops for counting), and add regression tests targeting the specific failure mode.

## Step-by-Step Workflow

1. **Classify the task by risk profile.** Before generating or reviewing code, identify which failure categories are relevant. Compositional reasoning? Flag multi-hop conditionals. Arithmetic? Flag numeric computation. Counting? Flag any enumeration or string-length logic. Multi-agent? Flag coordination and state-tracking code.

2. **Decompose compositional chains explicitly.** For any logic combining 2+ facts or conditions, break the reasoning into isolated, verifiable single-hop steps. Write each intermediate result to a variable or comment. Never rely on a single compound expression to be "obviously correct."

3. **Check for reversal curse violations.** Scan for any bidirectional relationship expressed in only one direction. If code checks `parent.contains(child)`, verify the inverse `child.belongsTo(parent)` is also handled. If a mapping is built A->B, confirm B->A lookups work when needed.

4. **Verify counting and arithmetic by execution.** Never trust generated counting logic (string lengths, list sizes, loop bounds, character occurrences) without running concrete test cases. Generate at least 3 edge-case inputs: empty, single-element, and a non-trivial case with a known answer.

5. **Test for order sensitivity.** Take any generated logic involving ordered alternatives (if/else chains, switch statements, priority rules, ranked lists) and mentally or literally reorder the branches. If the output would change due to short-circuit evaluation or implicit priority, the logic has an order-bias vulnerability. Make ordering explicit.

6. **Audit for framing and anchoring effects.** Rephrase the problem statement or requirements in an equivalent but differently structured way. If this changes your approach or output, the solution is fragile. Pin down the actual logical invariant and code to that, not to the phrasing.

7. **Guard multi-agent and long-horizon logic.** For systems with multiple components, state machines, or long transaction chains: add explicit verification checkpoints, ensure each agent/component can independently validate its inputs, and add a "global consistency check" that cross-references local state against system-wide invariants.

8. **Write failure-mode-targeted tests.** For each identified risk from steps 1-7, write a specific test case that would catch that failure category. Name tests to reflect the failure mode: `test_reversal_relationship`, `test_reordered_options_same_result`, `test_count_boundary_cases`.

9. **Document the reasoning chain.** For non-trivial logic, add a brief comment block showing the explicit reasoning decomposition — not what the code does, but why each step follows from the previous one. This makes future audits possible and catches implicit compositional leaps.

10. **Regression-guard against robustness drift.** Add property-based or fuzzing tests that permute input order, variable names, and phrasing where applicable. These catch the class of robustness failures where "it works for this input but breaks for a logically equivalent variant."

## Concrete Examples

**Example 1: Catching a compositional reasoning bug in access control**

User: "Review this function that checks if a user can access a resource."

```python
def can_access(user, resource):
    return (user.role == "admin" or
            resource.owner == user.id and
            resource.is_public == False and
            user.department == resource.department)
```

Approach:
1. Classify: compositional reasoning (4 conditions combined), plus operator precedence risk
2. Decompose: `and` binds tighter than `or`, so this actually reads as `admin OR (owner AND not-public AND same-dept)` — the `not-public` condition is inverted from likely intent
3. Check reversal: the ownership check is one-directional (fine here, but note it)
4. Identify the bug: the developer likely meant `is_public == True` (public resources accessible to same-dept users) or needs parentheses to group correctly

Output:
```python
def can_access(user, resource):
    is_admin = user.role == "admin"
    is_owner = resource.owner == user.id
    is_shared_in_department = (
        resource.is_public and
        user.department == resource.department
    )
    return is_admin or is_owner or is_shared_in_department
```
Each condition is isolated, named, and independently testable. The compositional chain is explicit.

---

**Example 2: Hardening a counting-dependent function**

User: "Write a function to check if a string is a valid ISBN-10."

Naive approach (high failure risk):
```python
def is_valid_isbn10(isbn):
    digits = isbn.replace("-", "")
    if len(digits) != 10:  # Counting risk: tokenization can miscount
        return False
    total = sum((10 - i) * (10 if d == 'X' else int(d))
                for i, d in enumerate(digits))  # Compositional chain
    return total % 11 == 0
```

Approach:
1. Classify: counting (length check), arithmetic (weighted sum), compositional (combining stripping, validation, summation)
2. Decompose: separate stripping, validation, and checksum into distinct steps
3. Verify counting: test with `"0-306-40615-2"` (valid), `"0306406152"` (valid, no hyphens), `""` (empty), `"012345678X"` (X terminus)
4. Check edge cases: 'X' only valid in last position; non-digit non-X characters must be rejected

Output:
```python
def is_valid_isbn10(isbn: str) -> bool:
    stripped = isbn.replace("-", "")

    # Validate structure: exactly 10 chars, first 9 are digits, last is digit or X
    if len(stripped) != 10:
        return False
    if not stripped[:9].isdigit():
        return False
    if stripped[9] not in "0123456789X":
        return False

    # Compute checksum with explicit loop (no compressed compositional chain)
    total = 0
    for i in range(9):
        total += (10 - i) * int(stripped[i])
    last = 10 if stripped[9] == "X" else int(stripped[9])
    total += last

    return total % 11 == 0
```
Tests targeting failure modes:
```python
def test_count_boundary():
    assert not is_valid_isbn10("012345678")   # 9 chars
    assert not is_valid_isbn10("01234567890") # 11 chars

def test_x_position_matters():
    assert not is_valid_isbn10("X306406152")  # X in wrong position

def test_reordered_hyphens_same_result():
    assert is_valid_isbn10("0-306-40615-2") == is_valid_isbn10("03-064-0615-2")
```

---

**Example 3: Auditing a multi-agent workflow for long-horizon and ToM failures**

User: "I have a pipeline where Agent A extracts data, Agent B validates it, and Agent C writes it to the database. Sometimes data gets corrupted."

Approach:
1. Classify: multi-agent system with long-horizon planning and inter-agent communication risks
2. Check for ToM failures: does Agent B have full context of Agent A's assumptions? Does Agent C verify what Agent B validated?
3. Check for state drift: can Agent B's validation become stale by the time Agent C writes?
4. Check for cascade failures: if Agent A produces subtly wrong output, does anything catch it?

Output — add verification checkpoints:
```python
# Each agent produces a typed, checksummed result
@dataclass
class PipelineResult:
    data: dict
    checksum: str
    source_agent: str
    validation_status: Optional[str] = None

# Agent B must independently re-derive expectations, not just trust A's output
def agent_b_validate(result: PipelineResult) -> PipelineResult:
    # Verify checksum to catch transmission corruption
    assert compute_checksum(result.data) == result.checksum, "Data integrity failure from Agent A"

    # Independent validation — don't rely on A's framing of the data
    errors = validate_schema(result.data)
    errors += validate_business_rules(result.data)

    result.validation_status = "pass" if not errors else f"fail: {errors}"
    result.checksum = compute_checksum(result.data)  # Re-sign after validation
    return result

# Agent C re-verifies before write — guards against stale validation
def agent_c_write(result: PipelineResult) -> None:
    assert result.validation_status == "pass", "Cannot write unvalidated data"
    assert compute_checksum(result.data) == result.checksum, "Data modified after validation"
    database.write(result.data)
```

## Best Practices

**Do:**
- Decompose any chain of 3+ dependent conditions into named intermediate variables
- Write tests named after the failure mode they target (`test_reversal_*`, `test_order_invariance_*`)
- Verify all counting and arithmetic with concrete execution, never by visual inspection
- Re-derive expectations independently at each stage of a multi-step pipeline

**Avoid:**
- Trusting a single compound boolean expression that combines multiple conditions with mixed `and`/`or`
- Assuming generated code handles bidirectional relationships correctly just because one direction works
- Relying on the order of items in generated lists, switch cases, or if/else chains to encode priority without explicit documentation
- Skipping edge-case tests for counting (empty input, single element, off-by-one boundaries)
- Treating LLM-generated arithmetic as reliable without test verification — especially multiplication, modular arithmetic, and multi-digit operations

## Error Handling

**Compositional chain is too deep to decompose:** If a function requires 5+ dependent conditions that resist decomposition, refactor into a decision table or state machine. The inability to decompose is itself a signal that the logic is fragile.

**Reversal curse creates circular dependencies:** When checking A->B and B->A creates infinite recursion, use a canonical form — always normalize relationships to one direction and convert at the boundary.

**Counting tests reveal tokenization mismatch:** If a counting function works for ASCII but fails for Unicode (multi-byte characters, combining marks), the failure is a tokenization-adjacent bug. Use language-native length functions and test with emoji, accented characters, and zero-width joiners.

**Order-invariance test fails but reordering isn't possible:** When business logic genuinely depends on order (priority queues, rule precedence), document the ordering contract explicitly and test that the ordering itself is deterministic and documented, not implicit.

## Limitations

- This skill catches **known, patterned** failure modes. Novel failure types outside the taxonomy will not be flagged.
- The audit adds overhead. For trivial single-step logic (simple getters, direct mappings), the full workflow is overkill. Apply proportionally to complexity.
- Robustness testing via input permutation can be expensive for large input spaces. Use property-based testing frameworks (Hypothesis, fast-check) rather than exhaustive enumeration.
- Some fundamental failures (working memory limits in very long contexts, deep compositional chains beyond 5-6 hops) cannot be fully mitigated by prompting alone — they require architectural solutions or tool use.
- Multi-agent failure detection requires access to the full pipeline. If agents are black boxes, the verification checkpoint strategy degrades to input/output contract testing only.

## Reference

Song, P., Han, P., & Goodman, N. (2026). *Large Language Model Reasoning Failures.* Transactions on Machine Learning Research (TMLR), with Survey Certification. [arXiv:2602.06176](https://arxiv.org/abs/2602.06176v1). Repository: [Awesome-LLM-Reasoning-Failures](https://github.com/Peiyang-Song/Awesome-LLM-Reasoning-Failures). Look for: the three-axis failure taxonomy (Table 1), mitigation strategy summaries per failure type, and the root-cause analysis linking architectural properties to specific failure patterns.