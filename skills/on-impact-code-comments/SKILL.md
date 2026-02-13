---
name: "on-impact-code-comments"
description: |
  Use code comments as a bug-fixing amplifier: generate implementation-detail comments on buggy code before attempting repairs,
  improving fix accuracy by up to 3x. Based on empirical research showing that LLMs fix bugs far more accurately when methods
  contain comments describing implementation intent and logic.
  Trigger phrases:
  - "fix this bug" or "debug this code" (when code lacks comments)
  - "why isn't this working" (on uncommented methods)
  - "add comments then fix" or "comment-assisted fix"
  - "help me understand and fix this method"
  - "the fix isn't obvious, can you analyze this?"
  - "repair this function"
---

# Comment-Augmented Bug Fixing

This skill enables Claude to leverage code comments as a structured reasoning aid when fixing bugs. Research by Vitale et al. (ICPC 2026) demonstrated that LLMs fix bugs up to **three times more accurately** when buggy methods contain comments describing their implementation logic, compared to fixing uncommented code. The technique is straightforward: before attempting a fix, generate focused implementation-detail comments for the buggy method, then reason over the annotated code to produce the repair. This mirrors the finding that having comments present during both "training" and "inference" (the TC-IC condition) dramatically outperforms the standard practice of stripping comments.

## When to Use

