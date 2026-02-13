---
name: "context-sensitive-pointer-analysis-arkts"
description: "Perform context-sensitive pointer analysis for ArkTS/TypeScript code targeting OpenHarmony. Build precise call graphs, resolve indirect calls through closures and framework APIs, and detect vulnerability patterns. Use when the user asks to 'analyze ArkTS pointer flow', 'build a call graph for OpenHarmony app', 'resolve indirect calls in TypeScript', 'find data flow vulnerabilities in ArkTS', 'model ArkUI component state propagation', or 'reduce false positives in static analysis'."
---

# Context-Sensitive Pointer Analysis for ArkTS

This skill enables Claude to perform and guide context-sensitive pointer analysis on ArkTS and TypeScript codebases, following the APAK (ArkAnalyzer Pointer Analysis Kit) methodology. APAK is the first pointer analysis framework designed for ArkTS that uses callsite-sensitive, Andersen-style inclusion-based analysis with a plugin architecture for modeling OpenHarmony framework APIs. It reduces false positive rates from ~20% to 2% compared to class hierarchy analysis (CHA) by precisely tracking object references through closures, framework storage APIs, and component lifecycle callbacks.

## When to Use

- When the user asks to build or improve a **call graph** for an ArkTS or TypeScript project and needs precision beyond CHA/RTA
- When analyzing **indirect calls through closures or higher-order functions** to determine what functions a variable can point to
- When the user needs to trace **data flow through AppStorage/LocalStorage** bindings in OpenHarmony apps
- When performing **vulnerability pattern detection** that requires precise points-to information (e.g., taint analysis, injection detection)
- When the user wants to understand or resolve **ArkUI component state propagation** via @State, @Link, @Prop, @StorageProp decorators
- When reducing **false positives in static analysis** by replacing CHA/RTA call graph construction with pointer-analysis-driven resolution
- When modeling **Function.apply/call/bind** semantics to precisely track context injection in JavaScript/TypeScript code

## Key Technique

APAK implements **Andersen-style inclusion-based pointer analysis** with hybrid context sensitivity. The analysis builds a Pointer Assignment Graph (PAG) where nodes represent pointer variables and edges represent assignment relationships. It computes a points-to set `pts(v)` for each variable `v` using four core constraint rules: **Alloc** (allocation-site abstraction creates heap objects `o_i` at each `new` expression), **Assign** (`y = x` implies `pts(x) ⊆ pts(y)`), **Store** (`y.f = x` propagates to field pointers `o_j.f`), and **Load** (`x = y.f` retrieves field pointer contents). The analysis is field-sensitive, meaning `o_i.f` and `o_i.g` are tracked independently.

Context sensitivity uses a **hybrid strategy**: 2-CFA callsite sensitivity distinguishes calls through different call chains, function-sensitivity distinguishes function versions per definition site, and selective suppression disables context injection at `globalThis` access to prevent state fragmentation. The call graph is built **on-the-fly** -- as the pointer analysis discovers what functions a call expression can target, it adds those edges to the call graph and queues the newly reachable methods for analysis.

The critical differentiator is APAK's **plugin architecture** for framework API modeling. Three plugins handle cases where standard constraint rules are insufficient: (1) an **AppStorage/LocalStorage plugin** that injects bidirectional "backflow edges" for `@Link` bindings (creating strongly connected components in the PAG) and unidirectional edges for `@Prop`; (2) an **SDK plugin** that synthesizes abstract heap objects for black-box system API return values; and (3) a **Function plugin** that clones function object models for `apply/call/bind` with injected context. A plugin manager intercepts unresolved calls and delegates to registered plugins before falling back to standard resolution.

## Step-by-Step Workflow

1. **Collect entry points**: Identify all application entry points -- ability lifecycle methods (`onCreate`, `onDestroy`), component lifecycle methods (`aboutToAppear`, `build`), and event callbacks registered in ArkUI declarative syntax. These form the initial worklist.

