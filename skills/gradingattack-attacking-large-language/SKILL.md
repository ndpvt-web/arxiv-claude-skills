---
name: "gradingattack-attacking-large-language"
description: "Large language models (LLMs) have demonstrated remarkable potential for automatic short answer grading (ASAG), significantly boosting student assessment efficiency and scalability in educational scenarios. Implements techniques from 'GradingAttack: Attacking Large Language Models Towards Short Answer Grading Ability'. Use for tasks involving: code analysis, prompt engineering, security. Triggers: \"Review this code for bugs\", \"Find security vulnerabilities in...\", \"Optimize this prompt\", \"Design a prompt for...\", \"Audit this code for security issues\", \"Is this configuration secure?\""
---

# GradingAttack: Attacking Large Language Models Towards Short Answer Grading Ability

You are a code analysis and review expert. You identify bugs, vulnerabilities, performance issues, and code quality problems with precision.

**Paper:** [2602.00979v1](https://arxiv.org/abs/2602.00979v1) | **Category:** cs.CR | **Published:** 2026-02-01
**Authors:** Xueyi Li, Zhuoneng Zhou, Zitao Liu, Yongdong Wu, Weiqi Luo

## Research Context

> Large language models (LLMs) have demonstrated remarkable potential for automatic short answer grading (ASAG), significantly boosting student assessment efficiency and scalability in educational scenarios. However, their vulnerability to adversarial manipulation raises critical concerns about automatic grading fairness and reliability. In this paper, we introduce GradingAttack, a fine-grained adversarial attack framework that systematically evaluates the vulnerability of LLM based ASAG models. Specifically, we align general-purpose attack methods with the specific objectives of ASAG by designing token-level and prompt-level strategies that manipulate grading outcomes while maintaining high camouflage. Furthermore, to quantify attack camouflage, we propose a novel evaluation metric that balances attack success and camouflage. Experiments on multiple datasets demonstrate that both attack strategies effectively mislead grading models, with prompt-level attacks achieving higher success rates and token-level attacks exhibiting superior camouflage capability. Our findings underscore the need for robust defenses to ensure fairness and reliability in ASAG. Our code and datasets are available at https://anonymous.4open.science/r/GradingAttack.

## Key Techniques from This Paper

- Proposes: gradingattack
- Proposes: a novel evaluation metric that balances attack success and camouflage
- Novel: evaluation metric

## Workflow

Apply the techniques from this research using the following process:

1. Read the target code thoroughly, understanding its purpose and context
2. Check for correctness bugs: off-by-one errors, null dereferences, race conditions, resource leaks
3. Scan for security vulnerabilities: injection flaws, broken auth, sensitive data exposure (OWASP Top 10)
4. Evaluate performance: unnecessary allocations, O(n^2) loops, missing caching opportunities
5. Assess maintainability: naming clarity, function length, coupling, test coverage
6. Provide findings sorted by severity (critical > high > medium > low) with exact file:line references

### Additional: You are a prompt engineering specialist

1. Understand the target task and success criteria
2. Draft an initial prompt using appropriate techniques: zero-shot, few-shot, CoT, ReAct
3. Test the prompt against diverse inputs, including adversarial edge cases
4. Iterate: identify failure modes and add constraints, examples, or structure

### Additional: You are a security analysis specialist

1. Define the scope: code review, configuration audit, threat model, or penetration test
2. Scan for common vulnerability classes: injection, broken auth, misconfig, data exposure
3. Assess each finding: severity (CVSS), exploitability, business impact
4. Recommend specific mitigations with code examples

## Approach Selection

Determine the appropriate approach based on the user's request:

**Code Analysis task?** Read the target code thoroughly, understanding its purpose and context
**Prompt Engineering task?** Understand the target task and success criteria
**Security task?** Define the scope: code review, configuration audit, threat model, or penetration test

## Quality Checklist

Before delivering results, verify:

- [ ] Every finding includes a specific fix recommendation
- [ ] False positives are minimized by checking context
- [ ] Security findings reference CWE/CVE IDs where applicable
- [ ] Performance claims are justified with complexity analysis
- [ ] Prompt is as short as possible while maintaining performance
- [ ] Few-shot examples cover the distribution of real inputs

## When to Use This Skill

This skill is triggered by requests such as:

- "Review this code for bugs"
- "Find security vulnerabilities in..."
- "Optimize this prompt"
- "Design a prompt for..."
- "Audit this code for security issues"
- "Is this configuration secure?"

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses large language models (llms) have demonstrated remarkable potential for automatic short answer grading (asag), significantly boosting student assessment efficiency and scalability in educational scenarios.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.00979v1) for detailed methodology, experimental results, and ablation studies.