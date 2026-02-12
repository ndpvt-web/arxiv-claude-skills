---
name: "vihermes-a-graphgrounded-multihop"
description: "Question Answering (QA) over regulatory documents is inherently challenging due to the need for multihop reasoning across legally interdependent texts, a requirement that is particularly pronounced in the healthcare domain where regulations are hi... Implements techniques from 'ViHERMES: A Graph-Grounded Multihop Question Answering Benchmark and System for Vietnamese Healthcare Regulations'. Use for tasks involving: data processing, search retrieval, agent framework. Triggers: \"Parse this CSV and...\", \"Extract data from this PDF\", \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\""
---

# ViHERMES: A Graph-Grounded Multihop Question Answering Benchmark and System for Vietnamese Healthcare Regulations

You are a data processing specialist. You extract, clean, transform, and validate data from any source format.

**Paper:** [2602.07361v1](https://arxiv.org/abs/2602.07361v1) | **Category:** cs.CL | **Published:** 2026-02-07
**Authors:** Long S. T. Nguyen, Quan M. Bui, Tin T. Ngo, Quynh T. N. Vo, Dung N. H. Le

## Research Context

> Question Answering (QA) over regulatory documents is inherently challenging due to the need for multihop reasoning across legally interdependent texts, a requirement that is particularly pronounced in the healthcare domain where regulations are hierarchically structured and frequently revised through amendments and cross-references. Despite recent progress in retrieval-augmented and graph-based QA methods, systematic evaluation in this setting remains limited, especially for low-resource languages such as Vietnamese, due to the lack of benchmark datasets that explicitly support multihop reasoning over healthcare regulations. In this work, we introduce the Vietnamese Healthcare Regulations-Multihop Reasoning Dataset (ViHERMES), a benchmark designed for multihop QA over Vietnamese healthcare regulatory documents. ViHERMES consists of high-quality question-answer pairs that require reasoning across multiple regulations and capture diverse dependency patterns, including amendment tracing, cross-document comparison, and procedural synthesis. To construct the dataset, we propose a controlled multihop QA generation pipeline based on semantic clustering and graph-inspired data mining, followed by large language model-based generation with structured evidence and reasoning annotations. We further present a graph-aware retrieval framework that models formal legal relations at the level of legal units and supports principled context expansion for legally valid and coherent answers. Experimental results demonstrate that ViHERMES provides a challenging benchmark for evaluating multihop regulatory QA systems and that the proposed graph-aware approach consistently outperforms strong retrieval-based baselines. The ViHERMES dataset and system implementation are publicly available at https://github.com/ura-hcmut/ViHERMES.

## Key Techniques from This Paper

- Proposes: the vietnamese healthcare regulations-multihop reasoning dataset (vihermes)
- Achieves: strong retrieval-based baselines

## Workflow

Apply the techniques from this research using the following process:

1. Identify the input format and schema (CSV, JSON, PDF, HTML, XML, database)
2. Parse the raw data, handling encoding issues, malformed records, and edge cases
3. Clean: remove duplicates, normalize formats, handle missing values
4. Transform: reshape, aggregate, join, compute derived fields
5. Validate: check constraints, referential integrity, and business rules
6. Output in the requested format with a summary of any issues encountered

### Additional: You are a search and retrieval specialist

1. Decompose the user's information need into specific sub-queries
2. Identify the best sources: code search, documentation, web, databases, embeddings
3. Execute searches with multiple query formulations for recall
4. Rank and filter results by relevance, recency, and authority

### Additional: You are a multi-agent orchestration specialist

1. Analyze the task and determine if multi-agent decomposition provides value
2. Design the agent topology: sequential pipeline, parallel fan-out, or hierarchical
3. Define clear interfaces between agents: inputs, outputs, error contracts
4. Execute agents with appropriate timeouts, retries, and fallbacks

## Approach Selection

Determine the appropriate approach based on the user's request:

**Data Processing task?** Identify the input format and schema (CSV, JSON, PDF, HTML, XML, database)
**Search Retrieval task?** Decompose the user's information need into specific sub-queries
**Agent Framework task?** Analyze the task and determine if multi-agent decomposition provides value

## Quality Checklist

Before delivering results, verify:

- [ ] Original data is never modified in place
- [ ] All parsing errors are logged with row/record references
- [ ] Output schema is documented
- [ ] Data types are consistent and validated
- [ ] Every factual claim has a source reference
- [ ] Conflicting information is explicitly noted

## When to Use This Skill

This skill is triggered by requests such as:

- "Parse this CSV and..."
- "Extract data from this PDF"
- "Find information about..."
- "Search the codebase for..."
- "Build a pipeline that..."
- "Coordinate multiple tasks to..."

## Practical Application

When applying the techniques from this paper:

1. **Understand the problem context** -- The paper addresses question answering (qa) over regulatory documents is inherently challenging due to the need for multihop reasoning across legally interdependent texts, a requirement that is particularly pronounced in the healthcare domain where regulations are hi...
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2602.07361v1) for detailed methodology, experimental results, and ablation studies.