2. **Build method-local PAGs**: For each reachable method, parse the ArkIR (or AST) and generate local Pointer Assignment Graph nodes and edges. Create heap object abstractions `o_i` (indexed by allocation site line number) for every `new` expression, lambda definition, and container literal. Annotate lambda variables with `FunctionType`.

3. **Initialize constraint worklist**: Seed the worklist with all Alloc constraints from entry-point methods. For each allocation `let v = new T()` at line `i`, add `o_i` to `pts(v)`.

4. **Propagate constraints to fixed point**: Iteratively process the worklist:
   - **Assign**: For each edge `x → y`, propagate `pts(x)` into `pts(y)`. If `pts(y)` changed, add dependent edges to worklist.
   - **Store**: For `y.f = x`, for each `o_j ∈ pts(y)`, propagate `pts(x)` into `pts(o_j.f)`.
   - **Load**: For `x = y.f`, for each `o_j ∈ pts(y)`, propagate `pts(o_j.f)` into `pts(x)`.

5. **Resolve dynamic calls on-the-fly**: When processing a call expression `x.m(args)`:
   - **Virtual dispatch**: Look up `o_j ∈ pts(x)`, find method `m` in the class hierarchy of each `o_j`'s type.
   - **Function pointer calls**: For `f(args)` where `f` is a variable, check `pts(f)` for function objects and resolve by type signature matching.
   - **Plugin interception**: Before standard resolution, check if any plugin handles this call pattern (e.g., `AppStorage.setAndLink`, `Function.apply`).

6. **Apply framework plugins**:
   - For `AppStorage.setAndLink(key, value)` / `@Link` decorators: inject bidirectional PAG edges between the storage location and the component property to create a strongly connected component.
   - For `@Prop` decorators: inject unidirectional edge (parent → child only).
   - For SDK black-box calls: synthesize an abstract heap object matching the declared return type and add it to the return variable's points-to set.
   - For `fn.apply(ctx, args)` / `fn.call(ctx, ...args)` / `fn.bind(ctx)`: clone the function object model and inject `ctx` as the receiver.

7. **Expand reachable methods**: When a call is resolved to a new target method not yet analyzed, generate its local PAG, connect formal parameters to actual arguments, connect return sites, and add all new constraints to the worklist. Repeat from step 4.

8. **Apply context sensitivity**: Maintain context strings of length k (default k=2) as call-site chains. Each pointer variable becomes context-qualified: `pts(v, ctx)`. Suppress context injection at `globalThis` references to keep global state consistent across contexts.

9. **Extract results**: Once the worklist is empty (fixed point reached), output: (a) the final call graph with resolved edges, (b) points-to sets for all variables, (c) any unreachable code identified by the analysis.

10. **Validate and report**: Compare call graph edges against CHA/RTA baselines if available. Flag edges present in CHA but absent in APAK as likely false positives. Report precision metrics and identify areas where plugins may need extension for unmodeled APIs.

## Concrete Examples

**Example 1: Resolving Indirect Calls Through Closures**

User: "I have an ArkTS class where a lambda is stored in a field and called later. Help me determine what functions `this.f()` can invoke."

```typescript
// Line 1
class Func {
  f: () => void = () => {}
}

// Line 5
let a = new Func()       // o_5: Func
a.f = () => { console.log("hello") }  // o_6: lambda

// Line 8
let b = new Func()       // o_8: Func
b.f = () => { console.log("world") }  // o_9: lambda

// Line 11
function invoke(x: Func) {
  x.f()  // What can this call?
}

invoke(a)
invoke(b)
```

Approach:
1. Create heap objects: `o_5` (Func at line 5), `o_6` (lambda at line 6), `o_8` (Func at line 8), `o_9` (lambda at line 9).
2. Process Store: `a.f = lambda_6` means for `o_5 ∈ pts(a)`, add `o_6` to `pts(o_5.f)`. Similarly `o_9` to `pts(o_8.f)`.
3. At call `invoke(a)`: `pts(x) = {o_5}` in context `[invoke←line15]`. At `x.f()`: load `pts(o_5.f) = {o_6}`. Resolved target: lambda at line 6.
4. At call `invoke(b)`: `pts(x) = {o_8}` in context `[invoke←line16]`. At `x.f()`: load `pts(o_8.f) = {o_9}`. Resolved target: lambda at line 9.
5. With 1-CFA context sensitivity, each call to `invoke` is distinguished. Without it, `x.f()` would resolve to both lambdas at both call sites.

