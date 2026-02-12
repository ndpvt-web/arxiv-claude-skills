---
name: "reconstructing-training-data-from"
description: "Adapter-based Federated Large Language Models (FedLLMs) are widely adopted to reduce the computational, storage, and communication overhead of full-parameter fine-tuning for web-scale applications ... Implements techniques from the paper 'Reconstructing Training Data from Adapter-based Federated Large Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (security) or when the user references techniques from this research area."
---

# Reconstructing Training Data from Adapter-based Federated Large Language Models

**Source:** [https://arxiv.org/abs/2601.17533v1](https://arxiv.org/abs/2601.17533v1)
**Category:** cs.CR | **Published:** 2026-01-24 | **Skill Score:** 80
**Authors:** Silong Chen, Yuchuan Luo, Guilin Deng...

## Core Capability

Search, retrieve, and synthesize information.

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Security Considerations

- Check for OWASP Top 10 vulnerabilities
- Validate all user inputs at system boundaries
- Use parameterized queries for database access
- Follow the principle of least privilege

## Research Context

> Adapter-based Federated Large Language Models (FedLLMs) are widely adopted to reduce the computational, storage, and communication overhead of full-parameter fine-tuning for web-scale applications while preserving user privacy. By freezing the backbone and training only compact low-rank adapters, these methods appear to limit gradient leakage and thwart existing Gradient Inversion Attacks (GIAs).   Contrary to this assumption, we show that low-rank adapters create new, exploitable leakage channe

Refer to the [full paper](https://arxiv.org/abs/2601.17533v1) for detailed methodology.