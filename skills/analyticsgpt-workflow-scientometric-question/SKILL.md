---
name: "analyticsgpt-workflow-scientometric-question"
description: "Build sequential LLM pipelines for scientometric question answering over academic databases. Decomposes meta-scientific queries into entity recognition, multi-step planning, parallel data retrieval, and analytical synthesis. Use when: 'build a scientometric QA system', 'answer questions about research impact', 'query academic publication databases with natural language', 'analyze citation metrics for institutions or authors', 'create a pipeline for science-of-science questions', 'implement RAG over scholarly metadata'."
---

# AnalyticsGPT: Sequential LLM Workflow for Scientometric Question Answering

This skill teaches Claude to build end-to-end LLM-powered pipelines that answer scientometric questions -- meta-scientific queries about research output, impact, collaboration patterns, and publication trends. The core technique from the AnalyticsGPT paper (Ly et al., 2026) is a **four-stage sequential workflow**: (1) high-level planning with named-entity recognition of academic entities, (2) detailed plan generation with dependency-aware tool calls, (3) parallel execution of data retrieval steps, and (4) analytical synthesis into structured, citation-rich responses. This approach handles the unique challenge of scientometric QA where questions reference specific authors, institutions, journals, and topics that must be resolved to database identifiers before any data can be retrieved.

## When to Use

- When the user wants to build a natural-language interface to an academic or bibliometric database (Scopus, OpenAlex, Semantic Scholar, DBLP, institutional research platforms)
- When implementing a multi-agent pipeline that decomposes complex research-analytics questions into executable sub-queries
- When the task requires named-entity recognition and resolution for academic entities (author names, institution names, journal titles, research topics) against a knowledge base
- When building RAG systems over structured scholarly metadata rather than unstructured paper text
- When the user needs to compare research performance across institutions, authors, countries, or time periods
- When generating analytical reports from raw scientometric data (h-index, impact factors, citation counts, collaboration networks)

## Key Technique

**The core insight** is that scientometric questions differ from standard QA because they require a *planning phase* that traditional retrieve-then-read pipelines skip. A question like "How does MIT's AI research output compare to Stanford's over the last decade?" cannot be answered with a single retrieval call. It requires: (a) recognizing "MIT" and "Stanford" as institution entities and "AI" as a topic, (b) resolving each to database identifiers, (c) planning separate but parallel data retrievals for each institution filtered by topic and year range, and (d) synthesizing comparative analysis from the results.

**The two-tier planning architecture** separates concerns cleanly. A lightweight *high-level planner* (can use a smaller/faster model) extracts named entities with their types and outlines broad steps. A more capable *detailed planner* then expands each broad step into specific tool calls with exact parameters, dependency ordering, and entity IDs injected from the resolution stage. This separation means the expensive model only sees well-structured input, and the entity resolution step bridges the gap between natural language references and database-ready identifiers.

**Dependency-aware parallel execution** is the third critical element. Each plan step declares which prior steps it depends on. Independent steps execute concurrently via thread pools, while dependent steps wait for their prerequisites. This gives significant speedup on multi-entity comparison questions without sacrificing correctness when one retrieval's output parameterizes another.

## Step-by-Step Workflow

### 1. Define the Entity Type Schema

Enumerate all entity types your academic database supports. At minimum:

```python
ENTITY_TYPES = [
    "author",        # Individual researchers ("Last, First Middle")
    "institution",   # Universities, labs, companies
    "journal",       # Publication venues
    "topic",         # Research areas / keyword clusters
    "subject_area",  # Broad disciplinary categories
    "country",       # National affiliations
]
```

### 2. Build the High-Level Planning Agent

Create a prompt that accepts a natural-language scientometric question and outputs structured JSON with two fields: `named_entities` (list of `{name, type}` objects) and `high_level_steps` (ordered list of broad actions). Use few-shot examples covering comparison queries, trend queries, and single-entity lookup queries.

