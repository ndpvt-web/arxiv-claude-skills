---
name: "arkeval-benchmarking-evaluating-automated"
description: "Automated ArkTS code repair using retrieval-augmented generation, LLM-based test oracle synthesis, and structured benchmark evaluation for HarmonyOS development. Use when: 'fix this ArkTS error', 'repair HarmonyOS code', 'convert TypeScript to ArkTS', 'ArkTS compilation error', 'debug HarmonyOS component', 'generate tests for ArkTS code'."
---

# ArkEval: Retrieval-Augmented Automated Code Repair for ArkTS

This skill enables Claude to diagnose and repair ArkTS code using the retrieval-augmented repair (RAR) workflow from the ArkEval framework. ArkTS is a statically typed extension of TypeScript used for HarmonyOS development that rejects many valid TypeScript patterns at compile time. The ArkEval approach combines semantic fault localization, documentation-augmented patch generation, and LLM-consensus test oracle synthesis to systematically repair code in this low-resource language domain where conventional tools fall short.

## When to Use

- When the user has ArkTS compilation errors caused by TypeScript patterns that ArkTS's AOT compiler rejects (dynamic property access, `any` types, structural typing)
- When the user needs to convert standard TypeScript code to valid ArkTS for HarmonyOS applications
- When debugging UI state desynchronization bugs where imperative logic fails to trigger ArkTS declarative UI updates
- When fixing component lifecycle mismanagement in `aboutToAppear`/`aboutToDisappear` hooks
- When the user asks to generate test cases for ArkTS code and needs oracle-quality verification
- When building or evaluating a benchmark for automated program repair in any low-resource language
- When the user needs to understand why working TypeScript fails under ArkTS strict-mode constraints

## Key Technique

**Retrieval-Augmented Repair (RAR) Pipeline.** ArkEval's core workflow operates in three stages: (1) a Function Locator performs semantic fault localization to identify the buggy file and function, (2) a Patch Generator produces candidate fixes augmented by relevant HarmonyOS documentation and sample code retrieved via semantic search, and (3) a Patch Executor applies and verifies the fix against test oracles. The retrieval knowledge base is built from 15,000+ official HarmonyOS documentation pages and 400+ sample applications, chunked using AST-aware splitting (tree-sitter parsing at class/function boundaries) and a 512-token sliding window for prose, then embedded with a code-aware model and stored in a vector database for sub-millisecond retrieval.

**ArkTS "False Friends."** The central challenge is that ArkTS looks like TypeScript but enforces strict compile-time constraints that TypeScript allows at runtime. Common traps include: runtime property addition on objects (must use pre-declared class fields), `any`-typed variables (must use explicit types), `JSON.parse()` without class casting, dynamic property access via bracket notation, and event binding via `.on('click')` instead of ArkTS's `.onClick(() => {...})` pattern. These "false friends" cause LLMs trained on TypeScript to generate plausible but invalid ArkTS code. The repair workflow must retrieve ArkTS-specific documentation to override TypeScript habits.

**LLM-Consensus Test Oracle Synthesis.** When test suites are absent, ArkEval generates test oracles using a three-model committee: one model generates tests, two others independently score each test on syntax correctness, logic plausibility, and API validity (0-10 scale). A test is accepted only if the standard deviation of scores is below 1.5, ensuring strong inter-model agreement. Accepted tests must then pass dual verification: fail on the buggy code and pass on the fixed code.

## Step-by-Step Workflow

1. **Classify the defect type.** Examine the error and categorize it as one of three ArkTS-specific categories: (a) strict compile-time violation (35% of real bugs)--valid TypeScript rejected by ArkTS AOT compiler, (b) UI state desynchronization (42%)--imperative logic failing to trigger declarative UI updates, or (c) component lifecycle mismanagement (23%)--incorrect initialization/disposal in lifecycle hooks.

2. **Perform semantic fault localization.** Identify the specific file, class, and function containing the bug. For compile-time violations, trace the exact line from the compiler error. For UI state bugs, trace the data flow from the state variable through `@State`/`@Link`/`@Prop` decorators to the component that fails to re-render. For lifecycle bugs, check `aboutToAppear` and `aboutToDisappear` hook ordering.

