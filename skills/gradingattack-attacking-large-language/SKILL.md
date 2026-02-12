---
name: "gradingattack-attacking-large-language"
description: "Large language models (LLMs) have demonstrated remarkable potential for automatic short answer grading (ASAG), significantly boosting student assessment efficiency and scalability in educational sc... Implements techniques from the paper 'GradingAttack: Attacking Large Language Models Towards Short Answer Grading Ability' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (prompt engineering), (security) or when the user references techniques from this research area."
---

# GradingAttack: Attacking Large Language Models Towards Short Answer Grading Ability

**Source:** [https://arxiv.org/abs/2602.00979v1](https://arxiv.org/abs/2602.00979v1)
**Category:** cs.CR | **Published:** 2026-02-01 | **Skill Score:** 86
**Authors:** Xueyi Li, Zhuoneng Zhou, Zitao Liu...

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

- **Proposed technique:** gradingattack

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

> Large language models (LLMs) have demonstrated remarkable potential for automatic short answer grading (ASAG), significantly boosting student assessment efficiency and scalability in educational scenarios. However, their vulnerability to adversarial manipulation raises critical concerns about automatic grading fairness and reliability. In this paper, we introduce GradingAttack, a fine-grained adversarial attack framework that systematically evaluates the vulnerability of LLM based ASAG models. S

Refer to the [full paper](https://arxiv.org/abs/2602.00979v1) for detailed methodology.