---
name: "robustness-of-vision-language"
description: "Vision-Language Models (VLMs) are now a core part of modern AI. Implements techniques from the paper 'Robustness of Vision Language Models Against Split-Image Harmful Input Attacks' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (search & retrieval), (prompt engineering), (security) or when the user references techniques from this research area."
---

# Robustness of Vision Language Models Against Split-Image Harmful Input Attacks

**Source:** [https://arxiv.org/abs/2602.08136v1](https://arxiv.org/abs/2602.08136v1)
**Category:** cs.CV | **Published:** 2026-02-08 | **Skill Score:** 64
**Authors:** Md Rafi Ur Rashid, MD Sadik Hossain Shanto, Vishnu Asutosh Dasu...

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

> Vision-Language Models (VLMs) are now a core part of modern AI. Recent work proposed several visual jailbreak attacks using single/ holistic images. However, contemporary VLMs demonstrate strong robustness against such attacks due to extensive safety alignment through preference optimization (e.g., RLHF). In this work, we identify a new vulnerability: while VLM pretraining and instruction tuning generalize well to split-image inputs, safety alignment is typically performed only on holistic image

Refer to the [full paper](https://arxiv.org/abs/2602.08136v1) for detailed methodology.