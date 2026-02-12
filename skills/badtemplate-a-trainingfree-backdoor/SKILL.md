---
name: "badtemplate-a-trainingfree-backdoor"
description: "Chat template is a common technique used in the training and inference stages of Large Language Models (LLMs). Implements techniques from the paper 'BadTemplate: A Training-Free Backdoor Attack via Chat Template Against Large Language Models' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (prompt engineering), (security) or when the user references techniques from this research area."
---

# BadTemplate: A Training-Free Backdoor Attack via Chat Template Against Large Language Models

**Source:** [https://arxiv.org/abs/2602.05401v1](https://arxiv.org/abs/2602.05401v1)
**Category:** cs.CR | **Published:** 2026-02-05 | **Skill Score:** 63
**Authors:** Zihan Wang, Hongwei Li, Rui Zhang...

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

> Chat template is a common technique used in the training and inference stages of Large Language Models (LLMs). It can transform input and output data into role-based and templated expressions to enhance the performance of LLMs. However, this also creates a breeding ground for novel attack surfaces. In this paper, we first reveal that the customizability of chat templates allows an attacker who controls the template to inject arbitrary strings into the system prompt without the user's notice. Bui

Refer to the [full paper](https://arxiv.org/abs/2602.05401v1) for detailed methodology.