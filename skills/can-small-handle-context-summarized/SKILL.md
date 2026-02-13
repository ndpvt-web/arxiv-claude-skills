---
name: "can-small-handle-context-summarized"
description: "Build context-summarized multi-turn QA systems that let small language models (SLMs) handle customer-service dialogues with near-LLM quality. Implements conversation history summarization, stage-based evaluation, and prompt engineering for resource-constrained deployments. Use when: 'build a customer service chatbot with a small model', 'summarize conversation history for context window', 'evaluate SLM vs LLM on multi-turn QA', 'deploy a multi-turn QA system on limited hardware', 'optimize dialogue context for small language models', 'stage-based analysis of chatbot performance'."
---

# Context-Summarized Multi-Turn QA for Small Language Models

This skill enables Claude to design, implement, and evaluate multi-turn customer-service QA systems where small language models (3B-8B parameters) achieve near-LLM performance by using conversation history summarization. The core technique compresses prior dialogue turns into structured summaries that preserve essential conversational state -- issue status, verification steps, exact identifiers, commitments, and current phase -- allowing SLMs to maintain dialogue continuity within tight token budgets. Based on Cooray et al. (2026), this approach demonstrates that models like LLaMA-3.2-3B and Qwen-3-4B can match GPT-4.1-level quality on customer-service QA when fed properly summarized context.

## When to Use

- When building a customer-service chatbot that must run on limited GPU/CPU resources (edge, on-prem, or cost-constrained cloud)
- When designing a multi-turn dialogue system and the raw conversation history exceeds the model's effective context window
- When selecting which small language model to deploy for customer-service QA and needing a structured evaluation framework
- When implementing a history summarization pipeline to feed compressed context into any downstream LM
- When evaluating chatbot quality across different conversation stages (early issue identification, mid problem-solving, late resolution)
- When generating synthetic multi-turn customer-service training data from single-turn conversation logs

## Key Technique

**History Summarization as Context Engineering.** Instead of feeding raw multi-turn conversation history into an SLM (which wastes tokens on filler, greetings, and redundant information), this approach uses a summarization step that distills prior turns into a concise representation (capped at ~250 tokens). The summary is structured to preserve six categories of information: (1) the client's primary issue and its current status, (2) client and agent names, (3) completed and pending verification steps, (4) exact identifiers, dates, and monetary amounts, (5) commitments and deadlines made, and (6) the current conversation state. This compression lets SLMs operate on the *semantic essence* of a 20+ turn conversation rather than the raw transcript.

**Stage-Based Quality Analysis.** Customer-service conversations follow a predictable arc: Early (first ~20% of turns) for issue identification, Mid (~70%) for core information exchange and problem-solving, and Late (~10%) for resolution confirmation and closure. The paper shows that SLM performance varies significantly across these stages -- models tend to be most competitive with LLMs in late-stage resolution responses and weakest in early-stage context-setting. This insight is critical for deployment: you can allocate more powerful models to early-stage turns or add explicit stage-detection logic to route appropriately.

**Model Selection Findings.** Among 3-4B parameter models, LLaMA-3.2-3B-Instruct and Qwen-3-4B-Instruct consistently outperform alternatives, achieving human evaluation scores of 4.15/5.0 and 4.07/5.0 respectively (GPT-4.1 scored 4.15/5.0). In the 8B tier, LLaMA-3.1-8B-Instruct leads with 3.79/5.0 on LLM-as-judge. Critically, Qwen-3-8B achieved a 55.8% win rate against Gemini-2.5-Flash in pairwise comparison. Not all SLMs benefit equally -- SmolLM3-3B and Gemma-3-4B underperformed despite similar parameter counts, highlighting that architecture and instruction-tuning quality matter more than size alone.

## Step-by-Step Workflow

1. **Assess the conversation data.** Examine existing customer-service logs. Filter to retain conversations with 5-100 turns -- shorter ones lack multi-turn complexity, longer ones often contain noise or agent handoffs that break coherence.

