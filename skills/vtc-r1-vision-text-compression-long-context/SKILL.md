---
name: "vtc-r1-vision-text-compression-long-context"
description: "Implement VTC-R1 vision-text compression for efficient long-context reasoning. Renders intermediate reasoning segments into images as 'optical memory' fed back into vision-language models, achieving 3.4x token compression and 2.7x latency speedup. Use when: 'compress long reasoning chains', 'implement optical memory for VLMs', 'build iterative vision-text reasoning loop', 'reduce inference tokens with image rendering', 'VTC-R1 reasoning pipeline', 'text-to-image compression for LLM reasoning'."
---

# VTC-R1: Vision-Text Compression for Efficient Long-Context Reasoning

This skill enables Claude to implement the VTC-R1 paradigm — a technique that compresses lengthy textual reasoning traces into rendered images ("optical memory") that are iteratively fed back into vision-language models. Instead of carrying forward thousands of reasoning tokens across iterations, prior reasoning is rendered as PNG images using a text-to-image pipeline (ReportLab + pdf2image), then re-consumed by the VLM's vision encoder at dramatically lower token cost. This achieves 3.4x token compression and 2.7x end-to-end latency speedup without sacrificing — and often improving — reasoning accuracy.

## When to Use

- When building an iterative reasoning system for a VLM (e.g., Qwen-VL, GLM-4V, LLaVA) that needs to reason over many steps without hitting context limits
- When a user asks to reduce inference cost or latency for long chain-of-thought generation
- When implementing a multi-round reasoning agent that accumulates context across iterations
- When constructing training data that pairs text reasoning segments with their rendered image equivalents
- When the user wants to implement the specific VTC-R1 paper pipeline — rendering, iterative loop, completion detection, and answer extraction
- When optimizing a math/science reasoning pipeline where token budgets are a bottleneck

## Key Technique

**The core insight**: Vision encoders compress spatial information far more efficiently than text tokenizers handle raw character sequences. A 4,000-token reasoning segment rendered as a PNG image consumes only ~1,200 vision tokens when processed by a VLM's image encoder — a 3.4x compression ratio. VTC-R1 exploits this asymmetry by treating rendered images of prior reasoning as "optical memory."

**The iterative loop**: At iteration 1, the model generates a reasoning segment (up to a configurable token threshold, default 4K tokens) for a given question. This segment is rendered into one or more PNG images via a text-to-image pipeline (DejaVuSans font at 9pt, 72 DPI, 595x842 page, auto-cropped). At iteration i > 1, the model receives the original question plus all accumulated rendered images from prior iterations, and generates the next reasoning segment. The loop terminates when the model produces a `</think>` tag followed by a valid `<answer>...</answer>` block, or after a maximum of T=8 iterations.

**Why it works**: The VLM was fine-tuned to read its own rendered reasoning from images, effectively learning OCR as a byproduct of the reasoning task. Training data is constructed by segmenting solutions from OpenR1-Math-220K into 4K-token chunks, rendering earlier chunks as images, and training the model to continue reasoning given those images. The result: the model treats images of its prior work as reliable context, avoiding the quadratic attention cost of carrying forward raw text tokens.

## Step-by-Step Workflow

### 1. Set up the text-to-image rendering pipeline

Install core dependencies: `reportlab` for PDF generation, `pdf2image` + `poppler-utils` for PDF-to-PNG conversion. Create a rendering function that accepts text and a config (DPI=72, font=DejaVuSans 9pt, page=595x842, margins=10pt, auto-crop enabled).

```python
# Core rendering dependencies
# pip install reportlab pdf2image
# apt-get install poppler-utils

from reportlab.lib.pagesizes import A4
from reportlab.pdfgen import canvas
from reportlab.lib.units import inch
from pdf2image import convert_from_path
import tempfile, os

def render_text_to_images(text: str, config: dict) -> list["PIL.Image"]:
    """Render reasoning text to cropped PNG images."""
    pdf_path = tempfile.mktemp(suffix=".pdf")
    c = canvas.Canvas(pdf_path, pagesize=(config["width"], config["height"]))
    # Set font, margins, line spacing from config
    # Write text with automatic page breaks
    # ...
    c.save()
    images = convert_from_path(pdf_path, dpi=config["dpi"])
    os.unlink(pdf_path)
    return [autocrop(img) for img in images]
```

### 2. Define the rendering configuration

Use the paper's empirically validated defaults. These were selected via ablation studies across typography, layout, and resolution parameters.

```python
RENDER_CONFIG = {
    "dpi": 72,
    "width": 595,       # points (A4-ish)
    "height": 842,      # points
    "font": "DejaVuSans.ttf",  # handles math symbols
    "font_size": 9,
    "line_height": 10,
    "margin": 10,       # points, all sides
    "autocrop": True,
}
```

### 3. Implement the completion detection function

The loop terminates when the model outputs `</think>` followed by a non-empty `<answer>` block. This function must be called after every generation step.

