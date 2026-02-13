---
name: "codeocr-effectiveness-vision-code"
description: "Render source code as images for vision LLM processing to reduce token cost while preserving understanding. Use when: 'render code as image for LLM', 'compress code tokens with images', 'use vision model for code understanding', 'reduce token cost for large codebase analysis', 'code image compression for clone detection', 'syntax highlighted code screenshot for VLM'."
---

# CodeOCR: Vision-Based Code Understanding with Optical Compression

This skill enables Claude to guide users in converting source code into rendered images and feeding them to Vision Language Models (VLMs) instead of raw text tokens. Based on the CodeOCR research, this approach achieves up to 8x token compression by leveraging the inherent compressibility of image representations -- adjusting image resolution reduces token consumption while preserving enough visual structure for code understanding tasks. The technique is particularly effective for clone detection, code QA, and code completion when combined with syntax highlighting.

## When to Use

- When a user needs to analyze a large codebase with a VLM and wants to reduce API token costs
- When the user asks about sending code as images to GPT-4o, Gemini, or other multimodal models
- When performing clone detection across many file pairs and text token limits are a bottleneck
- When the user wants to compare image-based vs text-based code representations for efficiency
- When building pipelines that batch-process code files through vision APIs at scale
- When the user asks how to add syntax highlighting to code images for better VLM performance
- When exploring whether a code understanding task (summarization, QA, completion) can tolerate image compression

## Key Technique

**Optical compression** replaces the text-based paradigm (code as token sequences) with an image-based one (code as rendered screenshots). Text tokens scale linearly with code length and are hard to compress without losing semantics. Images, by contrast, can be resized to lower resolutions -- the VLM's vision encoder tiles the image into fixed-size patches (typically 112px tiles, each costing ~170 tokens), so reducing resolution directly reduces token count. A 1024x1024 rendered code image might cost ~1100 tokens regardless of how many lines of code it contains, whereas the same code as text could consume 4000+ tokens.

The critical insight is that **not all code tasks degrade equally under compression**. Clone detection is highly resilient -- structural similarity is visually apparent even at low resolution, and some compressed ratios slightly outperform raw text. Code completion benefits significantly from **syntax highlighting** (VS Code-style color themes), which provides visual cues that help VLMs distinguish keywords, strings, functions, and comments even when individual characters blur. Code summarization and QA degrade more, as they require fine-grained character-level reading. The practical recommendation: use 2-4x compression with syntax highlighting for most tasks, reserve 1x (no compression) for tasks requiring exact token recovery.

The rendering pipeline uses **Pygments** for syntax tokenization and **Pillow** for image generation. Code is drawn character-by-character onto an RGB canvas with configurable font size (default 32px), DPI (300), and dimensions (1024x1024). The "modern" theme mirrors VS Code Light Modern colors (magenta keywords, teal classes, green comments, red strings). Compression is applied by resizing with LANCZOS resampling to a target resolution calculated from the desired token budget.

## Step-by-Step Workflow

1. **Install CodeOCR** from the repository:
   ```bash
   git clone https://github.com/YerbaPage/CodeOCR.git
   cd CodeOCR
   pip install -r requirements.txt
   ```
   Requires Python >= 3.10, Pillow, Pygments, and tiktoken. GPU optional (only needed for embedding models in retrieval tasks).

2. **Render code to images** using the Python API with syntax highlighting enabled:
   ```python
   from CodeOCR import render_code_to_images

   with open("target.py") as f:
       code = f.read()

   images = render_code_to_images(
       code,
       language="python",
       enable_syntax_highlight=True,
       theme="modern",        # VS Code Light Modern colors
       auto_optimize=True,    # auto-adjust font/layout for content
       width=1024,
       height=1024,
       font_size=32
   )
   images[0].save("rendered_code.png")
   ```

3. **Calculate the baseline token costs** to understand your compression ratio:
   ```python
   import tiktoken
   enc = tiktoken.encoding_for_model("gpt-4")
   text_tokens = len(enc.encode(code))
   # Image tokens: ceil(width/112) * ceil(height/112) * 170
   image_tokens = (1024 // 112 + 1) ** 2 * 170  # ~13770 for 1024x1024
   ```

4. **Apply compression** by resizing images to a target token budget:
   ```python
   from CodeOCR import compress_images

   compressed = compress_images(
       images,
       text_tokens=text_tokens,
       compression_ratio=4.0    # 4x fewer tokens than text
   )
   ```
   The compressor calculates `image_token_limit = text_tokens / ratio`, finds the closest resolution from the tile grid, and resizes with LANCZOS.

