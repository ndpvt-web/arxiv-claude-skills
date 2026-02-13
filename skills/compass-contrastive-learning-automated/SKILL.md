---
name: "compass-contrastive-learning-automated"
description: "Assess patch correctness using contrastive learning on code representations. Applies semantic-preserving code transformations and multi-view embedding comparison to determine whether a code patch is genuinely correct or merely overfitting to test suites. Trigger phrases: 'is this patch correct', 'check patch correctness', 'assess this fix', 'validate this bug fix', 'detect overfitting patch', 'contrastive patch assessment'"
---

# ComPass: Contrastive Learning for Automated Patch Correctness Assessment

This skill enables Claude to assess whether a code patch (bug fix) is genuinely correct or merely overfitting -- passing existing tests without truly fixing the underlying bug. It applies the ComPass technique: encoding buggy and patched code separately, computing multi-view difference representations (subtraction, concatenation, cosine similarity, etc.), and reasoning about whether the semantic change matches the intended fix. The approach draws from contrastive learning principles where semantically equivalent code variants are pulled together in embedding space while semantically different code is pushed apart.

## When to Use

- When a user presents a bug fix patch and asks whether it's correct or just passing tests by coincidence
- When reviewing auto-generated patches from APR tools (e.g., GenProg, TBar, SimFix, Nopol, SequenceR) and needing to triage which ones are truly correct
- When a user has multiple candidate patches for the same bug and needs to rank them by likely correctness
- When assessing whether a minimal code change (one-liner fix) actually addresses root cause vs. masking the symptom
- When a user wants to understand *why* a patch might be overfitting -- what semantic gap exists between the buggy and patched versions
- When building or evaluating an automated patch correctness pipeline and needing the assessment logic

## Key Technique

**Contrastive Representation of Patches.** ComPass treats patch assessment as a binary classification problem over *paired code representations*. Rather than feeding a raw diff to a classifier, it encodes the buggy code (C_b) and patched code (C_p) independently through a pre-trained language model, producing embedding vectors E_b and E_p. The critical insight is that correctness signal lives in the *relationship* between these embeddings, not in either one alone. ComPass computes six views of this relationship: direct concatenation [E_b; E_p], element-wise addition (E_b + E_p), subtraction (E_b - E_p), multiplication (E_b * E_p), cosine similarity, and Euclidean distance. Ablation studies show subtraction is the most informative single operation (7.23% recall drop without it), as it directly captures what changed.

**Semantic-Preserving Data Augmentation.** The second key idea is using 18 code transformation rules (variable renaming, for-to-while conversion, if-else reversal, infix expression splitting, statement reordering, switch-to-if conversion, etc.) to generate semantically equivalent code variants. During pre-training, contrastive learning with InfoNCE loss pulls together representations of code and its transformed variants while pushing apart unrelated code. This teaches the model that `for(int i=0; i<n; i++)` and `int i=0; while(i<n) { ... i+=1; }` are the same semantics, making it robust to surface-level syntactic variation in patches.

**Joint Fine-Tuning.** The pre-trained encoder is fine-tuned jointly with a two-layer fully connected classifier on labeled correct/overfitting patches. The multi-view concatenated vector passes through FC layers with softmax to produce P(correct) vs P(overfitting). This achieves 88.35% accuracy on 2,274 real-world Defects4J patches, a 6.33% improvement over the prior state-of-the-art (APPT).

## Step-by-Step Workflow

1. **Extract the buggy and patched code.** Isolate the changed method or code block in both its pre-fix (buggy) and post-fix (patched) versions. Include sufficient surrounding context (the full method body) but exclude unchanged boilerplate. If given a unified diff, reconstruct both complete versions.

2. **Normalize both code versions.** Strip comments, normalize whitespace, and standardize formatting so that superficial differences do not dominate the comparison. Retain all semantic content (variable names, control flow, expressions).

3. **Apply semantic-preserving transformations mentally.** Consider whether the patch could be expressed equivalently in another syntactic form (e.g., a ternary instead of if-else, a while instead of for). This helps distinguish true semantic changes from syntactic reshuffling. If the patch is *only* a syntactic transformation with no semantic effect, flag it as suspicious.

