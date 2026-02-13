---
name: "medspeak-knowledge-graph-aided-asr"
description: "Build knowledge-graph-aided ASR error correction pipelines for medical speech, using phonetic similarity + semantic retrieval to fix misrecognized terminology before LLM reasoning. Use when: 'correct medical ASR errors', 'fix speech recognition for clinical terms', 'build medical speech QA pipeline', 'knowledge graph for ASR correction', 'phonetic matching for medical terms', 'improve Whisper transcription of medical audio'."
---

# MedSpeak: Knowledge Graph-Aided ASR Error Correction for Medical Speech

This skill enables Claude to build ASR error correction systems that leverage a dual-representation medical knowledge graph (semantic + phonetic) to identify and fix misrecognized medical terminology in speech transcriptions. The core technique from MedSpeak (arXiv:2602.00981) combines UMLS-derived semantic relationships with Double Metaphone phonetic encoding and Levenshtein distance filtering, then feeds structured evidence into an LLM to jointly correct transcripts and answer medical questions. This approach reduced word error rate from 77.2% to 29.9% and boosted QA accuracy from 50.2% to 93.4% on medical benchmarks.

## When to Use

- When the user needs to correct medical terminology errors in ASR/Whisper transcriptions (e.g., "chorioamnionitis" misrecognized instead of "chorioretinitis")
- When building a spoken medical QA system that must handle noisy audio input
- When the user wants to construct a phonetic similarity index for domain-specific vocabulary (medical, legal, pharmaceutical)
- When integrating knowledge graphs into an NLP pipeline to ground LLM corrections in structured medical knowledge
- When the user asks to improve Whisper or other ASR model output for clinical, biomedical, or healthcare content
- When building retrieval-augmented generation (RAG) pipelines where the retrieval source is a medical knowledge graph rather than a vector store

## Key Technique

**Dual Knowledge Graph Representation.** MedSpeak constructs two complementary views from the UMLS Metathesaurus (specifically the MRREL relationship table): (1) a *semantic KG* stored in SQLite that captures medical concept relationships (classifies, constitutes, due_to, plays_role), and (2) a *phonetic KG* stored as JSONL that maps each medical term to its Double Metaphone encoding and lists phonetically similar terms within a Levenshtein distance threshold. The semantic KG tells the LLM *what concepts are medically related*; the phonetic KG tells it *what the ASR system might have confused*.

**Retrieval-then-Correct Pipeline.** Given a noisy ASR transcript, the system extracts candidate terms and queries both KG representations. Semantic retrieval pulls related medical concepts (budgeted at ~600 tokens), while phonetic retrieval identifies sound-alike terms that the ASR may have substituted (~300 tokens). These evidence snippets are injected into the LLM prompt alongside the noisy transcript and any answer options. The LLM then performs two joint tasks: (1) output a corrected transcript, and (2) select the correct answer. This joint formulation is critical -- correction without the downstream task objective produces weaker results.

**Phonetic Filtering with Double Metaphone + Levenshtein.** Raw Double Metaphone encoding produces excessive false positives (e.g., many unrelated words share the same phonetic code). MedSpeak solves this by requiring candidates to pass *both* a Double Metaphone match *and* a Levenshtein distance threshold against the CMU Pronouncing Dictionary. This two-stage filter dramatically reduces noise while preserving genuine phonetic confusables like "hypoplasia"/"hyperplasia" and "atrophy"/"hypertrophy".

## Step-by-Step Workflow

### 1. Ingest and Parse the Medical Knowledge Source

Extract medical concept relationships from UMLS MRREL (or equivalent structured source). Filter out generic relationships, duplicates, and uninformative pairs. Store as CSV with columns: `term_a`, `relationship`, `term_b`.

