---
name: "internalizing-llm-reasoning-via"
description: "The internalization of chain-of-thought processes into hidden states has emerged as a highly efficient paradigm for scaling test-time compute. Implements techniques from 'Internalizing LLM Reasoning via Discovery and Replay of Latent Actions'. Use for tasks involving: search retrieval, agent framework. Triggers: \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\""
---

# Internalizing LLM Reasoning via Discovery and Replay of Latent Actions

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2602.04925v1](https://arxiv.org/abs/2602.04925v1) | **Category:** cs.LG | **Published:** 2026-02-04
**Authors:** Zhenning Shi, Yijia Zhu, Junhan Shi, Xun Zhang, Lei Wang

## Research Context

> The internalization of chain-of-thought processes into hidden states has emerged as a highly efficient paradigm for scaling test-time compute. However, existing activation steering methods rely on static control vectors that fail to adapt to the non-stationary evolution of complex reasoning tasks. To address this limitation, we propose STIR (Self-Distilled Tools for Internal Reasoning), a framework that reformulates reasoning enhancement as a dynamic latent trajectory control problem. STIR introduces a synergistic three-stage pipeline: (1) differential intrinsic action induction harvests latent reasoning successes to crystallize steering primitives; (2) sparse control basis construction curates a compact, geometrically diverse tool library; and (3) value-modulated trajectory intervention dynamically injects context-specific impulses via anchor-based gating. Extensive experiments on six arithmetic and logical benchmarks across four representative models demonstrate that STIR improves average accuracy by 1.9% to 7.5% while reducing average token consumption by up to 35% compared to vanilla decoding. These findings demonstrate that the benefits of explicit chain-of-thought can be realized through dynamic latent trajectory control, internalizing the reasoning process to bypass the explicit generation while achieving superior fidelity. Our code is available at https://github.com/sznnzs/LLM-Latent-Action.

## Key Techniques from This Paper

- Proposes: stir (self-distilled tools for internal reasoning)

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

## Approach Selection

Determine the appropriate approach based on the user's request:

**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value

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

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses the internalization of chain-of-thought processes into hidden states has emerged as a highly efficient paradigm for scaling test-time compute.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.04925v1) for detailed methodology, experimental results, and ablation studies.