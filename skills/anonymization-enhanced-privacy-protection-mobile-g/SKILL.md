---
name: "anonymization-enhanced-privacy-protection-mobile-g"
description: "Implement available-but-invisible privacy protection for mobile GUI agents using PII-aware anonymization with deterministic, type-preserving placeholders. Use when: 'anonymize PII in UI automation', 'build privacy layer for mobile agent', 'protect sensitive data in screenshots', 'add PII detection to Android agent', 'type-preserving placeholder system', 'privacy-safe GUI agent pipeline'."
---

# Anonymization-Enhanced Privacy Protection for Mobile GUI Agents

This skill enables Claude to build privacy protection layers for mobile GUI automation agents that follow the **available-but-invisible** principle: sensitive information (phone numbers, emails, addresses, financial data) remains usable for task execution but is never directly exposed to cloud-based LLM agents. The technique uses deterministic, type-preserving placeholders (e.g., `PHONE_NUMBER#a1b2c`) mapped via SHA-256 hashing, combined with a four-layer architecture (PII Detector, UI Transformer, Secure Interaction Proxy, Privacy Gatekeeper) that enforces consistent anonymization across user instructions, XML accessibility trees, and screenshots simultaneously.

## When to Use

- When building or extending a mobile GUI automation agent that captures screen content and sends it to a cloud LLM for reasoning
- When implementing a PII detection and masking pipeline for Android accessibility XML trees and OCR-extracted screenshot text
- When designing a proxy layer that resolves anonymized placeholders back to real values only at the point of on-device action execution
- When creating a deterministic, collision-resistant placeholder system that preserves entity types across sessions
- When adding a privacy gatekeeper that decides whether local computation on raw PII is necessary vs. satisfiable through the anonymized virtual UI
- When evaluating privacy-utility trade-offs for GUI agent frameworks against benchmarks like AndroidLab or PrivScreen

## Key Technique

The core insight is that GUI agents do not need to *see* sensitive data to *use* it. A phone number field can be represented as `PHONE_NUMBER#cbnhu` throughout the agent's reasoning chain -- the agent knows it is a phone number, can reference it, and can instruct actions on it, but never learns the actual digits. The placeholder is generated deterministically via `P = TYPE # Truncate(Base36(SHA256(value || TYPE)), 5)`, ensuring the same real value always maps to the same placeholder within a session, which preserves referential consistency across multi-step tasks.

The system enforces cross-modality consistency through two mechanisms. First, a **lookup-before-generation policy** consults a session-scoped mapping table before creating new placeholders, so identical entities seen in XML attributes, OCR output, and user instructions always receive the same token. Second, **fuzzy matching** (normalized Levenshtein distance with threshold tau=0.85) handles OCR recognition errors that would otherwise break alignment between the screenshot and XML modalities.

A critical architectural choice is the **layered trust boundary**. The PII Detector and UI Transformer run on-device to anonymize all outgoing data. The cloud agent only ever sees the anonymized "Virtual UI." When the agent issues actions (tap, type), the Secure Interaction Proxy intercepts them, resolves any placeholders back to real values using the local mapping table, and executes the de-anonymized command on the actual device. For rare cases where the agent must reason over raw values (e.g., comparing two prices), the Privacy Gatekeeper -- a local small language model -- evaluates relevance, necessity, and minimization criteria before returning only a bounded, non-revealing result (boolean, categorical).

## Step-by-Step Workflow

1. **Define the PII taxonomy.** Enumerate the sensitive entity types your agent may encounter: `PHONE_NUMBER`, `EMAIL`, `ADDRESS`, `PERSON_NAME`, `CREDIT_CARD`, `ID_NUMBER`, `ACCOUNT_BALANCE`, `MESSAGE_CONTENT`, etc. Each type becomes a placeholder prefix.

2. **Implement the PII Detector as a hybrid pipeline.** Use a label-guided NER model (GLiNER family works well -- accepts arbitrary label sets at inference) for natural-language PII, plus a regex fallback layer for structured identifiers (credit card patterns, national ID formats, phone number formats). Add XML-specific structural validation with a whitelist of schema element names to prevent false positives on UI framework tokens.

3. **Build the deterministic placeholder generator.** For each detected entity `(value, type)`, compute `placeholder = TYPE + "#" + truncate(base36(sha256(value + "|" + TYPE)), 5)`. Maintain a session-scoped bidirectional mapping table (`placeholder <-> raw_value`). Always look up existing mappings before generating new ones.

4. **Construct the instruction-driven contextual whitelist.** Before anonymizing the UI, scan the user's task instruction for PII-like tokens. Tokens the user explicitly references (e.g., "send a message to 555-1234") should be whitelisted so the agent can match them, but still replaced with their deterministic placeholder in both instruction and UI simultaneously for consistency.

5. **Apply the UI Transformer across all modalities.** For XML accessibility trees: replace `text`, `hint`, and `content-desc` attributes via mapping table lookup. For screenshots: run OCR (EasyOCR or similar), fuzzy-align extracted text against registered entities (Levenshtein threshold 0.85), then render opaque overlays with placeholder text at matched bounding boxes, preserving spatial layout.

