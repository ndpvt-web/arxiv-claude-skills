---
name: "agentskiller-scaling-generalist-agent"
description: "Large Language Model agents demonstrate potential in solving real-world problems via tools, yet generalist intelligence is bottlenecked by scarce high-quality, long-horizon data. Implements techniques from the paper 'AgentSkiller: Scaling Generalist Agent Intelligence through Semantically Integrated Cross-Domain Data Synthesis' for build and orchestrate ai agent workflows. Use when tasks involve (agent framework), (security), (database & query) or when the user references techniques from this research area."
---

# AgentSkiller: Scaling Generalist Agent Intelligence through Semantically Integrated Cross-Domain Data Synthesis

**Source:** [https://arxiv.org/abs/2602.09372v1](https://arxiv.org/abs/2602.09372v1)
**Category:** cs.CL | **Published:** 2026-02-10 | **Skill Score:** 74
**Authors:** Zexu Sun, Bokai Ji, Hengyi Cai...

## Core Capability

Build and orchestrate AI agent workflows.

## Key Techniques

- **Proposed technique:** agentskiller

## Workflow

1. Decompose complex tasks into subtasks
2. Plan agent coordination and communication patterns
3. Execute subtasks with appropriate tools and models
4. Monitor agent progress and handle failures
5. Aggregate results and present final output

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

> Large Language Model agents demonstrate potential in solving real-world problems via tools, yet generalist intelligence is bottlenecked by scarce high-quality, long-horizon data. Existing methods collect privacy-constrained API logs or generate scripted interactions lacking diversity, which struggle to produce data requisite for scaling capabilities. We propose AgentSkiller, a fully automated framework synthesizing multi-turn interaction data across realistic, semantically linked domains. It emp

Refer to the [full paper](https://arxiv.org/abs/2602.09372v1) for detailed methodology.