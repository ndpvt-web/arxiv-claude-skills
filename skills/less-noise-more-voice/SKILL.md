---
name: "less-noise-more-voice"
description: "Identify and remove interference tokens from prompts to improve LLM reasoning accuracy. Based on the LENS framework (Less Noise Sampling). Use when: 'clean up this prompt', 'my prompt isn't working well', 'purify this instruction', 'remove noise from prompt', 'improve prompt for reasoning', 'why is the model failing on this prompt'."
---

# Less Noise, More Voice: Prompt Interference Purification for Better LLM Reasoning

This skill enables Claude to apply the LENS (Less Noise Sampling) framework to identify and surgically remove interference tokens from prompts that degrade LLM reasoning performance. The core insight from the paper is that exploration failures in complex reasoning tasks often arise not from problem difficulty, but from a small number of prompt tokens (~5%) that introduce noise. Removing these tokens can improve rollout success rates by over 20%. Claude uses this technique to diagnose prompt failures, purify instructions for reasoning tasks, and help users build robust prompts that work even in noisy real-world settings.

## When to Use

- When a user reports that an LLM consistently fails on a reasoning task despite the problem being solvable
- When a user asks to debug, clean, or improve a prompt that produces inconsistent or poor results
- When building prompts for mathematical reasoning, code generation, or multi-step logic tasks
- When a user wants to understand why a specific prompt phrasing causes failures
- When optimizing prompts for use with reinforcement learning from verifiable rewards (RLVR) pipelines
- When a user asks to "purify", "denoise", or "strip noise from" a prompt or instruction
- When reducing prompt token count while preserving or improving task accuracy

## Key Technique

**Interference Tokens.** The LENS framework identifies that certain tokens in a prompt cause disproportionate deviation between a model's learned policy and its reference distribution. Formally, an interference score is computed per token: `S_I(s, a) = |log π_θ(a|s) - log π_ref(a|s)|`. Tokens with high scores act as noise — they push the model toward reward over-optimization or misleading reasoning paths. Crucially, only about 5% of tokens are typically problematic, and they can be found by ranking all prompt tokens by this deviation score and selecting the top-k (where `k = ceil(γ * |prompt_length|)` with γ between 1-5%).

**Two-Phase Purification and Transfer.** Phase 1 removes the identified interference tokens to produce a "purified" prompt, then generates rollouts (model completions) from this cleaner version. Phase 2 is the key innovation: rather than simply using the purified prompt going forward, successful rollouts from the purified version are transferred back to supervise policy optimization on the *original noisy prompt*. This teaches the model to ignore interference tokens in real-world conditions rather than depending on sanitized inputs. The transfer activates only when the success rate on the original prompt falls below a threshold τ (default 0.5).

**Why this matters for prompt engineering.** Even without access to model internals, the conceptual framework is directly applicable: prompts fail because of specific token-level noise, not holistic complexity. Systematically identifying and removing low-information or misleading tokens — filler phrases, ambiguous qualifiers, redundant context, contradictory constraints — can dramatically improve reasoning accuracy without changing the core problem.

## Step-by-Step Workflow

1. **Receive the failing prompt and task description.** Collect the full prompt, the expected output type (math answer, code, logical conclusion), and examples of observed failures or inconsistencies.

2. **Tokenize and segment the prompt.** Break the prompt into logical segments: system instructions, context/background, the core problem statement, formatting directives, and any few-shot examples. Identify which segments are essential to the task vs. supplementary.

3. **Score each segment for interference potential.** For each segment, assess: (a) Does it introduce ambiguity or contradiction with other segments? (b) Does it contain filler words, hedging language, or redundant restatements? (c) Does it shift the model's attention away from the core reasoning task? (d) Could it trigger pattern-matching on superficial features rather than genuine reasoning? Assign a qualitative interference score (high/medium/low) to each segment.

4. **Identify candidate interference tokens.** Within high-scoring segments, pinpoint specific tokens or phrases that are most likely causing deviation. Common culprits: unnecessary qualifiers ("try to", "if possible"), contradictory constraints ("be concise but thorough"), irrelevant context that primes wrong associations, ambiguous pronouns, and formatting directives that conflict with reasoning flow. Target roughly 3-7% of total tokens for removal.

5. **Produce the purified prompt.** Remove or replace the identified interference tokens. Preserve all semantically essential information — the core problem, necessary constraints, required output format. Verify the purified version is grammatically coherent and unambiguous.

