---
name: "mrag-benchmarking-retrieval-augmented-generation"
description: "Build and evaluate biomedical RAG pipelines using the MRAG benchmark methodology. Configures retrieval, prompting, and generation components for medical QA systems. Use when: 'build a medical RAG pipeline', 'evaluate my biomedical QA system', 'optimize retrieval for clinical questions', 'set up PubMed-based RAG', 'benchmark RAG on medical datasets', 'configure RAG for drug interaction extraction'."
---

This skill enables Claude to design, configure, and evaluate Retrieval-Augmented Generation pipelines specifically for biomedical and clinical question-answering. It applies findings from the MRAG benchmark — which tested 13 datasets across 14,816 samples covering multi-choice QA, long-form QA, information extraction, and link prediction — to make empirically grounded decisions about retriever selection, document quantity, corpus composition, prompting strategy, and generation model sizing for medical RAG systems.

## When to Use

- When the user asks to build a RAG system for medical, clinical, or biomedical question-answering
- When evaluating an existing medical QA pipeline and deciding which components to swap or tune
- When choosing between sparse retrieval (BM25), dense retrieval (BGE, E5, MedCPT), or hybrid fusion for a health domain corpus
- When deciding how many retrieved documents to feed into a medical LLM and which prompting strategy to use
- When building a bilingual (English/Chinese) medical QA system over PubMed and Wikipedia
- When extracting drug-drug interactions, chemical-disease relations, or knowledge graph links from biomedical literature
- When the user wants to benchmark their RAG pipeline against established medical QA datasets (MedQA, PubMedQA, BioASQ, MedMCQA)

## Key Technique

The MRAG methodology treats a biomedical RAG pipeline as a configurable system with four independent axes: **corpus**, **retriever**, **prompting strategy**, and **generation model**. Rather than treating RAG as a monolithic black box, MRAG isolates each component's contribution through controlled ablation. The core insight is that optimal configuration varies dramatically by task type — factoid QA needs few, precise documents while complex reasoning benefits from larger context windows, and domain-specific corpora only help when the task genuinely requires specialized literature.

Three prompting strategies were benchmarked: **Direct Answer** (baseline), **Chain-of-Thought (CoT)** (explicit reasoning steps before answering), and **CoT-Refine** (generate an initial answer, then re-read retrieved documents and revise reasoning). CoT-Refine consistently outperformed alternatives by 1-5 percentage points because it forces the model to reconcile its parametric knowledge with retrieved evidence in a second pass. This two-pass pattern is the single most transferable technique from the paper.

Retrieval fusion via **Reciprocal Rank Fusion (RRF)** — combining BM25 sparse scores with dense embedding scores — proved more robust than any single retriever across task types. The paper also uncovered a readability paradox: RAG improves factual accuracy and reasoning quality but makes long-form answers more technical and less readable, which matters for patient-facing applications.

## Step-by-Step Workflow

1. **Classify the biomedical task type.** Determine whether the target task is multi-choice QA (e.g., board exam questions), long-form QA (e.g., patient queries), information extraction (e.g., drug interactions, gene-disease relations), or link prediction (e.g., knowledge graph completion). This determines every downstream configuration choice.

2. **Select and construct the retrieval corpus.** For literature-dependent tasks (PubMedQA, drug interactions), index PubMed abstracts. For general medical knowledge (board-style QA), use Wikipedia medical articles. Do NOT blindly combine corpora — the MRAG finding is that mixed corpora don't improve average performance and can hurt precision. Match corpus to task.

3. **Configure the retriever using hybrid fusion.** Implement BM25 for sparse retrieval alongside a dense encoder (BGE-base is sufficient — domain-specialized encoders like MedCPT did not consistently outperform it). Combine results using Reciprocal Rank Fusion: `RRF_score(d) = sum(1 / (k + rank_i(d)))` where k=60 is standard, and rank_i is the document's rank from retriever i.

4. **Set the document retrieval count based on task type.** For factoid tasks with directly retrievable answers (PubMedQA-style), use 2 snippets — more documents degrade accuracy by introducing noise. For complex multi-hop reasoning tasks (drug interactions, clinical reasoning), use 4-8 snippets. Never exceed 32 without evidence it helps your specific task. Start low and increase only if evaluation shows improvement.

