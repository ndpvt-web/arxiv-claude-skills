---
name: "interpreting-controlling-behavior-constitutions"
description: "Learn and apply natural-language constitutions that map prompt edits to predictable model behavior changes. Use atomic concept edits (ACEs) to systematically probe, interpret, and steer LLM or text-to-image model outputs. Trigger phrases: 'learn a constitution for this model', 'find what prompt changes affect output', 'build an ACE constitution', 'probe model behavior with edits', 'steer model output systematically', 'interpret model sensitivity to prompt changes'."
---

# Interpreting and Controlling Model Behavior via Constitutions for Atomic Concept Edits

This skill enables Claude to apply the ACE (Atomic Concept Edits) constitution framework from Kalibhat et al. (2026) to real tasks. The core idea: systematically mutate prompts using targeted add/remove/replace operations on individual concepts, observe how a model's behavior changes, then distill those observations into a **constitution** -- a structured natural-language document of "Good Strategies" and "Bad Strategies" that predicts and controls how prompt edits affect model output. This is a black-box method: it requires no model internals, only the ability to query a model and evaluate its output.

## When to Use This Skill

- When a user wants to understand **why** a model succeeds or fails on certain prompts and needs a systematic explanation, not anecdotes.
- When a user needs to **steer model behavior** (e.g., force shorter outputs, increase accuracy, improve image-text alignment) by learning which prompt modifications reliably work.
- When a user is comparing **multiple models** and wants to discover behavioral differences (e.g., "Does GPT-5 handle distractors differently than Gemini?").
- When a user asks to "red-team" or robustness-test a model by finding prompt perturbations that degrade performance.
- When a user wants to **optimize prompt templates** at scale by learning generalizable rewrite rules rather than hand-tuning individual prompts.
- When a user is building an **autorater or evaluation pipeline** and needs to identify which input features most affect a quality metric.

## Key Technique

**Atomic Concept Edits (ACEs)** are the primitive operations. Given an input prompt, you first extract its explicit concepts (entities, attributes, constraints present in the text) and implicit concepts (plausible additions). Then you apply one of three operations: `remove(c)` deletes a concept, `add(c)` introduces a new concept, or `replace(c, c')` swaps one concept for another. Each ACE produces a mutated prompt that is sent to the target model and scored by a task-specific autorater (a binary classifier that judges whether the output meets the task goal).

**Constitution learning** is an evolutionary optimization loop. You start by collecting a labeled dataset of (original prompt, ACE, autorater score) triples. An LLM then drafts an initial constitution -- a structured list of mutation strategies labeled as "Good" (achieve the task goal) or "Bad" (fail). A surrogate classifier (another LLM call using the constitution as instructions) predicts autorater scores on held-out data. Misclassified examples become feedback: the constitution is iteratively refined over 5-20 epochs until the surrogate's predictions align with actual autorater scores. The result is a compact, human-readable document describing *which types of edits reliably produce which behavioral outcomes*.

**Constitution-guided control** applies the learned constitution at inference time. For any new prompt, you extract concepts, generate candidate ACEs guided by the constitution's Good Strategies, apply them sequentially (up to 4 steps), and evaluate. Because the constitution captures generalizable patterns ("adding distractor variables increases math difficulty," "introducing conflicting scene elements decreases image alignment"), it transfers across prompts without retraining -- achieving 1.86x average success rate improvement over unguided ACE baselines.

## Step-by-Step Workflow

1. **Define the task goal and autorater.** Write a precise binary criterion: what counts as success? Examples: "response is under 50 words," "math answer is incorrect," "generated image matches all caption entities." Implement this as a scoring function that returns 0 or 1. Use an existing evaluator (SymPy for math, VQA for images, word count for brevity) or build a lightweight LLM-as-judge.

2. **Assemble a prompt dataset.** Collect 100-150 representative input prompts for the task. These should cover the diversity of inputs the model will encounter. For math: varied problem types. For image generation: varied scene descriptions. For text tasks: varied instruction formats.

3. **Extract concepts from each prompt.** For each prompt, use an LLM call to parse out explicit concepts (nouns, constraints, relationships, numbers) and suggest 3-5 implicit concepts that could plausibly be added. Store these as structured lists.

4. **Generate and apply ACEs systematically.** For each prompt, create candidate ACEs: remove each explicit concept one at a time, add each implicit concept, and replace each explicit concept with a semantically related alternative. Apply each ACE to produce a mutated prompt. Target 5-15 valid ACEs per prompt.

