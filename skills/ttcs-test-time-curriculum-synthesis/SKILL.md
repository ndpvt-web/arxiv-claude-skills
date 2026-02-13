---
name: "ttcs-test-time-curriculum-synthesis"
description: "Implement a co-evolving test-time curriculum synthesis framework where a question synthesizer and reasoning solver iteratively improve each other. The synthesizer generates progressively harder problem variants matched to the solver's capability frontier, while the solver trains on both original and synthetic problems using self-consistency rewards. Use when: 'build a self-improving reasoning pipeline', 'create adaptive difficulty curriculum for LLM', 'implement test-time training with curriculum', 'co-evolving synthesizer-solver loop', 'self-consistency reward for reasoning', 'progressive difficulty generation for math problems'."
---

# TTCS: Test-Time Curriculum Synthesis for Self-Evolving Reasoning

This skill enables Claude to build **co-evolving agent frameworks** where two specialized policies — a question synthesizer and a reasoning solver — iteratively bootstrap each other's capabilities at inference time. Derived from the TTCS paper, the core insight is that raw difficult problems produce poor pseudo-labels, so you first synthesize intermediate-difficulty variants calibrated to the solver's current frontier, then use self-consistency across multiple reasoning chains as a supervision signal. This creates a stable, scalable test-time training loop without external labels.

## When to Use

- When the user wants to build a **self-improving reasoning pipeline** that adapts at inference time without ground-truth labels
- When implementing **adaptive curriculum generation** that matches problem difficulty to a model's current capability level
- When designing a system where **two LLM policies co-evolve**: one generating problems, the other solving them
- When the user needs **self-consistency-based reward signals** (majority voting across sampled chains) for reinforcement-style updates
- When building **test-time training loops** for mathematical reasoning, code generation, or logical deduction tasks
- When augmenting a small test set with synthetic variants to **stabilize online model updates**

## Key Technique

TTCS initializes two policies from the same pretrained model checkpoint. The **question synthesizer** (pi_phi) takes an original test question and generates variants at controlled difficulty levels — adding constraints, increasing numerical complexity, or deepening logical dependencies — while preserving the core reasoning structure. The **reasoning solver** (pi_theta) attempts each variant by sampling N independent chain-of-thought reasoning paths and computing a self-consistency score: the fraction of sampled paths that agree on the same final answer. This score serves as a pseudo-reward without any oracle.

The co-evolution loop works as follows: the solver's self-consistency scores on synthesized questions become the reward signal for the synthesizer. Questions where the solver achieves ~50% consistency (the capability frontier) are most valuable — too easy means no learning, too hard means unreliable pseudo-labels. The synthesizer learns to target this frontier. Meanwhile, the solver performs GRPO (Group Relative Policy Optimization) updates on a mixture of original test questions and synthesized variants, using majority-vote pseudo-labels as supervision. The synthetic questions act as a data augmentation buffer that prevents the instability typical of training on tiny test sets.

What makes this different from standard test-time training: instead of naively self-training on questions that may be too hard (yielding garbage pseudo-labels), TTCS constructs a smooth difficulty ramp. The synthesizer generates easier stepping-stone problems first, the solver improves on those, and then the synthesizer ratchets up difficulty. This curriculum effect is what enables stable self-improvement on problems the base model initially cannot solve.

## Step-by-Step Workflow

1. **Initialize both policies from the same base model.** Clone the pretrained checkpoint into two instances: one prompted/fine-tuned as a question synthesizer, the other as a reasoning solver. They share the same architecture and initial weights.

2. **Define the self-consistency reward function.** For a given question q, sample N reasoning chains from the solver (N=8 is typical, temperature=0.7-0.9 for diversity). Compute the consistency score as `R(q) = count(most_common_answer) / N`. This is the pseudo-reward — no ground truth needed.

