---
name: "calliope-tts-based-narrated-e-book"
description: "Build offline TTS-narrated e-books with exact audio-text synchronization in EPUB 3 Media Overlay format. Use when the user asks to 'create a narrated ebook', 'add TTS audio to an epub', 'build an audiobook with text highlighting', 'synchronize speech with ebook text', 'convert epub to read-aloud format', or 'generate media overlays for epub'."
---

# Calliope: TTS-Based Narrated E-book Creation with Exact Synchronization

This skill enables Claude to help users build offline pipelines that convert standard text EPUB files into narrated e-books with synchronized text highlighting. The core technique, from the Calliope framework (arXiv:2602.10735), captures audio timestamps *during* TTS synthesis rather than aligning them after the fact, achieving zero drift between spoken audio and on-screen text highlighting. This eliminates the synchronization errors (up to 30+ seconds of drift) that plague forced-alignment approaches like Afaligner or Whisper-based tools.

## When to Use

- When the user wants to add narration to an existing EPUB file with word/sentence highlighting
- When building a read-aloud e-book pipeline that must work entirely offline (no cloud APIs)
- When the user needs to preserve the publisher's original CSS typography, layout, and embedded media while adding audio
- When generating EPUB 3 Media Overlay SMIL files that link text spans to audio clip timestamps
- When the user asks about synchronizing TTS output with source text and wants to avoid forced alignment drift
- When building accessible reading tools for early literacy or readers with dyslexia
- When the user wants to clone a narrator voice from a short WAV sample and apply it to an entire book

## Key Technique

**Deterministic timestamp capture during synthesis, not post-hoc alignment.** Most narrated e-book tools generate audio first, then use forced alignment (Dynamic Time Warping or Whisper transcription) to match audio segments back to source text. This introduces cumulative drift: experiments show Storyteller (Whisper-based) drifts over 30 seconds on a short story, and even the best aligner (Syncabook) exceeds the 50ms perceptual lag threshold for over 30% of sentences. Humans tolerate text appearing up to 150ms early but become sensitive to lags beyond just 50ms.

Calliope sidesteps this entirely. Each sentence is synthesized individually through the TTS engine, and its exact duration is recorded at generation time. Timestamps are then computed deterministically: a sentence's start time equals the cumulative duration of all preceding sentences (plus silence padding), and its end time extends to the next sentence's start. This "gapless" contiguity prevents highlight flicker during inter-sentence pauses. The per-sentence waveforms receive a ~50ms fade-out to eliminate boundary click artifacts, and configurable silence padding (default 0.15s) is inserted between sentences.

**EPUB 3 Media Overlay packaging.** The pipeline wraps each sentence in a `<span>` with a unique ID, generates per-chapter SMIL files mapping those IDs to `clipBegin`/`clipEnd` timestamps, updates the OPF manifest with `media-overlay` attributes and `media:duration` metadata, and injects CSS for `media:active-class` highlighting that adapts to light/dark mode via `prefers-color-scheme` media queries. The result is a standards-compliant EPUB 3 readable in Thorium Reader, BookFusion, and similar apps.

## Step-by-Step Workflow

1. **Parse the EPUB container.** Use EbookLib to extract the OPF package document, read the spine order, and enumerate all XHTML content documents and their linked CSS stylesheets.

2. **Traverse each XHTML chapter's DOM.** Using BeautifulSoup4, walk block-level elements (paragraphs, headers) to extract narrative text. Skip non-narrative elements like image captions, tables, and metadata blocks.

3. **Segment text into sentences.** Apply Unicode canonicalization (NFC), standardize punctuation (curly quotes to straight, em-dashes to standard), then tokenize into sentences. Assign each sentence a unique span ID following the pattern `calliope-s{chapter}-{index}`.

4. **Handle TTS input constraints.** Recursively split sentences exceeding 200 characters at whitespace midpoints. Merge fragments under 60 characters with adjacent sentences. Wrap a try/except around synthesis to catch token-overflow errors and trigger recursive splitting with concatenated output.

5. **Synthesize audio per sentence with timestamp capture.** Feed each sentence to the TTS engine (XTTS-v2 or Chatterbox) with the reference voice WAV. Record the exact duration of each generated waveform. Apply a 50ms fade-out filter to the tail of each waveform to eliminate boundary artifacts. Insert silence padding (0.15s default) between sentences.

