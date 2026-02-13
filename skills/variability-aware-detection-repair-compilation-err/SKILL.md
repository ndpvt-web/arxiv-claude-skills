---
name: "variability-aware-detection-repair-compilation-err"
description: "Detect and repair compilation errors hidden behind #ifdef/#ifndef/#if defined() preprocessor directives in configurable C/C++ systems. Analyzes all feature combinations to find errors that only manifest under specific configurations. Use when: 'check my C code for ifdef errors', 'find compilation bugs across configurations', 'fix variability-induced compilation errors', 'analyze preprocessor conditionals for hidden bugs', 'audit configurable system for compile failures', 'detect errors in Linux kernel config-dependent code'."
---

# Variability-Aware Detection and Repair of Compilation Errors

This skill enables Claude to systematically detect and fix compilation errors that are hidden behind conditional compilation directives (`#ifdef`, `#ifndef`, `#if defined()`) in configurable C and C++ systems. These errors only manifest under specific feature flag combinations and are invisible to standard compilers that process one configuration at a time. The technique, based on Gheyi et al. (2026), treats the LLM as a variability-aware analyzer that reasons across all possible feature combinations simultaneously — achieving 0.97 precision and 0.94 accuracy on configurable systems, and producing compilable fixes in over 70% of error cases.

## When to Use

- When the user asks to audit C/C++ code containing `#ifdef`/`#ifndef`/`#if defined()` blocks for hidden compilation errors
- When reviewing a git diff or commit that modifies conditionally-compiled code (e.g., Linux kernel, BusyBox, OpenSSL)
- When the user reports a build failure that only occurs with certain `-D` flags or `CONFIG_*` settings enabled/disabled
- When refactoring code inside preprocessor conditionals and wanting to verify all configurations still compile
- When the user asks to fix a type error, undeclared variable, or missing declaration that appears under a specific feature combination
- When performing mutation testing or fault injection on configurable systems

## Key Technique

Traditional compilers like `gcc` and `clang` analyze a single preprocessor configuration at a time, so they cannot detect errors that only appear when a specific combination of feature flags is active. Specialized tools like TypeChef can analyze all configurations simultaneously, but they require complex setup, custom grammars, and heavy computation. The insight from this paper is that foundation models can perform variability-aware analysis by reasoning symbolically about all possible feature combinations directly from the source code — no special tooling required.

The approach works by providing the LLM with the complete source file (including all `#ifdef` branches) and explicitly instructing it to treat the code as a **software product line** where each preprocessor macro represents an optional feature. The model must enumerate which feature combinations (products) produce compilation errors under C99/C11 semantics, explain the root cause, and generate a minimal fix that preserves the original variability structure. The key constraint is that fixes must be **localized** — no removing features, no collapsing `#ifdef` blocks, no relying on external build flags.

The most common variability-induced errors fall into these categories: (1) **type resolution failures** — a struct or typedef defined inside one `#ifdef` block is referenced outside it or in a differently-guarded block; (2) **undeclared identifiers** — a variable or function declared under feature A is used under feature B; (3) **redeclaration conflicts** — incompatible declarations across different `#ifdef` branches; (4) **syntax/grammar errors** — malformed code in rarely-activated branches; (5) **implicit typing violations** — C99-invalid implicit declarations hidden behind feature flags.

## Step-by-Step Workflow

1. **Identify all preprocessor feature macros** in the source file. Extract every macro used in `#ifdef`, `#ifndef`, `#if defined()`, and `#elif` directives. List them as the feature set (e.g., `{M1, M2, CONFIG_MODULES}`).

2. **Enumerate the configuration space.** For N boolean features, there are 2^N configurations. For small N (1-5 features), enumerate all products explicitly. For larger N, focus on configurations where conditional blocks interact — specifically where a declaration in one guarded block is referenced in another.

3. **Analyze each conditional block's visibility.** For every declaration (variable, type, function, macro), determine under which feature combinations it is visible. Map each identifier to its set of enabling configurations.

4. **Cross-reference uses against declarations.** For every use of an identifier, verify that at least one declaration is visible under the same configuration. Flag any configuration where a use exists without a corresponding declaration, or where conflicting declarations are both visible.

