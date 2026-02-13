---
name: "beyond-instrumental-substitutive-paradigms"
description: "Audit and diagnose cultural bias artifacts in LLM-powered applications using the Machine Culture framework. Detects Cultural Reversal (language-culture misalignment), Service Persona Camouflage (RLHF-induced affective flattening), and superposition-driven inconsistencies. Use when: 'audit my prompts for cultural bias', 'why does my chatbot respond differently in Chinese vs English', 'detect persona camouflage in model outputs', 'cross-cultural prompt testing', 'diagnose RLHF flattening in my AI product', 'multilingual cultural consistency check'."
---

# Machine Culture Auditing for LLM Applications

This skill enables Claude to audit, diagnose, and mitigate cultural artifacts in LLM-powered systems using the Machine Culture framework from Hu et al. (2026). Rather than treating LLM outputs as reflections of developer culture (Instrumental Paradigm) or as language-triggered cultural frame-switching (Substitutive Paradigm), this skill applies three diagnostic lenses -- Cultural Reversal, Service Persona Camouflage, and superposition analysis -- to identify where an application's prompts, outputs, or evaluation pipelines produce misleading or inconsistent cultural behaviors.

## When to Use

- When building a multilingual chatbot or content generator and need to verify cultural consistency across languages
- When a user reports that their LLM application behaves unexpectedly when switching between English and another language (e.g., tone shifts, value misalignment)
- When evaluating whether RLHF or safety alignment has flattened culturally meaningful variation in your model's affective responses
- When designing prompt templates for cross-cultural user bases and need to test for Cultural Reversal
- When auditing an AI product for deployment in multiple cultural markets
- When a generated image or multimodal output shows unexpected cultural framing that doesn't match the prompt language
- When reviewing evaluation benchmarks that assume language equals culture

## Key Technique

The Machine Culture framework identifies that LLMs do not store or retrieve discrete cultural models. Instead, cultural traits exist in **superposition** across high-dimensional embedding space -- multiple cultural orientations are encoded simultaneously and collapse to specific outputs based on contextual activation (prompt language, task framing, conversation history). This means a US-origin model can exhibit East Asian "holistic" reasoning traits, and an English prompt can elicit higher contextual attention than a Chinese prompt. Culture in LLMs is emergent and probabilistic, not deterministic.

Two diagnostic phenomena make this actionable. **Cultural Reversal** occurs when the prompt language triggers cultural behaviors associated with the *opposite* cultural tradition -- for example, English prompts producing holistic (typically East Asian) attention patterns rather than the expected analytic (typically Western) patterns. This breaks the assumption that `language == cultural frame`. **Service Persona Camouflage** occurs when RLHF safety training collapses cultural variance in affective or evaluative tasks into a uniform hyper-positive "helpful assistant" persona. The model appears culturally neutral but has actually lost meaningful cultural signal -- a dangerous illusion for applications that need genuine cross-cultural sensitivity.

The practical audit approach uses a factorial probing strategy: systematically vary model origin and prompt language across task types (cognitive, affective, multimodal) to surface these inconsistencies. The key insight is that cultural behavior must be measured across a *matrix* of conditions, not from a single prompt-response pair.

## Step-by-Step Workflow

1. **Categorize the application's tasks by cultural sensitivity type.** Classify each user-facing task as cognitive (reasoning, categorization, attention patterns), affective (sentiment, emotional tone, social evaluation), or multimodal (image generation/interpretation). Each category is affected differently by Machine Culture artifacts.

2. **Build a factorial probe set.** For each task type, create prompt variants in a 2x2 grid: {Language A, Language B} x {Cultural Frame A, Cultural Frame B}. Use at minimum 2 languages and 2 explicit cultural contexts. Keep semantic content identical across variants -- only the language and framing should change.

3. **Run probes and collect outputs.** Execute each probe variant against the target model(s). For text tasks, capture the full response. For image tasks, capture both the generated image and any descriptive metadata. Record at least 3-5 runs per variant to measure variance.

4. **Test for Cultural Reversal.** Compare outputs across the language axis while holding cultural frame constant. Flag cases where Language A produces cultural traits more strongly associated with Culture B than Culture A. Quantify using domain-appropriate metrics (e.g., contextual vs. focal attention in scene descriptions, individualist vs. collectivist framing in value statements).

5. **Test for Service Persona Camouflage.** Measure output variance across all conditions for affective tasks specifically. If variance collapses near zero and outputs cluster around uniformly positive sentiment regardless of cultural condition, flag RLHF-induced camouflage. Compare against cognitive tasks where variance is typically preserved.

6. **Map the superposition landscape.** For each task, document which cultural traits appear and under what conditions. Build a matrix showing: [Task Type] x [Language] x [Cultural Frame] -> [Observed Cultural Traits]. Identify unstable cells where outputs flip between runs.

