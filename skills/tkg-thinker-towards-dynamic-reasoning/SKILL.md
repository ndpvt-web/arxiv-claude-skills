---
name: "tkg-thinker-towards-dynamic-reasoning"
description: "Temporal knowledge graph question answering (TKGQA) aims to answer time-sensitive questions by leveraging temporal knowledge bases. Implements techniques from 'TKG-Thinker: Towards Dynamic Reasoning over Temporal Knowledge Graphs via Agentic Reinforcement Learning'. Use for tasks involving: search retrieval, agent framework, prompt engineering. Triggers: \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\", \"Optimize this prompt\", \"Design a prompt for...\""
---

# TKG-Thinker: Towards Dynamic Reasoning over Temporal Knowledge Graphs via Agentic Reinforcement Learning

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2602.05818v2](https://arxiv.org/abs/2602.05818v2) | **Category:** cs.AI | **Published:** 2026-02-05
**Authors:** Zihao Jiang, Miao Peng, Zhenyan Shan, Wenjie Xu, Ben Liu

## Research Context

> Temporal knowledge graph question answering (TKGQA) aims to answer time-sensitive questions by leveraging temporal knowledge bases. While Large Language Models (LLMs) demonstrate significant potential in TKGQA, current prompting strategies constrain their efficacy in two primary ways. First, they are prone to reasoning hallucinations under complex temporal constraints. Second, static prompting limits model autonomy and generalization, as it lack optimization through dynamic interaction with temporal knowledge graphs (TKGs) environments. To address these limitations, we propose \textbf{TKG-Thinker}, a novel agent equipped with autonomous planning and adaptive retrieval capabilities for reasoning over TKGs. Specifically, TKG-Thinker performs in-depth temporal reasoning through dynamic multi-turn interactions with TKGs via a dual-training strategy. We first apply Supervised Fine-Tuning (SFT) with chain of thought data to instill core planning capabilities, followed by a Reinforcement Learning (RL) stage that leverages multi-dimensional rewards to refine reasoning policies under intricate temporal constraints. Experimental results on benchmark datasets with three open-source LLMs show that TKG-Thinker achieves state-of-the-art performance and exhibits strong generalization across complex TKGQA settings.

## Key Techniques from This Paper

- Proposes: \textbf{tkg-thinker}
- Novel: agent equipped with autonomous planning and adaptive retrieval capabilities
- Achieves: state-of-the-art performance and exhibits strong generalization across complex tkgqa settings

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

1. **Understand the problem context** -- The paper addresses temporal knowledge graph question answering (tkgqa) aims to answer time-sensitive questions by leveraging temporal knowledge bases.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.05818v2) for detailed methodology, experimental results, and ablation studies.