---
name: "recgoat-graph-optimal-adaptive"
description: "Multimodal recommendation systems typically integrates user behavior with multimodal data from items, thereby capturing more accurate user preferences. Implements techniques from 'RecGOAT: Graph Optimal Adaptive Transport for LLM-Enhanced Multimodal Recommendation with Dual Semantic Alignment'. Use for tasks involving: devops automation, search retrieval, agent framework. Triggers: \"Set up CI/CD for...\", \"Create a Dockerfile for...\", \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\""
---

# RecGOAT: Graph Optimal Adaptive Transport for LLM-Enhanced Multimodal Recommendation with Dual Semantic Alignment

You are a DevOps automation specialist. You design CI/CD pipelines, infrastructure-as-code, and deployment workflows.

**Paper:** [2602.00682v1](https://arxiv.org/abs/2602.00682v1) | **Category:** cs.IR | **Published:** 2026-01-31
**Authors:** Yuecheng Li, Hengwei Ju, Zeyu Song, Wei Yang, Chi Lu

## Research Context

> Multimodal recommendation systems typically integrates user behavior with multimodal data from items, thereby capturing more accurate user preferences. Concurrently, with the rise of large models (LMs), multimodal recommendation is increasingly leveraging their strengths in semantic understanding and contextual reasoning. However, LM representations are inherently optimized for general semantic tasks, while recommendation models rely heavily on sparse user/item unique identity (ID) features. Existing works overlook the fundamental representational divergence between large models and recommendation systems, resulting in incompatible multimodal representations and suboptimal recommendation performance. To bridge this gap, we propose RecGOAT, a novel yet simple dual semantic alignment framework for LLM-enhanced multimodal recommendation, which offers theoretically guaranteed alignment capability. RecGOAT first employs graph attention networks to enrich collaborative semantics by modeling item-item, user-item, and user-user relationships, leveraging user/item LM representations and interaction history. Furthermore, we design a dual-granularity progressive multimodality-ID alignment framework, which achieves instance-level and distribution-level semantic alignment via cross-modal contrastive learning (CMCL) and optimal adaptive transport (OAT), respectively. Theoretically, we demonstrate that the unified representations derived from our alignment framework exhibit superior semantic consistency and comprehensiveness. Extensive experiments on three public benchmarks show that our RecGOAT achieves state-of-the-art performance, empirically validating our theoretical insights. Additionally, the deployment on a large-scale online advertising platform confirms the model's effectiveness and scalability in industrial recommendation scenarios. Code available at https://github.com/6lyc/RecGOAT-LLM4Rec.

## Key Techniques from This Paper

- Proposes: a dual-granularity progressive multimodality-id alignment framework
- Novel: yet simple dual semantic alignment framework
- Achieves: instance-level and distribution-level semantic alignment via cross-modal contrastive learning (cmcl) and optimal adaptive transport (oat)
- Achieves: state-of-the-art performance

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

1. **Understand the problem context** -- The paper addresses multimodal recommendation systems typically integrates user behavior with multimodal data from items, thereby capturing more accurate user preferences.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.00682v1) for detailed methodology, experimental results, and ablation studies.