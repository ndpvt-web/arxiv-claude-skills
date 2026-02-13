---
name: "large-geolocation-extraction-humanitarian"
description: "Extract and geocode location mentions from humanitarian and crisis texts using a two-step LLM pipeline: few-shot NER for toponym extraction followed by agent-based geocoding for coordinate resolution. Handles ambiguous place names, distinguishes literal from associative mentions, and reduces geographic bias. Use when: 'extract locations from crisis reports', 'geocode humanitarian text', 'find place names in disaster documents', 'NER for geographic entities in unstructured text', 'resolve ambiguous toponyms from news articles', 'geolocate mentions in humanitarian datasets'."
---

# Geolocation Extraction from Humanitarian & Crisis Text

This skill enables Claude to build and apply a two-step geolocation extraction pipeline based on the framework from Cafferata et al. (2026). The technique pairs few-shot LLM-based Named Entity Recognition (NER) — which identifies place names (toponyms) in unstructured text — with an agent-based geocoding module that resolves each toponym to geographic coordinates using contextual reasoning. The approach outperforms rule-based and pretrained NER systems on both accuracy and fairness across underrepresented regions and income levels.

## When to Use

- When the user needs to extract geographic locations from humanitarian reports, situation updates, or crisis narratives
- When building a pipeline to convert unstructured text into structured geospatial data (place name, coordinates, country)
- When the user wants to geocode ambiguous toponyms that could refer to multiple places (e.g., "Springfield" in 30+ US states, or "Victoria" across continents)
- When processing documents from organizations like OCHA, UNHCR, or ACAPS and needing location-level tagging
- When the user wants to distinguish literal place references ("flooding in Beira") from associative/metonymic uses ("the Somali community," "the Geneva conventions")
- When building fairness-aware NLP systems that should perform equally well for locations in low-income countries as high-income ones

## Key Technique

**Step 1 — Few-Shot Toponym NER**: The text is chunked and sent to an LLM with a few-shot prompt that instructs it to identify literal toponyms — specific geographic places — while ignoring associative uses like demonyms ("Syrian refugees"), metonymy ("Washington announced"), or embedded references ("World Health Organization"). Two output formats are supported: **JSON format** returns a plain list of extracted names (e.g., `["Milan", "Naples"]`) allowing larger text chunks (1000-2000 chars) but requiring post-processing alignment; **Markdown format** reproduces the original text with toponyms delimited by `@@` and `##` markers, preserving exact positions but requiring smaller chunks (200-500 chars) for reliable verbatim reproduction.

**Step 2 — Agent-Based Geocoding**: Extracted toponyms are passed to a LangChain-based agent that queries the GeoNames database via the Pelias open-source geocoding service. The agent operates a tool-use loop: (1) a **Search** tool queries GeoNames with optional ISO country codes derived from document context, (2) a **Select** tool assigns a specific geoname ID with reasoning about why that candidate is correct given the surrounding text, and (3) a **Finish** tool returns all selections with explanations. This agent architecture resolves ambiguity far better than naive "pick the most populated match" heuristics — it uses document-level context (other mentioned locations, the humanitarian sector, the reporting organization) to infer the correct referent.

**Why this matters**: Rule-based geocoding systems consistently under-perform for locations in Africa, South Asia, and low-income countries because their disambiguation defaults to high-population Western cities. The agent-based approach achieves near-parity error rates across continents and income levels (false negative rate disparity ~0.25 vs ~0.40+ for rule-based systems).

## Step-by-Step Workflow

1. **Chunk the input text** using hierarchical separators: prefer double line breaks, then sentence boundaries (period + space), then single line breaks, then commas. Target 1000-2000 characters per chunk for JSON output or 200-500 characters for Markdown output.

2. **Construct the few-shot NER prompt** with 3-5 examples that demonstrate: (a) extracting literal toponyms like city/region/country names, (b) ignoring demonyms ("Afghan refugees"), (c) ignoring metonymic uses ("Brussels decided"), and (d) ignoring organization-embedded names ("Médecins Sans Frontières"). Include the instruction: "Extract only literal toponyms — specific geographic places being directly referenced."

3. **Run NER extraction** on each chunk. For JSON mode, parse the returned list. For Markdown mode, extract spans between `@@` and `##` delimiters and record their character offsets.

4. **Align JSON results back to source text** using dynamic programming to find the globally optimal order-preserving assignment of extracted names to text spans. Merge adjacent matches separated only by commas or conjunctions (e.g., "Beira, Sofala and Manica" should yield three toponyms, not one).

5. **Deduplicate and normalize** the extracted toponym list. Group identical mentions, preserve the first occurrence's position, and count frequency.

6. **Build geocoding context** from the document: extract any ISO country codes from metadata, identify the humanitarian sector (health, food security, protection, etc.), note the reporting organization, and collect the full set of extracted toponyms as mutual disambiguation signals.

7. **Run the agent-based geocoding loop** for each unique toponym: query GeoNames/Pelias with the toponym and any known country code constraint, evaluate candidate results against document context, select the best match with a confidence flag (literal vs. associative), and record the geoname ID, coordinates, country, and admin-level.

