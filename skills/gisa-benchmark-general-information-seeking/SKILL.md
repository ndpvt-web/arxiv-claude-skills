---
name: "gisa-benchmark-general-information-seeking"
description: >
  Build structured information-seeking agents that decompose complex queries into
  multi-turn search-and-browse workflows, aggregate results from multiple web sources,
  and return answers in typed structured formats (items, sets, lists, tables).
  Applies the GISA benchmark's ReAct-based agent architecture and evaluation methodology.
  Trigger phrases: "build an information-seeking agent", "search agent pipeline",
  "multi-turn web research agent", "structured web search workflow",
  "aggregate information from multiple sources", "web research with structured output"
---

# GISA: Structured Multi-Turn Information-Seeking Agent Framework

This skill enables Claude to build and operate information-seeking agents that go beyond single-query search. Based on the GISA benchmark methodology, it teaches how to decompose complex information needs into iterative search-and-browse cycles, aggregate findings from many web pages, and return results in strictly typed structured formats (single items, unordered sets, ordered lists, or multi-column tables). The core insight: real information-seeking requires both **deep reasoning** (multi-hop navigation to locate specific facts) and **wide aggregation** (synthesizing data scattered across many sources) -- and the agent must plan which strategy to use at each step.

## When to Use

- When building an agent that must answer questions requiring data from multiple web pages (e.g., "List all Nobel Prize winners in Physics from 2015-2024 with their nationalities and discoveries")
- When the user needs a ReAct-style search agent with structured tool use (search + browse) and typed output formatting
- When designing evaluation harnesses for web research agents, including metrics for sets, lists, and tables
- When a task requires iterating between issuing search queries, selecting URLs to browse, extracting facts, and deciding whether enough information has been collected
- When the user wants to aggregate scattered web data into a normalized table with specific columns and sort orders
- When building pipelines that must distinguish between stable facts and time-sensitive live data

## Key Technique

