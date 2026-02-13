---
name: "detecting-correcting-hallucinations-llm-generated"
description: >
  Detect and auto-correct hallucinated API calls in LLM-generated Python code using deterministic
  AST analysis and library introspection. Validates function signatures, parameter names, method
  existence, and identifier references against a dynamically-built Knowledge Base of real API specs.
  Use when: "check this generated code for hallucinated APIs", "validate these API calls are real",
  "fix the fake parameters in this code", "verify this code against actual library signatures",
  "detect hallucinations in this Python snippet", "auto-correct wrong API usage in generated code".
---

# Detecting and Correcting Hallucinations in LLM-Generated Code via AST Analysis

This skill enables Claude to apply a deterministic, static-analysis post-processing pipeline to
LLM-generated Python code. Rather than relying on execution or probabilistic LLM self-repair, the
technique parses code into an Abstract Syntax Tree (AST), builds a Knowledge Base (KB) of legitimate
API signatures via Python introspection (`inspect`, `dir()`, `getattr()`), and uses deterministic
rules to detect and auto-correct Knowledge Conflicting Hallucinations (KCHs) — subtle semantic
errors like non-existent parameters, fabricated method names, and wrong argument counts that pass
linters but cause runtime failures.

## When to Use

- When the user asks you to **validate generated Python code** against actual library APIs before running it
- When code uses third-party libraries (pandas, numpy, requests, sklearn, etc.) and the user suspects **hallucinated parameters or methods**
- When debugging a `TypeError: unexpected keyword argument` or `AttributeError: has no attribute` that came from generated code
- When the user wants a **pre-execution safety check** on code snippets produced by an LLM
- When reviewing code that makes many API calls and the user asks "are all these parameters real?"
- When auto-correcting code that has plausible-looking but non-existent API arguments (e.g., `pd.read_csv(file, delimeter=',')` — misspelled `delimiter`)

## Key Technique

**Knowledge Conflicting Hallucinations (KCHs)** are the most dangerous class of LLM code errors
because they are syntactically valid and survive linting. An LLM might generate
`requests.get(url, auth_token='abc')` — this parses fine, but `auth_token` is not a real parameter
of `requests.get()`. The real parameter is `auth`. These errors only surface at runtime, often in
production.

The core insight of this framework is that **most KCHs can be caught deterministically** by
comparing AST nodes against ground-truth API specifications extracted via Python introspection. You
build a Knowledge Base by importing the target library and using `inspect.signature()` to extract
every function's parameter names, types, defaults, and whether they accept `**kwargs`. Then you
parse the generated code into an AST, walk every `ast.Call` node, resolve the callee to a KB entry,
and check: (1) do all keyword arguments exist in the real signature? (2) are positional argument
counts within bounds? (3) does the method/attribute actually exist on the object? Mismatches are
flagged as hallucinations.

Auto-correction uses **edit-distance similarity matching**. When a hallucinated parameter like
`delimeter` is detected, the framework computes string similarity against all valid parameters of
that function and proposes `delimiter` as the fix. Non-existent parameters with no close match are
removed. Missing required parameters are injected with sensible defaults. This deterministic repair
achieves a 77% auto-correction rate without any LLM in the loop.

## Step-by-Step Workflow

1. **Identify target libraries.** Scan the code for `import` statements and `from X import Y`
   patterns. Extract every library, module, and imported name. These define the scope of the KB.

2. **Build the Knowledge Base via introspection.** For each imported library/module, use
   `inspect.signature()` on all public functions and methods to capture parameter names, type
   annotations, default values, and whether `*args`/`**kwargs` are accepted. Use `dir()` and
   `getattr()` to enumerate class attributes, constants, and enum values. Store this as a
   dictionary keyed by fully-qualified name (e.g., `pandas.DataFrame.merge`).

3. **Parse the generated code into an AST.** Use `ast.parse()` on the code string. Handle
   `SyntaxError` gracefully — if parsing fails, report the syntax issue before hallucination
   analysis can proceed.

4. **Walk all `ast.Call` nodes.** For each function/method call in the AST, resolve the callee:
   - `ast.Name` nodes → direct function calls (e.g., `open(...)`)
   - `ast.Attribute` nodes → method calls (e.g., `df.merge(...)`)
   - Chain attribute access to build the fully-qualified name and look it up in the KB.

5. **Validate keyword arguments.** For each call, compare every keyword argument name against the
   KB entry's parameter list. If the function does NOT accept `**kwargs`, any unrecognized keyword
   is a hallucination. If it does accept `**kwargs`, skip keyword validation for that call (the
   function is designed to accept arbitrary keywords).

