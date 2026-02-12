---
name: "alignment-drift-in-multimodal"
description: "Multimodal large language models (MLLMs) are increasingly deployed in real-world systems, yet their safety under adversarial prompting remains underexplored. Implements techniques from the paper 'Alignment Drift in Multimodal LLMs: A Two-Phase, Longitudinal Evaluation of Harm Across Eight Model Releases' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (prompt engineering), (security) or when the user references techniques from this research area."
---

# Alignment Drift in Multimodal LLMs: A Two-Phase, Longitudinal Evaluation of Harm Across Eight Model Releases

**Source:** [https://arxiv.org/abs/2602.04739v1](https://arxiv.org/abs/2602.04739v1)
**Category:** cs.CL | **Published:** 2026-02-04 | **Skill Score:** 80
**Authors:** Casey Ford, Madison Van Doren, Emily Dix

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

- **Proposed technique:** a two-phase evaluation of mllm harmlessness using a fixed benchmark of 726 adversarial prompts authored by 26 professional red teamers

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

> Multimodal large language models (MLLMs) are increasingly deployed in real-world systems, yet their safety under adversarial prompting remains underexplored. We present a two-phase evaluation of MLLM harmlessness using a fixed benchmark of 726 adversarial prompts authored by 26 professional red teamers. Phase 1 assessed GPT-4o, Claude Sonnet 3.5, Pixtral 12B, and Qwen VL Plus; Phase 2 evaluated their successors (GPT-5, Claude Sonnet 4.5, Pixtral Large, and Qwen Omni) yielding 82,256 human harm r

Refer to the [full paper](https://arxiv.org/abs/2602.04739v1) for detailed methodology.