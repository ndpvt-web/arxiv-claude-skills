---
name: "gaming-the-judge-unfaithful"
description: "Large language models (LLMs) are increasingly used as judges to evaluate agent performance, particularly in non-verifiable settings where judgments rely on agent trajectories including chain-of-tho... Implements techniques from the paper 'Gaming the Judge: Unfaithful Chain-of-Thought Can Undermine Agent Evaluation' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (agent framework), (prompt engineering), (security) or when the user references techniques from this research area."
---

# Gaming the Judge: Unfaithful Chain-of-Thought Can Undermine Agent Evaluation

**Source:** [https://arxiv.org/abs/2601.14691v2](https://arxiv.org/abs/2601.14691v2)
**Category:** cs.AI | **Published:** 2026-01-21 | **Skill Score:** 69
**Authors:** Muhammad Khalifa, Lajanugen Logeswaran, Jaekyeom Kim...

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

- **Chain-of-thought reasoning** for improved step-by-step problem solving

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

> Large language models (LLMs) are increasingly used as judges to evaluate agent performance, particularly in non-verifiable settings where judgments rely on agent trajectories including chain-of-thought (CoT) reasoning. This paradigm implicitly assumes that the agent's CoT faithfully reflects both its internal reasoning and the underlying environment state. We show this assumption is brittle: LLM judges are highly susceptible to manipulation of agent reasoning traces. By systematically rewriting 

Refer to the [full paper](https://arxiv.org/abs/2601.14691v2) for detailed methodology.