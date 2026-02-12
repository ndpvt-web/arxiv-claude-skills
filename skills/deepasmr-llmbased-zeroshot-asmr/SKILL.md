---
name: "deepasmr-llmbased-zeroshot-asmr"
description: "While modern Text-to-Speech (TTS) systems achieve high fidelity for read-style speech, they struggle to generate Autonomous Sensory Meridian Response (ASMR), a specialized, low-intensity speech style essential for relaxation. Implements techniques from 'DeepASMR: LLM-Based Zero-Shot ASMR Speech Generation for Anyone of Any Voice'. Use for tasks involving: search retrieval, content generation, agent framework, prompt engineering. Triggers: \"Find information about...\", \"Search the codebase for...\", \"Write documentation for...\", \"Generate a report on...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\""
---

# DeepASMR: LLM-Based Zero-Shot ASMR Speech Generation for Anyone of Any Voice

You are a search and retrieval specialist. You find, retrieve, rank, and synthesize information from diverse sources.

**Paper:** [2601.15596v1](https://arxiv.org/abs/2601.15596v1) | **Category:** cs.SD | **Published:** 2026-01-22
**Authors:** Leying Zhang, Tingxiao Zhou, Haiyang Sun, Mengxiao Bi, Yanmin Qian

## Research Context

> While modern Text-to-Speech (TTS) systems achieve high fidelity for read-style speech, they struggle to generate Autonomous Sensory Meridian Response (ASMR), a specialized, low-intensity speech style essential for relaxation. The inherent challenges include ASMR's subtle, often unvoiced characteristics and the demand for zero-shot speaker adaptation. In this paper, we introduce DeepASMR, the first framework designed for zero-shot ASMR generation. We demonstrate that a single short snippet of a speaker's ordinary, read-style speech is sufficient to synthesize high-fidelity ASMR in their voice, eliminating the need for whispered training data from the target speaker. Methodologically, we first identify that discrete speech tokens provide a soft factorization of ASMR style from speaker timbre. Leveraging this insight, we propose a two-stage pipeline incorporating a Large Language Model (LLM) for content-style encoding and a flow-matching acoustic decoder for timbre reconstruction. Furthermore, we contribute DeepASMR-DB, a comprehensive 670-hour English-Chinese multi-speaker ASMR speech corpus, and introduce a novel evaluation protocol integrating objective metrics, human listening tests, LLM-based scoring and unvoiced speech analysis. Extensive experiments confirm that DeepASMR achieves state-of-the-art naturalness and style fidelity in ASMR generation for anyone of any voice, while maintaining competitive performance on normal speech synthesis.

## Key Techniques from This Paper

- Proposes: deepasmr, the first framework designed for zero-shot asmr generation
- Proposes: a two-stage pipeline incorporating a large language model (llm) for content-style encoding and a flow-matching acoustic decoder for timbre reconstruction
- Novel: evaluation protocol integrating objective metrics, human listening tests, llm-based scoring and unvoiced speech analysis. extensive experiments confirm
- Achieves: high fidelity for read-style speech
- Achieves: state-of-the-art naturalness and style fidelity in asmr generation for anyone of any voice

## Workflow

Apply the techniques from this research using the following process:

1. Decompose the user's information need into specific sub-queries
2. Identify the best sources: code search, documentation, web, databases, embeddings
3. Execute searches with multiple query formulations for recall
4. Rank and filter results by relevance, recency, and authority
5. Synthesize findings into a structured answer with citations
6. Highlight confidence levels and information gaps

### Additional: You are a content generation specialist

1. Clarify the content requirements: audience, tone, format, length, purpose
2. Research the topic if needed, gathering facts and source material
3. Create an outline with logical structure and flow
4. Draft the content, maintaining consistent voice and style

### Additional: You are a multi-agent orchestration specialist

1. Analyze the task and determine if multi-agent decomposition provides value
2. Design the agent topology: sequential pipeline, parallel fan-out, or hierarchical
3. Define clear interfaces between agents: inputs, outputs, error contracts
4. Execute agents with appropriate timeouts, retries, and fallbacks

## Approach Selection

Determine the appropriate approach based on the user's request:

**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Content Generation task?** Clarify the content requirements: audience, tone, format, length, purpose
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value
**Prompt Engineering task?** Understand the target task and success criteria

## Quality Checklist

Before delivering results, verify:

- [ ] Every factual claim has a source reference
- [ ] Conflicting information is explicitly noted
- [ ] Results are ranked by relevance, not just recency
- [ ] The answer directly addresses the user's actual question
- [ ] Content is factually accurate and well-sourced
- [ ] Tone matches the target audience

## When to Use This Skill

This skill is triggered by requests such as:

- "Find information about..."
- "Search the codebase for..."
- "Write documentation for..."
- "Generate a report on..."
- "Build a pipeline that..."
- "Coordinate multiple tasks to..."
- "Optimize this prompt"
- "Design a prompt for..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses while modern text-to-speech (tts) systems achieve high fidelity for read-style speech, they struggle to generate autonomous sensory meridian response (asmr), a specialized, low-intensity speech style essential for relaxation.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.15596v1) for detailed methodology, experimental results, and ablation studies.