4. **Compute the multi-view difference analysis.** Systematically compare the buggy and patched versions across these dimensions:
   - **Subtraction (what changed):** What specific behavior is added, removed, or altered?
   - **Concatenation (full context):** Does the patched version still make sense within the broader method?
   - **Similarity (magnitude of change):** Is the change proportional to the bug? Tiny changes to fix complex bugs may indicate overfitting.
   - **Multiplication (interaction):** Do the changed elements interact correctly with unchanged elements?

5. **Assess control flow and data flow impact.** Trace how the patch modifies execution paths. A correct patch should address the root cause path. An overfitting patch often guards a specific failing input without fixing the general case (e.g., adding `if (x == null) return default;` instead of fixing why x is null).

6. **Check boundary conditions and edge cases.** Determine if the patch handles the bug's edge cases or only the specific test inputs. Overfitting patches frequently hard-code values, add narrow guards, or handle exact test scenarios.

7. **Evaluate semantic coherence.** Does the patched code express a coherent programming intent? Correct patches typically modify code in ways that a human developer would recognize as meaningful. Overfitting patches often feel arbitrary or disconnected from the surrounding logic.

8. **Classify and explain.** Provide a binary assessment (likely correct / likely overfitting) with a confidence level and specific reasoning tied to the analysis dimensions above.

## Concrete Examples

**Example 1: Null-guard patch for a NullPointerException**

User: "Is this patch correct? It fixes a NullPointerException in Apache Commons Math."

```java
// Buggy version
public double evaluate(double[] values, int begin, int length) {
    double sum = 0;
    for (int i = begin; i < begin + length; i++) {
        sum += values[i];
    }
    return sum / length;
}

// Patched version
public double evaluate(double[] values, int begin, int length) {
    if (length == 0) {
        return Double.NaN;
    }
    double sum = 0;
    for (int i = begin; i < begin + length; i++) {
        sum += values[i];
    }
    return sum / length;
}
```

Approach:
1. **Subtraction analysis:** The only change is an early-return guard for `length == 0`.
2. **Root cause check:** The NPE/ArithmeticException occurs from division by zero when `length == 0`. This guard directly prevents that.
3. **Semantic coherence:** Returning `NaN` for an empty input is standard mathematical convention (mean of nothing is undefined).
4. **Edge cases:** Handles the zero-length case; negative length is a separate concern but consistent with existing contract.

Assessment: **Likely correct (high confidence).** The patch addresses the root cause (division by zero) with a semantically appropriate return value. The guard is general, not test-specific.

---

**Example 2: Overfitting constant insertion**

User: "A repair tool generated this patch for a failing test in JFreeChart. Is it correct?"

```java
// Buggy version
public Range getDataRange(ValueAxis axis) {
    Range result = null;
    Iterator iterator = this.datasets.keySet().iterator();
    while (iterator.hasNext()) {
        Integer key = (Integer) iterator.next();
        XYDataset dataset = getDataset(key.intValue());
        // ... combines ranges from datasets
    }
    return result;
}

// Patched version
public Range getDataRange(ValueAxis axis) {
    Range result = null;
    if (axis == null) {
        return result;
    }
    Iterator iterator = this.datasets.keySet().iterator();
    while (iterator.hasNext()) {
        Integer key = (Integer) iterator.next();
        XYDataset dataset = getDataset(key.intValue());
        // ... combines ranges from datasets
    }
    return result;
}
```

Approach:
1. **Subtraction analysis:** Adds a null-check on `axis` that returns `null` early.
2. **Root cause check:** The method iterates datasets and combines ranges -- but the bug may be about *which* datasets to include based on the axis. The patch never uses `axis` to filter.
3. **Semantic coherence:** A null-axis guard returning null is defensive but doesn't fix any logic related to axis matching. The parameter `axis` is never used in the shown code, which itself is suspicious.
4. **Overfitting signals:** The patch adds a narrow guard that would pass a test calling `getDataRange(null)` but does not fix incorrect range computation for non-null axes.

