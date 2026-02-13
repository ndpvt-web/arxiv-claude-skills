---
name: "spell-synthesis-programmatic-edits"
description: "Automate library migrations by synthesizing reusable code transformation scripts. Uses LLM-generated migration examples distilled into structured, testable rewrite rules (PolyglotPiranha / comby / semgrep patterns). Triggers: 'migrate from X to Y library', 'generate migration script', 'automated API migration', 'rewrite all uses of library X', 'create transformation rules for migration', 'synthesize code edits for library swap'."
---

This skill enables Claude to perform automated library-to-library API migrations by applying the SPELL methodology: instead of rewriting every call site by hand or relying on a one-shot LLM rewrite, Claude generates diverse migration examples first, then generalizes those examples into reusable transformation rules that can be applied deterministically across an entire codebase. The result is structured, testable, repeatable migration logic — not fragile find-and-replace or non-reproducible LLM output.

## When to Use

- When the user asks to migrate a codebase from one library to another (e.g., `requests` to `httpx`, `logging` to `loguru`, `json` to `orjson`, `argparse` to `click`)
- When the user needs to replace deprecated API calls with modern equivalents across many files
- When the user wants a migration script they can review, test, and re-run — not a one-time LLM rewrite
- When the user asks to generate comby, semgrep, or PolyglotPiranha rules for a code transformation
- When a migration involves structural changes beyond simple renaming (e.g., different function signatures, changed return types, added boilerplate)
- When the user wants to validate a migration against existing tests before applying it project-wide

## Key Technique

**SPELL (Synthesis of Programmatic Edits using LLMs)** separates migration into two phases: *example generation* and *rule synthesis*. In phase one, the LLM generates multiple concrete migration examples — small, self-contained code snippets that use the source library, paired with equivalent code using the target library, plus tests validating both. This produces a diverse corpus of before/after pairs covering different API usages. Crucially, these examples are *validated*: each must pass its test suite in both the source and target versions.

In phase two, an agentic loop examines the diffs between source and target examples, identifies recurring transformation patterns, and generalizes them into rewrite rules using template variables (e.g., `:[var]` captures any expression). Rules are composed into a directed graph where edges define application order and scope — for example, "after replacing an import, replace all constructor calls within the same file." The agent iterates up to 10 rounds, testing each candidate rule set against the validation suite and refining based on error feedback.

This two-phase approach has a key advantage: it converts the LLM's latent knowledge of API correspondences into *deterministic, auditable transformation rules*. Once synthesized, the rules apply without further LLM calls, produce consistent results, and can be committed to version control alongside the migration.

## Step-by-Step Workflow

1. **Identify the migration pair.** Clarify the exact source and target libraries, including specific submodules (e.g., `cryptography.fernet` to `Crypto.Cipher` from pycryptodome, not just "cryptography to pycryptodome"). Pin versions if relevant.

2. **Enumerate the source API surface.** Scan the codebase for all imports and usages of the source library. Group call sites by API function/class (e.g., `Fernet()`, `cipher.encrypt()`, `cipher.decrypt()`). This defines the scope of the migration.

3. **Generate diverse migration examples.** For each API usage pattern found, produce 3-5 small, self-contained code snippets that:
   - Import and use the source library in a realistic way
   - Include a test (pytest-style) that validates behavior
   - Show the equivalent target-library implementation passing the same test
   Vary the examples: different variable names, different argument combinations, edge cases.

4. **Validate examples.** Confirm each source snippet passes its tests with the source library, and each target snippet passes the same tests with the target library. Discard any pair where tests fail.

5. **Diff and extract atomic transformation patterns.** Compare each validated source/target pair. Identify the minimal set of changes:
   - Import replacements (e.g., `from cryptography.fernet import Fernet` → `from Crypto.Cipher import AES`)
   - Constructor/factory call rewrites
   - Method call signature changes
   - Added boilerplate (e.g., padding setup for crypto migrations)
   Generalize concrete names into template variables: replace specific variable names with `:[var]`, specific arguments with `:[args]`.

