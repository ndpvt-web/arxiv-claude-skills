---
name: "dllm-searcher-adapting-diffusion-large"
description: "Implement the P-ReAct parallel reasoning-and-acting agent paradigm from DLLM-Searcher, which overlaps tool execution with continued reasoning to cut search agent latency by ~15-22%. Also covers the Agentic SFT and Agentic VRPO post-training pipeline for teaching diffusion LLMs agentic capabilities. Triggers: 'build a parallel ReAct agent', 'reduce search agent latency', 'overlap tool calls with reasoning', 'implement P-ReAct', 'diffusion LLM agent pipeline', 'agentic training for dLLMs'"
---

# DLLM-Searcher: Parallel Reasoning-and-Acting for Search Agents

This skill enables Claude to implement the P-ReAct (Parallel-Reasoning and Acting) agent paradigm and its supporting training pipeline from the DLLM-Searcher paper. The core idea: in standard ReAct agents, the model sits idle while waiting for tool responses (search APIs, databases, etc.), creating a serial bottleneck. P-ReAct restructures agent execution so that tool calls are dispatched as early as possible while the model continues generating reasoning tokens in parallel. Combined with a two-stage post-training pipeline (Agentic SFT + Agentic VRPO), this framework achieves search agent performance comparable to autoregressive LLM agents while delivering 12-22% inference time reduction.

## When to Use

- When building a multi-step search agent (ReAct-style) and the user wants to reduce end-to-end latency by overlapping tool execution with model reasoning
- When implementing a training pipeline to teach a diffusion LLM (SDAR, LLaDA, MDLM, Dream) to follow agentic formats like `<think>`, `<tool_call>`, and `<tool_response>`
- When designing a confidence-biased decoding strategy that prioritizes certain token spans (e.g., function calls) over others during generation
- When the user asks to build a search agent that calls Google/Bing APIs and needs to handle multi-hop question answering (HotpotQA, Musique-style tasks)
- When constructing preference optimization training data from agent rollouts, filtering for format compliance and answer correctness
- When adapting block-diffusion architectures (intra-block bidirectional attention + inter-block causal attention) for structured agent output

## Key Technique

**The Latency Problem.** Standard ReAct agents operate in a strict serial loop: Think -> generate tool_call -> wait for tool_response -> Think again. For search agents making multiple API calls, each round adds network latency on top of generation time. In a 5-round search session, tool wait time can dominate total latency.

**P-ReAct Solution.** Diffusion LLMs generate tokens non-autoregressively within blocks -- all positions are initialized as `[MASK]` and progressively unmasked based on confidence scores. P-ReAct exploits this by (1) pre-filling boundary markers `<tool_call>` and `</tool_call>` at designated positions to constrain the generation space, and (2) adding a positive confidence bias `alpha` to tokens within the tool_call span so they unmask first. Once the tool_call content is fully decoded, the API request fires immediately while the model continues unmasking reasoning tokens in the same block. The key insight: because diffusion models use bidirectional attention within blocks, the model can "know" what it wants to reason about even before those reasoning tokens are decoded -- it just surfaces the tool call first so the network request overlaps with continued generation.

**The Ability Problem and Post-Training.** Vanilla diffusion LLMs fail catastrophically at agentic tasks -- SDAR without post-training completed zero successful interactions out of 500 attempts, with 31% producing empty output and 28% never generating a tool_call. The two-stage fix: (1) **Agentic SFT** trains on ~4K high-quality trajectories from a teacher model, using a custom noising strategy that masks think/tool_call regions but carefully handles tool_response tokens to prevent train-inference mismatch. (2) **Agentic VRPO** (Variance-Reduced Preference Optimization) collects paired rollouts from the SFT model, filters for format-valid trajectories with differing correctness, and optimizes a DPO-style loss using the block-diffusion ELBO as the implicit reward, with a reference model baseline for variance reduction.

## Step-by-Step Workflow

1. **Define the agent format schema.** Establish the structured output format with three regions: `<think>...</think>` for reasoning, `<tool_call>{"name": "search", "arguments": {"query": "..."}}</tool_call>` for actions, and `<tool_response>...</tool_response>` for external results. Each agent turn is one block in the diffusion model's block-level generation.

2. **Collect teacher trajectories for Agentic SFT.** Run a strong autoregressive model (e.g., GPT-4, Claude) on your target query set using the ReAct format with your search tool. Filter trajectories that (a) produce correct final answers, (b) maintain valid format throughout all turns, and (c) contain complete reasoning paths. Target ~4K high-quality trajectories.

