---
name: "assessment-generative-named-entity"
description: "Build generative NER systems using LLMs with optimal output formats and prompt engineering. Use when: 'extract entities from text', 'build a NER pipeline with an LLM', 'named entity recognition with generative models', 'format NER output as XML or bracketed', 'fine-tune a model for entity extraction', 'nested entity recognition'."
---

# Generative Named Entity Recognition with LLMs

This skill enables Claude to design, implement, and optimize Named Entity Recognition (NER) pipelines that use large language models as generative entity extractors rather than traditional sequence labeling. Based on a systematic evaluation of eight open-source LLMs across four NER benchmarks, this skill teaches the precise prompt structures, output format choices, and fine-tuning configurations that achieve F1 scores competitive with encoder-based models (93.85 F1 on CoNLL2003) while supporting both flat and nested entities in a single unified approach.

## When to Use

- When the user asks to extract named entities from text using an LLM (local or API-based)
- When building a NER pipeline that must handle **nested entities** (e.g., a protein name inside a DNA binding site)
- When the user wants to fine-tune an open-source LLM (LLaMA, Qwen) for entity extraction
- When choosing between output formats (XML, bracketed, JSON) for a generative NER system
- When writing prompts or instructions for zero-shot or few-shot entity extraction
- When replacing a traditional NER model (spaCy, BERT-based) with a generative alternative
- When the user needs entity extraction in a specialized domain (biomedical, legal, financial) where label definitions matter

## Key Technique

**Generative NER** reformulates entity recognition from token-level classification into text generation. Instead of assigning a BIO tag to every token, the LLM receives an instruction containing: (1) a task description specifying the output format, (2) semantic definitions for each entity label, and (3) the input sentence. The model then generates the complete output -- either the original sentence with inline entity markup, or a structured JSON object listing extracted entities. This approach unifies flat NER (non-overlapping entities) and nested NER (entities within entities) under one framework.

The critical finding is that **output format choice dominates performance**. Inline bracketed (`[Entity | LABEL]`) and inline XML (`<LABEL>Entity</LABEL>`) formats achieve the highest F1 scores (91-94 on standard benchmarks), while offset-based JSON formats that require character positions collapse to under 40 F1. This is because inline formats preserve the original sentence structure, letting the model leverage its language modeling strengths, whereas offset computation requires precise counting that generative models handle poorly. For nested entities, XML is strictly superior since bracketed notation cannot represent overlapping spans.

Parameter-efficient fine-tuning with LoRA (rank=256, alpha=512) on as few as 14K training examples produces models that match or exceed encoder-based baselines. Crucially, this fine-tuning preserves the model's general capabilities -- benchmarks like MMLU and TruthfulQA remain stable, and entity-heavy tasks like DROP actually improve by 25-45 F1 points.

## Step-by-Step Workflow

1. **Define the entity schema with semantic label descriptions.** For each entity type, write a 1-2 sentence definition specifying what qualifies and what doesn't. Example: `ORG: Collective entities such as companies, institutions, and governmental bodies. Excludes informal groups.` These definitions go directly into the prompt and measurably improve extraction quality.

2. **Select the output format based on entity structure:**
   - **Flat NER (no nesting):** Use inline bracketed: `[Entity Text | LABEL]`
   - **Nested NER (overlapping entities):** Use inline XML: `<LABEL>Entity Text</LABEL>`
   - **Structured downstream consumption:** Use category-grouped JSON: `{"LABEL": ["entity1", "entity2"]}`
   - **Never use offset-based JSON** (character start/end positions) -- LLMs cannot reliably compute offsets.

3. **Construct the instruction prompt with three components:**
   - Task description: State the extraction goal and specify the exact output format with a mini-example
   - Label definitions: List every entity type with its semantic definition
   - Input sentence: The text to process, clearly delimited

4. **For zero-shot/few-shot usage (no fine-tuning):** Provide 2-3 annotated examples in the prompt demonstrating the exact output format. Ensure examples cover edge cases like multi-word entities and sentences with no entities.

5. **For fine-tuning:** Prepare training data as instruction-output pairs in JSONL format. Each entry contains the full instruction prompt as input and the formatted entity output as the target. Use LLaMA-Factory or similar frameworks with LoRA (r=256, alpha=512, lr=2e-5, 2 epochs, batch size 8).

6. **Run inference and parse the generated output.** Write a parser specific to your chosen format -- regex for bracketed (`\[([^|]+)\|\s*(\w+)\]`), XML parser for XML tags, or `json.loads` for JSON formats. Handle malformed output gracefully.

