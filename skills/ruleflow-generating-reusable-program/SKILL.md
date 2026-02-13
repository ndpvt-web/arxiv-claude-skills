---
name: "ruleflow-generating-reusable-program"
description: "Optimize Pandas code by discovering per-program improvements, generalizing them into reusable rewrite rules, and applying those rules as a lightweight compiler pass. Use when the user says 'optimize this Pandas code', 'speed up my DataFrame operations', 'rewrite my notebook for performance', 'apply Pandas rewrite rules', 'find slow Pandas patterns', or 'generate optimization rules for my Python data pipeline'."
---

# RuleFlow: Generating Reusable Pandas Program Optimizations

This skill enables Claude to optimize Pandas programs using the RuleFlow methodology — a 3-stage hybrid approach that (1) discovers per-program optimizations via LLM analysis, (2) generalizes them into abstract rewrite rules with pattern variables and runtime preconditions, and (3) applies those rules deterministically as a compiler pass. Rather than ad-hoc one-off suggestions, this produces principled, validated transformations that can be reused across codebases. The technique achieves up to 1770x speedups on individual patterns and consistently outperforms both systems-based (Modin) and compiler-based (Dias) Pandas optimization frameworks.

## When to Use

- When the user asks to optimize Pandas DataFrame operations in a Python script or Jupyter notebook
- When profiling reveals slow Pandas calls (`iterrows`, chained indexing, `apply` with lambdas, copy-heavy operations like `drop` or `rename`)
- When the user wants to generate reusable optimization rules for a codebase with many similar Pandas patterns
- When migrating a data pipeline and wanting to systematically replace slow idioms with fast equivalents
- When the user has a notebook collection and wants to batch-apply performance improvements
- When building a CI linter or pre-commit hook that flags slow Pandas patterns and suggests rewrites

## Key Technique

**The core insight is decoupling discovery from deployment.** Traditional LLM-based optimization asks the model to optimize each program individually — this is expensive, unreliable (3.79% yield), and non-reproducible. RuleFlow instead uses the LLM only twice: once to discover optimized code variants (SnippetGen), and once to generalize those variants into abstract rewrite rules (RuleGen). After that, a deterministic compiler (CodeGen) applies rules without any LLM involvement.

**Rewrite rules use a typed pattern language.** Each rule has three components: a Left-Hand Side (LHS) pattern with typed abstract variables like `@{Name: v1}` for identifiers or `@{Const(str): c1}` for string literals; a Right-Hand Side (RHS) replacement using those same variables; and runtime preconditions that must hold for the rewrite to be safe (e.g., `isinstance(@{v1}, pd.DataFrame)`). This makes rules both general enough to match many programs and safe enough to apply automatically.

**RuleGen uses a 4-agent decomposition** to achieve 68% rule validity (vs 18% for single-agent). Agent A1 identifies extractable variables and constants. Agent A2 resolves AST node types. Agent A3 constructs the abstract LHS/RHS patterns. Agent A4 synthesizes runtime preconditions. Each agent's output is verified by the compiler before passing to the next stage.

## Step-by-Step Workflow

1. **Profile the target code** to identify hot Pandas operations. Look for `drop`, `rename`, chained bracket indexing, `apply` with simple lambdas, `iterrows`, `groupby` followed by aggregation, and copy-heavy patterns. Measure baseline runtime.

2. **Generate optimized candidates** for each slow code cell. Produce 1-5 semantically equivalent rewrites that use faster Pandas APIs, in-place operations, vectorized methods, or specialized accessors (`.loc`, `.iloc`, `.at`, `.iat`).

3. **Validate equivalence** by executing both original and candidate code on representative DataFrames. Check that outputs match across multiple random inputs. Reject candidates that produce different results.

4. **Verify optimization** by timing both versions. Accept only candidates that clear a meaningful threshold (e.g., >2x relative speedup or >150ms absolute improvement). Discard marginal gains.

5. **Generalize into a rewrite rule** by abstracting concrete variable names into typed pattern variables (`@{Name: v1}`), concrete string/int literals into typed constants (`@{Const(str): c1}`), and arbitrary expressions into expression slots (`@{expr: e1}`).

6. **Synthesize runtime preconditions** that encode when the rewrite is safe. Typical preconditions: type checks (`isinstance(@{v1}, pd.DataFrame)`), membership checks (`@{c1} in @{v1}.columns`), and structural checks (`is(@{e1}, dict)`).

7. **Apply rules via pattern matching** against the user's code AST. For each match, bind abstract variables to concrete code fragments. Evaluate preconditions at the call site. If all pass, emit the rewritten code.

8. **Wrap in conditional guards** when preconditions cannot be statically verified. Emit `if precondition: optimized_version else: original_version` to guarantee correctness.

9. **Schedule multiple rules** using a greedy strategy ordered by observed speedup magnitude. When multiple rules match the same code cell, apply the highest-impact rule first and re-check remaining matches.

10. **Report results** with before/after code, estimated speedup per transformation, and any preconditions the user should verify hold in their specific context.

## Concrete Examples

**Example 1: Column Deletion with `drop` -> `pop`**

User: "Optimize this Pandas code — it's slow when processing large DataFrames"
```python
df = df.drop(['unnecessary_col'], axis=1)
df = df.drop(['temp_data'], axis=1)
```

Approach:
1. Identify the `drop` + reassignment pattern — this copies the entire DataFrame minus one column
2. Match against rule: `@{Name: v1} = @{Name: v1}.drop([@{Const(str): c1}], axis=1)`
3. Verify preconditions: `df` is a DataFrame, columns exist
4. Apply RHS: `@{v1}.pop(@{c1})` — modifies in-place, avoids full copy

