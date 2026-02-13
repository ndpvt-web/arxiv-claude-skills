---
name: "doc2spec-synthesizing-formal-programming"
description: "Synthesize formal programming specifications from natural-language API docs using grammar induction. Extracts rules from documentation, induces a domain-specific specification grammar (EBNF), and generates validated formal specs. Use when: 'formalize this API documentation', 'extract specifications from these docs', 'generate formal rules from this README', 'convert these requirements to formal specs', 'induce a grammar for these programming rules', 'validate API usage against documentation'."
---

# Doc2Spec: Formal Specification Synthesis via Grammar Induction

This skill enables Claude to convert natural-language programming documentation (API docs, READMEs, RFCs, ERC standards) into formal, machine-checkable specifications by first inducing a domain-specific grammar from the documentation and then generating specifications that conform to that grammar. The technique comes from the Doc2Spec multi-agent framework (Xia et al., 2026), which demonstrated that automated grammar induction constrains the specification space, enforces consistent representations, and produces higher-quality specs than direct LLM translation. It works across Solidity, Rust, Java, and other languages.

## When to Use

- When the user has API documentation and wants formal pre/post-conditions or usage contracts extracted from it
- When the user asks to "formalize" or "specify" rules from a natural-language source (README, RFC, ERC standard, man page, Javadoc)
- When the user wants to validate that code conforms to rules stated in documentation
- When building a linter, static analyzer, or verification harness and needs a specification DSL derived from docs
- When the user has inconsistent or scattered documentation rules and wants them unified into a single formal grammar
- When reviewing smart contract implementations against their ERC specification text
- When converting Rust safety invariants or Java API contracts from prose to checkable assertions

## Key Technique

**Grammar induction before specification generation.** The core insight of Doc2Spec is that directly asking an LLM to translate natural-language rules into formal specs produces inconsistent results -- equivalent semantics get encoded differently, predicate names drift, and the output format varies. Instead, Doc2Spec first *induces* a domain-specific specification grammar (in EBNF) from the rules, establishing a fixed vocabulary of **sorts** (types of programming entities like `Pointer`, `Allocator`, `Function`) and **predicates** (properties like `AllocatedByAllocator(Pointer, Allocator)` or `ThrowsOnUnauthorized(Function, Condition)`). All specs are then generated to conform to this grammar, validated by parsing.

**The universal DSL template.** Every specification follows a constrained structure: an optional Boolean condition plus a predicate check. The template grammar is:

```ebnf
<rule>   ::= <check> ["if" <expr>] ";"
<check>  ::= "check(" <predicate> "," BOOLEAN ")"
<expr>   ::= <unary> | <binary>
<unary>  ::= "not" <value>
<binary> ::= <value> ("and" | "or") <value>
<value>  ::= BOOLEAN | <predicate> | <expr>
```

Domain-specific sorts and predicates are plugged into this template. This constraint reduces average token count (~30% smaller specs) and enforces consistency across all rules in a document.

**Multi-agent pipeline with validation loops.** The workflow uses five specialized stages: entity localization, attribute extraction, NL rule extraction, grammar induction, and formal spec generation. Each stage validates its output (JSON schema checks, Lark parser validation for grammar conformance) and retries up to three times on failure. This makes the process robust to LLM formatting errors.

## Step-by-Step Workflow

1. **Identify the source document and target entities.** Read the documentation (API doc, ERC spec, Rustdoc, Javadoc). List every programming entity mentioned: functions, structs, classes, interfaces, parameters, return types. Use regex patterns or structural cues (headings, code blocks, function signatures) to locate them.

2. **Extract entity attributes.** For each entity, determine its attribute schema: name, parameter types, return type, visibility, mutability, ownership semantics. First discover the full schema across all entities, then instantiate values per entity.

3. **Extract natural-language rules.** Scan the documentation for constraint-bearing sentences. Classify each sentence as a rule or non-rule using dual evaluation: (a) does it describe a MUST/SHOULD/SHALL requirement? (b) does it constrain behavior, state, or usage? Aggregate scores using a confidence threshold. Discard purely descriptive text.

4. **Induce sorts from extracted rules.** Identify the types of programming entities referenced across all rules. These become the *sorts* of the grammar -- e.g., `Pointer`, `Allocator`, `Token`, `Address`, `Function`. Process rules incrementally; only introduce a new sort when no existing sort covers the entity.

