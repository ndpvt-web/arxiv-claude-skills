---
name: "deepread-document-structure-aware-reasoning"
description: |
  Structure-aware document reasoning that converts PDFs/long documents into hierarchically indexed
  paragraphs with coordinate metadata, then uses a dual-tool "locate then read" strategy (Retrieve + ReadSection)
  to answer complex questions requiring evidence scattered across distant document sections.
  Trigger phrases:
  - "analyze this PDF and answer questions about it"
  - "find evidence across sections of this document"
  - "search this long document for specific information"
  - "extract structured answers from this report"
  - "answer questions about this paper/filing/manual"
  - "build a document QA pipeline"
---

# DeepRead: Structure-Aware Document Reasoning

This skill enables Claude to process long documents (PDFs, reports, filings, manuals, papers) by preserving their hierarchical structure — headings, sections, paragraph boundaries — rather than treating them as flat text. Using the DeepRead paradigm, Claude converts documents into structured Markdown with coordinate-style metadata (`doc_id`, `sec_id`, `para_idx`), then reasons over them with a two-phase "locate then read" strategy: first retrieving relevant paragraphs by semantic search with scanning context, then reading contiguous sections in order to synthesize accurate answers. This approach dramatically outperforms flat-chunking RAG when answers require integrating evidence from multiple distant document regions.

## When to Use

- When the user provides a PDF or long document and asks questions that require cross-referencing multiple sections (e.g., "How does the methodology in Section 3 relate to the results in Section 5?")
- When building a document QA system that must handle 100K+ token documents like financial filings, legal contracts, or technical manuals
- When the user needs to extract structured answers from hierarchically organized documents (reports with numbered sections, papers with headings)
- When a simple keyword search or single-chunk retrieval fails because the answer spans non-adjacent paragraphs
- When the user asks to "deeply read" or "analyze" a document rather than just search it
- When building an agentic pipeline that must iteratively refine its understanding of a long document through multiple retrieval rounds

## Key Technique

**The problem with flat chunking:** Standard RAG systems split documents into fixed-size overlapping chunks (e.g., 800 tokens with 400-token overlap), discarding the document's native structure. This means a retrieval hit in one chunk provides no information about where in the document it sits, what section it belongs to, or what adjacent content exists. For questions requiring multi-hop reasoning across distant sections, flat chunking forces the model to guess context.

**DeepRead's structural indexing:** Instead of arbitrary chunks, DeepRead indexes at paragraph granularity. Each paragraph receives a coordinate-style metadata key: `{doc_id: d, sec_id: i, para_idx: j}` where `d` identifies the document, `i` identifies the section (mapped from the heading hierarchy), and `j` is the paragraph's sequential position within that section. This metadata is cheap to store but enables two powerful operations: (1) the Retrieve tool can return not just matching paragraphs but their exact structural coordinates plus a scanning window of surrounding paragraphs, and (2) the ReadSection tool can fetch any contiguous range of paragraphs within a section by specifying `[j_start, j_end]`, preserving reading order.

**The "locate then read" loop:** DeepRead wraps these tools in a ReAct-style multi-turn agent loop (up to 50 rounds). The agent receives the document's table of contents in its system prompt, uses Retrieve to find candidate locations, examines structural coordinates to understand where evidence lives, then uses ReadSection to read broader context around hits. This mimics how a human expert skims a table of contents, jumps to relevant sections, reads surrounding paragraphs for context, and iterates until the answer is complete.

## Step-by-Step Workflow

1. **Convert the document to structured Markdown.** Parse the PDF (or other format) into Markdown that preserves heading hierarchy (`#`, `##`, `###`) and paragraph boundaries. Use an OCR/parsing tool (e.g., marker, pymupdf4llm, or an LLM-based parser) that retains structural elements rather than flattening to plain text.

2. **Build the section-paragraph coordinate index.** Walk the Markdown AST to identify sections (by heading level) and paragraphs within each section. Assign each paragraph a coordinate triple `(doc_id, sec_id, para_idx)`. Store the mapping from `sec_id` to heading text for the table of contents.

3. **Generate the table of contents (TOC).** Extract the heading hierarchy into a compact TOC string that lists section IDs alongside their titles and paragraph counts. This TOC will be injected into the system prompt so the agent knows the document's structure without reading the full text.

4. **Create paragraph-level embeddings.** Embed each paragraph using a dense retriever (e.g., sentence-transformers, OpenAI embeddings, or a dedicated model like Qwen3-embedding). Store embeddings alongside the coordinate metadata in a vector index (FAISS, ChromaDB, or similar).

