---
name: "compactrag-reducing-calls-token"
description: |
  Build multi-hop RAG systems that answer complex questions with only 2 LLM calls total,
  regardless of reasoning depth. Applies CompactRAG's offline atomic QA decomposition and
  online entity-consistent retrieval to slash token costs by 2-5x vs iterative RAG.
  Trigger phrases:
  - "build a multi-hop RAG pipeline"
  - "reduce LLM calls in my RAG system"
  - "answer complex questions over a knowledge base efficiently"
  - "implement CompactRAG"
  - "optimize token usage in retrieval-augmented generation"
  - "build a cost-efficient question answering system"
---

# CompactRAG: Reducing LLM Calls and Token Overhead in Multi-Hop QA

This skill teaches Claude to design and implement multi-hop retrieval-augmented generation (RAG) systems using the CompactRAG architecture. The core idea: decouple offline corpus restructuring from online reasoning so that complex multi-hop questions are answered with exactly **two LLM calls** — one for sub-question decomposition and one for final answer synthesis — regardless of how many reasoning hops are needed. Intermediate hops use lightweight models (RoBERTa for extraction, Flan-T5 for rewriting) and dense retrieval, cutting token consumption from 5-10K tokens/query (typical iterative RAG) down to ~1.9K tokens/query.

## When to Use

- When the user wants to build a **multi-hop QA system** that chains facts across multiple documents (e.g., "Who directed the film starring the actor born in City X?")
- When an existing RAG pipeline makes **too many LLM calls per query** and the user wants to reduce API costs or latency
- When the user has a **static or slowly-changing corpus** that can be preprocessed offline into a structured knowledge base
- When entity references **drift across hops** (pronouns, ambiguous references) and the user needs robust entity grounding
- When building QA over datasets like HotpotQA, 2WikiMultiHopQA, MuSiQue, or similar multi-hop benchmarks
- When the user asks to **decompose complex questions** into sub-questions with dependency ordering

## Key Technique

**The Problem with Iterative RAG:** Standard multi-hop RAG systems (IRCoT, Self-Ask, Iter-RetGen) call the LLM at every hop — retrieve context, reason, generate a follow-up query, retrieve again, reason again. For a 3-hop question, that's 3+ LLM calls, each consuming thousands of tokens. Worse, entity references degrade across hops: "she" in hop 3 may no longer clearly refer to the person identified in hop 1.

**CompactRAG's Solution — Two-Phase Decoupling:**

*Offline Phase:* An LLM reads each document once and converts it into **atomic QA pairs** — minimal, fine-grained question-answer units where each pair encodes exactly one fact. Entities are pre-annotated with SpaCy and enforced in generation prompts to ensure consistent naming. The Q and A are concatenated (`[q;a]`) and encoded with Contriever into a dense retrieval index. This is a one-time cost amortized over all future queries.

*Online Phase:* A complex query triggers exactly two LLM calls. **Call 1** decomposes the query into sub-questions organized in a dependency graph (e.g., q1 must be answered before q2 can be resolved). Each sub-question is then processed *without* the LLM: a Flan-T5-small rewriter grounds entity references from prior answers into the current sub-question, Contriever retrieves top-k atomic QA pairs, and RoBERTa-base extracts the answer span. After all sub-questions resolve, **Call 2** synthesizes the final answer from the collected sub-answers. The result: competitive accuracy with ~80% fewer tokens than IRCoT.

## Step-by-Step Workflow

### Phase 1: Offline Corpus Restructuring

1. **Annotate entities in your corpus.** Run SpaCy NER over every document to tag named entities (people, organizations, locations, dates). Store annotations alongside the source text. These entities will be enforced during QA pair generation to prevent naming inconsistencies.

2. **Generate atomic QA pairs from each document.** Prompt an LLM (GPT-4 at temperature 0, or a strong open model) to decompose each document into atomic question-answer pairs. Each pair must encode a single fact at minimal granularity. Enforce that answers use the exact entity names from the SpaCy annotations. Example prompt structure:
   ```
   Given the following document with annotated entities [ENTITIES],
   generate atomic question-answer pairs where:
   - Each pair encodes exactly ONE factual statement
   - Answers use the exact entity names provided
   - Questions are self-contained (no pronouns or implicit references)
   - Pairs are non-overlapping in information content

   Document: [TEXT]
   ```

3. **Build the dense retrieval index.** Concatenate each QA pair into a single text segment `[question; answer]` and encode it using Contriever (or another unsupervised contrastive dense retriever). Store the embeddings in a vector index (FAISS, Qdrant, or similar). This concatenation maximizes semantic coherence during retrieval.

