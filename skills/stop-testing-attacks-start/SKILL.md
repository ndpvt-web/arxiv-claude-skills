---
name: "stop-testing-attacks-start"
description: "Large Language Models (LLMs) deploy safety mechanisms to prevent harmful outputs, yet these defenses remain vulnerable to adversarial prompts. Implements techniques from the paper 'Stop Testing Attacks, Start Diagnosing Defenses: The Four-Checkpoint Framework Reveals Where LLM Safety Breaks' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (testing), (search & retrieval), (prompt engineering), (security) or when the user references techniques from this research area."
---

# Stop Testing Attacks, Start Diagnosing Defenses: The Four-Checkpoint Framework Reveals Where LLM Safety Breaks

**Source:** [https://arxiv.org/abs/2602.09629v1](https://arxiv.org/abs/2602.09629v1)
**Category:** cs.CR | **Published:** 2026-02-10 | **Skill Score:** 83
**Authors:** Hayfa Dhabhi, Kashyap Thimmaraju

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

- **Proposed technique:** that llm safety operates as a sequential pipeline with distinct checkpoints
- **Proposed technique:** the \textbf{four-checkpoint framework}

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

> Large Language Models (LLMs) deploy safety mechanisms to prevent harmful outputs, yet these defenses remain vulnerable to adversarial prompts. While existing research demonstrates that jailbreak attacks succeed, it does not explain \textit{where} defenses fail or \textit{why}.   To address this gap, we propose that LLM safety operates as a sequential pipeline with distinct checkpoints. We introduce the \textbf{Four-Checkpoint Framework}, which organizes safety mechanisms along two dimensions: pr

Refer to the [full paper](https://arxiv.org/abs/2602.09629v1) for detailed methodology.