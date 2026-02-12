---
name: "efficient-adversarial-attacks-on"
description: "Bandit algorithms have recently emerged as a powerful tool for evaluating machine learning models, including generative image models and large language models, by efficiently identifying top-perfor... Implements techniques from the paper 'Efficient Adversarial Attacks on High-dimensional Offline Bandits' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (security) or when the user references techniques from this research area."
---

# Efficient Adversarial Attacks on High-dimensional Offline Bandits

**Source:** [https://arxiv.org/abs/2602.01658v1](https://arxiv.org/abs/2602.01658v1)
**Category:** cs.LG | **Published:** 2026-02-02 | **Skill Score:** 58
**Authors:** Seyed Mohammad Hadi Hosseini, Amir Najafi, Mahdieh Soleymani Baghshah

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

> Bandit algorithms have recently emerged as a powerful tool for evaluating machine learning models, including generative image models and large language models, by efficiently identifying top-performing candidates without exhaustive comparisons. These methods typically rely on a reward model, often distributed with public weights on platforms such as Hugging Face, to provide feedback to the bandit. While online evaluation is expensive and requires repeated trials, offline evaluation with logged d

Refer to the [full paper](https://arxiv.org/abs/2602.01658v1) for detailed methodology.