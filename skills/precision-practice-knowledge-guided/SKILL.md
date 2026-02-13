---
name: "precision-practice-knowledge-guided"
description: >
  Generate industrial-grade code summaries using the ExpSum knowledge-guided approach:
  function metadata extraction, domain term retrieval, function categorization, and
  constraint-driven prompting. Produces documentation that meets real developer expectations
  rather than generic semantic descriptions.
  Trigger phrases:
  - "Summarize this function/method/class"
  - "Generate code documentation"
  - "Write doc comments for this code"
  - "Document this API"
  - "Generate Javadoc/JSDoc/docstring"
  - "Improve these code summaries"
---

# Precision-Practice Knowledge-Guided Code Summarization

This skill enables Claude to generate code summaries that meet industrial documentation
standards by applying the ExpSum framework from Li et al. (2026). Rather than producing
generic paraphrases of code logic, this approach extracts structured function metadata,
retrieves project-specific domain terminology, classifies functions into behavioral
categories (field, procedural, constructor, callback, utility), and applies category-specific
linguistic templates. Research with HarmonyOS documentation experts showed that over 57% of
summaries from state-of-the-art LLMs were rejected for violating implicit industrial
standards -- this skill addresses those three core failure modes: wrong domain terms,
missing function category indication, and excessive implementation detail.

## When to Use

- When the user asks to document functions, methods, or classes and wants professional-quality summaries
- When generating JSDoc, Javadoc, docstrings, or header comments for a codebase
- When the user says existing auto-generated documentation is "too generic" or "doesn't match our style"
- When documenting an API surface where consistent terminology and categorization matter
- When reviewing or improving existing code summaries for accuracy and conciseness
- When the user is preparing code for a documentation review or public SDK release
- When batch-generating summaries across a module or package

## Key Technique

**The core insight:** Most LLM-generated code summaries fail industrial review not because
they are semantically wrong, but because they violate three implicit expectations: (1) they
use incorrect or inconsistent domain terminology, (2) they fail to indicate what category
of function is being described, and (3) they include redundant implementation details that
clutter comprehension. ExpSum addresses all three through a structured four-phase pipeline.

**How it works:** First, each function is modeled as structured metadata -- its signature
(name, parameters, return type), context (file path, package, dependencies), and behavior
(control flow skeleton, I/O patterns). Uninformative metadata is filtered out (placeholder
values, generic parameter names, stopword-only entries). Next, a cascaded knowledge
retrieval system matches the function's package context against a project knowledge base to
surface the correct domain terms (e.g., "RDBStore" means different things in different
packages). Finally, a constraint-driven prompt classifies the function into one of five
categories and applies category-specific linguistic patterns: field functions use "Indicates
whether..." or "Obtains the...", procedural functions use active verbs like "Sets..." or
"Deletes...", constructors use "Creates a...", and so on.

**Why it matters:** On industrial HarmonyOS benchmarks, this approach improved BLEU-4 by
26.7% and ROUGE-L by 20.1% over baselines. More importantly, expert acceptance rates rose
substantially because summaries matched the documentation conventions developers actually
expect.

## Step-by-Step Workflow

1. **Extract function metadata.** Parse the function signature: name, parameters (names and
   types), return type, access modifiers, annotations/decorators. Record the file path,
   module/package name, and any import dependencies. Capture the control-flow skeleton
   (conditionals, loops, early returns) without implementation details.

2. **Filter uninformative metadata.** Remove parameters with placeholder names (`arg0`,
   `param1`, `_`), unknown types (`any`, `object` without context), empty default values,
   and generic stopword-only descriptions. Retain only metadata that carries discriminating
   semantic signal.

3. **Identify the project's domain vocabulary.** Scan the surrounding codebase for
   domain-specific terms: CamelCase identifiers, abbreviations, project-specific nouns that
   resist synonym substitution (e.g., "RDBStore", "AbilityContext", "BundleInfo"). Use
   package-level README files, API docs, or module docstrings as the knowledge base. Match
   terms by path-context overlap (the function's package path vs. the term's source path).

4. **Rank and deduplicate domain terms.** From candidate terms, rank by lexical relevance
   to the function's metadata (TF-IDF cosine similarity between the function's tokens and
   each term's documentation). Remove near-duplicate terms (those sharing 75%+ token
   overlap) to avoid biasing the summary toward one variant.