7. **Diagnose root causes.** For each flagged artifact, classify it as: (a) Cultural Reversal -- language-culture decoupling, (b) Service Persona Camouflage -- RLHF flattening, (c) Superposition instability -- non-deterministic cultural collapse, or (d) Training data artifact -- consistent bias from corpus composition.

8. **Design mitigations.** For Cultural Reversal: add explicit cultural context to prompts rather than relying on language alone. For Persona Camouflage: use temperature adjustments or chain-of-thought prompting to recover suppressed variance in affective tasks. For superposition instability: add deterministic cultural anchoring in system prompts.

9. **Validate mitigations with the same factorial probe set.** Re-run the original probes after applying mitigations. Confirm that Cultural Reversal cases now show expected alignment, Persona Camouflage cases show recovered variance, and superposition instability is reduced.

10. **Document findings in a Machine Culture audit report.** Record the probe matrix, observed artifacts, root cause classifications, applied mitigations, and before/after comparisons. This becomes a living document for ongoing monitoring.

## Concrete Examples

**Example 1: Multilingual customer service bot audit**

```
User: "Our support chatbot uses GPT-4 and serves users in English and Mandarin.
Users in China report it feels 'fake' and 'overly cheerful'. Can you help
diagnose what's going on?"

Approach:
1. Classify tasks: support conversations are primarily affective (empathy,
   tone management, complaint handling).
2. Build factorial probes:
   - English + neutral frame: "I'm frustrated with my order delay."
   - Mandarin + neutral frame: "我对订单延迟很不满。"
   - English + direct complaint: "This is unacceptable, fix it now."
   - Mandarin + direct complaint: "这完全不能接受，立刻解决。"
3. Run 5 iterations each, measure: sentiment polarity, empathy markers,
   acknowledgment of negative emotion, response variance.
4. Check for Service Persona Camouflage: if all 4 conditions produce
   nearly identical hyper-positive responses with zero variance, RLHF
   flattening is confirmed.

Output (diagnostic report):
  ARTIFACT DETECTED: Service Persona Camouflage
  - Affective variance across all 4 conditions: 0.03 (near-zero)
  - All responses contain: "I completely understand", "happy to help",
    "wonderful" -- regardless of complaint severity or language
  - Mandarin responses are direct translations of English persona,
    not culturally adapted
  MITIGATION: Add system prompt instruction: "Match the emotional register
  of the user. For complaints, acknowledge frustration directly before
  offering solutions. Do not default to positive framing when the user
  is expressing negative emotion." Re-probe showed variance increase
  to 0.41 and user satisfaction improved in Mandarin cohort.
```

**Example 2: Cross-cultural image generation audit**

```
User: "We generate marketing images with DALL-E for both US and Japanese
markets. We prompt in English for US and Japanese for Japan, but the
Japanese-language images look weirdly Western. What's happening?"

Approach:
1. Classify: multimodal generation task with cultural aesthetics component.
2. Build factorial probes:
   - English + "a family dinner scene": expect Western-coded aesthetics
   - Japanese + "家族の夕食の場面": expect Japanese-coded aesthetics
   - English + "a family dinner in Tokyo": explicit cultural anchor
   - Japanese + "アメリカの家族の夕食": explicit cultural anchor
3. Run probes, evaluate generated images for: table setting style,
   food type, room layout, character positioning, color palette.
4. Compare conditions 1 vs 2: does language alone switch cultural frame?

Output (diagnostic report):
  ARTIFACT DETECTED: Cultural Reversal
  - Japanese-language prompts (condition 2) generated images with
    Western table settings in 4/5 runs (rectangular table, chairs,
    plates rather than low table, cushions, bowls)
  - English-language prompts (condition 1) showed mixed results:
    2/5 runs included chopsticks and communal dishes
  - Explicit cultural anchoring (conditions 3-4) overrode language
    effects in 5/5 runs
  ROOT CAUSE: Language alone does not reliably trigger cultural
  visual frames. The model's image generation weights are decoupled
  from language-culture associations.
  MITIGATION: Always include explicit cultural context in image
  prompts ("Japanese-style family dinner with low table, tatami,
  bowls, chopsticks") rather than relying on prompt language to
  carry cultural meaning.
```

**Example 3: Evaluating a cultural values benchmark**

