---
name: "emoara-emotion-preserving-english-speech"
description: "Build emotion-preserving cross-lingual speech pipelines that detect emotion from audio, transcribe, translate, and synthesize speech in a target language while retaining emotional nuance. Trigger phrases: 'build emotion-aware speech translation', 'preserve emotion across languages', 'cross-lingual speech pipeline with emotion', 'emotion-preserving TTS', 'translate speech keeping emotion', 'banking customer service speech system'"
---

# EmoAra: Emotion-Preserving Cross-Lingual Speech Pipeline

This skill enables Claude to design and implement end-to-end speech processing pipelines that detect emotion from input speech, transcribe it, translate to a target language, and synthesize emotionally appropriate output speech. The core technique chains four specialized models — a CNN-based Speech Emotion Recognition (SER) classifier, Whisper ASR, fine-tuned MarianMT translation, and MMS-TTS — so that emotion metadata detected at the input stage conditions downstream translation tone and synthesis prosody. The approach originates from a banking customer service use case where preserving caller frustration, satisfaction, or urgency across an English-to-Arabic language barrier directly impacts service quality.

## When to Use

- When the user asks to build a speech-to-speech translation system that preserves emotional tone across languages
- When the user needs to classify emotions from audio (angry, happy, sad, neutral, calm, fearful, disgusted, surprised) using spectral features and a CNN
- When the user wants to fine-tune MarianMT for domain-specific translation (e.g., banking, medical, legal) with parallel corpora
- When the user asks to chain ASR, MT, and TTS models into a single inference pipeline with metadata passing between stages
- When the user needs to extract audio features (ZCR, RMSE, MFCCs) and build a lightweight emotion classifier
- When the user is building a multilingual call center system where caller emotion must be logged, preserved, or acted upon
- When the user asks to augment a small speech emotion dataset with noise injection, pitch shifting, and time stretching

## Key Technique

EmoAra's core insight is that cross-lingual speech translation loses emotional context when each stage operates independently. A caller saying "I've been waiting for an hour!" with audible frustration gets translated into flat, neutral Arabic TTS output, stripping the urgency that a human agent needs to hear. The solution is a four-stage pipeline where emotion is detected first and propagated as metadata through all subsequent stages: (1) a CNN classifier extracts emotion from audio features, (2) Whisper transcribes the English speech, (3) a fine-tuned MarianMT model translates to Arabic with awareness of domain terminology, and (4) MMS-TTS-Ara synthesizes Arabic speech with prosody adjustments guided by the emotion label.

The CNN classifier uses three 1D convolutional layers (64, 128, 256 filters, kernel size 3) with batch normalization, max pooling, and dropout (0.15 for conv layers, 0.25 for dense), fed by concatenated Zero-Crossing Rate, Root Mean Square Energy, and MFCC features normalized via StandardScaler. Trained on RAVDESS (1,440 files, 8 emotion classes), it achieves 94% F1. The MarianMT model is fine-tuned with a learning rate of 3e-5, 1000 warmup steps, 10 epochs, batch size 8 with gradient accumulation of 4, beam search width 8, and FP16 mixed precision. The translation training data combines 24,000+ general English-Arabic pairs with 10,000 banking-domain sentences from Banking77 translated via Google Translate and MyMemory, achieving BLEU 56 and BERTScore F1 88.7%.

The critical architectural decision is where and how to inject the emotion label. The paper detects emotion at the audio input stage and carries it forward as a discrete tag. In practice, this tag can condition TTS prosody parameters (pitch range, speaking rate, energy contour) and optionally prefix the translation input to bias tone-appropriate word choices (e.g., translating "fine" differently when the speaker is angry vs. genuinely content).

## Step-by-Step Workflow

1. **Extract audio features from the input speech file.** Load the audio with librosa, compute Zero-Crossing Rate (ZCR), Root Mean Square Energy (RMSE), and MFCCs. Concatenate these into a single feature vector per utterance. Apply data augmentation (noise injection, pitch shifting ±2 semitones, time stretching 0.8x–1.2x) if training the classifier.

