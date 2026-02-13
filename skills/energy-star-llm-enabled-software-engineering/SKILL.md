---
name: energy-star-llm-enabled-software-engineering
description: >
  Optimize LLM-based code generation for energy efficiency by combining Retrieval-Augmented Generation (RAG) with
  Prompt Engineering Techniques (PETs). Analyzes code generation workflows and recommends model selection, prompt
  design, and retrieval strategies that reduce energy consumption while maintaining code quality.
  Trigger phrases: "energy efficient code generation", "optimize LLM energy usage", "green AI coding",
  "reduce inference cost for code", "energy star for coding tools", "sustainable code generation"
---

# Energy-Efficient LLM Code Generation with RAG + Prompt Engineering

This skill enables Claude to apply the RAG-enhanced Prompt Engineering framework from the "ENERGY STAR" paper (arXiv:2601.19260) to real coding workflows. The core insight: combining retrieval-augmented context injection with strategic prompt design lets smaller models (125M-1B parameters) match the code quality of models 3-7x their size while consuming a fraction of the energy. Claude uses this to help users choose optimal model configurations, design energy-aware prompts, build RAG pipelines for code generation, and audit existing LLM-powered coding workflows for energy waste.

## When to Use

- When the user wants to set up a local LLM for code generation and needs to choose between models of different sizes
- When building a RAG pipeline over a codebase or documentation to augment an LLM's code generation
- When designing prompts for code-generation LLMs and wanting to minimize token usage and inference cost
- When auditing an AI-powered IDE extension or CASE tool for energy efficiency
- When the user asks how to get better code output from a small model instead of scaling up to a larger one
- When comparing LLM inference costs across model architectures for a code generation use case
- When setting up CodeCarbon or similar energy monitoring for ML inference pipelines

## Key Technique

The paper's framework has three interlocking components: (1) a **RAG pipeline** that retrieves 2-3 relevant code examples from an external knowledge base using dense vector search, (2) **prompt engineering techniques** that structure the retrieved context for optimal model consumption, and (3) **real-time energy monitoring** via CodeCarbon to measure actual watts consumed during inference. The critical finding is that these components interact non-linearly -- RAG helps smaller models disproportionately more than larger ones, making the energy-quality tradeoff highly favorable at the small end of the model spectrum.

The RAG pipeline uses SentenceTransformers (`all-MiniLM-L6-v2`) to embed both the user's coding task description and a knowledge base of solved coding problems. FAISS performs cosine-similarity retrieval to find the 2-3 most relevant examples, which are injected into the prompt as concrete demonstrations. This approach reduced CodeLlama's inference time by 25% while improving code quality -- the retrieved examples act as implicit few-shot demonstrations that reduce the model's search space, leading to faster and more focused generation.

The energy efficiency ranking the paper establishes is: **GPT-2 (125M) with RAG > CodeLlama (7B) > DeepSeek Coder (6.7B) > Qwen 2.5 (7B)**. The key practitioner takeaway: GPT-2 with RAG achieved a 0.6 code quality score matching DeepSeek Coder's baseline -- at approximately 3.5x less energy. This means a well-designed RAG pipeline can substitute for billions of parameters when the task is scoped correctly.

## Step-by-Step Workflow

1. **Characterize the code generation task.** Determine the language (Python, Java, etc.), complexity (single function, class, algorithm), and whether the task is routine (CRUD, boilerplate) or novel (algorithmic, domain-specific). Routine tasks benefit most from RAG augmentation with smaller models.

2. **Select the right-sized model.** For routine code generation with a good knowledge base, start with the smallest viable model (125M-1B params). For algorithmic or novel tasks, use a 6-7B model. Only scale beyond 7B if quality metrics demand it -- each parameter tier roughly doubles energy consumption.

3. **Build the retrieval knowledge base.** Collect solved examples relevant to the target domain: project-specific code, language documentation, algorithm implementations, or curated datasets like CodeXGLUE CONCODE. Store them as text chunks (function-level granularity works well).

4. **Set up the embedding and retrieval pipeline.** Use `sentence-transformers/all-MiniLM-L6-v2` to embed the knowledge base. Index with FAISS for fast cosine-similarity search. This specific embedding model is lightweight (80MB) and performs well on code similarity tasks.

5. **Design the augmented prompt.** Retrieve 2-3 examples per query (not more -- additional examples increase prompt length and energy without proportional quality gains). Structure the prompt as: task description, then retrieved examples with clear delimiters, then the generation instruction.

6. **Instrument energy monitoring.** Integrate CodeCarbon to track per-inference energy consumption in kilowatt-hours. Log inference time alongside energy to compute efficiency ratios. This creates the feedback loop needed to optimize the pipeline.

