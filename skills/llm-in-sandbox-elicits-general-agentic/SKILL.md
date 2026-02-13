---
name: "llm-in-sandbox-elicits-general-agentic"
description: "Solve non-code tasks (math, science, long-context, formatting) by treating the terminal as a sandbox for exploration: writing scripts, installing tools, managing files, and fetching external resources. Triggers: 'solve this math problem using code', 'analyze this long document', 'compute this chemistry/physics problem', 'format this output precisely', 'verify this calculation programmatically', 'process this scientific data'"
---

# LLM-in-Sandbox: Elicit General Agentic Intelligence via Code Sandbox Exploration

This skill teaches Claude to systematically solve non-code problems -- mathematics, physics, chemistry, biomedicine, long-context understanding, and strict formatting tasks -- by leveraging the terminal as an exploratory sandbox. Instead of reasoning purely through natural language (which is error-prone for computation and domain-specific knowledge), Claude writes and executes scripts, installs domain-specific tools at runtime, stores large contexts in files for programmatic search, and iteratively refines solutions through execution feedback. The core insight from the LLM-in-Sandbox paper is that LLMs already possess latent agentic capabilities to use a sandbox effectively; this skill makes those strategies explicit and repeatable.

## When to Use

- When the user asks to solve a math, physics, or chemistry problem that involves numerical computation, symbolic manipulation, or domain-specific formulas
- When processing a document or dataset too large to reason about in a single pass (100K+ tokens) -- store it in files and search programmatically
- When the user needs output in a strict format (exact character counts, specific structural constraints, JSON schemas) that requires verification logic
- When a task requires domain-specific tools not available by default (e.g., chemistry SMILES parsers, bioinformatics tools, specialized solvers)
- When the user asks to "verify" or "double-check" a calculation or derivation
- When combining multiple knowledge sources or cross-referencing data to answer a complex question

## Key Technique

**Sandbox-as-cognitive-tool**: Rather than performing all reasoning in natural language, offload computation, search, and verification to executable code. The LLM-in-Sandbox paper demonstrates that strong models spontaneously adopt three strategies when given sandbox access: (1) **external resource acquisition** -- installing packages or fetching data to gain domain knowledge they lack; (2) **file system management** -- writing large contexts to files and using grep/sed/Python to navigate them, reducing token consumption by up to 8x; and (3) **script-based computation** -- writing Python scripts with NumPy/SciPy to perform numerical verification, combinatorial search, or simulation instead of mental math.

**Iterative exploration loop**: The approach follows a tight loop: generate a tool action (bash command or file edit), execute it, observe the result, and decide the next action. Effective trajectories average ~12 turns with purposeful actions. The key is to derive answers through program execution rather than hardcoding results -- write code that *computes* the answer, don't write code that just prints a pre-determined answer.

**Efficiency advantage**: For long-context tasks, placing documents in files and searching programmatically uses 0.49-0.84x the tokens of prompt-based reasoning. The sandbox environment itself is lightweight (50MB idle per container). This means the sandbox approach is often *cheaper* and *more accurate* than brute-force context stuffing.

## Step-by-Step Workflow

1. **Classify the task type**: Determine whether the problem primarily needs computation (math/physics), domain tools (chemistry/bio), file-based search (long context), formatting verification, or a combination. This determines which sandbox strategies to prioritize.

2. **Externalize large inputs to files**: If the user provides a long document, dataset, or multi-part input, write it to one or more files in `/tmp/` immediately. Do not attempt to hold 100K+ tokens in conversational context -- use `grep`, `sed`, or Python scripts to search and extract relevant sections.

3. **Install domain-specific tools if needed**: For chemistry (e.g., OPSIN, RDKit), bioinformatics (e.g., BioPython), symbolic math (e.g., SymPy), or other specialized domains, install the required packages at runtime. Use `pip install` for Python packages or `apt-get install` for system tools. Verify installation before proceeding.

