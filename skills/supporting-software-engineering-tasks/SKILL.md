---
name: "supporting-software-engineering-tasks"
description: "Generate test scenarios from requirements and retrieve/analyze software engineering documents using a supervisor-worker star topology of specialized agents. Use when: 'generate test cases from requirements', 'search project documents', 'summarize this specification', 'track changes across document versions', 'answer questions about our design docs', 'create test scenarios from this feature spec'."
---

# Supporting Software Engineering Tasks with Agentic AI

This skill enables Claude to apply a **supervisor-worker star topology** pattern to two core software engineering tasks: (1) automatically generating structured test scenarios from requirements documents, and (2) performing intelligent retrieval, search, Q&A, change tracking, and summarization across a corpus of software engineering documents. The approach, based on Kica et al. (2026), decomposes each task into subtasks handled by dedicated worker agents coordinated by a central supervisor, using LangGraph-style state management and RAG-based retrieval.

## When to Use

- When the user provides a requirements document (PRD, user stories, acceptance criteria) and asks to generate test scenarios or test cases from it
- When the user has a collection of software engineering documents (specs, design docs, RFCs, ADRs) and wants to search, query, or summarize them
- When the user asks to compare two versions of a specification and track what changed
- When the user wants structured Q&A over project documentation (e.g., "What does the spec say about authentication timeouts?")
- When the user needs to summarize a long specification or design document into actionable points
- When the user wants to generate a test matrix or test plan from functional requirements

## Key Technique

**Star Topology with Supervisor-Worker Agents.** The core architectural pattern places a supervisor agent at the center of a star graph, with specialized worker agents at each point. The supervisor receives the user's request, decomposes it into subtasks, routes each subtask to the appropriate worker, collects results, and synthesizes a final output. This differs from a linear chain (where agents pass work sequentially) by allowing parallel execution and selective routing -- the supervisor decides which workers are needed for each request and can invoke them concurrently.

**Test Scenario Generation** uses four worker roles: a *Requirement Parser* that extracts structured requirements (preconditions, inputs, expected behaviors, edge cases) from natural language; a *Test Case Generator* that produces individual test scenarios with IDs, descriptions, steps, inputs, and expected outcomes; a *Validation Agent* that checks generated scenarios for completeness, consistency, and traceability back to requirements; and a *Consolidation Agent* that merges, deduplicates, and formats the final output. The supervisor orchestrates these in sequence: parse first, then generate, then validate, then consolidate.

**Document Retrieval** uses four dedicated agents, one per use case: a *Search Agent* for semantic retrieval over chunked and embedded documents; a *QA Agent* that combines retrieval with answer synthesis; a *Change Tracking Agent* that diffs document versions and identifies meaningful modifications; and a *Summarization Agent* that produces hierarchical summaries of long documents. Each agent operates independently -- the supervisor routes the user's intent to the correct agent. RAG (Retrieval-Augmented Generation) with dense vector embeddings and re-ranking underpins the retrieval agents.

## Step-by-Step Workflow

### For Test Scenario Generation

1. **Ingest the requirements document.** Read the user's requirements input (PRD, user stories, acceptance criteria, or specification text). Identify the document structure -- numbered requirements, user stories with acceptance criteria, or prose descriptions.

2. **Parse and extract structured requirements.** Act as the Requirement Parser agent: extract each discrete requirement, identifying its ID/label, description, preconditions, inputs, expected behavior, and any edge cases or constraints mentioned. Output a structured list (JSON or markdown table).

3. **Generate test scenarios per requirement.** Act as the Test Case Generator agent: for each parsed requirement, produce one or more test scenarios. Each scenario must include: a unique Test ID, the Requirement ID it traces to, a description, preconditions, test steps (numbered), test inputs, expected outcome, and priority (critical/high/medium/low).

4. **Generate negative and boundary test cases.** Extend beyond happy-path scenarios: for each requirement, identify at least one negative test (invalid input, unauthorized access, missing data) and one boundary test (max/min values, empty inputs, concurrent access) where applicable.

5. **Validate completeness and consistency.** Act as the Validation Agent: verify every requirement has at least one test scenario, check that test steps are unambiguous and executable, confirm expected outcomes are verifiable, and flag any requirements that lack testable criteria.

6. **Consolidate and format output.** Act as the Consolidation Agent: deduplicate overlapping scenarios, assign final sequential Test IDs, organize by requirement or feature area, and produce the output in the user's preferred format (markdown table, CSV-ready text, or structured JSON).