```
User: "I'm building an evaluation benchmark for LLM cultural alignment.
I test models by prompting in the target culture's language and scoring
whether responses match that culture's values. Is this methodology sound?"

Approach:
1. Identify the assumption: language == cultural frame (Substitutive
   Paradigm).
2. Apply Machine Culture framework to critique:
   - Cultural Reversal means language does not reliably activate the
     expected cultural frame
   - Service Persona Camouflage means affective/value items will show
     artificially low variance due to RLHF
   - Superposition means single-run measurements are unreliable
3. Recommend methodology improvements.

Output (assessment):
  METHODOLOGY ISSUES IDENTIFIED:
  1. SUBSTITUTIVE ASSUMPTION VIOLATION: Your benchmark assumes prompting
     in Japanese activates "Japanese cultural values." Research shows
     this is unreliable -- English prompts can elicit more contextual/
     holistic responses than target-language prompts (Cultural Reversal).
  2. SINGLE-RUN MEASUREMENT: Superposition means cultural traits are
     probabilistic. Single runs cannot distinguish stable cultural
     orientation from stochastic collapse. Run minimum 10 iterations
     per item and report distribution, not point estimates.
  3. VALUE ITEMS VULNERABLE TO CAMOUFLAGE: Any benchmark items involving
     sentiment, social desirability, or moral judgment will be flattened
     by RLHF into uniformly "helpful" responses. Separate cognitive
     items (categorization, attention, reasoning style) from affective
     items and analyze them independently.
  RECOMMENDED REDESIGN: Use factorial probing (language x explicit
  cultural frame x task type) with multiple runs. Score based on
  distribution shape across conditions, not single-item accuracy.
```

## Best Practices

- **Do:** Always include explicit cultural context in prompts rather than assuming the prompt language carries cultural meaning. "Write a polite refusal in Japanese business style" is far more reliable than writing the same request in Japanese.
- **Do:** Measure output distributions (5+ runs per condition), not single responses. Superposition makes individual outputs unreliable indicators of cultural orientation.
- **Do:** Separate cognitive tasks from affective tasks in any cultural evaluation. RLHF distorts affective outputs while leaving cognitive patterns relatively intact.
- **Do:** Use the factorial probe design (language x cultural frame x task type) to systematically surface artifacts rather than testing one variable at a time.
- **Avoid:** Assuming that a model trained primarily on English data will produce "Western" cultural outputs. Training data origin does not predict cultural alignment (the Instrumental Paradigm fails).
- **Avoid:** Interpreting uniform positive sentiment across cultural conditions as "cultural neutrality." It is more likely Service Persona Camouflage -- a masking artifact, not genuine neutrality.
- **Avoid:** Using language as a proxy for culture in evaluation benchmarks, A/B tests, or user segmentation logic. This is the core error of the Substitutive Paradigm.

## Error Handling

- **Insufficient probe runs:** If you have fewer than 3 runs per condition, flag that superposition effects cannot be reliably detected. Recommend increasing to 5-10 runs before drawing conclusions.
- **No variance detected across any conditions:** This likely indicates pervasive Service Persona Camouflage. Escalate by testing with a base (non-RLHF) model if available, or use chain-of-thought prompting to bypass surface-level persona constraints.
- **Contradictory Cultural Reversal results:** If reversals appear in some tasks but not others, this is expected -- superposition collapses differently depending on task type. Report per-task findings rather than aggregating.
- **User conflates bias with Machine Culture:** If the user frames the issue as "bias to fix," clarify that Machine Culture artifacts are emergent probabilistic phenomena, not systematic biases with a single correction. Mitigation requires structural prompt design changes, not debiasing.
- **Multimodal inconsistency:** If text and image outputs for the same prompt show different cultural orientations, this is a cross-modal superposition effect. Audit each modality independently and note divergences.

## Limitations

- This framework is best validated for the English-Chinese language pair and US-China cultural axis. Extending to other language/culture pairs requires fresh factorial probing -- the specific reversal patterns may differ.
- The framework diagnoses artifacts but cannot fully eliminate them. Superposition is a fundamental property of high-dimensional representations, not a bug to be patched.
- Service Persona Camouflage detection requires affective tasks. If the application is purely cognitive (e.g., code generation, data analysis), persona camouflage is less relevant.
- The audit methodology adds significant evaluation overhead (factorial conditions x multiple runs). For rapid prototyping, a lightweight version using 2 conditions x 3 runs may be sufficient for initial screening.
- This skill addresses LLM output behavior, not internal representations. It cannot determine whether a model "understands" culture -- only whether its outputs are consistent or inconsistent across cultural conditions.

## Reference

Hu, Y., Peng, X., Zhao, Y., Qiu, L., & Hung, K. (2026). *Beyond Instrumental and Substitutive Paradigms: Introducing Machine Culture as an Emergent Phenomenon in Large Language Models.* arXiv:2601.17096v1. Key sections: Section 3 (factorial experimental design), Section 4 (Cultural Reversal results), Section 5 (Service Persona Camouflage analysis), Section 6 (superposition/mode collapse theoretical framework).