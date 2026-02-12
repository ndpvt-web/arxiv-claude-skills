---
name: "yasa-scalable-multilanguage-taint"
description: "Modern enterprises increasingly adopt diverse technology stacks with various programming languages, posing significant challenges for static application security testing (SAST). Implements techniques from the paper 'YASA: Scalable Multi-Language Taint Analysis on the Unified AST at Ant Group' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (search & retrieval), (security) or when the user references techniques from this research area."
---

# YASA: Scalable Multi-Language Taint Analysis on the Unified AST at Ant Group

**Source:** [https://arxiv.org/abs/2601.17390v1](https://arxiv.org/abs/2601.17390v1)
**Category:** cs.SE | **Published:** 2026-01-24 | **Skill Score:** 74
**Authors:** Yayi Wang, Shenao Wang, Jian Zhao...

## Core Capability

Automate deployment, CI/CD, and infrastructure tasks.

## Workflow

1. Analyze the current infrastructure and deployment setup
2. Design automation workflows for the target environment
3. Generate configuration files (Docker, K8s, CI/CD pipelines)
4. Implement monitoring and alerting
5. Validate the automation works correctly

## Security Considerations

- Check for OWASP Top 10 vulnerabilities
- Validate all user inputs at system boundaries
- Use parameterized queries for database access
- Follow the principle of least privilege

## Research Context

> Modern enterprises increasingly adopt diverse technology stacks with various programming languages, posing significant challenges for static application security testing (SAST). Existing taint analysis tools are predominantly designed for single languages, requiring substantial engineering effort that scales with language diversity. While multi-language tools like CodeQL, Joern, and WALA attempt to address these challenges, they face limitations in intermediate representation design, analysis pr

Refer to the [full paper](https://arxiv.org/abs/2601.17390v1) for detailed methodology.