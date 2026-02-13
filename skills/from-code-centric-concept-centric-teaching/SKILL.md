---
name: "from-code-centric-concept-centric-teaching"
description: "Generate LLM-assisted coding labs that teach concepts through 'Vibe Coding' — producing working code paired with mandatory conceptual reflection, prompt logging, and critical thinking assessments. Use when: 'create a vibe coding lab for transformers', 'design an NLP exercise with reflection questions', 'build a concept-focused coding tutorial', 'generate a lab that teaches X concept not just syntax', 'create a coding assignment with critical reflection', 'design a learn-by-prompting exercise'."
---

# From Code-Centric to Concept-Centric: Vibe Coding Skill

This skill enables Claude to design and deliver **Vibe Coding** learning experiences — a pedagogical method from Al-Khalifa (2026) where LLM-generated code serves as a vehicle for conceptual mastery rather than an end in itself. Instead of asking learners to write code from scratch (and get stuck on syntax), Claude generates working implementations and then guides the learner through structured reflection questions, prompt logging analysis, and concept-probing modifications. The result: learners spend cognitive effort on *understanding why* rather than *debugging what*.

## When to Use

- When a user asks to create a coding lab, tutorial, or exercise for teaching a technical concept (NLP, ML, data science, web dev, systems programming, etc.)
- When a user wants to learn a new library or framework conceptually without getting bogged down in boilerplate
- When a user says "teach me about X" or "help me understand X" for a programming-adjacent topic
- When designing coursework, workshops, or onboarding materials that pair code with comprehension checks
- When a user wants to explore a codebase or algorithm by having Claude generate it and then asking questions about it
- When building self-study materials that go beyond "copy this code" tutorials
- When a user explicitly asks for "vibe coding", "concept-focused coding", or "reflection-based labs"

## Key Technique

**Vibe Coding** inverts the traditional learn-to-code workflow. In conventional instruction, students write code to prove they understand concepts. In Vibe Coding, the LLM writes the code and the student proves understanding through three assessment channels: (1) **prompt logging** — documenting how they directed the LLM and why they made specific requests, (2) **critical reflection** — answering structured questions that probe conceptual understanding of the generated code, and (3) **modification exercises** — making targeted changes that require understanding the code's architecture, not just its syntax.

The core insight is that debugging syntax errors and fighting import paths consumes cognitive bandwidth that could be spent on understanding attention mechanisms, loss functions, or architectural tradeoffs. By offloading implementation to the LLM, the learner's limited attention is redirected toward the *concepts the code embodies*. This is not "letting the AI do the work" — the reflection and modification components are harder than writing the code, because they require genuine understanding.

The method works because it structures three layers of accountability: the prompt log shows the learner's *intent*, the reflection questions test their *comprehension*, and the modification tasks prove their *transfer ability*. A learner who cannot explain why a particular loss function was chosen, or what happens when you remove an attention head, has not learned — regardless of whether the code runs.

## Step-by-Step Workflow

1. **Identify the target concept.** Determine the core idea the learner should master (e.g., "self-attention in transformers", "TF-IDF weighting", "backpropagation through time"). Separate the concept from the implementation details.

2. **Generate a working, well-commented implementation.** Write complete, runnable code that demonstrates the concept. Include inline comments that label conceptual components (e.g., `# Query, Key, Value projections — the core of attention`). Use realistic data or provide sample data inline.

3. **Annotate the code with concept markers.** Add `# CONCEPT:` comments at critical junctures that link code to theory. For example: `# CONCEPT: Softmax here normalizes attention weights to sum to 1, creating a probability distribution over input tokens`.

4. **Write 4-6 critical reflection questions.** These must test understanding, not recall. Use question types:
   - **Explain-why**: "Why does this implementation use layer normalization before attention rather than after?"
   - **Predict-change**: "What would happen to the model's behavior if we replaced the softmax with a ReLU in the attention computation?"
   - **Connect-theory**: "How does the masking step in this code relate to the autoregressive property described in Vaswani et al.?"
   - **Identify-limitation**: "What class of inputs would cause this tokenizer to produce poor results, and why?"

5. **Design 2-3 targeted modification exercises.** Each modification should require conceptual understanding to complete correctly. Specify what to change, what the expected behavioral difference is, and what concept the modification tests. Example: "Replace the learned positional embeddings with sinusoidal encodings. Run both versions on the sample input and explain why the outputs differ."

6. **Create the prompt log template.** Provide a structured format for the learner to record: (a) the prompt they gave the LLM, (b) what they were trying to achieve conceptually, (c) whether the output matched their expectation, and (d) how they refined the prompt and why.

7. **Include a concept map or dependency note.** State what prerequisite concepts the learner should already know, and what concepts this lab prepares them for next. This situates the lab in a learning progression.

8. **Provide verification checkpoints.** Give the learner concrete ways to verify both that the code works (expected output snippets) and that their conceptual understanding is correct (key insights they should have reached by the end).

9. **Add an "explore further" section.** List 2-3 extensions that the learner could pursue using the same vibe coding approach — prompting the LLM for variants and then analyzing the differences conceptually.

## Concrete Examples

