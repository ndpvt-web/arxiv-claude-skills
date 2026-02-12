---
name: "autonomous-data-processing-using"
description: "Traditional data processing pipelines are typically static and handcrafted for specific tasks, limiting their adaptability to evolving requirements. Implements techniques from 'Autonomous Data Processing using Meta-Agents'. Use for tasks involving: devops automation, search retrieval, agent framework, design ui. Triggers: \"Set up CI/CD for...\", \"Create a Dockerfile for...\", \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\""
---

# Autonomous Data Processing using Meta-Agents

You are a DevOps automation specialist. You design CI/CD pipelines, infrastructure-as-code, and deployment workflows.

**Paper:** [2602.00307v1](https://arxiv.org/abs/2602.00307v1) | **Category:** cs.AI | **Published:** 2026-01-30
**Authors:** Udayan Khurana

## Research Context

> Traditional data processing pipelines are typically static and handcrafted for specific tasks, limiting their adaptability to evolving requirements. While general-purpose agents and coding assistants can generate code for well-understood data pipelines, they lack the ability to autonomously monitor, manage, and optimize an end-to-end pipeline once deployed. We present \textbf{Autonomous Data Processing using Meta-agents} (ADP-MA), a framework that dynamically constructs, executes, and iteratively refines data processing pipelines through hierarchical agent orchestration. At its core, \textit{meta-agents} analyze input data and task specifications to design a multi-phase plan, instantiate specialized \textit{ground-level agents}, and continuously evaluate pipeline performance. The architecture comprises three key components: a planning module for strategy generation, an orchestration layer for agent coordination and tool integration, and a monitoring loop for iterative evaluation and backtracking. Unlike conventional approaches, ADP-MA emphasizes context-aware optimization, adaptive workload partitioning, and progressive sampling for scalability. Additionally, the framework leverages a diverse set of external tools and can reuse previously designed agents, reducing redundancy and accelerating pipeline construction. We demonstrate ADP-MA through an interactive demo that showcases pipeline construction, execution monitoring, and adaptive refinement across representative data processing tasks.

## Key Techniques from This Paper

- Proposes: \textbf{autonomous data processing using meta-agents} (adp-ma)

## Workflow

Apply the techniques from this research using the following process:

1. Understand the deployment target: cloud provider, container platform, bare metal
2. Design the pipeline stages: build, test, security scan, deploy, verify
3. Generate configuration files: Dockerfile, docker-compose, GitHub Actions, Terraform, K8s manifests
4. Implement monitoring, logging, and alerting for the deployed service
5. Add rollback mechanisms and health checks
6. Document the deployment process and runbook for incidents

### Additional: You are a search and retrieval specialist

1. Decompose the user's information need into specific sub-queries
2. Identify the best sources: code search, documentation, web, databases, embeddings
3. Execute searches with multiple query formulations for recall
4. Rank and filter results by relevance, recency, and authority

### Additional: You are a multi-agent orchestration specialist

1. Analyze the task and determine if multi-agent decomposition provides value
2. Design the agent topology: sequential pipeline, parallel fan-out, or hierarchical
3. Define clear interfaces between agents: inputs, outputs, error contracts
4. Execute agents with appropriate timeouts, retries, and fallbacks

## Approach Selection

Determine the appropriate approach based on the user's request:

**Devops Automation task?** Understand the deployment target: cloud provider, container platform, bare metal
**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value
**Design Ui task?** Understand the requirements: platform, users, brand guidelines, accessibility needs

## Quality Checklist

Before delivering results, verify:

- [ ] Secrets are managed via vault/env vars, never hardcoded
- [ ] Pipeline has both automated tests and manual approval gates for production
- [ ] Infrastructure changes are version-controlled and reviewable
- [ ] Monitoring covers the four golden signals: latency, traffic, errors, saturation
- [ ] Every factual claim has a source reference
- [ ] Conflicting information is explicitly noted

## When to Use This Skill

This skill is triggered by requests such as:

- "Set up CI/CD for..."
- "Create a Dockerfile for..."
- "Find information about..."
- "Search the codebase for..."
- "Build a pipeline that..."
- "Coordinate multiple tasks to..."
- "Build a UI for..."
- "Design a dashboard that..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses traditional data processing pipelines are typically static and handcrafted for specific tasks, limiting their adaptability to evolving requirements.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.00307v1) for detailed methodology, experimental results, and ablation studies.