---
name: "fimi-domain-specific-indian-finance"
description: "Build domain-specialized AI agents for Indian financial systems (UPI, NPCI, RBI) using multi-stage training pipeline patterns and tool-driven multi-turn conversation design from the FiMI paper. Trigger phrases: 'build a UPI support agent', 'Indian finance chatbot', 'payment dispute resolution bot', 'mandate management workflow', 'domain-specific fintech tool-calling', 'Indian digital payments assistant'"
---

# FiMI: Domain-Specific Indian Finance Agent Framework

This skill teaches Claude to design and implement AI-powered agents for the Indian financial ecosystem — specifically UPI (Unified Payments Interface), NPCI workflows, and RBI-regulated digital payment systems. It applies the multi-stage domain adaptation and tool-driven conversation architecture from the FiMI paper to build agents that handle transaction disputes, mandate lifecycle management, and multilingual (English/Hindi/Hinglish) financial support through structured ReAct reasoning with explicit tool-calling.

## When to Use

- When the user asks to build a customer support agent for UPI or Indian digital payments
- When designing tool-calling workflows for transaction dispute resolution (chargebacks, failed transfers, pending status)
- When implementing mandate lifecycle management (view, pause, unpause, revoke automatic payments via UMN)
- When building multilingual fintech chatbots that must handle English, Hindi, and Hinglish queries
- When the user needs to structure multi-turn financial conversations with ReAct-style reasoning and tool invocation
- When adapting a general-purpose LLM to Indian finance domain with proper data curation strategy
- When implementing RRN-based transaction lookup or UMN-based mandate query systems
- When designing synthetic training data pipelines for financial domain specialization

## Key Technique

FiMI's core insight is a **three-stage domain adaptation pipeline** that progressively specializes a general model (Mistral Small 24B) for Indian finance. Stage 1 is Continuous Pre-Training (CPT) on 68 billion curated tokens across three phases: C1 builds general foundations, C2 introduces domain-heavy financial and multilingual data, and C3 consolidates domain knowledge. The data mix is critical — general English text (1.8B tokens), mathematics (151M), code (135M), Hindi/multilingual (36M), and Indian finance-specific content (700M tokens drawn from five topic clusters: economic systems, investment schemes, insurance, news, and payment ecosystems). Stage 2 is Instruction Fine-Tuning in two substages: reasoning emphasis (19B tokens of step-by-step problem solving) followed by behavioral alignment (435M tokens). Stage 3 is Domain-Specific SFT on synthetic multi-turn conversations modeled on real UPI Help workflows.

The **tool-calling architecture** is what makes FiMI practically useful. Conversations follow a ReAct (Reason + Act) pattern where the agent alternates between logical reasoning steps and tool invocations. The system uses structured special tokens — `[AVAILABLE_TOOLS]`, `[TOOL_CALLS]`, and `[TOOL_RESULTS]` — to demarcate tool boundaries, supporting both XML-based and Python-based function traces. This enables dual-mode calling where the agent can invoke transaction retrieval by RRN (Retrieval Reference Number), mandate queries by UMN (Unique Mandate Number), dispute lifecycle management, and error classification tools within a single multi-turn conversation.

The **synthetic data generation pipeline** for domain SFT is particularly replicable: derive user personas from real query logs, use prompt engineering to generate realistic multi-turn interactions, validate with subject matter experts across four quality pillars (domain accuracy, conversational naturalness, tool-use correctness, regulatory compliance), and iterate with multiple model candidates (including on-premises models like Gemma 3 27B for data privacy).

## Step-by-Step Workflow

1. **Define the financial domain scope and tool inventory.** Enumerate the specific Indian financial workflows to support (e.g., UPI transaction lookup, mandate management, dispute filing). For each workflow, define the tools the agent can call — including input parameters (RRN, UMN, transaction ID, date range), expected outputs, and error states.

2. **Design the tool schema with structured delimiters.** Define each tool as a JSON function spec wrapped in `[AVAILABLE_TOOLS]` blocks. Include parameter types, required fields, and descriptions. Example tools: `get_transaction_by_rrn(rrn: str)`, `get_mandate_by_umn(umn: str)`, `file_dispute(txn_id: str, reason: str)`, `update_mandate_status(umn: str, action: "pause"|"unpause"|"revoke")`.