6. **Validate positional argument counts.** Count positional arguments and compare against the
   function's minimum (required params) and maximum (total params before `*args`). Flag calls with
   too many or too few positional arguments.

7. **Validate attribute/method existence.** For `ast.Attribute` nodes, check whether the attribute
   exists on the resolved object/class in the KB. Flag references to non-existent methods,
   properties, or constants.

8. **Auto-correct detected hallucinations.** For each flagged issue:
   - **Misspelled parameter:** Compute edit distance (Levenshtein or `difflib.get_close_matches`)
     against valid parameters. If a match scores above 0.8 similarity, propose the substitution.
   - **Non-existent parameter with no close match:** Remove the argument.
   - **Missing required parameter:** Insert with its default value, or flag for user review if no
     default exists.
   - **Non-existent method:** Suggest the closest matching method name from the class.

9. **Produce a validation report.** For each hallucination found, report: the line number, the
   offending code fragment, the type of hallucination (wrong param / non-existent method / wrong
   arg count), and the proposed correction with confidence level.

10. **Output corrected code.** Apply all high-confidence fixes and return the corrected code
    alongside the report so the user can review changes.

## Concrete Examples

**Example 1: Hallucinated pandas parameter**

User: "Check this code for hallucinated API calls"

```python
import pandas as pd

df = pd.read_csv('data.csv', delimeter=',', skip_rows=5, encoding='utf-8')
result = df.groupby('category').agg(total=('amount', 'sum')).reset_index(drop_level=True)
```

Approach:
1. Build KB: introspect `pd.read_csv` — valid params include `delimiter`, `skiprows`, `encoding`, etc.
2. Parse AST, find `pd.read_csv(...)` call with keywords `delimeter`, `skip_rows`, `encoding`.
3. `delimeter` → not in signature → closest match: `delimiter` (edit distance 1) → **fix: rename**
4. `skip_rows` → not in signature → closest match: `skiprows` (edit distance 1) → **fix: rename**
5. `encoding` → valid parameter → **pass**
6. Check `reset_index(drop_level=True)` → `drop_level` not in signature → closest: `level` (0.6 similarity, too low) → **flag for removal**

Output:
```
Hallucination Report:
  Line 3: pd.read_csv — 'delimeter' is not a valid parameter.
          Fix: rename to 'delimiter' (confidence: high)
  Line 3: pd.read_csv — 'skip_rows' is not a valid parameter.
          Fix: rename to 'skiprows' (confidence: high)
  Line 4: DataFrame.reset_index — 'drop_level' is not a valid parameter.
          Fix: remove argument (confidence: medium, no close match found)

Corrected code:
  df = pd.read_csv('data.csv', delimiter=',', skiprows=5, encoding='utf-8')
  result = df.groupby('category').agg(total=('amount', 'sum')).reset_index()
```

**Example 2: Non-existent method on a library object**

User: "Validate the sklearn API calls in this snippet"

```python
from sklearn.ensemble import RandomForestClassifier

model = RandomForestClassifier(n_estimators=100, max_features='auto', verbose_level=1)
model.fit_transform(X_train, y_train)
predictions = model.predict_classes(X_test)
```

Approach:
1. Build KB: introspect `RandomForestClassifier` — valid params include `n_estimators`,
   `max_features`, `verbose`, etc. Valid methods: `fit`, `predict`, `predict_proba`, `score`, etc.
2. `verbose_level` → not a valid param → closest: `verbose` (similarity 0.82) → **fix: rename**
3. `fit_transform` → not a method on RandomForestClassifier → closest: `fit` → **flag as hallucination**
4. `predict_classes` → not a method → closest: `predict` (similarity 0.76) → **fix: rename**

Output:
```
Hallucination Report:
  Line 3: RandomForestClassifier() — 'verbose_level' is not a valid parameter.
          Fix: rename to 'verbose' (confidence: high)
  Line 4: RandomForestClassifier.fit_transform — method does not exist.
          Suggestion: use 'fit' instead (confidence: medium, review needed)
  Line 5: RandomForestClassifier.predict_classes — method does not exist.
          Fix: rename to 'predict' (confidence: high)
```

**Example 3: requests library with fabricated parameters**

User: "Are these requests calls using real parameters?"

```python
import requests

resp = requests.get('https://api.example.com', auth_token='abc', timeout=30, retry_count=3)
data = resp.json(parse_dates=True)
```

Approach:
1. `requests.get` accepts `**kwargs` passed to `requests.request` — introspect `request()`:
   valid named params include `params`, `headers`, `auth`, `timeout`, `cookies`, etc.
   However, `requests.get` passes unknown kwargs to `Session.request`, which does NOT accept
   arbitrary kwargs beyond its defined signature.
