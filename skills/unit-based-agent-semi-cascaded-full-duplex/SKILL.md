---
name: "unit-based-agent-semi-cascaded-full-duplex"
description: "Build full-duplex voice dialogue systems using unit-based agent decomposition and semi-cascaded pipelines. Trigger phrases: 'build a full-duplex dialogue system', 'implement voice interaction with interruption handling', 'create a real-time conversational agent', 'design a semi-cascaded speech pipeline', 'add backchannel and turn-taking to a voice agent', 'unit-based dialogue processing'"
---

# Unit-Based Agent for Semi-Cascaded Full-Duplex Dialogue Systems

This skill teaches Claude to design and implement **full-duplex voice dialogue systems** that can listen and speak simultaneously, handle interruptions, and manage natural turn-taking. The core technique decomposes dialogue into minimal **conversational units** (discrete semantic segments smaller than full speaker turns) and processes them through a **semi-cascaded pipeline** where upstream modules accept new input while downstream modules finish processing earlier units. This eliminates the rigid "listen then respond" pattern of half-duplex systems.

## When to Use

- When the user asks to build a real-time voice assistant that can be interrupted mid-sentence
- When implementing a spoken dialogue system that needs backchannel responses ("mm-hmm", "right")
- When designing an agent pipeline where VAD, ASR, LLM, and TTS must run with minimal latency
- When the user wants to add full-duplex (simultaneous listen + speak) capability to an existing voice agent
- When building a WebSocket-based audio streaming backend that manages dialogue state
- When the user needs to handle semantic shift detection (topic changes) during ongoing conversation
- When creating a system that distinguishes genuine interruptions from backchannels during agent speech

## Key Technique

### Conversational Units vs. Speaker Turns

Traditional dialogue systems operate on **speaker turns**: one party speaks, then the other responds. This creates unnatural pauses and cannot handle overlapping speech. The unit-based approach instead decomposes dialogue into **minimal conversational units** -- the smallest semantically complete segment of speech (a phrase, clause, or short sentence). Each unit is processed independently through the pipeline. The system predicts when to transition to the next unit using an LLM-based judge that classifies each segment as "continue" (user still speaking) or "switch" (unit complete, begin response generation).

### Semi-Cascaded Pipeline

A fully cascaded pipeline runs ASR -> LLM -> TTS sequentially, blocking at each stage. A fully end-to-end model processes everything jointly but sacrifices modularity. The **semi-cascaded** approach sits between these: components execute in sequence for a given unit, but the pipeline accepts new input units concurrently via async task scheduling. When the ASR finishes processing unit N, the LLM begins generating a response for unit N while the VAD and ASR are already processing unit N+1. This parallelism across units drastically reduces perceived latency while keeping each component independently swappable (train-free, plug-and-play).

### State Machine for Dialogue Flow

The system operates in two primary states -- **LISTEN** and **SPEAK** -- with transition logic driven by VAD events and LLM judgments:
- In LISTEN state: accumulate audio frames, detect speech end via silence threshold (`end_hold_frame`), then run ASR -> LLM judge -> response generation.
- In SPEAK state: continue monitoring user audio for interruptions. Short overlapping speech triggers an **interrupt judge** that classifies it as a genuine interruption ("switch" -> stop speaking, revert to LISTEN) or a backchannel ("continue" -> keep speaking).
- A **semantic shift detector** compares new user input against conversation history to decide whether to generate a fresh response or repeat/refine the previous one.

## Step-by-Step Workflow

1. **Set up the async WebSocket server** that accepts streaming audio chunks (16ms frames at 16kHz, 256 samples per frame) from clients over a persistent bidirectional connection.

2. **Initialize the VAD pipeline** using a lightweight model (e.g., Silero VAD). Buffer incoming audio frames and evaluate speech presence in windows of `2 * WINDOW_SIZE` samples. Emit "start" and "end" events with timestamps to mark speech segment boundaries.

3. **Implement the two-state dialogue manager** with LISTEN and SPEAK states. In LISTEN, accumulate audio into a per-turn buffer. Track turns with an incrementing `TURN_IDX`. Transition to SPEAK after generating a response; transition back to LISTEN on turn completion or interruption.

4. **Add the unit completion judge**. After detecting end-of-speech in LISTEN state (silence exceeds `end_hold_frame` threshold, ~0.64s), send the transcribed text to the LLM with a judge prompt that outputs "continue" or "switch". If "continue", arm a `CONTINUE_ARMED` flag and wait for `after_continue_time` (e.g., 2.5s) before forcing a switch. If "switch", immediately begin response generation.

5. **Build the ASR -> LLM -> TTS pipeline as async tasks**. Use `asyncio.create_task()` for each stage so they run concurrently across different units. The ASR transcribes the buffered audio, the LLM generates a text response (constrained to ~15 words for conversational brevity), and the TTS synthesizes audio. Resample TTS output to match the system sample rate (16kHz).

