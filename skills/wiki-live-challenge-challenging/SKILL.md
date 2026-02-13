---
name: "wiki-live-challenge-challenging"
description: "Evaluate deep research agents and LLM-generated long-form articles using the Wiki Live Challenge framework: 39 fine-grained writing criteria (well-written, broad coverage, neutral POV), factual verifiability via statement extraction and embedding-based matching, and citation accuracy checking. Use when: 'evaluate my research agent output', 'benchmark article quality against Wikipedia standards', 'check factual accuracy of generated report', 'audit citations in this article', 'score this article on Wikipedia Good Article criteria', 'build an evaluation pipeline for long-form generation'."
---

# Wiki Live Challenge: Evaluating Long-Form Generated Content Against Expert Standards

This skill enables Claude to evaluate LLM-generated long-form articles, research reports, and deep research agent outputs using the Wiki Live Challenge (WLC) framework from arXiv:2602.01590. The core technique uses Wikipedia Good Articles as expert-level references and applies a 39-criterion writing quality assessment combined with rigorous factual verifiability and citation accuracy metrics. Rather than relying on vague "quality scores," this framework decomposes article quality into three measurable dimensions -- writing quality (39 criteria across well-written, broad coverage, and neutral POV categories), factual coverage (statement-level consistency checking via embeddings), and citation validity (source-level verification). This lets you build evaluation pipelines that produce actionable, fine-grained feedback rather than a single score.

## When to Use

- When building or evaluating a Deep Research Agent (DRA) that generates long-form articles or reports
- When the user asks to benchmark generated content against Wikipedia-quality standards
- When auditing factual accuracy of an LLM-generated report by checking statements against reference material
- When verifying that citations in a generated article actually support the claims they're attached to
- When designing an evaluation framework for any long-form text generation system (RAG pipelines, report generators, automated journalism)
- When the user wants fine-grained feedback on writing quality beyond "good/bad" -- e.g., identifying peacock terms, editorial bias, or gaps in topic coverage
- When comparing multiple models or agent systems on article generation quality

## Key Technique

**Wiki Eval** decomposes article quality into three orthogonal dimensions, each with concrete metrics:

**1. Wiki Writing (39 criteria).** Instead of asking "is this well-written?", the framework breaks writing quality into 39 binary comparison criteria organized under three Wikipedia Good Article pillars: *Well-Written* (21 criteria covering encyclopedic style, lead section quality, clarity, avoidance of peacock terms like "legendary" or "iconic," proper attribution, and absence of weasel words), *Broad in Coverage* (8 criteria for topic completeness, structural organization, appropriate scope, and avoidance of irrelevant detail), and *Neutral Point of View* (10 criteria for balanced viewpoint representation, non-judgmental language, and proportional coverage of competing perspectives). Each criterion is evaluated as a pairwise comparison -- generated article wins, reference wins, or tie -- producing a `gen_win_rate` aggregate.

**2. Wiki Fact -- Verifiability.** Atomic factual statements are extracted from both the reference and generated articles using an LLM. Statements from the reference are matched against the generated article using embedding similarity (top-k retrieval), then an LLM judge determines whether each reference fact is *supported*, *conflicted*, or *not covered* by the generated text. This produces coverage ratio (what fraction of reference facts appear in the generated article) and conflict ratio (what fraction of generated claims contradict the reference).

**3. Wiki Fact -- Citation Accuracy.** For each generated statement that carries a citation, the cited URL is fetched and its content extracted. An LLM judge then determines whether the source content actually supports the claim. This catches "hallucinated citations" -- URLs that exist but don't substantiate the claim -- and produces support ratio and conflict ratio metrics.

The key insight is that these three dimensions are independent and each catches failures the others miss: an article can be well-written but factually wrong, factually correct but poorly structured, or well-sourced but biased. Evaluating all three gives a complete picture.

## Step-by-Step Workflow

### 1. Prepare the reference corpus
Collect expert-level reference articles. For Wikipedia-based evaluation, select recently promoted Good Articles (GAs) from after the model's training cutoff. Filter out list-style articles. Rank candidates by reference count and structural depth to ensure research complexity. Aim for topical diversity across categories.