```python
import pandas as pd

mrrel = pd.read_csv("MRREL.RRF", sep="|", header=None,
                     usecols=[0, 3, 4, 7], names=["CUI1", "REL", "RELA", "CUI2"])
# Keep only informative relationships
keep_rels = ["classifies", "constitutes", "due_to", "plays_role",
             "has_finding_site", "has_active_ingredient"]
mrrel_filtered = mrrel[mrrel["RELA"].isin(keep_rels)].drop_duplicates()
```

### 2. Build the Semantic Knowledge Graph (SQLite)

Create a SQLite database with an `edges` table storing `(term_a, relationship, term_b)` triples. Index on both `term_a` and `term_b` for fast bidirectional lookup.

```python
import sqlite3

conn = sqlite3.connect("medical_semantic_kg.db")
conn.execute("""CREATE TABLE IF NOT EXISTS edges (
    term_a TEXT, relationship TEXT, term_b TEXT
)""")
conn.execute("CREATE INDEX IF NOT EXISTS idx_a ON edges(term_a)")
conn.execute("CREATE INDEX IF NOT EXISTS idx_b ON edges(term_b)")
# Insert filtered UMLS triples
mrrel_filtered.to_sql("edges", conn, if_exists="append", index=False)
```

### 3. Build the Phonetic Knowledge Graph (JSONL)

For each medical term, compute Double Metaphone codes and find phonetically similar terms from the CMU Pronouncing Dictionary within a Levenshtein distance threshold (typically 1-2).

```python
import json
from metaphone import doublemetaphone
from Levenshtein import distance as lev_dist

def build_phonetic_entry(term, vocabulary, max_lev=2):
    code_primary, code_secondary = doublemetaphone(term)
    candidates = []
    for candidate in vocabulary:
        c_primary, _ = doublemetaphone(candidate)
        if c_primary == code_primary and lev_dist(term.lower(), candidate.lower()) <= max_lev:
            candidates.append(candidate)
    return {"term": term, "metaphone": code_primary, "phonetic_neighbors": candidates}

# Write one entry per line
with open("medical_phonetic_kg.jsonl", "w") as f:
    for term in medical_terms:
        entry = build_phonetic_entry(term, cmu_vocabulary)
        f.write(json.dumps(entry) + "\n")
```

### 4. Transcribe Audio with Whisper (or Target ASR)

Run the ASR model to produce noisy transcriptions. Use Whisper Small or Medium for realistic error rates on medical content.

```python
import whisper

model = whisper.load_model("small")
result = model.transcribe("patient_audio.wav")
noisy_transcript = result["text"]
# Example output: "The patient was diagnosed with chorioamnionitis"
# Ground truth: "The patient was diagnosed with chorioretinitis"
```

### 5. Extract Candidate Terms from the Noisy Transcript

Tokenize the transcript and identify terms that might be medical vocabulary. Use a simple heuristic: any word longer than 6 characters or any word partially matching the KG term list. Alternatively, use a medical NER model.

### 6. Retrieve Semantic Evidence (Budget: ~600 Tokens)

For each candidate term, query the SQLite semantic KG. Collect relationship triples up to the token budget.

```python
def retrieve_semantic(term, conn, budget=600):
    rows = conn.execute(
        "SELECT term_a, relationship, term_b FROM edges WHERE term_a=? OR term_b=?",
        (term, term)
    ).fetchall()
    evidence = []
    token_count = 0
    for a, rel, b in rows:
        snippet = f"{a} --[{rel}]--> {b}"
        token_count += len(snippet.split())
        if token_count > budget:
            break
        evidence.append(snippet)
    return evidence
```

### 7. Retrieve Phonetic Evidence (Budget: ~300 Tokens)

For each candidate term, look up its phonetic neighbors from the JSONL index. Format as correction candidates.

```python
def retrieve_phonetic(term, phonetic_index, budget=300):
    entry = phonetic_index.get(term.lower())
    if not entry:
        return []
    evidence = []
    token_count = 0
    for neighbor in entry["phonetic_neighbors"]:
        snippet = f"'{term}' sounds like '{neighbor}' [phonetic]"
        token_count += len(snippet.split())
        if token_count > budget:
            break
        evidence.append(snippet)
    return evidence
```

