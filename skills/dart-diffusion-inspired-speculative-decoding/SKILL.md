---
name: "dart-diffusion-inspired-speculative-decoding"
description: "Set up and use DART (Diffusion-Inspired Speculative Decoding) for fast LLM inference. DART replaces autoregressive draft models with parallel masked-position prediction using a single transformer layer, combined with N-gram-enforced tree pruning. Triggers: 'speed up LLM inference with DART', 'set up speculative decoding with DART', 'integrate DART for faster generation', 'configure DART draft model', 'compare DART vs EAGLE3 speculative decoding', 'optimize LLM serving latency with parallel drafting'."
---

DART (Diffusion-Inspired Speculative Decoding) enables Claude to help users set up, configure, and integrate a speculative decoding framework that achieves 2x-3.4x wall-clock speedup over standard autoregressive LLM inference. Unlike EAGLE3, which drafts tokens autoregressively (creating a bottleneck in the draft stage itself), DART predicts logits for multiple future masked positions in parallel through a single forward pass of a lightweight transformer layer, then constructs high-quality draft token trees using an N-gram-enforced pruning algorithm implemented in C++.

## When to Use

- When the user wants to accelerate inference for Qwen3-series models (1.7B through 32B) using speculative decoding
- When the user asks to set up DART from the `fvliang/DART` GitHub repository, including installation, model download, and configuration
- When the user wants to compare DART against EAGLE2/EAGLE3 baselines on their workloads
- When the user needs to integrate DART's Python API into an existing serving pipeline or Gradio demo
- When the user is building a speculative decoding system and wants to understand the parallel drafting + tree pruning approach
- When the user asks about optimizing LLM inference latency and speculative decoding is a viable strategy
- When the user wants to train or fine-tune a DART draft model head for a new base model

## Key Technique

**The Drafting Bottleneck Problem.** Standard speculative decoding uses a small "draft" model to propose candidate tokens, which the larger "target" model then verifies in parallel. Methods like EAGLE3 improve draft accuracy but still generate candidates autoregressively -- each draft token depends on the previous one, requiring multiple sequential forward passes through the draft model. This makes the drafting stage itself a latency bottleneck, especially as draft sequence length grows.

**DART's Parallel Drafting.** Inspired by diffusion-based language models (dLLMs), DART eliminates autoregressive rollouts in the draft model entirely. It takes hidden states from an intermediate layer of the target model and feeds them into a single lightweight transformer layer that predicts logits for multiple future token positions simultaneously in one forward pass. This is analogous to how diffusion models denoise multiple positions in parallel -- the draft model "fills in" masked future positions all at once rather than left-to-right. The result is dramatically lower drafting latency: one forward pass through one transformer layer versus N sequential passes through a multi-layer draft model.

**N-gram-Enforced Tree Pruning.** Raw parallel predictions lack the sequential coherence of autoregressive generation. DART compensates with a C++-implemented tree pruning algorithm that takes the top-K candidates at each position and filters them using N-gram frequency statistics from a prebuilt N-gram model. This enforces semantic continuity -- candidate branches that form implausible N-gram sequences are pruned, producing compact, high-quality draft trees that the target model can verify efficiently. The combination of cheap parallel drafting + smart tree construction yields 30% higher throughput than EAGLE3 on average, with up to 65% improvement on code-centric workloads.

## Step-by-Step Workflow

1. **Install DART and dependencies.** Clone the repository and use `uv` for dependency management:
   ```bash
   git clone https://github.com/fvliang/DART.git
   cd DART
   curl -LsSf https://astral.sh/uv/install.sh | sh
   uv sync
   uv pip install -e .
   ```

2. **Select the appropriate model trio.** DART requires three components: a base model, a DART draft head, and an N-gram model. Match the DART head to the base model size:
   | Base Model | DART Head | N-gram Model |
   |---|---|---|
   | `Qwen/Qwen3-1.7B` | `fvliang/qwen1.7b-dart` | `fvliang/dart-qwen3-ngram` |
   | `Qwen/Qwen3-4B` | `fvliang/qwen4b-dart` | `fvliang/dart-qwen3-ngram` |
   | `Qwen/Qwen3-8B` | `fvliang/qwen8b-dart` | `fvliang/dart-qwen3-ngram` |
   | `Qwen/Qwen3-14B` | `fvliang/qwen14b-dart` | `fvliang/dart-qwen3-ngram` |
   | `Qwen/Qwen3-32B` | `fvliang/qwen32b-dart` | `fvliang/dart-qwen3-ngram` |

3. **Load the model in Python.** Use `DartModel.from_pretrained` with all three paths:
   ```python
   import torch
   from dart import DartModel

   model = DartModel.from_pretrained(
       base_model_name_or_path="Qwen/Qwen3-8B",
       dart_model_name_or_path="fvliang/qwen8b-dart",
       ngram_model_name_or_path="fvliang/dart-qwen3-ngram",
       torch_dtype=torch.float16,
       device_map="auto",
       is_small_ngram=False  # True for faster loading during testing
   )
   ```