7. **Evaluate using entity-level micro-F1.** An entity is correct only if both its text span and label match exactly. Compute precision, recall, and F1 per label and overall. Track error categories: wrong type, boundary errors, omitted mentions.

8. **Iterate on label definitions for the worst-performing entity types.** If `MISC` entities have low recall, expand the definition with more examples of what qualifies. If boundary errors dominate, add examples showing exact span boundaries in your few-shot demonstrations.

## Concrete Examples

**Example 1: Building a flat NER prompt for news text**

User: "Extract named entities from news articles using an LLM. The entities are person, organization, location, and miscellaneous."

Approach:
1. Choose inline bracketed format (flat NER, no nesting needed)
2. Write semantic label definitions
3. Construct the full instruction prompt

Output prompt template:
```
Your task is to identify all named entities in the input sentence. Rewrite
the sentence by enclosing each entity in brackets with its label:
[Entity Text | LABEL].

Entity types:
- PER: Named individuals, including real people and fictional characters.
- ORG: Companies, institutions, governmental bodies, and other collective entities.
- LOC: Countries, cities, geographic features, and named structures.
- MISC: Nationalities, religions, languages, events, and other proper nouns
  not covered by PER, ORG, or LOC.

If the sentence contains no entities, output it unchanged.

Input: EU rejects German call to boycott British lamb.
Output: [EU | ORG] rejects [German | MISC] call to boycott [British | MISC] lamb.

Input: Peter Blackburn works at Reuters in Brussels.
Output: [Peter Blackburn | PER] works at [Reuters | ORG] in [Brussels | LOC].

Input: {sentence}
Output:
```

Parsing code:
```python
import re

def parse_bracketed_ner(text: str) -> list[dict]:
    """Parse inline bracketed NER output into structured entities."""
    entities = []
    for match in re.finditer(r'\[([^|\]]+)\|\s*([A-Z]+)\]', text):
        entities.append({
            "text": match.group(1).strip(),
            "label": match.group(2).strip(),
        })
    return entities

# Example:
output = "[EU | ORG] rejects [German | MISC] call to boycott [British | MISC] lamb."
print(parse_bracketed_ner(output))
# [{"text": "EU", "label": "ORG"}, {"text": "German", "label": "MISC"}, {"text": "British", "label": "MISC"}]
```

**Example 2: Nested NER for biomedical text using XML format**

User: "I need to extract protein and DNA entities from biomedical abstracts. Some DNA mentions contain protein names inside them."

Approach:
1. Choose XML format (nested entities required)
2. Define biomedical entity types
3. Build prompt with nesting examples

Output prompt template:
```
Extract all biomedical named entities from the input sentence. Mark entities
using XML tags: <LABEL>entity text</LABEL>. Entities may be nested --
place inner entity tags inside outer entity tags.

Entity types:
- Protein: Named proteins, transcription factors, enzymes, and protein families.
- DNA: Named DNA sequences, genes, promoters, and binding sites.
- RNA: Named RNA molecules including mRNA, tRNA, and regulatory RNA.
- Cell_line: Specific cultured cell lines used in experiments.
- Cell_type: General categories of cells (e.g., T cells, macrophages).

Input: PU.1 binds to a myeloid PU.1 binding site in the M-CSF receptor promoter.
Output: <Protein>PU.1</Protein> binds to a <DNA>myeloid <Protein>PU.1</Protein> binding site</DNA> in the <DNA>M-CSF receptor promoter</DNA>.

Input: {sentence}
Output:
```

Parsing code:
```python
import re
from dataclasses import dataclass

@dataclass
class Entity:
    text: str
    label: str
    start: int
    end: int

LABELS = {"Protein", "DNA", "RNA", "Cell_line", "Cell_type"}

def parse_xml_ner(tagged: str) -> list[Entity]:
    """Parse XML-tagged NER output, handling nested entities."""
    entities = []
    tag_pattern = re.compile(r'<(/?)(' + '|'.join(LABELS) + r')>')

    # Strip tags to get plain text, tracking entity spans
    plain_chars = []
    tag_stack = []  # (label, start_in_plain)

    i = 0
    while i < len(tagged):
        m = tag_pattern.match(tagged, i)
        if m:
            is_close = m.group(1) == '/'
            label = m.group(2)
            if not is_close:
                tag_stack.append((label, len(plain_chars)))
            else:
                if tag_stack and tag_stack[-1][0] == label:
                    lbl, start = tag_stack.pop()
                    text = ''.join(plain_chars[start:])
                    entities.append(Entity(text=text, label=lbl,
                                           start=start, end=len(plain_chars)))
            i = m.end()
        else:
            plain_chars.append(tagged[i])
            i += 1

    return entities
```

