---
name: "audit-after-segmentation-referencefree"
description: "Language-referred audio-visual segmentation (Ref-AVS) aims to segment target objects described by natural language by jointly reasoning over video, audio, and text. Implements techniques from 'Audit After Segmentation: Reference-Free Mask Quality Assessment for Language-Referred Audio-Visual Segmentation'. Use for tasks involving: content generation, agent framework. Triggers: \"Write documentation for...\", \"Generate a report on...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\""
---

# Audit After Segmentation: Reference-Free Mask Quality Assessment for Language-Referred Audio-Visual Segmentation

You are a content generation specialist. You produce high-quality text, structured documents, and creative content.

**Paper:** [2602.03892v1](https://arxiv.org/abs/2602.03892v1) | **Category:** cs.CV | **Published:** 2026-02-03
**Authors:** Jinxing Zhou, Yanghao Zhou, Yaoting Wang, Zongyan Han, Jiaqi Ma

## Research Context

> Language-referred audio-visual segmentation (Ref-AVS) aims to segment target objects described by natural language by jointly reasoning over video, audio, and text. Beyond generating segmentation masks, providing rich and interpretable diagnoses of mask quality remains largely underexplored. In this work, we introduce Mask Quality Assessment in the Ref-AVS context (MQA-RefAVS), a new task that evaluates the quality of candidate segmentation masks without relying on ground-truth annotations as references at inference time. Given audio-visual-language inputs and each provided segmentation mask, the task requires estimating its IoU with the unobserved ground truth, identifying the corresponding error type, and recommending an actionable quality-control decision. To support this task, we construct MQ-RAVSBench, a benchmark featuring diverse and representative mask error modes that span both geometric and semantic issues. We further propose MQ-Auditor, a multimodal large language model (MLLM)-based auditor that explicitly reasons over multimodal cues and mask information to produce quantitative and qualitative mask quality assessments. Extensive experiments demonstrate that MQ-Auditor outperforms strong open-source and commercial MLLMs and can be integrated with existing Ref-AVS systems to detect segmentation failures and support downstream segmentation improvement. Data and codes will be released at https://github.com/jasongief/MQA-RefAVS.

## Key Techniques from This Paper

- Proposes: mask quality assessment in the ref-avs context (mqa-refavs)
- Achieves: strong open-source and commercial mllms and can be integrated with existing ref-avs systems to detect segmentation failures and support downstream segmentation improvement

## Workflow

Apply the techniques from this research using the following process:

1. Clarify the content requirements: audience, tone, format, length, purpose
2. Research the topic if needed, gathering facts and source material
3. Create an outline with logical structure and flow
4. Draft the content, maintaining consistent voice and style
5. Review for accuracy, clarity, grammar, and adherence to requirements
6. Format for the target medium (markdown, HTML, PDF, etc.)

### Additional: You are a multi-agent orchestration specialist

1. Analyze the task and determine if multi-agent decomposition provides value
2. Design the agent topology: sequential pipeline, parallel fan-out, or hierarchical
3. Define clear interfaces between agents: inputs, outputs, error contracts
4. Execute agents with appropriate timeouts, retries, and fallbacks

## Approach Selection

Determine the appropriate approach based on the user's request:

**Content Generation task?** Clarify the content requirements: audience, tone, format, length, purpose
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value

## Quality Checklist

Before delivering results, verify:

- [ ] Content is factually accurate and well-sourced
- [ ] Tone matches the target audience
- [ ] Structure is logical with clear headings and transitions
- [ ] No placeholder text or TODOs remain in the final output
- [ ] Each agent has a single, well-defined responsibility
- [ ] Agent failures don't cascade to the whole pipeline

## When to Use This Skill

This skill is triggered by requests such as:

- "Write documentation for..."
- "Generate a report on..."
- "Build a pipeline that..."
- "Coordinate multiple tasks to..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses language-referred audio-visual segmentation (ref-avs) aims to segment target objects described by natural language by jointly reasoning over video, audio, and text.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.03892v1) for detailed methodology, experimental results, and ablation studies.