5. **Chunk documents into retrieval units.** Split corpus documents into 256-512 token chunks with 50-token overlap. Each chunk should carry metadata: source (PubMed ID or Wikipedia article title), section heading, and publication date. Preserve sentence boundaries during chunking to maintain coherence.

6. **Implement the CoT-Refine prompting strategy.** Structure the generation prompt in two passes:
   - **Pass 1**: Present the question with retrieved context and ask for a chain-of-thought answer.
   - **Pass 2**: Feed the initial answer back alongside the retrieved documents and instruct the model to verify claims against the evidence, correct errors, and produce a refined final answer.

7. **Select the generation model by balancing accuracy and latency.** Models at 70B+ parameters integrate retrieved documents significantly better than smaller models (the gap widens at 32B-72B). For production systems needing speed, 7B models still show 4-10 point improvements with RAG versus no-RAG baselines. Use temperature=0.7, top_p=0.8, repetition_penalty=1.05 as the MRAG-validated defaults.

8. **Evaluate with task-appropriate metrics.** For multi-choice and extraction tasks, use accuracy. For long-form QA, implement a 4-dimension evaluation: usefulness, readability, medical knowledge accuracy, and reasoning quality — scored by an LLM judge (GPT-4 level) using an Elo rating system with pairwise comparisons.

9. **Address the readability paradox for patient-facing outputs.** If the system serves non-expert users, add a post-generation simplification step: instruct the model to rewrite the RAG-grounded answer at a 6th-grade reading level while preserving medical accuracy. This counters the tendency of RAG to make outputs overly technical.

10. **Run ablation experiments before finalizing.** Systematically test: (a) RAG vs. no-RAG baseline, (b) sparse-only vs. dense-only vs. hybrid retrieval, (c) 2 vs. 4 vs. 8 retrieved documents, (d) Direct vs. CoT vs. CoT-Refine prompting. Log each configuration's score. The optimal combination is task-dependent — there is no universal best configuration.

## Concrete Examples

**Example 1: Building a PubMed-based clinical QA system**

User: "I need to build a RAG pipeline that answers clinical questions using PubMed abstracts."

Approach:
1. Classify as multi-choice/factoid clinical QA
2. Index PubMed abstracts — download via Entrez API, chunk at 512 tokens with sentence-boundary preservation
3. Set up hybrid retrieval: BM25 index via Elasticsearch + BGE-base dense embeddings via FAISS
4. Configure RRF fusion with k=60
5. Retrieve 2-4 snippets per query (start with 2 for factoid, 4 for reasoning)
6. Implement CoT-Refine prompting with a 70B model

Output — retrieval prompt template:
```
# Pass 1 (Chain-of-Thought)
Given the following medical context retrieved from PubMed:

{retrieved_snippets}

Question: {user_question}

Think step by step. Cite specific evidence from the provided context.
Provide your answer with reasoning.

# Pass 2 (Refine)
Your initial answer was:
{pass_1_answer}

Re-read the provided evidence:
{retrieved_snippets}

Verify each claim in your answer against the evidence. Correct any
unsupported statements. Provide your refined final answer.
```

**Example 2: Evaluating retrieval strategies for drug interaction extraction**

User: "I'm extracting drug-drug interactions from literature. Should I use BM25 or dense retrieval?"

Approach:
1. Classify as information extraction (DDI task)
2. This task requires PubMed corpus — drug interactions are rarely well-covered in Wikipedia
3. Test three configurations: BM25-only, BGE-base-only, and RRF hybrid
4. Use 4-8 snippets (extraction tasks benefit from more context than factoid QA)
5. Evaluate accuracy on a held-out DDI test set

Output — evaluation script structure:
```python
configs = [
    {"retriever": "bm25", "top_k": 4, "prompt": "cot_refine"},
    {"retriever": "bge_base", "top_k": 4, "prompt": "cot_refine"},
    {"retriever": "rrf_hybrid", "top_k": 4, "prompt": "cot_refine"},
    {"retriever": "rrf_hybrid", "top_k": 8, "prompt": "cot_refine"},
]

for cfg in configs:
    snippets = retrieve(query, corpus="pubmed", method=cfg["retriever"], k=cfg["top_k"])
    answer = generate(query, snippets, strategy=cfg["prompt"])
    score = evaluate_ddi_accuracy(answer, ground_truth)
    log_result(cfg, score)

# MRAG finding: RRF hybrid with k=4-8 typically wins for extraction tasks
# BM25-only is competitive when entity names are exact-match targets
```