### 2. Generate articles under controlled conditions
Run your DRA or generation system on the same topics as the reference articles. Block access to Wikipedia itself to prevent trivial copying. Use a standardized prompt: "Write a comprehensive, encyclopedic article about [TOPIC]." Save outputs as clean Markdown files.

### 3. Extract atomic statements from both reference and generated articles
Use an LLM (Gemini-2.5-flash or equivalent) to decompose each article into atomic, self-contained factual statements. Each statement should be a single verifiable claim. Assign citation indices to statements based on proximity to citation markers in the source text.

```python
EXTRACTION_PROMPT = """Extract all atomic factual statements from this article.
Each statement must be:
- Self-contained (understandable without surrounding context)
- A single verifiable claim (not compound)
- Paired with the nearest citation index if one exists

Output as JSON array: [{"fact": "...", "ref_idx": "1" or null}]"""
```

### 4. Compute embedding-based statement matching
Generate embeddings for all extracted statements from both articles (using OpenAI `text-embedding-3-small` or equivalent). For each reference statement, retrieve the top-10 most similar generated statements by cosine similarity. This creates candidate pairs for factual consistency checking.

```python
from sklearn.metrics.pairwise import cosine_similarity
import numpy as np

ref_embeddings = embed(reference_statements)
gen_embeddings = embed(generated_statements)
similarities = cosine_similarity(ref_embeddings, gen_embeddings)
top_k_indices = np.argsort(similarities, axis=1)[:, -10:]  # top 10 per ref statement
```

### 5. Run factual consistency verification
For each reference-generated statement pair (from top-k matching), use an LLM judge to classify the relationship as `supported` (generated content is consistent with the reference fact), `conflicted` (generated content contradicts the reference), or `not_covered` (no relevant generated content found). Aggregate into coverage and conflict ratios.

### 6. Fetch and verify cited sources
For each generated statement that includes a citation URL, fetch the source page content using a web reader (Jina Reader API or equivalent HTML-to-text converter). Then use an LLM judge to determine whether the fetched content actually supports the generated claim. Track support ratio and conflict ratio.

### 7. Run pairwise writing quality comparison across 39 criteria
For each criterion, present the LLM judge with both the generated and reference articles (anonymized as Article A and Article B in randomized order to reduce position bias). The judge determines which article better satisfies the criterion, or if they tie. Aggregate into `gen_win_rate`.

```python
WRITING_CRITERIA = {
    "well_written": [
        "prose_clarity", "lead_section_summary", "encyclopedic_tone",
        "no_peacock_terms", "no_weasel_words", "no_editorializing",
        "no_cliches", "no_euphemisms", "proper_attribution",
        "no_vague_temporal_refs", "no_contentious_labels",
        "no_unsupported_superlatives", "conciseness", "grammar",
        # ... 21 total
    ],
    "broad_coverage": [
        "main_aspects_covered", "appropriate_scope", "structural_organization",
        "no_irrelevant_detail", "section_coherence",
        # ... 8 total
    ],
    "neutral_pov": [
        "balanced_viewpoints", "opinion_attribution", "non_judgmental_language",
        "proportional_coverage", "no_contested_assertions_as_fact",
        # ... 10 total
    ]
}
```

### 8. Aggregate and report multi-dimensional scores
Produce a structured evaluation report with scores across all three dimensions. Include per-criterion breakdowns for writing quality, per-statement results for verifiability, and per-citation results for source accuracy.

### 9. Identify actionable improvement areas
Rank criteria by failure rate to surface systematic weaknesses. For example, if the generated article consistently loses on "broad coverage" criteria but wins on "well-written," the agent needs better information retrieval, not better prose generation.

## Concrete Examples

**Example 1: Evaluating a Research Agent's Output**

User: "I have a deep research agent that generates Wikipedia-style articles. Here's its output on 'Quantum Error Correction.' Evaluate it against the actual Wikipedia article."

Approach:
1. Load both the generated article and the Wikipedia reference
2. Extract atomic statements from both (e.g., generated yields 87 statements, reference yields 112)
3. Embed all statements and compute top-10 matches per reference statement
4. Run factual consistency checks on 1,120 candidate pairs
5. Fetch all 34 cited URLs from the generated article and verify claims
6. Run 39-criteria pairwise writing comparison