4. **Train or fine-tune the sub-question rewriter.** Using Flan-T5-small, train a rewriter that takes an ambiguous sub-question and a grounding entity, then outputs a rewritten question with the entity explicitly inserted. Training data consists of triples: `(ambiguous_question, grounding_entity, rewritten_question)`. Apply perturbations like entity masking for robustness.

5. **Train the answer extractor.** Fine-tune RoBERTa-base as a span-prediction model over the atomic QA pairs. Training samples include correct QA pairs with marked answer spans and distractor pairs (retrieved but irrelevant) to handle retrieval noise. Use start/end span prediction loss.

### Phase 2: Online Query Resolution

6. **Decompose the complex query (LLM Call 1).** Send the user's multi-hop question to the LLM with a prompt that outputs a sequence of sub-questions `{q1, q2, ..., qn}` and a dependency graph where edges `qi -> qj` indicate that qi's answer is needed to resolve qj. Process sub-questions in topological order of the dependency graph.

7. **Rewrite each sub-question for entity consistency.** For each sub-question (after the first), pass it through the Flan-T5 rewriter along with the answer entity from its parent sub-question. This replaces pronouns and vague references with explicit entity names. Example: "Who directed that film?" + entity="Inception" becomes "Who directed the film Inception?"

8. **Retrieve and extract answers for each sub-question.** Encode the rewritten sub-question with Contriever, retrieve top-5 atomic QA pairs by embedding similarity, and run RoBERTa-base span extraction over the retrieved pairs to identify the answer. No LLM call needed.

9. **Synthesize the final answer (LLM Call 2).** After all sub-questions are resolved, send the original query along with all sub-question/answer/evidence triples `{Q, {qi, ai, Pi}}` to the LLM for holistic reasoning and final answer generation.

10. **Return the answer with provenance.** Include the atomic QA pairs used as evidence so the user can trace each reasoning step back to source documents.

## Concrete Examples

**Example 1: Building a CompactRAG pipeline for a Wikipedia corpus**

User: "I have a corpus of 50K Wikipedia articles. Build me a multi-hop QA system that can answer questions like 'What university did the director of Inception attend?' without making an LLM call for every hop."

Approach:
1. Run SpaCy NER over all 50K articles, tagging entities
2. Batch-process articles through GPT-4 to generate atomic QA pairs (~10-20 pairs per article, yielding ~500K-1M pairs)
3. Encode all `[q;a]` pairs with Contriever into a FAISS index
4. Fine-tune Flan-T5-small on entity-grounded rewriting data
5. Fine-tune RoBERTa-base on span extraction over the atomic QA pairs
6. At query time, decompose "What university did the director of Inception attend?" into:
   - q1: "Who directed Inception?" (no dependency)
   - q2: "What university did [answer_q1] attend?" (depends on q1)
7. q1: Retrieve from index, RoBERTa extracts "Christopher Nolan"
8. Rewrite q2: "What university did Christopher Nolan attend?"
9. Retrieve and extract: "University College London"
10. LLM synthesizes: "The director of Inception, Christopher Nolan, attended University College London."

Output: 2 LLM calls, ~1.9K tokens total (vs. ~10K for IRCoT on equivalent queries)

**Example 2: Optimizing an existing iterative RAG system**

User: "My current RAG pipeline calls GPT-4 at every hop and costs $0.15 per query. Can I reduce this?"

Approach:
1. Audit current pipeline: identify how many LLM calls per query (typically 3-5 for multi-hop)
2. Restructure the corpus offline into atomic QA pairs (one-time LLM cost)
3. Replace intermediate LLM reasoning calls with: Flan-T5 rewriting + Contriever retrieval + RoBERTa extraction
4. Keep only two LLM calls: initial decomposition and final synthesis
5. Estimated cost reduction: from ~$0.15/query to ~$0.03/query (5x reduction)

Output architecture:
```
Query -> [LLM: decompose] -> sub-questions with dependency graph
  For each sub-question (in dependency order):
    -> [Flan-T5: rewrite with entity grounding]
    -> [Contriever: retrieve top-5 atomic QA pairs]
    -> [RoBERTa: extract answer span]
  Collected answers -> [LLM: synthesize final answer]
```

**Example 3: Implementing the atomic QA generation step**

User: "How do I convert my documents into the atomic QA knowledge base?"

Approach:
1. Install SpaCy and load an NER model (`en_core_web_sm` or `en_core_web_trf`)
2. For each document, extract entities and pass them to the generation prompt
3. Use batch processing with temperature=0 for deterministic output
4. Validate: each QA pair should be self-contained, reference exact entity names, and encode one fact