4. **Format the input using the chat template registry.** DART uses `TEMPLATE_REGISTRY` for prompt formatting:
   ```python
   from dart import TEMPLATE_REGISTRY
   template = TEMPLATE_REGISTRY["qwen"]
   prompt = template.format(messages=[{"role": "user", "content": "Explain speculative decoding."}])
   ```

5. **Run inference with `dart_generate`.** Configure sampling parameters and generation limits:
   ```python
   from dart import dart_generate

   output = dart_generate(
       model,
       prompt=prompt,
       temperature=0.7,
       top_p=0.9,
       top_k=50,
       max_new_token_num=512,
       max_length=2048,
   )
   ```

6. **Launch a Gradio demo for interactive testing.** Use the provided shell scripts or the direct CLI:
   ```bash
   uv run python dart/app/app.py \
       --base-model-name-or-path Qwen/Qwen3-4B \
       --dart-model-name-or-path fvliang/qwen4b-dart \
       --ngram-model-name-or-path fvliang/dart-qwen3-ngram \
       --device cuda \
       --max-new-tokens 2048 \
       --server-port 30000
   ```

7. **Benchmark against EAGLE3.** Add the `--compare-eagle3` flag to the Gradio app to run side-by-side comparisons on the same prompts.

8. **Tune tree construction parameters.** Adjust the top-K candidate count and N-gram pruning aggressiveness based on your latency/accuracy tradeoff. Larger K gives more candidates but bigger verification trees; stricter N-gram filtering gives smaller trees but may miss valid continuations.

9. **Use `--use-small-ngram` for rapid prototyping.** This flag loads a reduced N-gram model that is faster to initialize, useful during development before running full benchmarks.

10. **Profile end-to-end latency.** Measure three components separately: (a) target model forward pass, (b) DART draft head forward pass, and (c) tree construction + verification. DART's advantage is specifically in reducing component (b) to a single pass.

## Concrete Examples

**Example 1: Setting up DART for a Qwen3-8B deployment**

User: "I want to speed up my Qwen3-8B inference using DART speculative decoding. Help me set it up."

Approach:
1. Clone the DART repo and install dependencies with `uv sync && uv pip install -e .`
2. Verify CUDA availability and GPU memory (8B model needs ~16GB in fp16, plus overhead for DART head)
3. Write a Python script that loads `DartModel.from_pretrained` with `Qwen/Qwen3-8B`, `fvliang/qwen8b-dart`, and `fvliang/dart-qwen3-ngram`
4. Format a test prompt using the Qwen chat template
5. Call `dart_generate` with standard sampling parameters
6. Print both the output text and the generation speed (tokens/sec)

Output:
```python
import torch, time
from dart import DartModel, dart_generate, TEMPLATE_REGISTRY

model = DartModel.from_pretrained(
    base_model_name_or_path="Qwen/Qwen3-8B",
    dart_model_name_or_path="fvliang/qwen8b-dart",
    ngram_model_name_or_path="fvliang/dart-qwen3-ngram",
    torch_dtype=torch.float16,
    device_map="auto",
    is_small_ngram=False,
)

template = TEMPLATE_REGISTRY["qwen"]
prompt = template.format(messages=[{"role": "user", "content": "Write a Python quicksort."}])

start = time.time()
output = dart_generate(model, prompt=prompt, temperature=0.0, top_k=1, max_new_token_num=256, max_length=2048)
elapsed = time.time() - start
print(f"Generated {len(output.tokens)} tokens in {elapsed:.2f}s ({len(output.tokens)/elapsed:.1f} tok/s)")
print(output.text)
```

**Example 2: Launching a Gradio demo comparing DART vs EAGLE3**

User: "I want to visually compare DART and EAGLE3 speed on my prompts."

Approach:
1. Confirm the DART repo is installed and models are downloaded
2. Launch the Gradio app with the `--compare-eagle3` flag
3. Provide the user with the URL to access the web interface

Output:
```bash
cd DART
uv run python dart/app/app.py \
    --base-model-name-or-path Qwen/Qwen3-4B \
    --dart-model-name-or-path fvliang/qwen4b-dart \
    --ngram-model-name-or-path fvliang/dart-qwen3-ngram \
    --device cuda \
    --max-new-tokens 2048 \
    --compare-eagle3 \
    --server-port 30000
# Open http://localhost:30000 to see side-by-side generation
```

**Example 3: Integrating DART into an existing serving script**

User: "I have a batch inference script that processes prompts from a JSONL file. How do I swap in DART?"