Output:
```
=== Wiki Eval Report: Quantum Error Correction ===

WRITING QUALITY (gen_win_rate):
  Well-Written (21 criteria):   42.9% win | 38.1% loss | 19.0% tie
  Broad Coverage (8 criteria):  25.0% win | 62.5% loss | 12.5% tie
  Neutral POV (10 criteria):    60.0% win | 20.0% loss | 20.0% tie
  Overall:                      41.0% win rate

VERIFIABILITY:
  Reference facts covered:      68.7% (77/112 statements found)
  Conflicts detected:           4.5% (5 statements contradict reference)
  Generated-only claims:        23 statements not in reference

CITATION ACCURACY:
  Citations checked:            34
  Supported by source:          76.5% (26/34)
  Conflicted with source:       8.8% (3/34)
  Source unreachable:           14.7% (5/34)

TOP ISSUES:
  1. Broad coverage gap: Missing sections on topological codes and fault tolerance
  2. 5 factual conflicts found (see details below)
  3. 3 citations don't support their claims (hallucinated references)
```

**Example 2: Building an Evaluation Pipeline for a RAG System**

User: "I'm building a RAG pipeline that generates technical reports. Help me set up Wiki Eval-style evaluation."

Approach:
1. Set up the three-dimension evaluation architecture
2. Adapt the 39 criteria for technical reports (some Wikipedia-specific criteria like "lead section" map to "executive summary")
3. Implement statement extraction, embedding matching, and LLM-judge verification
4. Create a scoring harness that runs all dimensions

Output (pipeline skeleton):
```python
class WikiEvalPipeline:
    def __init__(self, judge_model="gemini-2.5-pro", embed_model="text-embedding-3-small"):
        self.judge = judge_model
        self.embedder = embed_model

    def evaluate(self, generated: str, reference: str, citations: dict) -> dict:
        # Dimension 1: Writing quality (39 criteria pairwise comparison)
        writing_scores = self._eval_writing(generated, reference)

        # Dimension 2: Factual verifiability (statement extraction + matching)
        gen_statements = self._extract_statements(generated)
        ref_statements = self._extract_statements(reference)
        verifiability = self._eval_verifiability(gen_statements, ref_statements)

        # Dimension 3: Citation accuracy (fetch + verify)
        citation_scores = self._eval_citations(gen_statements, citations)

        return {
            "writing": writing_scores,
            "verifiability": verifiability,
            "citation": citation_scores,
            "summary": self._aggregate(writing_scores, verifiability, citation_scores)
        }

    def _extract_statements(self, text: str) -> list[dict]:
        """Extract atomic factual statements with citation indices."""
        ...

    def _eval_verifiability(self, gen_stmts, ref_stmts) -> dict:
        """Embed, match top-k, judge consistency."""
        gen_embeds = self.embedder.encode([s["fact"] for s in gen_stmts])
        ref_embeds = self.embedder.encode([s["fact"] for s in ref_stmts])
        # Top-k matching + LLM judge for each pair
        ...

    def _eval_citations(self, statements, citations) -> dict:
        """Fetch cited URLs and verify claims against source content."""
        ...

    def _eval_writing(self, gen_text, ref_text) -> dict:
        """39-criteria pairwise LLM-judge comparison."""
        ...
```

**Example 3: Quick Single-Article Quality Audit**

User: "Check this generated article for peacock terms, editorial bias, and unsupported claims."

Approach:
1. Run the Well-Written criteria subset (focus on peacock terms, editorializing, weasel words)
2. Run the Neutral POV criteria subset (focus on contested assertions, judgmental language)
3. Extract statements and flag any without citations

Output:
```
=== Writing Quality Audit ===

PEACOCK TERMS FOUND:
  - Line 12: "groundbreaking discovery" -> replace with "notable discovery"
  - Line 45: "revolutionary approach" -> replace with factual description
  - Line 78: "widely regarded as the best" -> needs citation or removal

EDITORIAL BIAS:
  - Line 23: "Unfortunately, the project failed" -> "The project did not succeed"
  - Line 56: "Critics wrongly claimed" -> "Critics claimed" (remove judgment)

UNSUPPORTED CLAIMS (no citation):
  - Line 34: "Studies show a 40% improvement" -> needs specific citation
  - Line 67: "Most experts agree" -> weasel word; name specific experts
  - Line 91: "The technology was first developed in 2019" -> needs source

NEUTRAL POV ISSUES:
  - Paragraph 3 gives 80% coverage to proponents, 20% to critics
  - Competing approach (Method B) mentioned once but never explained
```

