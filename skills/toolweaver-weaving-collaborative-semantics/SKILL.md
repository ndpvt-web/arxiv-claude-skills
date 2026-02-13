---
name: "toolweaver-weaving-collaborative-semantics"
description: "Design scalable tool retrieval systems using hierarchical code tokenization that captures collaborative tool semantics. Use when: 'build a tool registry with hierarchical codes', 'scale tool selection for thousands of APIs', 'encode tool co-usage patterns', 'design a generative tool retrieval pipeline', 'organize APIs into a searchable hierarchy', 'implement constrained decoding for tool selection'."
---

# ToolWeaver: Hierarchical Code-Based Tool Retrieval Systems

This skill teaches you to build scalable tool retrieval and selection systems inspired by the ToolWeaver framework (ICLR 2026). Instead of mapping each tool to a unique opaque token — which breaks down at scale and loses collaborative relationships — you encode tools as hierarchical code sequences that share intermediate codes. This makes vocabulary growth logarithmic, captures which tools work together through shared prefixes, and enables generative (autoregressive) tool selection with constrained decoding. Apply this when building agent frameworks, API registries, or any system that must select from hundreds to tens of thousands of tools.

## When to Use

- When building an agent that must select from a large catalog of tools/APIs (hundreds to tens of thousands) and naive retrieval degrades
- When you need tool recommendations that respect co-usage patterns (e.g., "users who call Weather API also call Air Quality API")
- When designing a tool registry where new tools must integrate without retraining the entire system
- When implementing generative tool selection where an LLM autoregressively produces a tool identifier instead of choosing from a flat list
- When you want to organize a large API surface into a navigable hierarchy that reflects both functionality and real usage
- When optimizing token budget: representing 47K tools with ~512 special tokens instead of 47K unique tokens

## Key Technique

**The core problem.** Standard approaches assign each tool a unique token (e.g., `<tool_3847>`). This creates two failures at scale: (1) vocabulary explodes linearly with tool count, and (2) each token is semantically isolated — the model cannot learn that `<weather_api>` and `<air_quality_api>` are related without seeing them co-occur many times in training data, which is sparse across a large library.

**ToolWeaver's solution: hierarchical residual quantization.** Each tool is encoded as a sequence of L codes drawn from L codebooks of size K. For example, with L=2 and K=256, you get 256^2 = 65,536 representable tools using only 512 added tokens. The encoding is produced by a Residual Quantized VAE (RQ-VAE) that takes a tool's embedding and progressively quantizes it: the first codebook captures coarse category (e.g., "outdoor data APIs"), and subsequent codebooks refine within that group. Critically, the training loss includes a **collaborative regularization term** — tools that frequently co-occur in usage trajectories are pulled closer in code space via a graph Laplacian penalty: `lambda * sum(A_uv * ||z_u - z_v||^2)`, where A is derived from a co-occurrence matrix. This means shared codes emerge naturally for tools that work together.

**Generative alignment and constrained decoding.** After codes are assigned, the LLM is fine-tuned in two stages: first on (query, tool-code-sequence) pairs for retrieval, then on full usage trajectories for end-to-end reasoning. At inference, a pre-computed prefix trie of all valid code sequences constrains beam search so the model only generates legitimate tool identifiers. This eliminates hallucinated tool IDs and makes selection both fast and exact.

## Step-by-Step Workflow

1. **Inventory your tools.** Collect metadata for every tool: name, description, parameter schema, category tags. Store as structured records (JSON/YAML). For each tool, produce a dense embedding by encoding its description through a text encoder (e.g., sentence-transformers or OpenAI embeddings).

2. **Build the co-occurrence matrix.** From historical usage logs or synthetic trajectories, construct a tool-tool co-occurrence matrix C where C[i][j] counts how often tools i and j appear in the same session or workflow. Normalize to get a similarity matrix A. If no usage logs exist, synthesize trajectories by prompting an LLM with task descriptions and recording which tools it selects together.

3. **Train the RQ-VAE tokenizer.** Implement a Residual Quantized VAE with L codebooks of K centroids each. The loss function has three components:
   - **Reconstruction**: `||z - z_hat||^2` (embedding recovery)
   - **Quantization**: standard commitment loss from VQ-VAE
   - **Collaborative regularization**: `lambda * sum(A_uv * ||z_hat_u - z_hat_v||^2)` (pull co-used tools together in code space)

   After training, apply the Sinkhorn-Knopp algorithm to the final codebook assignments to ensure uniform distribution across codes and avoid collision clusters.