3. **Implement Agentic Noising for SFT.** When preparing training batches, apply diffusion noise (mask tokens) only to `think` and `tool_call` regions. For `tool_response` regions: if a tool_response appears before a generation region in the same block, fully mask it to prevent information leakage; if it appears after, preserve it as context. This prevents the model from memorizing tool outputs it won't have at inference time.

4. **Train with the Agentic ELBO objective.** Compute loss only over masked tokens that are NOT in `tool_response` regions. This teaches the model to generate reasoning and tool calls while treating external tool outputs as given context:
   ```
   Loss = E[1/t * sum over masked non-response tokens of log p(token | masked_block, previous_blocks, prompt)]
   ```

5. **Generate preference pairs for VRPO.** Run the SFT checkpoint twice per query using P-ReAct decoding with different random seeds. Keep pairs where both trajectories are format-compliant and complete, but one yields a correct answer (winner) and the other does not (loser). Target ~2K query pairs.

6. **Train Agentic VRPO.** Optimize the DPO-style loss using block-diffusion ELBO differences as implicit rewards:
   ```
   Loss_VRPO = -E[log sigmoid(beta * (advantage_winner - advantage_loser))]
   ```
   where `advantage(y) = ELBO_theta(y) - ELBO_ref(y)` and `ref` is the frozen SFT checkpoint.

7. **Implement P-ReAct decoding.** During inference, for each agent turn: (a) Initialize the block as all `[MASK]` tokens. (b) Pre-fill `<tool_call>` at position `pos_s` and `</tool_call>` at position `pos_e` (estimate positions from training data statistics). (c) At each denoising step, add bias `alpha` to confidence scores for positions between the markers. (d) Once the tool_call span is fully unmasked, dispatch the API call immediately. (e) Continue unmasking remaining think tokens while the API call is in flight.

8. **Wire up the async tool execution loop.** Structure the runtime as: decode block with P-ReAct -> fire tool call as soon as tool_call tokens are resolved -> continue block denoising -> receive tool response -> inject into next block's context -> repeat until `<answer>` tag is generated or max turns reached.

9. **Evaluate on multi-hop QA benchmarks.** Test on HotpotQA, 2WikiMultiHopQA, Musique (in-domain), and Bamboogle (out-of-domain). Measure both accuracy (exact match + LLM-as-judge) and latency (wall-clock time per query, number of denoising steps).

10. **Tune the confidence bias alpha and block size.** The bias `alpha` controls how aggressively tool_call tokens are prioritized. Too high causes quality degradation in reasoning tokens; too low fails to achieve meaningful overlap. Start with `alpha=2.0` and sweep. Block size (default 64 in the paper) controls the granularity of parallel generation -- larger blocks enable more overlap but may reduce coherence.

## Concrete Examples

**Example 1: Building a P-ReAct search agent loop in Python**

User: "I want to build a search agent that overlaps tool calls with reasoning, like P-ReAct from the DLLM-Searcher paper. Show me the core execution loop."

Approach:
1. Define an async agent loop that manages block-level generation
2. Implement confidence-biased unmasking within blocks
3. Fire tool calls as soon as the tool_call span resolves, before the block finishes

Output:
```python
import asyncio
from typing import Optional

class PReActAgent:
    """P-ReAct agent that overlaps tool execution with continued reasoning."""

    def __init__(self, dllm_model, search_tool, alpha=2.0, block_size=64, max_turns=8):
        self.model = dllm_model
        self.search_tool = search_tool
        self.alpha = alpha
        self.block_size = block_size
        self.max_turns = max_turns

    async def run(self, query: str) -> str:
        context = [{"role": "user", "content": query}]
        for turn in range(self.max_turns):
            block_result, tool_call = await self._decode_block_with_overlap(context)
            context.append({"role": "assistant", "content": block_result})
            if "<answer>" in block_result:
                return self._extract_answer(block_result)
            if tool_call:
                context.append({"role": "tool", "content": tool_call.result})
        return self._extract_answer(context[-1]["content"])

    async def _decode_block_with_overlap(self, context):
        """Decode one block using P-ReAct: bias tool_call tokens, fire async."""
        tokens = ["[MASK]"] * self.block_size
        # Pre-fill boundary markers at estimated positions
        pos_s, pos_e = self._estimate_tool_positions(context)
        tokens[pos_s] = "<tool_call>"
        tokens[pos_e] = "</tool_call>"

        tool_future: Optional[asyncio.Task] = None
        for step in range(self.block_size):
            logits, confidences = self.model.denoise_step(tokens, context)
            # Apply confidence bias to tool_call region
            for i in range(pos_s + 1, pos_e):
                if tokens[i] == "[MASK]":
                    confidences[i] += self.alpha
            # Unmask highest-confidence masked position
            best_idx = self._best_masked(tokens, confidences)
            tokens[best_idx] = self.model.decode_token(logits[best_idx])
            # Fire tool call as soon as tool_call span is fully resolved
            if tool_future is None and self._tool_span_complete(tokens, pos_s, pos_e):
                tool_query = self._parse_tool_call(tokens[pos_s:pos_e+1])
                tool_future = asyncio.create_task(self.search_tool.call(tool_query))
            if "[MASK]" not in tokens:
                break

        tool_result = await tool_future if tool_future else None
        return "".join(tokens), tool_result
```

