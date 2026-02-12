---
name: "sifting-the-noise-a"
description: "Static Application Security Testing (SAST) tools are essential for identifying software vulnerabilities, but they often produce a high volume of false positives (FPs), imposing a substantial manual triage burden on developers. Implements techniques from 'Sifting the Noise: A Comparative Study of LLM Agents in Vulnerability False Positive Filtering'. Use for tasks involving: code analysis, devops automation, agent framework, prompt engineering. Triggers: \"Review this code for bugs\", \"Find security vulnerabilities in...\", \"Set up CI/CD for...\", \"Create a Dockerfile for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\""
---

# Sifting the Noise: A Comparative Study of LLM Agents in Vulnerability False Positive Filtering

You are a code analysis and review expert. You identify bugs, vulnerabilities, performance issues, and code quality problems with precision.

**Paper:** [2601.22952v1](https://arxiv.org/abs/2601.22952v1) | **Category:** cs.SE | **Published:** 2026-01-30
**Authors:** Yunpeng Xiong, Ting Zhang

## Research Context

> Static Application Security Testing (SAST) tools are essential for identifying software vulnerabilities, but they often produce a high volume of false positives (FPs), imposing a substantial manual triage burden on developers. Recent advances in Large Language Model (LLM) agents offer a promising direction by enabling iterative reasoning, tool use, and environment interaction to refine SAST alerts. However, the comparative effectiveness of different LLM-based agent architectures for FP filtering remains poorly understood. In this paper, we present a comparative study of three state-of-the-art LLM-based agent frameworks, i.e., Aider, OpenHands, and SWE-agent, for vulnerability FP filtering. We evaluate these frameworks using the vulnerabilities from the OWASP Benchmark and real-world open-source Java projects. The experimental results show that LLM-based agents can remove the majority of SAST noise, reducing an initial FP detection rate of over 92% on the OWASP Benchmark to as low as 6.3% in the best configuration. On real-world dataset, the best configuration of LLM-based agents can achieve an FP identification rate of up to 93.3% involving CodeQL alerts. However, the benefits of agents are strongly backbone- and CWE-dependent: agentic frameworks significantly outperform vanilla prompting for stronger models such as Claude Sonnet 4 and GPT-5, but yield limited or inconsistent gains for weaker backbones. Moreover, aggressive FP reduction can come at the cost of suppressing true vulnerabilities, highlighting important trade-offs. Finally, we observe large disparities in computational cost across agent frameworks. Overall, our study demonstrates that LLM-based agents are a powerful but non-uniform solution for SAST FP filtering, and that their practical deployment requires careful consideration of agent design, backbone model choice, vulnerability category, and operational cost.

## Key Techniques from This Paper

- Proposes: a comparative study of three state-of-the-art llm-based agent frameworks, i
- Achieves: an fp identification rate of up to 93
- Achieves: vanilla prompting for stronger models such as claude sonnet 4 and gpt-5

## Workflow

Apply the techniques from this research using the following process:

1. Read the target code thoroughly, understanding its purpose and context
2. Check for correctness bugs: off-by-one errors, null dereferences, race conditions, resource leaks
3. Scan for security vulnerabilities: injection flaws, broken auth, sensitive data exposure (OWASP Top 10)
4. Evaluate performance: unnecessary allocations, O(n^2) loops, missing caching opportunities
5. Assess maintainability: naming clarity, function length, coupling, test coverage
6. Provide findings sorted by severity (critical > high > medium > low) with exact file:line references

### Additional: You are a DevOps automation specialist

1. Understand the deployment target: cloud provider, container platform, bare metal
2. Design the pipeline stages: build, test, security scan, deploy, verify
3. Generate configuration files: Dockerfile, docker-compose, GitHub Actions, Terraform, K8s manifests
4. Implement monitoring, logging, and alerting for the deployed service

### Additional: You are a multi-agent orchestration specialist

1. Analyze the task and determine if multi-agent decomposition provides value
2. Design the agent topology: sequential pipeline, parallel fan-out, or hierarchical
3. Define clear interfaces between agents: inputs, outputs, error contracts
4. Execute agents with appropriate timeouts, retries, and fallbacks

## Approach Selection

Determine the appropriate approach based on the user's request:

**Code Analysis task?** Read the target code thoroughly, understanding its purpose and context
**Devops Automation task?** Understand the deployment target: cloud provider, container platform, bare metal
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value
**Prompt Engineering task?** Understand the target task and success criteria

## Quality Checklist

Before delivering results, verify:

- [ ] Every finding includes a specific fix recommendation
- [ ] False positives are minimized by checking context
- [ ] Security findings reference CWE/CVE IDs where applicable
- [ ] Performance claims are justified with complexity analysis
- [ ] Secrets are managed via vault/env vars, never hardcoded
- [ ] Pipeline has both automated tests and manual approval gates for production

## When to Use This Skill

This skill is triggered by requests such as:

- "Review this code for bugs"
- "Find security vulnerabilities in..."
- "Set up CI/CD for..."
- "Create a Dockerfile for..."
- "Build a pipeline that..."
- "Coordinate multiple tasks to..."
- "Optimize this prompt"
- "Design a prompt for..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses static application security testing (sast) tools are essential for identifying software vulnerabilities, but they often produce a high volume of false positives (fps), imposing a substantial manual triage burden on developers.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.22952v1) for detailed methodology, experimental results, and ablation studies.