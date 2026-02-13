---
name: "ecg-agent-on-device-tool-calling-agent"
description: "Build on-device LLM tool-calling agents for multi-turn biomedical signal dialogue, following the ECG-Agent architecture. Use when: 'build an ECG analysis agent', 'create a tool-calling agent for time-series signals', 'multi-turn medical dialogue system', 'on-device biomedical agent with tool use', 'PQRST interval measurement agent', 'deploy a quantized medical LLM with external tools'."
---

# ECG-Agent: On-Device Tool-Calling Agent for Multi-Turn Biomedical Signal Dialogue

This skill teaches Claude to implement the ECG-Agent architecture -- a modular tool-calling agent framework where a small, quantized LLM acts as a reasoning core that delegates specialized tasks (classification, measurement, explanation) to external tools during multi-turn conversations about biomedical signals like ECGs. The key innovation is decomposing end-to-end signal understanding into a planner LLM + discrete tool pipeline, enabling on-device deployment (1B-3B parameters at 4-bit quantization fitting in ~2GB RAM) while maintaining performance comparable to 8B-32B models. This pattern generalizes beyond ECGs to any domain where a conversational agent needs precise, tool-assisted analysis of structured signals.

## When to Use This Skill

- When the user asks to build a conversational agent that analyzes ECG signals, physiological waveforms, or similar time-series biomedical data
- When implementing a tool-calling agent architecture where a small LLM delegates to specialized analysis tools (classification, measurement, explanation)
- When designing multi-turn dialogue systems for medical or scientific domains that require both natural language fluency and precise numerical measurements
- When deploying an LLM-based agent on resource-constrained devices (phones, edge hardware) using quantization and LoRA fine-tuning
- When constructing a synthetic multi-turn dialogue dataset with controlled action sequences, language proficiency levels, and domain-specific scenarios
- When building a turn-by-turn training pipeline where each agent turn becomes a training instance with full dialogue history as context

## Key Technique

**Modular Tool-Calling Architecture**: ECG-Agent separates reasoning from execution. Instead of training a single end-to-end model to both interpret ECG waveforms and produce measurements, the system uses a fine-tuned LLM as a planning/reasoning core that generates structured tool calls. Three specialized tools handle the actual signal processing: (1) a **Classification Tool** using self-supervised contrastive learning with Random Lead Masking for diagnosis across arbitrary lead configurations, (2) a **Measurement Tool** built on NeuroKit2 for extracting precise PQRST interval timings, and (3) an **Explanation Tool** using SpectralX for time-frequency analysis of specific waveform segments. Each agent turn produces a triplet: a thought (reasoning), an action (tool call or direct response), and the resulting content.

**Turn-by-Turn Training with Action Sequences**: The training approach creates one training instance per agent turn, using the full prior dialogue history as input context. This forces the model to learn when to call which tool based on conversational state. The dataset defines explicit action types -- for users: ECG inquiry, request follow-up, user bye; for agents: direct response, classification tool call, measurement tool call, explanation tool call, follow-up response, response failure, system bye. Twenty pre-defined action sequences model realistic conversation flows, ensuring the agent learns natural dialogue patterns rather than just single-shot tool selection.

**On-Device Viability**: The architecture proves that 4-bit quantized 1B-3B models fine-tuned with LoRA (rank 16, alpha 16) achieve >90% next-action prediction accuracy and comparable response quality to 32B models. A 4-bit quantized 3B model requires only ~2GB RAM, fitting comfortably on smartphones alongside the OS. The key insight is that domain-specific fine-tuning on well-structured tool-calling data compensates for reduced model capacity, outperforming much larger general-purpose models (e.g., Gemini-2.5-Flash at ~70% action prediction vs. ECG-Agent at >90%).

## Step-by-Step Workflow

