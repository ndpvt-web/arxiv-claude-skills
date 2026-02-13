---
name: "menaspeechbank-reference-voice-bank"
description: |
  Build persona-conditioned synthetic speech-text conversation datasets using the MENASpeechBank pipeline.
  Constructs culturally-grounded persona profiles, generates scenario taxonomies, matches personas to
  scenarios via semantic similarity, produces multi-turn role-play conversations with LLMs, and synthesizes
  speaker-conditioned audio. Use when:
  - "Build a persona-conditioned conversation dataset"
  - "Generate synthetic multi-turn dialogues with speaker diversity"
  - "Create a reference voice bank for AudioLLM training"
  - "Design a culturally-grounded persona profile system"
  - "Build a scenario taxonomy for conversational AI"
  - "Synthesize speech preserving speaker identity from reference audio"
---

# MENASpeechBank: Persona-Conditioned Synthetic Conversation Pipeline

This skill enables Claude to guide users through building large-scale, persona-conditioned synthetic speech-text conversation datasets following the MENASpeechBank methodology. The pipeline takes a small reference voice bank (e.g., 18K utterances from 124 speakers) and scales it to hundreds of thousands of multi-turn conversations with synthesized audio, preserving speaker identity and cultural diversity. It covers persona construction from demographic and value-survey data, hierarchical scenario taxonomy generation, semantic persona-scenario matching, LLM-driven role-play conversation generation, and reference-audio-conditioned TTS synthesis.

## When to Use

- When the user needs to create training data for Audio Large Language Models (AudioLLMs) and lacks diverse conversational speech-text pairs
- When building a persona system grounded in real cultural values (e.g., using World Values Survey data) rather than shallow demographic labels
- When the user wants to scale a small speaker corpus into a large synthetic conversation dataset while preserving speaker identity
- When designing a hierarchical scenario taxonomy for conversational AI training coverage
- When the user asks about persona-to-scenario matching using embedding-based semantic similarity
- When generating multi-turn role-play conversations where one side speaks "as" a specific persona
- When the user needs to condition TTS on reference audio to maintain voice consistency across a dataset

## Key Technique

The core insight of MENASpeechBank is **decoupling speaker identity from conversational content** through a five-stage pipeline. Rather than recording real multi-speaker conversations (expensive, slow, privacy-constrained), the pipeline constructs rich persona profiles from demographic metadata enriched with country-level World Values Survey (WVS Wave 7) aggregates — covering religiosity, family/gender norms, institutional trust, political preferences, and cultural dimensions (traditional vs. secular-rational; survival vs. self-expression). Each persona also receives a sampled OCEAN (Big Five) personality vector by perturbing fixed base values, producing natural variation.

The pipeline then generates a hierarchical scenario taxonomy (~5K scenarios) structured as Task/Service domains and Knowledge/Topic domains, with intermediate topics created via LLM generation and near-duplicate filtering (cosine similarity threshold 0.85). Personas are matched to scenarios using a hybrid score combining cosine similarity from `all-MiniLM-L6-v2` sentence embeddings with token-overlap Jaccard similarity (retention threshold ≥ 0.05). An LLM (e.g., GPT-4.1) then generates role-play conversations where the user speaks as the persona and the assistant responds as a helpful agent, conditioned on a first-person persona summary (90-180 words). Finally, only the user turns are synthesized via XTTS-v2 conditioned on a single reference audio clip per speaker, preserving speaker identity across all that speaker's conversations. This yields ~417K conversations from just 124 speakers.

The critical design choice is the **asymmetric synthesis**: only user turns become speech (simulating real AudioLLM input), while assistant turns stay as text. This mirrors how AudioLLMs actually operate — processing spoken user input and generating text/speech responses — making the training data distribution-matched to deployment.

## Step-by-Step Workflow

1. **Curate a reference voice bank.** Collect or assemble short audio clips (5-8 seconds each) from target speakers. Filter aggressively: WER ≤ 0.05 against reference transcripts, verify dialect with a dialect-ID tool (e.g., Tamyiz for Arabic, or language-ID for multilingual corpora). Aim for 10+ clean segments per speaker. Record metadata: country, language/dialect, age, gender.

2. **Construct persona profiles from demographics + values data.** For each speaker, retrieve country-level WVS Wave 7 aggregates (or equivalent cultural survey data) and adjust by age bracket. Append sampled OCEAN personality vectors (perturb a base vector with controlled noise). Generate a first-person persona summary (90-180 words) using an LLM, then validate with a deterministic Persona Quality Index (PQI) — a rule-based checklist ensuring coherence, non-contradiction, and cultural plausibility. Target ≥85% of personas passing all checks.

3. **Build a hierarchical scenario taxonomy.** Define two top-level branches: Task/Service domains (e.g., booking, customer support, form-filling) and Knowledge/Topic domains (e.g., culture, religion, science). Use an LLM to generate ~10 intermediate topics per domain path, then expand each topic into specific scenarios. Apply near-duplicate filtering (cosine similarity threshold 0.85) to remove redundant scenarios. Target ~5K unique scenarios with a distribution of ~80% Knowledge/Topic and ~20% Task/Service.

