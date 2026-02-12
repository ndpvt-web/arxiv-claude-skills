---
name: "evaluating-large-language-models"
description: "Early detection of security bug reports (SBRs) is critical for timely vulnerability mitigation. Implements techniques from the paper 'Evaluating Large Language Models for Security Bug Report Prediction' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (search & retrieval), (prompt engineering), (security) or when the user references techniques from this research area."
---

# Evaluating Large Language Models for Security Bug Report Prediction

**Source:** [https://arxiv.org/abs/2601.22921v1](https://arxiv.org/abs/2601.22921v1)
**Category:** cs.CR | **Published:** 2026-01-30 | **Skill Score:** 66
**Authors:** Farnaz Soltaniani, Shoaib Razzaq, Mohammad Ghafari

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

- **Proposed technique:** an evaluation of prompt-based engineering and fine-tuning approaches for predicting sbrs using large language models (llms)
- **Achievement:** a g-measure of 77% and a recall of 74% on average across all the datasets

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

> Early detection of security bug reports (SBRs) is critical for timely vulnerability mitigation. We present an evaluation of prompt-based engineering and fine-tuning approaches for predicting SBRs using Large Language Models (LLMs). Our findings reveal a distinct trade-off between the two approaches. Prompted proprietary models demonstrate the highest sensitivity to SBRs, achieving a G-measure of 77% and a recall of 74% on average across all the datasets, albeit at the cost of a higher false-posi

Refer to the [full paper](https://arxiv.org/abs/2601.22921v1) for detailed methodology.