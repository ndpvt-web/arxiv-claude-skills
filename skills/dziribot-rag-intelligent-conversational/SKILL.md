---
name: "dziribot-rag-intelligent-conversational"
description: "Build dialect-aware RAG conversational agents that handle non-standard orthography, code-switching, and multi-script input. Uses a dual-path architecture: deterministic NLU for structured flows + RAG fallback for open-domain queries. Trigger phrases: 'build a dialect chatbot', 'RAG agent for Arabic dialect', 'handle code-switching in chatbot', 'multi-script NLU pipeline', 'Algerian Arabic conversational agent', 'dialect-aware customer service bot'"
---

# Dialect-Aware RAG Conversational Agent (DziriBOT Architecture)

This skill teaches Claude to build hybrid conversational agents that handle **non-standardized dialects** with code-switching, multi-script input, and orthographic variation. The core architecture from DziriBOT combines a fast deterministic NLU path (Rasa DIET or fine-tuned transformer) for structured intent routing with a RAG fallback path for knowledge-intensive open-domain responses. This dual-path design achieves sub-100ms latency on structured queries while still answering novel questions grounded in enterprise documentation.

## When to Use

- When the user needs a chatbot that handles a dialect with non-standard spelling (Darja, Darija, Egyptian Arabic, Tunisian, or any low-resource vernacular)
- When building a customer service agent that must process mixed-language input (e.g., Arabic + French code-switching, Hinglish, Spanglish)
- When the user asks to combine Rasa NLU with a RAG pipeline for fallback on knowledge-intensive queries
- When designing an NLU pipeline for text written in multiple scripts (Arabic script + Latin/Arabizi transliteration)
- When the user wants to fine-tune a dialect BERT model (DziriBERT, AraBERT, CAMeLBERT) for intent classification
- When building a conversational agent where structured service flows and free-form Q&A must coexist in one system
- When preprocessing noisy user input with repeated characters, numeral-letter substitutions (3=a, 7=h, 9=q), or mixed punctuation

## Key Technique

**Dual-Path Routing Architecture.** DziriBOT routes every user message through two paths. The *deterministic path* uses a Rasa NLU pipeline (WhitespaceTokenizer + RegexFeaturizer + CountVectorsFeaturizer with character n-grams + DIET classifier) or a fine-tuned DziriBERT model to classify intents across 69 classes with sub-100ms latency. When the classifier confidence exceeds a threshold, the system triggers structured dialogue flows (forms, slot-filling, API calls). When confidence is low, the message falls through to the *dynamic RAG path*, which embeds the query with `intfloat/multilingual-e5-base`, retrieves relevant chunks from a FAISS HNSW index, re-ranks results, and generates a grounded answer via a quantized LLM (Llama-3.2-3B INT8).

**Multi-Script Preprocessing Pipeline.** The critical enabler is a normalization layer that unifies orthographic chaos before any model sees the text. For Arabic script: all Alef variants collapse to plain Alef, Ta Marbuta maps to terminal Ha, Alef Maqsura maps to Ya, Kashida decorative elongations are stripped, and diacritics are removed. For Latin/Arabizi script: phonetic numeral de-substitution (7->h, 3->a, 9->q), lowercasing, apostrophe normalization. Both scripts get repeated-character squashing ("baaaaazef" -> "bazef") and privacy masking (phone numbers -> `[PHONE]` token). This preprocessing alone accounts for significant accuracy gains, particularly on rare intents with high orthographic noise.

**Low-Resource Data Augmentation.** With 69 intent classes and a severe long-tail distribution (50% of classes have <10 examples), the paper addresses data scarcity through three augmentation strategies: manual paraphrasing (3-5 semantic variants per rare intent by native speakers), lexical synonym substitution within the dialect ("nheb" -> "bghit"/"hab"), and supervised back-translation through French with a semantic similarity threshold of >0.8 to filter bad translations. This brought the minimum class size to 13 (Arabic) and 28 (Latin) samples per intent.

## Step-by-Step Workflow

### 1. Define the Intent Taxonomy and Gather Seed Data
Enumerate all intent classes for your domain. Expect a long-tail distribution. For each intent, write at least 10-15 seed utterances covering common spelling variants, both scripts if applicable, and code-switched forms. Store as JSONL with fields: `text`, `intent`, `script` (arabic/latin), `entities`.