3. **Seed the synthesizer with the original test questions.** For each test question, prompt the synthesizer to generate K variants at the solver's estimated capability frontier. The prompt should instruct: "Generate a variant of this problem that preserves the reasoning type but adjusts difficulty to be [easier/similar/harder]."

4. **Score synthesized questions with the solver.** Run the solver's self-consistency evaluation on each synthetic variant. Partition results into three buckets: too easy (R > 0.8), frontier (0.3 < R < 0.8), and too hard (R < 0.3). The frontier bucket contains the most useful training signal.

5. **Compute the synthesizer reward.** Reward the synthesizer for producing questions in the frontier zone. Use the question quality reward: `R_synth = -|R(q_synth) - tau|` where tau is the target consistency threshold (typically 0.5). This incentivizes generating questions that are neither trivially solvable nor impossibly hard.

6. **Update the solver via GRPO on the mixed question pool.** Combine original test questions and frontier-zone synthetic variants. For each question, use the majority-vote answer as the pseudo-label. Apply group relative policy optimization: sample a group of responses, score them against the pseudo-label, and update the policy to upweight correct reasoning chains relative to incorrect ones.

7. **Update the synthesizer via GRPO on generation quality.** Using the synthesizer reward from step 5, update pi_phi to generate better-calibrated variants in the next iteration. The synthesizer learns which difficulty modifications land in the solver's frontier zone.

8. **Iterate the co-evolution loop.** Repeat steps 3-7 for T iterations (typically 10-20). Each iteration, the solver gets stronger, so the synthesizer must generate progressively harder variants to stay at the frontier. This creates the curriculum effect automatically.

9. **Extract final answers using the improved solver.** After the loop converges (or hits max iterations), run the final solver on the original test questions with majority voting across N chains. The improved solver should now handle problems it initially could not.

10. **Monitor convergence signals.** Track three metrics across iterations: (a) solver consistency on original test questions (should increase), (b) average difficulty of synthesized questions hitting the frontier (should increase), (c) solver loss stability (should not diverge). If the solver's consistency plateaus, increase N or allow more synthesizer exploration.

## Concrete Examples

**Example 1: Building a self-improving math reasoning pipeline**

User: "I want to build a system where my LLM gets better at solving competition math problems at test time, without any labeled solutions."

Approach:
1. Load the base model (e.g., Qwen2.5-Math-1.5B) and create two instances
2. Configure the synthesizer with a system prompt:
   ```
   You are a math problem designer. Given an original problem, generate a variant
   that tests the same mathematical concept but at a [DIFFICULTY] level.
   Preserve the problem type (algebra, geometry, etc.) but modify:
   - numerical values and constraints
   - number of reasoning steps required
   - level of abstraction
   Output only the new problem statement.
   ```
3. Configure the solver with a chain-of-thought system prompt
4. Run the co-evolution loop:
   ```python
   for iteration in range(T):
       # Synthesizer generates variants for each test question
       variants = []
       for q in test_questions:
           difficulty = estimate_difficulty(q, solver, target_consistency=0.5)
           v = synthesizer.generate(q, difficulty_level=difficulty)
           variants.append(v)

       # Score all questions (original + synthetic) with solver
       all_questions = test_questions + variants
       scores = {}
       for q in all_questions:
           chains = [solver.sample(q, temp=0.8) for _ in range(8)]
           answers = [extract_answer(c) for c in chains]
           majority = Counter(answers).most_common(1)[0]
           scores[q] = {
               "consistency": majority[1] / len(answers),
               "pseudo_label": majority[0],
               "chains": chains
           }

       # Filter frontier questions (0.3 < consistency < 0.8)
       frontier = [q for q, s in scores.items() if 0.3 < s["consistency"] < 0.8]

       # Update solver with GRPO on frontier + original questions
       solver.grpo_update(frontier + test_questions, scores)

       # Update synthesizer reward based on how many variants hit frontier
       synth_rewards = [-abs(scores[v]["consistency"] - 0.5) for v in variants]
       synthesizer.grpo_update(variants, synth_rewards)
   ```