5. **Choose the compression ratio based on task type**:
   - **Clone detection**: 4-8x compression works well (structural similarity survives heavy compression)
   - **Code completion**: 2-4x with syntax highlighting enabled (color cues compensate for resolution loss)
   - **Code QA / summarization**: 1-2x maximum (requires reading specific identifiers and logic)
   - **Code search/retrieval**: 2-4x (semantic gist is preserved)

6. **Send rendered images to the VLM** with a task-appropriate prompt:
   ```python
   from CodeOCR import create_client, call_llm_with_images

   client = create_client()  # reads OPENAI_API_KEY from env
   response, token_info = call_llm_with_images(
       client,
       model_name="gpt-4o",
       images=compressed,
       system_prompt="You are a code analysis assistant.",
       user_prompt="Are these two code snippets functionally equivalent?"
   )
   print(f"Response: {response}")
   print(f"Tokens used: {token_info}")
   ```

7. **For batch processing**, render and compress in a loop, tracking token savings:
   ```python
   total_text_tokens = 0
   total_image_tokens = 0
   for filepath in code_files:
       with open(filepath) as f:
           code = f.read()
       text_toks = len(enc.encode(code))
       imgs = render_code_to_images(code, language="python", enable_syntax_highlight=True)
       compressed = compress_images(imgs, text_tokens=text_toks, compression_ratio=4.0)
       total_text_tokens += text_toks
       total_image_tokens += sum(calculate_image_tokens(img) for img in compressed)
   print(f"Compression: {total_text_tokens / total_image_tokens:.1f}x")
   ```

8. **Use the CLI for quick experiments** without writing code:
   ```bash
   # Render a file to an image
   python -m CodeOCR.demo render --file example.py -o output.png

   # Query a VLM about rendered code
   python -m CodeOCR.demo query --file example.py -i "Explain this function"

   # OCR: render then reconstruct text (test round-trip fidelity)
   python -m CodeOCR.demo ocr --file example.py
   ```

9. **Run downstream task benchmarks** to validate on standard datasets:
   ```bash
   cd downstream
   # Clone detection with image compression
   python -u run_pipeline.py --run-tasks --task code_clone_detection \
       --models gpt-4o --resize-mode --code-clone-detection-separate-mode

   # Code completion with syntax highlighting
   python -u run_pipeline.py --run-tasks --task code_completion_rag \
       --models gpt-4o --resize-mode --preserve-newlines --enable-syntax-highlight
   ```

10. **Evaluate round-trip fidelity** with code reconstruction (RQ5) to gauge how much information is lost:
    ```bash
    cd reconstruction
    python run.py  # renders -> OCR -> compares against original
    ```

## Concrete Examples

**Example 1: Reducing token cost for bulk clone detection**

User: "I have 500 pairs of code files to check for clones. The text tokens would cost too much with GPT-4o. Can I use images instead?"

Approach:
1. Render each code file as a 1024x1024 syntax-highlighted PNG using `render_code_to_images()`
2. Apply 4x compression via `compress_images()` -- clone detection tolerates this well
3. Send each pair as two images to GPT-4o with the prompt: "Are these two code snippets functionally equivalent? Answer yes or no and explain."
4. Track token savings -- expect ~4x reduction vs sending raw text

Output:
```
File pair: auth_v1.py vs auth_v2.py
Text tokens (both files): 3,200
Image tokens (4x compressed): 780
Result: "Yes, functionally equivalent. Both implement OAuth2 flow with
        identical logic; v2 renames variables and extracts a helper method."
Savings: 4.1x token reduction
```

**Example 2: Code completion with syntax highlighting boost**

User: "I want to test whether sending code as a highlighted screenshot helps GPT-4o complete a function better than plain text at the same token budget."

Approach:
1. Render the code context with `enable_syntax_highlight=True, theme="modern"` and also with `enable_syntax_highlight=False`
2. Compress both versions to 4x the text token cost
3. Send each to GPT-4o with the prompt: "Complete the function that starts on the last line"
4. Compare exact-match and BLEU scores between highlighted vs plain rendering