2. **Normalize features with StandardScaler.** Fit the scaler on training data and transform all feature vectors to zero mean and unit variance. Save the fitted scaler for inference — this is a common source of bugs when the scaler is refit on test data.

3. **Classify emotion with the CNN.** Build a 1D CNN: three Conv1D layers (64/128/256 filters, kernel size 3, ReLU) each followed by BatchNorm and MaxPool (pool size 2), dropout 0.15 after conv blocks, a Dense(256, ReLU) layer with dropout 0.25, and a softmax output over 8 classes (neutral, calm, happy, sad, angry, fearful, disgust, surprised). Train on RAVDESS or a comparable dataset. At inference, output the predicted emotion label and confidence score.

4. **Transcribe the English speech with Whisper.** Use `openai-whisper` (base model for speed, large-v3 for accuracy). Pass the raw audio file; Whisper handles its own preprocessing. Capture the transcribed text and segment timestamps.

5. **Optionally prefix the transcription with an emotion tag.** Before translation, prepend a tag like `[EMOTION: angry]` to the source text if you have trained or prompted the translation model to be emotion-aware. For standard MarianMT fine-tuning without emotion tags, skip this and use the emotion label only for TTS conditioning.

6. **Translate with fine-tuned MarianMT.** Load `Helsinki-NLP/opus-mt-en-ar` and fine-tune on your domain-specific parallel corpus using AdamW (lr=3e-5, warmup=1000, 10 epochs, batch=8, grad_accum=4, FP16). At inference, encode the English text, decode with beam search (width 8), and post-process Arabic text (fix tokenization artifacts, normalize Unicode).

7. **Map the emotion label to TTS prosody parameters.** Define a prosody mapping table: angry → faster rate (1.15x), higher pitch (+20%), increased energy; sad → slower rate (0.85x), lower pitch (-10%), reduced energy; happy → slightly faster (1.1x), wider pitch range; neutral/calm → default parameters. These multipliers are applied to the TTS output.

8. **Synthesize Arabic speech with MMS-TTS-Ara.** Use Facebook's `facebook/mms-tts-ara` model from Hugging Face Transformers. Pass the translated Arabic text, generate the mel-spectrogram via the sequence generator, and vocode with HiFi-GAN. Apply the prosody modifications from step 7 via post-processing (time-stretch with librosa, pitch-shift, amplitude scaling).

9. **Package the output.** Return a structured result containing: the detected emotion label and confidence, the English transcript, the Arabic translation, the Arabic audio file path, and timing metadata for each pipeline stage. Log all intermediate outputs for debugging.

10. **Evaluate end-to-end quality.** Measure emotion classification F1 per class, translation BLEU and BERTScore, and conduct human evaluation on a 1–3 scale for accuracy, fluency, and domain terminology correctness. Target: F1 ≥ 90%, BLEU ≥ 50, human score ≥ 80%.

## Concrete Examples

**Example 1: Building the Emotion Classifier**

User: "I want to build a speech emotion classifier using RAVDESS. It should detect 8 emotions from audio files."

Approach:
1. Download RAVDESS dataset (1,440 files, 24 actors)
2. Extract features per file:
```python
import librosa
import numpy as np

def extract_features(file_path):
    y, sr = librosa.load(file_path, sr=None)
    zcr = np.mean(librosa.feature.zero_crossing_rate(y))
    rmse = np.mean(librosa.feature.rms(y=y))
    mfccs = np.mean(librosa.feature.mfcc(y=y, sr=sr, n_mfcc=40), axis=1)
    return np.concatenate([[zcr, rmse], mfccs])
```
3. Build and train CNN:
```python
from tensorflow.keras import layers, models

model = models.Sequential([
    layers.Conv1D(64, 3, activation='relu', input_shape=(42, 1)),
    layers.BatchNormalization(),
    layers.MaxPooling1D(2),
    layers.Dropout(0.15),
    layers.Conv1D(128, 3, activation='relu'),
    layers.BatchNormalization(),
    layers.MaxPooling1D(2),
    layers.Dropout(0.15),
    layers.Conv1D(256, 3, activation='relu'),
    layers.BatchNormalization(),
    layers.MaxPooling1D(2),
    layers.Dropout(0.15),
    layers.Flatten(),
    layers.Dense(256, activation='relu'),
    layers.Dropout(0.25),
    layers.Dense(8, activation='softmax')
])
model.compile(optimizer='adam', loss='categorical_crossentropy', metrics=['accuracy'])
```

