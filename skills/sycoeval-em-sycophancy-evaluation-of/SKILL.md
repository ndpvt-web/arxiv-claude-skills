---
name: "sycoeval-em-sycophancy-evaluation-of"
description: "Large language models (LLMs) show promise in clinical decision support yet risk acquiescing to patient pressure for inappropriate care. Implements techniques from the paper 'SycoEval-EM: Sycophancy Evaluation of Large Language Models in Simulated Clinical Encounters for Emergency Care' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (agent framework), (security) or when the user references techniques from this research area."
---

# SycoEval-EM: Sycophancy Evaluation of Large Language Models in Simulated Clinical Encounters for Emergency Care

**Source:** [https://arxiv.org/abs/2601.16529v1](https://arxiv.org/abs/2601.16529v1)
**Category:** cs.AI | **Published:** 2026-01-23 | **Skill Score:** 68
**Authors:** Dongshen Peng, Yi Wang, Carl Preiksaitis...

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

- **Proposed technique:** sycoeval-em
- **Multi-agent architecture** for task decomposition and parallel execution

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

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Large language models (LLMs) show promise in clinical decision support yet risk acquiescing to patient pressure for inappropriate care. We introduce SycoEval-EM, a multi-agent simulation framework evaluating LLM robustness through adversarial patient persuasion in emergency medicine. Across 20 LLMs and 1,875 encounters spanning three Choosing Wisely scenarios, acquiescence rates ranged from 0-100\%. Models showed higher vulnerability to imaging requests (38.8\%) than opioid prescriptions (25.0\%

Refer to the [full paper](https://arxiv.org/abs/2601.16529v1) for detailed methodology.