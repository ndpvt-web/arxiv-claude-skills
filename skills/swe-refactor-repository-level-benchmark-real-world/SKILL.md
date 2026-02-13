---
name: "swe-refactor-repository-level-benchmark-real-world"
description: >
  Perform repository-level code refactoring with semantics-preserving guarantees using the SWE-Refactor methodology.
  Supports atomic refactorings (Extract Method, Move Method, Inline Method) and compound refactorings
  (Extract+Move, Move+Rename, Move+Inline) with cross-file dependency tracking and multi-stage verification.
  Trigger phrases: "refactor this method across the repo", "extract method from this function",
  "move this method to another class", "inline this method", "refactor without breaking tests",
  "restructure this code while preserving behavior"
---

# SWE-Refactor: Repository-Level Semantics-Preserving Code Refactoring

This skill enables Claude to perform rigorous, repository-level code refactoring that preserves program behavior while improving structure. Based on the SWE-Refactor benchmark methodology, it applies a four-stage pipeline — dependency analysis, targeted transformation, cross-file consistency enforcement, and multi-stage verification — to execute refactorings that touch multiple files, respect class hierarchies, and maintain all caller-callee relationships. The approach addresses the primary failure mode identified in the research: compound refactorings that require coordinated edits across files fail at over 60% rates when done naively, but succeed reliably when decomposed into verified atomic steps.

## When to Use

- When the user asks to extract a code block into a new method, especially when callers exist in multiple files
- When moving a method from one class to another, requiring updates to imports, call sites, and inheritance chains
- When inlining a method body at its call sites and removing the original declaration
- When performing compound refactorings like "extract this logic into a helper and move it to a utility class"
- When the user says "refactor this without breaking anything" or "restructure but keep behavior the same"
- When cleaning up a codebase by relocating methods to more appropriate classes based on cohesion
- When a method needs to be renamed and moved simultaneously across a large project

## Key Technique

The SWE-Refactor methodology identifies that LLM-based refactoring fails primarily on **cross-file consistency** and **compound transformations**. A model can rewrite a single method correctly but miss updating an import statement three directories away, or forget to propagate a renamed parameter through a chain of callers. The solution is to treat refactoring not as text transformation but as a **graph operation on the dependency structure** of the codebase.

The core insight is **decomposition with verification gates**. Every compound refactoring (e.g., Extract+Move) is broken into atomic steps (first Extract, then Move), with compilation and test verification between each step. This mirrors how the benchmark validates ground truth: RefactoringMiner detects whether the intended structural change actually occurred via AST comparison, and the test suite confirms behavioral equivalence. For Claude, this translates to: (1) build a dependency map before touching any code, (2) execute one atomic refactoring at a time, (3) verify all affected files remain consistent after each step.

The research also reveals that **context gathering is the bottleneck**, not code generation. Models that received full class hierarchies, caller-callee graphs, and package structures dramatically outperformed those given only the target method and its file. A multi-agent reviewer pattern (Developer generates, Reviewer checks cross-file consistency) improved success rates by 32% over single-pass generation. Claude should therefore spend the majority of effort on understanding the repository structure before writing any refactored code.

## Step-by-Step Workflow

1. **Identify the refactoring type.** Classify the request as one of six operations: Extract Method, Move Method, Inline Method, Extract+Move Method, Move+Rename Method, or Move+Inline Method. If compound, decompose into ordered atomic steps.

2. **Map the dependency graph for the target method.** Find all callers (grep for method invocations), all callees (methods invoked within the body), the class hierarchy (parent classes, interfaces, overrides), and all import statements referencing the containing class. Record every file path that will need modification.

3. **Catalog the method signature and its contract.** Document the full signature (parameters, return type, access modifier, annotations, thrown exceptions), any generic type parameters, and whether the method overrides a superclass or implements an interface method. This signature is the invariant that must be preserved or correctly transformed.

4. **Execute the first atomic refactoring on the primary file.** For Extract Method: create the new method in the same class, replace the extracted block with a call to it. For Move Method: copy the method to the target class, adjust `this` references to use delegation or parameter passing. For Inline Method: replace each call site with the method body, handling variable name collisions.

5. **Propagate changes to all dependent files.** Update every caller to reference the new method location or name. Add/remove import statements in every affected file. If the method was moved, update any reflection-based references, configuration files, or annotation processors that reference it by string.