2. **Build the history summarization prompt.** Construct a summarization prompt that instructs a capable model (GPT-4o-mini or equivalent) to produce a factual, concise summary of all prior turns. Enforce a strict token cap (~250 tokens) and temperature 0.3 for consistency. The prompt must explicitly list what to preserve:
   ```
   Produce a clear, concise, and factual summary of the conversation so far.
   Include only information explicitly stated:
   - Client's primary issue and current resolution status
   - Client and agent names
   - Completed and pending verification steps
   - Exact identifiers, dates, and monetary amounts mentioned
   - Commitments or deadlines made by either party
   - Current state of the conversation
   Maximum 250 tokens. Do not infer or add information not present.
   ```

3. **Structure the SLM input as a four-part prompt.** Assemble each inference call with:
   - `[CONTEXT]`: The summarized conversation history from step 2
   - `[QUESTION]`: The client's current message/question
   - `[INSTRUCTION]`: Task-specific directive (e.g., "Respond as a professional customer-service agent. Address the client's concern directly.")
   - `[RESPONSE]`: Leave blank for the SLM to generate

4. **Select and configure the SLM.** Choose an instruction-tuned model from the top performers: LLaMA-3.2-3B-Instruct (best quality-to-size ratio), Qwen-3-4B-Instruct, or LLaMA-3.1-8B-Instruct for higher ceiling. Use 4-bit quantization (QLoRA) for deployment efficiency. Set max sequence length to 512 tokens.

5. **Fine-tune on domain-specific data (if needed).** Use Unsloth + HuggingFace with QLoRA (LoRA rank 16, 3 epochs). Training data should follow the four-part prompt structure from step 3. A dataset of ~130k context-summarized examples is sufficient for strong performance.

6. **Implement conversation stage detection.** Classify each turn into Early/Mid/Late based on its position in the conversation (first 20%, middle 70%, final 10%). Use this classification to:
   - Apply stricter context summarization in early turns (less history to summarize, more emphasis on issue identification)
   - Monitor quality degradation patterns by stage
   - Optionally route early-stage turns to a larger model if quality is critical

7. **Add response refinement post-processing.** Pass SLM outputs through a lightweight refinement step that ensures responses sound natural and spoken-style. The refinement prompt: "Rewrite so it sounds like a real phone response. Aim for 2-3 short spoken-style sentences. Keep all factual content. Remove robotic phrasing."

8. **Evaluate with multi-dimensional scoring.** Assess outputs on four dimensions (1-5 scale each): human-likeness, continuity/context understanding, tone/clarity, and task appropriateness. Use both LLM-as-judge (sample 6000+ responses) and human evaluation (sample 500+ per model) for production readiness.

9. **Deploy with a rolling summarization pipeline.** In production, after each turn pair (client message + agent response), update the running summary by re-summarizing the previous summary + latest turn pair. This keeps the context window bounded regardless of conversation length.

10. **Monitor stage-specific performance.** Track evaluation scores broken down by conversation stage. If late-stage (resolution) quality is high but early-stage quality dips, consider hybrid routing where the first 2-3 turns use a larger model and subsequent turns use the SLM.

## Concrete Examples

**Example 1: Building a Context-Summarized QA Pipeline**

User: "I need to build a customer service chatbot using Phi-4-Mini that can handle 30+ turn banking conversations. How do I keep it from losing context?"

Approach:
1. Implement a rolling summarization service that runs after each turn pair
2. Use GPT-4o-mini (or a local summarizer) to compress history into structured summaries
3. Feed the SLM a four-part prompt with the compressed context

