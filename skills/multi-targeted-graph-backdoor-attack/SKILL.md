---
name: "multi-targeted-graph-backdoor-attack"
description: "Graph neural network (GNN) have demonstrated exceptional performance in solving critical problems across diverse domains yet remain susceptible to backdoor attacks. Implements techniques from the paper 'Multi-Targeted Graph Backdoor Attack' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (security) or when the user references techniques from this research area."
---

# Multi-Targeted Graph Backdoor Attack

**Source:** [https://arxiv.org/abs/2601.15474v1](https://arxiv.org/abs/2601.15474v1)
**Category:** cs.LG | **Published:** 2026-01-21 | **Skill Score:** 72
**Authors:** Md Nabi Newaz Khan, Abdullah Arafat Miah, Yu Bi

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

- **Proposed technique:** the first multi-targeted backdoor attack for graph classification task

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

> Graph neural network (GNN) have demonstrated exceptional performance in solving critical problems across diverse domains yet remain susceptible to backdoor attacks. Existing studies on backdoor attack for graph classification are limited to single target attack using subgraph replacement based mechanism where the attacker implants only one trigger into the GNN model. In this paper, we introduce the first multi-targeted backdoor attack for graph classification task, where multiple triggers simult

Refer to the [full paper](https://arxiv.org/abs/2601.15474v1) for detailed methodology.