Output:
```
Task: Complete `def merge_sorted_lists(a, b):`
With syntax highlighting (4x compression):
  - Exact match: 42% | BLEU: 0.71
Without syntax highlighting (4x compression):
  - Exact match: 35% | BLEU: 0.63
Raw text (no compression):
  - Exact match: 45% | BLEU: 0.74

Syntax highlighting recovers most of the accuracy lost to compression.
```

**Example 3: Quick code explanation at minimal token cost**

User: "Explain this 200-line Python file but minimize API costs."

Approach:
1. Render with `render_code_to_images(code, language="python", enable_syntax_highlight=True)`
2. Apply 2x compression (moderate -- summarization needs some detail)
3. Query: "Summarize what this code does, its main classes, and key functions"
4. Compare token usage vs sending the raw text

Output:
```
Text approach: 1,450 tokens input
Image approach (2x): 725 tokens input
Response: "This module implements a rate limiter using the token bucket
algorithm. Class `RateLimiter` manages per-client buckets with
configurable refill rates. Key functions: `acquire()` blocks until
a token is available, `try_acquire()` returns immediately..."
```

## Best Practices

- **Do** enable syntax highlighting for all tasks -- it consistently improves VLM performance, especially under compression. The "modern" (VS Code) theme provides the strongest visual signal differentiation.
- **Do** use `auto_optimize=True` when rendering -- it adjusts font size and layout to fit the code naturally rather than clipping or leaving empty space.
- **Do** start with 2x compression for a new task and measure accuracy before increasing to 4x or 8x. Each task has a different compression tolerance curve.
- **Do** use LANCZOS resampling (the default) for downscaling -- it preserves more high-frequency detail (character edges) than bilinear or nearest-neighbor.
- **Avoid** compression ratios above 4x for tasks that require reading specific variable names, string literals, or numeric constants -- character-level detail is lost.
- **Avoid** rendering very long files (500+ lines) as a single image -- split across multiple pages using the multi-page support, or truncate to the relevant section. A single image with tiny text defeats the purpose.
- **Avoid** using this approach for tasks where exact character recovery is needed (e.g., generating a patch file) -- use raw text for those.

## Error Handling

- **Pygments import fails**: The renderer falls back to plain text (no highlighting). Install Pygments: `pip install Pygments`. Without it, you lose the syntax highlighting benefit.
- **tiktoken unavailable**: Token counting falls back to a ~4 chars/token heuristic. Install tiktoken for accurate compression ratio calculation: `pip install tiktoken`.
- **Image too small after compression**: At extreme ratios (8x+), the image may resize to fewer than 112px, producing only 1 tile (~170 tokens). The code handles ratio 0.0 by returning a 14x14 blank image. Check that your compressed resolution is at least 112px on each side.
- **Multi-page overflow**: Long code produces multiple images. Each image consumes its own tile budget. Account for total image tokens across all pages, not just one.
- **API rate limits**: Batch processing hundreds of image-based requests may hit rate limits faster than text -- vision requests are heavier per-call. Implement backoff and batching.
- **Font rendering differences**: Missing fonts on the system will cause Pillow to use a default bitmap font, producing lower-quality renders. Ensure a monospace TTF font is available.

## Limitations

- **Character-level precision**: VLMs cannot reliably OCR every character from compressed code images. Tasks requiring exact token recovery (code generation, patch creation) should use raw text.
- **Language sensitivity**: Results are validated primarily on Python and Java. Languages with dense syntax (Haskell, Perl) or non-Latin characters may behave differently.
- **Model dependency**: Effectiveness varies across VLMs. GPT-4o and Gemini show the strongest vision-code capabilities. Smaller or older vision models may struggle with code images entirely.
- **Long code files**: The approach works best for code segments under ~200 lines per image. Very long files require multi-page rendering, which increases total token cost and may negate compression benefits.
- **No structural encoding**: Unlike AST-based representations, images do not explicitly encode syntactic structure. The VLM must infer structure from visual layout and color cues.
- **Cost of rendering**: There is compute overhead for rendering and compressing images. For single small files, the rendering cost may exceed the token savings. The benefit materializes at scale or with large files.

## Reference

**Paper**: [CodeOCR: On the Effectiveness of Vision Language Models in Code Understanding](https://arxiv.org/abs/2602.01785v1) (Shi et al., 2026). Key sections: Section 4 for task-specific compression results, Section 5 for the syntax highlighting ablation, Table 3 for the compression-ratio-vs-accuracy tradeoffs across tasks. Code: [github.com/YerbaPage/CodeOCR](https://github.com/YerbaPage/CodeOCR).