6. **Compute deterministic timestamps.** For sentence `i`: `start_time = sum(durations[0..i-1] + padding[0..i-1])`, `end_time = start_time_of_sentence[i+1]`. The final sentence's end equals total chapter audio length. This gapless scheme prevents highlight flicker.

7. **Concatenate chapter audio.** Join all per-sentence waveforms (with padding) into a single WAV/MP3 per chapter. Use FFmpeg for final encoding if MP3 output is desired.

8. **Inject span tags into XHTML.** Replace each sentence's text node with a `<span id="calliope-s{ch}-{idx}">` wrapper, preserving all parent element attributes and CSS inheritance.

9. **Generate SMIL files.** For each chapter, produce a SMIL file containing `<seq>` of `<par>` elements, each with a `<text src="chapter.xhtml#calliope-s1-3"/>` and `<audio src="chapter_audio.mp3" clipBegin="12.450s" clipEnd="15.230s"/>`.

10. **Update OPF and inject CSS.** Add SMIL items to the manifest with `media-overlay` links. Set `media:duration` metadata per overlay and for the total publication. Inject active-class CSS: yellow highlight (`#fff3a8`) for light mode, muted purple (`#4a3a6b`) for dark mode, using `@media (prefers-color-scheme: dark)`.

11. **Repackage the EPUB.** Recompress all modified XHTML, new SMIL files, updated OPF, injected CSS, and audio files into a valid EPUB 3 OCF zip container.

## Concrete Examples

**Example 1: Convert a public-domain EPUB to a narrated e-book**

User: "I have `tale_of_two_cities.epub` and a 15-second WAV of a narrator voice. Help me set up a pipeline to create a narrated version with text highlighting."

Approach:
1. Set up a Python 3.11 virtual environment and install dependencies (EbookLib, BeautifulSoup4, torch, chatterbox-tts or coqui-tts, ffmpeg-python).
2. Write a script that opens the EPUB with EbookLib, iterates through spine items, parses each XHTML with BeautifulSoup.
3. Segment paragraphs into sentences, wrap in spans with unique IDs.
4. For each sentence, call the TTS model with the reference voice WAV and record the waveform duration.
5. Accumulate timestamps, generate SMIL files, update OPF, inject highlight CSS.
6. Save as `tale_of_two_cities_narrated.epub`.

Output structure:
```
tale_of_two_cities_narrated.epub
├── META-INF/container.xml
├── OEBPS/
│   ├── content.opf          # Updated with media-overlay refs
│   ├── chapter01.xhtml       # Sentences wrapped in <span> tags
│   ├── chapter01_overlay.smil # SMIL timing file
│   ├── audio/chapter01.mp3   # Synthesized narration
│   ├── styles/highlight.css  # Active-class highlight styles
│   └── ...
└── mimetype
```

**Example 2: Generate a SMIL file from pre-computed timestamps**

User: "I already have sentence-level audio durations in a JSON file. Help me generate the SMIL overlay file."

Approach:
1. Read the JSON mapping sentence IDs to durations.
2. Compute cumulative start/end times with gapless contiguity.
3. Generate SMIL XML.

Input (`durations.json`):
```json
[
  {"id": "s1-0", "text": "It was the best of times.", "duration": 2.31, "padding": 0.15},
  {"id": "s1-1", "text": "It was the worst of times.", "duration": 2.45, "padding": 0.15},
  {"id": "s1-2", "text": "It was the age of wisdom.", "duration": 2.12, "padding": 0.15}
]
```

Output (`chapter01_overlay.smil`):
```xml
<smil xmlns="http://www.w3.org/ns/SMIL" version="3.0">
  <body>
    <seq>
      <par>
        <text src="chapter01.xhtml#s1-0"/>
        <audio src="audio/chapter01.mp3" clipBegin="0.000s" clipEnd="2.460s"/>
      </par>
      <par>
        <text src="chapter01.xhtml#s1-1"/>
        <audio src="audio/chapter01.mp3" clipBegin="2.460s" clipEnd="5.060s"/>
      </par>
      <par>
        <text src="chapter01.xhtml#s1-2"/>
        <audio src="audio/chapter01.mp3" clipBegin="5.060s" clipEnd="7.330s"/>
      </par>
    </seq>
  </body>
</smil>
```

Note: each `clipEnd` extends to the next sentence's `clipBegin` (gapless), not to the raw audio end. This prevents highlight flicker during silence gaps.