6. **Compose rules into an ordered graph.** Define the execution order:
   - Typically: import rules first, then declaration/constructor rules, then method call rules
   - Specify scope: after an import rule fires, apply constructor rules *within the same file*
   - Handle dependencies: if a new import must exist before a method rewrite makes sense, encode that edge

7. **Express rules in the target format.** Write the rules as:
   - **comby patterns** (widely available): `:[x].encrypt(:[data])` → `:[iv] + :[x].encrypt(pad(:[data], AES.block_size))`
   - **semgrep rules** (YAML): pattern + fix fields with metavariables
   - **PolyglotPiranha TOML** (if using Piranha): `[[rules]]` blocks with `query`, `replace_node`, `replace`
   - **sed/codemod scripts** (minimal tooling): for simple renames

8. **Test the synthesized rules against the validation examples.** Apply the rules to the original source examples and verify the output matches the target examples (or at least passes the tests). If failures occur, refine: broaden a pattern, add a missing rule, fix ordering.

9. **Apply rules to the real codebase.** Run the finalized transformation rules across all files. Review the diff. Run the project's existing test suite to catch regressions.

10. **Handle residual cases manually.** Some migrations involve patterns too complex for structural rewrite rules (e.g., behavioral changes requiring new control flow). Flag these for manual review rather than forcing an incorrect automated rewrite.

## Concrete Examples

**Example 1: Migrate `requests` to `httpx`**

User: "Migrate our codebase from requests to httpx."

Approach:
1. Scan for all `import requests` and `requests.get/post/put/delete/session` usages.
2. Generate example pairs:

```python
# SOURCE (requests)
import requests
def fetch_data(url):
    resp = requests.get(url, params={"q": "test"})
    resp.raise_for_status()
    return resp.json()

# TARGET (httpx)
import httpx
def fetch_data(url):
    resp = httpx.get(url, params={"q": "test"})
    resp.raise_for_status()
    return resp.json()
```

3. Extract transformation rules (comby format):

```
# Rule 1: Replace import
import requests  →  import httpx

# Rule 2: Replace simple method calls
requests.:[method](:[args])  →  httpx.:[method](:[args])

# Rule 3: Replace Session usage
requests.Session()  →  httpx.Client()
```

4. Handle divergent APIs separately — `httpx` uses `httpx.Client()` not `requests.Session()`, and streaming differs. Flag `requests.get(..., stream=True)` for manual review since httpx uses `httpx.stream()`.

Output rules (semgrep YAML):
```yaml
rules:
  - id: requests-to-httpx-import
    pattern: import requests
    fix: import httpx
    languages: [python]

  - id: requests-to-httpx-call
    pattern: requests.$METHOD(...)
    fix: httpx.$METHOD(...)
    languages: [python]

  - id: requests-to-httpx-session
    pattern: requests.Session()
    fix: httpx.Client()
    languages: [python]
```

---

**Example 2: Migrate `json` to `orjson`**

User: "Switch our JSON handling from stdlib json to orjson for performance."

Approach:
1. Scan for `import json` and all `json.loads`, `json.dumps`, `json.load`, `json.dump` calls.
2. Key API differences: `orjson.dumps()` returns `bytes` not `str`; `orjson.loads()` accepts both; `orjson` has no `dump`/`load` (file-based) — those need wrapping.

Generate examples and distill rules:

```
# Rule 1: Replace import
import json  →  import orjson

# Rule 2: json.loads (direct replacement)
json.loads(:[arg])  →  orjson.loads(:[arg])

# Rule 3: json.dumps (must decode bytes to str if used as str)
json.dumps(:[arg])  →  orjson.dumps(:[arg]).decode()

# Rule 4: json.dump to file (no direct equivalent — synthesize)
json.dump(:[data], :[file])  →  :[file].write(orjson.dumps(:[data]))

# Rule 5: json.load from file
json.load(:[file])  →  orjson.loads(:[file].read())
```

3. Test these against generated examples, then apply project-wide. Run tests — the bytes/str boundary is the most common failure point; refine Rule 3 if `dumps` result is written to a file rather than used as a string.

---

**Example 3: Migrate `logging` to `loguru`**

User: "Replace all standard logging with loguru across the project."