Output:
```
Call graph edges for x.f() at line 13:
  Context [invoke←line15]: → lambda@line6 ("hello")
  Context [invoke←line16]: → lambda@line9 ("world")
CHA baseline would report: → lambda@line6, lambda@line9 (at BOTH call sites)
Precision gain: 2 spurious edges eliminated
```

**Example 2: Tracing Data Flow Through AppStorage**

User: "My OpenHarmony app uses AppStorage to share state between components. Trace how a tainted user input flows from ComponentA to ComponentB."

```typescript
// ComponentA.ets
@Entry @Component struct ComponentA {
  @StorageLink('userData') localData: string = ''

  build() {
    TextInput({ text: this.localData })
      .onChange((value: string) => {
        this.localData = value  // User input enters here
      })
  }
}

// ComponentB.ets
@Component struct ComponentB {
  @StorageLink('userData') mirroredData: string = ''

  build() {
    // mirroredData used in SQL query (potential injection)
    db.query(`SELECT * FROM users WHERE name = '${this.mirroredData}'`)
  }
}
```

Approach:
1. Identify `@StorageLink('userData')` in ComponentA -- this creates a bidirectional binding between `this.localData` and `AppStorage['userData']`.
2. Plugin injects backflow edges: `pts(this.localData) ⊆ pts(AppStorage['userData'])` AND `pts(AppStorage['userData']) ⊆ pts(this.localData)`.
3. Same plugin processes ComponentB's `@StorageLink('userData')`: bidirectional edges between `this.mirroredData` and `AppStorage['userData']`.
4. Propagation: user input at `onChange` → `this.localData` → `AppStorage['userData']` → `this.mirroredData` → SQL query sink.
5. Taint reaches the `db.query` call through the storage bridge that CHA/RTA analysis would miss entirely.

Output:
```
Data flow path detected (potential SQL injection):
  [SOURCE] TextInput.onChange callback (ComponentA.ets:8)
    → this.localData (ComponentA.ets:3, @StorageLink)
    → AppStorage['userData'] (framework bridge)
    → this.mirroredData (ComponentB.ets:3, @StorageLink)
    → db.query() argument (ComponentB.ets:7)
  [SINK] SQL query with string interpolation

Recommendation: Sanitize input before storage or parameterize the query.
```

**Example 3: Modeling Function.bind for Precise Call Resolution**

User: "I'm using `bind` to create callback handlers. Help me figure out what `handler()` actually calls."

```typescript
class Logger {
  prefix: string = "[LOG]"
  log(msg: string) { console.log(this.prefix + msg) }
}

class ErrorLogger {
  prefix: string = "[ERR]"
  log(msg: string) { console.error(this.prefix + msg) }
}

let logger = new Logger()         // o_10
let errLogger = new ErrorLogger() // o_11

let handler = logger.log.bind(errLogger)  // bind changes 'this'
handler("test")  // What does this call? With what receiver?
```

Approach:
1. `logger.log` resolves to `Logger.prototype.log` function object.
2. Function plugin intercepts `.bind(errLogger)`: clones the function object for `Logger.log`, injects `errLogger` (`o_11`) as the receiver context.
3. `pts(handler) = {cloned_log_with_receiver_o_11}`.
4. At `handler("test")`: resolves to `Logger.log` with `this = o_11` (ErrorLogger instance).
5. Inside the call, `this.prefix` loads from `pts(o_11.prefix) = {"[ERR]"}`.

Output:
```
handler("test") resolves to:
  Target: Logger.prototype.log
  Receiver: o_11 (ErrorLogger instance from line 11)
  this.prefix → "[ERR]"
  Effective output: console.error("[ERR]test")

Without bind modeling: analysis would lose the receiver binding,
treating 'this' as Logger instance or unknown.
```

