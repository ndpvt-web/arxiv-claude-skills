---
name: "improving-user-privacy-personalized"
description: >
  Build privacy-preserving personalized generation systems using the P^3 (Private Personalized Prediction)
  client-server split architecture. A large server model drafts tokens from the query alone; a small
  client model with retrieval access to the user's private profile verifies and corrects those drafts.
  Use this skill when asked to:
  - "Build a personalized LLM pipeline that keeps user data private"
  - "Implement speculative decoding with a client-side privacy filter"
  - "Design a retrieval-augmented system where the user profile never leaves the device"
  - "Create a draft-then-verify loop between a cloud LLM and an edge model"
  - "Add personalization to an LLM service without exposing user history to the server"
  - "Implement P^3 or privacy-preserving personalized generation"
---

# P^3: Privacy-Preserving Personalized Generation via Client-Side Draft Modification

This skill enables Claude to design and implement systems where a powerful server-side LLM generates draft tokens from a user query alone, and a small client-side model -- with retrieval access to the user's private profile -- evaluates and corrects those drafts token-by-token. The result is personalized output that recovers 90-96% of full-profile-exposure quality while leaking only 1.5-3.5% more information than sending the bare query. The technique is based on the P^3 framework (Salemi & Zamani, SIGIR 2026).

## When to Use

- When building a personalized LLM service and the user's profile, history, or preferences must not be sent to a cloud API
- When implementing a client-server generation pipeline where the server drafts and the client corrects
- When designing retrieval-augmented generation that runs retrieval on-device against a local user profile store
- When the user wants speculative-decoding-style token verification but with intentional distribution shifts for personalization
- When adding PII filtering to a generation pipeline that communicates between a local and remote model
- When deploying a lightweight edge model (3-4B parameters) that modifies output from a larger cloud model (12-14B)

## Key Technique

**Reversed speculative decoding with a privacy split.** Standard speculative decoding uses a small model to propose tokens and a large model to verify them, preserving the large model's distribution exactly. P^3 inverts this: the large server-side model proposes k draft tokens conditioned only on the user query (no profile data), and a small client-side model verifies each token against a retrieval-augmented prompt that includes the user's private context. When the client model's probability ratio for a draft token falls below a threshold tau, the token is rejected and replaced with the client model's top prediction. This intentionally shifts the output distribution toward personalization -- the opposite goal of standard speculative decoding.

**The privacy guarantee comes from the architecture itself.** The server never sees the user profile. It receives only the original query and the stream of accepted/corrected tokens (with PII scrubbed). The client model runs locally with full access to the user's profile via dense retrieval (Contriever over the profile documents). Privacy analysis using linkability attacks (can the server identify which user generated a response?) and attribute inference attacks (can the server infer user attributes from the response?) shows P^3 adds only 1.5-3.5% leakage over a non-personalized baseline -- essentially the information inherent in any personalized answer.

**Efficiency.** The client model generates only ~9.2% of total tokens (corrections), so the edge device does minimal compute. The bulk of generation stays on the server. With k=10 draft tokens per round, the system balances network round-trips against correction granularity.

## Step-by-Step Workflow

1. **Define the profile store.** Structure user data as a collection of text documents (past queries, preferences, interaction history, knowledge base entries). Store these locally on the client in a format amenable to dense retrieval (e.g., a FAISS index over Contriever embeddings).

2. **Set up the retrieval pipeline on the client.** Use `facebook/contriever-msmarco` or a similar dense retriever to index profile documents. At query time, retrieve the top m=10 documents most relevant to the user's current query. This retrieval runs entirely on-device.

3. **Implement PII detection and filtering.** Before any token stream is sent back to the server, scan for PII patterns (emails, phone numbers, SSNs, dates of birth, credit card numbers) using regex or a library like Presidio. Replace detected PII with `[PII]` markers. This is a safety net -- the architecture already limits exposure, but PII filtering adds defense in depth.

4. **Configure the server-side draft generator.** Deploy a large instruction-tuned model (e.g., Qwen 2.5 14B or Gemma 3 12B) via vLLM or a similar serving framework. The server prompt contains only the user query and the history of accepted tokens from prior rounds -- never any profile data.

5. **Configure the client-side verifier.** Deploy a small instruction-tuned model from the same family (e.g., Qwen 2.5 3B or Gemma 3 4B) on the client device. The client prompt contains the user query plus the retrieved profile context (m=10 documents).

