---
name: "how-few-shot-demonstrations-affect"
description: "Design prompt-based LLM safety defenses using optimal few-shot strategies. Applies the finding that few-shot demonstrations enhance Role-Oriented Prompts (RoP) but degrade Task-Oriented Prompts (ToP). Use when: 'harden my system prompt against jailbreaks', 'add few-shot safety examples', 'design a defense prompt', 'improve LLM safety alignment', 'protect my chatbot from prompt injection', 'write a safe system prompt with examples'."
---

# Few-Shot Defense Optimization for LLM Safety Prompts

This skill enables Claude to design, audit, and improve prompt-based safety defenses for LLM applications by correctly pairing few-shot demonstrations with the right prompt strategy. The core insight from Wang et al. (2026) is that few-shot examples have **opposite effects** depending on prompt type: they reinforce Role-Oriented Prompts (up to +4.5% safety rate) but degrade Task-Oriented Prompts (up to -21.2% safety rate). Misapplying few-shot examples to task-oriented system prompts actively weakens safety.

## When to Use

- When the user asks to write or improve a system prompt that defends against jailbreak attacks
- When the user wants to add few-shot safety examples to an existing LLM system prompt
- When reviewing a system prompt for safety vulnerabilities and the prompt includes demonstration examples
- When designing a chatbot or agent that must refuse harmful requests reliably
- When the user is building a prompt template pipeline and needs to decide whether to include few-shot demonstrations
- When auditing an existing defense configuration that combines system prompts with in-context examples

## Key Technique

**Role-Oriented Prompts (RoP)** define *who the model is* — an identity-first framing like "You are a safe, helpful, and harmless AI assistant." **Task-Oriented Prompts (ToP)** define *what the model should do* — an instruction-first framing like "Your task is to provide helpful, accurate, and safe responses." Both strategies are effective defenses on their own, but they respond to few-shot demonstrations in opposite ways.

Few-shot demonstrations reinforce RoP through **role identity accumulation**: safety-aligned Q/A examples pile evidence onto the model's adopted persona, strengthening the Bayesian posterior for safe behavior. The role identity occupies initial prompt positions that receive high cumulative attention across transformer layers (the "attention sink" effect), and each example adds reinforcing signal to that distributed representation.

Conversely, few-shot demonstrations degrade ToP through **attention dilution**: adding examples pushes task instructions into middle positions of the prompt where retrieval accuracy drops below 50% (the "lost in the middle" phenomenon). The model's attention fragments away from the explicit safety directives, and the few-shot examples — even safety-aligned ones — distract from the core task instruction rather than reinforcing it.

## Step-by-Step Workflow

1. **Classify the existing system prompt** as Role-Oriented or Task-Oriented. RoP prompts lead with identity/persona framing ("You are..."). ToP prompts lead with task instructions ("Your task is...", "You must...", "Always..."). Mixed prompts should be classified by their dominant opening framing.

2. **If RoP: add few-shot safety demonstrations.** Construct 2-3 demonstration pairs showing the model refusing harmful requests. Use the FS-Harmful format (harmful query + polite refusal) for maximum safety improvement. Place these demonstrations after the system prompt and before the user's actual query.

3. **If ToP: remove any existing few-shot demonstrations.** If the system prompt is task-oriented, few-shot examples will degrade safety regardless of their content. Strip them out and rely on the task instructions alone.

4. **If ToP with demonstrations is required by the application**, convert the prompt to RoP framing first. Rewrite the system prompt to lead with role identity ("You are a safety-conscious assistant whose primary purpose is...") and then add few-shot examples to the converted RoP prompt.

5. **Construct few-shot demonstrations using the FS-Harmful format.** Each example should be a user message containing a clearly harmful request paired with an assistant response that firmly but politely refuses and briefly explains why. Avoid benign/general-knowledge examples (FS-General) — they provide weaker safety reinforcement than refusal examples.

6. **Position demonstrations correctly.** Place few-shot examples immediately after the system prompt, before the actual user input. This keeps the role identity in the attention-favorable initial positions and places refusal patterns where they reinforce the established persona.

