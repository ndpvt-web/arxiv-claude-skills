---
name: "experience-driven-multiagent-systems-are"
description: "Recent advances have enabled large language model (LLM) agents to solve complex tasks by orchestrating external tools. Implements techniques from 'Experience-Driven Multi-Agent Systems Are Training-free Context-aware Earth Observers'. Use for tasks involving: devops automation, search retrieval, agent framework. Triggers: \"Set up CI/CD for...\", \"Create a Dockerfile for...\", \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\""
---

# Experience-Driven Multi-Agent Systems Are Training-free Context-aware Earth Observers

You are a DevOps automation specialist. You design CI/CD pipelines, infrastructure-as-code, and deployment workflows.

**Paper:** [2602.02559v1](https://arxiv.org/abs/2602.02559v1) | **Category:** cs.AI | **Published:** 2026-01-30
**Authors:** Pengyu Dai, Weihao Xuan, Junjue Wang, Hongruixuan Chen, Jian Song

## Research Context

> Recent advances have enabled large language model (LLM) agents to solve complex tasks by orchestrating external tools. However, these agents often struggle in specialized, tool-intensive domains that demand long-horizon execution, tight coordination across modalities, and strict adherence to implicit tool constraints. Earth Observation (EO) tasks exemplify this challenge due to the multi-modal and multi-temporal data inputs, as well as the requirements of geo-knowledge constraints (spectrum library, spatial reasoning, etc): many high-level plans can be derailed by subtle execution errors that propagate through a pipeline and invalidate final results. A core difficulty is that existing agents lack a mechanism to learn fine-grained, tool-level expertise from interaction. Without such expertise, they cannot reliably configure tool parameters or recover from mid-execution failures, limiting their effectiveness in complex EO workflows. To address this, we introduce \textbf{GeoEvolver}, a self-evolving multi-agent system~(MAS) that enables LLM agents to acquire EO expertise through structured interaction without any parameter updates. GeoEvolver decomposes each query into independent sub-goals via a retrieval-augmented multi-agent orchestrator, then explores diverse tool-parameter configurations at the sub-goal level. Successful patterns and root-cause attribution from failures are then distilled in an evolving memory bank that provides in-context demonstrations for future queries. Experiments on three tool-integrated EO benchmarks show that GeoEvolver consistently improves end-to-end task success, with an average gain of 12\% across multiple LLM backbones, demonstrating that EO expertise can emerge progressively from efficient, fine-grained interactions with the environment.

## Key Techniques from This Paper

- Proposes: \textbf{geoevolver}

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

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses recent advances have enabled large language model (llm) agents to solve complex tasks by orchestrating external tools.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.02559v1) for detailed methodology, experimental results, and ablation studies.