3. **Structure the system prompt with Indian finance domain context.** Embed domain knowledge directly: UPI transaction lifecycle states (initiated, pending, completed, failed, deemed), NPCI dispute TAT (Turn Around Time) rules, RBI consumer protection guidelines, and common error codes (U16 for risk threshold exceeded, U28 for unable to decrypt, etc.).

4. **Implement ReAct-style reasoning in the conversation template.** Each agent turn must follow: (a) Thought — reason about the user's issue using domain knowledge, (b) Action — select and invoke the appropriate tool with `[TOOL_CALLS]`, (c) Observation — parse the `[TOOL_RESULTS]` response, (d) Response — synthesize findings into a user-facing answer. Chain multiple reason-act cycles within a single turn when the workflow requires sequential tool calls.

5. **Build multi-turn conversation state management.** Track conversation context across turns: the user's original complaint, tools already invoked, results received, and resolution status. Use this to avoid redundant tool calls and to maintain coherent dispute escalation paths.

6. **Generate synthetic training conversations from persona templates.** Create user personas representing common archetypes (first-time UPI user confused by failed transaction, merchant disputing a chargeback, user wanting to revoke a mandate). For each persona, generate 5-10 turn conversations that exercise the full tool-calling pipeline, including edge cases like timeouts and partial failures.

7. **Implement multilingual handling for Hindi and Hinglish inputs.** Detect the user's language from their query. Respond in the same language. Maintain consistent financial terminology across languages — e.g., "transaction" / "lenden" / "transaction ho gaya" should all map to the same intent and tool call.

8. **Add domain-specific error classification and recovery.** When tool calls return errors, classify them: transient (retry), user-correctable (ask for correct RRN format), or system-level (escalate). Map UPI error codes to human-readable explanations in the user's language.

9. **Validate against the four quality pillars.** Before deployment, evaluate each conversation flow for: (a) domain accuracy — correct UPI/NPCI terminology and process flows, (b) conversational naturalness — flows like a real support interaction, (c) tool-use correctness — right tools called with right parameters in right order, (d) regulatory compliance — advice adheres to RBI guidelines.

10. **Implement guardrails for financial advice boundaries.** The agent must never provide investment advice, predict market outcomes, or share other users' transaction data. Restrict tool access to the authenticated user's own records. Include explicit disclaimers when the query borders on regulated financial advice.

## Concrete Examples

**Example 1: UPI Transaction Dispute Resolution Agent**

User: "My payment of Rs 5000 to PhonePe failed but money was debited. RRN is 432198765012."

Approach:
1. Parse the user's complaint: failed UPI transaction with debit, RRN provided
2. Invoke `get_transaction_by_rrn` with the RRN to retrieve transaction status
3. Analyze the response — check if status is "deemed" (auto-reversal eligible) or requires dispute
4. If pending beyond TAT, invoke `file_dispute` with appropriate reason code
5. Explain the resolution timeline per NPCI guidelines

Output:
```
System Prompt:
You are UPI Help, a support agent for Indian digital payments. You have access
to the following tools:

[AVAILABLE_TOOLS]
{
  "name": "get_transaction_by_rrn",
  "description": "Retrieve UPI transaction details using Retrieval Reference Number",
  "parameters": {
    "rrn": {"type": "string", "description": "12-digit RRN from bank SMS"},
    "user_id": {"type": "string", "description": "Authenticated user identifier"}
  }
}
{
  "name": "file_dispute",
  "description": "File a transaction dispute with NPCI",
  "parameters": {
    "txn_id": {"type": "string"},
    "reason": {"type": "string", "enum": ["amount_debited_not_credited", "duplicate_transaction", "wrong_beneficiary"]},
    "amount": {"type": "number"}
  }
}
[/AVAILABLE_TOOLS]

Agent Turn 1:
Thought: The user reports a failed transaction with debit. I need to first verify
the transaction status using the provided RRN before determining next steps.

[TOOL_CALLS]
{"name": "get_transaction_by_rrn", "arguments": {"rrn": "432198765012", "user_id": "usr_authenticated"}}
[/TOOL_CALLS]

[TOOL_RESULTS]
{"txn_id": "TXN9087654321", "status": "DEEMED", "amount": 5000, "timestamp": "2026-02-05T14:32:00Z", "payer_bank": "SBI", "payee_vpa": "merchant@ybl", "deemed_at": "2026-02-05T14:37:00Z"}
[/TOOL_RESULTS]

Response: I've checked your transaction (RRN: 432198765012). The status is
"Deemed" — this means the transaction was not completed successfully and NPCI
has flagged it for automatic reversal. Per NPCI guidelines, the refund of
Rs 5,000 should be credited to your SBI account within 5 business days from
Feb 5, 2026. If you don't receive the refund by Feb 12, I can file a formal
dispute on your behalf. Would you like me to do that now or would you prefer
to wait?
```