2. `auth_token` → not valid → closest: `auth` → **fix: rename** (but semantically different — flag for review)
3. `timeout` → valid → **pass**
4. `retry_count` → not valid → no close match → **flag for removal**
5. `resp.json()` → introspect `Response.json` — no `parse_dates` param → **flag for removal**

Output:
```
Hallucination Report:
  Line 3: requests.get — 'auth_token' is not a valid parameter.
          Closest match: 'auth' (review: semantics differ, auth expects a tuple)
  Line 3: requests.get — 'retry_count' is not a valid parameter.
          Fix: remove argument (no close match)
  Line 4: Response.json — 'parse_dates' is not a valid parameter.
          Fix: remove argument (this is a pandas concept, not requests)
```

## Best Practices

- **Do:** Always check whether a function accepts `**kwargs` before flagging unknown keywords.
  Functions with `**kwargs` intentionally accept arbitrary arguments — false-flagging these
  destroys trust in the tool.
- **Do:** Use `inspect.signature()` over manual signature parsing. It handles decorated functions,
  built-in functions (via `__doc__` fallback), and C-extension methods more reliably.
- **Do:** Report confidence levels with every fix. High-confidence fixes (edit distance <= 2,
  similarity > 0.85) can be auto-applied. Lower-confidence fixes should be presented as
  suggestions for user review.
- **Do:** Build the KB dynamically for the specific library version installed. API signatures
  change between versions — hardcoded signatures go stale.
- **Avoid:** Running the generated code to test whether parameters work. The whole point of this
  approach is non-executing, static validation. Execution introduces security risks with untrusted
  code.
- **Avoid:** Flagging parameters on functions you cannot introspect (C extensions without stubs,
  dynamically-generated APIs). When introspection fails, note the gap in the report rather than
  guessing.

## Error Handling

- **`SyntaxError` during `ast.parse()`:** Report the syntax error with line number. AST-based
  hallucination detection cannot proceed on unparseable code. Suggest the user fix syntax first.
- **Library not installed:** If an imported library is not available in the current environment,
  report that the KB cannot be built for it. Suggest `pip install` or note which calls could not
  be validated.
- **Introspection failure on C extensions:** Some functions (e.g., built-in C modules) do not
  expose full signatures via `inspect`. Fall back to `__doc__` parsing or type stub files
  (`.pyi`) if available. Report reduced confidence for these entries.
- **Dynamic attribute access:** Code using `getattr(obj, name)` or `**unpacked_dict` cannot be
  statically validated. Skip these nodes and note them as unanalyzable in the report.
- **Ambiguous call resolution:** If the callee cannot be resolved to a single KB entry (e.g.,
  variable reassignment obscures the type), flag the call as unresolvable rather than guessing.

## Limitations

- **`**kwargs`-heavy APIs** (Flask, Django, Click) intentionally accept arbitrary keyword
  arguments. The framework cannot distinguish hallucinated kwargs from legitimate ones in these
  cases. This is a fundamental limitation of static analysis without type narrowing.
- **Dynamically-generated APIs** (e.g., SQLAlchemy models, protocol buffers, metaprogramming)
  create methods at runtime that introspection cannot discover statically. These will produce
  false negatives (missed hallucinations) or false positives (valid calls flagged as hallucinations).
- **Cross-function data flow** is not tracked. If a variable is reassigned to a different type
  between its creation and a method call, the framework may validate against the wrong class.
- **Only Python is supported.** The technique relies on Python's `ast` and `inspect` modules.
  Adapting to other languages requires equivalent introspection infrastructure.
- **Semantic correctness is out of scope.** The framework catches wrong parameter *names* but not
  wrong parameter *values*. Passing `delimiter='\t'` when the user meant `','` is a logic error,
  not a hallucination.
- **Auto-correction is conservative.** The 77% correction rate means ~23% of hallucinations are
  detected but not auto-fixed. These require human review.

## Reference

**Paper:** Khati, D., Rodriguez-Cardenas, D., Pantzer, P., & Poshyvanyk, D. (2026). *Detecting
and Correcting Hallucinations in LLM-Generated Code via Deterministic AST Analysis.* FORGE 2026.
[arXiv:2601.19106](https://arxiv.org/abs/2601.19106v1)

Key takeaway: deterministic AST validation against introspected API signatures achieves 100%
precision at 87.6% recall for detecting hallucinated API usage — outperforming probabilistic
LLM self-repair while being fully reproducible.