6. **Verify cross-file type consistency.** Check that parameter types at every call site match the new signature. Verify that the target class has access to all types used in the moved method's body. Confirm no circular dependencies were introduced by the move.

7. **If compound, repeat steps 4-6 for each subsequent atomic step.** Run conceptual compilation checks between steps: verify no undefined references, no duplicate method signatures, no broken inheritance contracts. Each atomic step must leave the codebase in a valid state.

8. **Run compilation verification.** Execute the project's build command. If compilation fails, analyze the error (undefined symbol, type mismatch, access violation) and fix it. The most common failures are: missing imports in files that now reference the moved method, and access modifier conflicts when a private method is moved to a non-friend class.

9. **Run the test suite.** Execute all tests, especially those covering the refactored method's functionality. If tests fail, the refactoring introduced a behavioral change — diagnose whether it's a missed call-site update, an incorrect parameter ordering, or a scoping issue.

10. **Perform a self-review pass.** Re-read every modified file and verify: (a) no dead code was left behind (empty classes, orphaned imports), (b) the refactoring matches the user's intent (not an alternative transformation that happens to pass tests), (c) code style remains consistent with the project's conventions.

## Concrete Examples

**Example 1: Extract Method from a large function**

User: "Extract the validation logic from `processOrder()` in `OrderService.java` into its own method."

Approach:
1. Read `OrderService.java` and identify the validation block (lines doing input checks before the core logic).
2. Search the repo for all callers of `processOrder()` — they are unaffected since the public API doesn't change.
3. Search for any variables from the validation block that are used later in `processOrder()` — these become return values or out-parameters of the new method.
4. Create `validateOrder(Order order): ValidationResult` as a private method in the same class, moving the validation lines into it.
5. Replace the original validation block with `ValidationResult result = validateOrder(order);`.
6. Verify the class compiles and all existing tests pass.

Output:
```java
// Before
public void processOrder(Order order) {
    if (order.getItems().isEmpty()) throw new InvalidOrderException("No items");
    if (order.getCustomer() == null) throw new InvalidOrderException("No customer");
    BigDecimal total = calculateTotal(order);
    // ... 40 more lines of processing
}

// After
private void validateOrder(Order order) {
    if (order.getItems().isEmpty()) throw new InvalidOrderException("No items");
    if (order.getCustomer() == null) throw new InvalidOrderException("No customer");
}

public void processOrder(Order order) {
    validateOrder(order);
    BigDecimal total = calculateTotal(order);
    // ... 40 more lines of processing
}
```

**Example 2: Move Method to a different class (cross-file)**

User: "Move `calculateDiscount()` from `OrderService` to `PricingEngine` — it belongs there."

Approach:
1. Read `OrderService.java`, extract the full method body of `calculateDiscount()`.
2. Search the entire repo for all call sites: `grep -r "calculateDiscount"` — find hits in `OrderController.java`, `OrderServiceTest.java`, and `CheckoutFlow.java`.
3. Examine what `calculateDiscount` accesses from `OrderService` — if it uses `this.discountRepository`, that dependency must be available in `PricingEngine` (via constructor injection or parameter).
4. Add the method to `PricingEngine.java`, replacing `this.discountRepository` with an injected dependency or parameter.
5. In `OrderService.java`, remove the method. If other code in `OrderService` called it, delegate to `pricingEngine.calculateDiscount()`.
6. Update `OrderController.java`: change `orderService.calculateDiscount(order)` to `pricingEngine.calculateDiscount(order)`, add import for `PricingEngine` if not present.
7. Update `CheckoutFlow.java` similarly.
8. Update `OrderServiceTest.java`: move discount-related tests to `PricingEngineTest.java` or update the test to call through the new location.
9. Run build and full test suite.

Output: 5 files modified — `OrderService.java` (method removed), `PricingEngine.java` (method added), `OrderController.java` (call site updated), `CheckoutFlow.java` (call site updated), `OrderServiceTest.java` (test updated).

**Example 3: Compound refactoring — Extract + Move**

User: "Pull out the email-building logic from `NotificationService.sendAlert()` and put it in `EmailTemplateBuilder`."

