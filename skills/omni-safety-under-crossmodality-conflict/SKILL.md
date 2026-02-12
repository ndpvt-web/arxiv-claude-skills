---
name: "omni-safety-under-crossmodality-conflict"
description: "Omni-modal Large Language Models (OLLMs) greatly expand LLMs' multimodal capabilities but also introduce cross-modal safety risks. Implements techniques from the paper 'Omni-Safety under Cross-Modality Conflict: Vulnerabilities, Dynamics Mechanisms and Efficient Alignment' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (search & retrieval), (security) or when the user references techniques from this research area."
---

# Omni-Safety under Cross-Modality Conflict: Vulnerabilities, Dynamics Mechanisms and Efficient Alignment

**Source:** [https://arxiv.org/abs/2602.10161v1](https://arxiv.org/abs/2602.10161v1)
**Category:** cs.CR | **Published:** 2026-02-10 | **Skill Score:** 84
**Authors:** Kun Wang, Zherui Li, Zhenhong Zhou...

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

> Omni-modal Large Language Models (OLLMs) greatly expand LLMs' multimodal capabilities but also introduce cross-modal safety risks. However, a systematic understanding of vulnerabilities in omni-modal interactions remains lacking. To bridge this gap, we establish a modality-semantics decoupling principle and construct the AdvBench-Omni dataset, which reveals a significant vulnerability in OLLMs. Mechanistic analysis uncovers a Mid-layer Dissolution phenomenon driven by refusal vector magnitude sh

Refer to the [full paper](https://arxiv.org/abs/2602.10161v1) for detailed methodology.