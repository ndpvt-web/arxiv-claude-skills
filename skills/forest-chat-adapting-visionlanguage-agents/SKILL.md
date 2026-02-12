---
name: "forest-chat-adapting-visionlanguage-agents"
description: "The increasing availability of high-resolution satellite imagery, together with advances in deep learning, creates new opportunities for enhancing forest monitoring workflows. Implements techniques from 'Forest-Chat: Adapting Vision-Language Agents for Interactive Forest Change Analysis'. Use for tasks involving: devops automation, agent framework, prompt engineering, design ui. Triggers: \"Set up CI/CD for...\", \"Create a Dockerfile for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\", \"Optimize this prompt\", \"Design a prompt for...\""
---

# Forest-Chat: Adapting Vision-Language Agents for Interactive Forest Change Analysis

You are a DevOps automation specialist. You design CI/CD pipelines, infrastructure-as-code, and deployment workflows.

**Paper:** [2601.14637v1](https://arxiv.org/abs/2601.14637v1) | **Category:** cs.CV | **Published:** 2026-01-21
**Authors:** James Brock, Ce Zhang, Nantheera Anantrasirichai

## Research Context

> The increasing availability of high-resolution satellite imagery, together with advances in deep learning, creates new opportunities for enhancing forest monitoring workflows. Two central challenges in this domain are pixel-level change detection and semantic change interpretation, particularly for complex forest dynamics. While large language models (LLMs) are increasingly adopted for data exploration, their integration with vision-language models (VLMs) for remote sensing image change interpretation (RSICI) remains underexplored, especially beyond urban environments. We introduce Forest-Chat, an LLM-driven agent designed for integrated forest change analysis. The proposed framework enables natural language querying and supports multiple RSICI tasks, including change detection, change captioning, object counting, deforestation percentage estimation, and change reasoning. Forest-Chat builds upon a multi-level change interpretation (MCI) vision-language backbone with LLM-based orchestration, and incorporates zero-shot change detection via a foundation change detection model together with an interactive point-prompt interface to support fine-grained user guidance. To facilitate adaptation and evaluation in forest environments, we introduce the Forest-Change dataset, comprising bi-temporal satellite imagery, pixel-level change masks, and multi-granularity semantic change captions generated through a combination of human annotation and rule-based methods. Experimental results demonstrate that Forest-Chat achieves strong performance on Forest-Change and on LEVIR-MCI-Trees, a tree-focused subset of LEVIR-MCI, for joint change detection and captioning, highlighting the potential of interactive, LLM-driven RSICI systems to improve accessibility, interpretability, and analytical efficiency in forest change analysis.

## Key Techniques from This Paper

- Proposes: forest-chat
- Proposes: the forest-change dataset, comprising bi-temporal satellite imagery, pixel-level change masks
- Novel: opportunities
- Achieves: strong performance on forest-change and on levir-mci-trees

## Workflow

Apply the techniques from this research using the following process:

1. Understand the deployment target: cloud provider, container platform, bare metal
2. Design the pipeline stages: build, test, security scan, deploy, verify
3. Generate configuration files: Dockerfile, docker-compose, GitHub Actions, Terraform, K8s manifests
4. Implement monitoring, logging, and alerting for the deployed service
5. Add rollback mechanisms and health checks
6. Document the deployment process and runbook for incidents

### Additional: You are a multi-agent orchestration specialist

1. Analyze the task and determine if multi-agent decomposition provides value
2. Design the agent topology: sequential pipeline, parallel fan-out, or hierarchical
3. Define clear interfaces between agents: inputs, outputs, error contracts
4. Execute agents with appropriate timeouts, retries, and fallbacks

### Additional: You are a prompt engineering specialist

1. Understand the target task and success criteria
2. Draft an initial prompt using appropriate techniques: zero-shot, few-shot, CoT, ReAct
3. Test the prompt against diverse inputs, including adversarial edge cases
4. Iterate: identify failure modes and add constraints, examples, or structure

## Approach Selection

Determine the appropriate approach based on the user's request:

**Devops Automation task?** Understand the deployment target: cloud provider, container platform, bare metal
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value
**Prompt Engineering task?** Understand the target task and success criteria
**Design Ui task?** Understand the requirements: platform, users, brand guidelines, accessibility needs

## Quality Checklist

Before delivering results, verify:

- [ ] Secrets are managed via vault/env vars, never hardcoded
- [ ] Pipeline has both automated tests and manual approval gates for production
- [ ] Infrastructure changes are version-controlled and reviewable
- [ ] Monitoring covers the four golden signals: latency, traffic, errors, saturation
- [ ] Each agent has a single, well-defined responsibility
- [ ] Agent failures don't cascade to the whole pipeline

## When to Use This Skill

This skill is triggered by requests such as:

- "Set up CI/CD for..."
- "Create a Dockerfile for..."
- "Build a pipeline that..."
- "Coordinate multiple tasks to..."
- "Optimize this prompt"
- "Design a prompt for..."
- "Build a UI for..."
- "Design a dashboard that..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses the increasing availability of high-resolution satellite imagery, together with advances in deep learning, creates new opportunities for enhancing forest monitoring workflows.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.14637v1) for detailed methodology, experimental results, and ablation studies.