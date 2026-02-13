---
name: "vistira-closing-image-text-modality"
description: "Solve math problems from images by decomposing them into interleaved natural-language rationales and executable Python code, closing the image-text modality gap. Use when: 'solve this math image', 'extract and compute from this screenshot', 'OCR this equation and solve it', 'reason about this visual math problem', 'convert this handwritten math to code', 'help me with this homework photo'."
---

# VisTIRA: Tool-Integrated Reasoning for Visual Math Problems

This skill enables Claude to solve mathematical problems presented as images (screenshots, photos of textbooks, handwritten homework) by applying the VisTIRA framework: a structured loop of OCR extraction, natural-language rationale generation, and executable Python code execution. Instead of attempting to reason about math notation directly from pixels — where vision-language models suffer compounded failures in reading dense formulas, layout, and mixed symbolic-diagrammatic context — this approach decomposes the problem into a chain of (Rationale, Action, Observation) steps that ground visual content in text and delegate computation to Python.

## When to Use

- When the user shares a screenshot or photo of a math problem and asks for a solution
- When the user pastes an image containing equations, tables, or diagrams that require mathematical reasoning
- When solving problems from photographed homework, exams, or textbook pages
- When the user asks to extract mathematical content from a visual and compute an answer
- When building pipelines that convert visual math datasets into solvable text+code form
- When the user needs to debug why a VLM gave the wrong answer on a math image (likely a modality gap issue)
- When processing LaTeX-rendered PDFs or formula-heavy documents that need computational verification

## Key Technique

**The Modality Gap Problem.** The same math question presented as text yields significantly higher accuracy than its visually typeset counterpart. This happens because vision models compound errors across three stages: reading dense formulas (OCR-level mistakes), interpreting spatial layout (fractions, superscripts, matrix alignment), and integrating symbolic content with diagrams. VisTIRA addresses this by converting the problem from a single-shot visual reasoning task into a structured multi-step text+code pipeline.

**Rationale-Action-Observation Loop.** VisTIRA processes each math image through an iterative cycle: (1) generate a natural-language **Rationale** explaining the current reasoning step, (2) write an executable Python **Action** (using SymPy, NumPy, or standard math libraries) that performs the computation described in the rationale, (3) capture the **Observation** (execution output), and feed it back into the next iteration. The loop continues until a final answer is produced. This interleaving forces the model to articulate its reasoning before computing, catching interpretation errors early.

**OCR Grounding as a Complement.** For complex or dense images, prepending an OCR transcription of the image content provides a textual anchor that significantly improves accuracy — especially for smaller models. The OCR output serves as auxiliary context alongside the original image, giving the model a second modality to cross-reference. This benefit diminishes with model scale (large models already extract text well), so apply OCR grounding selectively based on problem complexity and model capability.

## Step-by-Step Workflow

1. **Extract visual content via OCR.** Read the image and produce a structured text transcription. Identify equations (render as LaTeX strings), prose, labeled diagrams, and any tabular data. Flag ambiguous symbols (e.g., `l` vs `1`, `O` vs `0`) for downstream verification.

2. **Classify the problem type.** From the OCR output and image, determine the mathematical domain: algebra, calculus, combinatorics, geometry, number theory, statistics, or mixed. This informs which Python libraries and solution strategies to invoke.

3. **State the problem in natural language.** Write a clean, unambiguous text version of the problem, resolving any OCR ambiguities by cross-referencing the image. This becomes the grounding text for all subsequent reasoning.

4. **Generate Rationale step 1.** Describe in plain English the first logical step toward solving the problem. Identify knowns, unknowns, applicable theorems, and the high-level strategy (e.g., "This is a 0/0 indeterminate form, so apply L'Hopital's rule").

5. **Write executable Python Action.** Translate the rationale into a Python code block that performs the described computation. Use `sympy` for symbolic math, `numpy` for numerical work, `math` for constants. Always `print()` the result so the observation is captured.

6. **Capture Observation.** Execute the code and record the output. Verify the output is consistent with the rationale — if the code errors or produces unexpected results, revise the rationale and code before proceeding.

7. **Iterate the Rationale-Action-Observation loop.** Each new rationale should reference the previous observation and describe the next logical step. Continue until the problem is fully solved. Most problems require 2-5 iterations.