5. **Induce predicates from extracted rules.** For each rule, identify the property or constraint it asserts and define a predicate with typed parameters -- e.g., `AllocatedByAllocator(Pointer, Allocator) -> Bool`. Reuse existing predicates when semantically equivalent rules appear. The set of sorts + predicates forms the domain-specific extension to the DSL template.

6. **Assemble the full EBNF grammar.** Combine the universal DSL template with the induced sorts and predicates to produce a complete grammar. Validate it parses correctly using a parser generator (Lark, ANTLR, or manual EBNF check).

7. **Generate formal specifications for each rule.** Translate each natural-language rule into a formal spec that conforms to the assembled grammar. Use the `check(predicate, BOOLEAN) [if expr]` structure. Validate each spec by parsing it against the grammar.

8. **Handle validation failures.** If a generated spec fails to parse against the grammar, feed the error message and original rule back for retry (up to 3 attempts). If retries fail, flag the rule for manual review and continue with remaining rules.

9. **Output the specification bundle.** Produce: (a) the induced grammar in EBNF, (b) the list of sorts with descriptions, (c) the predicate catalog with typed signatures, (d) the formal specs mapped to their source NL rules, and (e) any rules that could not be formalized.

10. **Optionally generate executable checks.** Convert the formal specs into runtime assertions, property-based tests, or static analysis rules in the target language (Solidity `require()` statements, Rust `assert!()` macros, Java `assert` or JML annotations).

## Concrete Examples

**Example 1: ERC-20 Token Standard**

User: "Formalize the rules in the ERC-20 specification for the `transferFrom` function."

Approach:
1. Extract entities: `transferFrom(address _from, address _to, uint256 _value)`, `allowance`, `Transfer` event, `balanceOf`
2. Extract NL rules from ERC-20 text:
   - "The function SHOULD throw unless the `_from` account has deliberately authorized the sender of the message via some mechanism"
   - "Transfers of 0 values MUST be treated as normal transfers and fire the Transfer event"
   - "The function SHOULD throw if the message caller's account balance does not have enough tokens to spend"
3. Induce sorts: `Function`, `Address`, `Amount`, `Event`, `AuthorizationCondition`
4. Induce predicates:
   - `IsAuthorized(Address, Address) -> Bool` -- sender is authorized by owner
   - `HasSufficientBalance(Address, Amount) -> Bool` -- address holds enough tokens
   - `EmitsEvent(Function, Event) -> Bool` -- function fires event
   - `ThrowsOnCondition(Function, Condition) -> Bool` -- function reverts when condition met

Output:
```
# Induced Grammar Extension
Sorts: Function, Address, Amount, Event, AuthorizationCondition
Predicates:
  IsAuthorized(Address, Address) -> Bool
  HasSufficientBalance(Address, Amount) -> Bool
  EmitsEvent(Function, Event) -> Bool
  ThrowsOnCondition(Function, Condition) -> Bool

# Formal Specifications
Rule 1: check(ThrowsOnCondition(transferFrom, not IsAuthorized(_from, msg.sender)), true);
Rule 2: check(EmitsEvent(transferFrom, Transfer), true) if check(IsValidTransfer(_from, _to, _value), true);
Rule 3: check(ThrowsOnCondition(transferFrom, not HasSufficientBalance(msg.sender, _value)), true);
```

**Example 2: Rust Memory Allocator Safety Rules**

User: "Extract formal specs from the Rust `GlobalAlloc` documentation."

Approach:
1. Extract entities: `alloc`, `dealloc`, `realloc`, `Layout`, pointer types
2. Extract NL rules:
   - "the `ptr` parameter must denote a block of memory currently allocated via this allocator"
   - "the `layout` parameter must match the `Layout` used to allocate the block"
   - "the caller must ensure that `layout` has non-zero size"
3. Induce sorts: `Pointer`, `Layout`, `Allocator`, `Function`
4. Induce predicates:
   - `AllocatedBy(Pointer, Allocator) -> Bool`
   - `LayoutMatches(Layout, Layout) -> Bool`
   - `NonZeroSize(Layout) -> Bool`

Output:
```
# Formal Specifications for dealloc(ptr, layout)
Precondition 1: check(AllocatedBy(ptr, self), true);
Precondition 2: check(LayoutMatches(layout, original_layout), true);
Precondition 3: check(NonZeroSize(layout), true);
```

**Example 3: Java NavigableMap Contract**

User: "Formalize the rules for `NavigableMap.subMap` from the Javadoc."

