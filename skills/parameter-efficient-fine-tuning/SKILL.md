---
name: "parameter-efficient-fine-tuning"
description: "This study uses Jordanian law as a case study to explore the fine-tuning of the Llama-3.1 large language model for Arabic question-answering. Implements techniques from the paper 'Parameter Efficient Fine Tuning Llama 3.1 for Answering Arabic Legal Questions: A Case Study on Jordanian Laws' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Parameter Efficient Fine Tuning Llama 3.1 for Answering Arabic Legal Questions: A Case Study on Jordanian Laws

**Source:** [https://arxiv.org/abs/2601.17364v1](https://arxiv.org/abs/2601.17364v1)
**Category:** cs.CL | **Published:** 2026-01-24 | **Skill Score:** 70
**Authors:** Mohammed Fasha, Bassam Hammo, Bilal Sowan...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Leverages:** the unsloth framework for accelerated and resource-efficient training

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> This study uses Jordanian law as a case study to explore the fine-tuning of the Llama-3.1 large language model for Arabic question-answering. Two versions of the model - Llama-3.1-8B-bnb-4bit and Llama-3.1-8B-Instruct-bnb-4bit - were fine-tuned using parameter-efficient fine-tuning (PEFT) with LoRA adapters and 4-bit quantized models, leveraging the Unsloth framework for accelerated and resource-efficient training. A custom dataset of 6000 legal question-answer pairs was curated from Jordanian l

Refer to the [full paper](https://arxiv.org/abs/2601.17364v1) for detailed methodology.