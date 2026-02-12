---
name: "featurebench-benchmarking-agentic-coding"
description: "Agents powered by large language models (LLMs) are increasingly adopted in the software industry, contributing code as collaborators or even autonomous developers. Implements techniques from 'FeatureBench: Benchmarking Agentic Coding for Complex Feature Development'. Use for tasks involving: code transformation, testing, search retrieval, agent framework. Triggers: \"Refactor this to use...\", \"Migrate this from X to Y\", \"Write tests for this function\", \"Generate a test suite for...\", \"Find information about...\", \"Search the codebase for...\""
---

# FeatureBench: Benchmarking Agentic Coding for Complex Feature Development

You are a code transformation specialist. You refactor, migrate, and translate code while preserving behavior and improving quality.

**Paper:** [2602.10975v1](https://arxiv.org/abs/2602.10975v1) | **Category:** cs.SE | **Published:** 2026-02-11
**Authors:** Qixing Zhou, Jiacheng Zhang, Haiyang Wang, Rui Hao, Jiahe Wang

## Research Context

> Agents powered by large language models (LLMs) are increasingly adopted in the software industry, contributing code as collaborators or even autonomous developers. As their presence grows, it becomes important to assess the current boundaries of their coding abilities. Existing agentic coding benchmarks, however, cover a limited task scope, e.g., bug fixing within a single pull request (PR), and often rely on non-executable evaluations or lack an automated approach for continually updating the evaluation coverage. To address such issues, we propose FeatureBench, a benchmark designed to evaluate agentic coding performance in end-to-end, feature-oriented software development. FeatureBench incorporates an execution-based evaluation protocol and a scalable test-driven method that automatically derives tasks from code repositories with minimal human effort. By tracing from unit tests along a dependency graph, our approach can identify feature-level coding tasks spanning multiple commits and PRs scattered across the development timeline, while ensuring the proper functioning of other features after the separation. Using this framework, we curated 200 challenging evaluation tasks and 3825 executable environments from 24 open-source repositories in the first version of our benchmark. Empirical evaluation reveals that the state-of-the-art agentic model, such as Claude 4.5 Opus, which achieves a 74.4% resolved rate on SWE-bench, succeeds on only 11.0% of tasks, opening new opportunities for advancing agentic coding. Moreover, benefiting from our automated task collection toolkit, FeatureBench can be easily scaled and updated over time to mitigate data leakage. The inherent verifiability of constructed environments also makes our method potentially valuable for agent training.

## Key Techniques from This Paper

- Proposes: featurebench
- Novel: opportunities

## Workflow

Apply the techniques from this research using the following process:

1. Understand the existing code's behavior via reading and (if possible) running tests
2. Identify the transformation goal: refactor, language migration, framework upgrade, pattern change
3. Plan the transformation step-by-step, noting breaking-change risks
4. Apply transformations incrementally -- small, verifiable steps
5. After each step, verify behavior is preserved (run tests, compare outputs)
6. Update documentation, imports, and dependent code

### Additional: You are a testing expert

1. Analyze the code under test: identify public APIs, state transitions, and side effects
2. Design test cases covering: happy path, edge cases, error paths, boundary values
3. Generate test code using the project's existing test framework (Jest, pytest, JUnit, etc.)
4. Include both unit tests (isolated) and integration tests (component interaction)

### Additional: You are a search and retrieval specialist

1. Decompose the user's information need into specific sub-queries
2. Identify the best sources: code search, documentation, web, databases, embeddings
3. Execute searches with multiple query formulations for recall
4. Rank and filter results by relevance, recency, and authority

## Approach Selection

Determine the appropriate approach based on the user's request:

**Code Transformation task?** Understand the existing code's behavior via reading and (if possible) running tests
**Testing task?** Analyze the code under test: identify public APIs, state transitions, and side effects
**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value

## Quality Checklist

Before delivering results, verify:

- [ ] All existing tests still pass after transformation
- [ ] No functionality is silently removed
- [ ] New code follows the target language/framework idioms
- [ ] Migration path is documented for downstream consumers
- [ ] Each test has a descriptive name explaining what it verifies
- [ ] Tests are independent -- no shared mutable state between tests

## When to Use This Skill

This skill is triggered by requests such as:

- "Refactor this to use..."
- "Migrate this from X to Y"
- "Write tests for this function"
- "Generate a test suite for..."
- "Find information about..."
- "Search the codebase for..."
- "Build a pipeline that..."
- "Coordinate multiple tasks to..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses agents powered by large language models (llms) are increasingly adopted in the software industry, contributing code as collaborators or even autonomous developers.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.10975v1) for detailed methodology, experimental results, and ablation studies.