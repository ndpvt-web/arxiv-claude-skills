---
name: "papersearchqa-learning-search-reason"
description: "Build iterative search-and-reason agents for scientific literature QA. Uses the PaperSearchQA pattern: interleaved thinking, querying, and verification loops over document corpora. Trigger phrases: 'search scientific papers', 'build a paper search agent', 'scientific QA pipeline', 'iterative retrieval agent', 'RLVR search agent', 'reason over research papers'"
---

# PaperSearchQA: Iterative Search-Reason Agents for Scientific Literature

This skill teaches Claude to build **search-reason agents** — systems that iteratively formulate queries, retrieve scientific documents, reason over results, and self-verify answers before responding. Based on the PaperSearchQA method (Burgess et al., EACL 2026), the core insight is that agents trained with reinforcement learning on verifiable rewards (RLVR) learn emergent behaviors — planning search queries, leveraging parametric knowledge before retrieval, and self-verifying answers — that outperform static RAG pipelines by 10-15 accuracy points on technical QA. This skill applies that architecture pattern to build practical search agents over any document corpus.

## When to Use

- When the user wants to build a QA system that searches over a corpus of scientific papers or technical documents
- When the user needs an agent that can perform multi-step retrieval — where the first search informs subsequent queries
- When building a RAG pipeline and the user wants to upgrade from single-shot retrieval to iterative search-reason loops
- When implementing a literature review agent that must find, cross-reference, and synthesize information from multiple papers
- When the user wants to construct a factoid QA dataset from a document corpus for training or evaluation
- When building an "AI Scientist" assistant that answers technical questions with cited evidence from a paper collection

## Key Technique

**The Search-Reason Loop.** Unlike standard RAG (retrieve once, generate once), PaperSearchQA agents operate in an interleaved loop: `<think>` to reason, `<query>` to search, receive `<information>`, then repeat as needed before producing an `<answer>`. The agent decides *when* to search, *what* to search for, and *when it has enough evidence* to answer. This is implemented by intercepting generation at `<query>` tokens, executing retrieval, injecting results as `<information>` blocks, and resuming generation. The agent learns to decompose complex questions into targeted sub-queries.

**RLVR Training with GRPO.** The agent is trained using Group Relative Policy Optimization (GRPO), where the only reward signal is whether the final answer matches the ground truth (exact match after normalization). No intermediate supervision is needed — the agent discovers effective search strategies through exploration. This is key: you don't need to label which documents are relevant or which queries are good. You only need question-answer pairs. The training uses a KL divergence penalty against a reference model to prevent reward hacking.

**Emergent Behaviors.** Trained agents exhibit three behaviors not explicitly taught: (1) *Planning* — extracting keywords and formulating targeted queries before searching, (2) *Pre-search reasoning* — using parametric knowledge to narrow the search space, and (3) *Self-verification* — performing additional searches to confirm an answer the agent already suspects is correct. These behaviors emerge naturally from RLVR and are what drive the performance gains over static retrieval.

## Step-by-Step Workflow

### Building an Iterative Search-Reason Agent

1. **Prepare the document corpus.** Index your documents (papers, abstracts, technical docs) into both a sparse retriever (BM25 via Pyserini or Elasticsearch) and optionally a dense retriever (e5, Contriever, or similar). Concatenate titles with body text for each document. For a PubMed-scale corpus (16M abstracts, ~23GB text), BM25 produces a 2.6GB index; dense embeddings require ~93GB.

2. **Define the agent interaction format.** Implement the token-delimited generation loop with four XML-style tags:
   - `<think>...</think>` — agent's internal reasoning (visible for debugging, not shown to users)
   - `<query>...</query>` — search query the agent wants to execute
   - `<information>...</information>` — retrieved documents injected by the system
   - `<answer>...</answer>` — the agent's final response

3. **Implement the generation-interruption loop.** During generation, monitor output tokens for `<query>`. When detected: (a) stop generation, (b) extract the query string, (c) retrieve top-k documents (k=3 to 5 is typical), (d) format results as `<information>` blocks with document titles and text, (e) append to the context, (f) resume generation. Allow multiple search iterations per question.

4. **Write a minimal system prompt.** Keep it sparse — describe the task ("answer questions using search"), explain the tag syntax, and set the output format. Do *not* over-specify search strategies; the value of this approach is that the agent learns its own strategies. Example:
   ```
   You are a scientific QA agent. Think step-by-step inside <think> tags.
   Search for papers using <query>your search terms</query>.
   Retrieved papers will appear in <information> tags.
   When ready, give your final answer in <answer> tags.
   ```

5. **Construct a QA dataset for training/evaluation.** Sample documents from your corpus, use an LLM (GPT-4 or Claude) to generate factoid question-answer pairs across defined categories (e.g., methods, results, mechanisms, comparisons). Generate answer synonyms to improve evaluation robustness. Paraphrase ~50% of questions to test generalization. Target at least 50k training and 5k test samples.

