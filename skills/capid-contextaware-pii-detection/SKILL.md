---
name: "capid-contextaware-pii-detection"
description: "Detecting personally identifiable information (PII) in user queries is critical for ensuring privacy in question-answering systems. Implements techniques from the paper 'CAPID: Context-Aware PII Detection for Question-Answering Systems' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (security) or when the user references techniques from this research area."
---

# CAPID: Context-Aware PII Detection for Question-Answering Systems

**Source:** [https://arxiv.org/abs/2602.10074v1](https://arxiv.org/abs/2602.10074v1)
**Category:** cs.CR | **Published:** 2026-02-10 | **Skill Score:** 62
**Authors:** Mariia Ponomarenko, Sepideh Abedini, Masoumeh Shafieinejad...

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

> Detecting personally identifiable information (PII) in user queries is critical for ensuring privacy in question-answering systems. Current approaches mainly redact all PII, disregarding the fact that some of them may be contextually relevant to the user's question, resulting in a degradation of response quality. Large language models (LLMs) might be able to help determine which PII are relevant, but due to their closed source nature and lack of privacy guarantees, they are unsuitable for sensit

Refer to the [full paper](https://arxiv.org/abs/2602.10074v1) for detailed methodology.