```python
import spacy
from openai import OpenAI

nlp = spacy.load("en_core_web_trf")
client = OpenAI()

def generate_atomic_qa(document: str) -> list[dict]:
    doc = nlp(document)
    entities = list({ent.text for ent in doc.ents})

    response = client.chat.completions.create(
        model="gpt-4",
        temperature=0,
        messages=[{
            "role": "user",
            "content": f"""Convert this document into atomic QA pairs.
Rules:
- Each pair encodes exactly ONE fact
- Use these exact entity names: {entities}
- Questions must be self-contained (no pronouns)
- Output as JSON array of {{"q": "...", "a": "..."}}

Document: {document}"""
        }]
    )
    return parse_qa_pairs(response.choices[0].message.content)

def build_retrieval_index(qa_pairs: list[dict], retriever):
    texts = [f"{pair['q']} {pair['a']}" for pair in qa_pairs]
    embeddings = retriever.encode(texts)
    # Add to FAISS index
    index.add(embeddings)
    return index
```

## Best Practices

- **Do:** Pre-annotate entities with SpaCy before generating atomic QA pairs. Enforcing entity names in the generation prompt prevents "John" vs. "Mr. Smith" vs. "he" inconsistencies across pairs.
- **Do:** Concatenate Q and A into `[q;a]` before encoding for retrieval. Encoding the question alone loses answer-side semantic signal, degrading retrieval quality.
- **Do:** Process sub-questions in strict topological order of the dependency graph. Parallel execution breaks entity grounding since later sub-questions depend on earlier answers.
- **Do:** Use temperature=0 for both the decomposition and synthesis LLM calls to ensure deterministic, stable outputs.
- **Avoid:** Skipping the rewriter module. Ablation studies show removing entity-consistent rewriting drops accuracy by 5-7 percentage points across all benchmarks.
- **Avoid:** Using the full LLM for intermediate hop reasoning. The entire point of CompactRAG is that RoBERTa (125M params) handles extraction at a fraction of the cost. Only escalate to the LLM for decomposition and synthesis.
- **Avoid:** Generating overlapping QA pairs from documents. Each pair should encode one unique fact. Overlapping pairs waste index space and can confuse the retriever with near-duplicate embeddings.

## Error Handling

- **Sub-question decomposition fails or produces poor splits:** Fall back to a simpler single-hop retrieval. If the LLM generates no dependency edges, treat the query as single-hop and retrieve directly.
- **Entity rewriting introduces hallucinated entities:** Validate that the grounding entity actually appeared in a previous sub-answer. If not, skip rewriting and use the original sub-question.
- **RoBERTa extraction returns low-confidence spans:** Set a confidence threshold (e.g., 0.3). If extraction confidence is below threshold, pass the retrieved QA pairs directly to the final synthesis LLM call for reasoning, gracefully degrading to a retrieve-then-read pattern for that hop.
- **Retrieval returns irrelevant QA pairs:** Include distractor-robust training data for RoBERTa. At inference, if all retrieved pairs have low similarity scores, flag the sub-question as unanswerable and let the synthesis call handle the gap.
- **Atomic QA generation produces malformed pairs:** Validate each pair programmatically — ensure both Q and A fields are non-empty, Q ends with a question mark, A contains at least one entity from the source document.

## Limitations

- **Requires offline preprocessing.** The atomic QA generation step uses an LLM over the entire corpus, which is expensive upfront. Not suitable for corpora that change in real-time (e.g., live news feeds) unless incremental updates are implemented.
- **Fixed decomposition.** The sub-question dependency graph is determined in a single LLM call. If the decomposition is wrong, there's no iterative self-correction — unlike IRCoT which can adjust reasoning mid-chain.
- **Span extraction limits answer form.** RoBERTa extracts spans from retrieved text, so answers must be extractive (present in the corpus). It cannot generate novel phrasing or perform numerical reasoning.
- **Entity-centric assumption.** The rewriter is designed for entity-grounding scenarios. Questions requiring non-entity reasoning (temporal, causal, comparative) may not benefit from the rewriting step.
- **Benchmark-validated scale.** Published results use 250-sample evaluations on curated datasets. Performance on noisy, real-world corpora with millions of documents is not yet validated.

## Reference

**Paper:** [CompactRAG: Reducing LLM Calls and Token Overhead in Multi-Hop Question Answering](https://arxiv.org/abs/2602.05728v1) (Yang et al., 2026)

Key takeaway: Look at Section 3 for the two-phase architecture, Section 3.3 for the entity-consistent rewriting formulation, and Table 1 for the token consumption comparison showing CompactRAG at 1.9K tokens/sample vs. IRCoT at 10.2K tokens/sample while maintaining competitive F1 scores.