Assessment: **Likely overfitting (high confidence).** The patch adds a null guard unrelated to the core range computation logic. It would pass a null-axis test case without correcting the actual dataset-axis mapping behavior.

---

**Example 3: Operator change in conditional**

User: "Check if this one-line fix for Apache Commons Lang is correct."

```java
// Buggy: uses <= causing off-by-one
if (index <= size) {
    throw new IndexOutOfBoundsException("...");
}

// Patched: uses < to fix boundary
if (index < size) {
    throw new IndexOutOfBoundsException("...");
}
```

Approach:
1. **Subtraction analysis:** Single operator change from `<=` to `<`.
2. **Boundary reasoning:** If `size` is the valid upper bound (exclusive), then `index == size` should throw. The buggy code with `<=` throws when `index == size`, which is correct for exclusive bounds. Wait -- re-read: the condition guards *when to throw*. If `index <= size` throws, that means even valid indices throw. Changing to `<` would still throw for indices below size, which is wrong. This needs careful context about what `size` represents.
3. **Context dependency:** Without knowing the full method contract, the operator direction is ambiguous. Need to check: is this a "throw if invalid" or "throw if valid" guard?

Assessment: **Uncertain -- requires broader method context.** One-line operator changes are the most common overfitting pattern in APR. Request the full method to determine whether the boundary condition aligns with the data structure's contract.

## Best Practices

- **Do:** Always reconstruct both complete code versions (buggy and patched) rather than reasoning from the diff alone. Context around the change is critical for correctness assessment.
- **Do:** Prioritize the subtraction view (what exactly changed) as the primary signal, then validate with similarity and coherence checks.
- **Do:** Consider whether the patch addresses the *general* bug or only the *specific* failing test input. Correct patches fix categories of failures; overfitting patches fix individual cases.
- **Do:** Apply semantic-preserving transformation reasoning -- if the patch is just a syntactic rearrangement (for-to-while, variable rename), it cannot fix a semantic bug.
- **Avoid:** Assuming a patch is correct just because it compiles and tests pass. The entire point of this technique is that test-passing is insufficient evidence of correctness.
- **Avoid:** Over-relying on patch size as a signal. Both tiny correct patches (fixing an off-by-one) and tiny overfitting patches (adding a narrow guard) exist. Analyze semantics, not size.

## Error Handling

- **Incomplete code context:** If only the diff is provided without surrounding method, ask the user for the full method body. Patch correctness depends on the contract of the enclosing function.
- **Multiple hunks:** When a patch modifies several locations, assess each hunk independently and then evaluate their combined effect. Overfitting patches sometimes combine a partial fix with an unrelated guard.
- **Language-specific semantics:** Be explicit about language-specific behavior (Java checked exceptions, Python truthiness, C pointer arithmetic) that affects whether a patch is semantically valid.
- **Ambiguous intent:** If the original bug description is not provided, state what bug the patch *appears* to fix and note that correctness depends on whether that matches the actual bug.

## Limitations

- This approach works best for single-method patches where buggy and patched code can be directly compared. Multi-file or architecture-level patches require broader analysis beyond pairwise code comparison.
- The technique is calibrated against patches from automated repair tools (GenProg, TBar, Nopol, etc.), which tend to produce small, localized changes. Large human-written refactoring patches are outside the typical scope.
- Without actually executing the code, this assessment is heuristic. A patch classified as "likely correct" may still have subtle bugs not detectable through static semantic analysis.
- Patches involving concurrency, I/O, or external state are harder to assess because correctness depends on runtime behavior not visible in the code structure.
- The original ComPass model was trained on Java patches from Defects4J. The principles generalize across languages, but language-specific idioms require language-specific reasoning.

## Reference

**Paper:** [ComPass: Contrastive Learning for Automated Patch Correctness Assessment in Program Repair](https://arxiv.org/abs/2602.07561v1) (Zhang et al., 2026). Look for: Table 1 (18 transformation rules), Equation 1 (InfoNCE contrastive loss), Figure 1 (multi-view representation integration architecture), and Table 6 (ablation study showing subtraction as most critical operation).