**Example 2: Constructing Agentic SFT training data with proper noising**

User: "How do I prepare training data for teaching a diffusion LLM to follow the ReAct agent format?"

Approach:
1. Collect teacher trajectories from a strong model
2. Parse each trajectory into think/tool_call/tool_response regions
3. Apply selective masking that prevents tool_response leakage

Output:
```python
import random

def prepare_agentic_sft_batch(trajectory: str, mask_rate: float = 0.5):
    """Apply Agentic Noising: mask think/tool_call, handle tool_response carefully."""
    regions = parse_regions(trajectory)
    # regions: [("think", tokens), ("tool_call", tokens), ("tool_response", tokens), ...]
    masked = []
    labels = []
    for i, (region_type, tokens) in enumerate(regions):
        if region_type in ("think", "tool_call"):
            # Standard diffusion noising: randomly mask tokens
            for tok in tokens:
                if random.random() < mask_rate:
                    masked.append("[MASK]")
                    labels.append(tok)  # supervised target
                else:
                    masked.append(tok)
                    labels.append("[IGNORE]")  # no loss
        elif region_type == "tool_response":
            # Check if this response appears BEFORE a generation region in the same block
            next_gen_in_block = any(
                regions[j][0] in ("think", "tool_call")
                for j in range(i + 1, min(i + 3, len(regions)))
            )
            if next_gen_in_block:
                # Fully mask to prevent leakage -- model won't have this at inference
                for tok in tokens:
                    masked.append("[MASK]")
                    labels.append("[IGNORE]")  # never train on response content
            else:
                # Preserve as context
                for tok in tokens:
                    masked.append(tok)
                    labels.append("[IGNORE]")
    return masked, labels

def parse_regions(trajectory: str):
    """Split trajectory into typed regions."""
    import re
    pattern = r'(<think>.*?</think>)|(<tool_call>.*?</tool_call>)|(<tool_response>.*?</tool_response>)'
    regions = []
    for match in re.finditer(pattern, trajectory, re.DOTALL):
        text = match.group()
        if text.startswith("<think>"):
            regions.append(("think", list(text)))
        elif text.startswith("<tool_call>"):
            regions.append(("tool_call", list(text)))
        elif text.startswith("<tool_response>"):
            regions.append(("tool_response", list(text)))
    return regions
```

**Example 3: Implementing Agentic VRPO preference training**

User: "I have an SFT-trained diffusion LLM. How do I collect preference pairs and run VRPO?"

Approach:
1. Generate two rollouts per query with different random seeds
2. Filter for valid format + differing correctness
3. Compute block ELBO advantages and optimize DPO-style loss

Output:
```python
import torch
import torch.nn.functional as F

def collect_vrpo_pairs(sft_model, queries, search_tool, answer_checker):
    """Generate preference pairs from SFT model rollouts."""
    pairs = []
    for query in queries:
        traj_a = sft_model.rollout(query, search_tool, seed=42)
        traj_b = sft_model.rollout(query, search_tool, seed=137)
        # Both must be format-valid
        if not (is_format_valid(traj_a) and is_format_valid(traj_b)):
            continue
        correct_a = answer_checker(traj_a, query)
        correct_b = answer_checker(traj_b, query)
        # Need exactly one correct, one incorrect
        if correct_a and not correct_b:
            pairs.append((query, traj_a, traj_b))  # (query, winner, loser)
        elif correct_b and not correct_a:
            pairs.append((query, traj_b, traj_a))
    return pairs

def vrpo_loss(model, ref_model, query, winner, loser, beta=0.1):
    """Agentic VRPO loss using block ELBO differences."""
    elbo_w = agentic_block_elbo(model, query, winner)
    elbo_l = agentic_block_elbo(model, query, loser)
    ref_elbo_w = agentic_block_elbo(ref_model, query, winner)
    ref_elbo_l = agentic_block_elbo(ref_model, query, loser)
    advantage_w = elbo_w - ref_elbo_w
    advantage_l = elbo_l - ref_elbo_l
    return -torch.log(torch.sigmoid(beta * (advantage_w - advantage_l)))
```