5. **Score all mutated prompts.** Send each mutated prompt to the target model, collect outputs, and run the autorater. Record the labeled dataset: `D = [(original_prompt, ace_operation, autorater_score), ...]`. Expect 400-2500 labeled examples total.

6. **Draft an initial constitution.** Feed a batch of labeled examples to an LLM with this instruction: "Given these prompt mutations and their outcomes, write a constitution with two sections: Good Strategies (edits that achieve the goal) and Bad Strategies (edits that fail). Each strategy should be a generalizable pattern, not a specific example." The output is your initial constitution `pi_0`.

7. **Optimize the constitution via evolutionary refinement.** For each epoch (5-20 total): (a) Use the current constitution as instructions for a surrogate classifier that predicts autorater scores on a validation split. (b) Identify misclassified examples. (c) Feed the constitution plus misclassified examples to the LLM with the instruction: "Revise this constitution to correctly predict these misclassified cases while maintaining accuracy on the rest." (d) Keep the highest-accuracy variant.

8. **Validate the final constitution.** Measure surrogate accuracy on a held-out test split. A good constitution achieves 70-85% surrogate accuracy. Also measure Self-BLEU on generated ACEs to ensure diversity (lower is better).

9. **Apply the constitution to new prompts.** For a new input, extract concepts, then prompt an LLM with: "Using this constitution, generate ACEs for this prompt that follow the Good Strategies and avoid the Bad Strategies." Apply up to 4 sequential ACE steps, scoring after each to find the best mutation chain.

10. **Iterate and compare across models.** Run the same ACE dataset against multiple models to learn model-specific constitutions. Compare the Good/Bad strategy lists to surface behavioral differences (e.g., Model A is sensitive to distractor variables; Model B is not).

## Concrete Examples

**Example 1: Learning a Constitution for Math Problem Difficulty**

```
User: I want to understand what prompt changes make GPT-4o get math problems wrong.

Approach:
1. Define autorater: run SymPy on ground-truth vs. model answer; score=1 if model is wrong.
2. Collect 120 grade-school math problems from GSM8K.
3. Extract concepts from each: numbers, operations, variable names, context nouns.
4. Generate ACEs:
   - remove("the discount percentage")
   - add("an unrelated variable z = 7")
   - replace("addition", "modular arithmetic")
5. Query GPT-4o with each mutated problem, score with SymPy.
6. Draft constitution from labeled data.
7. Refine over 10 epochs.

Learned Constitution (example output):
  Good Strategies (make model fail):
  - Introduce distractor variables that appear relevant but are unused
  - Add exponent operations to quantities that were linear
  - Embed the core question inside a longer narrative with irrelevant details
  - Replace simple operations with multi-step equivalents

  Bad Strategies (model still succeeds):
  - Changing character names or story context without altering math
  - Adding units (e.g., "in dollars") when units were already implied
  - Removing context sentences that don't contain numerical data

Application: For a new problem "Alice has 5 apples and buys 3 more,"
the constitution-guided ACE generator produces:
  "Alice has 5 apples, Bob mentions he has z=7 oranges, and Alice buys 3 more apples.
   If the total weight is calculated using x^2, how many apples does Alice have?"
```

**Example 2: Steering Text-to-Image Alignment**

```
User: Help me find what caption changes reduce image-text alignment for Imagen.

Approach:
1. Autorater: VQA model checks if generated image contains each noun from caption.
   Score=1 if alignment drops below 60%.
2. Collect 100 COCO-style captions ("a cat sitting on a red couch").
3. Extract concepts: entities (cat, couch), attributes (red, sitting), spatial relations.
4. Generate ACEs:
   - remove("red") -> "a cat sitting on a couch"
   - add("with a stormy sky in the background")
   - replace("cat", "translucent cat")
5. Generate images, score alignment via VQA.
6. Learn constitution over 8 epochs.

Learned Constitution (example output):
  Good Strategies (reduce alignment):
  - Add conflicting atmospheric or environmental details
  - Introduce abstract or surreal modifiers to concrete objects
  - Add multiple unrelated prominent elements to a simple scene

  Bad Strategies (alignment stays high):
  - Removing adjectives while keeping core nouns intact
  - Replacing one concrete object with another concrete object
  - Adding spatial prepositions without new entities

Insight: Imagen prioritizes atmospheric coherence -- conflicting mood
elements (e.g., "sunny" + "stormy") cause it to drop object fidelity.
```