```python
def check_completion(text: str) -> bool:
    if "</think>" not in text:
        return False
    after_think = text.split("</think>")[-1]
    if "<answer>" in after_think and "</answer>" in after_think:
        answer = after_think.split("<answer>")[1].split("</answer>")[0]
        return bool(answer.strip())
    return False
```

### 4. Construct the multimodal message format

Build the message list with a system prompt instructing continuation (not restart), the user question, and accumulated images from prior rounds.

```python
SYSTEM_PROMPT = (
    "These images record your previous reasoning process. "
    "Based on this reasoning, continue and complete the final answer. "
    "Do not restart the reasoning.\n"
    "If no images are provided, start the reasoning from scratch."
)

def build_messages(question: str, images: list) -> list:
    return [
        {"role": "system", "content": [{"type": "text", "text": SYSTEM_PROMPT}]},
        {"role": "user", "content":
            [{"type": "text", "text": question}] +
            [{"type": "image", "image": img} for img in images]
        }
    ]
```

### 5. Implement the iterative reasoning loop

This is the core VTC-R1 algorithm. Generate, check completion, render if incomplete, accumulate images, repeat.

```python
def vtc_r1_inference(question, model, processor, max_epochs=8, max_new_tokens=8192):
    accumulated_images = []
    all_outputs = []

    for epoch in range(max_epochs):
        messages = build_messages(question, accumulated_images)
        inputs = processor.apply_chat_template(
            messages, tokenize=True, add_generation_prompt=True,
            return_dict=True, return_tensors="pt"
        ).to(model.device)

        output_ids = model.generate(**inputs, max_new_tokens=max_new_tokens)
        output_text = processor.decode(
            output_ids[0][inputs["input_ids"].shape[1]:],
            skip_special_tokens=False
        )
        all_outputs.append(output_text)

        if check_completion(output_text):
            return {"answer": extract_answer(output_text), "rounds": epoch + 1}

        new_images = render_text_to_images(output_text, RENDER_CONFIG)
        accumulated_images.extend(new_images)

    return {"answer": extract_answer(all_outputs[-1]), "rounds": max_epochs}
```

### 6. Extract the final answer

Parse the answer from between `<answer>` and `</answer>` tags in the final output.

```python
def extract_answer(text: str) -> str:
    if "<answer>" in text and "</answer>" in text:
        return text.split("<answer>")[1].split("</answer>")[0].strip()
    return text  # fallback: return raw output
```

### 7. Construct training data (if fine-tuning)

Segment solutions from a reasoning dataset into chunks of ~4K tokens. For each segment boundary, render all prior segments as images. Format as ShareGPT-style conversations.

```python
def build_training_instance(question, solution, segment_size=4096, tokenizer=None):
    tokens = tokenizer.encode(solution)
    segments = [tokens[i:i+segment_size] for i in range(0, len(tokens), segment_size)]
    instances = []
    accumulated_images = []

    for i, seg_tokens in enumerate(segments):
        seg_text = tokenizer.decode(seg_tokens)
        is_last = (i == len(segments) - 1)
        instances.append({
            "messages": build_messages(question, list(accumulated_images)),
            "response": seg_text + (f"\n</think>\n<answer>{answer}</answer>" if is_last else ""),
            "images": [img.copy() for img in accumulated_images],
        })
        accumulated_images.extend(render_text_to_images(seg_text, RENDER_CONFIG))

    return instances  # yields ~len(segments) training samples per problem
```

### 8. Choose and load the base VLM

VTC-R1 was validated on Glyph (GLM-4.1V-9B-based, best accuracy) and Qwen3-VL-8B (best speedup). Use `transformers` with `AutoModelForImageTextToText`.

```python
from transformers import AutoProcessor, AutoModelForImageTextToText
import torch

model = AutoModelForImageTextToText.from_pretrained(
    "yiboowang/VTC-R1-Glyph",  # or a Qwen3-VL checkpoint
    torch_dtype=torch.bfloat16,
    device_map="auto",
)
processor = AutoProcessor.from_pretrained("yiboowang/VTC-R1-Glyph")
```

## Concrete Examples

**Example 1: Building a VTC-R1 inference pipeline from scratch**

User: "I want to implement the VTC-R1 optical memory loop for math reasoning. I have a Qwen-VL model and want to reduce token usage."

Approach:
1. Install `reportlab`, `pdf2image`, `poppler-utils`
2. Create `render_text_to_images()` with the default config (72 DPI, DejaVuSans 9pt, 595x842)
3. Implement `check_completion()` to detect `</think>` + `<answer>` termination
4. Build the iterative loop: generate -> check -> render -> accumulate -> repeat (max 8 rounds)
5. Wire up `build_messages()` to pass system prompt + question + accumulated images
6. Load the model via `AutoModelForImageTextToText` with bfloat16

Output: A working Python module with `vtc_r1_inference(question, model, processor)` that returns the answer and number of rounds used.

