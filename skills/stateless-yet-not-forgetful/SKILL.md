---
name: "stateless-yet-not-forgetful"
description: "Large language models (LLMs) are commonly treated as stateless: once an interaction ends, no information is assumed to persist unless it is explicitly stored and re-supplied. Implements techniques from 'Stateless Yet Not Forgetful: Implicit Memory as a Hidden Channel in LLMs'. Use for tasks involving: search retrieval, agent framework, prompt engineering. Triggers: \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\", \"Optimize this prompt\", \"Design a prompt for...\""
---

# Stateless Yet Not Forgetful: Implicit Memory as a Hidden Channel in LLMs

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2602.08563v1](https://arxiv.org/abs/2602.08563v1) | **Category:** cs.LG | **Published:** 2026-02-09
**Authors:** Ahmed Salem, Andrew Paverd, Sahar Abdelnabi

## Research Context

> Large language models (LLMs) are commonly treated as stateless: once an interaction ends, no information is assumed to persist unless it is explicitly stored and re-supplied. We challenge this assumption by introducing implicit memory-the ability of a model to carry state across otherwise independent interactions by encoding information in its own outputs and later recovering it when those outputs are reintroduced as input. This mechanism does not require any explicit memory module, yet it creates a persistent information channel across inference requests. As a concrete demonstration, we introduce a new class of temporal backdoors, which we call time bombs. Unlike conventional backdoors that activate on a single trigger input, time bombs activate only after a sequence of interactions satisfies hidden conditions accumulated via implicit memory. We show that such behavior can be induced today through straightforward prompting or fine-tuning. Beyond this case study, we analyze broader implications of implicit memory, including covert inter-agent communication, benchmark contamination, targeted manipulation, and training-data poisoning. Finally, we discuss detection challenges and outline directions for stress-testing and evaluation, with the goal of anticipating and controlling future developments. To promote future research, we release code and data at: https://github.com/microsoft/implicitMemory.

## Key Techniques from This Paper

- Proposes: a new class of temporal backdoors
- Novel: class of temporal backdoors,

## Workflow

Apply the techniques from this research using the following process:

1. Decompose the user's information need into specific sub-queries
2. Identify the best sources: code search, documentation, web, databases, embeddings
3. Execute searches with multiple query formulations for recall
4. Rank and filter results by relevance, recency, and authority
5. Synthesize findings into a structured answer with citations
6. Highlight confidence levels and information gaps

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

**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value
**Prompt Engineering task?** Understand the target task and success criteria

## Quality Checklist

Before delivering results, verify:

- [ ] Every factual claim has a source reference
- [ ] Conflicting information is explicitly noted
- [ ] Results are ranked by relevance, not just recency
- [ ] The answer directly addresses the user's actual question
- [ ] Each agent has a single, well-defined responsibility
- [ ] Agent failures don't cascade to the whole pipeline

## When to Use This Skill

This skill is triggered by requests such as:

- "Find information about..."
- "Search the codebase for..."
- "Build a pipeline that..."
- "Coordinate multiple tasks to..."
- "Optimize this prompt"
- "Design a prompt for..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses large language models (llms) are commonly treated as stateless: once an interaction ends, no information is assumed to persist unless it is explicitly stored and re-supplied.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.08563v1) for detailed methodology, experimental results, and ablation studies.