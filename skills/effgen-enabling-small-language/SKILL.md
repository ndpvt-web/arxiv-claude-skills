---
name: "effgen-enabling-small-language"
description: "Deploy and optimize small language models (SLMs) as autonomous agents using the effGen framework. Implements prompt compression (70-80% context reduction), five-factor complexity routing, intelligent task decomposition, and unified memory for local SLM-based agent systems. Triggers: 'set up effgen agent', 'deploy small language model agent', 'optimize prompts for small model', 'compress agent context for SLM', 'build local AI agent with effgen', 'route tasks by complexity for small models'"
---

This skill enables Claude to help users build, deploy, and optimize autonomous agent systems powered by small language models (1.5B-32B parameters) using the effGen framework. Rather than relying on API calls to large cloud models, effGen applies four complementary techniques — prompt compression, task decomposition, complexity-based routing, and unified memory — to make local SLMs perform agent tasks that normally require GPT/Claude-scale models. Claude can guide users through installing effGen, configuring agents with appropriate tools, writing optimized prompts that fit within tight context windows, and architecting multi-agent systems that decompose complex workflows.

## When to Use

- When the user wants to deploy an autonomous agent that runs entirely on local hardware using a small model (Qwen 1.5B-32B, Llama, Mistral, Phi)
- When the user asks to reduce token costs by moving agentic workflows from cloud LLM APIs to local SLMs
- When the user needs to compress prompts or context to fit within a small model's limited context window (2K-8K tokens)
- When the user is building a multi-step agent pipeline and needs to decide which tasks to run in parallel vs. sequentially
- When the user wants to route tasks of varying complexity to appropriately-sized models (small for simple, large for complex)
- When the user needs a privacy-preserving agent system that keeps sensitive data off external APIs
- When the user asks to set up an effGen agent with tools like web search, code execution, RAG retrieval, or file operations
- When the user wants to benchmark or compare SLM agent performance against frameworks like LangChain, AutoGen, or Smolagents

## Key Technique

effGen's core insight is that small language models fail at agent tasks not because they lack reasoning ability, but because standard frameworks waste their limited context windows on verbose prompts, redundant instructions, and tasks beyond their complexity ceiling. The framework addresses this through four interlocking techniques that have complementary scaling behavior: prompt optimization benefits smaller models more (11.2% gain at 1.5B vs. 2.4% at 32B), while complexity routing benefits larger models more (3.6% at 1.5B vs. 7.9% at 32B), meaning the combination yields consistent improvements at every scale.

**Prompt Optimization** uses a five-stage pipeline to compress context by 70-80% without losing task semantics. It replaces verbose phrases with concise equivalents (e.g., "due to the fact that" becomes "because"), eliminates redundant sentences, splits long sentences at conjunctions, converts prose to bullet-point structure, and reorganizes content into labeled sections (Task, Instructions, Output). The optimizer is model-size-aware — it categorizes models into four tiers (TINY <1B, SMALL 1-3B, MEDIUM 3-7B, LARGE 7B+) and adjusts token budgets, few-shot example counts, and compression aggressiveness per tier.

**Complexity-Based Routing** scores incoming tasks on five weighted factors: task length (15%), number of requirements (25%), domain breadth across 8 domains (20%), tool requirements across 8 categories (20%), and reasoning depth from simple lookup to multi-step synthesis (20%). Tasks scoring above a configurable threshold (default 7.0/10) get decomposed into subtasks; those below it execute directly. The router selects from five execution strategies — single agent, parallel sub-agents, sequential sub-agents, hierarchical, or hybrid — based on the dependency structure of the decomposed subtasks.

## Step-by-Step Workflow

1. **Install effGen and load a model.** Run `pip install effgen` (or `pip install effgen[vllm]` for 5-10x faster inference). Load a quantized SLM with `load_model("Qwen/Qwen2.5-1.5B-Instruct", quantization="4bit")` — 4-bit quantization is critical for fitting models into consumer GPU memory.

2. **Analyze task complexity before choosing architecture.** Use the five-factor scoring rubric: count requirements (questions, conjunctions, bullets), identify domains touched (technical, research, business, creative, data, scientific, legal, financial), check tool needs (web search, code execution, file I/O, retrieval), and assess reasoning depth (lookup/define vs. compare/synthesize/architect). Score 0-10; if above 7.0, plan for sub-agent decomposition.

3. **Compress the system prompt and task context.** Apply the five-stage optimization pipeline: (a) replace verbose phrases with concise equivalents, (b) split sentences longer than ~20 words at conjunctions, (c) remove duplicate or near-duplicate sentences, (d) convert paragraph-form instructions to bullet lists, (e) organize into labeled sections. Target 70-80% reduction. For a SMALL model (1-3B), budget roughly 512 tokens for the system prompt and cap few-shot examples at 30% of total prompt budget.

