---
name: "gflowpo-generative-flow-network"
description: "Optimize LLM prompts using GFlowPO's iterative generate-evaluate-refine loop with diversity-preserving exploration and dynamic memory. Use when: 'optimize this prompt', 'find a better prompt for this task', 'prompt engineering with examples', 'auto-tune my system prompt', 'improve prompt accuracy', 'generate prompt variations'."
---

# GFlowPO: Generative Flow Network Prompt Optimization

This skill enables Claude to systematically optimize prompts for language model tasks using the GFlowPO methodology -- a probabilistic framework that treats prompt search as posterior inference over a latent prompt space. Instead of greedy hill-climbing or random mutation, GFlowPO maintains a diverse pool of candidate prompts, scores them against real task examples, and uses a dynamic memory mechanism to progressively focus search on high-reward regions while preserving exploration diversity. Claude applies this as a structured iterative workflow: generate diverse candidates, evaluate on held-out examples, update a meta-prompt with both top-performing and diverse reference prompts, then repeat.

## When to Use

- When the user asks to optimize, improve, or tune a system prompt or task prompt for an LLM
- When a prompt works but the user wants measurably better accuracy on specific examples
- When the user has a set of input/output examples and wants to discover the best instruction prompt
- When the user wants to explore diverse prompt phrasings rather than converging on one style prematurely
- When the user asks to do automated prompt engineering or prompt search
- When the user needs to find an instruction that generalizes across varied inputs (classification, QA, generation)

## Key Technique

**Prompt optimization as posterior inference.** GFlowPO reframes the prompt search problem: instead of maximizing a single reward signal (which leads to mode collapse on a narrow prompt style), it samples prompts proportionally to a reward-weighted posterior. The target distribution is R(z) * p_ref(z|M), where R(z) is the task accuracy reward and p_ref(z|M) is a prior from a reference LM conditioned on a meta-prompt M. This means the optimizer naturally balances exploitation (high accuracy) with staying close to coherent, well-formed prompts.

**Dynamic Memory Update (DMU).** The core practical insight is the DMU mechanism: at each iteration, the meta-prompt is updated by injecting (1) diverse prompts sampled uniformly from a replay buffer of all previously seen candidates, and (2) the top-performing prompts from a priority queue. This is training-free -- no gradient updates, just swapping reference examples in the meta-prompt. The diversity injection prevents the search from collapsing to a single prompt family, while the top-performer injection steers generation toward proven strategies.

**Off-policy replay for sample efficiency.** Unlike on-policy RL prompt optimizers that discard past evaluations, GFlowPO stores all (prompt, score) pairs in a replay buffer and reuses them. The training policy mixes 50% fresh generations with 50% replay samples. This means every expensive LLM evaluation is used multiple times, making the approach 2-4x more sample-efficient than on-policy baselines like StablePrompt.

## Step-by-Step Workflow

1. **Define the task and collect evaluation examples.** Gather 10-50 input/output pairs that represent the target task. Split into a scoring set (used to evaluate candidate prompts) and a held-out test set (used only for final validation). Identify the evaluation metric (exact match, F1, contains-answer, etc.).