Output: A saved Keras model achieving ~94% F1 across 8 emotion classes, with per-class metrics showing lower scores for calm/fearful/sad due to acoustic overlap.

**Example 2: Fine-Tuning MarianMT for Banking Domain**

User: "Fine-tune an English-to-Arabic translation model for banking customer service."

Approach:
1. Load Banking77 from Hugging Face, translate English sentences to Arabic using batch translation APIs to create parallel pairs
2. Combine with a general-purpose en-ar corpus (24K+ pairs)
3. Fine-tune:
```python
from transformers import MarianMTModel, MarianTokenizer, Seq2SeqTrainer, Seq2SeqTrainingArguments

model_name = "Helsinki-NLP/opus-mt-en-ar"
tokenizer = MarianTokenizer.from_pretrained(model_name)
model = MarianMTModel.from_pretrained(model_name)

training_args = Seq2SeqTrainingArguments(
    output_dir="./marian-banking-ar",
    num_train_epochs=10,
    per_device_train_batch_size=8,
    gradient_accumulation_steps=4,
    learning_rate=3e-5,
    warmup_steps=1000,
    fp16=True,
    evaluation_strategy="epoch",
    save_strategy="epoch",
    predict_with_generate=True,
    generation_num_beams=8,
    generation_max_length=300,
)
```

Output: A fine-tuned model producing domain-accurate Arabic translations of banking terms (e.g., "overdraft" → "سحب على المكشوف", "wire transfer" → "حوالة مصرفية") with BLEU ~56 on the banking test set.

**Example 3: Full End-to-End Pipeline**

User: "Build a complete pipeline that takes an English audio file from a customer call, detects emotion, transcribes, translates to Arabic, and generates Arabic audio."

Approach:
1. Load the emotion classifier, Whisper model, MarianMT model, and MMS-TTS
2. Wire them into a single function:
```python
import torch
from transformers import pipeline, VitsModel, AutoTokenizer
import whisper

def emoara_pipeline(audio_path):
    # Stage 1: Emotion classification
    features = extract_features(audio_path)  # ZCR + RMSE + MFCCs
    features_scaled = scaler.transform(features.reshape(1, -1))
    emotion_probs = emotion_model.predict(features_scaled.reshape(1, -1, 1))
    emotion_label = EMOTIONS[emotion_probs.argmax()]
    confidence = float(emotion_probs.max())

    # Stage 2: ASR with Whisper
    whisper_model = whisper.load_model("base")
    result = whisper_model.transcribe(audio_path)
    english_text = result["text"]

    # Stage 3: Translation with fine-tuned MarianMT
    translated = tokenizer_mt.decode(
        mt_model.generate(**tokenizer_mt(english_text, return_tensors="pt", padding=True),
                         num_beams=8, max_length=300)[0],
        skip_special_tokens=True
    )

    # Stage 4: Arabic TTS with emotion-conditioned prosody
    tts_model = VitsModel.from_pretrained("facebook/mms-tts-ara")
    tts_tokenizer = AutoTokenizer.from_pretrained("facebook/mms-tts-ara")
    inputs = tts_tokenizer(translated, return_tensors="pt")
    with torch.no_grad():
        output = tts_model(**inputs).waveform

    # Apply prosody modification based on emotion
    audio_np = output.squeeze().numpy()
    audio_np = apply_prosody(audio_np, emotion_label, sr=tts_model.config.sampling_rate)

    return {
        "emotion": emotion_label,
        "confidence": confidence,
        "english_text": english_text,
        "arabic_text": translated,
        "audio": audio_np,
        "sample_rate": tts_model.config.sampling_rate
    }
```