The GISA framework uses a **ReAct (Reason + Act) agent loop** with exactly two tools: a **Search tool** (issues queries to a search API and receives ranked results) and a **Browse tool** (fetches a URL's content and summarizes it). The agent iterates through cycles of reasoning about what information is still missing, formulating targeted search queries, selecting promising URLs from search results, browsing those pages, and extracting structured facts. The critical architectural constraint is a fixed budget (e.g., 30 tool invocations), forcing the agent to plan efficiently rather than exhaustively crawling.

What distinguishes this approach from naive search is the **answer-type-aware planning**. Before searching, the agent classifies the expected output as one of four types -- item (single value), set (unordered collection), list (ordered sequence with explicit sort criteria), or table (multi-column schema with defined columns and tie-breaking rules). This classification drives the search strategy: items need precise targeted queries; sets need breadth-first coverage; lists need both coverage and ordering verification; tables need systematic column-by-column population across many rows.

The GISA error analysis reveals that **49.2% of failures occur at the search level** -- poor query formulation, failure to follow hyperlinks within pages, and inability to resolve conflicting information across sources. The key operational lesson: agents should issue fewer but more targeted queries (humans average 3.5 queries vs. agents' 7.6), browse more pages per query (humans browse 19 pages vs. agents' 4.6), and aggressively follow in-page hyperlinks rather than returning to search.

## Step-by-Step Workflow

1. **Classify the answer type.** Analyze the user's information need and determine the output format: `item` (single fact), `set` (unordered collection), `list` (ordered sequence -- identify the sort criterion), or `table` (define the column schema and any tie-breaking/sort rules). Write this classification explicitly before proceeding.

2. **Decompose the query into sub-questions.** Break the information need into atomic search tasks. For tables, each column may require separate searches. For lists, identify both the membership criterion (what belongs in the list) and the ordering criterion (how to sort). For sets, enumerate the categories of items to collect.

3. **Formulate the first search query.** Write a precise, keyword-optimized search query targeting the highest-priority sub-question. Prefer specific entity names, date ranges, and domain-specific terms over natural language questions. Avoid overly broad queries.

4. **Process search results and select URLs.** From the search API response (typically top-10 results), select 2-4 most promising URLs based on domain authority, snippet relevance, and likelihood of containing structured data (e.g., Wikipedia tables, official databases, ranking sites). Prioritize authoritative primary sources over aggregator blogs.

5. **Browse and extract structured facts.** Fetch each selected URL, parse the content, and extract facts relevant to the query. Store extracted data in a running structured format (JSON objects or TSV rows). Track which cells/fields are still empty and which sources conflict.

6. **Resolve conflicts and verify.** When sources disagree, issue targeted verification queries (e.g., add "site:official-source.org" or search for the specific conflicting fact). Apply a recency heuristic: for live data, prefer the most recently updated source.

7. **Assess completeness and iterate.** Check the accumulated data against the answer schema. Identify missing rows, empty cells, or unverified ordering. If gaps remain and the tool budget allows, return to step 3 with refined queries targeting the missing information specifically.

8. **Follow in-page links aggressively.** When a browsed page contains hyperlinks to related entities (e.g., a list page linking to individual entries), follow those links rather than issuing new search queries. This mirrors the human browsing pattern that GISA found correlates with higher accuracy.

9. **Normalize the output.** Apply consistent formatting: lowercase column headers, strip currency symbols and commas from numbers, round floats to a consistent precision, normalize strings to lowercase for comparison, represent missing values as empty strings. Format tables as TSV with a header row.

10. **Validate and emit the final answer.** Verify the output matches the declared answer type. For lists, confirm the sort order. For tables, confirm all required columns are present and rows are ordered correctly. Emit the answer in the specified structured format.

## Concrete Examples

**Example 1: Table query -- aggregate scattered data**

```
User: "Find the top 10 highest-grossing films of 2024 worldwide, with columns
for rank, title, worldwide gross, domestic gross, and distributor."

Approach:
1. Classify answer type: TABLE with schema [rank, title, worldwide_gross,
   domestic_gross, distributor], sorted by worldwide_gross descending.
2. Search query 1: "highest grossing films 2024 worldwide box office"
3. Browse Box Office Mojo and The Numbers -- extract top 10 with worldwide
   gross and domestic gross figures.
4. For any missing distributor fields, search: "[film title] 2024 distributor
   production company"
5. Cross-verify gross figures between Box Office Mojo and The Numbers.
   Resolve discrepancies by preferring Box Office Mojo (primary source).
6. Normalize: strip "$" and commas from gross figures, lowercase headers.

Output:
rank	title	worldwide_gross	domestic_gross	distributor
1	inside out 2	1696200000	652900000	walt disney studios
2	deadpool & wolverine	1338500000	636700000	walt disney studios
3	moana 2	1125700000	449600000	walt disney studios
...
```

**Example 2: List query -- ordered sequence with sort criterion**

```
User: "List the 5 most-spoken languages in the world by total number of
speakers (native + non-native), in descending order."

Approach:
1. Classify answer type: LIST, sort criterion = total speakers descending.
2. Search query: "most spoken languages world total speakers 2024 ethnologue"
3. Browse Ethnologue or Wikipedia's "List of languages by total number of
   speakers" page.
4. Extract language names and total speaker counts.
5. Verify ordering: confirm English > Mandarin in total speakers (native +
   L2) or vice versa by checking a second source.
6. Normalize: lowercase language names, represent speaker counts as integers.

Output:
1. english
2. mandarin chinese
3. hindi
4. spanish
5. french
```

**Example 3: Set query -- unordered collection**

```
User: "What countries have successfully landed a spacecraft on the Moon?"

Approach:
1. Classify answer type: SET (unordered, membership-only).
2. Search query: "countries successful moon landing spacecraft"
3. Browse NASA page and Wikipedia "Moon landing" article.
4. Extract country names. Follow in-page links to verify recent missions
   (e.g., India's Chandrayaan-3, Japan's SLIM).
5. Cross-verify with a second source to ensure no country is missed.
6. Normalize: lowercase country names.

Output:
{soviet union, united states, china, india, japan}
```

## Best Practices

**Do:**
- Always declare the answer type and schema before starting any search. This prevents aimless browsing and focuses every tool invocation.
- Issue fewer, more specific search queries rather than many broad ones. Targeted queries like `"2024 Nobel Physics winner nationality"` outperform `"Nobel Prize 2024"`.
- Follow hyperlinks within pages you are already browsing. A Wikipedia list page linking to individual entries is more efficient than issuing separate searches for each entry.
- Track your tool invocation budget explicitly. Reserve at least 3-5 invocations for verification and conflict resolution at the end.

**Avoid:**
- Do not issue a new search query for every row in a table. Instead, find aggregation pages (Wikipedia lists, ranking databases) that contain multiple rows, then browse individual pages only for missing cells.
- Do not trust a single source for ordered lists or numerical rankings. Ranking discrepancies are the most common error category -- always cross-verify ordering with at least two sources.
- Do not skip normalization. Inconsistent formatting (e.g., "United States" vs "USA" vs "US") causes silent evaluation failures even when the content is correct.
- Do not ignore conflicting information. When two sources disagree, explicitly reason about which source is more authoritative or recent rather than picking arbitrarily.

## Error Handling

| Error Type | Frequency | Mitigation |
|---|---|---|
| **Ineffective query formulation** | ~14% of failures | Reformulate with more specific terms, entity names, date qualifiers, or site-specific operators. If first query returns irrelevant results, do not repeat it -- change the query structure. |
| **Failure to follow hyperlinks** | ~18% of failures | When a browsed page references related pages (e.g., "see also", linked entity names), follow those links directly rather than returning to search. |
| **Conflicting information across sources** | ~18% of failures | Establish a source hierarchy before searching (e.g., official > Wikipedia > news > blog). When conflicts arise, search for the specific disputed fact with a site-restricted query. |
| **Extraction errors** | ~16% of failures | After extracting data, re-read the source passage to verify the extraction. Pay special attention to table cell alignment and header-to-value mapping. |
| **Instruction-following errors** | ~32% of failures | Re-read the output format requirements before emitting the final answer. Verify column order, sort direction, and delimiter format match the specification exactly. |

## Limitations

- **Single-page answers defeat this approach.** If the information exists on one authoritative page, a simple search suffices -- the multi-turn aggregation workflow adds unnecessary complexity.
- **Real-time or rapidly changing data.** The search-browse-verify cycle has latency. For data that changes within minutes (stock prices, live scores), this workflow cannot guarantee freshness.
- **Requires search API access.** The ReAct agent loop depends on programmatic search (e.g., Serper, SerpAPI, Brave Search). Without a search API, the workflow degrades to manual URL construction.
- **Budget-constrained accuracy.** With a 30-invocation cap, complex tables (20+ rows, 5+ columns) may have incomplete cells. The agent must triage which cells to verify and which to leave best-effort.
- **Best model accuracy is ~19% exact match on GISA.** This workflow improves systematic information gathering, but perfect accuracy on complex aggregation tasks remains an open problem. Expect diminishing returns on queries requiring 10+ sources.

## Reference

**Paper:** [GISA: A Benchmark for General Information-Seeking Assistant](https://arxiv.org/abs/2602.08543v1) (Zhu et al., 2026)

**Key takeaway:** Read Section 4 (Error Analysis) and Table 5 for the breakdown of where search agents fail -- 49% of errors are at the search level (bad queries, missed hyperlinks, unresolved conflicts), and 47% are at the output level (extraction and formatting mistakes). The human trajectory analysis in Section 3.3 shows that fewer queries + more browsing per query is the optimal strategy.