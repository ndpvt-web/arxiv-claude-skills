---
name: "iresolvex-multilayered-indirect-call"
description: "Indirect call resolution remains a key challenge in reverse engineering and control-flow graph recovery, especially for stripped or optimized binaries. Implements techniques from the paper 'iResolveX: Multi-Layered Indirect Call Resolution via Static Reasoning and Learning-Augmented Refinement' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (devops automation), (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# iResolveX: Multi-Layered Indirect Call Resolution via Static Reasoning and Learning-Augmented Refinement

**Source:** [https://arxiv.org/abs/2601.17888v1](https://arxiv.org/abs/2601.17888v1)
**Category:** cs.SE | **Published:** 2026-01-25 | **Skill Score:** 61
**Authors:** Monika Santra, Bokai Zhang, Mark Lim...

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Workflow

1. Read and parse the target source code files
2. Identify code smells, anti-patterns, and potential bugs
3. Check for security vulnerabilities (OWASP Top 10)
4. Assess code quality metrics and suggest improvements
5. Provide actionable feedback with specific line references

## Code Quality Standards

- Follow language-specific idioms and best practices
- Include appropriate error handling
- Add type annotations where applicable
- Avoid introducing security vulnerabilities

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Indirect call resolution remains a key challenge in reverse engineering and control-flow graph recovery, especially for stripped or optimized binaries. Static analysis is sound but often over-approximates, producing many false positives, whereas machine-learning approaches can improve precision but may sacrifice completeness and generalization. We present iResolveX, a hybrid multi-layered framework that combines conservative static analysis with learning-based refinement. The first layer applies

Refer to the [full paper](https://arxiv.org/abs/2601.17888v1) for detailed methodology.