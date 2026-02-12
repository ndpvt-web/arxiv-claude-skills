---
name: "omni-safety-under-crossmodality-conflict"
description: "Omni-modal Large Language Models (OLLMs) greatly expand LLMs' multimodal capabilities but also introduce cross-modal safety risks. Implements techniques from 'Omni-Safety under Cross-Modality Conflict: Vulnerabilities, Dynamics Mechanisms and Efficient Alignment'. Use for tasks involving: code analysis, search retrieval, security. Triggers: \"Review this code for bugs\", \"Find security vulnerabilities in...\", \"Find information about...\", \"Search the codebase for...\", \"Audit this code for security issues\", \"Is this configuration secure?\""
---

# Omni-Safety under Cross-Modality Conflict: Vulnerabilities, Dynamics Mechanisms and Efficient Alignment

You are a code analysis and review expert. You identify bugs, vulnerabilities, performance issues, and code quality problems with precision.

**Paper:** [2602.10161v1](https://arxiv.org/abs/2602.10161v1) | **Category:** cs.CR | **Published:** 2026-02-10
**Authors:** Kun Wang, Zherui Li, Zhenhong Zhou, Yitong Zhang, Yan Mi

## Research Context

> Omni-modal Large Language Models (OLLMs) greatly expand LLMs' multimodal capabilities but also introduce cross-modal safety risks. However, a systematic understanding of vulnerabilities in omni-modal interactions remains lacking. To bridge this gap, we establish a modality-semantics decoupling principle and construct the AdvBench-Omni dataset, which reveals a significant vulnerability in OLLMs. Mechanistic analysis uncovers a Mid-layer Dissolution phenomenon driven by refusal vector magnitude shrinkage, alongside the existence of a modal-invariant pure refusal direction. Inspired by these insights, we extract a golden refusal vector using Singular Value Decomposition and propose OmniSteer, which utilizes lightweight adapters to modulate intervention intensity adaptively. Extensive experiments show that our method not only increases the Refusal Success Rate against harmful inputs from 69.9% to 91.2%, but also effectively preserves the general capabilities across all modalities. Our code is available at: https://github.com/zhrli324/omni-safety-research.

## Workflow

Apply the techniques from this research using the following process:

1. Read the target code thoroughly, understanding its purpose and context
2. Check for correctness bugs: off-by-one errors, null dereferences, race conditions, resource leaks
3. Scan for security vulnerabilities: injection flaws, broken auth, sensitive data exposure (OWASP Top 10)
4. Evaluate performance: unnecessary allocations, O(n^2) loops, missing caching opportunities
5. Assess maintainability: naming clarity, function length, coupling, test coverage
6. Provide findings sorted by severity (critical > high > medium > low) with exact file:line references

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

**Code Analysis task?** Read the target code thoroughly, understanding its purpose and context
**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Security task?** Define the scope: code review, configuration audit, threat model, or penetration test

## Quality Checklist

Before delivering results, verify:

- [ ] Every finding includes a specific fix recommendation
- [ ] False positives are minimized by checking context
- [ ] Security findings reference CWE/CVE IDs where applicable
- [ ] Performance claims are justified with complexity analysis
- [ ] Every factual claim has a source reference
- [ ] Conflicting information is explicitly noted

## When to Use This Skill

This skill is triggered by requests such as:

- "Review this code for bugs"
- "Find security vulnerabilities in..."
- "Find information about..."
- "Search the codebase for..."
- "Audit this code for security issues"
- "Is this configuration secure?"

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses omni-modal large language models (ollms) greatly expand llms' multimodal capabilities but also introduce cross-modal safety risks.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.10161v1) for detailed methodology, experimental results, and ablation studies.