**Example 2: Mandate Lifecycle Management**

User: "Mera Netflix ka auto-payment band karna hai. UMN hai NPCI0001234567."

Approach:
1. Detect Hinglish input — user wants to stop Netflix auto-payment
2. Query mandate details using the UMN
3. Present options: pause (temporary) vs revoke (permanent)
4. Execute the user's chosen action
5. Respond in Hinglish to match user's language

Output:
```
Agent Turn 1:
Thought: User is speaking Hinglish and wants to stop a recurring mandate.
I should query the mandate first, then offer pause vs revoke options.

[TOOL_CALLS]
{"name": "get_mandate_by_umn", "arguments": {"umn": "NPCI0001234567", "user_id": "usr_authenticated"}}
[/TOOL_CALLS]

[TOOL_RESULTS]
{"umn": "NPCI0001234567", "merchant": "Netflix India", "amount": 649, "frequency": "monthly", "status": "ACTIVE", "next_debit": "2026-02-15", "created": "2025-06-01"}
[/TOOL_RESULTS]

Response: Aapka Netflix India ka mandate mil gaya — Rs 649 monthly, next debit
15 Feb ko hoga. Aapke paas do options hain:

1. **Pause** — temporarily rok denge, baad mein wapas start kar sakte ho
2. **Revoke** — permanently band ho jayega, dubara set up karna padega

Kya karna chahte ho?

Agent Turn 2 (after user says "revoke karo"):
[TOOL_CALLS]
{"name": "update_mandate_status", "arguments": {"umn": "NPCI0001234567", "action": "revoke"}}
[/TOOL_CALLS]

[TOOL_RESULTS]
{"status": "REVOKED", "effective_from": "2026-02-13T10:15:00Z"}
[/TOOL_RESULTS]

Response: Done! Aapka Netflix mandate revoke ho gaya hai. Ab 15 Feb ko koi
auto-debit nahi hoga. Agar future mein Netflix wapas subscribe karna ho toh
naya mandate set up karna padega app se. Kuch aur help chahiye?
```

**Example 3: Designing the Tool Schema for a New Payment Workflow**

User: "I need to build a support agent that handles UPI Lite balance inquiries and top-ups."

Approach:
1. Define tool inventory for UPI Lite operations
2. Structure the conversation flow with domain-specific states
3. Include error handling for common UPI Lite issues

Output:
```python
# Tool definitions following FiMI's structured schema pattern
UPI_LITE_TOOLS = [
    {
        "name": "get_upi_lite_balance",
        "description": "Check UPI Lite on-device wallet balance",
        "parameters": {
            "user_id": {"type": "string", "required": True},
            "vpa": {"type": "string", "description": "User's VPA linked to UPI Lite"}
        },
        "returns": {"balance": "number", "max_limit": "number", "last_topup": "datetime"}
    },
    {
        "name": "topup_upi_lite",
        "description": "Add funds to UPI Lite wallet from linked bank account",
        "parameters": {
            "user_id": {"type": "string", "required": True},
            "amount": {"type": "number", "min": 1, "max": 2000,
                       "description": "Amount in INR, max Rs 2000 per top-up"},
            "source_account": {"type": "string"}
        },
        "returns": {"new_balance": "number", "txn_id": "string", "status": "string"}
    },
    {
        "name": "get_upi_lite_txn_history",
        "description": "Retrieve recent UPI Lite transactions",
        "parameters": {
            "user_id": {"type": "string", "required": True},
            "days": {"type": "integer", "default": 7, "max": 30}
        },
        "returns": {"transactions": "array", "total_spent": "number"}
    }
]

# System prompt embedding domain constraints
SYSTEM_PROMPT = """You are a UPI Lite support agent. Key domain rules:
- UPI Lite max balance: Rs 2,000 (RBI mandate)
- Per-transaction limit: Rs 500
- No PIN required for UPI Lite transactions (offline-capable)
- Top-ups require full UPI PIN authentication
- UPI Lite is on-device, not server-side — balance discrepancies
  may occur if app data is cleared
Follow ReAct pattern: Thought -> Tool Call -> Observation -> Response"""
```