8. **State the final answer.** Present the answer clearly, formatted appropriately (exact symbolic form for algebra/calculus, numerical form for applied problems). Box or highlight the final result.

9. **Verify by back-substitution or sanity check.** Write one more Python Action that checks the answer — substitute back into the original equation, verify boundary conditions, or check units/magnitude. If verification fails, re-examine the reasoning chain.

10. **Present the complete solution.** Show the user the full Rationale-Action-Observation chain so they can follow the reasoning, with the verified final answer at the end.

## Concrete Examples

**Example 1: Calculus limit from a textbook photo**

User: [Uploads image of the problem: "Find lim(x -> -2) of (x+1)^100 - 1) / (x+2)"]

Approach:
1. OCR extracts: `lim_{x \to -2} \frac{(x+1)^{100} - 1}{x + 2}`
2. State problem: "Evaluate the limit as x approaches -2 of ((x+1)^100 - 1) / (x+2)"

Rationale 1: "Substituting x = -2 directly gives ((-1)^100 - 1) / 0 = (1-1)/0 = 0/0, an indeterminate form. Apply L'Hopital's rule: differentiate numerator and denominator separately."

Action 1:
```python
from sympy import symbols, limit, oo
x = symbols('x')
expr = ((x + 1)**100 - 1) / (x + 2)
result = limit(expr, x, -2)
print(f"Limit = {result}")
```

Observation 1: `Limit = -100`

Rationale 2: "Verify: The derivative of the numerator is 100(x+1)^99, the derivative of the denominator is 1. At x=-2: 100(-1)^99 / 1 = 100(-1) = -100. Confirmed."

Action 2:
```python
from sympy import diff
numerator = (x + 1)**100 - 1
denominator = x + 2
lhopital = diff(numerator, x) / diff(denominator, x)
print(f"L'Hopital evaluation at x=-2: {lhopital.subs(x, -2)}")
```

Observation 2: `L'Hopital evaluation at x=-2: -100`

Output: **The limit is -100.**

---

**Example 2: Geometry problem from a homework photo**

User: [Uploads photo of a triangle diagram with sides labeled a=5, b=7, angle C=60 degrees, asking "Find side c"]

Approach:
1. OCR extracts triangle with labeled sides and angle
2. State problem: "Given triangle with a=5, b=7, included angle C=60 degrees, find side c using the law of cosines"

Rationale 1: "This is a direct application of the law of cosines: c^2 = a^2 + b^2 - 2ab*cos(C). Substitute the known values."

Action 1:
```python
import math
a, b, C_deg = 5, 7, 60
C_rad = math.radians(C_deg)
c_squared = a**2 + b**2 - 2*a*b*math.cos(C_rad)
c = math.sqrt(c_squared)
print(f"c^2 = {c_squared}")
print(f"c = {c:.6f}")
print(f"c = sqrt({int(c_squared)}) = {c:.6f}")
```

Observation 1: `c^2 = 39.0`, `c = 6.244998`, `c = sqrt(39) = 6.244998`

Rationale 2: "Verify: 25 + 49 - 2(5)(7)(0.5) = 74 - 35 = 39. c = sqrt(39). Confirmed."

Output: **c = sqrt(39) approximately equal to 6.245.**

---

**Example 3: Multi-step algebra from a screenshot**

User: [Uploads screenshot: "Solve the system: 2x + 3y = 7, 4x - y = 1"]

Approach:
1. OCR extracts: system of equations 2x + 3y = 7 and 4x - y = 1
2. State problem: "Solve the linear system for x and y"

Rationale 1: "Use substitution or elimination. From equation 2: y = 4x - 1. Substitute into equation 1."

Action 1:
```python
from sympy import symbols, Eq, solve
x, y = symbols('x y')
eq1 = Eq(2*x + 3*y, 7)
eq2 = Eq(4*x - y, 1)
solution = solve((eq1, eq2), (x, y))
print(f"Solution: x = {solution[x]}, y = {solution[y]}")
```

Observation 1: `Solution: x = 5/7, y = 13/7`