Approach:
1. Replace the standard `model.generate()` call with `dart_generate()`
2. Load the DART model trio instead of the plain base model
3. Keep the existing prompt formatting and output handling
4. Add `is_small_ngram=True` for initial testing, switch to `False` for production

Output:
```python
import json, torch
from dart import DartModel, dart_generate, TEMPLATE_REGISTRY

model = DartModel.from_pretrained(
    base_model_name_or_path="Qwen/Qwen3-8B",
    dart_model_name_or_path="fvliang/qwen8b-dart",
    ngram_model_name_or_path="fvliang/dart-qwen3-ngram",
    torch_dtype=torch.float16,
    device_map="auto",
    is_small_ngram=False,
)
template = TEMPLATE_REGISTRY["qwen"]

with open("prompts.jsonl") as f:
    prompts = [json.loads(line) for line in f]

results = []
for item in prompts:
    formatted = template.format(messages=[{"role": "user", "content": item["prompt"]}])
    output = dart_generate(
        model, prompt=formatted,
        temperature=item.get("temperature", 0.7),
        top_p=0.9, top_k=50,
        max_new_token_num=item.get("max_tokens", 512),
        max_length=2048,
    )
    results.append({"prompt": item["prompt"], "response": output.text})

with open("results.jsonl", "w") as f:
    for r in results:
        f.write(json.dumps(r) + "\n")
```

## Best Practices

- **Do:** Use `torch.float16` (or `bfloat16` on Ampere+ GPUs) for both the base model and DART head to minimize memory and maximize throughput.
- **Do:** Start with `is_small_ngram=True` during development to speed up model loading, then switch to the full N-gram model for production benchmarks.
- **Do:** Set `temperature=0.0` and `top_k=1` when benchmarking raw speedup, to isolate the speculative decoding performance from sampling variance.
- **Do:** Profile the three stages separately (target forward, draft forward, tree construction) to identify where your specific bottleneck lies.
- **Avoid:** Using DART with models other than the supported Qwen3 series without first training a compatible DART draft head -- the hidden state dimensions and layer semantics must match.
- **Avoid:** Setting excessively large `max_new_token_num` without also increasing `max_length`, as the tree verification requires buffer space beyond the raw token count.
- **Avoid:** Expecting identical output to vanilla autoregressive decoding when using non-zero temperature -- speculative decoding is lossless only with greedy sampling; stochastic sampling introduces minor distribution shifts depending on the verification scheme.

## Error Handling

| Problem | Cause | Fix |
|---|---|---|
| `RuntimeError: CUDA out of memory` | Base model + DART head + N-gram model exceed GPU VRAM | Use a smaller base model, enable `is_small_ngram=True`, or use `device_map="auto"` for multi-GPU sharding |
| Draft head path not found | Mismatched DART head for the chosen base model | Verify you are using the correct `fvliang/qwen{size}b-dart` matching your `Qwen/Qwen3-{size}B` |
| Slow tree construction | C++ tree search extension not compiled | Run `uv sync` again to ensure native extensions are built; check that a C++ compiler is available |
| Low acceptance rate (tau) | N-gram model not loaded or `is_small_ngram=True` in production | Switch to the full N-gram model (`is_small_ngram=False`) for higher-quality tree pruning |
| No speedup over vanilla | Very short outputs (< 20 tokens) | Speculative decoding overhead is amortized over longer sequences; test with `max_new_token_num >= 128` |
| Template formatting errors | Wrong template name for model family | Use `TEMPLATE_REGISTRY["qwen"]` for all Qwen3 models |

## Limitations

- **Model support is currently limited to the Qwen3 family.** Using DART with LLaMA, Mistral, or other architectures requires training a new draft head on that model's hidden states.
- **Single-request latency optimization only.** DART does not address batched/continuous batching scenarios directly -- its speedup is per-request. Integration with vLLM or TGI would require custom verification kernels.
- **The N-gram model adds memory overhead.** The full N-gram model can consume several GB of RAM; on memory-constrained systems, `is_small_ngram=True` is necessary but reduces acceptance rates.
- **Greedy-only losslessness.** Like all speculative decoding methods, DART guarantees identical output to the target model only under greedy decoding. With temperature > 0, the token distribution may differ slightly.
- **No training pipeline is included in the public repo.** If you need to train a DART head for a new base model, you'll need to implement the training loop (supervised prediction of future tokens from intermediate hidden states) yourself.

## Reference

**Paper:** [DART: Diffusion-Inspired Speculative Decoding for Fast LLM Inference](https://arxiv.org/abs/2601.19278v1) (Liu et al., 2026) -- Focus on Section 3 (method) for the parallel drafting mechanism and Section 3.3 for the N-gram-enforced tree pruning algorithm.

**Code:** [https://github.com/fvliang/DART](https://github.com/fvliang/DART) -- Apache 2.0 licensed, supports Qwen3 1.7B through 32B with pre-trained draft heads.