## Best Practices

- **Do:** Always verify the user's identity context before invoking any tool that accesses financial records. FiMI's tools operate on authenticated user IDs — never assume or infer user identity.
- **Do:** Map Indian financial jargon accurately — "deemed" means auto-reversal flagged, "TAT" is Turn Around Time per NPCI SLAs, "VPA" is Virtual Payment Address. Incorrect terminology erodes trust.
- **Do:** Support code-switching naturally. Indian users frequently mix Hindi and English mid-sentence ("mera transaction fail ho gaya"). Parse intent from mixed-language input without forcing the user into a single language.
- **Do:** Include NPCI/RBI regulatory timelines in responses (e.g., "refund within 5 business days per NPCI circular"). Users need authoritative timeframes.
- **Avoid:** Never expose raw tool responses to users. Always translate technical statuses (DEEMED, PENDING_CREDIT, RRN_NOT_FOUND) into plain-language explanations.
- **Avoid:** Do not chain more than 3 tool calls without an intermediate user-facing response. Financial conversations require transparency — the user should understand what the agent is doing and why.

## Error Handling

| Error Scenario | Handling Strategy |
|---|---|
| Invalid RRN format (not 12 digits) | Ask user to re-check their bank SMS for the correct RRN. Provide format hint: "RRN is a 12-digit number starting with the date (e.g., 6035...)" |
| Transaction not found | Verify the RRN, check if the transaction is older than 90 days (outside lookup window), or suggest checking with the bank directly |
| Mandate UMN not found | Confirm the UMN format, check if the mandate was already revoked, or if it belongs to a different VPA |
| Tool timeout / service unavailable | Inform the user of temporary service issue, provide NPCI helpline (1800-120-1740) as fallback, offer to retry |
| Ambiguous user intent | Ask a clarifying question before invoking any tool — "Do you want to pause (temporary) or revoke (permanent) the mandate?" |
| Regulatory boundary crossed | If user asks for advice on investments or loans, redirect: "I can help with UPI transactions and mandates. For investment advice, please consult a SEBI-registered advisor." |

## Limitations

- **India-specific only.** The tool-calling schemas, regulatory references (RBI, NPCI), and domain terminology are specific to India's UPI ecosystem. Do not apply these patterns to SWIFT, SEPA, ACH, or other payment networks without significant adaptation.
- **No real-time data.** The agent framework does not connect to live NPCI/bank systems. In production, tool implementations must integrate with actual banking APIs behind proper authentication and compliance layers.
- **Language coverage gaps.** FiMI covers English, Hindi, and Hinglish. Regional languages (Tamil, Telugu, Kannada, Bengali, etc.) are not covered and would require additional multilingual adaptation.
- **Not for financial advice.** The framework handles operational support (transaction status, disputes, mandates) — not investment advice, credit scoring, or loan processing, which have separate regulatory requirements.
- **Synthetic data ceiling.** The domain SFT relies on synthetic conversations validated by SMEs. Edge cases not anticipated during data generation (novel fraud patterns, new UPI features) will require ongoing data pipeline updates.

## Reference

**Paper:** [FiMI: A Domain-Specific Language Model for Indian Finance Ecosystem](https://arxiv.org/abs/2602.05794v1) — Focus on Section 3 (multi-stage training pipeline), Section 4 (tool-calling architecture and ReAct conversation design), and Section 5 (synthetic data generation for domain SFT). The key takeaway is that structured tool-calling with domain-specific schemas yields 87% improvement over generic instruction-tuned models on financial workflows.