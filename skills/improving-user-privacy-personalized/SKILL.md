---
name: "improving-user-privacy-personalized"
description: "Build privacy-preserving personalized LLM systems using the P³ (Private Personalized Prediction) client-server architecture. A large server model generates draft tokens from the query alone, while a small client model with retrieval access to the user's private profile evaluates and modifies those drafts—keeping personal data off the server. Use when: 'build a personalized API that keeps user data private', 'implement speculative decoding with client-side personalization', 'design a privacy-preserving recommendation system', 'create a split client-server LLM pipeline', 'add retrieval-augmented personalization without exposing profiles', 'implement P³ framework for private personalized generation'."
---

# Private Personalized Prediction (P³): Client-Side Retrieval-Augmented Modification of Server Drafts

This skill enables Claude to design and implement privacy-preserving personalized generation systems based on the P³ framework. Instead of sending a user's private profile to a powerful cloud LLM, the system keeps all personal data on the client. A large server-side model generates draft token sequences from the query alone, and a small client-side model—augmented with retrieval over the user's private profile—evaluates, accepts, or modifies those drafts to reflect user preferences. This achieves 90-96% of the quality of full-profile exposure while leaking only 1.5-3.5% additional information beyond a non-personalized baseline.

## When to Use

- When building a personalized text generation service where user profile data must not leave the client device (e.g., health records, financial history, private preferences).
- When implementing a speculative-decoding pipeline where the verifier is a smaller, personalized client model rather than the same large model.
- When designing a retrieval-augmented generation (RAG) system that splits retrieval (client) from generation (server) for privacy.
- When the user needs personalized question answering, email drafting, or content recommendation without transmitting personal context to the cloud.
- When optimizing an edge-deployment scenario where the client should generate as few tokens as possible (target: ~9% of total tokens on-device).
- When evaluating privacy guarantees of a personalized system against linkability and attribute inference attacks.

## Key Technique

**The core insight of P³ is to repurpose speculative decoding for privacy.** In standard speculative decoding, a small draft model proposes tokens and a large verifier accepts or rejects them for speed. P³ inverts the roles: the *large* server model drafts tokens (without seeing any personal data), and the *small* client model verifies and modifies those drafts using retrieval-augmented context from the user's private profile. This means the server never sees personal information, yet the final output reflects user preferences because the client reshapes the server's high-quality drafts.

**The retrieval-augmented client operates as follows.** When the client receives k draft tokens from the server, it retrieves relevant passages from the user's local profile store (using a lightweight retriever like BM25 or a small dense encoder). It then runs a small local LLM (e.g., a 1-3B parameter model) conditioned on the query, the retrieved profile context, and the draft tokens. For each draft token position, the client computes its own probability distribution. If the server's draft token is acceptable under the client's personalized distribution (using a stochastic acceptance criterion similar to speculative decoding), it is kept. Otherwise, the client resamples from a corrected distribution that blends the server's general knowledge with the client's personalized knowledge. This continues until the end-of-sequence token is generated.

**Privacy is preserved structurally, not just procedurally.** The server only ever sees the user's query—never profile data, retrieved documents, or modified tokens beyond what it would see in a non-personalized interaction. Privacy analyses using linkability attacks (can an attacker link two sessions to the same user?) and attribute inference attacks (can the server infer user attributes from the interaction?) show P³ adds only 1.5-3.5% leakage over a fully non-personalized baseline. The client generates only ~9.2% of final tokens, making it practical for edge devices with limited compute.

## Step-by-Step Workflow

1. **Define the profile schema and storage.** Design the user profile as a collection of documents (past interactions, preferences, demographic notes, domain-specific records). Store these locally on the client in a format amenable to retrieval (e.g., a JSON document store or a local SQLite database with text fields).

2. **Set up the client-side retriever.** Implement a lightweight retrieval index over the user's profile documents. Use BM25 (via `rank_bm25` or Lucene) for keyword matching or a small dense retriever (e.g., `sentence-transformers/all-MiniLM-L6-v2`) for semantic search. The retriever must run entirely on-device.

3. **Select and deploy the server-side LLM.** Choose a large model (e.g., GPT-4, Claude, Llama-70B) deployed as an API. Configure it to accept a query and return k draft tokens per round. The server endpoint should support streaming or batched token generation with logits/probabilities exposed if possible.

4. **Select and deploy the client-side LLM.** Choose a small model that fits the client hardware (e.g., Llama-3.2-1B, Phi-3-mini, Gemma-2B). Quantize to 4-bit if needed for mobile/edge. This model must support conditional generation given a prompt with retrieved context.

5. **Implement the draft-verify-modify loop.** Code the core P³ cycle:
   - Server generates k draft tokens from `[query]` only.
   - Client retrieves top-n profile passages relevant to `[query + draft_so_far]`.
   - Client model computes token probabilities conditioned on `[query + retrieved_context + accepted_tokens_so_far]`.
   - For each draft token position i (1 to k): accept the server's token with probability `min(1, p_client(token_i) / p_server(token_i))`. If rejected, resample from the residual distribution `max(0, p_client - p_server)` normalized.
   - Append accepted/resampled tokens to the output. Send the last accepted position back to the server as context for the next round.
   - Repeat until EOS is generated.

