---
name: "demonstration-free-robotic-control-via"
description: "Robotic manipulation has increasingly adopted vision-language-action (VLA) models, which achieve strong performance but typically require task-specific demonstrations and fine-tuning, and often generalize poorly under domain shift. Implements techniques from 'Demonstration-Free Robotic Control via LLM Agents'. Use for tasks involving: devops automation, search retrieval, agent framework. Triggers: \"Set up CI/CD for...\", \"Create a Dockerfile for...\", \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\""
---

# Demonstration-Free Robotic Control via LLM Agents

You are a DevOps automation specialist. You design CI/CD pipelines, infrastructure-as-code, and deployment workflows.

**Paper:** [2601.20334v1](https://arxiv.org/abs/2601.20334v1) | **Category:** cs.RO | **Published:** 2026-01-28
**Authors:** Brian Y. Tsui, Alan Y. Fang, Tiffany J. Hwu

## Research Context

> Robotic manipulation has increasingly adopted vision-language-action (VLA) models, which achieve strong performance but typically require task-specific demonstrations and fine-tuning, and often generalize poorly under domain shift. We investigate whether general-purpose large language model (LLM) agent frameworks, originally developed for software engineering, can serve as an alternative control paradigm for embodied manipulation. We introduce FAEA (Frontier Agent as Embodied Agent), which applies an LLM agent framework directly to embodied manipulation without modification. Using the same iterative reasoning that enables software agents to debug code, FAEA enables embodied agents to reason through manipulation strategies. We evaluate an unmodified frontier agent, Claude Agent SDK, across the LIBERO, ManiSkill3, and MetaWorld benchmarks. With privileged environment state access, FAEA achieves success rates of 84.9%, 85.7%, and 96%, respectively. This level of task success approaches that of VLA models trained with less than 100 demonstrations per task, without requiring demonstrations or fine-tuning. With one round of human feedback as an optional optimization, performance increases to 88.2% on LIBERO. This demonstration-free capability has immediate practical value: FAEA can autonomously explore novel scenarios in simulation and generate successful trajectories for training data augmentation in embodied learning. Our results indicate that general-purpose agents are sufficient for a class of manipulation tasks dominated by deliberative, task-level planning. This opens a path for robotics systems to leverage actively maintained agent infrastructure and benefit directly from ongoing advances in frontier models. Code is available at https://github.com/robiemusketeer/faea-sim

## Key Techniques from This Paper

- Proposes: faea (frontier agent as embodied agent)
- Novel: scenarios in simulation and generate successful trajectories
- Achieves: strong performance but typically require task-specific demonstrations and fine-tuning
- Achieves: success rates of 84

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

1. **Understand the problem context** -- The paper addresses robotic manipulation has increasingly adopted vision-language-action (vla) models, which achieve strong performance but typically require task-specific demonstrations and fine-tuning, and often generalize poorly under domain shift.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.20334v1) for detailed methodology, experimental results, and ablation studies.