Approach:
1. Find all `import logging` / `logging.getLogger` / `logger.info/debug/warning/error` patterns.
2. Key structural change: loguru uses a singleton `logger` imported directly — no `getLogger()` calls.

Synthesized rules:
```
# Rule 1: Replace import
import logging  →  from loguru import logger

# Rule 2: Remove getLogger declarations
:[name] = logging.getLogger(:[args])  →  # (delete line — loguru uses global logger)

# Rule 3: Replace logging.info etc. (module-level calls)
logging.:[level](:[args])  →  logger.:[level](:[args])

# Rule 4: basicConfig removal
logging.basicConfig(:[args])  →  # (delete — loguru auto-configures)
```

Post-apply check: verify that code using `logger.setLevel()` or custom formatters is flagged for manual review, since loguru handles configuration differently.

## Best Practices

- **Do:** Generate at least 3-5 diverse examples per API pattern before attempting to generalize. Diversity in variable names and argument combinations reveals which parts of the pattern are fixed vs. variable.
- **Do:** Validate every example pair with tests before using them for rule extraction. An incorrect example produces an incorrect rule.
- **Do:** Start with the simplest, highest-frequency API calls first (imports, basic function calls). Get those rules passing before tackling complex structural changes.
- **Do:** Express rules in a standard format (semgrep, comby) so the user can version-control, review, and re-apply them. Avoid ad-hoc regex replacements.
- **Avoid:** Trying to handle every edge case with a single transformation rule. Complex migrations (e.g., pandas to polars) may require dozens of rules and some manual intervention.
- **Avoid:** Applying rules without running the project's test suite afterward. Structural rewrites can produce syntactically valid but semantically wrong code.
- **Avoid:** Generating migration rules from a single example. One example cannot distinguish fixed syntax from variable content — you need multiple examples to identify what generalizes.

## Error Handling

- **Tests fail after applying rules:** Inspect the failing test to identify which transformation produced incorrect output. Common cause: a rule is too broad (matches code that shouldn't change) or too narrow (misses a variant). Refine the pattern or add a new rule for the missed case.
- **Import conflicts:** After replacing imports, the old library may still be imported elsewhere (e.g., in test mocks). Scan for residual imports and handle them separately.
- **Partial API coverage:** If the source library's API surface includes functions with no target equivalent, flag these explicitly rather than silently skipping them. Create a list of unhandled call sites for the user.
- **Rule ordering issues:** If a later rule depends on changes made by an earlier rule (e.g., a method rewrite depends on a constructor rewrite), ensure the graph order is correct. Symptoms: rules that work in isolation but fail when composed.
- **Template variable over-capture:** A pattern like `:[x].encrypt(:[data])` might match unrelated objects. Add context constraints (e.g., require `:[x]` to appear in a line following a specific constructor) or use typed metavariables if the tool supports them.

## Limitations

- **Behavioral divergence:** When source and target libraries have fundamentally different semantics (not just syntax), rewrite rules cannot bridge the gap. For example, `pandas` to `polars` involves lazy vs. eager evaluation — this requires algorithmic changes, not pattern rewrites. The SPELL paper confirmed 0 valid migrations for this pair.
- **Complex control flow changes:** If a migration requires adding error handling, retry logic, or restructuring callbacks, structural rewrite rules are insufficient. These need manual implementation.
- **Dynamic API usage:** Code that constructs method names dynamically (e.g., `getattr(lib, method_name)(args)`) cannot be matched by static patterns.
- **Cross-file dependencies:** Rules typically operate per-file. Migrations that require coordinating changes across multiple files (e.g., a shared configuration object) need orchestration beyond single-rule application.
- **Language support:** PolyglotPiranha primarily supports Java, Swift, Kotlin, Go, JavaScript. For Python, comby or semgrep are more practical choices for expressing the synthesized rules.

## Reference

[SPELL: Synthesis of Programmatic Edits using LLMs](https://arxiv.org/abs/2602.01107v1) — Ramos et al., 2026. Focus on Section 3 (methodology: example generation + agentic rule synthesis), Section 4 (PolyglotPiranha rule format and graph semantics), and Table 1 (results across 10 Python library migration pairs showing 61.6% synthesis success rate).