### 8. Construct the LLM Prompt with KG Evidence

Combine the noisy transcript, semantic evidence, phonetic evidence, and answer options into a structured prompt. The prompt must instruct the LLM to output both a corrected transcript and an answer.

```
SYSTEM: You are a medical ASR error correction assistant. Use the provided
knowledge graph evidence to identify and correct misrecognized medical terms
in the transcript. Then answer the question.

SEMANTIC EVIDENCE:
{semantic_evidence_block}

PHONETIC EVIDENCE:
{phonetic_evidence_block}

NOISY TRANSCRIPT: {noisy_transcript}
QUESTION: {question}
OPTIONS: A) {opt_a}  B) {opt_b}  C) {opt_c}  D) {opt_d}

Respond in exactly this format:
Corrected Text: <corrected transcript>
Correct Option: <A|B|C|D>
```

### 9. Run LLM Inference and Parse Output

Send the prompt to a fine-tuned or instruction-following LLM. Parse the two-line output to extract the corrected transcript and the selected answer.

### 10. Evaluate with WER and QA Accuracy

Compute punctuation-insensitive Word Error Rate between the corrected transcript and the ground truth. Compute QA accuracy as the fraction of correctly answered questions.

## Concrete Examples

**Example 1: Correcting a Phonetically Confusable Medical Term**

User: "My Whisper transcription says 'The biopsy revealed hyperplasia of the tissue' but the correct term should be 'hypoplasia'. How do I build a system to catch these errors?"

Approach:
1. Build a phonetic KG entry: `{"term": "hypoplasia", "metaphone": "HPPLX", "phonetic_neighbors": ["hyperplasia", "hypoplasia", "hyperkinesia"]}`
2. When the ASR outputs "hyperplasia", phonetic retrieval flags "hypoplasia" as a neighbor
3. Semantic retrieval adds context: `hypoplasia --[plays_role]--> underdevelopment`, `hyperplasia --[plays_role]--> overgrowth`
4. The LLM sees both the phonetic candidates and semantic context, determining from the surrounding clinical context which term is correct

Output:
```
Corrected Text: The biopsy revealed hypoplasia of the tissue
Correct Option: B
Confidence: phonetic_neighbor_match + semantic_context_alignment
```

**Example 2: Building a Medical ASR Correction Pipeline from Scratch**

User: "I have 500 medical lecture recordings transcribed by Whisper. The transcriptions are full of errors on drug names and conditions. Help me build a correction pipeline."

Approach:
1. Download UMLS MRREL and MRCONSO tables; filter to drug and condition concepts
2. Build SQLite semantic KG with relationships (has_active_ingredient, treats, causes)
3. Generate phonetic KG using Double Metaphone + CMU dictionary with Levenshtein threshold of 2
4. Run Whisper on all recordings to get noisy transcripts
5. For each transcript, extract candidate medical terms (words >6 chars or partial KG matches)
6. Retrieve semantic evidence (600 token budget) and phonetic evidence (300 token budget) per candidate
7. Construct batch prompts with KG evidence, send to LLM (GPT-4 or fine-tuned Llama 3.1 8B)
8. Parse corrected transcripts; compute WER against any available ground truth

Output pipeline structure:
```
audio_files/ -> whisper_transcribe.py -> noisy_transcripts/
umls_data/  -> prepare_kg.py         -> semantic_kg.db + phonetic_kg.jsonl
noisy_transcripts/ + KGs -> retrieve_evidence.py -> evidence_cache/
evidence_cache/ -> llm_correct.py -> corrected_transcripts/
```

**Example 3: Adapting the Technique to Legal Domain ASR**

User: "I want to use this phonetic KG approach for legal transcription errors (e.g., 'tort' vs 'torte', 'lien' vs 'lean')."