**Example 1: Teaching Self-Attention via Vibe Coding**

User: "Create a vibe coding lab that teaches self-attention in transformers"

Approach:
1. Generate a standalone self-attention implementation in PyTorch (~40 lines)
2. Add CONCEPT markers at Q/K/V projection, scaled dot-product, softmax, and output projection
3. Write reflection questions probing understanding of each component
4. Design modifications that test transfer

Output structure:

```python
# === VIBE CODING LAB: Self-Attention Mechanism ===
# Goal: Understand how self-attention computes contextual representations
# Prerequisites: Matrix multiplication, softmax function, embeddings

import torch
import torch.nn.functional as F

def self_attention(x, W_q, W_k, W_v):
    """
    Compute self-attention for input sequence x.
    x: (batch, seq_len, d_model)
    """
    # CONCEPT: Each token gets three representations — what it's looking for (Q),
    # what it offers (K), and what it actually contains (V)
    Q = x @ W_q  # Queries
    K = x @ W_k  # Keys
    V = x @ W_v  # Values

    d_k = Q.shape[-1]

    # CONCEPT: Dot product measures similarity between queries and keys.
    # Scaling by sqrt(d_k) prevents softmax from saturating for large dimensions.
    scores = (Q @ K.transpose(-2, -1)) / (d_k ** 0.5)

    # CONCEPT: Softmax converts raw scores to a probability distribution.
    # Each token now has a weighted "attention" over all other tokens.
    attn_weights = F.softmax(scores, dim=-1)

    # CONCEPT: The output for each token is a weighted sum of all Value vectors,
    # where weights reflect relevance determined by Q-K similarity.
    output = attn_weights @ V
    return output, attn_weights

# --- Sample usage with synthetic data ---
torch.manual_seed(42)
batch, seq_len, d_model = 1, 4, 8
x = torch.randn(batch, seq_len, d_model)
W_q = torch.randn(d_model, d_model)
W_k = torch.randn(d_model, d_model)
W_v = torch.randn(d_model, d_model)

output, weights = self_attention(x, W_q, W_k, W_v)
print("Attention weights shape:", weights.shape)  # (1, 4, 4)
print("Attention weights (row sums should be 1.0):")
print(weights.squeeze().sum(dim=-1))  # Should print tensor of 1.0s
```

```markdown
## Critical Reflection Questions

1. **Explain-why**: Why do we divide by sqrt(d_k) before applying softmax?
   What would happen to the attention distribution without this scaling
   as d_k increases to 512 or 1024?

2. **Predict-change**: If you replaced `F.softmax(scores, dim=-1)` with
   `F.relu(scores)`, would the attention mechanism still work? What
   property of the output would be lost, and why does it matter?

3. **Connect-theory**: The attention weight matrix has shape (seq_len, seq_len).
   Each row sums to 1. What does row i, column j represent in plain language?
   How does this relate to the concept of "contextual embeddings"?

4. **Identify-limitation**: This implementation has no masking. For what
   NLP task would this be a problem, and what would go wrong specifically?

## Modification Exercises

**Mod 1**: Add a causal mask so that position i can only attend to positions
<= i. Print the attention weights and verify the upper triangle is zero.
*Concept tested*: Autoregressive generation and information leakage.

**Mod 2**: Split Q, K, V into 2 attention heads (d_model/2 each), compute
attention separately, and concatenate. Compare the attention patterns
of each head on the same input.
*Concept tested*: Why multi-head attention captures diverse relationships.
```

**Example 2: Teaching TF-IDF for Information Retrieval**

User: "Help me design a concept-focused tutorial on TF-IDF"

Approach:
1. Generate TF-IDF from scratch (no sklearn) to make each step visible
2. Apply it to a small document collection with interpretable results
3. Probe understanding of term frequency vs. inverse document frequency tradeoff

Output structure:

```python
# === VIBE CODING LAB: TF-IDF from Scratch ===
# Goal: Understand why TF-IDF works better than raw term frequency
# Prerequisites: Basic probability, logarithms

import math
from collections import Counter

documents = [
    "the cat sat on the mat",
    "the dog sat on the log",
    "cats and dogs are common pets",
]

# CONCEPT: Term frequency measures how important a word is WITHIN a document.
# A word that appears 5 times in a doc is more relevant to that doc than
# one appearing once.
def term_frequency(term, document):
    words = document.split()
    return words.count(term) / len(words)

# CONCEPT: Inverse document frequency measures how DISTINCTIVE a word is
# across the entire collection. Words appearing in every document (like "the")
# get low IDF; rare words get high IDF.
def inverse_document_frequency(term, all_docs):
    num_docs_containing = sum(1 for doc in all_docs if term in doc.split())
    if num_docs_containing == 0:
        return 0
    return math.log(len(all_docs) / num_docs_containing)

# CONCEPT: TF-IDF = TF * IDF. This balances local importance (TF) with
# global distinctiveness (IDF). Common words get suppressed; distinctive
# words get boosted.
def tfidf(term, document, all_docs):
    return term_frequency(term, document) * inverse_document_frequency(term, all_docs)

# Compute TF-IDF for key terms
for term in ["the", "cat", "pets"]:
    for i, doc in enumerate(documents):
        score = tfidf(term, doc, documents)
        print(f"TF-IDF('{term}', doc{i}) = {score:.4f}")
    print()
```

