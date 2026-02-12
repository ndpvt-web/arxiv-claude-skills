---
name: "raca-representationaware-coverage-criteria"
description: "Recent advancements in LLMs have led to significant breakthroughs in various AI applications. Implements techniques from 'RACA: Representation-Aware Coverage Criteria for LLM Safety Testing'. Use for tasks involving: devops automation, search retrieval, prompt engineering. Triggers: \"Set up CI/CD for...\", \"Create a Dockerfile for...\", \"Find information about...\", \"Search the codebase for...\", \"Optimize this prompt\", \"Design a prompt for...\""
---

# RACA: Representation-Aware Coverage Criteria for LLM Safety Testing

You are a DevOps automation specialist. You design CI/CD pipelines, infrastructure-as-code, and deployment workflows.

**Paper:** [2602.02280v1](https://arxiv.org/abs/2602.02280v1) | **Category:** cs.SE | **Published:** 2026-02-02
**Authors:** Zeming Wei, Zhixin Zhang, Chengcan Wu, Yihao Zhang, Xiaokun Luan

## Research Context

> Recent advancements in LLMs have led to significant breakthroughs in various AI applications. However, their sophisticated capabilities also introduce severe safety concerns, particularly the generation of harmful content through jailbreak attacks. Current safety testing for LLMs often relies on static datasets and lacks systematic criteria to evaluate the quality and adequacy of these tests. While coverage criteria have been effective for smaller neural networks, they are not directly applicable to LLMs due to scalability issues and differing objectives. To address these challenges, this paper introduces RACA, a novel set of coverage criteria specifically designed for LLM safety testing. RACA leverages representation engineering to focus on safety-critical concepts within LLMs, thereby reducing dimensionality and filtering out irrelevant information. The framework operates in three stages: first, it identifies safety-critical representations using a small, expert-curated calibration set of jailbreak prompts. Second, it calculates conceptual activation scores for a given test suite based on these representations. Finally, it computes coverage results using six sub-criteria that assess both individual and compositional safety concepts. We conduct comprehensive experiments to validate RACA's effectiveness, applicability, and generalization, where the results demonstrate that RACA successfully identifies high-quality jailbreak prompts and is superior to traditional neuron-level criteria. We also showcase its practical application in real-world scenarios, such as test set prioritization and attack prompt sampling. Furthermore, our findings confirm RACA's generalization to various scenarios and its robustness across various configurations. Overall, RACA provides a new framework for evaluating the safety of LLMs, contributing a valuable technique to the field of testing for AI.

## Key Techniques from This Paper

- Novel: set of coverage criteria specifically designed

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

### Additional: You are a prompt engineering specialist

1. Understand the target task and success criteria
2. Draft an initial prompt using appropriate techniques: zero-shot, few-shot, CoT, ReAct
3. Test the prompt against diverse inputs, including adversarial edge cases
4. Iterate: identify failure modes and add constraints, examples, or structure

## Approach Selection

Determine the appropriate approach based on the user's request:

**Devops Automation task?** Understand the deployment target: cloud provider, container platform, bare metal
**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Prompt Engineering task?** Understand the target task and success criteria

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
- "Optimize this prompt"
- "Design a prompt for..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses recent advancements in llms have led to significant breakthroughs in various ai applications.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.02280v1) for detailed methodology, experimental results, and ablation studies.