---
name: "capture-flags-family-based-evaluation"
description: >
  Generate semantics-preserving variants of Python CTF challenges to stress-test
  agentic LLM robustness. Applies the Evolve-CTF methodology: identifier renaming,
  dead code insertion, composite transforms, and obfuscation to create challenge
  families that share a single exploit but vary in surface-level code.
  Trigger phrases:
  - "generate CTF variants"
  - "obfuscate this challenge"
  - "create a challenge family"
  - "test agent robustness on CTF"
  - "semantics-preserving transformation"
  - "evolve this CTF challenge"
---

# Capture-the-Flag Family-Based Evaluation via Semantics-Preserving Transformations

This skill enables Claude to generate **families** of semantically-equivalent CTF challenges from a single Python source file, following the Evolve-CTF methodology. Given a base CTF challenge (a Python program with a known flag/exploit), Claude applies a structured tree of seven transformation types—identifier renaming, four dead-code insertion strategies, composite insertion, and deep obfuscation—to produce up to 24 variant instances. Each variant preserves the original exploit path while changing the code surface, enabling controlled evaluation of how well AI agents (or human solvers) generalize across code presentations.

## When to Use

- When a user wants to **generate variant CTF challenges** from an existing Python-based challenge to test solver robustness
- When building a **CTF benchmark suite** and needs controlled difficulty scaling without changing exploit logic
- When evaluating whether an **agentic LLM pipeline** (e.g., ReAct agent with bash/python tools) is genuinely understanding code vs. pattern-matching on surface features
- When a user asks to **obfuscate Python source code** while provably preserving semantics for security training
- When creating **anti-cheat variants** for CTF competitions—same solution, different code appearance
- When stress-testing a **static analysis tool or decompiler** against syntactically diverse but semantically identical inputs

## Key Technique

The core insight from the paper is **challenge families**: instead of evaluating an agent on a single CTF instance, you generate a family of variants via semantics-preserving program transformations and measure consistency of agent performance across the family. A robust agent should solve all members of a family (since the exploit is identical); performance drops reveal brittleness to surface-level code features rather than genuine reasoning failures.

Evolve-CTF defines seven transformations organized into a 24-node tree. The transformations use **libCST** (a concrete syntax tree parser) to manipulate Python source while preserving whitespace, comments, and formatting context. Dead-code insertions use **provably false conditions** (e.g., `while False:`, `if 0 > 1:`) so injected loops, conditionals, functions, and comments never execute. The **composite transform (T5)** applies all four insertion types sequentially with equal budgets to avoid code bloat. The **obfuscation transform (OO)** applies PyObfuscator at medium level: renaming all identifiers, removing docstrings, encrypting string literals, and gzip-compressing the result.

The family tree composes these transforms: `Original -> {RR, T1..T5, OO} -> {RR+T1..RR+T5, T1+OO..T5+OO} -> {RR+T1+OO..RR+T5+OO}`, yielding 24 instances. Validation is done by re-running the **golden solution** (the known exploit) against each variant to confirm solvability. This is critical—every variant must still be exploitable via the original strategy.

## Step-by-Step Workflow

1. **Parse the base CTF challenge.** Read the Python source file and parse it into a concrete syntax tree using libCST (`import libcst as cst; tree = cst.parse_module(source)`). Identify all eligible insertion points: function bodies, loop bodies, module-level statements.

2. **Extract the golden solution and flag.** Identify the known exploit script or flag value. This will be used to validate every generated variant. If no golden solution exists, ask the user to provide one before proceeding.

3. **Apply RR (Rename Identifiers).** Walk the CST and replace all user-defined variable, function, and class names with randomly-generated alternatives. Use concatenated programming terms (e.g., `buffer_stack_ptr`) or multilingual random strings. Preserve built-in names, imports, and string literals.

4. **Apply T1-T4 (Dead Code Insertions) independently.** For each transform, sample eligible locations in the CST and insert:
   - **T1 (Loops):** Nested `for`/`while` loops with provably false conditions (`while False:`, `for _ in range(0):`) containing plausible but unreachable code that references real variable names.
   - **T2 (Conditionals):** `if` statements with false guards (`if 0 > 1:`) or `try/except` blocks wrapping unreachable code.
   - **T3 (Functions):** `def` or `lambda` definitions with random signatures that are never called.
   - **T4 (Comments):** Natural-language English comments or multilingual gibberish strings inserted at random statement boundaries.

