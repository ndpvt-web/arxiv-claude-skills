---
name: "query-efficient-agentic-graph-extraction"
description: "Graph-based retrieval-augmented generation (GraphRAG) systems construct knowledge graphs over document collections to support multi-hop reasoning. Implements techniques from the paper 'Query-Efficient Agentic Graph Extraction Attacks on GraphRAG Systems' for extract, transform, and process data. Use when tasks involve (data processing), (search & retrieval), (agent framework), (security) or when the user references techniques from this research area."
---

# Query-Efficient Agentic Graph Extraction Attacks on GraphRAG Systems

**Source:** [https://arxiv.org/abs/2601.14662v1](https://arxiv.org/abs/2601.14662v1)
**Category:** cs.AI | **Published:** 2026-01-21 | **Skill Score:** 68
**Authors:** Shuhua Yang, Jiahao Zhang, Yilong Wang...

## Core Capability

Extract, transform, and process data.

## Key Techniques

- **Retrieval-augmented** approach for grounding responses in external knowledge

## Workflow

1. Identify the data source and format
2. Parse and extract relevant data fields
3. Transform data into the desired output format
4. Validate data integrity and handle errors
5. Output processed data in the requested format

## Security Considerations

- Check for OWASP Top 10 vulnerabilities
- Validate all user inputs at system boundaries
- Use parameterized queries for database access
- Follow the principle of least privilege

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> Graph-based retrieval-augmented generation (GraphRAG) systems construct knowledge graphs over document collections to support multi-hop reasoning. While prior work shows that GraphRAG responses may leak retrieved subgraphs, the feasibility of query-efficient reconstruction of the hidden graph structure remains unexplored under realistic query budgets. We study a budget-constrained black-box setting where an adversary adaptively queries the system to steal its latent entity-relation graph. We pro

Refer to the [full paper](https://arxiv.org/abs/2601.14662v1) for detailed methodology.