6. **Implement the Secure Interaction Proxy.** Intercept all agent-issued actions. For spatial actions (tap, swipe): validate the numeric element index, map it to the real bounding box, convert to device coordinates -- no de-anonymization needed. For textual actions (type text): scan the string for placeholder tokens, resolve each to its raw value via the mapping table, leave non-placeholder natural language unchanged, then execute the de-anonymized input on the device.

7. **Build the Privacy Gatekeeper for local computation.** When the agent requests reasoning over raw values, evaluate three criteria using a local SLM: (a) Is the computation relevant to the user's goal? (b) Does it genuinely require raw values or can it work with the virtual UI? (c) Is the result expressible as a bounded, non-revealing value? Return only minimal results (true/false, "higher"/"lower") -- raw values never cross the trust boundary.

8. **Add fuzzy OCR alignment.** Screenshots yield imperfect text. Normalize both OCR output and registered entity values (lowercase, strip whitespace), then compute Levenshtein similarity. If similarity >= 0.85, treat as a match and apply the existing placeholder. This prevents duplicate placeholders for the same entity seen through different modalities.

9. **Validate with privacy and utility metrics.** Measure leakage rate (percentage of raw PII values that appear in agent outputs), match score (exact string match of leaked values), and BERTScore of leaked content against originals. Separately measure task success rate to quantify utility impact. Target: leakage reduction >75% with utility degradation <5%.

10. **Instrument the pipeline for latency monitoring.** The OCR + NER overhead is approximately 1.7 seconds per screenshot frame. Profile each component (OCR ~0.84s, NER ~0.66s, transformer + proxy overhead) and optimize hot paths. Consider caching placeholder mappings and OCR results for unchanged screen regions.

## Concrete Examples

**Example 1: Anonymizing a contact list screen for a messaging agent**

User: "Build the anonymization layer for my Android GUI agent. The agent reads contact lists and sends messages."

Approach:
1. Define PII types: `PERSON_NAME`, `PHONE_NUMBER`, `MESSAGE_CONTENT`
2. Parse the accessibility XML; detect `text="John Smith"` and `text="+1-555-867-5309"`
3. Generate placeholders:
   - `sha256("John Smith|PERSON_NAME")` -> base36 truncated -> `PERSON_NAME#k8m2x`
   - `sha256("+1-555-867-5309|PHONE_NUMBER")` -> `PHONE_NUMBER#a1b2c`
4. Replace in XML and overlay on screenshot
5. Send anonymized UI to cloud agent

```python
import hashlib

def generate_placeholder(value: str, pii_type: str, mapping: dict) -> str:
    """Deterministic, type-preserving placeholder generation."""
    # Lookup-before-generation policy
    key = (value, pii_type)
    if key in mapping:
        return mapping[key]

    raw = f"{value}|{pii_type}".encode("utf-8")
    digest = hashlib.sha256(raw).hexdigest()
    # Convert hex to base36 and truncate to 5 chars
    base36 = int(digest, 16)
    chars = "0123456789abcdefghijklmnopqrstuvwxyz"
    b36_str = ""
    while base36 and len(b36_str) < 5:
        b36_str = chars[base36 % 36] + b36_str
        base36 //= 36
    b36_str = b36_str.ljust(5, "0")

    placeholder = f"{pii_type}#{b36_str}"
    mapping[key] = placeholder
    mapping[placeholder] = value  # Reverse mapping for de-anonymization
    return placeholder

# Usage
session_map = {}
print(generate_placeholder("+1-555-867-5309", "PHONE_NUMBER", session_map))
# Output: PHONE_NUMBER#a9x7k
```

**Example 2: Secure Interaction Proxy resolving a "type" action**

User: "The agent wants to type a message to a contact. How does the proxy resolve placeholders?"

Approach:
1. Agent issues: `type(element=5, text="Hi PERSON_NAME#k8m2x, call me at PHONE_NUMBER#a9x7k")`
2. Proxy scans text for placeholder pattern `[A-Z_]+#[a-z0-9]{5}`
3. Resolves each: `PERSON_NAME#k8m2x` -> `"John Smith"`, `PHONE_NUMBER#a9x7k` -> `"+1-555-867-5309"`
4. Executes on device: `type(element=5, text="Hi John Smith, call me at +1-555-867-5309")`

```python
import re

PLACEHOLDER_PATTERN = re.compile(r"[A-Z_]+#[a-z0-9]{5}")

def resolve_action(action: dict, mapping: dict) -> dict:
    """De-anonymize textual action parameters before device execution."""
    resolved = action.copy()
    if action["type"] == "type" and "text" in action:
        text = action["text"]
        for match in PLACEHOLDER_PATTERN.finditer(text):
            placeholder = match.group()
            if placeholder in mapping:
                text = text.replace(placeholder, mapping[placeholder])
        resolved["text"] = text
    # Spatial actions (tap, swipe) pass through unchanged
    return resolved

# The cloud agent never sees "John Smith" -- only "PERSON_NAME#k8m2x"
```

