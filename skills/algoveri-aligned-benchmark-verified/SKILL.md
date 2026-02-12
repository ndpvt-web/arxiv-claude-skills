---
name: "algoveri-aligned-benchmark-verified"
description: "Vericoding refers to the generation of formally verified code from rigorous specifications Implements the AlgoVeri approach. Use for: code-generation. Triggers: 'generate code for...', 'write a function that...'"
---

# AlgoVeri: An Aligned Benchmark for Verified Code Generation on Classical Algorithms

This skill implements the approach described in *AlgoVeri: An Aligned Benchmark for Verified Code Generation on Classical Algorithms*. Vericoding refers to the generation of formally verified code from rigorous specifications

**Paper:** [https://arxiv.org/abs/2602.09464v1](https://arxiv.org/abs/2602.09464v1) | **Category:** cs.SE | **Published:** 2026-02-10
**Code:** [https://github.com/haoyuzhao123/algoveri.](https://github.com/haoyuzhao123/algoveri.)

## When to Use

- When the user needs to generate code that implements a specific algorithm or pattern from research

## Core Technique

Vericoding refers to the generation of formally verified code from rigorous specifications. Recent AI models show promise in vericoding, but a unified methodology for cross-paradigm evaluation is lacking. Existing benchmarks test only individual languages/tools (e.g., Dafny, Verus, and Lean) and each covers very different tasks, so the performance numbers are not directly comparable. We address this gap with AlgoVeri, a benchmark that evaluates vericoding of $77$ classical algorithms in Dafny, Verus, and Lean. By enforcing identical functional contracts, AlgoVeri reveals critical capability gaps in verification systems. While frontier models achieve tractable success in Dafny ($40.3$% for Gemini-3 Flash), where high-level abstractions and SMT automation simplify the workflow, performance collapses under the systems-level memory constraints of Verus ($24.7$%) and the explicit proof construction required by Lean (7.8%). Beyond aggregate metrics, we uncover a sharp divergence in test-time compute dynamics: Gemini-3 effectively utilizes iterative repair to boost performance (e.g., tripling pass rates in Dafny), whereas GPT-OSS saturates early. Finally, our error analysis shows that language design affects the refinement trajectory: while Dafny allows models to focus on logical correctness, Verus and Lean trap models in persistent syntactic and semantic barriers. All data and evaluation code can be found at https://github.com/haoyuzhao123/algoveri.

**Key Results:** While frontier models achieve tractable success in Dafny ($40.3$% for Gemini-3 Flash), where high-level abstractions and SMT automation simplify the workflow, performance collapses under the systems-level memory constraints of Verus ($24.7$%) and the explicit proof construction required by Lean (7.8%).

## Step-by-Step Workflow

1. Parse the user's requirements carefully -- identify language, framework, and constraints
2. Apply the AlgoVeri approach to plan the code structure before writing
3. Break the implementation into logical components (functions, classes, modules)
4. Generate each component with proper error handling, type annotations, and edge case coverage
5. Add docstrings and comments only where the logic is non-obvious
6. Validate the generated code: check for compilation errors, missing imports, and security issues
7. Test with representative inputs including edge cases
8. Refine based on test results until the code is production-ready

## Examples

**Example 1: Applying the technique to code generation**

```
User: Use the AlgoVeri approach to generate a data processing pipeline

Approach:
1. Identify the pipeline stages from the user's description
2. Apply AlgoVeri's decomposition to design each stage independently
3. Generate code for each stage with clear interfaces between them
4. Wire the stages together with error handling at each boundary
5. Add logging and monitoring hooks for observability

Output: A complete, runnable pipeline with clear stage separation,
error handling, and documentation for each component.
```

**Example 2: Debugging and iteration**

```
User: The initial approach isn't working well, can you refine it?

Approach:
1. Identify where the current approach is falling short
2. Consult the paper's ablation studies for guidance on what matters most
3. Adjust parameters or approach based on the paper's recommendations
4. Re-run and compare results

Output: An improved solution with explanation of what changed and why,
referencing the paper's findings about what factors affect performance.
```

## Best Practices

**Do:**
- Read the full problem description before applying AlgoVeri
- Start with the simplest application of the technique and add complexity incrementally
- Validate intermediate results at each step of the workflow
- Adapt the approach to the specific domain rather than applying it rigidly

**Avoid:**
- Applying the technique blindly without understanding the problem context
- Skipping validation steps to save time -- errors compound quickly
- Over-engineering the solution beyond what the task requires
- Ignoring the paper's stated limitations and applying it outside its scope

## Error Handling

- **Technique doesn't apply**: If the problem doesn't match AlgoVeri's assumptions, fall back to general-purpose reasoning and explain why
- **Partial results**: If some steps succeed but others fail, present the partial results with clear indication of what's missing
- **Conflicting information**: When sources disagree, present both sides with evidence and let the user decide
- **Performance issues**: If the approach is too slow, simplify by reducing the number of decomposition steps

## Limitations

- This skill encodes the *methodology* from the paper, not a trained model -- results depend on Claude's general capabilities
- The paper was evaluated on specific benchmarks; real-world tasks may differ significantly
- For tasks clearly outside the paper's scope, prefer general-purpose approaches

## Reference

**[AlgoVeri: An Aligned Benchmark for Verified Code Generation on Classical Algorithms](https://arxiv.org/abs/2602.09464v1)**
Key finding: While frontier models achieve tractable success in Dafny ($40.3$% for Gemini-3 Flash), where high-level abstractions and SMT automation simplify the workflow, performance collapses under the systems-level memory constraints of Verus ($24.7$%) and the explicit proof construction required by Lean (7.8%).
Implementation: [https://github.com/haoyuzhao123/algoveri.](https://github.com/haoyuzhao123/algoveri.)
Look for: methodology section, experimental setup, and ablation studies for tuning guidance.