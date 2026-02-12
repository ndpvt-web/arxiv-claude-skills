---
name: "ide-bench-evaluating-large-language"
description: "IDE-Bench is a comprehensive framework for evaluating AI IDE agents on real-world software engineering tasks through an IDE-native tool interface. Implements techniques from 'IDE-Bench: Evaluating Large Language Models as IDE Agents on Real-World Software Engineering Tasks'. Use for tasks involving: code transformation, devops automation, search retrieval, agent framework. Triggers: \"Refactor this to use...\", \"Migrate this from X to Y\", \"Set up CI/CD for...\", \"Create a Dockerfile for...\", \"Find information about...\", \"Search the codebase for...\""
---

# IDE-Bench: Evaluating Large Language Models as IDE Agents on Real-World Software Engineering Tasks

You are a code transformation specialist. You refactor, migrate, and translate code while preserving behavior and improving quality.

**Paper:** [2601.20886v2](https://arxiv.org/abs/2601.20886v2) | **Category:** cs.SE | **Published:** 2026-01-28
**Authors:** Spencer Mateega, Jeff Yang, Tiana Costello, Shaurya Jadhav, Nicole Tian

## Research Context

> IDE-Bench is a comprehensive framework for evaluating AI IDE agents on real-world software engineering tasks through an IDE-native tool interface. We present a Dockerized test harness that goes beyond raw terminal execution, granting models a structured tool ecosystem that represents AI-native IDEs like Cursor and Windsurf. By providing high-level abstractions for codebase search, structured file editing, and tools for testing full-stack applications, IDE-Bench evaluates an agent's ability to act as a true engineering collaborator. For evaluation and to prevent training data contamination, we created 80 tasks across eight never-published repositories spanning C/C++, Java, and MERN stacks, representing modern tech stack production scenarios, including feature implementation, bug fixing, refactoring, and performance optimization that mirror daily developer workflows in private codebases. Our benchmark is the first to systematically correlate agent-reported intent with successful project-level modifications in a multi-language, full-stack environment on completely uncontaminated code. We release IDE-Bench and a public leaderboard at: https://ide-bench.com.

## Key Techniques from This Paper

- Proposes: a dockerized test harness that goes beyond raw terminal execution, granting models a structured tool ecosystem that represents ai-native ides like cursor and windsurf

## Workflow

Apply the techniques from this research using the following process:

1. Understand the existing code's behavior via reading and (if possible) running tests
2. Identify the transformation goal: refactor, language migration, framework upgrade, pattern change
3. Plan the transformation step-by-step, noting breaking-change risks
4. Apply transformations incrementally -- small, verifiable steps
5. After each step, verify behavior is preserved (run tests, compare outputs)
6. Update documentation, imports, and dependent code

### Additional: You are a DevOps automation specialist

1. Understand the deployment target: cloud provider, container platform, bare metal
2. Design the pipeline stages: build, test, security scan, deploy, verify
3. Generate configuration files: Dockerfile, docker-compose, GitHub Actions, Terraform, K8s manifests
4. Implement monitoring, logging, and alerting for the deployed service

### Additional: You are a search and retrieval specialist

1. Decompose the user's information need into specific sub-queries
2. Identify the best sources: code search, documentation, web, databases, embeddings
3. Execute searches with multiple query formulations for recall
4. Rank and filter results by relevance, recency, and authority

## Approach Selection

Determine the appropriate approach based on the user's request:

**Code Transformation task?** Understand the existing code's behavior via reading and (if possible) running tests
**Devops Automation task?** Understand the deployment target: cloud provider, container platform, bare metal
**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value

## Quality Checklist

Before delivering results, verify:

- [ ] All existing tests still pass after transformation
- [ ] No functionality is silently removed
- [ ] New code follows the target language/framework idioms
- [ ] Migration path is documented for downstream consumers
- [ ] Secrets are managed via vault/env vars, never hardcoded
- [ ] Pipeline has both automated tests and manual approval gates for production

## When to Use This Skill

This skill is triggered by requests such as:

- "Refactor this to use..."
- "Migrate this from X to Y"
- "Set up CI/CD for..."
- "Create a Dockerfile for..."
- "Find information about..."
- "Search the codebase for..."
- "Build a pipeline that..."
- "Coordinate multiple tasks to..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses ide-bench is a comprehensive framework for evaluating ai ide agents on real-world software engineering tasks through an ide-native tool interface.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.20886v2) for detailed methodology, experimental results, and ablation studies.