1. **Define the tool inventory for your domain.** Enumerate every specialized capability the agent needs -- classification, measurement, explanation, retrieval, etc. For each tool, specify: name, input schema (what signal data or parameters it accepts), output schema (structured result format), and the underlying implementation (library, model, or API). For ECG-Agent, this means a NeuroKit2-based measurement tool, a contrastive-learning classifier, and a SpectralX explainer.

2. **Design the action space and dialogue structure.** Define user action types (initial query, follow-up, clarification, goodbye) and agent action types (direct response, each tool call type, follow-up, failure, goodbye). Create 15-25 realistic action sequences that model how conversations naturally flow -- e.g., `[ecg_inquiry -> classification_tool -> follow_up_request -> measurement_tool -> user_bye -> system_bye]`. These sequences become the backbone of your training data.

3. **Build the multi-turn dialogue dataset.** For each action sequence, generate dialogues by combining: (a) domain-specific scenario categories (e.g., symptom-driven inquiry, routine checkup, device reading interpretation), (b) language proficiency levels (basic/intermediate/advanced vocabulary), and (c) signal configurations (e.g., 12-lead clinical, single-lead wearable). Use a strong LLM (e.g., GPT-4, Gemini) to synthesize realistic dialogues following these constraints. Target 5-10 turns per dialogue with ground-truth tool outputs embedded.

4. **Format training instances as turn-level triplets.** For each agent turn in every dialogue, create one training instance where the input is the full dialogue history up to that point and the target output is the agent's triplet: `{"thought": "reasoning about what to do", "action": "tool_name or direct_response", "content": "tool arguments or response text"}`. This turn-by-turn decomposition is critical -- do not train on entire dialogues as single sequences.

5. **Implement tool-call execution and output integration.** Build a runtime that parses the model's generated action, dispatches to the appropriate tool, captures the structured output, and injects it back into the dialogue context for the model's next turn. The model then generates a natural language response incorporating the tool output. Handle tool failures gracefully by mapping them to the "response_failure" action type.

6. **Fine-tune a small LLM with LoRA.** Use a 1B-3B parameter model (e.g., Llama 3.2 1B/3B) with LoRA (rank 16, alpha 16) via Unsloth or similar efficient training framework. Train with AdamW 8-bit optimizer, batch size 128, max sequence length 4096, learning rate 2e-4, weight decay 0.01, linear scheduler, for 3 epochs with early stopping on validation loss checked every 1000 steps. Split data 80/10/10 train/val/test.

7. **Quantize for on-device deployment.** Apply 4-bit quantization (GPTQ or AWQ) to the fine-tuned model. Verify the quantized model fits within target RAM budget (e.g., ~2GB for a 3B model). Benchmark inference latency on target hardware to confirm real-time conversational responsiveness.

8. **Evaluate on three axes.** Measure: (a) **Next-action prediction accuracy** -- does the model call the right tool at the right time? Target >90%. (b) **Response quality** -- score accuracy and completeness of natural language responses on a 1-5 scale using LLM-as-judge. (c) **Tool output faithfulness** -- compare tool-generated measurements/classifications against ground truth (e.g., TIoU for temporal intervals, F1 for classification).

9. **Integrate lead/signal configuration handling.** If your domain has multiple input configurations (e.g., 12-lead vs. single-lead ECG, different sensor placements), train separate configuration-aware models or use configuration tokens in the prompt. Restrict the classification label space per configuration to only diagnostically valid classes for that setup.

10. **Deploy with a lightweight inference server.** Package the quantized model with the tool implementations into a single deployable unit. Use llama.cpp, MLC-LLM, or similar on-device inference runtimes. Implement conversation state management to maintain multi-turn context across user sessions.

## Concrete Examples

**Example 1: Building the Tool Inventory for an ECG Agent**

User: "I want to build an ECG analysis chatbot that can classify rhythms, measure intervals, and explain findings. Help me set up the tool definitions."

Approach:
1. Define three tools with explicit schemas
2. Implement each using the paper's recommended libraries
3. Create a tool registry the agent can reference

