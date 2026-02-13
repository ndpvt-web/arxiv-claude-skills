---
name: "leveraging-turkish-skill-extraction"
description: "Extract and normalize skills from job postings using a two-stage LLM pipeline: dynamic few-shot skill identification followed by embedding retrieval + LLM reranking against a standardized taxonomy (ESCO). Optimized for morphologically complex and low-resource languages like Turkish. Use when: 'extract skills from job descriptions', 'parse job postings for competencies', 'match skills to ESCO taxonomy', 'build a skill extraction pipeline', 'normalize skills from Turkish job ads', 'link extracted skills to a standard taxonomy'."
---

# Skill Extraction from Job Postings via Two-Stage LLM Pipeline

This skill enables Claude to extract skill mentions from unstructured job postings and link them to standardized skill taxonomies (primarily ESCO), using the two-stage pipeline from Arslan et al. (2026). The approach combines dynamic few-shot prompting for span identification with embedding-based retrieval and LLM reranking for taxonomy alignment. It is specifically designed to handle morphologically complex and low-resource languages (Turkish, Finnish, Hungarian, Korean, etc.) but works equally well for English and other high-resource languages.

## When to Use

- When the user asks to extract skills, competencies, or qualifications from job descriptions or resumes
- When building a recruitment pipeline that needs to normalize free-text skills against ESCO, O*NET, or a custom skill taxonomy
- When the user wants to parse Turkish (or another morphologically rich language) job postings for structured skill data
- When linking extracted skill spans to standardized identifiers for labor market analytics
- When the user needs to compare skill requirements across job postings, occupations, or regions
- When building a dataset of annotated skills from job ads for NLP research or HR analytics

## Key Technique

The core insight is that **LLM-based extraction outperforms supervised sequence labeling** (e.g., fine-tuned BERT NER) for skill extraction in low-resource settings, provided you use a two-stage pipeline that separates identification from linking. Stage 1 (Skill Identification) uses dynamic few-shot prompting: instead of fixed example sets, you retrieve the most semantically similar annotated examples to the input job posting and include them as in-context demonstrations. This dramatically improves extraction quality compared to static few-shot or zero-shot approaches because the LLM sees examples from the same occupation domain as the input.

Stage 2 (Skill Linking) maps each extracted span to the nearest entry in a standardized taxonomy like ESCO. This uses a two-pass approach: first, an embedding model (e.g., `e5-large` or `multilingual-e5-large`) retrieves the top-K candidate taxonomy entries by cosine similarity. Then, an LLM reranks these candidates using the full context of the job posting, the extracted span, and the candidate definitions. This two-pass design combines the speed of dense retrieval with the contextual precision of LLM reasoning.

A critical design choice is **providing occupation context** (job title, occupation area) in both stages. Telling the LLM that a posting is for a "Software Engineer" vs. a "Nurse" significantly disambiguates skill spans like "monitoring" (system monitoring vs. patient monitoring). The paper also shows that **encouraging causal reasoning** — asking the LLM to explain *why* a span is a skill before committing to the extraction — improves precision by reducing false positives.

## Step-by-Step Workflow

1. **Prepare the taxonomy index.** Load the target skill taxonomy (ESCO preferred, or O*NET / custom). For each taxonomy entry, compute an embedding using a multilingual embedding model (`multilingual-e5-large` or equivalent). Store embeddings in a vector index (FAISS, Qdrant, or a simple numpy array for small taxonomies).

2. **Build a few-shot example bank.** Collect or create 50–200 annotated job posting excerpts with labeled skill spans. Embed each excerpt using the same embedding model. Store alongside their annotations for retrieval at inference time.

3. **Receive and preprocess the job posting.** Extract the raw text from the input job posting. Identify metadata if available: job title, occupation area/category, employer sector. Normalize whitespace and segment into paragraphs or bullet lists — preserve original structure as context.

4. **Retrieve dynamic few-shot examples.** Embed the input job posting text. Retrieve the top 3–5 most similar annotated examples from the example bank by cosine similarity. Prefer examples from the same occupation area if metadata is available.

5. **Run Stage 1: Skill Identification via LLM.** Construct a prompt with: (a) a system instruction defining what counts as a skill (knowledge, competence, tool proficiency, soft skill), (b) the retrieved few-shot examples with their annotated spans, (c) the occupation context (job title, area), (d) a causal reasoning instruction ("For each candidate span, briefly explain why it qualifies as a skill before including it"), and (e) the target job posting text. Parse the LLM response to extract skill spans and their character offsets in the original text.

6. **Embed each extracted skill span.** Compute embeddings for every extracted span using the same multilingual model used for the taxonomy index.