```python
HIGH_LEVEL_PLANNER_SYSTEM = """You are an AI planning assistant for scientometric queries.
Given a natural language question about research performance, output JSON with:
1. "named_entities": extract all academic entities with their types
2. "high_level_steps": broad actions needed to answer the query

Entity types: {entity_types}
Format author names as "Last, First Middle".
Correct obvious typos in entity names."""

HIGH_LEVEL_PLANNER_EXAMPLES = [
    {
        "query": "Compare publication output of University of Oxford and ETH Zurich in materials science since 2018",
        "output": {
            "named_entities": [
                {"name": "University of Oxford", "type": "institution"},
                {"name": "ETH Zurich", "type": "institution"},
                {"name": "Materials Science", "type": "subject_area"}
            ],
            "high_level_steps": [
                "Retrieve publication counts for University of Oxford in Materials Science from 2018-present",
                "Retrieve publication counts for ETH Zurich in Materials Science from 2018-present",
                "Compare trends and summarize findings"
            ]
        }
    }
]
```

### 3. Implement Entity Resolution

Build a resolver that maps extracted entity names to database identifiers. This is the critical bridge between natural language and structured queries. Use your database's search/autocomplete API, GraphQL endpoint, or fuzzy-match index.

```python
class EntityResolver:
    def resolve(self, entity_name: str, entity_type: str) -> list[dict]:
        """Return candidate matches with IDs and confidence scores.
        Handle ambiguity by returning top-k candidates."""
        # Query your database's entity search endpoint
        # Return: [{"id": "inst_12345", "name": "University of Oxford", "score": 0.98}]
```

Run resolution for all extracted entities in parallel. Inject the resolved IDs into the context passed to the detailed planner.

### 4. Build the Detailed Planning Agent

This agent receives the high-level steps plus resolved entity IDs and produces an executable plan. Each step specifies: `tool_name`, `parameters` (using resolved IDs), `question` (what this step answers), and `depends_on` (list of prior step indices).

```python
DETAILED_PLANNER_SYSTEM = """You are a research assistant that decomposes questions into executable tool calls.

Available tools:
- article_search(filters, sort_by, limit): Retrieve articles matching filters
- article_facet_search(filters, facet_field, top_k): Aggregate articles by a facet (author, institution, country, etc.)

For each step output JSON: {
  "step": <int>, "tool": <str>, "question": <str>,
  "parameters": {...}, "depends_on": [<int>, ...]
}

Use the resolved entity IDs provided in context. Never use raw names as parameters."""
```

### 5. Execute Plan Steps with Dependency Ordering

Implement a scheduler that respects `depends_on` declarations. Steps with no dependencies (or whose dependencies are satisfied) run in parallel. Pass results from completed steps as context to dependent steps.

```python
def execute_plan(steps, tool_registry):
    results = {}
    with ThreadPoolExecutor(max_workers=4) as executor:
        pending = list(steps)
        while pending:
            ready = [s for s in pending if all(d in results for d in s["depends_on"])]
            futures = {
                executor.submit(run_step, s, results, tool_registry): s
                for s in ready
            }
            for future in as_completed(futures):
                step = futures[future]
                results[step["step"]] = future.result()
                pending.remove(step)
    return results
```

### 6. Build an Action Agent for Adaptive Tool Calling

For each plan step, an action agent evaluates whether the suggested tool call is sufficient or needs modification based on conversation history. It avoids redundant calls by checking if the needed data was already retrieved in a prior step.

### 7. Build the Writing/Synthesis Agent

This agent transforms raw data into structured analytical output. Its prompt should enforce:
- Lead with the most compelling insight, not raw numbers
- Use markdown tables for comparative data
- Cite entities with a consistent bracket notation: `[Institution: Name]`, `[Author: Name]`
- Acknowledge data gaps explicitly rather than hallucinating
- Restrict all claims to the retrieved data

