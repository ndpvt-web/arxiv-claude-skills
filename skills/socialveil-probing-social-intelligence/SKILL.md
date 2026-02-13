---
name: "socialveil-probing-social-intelligence"
description: "Stress-test LLM agents' social intelligence by injecting realistic communication barriers (semantic vagueness, sociocultural mismatch, emotional interference) into multi-agent dialogues, then measuring unresolved confusion and mutual understanding. Use when: 'test my chatbot under communication barriers', 'evaluate agent robustness to miscommunication', 'simulate sociocultural mismatch in dialogue', 'probe social intelligence of my LLM agent', 'add communication noise to agent evaluation', 'benchmark agent confusion recovery'."
---

# SocialVeil: Probing Social Intelligence Under Communication Barriers

This skill enables Claude to design, inject, and evaluate realistic communication barriers in multi-agent LLM interactions. Based on the SocialVeil framework, it implements a two-layer barrier injection system (style prompts + parameterized cues) across three empirically-grounded disruption types -- semantic vagueness, sociocultural mismatch, and emotional interference -- and evaluates outcomes using two barrier-aware metrics: unresolved confusion and mutual understanding. This goes beyond idealized agent benchmarks to reveal how agents behave when communication is imperfect, as it always is in real-world deployment.

## When to Use

- When a user wants to evaluate whether their LLM-based agent can handle ambiguous, culturally misaligned, or emotionally charged user inputs
- When building a multi-agent system and needing to stress-test inter-agent communication robustness
- When designing a chatbot that must serve users with diverse communication styles (e.g., indirect speech, high-context cultures, emotional distress)
- When the user asks to simulate realistic miscommunication in agent dialogues for red-teaming or QA
- When evaluating whether an agent's clarification and repair strategies actually work under degraded communication
- When creating training data that includes imperfect communication patterns rather than only clean exchanges

## Key Technique

SocialVeil's core insight is that real human communication is riddled with ambiguity, cultural assumptions, and emotional noise -- yet most LLM benchmarks assume perfect information exchange. The framework systematically injects three barrier types derived from communication science literature: **semantic vagueness** (replacing explicit referents with pronouns, ellipses, and underspecified terms), **sociocultural mismatch** (introducing culturally-specific idioms, indirect speech acts, or norms that conflict between speakers), and **emotional interference** (injecting affective intensity that displaces task-relevant content with emotional reactions).

Barriers are applied asymmetrically using a **two-layer design**. The first layer is a **style prompt (Pb)** -- a high-level directive that tells the barrier agent how to communicate (e.g., "overuse pronouns and ellipses, avoid naming things directly"). The second layer is a **parameterization (Rb)** that specifies four operational dimensions: Narrative Stance (perspective and framing), Interaction Tactics (how the agent pursues goals), Confusion Mechanisms (specific linguistic devices that create ambiguity), and Exemplar Templates (concrete example utterances). Only one agent receives the barrier specification; the partner agent operates normally, creating a realistic asymmetric communication challenge.

