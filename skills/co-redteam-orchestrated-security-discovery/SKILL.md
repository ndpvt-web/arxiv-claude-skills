---
name: "co-redteam-orchestrated-security-discovery"
description: "Large language models (LLMs) have shown promise in assisting cybersecurity tasks, yet existing approaches struggle with automatic vulnerability discovery and exploitation due to limited interaction... Implements techniques from the paper 'Co-RedTeam: Orchestrated Security Discovery and Exploitation with LLM Agents' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (agent framework), (security) or when the user references techniques from this research area."
---

# Co-RedTeam: Orchestrated Security Discovery and Exploitation with LLM Agents

**Source:** [https://arxiv.org/abs/2602.02164v2](https://arxiv.org/abs/2602.02164v2)
**Category:** cs.LG | **Published:** 2026-02-02 | **Skill Score:** 74
**Authors:** Pengfei He, Ash Fox, Lesly Miculicich...

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

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

> Large language models (LLMs) have shown promise in assisting cybersecurity tasks, yet existing approaches struggle with automatic vulnerability discovery and exploitation due to limited interaction, weak execution grounding, and a lack of experience reuse. We propose Co-RedTeam, a security-aware multi-agent framework designed to mirror real-world red-teaming workflows by integrating security-domain knowledge, code-aware analysis, execution-grounded iterative reasoning, and long-term memory. Co-R

Refer to the [full paper](https://arxiv.org/abs/2602.02164v2) for detailed methodology.