Output architecture:
```python
# summarization_service.py
import openai

SUMMARIZE_PROMPT = """Produce a clear, concise, and factual summary of the conversation so far.
Include only information explicitly stated:
- Client's primary issue and current resolution status
- Client and agent names
- Completed and pending verification steps
- Exact identifiers, dates, and monetary amounts mentioned
- Commitments or deadlines made by either party
- Current state of the conversation
Maximum 250 tokens. Do not infer or add information not present."""

def summarize_history(conversation_turns: list[dict]) -> str:
    """Compress multi-turn history into a structured summary."""
    formatted_turns = "\n".join(
        f"{turn['role']}: {turn['content']}" for turn in conversation_turns
    )
    response = openai.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": SUMMARIZE_PROMPT},
            {"role": "user", "content": formatted_turns}
        ],
        max_tokens=250,
        temperature=0.3
    )
    return response.choices[0].message.content

def build_slm_prompt(summary: str, current_question: str) -> str:
    """Assemble the four-part prompt for the SLM."""
    return f"""[CONTEXT]
{summary}

[QUESTION]
{current_question}

[INSTRUCTION]
Respond as a professional customer-service agent. Address the client's
concern directly. Reference specific details from the conversation context.
Keep your response to 2-3 concise sentences.

[RESPONSE]"""
```

**Example 2: Evaluating SLM Candidates with Stage-Based Analysis**

User: "I'm choosing between LLaMA-3.2-3B, Qwen-3-4B, and Gemma-3-4B for our support chatbot. How should I benchmark them?"

Approach:
1. Prepare a test set with conversations segmented by stage (Early/Mid/Late)
2. Run inference with each model using context-summarized prompts
3. Score on four dimensions using LLM-as-judge, broken down by stage

Output evaluation framework:
```python
# evaluate_slm.py
JUDGE_PROMPT = """Rate this customer-service response on four dimensions (1-5 each):

1. Human-likeness: Does it sound like a real agent, not a bot?
2. Continuity: Does it correctly reference prior conversation context?
3. Tone/Clarity: Is it professional, clear, and appropriately empathetic?
4. Task Appropriateness: Does it actually address the client's question?

Context summary: {summary}
Client question: {question}
Agent response: {response}

Return JSON: {{"human_likeness": N, "continuity": N, "tone_clarity": N, "task_appropriateness": N}}"""

def classify_stage(turn_index: int, total_turns: int) -> str:
    """Classify conversation stage based on position."""
    progress = turn_index / total_turns
    if progress <= 0.2:
        return "early"
    elif progress <= 0.9:
        return "mid"
    return "late"

def evaluate_model(model_name: str, test_set: list) -> dict:
    """Run stage-stratified evaluation for an SLM candidate."""
    results = {"early": [], "mid": [], "late": []}
    for sample in test_set:
        stage = classify_stage(sample["turn_index"], sample["total_turns"])
        response = run_inference(model_name, sample["prompt"])
        scores = llm_judge(sample["summary"], sample["question"], response)
        results[stage].append(scores)
    return {
        stage: {dim: mean(s[dim] for s in scores) for dim in scores[0]}
        for stage, scores in results.items()
    }
```

Expected result pattern (based on paper findings):
```
Model               | Early | Mid  | Late | Overall
--------------------|-------|------|------|--------
LLaMA-3.2-3B       | 3.92  | 4.18 | 4.31 | 4.15
Qwen-3-4B          | 3.85  | 4.10 | 4.22 | 4.07
Gemma-3-4B         | 3.21  | 3.45 | 3.68 | 3.44
```

**Example 3: Synthetic Training Data Generation**

User: "I have 300k single-turn banking support transcripts. How do I create multi-turn training data for fine-tuning?"

Approach:
1. Filter to 5-100 turn conversations
2. Reconstruct multi-turn sequences from chronological single turns
3. Generate context summaries for each turn position
4. Refine target responses for naturalness

