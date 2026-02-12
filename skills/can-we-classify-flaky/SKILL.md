---
name: "can-we-classify-flaky"
description: "Flaky tests yield inconsistent results when they are repeatedly executed on the same code revision. Implements techniques from the paper 'Can We Classify Flaky Tests Using Only Test Code? An LLM-Based Empirical Study' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (prompt engineering) or when the user references techniques from this research area."
---

# Can We Classify Flaky Tests Using Only Test Code? An LLM-Based Empirical Study

**Source:** [https://arxiv.org/abs/2602.05465v1](https://arxiv.org/abs/2602.05465v1)
**Category:** cs.SE | **Published:** 2026-02-05 | **Skill Score:** 69
**Authors:** Alexander Berndt, Vekil Bekmyradov, Rainer Gemulla...

## Core Capability

Search, retrieve, and synthesize information.

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

> Flaky tests yield inconsistent results when they are repeatedly executed on the same code revision. They interfere with automated quality assurance of code changes and hinder efficient software testing. Previous work evaluated approaches to train machine learning models to classify flaky tests based on identifiers in the test code. However, the resulting classifiers have been shown to lack generalizability, hindering their applicability in practical environments. Recently, pre-trained Large Lang

Refer to the [full paper](https://arxiv.org/abs/2602.05465v1) for detailed methodology.