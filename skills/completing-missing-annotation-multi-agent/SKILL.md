---
name: "completing-missing-annotation-multi-agent"
description: "Multi-agent debate framework for relevance assessment and annotation completion. Uses opposing-stance LLM agents with iterative critique to label query-document relevance, detect missing annotations, and escalate uncertain cases to humans. Triggers: 'assess relevance of documents to queries', 'find missing annotations in IR benchmark', 'debate-based relevance labeling', 'complete missing labels in dataset', 'multi-agent relevance assessment', 'evaluate document relevance with debate'"
---

# DREAM: Multi-Agent Debate for Relevance Assessment and Annotation Completion

This skill enables Claude to implement the DREAM (Debate-based Relevance Assessment) framework from Ban et al. (2026). DREAM assigns two LLM agents opposing initial stances on whether a document chunk is relevant to a query, then runs iterative debate rounds where each agent critiques the other's reasoning. Agreement after debate yields a high-confidence label; persistent disagreement triggers escalation to human review. This achieves 95.2% balanced accuracy with only 3.5% human involvement, outperforming single-LLM judges and confidence-based escalation methods.

## When to Use

- When the user needs to assess relevance of document passages to search queries at scale
- When completing missing relevance labels in an IR benchmark or evaluation dataset
- When building a quality-assurance layer for automated annotation pipelines
- When the user wants to reduce annotation costs while maintaining human-level accuracy
- When implementing a multi-agent system that must distinguish high-confidence from uncertain judgments
- When the user asks to find unlabeled or mislabeled relevant documents in a retrieval dataset
- When building an annotation pipeline that needs principled human escalation rather than arbitrary confidence thresholds

## Key Technique

**Opposing-stance debate eliminates single-model overconfidence.** Traditional LLM-as-judge approaches assign one model to label relevance, but LLMs are poorly calibrated -- they output high confidence even on ambiguous cases. DREAM avoids confidence scores entirely. Instead, it initializes two agents with forced opposing stances: Agent A starts believing the chunk IS relevant; Agent B starts believing it is NOT. Each agent must defend its position with evidence extracted from the chunk and query.

**Iterative reciprocal critique refines judgments.** Over multiple rounds (typically 2), each agent receives the opponent's reasoning and label from the prior round, then produces an updated label with new reasoning. The key insight is that agents are not merely re-scoring -- they are actively critiquing specific claims made by the opponent, extracting counter-evidence, and revising their stance only when persuaded. This adversarial structure forces both sides of the argument to be explored before convergence.

**Agreement-based routing replaces confidence-based escalation.** After each debate round, the system checks whether both agents now agree on the label. If they do, that label is output with high reliability (95.2% accuracy on agreed cases). If agents still disagree after the maximum rounds, the case is escalated to a human annotator -- critically, the full debate history is provided, which has been shown to improve human inter-annotator agreement from kappa 0.50 to 0.62 and human accuracy from 87.3% to 92.0%. This means the 3.5% of escalated cases are both genuinely hard AND better-contextualized for human review.

## Step-by-Step Workflow

1. **Define the relevance task.** Specify the query, the candidate document chunk, and (if available) a set of known correct answers. The binary label to assign is: 1 if the chunk provides evidential support for answering the query, 0 otherwise.

2. **Initialize two agents with opposing stances.** Create Agent A with the initial stance "I think this chunk IS relevant to the query" and Agent B with "I think this chunk is NOT relevant to the query." Both agents receive the same context: the query text, the chunk text, and any answer set.

3. **Execute Round 1 of debate.** Each agent independently produces: (a) a relevance label (0 or 1), (b) a reasoning string that extracts specific evidence from the chunk supporting its stance, and (c) a critique of the opponent's initial stance. Use temperature 0.0 for deterministic outputs.

4. **Check for agreement.** If Agent A's label equals Agent B's label after Round 1, output that label immediately. The case is resolved with high confidence.

5. **Execute Round 2 (if disagreement persists).** Provide each agent with the opponent's Round 1 reasoning and label. Each agent must: address the opponent's specific arguments, identify any evidence they previously missed, and produce an updated label with revised reasoning.

6. **Check for agreement again.** If labels now match, output the agreed-upon label. If they still disagree after the maximum rounds (default: 2), mark the case for human escalation.

7. **Package escalation context.** For escalated cases, compile the full debate history: both agents' reasoning from each round, the specific points of disagreement, and the extracted evidence. This structured context dramatically improves human annotation quality.

8. **Aggregate results across the dataset.** Track three outcome categories: auto-labeled relevant, auto-labeled irrelevant, and escalated-to-human. Report the escalation ratio (target: under 5%) and the label distribution.

9. **Validate a sample of auto-labeled cases.** Randomly sample agreed-upon labels and verify against human judgment to confirm the expected ~95% accuracy. Adjust maximum debate rounds if accuracy is below threshold.