**Example 3: Add highlight CSS with dark mode support**

User: "I need CSS for the media overlay active class that works in both light and dark reading modes."

Output (`highlight.css`):
```css
/* Light mode: warm yellow highlight */
.-epub-media-overlay-active {
  background-color: #fff3a8;
  color: #1a1a1a;
}

/* Dark mode: muted purple highlight */
@media (prefers-color-scheme: dark) {
  .-epub-media-overlay-active {
    background-color: #4a3a6b;
    color: #e8e8e8;
  }
}
```

The class name `.-epub-media-overlay-active` is specified via the `media:active-class` metadata property in the OPF file.

## Best Practices

- **Do:** Synthesize each sentence individually and record its exact duration. This is the core insight -- deterministic timestamps beat any post-hoc alignment.
- **Do:** Use gapless `clipEnd` boundaries (each sentence's end = next sentence's start) to prevent highlight flicker during silence padding.
- **Do:** Apply a short fade-out (~50ms) at each waveform's tail to avoid audible clicks at sentence boundaries.
- **Do:** Validate output EPUBs with EPUBCheck and test in Thorium Reader, which has strong Media Overlay support.
- **Avoid:** Using forced alignment (Afaligner, Whisper-based tools) for synchronization. Even the best aligners exceed 50ms drift on over 30% of sentences, and Whisper-based tools can drift 30+ seconds.
- **Avoid:** Word-level synchronization unless specifically required. Sentence-level sync provides a smooth reading experience without the computational overhead and fragility of word-level timing.
- **Avoid:** Sending book content to cloud TTS APIs when privacy or copyright compliance matters. The offline pipeline eliminates these concerns entirely.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| TTS token overflow | Sentence exceeds model's context window | Recursively split at whitespace midpoint, synthesize fragments, concatenate waveforms and sum durations |
| Very short fragments (<60 chars) | Over-aggressive sentence splitting | Merge with adjacent sentence before synthesis to avoid unnatural prosody breaks |
| Audio click artifacts at boundaries | Abrupt waveform termination | Apply 50ms exponential fade-out to each sentence waveform tail |
| EPUB validation failure | Missing SMIL references in OPF | Ensure every `media-overlay` attribute in the manifest points to an existing SMIL item, and all `media:duration` values are present |
| Highlight not appearing in reader | Missing or wrong active-class CSS | Verify `media:active-class` in OPF metadata matches the CSS class name exactly (including the leading dot convention) |
| Unicode text garbling | Mixed encodings in source EPUB | Apply NFC Unicode canonicalization and normalize punctuation (curly quotes, em-dashes) before sentence tokenization |

## Limitations

- **Sentence-level granularity only.** The technique synchronizes at the sentence level, not individual words. Word-level highlighting would require a fundamentally different approach (and the paper demonstrates sentence-level is sufficient for reading experience).
- **TTS quality ceiling.** Output quality is bounded by the open-source TTS model used. XTTS-v2 and Chatterbox are strong but not yet indistinguishable from professional human narrators, particularly for dialogue-heavy or emotionally varied text.
- **Voice cloning requires a clean reference.** The 10-15 second WAV sample must be clean speech without background noise or music. Poor reference audio degrades all synthesized output.
- **Processing time on CPU.** Without GPU acceleration, synthesizing a full-length novel can be very slow. GPU (`--gpu` flag) is strongly recommended for production use.
- **Language support.** Currently optimized for English. Other languages depend on the TTS model's multilingual capabilities (XTTS-v2 supports more languages than Chatterbox).
- **Complex EPUB layouts.** EPUBs with heavy JavaScript interactivity, complex table layouts, or non-standard DOM structures may require manual intervention in the text extraction step.

## Reference

- **Paper:** [Calliope: A TTS-based Narrated E-book Creator Ensuring Exact Synchronization, Privacy, and Layout Fidelity](https://arxiv.org/abs/2602.10735v1) -- Focus on Section 3 (pipeline architecture), Section 4 (timestamp computation and SMIL generation), and Section 5 (forced alignment drift experiments showing why deterministic timestamps are essential).
- **Code:** [github.com/hugohammer/TTS-Narrated-Ebook-Creator](https://github.com/hugohammer/TTS-Narrated-Ebook-Creator) -- Reference implementations for both XTTS-v2 and Chatterbox backends.