Output:
```python
# tool_registry.py
from dataclasses import dataclass
from typing import Any

@dataclass
class ToolDefinition:
    name: str
    description: str
    input_schema: dict
    execute: callable

def classification_tool(ecg_signal, lead_config, sampling_rate=500):
    """SSL-based ECG classifier with Random Lead Masking for arbitrary lead support."""
    import torch
    model = load_classification_model(lead_config)  # Contrastive-pretrained encoder
    # Random Lead Masking: during training, randomly zero out leads
    # to learn representations robust to missing/variable lead configs
    prediction = model.predict(ecg_signal, sampling_rate=sampling_rate)
    return {"diagnosis": prediction.label, "confidence": prediction.score,
            "scp_code": prediction.scp_code}

def measurement_tool(ecg_signal, lead_index=0, sampling_rate=500):
    """Extract PQRST intervals using NeuroKit2."""
    import neurokit2 as nk
    signals, info = nk.ecg_process(ecg_signal[:, lead_index], sampling_rate=sampling_rate)
    waves = nk.ecg_delineate(signals, info, sampling_rate=sampling_rate, method="dwt")
    return {
        "heart_rate_bpm": signals["ECG_Rate"].mean(),
        "pr_interval_ms": compute_interval(waves, "P_Onset", "R_Onset", sampling_rate),
        "qrs_duration_ms": compute_interval(waves, "Q_Peak", "S_Peak", sampling_rate),
        "qt_interval_ms": compute_interval(waves, "Q_Peak", "T_Offset", sampling_rate),
        "rr_interval_ms": compute_rr_interval(info, sampling_rate),
    }

def explanation_tool(ecg_signal, diagnosis, lead_index=0, sampling_rate=500):
    """Time-frequency explanation using SpectralX for single-lead signals."""
    from spectralx import SpectralExplainer
    explainer = SpectralExplainer()
    result = explainer.explain(ecg_signal[:, lead_index], diagnosis, sampling_rate)
    return {"salient_segments": result.segments, "tfiou_score": result.tfiou,
            "spectrogram_path": result.save_visualization()}

TOOLS = {
    "classify": ToolDefinition(
        name="classify",
        description="Classify the ECG rhythm/morphology diagnosis",
        input_schema={"ecg_signal": "ndarray", "lead_config": "str"},
        execute=classification_tool,
    ),
    "measure": ToolDefinition(
        name="measure",
        description="Extract precise PQRST interval measurements from the ECG",
        input_schema={"ecg_signal": "ndarray", "lead_index": "int"},
        execute=measurement_tool,
    ),
    "explain": ToolDefinition(
        name="explain",
        description="Generate time-frequency explanation for a diagnosis",
        input_schema={"ecg_signal": "ndarray", "diagnosis": "str"},
        execute=explanation_tool,
    ),
}
```

**Example 2: Generating Turn-Level Training Data**

User: "How do I format multi-turn ECG dialogues into training instances for the tool-calling agent?"

Approach:
1. Structure each dialogue as a sequence of turns with explicit action types
2. Create one training instance per agent turn with full history as context
3. Format as thought-action-content triplets