7. **Run baseline vs. RAG-augmented comparison.** Generate code for a test set of 20-50 representative tasks both with and without RAG. Measure energy consumption, inference time, and code quality (functional correctness, BLEU/CodeBLEU score, or pass@k on unit tests).

8. **Tune the retrieval count and prompt structure.** If energy is too high, reduce retrieved examples to 1-2 or shorten them to signatures + docstrings only. If quality is too low, increase to 3 examples or add explicit chain-of-thought instructions in the prompt template.

9. **Establish an energy budget.** Set a per-inference energy ceiling (e.g., 0.001 kWh per generation). Route tasks that exceed this budget with small models to a larger model, creating a tiered system that minimizes total energy while maintaining quality.

10. **Monitor and iterate.** Track energy-per-quality-point over time. As the knowledge base grows with project-specific examples, smaller models become increasingly competitive, allowing further downsizing.

## Concrete Examples

**Example 1: Setting up a RAG-augmented code generation pipeline**

User: "I want to use a local LLM for Python code generation but keep energy costs low. Help me set this up."

Approach:
1. Recommend CodeLlama-7B as the generation model (best energy-quality tradeoff for Python)
2. Build a FAISS index over the project's existing Python codebase
3. Create a retrieval-augmented prompt template
4. Instrument with CodeCarbon

Output:
```python
# 1. Build the knowledge base index
from sentence_transformers import SentenceTransformer
import faiss
import numpy as np
import glob

# Embed existing codebase at function-level granularity
embedder = SentenceTransformer("all-MiniLM-L6-v2")

# Collect functions from project
code_chunks = []
for py_file in glob.glob("src/**/*.py", recursive=True):
    with open(py_file) as f:
        content = f.read()
    # Split by function/class definitions
    for block in content.split("\ndef "):
        if block.strip():
            code_chunks.append(f"def {block}" if not block.startswith("def") else block)

embeddings = embedder.encode(code_chunks, convert_to_numpy=True)
index = faiss.IndexFlatIP(embeddings.shape[1])
faiss.normalize_L2(embeddings)
index.add(embeddings)

# 2. Retrieval function
def retrieve_examples(query: str, k: int = 2) -> list[str]:
    query_vec = embedder.encode([query], convert_to_numpy=True)
    faiss.normalize_L2(query_vec)
    scores, indices = index.search(query_vec, k)
    return [code_chunks[i] for i in indices[0]]

# 3. Augmented prompt template
def build_prompt(task_description: str) -> str:
    examples = retrieve_examples(task_description, k=2)
    prompt = f"# Task: {task_description}\n\n"
    prompt += "# Reference examples from codebase:\n"
    for i, ex in enumerate(examples, 1):
        prompt += f"# Example {i}:\n{ex}\n\n"
    prompt += f"# Now implement: {task_description}\n"
    return prompt

# 4. Energy-monitored generation
from codecarbon import EmissionsTracker

tracker = EmissionsTracker(project_name="code-gen", measure_power_secs=1)
tracker.start()
# ... run model inference here ...
emissions = tracker.stop()
print(f"Energy consumed: {emissions:.6f} kWh")
```

**Example 2: Auditing an existing LLM coding tool for energy waste**

User: "We're running DeepSeek Coder 6.7B for our internal code assistant. It feels expensive. Can we do better?"

Approach:
1. Profile current energy consumption with CodeCarbon
2. Test if RAG-augmented smaller models can match quality
3. Recommend a tiered routing strategy

Output:
```
Audit findings based on the ENERGY STAR framework:

Current setup: DeepSeek Coder 6.7B (no RAG)
- Typical energy: ~0.003 kWh per inference
- Quality score: 0.6 (on internal benchmark)

Recommended alternative: CodeLlama 7B + RAG over your internal codebase
- Expected energy: ~0.001 kWh per inference (3x reduction)
- Expected quality: 0.55-0.65 (comparable, with retrieval augmentation)
- 25% faster inference time from reduced search space

For routine tasks (boilerplate, CRUD, test generation):
  Use GPT-2 125M + RAG -> ~0.0008 kWh (3.5x less than DeepSeek)
  Quality with RAG: 0.6 (matches DeepSeek baseline)

For complex tasks (algorithms, architecture):
  Keep DeepSeek Coder 6.7B (quality ceiling matters more)

Tiered routing saves ~60-70% total energy for typical workloads
where 70%+ of requests are routine code generation.
```

**Example 3: Designing energy-efficient prompts for code generation**

User: "I'm writing prompts for our code generation API. How do I make them energy-efficient?"