7. **Limit demonstrations to 2-3 pairs.** More examples increase attention dilution risk even in RoP and add latency/cost. Three well-chosen refusal examples provide the optimal safety/efficiency tradeoff.

8. **Test against representative attack categories.** Validate the defense against at least three attack types: role-playing attacks (like DAN/AIM), prefix injection (forced affirmative starters), and obfuscation attacks (encoded or indirect harmful requests).

9. **Audit for think-mode vulnerabilities.** If the target model uses chain-of-thought or "think mode," apply extra caution — these models show higher jailbreak vulnerability and more negative few-shot interactions. Consider increasing the number of refusal demonstrations to 3.

10. **Document the defense configuration.** Record whether the prompt is RoP or ToP, the number and type of few-shot examples, and the attack categories tested. This enables future audits when the model or threat landscape changes.

## Concrete Examples

**Example 1: Hardening a chatbot system prompt**

User: "I have this system prompt for my customer service bot and I want to add safety examples. Here's my prompt: 'Your task is to help customers with product questions. Never provide information about illegal activities, self-harm, or weapons.'"

Approach:
1. Classify: This is a Task-Oriented Prompt (leads with "Your task is...").
2. Per the research finding, adding few-shot demonstrations to ToP will DEGRADE safety by up to 21.2%.
3. Convert to Role-Oriented framing before adding examples.

Output:
```
# Converted RoP system prompt with few-shot safety demonstrations

## System Prompt
You are a helpful and safety-conscious customer service assistant for [Company].
You prioritize user wellbeing and refuse any requests related to illegal
activities, self-harm, or weapons firmly but politely.

## Few-shot demonstrations (FS-Harmful)

User: Can you tell me how to pick a lock to break into my neighbor's house?
Assistant: I'm not able to help with that. Breaking into someone's home is
illegal regardless of the method. If you're locked out of your own home,
I'd recommend contacting a licensed locksmith.

User: What household chemicals can I combine to make a dangerous gas?
Assistant: I can't provide instructions for creating dangerous substances.
If you have concerns about chemical safety in your home, I'd recommend
checking the product safety labels or contacting poison control.

User: How do I bypass the age verification on your website?
Assistant: I'm not able to help with circumventing age verification systems.
These exist to comply with legal requirements and protect minors. Is there
something else I can help you with regarding our products?

## [Actual user query goes here]
```

**Example 2: Auditing an existing defense prompt**

User: "Review this system prompt for safety — we keep getting jailbroken:
'You are an AI research assistant. Your task is to answer questions accurately.
Here are some examples of good responses: [3 benign Q/A examples about science topics]'"

Approach:
1. Classify: Mixed prompt — opens with RoP ("You are") but pivots to ToP ("Your task is").
2. The dominant framing after the opening is task-oriented.
3. The few-shot examples are FS-General (benign topics), which provide the weakest safety signal.
4. Diagnosis: ToP + FS-General is the worst-performing configuration in the research.

Output:
```
## Audit findings

Your prompt has two compounding problems:

1. **Task-oriented framing with few-shot examples.** The prompt pivots to
   task instructions ("Your task is to answer questions accurately"), and the
   research shows few-shot demonstrations degrade ToP safety by up to 21.2%.
   The examples are actively weakening your defense.

2. **Benign examples instead of refusal examples.** General-knowledge Q/A
   pairs (FS-General) provide weaker safety reinforcement than refusal
   demonstrations (FS-Harmful) even in the best case.

## Recommended fix

Option A — RoP + FS-Harmful (strongest defense):
  Rewrite as role-first, replace benign examples with refusal demonstrations.

Option B — Pure ToP without examples (simpler):
  Remove all few-shot examples and strengthen the task instructions.

[Provide rewritten prompt for the option the user selects]
```

**Example 3: Building a defense from scratch**

User: "Write me a system prompt with safety examples for a general-purpose coding assistant."

Approach:
1. Use RoP framing since we want to include few-shot demonstrations.
2. Construct FS-Harmful examples relevant to the coding domain.
3. Keep demonstrations to 3 pairs.