## Best Practices

- **Do:** Use a strong model (Gemini-2.5-pro or GPT-4 class) as the LLM judge for writing criteria. The paper found Gemini-2.5-pro achieved 83.6% agreement with human annotators -- weaker models produce unreliable judgments.
- **Do:** Randomize article presentation order (A/B vs B/A) when running pairwise comparisons to neutralize position bias.
- **Do:** Use the three dimensions independently. A high writing score does not imply factual accuracy. Always check all three.
- **Do:** Filter reference articles for quality. The benchmark uses Wikipedia Good Articles specifically because they pass rigorous peer review. Using random Wikipedia articles as references degrades evaluation reliability.
- **Avoid:** Evaluating with a single overall "quality score." The whole point of the 39-criteria decomposition is that aggregate scores hide systematic weaknesses. Always report per-category breakdowns.
- **Avoid:** Using the generated article's own citations as ground truth without fetching and verifying the actual source content. Many LLM-generated citations point to real URLs that don't support the claim.
- **Avoid:** Skipping the embedding-based matching step and comparing statements naively by string similarity. Semantic embeddings are essential for catching paraphrased facts and near-misses.

## Error Handling

- **Unreachable citation URLs:** Some cited sources may be behind paywalls, rate-limited, or dead links. Track these separately as "unverifiable" rather than counting them as failures. The paper uses Jina Reader for fetching; fall back to a secondary fetcher or mark as unresolvable.
- **Statement extraction failures:** LLMs may extract compound statements or miss implicit claims. Validate extraction quality on a small sample before running full evaluation. If statements are too coarse, re-run extraction with a stricter prompt requiring single-fact-per-statement output.
- **Judge model inconsistency:** LLM judges can disagree with themselves on repeated runs. For high-stakes evaluation, run each criterion 3 times and take the majority vote. The paper validates judge reliability by comparing against human annotations.
- **Embedding dimension mismatch:** Ensure the same embedding model is used for both reference and generated statements. Mixing models produces meaningless similarity scores.
- **Very short or very long articles:** The 39 criteria assume articles of substantial length. For very short content (under 500 words), some criteria (like "broad coverage" and "structural organization") may not apply meaningfully. Adapt the criteria set to the content length.

## Limitations

- The 39 criteria are designed for encyclopedic, Wikipedia-style articles. They need adaptation for other genres (blog posts, technical documentation, news articles). Criteria like "lead section" and "encyclopedic tone" are Wikipedia-specific.
- The framework requires a high-quality reference article for pairwise comparison. It cannot evaluate generated content in isolation (without a reference). For reference-free evaluation, only the citation accuracy dimension works standalone.
- LLM-as-judge evaluation has inherent variance. The paper reports 83.6% human agreement for the best judge model, meaning ~16% of individual criterion judgments may be unreliable.
- Statement extraction and embedding matching add significant computational cost. Evaluating a single article pair requires hundreds of LLM calls (extraction + matching + judging). Budget accordingly for batch evaluation.
- The benchmark's "live" nature (rolling window of new Wikipedia articles) means results are not reproducible across time periods unless the same snapshot is used.
- Top DRA performance in the paper reached only ~58% writing win rate and ~31% factual coverage, indicating this is a genuinely hard benchmark. Don't expect near-100% scores from current systems.

## Reference

**Paper:** "Wiki Live Challenge: Challenging Deep Research Agents with Expert-Level Wikipedia Articles" (arXiv:2602.01590v2)
**Repository:** https://github.com/WangShao2000/Wiki_Live_Challenge
**Key takeaway:** Decomposing article evaluation into 39 writing criteria + statement-level factual verification + citation source checking produces far more actionable feedback than single-score evaluation, and reveals that current DRAs cover less than a third of expert-level reference content.