4. **Write an executable script that computes the answer**: Write a Python script (or shell commands) that performs the core computation. Use numerical solvers (`scipy.optimize`, `numpy.linalg`), symbolic algebra (`sympy`), or domain libraries as appropriate. The script must *derive* the answer through computation, not merely print a hardcoded value.

5. **Execute and observe**: Run the script, capture stdout/stderr. If it errors, diagnose the issue from the traceback, fix the script, and re-execute. Expect 2-4 iterations for non-trivial problems.

6. **Verify the result independently**: Write a separate verification check -- a different method, a sanity bound, or a format validator. For math, verify by substitution. For formatting, write a validator script that checks all constraints. Do not skip verification.

7. **For formatting-constrained tasks, build a constraint checker**: If the output must satisfy structural rules (word count, character count, no overlapping words, specific JSON structure), write a Python function that programmatically checks every constraint against the candidate output. Iterate until all checks pass.

8. **Extract and present the final answer**: Once verified, extract the clean result from script output. Present it to the user with a brief explanation of the method used. Include the key computation or reasoning steps, not just the raw answer.

9. **Clean up temporary files**: Remove any large temporary files written to `/tmp/` to avoid cluttering the workspace.

## Concrete Examples

**Example 1: Solving a physics problem with numerical computation**

User: "What is the escape velocity from a neutron star with mass 2.0 solar masses and radius 12 km?"

Approach:
1. Classify: computation task requiring physical constants and formula application.
2. Write a Python script using known constants:

```python
import math

G = 6.674e-11       # gravitational constant (m^3 kg^-1 s^-2)
M_sun = 1.989e30    # solar mass (kg)
M = 2.0 * M_sun     # neutron star mass
R = 12e3             # radius in meters

v_escape = math.sqrt(2 * G * M / R)
print(f"Escape velocity: {v_escape:.4e} m/s")
print(f"As fraction of c: {v_escape / 3e8:.4f}")
```

3. Execute the script.
4. Verify: check that the result is a significant fraction of c (expected for neutron stars).

Output:
```
Escape velocity: 1.9277e+08 m/s
As fraction of c: 0.6426
```
Approximately 64% the speed of light, consistent with known neutron star escape velocities.

---

**Example 2: Long-context document analysis**

User: Provides a 150K-token legal contract and asks "List all clauses that reference indemnification and summarize the liability caps."

Approach:
1. Classify: long-context file-based search task.
2. Write the document to `/tmp/contract.txt`.
3. Use grep to find all sections mentioning indemnification:

```bash
grep -n -i "indemnif" /tmp/contract.txt
```

4. From the line numbers, extract surrounding context (e.g., 20 lines around each match) using a Python script.
5. Write a second script that parses the extracted sections for monetary amounts (regex for dollar signs, "cap", "limit", "not to exceed").
6. Compile and present the findings.

Output:
```
Found 7 clauses referencing indemnification:
- Section 4.2 (line 1,204): Mutual indemnification for IP infringement. Cap: $5M.
- Section 4.3 (line 1,287): Vendor indemnifies for data breach. Cap: 2x annual fees.
- Section 8.1 (line 3,102): General liability cap of $10M aggregate.
...
```

---

**Example 3: Strict formatting with constraint verification**

User: "Write exactly 3 sentences about climate change. Each sentence must have exactly 100 characters. No two sentences may share any word."

Approach:
1. Classify: formatting-constrained generation requiring verification.
2. Draft three candidate sentences.
3. Write a Python validator:

```python
sentences = [
    "Rising ocean temperatures...",  # candidates
    "Glacial retreat accelerates...",
    "Atmospheric carbon dioxide...",
]

for i, s in enumerate(sentences):
    print(f"Sentence {i+1}: {len(s)} chars")

words = [set(s.lower().replace('.','').replace(',','').split()) for s in sentences]
for i in range(len(words)):
    for j in range(i+1, len(words)):
        overlap = words[i] & words[j]
        if overlap:
            print(f"OVERLAP between {i+1} and {j+1}: {overlap}")
        else:
            print(f"No overlap between {i+1} and {j+1}: OK")
```

