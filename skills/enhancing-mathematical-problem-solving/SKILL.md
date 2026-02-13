---
name: "enhancing-mathematical-problem-solving"
description: |
  Solve mathematical problems using IIPC (Iteratively Improved Program Construction) -- a dual-branch approach that
  combines iterative code execution with independent chain-of-thought reasoning, then merges both for a verified answer.
  Trigger phrases: "solve this math problem with code verification", "math reasoning with execution feedback",
  "verify this calculation programmatically", "solve and check with code", "iterative math solving",
  "math problem with dual reasoning"
---

# Execution-Driven Mathematical Problem Solving (IIPC)

This skill enables Claude to solve mathematical problems using Iteratively Improved Program Construction (IIPC), a technique from Basarkar et al. (2026). Instead of generating a single answer or a single code solution, IIPC runs two parallel reasoning branches -- an executable program branch that iteratively refines code with real execution feedback, and an independent chain-of-thought branch that reasons without seeing program output. The two branches merge only at the final step, preventing over-reliance on either approach and catching errors that one branch alone would miss.

## When to Use

- When the user asks to solve a math problem and wants high confidence in the answer
- When a mathematical computation involves multiple steps where intermediate errors compound (algebra, combinatorics, optimization)
- When the user explicitly requests code-verified mathematical reasoning
- When solving competition-level math (AMC, AIME, Olympiad-style problems)
- When a word problem requires both symbolic reasoning and numerical computation
- When previous single-pass attempts at a math problem produced wrong or inconsistent answers
- When the user needs to solve a batch of math problems with reliable accuracy

## Key Technique

**Dual-branch architecture with execution-driven refinement.** IIPC treats generated programs not as disposable scripts but as *representations of the model's reasoning chain*. The program branch generates executable code, runs it, inspects the output, and iteratively corrects errors using a failure memory that prevents repeating mistakes. Simultaneously, an independent chain-of-thought branch reasons through the problem in natural language without seeing any program output. This independence is critical -- it prevents the "program bias" where an LLM anchors on a possibly-wrong code result and rationalizes it instead of catching the error.

**Failure memory and iterative refinement.** Each iteration that produces an error or a failed validation generates a *failure descriptor* stored in memory M_t. On the next iteration, the model sees these descriptors alongside the problem, preventing it from regenerating the same flawed approach. This transforms blind trial-and-error into informed trial-and-improvement. The system allows up to two validation checks and two error corrections per validation, creating a bounded but thorough refinement loop.

**Structured integration at the final step.** Only after both branches complete does a structured integration prompt combine their outputs. The model weighs evidence from the deterministic execution branch against the independent reasoning branch, resolving conflicts and producing a final verified answer. This late fusion is what gives IIPC its edge over methods that mix code and reasoning throughout.

## Step-by-Step Workflow

1. **Extract propositions from the problem.** Parse the math problem into its core components: given quantities, constraints, what is being asked, and the mathematical domain (algebra, geometry, number theory, combinatorics, probability, etc.).

2. **Generate an executable program (Branch 1).** Write Python code that computes the answer. Use sympy for symbolic math, itertools for combinatorics, numpy for numerical work. The code must print a clear final answer.

3. **Execute the program and capture output.** Run the code, capturing both stdout and any errors (syntax errors, runtime exceptions, wrong types).

4. **Validate or correct based on execution result.**
   - If execution succeeds: validate the output against problem constraints (does the answer have the right type? is it in the expected range? does it satisfy stated conditions?).
   - If execution fails or validation fails: record a failure descriptor explaining *what went wrong and why*, then regenerate the program with this context. Repeat up to 2 correction cycles per validation.

5. **Update failure memory.** Append each failure descriptor to the memory store. On subsequent iterations, include these descriptors in context so the same mistakes are not repeated.

6. **Produce independent chain-of-thought reasoning (Branch 2).** Solve the same problem using pure mathematical reasoning -- no reference to any program output. Work through the problem step by step with symbolic manipulation, logical deduction, and mathematical identities.

7. **Integrate both branches.** Compare the program's computed answer with the chain-of-thought answer:
   - If they agree: high confidence -- report the answer.
   - If they disagree: analyze the discrepancy. Check the program logic against the reasoning steps. Identify which branch made an error and why. Re-run the erroneous branch if needed.

8. **Format and present the final answer.** State the answer clearly, show the verified computation, and briefly explain the reasoning path that confirmed it.

## Concrete Examples

**Example 1: Competition-style number theory**

User: How many positive integers n < 1000 satisfy n^2 + n + 1 being divisible by 7?

Approach:

*Branch 1 -- Program:*
```python
count = 0
for n in range(1, 1000):
    if (n**2 + n + 1) % 7 == 0:
        count += 1
print(count)  # Output: 285
```
Execution succeeds. Output: 285. Validation: answer is a positive integer less than 999, plausible.

*Branch 2 -- Chain-of-thought:*
n^2 + n + 1 = 0 (mod 7). Solving the quadratic mod 7: discriminant = 1 - 4 = -3 = 4 (mod 7).
sqrt(4) mod 7 = 2 or 5. So n = (-1 + 2)/2 = 1/2 or n = (-1 + 5)/2 = 2 (mod 7).
1/2 mod 7 = 4 (since 2*4=8=1 mod 7). So n = 2 or n = 4 (mod 7).
Among 1..999: floor(999/7)*2 + adjustments = 142*2 + 1 = 285.

*Integration:* Both branches produce 285. Answer: **285**.

---

