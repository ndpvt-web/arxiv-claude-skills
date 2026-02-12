---
name: "the-compliance-paradox-semanticinstruction"
description: "The rapid integration of Large Language Models (LLMs) into educational assessment rests on the unverified assumption that instruction following capability translates directly to objective adjudicat... Implements techniques from the paper 'The Compliance Paradox: Semantic-Instruction Decoupling in Automated Academic Code Evaluation' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (search & retrieval), (prompt engineering), (security) or when the user references techniques from this research area."
---

# The Compliance Paradox: Semantic-Instruction Decoupling in Automated Academic Code Evaluation

**Source:** [https://arxiv.org/abs/2601.21360v1](https://arxiv.org/abs/2601.21360v1)
**Category:** cs.CL | **Published:** 2026-01-29 | **Skill Score:** 95
**Authors:** Devanshu Sahoo, Manish Prasad, Vasudev Majhi...

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

## Security Considerations

- Check for OWASP Top 10 vulnerabilities
- Validate all user inputs at system boundaries
- Use parameterized queries for database access
- Follow the principle of least privilege

## Research Context

> The rapid integration of Large Language Models (LLMs) into educational assessment rests on the unverified assumption that instruction following capability translates directly to objective adjudication. We demonstrate that this assumption is fundamentally flawed. Instead of evaluating code quality, models frequently decouple from the submission's logic to satisfy hidden directives, a systemic vulnerability we term the Compliance Paradox, where models fine-tuned for extreme helpfulness are vulnera

Refer to the [full paper](https://arxiv.org/abs/2601.21360v1) for detailed methodology.