5. **Implement the Retrieve tool.** Given a query string, perform semantic search to find the top-K most relevant paragraphs. For each hit at coordinate `(d, i, j)`, expand with a scanning window `W = (w_up, w_down)` to include paragraphs `[max(1, j - w_up), min(n_section, j + w_down)]`. Deduplicate overlapping windows. Return results sorted by `(doc_id, sec_id, para_idx)` with coordinates visible in the output.

6. **Implement the ReadSection tool.** Given a `doc_id`, `sec_id`, and paragraph range `[j_start, j_end]`, return the contiguous paragraphs in reading order, clipped to valid boundaries. This tool takes no query — it reads exactly what the agent asks for.

7. **Compose the agent system prompt.** Include: (a) the task description, (b) the full TOC with section IDs and paragraph counts, (c) tool descriptions for Retrieve and ReadSection with parameter specs, (d) instructions to use "locate then read" — first Retrieve to find candidates, then ReadSection to expand context.

8. **Run the multi-turn reasoning loop.** Execute a ReAct loop where the agent alternates between reasoning (thinking about what it knows and what it still needs) and tool calls. Cap at a maximum number of rounds (e.g., 15-50 depending on document complexity). The agent terminates by emitting a FINAL action with its answer.

9. **Post-process and validate the answer.** Extract the final answer, verify it references specific document locations (section and paragraph coordinates), and format it with citations pointing back to source coordinates.

## Concrete Examples

**Example 1: Financial Filing Analysis**

```
User: "I have a 10-K filing (200 pages). Does the company's risk factor
discussion about supply chain match what they report in the MD&A section
about actual supply chain disruptions?"

Approach:
1. Convert the 10-K PDF to structured Markdown preserving Item numbers
   (Item 1A: Risk Factors, Item 7: MD&A, etc.) as sections.
2. Index paragraphs with coordinates like:
   {doc_id: "10K_2025", sec_id: "item_1a", para_idx: 3}
   {doc_id: "10K_2025", sec_id: "item_7",  para_idx: 12}
3. Retrieve("supply chain risk") → hits in Item 1A paragraphs 3, 7, 15
   with scanning window showing surrounding context.
4. ReadSection(doc_id="10K_2025", sec_id="item_1a", j_start=1, j_end=20)
   to read the full supply chain risk subsection.
5. Retrieve("supply chain disruption impact revenue") → hits in Item 7
   paragraphs 12, 14.
6. ReadSection(doc_id="10K_2025", sec_id="item_7", j_start=10, j_end=18)
   to read surrounding MD&A discussion.
7. Synthesize: compare the stated risks against reported outcomes.

Output:
"The Risk Factors section (Item 1A, paras 3-8) identifies three supply chain
risks: semiconductor shortages, single-source dependencies, and logistics
delays. The MD&A section (Item 7, paras 12-16) reports that semiconductor
shortages reduced Q3 revenue by $45M, confirming Risk Factor #1. However,
the single-source dependency risk (Item 1A, para 7) is not addressed in
MD&A, suggesting either no materialization or incomplete disclosure."
```

**Example 2: Technical Manual Troubleshooting**

```
User: "Our deployment keeps failing at the TLS handshake step. Search the
800-page infrastructure manual for relevant configuration steps."

Approach:
1. Parse manual into structured Markdown. TOC reveals:
   sec_id: "ch4_networking" (Chapter 4: Networking)
   sec_id: "ch4_3_tls"      (Section 4.3: TLS Configuration)
   sec_id: "ch9_troubleshoot" (Chapter 9: Troubleshooting)
   sec_id: "appendix_b"      (Appendix B: Error Codes)
2. Retrieve("TLS handshake failure configuration") → hits in:
   - ch4_3_tls, para 5 (certificate chain setup)
   - ch9_troubleshoot, para 22 (handshake timeout errors)
   - appendix_b, para 8 (error code TLS_HANDSHAKE_FAILED)
3. ReadSection("manual", "ch4_3_tls", 1, 15) → full TLS config section.
4. ReadSection("manual", "ch9_troubleshoot", 20, 28) → troubleshooting
   steps around the hit.
5. Cross-reference: config step in para 5 requires setting
   `tls_min_version=1.2`, and troubleshooting para 23 notes that
   omitting this causes handshake failures with newer clients.

Output:
"Based on Section 4.3 (paras 5-7) and Chapter 9 (para 23):
1. Verify `tls_min_version` is set to 1.2+ in your config (Section 4.3, para 5)
2. Ensure the full certificate chain is provided, not just the leaf cert
   (Section 4.3, para 7)
3. If using mutual TLS, the client CA bundle path must be absolute
   (Chapter 9, para 23 — this is the most common cause of handshake failures)"
```

**Example 3: Building a Document QA Pipeline in Code**