```python
WRITER_SYSTEM = """You are a strategic research writing assistant.
Transform raw scientometric query results into insightful analysis.

Rules:
- Lead with key findings and implications
- Use markdown tables for comparisons
- Cite all entities with bracket notation
- State timeframes explicitly
- If data is insufficient, say so clearly
- Use gender-neutral pronouns for authors"""
```

### 8. Wire the Full Pipeline

Connect all stages in sequence, passing outputs forward:

```python
class ScientometricQAPipeline:
    def run(self, query: str) -> str:
        # Stage 1: High-level plan + entity extraction
        high_level = self.high_level_planner.invoke(query)

        # Stage 2: Entity resolution (parallel)
        resolved = self.resolver.resolve_all(high_level["named_entities"])

        # Stage 3: Detailed plan with resolved IDs
        plan = self.detailed_planner.invoke(query, high_level, resolved)

        # Stage 4: Execute tool calls (parallel where possible)
        results = self.executor.run(plan["steps"])

        # Stage 5: Synthesize analytical response
        response = self.writer.invoke(query, results)
        return response
```

### 9. Implement LLM-as-Judge Evaluation

For evaluating answer quality, use a multi-judge approach: have multiple LLM instances score responses on relevance, completeness, accuracy, and presentation. Average the scores and flag disagreements for human review.

### 10. Add a Naive Fallback Path

For simple single-entity lookups ("What is the h-index of Author X?"), implement a lightweight path that skips the two-tier planning and goes directly from NER to a single tool call to synthesis. Route queries based on complexity detected in the high-level planning stage.

## Concrete Examples

**Example 1: Institutional Comparison Query**

```
User: "How does the University of Cambridge compare to Imperial College London
in terms of AI publications and citation impact over the last 5 years?"

Approach:
1. High-level planner extracts:
   - Entities: [{name: "University of Cambridge", type: "institution"},
                {name: "Imperial College London", type: "institution"},
                {name: "Artificial Intelligence", type: "topic"}]
   - Steps: [retrieve Cambridge AI pubs, retrieve Imperial AI pubs, compare]

2. Entity resolver maps:
   - "University of Cambridge" -> inst_id: 60000356
   - "Imperial College London" -> inst_id: 60015012
   - "Artificial Intelligence" -> topic_id: T.45

3. Detailed planner generates:
   Step 1: article_facet_search(institution=60000356, topic=T.45, years=2021-2026, facet="year") [depends: none]
   Step 2: article_facet_search(institution=60015012, topic=T.45, years=2021-2026, facet="year") [depends: none]
   Step 3: article_search(institution=60000356, topic=T.45, years=2021-2026, sort="citations") [depends: none]

4. Steps 1-3 execute in parallel (no dependencies).

5. Writer synthesizes:

Output:
## AI Research: Cambridge vs Imperial (2021-2026)

| Year | [Institution: Cambridge] | [Institution: Imperial] |
|------|--------------------------|-------------------------|
| 2021 | 342 publications         | 289 publications        |
| 2022 | 401 publications         | 318 publications        |
| ...  | ...                      | ...                     |

**Key Finding**: Cambridge has maintained a ~20% higher publication volume
in AI, but Imperial shows stronger citation impact per paper (avg 12.3 vs 10.1)...
```

**Example 2: Author Trend Analysis**

```
User: "What topics has Yoshua Bengio published on most in the last 3 years,
and how do his citation patterns compare to his career average?"

Approach:
1. High-level planner extracts:
   - Entities: [{name: "Bengio, Yoshua", type: "author"}]
   - Steps: [get recent topic distribution, get career citation stats, compare]

2. Entity resolver maps "Bengio, Yoshua" -> author_id: 7004326836

3. Detailed planner generates:
   Step 1: article_facet_search(author=7004326836, years=2023-2026, facet="topic") [depends: none]
   Step 2: article_facet_search(author=7004326836, facet="year") [depends: none]
   Step 3: article_search(author=7004326836, years=2023-2026, sort="citations", limit=10) [depends: none]

4. All steps execute in parallel.

5. Writer produces topic breakdown table, career citation trend chart data,
   and comparative analysis noting shifts toward AI safety topics.
```

