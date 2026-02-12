---
name: "persona-jailbreaking-in-large"
description: "Large Language Models (LLMs) are increasingly deployed in domains such as education, mental health and customer support, where stable and consistent personas are critical for reliability. Implements techniques from the paper 'Persona Jailbreaking in Large Language Models' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (agent framework), (security) or when the user references techniques from this research area."
---

# Persona Jailbreaking in Large Language Models

**Source:** [https://arxiv.org/abs/2601.16466v1](https://arxiv.org/abs/2601.16466v1)
**Category:** cs.CL | **Published:** 2026-01-23 | **Skill Score:** 78
**Authors:** Jivnesh Sandhan, Fei Cheng, Tushar Sandhan...

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

- **Proposed technique:** the task of persona editi

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

> Large Language Models (LLMs) are increasingly deployed in domains such as education, mental health and customer support, where stable and consistent personas are critical for reliability. Yet, existing studies focus on narrative or role-playing tasks and overlook how adversarial conversational history alone can reshape induced personas. Black-box persona manipulation remains unexplored, raising concerns for robustness in realistic interactions. In response, we introduce the task of persona editi

Refer to the [full paper](https://arxiv.org/abs/2601.16466v1) for detailed methodology.