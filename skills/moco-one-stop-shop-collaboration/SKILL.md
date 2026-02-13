---
name: "moco-one-stop-shop-collaboration"
description: "Design and implement multi-LM collaboration pipelines using the MoCo framework's 26 methods across four collaboration levels (API routing, text exchange, logit fusion, weight merging). Use when: 'set up model collaboration pipeline', 'combine multiple LLMs for better accuracy', 'implement multi-agent debate or refinement', 'route queries to the best model', 'merge model weights', 'build an LLM ensemble system'."
---

# MoCo: Multi-Model Collaboration Pipelines

This skill enables Claude to design, configure, and implement multi-LM collaboration systems using the MoCo library and its underlying principles. MoCo provides 26 collaboration methods organized across four levels of cross-model information exchange: API-level routing (directing queries to the right model), text-level collaboration (models refining each other's outputs), logit-level fusion (combining token probability distributions), and weight-level merging (combining model parameters). Claude uses this skill to help users select the right collaboration strategy for their task, write MoCo configs, build custom collaboration pipelines, and avoid pitfalls identified in large-scale benchmarking of these methods.

## When to Use

- When the user wants to combine multiple language models to outperform any single model on reasoning, QA, code generation, or safety tasks
- When the user asks to implement multi-agent debate, refinement loops, or voting systems across LLMs
- When the user needs to route incoming queries to the most suitable model from a pool (prompt routing, trained routers, cascade)
- When the user wants to merge model weights (model soups, DARE-TIES, LoraHub) for a single stronger model
- When the user asks to set up the MoCo library, write configuration files, or benchmark collaboration strategies
- When the user wants to understand which collaboration method fits their compute budget, model pool, and task type
- When the user asks about LLM ensembling, model fusion, or collaborative decoding techniques

## Key Technique

MoCo's core insight is that model collaboration operates at four distinct levels of information exchange, each with different tradeoffs. **API-level** methods (9 methods: prompt routing, trained router, graph routing, cascade, Co-LLM, switch generation, nudging, mentor collab, Collm) treat models as black boxes and select or sequence them per query — lowest overhead but limited synergy. **Text-level** methods (11 methods: multiagent refine, multiagent feedback, majority vote, LLM Blender, heterogeneous swarms, knowledge cards, structured interaction, BBMAS, Sparta alignment, AggLM, multiagent finetuning) exchange generated text between models for iterative improvement — highest general applicability and strongest average gains. **Logit-level** methods (2 methods: logit fusion, logit contrastive) merge next-token probability distributions during decoding — requires access to model logits but enables fine-grained collaboration. **Weight-level** methods (5 methods: greedy soup, DARE-TIES, model swarms, LoraHub, ExPO) operate directly on model parameters — requires shared architecture but produces a single merged model with zero additional inference cost.

The empirical finding across 25 benchmarks is that collaboration outperforms single models in 61% of (model, data) settings, with the best methods achieving up to 25.8% improvement. Text-level and weight-level methods scale best with increasing model pool size. Critically, model diversity matters more than model count: 8 diverse models outperform 8 copies of the same architecture. Collaboration also exhibits "collaborative emergence" — solving 18.5% of problems that no individual model could solve alone.

The practical decision framework is: use **API-level routing** when you need low latency and have heterogeneous specialized models; use **text-level collaboration** as the default for general tasks since it works across any model combination; use **logit-level** methods when you need fine-grained token-level control and have logit access; use **weight-level** methods when all models share an architecture and you want a single deployable model with no runtime overhead.

## Step-by-Step Workflow

1. **Characterize the task and constraints.** Identify the task type (reasoning, QA, code, safety), available models (homogeneous vs. heterogeneous, open-weight vs. API-only), compute budget (can you run multiple models at inference, or need a single merged model?), and whether you have a development set for training routers/rankers.

2. **Select the collaboration level.** Apply this decision tree:
   - API-only access to models + low latency needed → **API-level** (prompt routing, cascade)
   - Open models + heterogeneous architectures + accuracy priority → **Text-level** (multiagent refine, LLM Blender, majority vote)
   - Open models + logit access + token-level precision needed → **Logit-level** (logit fusion)
   - Open models + shared architecture + want single deployable artifact → **Weight-level** (model swarms, DARE-TIES, LoraHub)

3. **Install MoCo and prepare the environment.**
   ```bash
   pip install modelco
   # or from source:
   conda env create -f environment.yml && conda activate model_collaboration
   ```

4. **Write a MoCo configuration file** specifying the collaboration method, model pool, dataset, GPU allocation, and batch size. Use the examples/ directory as templates. Key fields: `method`, `models` (list of HuggingFace model IDs or API endpoints), `task`, `task_type`, `gpu_ids`, `batch_size`.

5. **Configure the model pool for maximum diversity.** Select models that differ in training data, architecture, or specialization. Avoid pools of near-identical models — the "artificial hivemind" phenomenon degrades routing and voting methods.

6. **Run the collaboration pipeline.**
   ```bash
   moco -c config.json --log_dir logs/
   # or:
   python -m model_collaboration.main -c config.json
   ```

7. **Evaluate and compare results.** MoCo outputs per-method, per-dataset metrics to the log directory. Compare against single-model baselines that MoCo also generates. Look for collaborative emergence — problems solved by the ensemble that no single model got right.

8. **Iterate on method selection.** If text-level refinement degrades safety tasks (a known failure mode where consensus-seeking compounds refusal errors), switch to routing or voting. If routing underperforms with general-purpose models, switch to text-level debate.

9. **For production deployment**, decide between runtime collaboration (text/logit-level, higher latency but adaptive) or offline merging (weight-level, single model inference cost). Weight-level methods like model swarms or LoraHub produce a single checkpoint you deploy normally.

10. **Build custom collaboration methods** by extending MoCo's base classes. Implement the `collaborate()` interface, register the method, and benchmark against existing methods on your target dataset.

## Concrete Examples

**Example 1: Setting up multi-agent debate for math reasoning**

User: "I have access to Llama-3-70B and Mixtral-8x22B via API. I want them to collaborate on GSM8K math problems. What's the best approach?"

Approach:
1. Task is math reasoning with heterogeneous open models → text-level collaboration is the best fit
2. Multiagent Refine is the strongest text-level method for reasoning: each model generates an answer, then iteratively refines based on seeing the other's response
3. Write MoCo config:

```json
{
  "method": "multiagent_refine",
  "models": ["meta-llama/Llama-3-70B-Instruct", "mistralai/Mixtral-8x22B-Instruct-v0.1"],
  "task": "gsm8k",
  "task_type": "reasoning",
  "refinement_rounds": 3,
  "gpu_ids": [0, 1, 2, 3],
  "batch_size": 4
}
```

4. Run: `moco -c math_collab.json --log_dir logs/gsm8k_refine/`
5. Compare against single-model baselines in logs. Expect 5-15% improvement on GSM8K with 2-3 refinement rounds.

Output: Per-problem accuracy scores, ensemble accuracy vs. individual model accuracy, and a breakdown of "collaboratively emerged" solutions (problems neither model solved alone).

---

**Example 2: Merging LoRA adapters for a single deployable model**

User: "I have 6 task-specific LoRA adapters fine-tuned on Llama-3-8B. I want to combine them into one general-purpose model."

Approach:
1. Shared base architecture (Llama-3-8B) + want single model → weight-level collaboration
2. LoraHub is designed exactly for this: gradient-free optimization of LoRA combination coefficients using a small dev set
3. Write MoCo config:

```json
{
  "method": "lorahub",
  "base_model": "meta-llama/Llama-3-8B-Instruct",
  "lora_adapters": [
    "adapters/math_lora", "adapters/code_lora", "adapters/safety_lora",
    "adapters/qa_lora", "adapters/summarization_lora", "adapters/chat_lora"
  ],
  "dev_set": "data/general_eval_200.jsonl",
  "task_type": "general",
  "optimization_steps": 100
}
```

4. Run: `moco -c lorahub_merge.json --log_dir logs/lorahub/`
5. Output is a single merged LoRA checkpoint with optimized combination weights. Deploy as a standard LoRA adapter on the base model — zero additional inference cost.

Output: Merged adapter at `logs/lorahub/merged_lora/`, combination coefficients per adapter, and evaluation scores on the dev set showing merged model vs. individual adapters.

---

**Example 3: Building a cost-efficient query routing system**

User: "I have GPT-4o, Claude, and several smaller open models. I want to route easy queries to cheap models and hard queries to expensive ones."

Approach:
1. Heterogeneous models including API-only + cost constraint → API-level cascade or trained router
2. Cascade is ideal: try the cheapest model first, escalate to more expensive models only when confidence is low
3. If a development set is available, use a trained router instead for direct query-to-model mapping

```json
{
  "method": "cascade",
  "models": [
    {"name": "Llama-3-8B", "cost": 0.01},
    {"name": "Llama-3-70B", "cost": 0.10},
    {"name": "gpt-4o", "cost": 1.00}
  ],
  "confidence_threshold": 0.85,
  "task": "custom",
  "data_path": "data/mixed_queries.jsonl",
  "task_type": "qa"
}
```

4. Run the cascade. Queries where the small model's confidence exceeds 0.85 get answered directly; uncertain queries escalate up the chain.
5. Monitor the routing distribution in logs to verify cost savings.

Output: Per-query routing decisions, total cost reduction vs. always using the most expensive model (typically 40-70% savings for <5% accuracy loss), and accuracy breakdown by routing tier.

---

**Example 4: Implementing collaboration without MoCo (standalone Python)**

User: "I don't want to install MoCo. Can you implement majority voting across three models in plain Python?"

Approach:
1. Majority vote is the simplest text-level collaboration method — each model generates independently, the most common answer wins

```python
import asyncio
from openai import AsyncOpenAI

client = AsyncOpenAI()

MODELS = ["gpt-4o-mini", "gpt-4o", "o3-mini"]

async def get_response(model: str, prompt: str) -> str:
    response = await client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
        temperature=0.0,
    )
    return response.choices[0].message.content.strip()

async def majority_vote(prompt: str) -> str:
    responses = await asyncio.gather(
        *[get_response(m, prompt) for m in MODELS]
    )
    # Normalize and count
    from collections import Counter
    vote_counts = Counter(responses)
    winner, count = vote_counts.most_common(1)[0]
    return winner

# Usage
answer = asyncio.run(majority_vote("What is 127 * 43?"))
```

2. For structured answers, extract the final answer (e.g., a number) before voting to handle formatting differences.
3. This can be extended to weighted voting by using model confidence scores.

## Best Practices

- **Do:** Maximize model diversity in your pool. 4 architecturally different models consistently outperform 8 similar ones. Mix model sizes, training data distributions, and specializations.
- **Do:** Start with text-level methods (majority vote or multiagent refine) as your baseline — they work across any model combination and require no special access.
- **Do:** Use a held-out development set when using trained routers, rankers, or weight-merging optimizers. Without one, gradient-free methods like LoraHub and prompt routing degrade.
- **Do:** Monitor for collaborative emergence — check whether the ensemble solves problems no individual model handles. This validates that collaboration is adding value beyond just picking the best single model.
- **Avoid:** Using multiagent refinement or debate on safety/alignment tasks. Consensus-seeking between models can compound refusal errors or override individual safety guardrails.
- **Avoid:** Building homogeneous pools of models from the same family for routing — the "artificial hivemind" phenomenon makes routing decisions nearly random. Routing works best with genuinely specialized models.
- **Avoid:** Logit-level or weight-level methods when models don't share a tokenizer or architecture. These methods require compatible parameter/vocabulary spaces.

## Error Handling

- **Models produce inconsistent output formats:** In text-level collaboration (voting, blending), models may format answers differently. Normalize outputs by extracting structured answers (numbers, option letters, code blocks) before aggregation.
- **Refinement loops diverge or degrade:** Set a maximum number of refinement rounds (2-3 is usually optimal). Monitor for quality degradation across rounds — if round N+1 scores lower than round N, stop early.
- **Routing sends everything to one model:** This indicates the router can't distinguish query difficulty or model strengths. Either use a more diverse model pool, train the router on a larger dev set, or fall back to text-level collaboration.
- **Weight merging produces a degenerate model:** Check that all models share the exact same architecture and tokenizer. Even minor differences (different vocab sizes, different head counts) will produce garbage. Use DARE-TIES with sparsity parameters to reduce interference between merged weights.
- **MoCo out-of-memory errors:** Reduce `batch_size`, reduce `gpu_ids` to match available GPUs, or use API-level methods that don't require loading models locally.

## Limitations

- **Latency:** Text-level collaboration (especially multi-round refinement) multiplies inference time by the number of models times the number of rounds. Not suitable for real-time applications without careful latency budgeting.
- **39% failure rate:** Collaboration does *not* universally help — in 39% of (model, data) settings, single models match or beat the collaborative system. Always benchmark against single-model baselines.
- **Weight-level methods require architectural homogeneity:** You cannot merge a Llama model with a Mistral model at the parameter level. This limits weight-level collaboration to model families.
- **Logit-level methods require a shared tokenizer:** Logit fusion across different tokenizers requires expensive vocabulary alignment that often eliminates the performance benefit.
- **MoCo focuses on open-weight models for most methods.** API-level routing and text-level methods work with API-only providers, but logit and weight methods need full model access.
- **Development set dependency:** Trained routers, LoraHub, model swarms, and AggLM all need labeled examples for optimization. Performance degrades significantly without representative dev data.

## Reference

- **Paper:** [MoCo: A One-Stop Shop for Model Collaboration Research](https://arxiv.org/abs/2601.21257) — Feng et al., 2026. See Table 2 for the full method-by-task performance matrix, Section 5.2 for scaling analysis, and Section 5.4 for collaborative emergence results.
- **Code:** [github.com/BunsenFeng/model_collaboration](https://github.com/BunsenFeng/model_collaboration) — Install via `pip install modelco`. Check `examples/` for config templates per method.