3. **Retrieve relevant ArkTS documentation and examples.** Search for official HarmonyOS API documentation and sample code that covers the specific component, decorator, or API involved. Focus on ArkTS-specific patterns that differ from standard TypeScript. Prioritize official samples over generic TypeScript advice.

4. **Identify the TypeScript-to-ArkTS "false friend" pattern.** Check if the buggy code uses a TypeScript idiom that ArkTS rejects. Common false friends:
   - `let x: any = {...}` --> must declare a typed class
   - `obj.newProp = value` --> must pre-declare all properties in class definition
   - `JSON.parse(str)` --> must cast to explicit class type
   - `component.on('event')` --> must use `.onClick(() => {...})` style
   - Array mutation via `.push()` without triggering reactivity --> use `ObservedArray` or replace reference

5. **Generate the patch using documentation-augmented context.** Write the corrected code incorporating the retrieved ArkTS patterns. Ensure the patch:
   - Uses explicit type declarations instead of `any`
   - Declares all object properties in class definitions upfront
   - Uses ArkTS-specific UI event handlers and decorators
   - Preserves the original code's intent and structure

6. **Verify compilation.** Mentally or actually compile the patched code against ArkTS constraints. Check that no dynamic property access, undeclared fields, or implicit `any` types remain.

7. **Generate a test oracle if no tests exist.** Write at least one test that exercises the repaired behavior. Apply the dual-verification principle: the test should fail on the original buggy code and pass on the repaired code. If uncertain about test quality, generate multiple candidate tests and keep only those where the expected behavior is unambiguous.

8. **Validate reactivity for UI bugs.** If the bug involved state desynchronization, verify that the fix causes a reference change (not just mutation) for `@State` arrays/objects, or uses `@Observed`/`@ObjectLink` decorators correctly, so the declarative UI layer detects the change.

9. **Document the repair rationale.** Explain which ArkTS constraint was violated and why the TypeScript pattern was invalid, so the user understands the root cause and avoids the pattern in future code.

## Concrete Examples

**Example 1: Strict compile-time violation -- dynamic property addition**

User: "I'm getting a compilation error on this ArkTS code: `let user: any = { name: 'Alice' }; user.age = 30;`"

Approach:
1. Classify: Strict compile-time violation. ArkTS forbids `any` type and runtime property addition.
2. Localize: The `any` type annotation and dynamic `.age` assignment.
3. Retrieve: ArkTS requires all object shapes to be pre-declared as classes with typed fields.
4. Identify false friend: TypeScript allows `any` and dynamic properties; ArkTS does not.

Output:
```typescript
// BEFORE (invalid ArkTS -- TypeScript "false friend")
let user: any = { name: "Alice" };
user.age = 30;

// AFTER (valid ArkTS)
class User {
  name: string = "";
  age: number = 0;
}

let user = new User();
user.name = "Alice";
user.age = 30;
```
Explanation: ArkTS's AOT compiler requires statically known types. Replace `any`-typed object literals with class instances where all properties are pre-declared.

---

**Example 2: UI state desynchronization -- array mutation not triggering re-render**

User: "My ArkTS list component doesn't update when I push items to the array."

Approach:
1. Classify: UI state desynchronization. Array `.push()` mutates in-place without changing the reference.
2. Localize: The `@State` array and the method that pushes to it.
3. Retrieve: ArkTS declarative UI requires reference changes to detect state updates for arrays.
4. Generate patch: Replace in-place mutation with a new array reference.

Output:
```typescript
// BEFORE (UI does not re-render)
@Entry
@Component
struct TodoList {
  @State items: string[] = [];

  addItem(item: string) {
    this.items.push(item); // Mutation -- UI won't detect change
  }
}

// AFTER (UI re-renders correctly)
@Entry
@Component
struct TodoList {
  @State items: string[] = [];

  addItem(item: string) {
    this.items = [...this.items, item]; // New reference -- triggers UI update
  }
}
```
Explanation: ArkTS's declarative UI framework detects state changes by reference comparison. In-place mutations like `.push()` don't create a new reference. Spread into a new array to trigger reactivity.

---

**Example 3: Component lifecycle mismanagement -- event listener leak**

User: "My HarmonyOS app crashes after navigating away from a page that listens to sensor events."

Approach:
1. Classify: Component lifecycle mismanagement. Listener registered but never disposed.
2. Localize: Check `aboutToAppear` for listener registration, check `aboutToDisappear` for missing cleanup.
3. Retrieve: ArkTS lifecycle hooks require symmetric registration/deregistration.
4. Generate patch with proper cleanup.

