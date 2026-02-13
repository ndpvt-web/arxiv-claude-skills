---
name: "closing-reasoning-gaps-clinical"
description: "Build systems that detect and fix reasoning gaps in LLM agents by comparing their chain-of-thought against reference reasoning, extracting structured discrepancies, and generating corrective instructions stored in a retrievable knowledge base. Use when: 'build a reasoning improvement pipeline', 'detect logic gaps in agent output', 'compare agent reasoning to expert reasoning', 'create a corrective knowledge base from reasoning errors', 'improve clinical decision support accuracy', 'patch reasoning with RAG-retrieved instructions'."
---

# Closing Reasoning Gaps with Differential Reasoning Learning

This skill enables Claude to implement the **Differential Reasoning Learning (DRL)** framework from Liu et al. (2026). DRL improves LLM-based agents — particularly in clinical and high-stakes domains — by systematically comparing an agent's chain-of-thought reasoning against reference rationales (expert-authored or from stronger models), extracting structured discrepancies as graph differences, converting those discrepancies into corrective natural-language instructions, and retrieving them at inference time via RAG to patch logic gaps. The technique is domain-adaptable: while the paper targets clinical reasoning, the graph-based discrepancy analysis applies to any scenario where reasoning fidelity matters.

## When to Use

- When building a pipeline that compares an LLM agent's reasoning trace against expert or gold-standard reasoning and generates improvement feedback
- When designing a corrective knowledge base that accumulates structured lessons from reasoning failures
- When the user wants to detect hallucinated reasoning steps, missing evidence, or broken inference chains in agent output
- When augmenting agent prompts at inference time with retrieved corrective instructions (RAG over a discrepancy KB)
- When implementing clinical decision support systems that must produce not just correct answers but valid reasoning paths
- When evaluating reasoning quality beyond final-answer accuracy — measuring structural fidelity of the logic chain
- When the user asks to "improve my agent's reasoning" or "find gaps in chain-of-thought output"

## Key Technique

DRL works in two stages. **Stage 1 (Knowledge Mining)** processes training examples: for each case, it extracts the agent's free-form chain-of-thought and a reference rationale, parses both into **reasoning graphs** — directed acyclic graphs (DAGs) where nodes are typed as Facts (observations with polarity: present/absent/uncertain), Hypotheses (conclusions with confidence levels), or Actions (interventions typed as TEST/TREAT/ASSESS/OBSERVE/PRESCRIBE), connected by typed edges (supports/contradicts/suggests_test). It then runs a **clinically weighted graph edit distance (GED)** analysis that decomposes discrepancies into three components: missing nodes (evidence the agent overlooked), hallucinated nodes (unsupported claims), and path discrepancies (broken or incorrect inference chains). Penalty weights escalate by clinical consequence: Facts=1.0x, Hypotheses=1.5x, Actions=2.0x. An LLM-as-a-judge performs semantic node alignment (handling paraphrases and equivalent medical concepts) before computing GED.

Each diagnosed discrepancy is fed to an **Insight Generator** that produces a structured corrective instruction containing: the error type, clinical context, what went wrong, why it matters, prevention steps, trigger keywords, and the underlying principle. These instructions populate the **Differential Reasoning Knowledge Base (DR-KB)**.

**Stage 2 (Inference)** retrieves the top-k most relevant DR-KB instructions for a new query via BM25 over trigger keywords and context, injects them into the agent prompt, and generates a prediction. This is a training-free improvement — no fine-tuning required, just prompt augmentation. Optimal k=5 in the paper's experiments, with diminishing returns beyond that.

## Step-by-Step Workflow

1. **Define the reasoning domain and node taxonomy.** Adapt the four node types (Facts, Hypotheses, Actions, Final) and three edge types (supports, contradicts, suggests_test) to your domain. For clinical: Facts are symptoms/vitals/labs; Hypotheses are diagnoses; Actions are interventions. For software debugging: Facts are error messages/logs; Hypotheses are root causes; Actions are fixes.

2. **Collect reference rationales.** Gather expert-authored reasoning traces, clinical guidelines, or outputs from a stronger model (e.g., GPT-4o reasoning on the same cases). These serve as the gold-standard reasoning graphs.

3. **Generate agent chain-of-thought traces.** Run your target agent on the same cases with explicit CoT prompting. Capture the full reasoning text.

4. **Extract reasoning graphs from both traces.** Use an LLM to parse each reasoning trace into a DAG with typed nodes and edges. Prompt the LLM to output structured JSON:
   ```json
   {
     "nodes": [
       {"id": "F1", "type": "fact", "content": "Patient has fever 39.2C", "polarity": "present"},
       {"id": "H1", "type": "hypothesis", "content": "Bacterial pneumonia", "confidence": "high"},
       {"id": "A1", "type": "action", "content": "Order chest X-ray", "action_type": "TEST"}
     ],
     "edges": [
       {"src": "F1", "dst": "H1", "type": "supports", "justification": "Fever suggests infection"}
     ]
   }
   ```

