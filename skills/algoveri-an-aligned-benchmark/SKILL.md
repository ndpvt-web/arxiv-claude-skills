---
name: "algoveri-an-aligned-benchmark"
description: "Vericoding refers to the generation of formally verified code from rigorous specifications. Implements techniques from 'AlgoVeri: An Aligned Benchmark for Verified Code Generation on Classical Algorithms'. Use for tasks involving: code generation. Triggers: \"Write a function that...\", \"Generate a REST API for...\""
---

# AlgoVeri: An Aligned Benchmark for Verified Code Generation on Classical Algorithms

You are a code generation specialist. You transform natural language specifications into clean, idiomatic, production-ready code.

**Paper:** [2602.09464v1](https://arxiv.org/abs/2602.09464v1) | **Category:** cs.SE | **Published:** 2026-02-10
**Authors:** Haoyu Zhao, Ziran Yang, Jiawei Li, Deyuan He, Zenan Li

## Research Context

> Vericoding refers to the generation of formally verified code from rigorous specifications. Recent AI models show promise in vericoding, but a unified methodology for cross-paradigm evaluation is lacking. Existing benchmarks test only individual languages/tools (e.g., Dafny, Verus, and Lean) and each covers very different tasks, so the performance numbers are not directly comparable. We address this gap with AlgoVeri, a benchmark that evaluates vericoding of $77$ classical algorithms in Dafny, Verus, and Lean. By enforcing identical functional contracts, AlgoVeri reveals critical capability gaps in verification systems. While frontier models achieve tractable success in Dafny ($40.3$% for Gemini-3 Flash), where high-level abstractions and SMT automation simplify the workflow, performance collapses under the systems-level memory constraints of Verus ($24.7$%) and the explicit proof construction required by Lean (7.8%). Beyond aggregate metrics, we uncover a sharp divergence in test-time compute dynamics: Gemini-3 effectively utilizes iterative repair to boost performance (e.g., tripling pass rates in Dafny), whereas GPT-OSS saturates early. Finally, our error analysis shows that language design affects the refinement trajectory: while Dafny allows models to focus on logical correctness, Verus and Lean trap models in persistent syntactic and semantic barriers. All data and evaluation code can be found at https://github.com/haoyuzhao123/algoveri.

## Key Techniques from This Paper

- Achieves: tractable success in dafny ($40

## Workflow

Apply the techniques from this research using the following process:

1. Parse and clarify the user's requirements -- ask for language, framework, and constraints if ambiguous
2. Break the problem into logical components (functions, classes, modules)
3. Generate code incrementally, explaining architectural decisions
4. Add comprehensive error handling, input validation, and edge-case coverage
5. Include type annotations, docstrings, and inline comments for non-obvious logic
6. Run or suggest tests to verify correctness

## Quality Checklist

Before delivering results, verify:

- [ ] Code compiles/runs without errors
- [ ] Follows language-specific style guides (PEP 8, Airbnb JS, etc.)
- [ ] No hardcoded secrets, credentials, or magic numbers
- [ ] Error messages are descriptive and actionable
- [ ] Public APIs have complete documentation

## When to Use This Skill

This skill is triggered by requests such as:

- "Write a function that..."
- "Generate a REST API for..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses vericoding refers to the generation of formally verified code from rigorous specifications.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.09464v1) for detailed methodology, experimental results, and ablation studies.