10. **Integrate human labels for escalated cases.** Merge human judgments on the escalated subset back into the dataset to produce a complete, high-quality annotation set.

## Concrete Examples

**Example 1: Assessing relevance for a QA retrieval benchmark**

User: "I have a dataset of 5,000 query-chunk pairs from a biomedical QA benchmark. About 40% are labeled relevant, but I suspect many relevant chunks are missing labels. Help me find the missing relevant ones."

Approach:
1. Load the dataset; isolate the unlabeled query-chunk pairs
2. For each unlabeled pair, run the DREAM debate pipeline:
   - Agent A argues the chunk IS relevant, extracting biomedical evidence
   - Agent B argues it is NOT relevant, pointing to missing connections
   - After up to 2 rounds, record the agreed label or escalate
3. Flag all pairs where agents agreed on "relevant" as newly discovered annotations
4. Send the ~3-5% disagreement cases to domain experts with debate transcripts

Output:
```
DREAM Assessment Results
========================
Total unlabeled pairs assessed: 3,000
Auto-labeled RELEVANT (agreed):    847  (28.2%)
Auto-labeled IRRELEVANT (agreed): 2,048  (68.3%)
Escalated to human review:          105  ( 3.5%)

Sample agreed-relevant case:
  Query: "What are the side effects of metformin?"
  Chunk: "...patients receiving metformin reported gastrointestinal
          symptoms including nausea (12%), diarrhea (8%), and
          abdominal discomfort (5%) in the Phase III trial..."
  Agent A (Round 1): RELEVANT - Chunk directly lists side effects
                     with prevalence data from clinical trial.
  Agent B (Round 1): RELEVANT - [Revised from NOT RELEVANT]
                     Opponent correctly identifies direct evidential
                     support. Chunk enumerates specific adverse events.
  Result: Agreement at Round 1 -> Label: RELEVANT
```

**Example 2: Building a debate-based annotation pipeline in Python**

User: "Write me a Python implementation of the DREAM debate framework that I can plug into my annotation workflow."

Approach:
1. Define the agent prompt templates with stance injection
2. Implement the multi-round debate loop with agreement checking
3. Add escalation routing and debate history packaging

Output:
```python
from dataclasses import dataclass

@dataclass
class DebateResult:
    query: str
    chunk: str
    label: int | None        # None if escalated
    escalated: bool
    rounds_used: int
    debate_history: list[dict]

def run_dream_assessment(
    query: str,
    chunk: str,
    answers: list[str],
    llm_call: callable,
    max_rounds: int = 2,
) -> DebateResult:
    """Run DREAM debate between two opposing-stance agents."""

    history = []

    # Initialize opposing stances
    stances = [
        "I think this chunk IS relevant to the query.",
        "I think this chunk is NOT relevant to the query.",
    ]

    agent_labels = [None, None]
    agent_reasons = [None, None]

    for round_num in range(1, max_rounds + 1):
        for agent_idx in range(2):
            opponent_idx = 1 - agent_idx
            prompt = build_agent_prompt(
                query=query,
                chunk=chunk,
                answers=answers,
                initial_stance=stances[agent_idx],
                opponent_reasoning=agent_reasons[opponent_idx],
                opponent_label=agent_labels[opponent_idx],
                round_num=round_num,
            )
            response = llm_call(prompt, temperature=0.0)
            agent_labels[agent_idx] = parse_label(response)
            agent_reasons[agent_idx] = parse_reasoning(response)

        history.append({
            "round": round_num,
            "agent_a": {"label": agent_labels[0], "reasoning": agent_reasons[0]},
            "agent_b": {"label": agent_labels[1], "reasoning": agent_reasons[1]},
        })

        # Check agreement
        if agent_labels[0] == agent_labels[1]:
            return DebateResult(
                query=query, chunk=chunk,
                label=agent_labels[0], escalated=False,
                rounds_used=round_num, debate_history=history,
            )

    # Persistent disagreement -> escalate
    return DebateResult(
        query=query, chunk=chunk,
        label=None, escalated=True,
        rounds_used=max_rounds, debate_history=history,
    )


def build_agent_prompt(query, chunk, answers, initial_stance,
                       opponent_reasoning, opponent_label, round_num):
    """Construct the debate prompt for one agent in one round."""
    base = (
        f"Query: {query}\n"
        f"Chunk: {chunk}\n"
        f"Known answers: {answers}\n\n"
        f"Your initial stance: {initial_stance}\n\n"
        "Task: Determine if this chunk provides evidential support "
        "for answering the query. Output a label (1=relevant, "
        "0=irrelevant) and your reasoning with specific evidence.\n"
    )
    if round_num > 1 and opponent_reasoning:
        base += (
            f"\nOpponent's previous label: {opponent_label}\n"
            f"Opponent's reasoning: {opponent_reasoning}\n\n"
            "Critique the opponent's arguments. Address their specific "
            "claims. Update your label if persuaded by their evidence.\n"
        )
    base += "\nFormat: LABEL: [0 or 1]\nREASONING: [your analysis]"
    return base
```

