---
name: "llm-prompt-evaluation-educational"
description: "Systematically design, evaluate, and rank LLM prompts for educational applications using tournament-style Glicko-2 comparison and pedagogical alignment criteria. Use when the user says 'evaluate my prompts', 'compare prompt templates', 'which prompt is better for teaching', 'optimize educational prompts', 'run a prompt tournament', or 'test prompt variations for learning'."
---

# LLM Prompt Evaluation for Educational Applications

This skill enables Claude to apply a rigorous, evidence-based methodology for designing and evaluating LLM prompts intended for educational use. Rather than ad-hoc prompt tweaking, the approach uses a tournament-style pairwise comparison framework scored with the Glicko-2 rating system across three pedagogical dimensions: format quality, dialogue support, and learner appropriateness. The method comes from Holmes et al. (2026), who demonstrated that a prompt combining the **persona pattern** and **context manager pattern** — designed around metacognitive learning strategies — outperformed five other templates with 81–100% win probability in pairwise matchups.

## When to Use

- When a user asks to compare multiple prompt templates for a tutoring system, quiz generator, or educational chatbot
- When building a system that generates follow-up questions, explanations, or formative assessments for learners
- When the user wants to move beyond gut-feel prompt selection toward measurable evaluation of educational prompt quality
- When designing prompts that must align with specific pedagogical strategies (Socratic questioning, metacognition, scaffolding, self-directed learning)
- When the user has 3+ prompt variants and needs to determine which performs best across real student interaction data
- When building an automated or semi-automated prompt evaluation pipeline for an edtech product

## Key Technique

**Tournament-style pairwise evaluation with Glicko-2 ratings.** Instead of scoring prompts in isolation (which introduces scale bias), this method generates outputs from each prompt template against the same input contexts, then presents output pairs to judges who select a winner on each of three dimensions. The Glicko-2 system — originally designed for chess ratings — tracks each prompt's rating, rating deviation (uncertainty), and volatility (consistency). After sufficient rounds, prompts converge to stable ratings that reflect true relative quality. This is superior to Likert-scale rubrics because judges only need to make relative comparisons ("Which is better?"), which humans do more reliably than absolute scoring.

**Three evaluation dimensions from educational research.** Each pair is judged on: (1) **Format** — Is the output well-structured, appropriately scoped, and clear? (2) **Dialogue support** — Does the output sustain productive educational dialogue, inviting elaboration and deeper thinking? (3) **Appropriateness** — Is the output suitable for the target learner's level, avoiding jargon overload or condescension? These dimensions can be adapted for different educational contexts but should always cover structural quality, conversational effectiveness, and learner fit.

**Prompt design using established patterns.** The winning prompt in the study combined two patterns from the prompt engineering pattern catalog (White et al., 2023): the **persona pattern** (instructing the LLM to adopt a specific expert role, e.g., "You are a reading comprehension tutor who specializes in metacognitive strategies") and the **context manager pattern** (explicitly defining what context the LLM should attend to and what to ignore). The winning prompt specifically targeted strategic reading and self-directed learning, suggesting that prompts anchored in well-defined pedagogical theories outperform generic instructional prompts.

## Step-by-Step Workflow

1. **Define the educational task precisely.** Specify what the LLM must produce (follow-up questions, hints, explanations, feedback) and for what learner population (grade level, domain, prior knowledge assumptions). Write this as a one-paragraph task specification.

2. **Design 4–6 prompt template variants.** Each template should combine at least one prompt engineering pattern (persona, context manager, template, flipped interaction, chain-of-thought) with a distinct pedagogical strategy (Socratic questioning, scaffolded hints, metacognitive reflection, direct instruction, elaborative interrogation, self-explanation). Document which pattern and strategy each template uses.

3. **Collect or curate 20+ authentic input contexts.** These are real student interactions, questions, or learning scenarios that each prompt will be tested against. Source from actual deployment logs, classroom transcripts, or synthetic scenarios modeled on real student behavior. Aim for diversity across difficulty levels and topic areas.

4. **Generate outputs from every prompt against every input.** Run each prompt template against each input context, producing a matrix of (prompts × inputs) outputs. Store these with metadata linking each output to its prompt template and input context.

5. **Construct pairwise matchups.** For each input context, create all unique prompt pairs (for 6 prompts, that is 15 pairs per input). Randomize the presentation order (left/right) to eliminate position bias. Each matchup presents two outputs side by side, stripped of prompt identity.

6. **Recruit 3–8 qualified judges and define the rubric.** Judges should have domain expertise (educators, instructional designers, or subject-matter experts). Provide them with the three evaluation dimensions — format, dialogue support, and appropriateness — each defined in 2–3 sentences with examples of what constitutes a "win" on that dimension. Allow ties only when outputs are genuinely indistinguishable.

7. **Run the tournament and compute Glicko-2 ratings.** Initialize each prompt with rating=1500, RD=350, volatility=0.06 (standard Glicko-2 defaults). After each judged pair, update ratings for both prompts. Process all judgments in chronological order. After all rounds, each prompt has a final rating, RD (lower = more certain), and volatility.