Approach:
1. **Decompose into atomic steps**: First Extract Method (create `buildAlertEmail()` in `NotificationService`), then Move Method (relocate `buildAlertEmail()` to `EmailTemplateBuilder`).
2. **Step 1 — Extract**: Read `sendAlert()`, identify the email-construction lines. Create `buildAlertEmail(Alert alert): EmailMessage` in `NotificationService`. Replace the inline code in `sendAlert()` with a call to `buildAlertEmail()`. **Verify compilation passes.**
3. **Step 2 — Move**: Read `EmailTemplateBuilder.java` to understand its structure and dependencies. Move `buildAlertEmail()` there. Update `NotificationService.sendAlert()` to call `emailTemplateBuilder.buildAlertEmail(alert)`. Search for any other callers and update them. **Verify compilation and tests pass.**
4. This two-phase approach prevents the failure mode where the model tries to do both in one shot and misses updating intermediate references.

## Best Practices

- **Do:** Always build the full dependency map before making any edits. The number one cause of refactoring failure is missing a call site in a distant file.
- **Do:** Decompose compound refactorings into sequential atomic steps with verification between each step. Never attempt Extract+Move as a single transformation.
- **Do:** Preserve the exact method signature (parameter types, order, return type) unless the refactoring type explicitly changes it (e.g., Move+Rename). Even whitespace in annotations matters for some frameworks.
- **Do:** Check for reflection-based access, string references in configuration files (Spring XML, dependency injection configs), and annotation processors that reference methods by name — these won't appear in static call-site searches.
- **Avoid:** Generating "improved" code that goes beyond the requested refactoring. If the user asked to move a method, don't also rename variables or add error handling. Refactoring means structural change only.
- **Avoid:** Leaving dead imports, empty classes, or orphaned delegate methods after a move. Clean up every file touched by the refactoring.
- **Avoid:** Assuming a method can be safely moved without checking access modifiers. A private method accessing package-private fields of its original class cannot simply be moved to a class in a different package.

## Error Handling

| Failure | Cause | Fix |
|---------|-------|-----|
| Compilation error: undefined symbol | A call site was missed during propagation | Search the entire repo for the old method reference, update the remaining call site |
| Compilation error: type mismatch | Moved method uses a type not imported in the target file | Add the missing import to the target file |
| Compilation error: access violation | Moved method accesses private members of original class | Either change access modifiers, pass the data as parameters, or reconsider the move target |
| Test failure: behavioral change | Method body references `this` which now points to a different class | Replace implicit `this` references with explicit delegation to the original class |
| Test failure: null pointer | Dependency injection not configured for the new method location | Register the moved method's class in the DI container, or add constructor injection for required dependencies |
| Refactoring not detected | The transformation produced correct code but via an alternative pattern | Review whether the output matches the intended refactoring type (e.g., the user asked for Extract but you did Inline+Extract) |

## Limitations

- **Method-level only.** This approach covers method extraction, movement, inlining, and their compounds. Class-level refactorings (Extract Class, Merge Class), package reorganizations, and design pattern introductions require different strategies.
- **Statically-typed languages work best.** The dependency mapping and verification steps rely on clear type information. Dynamically-typed languages (Python, JavaScript) lack the compile-time guarantees that make cross-file consistency verifiable.
- **Reflection and metaprogramming are blind spots.** Code that references methods by string name (Java reflection, Spring annotations with string values, serialization frameworks) will not be caught by static call-site analysis. Always ask the user about framework usage.
- **Test coverage is required for behavioral verification.** If the refactored method has no test coverage, there is no automated way to confirm behavior preservation. Flag this to the user and recommend writing tests before refactoring.
- **Compound refactorings involving 3+ atomic steps have high failure rates.** Even with decomposition, long chains of transformations accumulate risk. For complex restructurings, consider breaking the user's request into separate, independently verified refactoring sessions.

## Reference

[SWE-Refactor: A Repository-Level Benchmark for Real-World LLM-Based Code Refactoring](https://arxiv.org/abs/2602.03712v1) — Xu, Yang & Chen, 2026. Key insight: decomposing compound refactorings into verified atomic steps and building complete dependency graphs before editing are the two highest-leverage practices for reliable LLM-based refactoring. See Tables 3-5 for per-type success rates and Section 5.2 for the multi-agent reviewer pattern that improves success by 32%.