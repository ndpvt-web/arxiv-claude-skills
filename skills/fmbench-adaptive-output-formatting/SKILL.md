---
name: "fmbench-adaptive-output-formatting"
description: |
  Adaptive Markdown output formatting that balances semantic fidelity with structural correctness.
  Applies the FMBench methodology to detect and fix broken lists, malformed tables, inconsistent
  headings, and invalid code blocks in LLM-generated Markdown. Use this skill when asked to:
  - "Format this output as clean Markdown"
  - "Fix the formatting in this document"
  - "Generate a well-structured report with tables and code blocks"
  - "Convert this content into properly nested Markdown"
  - "Clean up this Markdown — the lists and headings are broken"
  - "Reformat this response so it renders correctly"
---

# FMBench: Adaptive Large Language Model Output Formatting

This skill teaches Claude to produce and repair Markdown output that is both semantically faithful to the intended content and structurally correct according to CommonMark specification. Drawing from the FMBench framework (Wang, Zhou & Ding, 2026), the core insight is that formatting errors in LLM output — broken nested lists, malformed tables, inconsistent heading hierarchies, unbalanced code fences — are not cosmetic annoyances but structural defects that break downstream rendering, tool integration, and automated parsing. The skill implements a systematic validation-and-correction pipeline that treats semantic preservation and structural correctness as dual objectives to be balanced, not traded off.

## When to Use

- When generating long-form Markdown output containing mixed content types (prose, lists, tables, code blocks, blockquotes) that must render correctly
- When a user provides broken or inconsistently formatted Markdown and asks you to fix it
- When building structured documents like reports, READMEs, changelogs, or API documentation that will be consumed by parsers or rendered in browsers
- When producing output that will feed into tool-augmented pipelines (e.g., documentation generators, static site builders, Notion imports) where structural errors cause failures
- When reformatting content from one structure to another (e.g., flat text to nested outline, raw data to Markdown table) and the target format has strict constraints
- When a user specifies explicit layout constraints ("use H2 for sections, H3 for subsections, include a table of contents")

## Key Technique

**The Semantic-Structural Dual Objective.** FMBench reveals that LLM outputs suffer from an inherent tension: optimizing for content accuracy (semantic fidelity) can produce structurally malformed Markdown, while enforcing rigid structural templates can distort or omit content. The paper measures this with two independent scores — a semantic score (BERTScore-F1 between output and reference content) and a structure score (similarity of abstractive structural summaries). The practical takeaway: never sacrifice content to fix formatting, and never ignore formatting to preserve content. Both must be addressed in sequence.

**Atomic Unit Segmentation + Structure-Constrained Construction.** The FMBench pipeline works by first decomposing content into atomic units (individual headings, sentences, metadata fields, code snippets) using conservative rule-based splitting. It then consumes these units strictly in their original order while fitting them into a target Markdown structure specification. This "content-order-preserving" constraint prevents the common failure mode where reformatting scrambles or silently drops content. The structural specification defines section hierarchy depth, permitted block types, and nesting rules — all determined before content placement begins.

**Complexity-Aware Validation.** The paper finds that formatting difficulty is primarily driven by list nesting depth and item count — not by section count or blockquote usage. This means validation effort should concentrate on nested lists and tables (the highest error-rate structures) rather than spreading attention uniformly across all Markdown elements.

## Step-by-Step Workflow

1. **Audit the input content for atomic units.** Identify every discrete content element: headings, paragraph sentences, list items, table rows, code blocks, blockquotes, and metadata. Do not merge or reorder these units — preserve their original sequence as a strict constraint.

2. **Define the target structural specification.** Before writing any Markdown, establish the document skeleton: heading hierarchy depth (H1 through H4 max), which block types are permitted (ordered lists, unordered lists, tables, fenced code blocks, blockquotes), and nesting limits (e.g., lists nested no more than 3 levels deep).

3. **Validate heading hierarchy consistency.** Ensure headings follow a monotonically descending structure without skipping levels. An H1 must precede any H2; an H2 must precede any H3. Never jump from H1 to H3. If the input violates this, insert or adjust intermediate headings.

