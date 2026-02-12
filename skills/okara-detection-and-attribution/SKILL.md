---
name: "okara-detection-and-attribution"
description: "Transport Layer Security (TLS) is fundamental to secure online communication, yet vulnerabilities in certificate validation that enable Man-in-the-Middle (MitM) attacks remain a pervasive threat in... Implements techniques from the paper 'Okara: Detection and Attribution of TLS Man-in-the-Middle Vulnerabilities in Android Apps with Foundation Models' for analyze code for bugs, vulnerabilities, and quality issues. Use when tasks involve (code analysis), (search & retrieval), (agent framework), (security), (design & ui) or when the user references techniques from this research area."
---

# Okara: Detection and Attribution of TLS Man-in-the-Middle Vulnerabilities in Android Apps with Foundation Models

**Source:** [https://arxiv.org/abs/2601.22770v1](https://arxiv.org/abs/2601.22770v1)
**Category:** cs.CR | **Published:** 2026-01-30 | **Skill Score:** 60
**Authors:** Haoyun Yang, Ronghong Huang, Yong Fang...

## Core Capability

Analyze code for bugs, vulnerabilities, and quality issues.

## Key Techniques

- **Leverages:** foundation models to automate the detection and deep attribution of tls mitm vulnerabilities (tmvs)

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

> Transport Layer Security (TLS) is fundamental to secure online communication, yet vulnerabilities in certificate validation that enable Man-in-the-Middle (MitM) attacks remain a pervasive threat in Android apps. Existing detection tools are hampered by low-coverage UI interaction, costly instrumentation, and a lack of scalable root-cause analysis. We present Okara, a framework that leverages foundation models to automate the detection and deep attribution of TLS MitM Vulnerabilities (TMVs). Okar

Refer to the [full paper](https://arxiv.org/abs/2601.22770v1) for detailed methodology.