8. **Flag and filter associative mentions** the geocoding agent identifies as non-literal (demonyms that slipped through NER, metonymic uses). Mark these in the output but do not include them in the primary coordinate set.

9. **Validate output** by checking internal consistency: do all geocoded points fall within the countries/regions mentioned in the document? Are there outlier coordinates that suggest a disambiguation error?

10. **Return structured results** as a list of objects containing: `toponym`, `text_offset`, `latitude`, `longitude`, `geoname_id`, `country_iso`, `admin_level`, `is_literal`, and `confidence`.

## Concrete Examples

**Example 1: Humanitarian Situation Report**

User: "Extract and geocode all locations from this OCHA situation report excerpt."

Input text:
```
Heavy rainfallثم in Beira and Sofala province has displaced over 45,000 people
since January. The Mozambican Red Cross reports that Dondo and Nhamatanda
districts are worst affected. UNHCR is coordinating the response from Maputo,
with support from the regional office in Nairobi.
```

Approach:
1. Chunk text (fits in single chunk at ~300 chars)
2. Few-shot NER prompt extracts: `["Beira", "Sofala", "Dondo", "Nhamatanda", "Maputo", "Nairobi"]`
3. "Mozambican Red Cross" is correctly skipped (organization, not literal toponym)
4. Geocoding agent uses context (Mozambique is dominant) to resolve:
   - "Beira" -> Beira, Sofala, Mozambique (not Beira, Portugal)
   - "Dondo" -> Dondo district, Sofala, Mozambique (not Dondo, Angola)
5. "Nairobi" resolved to Kenya (regional hub context)

Output:
```json
[
  {"toponym": "Beira", "latitude": -19.8436, "longitude": 34.8389, "country_iso": "MZ", "admin_level": "city", "is_literal": true},
  {"toponym": "Sofala", "latitude": -19.0, "longitude": 34.75, "country_iso": "MZ", "admin_level": "province", "is_literal": true},
  {"toponym": "Dondo", "latitude": -19.6094, "longitude": 34.7431, "country_iso": "MZ", "admin_level": "district", "is_literal": true},
  {"toponym": "Nhamatanda", "latitude": -19.1833, "longitude": 34.75, "country_iso": "MZ", "admin_level": "district", "is_literal": true},
  {"toponym": "Maputo", "latitude": -25.9653, "longitude": 32.5892, "country_iso": "MZ", "admin_level": "city", "is_literal": true},
  {"toponym": "Nairobi", "latitude": -1.2921, "longitude": 36.8219, "country_iso": "KE", "admin_level": "city", "is_literal": true}
]
```

**Example 2: Distinguishing Literal vs. Associative Mentions**

User: "Parse locations from this protection sector report."

Input text:
```
Syrian refugees in the Bekaa Valley continue to face protection risks.
The Lebanese government has restricted access to the Turkish border area.
A Geneva-based NGO reported increased gender-based violence in Arsal.
```

Approach:
1. NER extracts candidates: `["Bekaa Valley", "Turkish", "Geneva", "Arsal"]`
2. "Syrian" is correctly excluded (demonym, not literal location)
3. Geocoding agent flags:
   - "Turkish" -> associative (adjective modifying "border area"), flagged non-literal
   - "Geneva" -> context is "Geneva-based NGO", could be literal (HQ location) or associative
4. Agent resolves "Bekaa Valley" and "Arsal" to Lebanon with high confidence

Output:
```json
[
  {"toponym": "Bekaa Valley", "latitude": 33.85, "longitude": 36.0, "country_iso": "LB", "is_literal": true, "confidence": "high"},
  {"toponym": "Turkish border area", "latitude": null, "longitude": null, "is_literal": false, "note": "Associative - adjective use, not a specific place"},
  {"toponym": "Geneva", "latitude": 46.2044, "longitude": 6.1432, "country_iso": "CH", "is_literal": true, "confidence": "medium"},
  {"toponym": "Arsal", "latitude": 34.1833, "longitude": 36.3833, "country_iso": "LB", "is_literal": true, "confidence": "high"}
]
```

**Example 3: Building the Pipeline in Python**

User: "Write me a Python script that extracts and geocodes locations from a batch of humanitarian documents."

Approach:
1. Implement chunking with hierarchical separators
2. Build the few-shot NER prompt with JSON output mode
3. Use dynamic programming alignment for span recovery
4. Implement LangChain agent with GeoNames/Pelias tools
5. Return structured GeoJSON output

