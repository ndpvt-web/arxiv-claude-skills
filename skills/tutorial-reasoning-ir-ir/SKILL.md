---
name: "tutorial-reasoning-ir-ir"
description: "Build reasoning-enhanced information retrieval pipelines that go beyond semantic matching. Applies five methodological families — LLM inference-time strategies, RL-guided search, neuro-symbolic verification, Bayesian uncertainty modeling, and geometric embeddings — to handle negation, multi-hop inference, exclusion, and constraint enforcement in retrieval systems. Triggers: 'build a reasoning-aware search pipeline', 'retrieval with logical constraints', 'multi-hop retrieval system', 'search that handles negation', 'neuro-symbolic retrieval', 'reasoning-enhanced RAG pipeline'"
---

# Reasoning-Enhanced Information Retrieval Pipelines

This skill enables Claude to design and implement retrieval systems that reason — not just match semantically. Standard embedding-based retrieval fails on queries requiring negation ("hotels without pools"), multi-hop inference ("papers cited by works that influenced transformer architectures"), set composition, temporal constraints, and evidence synthesis across documents. This skill applies the unified analytical framework from Hoveyda et al. (2026) to select and combine five methodological families — inference-time chain-of-thought interleaving, reinforcement-learning-guided search, neuro-symbolic verification, probabilistic inference, and geometric representation spaces — evaluating each along three axes: representational adequacy, inference mechanism quality, and computational viability at retrieval scale.

## When to Use

- When the user asks to build a search system that must handle negation or exclusion queries (e.g., "find documents about X but not Y")
- When designing a RAG pipeline that requires multi-hop reasoning across multiple retrieved passages before answering
- When the user needs retrieval with logical constraint enforcement (Boolean conditions, temporal ordering, set operations)
- When building a pipeline that must synthesize evidence from multiple documents to answer a complex question
- When the user asks to improve retrieval quality on benchmarks like NevIR, ExcluIR, QUEST, BRIGHT, or BrowseComp
- When implementing iterative retrieve-then-reason loops (IRCoT pattern) for complex question answering
- When the user wants to combine symbolic logic verification with neural retrieval for high-stakes domains (legal, medical, compliance)

## Key Technique

**The core insight**: Semantic similarity is necessary but insufficient for real information needs. The paper identifies five reasoning failures in standard retrieval: (1) negation blindness — embeddings cannot reliably distinguish "with X" from "without X"; (2) exclusion failure — inability to enforce set subtraction; (3) multi-hop gaps — no mechanism to chain evidence across documents; (4) constraint violation — ignoring temporal, numerical, or logical conditions; and (5) synthesis inability — failing to combine partial evidence into a complete answer. Each failure maps to a specific methodological remedy.

**The five families and when to apply each**: *Inference-time strategies* (chain-of-thought interleaving, iterative refinement) work best when you need flexible multi-step reasoning and can tolerate higher latency — use these for complex QA over retrieved documents. *RL-guided search* (Search-R1 pattern) suits scenarios where you can define reward signals for retrieval quality and want the system to learn retrieval-reasoning policies. *Neuro-symbolic integration* (the LINC pattern: LLM parses to formal logic, symbolic prover verifies) is ideal when correctness guarantees matter — legal compliance, medical protocols, formal verification. *Bayesian/probabilistic* approaches model uncertainty explicitly, critical when evidence is incomplete or contradictory. *Geometric representations* (hyperbolic embeddings, Box Embeddings) encode hierarchical and containment relationships that flat vector spaces cannot represent.

**The three evaluation axes** guide selection: *Representational adequacy* — can the representation encode the logical structures your queries require? Box Embeddings handle containment; hyperbolic spaces handle hierarchies; flat embeddings handle neither well. *Inference and learning mechanisms* — does the approach support the type of reasoning needed? Symbolic provers give deductive guarantees; chain-of-thought gives abductive flexibility. *Computational viability* — can it scale to your corpus size? Neuro-symbolic verification is precise but expensive per-query; geometric embeddings are cheap at inference time but require specialized training.

## Step-by-Step Workflow

1. **Classify the reasoning requirement**: Analyze the user's retrieval task and identify which reasoning failures apply. Map each to one of: negation/exclusion (needs constraint-aware representations), multi-hop (needs iterative retrieval-reasoning), evidence synthesis (needs aggregation logic), logical constraints (needs symbolic verification), or uncertainty handling (needs probabilistic modeling).