6. **Implement the draft-verify-modify loop.** For each round:
   - Server generates k=10 draft tokens autoregressively from its prompt.
   - Server sends these k tokens to the client.
   - Client evaluates each token T_i by computing: `ratio = P_client(T_i | H_client) / P_client(T_i_star | H_client)` where `T_i_star = argmax P_client(token | H_client, verified_tokens_so_far)`.
   - If `ratio >= tau` (default tau=0.05), accept the draft token.
   - If `ratio < tau`, reject it and substitute `T_i_star`.
   - After processing all k tokens, PII-filter the accepted/corrected sequence.
   - Send the verified token sequence back to the server.
   - Both sides append verified tokens to their respective histories.

7. **Handle termination.** The loop ends when an end-of-sequence token is generated (by either side). The client assembles the full response from all verified token sequences.

8. **Tune hyperparameters.** Start with k=10 and tau=0.05. Increase k to reduce network round-trips (at the cost of more wasted draft tokens on rejection). Decrease tau to accept more server tokens (faster, less personalized). Increase tau to reject more (slower, more personalized). Test on a held-out set from your domain.

9. **Run privacy validation.** Implement linkability and attribute inference attacks against your own system: given the query and response, can an attacker (using a strong model like GPT-4) identify the user's profile or infer sensitive attributes? Compare against a non-personalized baseline to quantify leakage.

10. **Deploy with monitoring.** Log the accept/reject ratio per round. A healthy system shows 85-92% acceptance (client generates ~9% of tokens). If rejection rates spike, the server and client models may be too divergent -- consider fine-tuning the client model or adjusting tau.

## Concrete Examples

**Example 1: Personalized Q&A Service**

```
User: "Build a personalized question-answering API where user reading history
stays on-device but answers are generated by a cloud LLM."

Approach:
1. Store each user's reading history as documents in a local SQLite + FAISS index.
2. Deploy Qwen 2.5 14B on the server behind a /draft endpoint that accepts
   a query and token history, returns k=10 draft tokens.
3. On the client, run Qwen 2.5 3B with Contriever retrieval over the user's
   reading history (top 10 documents).
4. Implement the verify loop:

   # Pseudocode for client-side verification
   verified_tokens = []
   while not eos:
       drafts = server.generate_drafts(query, verified_tokens, k=10)
       for token in drafts:
           client_top = client_model.top_token(query, context, verified_tokens)
           p_draft = client_model.prob(token, query, context, verified_tokens)
           p_top = client_model.prob(client_top, query, context, verified_tokens)
           if p_draft / p_top >= 0.05:
               verified_tokens.append(token)
           else:
               verified_tokens.append(client_top)
           if token_is_eos(verified_tokens[-1]):
               break
       server_update = pii_filter(verified_tokens[-k:])
       server.update_history(server_update)

5. Return the assembled response to the user.

Output: A REST API where POST /ask with {"user_id": "abc", "query": "..."}
returns a personalized answer. The server logs show only the query and
PII-filtered token stream -- never the user's reading history.
```

**Example 2: Privacy-Preserving Email Assistant**

```
User: "I want an email draft assistant that knows my writing style and contacts
but doesn't send my email archive to OpenAI."

Approach:
1. Index the user's local email archive with Contriever embeddings in a
   local FAISS store. Each email is one document.
2. Server (cloud LLM) receives only: "Draft a reply to: [subject line +
   sender name]" -- no email body, no archive.
3. Client retrieves 10 most relevant past emails from the local archive.
4. Client model (3-4B) verifies server drafts token-by-token:
   - Accepts tokens matching the user's tone and factual context.
   - Replaces tokens that contradict the user's style or miss context
     only available in the private archive.
5. PII filter catches any names, emails, or phone numbers before tokens
   flow back to the server.

Output: Email drafts that reflect the user's voice and reference relevant
context from past correspondence, with the cloud provider seeing only the
subject line and generic token corrections.
```

**Example 3: On-Device Health Query Personalization**

