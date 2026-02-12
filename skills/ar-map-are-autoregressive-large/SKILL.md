---
name: "ar-map-are-autoregressive-large"
description: "Diffusion Large Language Models (DLLMs) have emerged as a powerful alternative to autoregressive models, enabling parallel token generation across multiple positions. Implements techniques from the paper 'AR-MAP: Are Autoregressive Large Language Models Implicit Teachers for Diffusion Large Language Models?' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (security) or when the user references techniques from this research area."
---

# AR-MAP: Are Autoregressive Large Language Models Implicit Teachers for Diffusion Large Language Models?

**Source:** [https://arxiv.org/abs/2602.02178v2](https://arxiv.org/abs/2602.02178v2)
**Category:** cs.CL | **Published:** 2026-02-02 | **Skill Score:** 70
**Authors:** Liang Lin, Feng Xiong, Zengbin Wang...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Novel approach:** transfer learning framework
- **Leverages:** preference-aligned autoregressive llms (ar-llms) as implicit teachers for dllm alignment

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

> Diffusion Large Language Models (DLLMs) have emerged as a powerful alternative to autoregressive models, enabling parallel token generation across multiple positions. However, preference alignment of DLLMs remains challenging due to high variance introduced by Evidence Lower Bound (ELBO)-based likelihood estimation. In this work, we propose AR-MAP, a novel transfer learning framework that leverages preference-aligned autoregressive LLMs (AR-LLMs) as implicit teachers for DLLM alignment. We revea

Refer to the [full paper](https://arxiv.org/abs/2602.02178v2) for detailed methodology.