---
name: "confundo-learning-to-generate"
description: "Retrieval-augmented generation (RAG) is increasingly deployed in real-world applications, where its reference-grounded design makes outputs appear trustworthy. Implements techniques from the paper 'Confundo: Learning to Generate Robust Poison for Practical RAG Systems' for automate deployment, ci/cd, and infrastructure tasks. Use when tasks involve (devops automation), (data processing), (search & retrieval), (security), (web automation) or when the user references techniques from this research area."
---

# Confundo: Learning to Generate Robust Poison for Practical RAG Systems

**Source:** [https://arxiv.org/abs/2602.06616v1](https://arxiv.org/abs/2602.06616v1)
**Category:** cs.CR | **Published:** 2026-02-06 | **Skill Score:** 60
**Authors:** Haoyang Hu, Zhejun Jiang, Yueming Lyu...

## Core Capability

Automate deployment, CI/CD, and infrastructure tasks.

## Key Techniques

- **Retrieval-augmented** approach for grounding responses in external knowledge

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

> Retrieval-augmented generation (RAG) is increasingly deployed in real-world applications, where its reference-grounded design makes outputs appear trustworthy. This trust has spurred research on poisoning attacks that craft malicious content, inject it into knowledge sources, and manipulate RAG responses. However, when evaluated in practical RAG systems, existing attacks suffer from severely degraded effectiveness. This gap stems from two overlooked realities: (i) content is often processed befo

Refer to the [full paper](https://arxiv.org/abs/2602.06616v1) for detailed methodology.