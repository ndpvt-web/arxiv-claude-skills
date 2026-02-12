---
name: "beyond-blame-rethinking-szz"
description: "Identifying Bug-Inducing Commits (BICs) is fundamental for understanding software defects and enabling downstream tasks such as defect prediction and automated program repair. Implements techniques from the paper 'Beyond Blame: Rethinking SZZ with Knowledge Graph Search' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework), (security), (design & ui) or when the user references techniques from this research area."
---

# Beyond Blame: Rethinking SZZ with Knowledge Graph Search

**Source:** [https://arxiv.org/abs/2602.02934v1](https://arxiv.org/abs/2602.02934v1)
**Category:** cs.SE | **Published:** 2026-02-03 | **Skill Score:** 67
**Authors:** Yu Shi, Hao Li, Bram Adams...

## Core Capability

Search, retrieve, and synthesize information.

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

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

> Identifying Bug-Inducing Commits (BICs) is fundamental for understanding software defects and enabling downstream tasks such as defect prediction and automated program repair. Yet existing SZZ-based approaches are limited by their reliance on git blame, which restricts the search space to commits that directly modified the fixed lines. Our preliminary study on 2,102 validated bug-fixing commits reveals that this limitation is significant: over 40% of cases cannot be solved by blame alone, as 28%

Refer to the [full paper](https://arxiv.org/abs/2602.02934v1) for detailed methodology.