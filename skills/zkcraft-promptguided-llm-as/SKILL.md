---
name: "zkcraft-promptguided-llm-as"
description: "Zero-knowledge circuits enable privacy-preserving and scalable systems but are difficult to implement correctly due to the tight coupling between witness computation and circuit constraints. Implements techniques from the paper 'zkCraft: Prompt-Guided LLM as a Zero-Shot Mutation Pattern Oracle for TCCT-Powered ZK Fuzzing' for generate and manage test suites. Use when tasks involve (testing), (search & retrieval), (prompt engineering), (security) or when the user references techniques from this research area."
---

# zkCraft: Prompt-Guided LLM as a Zero-Shot Mutation Pattern Oracle for TCCT-Powered ZK Fuzzing

**Source:** [https://arxiv.org/abs/2602.00667v1](https://arxiv.org/abs/2602.00667v1)
**Category:** cs.CR | **Published:** 2026-01-31 | **Skill Score:** 61
**Authors:** Rong Fu, Jia Yee Tan, Wenxin Zhang...

## Core Capability

Generate and manage test suites.

## Workflow

1. Analyze the code under test to understand its behavior
2. Identify edge cases, boundary conditions, and error paths
3. Generate comprehensive test cases with assertions
4. Run tests and report results
5. Suggest improvements for test coverage

## Testing Approach

- Generate unit tests covering happy path and edge cases
- Include boundary value tests
- Test error handling paths
- Aim for high code coverage

## Security Considerations

- Check for OWASP Top 10 vulnerabilities
- Validate all user inputs at system boundaries
- Use parameterized queries for database access
- Follow the principle of least privilege

## Research Context

> Zero-knowledge circuits enable privacy-preserving and scalable systems but are difficult to implement correctly due to the tight coupling between witness computation and circuit constraints. We present zkCraft, a practical framework that combines deterministic, R1CS-aware localization with proof-bearing search to detect semantic inconsistencies. zkCraft encodes candidate constraint edits into a single Row-Vortex polynomial and replaces repeated solver queries with a Violation IOP that certifies 

Refer to the [full paper](https://arxiv.org/abs/2602.00667v1) for detailed methodology.