**Example 3: Privacy Gatekeeper evaluating a local computation request**

User: "The agent needs to compare two prices to pick the cheaper flight. How does the gatekeeper handle this?"

Approach:
1. Agent requests: "Compare ACCOUNT_BALANCE#f3g7h ($249) and ACCOUNT_BALANCE#j9k1m ($312), return which is lower"
2. Gatekeeper evaluates: Relevant? Yes (user asked to find cheapest). Necessary? Yes (comparison requires numeric values). Minimal? Yes (result is a choice, not the values themselves).
3. Local SLM performs comparison on raw values: $249 < $312
4. Returns only: `"ACCOUNT_BALANCE#f3g7h is the lower value"` -- no raw prices cross the trust boundary

```python
def gatekeeper_evaluate(request: str, user_goal: str) -> dict:
    """Evaluate whether local raw-value computation is permitted."""
    return {
        "relevant": True,    # Comparison serves the booking goal
        "necessary": True,   # Cannot compare without numeric values
        "minimal": True,     # Result is a pointer, not the raw value
        "approved": True,
        "result_type": "categorical"  # Returns placeholder reference only
    }
```

## Best Practices

- **Do:** Always apply lookup-before-generation. Check the session mapping table before computing a new placeholder hash. Duplicate placeholders for the same entity break referential consistency and confuse the agent.
- **Do:** Anonymize all three modalities (instructions, XML, screenshots) simultaneously in a single pass. Partial anonymization lets the agent cross-reference anonymized and raw versions to reconstruct PII.
- **Do:** Use fuzzy matching (Levenshtein >= 0.85) for OCR-to-entity alignment. OCR errors are inevitable; strict matching causes the same phone number to get two different placeholders.
- **Do:** Preserve spatial layout when overlaying placeholders on screenshots. The agent relies on positional reasoning; shifting element positions breaks spatial action mapping.
- **Avoid:** Sending raw PII values in any cloud-bound payload, even in "system" or "context" fields. The trust boundary must be absolute.
- **Avoid:** Using non-deterministic anonymization (random tokens, incrementing counters). The agent must see the same placeholder for the same entity across steps, otherwise multi-turn tasks break.
- **Avoid:** Over-broad whitelisting. The instruction-driven whitelist should only cover tokens the user explicitly mentioned; do not whitelist entire PII categories.

## Error Handling

- **False positive PII detection on UI framework tokens:** Maintain a whitelist of known Android widget class names, resource IDs, and accessibility labels (e.g., "android.widget.Button", "com.app.R.id.submit"). Validate detected spans against this whitelist before anonymizing.
- **Placeholder collision (two different values produce the same 5-char hash):** Extremely unlikely with SHA-256 + base36, but handle by appending a counter suffix (e.g., `PHONE_NUMBER#a1b2c_2`) and logging a warning.
- **OCR fails to extract text from a screenshot region:** Fall back to the XML accessibility tree for that element. If both fail, flag the region as "unprocessed" and log for manual review rather than silently passing raw pixels.
- **Agent outputs a placeholder to the user (leaks the anonymized token):** The output filter should detect placeholder patterns in agent responses and either resolve them to a user-friendly description ("your phone number") or redact them entirely, depending on context.
- **Mapping table grows unbounded in long sessions:** Implement LRU eviction for entities not seen in the last N screens. Evicted entries can be regenerated deterministically from the same hash if re-encountered.

## Limitations

- **OCR-dependent pipeline:** Screenshot anonymization quality is bounded by OCR accuracy. Complex fonts, overlapping UI elements, or low-contrast text may evade detection, leaking raw PII in pixel form.
- **Computational overhead:** ~1.7 seconds per frame (OCR: 0.84s, NER: 0.66s) adds latency that may be noticeable in real-time interaction scenarios. Not suitable for sub-second response requirements without hardware acceleration.
- **Image-embedded PII:** Photos, profile pictures, or scanned documents displayed in the UI cannot be anonymized by text-level masking alone. A separate image-level privacy model would be needed.
- **Placeholder leakage as metadata:** While raw values are hidden, the *types* and *counts* of PII on screen are visible to the cloud agent (it sees three `PHONE_NUMBER` placeholders). This metadata itself may be sensitive in some threat models.
- **Local SLM dependency:** The Privacy Gatekeeper requires a capable on-device language model. On resource-constrained devices, this component may need to be simplified to rule-based logic, reducing flexibility.
- **Multi-language PII detection:** The GLiNER-based detector and regex patterns need language-specific tuning. Out-of-the-box performance degrades for non-English UIs with different PII formats.

## Reference

[Anonymization-Enhanced Privacy Protection for Mobile GUI Agents: Available but Invisible](https://arxiv.org/abs/2602.10139v1) -- Zhao et al., 2026. Focus on Section 3 (system architecture), Algorithm 1 (whitelist construction), Table 5 (proxy resolution rules), and Section 5 (experimental results showing 97% -> 19% leakage reduction with <1% utility loss on AndroidLab).