4. **Normalize list structures.** For each list, verify: (a) consistent marker usage (all `-` or all `*` for unordered; all `1.` for ordered), (b) correct indentation at each nesting level (4 spaces or 1 tab per level), (c) blank lines before and after the list block, (d) no orphaned list items that break the parent-child nesting tree.

5. **Validate table formatting.** Ensure every table has: (a) a header row, (b) a separator row with `---` alignment markers matching column count, (c) consistent column count across all rows, (d) pipe characters `|` at start and end of every row. Pad cells to equalize column widths for readability.

6. **Verify code block integrity.** Check that every opening ` ``` ` fence has a matching closing fence. Validate that language identifiers are present and correct (e.g., ` ```python ` not ` ```py `). Ensure no content outside the fences is accidentally captured inside, and no code inside the fences leaks out.

7. **Enforce blank line boundaries.** Insert blank lines before and after every block-level element (headings, lists, tables, code blocks, blockquotes). This is the single most common source of rendering failures in mixed-content Markdown — a missing blank line before a list causes it to merge with the preceding paragraph.

8. **Perform a semantic preservation check.** After structural correction, compare the reformatted output against the original content. Verify that no sentences, list items, table rows, or code snippets were dropped, duplicated, or reordered. Content fidelity is non-negotiable.

9. **Stress-test nested structures.** Focus validation effort on the highest-risk elements: deeply nested lists (3+ levels) and complex tables (5+ columns or merged conceptual groups). These are where formatting errors concentrate according to FMBench's difficulty analysis.

10. **Render-test mentally or explicitly.** Walk through the output as a CommonMark parser would. Check that each element would render as intended in a standard Markdown viewer. If any ambiguity exists, add explicit structural cues (blank lines, indentation) rather than relying on parser leniency.

## Concrete Examples

**Example 1: Fixing a broken nested list**

User: "Fix the formatting in this Markdown — the nested list isn't rendering right"

```markdown
## Features
- Authentication
  - OAuth 2.0 support
  - JWT token management
    - Access tokens
    - Refresh tokens
  - Session handling
- Database
- PostgreSQL with connection pooling
- Redis caching layer
- API Layer
  - REST endpoints
  - GraphQL schema
```

Approach:
1. Segment into atomic units: 3 top-level items (Authentication, Database, API Layer) with nested children
2. Identify the structural break: "PostgreSQL with connection pooling" and "Redis caching layer" lost their indentation as children of "Database"
3. Restore nesting, preserve original content order, add consistent indentation

Output:
```markdown
## Features

- Authentication
    - OAuth 2.0 support
    - JWT token management
        - Access tokens
        - Refresh tokens
    - Session handling
- Database
    - PostgreSQL with connection pooling
    - Redis caching layer
- API Layer
    - REST endpoints
    - GraphQL schema
```

**Example 2: Generating a well-structured report with mixed content**

User: "Write a deployment checklist as Markdown with a summary table, nested task lists, and code examples"

Approach:
1. Define structure specification: H1 title, H2 sections, one summary table, nested unordered lists, fenced code blocks with language tags
2. Build the skeleton before filling content
3. Validate blank lines between every block-level element
4. Verify table column consistency and code fence closure

Output:
```markdown
# Deployment Checklist

## Summary

| Phase       | Tasks | Owner    | Status  |
| ----------- | ----- | -------- | ------- |
| Pre-deploy  | 4     | Backend  | Pending |
| Deploy      | 3     | DevOps   | Pending |
| Post-deploy | 2     | QA       | Pending |

## Pre-deploy Tasks

- Run test suite
    - Unit tests
    - Integration tests
    - Load tests (threshold: p99 < 200ms)
- Database migrations
    - Back up production database
    - Run migrations in staging first

```bash
./scripts/run-migrations.sh --env staging --dry-run
```

- Environment variable audit
- Cache warm-up configuration

## Deploy

- Tag release:

```bash
git tag -a v2.4.0 -m "Release 2.4.0" && git push origin v2.4.0
```

- Execute rolling deployment
- Monitor error rate dashboard for 15 minutes

## Post-deploy

- Smoke test critical user flows
- Confirm monitoring alerts are active
```

**Example 3: Reformatting a malformed table**

User: "This table is broken, fix it"

```markdown
| Name | Role | Team
| --- | ---
Alice | Engineer | Platform |
| Bob | Designer | Product
| Carol | PM | Platform |
```

Approach:
1. Count the maximum column span: 3 columns (Name, Role, Team)
2. Fix missing pipes at row boundaries
3. Add missing separator column to match header count
4. Normalize every row to exactly 3 pipe-delimited cells

Output:
```markdown
| Name  | Role     | Team     |
| ----- | -------- | -------- |
| Alice | Engineer | Platform |
| Bob   | Designer | Product  |
| Carol | PM       | Platform |
```

## Best Practices

**Do:**
- Always define the document structure skeleton (heading hierarchy, block types, nesting limits) before placing content — structure-first prevents drift
- Preserve content order strictly; never silently reorder, merge, or drop content units during reformatting
- Concentrate validation effort on nested lists and tables — these account for the majority of rendering failures
- Insert blank lines before and after every block-level element without exception

**Avoid:**
- Over-regularizing output into rigid templates that distort the original content's meaning or omit nuanced detail
- Skipping heading levels (H1 to H3 directly) even when it seems aesthetically fine — parsers and accessibility tools depend on sequential hierarchy
- Using inconsistent list markers within the same list tree (mixing `-` and `*`, or mixing `1.` and `-` at the same nesting level)
- Assuming the output "looks fine" without mentally parsing it as a CommonMark renderer would — visual inspection misses invisible structural errors like missing blank lines

## Error Handling

**Ambiguous nesting:** When input content has ambiguous parent-child relationships in lists, default to the shallowest valid nesting. Ask the user for clarification rather than guessing deep nesting that may misrepresent the hierarchy.

**Column count mismatch in tables:** When table rows have inconsistent cell counts, pad shorter rows with empty cells rather than truncating longer rows. Never silently drop data.

**Unclosed code fences:** If a code block lacks a closing fence, close it at the next heading or end-of-document boundary. Flag this to the user — an unclosed fence typically indicates content that was accidentally captured inside the block.

**Content loss during reformatting:** If any atomic unit from the original cannot be placed within the target structure without distortion, preserve it as-is in a dedicated section and flag the conflict rather than silently dropping it. Semantic fidelity takes precedence over structural perfection.

**Heading hierarchy violations in source:** When the input has heading level jumps (e.g., H1 directly to H4), insert corrective intermediate headings only if the user approves. Otherwise, normalize to the nearest valid level while noting the adjustment.

## Limitations

- This skill addresses CommonMark-compatible Markdown only. Extended syntaxes (GitHub Flavored Markdown extensions like task lists, footnotes, or math blocks) may require additional validation beyond the core pipeline.
- The technique is structural, not semantic — it catches formatting errors but does not evaluate whether the content itself is correct, complete, or well-organized.
- Very large documents (100+ sections, deeply nested structures) benefit from incremental validation rather than a single pass; the cognitive load of tracking all structural constraints simultaneously increases error risk.
- The approach assumes content order is meaningful and must be preserved. For content where reordering is desired (e.g., alphabetical sorting), the strict order-preservation constraint must be explicitly relaxed.
- Formatting corrections cannot fix fundamentally ambiguous source material where the intended structure is unknowable without user input.

## Reference

**Paper:** [FMBench: Adaptive Large Language Model Output Formatting](https://arxiv.org/abs/2602.06384v1) — Wang, Zhou & Ding (2026). Focus on Section 3 (pipeline architecture and atomic unit segmentation), Table 1 (semantic vs. structural score trade-offs), and Figure 2 (complexity analysis showing nested lists as the primary difficulty driver).

**Code:** [github.com/FudanCVL/FMBench](https://github.com/FudanCVL/FMBench)