7. **Run Stage 2a: Embedding-Based Retrieval.** For each skill span embedding, retrieve the top 10–20 candidate taxonomy entries from the vector index by cosine similarity.

8. **Run Stage 2b: LLM-Based Reranking.** For each skill span, construct a reranking prompt containing: (a) the original span text, (b) surrounding sentence context from the job posting, (c) the job title / occupation area, and (d) the list of candidate taxonomy entries with their definitions. Ask the LLM to select the best-matching taxonomy entry or indicate "no match" if none fit. Instruct the LLM to explain its ranking rationale.

9. **Assemble structured output.** Return a list of objects, each containing: the extracted skill span, its position in the source text, the linked taxonomy URI/ID, the taxonomy preferred label, and a confidence indicator (e.g., cosine similarity score from retrieval + LLM match confirmation).

10. **Validate and post-process.** Deduplicate skills that map to the same taxonomy entry. Flag spans the LLM marked as uncertain. Optionally group results by skill type (knowledge vs. competence vs. tool vs. language vs. soft skill) using the taxonomy hierarchy.

## Concrete Examples

**Example 1: Turkish Job Posting Skill Extraction**

```
User: Extract skills from this Turkish job posting and map them to ESCO:

"Yazılım Geliştirme Uzmanı - İstanbul
Aranan Nitelikler:
- Java ve Python programlama dillerinde en az 3 yıl deneyim
- RESTful API tasarımı ve geliştirme konusunda bilgi
- PostgreSQL ve MongoDB veritabanı yönetimi
- Agile/Scrum metodolojilerine hakimiyet
- Takım çalışmasına yatkın, iletişim becerileri güçlü"

Approach:
1. Identify occupation context: "Yazılım Geliştirme Uzmanı" (Software Developer), Istanbul
2. Retrieve few-shot examples from software engineering domain
3. Run skill identification with dynamic examples and causal reasoning
4. For each span, retrieve ESCO candidates and rerank

Output:
| Extracted Span          | ESCO Skill ID         | ESCO Preferred Label        | Type       |
|-------------------------|-----------------------|-----------------------------|------------|
| Java                    | S5.7.1.3              | Java (computer programming) | Knowledge  |
| Python programlama      | S5.7.1.6              | Python (computer programming)| Knowledge |
| RESTful API tasarımı    | S5.8.0.2              | design application interfaces| Competence|
| PostgreSQL              | S5.6.1.1              | database management systems | Knowledge  |
| MongoDB                 | S5.6.1.1              | database management systems | Knowledge  |
| Agile/Scrum             | S4.3.2.0              | agile project management    | Competence |
| Takım çalışması         | S1.3.1.0              | work in teams               | Competence |
| iletişim becerileri     | S1.1.0.0              | communication               | Competence |
```

**Example 2: Building the Extraction Pipeline in Python**

```
User: Help me build a skill extraction pipeline for job postings using the two-stage approach.

Approach:
1. Set up the taxonomy vector index
2. Implement dynamic few-shot retrieval
3. Write the skill identification prompt template
4. Write the reranking prompt template
5. Wire stages together

Output (key code components):

# --- Stage 1: Dynamic Few-Shot Skill Identification ---

SKILL_ID_SYSTEM_PROMPT = """You are a skill extraction specialist. Extract all skill,
competence, knowledge, and tool mentions from job postings.

For each span you identify:
1. Quote the exact text from the posting
2. Briefly explain WHY this is a skill (causal reasoning)
3. Classify as: knowledge | competence | tool | language | soft-skill

Context: Job title is "{job_title}", occupation area is "{occupation_area}".

Here are annotated examples from similar postings:
{dynamic_examples}

Now extract skills from the following posting. Return JSON array."""

# --- Few-shot retrieval ---
import numpy as np
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("intfloat/multilingual-e5-large")

def get_dynamic_examples(posting_text, example_bank, top_k=5):
    query_emb = model.encode(posting_text, normalize_embeddings=True)
    similarities = np.dot(example_bank["embeddings"], query_emb)
    top_indices = np.argsort(similarities)[-top_k:][::-1]
    return [example_bank["annotations"][i] for i in top_indices]

# --- Stage 2: Embedding Retrieval + LLM Reranking ---

RERANK_PROMPT = """Given the skill span "{span}" extracted from a {job_title} posting,
which of these ESCO taxonomy entries is the best match?

Context sentence: "{context_sentence}"

Candidates:
{candidates_with_definitions}

Select the best match or respond "NO_MATCH". Explain your reasoning."""

def link_skill(span, context, job_title, taxonomy_index, top_k=10):
    span_emb = model.encode(span, normalize_embeddings=True)
    scores = np.dot(taxonomy_index["embeddings"], span_emb)
    top_indices = np.argsort(scores)[-top_k:][::-1]
    candidates = [taxonomy_index["entries"][i] for i in top_indices]
    # LLM reranking call
    result = llm_rerank(span, context, job_title, candidates)
    return result
```

