---
name: "jailbreaks-on-vision-language"
description: "Vision-language models (VLMs) have become central to tasks such as visual question answering, image captioning, and text-to-image generation. Implements techniques from the paper 'Jailbreaks on Vision Language Model via Multimodal Reasoning' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (content generation), (agent framework), (prompt engineering), (security) or when the user references techniques from this research area."
---

# Jailbreaks on Vision Language Model via Multimodal Reasoning

**Source:** [https://arxiv.org/abs/2601.22398v1](https://arxiv.org/abs/2601.22398v1)
**Category:** cs.CV | **Published:** 2026-01-29 | **Skill Score:** 67
**Authors:** Aarush Noheria, Yuguang Yao

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** a jailbreak framework that exploits post-training chain-of-thought (cot) prompting to construct stealthy prompts capable of bypassing safety filters
- **Chain-of-thought reasoning** for improved step-by-step problem solving

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

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Vision-language models (VLMs) have become central to tasks such as visual question answering, image captioning, and text-to-image generation. However, their outputs are highly sensitive to prompt variations, which can reveal vulnerabilities in safety alignment. In this work, we present a jailbreak framework that exploits post-training Chain-of-Thought (CoT) prompting to construct stealthy prompts capable of bypassing safety filters. To further increase attack success rates (ASR), we propose a Re

Refer to the [full paper](https://arxiv.org/abs/2601.22398v1) for detailed methodology.