2. **Construct the initial meta-prompt.** Build a template with three sections: (a) a task-agnostic instruction like "Generate a clear, concise instruction for the following task", (b) 3-5 randomly sampled input/output demonstration pairs from the scoring set, and (c) 2-3 initial reference prompts (the user's current best prompt plus 1-2 simple variations).

3. **Generate a diverse batch of candidate prompts.** Using the meta-prompt, generate 8-16 candidate prompts. Apply temperature 0.7-1.0 to encourage diversity. Each candidate should be a complete instruction that could replace the original prompt. Vary phrasing, structure, level of detail, and tone.

4. **Score each candidate on the evaluation set.** For every candidate prompt, run it against all examples in the scoring set. Compute accuracy as R(z) = epsilon + sum of correct predictions, where epsilon is a small constant (0.01) to avoid zero rewards. Record (prompt, score) pairs.

5. **Update the replay buffer and priority queue.** Add all (prompt, score) pairs to the replay buffer. Update the priority queue to hold the top-5 highest-scoring prompts seen across all iterations.

6. **Perform Dynamic Memory Update on the meta-prompt.** Sample 2 prompts uniformly at random from the replay buffer (diversity injection) and 1 prompt from the top of the priority queue (exploitation injection). Replace the reference prompts in the meta-prompt with these 3 prompts and their scores.

7. **Generate the next batch of candidates using the updated meta-prompt.** Now the meta-prompt contains both high-performing examples and diverse alternatives, steering generation toward promising yet varied regions of prompt space.

8. **Repeat steps 4-7 for 3-6 iterations.** Each iteration refines the meta-prompt's reference prompts, progressively improving candidate quality. Monitor whether the top score plateaus -- if the best score hasn't improved in 2 consecutive iterations, stop.

9. **Select the final prompt from the priority queue.** Take the top-3 prompts from the priority queue and evaluate each on the held-out test set. Report the winner with its test accuracy and the runner-up alternatives.

10. **Document the optimization trace.** Output a summary showing: initial prompt score, final prompt score, number of iterations, total candidates evaluated, and the top-3 prompts with their scores for reproducibility.

## Concrete Examples

**Example 1: Optimizing a sentiment classification prompt**

User: "I have this prompt for sentiment analysis: 'Classify the following review as positive or negative.' It gets 72% accuracy on my test set. Can you optimize it?"

Approach:
1. Take the user's evaluation examples (assume 30 labeled reviews provided)
2. Split: 20 for scoring, 10 held out for final test
3. Build meta-prompt with task instruction + 4 example reviews + the original prompt as initial reference
4. Generate 12 diverse candidates:
   - "Read the customer review below and determine its sentiment. Reply with exactly 'positive' or 'negative'."
   - "You are a sentiment analysis expert. Analyze the emotional tone of this review. Output: positive or negative."
   - "Based on the overall opinion expressed, classify this review. Answer only 'positive' or 'negative', nothing else."
   - ... (9 more variations)
5. Score all 12 on the 20-example scoring set
6. Top scorer: "Read the customer review carefully. Based on the overall sentiment expressed by the reviewer, classify it as exactly 'positive' or 'negative'. Output only the label." (Score: 85%)
7. DMU: inject 2 random buffer prompts + this top scorer into meta-prompt
8. Generate 12 more candidates informed by the updated meta-prompt
9. After 4 iterations, best prompt scores 90% on scoring set, 88% on held-out test

Output:
```
Optimization complete (4 iterations, 48 candidates evaluated)

Best prompt (88% test accuracy):
"Read the customer review carefully. Based on the overall sentiment
expressed by the reviewer, classify it as exactly 'positive' or
'negative'. Output only the classification label, no explanation."

Runner-up (86% test accuracy):
"Determine whether the following product review is positive or negative.
Consider the reviewer's overall satisfaction. Respond with one word:
positive or negative."

Improvement: 72% -> 88% (+16 percentage points)
```

**Example 2: Finding an instruction for a code explanation task**

User: "I need a prompt that makes GPT-4 explain code snippets to beginners. I have 15 example pairs of (code, good_explanation). Help me find the best instruction."

Approach:
1. Use 10 examples for scoring (ROUGE-L against reference explanations), 5 held out
2. Initial meta-prompt with 3 code/explanation demos + starter prompt: "Explain this code to a beginner."
3. Iteration 1: Generate 10 candidates, score via ROUGE-L
4. Iteration 2: DMU injects top performer + 2 diverse samples, generate 10 more
5. Iteration 3: Repeat, observe convergence

Output:
```
Optimization complete (3 iterations, 30 candidates evaluated)

Best prompt (ROUGE-L: 0.71):
"You are a patient programming tutor. Explain the following code snippet
line by line in plain English. Assume the reader has no programming
experience. Use analogies where helpful. End with a one-sentence summary
of what the code does overall."

Runner-up (ROUGE-L: 0.68):
"Break down this code for someone who has never programmed before.
Explain what each line does using everyday language, then summarize
the overall purpose."

Original prompt ROUGE-L: 0.54 -> Best: 0.71 (+31% relative improvement)
```

**Example 3: Optimizing a multi-step reasoning prompt**

User: "My chain-of-thought prompt for math word problems only gets 60% on GSM8K-style questions. I have 25 example problems with solutions. Optimize the prompt."

Approach:
1. Split 25 examples: 18 scoring, 7 held out
2. Initial reference: user's CoT prompt + "Let's think step by step."
3. Generate candidates that vary: step-by-step framing, verification instructions, output format constraints, reasoning scaffolds
4. Key insight from GFlowPO: diversity injection surfaces non-obvious prompt structures (e.g., "First estimate the answer, then solve precisely" or "Identify what quantities are given and what is asked before computing")
5. Run 5 iterations of 8 candidates each

Output:
```
Optimization complete (5 iterations, 40 candidates evaluated)

Best prompt (77% test accuracy):
"Solve this math problem step by step. First, identify all given
quantities and the question being asked. Then set up the equations
needed. Solve each step showing your arithmetic. Finally, verify
your answer by checking it satisfies the original conditions.
Write your final numerical answer after 'Answer:'."

Improvement: 60% -> 77% (+17 percentage points)
Top-3 prompts preserved for A/B testing.
```

## Best Practices

- **Do:** Always include both diverse AND top-performing prompts in the meta-prompt update. Using only top performers causes mode collapse; using only diverse samples wastes signal from good results.
- **Do:** Use exact-match or structured scoring metrics whenever possible. Fuzzy metrics like "sounds good" make the reward signal noisy and slow convergence.
- **Do:** Keep the replay buffer unlimited -- every evaluated prompt is useful. The cost is negligible (just storing strings and scores) and replay diversity improves with buffer size.
- **Do:** Fix the evaluation examples across iterations so scores are comparable. Changing the scoring set between iterations invalidates the priority queue rankings.
- **Avoid:** Generating fewer than 8 candidates per iteration. Below this, you lack the diversity needed for the DMU mechanism to work effectively.
- **Avoid:** Running more than 6 iterations without improvement. If the top score plateaus for 2+ rounds, the search has likely converged and further iterations waste evaluations.
- **Avoid:** Optimizing prompts on fewer than 10 evaluation examples. Small scoring sets lead to overfitting -- a prompt that aces 5 examples may fail on the 6th.

## Error Handling

- **All candidates score near zero:** The meta-prompt is likely misspecified. Check that the demonstration examples in the meta-prompt actually represent the task correctly. Reset with simpler, more explicit task descriptions.
- **Scores oscillate without improving:** The scoring metric may be too noisy. Increase the scoring set size, or switch to a more deterministic metric (exact match instead of semantic similarity).
- **Candidates are too similar to each other:** Increase generation temperature to 1.0, or explicitly add diversity instructions to the meta-prompt like "Generate a prompt that takes a completely different approach from the reference prompts."
- **Priority queue dominated by one prompt style:** Force the DMU to sample 3 diverse prompts instead of 2 from the replay buffer, temporarily reducing exploitation pressure.
- **User has no evaluation examples:** Help the user create 10-20 synthetic examples first. GFlowPO fundamentally requires a scoring function -- without examples, fall back to manual prompt iteration.

## Limitations

- Requires evaluation examples with ground-truth labels or reference outputs. Cannot optimize prompts for purely subjective or creative tasks without a measurable scoring function.
- Each iteration requires running the target LLM on all scoring examples for every candidate, so cost scales as (candidates_per_iteration * scoring_set_size * iterations). For expensive APIs, keep the scoring set to 15-25 examples.
- Optimizes the instruction/system prompt only. Does not optimize few-shot example selection, output format constraints, or tool-use prompts (though these could be incorporated as part of the prompt string).
- The approach discovers better phrasings within the space reachable from the meta-prompt. It cannot invent fundamentally new prompting paradigms (like discovering chain-of-thought from scratch).
- Works best for classification, QA, and structured-output tasks where accuracy is clearly measurable. Less effective for open-ended generation tasks.

## Reference

**Paper:** [GFlowPO: Generative Flow Network as a Language Model Prompt Optimizer](https://arxiv.org/abs/2602.03358v1) (Cho et al., 2026)
**Key insight:** Casting prompt search as posterior inference with R(z) * p_ref(z|M) as the target distribution, combined with a training-free Dynamic Memory Update that injects both diverse and top-performing prompts into the meta-prompt, achieves state-of-the-art prompt optimization with 2-4x better sample efficiency than on-policy RL baselines.