6. **Tune the draft length k.** Experiment with k values (e.g., 4, 8, 16). Larger k reduces round-trips but increases the chance of rejection cascades. A good starting point is k=5 for QA tasks and k=8 for longer-form generation. Measure acceptance rate—target >70% for efficiency.

7. **Implement the communication protocol.** Design the client-server API contract:
   - Request: `{query: str, prefix: str, k: int}`
   - Response: `{draft_tokens: list[int], draft_logprobs: list[dict]}` (logprobs per token over vocabulary)
   - Keep payloads minimal. Never send profile data, retrieved passages, or client-model outputs to the server.

8. **Add privacy safeguards.** Implement guardrails: (a) strip any profile-derived content from server-bound messages, (b) audit the communication channel to ensure only query + prefix are transmitted, (c) optionally add differential noise to the prefix to limit linkability across sessions.

9. **Evaluate quality and privacy.** Measure output quality using task-specific metrics (ROUGE, F1, accuracy) against three baselines: non-personalized server-only, personalized client-only, and leaky full-profile-to-server upper bound. Target 90%+ recovery of the leaky upper bound. Run linkability and attribute inference attacks to quantify privacy leakage.

10. **Optimize for deployment.** Profile the client-side compute: measure what fraction of tokens the client generates (target ≤10%), latency per round-trip, and peak memory on the target device. Reduce retriever index size, quantize the client model further, or reduce k if constraints are tight.

## Concrete Examples

**Example 1: Privacy-Preserving Personalized QA Service**

User: "Build a personalized question-answering API where user reading history stays on-device but answers reflect their expertise level."

Approach:
1. Store each user's reading history as a local document collection indexed by BM25.
2. Deploy Llama-70B on the server behind a `/draft` endpoint that accepts `{query, prefix, k}` and returns draft tokens with log-probabilities.
3. Run Llama-3.2-1B quantized (4-bit) on the client.
4. On each question, the client retrieves the 5 most relevant articles from the user's history, then runs the P³ draft-verify-modify loop with k=5.
5. The server never sees the user's reading history—only the question and accepted prefix.

Output architecture:
```
Client Device                          Cloud Server
+--------------------------+           +-------------------------+
| User Profile Store       |           | Llama-70B /draft API    |
|  - reading_history.db    |           |  Input: query + prefix  |
|  - BM25 index            |           |  Output: k draft tokens |
|                          |  query    |         + logprobs      |
| Retriever (BM25)         | -------> |                         |
|    |                     | <------- |                         |
|    v                     |  drafts   +-------------------------+
| Llama-3.2-1B (4-bit)    |
|  - Verify/modify drafts  |
|  - Output final tokens   |
+--------------------------+
```

**Example 2: Private Email Draft Personalization**

User: "I want an email assistant that matches my writing style from past emails, but I don't want my emails sent to the cloud."

Approach:
1. Index the user's sent emails locally using a dense retriever (`all-MiniLM-L6-v2`).
2. Server-side model generates draft email tokens from the subject line and recipient only.
3. Client retrieves 3 stylistically similar past emails and uses Phi-3-mini to evaluate drafts.
4. Acceptance criterion: if the server's token aligns with the user's typical phrasing (high client probability), keep it. Otherwise, resample to inject the user's voice.
5. Result: emails sound like the user wrote them; the server only ever sees "Write an email to [recipient] about [subject]."

Implementation snippet (Python pseudocode):
```python
def p3_generate(query, server, client_model, retriever, profile, k=5):
    prefix = ""
    output_tokens = []

    while True:
        # Server drafts k tokens from query + prefix (no profile data)
        drafts = server.generate_drafts(query=query, prefix=prefix, k=k)

        # Client retrieves relevant profile context
        context = retriever.search(profile, query + prefix, top_n=5)

        # Client evaluates each draft token
        client_logprobs = client_model.get_logprobs(
            prompt=f"{query}\n{context}\n{prefix}",
            num_tokens=k
        )

        accepted = 0
        for i, draft_token in enumerate(drafts.tokens):
            p_client = client_logprobs[i][draft_token]
            p_server = drafts.logprobs[i][draft_token]
            # Stochastic acceptance
            if random.random() < min(1.0, p_client / p_server):
                output_tokens.append(draft_token)
                accepted += 1
            else:
                # Resample from corrected distribution
                residual = normalize(max(0, p_client - p_server))
                resampled = sample(residual)
                output_tokens.append(resampled)
                break  # Restart drafting from this point

        prefix = decode(output_tokens)
        if output_tokens[-1] == EOS_TOKEN:
            break

    return prefix
```

**Example 3: Adding P³ to an Existing RAG Pipeline**

User: "We already have a RAG system that sends user documents to our cloud LLM. Refactor it so user documents stay local."

