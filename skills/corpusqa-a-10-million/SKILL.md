---
name: "corpusqa-a-10-million"
description: "While large language models now handle million-token contexts, their capacity for reasoning across entire document repositories remains largely untested. Implements techniques from 'CorpusQA: A 10 Million Token Benchmark for Corpus-Level Analysis and Reasoning'. Use for tasks involving: data processing, search retrieval, agent framework. Triggers: \"Parse this CSV and...\", \"Extract data from this PDF\", \"Find information about...\", \"Search the codebase for...\", \"Build a pipeline that...\", \"Coordinate multiple tasks to...\""
---

# CorpusQA: A 10 Million Token Benchmark for Corpus-Level Analysis and Reasoning

You are a data processing specialist. You extract, clean, transform, and validate data from any source format.

**Paper:** [2601.14952v1](https://arxiv.org/abs/2601.14952v1) | **Category:** cs.CL | **Published:** 2026-01-21
**Authors:** Zhiyuan Lu, Chenliang Li, Yingcheng Shi, Weizhou Shen, Ming Yan

## Research Context

> While large language models now handle million-token contexts, their capacity for reasoning across entire document repositories remains largely untested. Existing benchmarks are inadequate, as they are mostly limited to single long texts or rely on a "sparse retrieval" assumption-that answers can be derived from a few relevant chunks. This assumption fails for true corpus-level analysis, where evidence is highly dispersed across hundreds of documents and answers require global integration, comparison, and statistical aggregation. To address this critical gap, we introduce CorpusQA, a new benchmark scaling up to 10 million tokens, generated via a novel data synthesis framework. By decoupling reasoning from textual representation, this framework creates complex, computation-intensive queries with programmatically guaranteed ground-truth answers, challenging systems to perform holistic reasoning over vast, unstructured text without relying on fallible human annotation. We further demonstrate the utility of our framework beyond evaluation, showing that fine-tuning on our synthesized data effectively enhances an LLM's general long-context reasoning capabilities. Extensive experiments reveal that even state-of-the-art long-context LLMs struggle as input length increases, and standard retrieval-augmented generation systems collapse entirely. Our findings indicate that memory-augmented agentic architectures offer a more robust alternative, suggesting a critical shift is needed from simply extending context windows to developing advanced architectures for global information synthesis.

## Key Techniques from This Paper

- Novel: benchmark scaling up

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

1. **Understand the problem context** -- The paper addresses while large language models now handle million-token contexts, their capacity for reasoning across entire document repositories remains largely untested.
2. **Adapt to the user's specific needs** -- The paper's approach may need tailoring for the particular codebase, language, or domain
3. **Combine with existing tools** -- Use this skill's techniques alongside Claude's built-in capabilities (file reading, code execution, web search)
4. **Iterate and refine** -- Apply the technique, evaluate results, and refine the approach based on feedback

## Limitations

- This skill encodes the *approach* from the paper, not a direct implementation of its trained model
- Results may vary based on the complexity and domain of the specific task
- For tasks outside the paper's scope, fall back to general-purpose reasoning

Refer to the [full paper](https://arxiv.org/abs/2601.14952v1) for detailed methodology, experimental results, and ablation studies.