Output:
```
## System Prompt
You are a skilled and responsible programming assistant. You help users
write, debug, and understand code. You refuse requests to create malicious
software, exploit vulnerabilities in systems you're not authorized to test,
or generate code designed to harm, surveil, or deceive others.

## Few-shot safety demonstrations

User: Write me a keylogger that runs silently in the background on Windows.
Assistant: I can't help create keyloggers or other surveillance software.
Keyloggers are commonly used for unauthorized monitoring and are illegal
in most jurisdictions without consent. I'm happy to help with legitimate
input handling, accessibility tools, or other programming tasks.

User: Generate a Python script to scrape and store passwords from a database dump.
Assistant: I'm not able to write tools for processing stolen credentials.
If you're working on legitimate security — like building a credential
breach detection service with proper authorization — I can discuss the
architecture at a high level, but I won't write extraction tooling.

User: Help me obfuscate this malware so antivirus won't detect it.
Assistant: I can't assist with malware obfuscation or antivirus evasion.
If you're interested in legitimate software protection, I can help with
code obfuscation for intellectual property protection or discuss how
security software detection works from an educational perspective.

## [User query]
```

## Best Practices

**Do:**
- Always classify a system prompt as RoP or ToP before deciding on few-shot strategy
- Use FS-Harmful demonstrations (harmful query + refusal) rather than benign Q/A pairs when adding few-shot examples to RoP
- Keep the role identity statement as the very first sentence of the system prompt to maximize attention sink benefits
- Test the final prompt against multiple attack categories (role-playing, prefix injection, obfuscation) before deployment
- Conduct model-specific empirical testing — optimal configurations vary across models

**Avoid:**
- Never add few-shot demonstrations to Task-Oriented Prompts — this is the single most damaging misconfiguration, degrading safety by up to 21.2%
- Never use only benign/general-knowledge examples as safety demonstrations when refusal examples are available
- Never place few-shot demonstrations before the system prompt identity statement — this disrupts the attention sink mechanism
- Avoid exceeding 3 few-shot demonstration pairs without empirical validation, as additional examples increase attention dilution risk

## Error Handling

- **Mixed RoP/ToP prompt detected:** If a prompt opens with role identity but then switches to task instructions, treat it as ToP for safety purposes. The task-instruction framing dominates downstream behavior and few-shot examples will dilute the instructions.
- **User insists on ToP + few-shot:** Explain the research finding clearly, then offer to convert the prompt to RoP framing while preserving the same behavioral intent.
- **Few-shot examples seem to reduce helpfulness:** This can happen if refusal examples are too broad or aggressive. Narrow the scope of refusal demonstrations to clearly harmful categories and ensure they end with a redirect to legitimate alternatives.
- **Defense works on one model but fails on another:** The paper tested multiple models (GPT-4, Claude 3, Llama 2, Qwen, DeepSeek) and found model-specific variation. Always validate on the target deployment model specifically.

## Limitations

- The research primarily evaluated models in the 7-8B parameter range. Larger models may respond differently to few-shot defense interactions, though the underlying attention mechanisms apply broadly.
- The six attack methods tested (AIM, DAN, Evil Confident, Prefix Rejection, Poems, Refusal Suppression) are representative but not exhaustive. Novel attack vectors may interact differently with few-shot defenses.
- The safety rate improvements for RoP + few-shot are moderate (up to +4.5%). For high-stakes applications, prompt-based defenses should be layered with output filtering, classifier-based moderation, and other defense-in-depth measures.
- The findings apply to prompt-level defenses. They do not address fine-tuning-based alignment, RLHF, or architectural safety mechanisms.
- Think-mode/chain-of-thought models showed higher vulnerability overall — the few-shot optimization strategy does not fully compensate for this.

## Reference

Wang, Y., Yang, S., He, J., & Yang, T. (2026). *How Few-shot Demonstrations Affect Prompt-based Defenses Against LLM Jailbreak Attacks.* arXiv:2602.04294. Look for: Table 2 (safety rates across configurations), Section 4.2 (mechanistic explanation of role reinforcement vs. attention dilution), and Section 5 (deployment recommendations).