```
User: "Build me a Python pipeline that indexes a PDF using the DeepRead
approach and answers questions about it."

Approach:
1. Write a document parser that converts PDF → structured Markdown
   (using pymupdf4llm or marker).
2. Implement coordinate indexing — walk the Markdown to extract sections
   and assign (sec_id, para_idx) to each paragraph.
3. Build a vector index over paragraphs with metadata.
4. Implement Retrieve and ReadSection as callable tool functions.
5. Wire into an LLM agent loop with the TOC in the system prompt.

Output (key code structure):

  # document_parser.py
  def pdf_to_structured_markdown(pdf_path: str) -> str: ...
  def extract_sections(markdown: str) -> list[Section]: ...
  def build_paragraph_index(sections: list[Section]) -> ParagraphIndex: ...

  # tools.py
  def retrieve(query: str, index: ParagraphIndex, top_k=5,
               window=(2, 2)) -> list[ParagraphHit]: ...
  def read_section(doc_id: str, sec_id: str, j_start: int,
                   j_end: int, index: ParagraphIndex) -> str: ...

  # agent.py
  def build_system_prompt(toc: str, tool_descriptions: str) -> str: ...
  def run_deepread_agent(question: str, index: ParagraphIndex,
                         max_rounds: int = 20) -> str: ...
```

## Best Practices

- **Do:** Preserve the original heading hierarchy faithfully during Markdown conversion. A mis-parsed heading collapses two sections into one, breaking all downstream coordinate references.
- **Do:** Include the full TOC in the system prompt. This is cheap (a few hundred tokens even for large documents) and gives the agent a map of the entire document without reading it all.
- **Do:** Use the scanning window in Retrieve (2-3 paragraphs up and down) to provide local context around hits. Isolated paragraphs often lack referential clarity (pronouns, abbreviations defined earlier).
- **Do:** Sort retrieval results by document order `(doc_id, sec_id, para_idx)` rather than relevance score. Reading in document order helps the agent follow the author's logic.
- **Avoid:** Falling back to flat chunking "for simplicity." The entire value of DeepRead is structural awareness — without coordinates, ReadSection becomes meaningless.
- **Avoid:** Setting the scanning window too large (e.g., 10+ paragraphs). This floods the context with irrelevant text and negates the precision benefit of paragraph-level indexing. Start with `W = (2, 2)` and increase only if recall is low.
- **Avoid:** Skipping deduplication when scanning windows of nearby hits overlap. Duplicate paragraphs waste context tokens and confuse the model.

## Error Handling

| Problem | Cause | Fix |
|---|---|---|
| Paragraphs have no section coordinates | PDF parser failed to detect headings (scanned PDF, non-standard formatting) | Fall back to page-level sectioning: treat each page as a "section" with paragraphs numbered sequentially |
| ReadSection returns empty | `j_start` exceeds actual paragraph count in section | Clip to valid range `[1, n_section]`; return the closest valid range with a note that the requested range was adjusted |
| Retrieve returns irrelevant hits | Embedding model struggles with domain-specific terminology | Add a reranking stage (cross-encoder) or prepend section titles to paragraph text before embedding to boost topical signal |
| Agent loops without converging | Question requires reasoning the LLM cannot perform, or evidence genuinely isn't in the document | Set a hard round limit (20-50); if the agent exhausts rounds, return the best partial answer with a confidence disclaimer |
| TOC is too large for system prompt | Document has hundreds of fine-grained subsections | Collapse the TOC to the top 2-3 heading levels; let Retrieve discover deeper subsections on demand |

## Limitations

- **Depends on heading quality:** Documents without clear heading structure (e.g., plain-text transcripts, chat logs, stream-of-consciousness writing) won't benefit from structural indexing. Fall back to standard chunked RAG for these.
- **OCR/parsing fidelity:** The coordinate system is only as reliable as the Markdown conversion. Complex layouts (multi-column, nested tables, marginalia) can produce garbled section boundaries.
- **Not for sub-paragraph precision:** If the answer is a single sentence within a paragraph, DeepRead's paragraph-level granularity works but doesn't provide sentence-level coordinates.
- **Multi-document scaling:** With many documents, the TOC in the system prompt can grow large. For corpora of 50+ documents, consider a two-stage approach: first select relevant documents, then apply DeepRead to the shortlist.
- **Latency:** The multi-turn agent loop requires several LLM calls per question. For latency-sensitive applications, consider caching retrieval results and limiting rounds.

## Reference

**Paper:** [DeepRead: Document Structure-Aware Reasoning to Enhance Agentic Search](https://arxiv.org/abs/2602.05014v2) — Li et al., 2026. Focus on Section 3 (method), especially the coordinate metadata schema `Gamma_{d,i,j}`, the Retrieve/ReadSection tool definitions, and Algorithm 1 for the full agent loop.