4. **Select and configure tools.** Choose from effGen's built-in tools: `Calculator`, `WebSearch` (DuckDuckGo), `PythonREPL`, `CodeExecutor` (Docker-sandboxed), `FileOps`, `Retrieval` (embedding-based RAG), `AgenticSearch` (grep-based exact match). Register them in the `AgentConfig.tools` list. For retrieval agents, prepare your knowledge base and select the embedding backend (SentenceTransformer or SimpleEmbedding) and vector store (FAISS or Chroma).

5. **Configure the agent with an optimized system prompt.** Create an `AgentConfig` with the compressed prompt, selected tools, and model. Use task-specific prompt adaptations: for coding tasks, preserve syntax structure; for reasoning tasks, inject "step by step" directives; for analysis tasks, request structured output.

6. **Decompose complex tasks into subtasks.** For tasks scoring above the complexity threshold, use the decomposition engine to split into parallel (independent) or sequential (dependent) subtasks. Validate: check for circular dependencies, balance complexity across subtasks, and run topological sort for execution order. Visualize the dependency graph before execution.

7. **Route subtasks to appropriate execution strategies.** Map each subtask to a specialization (research, coding, analysis, synthesis) and assign to sub-agents. Choose parallel execution for independent subtasks (e.g., "search for X" and "search for Y"), sequential for dependent chains (e.g., "fetch data" then "analyze data"), or hybrid for mixed dependency structures.

8. **Configure the unified memory system.** Set up short-term memory for active conversation context, long-term memory (JSON or SQLite backend) for persistent facts and session history, and vector memory (FAISS or Chroma) for semantic search over past interactions. Classify stored information by `MemoryType` and `ImportanceLevel` to manage retrieval priority.

9. **Run the agent and monitor execution.** Execute with `agent.run("task")` or use the CLI (`effgen run "task"`). For interactive sessions, use `effgen chat`. For production deployment, use `effgen serve --port 8000` to expose an API. Enable `DETAILED_LOGGING` during development to trace tool calls, decomposition decisions, and memory operations.

10. **Iterate on the complexity threshold and prompt compression.** Review the router's decision history to analyze strategy distribution and sub-agent usage rates. If simple tasks are being over-decomposed, raise the threshold above 7.0. If complex tasks fail in single-agent mode, lower it. Monitor prompt token counts and adjust compression aggressiveness per model tier.

## Concrete Examples

**Example 1: Local calculator agent with a 1.5B model**

User: "Set up a basic effGen agent that can do math and run Python code locally on my machine with a small model."

Approach:
1. Install effGen and load a quantized Qwen 1.5B model
2. Register Calculator and PythonREPL tools
3. Write a compressed system prompt optimized for SMALL tier
4. Run the agent on a sample task

Output:
```python
from effgen import Agent, load_model
from effgen.core.agent import AgentConfig
from effgen.tools.builtin import Calculator, PythonREPL

model = load_model("Qwen/Qwen2.5-1.5B-Instruct", quantization="4bit")

# Compressed system prompt for SMALL model tier (~80 tokens vs ~400 verbose)
system_prompt = """Role: math/code assistant
- Solve math with Calculator tool
- Run code with PythonREPL tool
- Return final numeric answer
- Show work in steps"""

config = AgentConfig(
    name="math_agent",
    model=model,
    tools=[Calculator(), PythonREPL()],
    system_prompt=system_prompt
)
agent = Agent(config=config)
result = agent.run("Calculate compound interest on $10000 at 5% for 3 years")
print(result)
```

**Example 2: Complexity-scored task routing for a multi-domain query**

User: "I have queries of varying difficulty — some are simple lookups, others need web research and code generation. How do I route them efficiently?"

Approach:
1. Score each incoming task on the five complexity factors
2. Route simple tasks (score < 7) to a single lightweight agent
3. Decompose complex tasks (score >= 7) into parallel/sequential subtasks
4. Assign subtasks to specialized sub-agents

Output:
```python
from effgen.core.complexity_analyzer import ComplexityAnalyzer
from effgen.core.router import SubAgentRouter

analyzer = ComplexityAnalyzer()

# Simple task — scores low, runs as single agent
simple = analyzer.analyze("What is the capital of France?")
# => ComplexityScore(overall=2.1, should_decompose=False)

# Complex task — scores high, triggers decomposition
complex_task = analyzer.analyze(
    "Research the latest advances in quantum computing, write a Python "
    "simulation of Grover's algorithm, and produce a comparison table "
    "of gate-based vs. annealing approaches with citations."
)
# => ComplexityScore(overall=8.7, should_decompose=True)
# Factors: requirements=3, domains=[technical,research,scientific],
#   tools=[web_search, code_executor], reasoning=synthesis

router = SubAgentRouter(complexity_analyzer=analyzer)
decision = router.route(complex_task)
# => Strategy: hybrid
#    Subtask 1 (parallel): "Research quantum computing advances" -> research agent
#    Subtask 2 (parallel): "Write Grover's algorithm simulation" -> coding agent
#    Subtask 3 (sequential, depends on 1): "Compare gate-based vs annealing" -> analysis agent
```