**Example 3: Simple Single-Entity Lookup (Naive Path)**

```
User: "How many papers has Nature published in 2025?"

Approach (naive fallback - no multi-step planning needed):
1. NER: [{name: "Nature", type: "journal"}]
2. Resolve: "Nature" -> journal_id: 21206
3. Single call: article_search(journal=21206, year=2025, count_only=True)
4. Writer: "Nature published 4,312 articles in 2025."
```

## Best Practices

**Do:**
- Normalize author names to "Last, First Middle" format before entity resolution -- the high-level planner should handle this, but verify in post-processing
- Return top-k entity resolution candidates with confidence scores, and let the detailed planner choose when ambiguity exists (e.g., "J. Smith" matching many authors)
- Log the full plan and tool responses for each session to enable debugging and evaluation
- Use a smaller, faster model (e.g., GPT-4o-mini, Haiku) for the high-level planner and writer stages, reserving the most capable model for the detailed planner where reasoning complexity is highest

**Avoid:**
- Passing raw entity names directly to database queries -- always resolve to IDs first. Name variations and typos will cause silent failures
- Letting the planner generate tool calls for tools that don't exist. Constrain output with Pydantic schemas or function-calling tool definitions
- Assuming plan steps are always independent -- always implement dependency tracking even if most queries don't need it, because comparison queries always do
- Stuffing all retrieved data into the writer's context. Summarize large result sets before synthesis to stay within context limits

## Error Handling

| Failure Mode | Detection | Recovery |
|---|---|---|
| Entity not resolved | Resolver returns empty or low-confidence matches | Ask user for clarification; suggest closest matches |
| Ambiguous entity | Multiple high-confidence matches for one name | Include disambiguation in writer output, or ask user |
| Tool call returns empty data | Zero results from database query | Writer should state "No data found for [Entity] in the specified period" rather than hallucinate |
| Dependency cycle in plan | Cycle detection during scheduling | Reject plan, re-invoke detailed planner with explicit acyclicity instruction |
| Rate limiting on database API | HTTP 429 or timeout | Exponential backoff with jitter; reduce parallelism |
| Plan step produces unexpected schema | Pydantic validation failure on tool parameters | Log error, skip step, let writer note incomplete data |

## Limitations

- **Entity resolution is the bottleneck**: The entire pipeline fails if entities cannot be mapped to database IDs. Databases without good search/autocomplete APIs will require building a separate entity index.
- **Not for open-ended paper QA**: This workflow answers questions *about* research (metrics, trends, comparisons), not questions *from* research papers. For content-based QA over paper text, use standard RAG.
- **Database-dependent**: The tool definitions and plan structures are tightly coupled to your specific database schema. Switching from OpenAlex to Scopus requires rewriting tool definitions and replanning prompts.
- **Complex nested queries degrade**: Questions requiring 3+ levels of dependency chaining (e.g., "Find the top collaborator of the most-cited author in the top institution in field X") push planning quality down. Consider limiting plan depth.
- **Temporal reasoning is fragile**: LLMs often miscalculate date ranges. Validate year parameters in tool calls programmatically rather than trusting the planner.

## Reference

Ly, K., Cheirmpos, G., Raudaschl, A., James, C., & Tabatabaei, S. A. (2026). *AnalyticsGPT: An LLM Workflow for Scientometric Question Answering*. arXiv:2602.09817. [https://arxiv.org/abs/2602.09817v1](https://arxiv.org/abs/2602.09817v1)

Key sections to study: the two-tier planning architecture (high-level vs. detailed), the entity resolution bridge between NER and database queries, and the dependency-aware parallel execution model. Skeleton code and prompts: [https://github.com/lyvykhang/llm-agents-scientometric-qa/tree/acl](https://github.com/lyvykhang/llm-agents-scientometric-qa/tree/acl)