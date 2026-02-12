---
name: "when-agents-misremember-collectively"
description: "Recent advancements in large language models (LLMs) have significantly enhanced the capabilities of collaborative multi-agent systems, enabling them to address complex challenges. Implements techniques from the paper 'When Agents \"Misremember\" Collectively: Exploring the Mandela Effect in LLM-based Multi-Agent Systems' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (search & retrieval), (agent framework), (prompt engineering), (security) or when the user references techniques from this research area."
---

# When Agents "Misremember" Collectively: Exploring the Mandela Effect in LLM-based Multi-Agent Systems

**Source:** [https://arxiv.org/abs/2602.00428v1](https://arxiv.org/abs/2602.00428v1)
**Category:** cs.CL | **Published:** 2026-01-31 | **Skill Score:** 82
**Authors:** Naen Xu, Hengyu An, Shuo Shi...

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

> Recent advancements in large language models (LLMs) have significantly enhanced the capabilities of collaborative multi-agent systems, enabling them to address complex challenges. However, within these multi-agent systems, the susceptibility of agents to collective cognitive biases remains an underexplored issue. A compelling example is the Mandela effect, a phenomenon where groups collectively misremember past events as a result of false details reinforced through social influence and internali

Refer to the [full paper](https://arxiv.org/abs/2602.00428v1) for detailed methodology.