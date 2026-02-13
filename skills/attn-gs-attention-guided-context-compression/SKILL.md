---
name: "attn-gs-attention-guided-context-compression"
description: "Compress long user contexts (profiles, histories, documents) into concise, high-quality summaries using attention-guided importance marking. Use when asked to: 'compress this user profile', 'shorten this context for the prompt', 'reduce token usage for personalization', 'summarize interaction history preserving key signals', 'fit this long context into a token budget', 'extract the most relevant parts of this document for a query'."
---

# Attn-GS: Attention-Guided Context Compression

This skill enables Claude to compress long user contexts — interaction histories, user profiles, document collections, conversation logs — into compact, high-fidelity summaries that preserve the information most relevant to a downstream task. The core technique, from the Attn-GS paper, uses a two-stage mark-then-compress pipeline: first, identify which sentences carry the strongest task-relevant signals (the "marking" stage), then generate a compressed summary that prioritizes those marked sentences (the "compression" stage). This achieves near-full-context performance at up to 50x token reduction.

## When to Use

- When a user provides a long user profile, interaction history, or document collection and needs it compressed to fit a token budget (e.g., "I have 10K tokens of user history but only 200 tokens of context space")
- When building personalization prompts that must include user preferences distilled from extensive data
- When designing a retrieval-augmented system where retrieved context exceeds the prompt window and must be intelligently trimmed
- When the user asks to "compress this context," "reduce these tokens," or "extract what matters from this profile"
- When creating API pipelines where inference cost is proportional to input length and the user wants to minimize cost without losing quality
- When a user has multiple documents or conversation turns and needs a single compressed representation focused on a specific task or query

## Key Technique

**The Problem with Naive Compression.** Standard approaches to context compression — truncating to recent items, prompting an LLM to "summarize this" — treat the entire context as a flat blob. They don't know which parts matter for the downstream task. A generic summary of a user's movie review history might emphasize prolific reviewing habits when the actual task is predicting genre preferences. This mismatch causes information loss where it hurts most.

**Attention as an Importance Signal.** Attn-GS exploits a key insight: when an LLM processes a context alongside a task query, its internal attention patterns naturally highlight which input sentences are most relevant to producing the answer. Specifically, the attention weights from the final generated token back to the input sequence — averaged across attention heads at a middle layer — produce a reliable per-token importance score. These token scores are aggregated to sentence-level scores by averaging, then thresholded to identify the most task-relevant sentences. Middle layers (e.g., layer 6 in a 1B model, layer 12 in an 8B model) carry the strongest signal; early layers attend too broadly and late layers over-specialize.

**Mark, Then Compress.** The two-stage pipeline works as follows. The *marking model* processes the full context with the task query, extracts attention scores, and wraps high-scoring sentences with explicit importance markers (`<start_important>`, `<end_important>`). The *compression model* then receives this marked context and generates a compressed version under a target token budget, with explicit instructions to prioritize marked content. The markers act as a structured channel between the two stages — they transform implicit attention signals into explicit instructions that the compression model can follow reliably. Fine-tuning both models on task-specific data substantially improves performance: fine-tuned marking models distinguish relevant from irrelevant content far better than zero-shot attention, and fine-tuned compression models learn to respect markers faithfully.

## Step-by-Step Workflow

1. **Define the task and token budget.** Identify the downstream task (e.g., "predict the user's rating for this movie," "generate a personalized email") and the target compressed length (e.g., 200 tokens, 500 tokens). The compression ratio guides how aggressively to filter.

2. **Segment the context into sentences.** Split the full user context (profile, history, documents) into individual sentences or logical units. Each unit will receive its own importance score. For structured data (JSON, tables), treat each record or row as a unit.