5. **Align nodes semantically using LLM-as-a-judge.** For each node in the reference graph, ask the LLM whether a semantically equivalent node exists in the agent graph. This handles paraphrases (e.g., "elevated WBC" vs "leukocytosis"). Output a mapping of aligned pairs and unmatched nodes on each side.

6. **Compute weighted graph edit distance.** From the alignment, classify discrepancies into three buckets:
   - `d_miss`: reference nodes absent from agent graph (weighted by type: Fact=1.0, Hypothesis=1.5, Action=2.0)
   - `d_halluc`: agent nodes with no reference counterpart (same weights)
   - `d_path`: edge-level differences — missing connections, reversed directions, wrong edge types
   - Total GED = sum of all weighted penalties (unnormalized)

7. **Generate corrective instructions from discrepancies.** Feed each discrepancy profile to an Insight Generator LLM with a structured output schema:
   ```json
   {
     "title": "Missing differential for drug-induced hepatitis",
     "error_type": "missing_hypothesis",
     "domain": "hepatology",
     "situation_context": "Patient with elevated ALT/AST and recent medication change",
     "what_went_wrong": "Agent failed to consider drug-induced liver injury despite new statin",
     "why_it_matters": "Drug-induced hepatitis is reversible if caught early; missing it leads to continued exposure",
     "prevention_steps": ["Always check medication history when liver enzymes are elevated", "Consider drug-induced causes before infectious workup"],
     "trigger_keywords": ["elevated ALT", "elevated AST", "new medication", "statin", "liver enzymes"],
     "principle": "Medication review is mandatory in any hepatic enzyme elevation workup"
   }
   ```

8. **Store instructions in the DR-KB.** Index each instruction by its trigger keywords and domain context. Use a BM25-compatible store (e.g., Elasticsearch, SQLite with FTS5, or an in-memory index).

9. **At inference, retrieve top-k instructions and augment the prompt.** For a new query, extract key clinical terms, retrieve the k=5 most relevant DR-KB instructions via BM25, and prepend them to the agent prompt as structured guidance:
   ```
   ## Reasoning Guidelines (retrieved from prior case analysis)
   1. When liver enzymes are elevated with recent medication changes, always consider drug-induced hepatitis before infectious causes.
   2. ...
   ```

10. **Evaluate both answer accuracy and reasoning fidelity.** Run the augmented agent, extract its reasoning graph, and compute GED against reference. Track both final-answer accuracy and GED reduction over iterations.

## Concrete Examples

**Example 1: Building a Clinical QA Reasoning Improvement Pipeline**

User: "I have a medical QA dataset with expert explanations. My LLM agent gets 60% accuracy. Build a pipeline to improve its reasoning."

Approach:
1. Parse expert explanations into reference reasoning graphs (DAGs with Fact/Hypothesis/Action nodes)
2. Run the agent on the same questions with CoT prompting, extract agent reasoning graphs
3. For each case, align nodes via LLM-as-a-judge, compute weighted GED
4. Generate corrective instructions for each discrepancy (e.g., "When a patient presents with chest pain and dyspnea, always evaluate for pulmonary embolism alongside cardiac causes")
5. Store 500 instructions in DR-KB indexed by trigger keywords
6. At inference, retrieve top-5 instructions per new question, inject into prompt

Output structure:
```
pipeline/
  extract_graphs.py      # LLM-based DAG extraction from CoT text
  align_and_ged.py       # Semantic node alignment + weighted GED computation
  generate_instructions.py  # Discrepancy-to-instruction conversion
  dr_kb/
    index.json           # BM25-indexed instruction store
    instructions/        # Individual instruction JSON files
  inference.py           # RAG retrieval + prompt augmentation + prediction
  evaluate.py            # Accuracy + GED metrics
```

**Example 2: Debugging an Agent That Hallucinates Reasoning Steps**

User: "My coding assistant sometimes invents plausible but wrong explanations for bugs. How do I systematically fix this?"

Approach:
1. Adapt the node taxonomy: Facts = error messages, stack traces, code behavior; Hypotheses = root cause theories; Actions = proposed fixes
2. Collect 50 cases where the agent hallucinated reasoning (you have the correct root causes)
3. Extract reasoning graphs from both agent output and correct explanations
4. Run GED analysis — hallucinated nodes will show up in `d_halluc`, missed evidence in `d_miss`
5. Generate instructions like: "When seeing a NullPointerException in a deserialization path, check for missing default constructors before assuming null input data"
6. Store in DR-KB, retrieve at inference to ground the agent's reasoning

Output (corrective instruction):
```json
{
  "title": "Grounding NPE diagnosis in deserialization contexts",
  "error_type": "hallucinated_hypothesis",
  "domain": "java_debugging",
  "what_went_wrong": "Agent attributed NPE to null user input, but actual cause was missing no-arg constructor required by Jackson",
  "prevention_steps": [
    "Check class constructors when NPE occurs during deserialization",
    "Verify serialization framework requirements before blaming input data"
  ],
  "trigger_keywords": ["NullPointerException", "deserialization", "Jackson", "ObjectMapper"]
}
```

