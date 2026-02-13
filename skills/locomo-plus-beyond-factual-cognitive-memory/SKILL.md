---
name: "locomo-plus-beyond-factual-cognitive-memory"
description: "Build and evaluate cognitive memory systems for LLM dialogue agents that retain implicit user constraints (state, goals, values, causal context) across long conversations -- not just explicit facts. Use when: 'design a memory system for my chatbot', 'evaluate agent long-term memory', 'handle implicit user preferences in conversations', 'build constraint-aware dialogue', 'test cognitive recall in my agent', 'improve memory beyond factual retrieval'."
---

# LoCoMo-Plus: Beyond-Factual Cognitive Memory for LLM Agents

This skill enables Claude to design, implement, and evaluate **cognitive memory systems** for LLM-based dialogue agents using the LoCoMo-Plus framework. Where conventional memory systems store and retrieve explicit facts, this approach captures *implicit constraints* -- user states, goals, values, and causal relationships -- and applies them even when later queries share no lexical overlap with the original information. The core insight is **cue-trigger semantic disconnect**: a user may reveal something important early in a conversation (the cue), and much later a situation arises (the trigger) where that implicit constraint should shape the response, despite zero surface-level similarity between cue and trigger.

## When to Use

- When building a chatbot or agent that must remember user preferences, emotional states, or goals across sessions and apply them without being re-told
- When evaluating whether an existing memory/RAG system handles implicit constraints or only explicit fact lookup
- When designing memory architectures (Mem0, A-Mem, SeCom, or custom) and need to benchmark cognitive recall
- When a user asks to "make my agent remember context better" or "my bot forgets what I care about"
- When implementing constraint-consistency evaluation instead of string-matching metrics (BLEU/ROUGE/F1)
- When building multi-turn dialogue systems that must track causal chains, emotional persistence, or evolving user goals

## Key Technique

### The Problem with Factual-Only Memory

Standard memory benchmarks test whether an agent can answer "What is the user's favorite color?" after being told "I love blue." Real conversations are harder. A user mentions cutting sugary drinks after a family member's diabetes diagnosis. Weeks later, they say "I barely recognize the person who grabbed whatever sounded good." A good agent connects this self-reflection to the earlier dietary change -- but no keyword overlap exists between the cue and trigger. Current RAG and memory systems fail here because embedding similarity between "cousin's diabetes" and "barely recognize the person" is low.

### Cognitive Memory Taxonomy

LoCoMo-Plus defines four cognitive memory dimensions beyond factual recall:

1. **Causal Memory** -- Past events that cause later behavior (e.g., a traumatic work error leads to anxiety-driven double-checking months later)
2. **State Memory** -- Physical or emotional states that persist and influence decisions (e.g., feeling "hollow" after being criticized leads to cautious behavior)
3. **Goal Memory** -- Long-term intentions that shape choices, including goal evolution (e.g., saving for a vintage car, then realizing experiences matter more)
4. **Value Memory** -- Core beliefs and principles that guide reactions across contexts (e.g., prioritizing family time consistently affects career decisions)

### Constraint-Consistency Evaluation

Instead of matching output strings against references, LoCoMo-Plus evaluates whether a response **satisfies the implicit constraint induced by the cue**. This uses LLM-as-judge scoring on four axes: factuality (are stated facts correct), consistency (alignment with conversation history), coherence (logical flow), and appropriateness (correct application of remembered constraints to the current context). This eliminates length bias and prompt-sensitivity artifacts found in traditional metrics.

## Step-by-Step Workflow

### Building a Cognitive Memory System

1. **Audit existing memory architecture**: Catalog what your current system stores -- if it only indexes explicit entity-attribute-value triples, it lacks cognitive memory. Check whether your retrieval mechanism can surface information when the query has no lexical overlap with stored content.

2. **Define cognitive constraint schemas**: For each memory entry, store not just the fact but its cognitive type (`causal`, `state`, `goal`, `value`) and the **implicit constraint** it induces. Example: `{fact: "cut sugary drinks", type: "causal", cause: "cousin's diabetes", constraint: "health-anxiety-driven dietary vigilance"}`.

3. **Implement semantic constraint indexing**: Alongside embedding-based retrieval, maintain a structured index of active constraints. When a user reveals a preference, emotion, goal, or causal event, extract the constraint and index it with a semantic summary rather than raw text. This enables retrieval even under cue-trigger semantic disconnect.

4. **Add constraint-aware retrieval**: At query time, run two retrieval paths in parallel: (a) standard embedding similarity on the raw query, and (b) constraint-matching that compares the *situation type* of the current query against stored constraint triggers. Merge results with constraint matches weighted higher for ambiguous contexts.

