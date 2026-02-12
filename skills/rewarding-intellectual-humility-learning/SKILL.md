---
name: "rewarding-intellectual-humility-learning"
description: "Large Language Models (LLMs) often produce hallucinated or unverifiable content, undermining their reliability in factual domains. Implements techniques from 'Rewarding Intellectual Humility Learning When Not To Answer In Large Language Models'. Use for tasks involving: search retrieval. Triggers: \"Find information about...\", \"Search the codebase for...\""
---

# Rewarding Intellectual Humility Learning When Not To Answer In Large Language Models

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2601.20126v1](https://arxiv.org/abs/2601.20126v1) | **Category:** cs.CL | **Published:** 2026-01-27
**Authors:** Abha Jha, Akanksha Mahajan, Ashwath Vaithinathan Aravindan, Praveen Saravanan, Sai Sailaja Policharla

## Research Context

> Large Language Models (LLMs) often produce hallucinated or unverifiable content, undermining their reliability in factual domains. This work investigates Reinforcement Learning with Verifiable Rewards (RLVR) as a training paradigm that explicitly rewards abstention ("I don't know") alongside correctness to promote intellectual humility. We fine-tune and evaluate Granite-3.3-2B-Instruct and Qwen-3-4B-Instruct on the MedMCQA and Hendrycks Math benchmarks using a ternary reward structure ($-1$, r_abs, 1) under varying abstention reward structures. We further study the effect of combining RLVR with supervised fine-tuning strategies that teach abstention prior to reinforcement learning. Our results show that moderate abstention rewards (r_abs $\approx -0.25$ to 0.3) consistently reduce incorrect responses without severe accuracy degradation on multiple-choice tasks, with larger models exhibiting greater robustness to abstention incentives. On open-ended question answering, we observe limitations due to insufficient exploration, which can be partially mitigated through supervised abstention training. Overall, these findings demonstrate the feasibility and flexibility of verifiable reward design as a practical approach for hallucination mitigation in language models. Reproducible code for our abstention training framework is available here https://github.com/Mystic-Slice/rl-abstention.

## Workflow

Apply the techniques from this research using the following process:

1. Decompose the user's information need into specific sub-queries
2. Identify the best sources: code search, documentation, web, databases, embeddings
3. Execute searches with multiple query formulations for recall
4. Rank and filter results by relevance, recency, and authority
5. Synthesize findings into a structured answer with citations
6. Highlight confidence levels and information gaps

## Quality Checklist

Before delivering results, verify:

- [ ] Every factual claim has a source reference
- [ ] Conflicting information is explicitly noted
- [ ] Results are ranked by relevance, not just recency
- [ ] The answer directly addresses the user's actual question

## When to Use This Skill

This skill is triggered by requests such as:

- "Find information about..."
- "Search the codebase for..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses large language models (llms) often produce hallucinated or unverifiable content, undermining their reliability in factual domains.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.20126v1) for detailed methodology, experimental results, and ablation studies.