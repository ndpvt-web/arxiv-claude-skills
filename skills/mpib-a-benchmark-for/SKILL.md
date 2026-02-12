---
name: "mpib-a-benchmark-for"
description: "Large Language Models (LLMs) and Retrieval-Augmented Generation (RAG) systems are increasingly integrated into clinical workflows; however, prompt injection attacks can steer these systems toward c... Implements techniques from the paper 'MPIB: A Benchmark for Medical Prompt Injection Attacks and Clinical Safety in LLMs' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (devops automation), (data processing), (search & retrieval), (prompt engineering) or when the user references techniques from this research area."
---

# MPIB: A Benchmark for Medical Prompt Injection Attacks and Clinical Safety in LLMs

**Source:** [https://arxiv.org/abs/2602.06268v1](https://arxiv.org/abs/2602.06268v1)
**Category:** cs.CL | **Published:** 2026-02-06 | **Skill Score:** 89
**Authors:** Junhyeok Lee, Han Jang, Kyu Sung Choi

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

- **Proposed technique:** the medical prompt injection benchmark (mpib)
- **Retrieval-augmented** approach for grounding responses in external knowledge

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

## Research Context

> Large Language Models (LLMs) and Retrieval-Augmented Generation (RAG) systems are increasingly integrated into clinical workflows; however, prompt injection attacks can steer these systems toward clinically unsafe or misleading outputs. We introduce the Medical Prompt Injection Benchmark (MPIB), a dataset-and-benchmark suite for evaluating clinical safety under both direct prompt injection and indirect, RAG-mediated injection across clinically grounded tasks. MPIB emphasizes outcome-level risk v

Refer to the [full paper](https://arxiv.org/abs/2602.06268v1) for detailed methodology.