5. **Apply C99/C11 semantic rules.** Check for: implicit function declarations (error in C99+), incompatible pointer assignments, duplicate struct field names, macro redefinitions with different replacement text, and missing type specifiers.

6. **Report failing configurations as structured output.** For each detected error, specify: the exact feature combination that triggers it (e.g., `M1=0, M2=1`), the line(s) involved, the error category, and a brief explanation.

7. **Generate a minimal, variability-preserving fix.** The fix must: (a) compile under ALL feature combinations, (b) not remove any `#ifdef`/`#ifndef` blocks or feature flags, (c) not introduce new external dependencies, (d) prefer adding missing declarations or adjusting guard conditions over restructuring code.

8. **Validate the fix mentally against all configurations.** Walk through each product configuration and verify the fixed code compiles. Pay special attention to edge configurations (all features off, all features on, and each feature individually toggled).

9. **If the input is a git diff**, focus analysis on added and context lines in the post-commit version. Identify which modified files contain preprocessor conditionals, reconstruct the post-commit code paths, and analyze those for configuration-dependent errors.

10. **Output the results in structured format** — a JSON object or clear markdown with: `result` (list of failing configurations), `explanation` (root cause per error), and `fixed_code` (corrected source preserving variability).

## Concrete Examples

**Example 1: Struct visibility across feature guards**

User: "Check this code for compilation errors across all configurations."

```c
#if M1 && M2
  struct info {
    const char *name;
    const char *pw;
  };
#endif
#ifdef M2
  struct info userinfo;
#endif
```

Approach:
1. Feature macros: `{M1, M2}` — 4 configurations: (0,0), (0,1), (1,0), (1,1)
2. `struct info` is defined only when `M1=1 AND M2=1`
3. `struct info userinfo` is declared when `M2=1` (regardless of M1)
4. Configuration `M1=0, M2=1`: `userinfo` declared with type `struct info`, but `struct info` is not defined

Output:
```json
{
  "result": "Configuration M1=0, M2=1 does not compile.",
  "explanation": "When M2 is enabled but M1 is disabled, 'struct info' is used but never defined. The struct definition is guarded by (M1 && M2), but the variable declaration only requires M2.",
  "fixed_code": "#ifdef M2\n  struct info {\n    const char *name;\n    const char *pw;\n  };\n  struct info userinfo;\n#endif"
}
```

**Example 2: Undeclared variable across configuration boundaries**

User: "Will this compile under all feature settings?"

```c
#include <stdio.h>

#ifdef FEATURE_LOGGING
  int log_level = 3;
#endif

void process(void) {
#ifdef FEATURE_VERBOSE
  if (log_level > 2) {
    printf("Processing...\n");
  }
#endif
}
```

Approach:
1. Feature macros: `{FEATURE_LOGGING, FEATURE_VERBOSE}` — 4 configurations
2. `log_level` defined only when `FEATURE_LOGGING` is set
3. `log_level` used inside `FEATURE_VERBOSE` block — no dependency on `FEATURE_LOGGING`
4. Configuration `FEATURE_LOGGING=0, FEATURE_VERBOSE=1`: `log_level` is undeclared

Output:
```json
{
  "result": "Configuration FEATURE_VERBOSE=1, FEATURE_LOGGING=0 fails.",
  "explanation": "Variable 'log_level' is declared under FEATURE_LOGGING but referenced under FEATURE_VERBOSE. When FEATURE_VERBOSE is enabled without FEATURE_LOGGING, 'log_level' is undeclared.",
  "fixed_code": "#include <stdio.h>\n\n#ifdef FEATURE_LOGGING\n  int log_level = 3;\n#endif\n\nvoid process(void) {\n#if defined(FEATURE_VERBOSE) && defined(FEATURE_LOGGING)\n  if (log_level > 2) {\n    printf(\"Processing...\\n\");\n  }\n#endif\n}"
}
```

**Example 3: Analyzing a git diff for variability bugs**

User: "Review this commit diff for config-dependent compilation issues."

```diff
--- a/drivers/net/ethernet/config.c
+++ b/drivers/net/ethernet/config.c
@@ -45,6 +45,10 @@
 #ifdef CONFIG_NET_POLL
   void net_poll_cleanup(struct net_device *dev);
 #endif
+
+void reset_device(struct net_device *dev) {
+  net_poll_cleanup(dev);
+}
```