4. Execute, check constraints, revise sentences, re-validate. Repeat until all constraints pass.
5. Present the validated output.

---

**Example 4: Chemistry problem requiring domain tools**

User: "Convert the IUPAC name '2,4,6-trinitrotoluene' to its SMILES representation and compute its molecular weight."

Approach:
1. Classify: domain-tool task requiring chemistry packages.
2. Install RDKit: `pip install rdkit-pypi`
3. Write and execute:

```python
from rdkit import Chem
from rdkit.Chem import Descriptors

# RDKit can parse common names or we build from known SMILES
smiles = "Cc1c(cc(cc1[N+](=O)[O-])[N+](=O)[O-])[N+](=O)[O-]"
mol = Chem.MolFromSmiles(smiles)
print(f"SMILES: {smiles}")
print(f"Molecular weight: {Descriptors.MolWt(mol):.2f} g/mol")
print(f"Formula: {Chem.rdMolDescriptors.CalcMolFormula(mol)}")
```

4. Verify molecular weight against known value (227.13 g/mol for TNT).

## Best Practices

- **Do: Compute, don't hardcode.** Write scripts that derive answers through calculation. A script that just `print("42")` defeats the purpose -- the sandbox should do the cognitive work.
- **Do: Install tools at runtime when needed.** Don't assume packages are pre-installed. Use `pip install` or `apt-get` and verify before using domain libraries.
- **Do: Write verification scripts.** Always validate results with an independent check -- substitution, bounds checking, or a different algorithmic approach.
- **Do: Externalize large contexts to files.** For documents over ~10K tokens, write to files and search programmatically. This is both faster and more reliable than scanning in-context.
- **Avoid: Performing complex arithmetic in natural language.** Multiplying large numbers, evaluating integrals, or counting characters by hand leads to errors. Use the sandbox.
- **Avoid: Monolithic scripts.** Break complex problems into smaller scripts that each solve one sub-problem. This makes debugging easier when individual steps fail.

## Error Handling

| Problem | Solution |
|---------|----------|
| Package installation fails | Try alternative package names, use `--break-system-packages` flag if in isolated env, or fall back to pure Python implementation |
| Script produces wrong result | Check units, boundary conditions, and data types. Print intermediate values. Compare against known reference values |
| Long-context grep returns too many matches | Refine search patterns, use `-C` for context, or write a Python script with more precise filtering logic |
| Timeout on expensive computation | Add progress indicators, reduce problem size for initial validation, then scale up. Set reasonable iteration limits |
| Formatting constraint impossible to satisfy | Verify constraints are jointly satisfiable before iterating. Report to the user if constraints conflict |

## Limitations

- **Network access**: If the sandbox has no internet access, external resource acquisition (installing packages, fetching data) will fail. Fall back to built-in Python standard library.
- **Execution time**: Very large simulations or brute-force searches may exceed reasonable time limits. Prefer analytical or algorithmic solutions over exhaustive enumeration.
- **Domain depth**: Installing a package gives access to its functions but does not give Claude domain expertise. For highly specialized domains (e.g., quantum chemistry), the scripts may be syntactically correct but methodologically flawed. Flag uncertainty to the user.
- **Non-determinism**: Some numerical methods (Monte Carlo, stochastic optimization) produce variable results across runs. Use fixed seeds and report confidence intervals.
- **Not a replacement for reasoning**: The sandbox augments reasoning, it does not replace it. Claude must still understand the problem to write the right script. The sandbox catches computational errors, not conceptual ones.

## Reference

- **Paper**: [LLM-in-Sandbox Elicits General Agentic Intelligence](https://arxiv.org/abs/2601.16206v1) (Cheng et al., 2026)
- **Key takeaway**: Section 3 details the three emergent sandbox strategies (external resources, file management, computation) with quantified usage frequencies. Table 2 shows benchmark gains. Algorithm 1 defines the explore-execute-observe loop. The system prompt principles in Appendix F are directly applicable to prompt design.
- **Project page**: https://llm-in-sandbox.github.io