**Example 3: Quality audit of an existing labeled dataset**

User: "I want to audit 500 query-document pairs that were labeled 'irrelevant' by a single annotator. How do I use the debate approach to find potential errors?"

Approach:
1. Treat each "irrelevant"-labeled pair as a candidate for reassessment
2. Run DREAM: Agent A argues for relevance, Agent B argues for irrelevance
3. Cases where both agents agree on RELEVANT after debate are likely annotation errors
4. Cases that escalate deserve careful human re-examination

Output:
```
Audit Results for 500 "Irrelevant"-Labeled Pairs
=================================================
Confirmed irrelevant (both agents agree):     431 (86.2%)
Likely mislabeled as irrelevant (both agree): 38  ( 7.6%)
Uncertain - escalated for re-annotation:      31  ( 6.2%)

Mislabeled example:
  Query: "renewable energy storage methods"
  Chunk: "Lithium-ion batteries dominate grid-scale storage,
          but emerging vanadium redox flow batteries offer
          longer cycle life for renewable integration..."
  Original label: IRRELEVANT
  Agent A (R1): RELEVANT - Directly discusses storage methods
  Agent B (R1): RELEVANT - [Revised] Chunk explicitly covers
                battery technologies for renewable energy storage
  DREAM label: RELEVANT (agreement at Round 1)
  -> Flagged as annotation error
```

## Best Practices

- **Do:** Always use temperature 0.0 (or near-zero) for debate agents to ensure deterministic, reproducible judgments across runs.
- **Do:** Provide the full debate history to human annotators on escalated cases -- this measurably improves human accuracy from 87% to 92%.
- **Do:** Keep maximum debate rounds to 2. The paper found diminishing returns beyond 2 rounds, with most agreements occurring in Round 1.
- **Do:** Track class-wise recall (relevant vs. irrelevant) separately. DREAM achieves 98.4% relevant recall but 91.9% irrelevant recall -- the asymmetry matters for your use case.
- **Avoid:** Using confidence scores or probability outputs to decide escalation. The entire point of DREAM is that inter-agent agreement is a more reliable indicator than self-reported confidence.
- **Avoid:** Using the same initial stance for both agents. The opposing-stance initialization is critical -- without it, both agents converge prematurely and the method degrades to a single-judge baseline.
- **Avoid:** Skipping the critique step. Agents must explicitly address the opponent's reasoning, not just independently re-assess. The adversarial structure is what drives accuracy.

## Error Handling

- **Both agents flip to opponent's stance:** If both agents switch positions in Round 1 (A goes irrelevant, B goes relevant), this counts as disagreement. Continue to Round 2 or escalate. This pattern suggests a genuinely ambiguous case.
- **Parsing failures:** If an agent's response doesn't contain a clear label, re-prompt once with explicit formatting instructions. If it fails again, treat the case as escalated.
- **Skewed escalation rates:** If escalation exceeds 10%, the task definition may be ambiguous. Refine the relevance criteria in the prompt or add more examples to the agent instructions.
- **High disagreement on one class:** If agents systematically disagree on "relevant" cases but agree on "irrelevant," the relevance criteria may be too loose. Tighten the evidential support requirement in the prompt.
- **LLM context limits:** For very long chunks, extract the most query-relevant paragraphs before running debate. The agents need focused context to produce targeted critiques.

## Limitations

- Requires two full LLM calls per round per item (4 calls total for 2 rounds), so cost is ~2-4x a single-judge approach. The tradeoff is justified for high-stakes annotation but may be excessive for casual labeling.
- Binary relevance only. The paper validates on relevant/irrelevant labels. Extending to graded relevance (e.g., 0-3 scale) would require adapting the stance initialization and agreement criteria.
- Validated primarily on English-language QA-style retrieval tasks (BEIR, RobustQA). Performance on other languages, modalities, or non-QA retrieval tasks is unverified.
- The framework assumes an answer set is available to define relevance. For open-ended queries without known answers, the relevance criteria must be specified differently.
- Escalation quality depends on having competent human annotators available. The 3.5% escalation rate is low, but zero-human pipelines cannot use this method as designed.

## Reference

Ban, M., Choi, J., Min, H., Kim, N. H.-Y., & Kim, M. (2026). *Completing Missing Annotation: Multi-Agent Debate for Accurate and Scalable Relevant Assessment for IR Benchmarks.* arXiv:2602.06526v1. [https://arxiv.org/abs/2602.06526v1](https://arxiv.org/abs/2602.06526v1)

Key takeaway: Look at Table 1 for class-wise recall comparison, Figure 2 for accuracy-escalation tradeoff curves, and Appendix A.1 for the exact agent prompt templates used in the DREAM framework.