Approach:
1. Replace UMLS with a legal terminology source (Black's Law Dictionary entries, legal ontology)
2. Build semantic KG: `lien --[classifies]--> security_interest`, `tort --[classifies]--> civil_wrong`
3. Build phonetic KG with identical Double Metaphone + Levenshtein pipeline
4. Adjust token budgets (legal terms are shorter than medical terms, so 400/200 may suffice)
5. Fine-tune the prompt to instruct the LLM about legal context rather than medical context
6. The same retrieval-then-correct architecture applies without modification

## Best Practices

- **Do:** Use both phonetic AND semantic evidence together. Ablation studies show removing either component degrades accuracy by 4+ percentage points. The combination is the key insight.
- **Do:** Enforce strict token budgets (600 semantic, 300 phonetic) for KG evidence. Overloading the prompt with evidence causes the LLM to lose focus on the correction task.
- **Do:** Use the Double Metaphone + Levenshtein two-stage filter. Double Metaphone alone produces too many false positives; Levenshtein alone misses phonetic patterns.
- **Do:** Train the LLM to jointly correct and answer. Joint optimization produces better corrections than correction-only training because the answer task provides a supervision signal for what matters.
- **Avoid:** Relying on embedding-based similarity alone for medical term matching. Phonetically confusable terms often have very different embeddings ("hypoplasia" vs "hyperplasia" are semantically distant but phonetically close).
- **Avoid:** Skipping the semantic KG and using only phonetic matching. Without semantic grounding, the LLM cannot disambiguate between multiple phonetic candidates that are all plausible.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Too many phonetic candidates returned | Levenshtein threshold too high | Reduce `max_lev` from 2 to 1; or add frequency-based ranking |
| SQLite query returns no semantic evidence | Term not in UMLS or spelled differently in KG | Fall back to fuzzy string matching against KG term list before querying |
| LLM output doesn't follow the two-line format | Prompt not strict enough or model too creative | Add few-shot examples to the prompt; use constrained decoding or regex parsing with fallback |
| Corrected transcript introduces new errors | LLM hallucinating medical terms not in evidence | Post-filter corrections: only accept term replacements that appear in the retrieved KG evidence |
| Token budget exceeded for long transcripts | Transcript has many candidate terms | Segment long transcripts into sentence-level chunks; process each chunk independently |

## Limitations

- **UMLS dependency:** The semantic KG requires UMLS access, which needs a (free) license from the NIH. Without it, you need an alternative medical ontology.
- **English-centric phonetics:** Double Metaphone and CMU Pronouncing Dictionary are designed for English. Non-English medical ASR requires language-specific phonetic algorithms (e.g., Pinyin for Chinese medical terms).
- **Multiple-choice assumption:** The paper's evaluation uses multiple-choice QA. Open-ended medical QA or free-form clinical note correction would need a different output formulation and evaluation approach.
- **Latency:** KG retrieval + LLM inference adds latency on top of ASR. This pipeline suits batch processing or offline correction, not real-time clinical dictation.
- **Coverage gaps:** Rare or newly coined medical terms absent from UMLS will have no KG evidence. The system degrades to baseline LLM correction for unknown terms.
- **Fine-tuning requirement for best results:** The full MedSpeak pipeline uses a fine-tuned Llama 3.1 8B. Zero-shot prompting with KG evidence still helps but yields lower accuracy than the fine-tuned version.

## Reference

**Paper:** [MedSpeak: A Knowledge Graph-Aided ASR Error Correction Framework for Spoken Medical QA](https://arxiv.org/abs/2602.00981v1) (Song et al., 2026)
**Code:** [github.com/RainieLLM/MedSpeak](https://github.com/RainieLLM/MedSpeak)
**Key insight:** The dual phonetic + semantic KG retrieval with token-budgeted evidence injection into an LLM prompt achieves 93.4% QA accuracy vs 50.2% for raw ASR, with the phonetic component specifically targeting the ASR-unique failure mode of sound-alike medical term substitution.