### 2. Build the Multi-Script Preprocessing Pipeline
Implement a normalization module with these ordered steps:
```python
import re

def normalize_arabic(text: str) -> str:
    """Normalize Arabic-script dialect text."""
    # 1. Strip diacritics (tashkeel)
    text = re.sub(r'[\u064B-\u065F\u0670]', '', text)
    # 2. Unify Alef variants -> plain Alef
    text = re.sub(r'[\u0622\u0623\u0625]', '\u0627', text)
    # 3. Alef Maqsura -> Ya
    text = text.replace('\u0649', '\u064A')
    # 4. Ta Marbuta -> Ha
    text = text.replace('\u0629', '\u0647')
    # 5. Remove Kashida (Tatweel)
    text = text.replace('\u0640', '')
    # 6. Squash repeated characters (3+ -> 1)
    text = re.sub(r'(.)\1{2,}', r'\1', text)
    return text.strip()

def normalize_arabizi(text: str) -> str:
    """Normalize Latin-script (Arabizi) dialect text."""
    text = text.lower()
    # Phonetic numeral de-substitution
    numeral_map = {'7': 'h', '3': 'a', '9': 'q', '5': 'kh', '2': 'a'}
    for num, letter in numeral_map.items():
        text = text.replace(num, letter)
    # Normalize apostrophes
    text = re.sub(r"['\u2018\u2019\u0060]", "'", text)
    # Squash repeated characters
    text = re.sub(r'(.)\1{2,}', r'\1', text)
    return text.strip()

def mask_pii(text: str) -> str:
    """Replace phone numbers with [PHONE] token."""
    text = re.sub(r'\b0[567]\d{8}\b', '[PHONE]', text)
    return text
```

### 3. Detect Script and Route to Appropriate Normalizer
Use Unicode block detection to classify input as Arabic-script or Latin-script, then apply the corresponding normalizer:
```python
def detect_script(text: str) -> str:
    arabic_chars = sum(1 for c in text if '\u0600' <= c <= '\u06FF')
    latin_chars = sum(1 for c in text if 'a' <= c.lower() <= 'z')
    return 'arabic' if arabic_chars > latin_chars else 'latin'
```

### 4. Augment Rare Intent Classes
For each intent with fewer than 30 examples: (a) write 3-5 manual paraphrases per utterance, (b) apply lexical synonym substitution using a dialect synonym dictionary, (c) optionally back-translate through a pivot language (e.g., French or MSA) and filter with cosine similarity > 0.8 against the original embedding.

### 5. Configure the Rasa NLU Deterministic Path
Set up a Rasa `config.yml` pipeline optimized for dialect:
```yaml
pipeline:
  - name: WhitespaceTokenizer     # Preserves dialect spelling artifacts
  - name: RegexFeaturizer          # Domain patterns: USSD codes, phone numbers
  - name: CountVectorsFeaturizer   # Unigram lexical features
  - name: CountVectorsFeaturizer
    analyzer: char_wb
    min_ngram: 3
    max_ngram: 4                   # Character n-grams capture subword patterns
  - name: DIETClassifier
    epochs: 100
    embedding_dimension: 128
    learning_rate: 0.001
    drop_rate: 0.2
    weight_sparsity: 0.1
    sparse_input_dropout_rate: 0.2
    number_of_transformer_layers: 1
```

### 6. (Optional) Fine-Tune a Dialect Transformer
For higher accuracy (+1-5% F1 over DIET), fine-tune a pretrained dialect BERT:
- Use DziriBERT for Algerian, AraBERT v2 for broader Arabic, CAMeLBERT for other dialects
- Append a classification head to the `[CLS]` token output (768-dim -> num_intents)
- Train with WordPiece tokenizer, max_length=128 (Arabic) or 96 (Latin)
- Use stratified train/val/test split (80/10/10) to preserve class distribution

### 7. Build the RAG Fallback Path
When NLU confidence is below threshold (e.g., < 0.65):
1. **Chunk documents** by semantic boundaries (section headers, service categories) — not fixed token windows. Aim for ~245 contextual chunks bound by topic headers.
2. **Embed with multilingual model**: Use `intfloat/multilingual-e5-base` with prefix encoding (`"query: "` for user questions, `"passage: "` for document chunks).
3. **Index in FAISS HNSW** for approximate nearest neighbor search (~200ms retrieval).
4. **Re-rank** top-k results with a cross-encoder or lightweight scorer (~15ms).
5. **Generate** with a quantized LLM (Llama-3.2-3B INT8 or similar) using a bilingual prompt template that instructs the model to answer in the user's dialect.

