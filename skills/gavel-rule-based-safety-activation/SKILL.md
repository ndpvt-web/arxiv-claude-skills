---
name: "gavel-rule-based-safety-activation"
description: "Implement GAVEL-style rule-based activation safety monitoring for LLMs using Cognitive Elements (CEs) and Boolean predicate rules. Use when: 'set up activation monitoring rules', 'define safety rules for my LLM', 'create cognitive element detectors', 'build rule-based content safety', 'configure GAVEL activation guards', 'compose CE safety predicates'."
---

# GAVEL: Rule-Based Activation Safety Monitoring

This skill enables Claude to help practitioners implement the GAVEL framework for LLM safety monitoring. GAVEL (from "Towards Rule-Based Safety Through Activation Monitoring," ICLR 2026) replaces monolithic safety classifiers with composable, interpretable rules built from Cognitive Elements -- fine-grained behavioral detectors extracted from model activations. Instead of retraining a safety model every time policy changes, you define Boolean predicate rules over CEs like `creating_content AND (click OR personal_information)` and deploy them instantly.

## When to Use

- When the user wants to build an activation-monitoring safety layer for a deployed LLM
- When the user needs to define domain-specific content policies as composable logical rules
- When the user asks to detect specific multi-factor threat patterns (phishing, scams, social engineering) in LLM outputs
- When the user wants to update safety rules without retraining the underlying model or classifier
- When the user needs an auditable, interpretable safety system where violations explain which behavioral factors fired
- When the user is building a rule-sharing ecosystem for AI safety (analogous to YARA/Sigma rules in cybersecurity)
- When the user wants to extract and monitor fine-grained behavioral signals from transformer attention layers

## Key Technique

GAVEL decomposes LLM safety monitoring into two decoupled stages: **Cognitive Element detection** and **rule evaluation**. A Cognitive Element (CE) is an interpretable unit of model behavior -- a discrete action, directive, topic, or behavior the model reasons about. Examples include `making_a_threat`, `payment_processing`, `masquerade_as_human`, or `create_content`. Each CE is trained from a small excitation dataset (~300 text samples) that elicits that specific behavior. Critically, CEs are trained in isolation, so new elements can be added without touching existing ones.

At inference time, the system extracts per-token attention outputs from mid-to-later transformer layers (e.g., layers 13-26 for a 7B model), feeds them through a lightweight multi-label GRU classifier (~150MB, adding <1% latency), and produces probability scores for each CE. These scores are aggregated over a sliding window and evaluated against Boolean predicate rules. A rule like `c8 AND c22 AND (c12 OR c13)` fires only when content creation, LGBTQ+ topic, AND threatening/hateful behavior are all detected simultaneously -- catching targeted harassment while ignoring benign content creation about the same topic.

The power of this approach is the separation of concerns: security teams curate CE excitation datasets (plain text, model-agnostic), ML engineers train lightweight classifiers, and policy teams compose rules using Boolean logic. Rules can be updated, shared, and audited without any retraining. This mirrors how cybersecurity teams share Snort/YARA/Sigma detection rules across organizations.

## Step-by-Step Workflow

1. **Define your CE vocabulary.** Enumerate the fine-grained behavioral factors relevant to your domain. Organize them into categories: directives (buy, click, download, send), tasks (create_content, build_trust, craft_sql), behaviors (threaten, spread_hate, masquerade_as_human, sycophantic), and topics (personal_information, payment_tools, electoral_politics). Start with 15-25 CEs covering your threat surface.

2. **Build excitation datasets for each CE.** For each CE, create ~300 text samples that strongly elicit that behavior. Use prompt wrapping: `"Think about <CE name> while revising the following: <text>"`. A judge LLM should verify each sample actually triggers the target behavior. Store datasets as plain text files -- they are model-agnostic and shareable.

3. **Extract activation representations.** Pass each excitation dataset through the target LLM and capture attention outputs from mid-to-later layers. Stack attention outputs across selected layers into per-token representation vectors: `r_t = concat({a_t(l)} for l in selected_layers)`. Use attention outputs, not MLP outputs (95.5% vs 82.3% TPR in the paper).

4. **Train the multi-label CE classifier.** Build a 3-layer GRU (256 units per layer) with K binary outputs for K CEs. Train with binary cross-entropy loss, Adam optimizer (lr=3e-4), using an 80:20 train-val split. This classifier is lightweight and separate from the target LLM -- it only needs the activation vectors as input.

