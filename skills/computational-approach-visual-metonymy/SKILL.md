---
name: "computational-approach-visual-metonymy"
description: "Generate and evaluate visual metonymy -- indirect visual representations that evoke concepts through associated cues rather than literal depiction. Uses a semiotic-theory-grounded pipeline (representamen generation, chain-of-thought visual description, image synthesis) to create images where meaning is implied, not shown. Trigger phrases: 'generate visual metonymy', 'create indirect visual representation', 'visual metaphor pipeline', 'metonymic image generation', 'semiotic image prompt', 'evoke concept visually without showing it'."
---

# Visual Metonymy Generation and Evaluation Pipeline

This skill enables Claude to apply a semiotic-theory-grounded pipeline for generating visual metonymy -- images that communicate a target concept (e.g., "artist," "justice," "Japan") through associated visual cues (e.g., palette and canvas, blindfolded scales, torii gate) rather than depicting the concept directly. The technique, from Ghosh et al. (EACL 2026), chains LLM-based representamen generation, chain-of-thought visual description, and text-to-image synthesis to produce images that require inferential reasoning to decode. It also provides a framework for evaluating whether vision-language models can interpret such indirect references.

## When to Use

- When the user wants to generate image prompts that **evoke a concept indirectly** without naming or literally depicting it (e.g., "show me an image that makes people think of 'freedom' without showing the word or a person being free")
- When building a **visual reasoning benchmark** or dataset that tests whether VLMs understand implied meaning in images
- When designing **brand imagery, editorial illustrations, or symbolic art** where the subject must be communicated through association, not literal depiction
- When constructing **multiple-choice visual question sets** with semantically and visually plausible distractors
- When the user asks to **evaluate a multimodal model's ability** to interpret indirect or metonymic visual content
- When implementing a **semiotic analysis pipeline** that decomposes an image into object, representamen, and interpretant

## Key Technique

**The Semiotic Triad as Computational Scaffold.** The pipeline operationalizes Peirce's semiotic triad: the *object* is the target concept to evoke (e.g., "surgeon"); the *representamen* consists of associated concrete cues that stand in for it (scalpel, surgical mask, operating light); the *interpretant* is the meaning recovered by the viewer. Visual metonymy occurs when a viewer sees the representamens and infers the object without it being shown. This differs from literal image generation (which depicts the concept directly) and from visual metaphor (which maps across conceptual domains). Metonymy stays within one domain -- the cues are genuinely associated with the concept, not analogically mapped from elsewhere.

**Three-Stage Generation Pipeline.** Stage 1 prompts an LLM to generate 5 diverse, concrete objects (representamens) associated with the target concept, covering cultural, contextual, and perceptual links. Stage 2 uses chain-of-thought prompting to compose a visual scene description that integrates a subset of those representamens, explicitly forbidding any mention of the target concept and adding compositional details (lighting, color, tone). Two stylistic variants are produced: *naturalistic* (photorealistic, compositional) and *stylistic* (abstract art styles like Cubism or Surrealism). Stage 3 feeds these descriptions into a text-to-image model. The pipeline achieves 84.3% metonymic quality compared to 41.2% with naive prompting, and generalizes across LLM/image-model combinations (80-87% quality).

**Distractor-Hardened Evaluation.** For benchmarking, distractors are generated through a multi-signal process: CLIP embeddings retrieve visually similar concepts, ConceptNet relations supply semantic neighbors, and three filtering steps (synonym removal, BERT similarity thresholding, ConceptNet graph distance) ensure distractors are plausible but distinguishable. This produces challenging multiple-choice questions where state-of-the-art VLMs reach only 65.9% versus human 86.9%.

## Step-by-Step Workflow

1. **Define the target concept (object).** Identify the abstract or concrete noun the image should evoke. Filter for suitability: concreteness score below 3.5 (on a 1-5 scale) works best -- concepts that are real but not trivially depictable (e.g., "justice," "autumn," "surgeon," "Japan"). Overly abstract concepts (e.g., "epistemology") lack concrete associations; overly concrete ones (e.g., "hammer") are trivially depictable.

2. **Generate representamens.** Prompt the LLM: *"Given the concept '[X]', list 5 representamens -- diverse, concrete physical objects that reflect common cultural, contextual, or perceptual links to this concept. Each must be something that can appear in an image. Do not include the concept itself or any synonym of it."* Validate that no output is a synonym or direct depiction of the concept.

3. **Classify the association type.** Label each representamen as *symbolic* (culturally codified mapping, e.g., dove -> peace), *cultural* (tied to a specific culture or subculture, e.g., kimono -> Japan), or *contextual* (co-occurs in real-world scenarios, e.g., stethoscope -> doctor). This informs how difficult the resulting image will be to interpret -- contextual associations are hardest (54.5% VLM accuracy vs. 76.3% for symbolic).