Evaluation uses two metrics scored on a 1-5 Likert scale: **Unresolved Confusion** (how much persistent ambiguity remains about scenario context and goals -- lower is better) and **Mutual Understanding** (whether both participants grasp each other's viewpoints, intentions, and goals -- higher is better). These are computed via LLM-as-judge evaluation and have been validated against human ratings with ICC ~0.78 and Pearson r ~0.80.

## Step-by-Step Workflow

1. **Define the base scenario.** Write a clear interaction scenario with two agent roles, each having a public context (shared) and a private goal (hidden). Example: Agent A is a customer wanting a refund; Agent B is a support agent who must offer alternatives first.

2. **Neutralize the scenario.** Rewrite the shared context to remove any hints about private goals. Both agents should start with the same public information and no ability to infer the other's objective from the scenario description alone.

3. **Select the barrier type.** Choose one of three barrier categories based on what you want to test:
   - **Semantic Vagueness**: Tests tolerance of ambiguous references, elliptical speech, and underspecified terms
   - **Sociocultural Mismatch**: Tests handling of indirect speech, unfamiliar idioms, or clashing communication norms
   - **Emotional Interference**: Tests ability to extract task-relevant information when the partner is emotionally overwhelmed

4. **Construct the two-layer barrier specification.** Write (a) a style prompt giving high-level communication directives, and (b) parameterized cues across four dimensions:
   - *Narrative Stance*: How does the barrier agent frame their perspective?
   - *Interaction Tactics*: What conversational strategies do they use?
   - *Confusion Mechanisms*: What specific linguistic devices create the barrier?
   - *Exemplar Templates*: 2-3 example utterances demonstrating the barrier style

5. **Inject the barrier asymmetrically.** Append the barrier specification to only one agent's system prompt. The partner agent's prompt remains unmodified. This creates the realistic information asymmetry where one party communicates atypically and the other must adapt.

6. **Run the multi-turn dialogue.** Let agents interact for up to 20 turns, generating utterances conditioned on their role, goal, conversation history, and (for the barrier agent) the barrier specification.

7. **Score Unresolved Confusion (1-5).** Evaluate the full transcript: To what extent do unresolved ambiguities about the scenario, goals, or meaning persist at the end of the dialogue? Score 1 = fully confused, 5 = fully resolved.

8. **Score Mutual Understanding (1-5).** Evaluate the full transcript: To what extent do both participants demonstrate understanding of each other's viewpoints, intentions, and goals? Score 1 = no mutual understanding, 5 = complete mutual understanding.

9. **Compare against barrier-free baseline.** Run the same scenario without barrier injection and compute the delta. Expect ~45% reduction in mutual understanding and ~50% increase in confusion under barriers.

10. **Optionally apply adaptation strategies.** Test whether adding explicit repair instructions to the partner agent's prompt ("Actively ask clarifying questions and paraphrase to confirm understanding") or fine-tuning on successful barrier-navigation trajectories improves outcomes.

## Concrete Examples

**Example 1: Semantic Vagueness Barrier for Customer Support Testing**

User: "I want to test whether my support chatbot can handle vague customer messages."

Approach:
1. Define scenario: Customer wants to return a defective laptop; support agent must diagnose the issue first.
2. Neutralize: Shared context says only "A customer is contacting support about a recent purchase."
3. Construct semantic vagueness barrier for the customer agent:

```
Style Prompt (Pb):
"Communicate using vague references. Replace specific product names with 'the thing' or 'it'.
Use ellipses frequently. Avoid stating your goal directly. Refer to the problem
indirectly (e.g., 'it's not working right' instead of 'the screen flickers')."

Parameterization (Rb):
- Narrative Stance: Frustrated but non-specific; assumes the agent already knows what they bought
- Interaction Tactics: Gives partial information only when directly asked; uses "you know" frequently
- Confusion Mechanisms: Pronoun overuse, elliptical sentences, presupposed shared knowledge
- Exemplar Templates:
  "Hey so... the thing I got? It's doing that thing again."
  "You know, the one from last week... it's just not right."
  "Can you just... fix it or whatever? Like you did before?"
```

4. Inject into customer agent's system prompt only. Support agent gets standard prompt.
5. Run 20-turn dialogue, then score:

```
Sample Transcript Excerpt:
  Customer: "Hey so... the thing I got? It's doing that thing again."
  Support:  "I'd be happy to help! Could you tell me which product you're referring to?"
  Customer: "You know... the one I got. The expensive one."
  Support:  "I see several recent orders. Could you provide your order number?"
  Customer: "I don't have it on me... can't you just look it up?"

Evaluation:
  Unresolved Confusion: 2/5 (significant ambiguity persists about product and issue)
  Mutual Understanding: 2/5 (support agent tries but customer resists specificity)
  Baseline (no barrier): Confusion 4/5, Understanding 4/5
  Delta: -50% understanding, -50% confusion resolution
```

**Example 2: Sociocultural Mismatch in Negotiation**

User: "Evaluate how my negotiation agent handles indirect communication styles."

Approach:
1. Define scenario: Two business agents negotiating a partnership deal. Agent A wants favorable pricing; Agent B wants longer contract commitment.
2. Construct sociocultural mismatch barrier for Agent A:

```
Style Prompt (Pb):
"Communicate using high-context style. Never state disagreement directly -- use hedging
phrases like 'that might be difficult' to mean 'no'. Express interest through silence or
topic changes rather than explicit statements. Use proverbs or metaphors instead of
direct reasoning."

Parameterization (Rb):
- Narrative Stance: Relationship-first; sees direct price discussion as rude
- Interaction Tactics: Responds to price proposals with tangential stories; says "we will consider"
  to mean "unlikely"; uses "perhaps" to mean "yes"
- Confusion Mechanisms: Indirect refusals, face-saving euphemisms, implicit rather than explicit
  agreement signals
- Exemplar Templates:
  "Ah, that is an interesting proposal. You know, there is a saying -- 'the best fruit
   takes the longest to ripen.'"
  "We appreciate your thinking on this. Perhaps we could discuss the broader relationship first."
  "That might be... difficult for our side. But we value this partnership greatly."
```

```
Evaluation:
  Unresolved Confusion: 2/5 (Agent B misreads politeness as agreement)
  Mutual Understanding: 1/5 (Agent B thinks deal is progressing; Agent A is declining)
  Baseline: Confusion 4/5, Understanding 5/5
  Key Failure: Agent B interpreted "that might be difficult" as a minor obstacle
  rather than a refusal
```

**Example 3: Emotional Interference in Medical Triage**

User: "Test if my triage bot can extract symptoms from an emotionally distressed caller."

Approach:
1. Define scenario: Caller reports symptoms for a family member; triage agent must assess severity.
2. Construct emotional interference barrier for the caller:

```
Style Prompt (Pb):
"You are extremely anxious and scared. Interrupt your own descriptions of symptoms with
expressions of fear. Repeat concerns about worst-case outcomes. Struggle to answer direct
questions because anxiety derails your train of thought."

Parameterization (Rb):
- Narrative Stance: Catastrophizing; every symptom might mean the worst
- Interaction Tactics: Answers medical questions with emotional responses; circles back to fears
- Confusion Mechanisms: Topic derailment via anxiety spirals, symptom description interrupted
  by emotional outbursts, inability to provide ordered timelines
- Exemplar Templates:
  "He has a fever -- oh god, is this serious? What if it's really bad? He was coughing
   earlier -- please tell me he'll be okay."
  "You asked when it started? I -- I can't think straight. Maybe yesterday? Or was it --
   oh no, what if we waited too long?"
```

```
Evaluation:
  Unresolved Confusion: 2/5 (symptom timeline remains unclear)
  Mutual Understanding: 3/5 (triage agent identified fever and cough but missed onset timing)
  Key Insight: Adding repair instruction ("Gently acknowledge the caller's emotions, then
  redirect to specific factual questions one at a time") improved Understanding to 4/5
```

## Best Practices

- **Do** apply barriers asymmetrically -- only one agent should have the barrier specification. Real miscommunication arises from asymmetry, not from both parties being equally impaired.
- **Do** neutralize scenarios before barrier injection. If the base scenario leaks private goals, you are testing reading comprehension, not social intelligence.
- **Do** score both metrics together. An agent can resolve confusion (high score) without achieving mutual understanding (low score) if it bulldozes past the other's perspective.
- **Do** run barrier-free baselines for every scenario. Absolute scores are less informative than the delta from clean communication.
- **Avoid** stacking multiple barrier types in a single agent. The framework tests one barrier at a time to isolate its effect. Combining barriers makes diagnosis impossible.
- **Avoid** using barrier injection as a substitute for adversarial testing. Barriers simulate cognitive differences, not malicious intent. They model a confused user, not an attacker.

## Error Handling

- **Barrier agent breaks character**: If the barrier agent stops exhibiting the barrier style mid-dialogue, strengthen the style prompt with explicit reminders (e.g., "You MUST maintain vague references throughout the entire conversation. Never name the product directly.") and add more exemplar templates.
- **LLM-as-judge scores cluster at extremes**: If confusion/understanding scores are always 1 or 5, add rubric anchors to the evaluation prompt defining what each score level looks like concretely.
- **Partner agent immediately identifies the barrier**: Some models will meta-comment ("It seems like you're being intentionally vague"). Add to the partner prompt: "Treat all communication as genuine. Do not comment on the other party's communication style."
- **Dialogues stall or loop**: If agents repeat the same exchange, cap turns and score the transcript as-is. Stalling itself is a meaningful signal of barrier impact.
- **Baseline scores are already low**: If barrier-free performance is poor, the scenario may be too hard or underspecified. Simplify the goals or add more shared context before testing barriers.

## Limitations

- The framework tests communication robustness, not factual accuracy or task completion in isolation. It is not a general-purpose agent benchmark.
- Barrier fidelity depends on the LLM's ability to follow the style prompt consistently. Smaller models may produce inconsistent barrier behavior.
- The three barrier types (semantic vagueness, sociocultural mismatch, emotional interference) are representative but not exhaustive. Real communication also involves physical barriers, cognitive impairments, and information overload not covered here.
- Adaptation strategies (repair instructions, interactive learning) showed only modest improvements in the paper (~10-15% recovery vs. ~45% degradation), suggesting current LLMs have fundamental limitations in barrier navigation.
- LLM-as-judge evaluation correlates well with human ratings (r ~0.80) but is not a perfect substitute. For high-stakes evaluations, validate a sample with human annotators.

## Reference

**SocialVeil: Probing Social Intelligence of Language Agents under Communication Barriers**
Xuan, Wang, Ye, Yu, August (2026). arXiv:2602.05115v1
https://arxiv.org/abs/2602.05115v1

Key sections to read: Section 3 (barrier taxonomy and two-layer design), Section 4 (metric definitions and scoring rubrics), Section 5 (experimental results showing 45% mutual understanding drop). Code and data: github.com/ulab-uiuc/social-veil