Output:
```python
# build_training_instances.py
import json

def dialogue_to_training_instances(dialogue: dict) -> list[dict]:
    """Convert a multi-turn dialogue into per-agent-turn training instances."""
    instances = []
    history = []

    for turn in dialogue["turns"]:
        history.append({"role": turn["role"], "content": turn["content"]})

        if turn["role"] == "agent":
            # Each agent turn becomes one training instance
            instance = {
                "input": {
                    "ecg_metadata": dialogue["ecg_metadata"],  # lead config, patient info
                    "dialogue_history": history[:-1],  # everything before this turn
                },
                "target": {
                    "thought": turn["thought"],       # "User asks about heart rate, need measurement tool"
                    "action": turn["action"],          # "measure" | "classify" | "explain" | "direct_response"
                    "tool_args": turn.get("tool_args", {}),  # {"lead_index": 0}
                    "tool_output": turn.get("tool_output"),  # structured result from tool
                    "content": turn["content"],        # natural language response to user
                },
            }
            instances.append(instance)

    return instances

# Example dialogue structure following ECG-Agent action sequences
sample_dialogue = {
    "ecg_metadata": {"lead_config": "12-lead", "source": "PTB-XL", "record_id": 100},
    "turns": [
        {"role": "user", "action": "ecg_inquiry",
         "content": "Can you tell me what this ECG shows?"},
        {"role": "agent", "action": "classify",
         "thought": "User wants a diagnosis. I should run classification on this 12-lead ECG.",
         "tool_args": {"lead_config": "12-lead"},
         "tool_output": {"diagnosis": "Sinus Bradycardia", "confidence": 0.92},
         "content": "Based on the classification, this ECG shows Sinus Bradycardia with 92% confidence."},
        {"role": "user", "action": "request_follow_up",
         "content": "What's the exact heart rate and QT interval?"},
        {"role": "agent", "action": "measure",
         "thought": "User wants specific measurements. I need the measurement tool for heart rate and QT.",
         "tool_args": {"lead_index": 1},
         "tool_output": {"heart_rate_bpm": 52.3, "qt_interval_ms": 420},
         "content": "The heart rate is 52.3 bpm and the QT interval is 420ms, both consistent with bradycardia."},
        {"role": "user", "action": "user_bye",
         "content": "Thanks, that's helpful."},
        {"role": "agent", "action": "system_bye",
         "thought": "User is done. End conversation.",
         "content": "You're welcome. Please consult a cardiologist for clinical decisions."},
    ],
}

instances = dialogue_to_training_instances(sample_dialogue)
# Produces 3 training instances (one per agent turn)
```

**Example 3: LoRA Fine-Tuning Configuration for On-Device Deployment**

User: "Set up the training config for fine-tuning a 3B model to be an ECG tool-calling agent that runs on a phone."

Approach:
1. Configure Unsloth with LoRA parameters matching the paper
2. Set up 4-bit quantization for deployment
3. Validate RAM footprint fits mobile constraints

Output:
```python
# train_ecg_agent.py
from unsloth import FastLanguageModel

# Load base model with 4-bit quantization for training efficiency
model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="meta-llama/Llama-3.2-3B-Instruct",
    max_seq_length=4096,
    load_in_4bit=True,
)

# Apply LoRA -- paper uses rank=16, alpha=16
model = FastLanguageModel.get_peft_model(
    model,
    r=16,
    lora_alpha=16,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                     "gate_proj", "up_proj", "down_proj"],
    lora_dropout=0.0,
    use_gradient_checkpointing="unsloth",  # memory optimization
)

from transformers import TrainingArguments
from trl import SFTTrainer

training_args = TrainingArguments(
    output_dir="./ecg-agent-3b",
    per_device_train_batch_size=2,
    gradient_accumulation_steps=64,   # effective batch size = 128
    num_train_epochs=3,
    learning_rate=2e-4,
    weight_decay=0.01,
    lr_scheduler_type="linear",
    warmup_steps=100,
    eval_strategy="steps",
    eval_steps=1000,
    save_strategy="steps",
    save_steps=1000,
    load_best_model_at_end=True,       # early stopping on val loss
    metric_for_best_model="eval_loss",
    fp16=True,
    optim="adamw_8bit",
)

trainer = SFTTrainer(
    model=model,
    tokenizer=tokenizer,
    train_dataset=train_dataset,
    eval_dataset=val_dataset,
    args=training_args,
    max_seq_length=4096,
    dataset_text_field="formatted_text",  # turn-level triplet format
)

trainer.train()

# Export quantized model for on-device deployment (~2GB for 3B at 4-bit)
model.save_pretrained_gguf("ecg-agent-3b-q4", tokenizer, quantization_method="q4_k_m")
# Deployable via llama.cpp or MLC-LLM on mobile
```