4. **Compose the visual scene description (naturalistic variant).** Prompt the LLM with chain-of-thought: *"Using 2-4 of the following representamens [list], describe a photorealistic scene. Rules: (a) Do NOT name or describe '[X]' anywhere. (b) The representamens must be the focal point. (c) Include sensory details: lighting, color palette, spatial arrangement. (d) The scene should feel coherent, not like a random collection of objects."* If the concept word appears in the output, re-generate.

5. **Compose the visual scene description (stylistic variant).** Use the same representamens but prompt for abstract art: *"Depict a scene in the style of abstract art (Cubism, Futurism, or Surrealism) using these representamens [list]. Use shape deformation, color dissonance, and geometric rearrangement. Do NOT name '[X]'."*

6. **Generate the image.** Pass each description to a text-to-image model (e.g., Stable Diffusion 3.5 Large with 35 inference steps, guidance scale 7.5). Inspect the output to confirm the target concept is not literally depicted (no text overlays, no direct representation).

7. **Validate metonymic quality.** Ask a VLM or human annotator: *"What concept does this image evoke? Choose from: [target, distractor1, distractor2, distractor3]."* If the target is not recoverable, revisit representamen selection or scene composition. Aim for >80% human recovery rate.

8. **Generate hard distractors (for benchmark use).** For each concept: (a) compute CLIP embeddings of the image and retrieve the top-K visually similar concepts, (b) query ConceptNet `/r/RelatedTo` for semantically adjacent concepts, (c) filter out synonyms (via ConceptNet), near-duplicates (BERT cosine similarity > threshold), and concepts within 2 hops on the ConceptNet graph.

9. **Assemble the evaluation item.** Package as a multiple-choice question: the image, the correct concept, and 3 distractors. Ensure one distractor is visually plausible (CLIP-based) and one is semantically plausible (ConceptNet-based).

10. **Score and analyze.** Run VLMs on the dataset. Break results down by association type (symbolic/cultural/contextual) and style (naturalistic/stylistic) to diagnose where models fail. The largest gaps appear on contextual associations and stylistic images.

## Concrete Examples

**Example 1: Generating a metonymic image prompt for "surgeon"**

User: "Create an image prompt that makes viewers think of a surgeon without showing any person."

Approach:
1. Target concept: "surgeon"
2. Generate representamens: scalpel, surgical mask, operating room light, latex gloves, sterile drape
3. Association type: contextual (these objects co-occur with surgeons in operating rooms)
4. Compose naturalistic description:

Output:
```
Representamens: scalpel, surgical mask, operating room light, latex gloves, sterile drape

Naturalistic scene description:
"A stainless steel surgical tray rests under the cold white glow of an
overhead operating light. On the tray, a scalpel with a polished blade
catches the light beside a pair of powder-blue latex gloves laid flat,
fingers slightly curled. A crumpled surgical mask sits at the edge of
the tray. Behind it, a sterile green drape hangs from an IV stand,
its folds casting soft geometric shadows across the tiled floor."

Stylistic scene description:
"A Cubist composition fragments a surgical tray into overlapping
geometric planes of chrome and teal. A scalpel blade, rendered as a
sharp silver parallelogram, slices across the canvas. Disembodied
latex gloves float as translucent blue polygons. An operating light
above is a radiant white hexagon casting prismatic beams downward."

Image generation parameters:
- Model: Stable Diffusion 3.5 Large
- Steps: 35, Guidance scale: 7.5

Distractors for MCQ: dentist, butcher, scientist
```

**Example 2: Building a visual metonymy evaluation set for "autumn"**

User: "I need a benchmark question that tests whether a VLM understands visual metonymy for the concept 'autumn'."

Approach:
1. Target concept: "autumn"
2. Generate representamens: fallen maple leaves, pumpkin, warm amber light, knit scarf, bare tree branches
3. Compose scene, generate image, then build MCQ with hard distractors

Output:
```
Image description (naturalistic):
"A wooden park bench sits beneath a canopy of bare branches. A scatter
of red and gold maple leaves covers the seat and the path below. Warm
amber light filters through the remaining foliage, casting long
shadows. A folded knit scarf in burnt orange drapes over one armrest."

MCQ structure:
  Image: [generated from above description]
  Question: "What concept does this image evoke?"
  A) Autumn  (correct)
  B) Winter  (distractor: seasonal neighbor, visually similar bare trees)
  C) Harvest (distractor: ConceptNet related, pumpkin overlap)
  D) Hiking  (distractor: park/outdoor setting overlap)

Distractor rationale:
- "Winter" shares bare branches (CLIP visual similarity)
- "Harvest" shares pumpkin/leaf associations (ConceptNet semantic)
- "Hiking" shares outdoor park setting (contextual overlap)
```