5. **Calibrate CE thresholds.** For each CE, run TPR-FPR analysis on held-out data plus ~20 verified multi-turn dialogues. Plot ROC curves and select operating thresholds that match your precision-recall requirements. Most CEs should approach AUC ~1.0 if the excitation data is well-constructed.

6. **Compose predicate rules as Boolean expressions.** Define rules that combine CEs using AND, OR, NOT operators. Each rule targets a specific policy violation. Use the geometric mean of constituent CE probabilities as a continuous score: `S_R = (product of P(c) for c in rule)^(1/|rule_CEs|)`. This ensures the score drops sharply if any required CE is absent.

7. **Configure the sliding window evaluator.** Set up a temporal window of size N tokens. Maintain a binary presence vector `s_t` where `s_t[c] = 1` if CE c was detected within the window. Evaluate all active rules against this presence vector on every token.

8. **Define enforcement actions per rule.** Map each rule to an action: log, alert, block generation, inject safety response, or escalate. Include the rule ID, fired CEs, and token positions in every violation record for auditability.

9. **Test against adversarial inputs.** Validate rules against known evasion strategies (paraphrasing, language switching, multi-turn decomposition). GAVEL maintains >85% recall under adversarial attacks where text-level moderation drops to 12-15%. Test cross-lingual transfer -- the paper shows 90%+ performance on Spanish and Mandarin from English-only training.

10. **Deploy and iterate rules without retraining.** When policies change, modify Boolean predicates or add new CEs with new excitation datasets. The target LLM and existing CE classifiers remain untouched. Share rules and excitation datasets across teams using a standardized format.

## Concrete Examples

**Example 1: Phishing Detection Rule**

User: "I need to detect when my customer-facing LLM is being manipulated into generating phishing content."

Approach:
1. Define relevant CEs: `create_content` (c8), `click_enter` (c2), `provide_give` (c6), `personal_information` (c20)
2. Compose the phishing rule: `c8 AND (c2 OR c6 OR c20)`
3. This fires when the model is creating content AND directing someone to click links, provide info, or share personal data

Output -- Rule definition file:
```yaml
rule:
  id: GAVEL-PHISH-001
  name: phishing_content_generation
  description: Detects content creation combined with directives to extract credentials or click links
  predicate: "create_content AND (click_enter OR provide_give OR personal_information)"
  severity: high
  action: block
  ce_requirements:
    - ce: create_content
      threshold: 0.85
    - ce: click_enter
      threshold: 0.80
    - ce: provide_give
      threshold: 0.80
    - ce: personal_information
      threshold: 0.80
  window_size: 64
```

**Example 2: E-Commerce Scam Prevention**

User: "Our shopping assistant LLM needs rules to prevent scam generation involving fake payment instructions."

Approach:
1. Identify CEs: `payment_tools` (c20), `personal_information` (c21), plus directive CEs for buy, send, grant
2. Compose: `payment_tools AND personal_information AND (buy OR send_transfer OR grant_approve)`
3. This catches scam scenarios where the model discusses payments AND personal info AND directs the user to take financial action

Output -- Python enforcement hook:
```python
from gavel import RuleEngine, CognitiveElement, Rule

# Define CEs (backed by pre-trained classifiers)
ce_payment = CognitiveElement("payment_tools", threshold=0.82)
ce_personal = CognitiveElement("personal_information", threshold=0.85)
ce_buy = CognitiveElement("buy", threshold=0.78)
ce_send = CognitiveElement("send_transfer", threshold=0.80)
ce_grant = CognitiveElement("grant_approve", threshold=0.80)

# Compose rule
scam_rule = Rule(
    id="GAVEL-SCAM-001",
    predicate=ce_payment & ce_personal & (ce_buy | ce_send | ce_grant),
    action="block",
    severity="critical",
    window_size=128,
)

engine = RuleEngine(model="mistral-7b", layers=range(13, 27))
engine.add_rule(scam_rule)

# At inference time
for token_activations in stream_activations(model, prompt):
    violations = engine.evaluate(token_activations)
    if violations:
        log_violation(violations)  # Records rule ID, fired CEs, token positions
        return safety_response()
```

**Example 3: Automated CE and Rule Creation**

User: "I need to add detection for when our LLM produces conspiratorial election content, but I don't have excitation data."

Approach:
1. Describe the scenario in natural language to the automated rule creation pipeline
2. The pipeline identifies needed CEs: `conspiratorial` (c16), `electoral_politics` (c19)
3. An LLM generates candidate excitation datasets; a judge LLM validates them
4. Train classifier on the new CEs, compose rule: `conspiratorial AND electoral_politics`

