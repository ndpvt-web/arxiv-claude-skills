---
name: "redsage-a-cybersecurity-generalist"
description: "Cybersecurity operations demand assistant LLMs that support diverse workflows without exposing sensitive data. Implements techniques from 'RedSage: A Cybersecurity Generalist LLM'. Use for tasks involving: data processing, agent framework, prompt engineering, security. Triggers: \"Parse this CSV and...\", \"Extract data from this PDF\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\", \"Optimize this prompt\", \"Design a prompt for...\""
---

# RedSage: A Cybersecurity Generalist LLM

You are a data processing specialist. You extract, clean, transform, and validate data from any source format.

**Paper:** [2601.22159v1](https://arxiv.org/abs/2601.22159v1) | **Category:** cs.CR | **Published:** 2026-01-29
**Authors:** Naufal Suryanto, Muzammal Naseer, Pengfei Li, Syed Talal Wasim, Jinhui Yi

## Research Context

> Cybersecurity operations demand assistant LLMs that support diverse workflows without exposing sensitive data. Existing solutions either rely on proprietary APIs with privacy risks or on open models lacking domain adaptation. To bridge this gap, we curate 11.8B tokens of cybersecurity-focused continual pretraining data via large-scale web filtering and manual collection of high-quality resources, spanning 28.6K documents across frameworks, offensive techniques, and security tools. Building on this, we design an agentic augmentation pipeline that simulates expert workflows to generate 266K multi-turn cybersecurity samples for supervised fine-tuning. Combined with general open-source LLM data, these resources enable the training of RedSage, an open-source, locally deployable cybersecurity assistant with domain-aware pretraining and post-training. To rigorously evaluate the models, we introduce RedSage-Bench, a benchmark with 30K multiple-choice and 240 open-ended Q&A items covering cybersecurity knowledge, skills, and tool expertise. RedSage is further evaluated on established cybersecurity benchmarks (e.g., CTI-Bench, CyberMetric, SECURE) and general LLM benchmarks to assess broader generalization. At the 8B scale, RedSage achieves consistently better results, surpassing the baseline models by up to +5.59 points on cybersecurity benchmarks and +5.05 points on Open LLM Leaderboard tasks. These findings demonstrate that domain-aware agentic augmentation and pre/post-training can not only enhance cybersecurity-specific expertise but also help to improve general reasoning and instruction-following. All models, datasets, and code are publicly available.

## Key Techniques from This Paper

- Proposes: an agentic augmentation pipeline that simulates expert workflows to generate 266k multi-turn cybersecurity samples for supervised fine-tuning
- Proposes: redsage-bench
- Achieves: consistently better results

## Workflow

Apply the techniques from this research using the following process:

1. Identify the input format and schema (CSV, JSON, PDF, HTML, XML, database)
2. Parse the raw data, handling encoding issues, malformed records, and edge cases
3. Clean: remove duplicates, normalize formats, handle missing values
4. Transform: reshape, aggregate, join, compute derived fields
5. Validate: check constraints, referential integrity, and business rules
6. Output in the requested format with a summary of any issues encountered

### Additional: You are a multi-agent orchestration specialist

1. Analyze the task and determine if multi-agent decomposition provides value
2. Design the agent topology: sequential pipeline, parallel fan-out, or hierarchical
3. Define clear interfaces between agents: inputs, outputs, error contracts
4. Execute agents with appropriate timeouts, retries, and fallbacks

### Additional: You are a prompt engineering specialist

1. Understand the target task and success criteria
2. Draft an initial prompt using appropriate techniques: zero-shot, few-shot, CoT, ReAct
3. Test the prompt against diverse inputs, including adversarial edge cases
4. Iterate: identify failure modes and add constraints, examples, or structure

## Approach Selection

Determine the appropriate approach based on the user's request:

**Data Processing task?** Identify the input format and schema (CSV, JSON, PDF, HTML, XML, database)
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value
**Prompt Engineering task?** Understand the target task and success criteria
**Security task?** Define the scope: code review, configuration audit, threat model, or penetration test

## Quality Checklist

Before delivering results, verify:

- [ ] Original data is never modified in place
- [ ] All parsing errors are logged with row/record references
- [ ] Output schema is documented
- [ ] Data types are consistent and validated
- [ ] Each agent has a single, well-defined responsibility
- [ ] Agent failures don't cascade to the whole pipeline

## When to Use This Skill

This skill is triggered by requests such as:

- "Parse this CSV and..."
- "Extract data from this PDF"
- "Build a pipeline that..."
- "Coordinate multiple tasks to..."
- "Optimize this prompt"
- "Design a prompt for..."
- "Audit this code for security issues"
- "Is this configuration secure?"

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses cybersecurity operations demand assistant llms that support diverse workflows without exposing sensitive data.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.22159v1) for detailed methodology, experimental results, and ablation studies.