2. **Select the methodological family**: Based on the classification, choose the primary approach. For multi-hop QA, use the IRCoT interleaving pattern. For constraint enforcement, use neuro-symbolic verification. For hierarchical or containment queries, use geometric embeddings. For uncertain or contradictory evidence, use Bayesian scoring. Document the rationale by scoring against the three axes (representation, inference, compute).

3. **Design the retrieval-reasoning pipeline architecture**: Structure the pipeline as a directed graph of retrieval and reasoning stages. A typical IRCoT pipeline: `query -> initial retrieval -> LLM reasoning step -> refined query -> second retrieval -> LLM synthesis -> answer`. A neuro-symbolic pipeline: `query -> LLM parse to logical form -> retrieval with constraints -> symbolic verification -> filtered results`.

4. **Implement the retrieval layer**: Build the base retrieval using dense embeddings (sentence-transformers, OpenAI embeddings, or similar). For negation-aware retrieval, add a re-ranking stage that explicitly checks constraint satisfaction. For hierarchical needs, consider hyperbolic embeddings via libraries like `geoopt`.

5. **Implement the reasoning layer**: For IRCoT, implement the interleaving loop: generate a chain-of-thought step, extract a sub-query, retrieve, incorporate new evidence, repeat until the reasoning chain is complete or a maximum depth is reached. For neuro-symbolic, implement an LLM-to-logic parser and integrate a symbolic solver (Z3, Prolog, or custom rule engine).

6. **Add constraint verification**: After retrieval, verify that results satisfy the original logical constraints. Parse the query for negation markers, temporal bounds, numerical ranges, and set operations. Filter or re-rank results that violate constraints. This is the step most retrieval systems skip, and it is where most reasoning failures occur.

7. **Implement evidence synthesis**: When the answer requires combining information from multiple documents, build an explicit synthesis stage. Collect retrieved passages, extract relevant facts into a structured format (JSON or knowledge graph triples), resolve contradictions using confidence scores or source authority, and generate the final answer from the structured evidence — not from raw concatenated passages.

8. **Evaluate against reasoning-specific benchmarks**: Test the pipeline on tasks that expose reasoning failures. Use negation pairs (does swapping a negation change the retrieval result?), multi-hop chains (does the system find the intermediate evidence?), and constraint satisfaction (do results actually satisfy stated conditions?). Track both retrieval metrics (nDCG, recall) and reasoning correctness separately.

9. **Optimize the compute-quality tradeoff**: Profile latency and cost. IRCoT loops multiply retrieval calls — add caching for sub-queries. Neuro-symbolic verification is expensive — batch symbolic checks. Geometric embeddings are cheap at inference but need offline training. Set a latency budget and degrade gracefully (e.g., fall back to single-hop retrieval under load).

10. **Document the reasoning contract**: Explicitly specify what reasoning capabilities the pipeline supports and what it does not. A system built for multi-hop QA may still fail on numerical constraints. Transparency about limitations prevents misuse and guides future improvements.

## Concrete Examples

**Example 1: Multi-Hop RAG with IRCoT Interleaving**

User: "Build a RAG pipeline that can answer questions like 'What university did the mass spectrometry inventor attend?' — where the answer requires first finding who invented mass spectrometry, then finding their university."

Approach:
1. Detect that this is a multi-hop question (the answer entity is not directly mentioned in the query).
2. Implement an IRCoT loop:
   - Step 1: LLM generates reasoning: "I need to find who invented mass spectrometry."
   - Step 2: Retrieve documents for sub-query "inventor of mass spectrometry" -> finds "J.J. Thomson" and "Arthur Dempster."
   - Step 3: LLM reasons: "Thomson developed early concepts; Dempster built the first practical instrument. I need the university for the most credited inventor."
   - Step 4: Retrieve documents for "J.J. Thomson university education" -> finds "Trinity College, Cambridge."
   - Step 5: LLM synthesizes the final answer with a citation chain.
3. Return the answer with the full reasoning trace for transparency.