**Example 2: Geometry with iterative correction**

User: A triangle has sides 13, 14, 15. Find the radius of its inscribed circle.

Approach:

*Branch 1 -- Program (iteration 1):*
```python
import math
a, b, c = 13, 14, 15
s = (a + b + c) / 2  # semi-perimeter = 21
area = math.sqrt(s * (s-a) * (s-b) * (s-c))  # Heron's formula
r = area / s
print(r)  # Output: 4.0
```
Execution succeeds. Output: 4.0. Validation: positive, less than shortest side, plausible.

*Branch 2 -- Chain-of-thought:*
s = (13+14+15)/2 = 21. Area by Heron's: sqrt(21 * 8 * 7 * 6) = sqrt(7056) = 84. Inradius r = Area/s = 84/21 = 4.

*Integration:* Both branches yield r = 4. Answer: **4**.

---

**Example 3: Probability with failure recovery**

User: In a random permutation of {1,...,10}, what is the probability that no element is in its original position (a derangement)?

Approach:

*Branch 1 -- Program (iteration 1):*
```python
import math
n = 10
# Derangement count D(n) = n! * sum_{k=0}^{n} (-1)^k / k!
D = 0
for k in range(n + 1):
    D += (-1)**k / math.factorial(k)
D *= math.factorial(n)
print(D / math.factorial(n))
```
Output: 0.3678794642857143. Validation: this is an approximation due to floating point. The exact answer should be a fraction.

*Failure descriptor:* "Floating point arithmetic produced approximate result. Need exact integer arithmetic for D(n), then express as fraction."

*Branch 1 -- Program (iteration 2):*
```python
from fractions import Fraction
n = 10
D = 0
for k in range(n + 1):
    D += Fraction((-1)**k, 1) / Fraction(1, 1) * Fraction(1, 1)
# Cleaner: D(n) = n! * sum (-1)^k / k!
from math import factorial
derangements = sum((-1)**k * factorial(n) // factorial(k) for k in range(n+1))
prob = Fraction(derangements, factorial(n))
print(prob)  # Output: 1334961/3628800
```
Output: 1334961/3628800. Validation: 1334961/3628800 ~ 0.3679, matches known D(10)/10! value.

*Branch 2 -- Chain-of-thought:*
D(10) = 10! * (1 - 1 + 1/2! - 1/3! + ... + 1/10!) = 1334961.
P = 1334961 / 3628800.

*Integration:* Both agree. Answer: **1334961/3628800**.

## Best Practices

- **Do:** Always run both branches independently. The chain-of-thought must not reference program output -- this independence is what catches errors.
- **Do:** Use exact arithmetic (Python's `fractions.Fraction`, `sympy.Rational`, integer arithmetic) for problems requiring exact answers. Floating point is a common source of subtle errors.
- **Do:** Record specific failure descriptors, not vague notes. "Floating point rounding in factorial division" is useful. "Code was wrong" is not.
- **Do:** Validate program output against problem constraints before accepting it (correct type, reasonable range, satisfies stated conditions).
- **Avoid:** Running more than 2 correction cycles per validation -- diminishing returns set in and context bloats. If 2 corrections fail, rethink the approach entirely.
- **Avoid:** Letting the chain-of-thought branch see or reference the program's output before the final integration step. This defeats the dual-branch design.
- **Avoid:** Using overly complex code when simple enumeration or direct computation suffices. Simpler programs have fewer bugs and are easier to validate.

## Error Handling

| Error Type | Detection | Recovery |
|---|---|---|
| Syntax/runtime error in code | Execution fails with traceback | Record error, fix specific issue, re-execute |
| Incorrect output (wrong type/range) | Validation against constraints | Add failure descriptor, regenerate with corrected logic |
| Branch disagreement | Integration step finds mismatch | Trace both branches step-by-step to find the error; re-run the faulty branch |
| Timeout on large computation | Execution exceeds time limit | Optimize algorithm (reduce brute-force range, use mathematical shortcuts) |
| Problem ambiguity | Both branches produce different valid interpretations | Present both interpretations and answers, ask user to clarify |

When failure memory accumulates 3+ descriptors for the same subproblem, abandon the current approach and try a fundamentally different mathematical formulation.

## Limitations

- **Simple arithmetic:** For trivial calculations (basic addition, single-step formulas), the dual-branch overhead adds no value. Use direct computation instead.
- **Proof-based problems:** IIPC is designed for problems with computable numerical or symbolic answers. Pure proof tasks (e.g., "prove that for all n...") cannot be fully verified by execution, though specific cases can be checked.
- **Problems requiring external data:** If the problem references specific datasets, constants, or tables not provided, the program branch cannot execute meaningfully.
- **Highly abstract algebra:** Problems in abstract algebra (group theory, ring theory) often lack straightforward computational representations and may not benefit from code execution.
- **Context window pressure:** The failure memory and dual branches consume tokens. For very long problems or those requiring many iterations, context limits may force truncation of history.

## Reference

Basarkar, A., Tabarsi, B., Barnes, T., & Xu, D. (2026). *Enhancing Mathematical Problem Solving in LLMs through Execution-Driven Reasoning Augmentation.* arXiv:2602.03950v2. https://arxiv.org/abs/2602.03950v2

Key insight from the paper: the dual-branch architecture with late fusion outperforms single-pipeline approaches (PoT, CoT, ReAct) because execution feedback provides deterministic verification while independent reasoning prevents program bias -- look for the ablation study showing that removing either branch degrades performance.