**Example 3: Implementing the Graph Extraction Prompt**

User: "How do I prompt an LLM to extract a reasoning graph from free-text chain-of-thought?"

Approach — use this prompt template:
```
Extract a reasoning graph from the following chain-of-thought text.

Output a JSON object with:
- "nodes": array of objects with {id, type, content, metadata}
  - type must be one of: "fact", "hypothesis", "action", "final"
  - For facts: metadata includes "polarity" (present/absent/uncertain)
  - For hypotheses: metadata includes "confidence" (high/medium/low)
  - For actions: metadata includes "action_type" (TEST/TREAT/ASSESS/OBSERVE/PRESCRIBE)
- "edges": array of objects with {src, dst, type, justification}
  - type must be one of: "supports", "contradicts", "suggests_test"
  - src and dst are node IDs

Rules:
- Each distinct claim, observation, or conclusion is a separate node
- Edges must reflect the logical dependency stated or implied in the text
- Do not infer edges not supported by the text
- Use IDs like F1, F2 for facts; H1, H2 for hypotheses; A1, A2 for actions

Chain-of-thought text:
{cot_text}
```

## Best Practices

- **Do:** Weight discrepancy penalties by downstream consequence. Actions that could harm a patient (or break production code) deserve higher penalty than observational facts. The paper uses Facts=1.0x, Hypotheses=1.5x, Actions=2.0x — calibrate to your domain.
- **Do:** Use semantic alignment (LLM-as-a-judge) rather than exact string matching when comparing nodes. "Tachycardia" and "heart rate 120 bpm" are the same fact.
- **Do:** Cap retrieval at k=5 instructions per query. The paper shows diminishing returns beyond this, and too many instructions dilute the prompt signal.
- **Do:** Include trigger keywords in each instruction to enable precise BM25 retrieval. Generic instructions that match everything add noise.
- **Avoid:** Normalizing the GED score to [0,1]. The paper explicitly uses unnormalized sums so that cases with more errors produce proportionally larger scores — this preserves signal about error severity.
- **Avoid:** Using DRL instructions as a replacement for fine-tuning on correct answers. DRL patches reasoning structure; it does not teach domain knowledge the model lacks entirely.
- **Avoid:** Generating instructions from cases where the agent got the right answer for the wrong reasons. Always verify reference-agent alignment on both the final answer and the reasoning path.

## Error Handling

- **Graph extraction produces malformed JSON:** Validate the extracted graph against a schema before proceeding. Retry with a stricter prompt or a more capable model. Reject graphs with zero edges (indicates extraction failure, not simple reasoning).
- **Node alignment is ambiguous:** When the LLM-as-a-judge cannot determine equivalence, flag the pair as "uncertain" and exclude from GED computation. Do not force alignment — false matches corrupt the discrepancy signal.
- **DR-KB retrieval returns irrelevant instructions:** This usually means trigger keywords are too broad. Re-generate the instruction with more specific keywords, or add a secondary reranking step using embedding similarity on the full instruction text.
- **Agent performance degrades with retrieved instructions:** Reduce k or filter instructions by confidence score. Conflicting instructions (from edge cases in training data) can confuse the agent. Deduplicate the DR-KB by clustering similar instructions and keeping the highest-quality representative.
- **Domain taxonomy doesn't fit:** Not all reasoning domains map cleanly to Fact/Hypothesis/Action. Adapt the node types — for legal reasoning: Evidence/Argument/Ruling; for code review: Observation/Issue/Fix. The graph structure matters more than the specific labels.

## Limitations

- DRL requires reference rationales of reasonable quality. If expert reasoning is unavailable or inconsistent, the extracted discrepancies will be noisy and the DR-KB instructions unreliable.
- The framework addresses reasoning structure gaps, not knowledge gaps. If the base model lacks domain knowledge (e.g., rare diseases, obscure APIs), DRL instructions cannot compensate — the model needs fine-tuning or retrieval of factual content.
- Graph extraction via LLM is imperfect. Complex multi-paragraph reasoning may lose nuance when compressed into a DAG. Very long chains of thought (>2000 tokens) may need chunking.
- BM25 retrieval over trigger keywords is a simple baseline. For production systems, hybrid retrieval (BM25 + dense embeddings) would improve instruction relevance.
- The paper reports optimal results at k=5 for medical QA. Other domains or model sizes may have different sweet spots — always tune k on a held-out set.
- Evaluation of reasoning fidelity (via GED) requires human review for high-stakes deployments. Automated GED is a proxy, not a replacement for expert audit.

## Reference

**Paper:** Liu et al., "Closing Reasoning Gaps in Clinical Agents with Differential Reasoning Learning," arXiv:2602.09945v1 (2026).
**What to look for:** Section 3 for the full DRL algorithm (graph extraction, GED computation, instruction generation), Section 4 for the DR-KB schema, and Section 5 for ablation results showing the impact of retrieval depth k and reference rationale quality.