**Example 3: Cross-Language Low-Resource Adaptation**

```
User: I want to adapt this pipeline for Finnish job postings. What do I need to change?

Approach:
1. The pipeline is language-agnostic by design — the key change is the few-shot bank
2. Use the same multilingual embedding model (multilingual-e5-large handles Finnish)
3. ESCO taxonomy already has Finnish labels — use fi locale entries
4. Build 50+ annotated Finnish job posting examples for the few-shot bank
5. Keep the same prompt structure but translate system instructions to Finnish
   or keep them in English (LLMs handle cross-lingual prompting well)

Minimal changes needed:
- Load ESCO taxonomy with Finnish preferred labels (fi locale)
- Create annotated Finnish few-shot examples (even 50 provides strong results)
- Optionally translate the causal reasoning instruction to Finnish
- No model retraining required — this is the core advantage of the LLM approach
```

## Best Practices

- **Do:** Always include occupation context (job title, sector) in prompts — the paper shows this significantly reduces ambiguity in skill identification and linking.
- **Do:** Use dynamic few-shot selection over static examples — retrieving domain-similar examples improves F1 by a meaningful margin over fixed example sets.
- **Do:** Require causal reasoning in the identification prompt ("explain why this is a skill") — this reduces false positives by forcing the LLM to justify each extraction.
- **Do:** Use a two-pass linking strategy (embedding retrieval then LLM rerank) — pure embedding matching misses contextual nuance, while pure LLM search over the full taxonomy is too slow and expensive.
- **Avoid:** Skipping the reranking step and relying solely on embedding cosine similarity — top-1 embedding matches are often wrong for ambiguous or domain-specific skills.
- **Avoid:** Using zero-shot prompting for morphologically complex languages — agglutinative languages like Turkish and Finnish produce surface forms that diverge heavily from dictionary forms, making few-shot examples essential for the LLM to learn extraction patterns.
- **Avoid:** Extracting from the full posting in one shot for very long texts — segment into sections (requirements, responsibilities, qualifications) and process each separately, then deduplicate.

## Error Handling

- **No matching taxonomy entry:** The reranker should be instructed to return "NO_MATCH" rather than force-fitting. Log unmatched spans for human review or taxonomy extension.
- **Overlapping/nested spans:** When the LLM extracts both "Python" and "Python programlama dilinde deneyim", prefer the shorter normalized form unless the longer span carries distinct meaning. Deduplicate by taxonomy ID.
- **Hallucinated skills:** The LLM may infer skills not explicitly stated in the text (e.g., inferring "Linux" from a DevOps posting). The causal reasoning step helps catch this — reject spans where the LLM's reasoning references inference rather than textual evidence.
- **Embedding model language gaps:** If the multilingual embedding model underperforms on your target language, try translating extracted spans to English before retrieval, then map back. This is a practical fallback for extremely low-resource languages.
- **Rate limits / cost:** The reranking step makes one LLM call per extracted span. For postings with 20+ skills, batch multiple spans into a single reranking prompt (5 spans per call) to reduce cost.

## Limitations

- **Taxonomy coverage:** ESCO does not cover every possible skill, especially emerging technologies or highly domain-specific competencies. Unmatched spans require manual taxonomy extension.
- **End-to-end F1 ceiling:** The paper reports 0.56 end-to-end F1 for Turkish, meaning roughly half of skills are either missed or incorrectly linked. This is competitive with other languages but not production-perfect — human review is still needed for high-stakes applications.
- **Few-shot bank bootstrapping:** You need at least 50 annotated examples to make dynamic retrieval effective. For a truly new language with zero annotations, start with zero-shot, manually correct outputs, and iteratively build the bank.
- **Cost at scale:** Two LLM calls per posting (identification + reranking) plus embedding computations make this more expensive than a fine-tuned NER model. For millions of postings, consider using the LLM pipeline to generate training data, then distill into a supervised model.
- **Temporal drift:** Skill taxonomies and job market terminology evolve. The few-shot bank and taxonomy index need periodic updates to stay relevant.

## Reference

**Paper:** Arslan et al., "Leveraging LLMs For Turkish Skill Extraction" (2026). arXiv:2601.22885v1 — https://arxiv.org/abs/2601.22885v1

Look for: Table comparisons of dynamic vs. static few-shot prompting, the full prompt templates in the appendix, the ESCO linking evaluation methodology, and the annotation guidelines for building your own skill extraction dataset.