5. **Classify the function category.** Determine which category the function belongs to
   using these decision criteria:
   - **Field function:** Empty/void return body, noun-like name, represents a property or
     enumeration value. Often boolean (use "Indicates whether...") or data (use "Obtains
     the...").
   - **Procedural function:** Active verb name, modifies state, has side effects. Use
     imperative verbs: "Sets...", "Deletes...", "Sends...".
   - **Constructor/factory:** Creates and returns an instance. Use "Creates a..." or
     "Constructs a...".
   - **Callback/handler:** Name contains "on", "handle", "listener", or is passed as a
     parameter. Use "Called when..." or "Handles the...".
   - **Utility function:** Pure computation, no side effects, often static. Use "Computes...",
     "Converts...", "Parses...".

6. **Draft the summary using category-specific templates.** Compose a one-sentence summary
   that (a) starts with the category-appropriate verb form, (b) incorporates the top-ranked
   domain terms, and (c) omits implementation details already implied by the function's
   module or class context.

7. **Validate and refine.** Re-check the draft against the function metadata: Does the
   category match the actual signature and behavior? Are the domain terms used in their
   correct contextual meaning? Is there any redundant detail that duplicates information
   available from the class/module context? Fix any mismatches.

8. **Format for the target documentation system.** Output the summary in the appropriate
   format: JSDoc `/** */`, Python docstring `"""..."""`, Javadoc, Rust `///`, etc. Include
   `@param`, `@returns`, and `@throws` tags only when they add information beyond what the
   types already convey.

## Concrete Examples

**Example 1: Field/Enum Function**

User: "Document this TypeScript function."

```typescript
// src/ability/abilityContext.ts
export enum StartupVisibility {
  STARTUP_HIDE = 0,
  STARTUP_SHOW = 1,
}
```

Approach:
1. Metadata: Enum named `StartupVisibility` with two members, in `ability` package.
2. Domain terms from package: "ability" (HarmonyOS concept), "startup" (application lifecycle).
3. Category: Field function (enumeration).
4. Template: "Enumerates {domain concept}."

Output:
```typescript
/**
 * Enumerates the visibility options for an ability during startup.
 */
export enum StartupVisibility {
  /** The ability is hidden during startup. */
  STARTUP_HIDE = 0,
  /** The ability is visible during startup. */
  STARTUP_SHOW = 1,
}
```

Note: A naive LLM might write "Sets the startup visibility" (wrong category -- procedural
verb for a field/enum) or "Defines a constant for whether the app is shown or not when it
starts up, with 0 meaning hidden and 1 meaning shown" (redundant implementation detail).

---

**Example 2: Procedural Function with Domain Terms**

User: "Write a doc comment for this Java method."

```java
// com/storage/rdb/RDBStore.java
public void delete(RdbPredicates predicates) {
    if (predicates == null) throw new IllegalArgumentException();
    nativeDelete(storeId, predicates.toNative());
}
```

Approach:
1. Metadata: Method `delete` in `RDBStore` class, takes `RdbPredicates`, void return, in
   `storage/rdb` package.
2. Domain terms: "relational database store" (from RDB package docs), "predicates" (query
   condition object specific to this API).
3. Category: Procedural (active verb, modifies state, side effects).
4. Template: "Deletes {target} from {context} based on {condition}."

Output:
```java
/**
 * Deletes rows from the relational database store that match the specified predicates.
 *
 * @param predicates the query conditions identifying rows to delete
 * @throws IllegalArgumentException if predicates is null
 */
public void delete(RdbPredicates predicates) {
```

Note: Avoids mentioning `nativeDelete`, `storeId`, or `toNative()` -- these are
implementation details that clutter the summary.

---

**Example 3: Utility Function in Python**

User: "Generate a docstring for this function."

```python
# utils/metrics.py
def compute_bleu(reference: list[str], hypothesis: list[str], max_n: int = 4) -> float:
    precisions = []
    for n in range(1, max_n + 1):
        ref_ngrams = extract_ngrams(reference, n)
        hyp_ngrams = extract_ngrams(hypothesis, n)
        clipped = sum(min(hyp_ngrams[g], ref_ngrams.get(g, 0)) for g in hyp_ngrams)
        total = max(sum(hyp_ngrams.values()), 1)
        precisions.append(clipped / total)
    brevity = min(1.0, len(hypothesis) / max(len(reference), 1))
    return brevity * geometric_mean(precisions)
```

Approach:
1. Metadata: Function `compute_bleu`, takes reference/hypothesis token lists, returns float.
2. Domain terms: "BLEU score" (standard NLP metric), "n-gram precision", "brevity penalty".
3. Category: Utility (pure computation, no side effects, static-like).
4. Template: "Computes {metric} between {inputs}."

Output:
```python
def compute_bleu(reference: list[str], hypothesis: list[str], max_n: int = 4) -> float:
    """Computes the BLEU score between a reference and hypothesis token sequence.

    Args:
        reference: The ground-truth token sequence.
        hypothesis: The candidate token sequence to evaluate.
        max_n: Maximum n-gram order for precision calculation. Defaults to 4.

    Returns:
        The BLEU score as a float in [0, 1].
    """
```

Note: Does not restate the algorithm (n-gram extraction, clipping, geometric mean) because
that is visible in the code. The summary tells a developer *what* the function computes,
not *how*.

## Best Practices

**Do:**
- Start every summary with a verb appropriate to the function's category. Field functions
  get "Obtains", "Indicates", "Enumerates"; procedural functions get active verbs matching
  their action; constructors get "Creates"; callbacks get "Called when".
- Use the project's established domain terms, not generic synonyms. If the codebase calls
  it an "ability," do not write "feature" or "capability."
- Omit implementation details that are already visible in the code or implied by the
  class/module hierarchy. A method on `DatabaseConnection` does not need to say "connects
  to the database."
- Verify function category against the actual signature. A method named `getX()` that
  modifies state is procedural despite its getter-like name.

**Avoid:**
- Writing summaries that simply paraphrase the function name with extra words
  ("getBatteryLevel" -> "Gets the battery level" without adding context like "as a
  percentage" or noting side effects).
- Including parameter type information in the description when it is already expressed in
  the signature's type annotations.
- Using different terminology for the same concept across summaries in the same module.
  Consistency across a package is a core industrial requirement.
- Generating multi-sentence summaries when a single precise sentence suffices. Brevity with
  accuracy outperforms verbose explanations.

## Error Handling

- **Ambiguous function category:** When a function does not clearly fit one category (e.g.,
  a getter with side effects), default to the category matching its primary *intent* as
  indicated by its name and caller context. Note the ambiguity in your reasoning but commit
  to one category in the output.
- **No domain knowledge available:** If no package docs, README, or surrounding code
  provides domain terms, fall back to the function's own identifiers (class name, parameter
  names) as domain vocabulary. Flag to the user that domain-specific terminology may need
  manual review.
- **Overloaded functions:** When multiple overloads exist, summarize the shared behavior in
  the base summary and note parameter-specific differences in `@param` tags, not in the
  main sentence.
- **Generated code or boilerplate:** For auto-generated code (e.g., protobuf stubs, ORM
  models), keep summaries minimal -- one sentence stating the entity's role. Do not
  fabricate behavioral descriptions for pass-through methods.

## Limitations

- This approach works best when there is an existing knowledge base (package docs, README
  files, API references) to draw domain terms from. For brand-new greenfield code with no
  documentation context, the domain term retrieval phase has limited material to work with.
- Function category classification relies on naming conventions and signature patterns. Code
  that uses unconventional naming (e.g., all lowercase, cryptic abbreviations) may be
  misclassified and require manual correction.
- The technique is optimized for function/method-level summaries. Class-level or
  module-level documentation requires additional architectural context beyond what this
  workflow provides.
- Summaries are only as accurate as the metadata extraction. Dynamically typed languages
  with no type hints provide less signal for category classification and parameter
  documentation.

## Reference

Li, J., Chen, S., Jin, S., & Xie, X. (2026). *Precision in Practice: Knowledge Guided
Code Summarizing Grounded in Industrial Expectations.* arXiv:2602.03400v1.
https://arxiv.org/abs/2602.03400v1

Read this paper for: the empirical evidence that 57%+ of LLM summaries fail industrial
review, the three core developer expectations (domain terms, function categorization,
detail mitigation), and the full ExpSum four-phase pipeline with constraint-driven prompting.