### For Document Retrieval and Analysis

1. **Identify the use case.** Determine which of the four retrieval operations the user needs: semantic search, question answering, change tracking, or summarization.

2. **For Search:** Chunk the document(s) into semantically coherent sections. Identify the user's query intent. Search across chunks using semantic similarity, then re-rank results by relevance. Return the top passages with source references (document name, section, page/line).

3. **For Q&A:** Retrieve the most relevant chunks for the user's question, then synthesize a direct answer grounded in the retrieved text. Always cite which document section supports the answer. If the documents don't contain sufficient information, say so explicitly.

4. **For Change Tracking:** Accept two versions of a document. Perform structural alignment (match sections by heading/ID). Identify additions, deletions, and modifications at the requirement/section level. Summarize changes with their likely impact on downstream artifacts (tests, design, code).

5. **For Summarization:** For long documents, apply hierarchical summarization -- first summarize each section independently, then produce a top-level summary that synthesizes section summaries. Preserve key technical details, decisions, and action items. Flag any ambiguities or contradictions found.

6. **Return structured results.** Always provide traceable outputs: cite document names, section headings, and paragraph numbers. Format results as the user needs (bullet points, tables, or prose).

## Concrete Examples

**Example 1: Generating test scenarios from a feature specification**

User: "Generate test cases from this login feature spec: Users can log in with email and password. Passwords must be 8+ characters. After 5 failed attempts, the account locks for 30 minutes. Users can reset passwords via email link."

Approach:
1. Parse into four discrete requirements: R1 (email/password login), R2 (password length validation), R3 (account lockout), R4 (password reset)
2. Generate test scenarios for each requirement including happy path, negative, and boundary cases
3. Validate all requirements are covered
4. Consolidate into structured output

Output:
```
| Test ID | Req ID | Description                          | Preconditions              | Steps                                                    | Expected Outcome                          | Priority |
|---------|--------|--------------------------------------|----------------------------|----------------------------------------------------------|-------------------------------------------|----------|
| TC-001  | R1     | Successful login with valid creds    | User account exists         | 1. Enter valid email 2. Enter valid password 3. Click Login | User redirected to dashboard              | Critical |
| TC-002  | R1     | Login with unregistered email        | Email not in system          | 1. Enter unregistered email 2. Enter password 3. Click Login | Error: "Invalid credentials"              | High     |
| TC-003  | R2     | Password below minimum length        | Registration/login form open | 1. Enter email 2. Enter 7-char password 3. Submit        | Validation error: "Min 8 characters"      | High     |
| TC-004  | R2     | Password exactly 8 characters        | Registration/login form open | 1. Enter email 2. Enter 8-char password 3. Submit        | Password accepted                         | Medium   |
| TC-005  | R3     | Account locks after 5 failed attempts| User account exists          | 1. Enter valid email 2. Enter wrong password 5 times     | Account locked, message shown             | Critical |
| TC-006  | R3     | Login attempt during lockout period  | Account locked (< 30 min)   | 1. Enter valid credentials 2. Click Login                | Error: "Account locked, try after X min"  | Critical |
| TC-007  | R3     | Login after lockout expires          | Account locked (> 30 min)   | 1. Wait 30 minutes 2. Enter valid credentials 3. Login   | Successful login                          | High     |
| TC-008  | R4     | Password reset with valid email      | User account exists          | 1. Click "Forgot Password" 2. Enter email 3. Submit      | Reset email sent, confirmation shown      | Critical |
| TC-009  | R4     | Password reset with unregistered email| Email not in system         | 1. Click "Forgot Password" 2. Enter unknown email        | Generic message (no info leak)            | High     |
```

**Example 2: Q&A over project documentation**

User: "Based on our API design doc, what authentication method does the payments endpoint use?"

Approach:
1. Identify use case: Question Answering
2. Search the provided document for sections mentioning "payments endpoint", "authentication", and "authorization"
3. Retrieve relevant chunks, re-rank by relevance to the specific question
4. Synthesize answer with citations