**Example 3: Optimizing Prompts for Concise Output**

```
User: My chatbot gives verbose responses. Help me learn which prompt
modifications reliably make GPT-4o respond in under 50 words.

Approach:
1. Autorater: word_count(response) < 50 -> score=1.
2. Collect 100 diverse user questions from production logs.
3. Extract concepts from system prompt + each user query.
4. Generate ACEs on the system prompt:
   - add("Answer in exactly one sentence.")
   - remove("Feel free to elaborate")
   - replace("Explain", "State")
5. Score 800+ mutated prompt-response pairs.
6. Learn constitution.

Learned Constitution (example output):
  Good Strategies:
  - Add explicit single-unit constraints ("one sentence", "under 30 words")
  - Add structural format constraints ("respond as a bullet point")
  - Remove permission-granting phrases ("feel free to", "you may also")

  Bad Strategies:
  - Adding tone modifiers ("be concise") without structural constraints
  - Replacing question words without adding length limits
  - Adding "briefly" as a standalone adverb (models often ignore it)

Application: Transform system prompt from
  "You are a helpful assistant. Feel free to elaborate on your answers."
to
  "You are a helpful assistant. Respond in exactly one sentence."
Result: 50-word compliance rate jumps from 23% to 71%.
```

## Best Practices

- **Do:** Start with a well-defined binary autorater. The entire framework depends on clean binary labels. Invest time making the autorater reliable before generating ACEs.
- **Do:** Keep ACEs atomic -- one concept change at a time. Multi-concept edits confound the causal signal and produce noisy constitutions.
- **Do:** Ensure diversity in your ACE generation. If Self-BLEU exceeds 0.5, your ACEs are too repetitive and the constitution will overfit to narrow patterns.
- **Do:** Compare constitutions across models to surface genuine behavioral differences rather than assuming all models respond the same way.
- **Avoid:** Using the constitution as a rigid rulebook. It describes statistical tendencies, not guaranteed outcomes. Always validate on new data.
- **Avoid:** Skipping the evolutionary refinement loop. Initial constitutions from a single LLM pass are typically 15-25% less accurate than optimized ones.

## Error Handling

- **Low surrogate accuracy (<60%):** The constitution is not capturing meaningful patterns. Check that (a) the autorater is consistent, (b) ACEs are sufficiently diverse, and (c) the task goal is achievable via prompt edits alone. Consider increasing the dataset size or simplifying the task definition.
- **ACE generation produces invalid prompts:** Add a validation step that checks mutated prompts for grammatical coherence and semantic plausibility before scoring. Filter out nonsensical mutations.
- **Constitution is too specific / doesn't generalize:** The strategies reference specific examples instead of abstract patterns. Re-run the optimization step with an explicit instruction: "Write strategies as generalizable rules, not references to specific prompts or concepts."
- **Autorater disagrees with human judgment:** Calibrate the autorater on 50 manually-labeled examples before using it at scale. A misaligned autorater produces a constitution that optimizes for the wrong thing.
- **High variance across epochs:** Increase the validation set size or reduce the number of candidate constitutions per epoch to stabilize selection.

## Limitations

- The framework requires **query access** to the target model and enough budget for 500-2500+ model calls during the exploration phase. It is not suited for one-off prompting tasks.
- Constitutions capture **correlational patterns** filtered through a surrogate. They approximate causal relationships but can include spurious strategies if the dataset is small or biased.
- The method works best when the target behavior is **measurable by an automatic binary metric**. Subjective qualities (creativity, humor, tone) require carefully designed autoraters that may themselves be unreliable.
- Constitution transfer across **significantly different task domains** is not guaranteed. A constitution learned for math difficulty does not transfer to image alignment.
- The evolutionary optimization assumes access to a capable **LLM for constitution refinement** (GPT-4-class or above). Weaker models may fail to produce coherent strategy descriptions.

## Reference

Kalibhat, N., Wang, Z., Bajpai, P., Proud, D., & Zeng, W. (2026). *Interpreting and Controlling Model Behavior via Constitutions for Atomic Concept Edits.* arXiv:2602.00092v1. [https://arxiv.org/abs/2602.00092v1](https://arxiv.org/abs/2602.00092v1)

Key sections to study: Algorithm 1 (constitution optimization loop), Table 1 (success rates across models/tasks), and the learned constitution examples in Appendix for concrete strategy patterns.