5. **Apply T5 (Composite).** Apply T1 through T4 sequentially to the original source, allocating an equal insertion budget to each (e.g., 3 insertions per type for a budget of 12).

6. **Apply OO (Obfuscation).** Run PyObfuscator at medium level on the source: rename all identifiers to short opaque names, strip docstrings, encrypt string literals, and optionally gzip-compress. Note: OO is a terminal transform—no further insertions are meaningful after it.

7. **Compose transforms into the family tree.** Generate all valid compositions: `RR+T1`, `RR+T2`, ..., `RR+T5`, `T1+OO`, ..., `T5+OO`, `RR+T1+OO`, ..., `RR+T5+OO`. Apply transforms left-to-right (e.g., rename first, then insert dead code, then obfuscate).

8. **Validate every variant.** Execute the golden solution against each of the 24 generated variants. Confirm the flag is captured successfully. Discard and regenerate any variant that breaks solvability (this indicates a transformation bug, not a design flaw).

9. **Package the challenge family.** Output a directory structure: `family/<challenge_name>/original.py`, `family/<challenge_name>/RR.py`, `family/<challenge_name>/T1.py`, ..., `family/<challenge_name>/RR_T3_OO.py`. Include a `manifest.json` mapping variant IDs to transformation chains and validation status.

10. **Run agent evaluation (optional).** If the user wants to benchmark an agentic LLM, execute each variant with a ReAct-style agent loop (bash + python + submit tools), 5 repeats per variant, with a fixed token budget (e.g., 200K tokens). Record binary success/failure per run and compute mean solvability per variant and per transformation type.

## Concrete Examples

**Example 1: Generating a family from a simple crypto CTF**

User: "I have a Python CTF challenge where the flag is encrypted with a Caesar cipher. Generate a challenge family to test my agent."

Approach:
1. Read the base challenge (`caesar.py`) containing the encryption logic and ciphertext.
2. Parse with libCST. Identify 4 functions, 12 variables eligible for renaming.
3. Generate RR variant: rename `decrypt`, `shift`, `ciphertext` to `invoke_handler`, `offset_magnitude`, `encoded_payload`.
4. Generate T1 variant: insert `while False: for i in range(10): shift += 1` after line 8.
5. Generate T2 variant: insert `if len("") > 5: print(ciphertext)` before the main decryption call.
6. Generate T5 variant: apply all insertions with budget=2 each (8 total insertions).
7. Generate OO variant: all names become `_0x1a`, `_0x1b`, etc.; string `"flag{"` becomes encrypted.
8. Compose remaining 17 variants. Validate all 24 with the golden solution `python3 solve.py`.
9. Output `family/caesar/` with 24 `.py` files and `manifest.json`.

Output structure:
```
family/caesar/
  original.py          # Base challenge
  RR.py                # Renamed identifiers
  T1.py                # Dead loops inserted
  T2.py                # Dead conditionals inserted
  T3.py                # Dead functions inserted
  T4.py                # Misleading comments inserted
  T5.py                # Composite (T1-T4 applied)
  OO.py                # Obfuscated
  RR_T1.py             # Renamed + dead loops
  ...
  RR_T5_OO.py          # Renamed + composite + obfuscated
  manifest.json        # Transformation metadata + validation results
```

**Example 2: Evaluating agent robustness on an RSA challenge**

User: "I want to see if my LLM agent is actually reasoning about RSA or just pattern-matching. Test it with variants."

Approach:
1. Take the RSA challenge (`rsa_challenge.py`) with known `n`, `e`, `c` and a factorizable `n`.
2. Generate the full 24-member family using Evolve-CTF transforms.
3. Run the user's agent against each variant 5 times with a 200K token budget.
4. Collect results into a matrix: rows = variants, columns = runs.
5. Compute per-transform-type mean success rate.