6. **Implement answer normalization for evaluation.** Normalize both predictions and gold answers: lowercase, strip whitespace, remove articles ("a", "an", "the"), and compare against all answer synonyms. Use exact match as the primary metric — it's strict but provides a clean reward signal for RL.

7. **Train with GRPO (if fine-tuning).** Using the veRL framework or Search-R1 codebase: set batch size 512, run ~150 GRPO steps with a binary reward (1 if normalized answer matches any gold synonym, 0 otherwise). Apply KL penalty against the base model. Train on 8 GPUs for models in the 3B-7B range. Monitor for decreasing behavioral diversity after extended training.

8. **For inference-only deployments, implement the loop with a frontier LLM.** You don't need RLVR training to use this pattern. Implement the search-reason loop with Claude or GPT-4 as the backbone, using the system prompt from step 4 and the generation-interruption mechanism from step 3. The structured loop alone yields significant gains over single-shot RAG.

9. **Add self-verification.** Instruct (or let the agent learn to) perform a confirmation search after generating a candidate answer. This catches errors from parametric knowledge that contradict the corpus and is one of the highest-value emergent behaviors observed in trained agents.

10. **Evaluate and iterate.** Compare against baselines: direct answering (no retrieval), chain-of-thought only, single-shot RAG, and the full search-reason loop. Expect the iterative agent to outperform RAG by 10-15 points on exact match for domain-specific QA.

## Concrete Examples

**Example 1: Building a search-reason loop for a biomedical QA system**

User: "I have 500k PubMed abstracts indexed in Elasticsearch. Build me a search agent that can answer technical biomedical questions by iteratively searching and reasoning."

Approach:
1. Define the agent format with `<think>`, `<query>`, `<information>`, `<answer>` tags
2. Implement an Elasticsearch retrieval function that takes a query string and returns top-5 abstracts
3. Build the generation loop that intercepts `<query>` tokens and injects search results
4. Write the minimal system prompt
5. Wire it together as a Python class with a `run(question: str) -> str` method

Output:
```python
import openai  # or anthropic
from elasticsearch import Elasticsearch

class SearchReasonAgent:
    def __init__(self, es_client: Elasticsearch, index: str, model: str, max_turns: int = 5):
        self.es = es_client
        self.index = index
        self.model = model
        self.max_turns = max_turns
        self.system_prompt = (
            "You are a biomedical QA agent. Reason inside <think> tags. "
            "Search using <query>terms</query>. Results appear in <information> tags. "
            "Give your final answer in <answer> tags. Be precise and cite evidence."
        )

    def retrieve(self, query: str, k: int = 5) -> str:
        results = self.es.search(index=self.index, query={"match": {"text": query}}, size=k)
        docs = []
        for hit in results["hits"]["hits"]:
            title = hit["_source"].get("title", "Untitled")
            abstract = hit["_source"]["abstract"]
            docs.append(f"[{title}]\n{abstract}")
        return "\n\n".join(docs)

    def run(self, question: str) -> str:
        messages = [
            {"role": "system", "content": self.system_prompt},
            {"role": "user", "content": question},
        ]
        for turn in range(self.max_turns):
            response = self._generate(messages)
            if "<answer>" in response:
                return self._extract_tag(response, "answer")
            if "<query>" in response:
                query = self._extract_tag(response, "query")
                docs = self.retrieve(query)
                # Append agent output + retrieved info, continue generation
                messages.append({"role": "assistant", "content": response})
                messages.append({"role": "user", "content": f"<information>{docs}</information>"})
            else:
                break
        return self._extract_tag(response, "answer") if "<answer>" in response else response
```

**Example 2: Constructing a factoid QA dataset from a paper corpus**

User: "I have a corpus of 10k ML papers. Help me generate a QA evaluation dataset to test retrieval agents."

Approach:
1. Define QA categories relevant to ML: methods, architectures, datasets, metrics, ablation results, hyperparameters
2. Sample 200 abstracts per category
3. Prompt an LLM to generate factoid QA pairs from each abstract
4. Generate answer synonyms for evaluation robustness
5. Paraphrase 50% of questions to test generalization

Output:
```python
CATEGORIES = [
    "model architecture details",
    "training methodology and optimization",
    "dataset characteristics and benchmarks",
    "quantitative results and metrics",
    "ablation study findings",
    "hyperparameter configurations",
]

QA_GENERATION_PROMPT = """Given this paper abstract, generate a factoid question-answer pair
in the category "{category}".

Requirements:
- The answer must be a short phrase (1-5 words) found in the abstract
- The question should require understanding the abstract to answer
- Generate 2-3 answer synonyms for evaluation

Abstract: {abstract}

Output JSON: {{"question": "...", "answer": "...", "synonyms": ["...", "..."], "category": "..."}}"""

PARAPHRASE_PROMPT = """Rephrase this question to ask the same thing differently.
Keep it natural and domain-appropriate. Do not change the expected answer.
Original: {question}
Paraphrased:"""
```

**Example 3: Upgrading a single-shot RAG pipeline to iterative search-reason**

User: "My current RAG pipeline retrieves 5 documents and generates an answer in one shot. How do I upgrade it to use iterative search-reason?"

