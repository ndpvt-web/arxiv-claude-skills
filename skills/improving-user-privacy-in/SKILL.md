---
name: "improving-user-privacy-in"
description: "Personalization is crucial for aligning Large Language Model (LLM) outputs with individual user preferences and background knowledge. Implements techniques from 'Improving User Privacy in Personalized Generation: Client-Side Retrieval-Augmented Modification of Server-Side Generated Speculations'. Use for tasks involving: devops automation, search retrieval, security. Triggers: \"Set up CI/CD for...\", \"Create a Dockerfile for...\", \"Find information about...\", \"Search the codebase for...\", \"Audit this code for security issues\", \"Is this configuration secure?\""
---

# Improving User Privacy in Personalized Generation: Client-Side Retrieval-Augmented Modification of Server-Side Generated Speculations

You are a DevOps automation specialist. You design CI/CD pipelines, infrastructure-as-code, and deployment workflows.

**Paper:** [2601.17569v1](https://arxiv.org/abs/2601.17569v1) | **Category:** cs.CL | **Published:** 2026-01-24
**Authors:** Alireza Salemi, Hamed Zamani

## Research Context

> Personalization is crucial for aligning Large Language Model (LLM) outputs with individual user preferences and background knowledge. State-of-the-art solutions are based on retrieval augmentation, where relevant context from a user profile is retrieved for LLM consumption. These methods deal with a trade-off between exposing retrieved private data to cloud providers and relying on less capable local models. We introduce $P^3$, an interactive framework for high-quality personalization without revealing private profiles to server-side LLMs. In $P^3$, a large server-side model generates a sequence of $k$ draft tokens based solely on the user query, while a small client-side model, with retrieval access to the user's private profile, evaluates and modifies these drafts to better reflect user preferences. This process repeats until an end token is generated. Experiments on LaMP-QA, a recent benchmark consisting of three personalized question answering datasets, show that $P^3$ consistently outperforms both non-personalized server-side and personalized client-side baselines, achieving statistically significant improvements of $7.4%$ to $9%$ on average. Importantly, $P^3$ recovers $90.3%$ to $95.7%$ of the utility of a ``leaky'' upper-bound scenario in which the full profile is exposed to the large server-side model. Privacy analyses, including linkability and attribute inference attacks, indicate that $P^3$ preserves the privacy of a non-personalized server-side model, introducing only marginal additional leakage ($1.5%$--$3.5%$) compared to submitting a query without any personal context. Additionally, the framework is efficient for edge deployment, with the client-side model generating only $9.2%$ of the total tokens. These results demonstrate that $P^3$ provides a practical, effective solution for personalized generation with improved privacy.

## Key Techniques from This Paper

- Achieves: both non-personalized server-side and personalized client-side baselines

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

### Additional: You are a security analysis specialist

1. Define the scope: code review, configuration audit, threat model, or penetration test
2. Scan for common vulnerability classes: injection, broken auth, misconfig, data exposure
3. Assess each finding: severity (CVSS), exploitability, business impact
4. Recommend specific mitigations with code examples

## Approach Selection

Determine the appropriate approach based on the user's request:

**Devops Automation task?** Understand the deployment target: cloud provider, container platform, bare metal
**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Security task?** Define the scope: code review, configuration audit, threat model, or penetration test

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
- "Audit this code for security issues"
- "Is this configuration secure?"

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses personalization is crucial for aligning large language model (llm) outputs with individual user preferences and background knowledge.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.17569v1) for detailed methodology, experimental results, and ablation studies.