## Best Practices

- **Do:** Pre-fill `<tool_call>` boundary markers at fixed positions estimated from your training data distribution. The paper uses statistics from SFT trajectories to determine typical tool_call positions within blocks.
- **Do:** Set the confidence bias `alpha` high enough to guarantee tool_call-first decoding (near 100% success rate) but validate that reasoning quality doesn't degrade. The paper achieves this at alpha values where tool_call tokens consistently unmask before think tokens.
- **Do:** Filter SFT training trajectories aggressively -- only keep trajectories with correct answers, valid format across all turns, and complete reasoning chains. Quality matters far more than quantity (the paper uses only ~4K trajectories).
- **Do:** Use the frozen SFT checkpoint as the reference model in VRPO to reduce variance. The advantage formulation `ELBO_theta - ELBO_ref` stabilizes gradients compared to raw ELBO optimization.
- **Avoid:** Training on `tool_response` content. The model should never learn to predict what a search API returns -- this creates a train-inference mismatch where the model hallucinates tool outputs instead of relying on actual API results.
- **Avoid:** Using P-ReAct with autoregressive models. The entire approach depends on bidirectional attention within blocks -- AR models cannot "know the answer before decoding it" because they generate left-to-right. P-ReAct is specific to diffusion/masked LLM architectures.

## Error Handling

- **Empty output / generation collapse:** Vanilla dLLMs produce empty output ~31% of the time without post-training. If you see this after SFT, increase training data diversity and verify that the noising strategy isn't masking too aggressively (start with mask_rate=0.5).
- **Format errors in tool_call:** ~7% of failures come from malformed JSON in tool_call regions. Add format-checking during VRPO data collection to filter these out, and consider adding a JSON schema validator as a post-processing step during inference.
- **Tool_call position estimation mismatch:** If pre-filled boundary markers are placed at positions that don't match the actual tool_call length needed, the model may truncate or pad with garbage tokens. Use dynamic position estimation based on query complexity, or allow a variable-length tool_call region with a generous buffer.
- **P-ReAct bias too aggressive:** If `alpha` is set too high, the reasoning region gets starved of unmasking budget in early steps, producing lower-quality thinking. Monitor per-region token quality (e.g., perplexity of think tokens) as you sweep alpha.
- **Multi-turn context overflow:** Each turn adds think + tool_call + tool_response to context. For long multi-hop queries (5+ turns), implement context compression or summarization of earlier tool_response content to stay within block-attention windows.

## Limitations

- **Architecture dependency:** P-ReAct requires a block-diffusion or masked-diffusion LLM backbone. It cannot be applied to standard autoregressive transformers (GPT, LLaMA, etc.), which rules out the vast majority of deployed LLMs.
- **Available backbones are weak:** Current dLLM backbones (SDAR, LLaDA, Dream) are substantially less capable than frontier AR models. Even after post-training, DLLM-Searcher trails top AR search agents on complex multi-hop tasks like Musique.
- **Fixed tool_call positioning:** Pre-filling boundary markers at estimated positions is fragile. Queries requiring unusually long or short tool calls may suffer from position mismatch.
- **Single-tool assumption:** The paper evaluates only with a single search tool. Agents needing to select from multiple tools (calculator, code interpreter, database) would need extensions to the confidence biasing scheme.
- **15-22% speedup ceiling:** The latency reduction is meaningful but modest. For applications where tool latency is the dominant bottleneck (e.g., 2+ second API calls), the overlap benefit may be small relative to total wait time since reasoning generation is fast by comparison.
- **Training data requirements:** Agentic SFT needs a strong teacher model to generate trajectories, creating a dependency on the very AR models that P-ReAct aims to replace.

## Reference

[DLLM-Searcher: Adapting Diffusion Large Language Model for Search Agents](https://arxiv.org/abs/2602.07035v1) -- Read Sections 3.2 (Agentic Noising), 3.3 (Agentic VRPO), and 4.1 (P-ReAct decoding with confidence biasing) for the core algorithmic contributions. Table 1 has the main benchmark results; Table 5 shows the alpha sensitivity analysis.