3. **Score each sentence for task relevance.** Process the full context concatenated with the task query through a marking pass. For each sentence, compute an importance score. In Attn-GS, this is done by extracting attention weights from a middle layer of the LLM — but when implementing with Claude (where internal attention isn't directly accessible), simulate this by prompting Claude to rate each sentence's relevance to the specific task on a 0-10 scale, or by using a dedicated scoring prompt that asks "Which of these sentences would most help answer the query?"

4. **Apply a threshold to select important sentences.** Set a threshold at `alpha * max_score` where alpha is between 0.2 and 0.4. Sentences scoring above the threshold are marked as important. For a 50x compression target, expect roughly 5-15% of sentences to be marked. Adjust alpha to control the specificity: lower alpha marks more sentences (higher recall), higher alpha marks fewer (higher precision).

5. **Wrap important sentences with explicit markers.** Insert `<start_important>` before and `<end_important>` after each selected sentence in the original context. Preserve the original ordering — do not reorder sentences. This produces the "marked context."

6. **Generate the compressed context.** Pass the marked context to a compression prompt with instructions: "Compress this user context into at most [N] tokens. Prioritize sentences marked with `<start_important>`/`<end_important>` as they contain the most task-relevant information. Preserve specific details (names, numbers, preferences) from marked sentences. Omit or briefly summarize unmarked content."

7. **Validate the compressed output.** Check that (a) the output respects the token budget, (b) key details from marked sentences appear in the compression, and (c) the compression reads coherently. If the budget is exceeded, re-run with a tighter alpha or stricter length instruction.

8. **Plug the compressed context into the downstream prompt.** Replace the full context with the compressed version in the final task prompt. The compressed context should be a drop-in replacement — no structural changes to the downstream prompt are needed.

9. **Iterate if quality is insufficient.** If downstream task quality drops, lower the threshold (mark more sentences), increase the token budget, or add task-specific instructions to the compression prompt. The marking-compression pipeline is modular — each stage can be tuned independently.

## Concrete Examples

**Example 1: Compressing a User Movie Profile for Rating Prediction**

```
User: I have a user's 50 movie reviews (about 8,000 tokens). I need to predict
their rating for "The Grand Budapest Hotel" but my prompt budget is 300 tokens.
Compress the profile to fit.

Approach:
1. Segment the 50 reviews into individual review units.
2. Score each review for relevance to predicting a rating for "The Grand Budapest
   Hotel" — a Wes Anderson comedy-drama. Prioritize reviews of:
   - Other Wes Anderson films (direct signal)
   - Comedy-dramas, quirky/stylized films (genre signal)
   - Reviews expressing strong positive/negative opinions (preference signal)
3. Apply threshold (alpha=0.3): mark ~5-8 of the 50 reviews as important.
4. Wrap marked reviews with <start_important>/<end_important> tags.
5. Compress with: "Summarize this user's movie preferences in under 300 tokens.
   Prioritize marked reviews. Include specific ratings and genre preferences."

Output:
"User strongly prefers visually stylized, character-driven films. Rated
Moonrise Kingdom 5/5 ('perfect Wes Anderson whimsy'), The Royal Tenenbaums 4/5.
Enjoys dark comedies (In Bruges 5/5, Fargo 4/5) but dislikes slow-paced dramas
(Lost in Translation 2/5). Appreciates ensemble casts and dry humor. Average
rating for comedies: 4.1/5. Dislikes excessive CGI and action-heavy plots."
```

**Example 2: Compressing Chat History for a Personalized Assistant**

```
User: I have 3 months of chat history with a user (12,000 tokens). I need to
build a system prompt that captures their preferences and communication style.
Target: 400 tokens.

Approach:
1. Segment chat history into individual messages/exchanges.
2. Score each exchange for preference-revealing content. High-scoring examples:
   - User correcting the assistant ("No, I prefer bullet points over paragraphs")
   - Explicit preference statements ("I always want code in Python, not JS")
   - Repeated patterns (user consistently asks for concise answers)
   Low-scoring: routine greetings, one-off factual questions.
3. Apply threshold (alpha=0.25): mark ~15-20% of exchanges.
4. Mark and compress with: "Extract this user's communication preferences,
   topic interests, and interaction patterns. Under 400 tokens. Prioritize
   marked exchanges containing explicit preferences or corrections."

Output:
"User preferences: Prefers concise, bullet-point responses. Always wants code
in Python 3.10+ with type hints. Dislikes verbose explanations — cut to the
answer first, explain only if asked. Works on ML pipelines (PyTorch, HuggingFace).
Timezone: PST, typically active evenings. Frequently asks about performance
optimization. Corrected assistant twice on: (1) not using f-strings, (2) adding
unnecessary try/except blocks. Prefers 'you' over 'the user' in responses."
```

**Example 3: Compressing Retrieved Documents for RAG**

```
User: My RAG pipeline retrieved 15 documents (6,000 tokens) for the query
"How do I handle database migrations in Django?" but I want to compress
them to 500 tokens to leave room for the system prompt and response.

Approach:
1. Segment each document into paragraphs (logical units).
2. Score paragraphs against the specific query. High-scoring:
   - Paragraphs mentioning Django migrations, makemigrations, migrate commands
   - Paragraphs covering migration conflicts, squashing, data migrations
   Low-scoring: general Django setup, unrelated ORM features, boilerplate.
3. Apply threshold (alpha=0.3): mark the most query-relevant paragraphs.
4. Compress: "Synthesize these documents into a 500-token reference on Django
   database migrations. Prioritize marked paragraphs. Include specific commands,
   common pitfalls, and best practices."

Output:
"Django Migrations Guide: Run `python manage.py makemigrations` to generate
migration files from model changes, then `python manage.py migrate` to apply.
Key commands: `showmigrations` (list status), `sqlmigrate app_name 0001`
(preview SQL), `migrate app_name 0001` (roll back to specific migration).
Common issues: (1) Migration conflicts — run `makemigrations --merge`.
(2) Data migrations — create empty migration with `makemigrations --empty`,
add RunPython operation. (3) Squashing — use `squashmigrations app_name 0001 0010`
to reduce file count. Always commit migration files to version control."
```

## Best Practices

- **Do:** Score sentences against the *specific downstream task*, not general importance. A sentence about a user's job title might be critical for professional email generation but irrelevant for movie recommendations.
- **Do:** Preserve concrete details (numbers, names, dates, specific preferences) from marked sentences during compression. Vague summaries like "user likes movies" lose the signal that "user rated Moonrise Kingdom 5/5" provides.
- **Do:** Keep the original sentence ordering in the marked context. Temporal and logical relationships between sentences carry information that reordering destroys.
- **Do:** Tune the threshold alpha based on the compression ratio needed. For extreme compression (50x), use alpha=0.3-0.4. For moderate compression (5-10x), use alpha=0.15-0.25.
- **Avoid:** Marking everything as important. The "Mark All" strategy (equivalent to generic summarization) consistently underperforms selective marking in the paper's ablations.
- **Avoid:** Using only recency as a proxy for importance. Recent interactions are not always the most informative — a preference stated 3 months ago may be more relevant than yesterday's weather query.

## Error Handling

- **Compressed output exceeds token budget:** Re-run the compression step with an explicit hard limit ("You must use fewer than N tokens. Ruthlessly cut unmarked content first.") or increase the marking threshold to reduce the number of marked sentences.
- **Compression loses critical details:** Check whether the relevant sentences were marked. If not, lower the threshold. If they were marked but still omitted, add explicit instructions: "You must include the following marked details: [list specifics]."
- **Scoring produces flat/uniform scores:** This happens when sentences are too similar or the task query is too vague. Make the task query more specific (not "summarize preferences" but "identify preferences relevant to recommending sci-fi novels") to differentiate scores.
- **Context contains structured data (JSON, tables):** Segment by records/rows rather than sentences. Score each record as a unit. For deeply nested structures, flatten to key-value pairs before scoring.

## Limitations

- **No access to real attention weights in Claude.** The original Attn-GS extracts internal attention scores from model layers. When using Claude, you must approximate this with prompted relevance scoring, which is less precise than true attention extraction. For production systems needing maximum fidelity, consider running the marking stage on an open-weight model (e.g., Llama) where attention is accessible.
- **Compression is lossy by definition.** At extreme ratios (50x), some information will be lost. Tasks requiring exhaustive recall over the full context (e.g., "list every movie the user mentioned") will degrade.
- **Single-task optimization.** Context compressed for one task (movie rating prediction) may not be optimal for another task (generating a bio). If multiple downstream tasks exist, either compress separately for each or use a broader task description during marking.
- **Computational cost of the marking pass.** The two-stage pipeline requires processing the full context at least once (for scoring). This is worthwhile when the compressed context will be reused across many queries, but not for one-shot use where you process the full context only once anyway.

## Reference

[Attn-GS: Attention-Guided Context Compression for Efficient Personalized LLMs](https://arxiv.org/abs/2602.07778v1) — Zeng et al., 2026. Key insight: LLM attention weights at middle layers reliably identify task-relevant sentences; marking those sentences explicitly before compression produces dramatically better compressed contexts than generic summarization (50x compression with <2% quality loss).