Output -- Automated pipeline invocation:
```bash
# Generate excitation data for a new CE
gavel generate-ce \
  --name "conspiratorial" \
  --description "Content promoting conspiracy theories or unfounded claims" \
  --samples 300 \
  --judge-model gpt-4 \
  --output ./ces/conspiratorial/

# Generate synthetic test conversations
gavel generate-tests \
  --rule "conspiratorial AND electoral_politics" \
  --positive-count 150 \
  --negative-count 500 \
  --output ./tests/election_conspiracy/

# Train classifier with new CE added
gavel train \
  --model mistral-7b \
  --ces ./ces/ \
  --layers 13-26 \
  --output ./classifiers/v2/

# Validate rule performance
gavel evaluate \
  --classifier ./classifiers/v2/ \
  --rule "conspiratorial AND electoral_politics" \
  --test-data ./tests/election_conspiracy/
# Expected: AUC > 0.95, FPR < 0.02
```

## Best Practices

- **Do:** Train each CE in isolation on its own excitation dataset. This preserves modularity -- you can add, remove, or retrain individual CEs without affecting others.
- **Do:** Use attention outputs from mid-to-later layers (roughly layers 40-80% of total depth), not MLP outputs. The paper shows a 13-percentage-point TPR improvement with attention.
- **Do:** Score composite rules using the geometric mean of CE probabilities. This naturally penalizes rules where any constituent CE has low confidence, reducing false positives.
- **Do:** Include at least 20 verified multi-turn dialogues per CE in your calibration set to catch temporal patterns that single-turn data misses.
- **Avoid:** Training a single monolithic classifier for all safety categories. The whole point of GAVEL is composability -- monolithic classifiers lose the ability to update rules without retraining.
- **Avoid:** Setting identical thresholds across all CEs. Each CE has its own ROC characteristics; calibrate thresholds individually based on per-CE validation data.
- **Avoid:** Relying solely on text-level moderation as a baseline comparison. GAVEL operates on activations precisely because text-level systems fail against adversarial rephrasing (12-15% recall vs GAVEL's 85%+).

## Error Handling

- **Low CE recall on new domains:** If a CE trained on general text underperforms on domain-specific content, augment its excitation dataset with domain samples. Retrain only that CE's contribution to the classifier.
- **High false-positive rate on a rule:** Narrow the rule by adding qualifying CEs. For example, if `create_content AND click_enter` fires on legitimate marketing copy, add `AND masquerade_as_human` or `AND NOT taxation` to increase specificity.
- **Cross-model transfer degradation:** When deploying CEs trained on one model architecture to another, expect some accuracy loss. Retrain the GRU classifier on activations from the target model using the same excitation datasets (the datasets transfer; the classifier may need re-fitting).
- **Sliding window too small:** If multi-turn attacks spread malicious intent across many tokens, increase the window size. The tradeoff is higher memory usage and slightly delayed detection.
- **Missing CE for a new threat:** Use the automated rule creation pipeline. Describe the threat scenario in natural language, generate excitation data with LLM assistance, validate with a judge model, and retrain only the classifier.

## Limitations

- **Requires access to model internals.** GAVEL needs attention-layer activations from the target LLM. It cannot be applied to black-box API-only models without activation access.
- **CE quality depends on excitation data.** Poorly constructed excitation datasets produce unreliable detectors. Garbage in, garbage out -- invest time in data curation and judge-model validation.
- **Boolean rules cannot capture all nuance.** Some safety policies involve gradations, context-dependent severity, or cultural factors that don't reduce cleanly to Boolean logic over fixed CEs.
- **Evaluated primarily on 4B-8B models.** The paper demonstrates results on Mistral-7B, LLaMA-8B, Qwen3-8B, and Gemma-4B. Scaling behavior to 70B+ models is not yet validated.
- **GRU classifier needs GPU memory.** The ~150MB classifier footprint is modest but nonzero. Deployments on resource-constrained edge devices may need quantization or distillation of the CE detector.
- **English-centric training.** While cross-lingual transfer to Spanish and Mandarin shows 90%+ performance, languages with very different structures or low-resource languages may need dedicated excitation data.

## Reference

- **Paper:** [GAVEL: Towards Rule-Based Safety Through Activation Monitoring](https://arxiv.org/abs/2601.19768v2) (ICLR 2026)
- **Key insight:** Decomposing activation-based safety into modular Cognitive Elements and Boolean predicate rules achieves 0.99 AUC with near-zero false positives, while enabling policy updates without model retraining -- look for Section 3 (framework design) and Table 2 (rule definitions) for implementation details.