**Example 3: Prompt compression for a retrieval-augmented agent**

User: "My RAG agent's prompts are too long for my 3B model's context window. Help me compress them."

Approach:
1. Identify the model tier (SMALL: 1-3B)
2. Apply the five-stage compression pipeline to the system prompt
3. Cap few-shot examples at 30% of token budget
4. Restructure retrieved context into bullet format

Output:
```python
from effgen.prompts.optimizer import PromptOptimizer, OptimizerConfig

config = OptimizerConfig(model_size="SMALL")  # 1-3B tier
optimizer = PromptOptimizer(config=config)

verbose_prompt = """
You are a helpful research assistant. Please carefully analyze the provided
documents and answer the user's question based on the information contained
within them. If the answer is not found in the documents, please let the
user know that you could not find the relevant information. Please provide
citations for any claims you make. Due to the fact that accuracy is important,
please double-check your answers before responding.
"""

optimized = optimizer.optimize(verbose_prompt)
# Result (~75% compression):
# "Role: research assistant
# - Answer from provided docs only
# - Cite sources
# - State if info not found
# - Verify before responding"

print(f"Original: ~{len(verbose_prompt.split())} words")
print(f"Optimized: ~{len(optimized.split())} words")
# Original: ~65 words -> Optimized: ~18 words
```

## Best Practices

- **Do:** Always specify `quantization="4bit"` when loading models for consumer GPUs — this is essential for fitting 3B+ models into 8GB VRAM while maintaining adequate quality for agent tasks.
- **Do:** Structure system prompts as short bullet lists rather than prose paragraphs — SLMs parse structured formats more reliably than flowing text, and bullets compress better.
- **Do:** Use the complexity analyzer on every incoming task before choosing an execution strategy — the 7.0 threshold is a starting point; tune it based on your model's actual capability on your domain.
- **Do:** Enable Docker sandboxing (`CodeExecutor`) for any agent that runs user-provided or LLM-generated code — SLMs are more prone to generating unsafe code than large models.
- **Avoid:** Sending uncompressed prompts to models under 3B parameters — without optimization, TINY/SMALL models waste most of their context window on instruction formatting rather than task content.
- **Avoid:** Defaulting to sequential execution for all subtasks — the decomposition engine's parallelizability analysis exists for a reason; independent subtasks running in parallel significantly reduce end-to-end latency.
- **Avoid:** Using verbose few-shot examples with small models — cap examples at 30% of the prompt token budget and prefer single concise examples over multiple detailed ones for TINY/SMALL tiers.

## Error Handling

- **Context window overflow:** If the optimized prompt still exceeds the model's context limit, the `ensure_within_context()` method automatically truncates supplementary context while preserving the core task prompt. Monitor for this and consider moving to the next model tier up.
- **Decomposition produces circular dependencies:** The engine runs circular dependency detection and topological sorting. If cycles are found, it breaks them by converting the weakest dependency link to a parallel task. Review the dependency visualization output to verify.
- **Tool call failures in SLMs:** Small models generate malformed tool calls more frequently than large models. effGen includes automatic input validation and sanitization. If a tool call fails, the agent retries with a simplified prompt. Set `max_retries` in the agent config.
- **Memory backend unavailable:** If FAISS or Chroma is not installed, vector memory falls back to `SimpleEmbedding` with cosine similarity over numpy arrays. For production, install the full backend (`pip install effgen[faiss]` or `pip install effgen[chroma]`).
- **vLLM not available:** If vLLM is not installed or GPU drivers are incompatible, the framework falls back to standard HuggingFace Transformers inference. Performance drops 5-10x but functionality is preserved.

## Limitations

- Models under 1B parameters (TINY tier) still struggle with multi-tool orchestration even with full optimization — expect single-tool tasks only at this scale.
- The 70-80% prompt compression rate is measured on English-language instructions; compression ratios may differ significantly for other languages or highly technical notation.
- Complexity routing uses keyword-based heuristics for domain and tool detection, not semantic understanding — unusual task phrasings may be misrouted. Monitor and adjust the router's keyword dictionaries for your domain.
- The framework targets local deployment; if the user actually needs cloud-scale throughput (hundreds of concurrent agent sessions), a hosted large model behind an API may still be more practical.
- Unified memory with vector search requires embedding model loading, which adds ~500MB-1GB of memory overhead on top of the SLM itself.

## Reference

**Paper:** [EffGen: Enabling Small Language Models as Capable Autonomous Agents](https://arxiv.org/abs/2602.00887v1) (Srivastava et al., 2026). Focus on Section 3 for the four-technique architecture, Table 2 for the complementary scaling analysis (prompt optimization vs. complexity routing across model sizes), and the 13-benchmark evaluation in Section 5. **Code:** [github.com/ctrl-gaurav/effGen](https://github.com/ctrl-gaurav/effGen) — MIT licensed, installable via `pip install effgen`.