Approach:
1. Extract entities: `subMap(K fromKey, boolean fromInclusive, K toKey, boolean toInclusive)`, `NavigableMap<K,V>`
2. Extract NL rules:
   - "Throws `IllegalArgumentException` if `fromKey` is greater than `toKey`"
   - "Throws `NullPointerException` if `fromKey` or `toKey` is null and this map does not permit null keys"
   - "Returns a view of the portion of this map whose keys range from `fromKey` to `toKey`"
3. Induce sorts: `Key`, `Map`, `Function`, `Exception`
4. Induce predicates:
   - `KeyOrder(Key, Key) -> Bool` -- first key <= second key
   - `PermitsNullKeys(Map) -> Bool`
   - `IsNull(Key) -> Bool`
   - `ThrowsException(Function, Exception) -> Bool`

Output:
```
# Formal Specifications for subMap
Rule 1: check(ThrowsException(subMap, IllegalArgumentException), true)
         if not check(KeyOrder(fromKey, toKey), true);
Rule 2: check(ThrowsException(subMap, NullPointerException), true)
         if check(IsNull(fromKey), true) or check(IsNull(toKey), true)
         and not check(PermitsNullKeys(this), true);
```

## Best Practices

- **Do:** Process rules incrementally when inducing sorts and predicates. Feed previously defined sorts/predicates into each subsequent induction step so the vocabulary stays compact and consistent.
- **Do:** Validate every generated spec by parsing it against the induced grammar. A spec that doesn't parse is a spec that can't be trusted.
- **Do:** Separate rule extraction from formalization. First identify which sentences are actual constraints (using MUST/SHOULD/SHALL keywords as strong signals), then formalize only those.
- **Do:** Reuse predicates aggressively. If two NL rules describe the same property with different wording, map them to the same predicate. This is the primary advantage of grammar induction over direct translation.
- **Avoid:** Skipping the grammar induction step and going straight to spec generation. Without a fixed grammar, equivalent rules will get inconsistent representations, predicate names will drift, and the output won't be parseable by downstream tools.
- **Avoid:** Inducing overly fine-grained sorts. A sort should represent a category of entity (e.g., `Address`, `Pointer`), not an individual instance. Keep the sort count manageable (typically 3-8 per domain).
- **Avoid:** Attempting to encode temporal properties or quantification in this framework. The DSL template supports Boolean conditions and predicate checks, not full temporal logic or first-order quantification.

## Error Handling

| Problem | Symptom | Resolution |
|---------|---------|------------|
| Ambiguous NL rule | Rule sentence contains hedging language ("may", "could", "typically") | Mark as low-confidence; extract only MUST/SHOULD/SHALL rules by default, flag others for user review |
| Predicate drift | Same property named differently across rules (e.g., `HasBalance` vs `SufficientFunds`) | During grammar induction, explicitly check each new predicate against existing ones for semantic overlap before adding |
| Spec fails grammar validation | Generated spec uses a predicate or sort not in the grammar | Retry with the error message and grammar definition provided as context; if 3 retries fail, flag for manual review |
| Entity extraction misses entities | Documentation uses non-standard formatting | Fall back to LLM-based entity extraction without regex, or ask the user to provide an entity list |
| Over-extraction of rules | Descriptive text misclassified as rules | Tighten the dual-evaluation threshold; require both evaluations to agree before classifying as rule |

## Limitations

- **No temporal properties.** The DSL template supports Boolean conditions and predicate checks but cannot express ordering constraints like "function A must be called before function B" or liveness properties. For typestate or temporal logic specs, a different formalism is needed.
- **No quantification.** Rules like "for all elements in the collection" cannot be directly expressed. The framework handles per-entity, per-function rules, not universal/existential quantification over collections.
- **Grammar quality depends on documentation quality.** Vague, incomplete, or contradictory documentation will produce vague or contradictory specs. Garbage in, garbage out.
- **Predicate semantics are symbolic, not executable.** The induced predicates describe *what* should hold, but generating the executable check (the implementation of `AllocatedBy(ptr, self)`) requires separate work specific to the target language and verification framework.
- **Scalability with large documents.** Very long documents (>100 rules) may require batching the grammar induction step to avoid context window limits. Process in windows of 20-30 rules, merging induced grammar components between batches.

## Reference

**Paper:** [Doc2Spec: Synthesizing Formal Programming Specifications from Natural Language via Grammar Induction](https://arxiv.org/abs/2602.04892v1) (Xia et al., 2026). Look for: the EBNF DSL template (Figure 5), the five-agent pipeline architecture (Section 3), the incremental sort/predicate induction algorithm (Section 3.4), and the evaluation across ERC standards, Rust allocators, and Java APIs (Section 4).