4. **Match personas to scenarios via hybrid semantic similarity.** Embed both persona summaries and scenario descriptions using a sentence-transformer model (e.g., `all-MiniLM-L6-v2`). Compute a hybrid score: weighted combination of cosine similarity (semantic) and Jaccard similarity (lexical token overlap). Filter pairs below a minimum threshold (0.05). Each persona can match to multiple scenarios and vice versa.

5. **Generate multi-turn role-play conversations with an LLM.** For each persona-scenario pair, prompt the LLM with: (a) system instructions enforcing language constraints (e.g., no code-switching, MSA-only for Arabic), (b) the persona summary as user context, (c) the scenario description. The LLM generates a multi-turn conversation where the "user" speaks in character as the persona and the "assistant" provides helpful responses. Vary conversation initiation: ~50-60% user-initiated, ~40-50% assistant-initiated.

6. **Filter and deduplicate generated conversations.** Remove near-duplicate conversations (cosine similarity threshold 0.80). Run automated quality checks: coherence, persona fidelity, language correctness, safety. Discard conversations that fail quality gates.

7. **Synthesize user turns with reference-audio-conditioned TTS.** Use a speaker-cloning TTS model (e.g., XTTS-v2) to synthesize only the user turns, conditioning on one reference audio clip per speaker. This preserves speaker identity while generating novel utterances the speaker never actually said. Leave assistant turns as text.

8. **Evaluate synthesis quality.** Measure Word Error Rate (WER) by transcribing synthesized audio back to text (target ≤10%), perceptual quality via NISQA (target ≥3.5), and speaker similarity via cosine similarity of speaker embeddings between reference and synthesized audio (target ≥0.45).

9. **Split the dataset.** Stratify by scenario for train/dev splits (e.g., 90%/10% of non-test data). Hold out a speaker-profile-stratified test set (~12%) to evaluate generalization to unseen persona-scenario combinations.

10. **Validate with downstream benchmarks.** Fine-tune a target AudioLLM (e.g., Qwen2.5-Omni-7B with LoRA r=32, alpha=64) on the synthetic data. Evaluate with LLM-as-judge scoring across rubrics: relevance, completeness, specificity, coherence, context tracking, calibration, language/tone match, and safety.

## Concrete Examples

**Example 1: Building a persona profile from speaker metadata**

User: "I have a speaker from Egypt, female, age 25. How do I create a persona profile for her?"

Approach:
1. Retrieve Egypt's WVS Wave 7 aggregates for the 20-29 age bracket
2. Map values to persona attributes (e.g., moderate religiosity, high family orientation, secular-leaning on governance)
3. Sample an OCEAN vector: e.g., O=0.72, C=0.58, E=0.65, A=0.70, N=0.40
4. Generate a first-person summary with the LLM

Output:
```
Persona Summary (142 words):
"I'm a 25-year-old woman from Cairo. I studied computer engineering at
Ain Shams University and now work as a junior software developer at a
fintech startup. Family is central to my life — I visit my parents in
Heliopolis every Friday and we always have lunch together. I consider
myself moderately religious; I pray regularly but I also believe in
personal freedom when it comes to lifestyle choices. I'm curious about
new technologies, especially how AI might reshape banking in Egypt.
I enjoy reading contemporary Arabic literature and watching Turkish
dramas with my mother. I'm generally optimistic about Egypt's economic
future, though I worry about inflation and housing costs. I value
honesty and directness in conversations and tend to be organized
and goal-oriented in my work."

PQI Check: 12/12 (all attributes coherent, no contradictions, culturally plausible)
```

**Example 2: Generating a scenario taxonomy branch**

User: "Generate the Knowledge/Topic branch of the scenario taxonomy for culture-related conversations."

Approach:
1. Define the domain path: Knowledge > Culture & Society
2. Prompt LLM to generate 10 intermediate topics (e.g., traditional cuisine, wedding customs, folk music, urban vs. rural life, diaspora identity, generational gaps, language preservation, religious festivals, social media influence, craftsmanship)
3. Expand each topic into 5-10 specific scenarios
4. Filter near-duplicates at cosine threshold 0.85

Output:
```yaml
domain: Knowledge/Topic
category: Culture & Society
topics:
  - name: "Traditional Cuisine"
    scenarios:
      - "Discussing how recipes are passed down through generations in Gulf families"
      - "Comparing street food culture between Cairo and Casablanca"
      - "Explaining the significance of dates and coffee in Bedouin hospitality"
      - "Debating whether modern restaurants preserve authentic Levantine cooking"
  - name: "Wedding Customs"
    scenarios:
      - "Describing the stages of a traditional Moroccan wedding ceremony"
      - "Discussing how wedding traditions have changed for young Saudis"
      - "Comparing Coptic and Muslim wedding customs in Egypt"
# ... (continues for all 10 topics, ~50 scenarios total for this category)
```