Key code structure:
```python
import json
from langchain.agents import AgentExecutor, create_tool_calling_agent
from langchain.tools import tool

# Step 1: Chunk text with hierarchical separators
def chunk_text(text: str, max_chars: int = 1500) -> list[str]:
    separators = ["\n\n", ". ", "\n", ", "]
    # Split on preferred separator that yields chunks under max_chars
    ...

# Step 2: Few-shot NER prompt
NER_PROMPT = """Extract all literal toponyms (specific geographic places) from the text.
Return ONLY a JSON list of place names. Exclude:
- Demonyms (e.g., "Afghan", "Syrian")
- Metonymic uses (e.g., "Washington announced")
- Organization-embedded names (e.g., "African Union")

Examples:
Text: "Flooding in Cox's Bazar displaced Rohingya refugees from Myanmar."
Output: ["Cox's Bazar", "Myanmar"]

Text: "The Nigerian Red Cross deployed teams to Maiduguri and Borno state."
Output: ["Maiduguri", "Borno"]

Text: "{chunk}"
Output:"""

# Step 3: Geocoding agent tools
@tool
def search_geonames(query: str, country_code: str = None) -> str:
    """Search GeoNames for candidate locations matching the query."""
    ...

@tool
def select_location(toponym: str, geoname_id: int, reasoning: str) -> str:
    """Select the best GeoNames match for a toponym with reasoning."""
    ...

# Step 4: Pipeline orchestration
def extract_and_geocode(documents: list[str]) -> list[dict]:
    results = []
    for doc in documents:
        chunks = chunk_text(doc)
        toponyms = []
        for chunk in chunks:
            extracted = llm_ner(chunk)  # Few-shot NER call
            aligned = align_to_source(extracted, chunk)  # DP alignment
            toponyms.extend(aligned)
        geocoded = geocoding_agent.run(toponyms, context=doc)
        results.append(geocoded)
    return results
```

## Best Practices

- **Do**: Include 3-5 diverse few-shot examples in the NER prompt covering different geographic regions, admin levels (city/district/province/country), and edge cases (demonyms, organization names)
- **Do**: Use document-level context (all extracted toponyms, metadata country codes, humanitarian sector) as input to the geocoding agent — a toponym like "Victoria" resolves differently in an East Africa report vs. a Canada report
- **Do**: Apply dynamic programming alignment rather than greedy string matching when recovering JSON-mode toponym positions in the source text — greedy matching fails when the same name appears multiple times
- **Do**: Chunk Markdown-format prompts at 200-500 characters to ensure the LLM reproduces text verbatim; use larger chunks (1000-2000) only for JSON-format extraction
- **Avoid**: Defaulting to "largest population" disambiguation — this is the primary source of geographic bias against low-income regions. Always use contextual signals
- **Avoid**: Treating demonyms and adjective forms as literal locations. "Afghan refugees in Pakistan" contains one literal toponym (Pakistan), not two
- **Avoid**: Processing entire documents in a single LLM call — long texts cause both NER recall drops and Markdown reproduction errors

## Error Handling

| Problem | Cause | Fix |
|---------|-------|-----|
| Toponym not found in GeoNames | Misspelling, local name, or very small settlement | Retry with fuzzy matching; try alternate transliterations; fall back to parent admin region |
| Wrong country resolution | Ambiguous toponym without enough context | Inject country-code constraint from document metadata or co-occurring toponyms |
| NER returns organization names | Prompt did not sufficiently exclude embedded names | Add explicit negative examples to the few-shot prompt (e.g., "African Development Bank" -> no extraction) |
| Markdown mode corrupts text | Chunk too large for faithful reproduction | Reduce chunk size to 200-300 characters; switch to JSON mode for long documents |
| Duplicate toponyms with different coordinates | Same name at different admin levels (city vs. province) | Use admin-level hierarchy from context; keep both if genuinely distinct references |
| Geocoding agent loops | Too many candidates, no clear winner | Set max iterations (5-7); fall back to highest-population candidate with low-confidence flag |

## Limitations

- **Language coverage**: The framework is validated on English humanitarian texts. Performance on French, Spanish, Arabic, or multilingual documents is uncharacterized and likely lower for the NER step.
- **GeoNames dependency**: Geocoding quality is bounded by GeoNames coverage. Very small settlements, informal place names, and refugee camp names may not appear in the database.
- **Latency**: The two-step pipeline with agent-based geocoding is substantially slower than rule-based systems like spaCy + Nominatim. Not suitable for real-time streaming ingestion without caching and parallelization.
- **Few-shot sensitivity**: NER performance depends on example selection. Poorly chosen examples degrade recall for underrepresented regions — the exact bias the method aims to fix.
- **Cost at scale**: Processing thousands of documents with LLM API calls for both NER and geocoding is expensive. Consider fine-tuned spaCy models for high-volume steady-state use, reserving the LLM pipeline for new domains or fairness-critical applications.
- **Associative vs. literal boundary**: Some mentions are genuinely ambiguous (e.g., "Geneva-based" — is Geneva a relevant location or just an HQ reference?). The framework flags these but cannot always resolve the ambiguity without human review.

## Reference

Cafferata, G., Demarco, T., Kalimeri, K., Mejova, Y., & Beiró, M. G. (2026). *Large Language Models for Geolocation Extraction in Humanitarian Crisis Response*. arXiv:2602.08872v1. [https://arxiv.org/abs/2602.08872v1](https://arxiv.org/abs/2602.08872v1)

Key sections to consult: Section 3 for the two-step framework architecture, Section 4 for the agent-based geocoding tool design, and Section 5 for fairness evaluation methodology across geographic and socioeconomic strata.