Output: A dictionary with detected emotion ("angry", 0.92 confidence), English transcript, Arabic translation, and a NumPy audio array of the Arabic speech synthesized with faster rate and elevated pitch matching the angry emotion.

## Best Practices

- **Do:** Detect emotion from the raw audio *before* ASR — acoustic features carry emotional information that text alone cannot capture (sarcasm, frustration intensity, trembling voice).
- **Do:** Save and version the StandardScaler fitted on training data. Using a differently-fitted scaler at inference silently degrades classifier accuracy with no error raised.
- **Do:** Fine-tune MarianMT on in-domain parallel data even if the base model handles general translation well. Banking/medical/legal terminology requires domain adaptation to avoid critical mistranslations.
- **Do:** Apply prosody modifications as post-processing on the TTS waveform (librosa time_stretch, pitch_shift) rather than trying to modify model internals, which requires retraining.
- **Avoid:** Sending emotion tags in the translation input unless the MT model was specifically trained to interpret them — otherwise the model will attempt to literally translate "[EMOTION: angry]" into Arabic.
- **Avoid:** Using the CNN emotion classifier on audio shorter than 1 second or with heavy background noise without preprocessing. Apply noise reduction (noisereduce library) and pad short clips before feature extraction.

## Error Handling

| Problem | Cause | Solution |
|---------|-------|----------|
| Emotion classifier always predicts "neutral" | Feature scaler mismatch between train/inference | Reload the exact scaler fitted during training; verify feature vector length matches |
| Whisper produces garbled or empty transcript | Audio sample rate mismatch or corrupt file | Resample to 16kHz with librosa before passing to Whisper |
| MarianMT output contains `<unk>` tokens | Out-of-vocabulary terms not in training data | Add domain-specific vocabulary; increase beam width; try a larger base model |
| MMS-TTS-Ara produces robotic/unintelligible speech | Input text contains Latin characters or numbers | Transliterate numbers to Arabic words; strip any remaining English characters |
| Prosody post-processing creates audio artifacts | Extreme pitch/speed multipliers | Clamp prosody multipliers to ±25% of default values; use higher-quality time-stretching (phase vocoder) |
| Pipeline latency too high for real-time use | Sequential model loading on each call | Pre-load all models at startup; use Whisper "base" instead of "large"; batch short utterances |

## Limitations

- **Emotion propagation is indirect.** The detected emotion label conditions TTS prosody via post-processing heuristics, not through a learned end-to-end emotion transfer mechanism. Subtle emotional nuances (light irony, passive aggression) are lost.
- **Limited to 8 discrete emotions.** The RAVDESS-trained classifier cannot distinguish fine-grained states like "frustrated" vs. "angry" or "anxious" vs. "fearful." Dimensional emotion models (valence-arousal) would offer more granularity.
- **Calm, fearful, and sad have lower accuracy** due to overlapping acoustic features in these categories. Expect ~85% F1 for these vs. ~97% for angry/happy.
- **Translation degrades beyond ~20 words** per segment because the fine-tuning data consists of short banking sentences. Long utterances should be split at sentence boundaries before translation.
- **MMS-TTS-Ara produces flat prosody natively.** All emotional expression relies on post-processing heuristics — there is no emotion-conditioned Arabic TTS model in the pipeline. This is the weakest link.
- **English-to-Arabic only.** The pipeline architecture generalizes to other language pairs, but the fine-tuned MarianMT and MMS-TTS components are language-specific and must be retrained.
- **No streaming support.** The pipeline processes complete utterances, making it unsuitable for real-time simultaneous interpretation without buffering.

## Reference

**Paper:** [EmoAra: Emotion-Preserving English Speech Transcription and Cross-Lingual Translation with Arabic Text-to-Speech](https://arxiv.org/abs/2602.01170v1) (Hassan et al., 2026). Look for: the CNN architecture details in Section 3, the MarianMT fine-tuning hyperparameters in Section 4, the emotion classification confusion matrix showing per-class F1 scores, and the human evaluation methodology on banking-domain translations.