Output:
```python
from dataclasses import dataclass

@dataclass
class RetrievalStep:
    sub_query: str
    retrieved_docs: list[str]
    reasoning: str

@dataclass
class MultiHopAnswer:
    answer: str
    confidence: float
    reasoning_chain: list[RetrievalStep]
    sources: list[str]

def ircot_pipeline(question: str, retriever, llm, max_hops: int = 3) -> MultiHopAnswer:
    chain = []
    context = ""
    for hop in range(max_hops):
        # LLM decides next sub-query based on accumulated context
        prompt = f"Question: {question}\nContext so far: {context}\n"
        prompt += "What specific fact do I still need? Generate a search query for it. "
        prompt += "If the question is answerable, respond with DONE."
        sub_query_or_done = llm.generate(prompt)

        if "DONE" in sub_query_or_done:
            break

        docs = retriever.search(sub_query_or_done, top_k=5)
        reasoning = llm.generate(
            f"Given these documents about '{sub_query_or_done}':\n"
            f"{docs}\nWhat did I learn? Summarize the key fact."
        )
        chain.append(RetrievalStep(sub_query_or_done, docs, reasoning))
        context += f"\nFact {hop+1}: {reasoning}"

    final_answer = llm.generate(
        f"Question: {question}\nEvidence chain: {context}\nProvide a final answer."
    )
    return MultiHopAnswer(
        answer=final_answer,
        confidence=compute_chain_confidence(chain),
        reasoning_chain=chain,
        sources=[doc for step in chain for doc in step.retrieved_docs],
    )
```

**Example 2: Negation-Aware Retrieval with Constraint Verification**

User: "My search system returns beach hotels when users search for 'hotels in Miami without beach access.' Fix this."

Approach:
1. Identify the negation failure: embeddings for "hotels with beach" and "hotels without beach" are nearly identical because embedding models emphasize content words and underweight negation.
2. Add a constraint extraction + verification layer after retrieval.
3. Parse the query to extract positive requirements ("hotels in Miami") and negation constraints ("without beach access").
4. Retrieve using the positive requirements, then filter results by checking the negation constraints.

Output:
```python
import re

def extract_constraints(query: str, llm) -> dict:
    """Use an LLM to parse logical constraints from a natural language query."""
    prompt = (
        f"Parse this search query into structured constraints.\n"
        f"Query: {query}\n"
        f"Return JSON with 'positive' (required attributes) and "
        f"'negative' (excluded attributes) lists."
    )
    parsed = llm.generate_json(prompt)
    # Example output: {"positive": ["hotel", "Miami"], "negative": ["beach access"]}
    return parsed

def negation_aware_search(query: str, retriever, llm, top_k: int = 20):
    constraints = extract_constraints(query, llm)

    # Retrieve using positive terms only (what the user WANTS)
    positive_query = " ".join(constraints["positive"])
    candidates = retriever.search(positive_query, top_k=top_k)

    # Verify negation constraints against each candidate
    verified = []
    for doc in candidates:
        violates = False
        for neg in constraints["negative"]:
            if llm.check(
                f"Does this document indicate the property has '{neg}'?\n"
                f"Document: {doc.text}\nAnswer yes or no."
            ) == "yes":
                violates = True
                break
        if not violates:
            verified.append(doc)

    return verified
```

**Example 3: Neuro-Symbolic Retrieval for Compliance Queries**

User: "Build a retrieval system for legal documents where queries like 'contracts signed after 2023 that do NOT contain an arbitration clause' must return precisely correct results."

Approach:
1. This requires constraint enforcement with correctness guarantees — select the neuro-symbolic family.
2. Use the LLM to parse the query into a logical form.
3. Retrieve candidate contracts using semantic search for broad recall.
4. Apply symbolic verification to enforce the exact constraints.

Output:
```python
from datetime import date
from z3 import Solver, Int, Bool, sat

def parse_to_logical_form(query: str, llm) -> dict:
    """LLM converts NL query to structured logical form."""
    result = llm.generate_json(
        f"Convert to logical constraints: {query}\n"
        f"Fields: signed_date (ISO date), has_arbitration_clause (bool), "
        f"document_type (string). Return JSON."
    )
    # {"document_type": "contract", "signed_date_after": "2023-01-01",
    #  "has_arbitration_clause": false}
    return result

def symbolic_verify(doc_metadata: dict, constraints: dict) -> bool:
    """Verify document metadata against parsed constraints using Z3."""
    s = Solver()
    # Date constraint
    if "signed_date_after" in constraints:
        threshold = date.fromisoformat(constraints["signed_date_after"])
        doc_date = date.fromisoformat(doc_metadata["signed_date"])
        if doc_date <= threshold:
            return False
    # Boolean exclusion constraint
    if "has_arbitration_clause" in constraints:
        if doc_metadata.get("has_arbitration_clause") != constraints["has_arbitration_clause"]:
            return False
    # Type constraint
    if "document_type" in constraints:
        if doc_metadata.get("document_type") != constraints["document_type"]:
            return False
    return True

def neurosymbolic_retrieval(query, retriever, llm, metadata_store, top_k=50):
    constraints = parse_to_logical_form(query, llm)
    candidates = retriever.search(query, top_k=top_k)
    verified = [
        doc for doc in candidates
        if symbolic_verify(metadata_store[doc.id], constraints)
    ]
    return verified
```