Pipeline:
```python
# data_pipeline.py
def build_training_set(raw_conversations: list[dict]) -> list[dict]:
    """Five-stage synthetic data pipeline."""
    # Stage 1: Filter by turn count
    filtered = [c for c in raw_conversations if 5 <= len(c["turns"]) <= 100]

    # Stage 2: Build multi-turn sequences with history
    sequences = []
    for conv in filtered:
        for i, turn in enumerate(conv["turns"]):
            if turn["role"] == "agent":
                history = conv["turns"][:i]
                sequences.append({
                    "history": history,
                    "question": conv["turns"][i-1]["content"],
                    "target": turn["content"],
                    "turn_index": i,
                    "total_turns": len(conv["turns"])
                })

    # Stage 3: Summarize history for each sequence
    for seq in sequences:
        seq["summary"] = summarize_history(seq["history"])

    # Stage 4: Refine target responses
    for seq in sequences:
        seq["refined_target"] = refine_response(seq["target"])

    # Stage 5: Format as four-part prompt training examples
    return [format_training_example(seq) for seq in sequences]
```

## Best Practices

- **Do** enforce a strict token cap (250 tokens) on summaries -- longer summaries defeat the purpose and degrade SLM performance
- **Do** preserve exact identifiers, amounts, and dates in summaries -- SLMs hallucinate these details when they're missing from context
- **Do** use temperature 0.3 for summarization to ensure factual consistency across conversation turns
- **Do** evaluate by conversation stage, not just overall score -- aggregate metrics hide stage-specific weaknesses
- **Avoid** feeding raw conversation history to SLMs even if it fits in the context window -- summarized context consistently outperforms raw history for models under 8B parameters
- **Avoid** assuming all SLMs of similar size perform equally -- architecture and instruction-tuning quality dominate; Gemma-3-4B scored ~0.6 points lower than LLaMA-3.2-3B despite similar parameter counts
- **Avoid** skipping response refinement post-processing -- SLM outputs often sound robotic without a lightweight naturalness pass

## Error Handling

- **Summary drift over long conversations**: Rolling summaries can gradually lose early details. Mitigate by periodically re-summarizing from the full transcript (every 15-20 turns) rather than always summarizing summary + latest turn.
- **Factual hallucination in responses**: When the SLM generates account numbers, dates, or amounts not present in the summary, implement a post-generation check that validates mentioned entities against the summary content.
- **Stage misclassification**: In production you may not know the total turn count in advance. Use heuristic stage detection: classify as "early" for turns 1-3, "late" when resolution keywords appear (e.g., "anything else", "resolved", "thank you"), and "mid" for everything between.
- **Summarizer failure/timeout**: Cache the previous summary. If the summarization service fails, fall back to the last successful summary plus the raw latest turn pair appended.
- **Context window overflow**: If the summarized context + current question + instruction exceeds 512 tokens, truncate the summary from the middle (preserving the first and last sentences which contain issue identification and current state).

## Limitations

- **Domain specificity**: Results are validated on banking customer service. Other domains (healthcare, legal, technical support) may require domain-adapted summarization prompts and different fine-tuning data.
- **Summarization dependency**: The pipeline requires a capable summarizer (GPT-4o-mini tier). If you need a fully self-contained on-prem solution, you must also deploy a separate summarization model, adding complexity.
- **Early-stage weakness**: SLMs consistently underperform LLMs in the first few turns of a conversation where issue identification and empathetic framing are critical. For high-stakes first impressions, consider hybrid routing.
- **Language coverage**: The study evaluates English-language conversations only. Multi-lingual customer service would require additional validation.
- **Quantitative ceiling**: Even the best SLMs (Qwen-3-8B) only achieve a 23.8% win rate against GPT-4.1 in pairwise comparison. The approach narrows the gap substantially but does not close it for the most demanding use cases.

## Reference

**Paper**: Cooray, Sumanathilaka & Raju (2026). "Can Small Language Models Handle Context-Summarized Multi-Turn Customer-Service QA?" [arXiv:2602.00665v1](https://arxiv.org/abs/2602.00665v1). Look for: Appendix G (summarization prompt template), Appendix H (response refinement prompt), Table 3 (stage-based LLM-as-judge scores), and Table 5 (pairwise win rates) for implementation-critical details.