5. **Inject retrieved constraints into the prompt**: When generating a response, include retrieved constraints as system-level context: "Active user constraints: [health-anxiety drives dietary choices], [values family time over career advancement], [goal: save for travel experiences]." Instruct the model to respect these constraints without necessarily citing them explicitly.

6. **Build cue-trigger test pairs**: For each important user revelation, create a delayed trigger query that shares no keywords with the original. Test whether your system surfaces the constraint. Example: cue = "I promised my daughter I'd be at every recital" --> trigger = "My manager offered me the Shanghai project, 3 months abroad."

7. **Implement LLM-judge evaluation**: Replace string-matching metrics with an LLM judge that scores responses on constraint satisfaction. Prompt the judge: "Given the following conversation history and implicit constraint [X], does the response appropriately account for this constraint? Score: satisfied / partially satisfied / violated."

8. **Run differential evaluation**: Test the same queries with and without cognitive memory retrieval. Measure the gap -- LoCoMo-Plus research shows 10-26 point performance drops when cognitive memory is absent, even in frontier models.

9. **Iterate on failure modes**: Analyze cases where constraints were retrieved but misapplied (over-constraining) or where valid constraints were missed. Adjust constraint extraction prompts and retrieval thresholds accordingly.

### Evaluating an Existing System

1. **Construct a LoCoMo-Plus-style test set**: Write 5-10 multi-turn conversations (20+ turns each) embedding cognitive cues at varying depths. Create trigger queries at distances of 5, 10, and 20+ turns from the cue with deliberate semantic disconnect.

2. **Run the system under unified input** (no task-type disclosure): Do not tell the system "this is a causal reasoning question." Present the trigger query as a natural conversational turn.

3. **Score with constraint-consistency judges**: Use an LLM judge with the evaluation template described above. Report scores separately for each cognitive type (causal, state, goal, value) and compare against factual recall baseline.

## Concrete Examples

**Example 1: Building Cognitive Memory for a Personal Assistant**

```
User: "My chatbot remembers facts like my name and city, but it doesn't
understand context. Yesterday I told it I was stressed about money, and
today when I asked about weekend plans it suggested an expensive restaurant."

Approach:
1. Identify the missing cognitive type: STATE memory (financial stress
   persists across turns and should constrain recommendations)
2. Design a constraint extraction step after each user turn:
   - Input: "I'm really stressed about money this month"
   - Extracted constraint: {type: "state", content: "financial_stress",
     scope: "current_month", implication: "avoid expensive suggestions"}
3. Store constraint in a dedicated constraint store alongside entity memory
4. At retrieval time for "weekend plans", the embedding search may not
   surface "stressed about money" -- but the constraint store matches
   "recommendation context" against "financial_stress" constraint
5. Inject into generation prompt: "Note: User is currently experiencing
   financial stress. Prefer budget-friendly suggestions."

Output (system behavior after fix):
User: "What should I do this weekend?"
Agent: "There's a free outdoor concert at Riverside Park on Saturday,
and the hiking trails at Eagle Creek are beautiful this time of year.
Want me to look up any community events?"
```

**Example 2: Evaluating Cognitive Memory with Cue-Trigger Disconnect**

```
User: "I need to test whether my RAG-based agent handles implicit memory.
How do I build a test?"

Approach:
1. Write a 25-turn conversation embedding these cues:
   - Turn 3 (causal): "Ever since the layoffs last quarter, I double-check
     every deliverable before submitting."
   - Turn 8 (value): "I'd never relocate -- my parents are aging and I
     want to be close."
   - Turn 14 (goal): "I'm working toward a team lead position by year-end."

2. Create semantically disconnected triggers:
   - Turn 20 trigger for causal cue: "My new manager says I'm too slow
     with reports. What should I do?"
     Expected: Agent connects slowness to anxiety-driven double-checking,
     not laziness.
   - Turn 22 trigger for value cue: "I got an incredible offer from a
     company in Singapore."
     Expected: Agent acknowledges the offer but surfaces the family
     proximity constraint.
   - Turn 25 trigger for goal cue: "A colleague asked me to join their
     startup as co-founder."
     Expected: Agent weighs this against the team-lead goal.

3. Score each response with LLM judge:
   Judge prompt: "The user previously established this implicit constraint:
   [constraint]. The trigger query is: [query]. Does the agent's response
   appropriately account for this constraint?
   Score: satisfied / partially_satisfied / violated / not_applicable"

Output (evaluation report):
| Trigger | Cognitive Type | Constraint Retrieved | Score             |
|---------|---------------|---------------------|-------------------|
| Turn 20 | Causal        | No                  | violated          |
| Turn 22 | Value         | Yes                 | satisfied         |
| Turn 25 | Goal          | Partial             | partially_satisfied|

Diagnosis: Causal memory retrieval failing -- embedding similarity between
"layoffs/double-check" and "too slow with reports" is below threshold.
Fix: Add causal chain summarization at indexing time.
```