```
User: "Design a health Q&A system where patient history stays on their phone
but we use a powerful cloud model for medical reasoning."

Approach:
1. Patient medical records stored in an encrypted local DB on the phone.
   Contriever index over record summaries.
2. Server LLM (14B medical-tuned model) answers health queries with no
   patient context -- generates medically accurate but generic drafts.
3. Client model (3B, fine-tuned on medical text) retrieves top 10 relevant
   records and verifies drafts:
   - Accepts tokens for general medical facts.
   - Rejects/replaces tokens that should reference the patient's specific
     conditions, medications, or history.
4. Aggressive PII filtering: names, dates, medication names mapped to
   generic classes before any server communication.
5. Tau set higher (0.1) to force more personalization corrections given
   the high stakes of medical context accuracy.

Output: Medically informed answers personalized to the patient's history,
with zero patient data reaching the cloud. Audit log shows ~15% token
replacement rate due to higher tau.
```

## Best Practices

- **Do:** Use models from the same family for server and client (e.g., both Qwen 2.5 or both Gemma 3). Vocabulary alignment is critical -- the client must be able to score the exact tokens the server produces.
- **Do:** Start with the paper's defaults (k=10, tau=0.05, m=10 retrieved documents) and tune from there. These values are empirically validated across multiple datasets.
- **Do:** Run PII filtering on every token sequence sent from client to server, even though the architecture already limits exposure. Defense in depth matters.
- **Do:** Log accept/reject ratios per query. This is your primary diagnostic signal -- low acceptance means the server and client are misaligned.
- **Avoid:** Sending retrieved profile documents (or summaries of them) to the server. The entire point is that retrieval happens client-side only. Even "anonymized" summaries leak information.
- **Avoid:** Using tau=0 (accept everything) in production. This disables personalization and turns the system into a non-personalized server generator. Use tau >= 0.025.
- **Avoid:** Setting k too high (>20). Beyond k=10, performance saturates and you waste compute on draft tokens that will be rejected. Higher k also increases latency per round.

## Error Handling

| Problem | Symptom | Fix |
|---------|---------|-----|
| Vocabulary mismatch | Client cannot score server tokens; crashes or garbage probabilities | Use same-family models (Qwen+Qwen, Gemma+Gemma) with shared tokenizer |
| Client model too weak | >30% token rejection rate; responses are incoherent | Increase client model size or fine-tune on domain data with LoRA |
| Tau too high | Responses lose fluency; client model dominates output quality | Lower tau toward 0.05; monitor output coherence |
| Tau too low | Responses are generic, not personalized | Raise tau toward 0.1-0.2; verify retrieval is returning relevant documents |
| PII leakage in corrections | Sensitive tokens pass through to server | Strengthen PII regex patterns; add named entity recognition as second pass |
| High latency per round | Network overhead dominates | Increase k to reduce round-trips (up to k=20); consider batching queries |
| Retrieval returns irrelevant docs | Personalization is poor despite corrections | Re-index profile with a better retriever; increase m beyond 10; check query formulation |

## Limitations

- **Same-family model requirement.** Server and client models must share a tokenizer for probability scoring to work. You cannot pair an OpenAI server model with a Llama client model without a token-mapping layer.
- **Client hardware floor.** The client must run a 3-4B parameter model with acceptable latency. This rules out very low-end devices without quantization (GGUF/GPTQ).
- **Not suitable for real-time streaming UIs** in the current design. Each round requires a network hop, so token-level streaming to the user has k-token granularity latency. Acceptable for batch/async but noticeable in chat.
- **Short-form output bias.** The framework is validated on QA tasks (short answers). For long-form generation (essays, reports), the iterative loop may need adaptation -- more rounds mean more latency and more drift between server and client contexts.
- **Privacy is architectural, not cryptographic.** The server still sees the query and a filtered token stream. For adversaries with auxiliary data, even this may reveal information. P^3 is not a substitute for differential privacy or homomorphic encryption when formal guarantees are required.
- **Profile freshness.** The client-side retrieval index must be kept up to date as the user profile evolves. Stale indices produce stale personalization.

## Reference

**Paper:** Salemi, A. & Zamani, H. (2026). *Improving User Privacy in Personalized Generation: Client-Side Retrieval-Augmented Modification of Server-Side Generated Speculations.* SIGIR 2026. [arXiv:2601.17569](https://arxiv.org/abs/2601.17569v1)

**Code:** [github.com/alirezasalemi7/PPP](https://github.com/alirezasalemi7/PPP)

**What to look for:** Algorithm 1 (the full draft-verify-modify pseudocode), Table 2 (hyperparameter sensitivity for k and tau), Section 5.3 (privacy attack results showing linkability and attribute inference leakage), and Figure 2 (prompt templates for server and client models).