4. **Assign hierarchical codes to every tool.** Run each tool's embedding through the trained RQ-VAE encoder to produce an L-length code sequence (e.g., `<C1_42><C2_197>`). Verify that no two tools share the same full sequence. Store the mapping in a lookup table.

5. **Build the prefix trie.** Construct a trie from all valid code sequences. Each path from root to leaf corresponds to exactly one tool. This trie will be used during constrained decoding to mask invalid next tokens at each generation step.

6. **Fine-tune the LLM — Stage 1 (Retrieval).** Create training pairs of (user_query, tool_code_sequence). Fine-tune the LLM with standard cross-entropy loss to generate the correct code sequence given a query. Add the L*K new code tokens to the tokenizer and initialize their embeddings from the RQ-VAE codebook centroids.

7. **Fine-tune the LLM — Stage 2 (Trajectory).** Using full interaction trajectories (query -> reasoning -> tool_call(code_sequence, params) -> observation -> answer), fine-tune the model to handle end-to-end tool use including parameter generation and multi-step reasoning.

8. **Implement constrained beam search for inference.** During generation, at each step where a code token is expected, mask logits of all tokens not valid according to the current trie prefix. This guarantees every generated code sequence maps to a real tool.

9. **Resolve codes to tool calls.** After the model generates a code sequence, look it up in the mapping table to get the tool name and schema. Parse the subsequent generated tokens as parameters against the tool's schema. Execute the tool and feed the result back.

10. **Iterate: add new tools incrementally.** When new tools arrive, embed them, quantize through the existing RQ-VAE (or retrain if the domain shifts significantly), assign codes to unused trie leaves, and update the trie. Minimal or no LLM retraining is needed for tools that fall within existing code clusters.

## Concrete Examples

**Example 1: Building a hierarchical tool registry for a developer platform**

```
User: I have 2,000 internal APIs and want to build a system where our
      coding assistant can select the right API from a natural language request.

Approach:
1. Export each API's OpenAPI spec. For each endpoint, concatenate the
   summary + description + parameter names into a single text block.
2. Embed all 2,000 descriptions using a sentence-transformer model,
   producing 2,000 vectors of dimension 768.
3. From internal usage telemetry, build a 2000x2000 co-occurrence matrix:
   C[i][j] = number of sessions where both API i and API j were called.
4. Train an RQ-VAE with L=2, K=64 (64^2 = 4,096 capacity, headroom for
   growth). Use lambda=0.1 for collaborative regularization.
5. Assign codes. Example output:
   - POST /weather/current  -> <C1_12><C2_31>
   - GET /air-quality       -> <C1_12><C2_45>   (same C1 = "env data" cluster)
   - POST /payments/charge  -> <C1_03><C2_17>
   - GET /payments/status   -> <C1_03><C2_22>   (same C1 = "payments" cluster)
6. Build the trie and fine-tune a local LLM (e.g., Llama-3-8B) on
   (query, code_sequence) pairs from historical request logs.
7. At inference: user asks "check if it's safe to bike outside" ->
   model generates <C1_12><C2_31>, <C1_12><C2_45> (weather + air quality).

Output: Two API calls dispatched with correct parameters, results
composed into a natural language answer.
```

**Example 2: Encoding tool relationships for a multi-agent system**

```
User: My agents use 150 tools. Some tools are commonly chained
      (e.g., search -> summarize -> translate). I want the tool selector
      to learn these chains implicitly.

Approach:
1. Log 10,000 agent trajectories. Parse each into an ordered list of
   tool calls. Build co-occurrence matrix from consecutive tool pairs
   (bigram co-occurrence captures chaining patterns).
2. Embed each tool description. Train RQ-VAE (L=3, K=16, capacity
   16^3=4,096) with collaborative regularization weighted heavily
   (lambda=0.5) to prioritize chain relationships.
3. Result: tools in common chains share prefixes:
   - web_search      -> <C1_02><C2_07><C3_11>
   - summarize_text  -> <C1_02><C2_07><C3_14>  (shares 2 prefix codes)
   - translate_text  -> <C1_02><C2_09><C3_03>  (shares 1 prefix code)
   - send_email      -> <C1_08><C2_01><C3_05>  (different cluster)
4. Fine-tune on trajectories. The model learns: after generating
   <C1_02><C2_07>..., the next tool is likely also in the <C1_02> cluster.
5. At inference: "Find recent news about fusion energy, summarize it,
   and send me the summary in French" -> model generates the three
   chained tool codes in sequence, leveraging shared prefix patterns.

Output: Correct 3-tool chain selected and executed without explicit
chain-of-thought prompting about tool relationships.
```