6. **Implement interruption handling in SPEAK state**. While the agent speaks, continue running VAD on incoming user audio. If speech is detected for more than a brief threshold (e.g., 1.5s), invoke the interrupt judge prompt. On "switch" (genuine interruption): cancel current TTS playback, send a `stop_audio` event to the client, increment `TURN_IDX`, and revert to LISTEN. On "continue" (backchannel): ignore and keep speaking.

7. **Add semantic shift detection**. Before generating a response, send conversation history and the new user utterance to the LLM with a shift prompt that classifies whether the topic changed ("yes"/"no"). If "yes", use the standard response prompt. If "no" (user is asking to repeat or continuing the same topic), use a specialized re-prompt that references prior context.

8. **Manage conversation history**. Maintain parallel `user_history` and `assistant_history` lists. Pass these to the LLM via a `build_messages()` function when `use_history=True` to preserve multi-turn context. Keep history bounded to prevent context window overflow.

9. **Stream TTS audio back to the client** over the same WebSocket as timestamped binary frames. Send a `tts_done` JSON event with the timestamp after each TTS segment completes so the client can coordinate playback with its own audio capture.

10. **Test with simulated real-time input** by reading WAV files and feeding chunks at real-time rate (sleeping `frame_time` between sends) to validate timing behavior under realistic conditions.

## Concrete Examples

**Example 1: Building a full-duplex customer service agent**

User: "I want to build a voice agent for customer support that can handle interruptions naturally. The user should be able to cut in while the agent is speaking."

Approach:
1. Create a WebSocket server (`backend.py`) with asyncio that accepts 16kHz/16-bit audio streams.
2. Initialize Silero VAD and define the LISTEN/SPEAK state machine:

```python
import asyncio
import websockets
import torch

class DialogueAgent:
    def __init__(self):
        self.state = "LISTEN"
        self.turn_idx = 0
        self.buffer = bytearray()
        self.user_history = []
        self.assistant_history = []
        self.continue_armed = False

        # VAD setup
        self.vad_model, _ = torch.hub.load('snakers4/silero-vad', 'silero_vad')
        self.vad_iterator = self.vad_model.reset_states()

    async def handle_audio_frame(self, frame: bytes):
        vad_event = self.detect_vad(frame)

        if self.state == "LISTEN":
            await self.handle_listen(frame, vad_event)
        elif self.state == "SPEAK":
            await self.handle_speak(frame, vad_event)

    async def handle_listen(self, frame, vad_event):
        if vad_event == "speech_active":
            self.buffer.extend(frame)
        elif vad_event == "speech_end":
            # Unit complete -- run pipeline
            text = await self.asr(self.buffer)
            judgment = await self.judge_completion(text)
            if judgment == "switch":
                response = await self.generate_response(text)
                self.state = "SPEAK"
                asyncio.create_task(self.speak_response(response))
            else:
                self.continue_armed = True
                # Wait for more input or timeout
                asyncio.create_task(self.continue_timeout())

    async def handle_speak(self, frame, vad_event):
        if vad_event == "speech_active":
            # Potential interruption -- classify it
            text = await self.asr(frame)
            judgment = await self.judge_interrupt(text)
            if judgment == "switch":
                await self.stop_speaking()
                self.turn_idx += 1
                self.state = "LISTEN"
            # "continue" = backchannel, ignore
```

3. Add the judge and interrupt prompts as LLM calls with constrained output ("continue"/"switch").
4. Wire up TTS streaming so `speak_response` sends audio chunks back through the WebSocket.
5. On the client side, run concurrent send/receive tasks for full-duplex audio.

Output: A real-time voice agent that processes speech in units, responds within ~1s, and gracefully handles user interruptions by canceling TTS and reverting to listen mode.

---

**Example 2: Adding semantic shift detection to an existing chatbot**

User: "My voice chatbot sometimes repeats itself when the user changes topic. How do I detect topic changes and respond appropriately?"

Approach:
1. Add a semantic shift classifier using the LLM before response generation:

```python
SHIFT_PROMPT = """Given the conversation history and the new user utterance,
determine if the topic has significantly changed.
Output ONLY "yes" or "no".

History: {history}
New utterance: {utterance}"""

async def detect_semantic_shift(self, utterance: str) -> bool:
    history_text = "\n".join(
        f"User: {u}\nAssistant: {a}"
        for u, a in zip(self.user_history, self.assistant_history)
    )
    prompt = SHIFT_PROMPT.format(history=history_text, utterance=utterance)
    result = await self.llm_call(prompt, max_tokens=4)
    return result.strip().lower() == "yes"
```

2. Branch response generation based on shift detection:

```python
async def generate_response(self, user_text: str) -> str:
    shifted = await self.detect_semantic_shift(user_text)
    if shifted:
        # New topic: generate fresh response
        prompt = RESPONSE_PROMPT.format(input=user_text)
    else:
        # Same topic: may be asking to repeat -- reference history
        prompt = SHIFT_RE_PROMPT.format(
            input=user_text,
            previous_response=self.assistant_history[-1]
        )
    response = await self.llm_call(prompt, max_tokens=256)
    self.user_history.append(user_text)
    self.assistant_history.append(response)
    return response
```

