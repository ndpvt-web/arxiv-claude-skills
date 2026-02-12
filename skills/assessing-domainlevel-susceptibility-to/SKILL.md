---
name: "assessing-domainlevel-susceptibility-to"
description: "Emergent misalignment poses risks to AI safety as language models are increasingly used for autonomous tasks. Implements techniques from the paper 'Assessing Domain-Level Susceptibility to Emergent Misalignment from Narrow Finetuning' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (search & retrieval), (agent framework), (prompt engineering), (security) or when the user references techniques from this research area."
---

# Assessing Domain-Level Susceptibility to Emergent Misalignment from Narrow Finetuning

**Source:** [https://arxiv.org/abs/2602.00298v1](https://arxiv.org/abs/2602.00298v1)
**Category:** cs.AI | **Published:** 2026-01-30 | **Skill Score:** 73
**Authors:** Abhishek Mishra, Mugilan Arulvanan, Reshma Ashok...

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

- **Proposed technique:** a population of large language models (llms) fine-tuned on insecure datasets spanning 11 diverse domains

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

> Emergent misalignment poses risks to AI safety as language models are increasingly used for autonomous tasks. In this paper, we present a population of large language models (LLMs) fine-tuned on insecure datasets spanning 11 diverse domains, evaluating them both with and without backdoor triggers on a suite of unrelated user prompts. Our evaluation experiments on \texttt{Qwen2.5-Coder-7B-Instruct} and \texttt{GPT-4o-mini} reveal two key findings: (i) backdoor triggers increase the rate of misali

Refer to the [full paper](https://arxiv.org/abs/2602.00298v1) for detailed methodology.