Output:
```
Based on the API Design Document (Section 4.3 - Payment Endpoints):

The payments endpoint uses **OAuth 2.0 Bearer tokens** with the `payments:write` scope.
All requests to `/api/v2/payments/*` require:
- A valid access token in the Authorization header
- The token must have been issued within the last 60 minutes (Section 4.3.2)
- Webhook callbacks use HMAC-SHA256 signature verification (Section 4.3.5)

Source: api-design-v2.md, Sections 4.3, 4.3.2, 4.3.5
```

**Example 3: Tracking changes between specification versions**

User: "What changed between v1.2 and v1.3 of our requirements spec?"

Approach:
1. Identify use case: Change Tracking
2. Align sections structurally between the two versions
3. Diff each aligned section, categorize as added/modified/removed
4. Assess downstream impact

Output:
```
## Changes: Requirements Spec v1.2 → v1.3

### Added (3 requirements)
- **REQ-45**: Two-factor authentication for admin users
  Impact: New test scenarios needed, auth module changes required
- **REQ-46**: Audit log retention policy (90 days)
  Impact: New database migration, storage capacity planning
- **REQ-47**: GDPR data export endpoint
  Impact: New API endpoint, test scenarios for data formats

### Modified (2 requirements)
- **REQ-12**: Password policy changed from 8+ chars to 12+ chars with complexity rules
  Impact: Update validation logic, update 4 existing test cases (TC-003, TC-004, TC-010, TC-011)
- **REQ-23**: Session timeout reduced from 60 min to 30 min
  Impact: Update session config, update TC-031

### Removed (1 requirement)
- **REQ-08**: Legacy XML export (deprecated in v1.1, now removed)
  Impact: Remove 3 test cases (TC-020, TC-021, TC-022), remove endpoint code
```

## Best Practices

- **Do:** Trace every test scenario back to a specific requirement ID. Traceability is the primary value of structured test generation -- without it, you lose the ability to verify coverage.
- **Do:** Generate negative and boundary test cases for every requirement, not just happy paths. The paper emphasizes completeness validation as a distinct agent responsibility.
- **Do:** When answering questions over documents, always cite the specific section and document. Never synthesize answers from general knowledge when the user expects document-grounded responses.
- **Do:** For change tracking, categorize changes by impact severity (breaking, behavioral, cosmetic) to help the user prioritize review.
- **Avoid:** Generating test scenarios that are not independently executable. Each test case must have sufficient preconditions and steps to run without implicit dependencies on other tests.
- **Avoid:** Summarizing documents by only capturing the first and last sections. Use hierarchical summarization (section-by-section, then synthesis) to preserve details from the middle of long documents.

## Error Handling

- **Ambiguous requirements:** When a requirement is vague or untestable (e.g., "the system should be fast"), flag it explicitly in the validation step. Generate a placeholder test scenario with a note that the requirement needs quantifiable acceptance criteria.
- **Missing preconditions:** If a requirement references another requirement that wasn't provided, note the dependency and generate test scenarios with assumed preconditions, clearly marked as assumptions.
- **Document format issues:** If the input document has inconsistent structure (mixed numbering, missing section headers), normalize it during the parsing step and note any structural assumptions made.
- **Contradictory requirements:** When two requirements conflict (e.g., REQ-5 says timeout is 30s, REQ-12 says 60s), flag the contradiction in the validation output and generate test scenarios for both interpretations.
- **Insufficient context for Q&A:** If the provided documents don't contain enough information to answer a question, say so directly rather than guessing. Suggest which document or section might contain the answer.

## Limitations

- Test scenario generation works best with well-structured requirements (numbered, with clear acceptance criteria). Free-form prose requirements will produce lower-quality output that needs more manual review.
- The approach generates functional test scenarios. It does not cover non-functional testing (performance benchmarks, load testing specifics, security penetration tests) without explicit non-functional requirements as input.
- Document retrieval quality depends on document length and structure. Very short documents (< 1 page) don't benefit from chunking and retrieval -- direct reading is better. Very long documents (> 100 pages) may require iterative refinement of queries.
- Change tracking assumes the two document versions share a common structure. If a document was completely reorganized between versions, section-level alignment will fail and a full content-level diff is needed instead.
- This approach is designed for software engineering documents specifically. It assumes domain vocabulary (requirements, test cases, APIs, etc.) and may not generalize well to other document types.

## Reference

Kica, M., Radosky, L., Slivka, D., Kubinova, K., & Dovhun, D. (2026). *Supporting software engineering tasks with agentic AI: Demonstration on document retrieval and test scenario generation.* arXiv:2602.04726v1. [https://arxiv.org/abs/2602.04726v1](https://arxiv.org/abs/2602.04726v1)

Key takeaway: The star topology supervisor-worker pattern with dedicated agents per subtask (parsing, generation, validation, consolidation) produces more complete and traceable test scenarios than single-pass generation, and dedicated per-use-case agents for document retrieval outperform monolithic RAG systems on software engineering documents.