6. **Test the purified prompt against the original.** If the user can test with the target model, recommend running the purified version on the same set of problems. Compare success rates. If the purified version underperforms, the removed tokens were not interference — restore them and re-analyze.

7. **Generate a transfer version for robustness.** Create a final prompt that retains the original structure but neutralizes interference through rephrasing rather than deletion. This mirrors the paper's Phase 2: the goal is a prompt that works in real-world noisy conditions, not one that only works when perfectly sanitized.

8. **Document the interference analysis.** Provide the user with a clear report: which tokens were identified as interference, why, what was removed or rephrased, and the expected improvement. Include the original, purified, and transfer versions side by side.

9. **Calibrate pruning aggressiveness to model capability.** For weaker or smaller models, recommend a higher pruning ratio (~5% of tokens). For stronger models, recommend conservative pruning (~1-2%). This follows the paper's finding that optimal pruning ratio has an inverse correlation with model capacity.

10. **Iterate if needed.** If the purified prompt still fails, repeat the process focusing on the next tier of interference candidates. Each iteration should target a smaller token set, converging on the minimal effective prompt.

## Concrete Examples

**Example 1: Math reasoning prompt with noisy instructions**

User: "My prompt keeps failing on math word problems. The model gives wrong answers about 40% of the time."

Original prompt:
```
You are a helpful math tutor. Please try your best to solve the following problem
carefully and accurately. Show your work step by step if you can. Remember to
double-check your answer. Note that some problems may have tricky edge cases, so
be careful. Here is the problem:

A train leaves Station A at 9:00 AM traveling at 60 mph. Another train leaves
Station B at 10:00 AM traveling at 80 mph toward Station A. The stations are
280 miles apart. At what time do the trains meet?
```

Interference analysis:
```
HIGH interference (remove):
- "Please try your best to" → hedging, reduces model confidence
- "if you can" → implies the task might be optional or too hard
- "Note that some problems may have tricky edge cases, so be careful"
  → primes the model to second-guess straightforward calculations
- "Remember to double-check your answer" → can cause circular re-evaluation

LOW interference (keep):
- "Show your work step by step" → directly supports chain-of-thought reasoning
- The problem statement itself → essential content
```

Purified prompt:
```
Solve the following problem step by step.

A train leaves Station A at 9:00 AM traveling at 60 mph. Another train leaves
Station B at 10:00 AM traveling at 80 mph toward Station A. The stations are
280 miles apart. At what time do the trains meet?
```

Transfer version (robust to noisy contexts):
```
Solve this step by step, focusing only on the mathematical relationships.

A train leaves Station A at 9:00 AM traveling at 60 mph. Another train leaves
Station B at 10:00 AM traveling at 80 mph toward Station A. The stations are
280 miles apart. At what time do the trains meet?
```

Tokens removed: ~45% of instruction tokens, 0% of problem tokens. Expected improvement: 15-25% accuracy gain on similar problems.

---

**Example 2: Code generation prompt with contradictory constraints**

User: "The model keeps generating overly complex code or ignoring requirements."

Original prompt:
```
Write a Python function that takes a list of integers and returns the second
largest unique value. Make it production-ready with full error handling, type
hints, and docstrings. Keep it simple and concise. Handle edge cases like empty
lists, single-element lists, and lists with all duplicate values. Use only
built-in Python — no imports. Optimize for readability and performance.
```

Interference analysis:
```
HIGH interference (contradictions and noise):
- "production-ready with full error handling, type hints, and docstrings"
  CONTRADICTS "Keep it simple and concise" → model oscillates between verbose
  and minimal styles
- "Optimize for readability and performance" → these often trade off against
  each other, causing the model to hedge
- "Use only built-in Python — no imports" → redundant constraint for this
  problem (no imports needed anyway), adds cognitive load

MEDIUM interference:
- "Handle edge cases like empty lists, single-element lists, and lists with
  all duplicate values" → specific enough to keep, but enumeration style can
  cause the model to over-index on edge cases vs. core logic
```

Purified prompt:
```
Write a Python function that takes a list of integers and returns the second
largest unique value. Include type hints. Raise ValueError for inputs with
fewer than 2 unique values.
```

Transfer version:
```
Write a Python function: given a list of integers, return the second largest
unique value. Add type hints. Raise ValueError when fewer than 2 unique values
exist. Prioritize clarity.
```