Output:
```typescript
// BEFORE (listener leak -- crashes on navigation)
@Entry
@Component
struct SensorPage {
  aboutToAppear() {
    sensor.on(sensor.SensorId.ACCELEROMETER, (data) => {
      // handle sensor data
    });
  }
  // Missing: aboutToDisappear with sensor.off()
}

// AFTER (properly managed lifecycle)
@Entry
@Component
struct SensorPage {
  private sensorCallback = (data: sensor.AccelerometerResponse) => {
    // handle sensor data
  };

  aboutToAppear() {
    sensor.on(sensor.SensorId.ACCELEROMETER, this.sensorCallback);
  }

  aboutToDisappear() {
    sensor.off(sensor.SensorId.ACCELEROMETER, this.sensorCallback);
  }
}
```
Explanation: ArkTS components must clean up external subscriptions in `aboutToDisappear`. Store the callback as a class field so the same reference can be used for both `.on()` and `.off()`.

## Best Practices

- **Do:** Always check whether the error stems from an ArkTS-specific constraint vs. a general logic bug. The repair strategy differs fundamentally between the two.
- **Do:** Retrieve official HarmonyOS documentation before generating patches. ArkTS APIs evolve across SDK versions and generic TypeScript knowledge is frequently wrong for ArkTS.
- **Do:** Use AST-aware chunking (splitting at class/function boundaries) when building retrieval context, rather than naive text splitting. This preserves semantic coherence.
- **Do:** Generate test oracles that satisfy dual verification: they must fail on buggy code and pass on fixed code. A test that passes on both proves nothing.
- **Avoid:** Assuming TypeScript idioms work in ArkTS. The most dangerous bugs come from patterns that are valid TypeScript but rejected by ArkTS's AOT compiler.
- **Avoid:** Mutating `@State`-decorated arrays or objects in place. Always create a new reference to trigger the declarative UI update cycle.

## Error Handling

- **Patch applies but fails to compile:** The repair likely introduced a different ArkTS constraint violation. Re-examine the patch for remaining `any` types, dynamic property access, or unsupported TypeScript syntax (e.g., optional chaining on untyped values). Retrieve documentation for the specific API being used.
- **Patch compiles but tests fail:** The logic fix is incorrect even though the syntax is valid. Re-examine the defect category -- a compile-time fix may have masked an underlying UI state or lifecycle bug.
- **No relevant documentation found:** ArkTS is a low-resource language. Fall back to official Huawei sample applications as ground truth. If no sample matches, apply first-principles reasoning from ArkTS's core rules: no `any`, no dynamic properties, reference-based reactivity, symmetric lifecycle hooks.
- **Test oracle is ambiguous:** If the expected behavior of the fixed code is unclear, generate multiple candidate tests and apply the consensus approach -- keep only tests where multiple independent analyses agree on the expected output.

## Limitations

- ArkTS APIs and constraints evolve with each HarmonyOS SDK version. Repairs valid for one SDK version may not compile on another. Always confirm the target SDK version.
- The ArkEval benchmark found that even the best LLM (Claude) achieved only 3.13% Pass@1 on the full 502-issue benchmark, indicating that fully automated repair of real-world ArkTS bugs remains extremely difficult. Complex multi-file bugs and domain-specific API misuse are the hardest categories.
- This approach is most effective for single-file, localized bugs under 300 lines of change. Large architectural refactorings or bugs requiring cross-module reasoning exceed the RAR pipeline's capability.
- Retrieval quality depends on having up-to-date HarmonyOS documentation. Stale or missing docs degrade patch quality significantly, especially for newly introduced APIs.
- The LLM-consensus test generation works best for functional correctness. UI rendering bugs, performance issues, and race conditions are difficult to capture in automated test oracles.

## Reference

**Paper:** [ArkEval: Benchmarking and Evaluating Automated Code Repair for ArkTS](https://arxiv.org/abs/2602.08866v1) (Xie et al., 2026). Key sections: Section 3 for the five-phase benchmark construction pipeline, Section 4 for the RAR workflow architecture and retrieval knowledge base design, and Section 5 for the evaluation results showing the three ArkTS defect categories and per-model repair rates.