Output: The chatbot correctly distinguishes "Tell me about restaurants nearby" (new topic) from "Can you say that again?" (same topic, repeat previous answer).

---

**Example 3: Designing the async pipeline for low latency**

User: "How do I structure the ASR -> LLM -> TTS pipeline so processing one utterance doesn't block the next?"

Approach:
1. Use `asyncio.create_task()` to launch each pipeline stage as a non-blocking coroutine:

```python
async def process_unit(self, audio_buffer: bytes, turn_idx: int):
    """Process one conversational unit through the full pipeline."""
    # Each stage awaits its own I/O but doesn't block the event loop
    transcript = await self.async_asr(audio_buffer)

    # Check if this turn is still current (may have been interrupted)
    if self.turn_idx != turn_idx:
        return  # Stale unit, discard

    response_text = await self.async_llm(transcript)

    if self.turn_idx != turn_idx:
        return

    audio_bytes = await self.async_tts(response_text)

    if self.turn_idx != turn_idx:
        return

    await self.stream_audio_to_client(audio_bytes)
```

2. The key insight: each `process_unit` call is launched with `asyncio.create_task(self.process_unit(buf, idx))`. The main event loop continues accepting new audio frames immediately. If an interruption increments `turn_idx`, in-flight tasks for stale turns detect the mismatch and abort early.

3. This is the "semi-cascaded" pattern: sequential within a unit, concurrent across units.

Output: Pipeline latency drops from the sum of all stages (~2-3s) to roughly the duration of the slowest single stage (~0.5-1s) for successive units, because earlier stages on the next unit overlap with later stages on the current unit.

## Best Practices

- **Do** keep LLM responses short (10-20 words) for conversational units. Long responses increase TTS latency and the probability of being interrupted mid-sentence.
- **Do** check `turn_idx` at each pipeline stage boundary to detect stale units early and avoid wasted computation on interrupted turns.
- **Do** use a `continue_armed` timeout mechanism (2-3 seconds) after the judge says "continue" to handle cases where the user simply paused mid-thought. This prevents the system from hanging indefinitely.
- **Do** separate the interrupt judge from the completion judge -- they serve different purposes and need different prompts (interrupt classification considers whether overlapping speech is a backchannel vs. a real interruption).
- **Avoid** processing the full audio history through ASR on every turn. Buffer and transcribe only the current unit's audio segment.
- **Avoid** blocking the WebSocket event loop with synchronous model calls. All ASR/LLM/TTS invocations must be async (use `aiohttp` or run synchronous calls in a thread executor).

## Error Handling

- **VAD false positives (noise detected as speech)**: Set a minimum speech duration threshold (e.g., 200ms). Discard segments shorter than this before sending to ASR.
- **ASR returns empty or garbage text**: Skip the LLM/TTS pipeline for that unit. Reset the buffer and continue listening. Do not pass empty strings to the judge prompt.
- **TTS service timeout or failure**: Send a `tts_error` event to the client so it can display a fallback (text response). Do not let TTS failures block the state machine -- revert to LISTEN state.
- **LLM judge returns unexpected output** (neither "continue" nor "switch"): Default to "switch" to avoid hanging in LISTEN state. Log the anomalous response for debugging.
- **Turn index race conditions**: Always capture `turn_idx` at task creation time and compare at each stage boundary. Use the stale-check pattern shown in Example 3.
- **WebSocket disconnection during SPEAK**: Catch `ConnectionClosed` exceptions in the audio streaming loop. Clean up the turn state and reset to LISTEN for the next connection.

## Limitations

- **Language-dependent prompts**: The judge, interrupt, and shift prompts are tuned for specific interaction patterns. They need adaptation for different languages or cultural norms around turn-taking (e.g., overlap tolerance varies across cultures).
- **Single-speaker assumption**: The pipeline assumes one user and one agent. Multi-party dialogue requires speaker diarization upstream, which this framework does not address.
- **LLM latency floor**: Even with the semi-cascaded design, initial response latency is bounded by a single ASR + LLM + TTS pass. For sub-200ms responses, you need a fully end-to-end model.
- **No streaming LLM generation**: The current design waits for complete LLM output before sending to TTS. Streaming LLM tokens to an incremental TTS system could further reduce latency but adds complexity.
- **Train-free means prompt-dependent**: All behavioral control (judging, interruption, shift detection) relies on prompt engineering rather than fine-tuned classifiers. This is flexible but less robust than trained models for edge cases.
- **VAD-only segmentation**: The system relies on silence gaps to identify unit boundaries, which can fail in fast-paced speech or noisy environments.

## Reference

**Paper**: [Unit-Based Agent for Semi-Cascaded Full-Duplex Dialogue Systems](https://arxiv.org/abs/2601.20230v2) (ICASSP 2026). Look for: the formal definition of conversational units, the state transition diagram between LISTEN/SPEAK modes, and the prompt templates for the judge/interrupt/shift classifiers.

**Code**: [github.com/yu-haoyuan/fd-badcat](https://github.com/yu-haoyuan/fd-badcat) -- Reference implementation using Qwen3-Omni, Silero VAD, Sherpa-ONNX ASR, and Index-TTS.