## Best Practices

- **Do**: Use allocation-site abstraction (one abstract object per `new` expression). This is the right granularity for TypeScript -- finer-grained abstractions explode in cost, coarser ones lose precision on field-sensitive tracking.
- **Do**: Start with k=2 callsite sensitivity. The APAK evaluation shows k=2 provides the best precision/cost tradeoff for real OpenHarmony apps (370-720s for 1.2M LOC). Only increase to k=3 if precision on specific call chains is insufficient.
- **Do**: Model framework APIs via plugins rather than inlining their implementations. AppStorage, SDK calls, and Function.apply/call/bind have semantics that don't map to standard constraint rules -- plugin interception keeps the core analysis clean.
- **Do**: Suppress context injection at `globalThis` access. Global state must have a consistent points-to set across all calling contexts; adding context to globals fragments the state and causes unsoundness.
- **Avoid**: Running pointer analysis without first identifying all entry points. Missing an ability lifecycle method or event callback means entire subgraphs are unreachable, producing incomplete call graphs that look precise but have poor recall.
- **Avoid**: Treating `@Link` and `@Prop` identically. `@Link` requires bidirectional PAG edges (parent and child synchronize state), while `@Prop` is unidirectional (parent → child only). Modeling both as bidirectional creates false data flow paths; modeling both as unidirectional misses real flows.

## Error Handling

- **Unresolvable dynamic calls**: When `pts(receiver)` is empty at a call site, the call cannot be resolved. Log it as a potential analysis gap. Common causes: missing entry points, unmodeled framework callbacks, or reflective calls. Fall back to CHA for that specific call site rather than dropping the edge entirely.
- **Non-termination / excessive iteration**: If the worklist does not converge within a reasonable bound, check for cyclic PAG edges introduced by `@Link` bidirectional bindings. These create SCCs that are correct but can cause repeated re-propagation. Use a visited-set or delta-based propagation to only propagate new additions.
- **Memory pressure on large apps**: If PAG node count exceeds ~100K, consider reducing context depth to k=1 or applying selective context sensitivity (full k=2 only for application code, k=0 for library code). APAK's 95th percentile memory is 284 MB for typical apps.
- **Missing SDK type information**: When the SDK plugin encounters an API with no declared return type, it cannot synthesize an abstract heap object. Flag these as analysis holes and recommend adding type stubs.

## Limitations

- **Dynamic property access with computed keys**: Expressions like `obj[expr]` where `expr` is not a string literal cannot be resolved statically. The analysis conservatively ignores these, potentially missing data flow through dynamically-keyed properties.
- **Reflection and eval**: `eval()`, `new Function()`, and full reflection are not modeled. These are uncommon in ArkTS (which restricts some JS dynamism) but will cause unsoundness if present.
- **Prototype chain mutations**: Runtime modifications to prototype chains (`Object.setPrototypeOf`) are not tracked. The analysis assumes the static class hierarchy is stable.
- **Third-party native modules**: C/C++ code invoked through N-API is a black box. The SDK plugin can synthesize return objects but cannot model side effects within native code.
- **Scalability ceiling**: At k=3, analysis of a 1.2M LOC app takes ~720 seconds. For very large monorepos, modular analysis with summary-based composition would be needed but is not yet implemented.
- **ArkTS-specific**: The heap object model and plugin system are designed for ArkTS/OpenHarmony. Applying this to general TypeScript requires replacing the ArkUI and AppStorage plugins with equivalents for the target framework (e.g., React, Angular).

## Reference

**Paper**: "Context-Sensitive Pointer Analysis for ArkTS" by Yizhuo Yang, Lingyun Xu, Mingyi Zhou, Li Li (ASE Industry 2025).
**Link**: [https://arxiv.org/abs/2602.00457v1](https://arxiv.org/abs/2602.00457v1)
**Key sections**: Table III for constraint rules, Algorithm 1 for the iterative worklist procedure, Section IV-B for the plugin architecture, and Section V for evaluation on 1,663 real-world apps showing false positive reduction from 20% to 2%.