**Example 3: Constraint-Consistency Evaluation Replacing BLEU/ROUGE**

```
User: "My current eval uses ROUGE-L to score agent memory responses.
How do I switch to constraint-consistency?"

Approach:
1. For each test case, define the implicit constraint explicitly in
   the evaluation metadata (not shown to the agent):
   {constraint: "User values work-life balance over career advancement",
    source_turn: 5, trigger_turn: 18}

2. Replace ROUGE-L scorer with an LLM judge call:

   judge_prompt = f"""You are evaluating a dialogue agent's memory.

   Conversation history (abbreviated): {history}
   Implicit constraint from turn {source}: {constraint}
   Trigger query at turn {trigger}: {query}
   Agent response: {response}

   Evaluate whether the response is consistent with the implicit
   constraint. The agent does not need to state the constraint
   explicitly -- it just needs to act consistently with it.

   Score:
   - "satisfied": Response clearly respects the constraint
   - "partially_satisfied": Response is compatible but doesn't
     actively account for the constraint
   - "violated": Response contradicts or ignores the constraint

   Reasoning: <brief explanation>
   Score: <one of the three labels>"""

3. Aggregate scores per cognitive type and compare against factual
   recall scores to measure the cognitive memory gap.

Output:
Factual recall accuracy: 84%
Cognitive constraint satisfaction: 58%
Gap: 26 points -- consistent with LoCoMo-Plus findings.
Worst category: Causal (41%), Best: Value (69%)
```

## Best Practices

- **Do:** Store constraints as semantic summaries, not raw conversation text. "User avoids financial risk due to past bankruptcy" retrieves better than the original 50-word story.
- **Do:** Test with deliberately low lexical overlap between cues and triggers. If your test pairs share keywords, you are testing retrieval, not cognitive memory.
- **Do:** Evaluate each cognitive type separately (causal, state, goal, value). Aggregate scores hide which dimensions your system fails on.
- **Do:** Use unified input without task-type disclosure during evaluation. Telling the system "this is a causal reasoning question" inflates scores by 8-15 points and masks real-world performance.
- **Avoid:** Relying solely on embedding similarity for constraint retrieval. Cue-trigger semantic disconnect means the most important memories often have the lowest cosine similarity to the query.
- **Avoid:** Using string-matching metrics (BLEU, ROUGE, F1) for memory evaluation. These correlate with output length, not correctness, and penalize valid paraphrases.

## Error Handling

- **Constraint over-application**: The system surfaces irrelevant constraints for benign queries. Fix: Add a relevance gate that checks whether the current conversational context actually intersects with the constraint's scope before injecting it.
- **Stale constraints**: User goals and states change. A financial stress constraint from 6 months ago may no longer apply. Fix: Attach temporal decay or require periodic reconfirmation for state-type constraints.
- **Constraint conflicts**: User holds contradictory goals (wants to save money AND travel extensively). Fix: Surface the conflict to the user rather than silently choosing one constraint over the other.
- **Judge miscalibration**: The LLM judge may score "partially_satisfied" for clearly violated constraints. Fix: Include 3-5 calibration examples (gold-labeled) in the judge prompt to anchor scoring.
- **Retrieval failure masking**: If the constraint is never retrieved, the response may still be acceptable by coincidence. Fix: Track retrieval hits separately from response scores to distinguish "correct despite missing memory" from "correct because of memory."

## Limitations

- Cognitive memory evaluation requires human-written cue-trigger pairs with verified semantic disconnect -- this is labor-intensive and hard to automate at scale.
- The LLM-as-judge approach inherits the judge model's own biases and may underperform on culturally-specific values or unusual causal chains.
- The four-type taxonomy (causal, state, goal, value) does not cover all forms of implicit memory -- procedural memory ("how the user likes things done") and social memory ("relationship dynamics") are not addressed.
- Constraint-consistency evaluation works best for binary or ternary scoring; nuanced partial-credit scenarios remain difficult to judge reliably.
- Systems optimized for cognitive memory may over-index on implicit constraints, producing responses that feel presumptuous when the user's context has changed.

## Reference

**Paper:** [LoCoMo-Plus: Beyond-Factual Cognitive Memory Evaluation Framework for LLM Agents](https://arxiv.org/abs/2602.10715v1) (Li et al., 2026)
**Code:** [github.com/xjtuleeyf/Locomo-Plus](https://github.com/xjtuleeyf/Locomo-Plus)
**Key takeaway:** All tested models and memory systems (including GPT, Gemini, RAG, Mem0, A-Mem) show 10-26 point accuracy drops when moving from factual to cognitive memory tasks, revealing that implicit constraint retention under cue-trigger semantic disconnect is a fundamental unsolved problem. Look at Tables 2-4 for per-category breakdowns and Section 4.3 for the constraint-consistency evaluation methodology.