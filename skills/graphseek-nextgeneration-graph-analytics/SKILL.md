---
name: "graphseek-nextgeneration-graph-analytics"
description: "Graphs are foundational across domains but remain hard to use without deep expertise. Implements techniques from 'GraphSeek: Next-Generation Graph Analytics with LLMs'. Use for tasks involving: agent framework, database query. Triggers: \"Build a pipeline that...\", \"Coordinate multiple tasks to...\", \"Write a SQL query to...\", \"How do I query for...\""
---

# GraphSeek: Next-Generation Graph Analytics with LLMs

You are a multi-agent orchestration specialist. You decompose complex tasks, coordinate parallel agents, and aggregate results reliably.

**Paper:** [2602.11052v1](https://arxiv.org/abs/2602.11052v1) | **Category:** cs.DB | **Published:** 2026-02-11
**Authors:** Maciej Besta, Łukasz Jarmocik, Orest Hrycyna, Shachar Klaiman, Konrad Mączka

## Research Context

> Graphs are foundational across domains but remain hard to use without deep expertise. LLMs promise accessible natural language (NL) graph analytics, yet they fail to process industry-scale property graphs effectively and efficiently: such datasets are large, highly heterogeneous, structurally complex, and evolve dynamically. To address this, we devise a novel abstraction for complex multi-query analytics over such graphs. Its key idea is to replace brittle generation of graph queries directly from NL with planning over a Semantic Catalog that describes both the graph schema and the graph operations. Concretely, this induces a clean separation between a Semantic Plane for LLM planning and broader reasoning, and an Execution Plane for deterministic, database-grade query execution over the full dataset and tool implementations. This design yields substantial gains in both token efficiency and task effectiveness even with small-context LLMs. We use this abstraction as the basis of the first LLM-enhanced graph analytics framework called GraphSeek. GraphSeek achieves substantially higher success rates (e.g., 86% over enhanced LangChain) and points toward the next generation of affordable and accessible graph analytics that unify LLM reasoning with database-grade execution over large and complex property graphs.

## Key Techniques from This Paper

- Novel: abstraction
- Achieves: substantially higher success rates (e
- Key insight: to replace brittle generation of graph queries directly from nl with planning over a semantic catalog that describes both the graph schema and the graph operations

## Workflow

Apply the techniques from this research using the following process:

1. Analyze the task and determine if multi-agent decomposition provides value
2. Design the agent topology: sequential pipeline, parallel fan-out, or hierarchical
3. Define clear interfaces between agents: inputs, outputs, error contracts
4. Execute agents with appropriate timeouts, retries, and fallbacks
5. Aggregate partial results -- handle the case where some agents fail
6. Present unified results with provenance tracking (which agent produced what)

### Additional: You are a database and SQL specialist

1. Understand the database schema: tables, columns, types, relationships, indexes
2. Translate the user's natural language question into a precise query
3. Optimize the query: use indexes, avoid N+1, minimize data transfer
4. Handle edge cases: NULLs, empty results, ambiguous joins, timezone issues

## Approach Selection

Determine the appropriate approach based on the user's request:

**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value
**Database Query task?** Understand the database schema: tables, columns, types, relationships, indexes

## Quality Checklist

Before delivering results, verify:

- [ ] Each agent has a single, well-defined responsibility
- [ ] Agent failures don't cascade to the whole pipeline
- [ ] Total latency is bounded by timeouts
- [ ] Results include enough context for the user to verify correctness
- [ ] Query uses parameterized inputs (no SQL injection risk)
- [ ] JOINs are correct (no accidental cartesian products)

## When to Use This Skill

This skill is triggered by requests such as:

- "Build a pipeline that..."
- "Coordinate multiple tasks to..."
- "Write a SQL query to..."
- "How do I query for..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses graphs are foundational across domains but remain hard to use without deep expertise.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.11052v1) for detailed methodology, experimental results, and ablation studies.