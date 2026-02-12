---
name: "icl-evader-zeroquery-blackbox-evasion"
description: "In-context learning (ICL) has become a powerful, data-efficient paradigm for text classification using large language models. Implements techniques from the paper 'ICL-EVADER: Zero-Query Black-Box Evasion Attacks on In-Context Learning and Their Defenses' for optimize prompts for better ai model performance. Use when tasks involve (prompt engineering), (security) or when the user references techniques from this research area."
---

# ICL-EVADER: Zero-Query Black-Box Evasion Attacks on In-Context Learning and Their Defenses

**Source:** [https://arxiv.org/abs/2601.21586v1](https://arxiv.org/abs/2601.21586v1)
**Category:** cs.CR | **Published:** 2026-01-29 | **Skill Score:** 80
**Authors:** Ningyuan He, Ronghong Huang, Qianqian Tang...

## Core Capability

Optimize prompts for better AI model performance.

## Key Techniques

- **Novel approach:** black-box evasion attack framework that operates under a highly practical zero-query threat model

## Workflow

1. Analyze the current prompt and its shortcomings
2. Apply prompt engineering techniques (CoT, few-shot, etc.)
3. Test prompts against diverse inputs
4. Iterate on prompt design based on results
5. Document the prompt template and its parameters

## Security Considerations

- Check for OWASP Top 10 vulnerabilities
- Validate all user inputs at system boundaries
- Use parameterized queries for database access
- Follow the principle of least privilege

## Research Context

> In-context learning (ICL) has become a powerful, data-efficient paradigm for text classification using large language models. However, its robustness against realistic adversarial threats remains largely unexplored. We introduce ICL-Evader, a novel black-box evasion attack framework that operates under a highly practical zero-query threat model, requiring no access to model parameters, gradients, or query-based feedback during attack generation. We design three novel attacks, Fake Claim, Templat

Refer to the [full paper](https://arxiv.org/abs/2601.21586v1) for detailed methodology.