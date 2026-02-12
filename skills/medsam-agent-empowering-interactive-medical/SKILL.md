---
name: "medsam-agent-empowering-interactive-medical"
description: "Medical image segmentation is evolving from task-specific models toward generalizable frameworks. Implements techniques from 'MedSAM-Agent: Empowering Interactive Medical Image Segmentation with Multi-turn Agentic Reinforcement Learning'. Use for tasks involving: search retrieval, agent framework, prompt engineering, security. Triggers: \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\", \"Optimize this prompt\", \"Design a prompt for...\""
---

# MedSAM-Agent: Empowering Interactive Medical Image Segmentation with Multi-turn Agentic Reinforcement Learning

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2602.03320v1](https://arxiv.org/abs/2602.03320v1) | **Category:** cs.CV | **Published:** 2026-02-03
**Authors:** Shengyuan Liu, Liuxin Bao, Qi Yang, Wanting Geng, Boyun Zheng

## Research Context

> Medical image segmentation is evolving from task-specific models toward generalizable frameworks. Recent research leverages Multi-modal Large Language Models (MLLMs) as autonomous agents, employing reinforcement learning with verifiable reward (RLVR) to orchestrate specialized tools like the Segment Anything Model (SAM). However, these approaches often rely on single-turn, rigid interaction strategies and lack process-level supervision during training, which hinders their ability to fully exploit the dynamic potential of interactive tools and leads to redundant actions. To bridge this gap, we propose MedSAM-Agent, a framework that reformulates interactive segmentation as a multi-step autonomous decision-making process. First, we introduce a hybrid prompting strategy for expert-curated trajectory generation, enabling the model to internalize human-like decision heuristics and adaptive refinement strategies. Furthermore, we develop a two-stage training pipeline that integrates multi-turn, end-to-end outcome verification with a clinical-fidelity process reward design to promote interaction parsimony and decision efficiency. Extensive experiments across 6 medical modalities and 21 datasets demonstrate that MedSAM-Agent achieves state-of-the-art performance, effectively unifying autonomous medical reasoning with robust, iterative optimization. Code is available \href{https://github.com/CUHK-AIM-Group/MedSAM-Agent}{here}.

## Key Techniques from This Paper

- Proposes: medsam-agent
- Proposes: a hybrid prompting strategy for expert-curated trajectory generation, enabling the model to internalize human-like decision heuristics and adaptive refinement strategies
- Achieves: state-of-the-art performance

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
**Security task?** Define the scope: code review, configuration audit, threat model, or penetration test

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
- "Audit this code for security issues"
- "Is this configuration secure?"

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses medical image segmentation is evolving from task-specific models toward generalizable frameworks.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.03320v1) for detailed methodology, experimental results, and ablation studies.