---

**Example 3: Diagnosing intermittent failures in a chain-of-thought prompt**

User: "This prompt works ~60% of the time but randomly fails. Why?"

Original prompt:
```
Think step by step. You are an expert logician. Given the following premises,
determine if the conclusion is valid. Be thorough but efficient. Consider all
possible interpretations.

Premises:
1. All managers attend meetings.
2. Some engineers are managers.
3. No interns attend meetings.

Conclusion: Some engineers attend meetings.
```

Interference analysis:
```
HIGH interference:
- "Consider all possible interpretations" → on a formal logic problem, this
  causes the model to explore irrelevant semantic ambiguities instead of
  applying syllogistic rules directly. This is the primary failure cause.
- "Be thorough but efficient" → contradictory directive creating decision
  paralysis about depth of analysis

MEDIUM interference:
- "You are an expert logician" → persona framing can sometimes help, but
  can also cause the model to over-elaborate to "prove" expertise

KEEP:
- "Think step by step" → well-established chain-of-thought trigger
- All premises and conclusion → essential content
```

Purified prompt:
```
Think step by step. Determine whether the conclusion follows logically from
the premises using standard syllogistic reasoning.

Premises:
1. All managers attend meetings.
2. Some engineers are managers.
3. No interns attend meetings.

Conclusion: Some engineers attend meetings.

State whether the conclusion is VALID or INVALID, with your reasoning.
```

Expected improvement: Intermittent failures drop significantly because the model no longer wastes reasoning capacity exploring "all possible interpretations" of unambiguous formal statements.

## Best Practices

**Do:**
- Target instruction/framing tokens first — the problem statement itself is rarely the source of interference
- Keep pruning conservative (3-7% of total tokens) to avoid removing essential context
- Preserve chain-of-thought triggers ("step by step", "let's think about this") which consistently improve reasoning
- Test purified prompts on multiple problem instances, not just the one that failed
- Adjust pruning aggressiveness based on model size: prune more aggressively for smaller models (7B and below), more conservatively for larger models

**Avoid:**
- Removing domain-specific technical terms or constraints that define the problem
- Adding new instructions to "compensate" for removed tokens — this introduces new interference
- Pruning few-shot examples unless they contain contradictory demonstrations
- Assuming longer prompts are always noisier — a well-structured long prompt can outperform a short ambiguous one
- Removing output format specifications (JSON schema, markdown structure) which are usually low-interference and high-value

## Error Handling

- **Purified prompt loses essential meaning:** If removing tokens changes the task semantics, the tokens were not interference. Restore them and look for interference elsewhere in the prompt. Always verify semantic equivalence between original and purified versions.
- **Model performs worse on purified prompt:** The "interference" tokens may have been providing useful signal (e.g., persona framing that activates relevant knowledge). Re-classify them as low-interference and restore.
- **No obvious interference tokens found:** The failure may not be prompt-related. Consider whether the task exceeds the model's capability, the verification criteria are too strict, or the problem requires knowledge the model lacks.
- **Different failures on purified vs. original prompt:** The purification exposed a different weakness. Treat this as a new diagnosis — the original interference was real, but there's an additional issue underneath.

## Limitations

- This technique addresses prompt-level noise, not fundamental capability gaps. If a model cannot solve a problem type regardless of prompt phrasing, purification won't help.
- Without access to model logits and a reference model, interference scoring is qualitative and based on heuristics about what constitutes noise. The paper's formal method (`S_I` scores) requires model internals.
- The approach is most effective for reasoning tasks (math, logic, code) where the relationship between prompt clarity and output quality is strong. Creative or open-ended tasks may not benefit as much.
- Prompt purification is not a substitute for better training data or model fine-tuning — it optimizes the interface, not the model.
- The "transfer" concept (building robustness to noise) is harder to apply in one-shot prompt engineering than in training loops. In practice, rephrasing is the closest available approximation.

## Reference

**Paper:** [Less Noise, More Voice: Reinforcement Learning for Reasoning via Instruction Purification](https://arxiv.org/abs/2601.21244v2) — Guo et al., 2026. Look for: Section 2.1 on interference token identification via deviation scoring, Section 2.2 on the Calibrated Rollout Policy Optimization (CRPO) algorithm, and Appendix E for ablation studies comparing pruning strategies.