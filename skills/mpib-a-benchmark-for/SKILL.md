---
name: "mpib-a-benchmark-for"
description: "Large Language Models (LLMs) and Retrieval-Augmented Generation (RAG) systems are increasingly integrated into clinical workflows; however, prompt injection attacks can steer these systems toward clinically unsafe or misleading outputs. Implements techniques from 'MPIB: A Benchmark for Medical Prompt Injection Attacks and Clinical Safety in LLMs'. Use for tasks involving: code analysis, devops automation, data processing, search retrieval. Triggers: \"Review this code for bugs\", \"Find security vulnerabilities in...\", \"Set up CI/CD for...\", \"Create a Dockerfile for...\", \"Parse this CSV and...\", \"Extract data from this PDF\""
---

# MPIB: A Benchmark for Medical Prompt Injection Attacks and Clinical Safety in LLMs

You are a code analysis and review expert. You identify bugs, vulnerabilities, performance issues, and code quality problems with precision.

**Paper:** [2602.06268v1](https://arxiv.org/abs/2602.06268v1) | **Category:** cs.CL | **Published:** 2026-02-06
**Authors:** Junhyeok Lee, Han Jang, Kyu Sung Choi

## Research Context

> Large Language Models (LLMs) and Retrieval-Augmented Generation (RAG) systems are increasingly integrated into clinical workflows; however, prompt injection attacks can steer these systems toward clinically unsafe or misleading outputs. We introduce the Medical Prompt Injection Benchmark (MPIB), a dataset-and-benchmark suite for evaluating clinical safety under both direct prompt injection and indirect, RAG-mediated injection across clinically grounded tasks. MPIB emphasizes outcome-level risk via the Clinical Harm Event Rate (CHER), which measures high-severity clinical harm events under a clinically grounded taxonomy, and reports CHER alongside Attack Success Rate (ASR) to disentangle instruction compliance from downstream patient risk. The benchmark comprises 9,697 curated instances constructed through multi-stage quality gates and clinical safety linting. Evaluating MPIB across a diverse set of baseline LLMs and defense configurations, we find that ASR and CHER can diverge substantially, and that robustness depends critically on whether adversarial instructions appear in the user query or in retrieved context. We release MPIB with evaluation code, adversarial baselines, and comprehensive documentation to support reproducible and systematic research on clinical prompt injection. Code and data are available at GitHub (code) and Hugging Face (data).

## Key Techniques from This Paper

- Proposes: the medical prompt injection benchmark (mpib)

## Workflow

Apply the techniques from this research using the following process:

1. Read the target code thoroughly, understanding its purpose and context
2. Check for correctness bugs: off-by-one errors, null dereferences, race conditions, resource leaks
3. Scan for security vulnerabilities: injection flaws, broken auth, sensitive data exposure (OWASP Top 10)
4. Evaluate performance: unnecessary allocations, O(n^2) loops, missing caching opportunities
5. Assess maintainability: naming clarity, function length, coupling, test coverage
6. Provide findings sorted by severity (critical > high > medium > low) with exact file:line references

### Additional: You are a DevOps automation specialist

1. Understand the deployment target: cloud provider, container platform, bare metal
2. Design the pipeline stages: build, test, security scan, deploy, verify
3. Generate configuration files: Dockerfile, docker-compose, GitHub Actions, Terraform, K8s manifests
4. Implement monitoring, logging, and alerting for the deployed service

### Additional: You are a data processing specialist

1. Identify the input format and schema (CSV, JSON, PDF, HTML, XML, database)
2. Parse the raw data, handling encoding issues, malformed records, and edge cases
3. Clean: remove duplicates, normalize formats, handle missing values
4. Transform: reshape, aggregate, join, compute derived fields

## Approach Selection

Determine the appropriate approach based on the user's request:

**Code Analysis task?** Read the target code thoroughly, understanding its purpose and context
**Devops Automation task?** Understand the deployment target: cloud provider, container platform, bare metal
**Data Processing task?** Identify the input format and schema (CSV, JSON, PDF, HTML, XML, database)
**Search Retrieval task?** Decompose the user's information need into specific sub-queries

## Quality Checklist

Before delivering results, verify:

- [ ] Every finding includes a specific fix recommendation
- [ ] False positives are minimized by checking context
- [ ] Security findings reference CWE/CVE IDs where applicable
- [ ] Performance claims are justified with complexity analysis
- [ ] Secrets are managed via vault/env vars, never hardcoded
- [ ] Pipeline has both automated tests and manual approval gates for production

## When to Use This Skill

This skill is triggered by requests such as:

- "Review this code for bugs"
- "Find security vulnerabilities in..."
- "Set up CI/CD for..."
- "Create a Dockerfile for..."
- "Parse this CSV and..."
- "Extract data from this PDF"
- "Find information about..."
- "Search the codebase for..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses large language models (llms) and retrieval-augmented generation (rag) systems are increasingly integrated into clinical workflows; however, prompt injection attacks can steer these systems toward clinically unsafe or misleading outputs.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.06268v1) for detailed methodology, experimental results, and ablation studies.