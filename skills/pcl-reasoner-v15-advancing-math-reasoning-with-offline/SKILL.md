---
name: "pcl-reasoner-v15-advancing-math-reasoning-with-offline"
description: "We present PCL-Reasoner-V1.5, a 32-billion-parameter large language model (LLM) for mathematical reasoning. Implements techniques from the paper 'PCL-Reasoner-V1.5: Advancing Math Reasoning with Offline Reinforcement Learning' for search, retrieve, and synthesize information. Use when tasks involve (search & retrieval), (agent framework) or when the user references techniques from this research area."
---

# PCL-Reasoner-V1.5: Advancing Math Reasoning with Offline Reinforcement Learning

**Source:** [https://arxiv.org/abs/2601.14716v1](https://arxiv.org/abs/2601.14716v1)
**Category:** cs.LG | **Published:** 2026-01-21 | **Skill Score:** 68
**Authors:** Yao Lu, Dengdong Fan, Jianzheng Nie...

## Core Capability

Search, retrieve, and synthesize information.

## Key Techniques

- **Proposed technique:** pcl-reasoner-v1
- **Achievement:** state-of-the-art performance among models post-trained on qwen2

## Workflow

1. Parse the user's information need or query
2. Formulate effective search strategies
3. Retrieve relevant documents, code, or data
4. Synthesize findings into a coherent response
5. Provide citations and references

## Agent Coordination

- Decompose complex tasks into independent subtasks where possible
- Use parallel execution for independent subtasks
- Implement error recovery and retry logic
- Aggregate partial results when full completion fails

## Research Context

> We present PCL-Reasoner-V1.5, a 32-billion-parameter large language model (LLM) for mathematical reasoning. The model is built upon Qwen2.5-32B and refined via supervised fine-tuning (SFT) followed by reinforcement learning (RL). A central innovation is our proposed offline RL method, which provides superior training stability and efficiency over standard online RL methods such as GRPO. Our model achieves state-of-the-art performance among models post-trained on Qwen2.5-32B, attaining average ac

Refer to the [full paper](https://arxiv.org/abs/2601.14716v1) for detailed methodology.