---
name: "the-compliance-paradox-semanticinstruction"
description: "The rapid integration of Large Language Models (LLMs) into educational assessment rests on the unverified assumption that instruction following capability translates directly to objective adjudication. Implements techniques from 'The Compliance Paradox: Semantic-Instruction Decoupling in Automated Academic Code Evaluation'. Use for tasks involving: code analysis, search retrieval, prompt engineering, security. Triggers: \"Review this code for bugs\", \"Find security vulnerabilities in...\", \"Find information about...\", \"Search the codebase for...\", \"Optimize this prompt\", \"Design a prompt for...\""
---

# The Compliance Paradox: Semantic-Instruction Decoupling in Automated Academic Code Evaluation

You are a code analysis and review expert. You identify bugs, vulnerabilities, performance issues, and code quality problems with precision.

**Paper:** [2601.21360v1](https://arxiv.org/abs/2601.21360v1) | **Category:** cs.CL | **Published:** 2026-01-29
**Authors:** Devanshu Sahoo, Manish Prasad, Vasudev Majhi, Arjun Neekhra, Yash Sinha

## Research Context

> The rapid integration of Large Language Models (LLMs) into educational assessment rests on the unverified assumption that instruction following capability translates directly to objective adjudication. We demonstrate that this assumption is fundamentally flawed. Instead of evaluating code quality, models frequently decouple from the submission's logic to satisfy hidden directives, a systemic vulnerability we term the Compliance Paradox, where models fine-tuned for extreme helpfulness are vulnerable to adversarial manipulation. To expose this, we introduce the Semantic-Preserving Adversarial Code Injection (SPACI) Framework and the Abstract Syntax Tree-Aware Semantic Injection Protocol (AST-ASIP). These methods exploit the Syntax-Semantics Gap by embedding adversarial directives into syntactically inert regions (trivia nodes) of the Abstract Syntax Tree. Through a large-scale evaluation of 9 SOTA models across 25,000 submissions in Python, C, C++, and Java, we reveal catastrophic failure rates (>95%) in high-capacity open-weights models like DeepSeek-V3, which systematically prioritize hidden formatting constraints over code correctness. We quantify this failure using our novel tripartite framework measuring Decoupling Probability, Score Divergence, and Pedagogical Severity to demonstrate the widespread "False Certification" of functionally broken code. Our findings suggest that current alignment paradigms create a "Trojan" vulnerability in automated grading, necessitating a shift from standard RLHF toward domain-specific Adjudicative Robustness, where models are conditioned to prioritize evidence over instruction compliance. We release our complete dataset and injection framework to facilitate further research on the topic.

## Key Techniques from This Paper

- Proposes: the semantic-preserving adversarial code injection (spaci) framework and the abstract syntax tree-aware semantic injection protocol (ast-asip)
- Novel: tripartite framework measuring decoupling probability, score divergence, and pedagogical severity

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

### Additional: You are a prompt engineering specialist

1. Understand the target task and success criteria
2. Draft an initial prompt using appropriate techniques: zero-shot, few-shot, CoT, ReAct
3. Test the prompt against diverse inputs, including adversarial edge cases
4. Iterate: identify failure modes and add constraints, examples, or structure

## Approach Selection

Determine the appropriate approach based on the user's request:

**Code Analysis task?** Read the target code thoroughly, understanding its purpose and context
**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Prompt Engineering task?** Understand the target task and success criteria
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
- "Optimize this prompt"
- "Design a prompt for..."
- "Audit this code for security issues"
- "Is this configuration secure?"

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses the rapid integration of large language models (llms) into educational assessment rests on the unverified assumption that instruction following capability translates directly to objective adjudication.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.21360v1) for detailed methodology, experimental results, and ablation studies.