## Best Practices

- **Do** define explicit action types for both user and agent turns. The structured action space (classify, measure, explain, direct_response, failure) is what makes tool selection learnable -- without it, the model must infer intent from free-text alone.
- **Do** train turn-by-turn rather than on whole dialogues. Each agent turn as a separate training instance with full history forces the model to learn contextual tool selection at every decision point.
- **Do** restrict the classification label space per input configuration. A single-lead wearable ECG cannot diagnose conditions that require multi-lead views (e.g., STEMI localization). Hardcode these constraints into your tool rather than relying on the LLM to learn them.
- **Do** include "response_failure" as an explicit action type. The agent should learn to say "I cannot determine this" rather than hallucinate a diagnosis when tool outputs are ambiguous or confidence is low.
- **Avoid** training an end-to-end model to produce precise measurements directly. Numerical values like interval durations in milliseconds are poorly learned by LLMs -- always delegate to dedicated signal processing tools (NeuroKit2, SciPy) for quantitative outputs.
- **Avoid** using models larger than 3B if targeting on-device deployment. The paper shows 1B and 3B models achieve comparable quality to 8B and 32B after domain-specific fine-tuning, while 8B at 4-bit already exceeds typical smartphone RAM budgets (~5-6GB).

## Error Handling

- **Tool execution failures**: When a measurement or classification tool raises an exception (corrupted signal, unsupported lead configuration), the agent should emit the `response_failure` action with a natural language explanation. Train on failure examples so the model learns this gracefully rather than hallucinating.
- **Out-of-distribution signals**: If the ECG signal has excessive noise or artifacts, the measurement tool may return unreliable PQRST delineation. Implement confidence thresholds in each tool and surface low-confidence warnings in the agent's response.
- **Context window overflow**: Multi-turn dialogues with embedded tool outputs can exceed the 4096-token sequence length. Implement a sliding window that preserves the most recent N turns plus the original ECG metadata, or summarize earlier turns.
- **Quantization degradation**: After 4-bit quantization, evaluate next-action prediction accuracy. If it drops below 85%, increase to 8-bit quantization or use a larger base model. The paper shows minimal degradation at 4-bit for 3B models, but this is data-dependent.
- **Lead configuration mismatch**: If the user's ECG has a different lead configuration than what the classification model expects, the agent should detect this and either switch to a configuration-appropriate model or inform the user of the limitation.

## Limitations

- **Not a certified medical device.** ECG-Agent is a research prototype. Its outputs must not be used for clinical decision-making without physician oversight. Always include this disclaimer in deployed systems.
- **Single-lead explanation tool (SpectralX) does not generalize to 12-lead.** The paper explicitly excludes the explanation tool from 12-lead configurations. Multi-lead explanations require different visualization approaches.
- **Dataset bias toward PTB-XL.** The training data derives from one ECG database. Performance on ECGs from different devices, populations, or recording conditions may degrade. Evaluate on held-out datasets from different sources before deployment.
- **Fixed action sequence patterns.** The 20 pre-defined action sequences may not cover all real-world conversation patterns. Edge cases like mid-conversation topic changes or ambiguous user intent may cause incorrect tool selection.
- **Latency of chained tool calls.** In conversations requiring multiple sequential tools (classify then measure then explain), each tool adds inference latency. On-device, this can compound to noticeable delays. Profile and optimize the slowest tools first.

## Reference

- **Paper**: [ECG-Agent: On-Device Tool-Calling Agent for ECG Multi-Turn Dialogue](https://arxiv.org/abs/2601.20323v1) -- Focus on Section 3 (Architecture & Tool Definitions), Section 4 (ECG-MTD Dataset Construction), and Section 5 (Turn-by-Turn Training) for implementation details.
- **Code**: [github.com/gustmd0121/ECG-Agent](https://github.com/gustmd0121/ECG-Agent)