8. **Analyze pairwise win probabilities.** From the final Glicko-2 ratings, compute the expected win probability for every prompt pair using the formula: `E(A beats B) = 1 / (1 + 10^((RB - RA) / 400))`. A prompt with ≥75% win probability against all others is a clear winner. If no prompt dominates, examine dimension-specific results to understand tradeoffs.

9. **Validate the winning prompt.** Deploy the top-rated prompt on a fresh set of 20+ input contexts not used in the tournament. Have 2+ judges confirm it maintains quality. Check for failure modes on edge cases (very short inputs, off-topic inputs, advanced learners).

10. **Document and iterate.** Record the winning prompt's pattern combination, pedagogical strategy, and Glicko-2 trajectory. When educational requirements change or new prompt ideas emerge, run a new tournament with the incumbent champion included as a baseline.

## Concrete Examples

**Example 1: Evaluating follow-up question prompts for a reading comprehension tutor**

```
User: I have 4 different prompts for generating follow-up questions after
a student reads a passage. How do I figure out which one actually helps
students learn the most?

Approach:
1. Identify the 4 prompt variants and label them (P1–P4). For each,
   note the pedagogical strategy:
   - P1: Direct comprehension check ("Ask a factual question about...")
   - P2: Socratic probing ("Ask a question that challenges the student
     to identify assumptions in...")
   - P3: Metacognitive reflection with persona pattern ("You are a
     reading coach. Ask the student what strategies they used to
     understand...")
   - P4: Elaborative interrogation ("Ask the student to explain WHY
     a key claim in the passage is true or false...")

2. Collect 25 real reading passages with student summaries from your
   deployment logs.

3. Generate follow-up questions from all 4 prompts × 25 passages
   = 100 outputs.

4. Create 6 pairwise matchups per passage × 25 passages = 150 total
   comparisons. Randomize left/right order.

5. Have 4 judges (2 teachers, 2 instructional designers) evaluate each
   pair on:
   - Format: Is the question clear, well-phrased, single-focus?
   - Dialogue support: Does it invite a substantive student response?
   - Appropriateness: Is it at the right level for the target students?

6. Compute Glicko-2 ratings after all 150 × 4 judge evaluations.

Output:
┌────────┬────────┬───────┬────────────┬─────────────────────────┐
│ Prompt │ Rating │ RD    │ Volatility │ Win% vs others          │
├────────┼────────┼───────┼────────────┼─────────────────────────┤
│ P3     │ 1738   │ 62    │ 0.052      │ P1:94% P2:81% P4:87%   │
│ P4     │ 1612   │ 68    │ 0.058      │ P1:79% P2:63%          │
│ P2     │ 1551   │ 71    │ 0.061      │ P1:68%                 │
│ P1     │ 1399   │ 65    │ 0.055      │ —                      │
└────────┴────────┴───────┴────────────┴─────────────────────────┘

Conclusion: P3 (metacognitive + persona pattern) dominates. Its RD of
62 indicates high confidence. Deploy P3 as the production prompt.
```

**Example 2: Building an automated evaluation pipeline in Python**

```
User: I want to automate prompt evaluation for my educational chatbot.
Can you set up the tournament framework?

Approach:
1. Install the glicko2 library: pip install glicko2
2. Structure the evaluation pipeline as three modules:
   - generator.py: Runs each prompt template against input contexts
   - comparator.py: Presents pairs to judges (human UI or LLM-as-judge)
   - ranker.py: Maintains Glicko-2 ratings and computes win probabilities

Output (ranker.py core logic):

import glicko2

def initialize_prompts(prompt_ids):
    """Each prompt starts at rating 1500, RD 350, vol 0.06."""
    return {
        pid: glicko2.Player(rating=1500, rd=350, vol=0.06)
        for pid in prompt_ids
    }

def record_match(players, winner_id, loser_id):
    """Update ratings after a pairwise judgment."""
    winner = players[winner_id]
    loser = players[loser_id]
    winner.update_player([loser.rating], [loser.rd], [1])  # win=1
    loser.update_player([winner.rating], [winner.rd], [0])  # loss=0

def win_probability(player_a, player_b):
    """Expected probability that A beats B."""
    return 1.0 / (1.0 + 10 ** ((player_b.rating - player_a.rating) / 400))

def run_tournament(judgments, prompt_ids):
    """judgments: list of (winner_id, loser_id) tuples."""
    players = initialize_prompts(prompt_ids)
    for winner_id, loser_id in judgments:
        record_match(players, winner_id, loser_id)
    return {
        pid: {"rating": p.rating, "rd": p.rd, "vol": p.vol}
        for pid, p in players.items()
    }
```

**Example 3: Designing prompt variants with pedagogical pattern combinations**