Approach:
1. Move the retriever and document store from the server to the client.
2. Replace the single server call (`generate(query + retrieved_docs)`) with the P³ loop: server generates drafts from query alone, client modifies using local retrieval.
3. Update the server API to expose log-probabilities alongside generated tokens.
4. Add a lightweight client-side model (can be a fine-tuned distillation of the server model).
5. Measure quality regression—expect to recover 90-96% of original quality while eliminating profile exposure.

Migration checklist:
```
[x] Move document store to client (SQLite + FTS5 or FAISS)
[x] Move retriever to client (BM25 or MiniLM)
[x] Add client-side LLM (Gemma-2B or Phi-3-mini)
[x] Modify server endpoint to return logprobs
[x] Implement draft-verify-modify loop on client
[x] Remove profile/document fields from server API contract
[x] Audit network traffic to confirm no profile leakage
[x] Benchmark quality vs. old pipeline (target ≥90% recovery)
```

## Best Practices

- **Do:** Expose token-level log-probabilities from the server model. The acceptance criterion requires comparing `p_client / p_server` per token. Without server logprobs, you must fall back to heuristic acceptance (e.g., perplexity-based), which degrades quality.
- **Do:** Keep the client retriever fast and lightweight. BM25 is often sufficient and avoids GPU requirements on-device. Only use dense retrieval if semantic matching is critical for your domain.
- **Do:** Fine-tune or distill the client model on domain-relevant data if quality recovery falls below 90%. The client model does not need to be generally capable—it needs to be good at distinguishing personalized from generic tokens in your domain.
- **Do:** Audit the wire protocol regularly. Log and inspect every byte sent to the server. The privacy guarantee depends entirely on the server never receiving profile data.
- **Avoid:** Sending the modified output back to the server as a "full context" for the next turn. Only send the accepted token prefix—never explanations of why tokens were modified or what profile content influenced the change.
- **Avoid:** Using very large k values (>16) without measuring acceptance rates. High rejection rates cause the client to generate most tokens, defeating the efficiency benefit and potentially degrading quality since the small model takes over.

## Error Handling

- **Server logprobs unavailable:** If the server API doesn't expose per-token log-probabilities, implement an approximation: run the client model in scoring mode on the draft sequence and use a threshold on the client's perplexity per token to decide acceptance. Quality will degrade ~3-5%.
- **Client model too slow:** If the verify step creates unacceptable latency, reduce k to lower per-round client compute, or use a smaller/more-quantized client model. Alternatively, batch the verification step using GPU parallelism over the k positions.
- **Low acceptance rate (<50%):** This indicates the server and client distributions diverge significantly—likely the client's personalized preferences are far from the server's generic output. Increase the retrieval context (more profile passages), fine-tune the client model, or reduce k so the server can adapt sooner to client modifications.
- **Profile retrieval returns irrelevant results:** If personalization quality is poor despite correct architecture, the bottleneck is likely retrieval. Improve the profile index: add metadata filters, use hybrid retrieval (BM25 + dense), or re-chunk profile documents into smaller, more topically focused passages.
- **Privacy leakage detected in audit:** If network inspection reveals profile-derived content in server-bound messages, check for prompt injection through retrieved content, ensure the prefix encoder strips non-token data, and add an output sanitizer between the client's modification step and the server communication layer.

## Limitations

- **Requires server-side logprob access.** Many commercial LLM APIs (e.g., some configurations of GPT-4, Claude) do not expose per-token log-probabilities, which is needed for the stochastic acceptance criterion. Workarounds (perplexity thresholds) reduce quality.
- **Client must run an LLM.** Even a small model (1-3B parameters, 4-bit quantized) requires ~1-2 GB RAM and a capable processor. This is feasible on modern smartphones and laptops but not on all IoT or embedded devices.
- **Added latency from round-trips.** Each draft-verify cycle requires a network round-trip. For interactive applications, this adds latency proportional to `(output_length / k) * round_trip_time`. Use larger k and optimize network to mitigate.
- **Not suitable for fully offline use.** The server-side LLM is essential for quality. If the server is unreachable, the system must fall back to the client model alone, which produces noticeably lower quality output.
- **Domain-specific fine-tuning often needed.** The client model's effectiveness at personalization depends on its ability to score tokens in the target domain. Out-of-the-box small models may not adequately distinguish between personalized and generic tokens for specialized domains (medical, legal, etc.).

## Reference

**Paper:** Salemi, A. & Zamani, H. (2026). *Improving User Privacy in Personalized Generation: Client-Side Retrieval-Augmented Modification of Server-Side Generated Speculations.* arXiv:2601.17569v1. [https://arxiv.org/abs/2601.17569v1](https://arxiv.org/abs/2601.17569v1)

Look for: Algorithm 1 (the P³ loop pseudocode), Table 2 (quality recovery percentages across LaMP-QA datasets), Section 5 (privacy analysis with linkability and attribute inference metrics), and Section 6 (efficiency analysis showing the 9.2% client token generation ratio).