Output analysis:
```
Transform       | Success Rate | Delta from Original
----------------|-------------|--------------------
Original        | 80%         | --
RR (rename)     | 76%         | -4%
T1 (loops)      | 80%         | 0%
T2 (conds)      | 72%         | -8%
T5 (composite)  | 64%         | -16%
OO (obfuscate)  | 36%         | -44%
RR+T5+OO        | 28%         | -52%
```
Interpretation: The agent is robust to simple insertions but struggles significantly with obfuscation, suggesting it relies on identifier names and string literals rather than structural analysis of the RSA math.

**Example 3: Creating anti-cheat CTF variants for a competition**

User: "I'm running a CTF competition and want each team to get a slightly different version of the same challenge so they can't share exact solutions."

Approach:
1. Take the base challenge and generate 24 family members.
2. Assign each team a unique variant (e.g., Team 1 gets `RR_T2.py`, Team 2 gets `T1_OO.py`).
3. All teams must find the same flag—the exploit logic is identical—but copy-pasting code analysis between teams won't directly transfer.
4. Validate that all variants produce the same flag when solved correctly.

## Best Practices

- **Do:** Always validate every variant with the golden solution before using it. A single broken variant invalidates the family.
- **Do:** Use provably false guards for dead code (`while False:`, `if 0 > 1:`, `for _ in range(0):`). Never use conditions that could evaluate to true in edge cases.
- **Do:** Reference real variable names inside dead code blocks to make them appear connected to the logic. This is what makes T1-T3 effective at confusing solvers.
- **Do:** Set a fixed insertion budget (e.g., 2-4 insertions per transform type) to prevent code bloat that makes variants trivially distinguishable from the original.
- **Avoid:** Applying OO (obfuscation) before other transforms. OO is a terminal transform—gzip compression and identifier encryption prevent further meaningful insertion.
- **Avoid:** Transforming code outside the challenge source (e.g., Dockerfiles, setup scripts, flag-checking infrastructure). Only the vulnerable/exploitable Python source should be varied.

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| Golden solution fails on a variant | Transformation broke semantics (e.g., renamed a built-in or import) | Check that RR only renames user-defined identifiers; re-parse and filter the rename map against Python builtins and imported names |
| Inserted dead code causes `SyntaxError` | Insertion point was inside an expression or string | Restrict insertion to statement-level CST nodes only (simple statements and compound statement bodies) |
| OO variant crashes at runtime | PyObfuscator renamed an external library call | Exclude identifiers that resolve to imported module attributes from the obfuscation scope |
| Variant is trivially distinguishable | Too many insertions in a small file | Reduce insertion budget proportionally to file length (e.g., 1 insertion per 20 lines of original code) |
| libCST fails to parse source | Source uses Python 3.12+ syntax not yet supported | Fall back to `ast` module for parsing, though you lose concrete syntax preservation |

## Limitations

- **Python only.** The Evolve-CTF methodology as described operates on Python source via libCST. Extending to C, JavaScript, or binary challenges requires different parsing infrastructure.
- **Static transforms only.** The current approach transforms source code text. It does not modify runtime behavior, network protocols, or binary formats that some CTF challenges involve.
- **Obfuscation ceiling.** The OO transform uses a specific obfuscator (PyObfuscator) at medium level. Truly adversarial obfuscation (virtualization, jitting, polyglot code) is out of scope.
- **Exploit must be code-level.** Challenges where the exploit targets a network service, timing side-channel, or hardware feature rather than source code logic won't benefit from source-level transformations.
- **24-variant cap.** The family tree structure produces exactly 24 variants. For larger evaluation suites, you'd need to increase insertion randomness seeds or add new transform types.
- **No difficulty grading.** Variants are not ordered by difficulty—OO is harder for agents than T4, but the framework doesn't assign difficulty scores. Interpret performance drops as robustness signals, not difficulty rankings.

## Reference

**Paper:** Honarvar, S., Gorzynski, A., Lee-Jones, J., Coppock, H., & Rei, M. (2026). *Capture the Flags: Family-Based Evaluation of Agentic LLMs via Semantics-Preserving Transformations.* arXiv:2602.05523v1. [https://arxiv.org/abs/2602.05523v1](https://arxiv.org/abs/2602.05523v1)

**Key takeaway:** Models are remarkably robust to renaming and dead-code insertion but degrade significantly under composed transforms and obfuscation—revealing that surface features (identifier names, string literals) disproportionately drive agent performance on CTF tasks.