Approach:
1. The added function `reset_device` calls `net_poll_cleanup` unconditionally
2. `net_poll_cleanup` is only declared when `CONFIG_NET_POLL` is defined
3. When `CONFIG_NET_POLL=0`, `reset_device` calls an undeclared function

Output:
```
**Variability-induced error detected.**

- **Failing configuration:** CONFIG_NET_POLL=0 (disabled)
- **Error:** Implicit declaration of function 'net_poll_cleanup' (error in C99+)
- **Location:** The new `reset_device` function calls `net_poll_cleanup` outside
  the CONFIG_NET_POLL guard
- **Fix:** Either wrap the call in `#ifdef CONFIG_NET_POLL` or move the entire
  `reset_device` function inside the existing CONFIG_NET_POLL block.
```

## Best Practices

- **Do:** Enumerate all feature macros before reasoning about configurations. Missing a macro means missing error-producing configurations.
- **Do:** Apply strict C99/C11 rules. Implicit function declarations are errors in C99+, not just warnings. Treat macro redefinitions with different replacement text as errors.
- **Do:** Preserve the original variability structure in fixes. The goal is to make all configurations compile, not to simplify the code by removing features.
- **Do:** Pay special attention to nested `#ifdef` blocks — a declaration inside `#if A && B` has narrower visibility than a use inside `#ifdef B` alone.
- **Avoid:** Collapsing or removing `#ifdef` blocks to "fix" errors. This destroys the intended configurability of the system.
- **Avoid:** Assuming all configurations are equally likely. Even rare configurations (all features off, obscure combinations) must compile correctly.
- **Avoid:** Ignoring `#elif` and `#else` branches — these create additional configuration-dependent paths that may contain latent errors.
- **Avoid:** Treating preprocessor macro redefinitions as mere warnings when the replacement text differs — under strict compilation, these are errors.

## Error Handling

- **Large feature spaces (>5 macros):** Full enumeration of 2^N configurations is impractical. Focus on interacting features — pairs of macros where one guards a declaration and another guards a use of the same identifier. Analyze only configurations where these interact.
- **External header dependencies:** If the code includes headers not provided, note which declarations are assumed to exist and flag that the analysis is partial. Do not invent header contents.
- **Complex macro expressions:** For guards like `#if (defined(A) || defined(B)) && !defined(C)`, convert the expression to a truth table before reasoning about which configurations enable each block.
- **Model confidence:** If the feature interaction is ambiguous (e.g., macros set by build systems with unknown constraints), state the assumption explicitly and flag the analysis as conditional.
- **False positives on warnings vs. errors:** Some compilers treat certain issues (e.g., macro redefinition with identical text) as warnings, not errors. Be explicit about whether the diagnostic is an error or warning under the specified standard.

## Limitations

- **Context window limits:** Very large files (>1000 lines) with many feature macros may exceed practical analysis capacity. For these, the user should isolate the relevant sections or focus on specific subsystems.
- **Build system constraints:** Some feature combinations may be impossible in practice due to Kconfig/Makefile constraints (e.g., `CONFIG_A` implies `CONFIG_B`). This analysis treats all combinations as possible unless the user provides constraints.
- **Linking errors:** This technique detects compilation errors (syntax, type, declaration). It does not detect linker errors from missing symbol definitions in other translation units, though it can flag suspicious forward declarations.
- **Semantic correctness:** A fix that compiles under all configurations is not guaranteed to be semantically correct. The approach verifies compilability, not runtime behavior.
- **Preprocessor arithmetic and stringification:** Complex `#if` expressions with arithmetic, token pasting (`##`), or stringification (`#`) may not be analyzed correctly in all cases.

## Reference

Gheyi, R., Albuquerque, L., Ribeiro, M., Almeida, E., & Albuquerque, D. (2026). *Variability-Aware Detection and Repair of Compilation Errors Using Foundation Models in Configurable Systems.* arXiv:2601.16755v1. Key takeaway: Foundation models can serve as practical, zero-setup variability-aware analyzers for configurable C systems, achieving 0.97 precision and 0.94 accuracy on detection, and producing compilable fixes in 70%+ of cases — look for the structured JSON prompt template and the taxonomy of variability-induced error categories.