Approach:
1. Apply the RAG+PET framework principles to prompt design
2. Minimize token count while maximizing signal
3. Use retrieved examples instead of verbose instructions

Output:
```
Energy-efficient prompt design principles (from RAG+PET framework):

WASTEFUL PROMPT (high token count, vague, no examples):
"You are an expert Python programmer. Please write a well-documented,
efficient, production-ready function that takes a list of integers and
returns the two numbers that add up to a target sum. Make sure to handle
edge cases, add type hints, and include docstrings. The function should
be optimized for performance..."
-> ~80 tokens of instruction, model explores broadly, high energy

EFFICIENT PROMPT (low token count, concrete example, focused):
"# Similar solved problem:
def find_pair_with_product(nums: list[int], target: int) -> tuple[int, int]:
    seen = {}
    for n in nums:
        if target // n in seen and target % n == 0:
            return (target // n, n)
        seen[n] = True
    return (-1, -1)

# Implement: two_sum(nums: list[int], target: int) -> tuple[int, int]
def"
-> ~60 tokens, model has concrete pattern to follow, 25-40% less energy

Key rules:
1. Replace verbose instructions with 1-2 retrieved code examples
2. End the prompt mid-token (e.g., "def") to constrain generation start
3. Keep retrieved examples to signatures + core logic (trim docstrings)
4. Use 2-3 examples max -- additional examples increase energy linearly
   but quality plateaus after 3
```

## Best Practices

- **Do:** Start with the smallest model that could plausibly work, then add RAG to close quality gaps. A 125M model + RAG often matches a 7B model without RAG on routine tasks.
- **Do:** Retrieve at function-level granularity. Full-file retrieval wastes context tokens on imports, comments, and boilerplate that don't help generation.
- **Do:** Measure energy per quality point (kWh / quality_score) rather than raw energy or raw quality alone. This composite metric reveals the true efficiency frontier.
- **Do:** Use `all-MiniLM-L6-v2` for code embeddings -- it's 80MB, fast, and empirically effective for code similarity despite not being code-specific.
- **Avoid:** Retrieving more than 3 examples per query. Energy scales linearly with prompt length, but quality gains plateau after 2-3 examples.
- **Avoid:** Using RAG with already-large models (7B+) for routine tasks. The paper found DeepSeek and Qwen actually consumed *more* energy with RAG, likely because the added context lengthened inference without proportional quality gains.
- **Avoid:** Relying on CO2 emission estimates for energy auditing. Measure actual energy consumption in kWh via CodeCarbon -- CO2 conversion factors vary wildly by grid region and time of day.

## Error Handling

- **FAISS index returns irrelevant examples:** Add a similarity score threshold (e.g., cosine > 0.5). Below threshold, fall back to zero-shot generation rather than injecting misleading context. Bad retrieved examples hurt quality more than no examples.
- **CodeCarbon fails to detect hardware:** On machines without Intel RAPL or NVIDIA SMI, CodeCarbon falls back to TDP-based estimates. Flag these measurements as approximate and prefer relative comparisons (before/after RAG) over absolute values.
- **Model generates code that ignores retrieved examples:** The prompt template may not clearly delineate examples from the task. Use explicit delimiters (`### Example ###` / `### Your Task ###`) and ensure the generation instruction comes last.
- **Energy measurement variance between runs:** GPU power draw varies with thermal state and background processes. Average over 5+ runs per configuration and perform measurements on a quiet machine.

## Limitations

- The paper validates on models up to 7B parameters. The RAG+PET energy savings may not extrapolate to 70B+ models where inference dynamics differ substantially.
- Code quality was measured primarily via BLEU-style metrics, not functional correctness (pass@k). A high-BLEU but non-functional output wastes all energy spent generating it.
- The knowledge base quality is the ceiling -- RAG cannot retrieve examples that don't exist. For novel domains without existing solved examples, the framework provides less benefit.
- Energy measurements were performed on specific hardware configurations. Absolute kWh numbers will differ across GPU types, but relative rankings (small+RAG vs. large) should hold.
- The framework focuses on single-function generation. Multi-file or architecture-level code generation involves different energy profiles not covered by this approach.

## Reference

**Paper:** Thakur, H. & Moin, A. (2026). "ENERGY STAR" LLM-Enabled Software Engineering Tools. *CAIN 2026*. [arXiv:2601.19260](https://arxiv.org/abs/2601.19260v1)

Key finding to internalize: GPT-2 (125M) + RAG achieved the same code quality score (0.6) as DeepSeek Coder (6.7B) baseline while using 3.5x less energy. The optimal energy-quality configuration was CodeLlama 7B + RAG, which reduced inference time by 25% while improving quality.