5. Final evaluation: run improved solver on original test set with majority voting

Output: The solver's accuracy on competition math improves by 5-15% over the base model without any ground-truth labels, with gains accumulating across iterations.

**Example 2: Adaptive code debugging curriculum**

User: "I want my coding agent to improve at fixing bugs by generating progressively harder buggy code variants to practice on."

Approach:
1. Treat buggy code snippets as "test questions" — the solver must identify and fix the bug
2. Configure the synthesizer to generate buggy code variants:
   ```
   Given this buggy code, generate a new code snippet with a similar bug type
   but at difficulty level [LEVEL]. Adjust by:
   - Level 1-3: Single obvious bug (typo, off-by-one, wrong operator)
   - Level 4-6: Subtle logic error requiring multi-step reasoning
   - Level 7-9: Interacting bugs across functions, race conditions, edge cases
   Include the correct fix as a hidden comment for self-consistency checking.
   ```
3. Self-consistency check: sample 8 fix attempts from the solver, check if majority agree on the same patch
4. Run the TTCS loop — the synthesizer learns to generate bugs at the solver's frontier difficulty
5. After convergence, the solver handles bug patterns it initially missed

Output: A debugging agent that has adapted to the specific distribution of bugs in the test set, with measurable improvement on fix-rate.

**Example 3: Implementing the co-evolution loop as a reusable framework**

User: "Give me a modular framework for the TTCS co-evolution pattern I can adapt to different tasks."

Approach:
1. Define the abstract interface:
   ```python
   class TTCSFramework:
       def __init__(self, base_model, n_chains=8, target_consistency=0.5,
                    n_iterations=15, frontier_low=0.3, frontier_high=0.8):
           self.synthesizer = init_policy(base_model, role="synthesizer")
           self.solver = init_policy(base_model, role="solver")
           self.n_chains = n_chains
           self.tau = target_consistency
           self.T = n_iterations
           self.frontier = (frontier_low, frontier_high)

       def self_consistency_score(self, question):
           """Sample N chains, return (consistency_ratio, majority_answer, chains)."""
           chains = [self.solver.sample(question) for _ in range(self.n_chains)]
           answers = [self.extract_answer(c) for c in chains]
           majority_ans, majority_count = Counter(answers).most_common(1)[0]
           return majority_count / len(answers), majority_ans, chains

       def synthesize_variants(self, questions, k_per_question=3):
           """Generate k variants per question targeting the solver's frontier."""
           variants = []
           for q in questions:
               consistency, _, _ = self.self_consistency_score(q)
               # If already too easy, request harder; if too hard, request easier
               if consistency > self.frontier[1]:
                   target = "significantly harder"
               elif consistency < self.frontier[0]:
                   target = "easier, stepping-stone"
               else:
                   target = "similar difficulty"
               for _ in range(k_per_question):
                   v = self.synthesizer.generate(q, difficulty=target)
                   variants.append((v, q))  # track parent question
           return variants

       def run(self, test_questions):
           """Execute the full co-evolution loop."""
           for t in range(self.T):
               # Phase 1: Synthesize curriculum
               variants = self.synthesize_variants(test_questions)
               all_qs = test_questions + [v for v, _ in variants]

               # Phase 2: Score everything
               scores = {q: self.self_consistency_score(q) for q in all_qs}

               # Phase 3: Filter frontier questions
               frontier_qs = [q for q in all_qs
                              if self.frontier[0] < scores[q][0] < self.frontier[1]]

               # Phase 4: Update solver (GRPO with pseudo-labels)
               self.solver.update(frontier_qs + test_questions, scores)

               # Phase 5: Update synthesizer (reward = proximity to frontier)
               synth_rewards = {v: -abs(scores[v][0] - self.tau)
                                for v, _ in variants if v in scores}
               self.synthesizer.update(synth_rewards)

           # Final inference
           return {q: self.self_consistency_score(q) for q in test_questions}
   ```