Output:
```python
# Optimized: pop() modifies in-place, avoiding a full DataFrame copy
# Precondition: df is a DataFrame and columns exist
df.pop('unnecessary_col')
df.pop('temp_data')
# Speedup: up to 1770x on large DataFrames
```

**Example 2: In-place Rename**

User: "Speed up my notebook — I have hundreds of rename calls across cells"
```python
sales_df = sales_df.rename(columns={'old_name': 'new_name', 'qty': 'quantity'})
```

Approach:
1. Match pattern: `@{Name: v1} = @{Name: v1}.rename(columns=@{expr: e1})`
2. Verify preconditions: `sales_df` is a DataFrame, `e1` is a dict
3. Apply inplace transformation

Output:
```python
# Optimized: inplace=True avoids copying the entire DataFrame
sales_df.rename(columns={'old_name': 'new_name', 'qty': 'quantity'}, inplace=True)
# Speedup: avg 22.57x, up to 130x on wide DataFrames
```

**Example 3: Bracket Indexing -> `.loc` Accessor**

User: "My column selection is slow on a DataFrame with 500 columns"
```python
subset = data[['revenue', 'cost', 'margin']]
```

Approach:
1. Match pattern: `@{Name: n1} = @{Name: n2}[@{List: l1}]`
2. Verify: `data` is a DataFrame, all items in list are valid column names
3. Replace with `.loc` label-based accessor

Output:
```python
# Optimized: .loc is specialized for label-based column selection
subset = data.loc[:, ['revenue', 'cost', 'margin']]
# Speedup: avg 1.32x, up to 11.4x depending on DataFrame shape
```

**Example 4: Generating a Custom Rewrite Rule**

User: "I see `df.apply(lambda x: x.strip())` all over my codebase. Can you make a rewrite rule?"

Approach:
1. Discover optimization: `df.apply(lambda x: x.strip())` -> `df.str.strip()` for string Series
2. Generalize variables: abstract `df` to `@{Name: v1}`, `strip` to `@{Name: m1}`
3. Synthesize preconditions

Output — rewrite rule specification:
```
Rule: apply-str-method-to-vectorized
LHS:  @{Name: v1}.apply(lambda x: x.@{Name: m1}())
RHS:  @{v1}.str.@{m1}()
Preconditions:
  - isinstance(@{v1}, pd.Series)
  - @{v1}.dtype == object
  - hasattr(str, @{m1})
```

## Best Practices

- **Do:** Always validate equivalence on representative data before suggesting a rewrite. A 1000x speedup is worthless if it produces wrong results.
- **Do:** Include runtime preconditions with every rule. The preconditions are what make rules safe to apply automatically rather than requiring manual review.
- **Do:** Prefer in-place operations (`inplace=True`, `pop`, direct assignment) over copy-and-reassign patterns when the original code reassigns to the same variable.
- **Do:** Stack multiple compatible rules on the same notebook. Individual rules compose — a notebook can benefit from 13+ distinct rules simultaneously.
- **Avoid:** Applying rules that change observable behavior beyond the target operation. If `drop` is called without reassignment to the same variable, `pop` is not a valid replacement since `pop` returns the removed column.
- **Avoid:** Suggesting `.loc`/`.iloc` rewrites when the user relies on chained indexing for setting values — use `.loc` for both get and set, or warn about SettingWithCopyWarning.
- **Avoid:** Generating rules from single examples. Rules should be validated across multiple DataFrame shapes, sizes, and dtypes to ensure generality.

## Error Handling

- **Equivalence failure:** If original and optimized code produce different results on test inputs, discard the candidate. Do not try to "fix" an incorrect optimization — generate a new one.
- **Precondition too restrictive:** If a rule matches very few cases, relax type constraints (e.g., `@{expr: e1}` instead of `@{Const(str): c1}`) and re-validate.
- **Precondition too permissive:** If a rule produces wrong results on edge cases, strengthen preconditions. Add dtype checks, shape checks, or NaN-handling guards.
- **Multiple rules conflict on same cell:** Apply the rule with the highest measured speedup first. After rewriting, re-match remaining rules against the new code — some may no longer apply.
- **Runtime precondition cannot be evaluated statically:** Wrap the optimized code in a conditional guard that checks the precondition at runtime, falling back to the original code if it fails.

## Limitations

- **Pandas-specific.** The rule format and optimization patterns target Pandas DataFrame/Series operations. The methodology generalizes to other libraries, but the concrete rules in this skill are Pandas-focused.
- **Semantic equivalence is tested, not proven.** Validation uses random test DataFrames, not formal verification. Edge cases (NaN handling, dtype coercion, empty DataFrames) may not be covered by random testing.
- **In-place rewrites change aliasing behavior.** If other variables reference the same DataFrame, in-place modifications (like `pop` or `inplace=True`) will affect those aliases. Only apply in-place rules when the variable is not shared.
- **Discovery yield is low.** LLM-generated candidates have a ~3.79% acceptance rate after validation. Most suggestions are incorrect or offer negligible speedup. The value comes from the surviving rules being reusable.
- **Rules assume standard Pandas.** Custom DataFrame subclasses, monkey-patched methods, or non-standard dtypes may violate rule preconditions silently.

## Reference

[RuleFlow: Generating Reusable Program Optimizations with LLMs](https://arxiv.org/abs/2602.09051v1) — Singh et al., 2026. Focus on Section 3 (SnippetGen validation pipeline), Section 4 (RuleGen's 4-agent decomposition and the `@{Type: var}` pattern syntax), and Table 2 (per-rule speedup measurements). Code: [github.com/ADAPT-uiuc/RuleFlow](https://github.com/ADAPT-uiuc/RuleFlow).