- When the user presents a buggy method or function that has **no comments or sparse comments** and asks for a fix
- When a bug is **non-obvious** — the logic error is subtle, involves off-by-one mistakes, incorrect boundary conditions, or wrong control flow
- When the user says "I can't figure out why this breaks" on a method that lacks documentation of its intended behavior
- When fixing code where the **intent vs. implementation gap** is the root cause (the code does something different from what was intended, but without comments you can't tell what was intended)
- When reviewing a pull request diff that removes or changes logic in an uncommented method and you need to assess correctness
- When the user asks you to debug a failing test and the relevant method is uncommented

## Key Technique

The paper evaluated CodeT5 and CodeT5+ models across four conditions: training with/without comments crossed with inference with/without comments (TC-IC, TC-INC, TNC-IC, TNC-INC). The critical finding is that TC-IC — comments present in both phases — improved bug-fix accuracy by up to 3x over TNC-INC (no comments in either phase). Importantly, training with comments did **not** degrade performance when comments were absent at inference time (TC-INC performed comparably to TNC-INC), meaning there is no downside to working with commented code.

An interpretability analysis revealed that **implementation-detail comments** — those describing what the code does step-by-step and why — were the most effective category. Comments about method purpose, parameter descriptions, or licensing headers provided little bug-fixing benefit. The actionable insight: comments that narrate the intended logic of each code block give the model a "second channel" of information to detect where the implementation diverges from the stated intent.

Since real-world code often lacks comments, the researchers used an LLM to generate synthetic comments for methods missing them. This generated-comment approach preserved the bug-fixing accuracy gains, meaning Claude can apply the same strategy: generate implementation comments first, then fix the bug using the annotated code as context.

## Step-by-Step Workflow

1. **Receive the buggy code.** Read the full method or function the user identifies as buggy. Note any existing comments, the method signature, and the reported symptom or failing test.

2. **Assess comment coverage.** Determine whether the code has meaningful implementation-detail comments. If the method already has rich inline comments explaining its logic, skip to step 5. If comments are absent, sparse, or only contain trivial information (e.g., auto-generated param docs), proceed to step 3.

3. **Generate implementation-detail comments.** Annotate the buggy method with inline comments that describe the *intended* logic of each significant code block. Focus on:
   - What each conditional branch is supposed to handle
   - What each loop is iterating over and why
   - What invariants or preconditions are assumed
   - What the expected state is after each transformation

   **Do NOT fix the bug yet.** Comment the code as-is, describing what it *appears* to intend, flagging any spots where the intent is ambiguous.

4. **Present the annotated code to identify the discrepancy.** With comments in place, read through the annotated method and look for mismatches between the stated intent (comments) and the actual implementation. The bug is where these diverge.

5. **Diagnose the bug.** Articulate the specific defect: which line(s) are wrong, what they do versus what they should do, and why the current behavior causes the reported symptom.

6. **Generate the fix.** Produce the corrected code. Keep the implementation-detail comments in the fixed version — they serve as documentation and make the fix reviewable.

7. **Verify the fix against the comments.** Walk through the fixed code line-by-line, checking that each block now matches its comment. If any comment no longer accurately describes the code, update the comment.

8. **Explain the fix.** Provide a concise summary: what was wrong, why, and what changed. Reference specific comments that helped identify the issue.

## Concrete Examples

**Example 1: Off-by-one error in array processing**

User: "This function should return the sum of every other element starting from the first, but it's giving wrong results."

```java
public int sumAlternating(int[] arr) {
    int sum = 0;
    for (int i = 1; i < arr.length; i += 2) {
        sum += arr[i];
    }
    return sum;
}
```

Approach:
1. Annotate with implementation-detail comments:
```java
public int sumAlternating(int[] arr) {
    int sum = 0;
    // Iterate over every other element starting from index 1 (second element)
    // Intent: should start from the first element (index 0)
    for (int i = 1; i < arr.length; i += 2) {
        sum += arr[i];
    }
    return sum;
}
```
2. The comment reveals the discrepancy: the loop starts at index 1 but should start at index 0 per the stated requirement "starting from the first."
3. Fix:
```java
public int sumAlternating(int[] arr) {
    int sum = 0;
    // Iterate over every other element starting from index 0 (first element)
    for (int i = 0; i < arr.length; i += 2) {
        sum += arr[i];
    }
    return sum;
}
```

**Example 2: Incorrect boundary condition in binary search**

User: "My binary search returns -1 even when the element exists."

```python
def binary_search(arr, target):
    low, high = 0, len(arr)
    while low < high:
        mid = (low + high) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            low = mid
        else:
            high = mid
    return -1
```

Approach:
1. Annotate:
```python
def binary_search(arr, target):
    low, high = 0, len(arr)         # high is exclusive (one past last valid index)
    while low < high:               # continue while search space is non-empty
        mid = (low + high) // 2     # pick middle index
        if arr[mid] == target:
            return mid              # found the target
        elif arr[mid] < target:
            low = mid               # narrow to upper half — but mid is NOT excluded,
                                    # so if mid == low, the loop never progresses
        else:
            high = mid              # narrow to lower half (mid excluded since high is exclusive)
    return -1
```
2. The comment on `low = mid` exposes the bug: when `mid == low` (which happens when `high == low + 1`), the search space never shrinks, causing an infinite loop or missed element.
3. Fix:
```python
def binary_search(arr, target):
    low, high = 0, len(arr)         # high is exclusive (one past last valid index)
    while low < high:               # continue while search space is non-empty
        mid = (low + high) // 2     # pick middle index
        if arr[mid] == target:
            return mid              # found the target
        elif arr[mid] < target:
            low = mid + 1           # exclude mid, narrow to upper half
        else:
            high = mid              # exclude mid, narrow to lower half
    return -1
```

**Example 3: Logic error in permission checking**

User: "Users with admin role are being denied access."

```javascript
function canAccess(user, resource) {
  if (user.role !== 'admin' || !resource.isPublic) {
    return false;
  }
  return true;
}
```

Approach:
1. Annotate:
```javascript
function canAccess(user, resource) {
  // Deny access if user is not admin OR resource is not public
  // Intent: admins should always have access; non-admins only to public resources
  // Bug: this denies access to admins when resource is not public,
  // because the OR makes either condition sufficient for denial
  if (user.role !== 'admin' || !resource.isPublic) {
    return false;
  }
  return true;
}
```
2. The comment-writing process forces articulation of the intended logic, revealing the OR should be AND — admins should be denied only if they're not admin AND the resource isn't public (which is never true for admins).
3. Fix:
```javascript
function canAccess(user, resource) {
  // Allow access if user is admin or resource is public
  if (user.role === 'admin' || resource.isPublic) {
    return true;
  }
  return false;
}
```

## Best Practices

- **Do:** Write comments that describe the *intended behavior* of each code block, not just what the syntax does. "Exclude mid from search space" is useful; "increment low" is not.
- **Do:** Comment the code *before* attempting any fix. The annotation step is the diagnostic — skipping it defeats the purpose.
- **Do:** Keep implementation-detail comments in the final fixed code. They prevent regression and help reviewers.
- **Do:** Focus comments on control flow (branches, loops, early returns) and state transformations — these are where bugs hide.
- **Avoid:** Writing comments that merely restate the code (`i++ // increment i`). These add noise without revealing intent.
- **Avoid:** Generating only method-level Javadoc/docstring summaries. The paper found that high-level purpose descriptions are far less effective than inline implementation comments for bug fixing.
- **Avoid:** Removing comments from code before attempting a fix, even if the codebase convention is minimal comments. The comments are a reasoning scaffold.

## Error Handling

- **Ambiguous intent:** If you cannot determine the intended behavior from the code alone, ask the user what the method is supposed to do before writing comments. Incorrect intent comments will lead to incorrect fixes.
- **Multiple bugs:** If comment annotation reveals more than one discrepancy, address them one at a time. Fix the most impactful bug first, re-verify, then proceed to the next.
- **Comments disagree with tests:** If existing comments describe behavior that contradicts passing tests, trust the tests — the comments may be stale. Note this conflict to the user.
- **Large methods:** For methods over ~50 lines, break the annotation into logical sections rather than commenting every line. Focus on the section relevant to the reported bug.

## Limitations

- **Performance bugs:** This technique targets logical/functional bugs. Comments about intended behavior won't help identify performance regressions, memory leaks, or concurrency issues where the code does what's intended but too slowly or unsafely.
- **Bugs outside the method:** If the defect is in a caller, a dependency, or configuration rather than in the method itself, annotating the method won't surface the root cause.
- **Already well-commented code:** If the code already has accurate implementation comments and the bug persists, the technique provides no additional benefit — the issue may be a misunderstanding of requirements rather than an intent-implementation gap.
- **Trivial bugs:** Syntax errors, typos, or missing imports don't benefit from comment augmentation. Use this for logic errors where understanding intent matters.
- **Generated comment quality:** The technique's effectiveness depends on comment quality. If the generated comments incorrectly describe the intent (because the code is too obfuscated), the fix may also be wrong.

## Reference

Vitale, A., Guglielmi, E., Scalabrino, S., & Oliveto, R. (2026). *On the Impact of Code Comments for Automated Bug-Fixing: An Empirical Study.* ICPC 2026. [arXiv:2601.23059](https://arxiv.org/abs/2601.23059v1) — Look for: Table 2 (accuracy across TC-IC/TNC-INC conditions), Section 4.2 (interpretability analysis showing implementation-detail comments are most effective), and Section 3.2 (LLM-based comment generation methodology).