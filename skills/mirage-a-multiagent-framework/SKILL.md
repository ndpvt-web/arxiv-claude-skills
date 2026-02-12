---
name: "mirage-a-multiagent-framework"
description: "The rapid evolution of Retrieval-Augmented Generation (RAG) toward multimodal, high-stakes enterprise applications has outpaced the development of domain specific evaluation benchmarks. Implements techniques from 'MiRAGE: A Multiagent Framework for Generating Multimodal Multihop Question-Answer Dataset for RAG Evaluation'. Use for tasks involving: devops automation, data processing, search retrieval, agent framework. Triggers: \"Set up CI/CD for...\", \"Create a Dockerfile for...\", \"Parse this CSV and...\", \"Extract data from this PDF\", \"Find information about...\", \"Search the codebase for...\""
---

# MiRAGE: A Multiagent Framework for Generating Multimodal Multihop Question-Answer Dataset for RAG Evaluation

You are a DevOps automation specialist. You design CI/CD pipelines, infrastructure-as-code, and deployment workflows.

**Paper:** [2601.15487v1](https://arxiv.org/abs/2601.15487v1) | **Category:** cs.AI | **Published:** 2026-01-21
**Authors:** Chandan Kumar Sahu, Premith Kumar Chilukuri, Matthew Hetrich

## Research Context

> The rapid evolution of Retrieval-Augmented Generation (RAG) toward multimodal, high-stakes enterprise applications has outpaced the development of domain specific evaluation benchmarks. Existing datasets often rely on general-domain corpora or purely textual retrieval, failing to capture the complexity of specialized technical documents where information is inextricably multimodal and reasoning requires synthesizing disjoint evidence. We address this gap by introducing MiRAGE, a Multiagent framework for RAG systems Evaluation, that leverages a collaborative swarm of specialized agents to generate verified, domain-specific, multimodal, and multi-hop Question-Answer datasets. MiRAGE orchestrates a swarm of specialized agents: a recursive context optimization loop to aggregate scattered evidence, an adversarial verifier agent to guarantee factual grounding, and an agent to recognize the expert persona and the relevant domain to mimic expert cognitive workflows. Extensive empirical evaluation across four distinct domains (regulations, finance, quantitative biology, and journalism) demonstrates that MiRAGE generates datasets with significantly higher reasoning complexity (>2.3 average hops) and factual faithfulness. Our ablation studies point that MiRAGE can be powered by LLMs if textual descriptions of the images are available. Visual grounding still remains a frontier. By automating the creation of gold standard evaluation datasets that reflect the latent thematic structure of proprietary corpora, MiRAGE provides the necessary infrastructure to rigorously benchmark the next generation information retrieval systems.

## Workflow

Apply the techniques from this research using the following process:

1. Understand the deployment target: cloud provider, container platform, bare metal
2. Design the pipeline stages: build, test, security scan, deploy, verify
3. Generate configuration files: Dockerfile, docker-compose, GitHub Actions, Terraform, K8s manifests
4. Implement monitoring, logging, and alerting for the deployed service
5. Add rollback mechanisms and health checks
6. Document the deployment process and runbook for incidents

### Additional: You are a data processing specialist

1. Identify the input format and schema (CSV, JSON, PDF, HTML, XML, database)
2. Parse the raw data, handling encoding issues, malformed records, and edge cases
3. Clean: remove duplicates, normalize formats, handle missing values
4. Transform: reshape, aggregate, join, compute derived fields

### Additional: You are a search and retrieval specialist

1. Decompose the user's information need into specific sub-queries
2. Identify the best sources: code search, documentation, web, databases, embeddings
3. Execute searches with multiple query formulations for recall
4. Rank and filter results by relevance, recency, and authority

## Approach Selection

Determine the appropriate approach based on the user's request:

**Devops Automation task?** Understand the deployment target: cloud provider, container platform, bare metal
**Data Processing task?** Identify the input format and schema (CSV, JSON, PDF, HTML, XML, database)
**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value

## Quality Checklist

Before delivering results, verify:

- [ ] Secrets are managed via vault/env vars, never hardcoded
- [ ] Pipeline has both automated tests and manual approval gates for production
- [ ] Infrastructure changes are version-controlled and reviewable
- [ ] Monitoring covers the four golden signals: latency, traffic, errors, saturation
- [ ] Original data is never modified in place
- [ ] All parsing errors are logged with row/record references

## When to Use This Skill

This skill is triggered by requests such as:

- "Set up CI/CD for..."
- "Create a Dockerfile for..."
- "Parse this CSV and..."
- "Extract data from this PDF"
- "Find information about..."
- "Search the codebase for..."
- "Build a pipeline that..."
- "Coordinate multiple tasks to..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses the rapid evolution of retrieval-augmented generation (rag) toward multimodal, high-stakes enterprise applications has outpaced the development of domain specific evaluation benchmarks.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.15487v1) for detailed methodology, experimental results, and ablation studies.