## Best Practices

- **Do**: Always separate retrieval from reasoning. Retrieve broadly for recall, then reason precisely for precision. This two-stage pattern is more robust than trying to embed reasoning into the retrieval step itself.
- **Do**: Make the reasoning trace visible. Every multi-hop step, every constraint check, every re-ranking decision should be logged. This is essential for debugging and for user trust.
- **Do**: Use the three-axis evaluation (representation, inference, compute) when choosing between approaches. Write down the scores. A 30-second analysis prevents weeks of building the wrong system.
- **Do**: Test with adversarial negation pairs. For any retrieval system, check whether swapping "with X" and "without X" changes the results. If it doesn't, your system has a negation blindness problem.
- **Avoid**: Concatenating all retrieved passages into a single LLM context without structure. This loses provenance and makes synthesis unreliable. Instead, extract structured facts per passage and reason over the structure.
- **Avoid**: Assuming embedding similarity handles logical operators. Cosine similarity has no concept of AND, OR, NOT, or temporal ordering. If your query requires these, you need explicit constraint handling.

## Error Handling

- **Sub-query loops**: IRCoT can loop if the LLM generates the same sub-query repeatedly. Implement a visited-queries set and a maximum hop limit (3-5 is typical). If the loop is detected, fall back to single-hop retrieval with a warning.
- **Constraint parse failures**: The LLM may fail to extract constraints correctly from ambiguous queries. Always validate the parsed constraints with the user or implement a confidence threshold. When confidence is low, return unfiltered results with a note that constraint enforcement could not be applied.
- **Empty result sets after filtering**: Strict constraint verification can filter out all candidates. Implement a degradation strategy: first relax the softest constraint, then expand the retrieval pool, then inform the user that no documents satisfy all constraints simultaneously.
- **Symbolic solver timeouts**: Z3 or Prolog queries can be slow on complex constraint sets. Set a per-query timeout (e.g., 500ms) and fall back to heuristic filtering if the solver times out.
- **Contradictory evidence across documents**: When synthesizing multi-document evidence, two sources may disagree. Surface the contradiction explicitly rather than silently picking one. Let the user or a downstream system resolve it.

## Limitations

- **Latency**: Multi-hop IRCoT pipelines multiply retrieval and LLM calls. A 3-hop pipeline is roughly 3x the latency of single-hop. This is unsuitable for real-time autocomplete or sub-100ms retrieval requirements.
- **Metadata dependency**: Neuro-symbolic constraint verification requires structured metadata (dates, categories, Boolean flags). If your corpus lacks metadata, you must extract it first — which introduces its own error rate.
- **Scale ceiling for symbolic methods**: Symbolic verification is per-document. At millions of candidates, you must aggressively pre-filter with fast retrieval before applying symbolic checks. The symbolic stage is a precision tool, not a recall tool.
- **LLM parsing reliability**: The quality of the entire pipeline depends on the LLM's ability to correctly decompose queries and extract constraints. Ambiguous queries will produce unreliable parses. Domain-specific fine-tuning or few-shot examples significantly improve this.
- **No single family solves everything**: The paper's central lesson is that each methodological family has fundamental tradeoffs. A system handling both hierarchical taxonomy queries and temporal constraint queries may need to combine geometric embeddings with symbolic verification — increasing architectural complexity.

## Reference

**Paper**: Hoveyda, M., Efstratiadis, P., de Vries, A., & de Rijke, M. (2026). *Tutorial on Reasoning for IR & IR for Reasoning*. [arXiv:2602.03640](https://arxiv.org/abs/2602.03640v1)

**What to look for**: The unified analytical framework mapping five methodological families across three evaluation axes (representational adequacy, inference mechanisms, computational viability). Section on reasoning challenges in IR gives the clearest taxonomy of retrieval failures that motivate the approach. The synthesis section provides guidance on combining families for hybrid systems.