```
User: Help me create 6 prompt templates for generating math hints that
I can then evaluate with a tournament.

Output:

Template 1 — Direct Instruction (Template Pattern)
"Given the problem: {problem}. The student's incorrect answer is:
{answer}. Generate a hint that directly addresses their misconception."

Template 2 — Socratic Questioning (Flipped Interaction Pattern)
"Given the problem: {problem}. Instead of telling the student the
answer, ask them a question that guides them to discover their error
in: {answer}."

Template 3 — Metacognitive Coaching (Persona + Context Manager)
"You are a math tutor who specializes in helping students develop
problem-solving awareness. Given {problem} and the student's work
{answer}, ask the student to reflect on which step they feel least
confident about and why. Focus only on the student's reasoning
process, not the correct answer."

Template 4 — Scaffolded Decomposition (Chain-of-Thought Pattern)
"Break the problem {problem} into smaller sub-problems. Identify
which sub-problem the student's answer {answer} fails on. Generate
a hint for ONLY that sub-problem."

Template 5 — Analogical Reasoning (Context Manager Pattern)
"Consider only problems structurally similar to {problem}. Generate
a simpler analogous problem that, if the student can solve it, will
reveal the principle needed to correct {answer}."

Template 6 — Self-Explanation Prompt (Elaborative Interrogation)
"The student answered {answer} to {problem}. Ask the student to
explain their reasoning step by step, then ask them to verify each
step against the problem constraints."

Each template combines a prompt engineering pattern with a distinct
pedagogical strategy, ensuring the tournament tests meaningfully
different approaches rather than surface-level wording variations.
```

## Best Practices

- **Do** use authentic student interaction data as input contexts — synthetic data biases results toward well-formed inputs that don't reflect real learner messiness.
- **Do** blind the judges to which prompt template produced each output. Label outputs as "Response A" and "Response B" with randomized assignment.
- **Do** include at least one baseline "naive" prompt (simple instruction without patterns) to anchor the ratings and measure how much patterns actually help.
- **Do** keep the three evaluation dimensions separate in judging — a prompt can win on format but lose on dialogue support, and collapsing dimensions hides this.
- **Avoid** evaluating prompts on fewer than 15 input contexts per prompt pair. Glicko-2 needs sufficient matches for rating deviation to decrease below ~80 (high confidence).
- **Avoid** using a single LLM-as-judge without human validation. LLM judges have systematic biases (preferring longer outputs, favoring their own generation style). Use LLM judges only as a pre-filter, with humans making final calls.
- **Avoid** changing prompt templates mid-tournament. If you discover a promising variant, add it as a new entrant rather than modifying an existing one, so all ratings remain comparable.

## Error Handling

| Problem | Cause | Solution |
|---|---|---|
| All prompts have similar ratings (within 50 points) | Templates are too similar or sample size too small | Ensure templates use genuinely different pedagogical strategies; increase to 30+ input contexts |
| Rating deviation stays above 100 after full tournament | Too few judgments per prompt | Each prompt needs ~30+ total matches (wins + losses) for RD to converge |
| Judges disagree sharply on the same pairs | Rubric dimensions are ambiguous | Refine dimension definitions, run a calibration round where judges discuss disagreements |
| Winning prompt fails on edge cases in production | Tournament inputs didn't cover the full distribution | Add adversarial inputs (empty responses, off-topic text, very advanced questions) to the test set |
| Glicko-2 volatility spikes for one prompt | Prompt performs inconsistently across input types | Segment inputs by category and run sub-tournaments to find where the prompt breaks down |

## Limitations

- **Pairwise judging is labor-intensive.** For 6 prompts and 25 inputs, you generate 375 pairwise comparisons. With 4 judges, that is 1,500 individual judgments. Consider using stratified sampling to reduce volume while maintaining statistical power.
- **Glicko-2 assumes a single latent skill.** A prompt that excels at dialogue support but fails at format will get a blended rating that masks this. Always examine dimension-specific results alongside the overall rating.
- **The method evaluates output quality, not learning outcomes.** A prompt that judges rate highly may not actually improve student learning. This framework is a necessary but not sufficient step — follow up with A/B testing on actual learning gains when possible.
- **Context-dependent results.** The winning prompt for reading comprehension follow-up questions may not win for math tutoring or essay feedback. Re-run the tournament for each substantially different educational task.
- **Judge expertise matters.** Ratings are only as meaningful as the judges' ability to assess pedagogical quality. Domain experts consistently outperform general raters on the "appropriateness" dimension.

## Reference

Holmes, L., Coscia, A., Crossley, S., Choi, J. S., & Morris, W. (2026). *LLM Prompt Evaluation for Educational Applications.* arXiv:2601.16134. [https://arxiv.org/abs/2601.16134](https://arxiv.org/abs/2601.16134)

Key takeaway: The paper's core contribution is the tournament-style evaluation framework using Glicko-2 ratings across three pedagogical dimensions, demonstrating that a persona + context manager prompt targeting metacognitive strategies dominated five alternatives. Read Sections 3–4 for the full tournament methodology and Section 5 for the statistical analysis of win probabilities.