```markdown
## Critical Reflection Questions

1. **Explain-why**: "the" appears in docs 0 and 1 but its TF-IDF is 0.0000
   for doc 2. Meanwhile "pets" has a high score for doc 2. Explain the
   mathematical reason AND the intuitive reason this is desirable for search.

2. **Predict-change**: If you added 100 more documents that all contain
   the word "cat", what would happen to the TF-IDF score of "cat" in doc 0?
   What does this tell you about TF-IDF's behavior with corpus growth?

3. **Connect-theory**: Search engines moved beyond TF-IDF to neural
   retrieval. What fundamental limitation of TF-IDF does neural retrieval
   address? (Hint: consider synonyms.)

## Modification Exercise

Change the IDF formula to use `log(1 + N/df)` instead of `log(N/df)`.
Recompute scores and explain what practical problem the +1 smoothing solves.
```

**Example 3: Generating a Prompt Log Template**

User: "I want to learn about CNNs for text classification using vibe coding"

Claude provides the code lab, then includes this prompt log template:

```markdown
## Prompt Log (fill this in as you work)

| # | Your Prompt to the LLM | Conceptual Goal | Output Match Expectation? | Refinement & Why |
|---|------------------------|-----------------|--------------------------|------------------|
| 1 | "Write a 1D CNN for text classification in PyTorch" | Get a baseline implementation to study conv-over-text architecture | Partially — used 2D conv instead of 1D | Asked specifically for Conv1d with embedding input shape |
| 2 | "Add multiple filter sizes (3, 4, 5) like the Kim 2014 paper" | Understand how different n-gram windows capture different features | Yes — parallel convolutions with concat | N/A |
| 3 | ... | ... | ... | ... |

### Prompt Log Reflection
After completing the lab, answer: What did your prompt refinements reveal
about your evolving understanding of the concept? Which misconceptions
did prompting help you identify?
```

## Best Practices

- **Do**: Always separate working code from conceptual assessment. The code should run; the learning happens in the reflection.
- **Do**: Write reflection questions that have no single correct syntax-level answer — they should require reasoning about behavior, tradeoffs, and design choices.
- **Do**: Make modification exercises targeted and falsifiable. The learner should be able to verify whether their modification produced the expected conceptual effect.
- **Do**: Include `# CONCEPT:` annotations in generated code so the learner can map code regions to ideas even if they struggle with the language.
- **Avoid**: Generating trivially simple code that requires no conceptual depth to understand. The code should be complex enough that reflection is genuinely necessary.
- **Avoid**: Writing reflection questions that can be answered by copying the code comments back. Questions must require synthesis or prediction.
- **Avoid**: Skipping the prompt log component. The prompt log is what distinguishes vibe coding from "just using ChatGPT" — it creates metacognitive accountability.

## Error Handling

- **Learner submits code-only answers without reflection**: Redirect them. Explain that in vibe coding, the code is the *starting point*, not the deliverable. The reflection answers are the actual assessment.
- **Generated code doesn't run**: Fix it immediately. The entire pedagogical value depends on the learner being able to experiment with working code. Non-functional code wastes cognitive load on debugging — the exact problem vibe coding is designed to eliminate.
- **Reflection questions are too easy**: If a learner can answer all questions by reading the code comments alone, the questions need to be rewritten to require prediction, comparison, or transfer to novel scenarios.
- **Learner copies LLM explanations into reflections**: Design questions that require referencing the specific code and data in the lab, not generic explanations. Ask "What does row 2, column 3 of the attention matrix in *this* run mean?" not "What is attention?"
- **Concept is too advanced for single lab**: Split into a sequence of labs with explicit dependency notes. Each lab should have a single primary concept with at most two secondary concepts.

## Limitations

- **Not suitable for teaching debugging itself.** If the learning objective is debugging, error diagnosis, or code review, vibe coding's removal of implementation struggle defeats the purpose.
- **Requires a runnable environment.** The learner needs to execute the code and experiment with modifications. Purely reading the code without running it loses the empirical verification that makes the reflection questions meaningful.
- **Does not build implementation fluency.** Learners who only do vibe coding may understand transformers conceptually but struggle to implement one from memory. Pair with traditional coding exercises if implementation skill is a goal.
- **Reflection quality depends on question quality.** Poorly written reflection questions reduce the method to "AI writes my homework." The questions are the hardest part to design well.
- **Scales best for conceptually rich domains.** Topics like NLP, ML, distributed systems, and compilers benefit most. Purely procedural tasks (CRUD operations, config file editing) have less conceptual depth to probe.

## Reference

Al-Khalifa, H. (2026). *From Code-Centric to Concept-Centric: Teaching NLP with LLM-Assisted "Vibe Coding."* arXiv:2602.01919v1. Accepted at Teaching NLP Workshop @ EACL 2026. Key takeaway: The three-component assessment structure (prompt log + critical reflection + modification exercises) is what makes LLM-assisted learning rigorous rather than passive.