**Example 3: Category-grouped JSON for structured pipelines**

User: "I want entity extraction that returns clean JSON I can feed into a database, not inline markup."

Approach:
1. Use category-grouped JSON format (second-best performance, clean structure)
2. Construct prompt requiring JSON output

Output prompt template:
```
Extract all named entities from the input sentence. Return a JSON object
where keys are entity type labels and values are lists of entity text strings.
Include only labels that have at least one entity. If no entities exist,
return {}.

Entity types: PER, ORG, LOC, MISC

Input: EU rejects German call to boycott British lamb.
Output: {"ORG": ["EU"], "MISC": ["German", "British"]}

Input: The match ended in a 1-1 draw.
Output: {}

Input: {sentence}
Output:
```

Note: Category-grouped JSON loses positional information and deduplicates by default. If the same entity appears twice in a sentence, include it twice in the list.

## Best Practices

- **Do:** Include semantic label definitions in every prompt. Models perform measurably better when they understand what each label means rather than relying on the label name alone.
- **Do:** Use inline bracketed format for flat NER and XML for nested NER. These formats achieved 91-94 F1 on standard benchmarks, outperforming all JSON variants.
- **Do:** Provide 2-3 diverse few-shot examples including at least one sentence with no entities and one with multiple entity types adjacent to each other.
- **Do:** Fine-tune with LoRA at high rank (r=256) if you have labeled data. Even 2 epochs on 14K examples produces competitive results without degrading the model's general abilities.
- **Avoid:** Offset-based JSON formats that require character start/end positions. LLMs cannot reliably count characters, and this format scored under 40 F1 across all models tested.
- **Avoid:** Assuming the model will produce perfectly formatted output. Always write a robust parser with fallback handling for malformed brackets, unclosed XML tags, or invalid JSON.
- **Avoid:** Using occurrence-index JSON (tracking the Nth occurrence of an entity). This format performed 10-20 F1 points below inline formats and adds unnecessary complexity.

## Error Handling

**Malformed output:** The model may produce unclosed brackets, mismatched XML tags, or invalid JSON. Implement fallback parsing: attempt strict parsing first, then try regex-based extraction, then return an empty result with a warning.

**Wrong entity types:** The most common error (38% of mistakes). The model correctly identifies entity boundaries but assigns the wrong label. Mitigate by adding disambiguation notes to label definitions (e.g., "ORG does not include sports teams mentioned as locations").

**Boundary errors:** The model captures too much or too little of an entity span. Address by including boundary-edge examples in few-shot prompts (e.g., showing that "the United Nations" should extract as "United Nations" without the article).

**Omitted entities:** Especially common for rare entity types or entities in unusual syntactic positions. If recall is low for a specific type, add more few-shot examples featuring that type.

**Hallucinated entities:** The model invents entities not in the input text. For inline formats, verify every extracted entity string appears in the original input. For JSON formats, cross-reference extracted text against the source sentence.

## Limitations

- **Offset computation is unreliable.** If your downstream system requires exact character offsets, compute them by string-matching the extracted entity text against the original input rather than asking the model to generate positions.
- **Very long documents** exceed context windows and degrade extraction quality. Chunk text into sentence-level or paragraph-level segments before extraction. Reunify entity spans afterward.
- **Rare or novel entity types** with insufficient label definitions will underperform. The model relies on understanding the semantic meaning of each label, not memorization of entity-label pairs.
- **Performance scales with model size.** 1-2B parameter models score 5-10 F1 points below 7-8B models. For production systems, 7B+ models are recommended.
- **Zero-shot generative NER** still underperforms fine-tuned encoder models by 5-15 F1 points on most benchmarks. Fine-tuning (even parameter-efficient) is needed to close the gap.
- **Languages other than English** were not evaluated in the study. The format recommendations (inline bracketed, XML) likely transfer, but performance numbers may differ.

## Reference

[Assessment of Generative Named Entity Recognition in the Era of Large Language Models](https://arxiv.org/abs/2601.17898v1) -- Systematic evaluation of output formats, prompt structures, and LoRA fine-tuning for converting open-source LLMs into competitive NER systems. Code: [github.com/szu-tera/LLMs4NER](https://github.com/szu-tera/LLMs4NER).