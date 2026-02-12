---
name: "cipher-cryptographic-insecurity-profiling"
description: "Large language models (LLMs) are increasingly used to assist developers with code, yet their implementations of cryptographic functionality often contain exploitable flaws. Implements techniques from the paper 'CIPHER: Cryptographic Insecurity Profiling via Hybrid Evaluation of Responses' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (search & retrieval), (prompt engineering), (security) or when the user references techniques from this research area."
---

# CIPHER: Cryptographic Insecurity Profiling via Hybrid Evaluation of Responses

**Source:** [https://arxiv.org/abs/2602.01438v2](https://arxiv.org/abs/2602.01438v2)
**Category:** cs.CR | **Published:** 2026-02-01 | **Skill Score:** 58
**Authors:** Max Manolov, Tony Gao, Siddharth Shukla...

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

- **Proposed technique:** cipher(cryptographic insecurity profiling via hybrid evaluation of responses)

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

## Security Considerations

- Check for OWASP Top 10 vulnerabilities
- Validate all user inputs at system boundaries
- Use parameterized queries for database access
- Follow the principle of least privilege

## Research Context

> Large language models (LLMs) are increasingly used to assist developers with code, yet their implementations of cryptographic functionality often contain exploitable flaws. Minor design choices (e.g., static initialization vectors or missing authentication) can silently invalidate security guarantees. We introduce CIPHER(Cryptographic Insecurity Profiling via Hybrid Evaluation of Responses), a benchmark for measuring cryptographic vulnerability incidence in LLM-generated Python code under contro

Refer to the [full paper](https://arxiv.org/abs/2602.01438v2) for detailed methodology.