Rationale 2: "Verify by substituting back: 2(5/7) + 3(13/7) = 10/7 + 39/7 = 49/7 = 7. 4(5/7) - 13/7 = 20/7 - 13/7 = 7/7 = 1. Both equations satisfied."

Action 2:
```python
from fractions import Fraction
x_val, y_val = Fraction(5, 7), Fraction(13, 7)
check1 = 2*x_val + 3*y_val
check2 = 4*x_val - y_val
print(f"Eq1 check: {check1} == 7? {check1 == 7}")
print(f"Eq2 check: {check2} == 1? {check2 == 1}")
```

Observation 2: `Eq1 check: 7 == 7? True`, `Eq2 check: 1 == 1? True`

Output: **x = 5/7, y = 13/7**

## Best Practices

- **Do:** Always produce a text transcription of the image content before reasoning. This OCR grounding step catches misread symbols early and provides a textual anchor for the reasoning chain.
- **Do:** Use `sympy` for symbolic computation (limits, derivatives, integrals, equation solving) and reserve `numpy`/`math` for numerical evaluation. Symbolic results are exact and easier to verify.
- **Do:** Print intermediate results in every Python Action block. The Observation must be explicit — never assume what code will output.
- **Do:** Verify every final answer with a separate computation (back-substitution, dimensional analysis, or boundary check). The verification step catches errors in both OCR and reasoning.
- **Avoid:** Attempting to solve complex equations purely through natural-language reasoning without code. The whole point of tool integration is to offload computation to Python, where it is deterministic and verifiable.
- **Avoid:** Skipping the OCR/transcription step for "simple-looking" images. Even clean typeset math frequently contains ambiguous characters (subscripts vs. multiplication, similar-looking symbols) that cause silent errors downstream.
- **Avoid:** Writing monolithic code blocks that solve the entire problem in one Action. Break computation into steps that align with your rationales so each Observation is interpretable and debuggable.

## Error Handling

| Error | Cause | Fix |
|-------|-------|-----|
| OCR misreads a symbol (e.g., `t` as `+`) | Low-resolution image or unusual font | Cross-reference OCR output with the image; ask the user to clarify ambiguous symbols |
| SymPy returns an unexpected form (e.g., unsimplified) | Expression complexity | Apply `simplify()`, `trigsimp()`, or `radsimp()` explicitly; try alternative solution methods |
| Code execution error (NameError, TypeError) | Missing import or wrong function signature | Check imports; use `sympy` for symbolic and `math`/`numpy` for numeric — don't mix them |
| Rationale-Observation mismatch | Reasoning error in the rationale step | Do not proceed — re-examine the rationale, check the code logic, and regenerate both |
| Image contains a diagram that OCR cannot transcribe | Pure geometric/visual content | Describe the diagram in natural language from the image; extract coordinates, labels, and relationships manually |
| Problem requires domain-specific knowledge (physics units, chemistry notation) | Out-of-scope visual content | State the interpretation explicitly in the rationale, convert to pure math, and solve the mathematical core |

## Limitations

- **Diagram-heavy problems.** Pure geometry diagrams (without labeled values) require spatial reasoning that OCR cannot capture. The model must interpret the diagram directly, which remains error-prone for complex figures.
- **Handwriting quality.** Heavily stylized or messy handwriting can defeat OCR entirely. If the user's photo is low-quality, ask for a clearer image or manual transcription of key parts.
- **Multi-page problems.** The framework processes one image at a time. For problems spanning multiple pages, each page must be processed separately, and the user should indicate continuity.
- **Non-mathematical visual content.** This technique is designed for mathematical reasoning. Charts, graphs, and data visualizations may benefit from a similar decomposition but require different tool integrations (e.g., data extraction rather than symbolic math).
- **Very large expressions.** Extremely long equations or systems with dozens of variables may exceed practical code generation limits. In such cases, break the problem into sub-problems before applying the loop.

## Reference

[VisTIRA: Closing the Image-Text Modality Gap in Visual Math Reasoning via Structured Tool Integration](https://arxiv.org/abs/2601.14440) — Look for Algorithm 1 (the Rationale-Action-Observation loop), the OCR grounding ablation (Table 2), and the modality gap measurements across model sizes (Table 1).