**Example 3: Implementing constrained decoding for tool selection**

```
User: How do I implement the trie-based constrained decoding so my
      model can't hallucinate invalid tool IDs?

Approach:
1. Build a trie from all assigned code sequences:

   trie = {}
   for tool_name, code_seq in tool_codes.items():
       node = trie
       for code in code_seq:
           node = node.setdefault(code, {})
       node["__tool__"] = tool_name

2. During autoregressive generation, at each code-token position:

   def get_valid_next_tokens(generated_so_far, trie):
       node = trie
       for code in generated_so_far:
           node = node[code]
       return list(node.keys() - {"__tool__"})

3. Apply logit masking before softmax:

   valid = get_valid_next_tokens(current_prefix, trie)
   mask = torch.full((vocab_size,), float('-inf'))
   for token_id in valid:
       mask[token_id] = 0.0
   logits = logits + mask

4. After L tokens are generated, look up the complete sequence in
   the trie to resolve the tool name. If beam search is used, each
   beam independently traverses the trie.

Output: Every generated code sequence is guaranteed to map to a
real tool. Zero hallucinated tool IDs.
```

## Best Practices

- **Do:** Use co-occurrence data from real usage logs when available. The collaborative regularization term is what distinguishes ToolWeaver from naive clustering — without it, you lose the chain-awareness that makes this approach powerful.
- **Do:** Choose L and K so that K^L exceeds your tool count by at least 2x, leaving room for new tools without retraining the codebook.
- **Do:** Initialize new code token embeddings from the RQ-VAE centroids rather than randomly. This gives the LLM a semantic starting point and accelerates fine-tuning convergence.
- **Do:** Apply the Sinkhorn-Knopp balancing step after RQ-VAE training. Without it, popular code clusters become overloaded and rare tools get poor representations.
- **Avoid:** Skipping Stage 2 (trajectory fine-tuning). Stage 1 alone teaches retrieval but not multi-step reasoning or parameter generation — you need both for production tool use.
- **Avoid:** Setting lambda too high for collaborative regularization. If lambda dominates the loss, tools that are functionally different but co-occur often will collide in code space. Start with lambda=0.1 and tune on a held-out set.

## Error Handling

- **Code collisions (two tools get the same sequence):** Increase K or L. Apply Sinkhorn-Knopp more aggressively. As a fallback, append a disambiguation suffix code.
- **Poor retrieval for niche tools:** These tools have sparse co-occurrence data, so the collaborative signal is weak. Supplement with stronger intrinsic signal by enriching their descriptions or manually assigning category tags before embedding.
- **Trie becomes stale after tool updates:** Rebuild the trie whenever tools are added or removed. This is cheap (linear in tool count) and should be part of the deployment pipeline.
- **LLM generates partial code sequence then stops:** The constrained decoder should force generation to continue until a leaf node is reached. Implement a minimum generation length equal to L for code segments.
- **New tools in unfamiliar domains:** If the RQ-VAE was trained on a narrow domain and new tools are from a different domain, the codebook may not represent them well. Retrain the RQ-VAE with the expanded tool set.

## Limitations

- Requires usage trajectory data (or synthetic surrogates) to build meaningful co-occurrence signals. Without this, the method reduces to standard embedding quantization and loses its key advantage.
- The RQ-VAE training and LLM fine-tuning stages require compute investment. This is not a zero-shot approach — it pays off only when you have a stable, large tool library that justifies the upfront cost.
- Constrained decoding adds inference latency proportional to beam width and code sequence length. For real-time applications with strict latency budgets, this tradeoff must be evaluated.
- Tool descriptions must be informative. If many tools have vague or identical descriptions, the intrinsic embeddings will cluster poorly regardless of the quantization method.
- The method assumes tools have relatively stable co-usage patterns. In environments where tool relationships change rapidly, the codebook needs frequent retraining.

## Reference

[ToolWeaver: Weaving Collaborative Semantics for Scalable Tool Use in Large Language Models](https://arxiv.org/abs/2601.21947v1) (ICLR 2026). Focus on Section 3 for the RQ-VAE tokenization with collaborative regularization (Eq. 7), Section 4 for the two-stage generative alignment procedure, and Figure 1a for the logarithmic scaling analysis.