**Example 3: Bilingual medical QA with readability optimization**

User: "Build a medical chatbot that answers patient questions in both English and Chinese. Answers need to be easy to understand."

Approach:
1. Classify as long-form QA (patient-facing)
2. Build dual corpus: English Wikipedia medical articles + Chinese medical consultation data
3. Use BGE-base for English, a multilingual encoder for Chinese
4. Retrieve 4 snippets per query
5. Apply CoT-Refine prompting
6. Add readability post-processing layer to counter the RAG readability paradox

Output — generation pipeline:
```python
def generate_patient_answer(question, lang):
    snippets = retrieve(question, corpus=lang_corpus[lang], k=4)

    # Pass 1: CoT generation grounded in evidence
    raw_answer = llm.generate(
        prompt=cot_prompt.format(context=snippets, question=question),
        temperature=0.7, top_p=0.8
    )

    # Pass 2: Refine against evidence
    refined_answer = llm.generate(
        prompt=refine_prompt.format(
            context=snippets, question=question, initial=raw_answer
        )
    )

    # Pass 3: Readability simplification (addresses MRAG paradox)
    patient_answer = llm.generate(
        prompt=f"Rewrite the following medical answer for a patient with "
               f"no medical background. Use simple language at a 6th-grade "
               f"reading level. Preserve all medical facts.\n\n{refined_answer}"
    )
    return patient_answer
```

## Best Practices

- **Do** use Reciprocal Rank Fusion to combine sparse and dense retrievers — it is more robust than either alone across all biomedical task types tested
- **Do** start with 2 retrieved snippets for factoid tasks and increase only if evaluation justifies it — more documents often hurt by introducing noise
- **Do** implement CoT-Refine (two-pass prompting) as the default strategy — the 1-5 point improvement is consistent and costs only one additional LLM call
- **Do** evaluate long-form answers on all four dimensions (usefulness, readability, knowledge, reasoning) rather than a single accuracy score
- **Avoid** assuming domain-specific embeddings outperform general-purpose ones — BGE-base matched MedCPT in MRAG experiments; always benchmark before committing
- **Avoid** combining PubMed and Wikipedia corpora by default — mixed corpora did not improve average performance; use task-matched single-domain corpora

## Error Handling

- **Retrieved documents contradict each other**: When snippets from different sources conflict, the CoT-Refine pass should explicitly note the disagreement and prefer the more recent or higher-evidence source (randomized controlled trial over case report)
- **Retrieval returns irrelevant documents**: If the top-k results have low relevance scores (BM25 < threshold or cosine similarity < 0.5), fall back to a no-RAG generation rather than forcing noisy context into the prompt. The MRAG data shows retrieved noise can degrade performance below the no-RAG baseline on some tasks
- **Model hallucinates despite retrieval**: This occurs more with smaller models (< 13B). Add an explicit instruction: "Only make claims supported by the provided context. If the context doesn't address the question, say so." Increase model size if hallucination rate remains high
- **Chinese corpus quality issues**: Medical consultation platform data contains colloquial language and potential misinformation. Apply quality filtering: remove answers shorter than 50 characters, flag answers contradicting established medical guidelines

## Limitations

- The MRAG benchmark focuses on English and Chinese — findings may not transfer to other languages without re-evaluation
- CoT-Refine doubles generation latency — for real-time clinical decision support, Direct prompting with hybrid retrieval may be the better latency-accuracy tradeoff
- The readability paradox means RAG-augmented systems are inherently worse for patient-facing communication without explicit post-processing
- PubMed indexing at scale (14.8M articles) requires significant infrastructure — smaller teams should consider pre-built indexes or subset corpora
- Evaluation using LLM-as-judge (GPT-4 for Elo ratings) introduces its own biases; human expert evaluation remains the gold standard for clinical deployment
- The benchmark does not cover real-time clinical scenarios with evolving patient context or multi-turn diagnostic conversations

## Reference

[MRAG: Benchmarking Retrieval-Augmented Generation for Bio-medicine](https://arxiv.org/abs/2601.16503v2) — Li & Zhu, 2026. Focus on Table 2 (retrieval method comparison), Table 4 (document quantity ablation), and Section 4.3 (prompting strategy analysis) for the most actionable configuration guidance.