**Example 2: Constructing VTC-R1 training data from an existing math dataset**

User: "I have a dataset of math problems with long chain-of-thought solutions. Help me convert it to VTC-R1 format for fine-tuning."

Approach:
1. Load the dataset and tokenize each solution
2. Segment each solution into 4K-token chunks (the paper's optimal threshold)
3. For each chunk boundary, render all preceding chunks as PNG images
4. Build ShareGPT-format training instances: `(system_prompt, question, images_so_far) -> next_segment`
5. For the final segment, append `</think><answer>FINAL_ANSWER</answer>`
6. Save as JSONL with image paths referenced in `dataset_info.json` for LLaMA-Factory

Output: A dataset of ~1.7x the original instance count (since multi-segment problems produce multiple training samples), with rendered PNG images stored alongside.

**Example 3: Adding VTC-R1 to an existing reasoning agent**

User: "My reasoning agent hits the 32K context limit on hard problems. Can I use VTC-R1 to compress prior reasoning?"

Approach:
1. Identify the point in the agent loop where reasoning context accumulates beyond a threshold (e.g., 4K tokens)
2. Insert a rendering step: take the accumulated reasoning text, render it to images via the VTC-R1 pipeline
3. Replace the raw text context with the rendered images in the next model call
4. Ensure the system prompt includes the "continue, do not restart" instruction
5. Keep the iterative structure: the agent can render and compress multiple times as reasoning grows

Output: Modified agent loop where context stays under 4K text tokens + N accumulated images, with each image representing ~4K tokens of prior reasoning at ~1.2K vision token cost.

## Best Practices

- **Do**: Use DejaVuSans (or another Unicode-rich font) for rendering — math symbols, Greek letters, and special characters must render correctly or the VLM loses critical information
- **Do**: Set the segment threshold to ~4K tokens — the paper's ablation shows this outperforms 2K (too fragmented) and 6K (insufficient compression benefit)
- **Do**: Auto-crop rendered images to remove whitespace margins — this reduces vision token waste on blank regions
- **Do**: Include the explicit "do not restart" instruction in the system prompt — without it, models tend to re-derive from scratch rather than continuing
- **Avoid**: Rendering at high DPI (>100) — it increases vision token count without improving VLM comprehension, hurting the compression ratio
- **Avoid**: Skipping the image accumulation — ablations show a 11-25% accuracy drop when images from prior rounds are omitted, confirming that optical memory is essential, not decorative

## Error Handling

| Problem | Cause | Solution |
|---|---|---|
| Rendering produces blank images | Text contains only whitespace or control characters | Validate text content before rendering; skip empty segments |
| Model restarts reasoning from scratch | Missing or incorrect system prompt | Ensure the "continue, do not restart" system prompt is present and positioned correctly |
| Loop never terminates | Model fails to produce `</think><answer>` | Enforce `max_epochs` (default 8); extract best partial answer from the last output |
| Poor image quality / unreadable text | Wrong font or DPI settings | Use DejaVuSans at 9pt/72 DPI; verify font file is accessible at runtime |
| `pdf2image` fails | Missing `poppler-utils` system package | Install via `apt-get install poppler-utils` (Debian/Ubuntu) or `brew install poppler` (macOS) |
| OOM during inference | Too many accumulated images in a single forward pass | Cap accumulated images (e.g., keep only last 5 rounds); the paper shows accuracy converges by round 5 |

## Limitations

- **Requires a vision-language model**: Standard text-only LLMs cannot consume the rendered images. The technique is inherently tied to VLMs like Qwen-VL, GLM-4V, or LLaVA.
- **Fine-tuning is needed for best results**: While the rendering loop can be applied to any VLM, the paper shows significant accuracy gains only after fine-tuning the VLM on VTC-R1-formatted data. Off-the-shelf VLMs may struggle to reliably OCR dense rendered text.
- **Math/code rendering only**: The technique was validated on mathematical reasoning. Domains with heavy diagrammatic or spatial reasoning (e.g., geometry proofs needing figures) may not compress well into text-rendered images.
- **Rendering adds latency per round**: While total end-to-end latency drops (2.7x speedup due to fewer tokens processed), each individual round adds ~0.12s of rendering overhead. For very short problems, the overhead may not be worthwhile.
- **Font dependency**: The rendering quality is sensitive to font availability. Missing or substituted fonts can make mathematical notation unreadable to the VLM.

## Reference

**Paper**: [VTC-R1: Vision-Text Compression for Efficient Long-Context Reasoning](https://arxiv.org/abs/2601.22069v2) (Wang et al., 2026). Key sections: Section 3 for the iterative optical memory algorithm, Section 4.3 for ablation studies on segment size and rendering config, Appendix A for the full rendering parameter space.

**Code**: [github.com/w-yibo/VTC-R1](https://github.com/w-yibo/VTC-R1) — contains `inference.py` (iterative loop), `evaluation/word2png_function.py` (rendering pipeline), and LLaMA-Factory training configs.