Approach:
1. Replace single-shot generation with a loop that checks for `<query>` tags
2. Allow the model to search multiple times before answering
3. Add the self-verification step
4. Keep the same retriever — only the generation logic changes

Output:
```python
# BEFORE: Single-shot RAG
def answer_question_rag(question, retriever, llm):
    docs = retriever.search(question, k=5)
    context = "\n".join(docs)
    return llm.generate(f"Context: {context}\n\nQuestion: {question}\n\nAnswer:")

# AFTER: Iterative search-reason loop
def answer_question_search_reason(question, retriever, llm, max_turns=5):
    system = ("Reason in <think> tags. Search with <query>terms</query>. "
              "Results appear in <information> tags. Answer in <answer> tags.")
    messages = [{"role": "system", "content": system},
                {"role": "user", "content": question}]

    for _ in range(max_turns):
        output = llm.generate(messages)
        if "<answer>" in output:
            return extract_between_tags(output, "answer")
        if "<query>" in output:
            query = extract_between_tags(output, "query")
            docs = retriever.search(query, k=5)
            formatted = "\n\n".join(f"[Doc {i+1}] {d}" for i, d in enumerate(docs))
            messages.append({"role": "assistant", "content": output})
            messages.append({"role": "user", "content": f"<information>{formatted}</information>"})
    return "Could not determine answer within search budget."
```

The key difference: the agent decides *what* to search for based on its reasoning, not just the original question. This lets it decompose complex questions, follow citation chains, and verify uncertain answers.

## Best Practices

- **Do:** Keep the system prompt minimal. Over-specifying search strategies prevents the agent from discovering better ones. The paper found that minimal prompts with just tag syntax descriptions yielded the best emergent behaviors.
- **Do:** Concatenate document titles with abstracts/body text in your index. This consistently improves retrieval quality for both BM25 and dense retrievers.
- **Do:** Generate answer synonyms when building evaluation datasets. A single canonical answer misses valid equivalent responses (e.g., "BERT" vs "Bidirectional Encoder Representations from Transformers").
- **Do:** Cap search turns (3-5 is usually sufficient). Unbounded search wastes tokens without improving accuracy after diminishing returns.
- **Avoid:** Using the agent's search queries as ground-truth retrieval labels. The queries are a means to an end — reward only the final answer correctness.
- **Avoid:** Training RLVR for too many steps. The paper observed decreasing behavioral diversity with extended training — the agent converges to fewer strategies. Monitor for this and stop early if needed.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Agent loops searching without answering | No max turn limit or question is unanswerable from corpus | Set `max_turns` (3-5), add fallback that forces `<answer>` after budget exhaustion |
| Agent ignores retrieval results | Context window overflow from too many retrieved docs | Reduce k (top-k documents) or truncate abstracts; keep total retrieval context under 2k tokens per turn |
| Exact match scores are misleadingly low | Answer normalization is too strict or synonyms are missing | Add comprehensive synonym lists, use fuzzy matching as a secondary metric, normalize punctuation and abbreviations |
| Agent hallucinates instead of searching | System prompt is too permissive or model is too confident | Add explicit instruction: "Always search before answering. Do not rely on memory alone." |
| BM25 retriever misses relevant papers | Query uses different terminology than the indexed documents | Combine BM25 with a dense retriever (hybrid search), or have the agent reformulate queries when initial results are poor |

## Limitations

- **Exact match evaluation is brittle.** Many correct answers fail exact match due to formatting differences, abbreviations, or paraphrasing. Synonym lists help but don't fully solve this. Consider adding LLM-as-judge evaluation for production systems.
- **Corpus coverage gaps.** The agent can only find what's in the index. If the answer requires information outside the corpus, the agent will either hallucinate or fail. Make corpus coverage explicit to users.
- **RLVR training requires significant compute.** Training a 7B model takes ~30 hours on 8 A100s. For most practical applications, using the search-reason loop pattern with a frontier LLM (no fine-tuning) is more cost-effective.
- **Paraphrased questions are harder.** The paper showed a 12-point accuracy drop on paraphrased vs. original questions (44.9 vs 57.2). Agents trained on original question phrasings may struggle with naturally diverse user queries.
- **Domain transfer is not free.** The dataset construction pipeline is general, but you need domain-specific QA categories and a relevant corpus for each new field. The architecture transfers; the data does not.

## Reference

**Paper:** [PaperSearchQA: Learning to Search and Reason over Scientific Papers with RLVR](https://arxiv.org/abs/2601.18207v1) (Burgess et al., EACL 2026)

**What to look for:** Section 3 for the search-reason loop architecture and GRPO training setup. Section 4 for dataset construction pipeline. Section 5.3 and Figure 4 for emergent agent behaviors (planning, pre-search reasoning, self-verification). Appendix F for the minimal system prompt template.

**Resources:** [HuggingFace Collection](https://huggingface.co/collections/jmhb/papersearchqa) — corpus, datasets, and benchmarks compatible with the Search-R1 codebase.