### 8. Implement the Dual-Path Router
```python
def route_message(text: str, nlu_result: dict, confidence_threshold: float = 0.65):
    """Route to deterministic flow or RAG fallback."""
    if nlu_result['confidence'] >= confidence_threshold:
        return {
            'path': 'deterministic',
            'intent': nlu_result['intent'],
            'entities': nlu_result['entities']
        }
    else:
        return {
            'path': 'rag',
            'query': text
        }
```

### 9. Design the RAG Prompt Template
Craft a bilingual system prompt that grounds responses in retrieved context and instructs the LLM to respond in the user's dialect:
```
You are a helpful customer service assistant. Answer ONLY based on the
provided context. If the context does not contain the answer, say you
don't know. Respond in the same language/script the user wrote in.

Context:
{retrieved_chunks}

User question: {query}
Answer:
```

### 10. Evaluate End-to-End with Script-Specific Metrics
Report accuracy, weighted F1, and macro F1 separately for each script (Arabic and Latin). Pay special attention to macro F1 — it exposes failures on rare intents that weighted F1 can mask. Target: >85% macro F1 on both scripts.

## Concrete Examples

**Example 1: Building a Telecom Customer Service Bot for Algerian Darja**

User: "I need a chatbot for an Algerian telecom company. Customers write in Arabic, French, and Arabizi. The bot should handle balance checks, offer inquiries, and complaints, but also answer general questions about promotions from our docs."

Approach:
1. Define intents: `check_balance`, `ask_offer_details`, `file_complaint`, `activate_service`, `general_promo_question`, etc.
2. Collect 15+ seed utterances per intent in both Arabic and Arabizi. Example for `check_balance`:
   - Arabic: "كيفاش نشوف الرصيد نتاعي"
   - Arabizi: "kifach nchouf rasidi"
   - Code-switched: "bghit nchouf mon solde"
3. Apply normalization: "baghiiiiit nchouuuf" -> "baghit nchouf"
4. Configure Rasa DIET pipeline with char n-grams (3-4) + regex for USSD codes like `*505#`.
5. Build RAG index from promotion PDFs, chunked by offer name (PixX, Win, Sama).
6. Route: high-confidence intents -> Rasa forms (slot-fill phone number, service name) -> API call. Low-confidence -> RAG retrieval + Llama generation.

Output: A dual-path bot where "bghit nactivi win max" triggers a structured activation flow, while "واش الفرق بين pixX و sama" retrieves promotional docs and generates a comparison answer in Darja.

**Example 2: Adapting the Architecture to Hinglish (Hindi + English)**

User: "I want to apply this dialect chatbot approach to Hinglish customer support for an Indian e-commerce company."