**Example 3: Full pipeline run for a small-scale proof of concept**

User: "I have 20 speakers with 10 audio clips each. Walk me through scaling this to 10K conversations."

Approach:
1. Curate: Filter the 200 clips to those with WER ≤ 0.05 (expect ~160-180 usable)
2. Personas: Create 20 persona profiles (one per speaker) with WVS enrichment
3. Taxonomy: Generate ~500 scenarios (scaled down from 5K proportionally)
4. Match: Embed and pair personas to scenarios; expect ~25 scenarios per persona on average
5. Generate: Produce 500 conversations per persona = 10K total multi-turn dialogues
6. Synthesize: Run XTTS-v2 on user turns, conditioning each on that speaker's reference clip
7. Evaluate: Sample 100 conversations for WER/NISQA/SpkCos checks

Output:
```
Pipeline Summary:
- Speakers: 20 (after filtering: 20)
- Reference clips: 178 (after WER filtering)
- Personas: 20 (PQI pass rate: 90%)
- Scenarios: 487 (after dedup from 530 generated)
- Persona-scenario pairs: 512 (above 0.05 threshold)
- Conversations generated: 10,240
- User turns synthesized: ~51,200 audio files
- Avg WER on synthesis: 9.3%
- Avg NISQA: 3.7
- Avg SpkCos: 0.51

Train/Dev/Test split: 8,192 / 1,024 / 1,024
```

## Best Practices

- **Do:** Filter reference audio aggressively (WER ≤ 0.05) before using it for TTS conditioning. Poor reference quality cascades into all synthesized utterances for that speaker.
- **Do:** Use hybrid similarity (semantic + lexical) for persona-scenario matching rather than pure cosine similarity. The lexical component catches surface-level topical alignment that embeddings may miss.
- **Do:** Generate first-person persona summaries (90-180 words) rather than structured attribute lists. LLMs produce more natural conversations when prompted with narrative persona descriptions.
- **Do:** Synthesize only user turns and keep assistant turns as text. This matches real AudioLLM deployment where users speak and the model responds.
- **Avoid:** Skipping near-duplicate filtering on scenarios or conversations. Without it, the dataset collapses into repetitive patterns that hurt downstream model diversity.
- **Avoid:** Using the same reference audio clip for all utterances of a speaker. Rotate among that speaker's clean clips to capture natural intra-speaker variation in prosody and energy.

## Error Handling

- **Low PQI scores:** If >15% of personas fail quality checks, review the WVS-to-attribute mapping. Common failures include contradictory values (e.g., extremely religious + extremely secular-liberal). Add explicit coherence constraints to the generation prompt.
- **Poor persona-scenario match rates:** If too few pairs exceed the 0.05 threshold, the persona summaries may be too generic. Regenerate with more specific cultural and professional details. Alternatively, lower the threshold to 0.03 for exploratory runs.
- **High WER on synthesized audio:** If WER exceeds 15%, check reference audio duration (should be 5-8 seconds) and SNR. XTTS-v2 degrades with noisy or too-short reference clips. Also verify that the text being synthesized matches the language of the reference audio.
- **Low speaker cosine similarity:** SpkCos below 0.40 usually indicates a language mismatch (e.g., English reference audio conditioning Arabic synthesis). Ensure reference and target language are aligned per speaker.
- **LLM code-switching in conversations:** If the LLM mixes languages despite instructions, strengthen the system prompt with explicit "respond ONLY in [language]" constraints and add a post-generation language-ID filter.

## Limitations

- **Speaker diversity ceiling:** The pipeline preserves but cannot create new speaker identities. If the reference bank has only 20 speakers, all 10K conversations will sound like 20 people regardless of persona diversity.
- **Cultural value approximation:** WVS country-level aggregates are statistical summaries, not individual profiles. A persona built from Egyptian WVS averages may not represent any actual Egyptian individual. This is an intentional design tradeoff for scalability.
- **TTS quality gap:** XTTS-v2 achieves SpkCos ~0.49, meaning synthesized speech is noticeably different from real recordings under careful listening. This gap is smaller for languages with more TTS training data (English) and larger for under-resourced dialects.
- **Persona-scenario matching is approximate:** Semantic similarity does not guarantee that a persona would realistically engage in a given scenario. Manual review of a sample is essential for quality-critical applications.
- **Arabic dialect limitation:** The current pipeline generates MSA text but synthesizes it through a TTS model that may not capture dialectal phonology. True dialectal coverage requires dialect-specific TTS models or real speaker recordings.

## Reference

**Paper:** [MENASpeechBank: A Reference Voice Bank with Persona-Conditioned Multi-Turn Conversations for AudioLLMs](https://arxiv.org/abs/2602.07036v1) (Ali et al., 2026). Focus on Section 3 (pipeline architecture), Table 2 (scenario taxonomy distribution), and Section 5 (evaluation protocol with LLM-as-judge rubrics).