2. Subclass `extract_answer()` for your domain (math answer extraction, code diff, classification label)
3. Subclass the synthesizer prompt template for your problem type

Output: A reusable framework that implements the TTCS co-evolution pattern, adaptable to math, code, logic, or any domain where self-consistency can be measured.

## Best Practices

- **Do:** Set the frontier zone (0.3-0.8 consistency) based on empirical calibration on your specific task. Start with these defaults and narrow the band if the solver improves too quickly.
- **Do:** Use at least N=8 sampled chains for self-consistency. Fewer chains produce noisy consistency estimates that destabilize the loop.
- **Do:** Mix original test questions with synthetic variants in every solver update batch (recommended ratio: 1:2 original-to-synthetic). The originals anchor training to the actual target distribution.
- **Do:** Track the average difficulty of frontier-hitting synthetic questions across iterations — this should monotonically increase if the curriculum is working.
- **Avoid:** Setting the target consistency threshold too high (>0.7). This causes the synthesizer to generate questions that are too easy, preventing meaningful solver improvement.
- **Avoid:** Running too many iterations without checking for solver divergence. Monitor the solver's consistency on a held-out validation set every 3-5 iterations. If it drops, reduce the learning rate or increase the original-question mixing ratio.

## Error Handling

- **Solver produces identical chains (consistency=1.0 on everything):** The sampling temperature is too low. Increase to 0.8-0.9, or add nucleus sampling (top-p=0.95) to ensure reasoning path diversity.
- **Synthesizer generates unsolvable questions:** The difficulty ramp is too aggressive. Tighten the frontier band (e.g., 0.4-0.7) and add a solvability check: if no chain produces a parseable answer, reject the variant.
- **Consistency scores oscillate between iterations:** The GRPO learning rate is too high, or the synthetic question pool is too small. Increase the number of variants per question (k=5+) and reduce the policy update step size.
- **Solver degrades after several iterations (catastrophic forgetting):** The solver is overfitting to synthetic patterns. Increase the ratio of original test questions in the update mix, or add a KL-divergence penalty anchoring the solver to its initial weights.
- **Answer extraction failures:** Self-consistency requires reliable answer extraction. Implement robust parsing with fallback regex patterns (e.g., `\boxed{}` for math, final-line extraction for code).

## Limitations

- **Requires a domain where self-consistency is meaningful.** The technique works best when correct reasoning chains converge on the same answer (math, code, factual QA). It is less effective for open-ended generation tasks (creative writing, summarization) where multiple valid answers exist.
- **Computationally expensive.** Each iteration requires N forward passes per question for self-consistency scoring, multiplied by the number of original + synthetic questions. Budget at least 8x the cost of single-pass inference per iteration.
- **Synthesizer quality bottleneck.** If the base model cannot generate meaningful problem variants in your domain, the curriculum effect breaks down. Pre-filter synthetic questions for structural validity before scoring.
- **No guarantee of monotonic improvement.** The co-evolution loop can enter cycles where the synthesizer and solver chase each other's weaknesses without net progress. Time-box iterations and fall back to the best-performing checkpoint.
- **Cold-start on very hard problems.** If the solver has near-zero consistency on the original test questions, the synthesizer must generate substantially easier variants to bootstrap the loop. This may require explicit "simplify this problem" prompting in early iterations.

## Reference

**Paper:** [TTCS: Test-Time Curriculum Synthesis for Self-Evolving](https://arxiv.org/abs/2601.22628v1) — Yang et al., 2026. Focus on Section 3 (Method) for the co-evolution algorithm, Section 3.2 for the self-consistency reward formulation, and Section 4 for ablations showing why curriculum scheduling outperforms fixed-difficulty augmentation.

**Code:** [github.com/XMUDeepLIT/TTCS](https://github.com/XMUDeepLIT/TTCS) — Reference implementation using GRPO with Qwen2.5-Math backbones.