Approach:
1. Replace Arabic normalization with Devanagari normalization (Nukta removal, Chandrabindu unification).
2. Replace Arabizi de-substitution with Romanized Hindi handling (no numeral substitution needed, but handle "kya" vs "kia" vs "kyaa" spelling variants).
3. Script detection: Devanagari block (`\u0900-\u097F`) vs Latin.
4. Replace DziriBERT with HindiBERT or MuRIL (Google's multilingual Indian BERT) for the fine-tuned classifier.
5. Keep the same DIET pipeline structure but retrain on Hinglish data with char n-grams tuned for Hindi morphology.
6. RAG: swap `multilingual-e5-base` embeddings (already supports Hindi) and reindex product catalog docs.

Output: Same dual-path architecture, but the preprocessing, base transformer, and training data are swapped for the target dialect. The routing logic, FAISS indexing, and prompt template structure remain identical.

**Example 3: Adding RAG Fallback to an Existing Rasa Bot**

User: "I have a working Rasa chatbot but it fails on questions about our product docs. How do I add a RAG fallback?"

Approach:
1. Add a custom Rasa action `action_rag_fallback` triggered when NLU confidence < 0.65.
2. In the action, embed the user message with `multilingual-e5-base` (prefix: `"query: "`).
3. Search a pre-built FAISS index of your product documentation chunks.
4. Pass top-3 retrieved chunks + user query to a quantized LLM with the grounded prompt template.
5. Add a Rasa rule:
```yaml
rules:
  - rule: RAG fallback on low confidence
    steps:
      - intent: nlu_fallback
      - action: action_rag_fallback
```
6. Set `FallbackClassifier` threshold to 0.65 in your pipeline config.

Output: The existing bot handles known intents as before. Unknown or ambiguous queries now get grounded answers from your documentation instead of "I don't understand."

## Best Practices

- **Do:** Use character n-grams (3-4) in your featurizers — they are critical for catching spelling variants in dialects without standardized orthography. DziriBOT's DIET pipeline relies on `char_wb` n-grams to generalize across "bghit"/"beghit"/"bghiit".
- **Do:** Chunk documents by semantic boundaries (section headers, product names), not fixed token windows. DziriBOT's 245-chunk index is organized by service offer, which keeps retrieval semantically coherent.
- **Do:** Use `intfloat/multilingual-e5-base` with explicit prefix encoding (`query:` / `passage:`) for embedding — it significantly outperforms symmetric embedding models on cross-lingual retrieval.
- **Do:** Report macro F1 alongside accuracy. With 69 intents and long-tail distribution, accuracy and weighted F1 hide failures on rare classes. Macro F1 forces equal weight per intent.
- **Avoid:** Transliterating everything to one script before classification. DziriBOT trains separate pipelines per script and achieves higher accuracy than a unified transliterated model would, because transliteration introduces its own noise.
- **Avoid:** Using fixed NLU confidence thresholds without calibration. Tune the threshold on your validation set by plotting precision-recall curves for the fallback trigger. Starting at 0.65 is reasonable but domain-dependent.

## Error Handling

- **Orthographic noise causes misclassification:** If the normalizer doesn't catch a spelling variant, the intent classifier may fail. Add the failing variant to training data and augment with synonym substitution. Monitor low-confidence classifications in production logs.
- **RAG retrieves wrong chunks:** If the retriever returns irrelevant documents, check that your chunk boundaries align with semantic topics. Overlapping or overly large chunks dilute relevance. Add a cross-encoder re-ranker to filter false positives.
- **LLM hallucinates beyond retrieved context:** Constrain the generation prompt to forbid answers not grounded in context. Add a post-generation check: if the answer contains named entities not present in any retrieved chunk, replace with "I don't have that information."
- **Script detection fails on short input:** Messages under 5 characters may not have enough signal. Default to the user's previously detected script, or ask the user to clarify.
- **Cold-start on rare intents:** Intents with <10 training examples will have poor recall. Prioritize augmentation for these classes. DziriBOT's minimum viable class size was 13 (Arabic) and 28 (Latin).

## Limitations

- **Latency on CPU:** The RAG path with Llama-3.2-3B INT8 takes ~85 seconds on CPU. Production deployment requires a GPU (RTX 3060: ~12s; A100: ~3s). The deterministic NLU path is fast (50-80ms) regardless.
- **Script-specific models:** Training separate models per script doubles maintenance overhead. If your dialect has more than two script systems, this approach scales linearly in complexity.
- **Domain-specific vocabulary:** The preprocessing and augmentation are tuned for telecom Darja. Adapting to a new domain (medical, legal) requires rebuilding the synonym dictionary, regex patterns, and training data from scratch.
- **Arabizi numeral mapping is lossy:** The mapping 3->a, 7->h is one-to-many in reverse. Some ambiguity is irrecoverable, especially for short words.
- **No end-to-end joint training:** The NLU and RAG paths are independently trained and stitched together at inference. There is no gradient flow between the intent classifier and the retriever, which means retrieval quality cannot benefit from intent signals.

## Reference

**Paper:** [DziriBOT: RAG Based Intelligent Conversational Agent for Algerian Arabic Dialect](https://arxiv.org/abs/2602.02270v1) — Bechiri & Lanasri, 2026. Look for: the five-tiered pipeline architecture, DIET vs DziriBERT comparison tables (Table results showing 87.38% vs 86.98% accuracy), the Arabic/Arabizi normalization rules, and the RAG latency breakdown across hardware configurations.