**Example 3: Diagnosing VLM failure on contextual metonymy**

User: "My VLM scores 70% on symbolic metonymy but only 50% on contextual. How do I analyze this?"

Approach:
1. Explain the association taxonomy from the paper
2. Identify why contextual is harder
3. Suggest targeted evaluation

Output:
```
Diagnosis framework:

Symbolic associations (dove -> peace, scales -> justice) are culturally
codified and frequently appear in training data. Models memorize these.

Contextual associations (stethoscope + clipboard -> doctor) require
understanding real-world co-occurrence patterns and inferring the
absent entity from its typical environment. These are harder because:
  - Multiple concepts share the same context (stethoscope could suggest
    nurse, hospital, diagnosis, not just doctor)
  - The mapping is not one-to-one; it requires disambiguation
  - Training data may not capture the inferential leap

Recommended analysis:
1. Partition your eval set by association type
2. For contextual failures, check if the model's top-2 predictions
   include the correct concept (partial credit analysis)
3. Test with increasing numbers of representamens per image (2 vs 4)
   to see if more cues help the model disambiguate
4. Compare naturalistic vs stylistic accuracy within contextual items
   -- if stylistic is much worse, the model relies on pixel patterns
   rather than conceptual reasoning
```

## Best Practices

- **Do:** Always validate that the target concept is never literally named or depicted in the generated scene description. Run a string check and semantic similarity check against the concept word and its synonyms.
- **Do:** Use at least 3 representamens per scene to provide sufficient inferential signal. Fewer than 2 makes the image ambiguous even for humans.
- **Do:** Generate both naturalistic and stylistic variants -- they test different reasoning capabilities (compositional understanding vs. abstract pattern recognition).
- **Do:** Use the multi-signal distractor generation process (CLIP + ConceptNet + filtering) for evaluation. Random distractors make benchmarks trivially easy.
- **Avoid:** Using overly abstract concepts (concreteness < 1.5) as targets. If a concept has no concrete physical associations, the pipeline cannot produce meaningful representamens.
- **Avoid:** Treating visual metonymy as visual metaphor. Metonymy operates within a single conceptual domain (surgeon -> surgical tools); metaphor maps across domains (time -> money). Conflating them produces incoherent images.

## Error Handling

- **Concept word leaks into description:** The pipeline includes a fail-safe re-generation step. If the target word or a close synonym (BERT cosine > 0.85) appears in the scene description, discard and re-prompt with an explicit reminder. Check both the description and the image for text overlays.
- **Representamens are too generic:** If generated associations are shared by many concepts (e.g., "table" for "dinner"), add a uniqueness constraint: each representamen should have higher pointwise mutual information with the target than with any distractor.
- **Image fails to convey the concept:** If human annotators cannot recover the target from the image, the representamen set may be too sparse or the scene composition may scatter cues without coherent spatial arrangement. Re-compose with fewer representamens arranged in a single focal area.
- **Distractors are too easy or too hard:** If VLM accuracy is >90%, distractors lack visual/semantic overlap with the target; regenerate using tighter CLIP similarity thresholds. If accuracy is <30%, distractors may be near-synonyms; verify ConceptNet graph distance is >= 3 hops.

## Limitations

- The pipeline depends on the LLM's cultural knowledge for representamen generation. Concepts from underrepresented cultures may yield stereotypical or incomplete associations.
- Text-to-image models may inject unintended elements (text, faces, symbols) that leak the target concept. Manual inspection of generated images remains necessary.
- Contextual metonymy (the most interesting type) is also the hardest -- both for models to interpret and for the pipeline to generate reliably. Expect ~55% VLM accuracy on contextual items versus ~76% on symbolic ones.
- The approach works best for concepts with strong, widely-shared physical associations. Highly personal or niche concepts (e.g., "nostalgia for a specific childhood home") lack universal representamens.
- Evaluation via multiple-choice questions caps the difficulty ceiling. Open-ended concept recovery (no options provided) is significantly harder and remains an open problem.

## Reference

Ghosh, S., Liu, L., & Jiang, T. (2026). *A Computational Approach to Visual Metonymy.* EACL 2026. [arXiv:2601.17706](https://arxiv.org/abs/2601.17706v1) -- See Section 3 for the full semiotic pipeline, Section 4 for ViMET dataset construction, and Appendix L for complete prompt templates. Dataset: [github.com/cincynlp/ViMET](https://github.com/cincynlp/ViMET).