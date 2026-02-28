---
language:
  - en
  - zh
  - de
  - it
  - pt
  - es
  - ja
  - ko
  - fr
  - ru
license: apache-2.0
tags:
  - rust
  - cpu-inference
  - quantized
  - q4
  - text-to-speech
  - tts
  - pure-rust
  - no-python
  - no-cuda
  - multi-speaker
  - multilingual
library_name: qora
pipeline_tag: text-to-speech
---

# QORA-TTS - Native Rust Text-to-Speech Engine

## Downlod 🤗: https://huggingface.co/qoranet/QORA-TTS

Pure Rust text-to-speech synthesis engine. No Python runtime, no CUDA, no external dependencies. Single executable + quantized weights = portable TTS on any machine.

## Overview

| Property | Value |
|----------|-------|
| **Engine** | QORA-TTS (Pure Rust) |
| **Base Model** | Qwen3-TTS 1.7B |
| **Parameters** | ~1.84B (Talker) + ~150M (Code Predictor) + Decoder |
| **Quantization** | Q4 (4-bit symmetric, group_size=32) |
| **Model Size** | 1.5 GB (Q4) |
| **Executable** | 4.0 MB |
| **Sample Rate** | 24 kHz (16-bit PCM WAV) |
| **Languages** | 12 (English, Chinese, German, Italian, Portuguese, Spanish, Japanese, Korean, French, Russian, + 2 dialects) |
| **Speakers** | 9 built-in voices |
| **Platform** | Windows x86_64 (CPU-only) |

## Architecture

### Talker (Main Model) - 28-layer Transformer

| Component | Details |
|-----------|---------|
| **Layers** | 28 decoder layers |
| **Hidden Size** | 2,048 |
| **Attention Heads** | 16 (Query) / 8 (KV) - Grouped Query Attention |
| **Head Dimension** | 128 |
| **MLP (Intermediate)** | 6,144 (SwiGLU) |
| **Text Vocabulary** | 151,936 tokens |
| **Codec Vocabulary** | 3,072 codes |
| **Max Context** | 32,768 tokens |
| **RoPE Theta** | 1,000,000 (multimodal, interleaved) |

### Code Predictor - 5-layer Transformer

| Component | Details |
|-----------|---------|
| **Layers** | 5 |
| **Hidden Size** | 1,024 |
| **Attention Heads** | 16 (Query) / 8 (KV) |
| **Intermediate** | 3,072 |
| **Code Groups** | 16 (generates codes 1-15 in parallel from code 0) |

### Speech Decoder (Vocos)

| Component | Details |
|-----------|---------|
| **Architecture** | 8-layer Transformer + 2x Upsampling + Vocos vocoder |
| **Codebook** | 512-dim embeddings |
| **Output** | 24 kHz 16-bit PCM WAV |

## Synthesis Pipeline

```
Text → Tokenize → Talker (28 layers, autoregressive)
                       ↓
                  Code 0 sequence (12.5 Hz)
                       ↓
              Code Predictor (5 layers)
                       ↓
              16 code groups per timestep
                       ↓
              Speech Decoder (Vocos)
                       ↓
              24 kHz WAV audio
```

## Files

```
model/
  qora-tts.exe        - 4.0 MB    Inference engine
  model.qora-tts      - 1.5 GB    Q4 quantized weights
  tokenizer.json      - 11 MB     Tokenizer (151,936 vocab)
  config.json         - 4.9 KB    Model configuration
  README.md           - This file
```

## Usage

```bash
# Basic TTS (default speaker: ryan)
qora-tts.exe --load model.qora-tts --text "Hello, how are you today?"

# Choose speaker and output file
qora-tts.exe --load model.qora-tts --text "Good morning!" --speaker serena --output greeting.wav

# Different language
qora-tts.exe --load model.qora-tts --text "Bonjour le monde" --speaker dylan --language french

# Control audio length
qora-tts.exe --load model.qora-tts --text "Short text" --max-codes 200
```

### CLI Arguments

| Flag | Default | Description |
|------|---------|-------------|
| `--load <path>` | - | Load from .qora-tts binary (fast, ~2-3s) |
| `--model-path <path>` | `.` | Path to safetensors model directory |
| `--text <text>` | "Hello, how are you today?" | Text to synthesize |
| `--speaker <name>` | ryan | Speaker voice (see list below) |
| `--language <name>` | english | Target language |
| `--output <path>` | output.wav | Output WAV file path |
| `--max-codes <n>` | 500 | Max audio code timesteps (~n/12.5 seconds) |
| `--f16` | off | Use F16 weights instead of Q4 |
| `--save <path>` | - | Save model as .qora-tts binary |

### Available Speakers

| Speaker | Language | ID |
|---------|----------|-----|
| **ryan** | English | 3061 |
| **serena** | English | 3066 |
| **vivian** | English | 3065 |
| **aiden** | English | 3062 |
| **eric** | English | 3063 |
| **uncle_fu** | Chinese | 3057 |
| **ono_anna** | Japanese | 3064 |
| **sohee** | Korean | 3067 |
| **dylan** | Beijing Dialect | 3060 |

### Supported Languages

English, Chinese, German, Italian, Portuguese, Spanish, Japanese, Korean, French, Russian, Beijing Dialect, Sichuan Dialect

## Performance Benchmarks

**Test Hardware:** Windows 11, CPU-only (no GPU acceleration)

### Inference Speed

| Phase | Time | Details |
|-------|------|---------|
| **Model Load** | ~20-23s | From .qora-tts binary (1.5 GB Q4) |
| **Prefill** | ~18s | Process prompt tokens (17-23 tokens) |
| **Code Generation** | ~1,200s | 484-499 steps at 0.4 codes/s |
| **Audio Decode** | ~600-800s | Vocos decoder (8 transformer + upsampling) |
| **Memory** | 1,511 MB | Total loaded model size |

### Audio Output

| Metric | Value |
|--------|-------|
| **Sample Rate** | 24,000 Hz |
| **Bit Depth** | 16-bit PCM |
| **Format** | WAV |
| **Audio Rate** | 12.5 Hz codec = ~40s audio from 500 codes |
| **Channels** | Mono |

## Test Results

### Test 1: Short Greeting (Speaker: Ryan)

**Input:** "Hello, how are you today?"

| Metric | Value |
|--------|-------|
| Prompt Tokens | 17 |
| Load Time | 22.9s |
| Prefill Time | 17.7s |
| Code Steps | 484 |
| Code Generation | 1,197.3s (0.4 codes/s) |
| Audio Duration | ~38.7s |
| Result | WAV generated successfully |

### Test 2: Pangram (Speaker: Serena)

**Input:** "The quick brown fox jumps over the lazy dog."

| Metric | Value |
|--------|-------|
| Prompt Tokens | 20 |
| Load Time | 18.9s |
| Prefill Time | 19.2s |
| Code Steps | 499 |
| Code Generation | 1,196.4s (0.4 codes/s) |
| Audio Duration | ~39.9s |
| Result | WAV generated successfully |

### Test 3: Science Text (Speaker: Vivian)

**Input:** "Quantum computing represents a fundamental shift in how we process information."

| Metric | Value |
|--------|-------|
| Prompt Tokens | 23 |
| Load Time | 20.8s |
| Prefill Time | 18.2s |
| Code Steps | 484 |
| Code Generation | 1,193.6s (0.4 codes/s) |
| Audio Duration | ~38.7s |
| Result | WAV generated successfully |

### Test Summary

| Test | Speaker | Text Length | Codes | Audio Length | Status |
|------|---------|-------------|-------|-------------|--------|
| Greeting | Ryan | 25 chars | 484 | ~38.7s | PASS |
| Pangram | Serena | 44 chars | 499 | ~39.9s | PASS |
| Science | Vivian | 79 chars | 484 | ~38.7s | PASS |

All three tests generated valid WAV audio files at 24 kHz with different speakers.

## Technical Details

### Quantization

- **Q4**: 4-bit symmetric, group_size=32 (default, 1.5 GB)
- **F16**: Half-precision floats (optional, ~3 GB)
- 6.4x compression ratio from f32 to Q4

### Generation Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| Temperature | 0.9 | Sampling randomness |
| Top-K | 50 | Top-K sampling |
| Repetition Penalty | 1.05 | Prevents repetitive codes |

### Special Tokens

| Token | ID | Purpose |
|-------|-----|---------|
| TTS BOS | 151,672 | Start of TTS sequence |
| TTS EOS | 151,673 | End of TTS sequence |
| Codec BOS | 2,149 | Start of codec sequence |
| Codec EOS | 2,150 | End of codec sequence |

## QORA Model Family

| Engine | Model | Params | Size (Q4) | Purpose |
|--------|-------|--------|-----------|---------|
| **QORA** | SmolLM3-3B | 3.07B | 1.68 GB | Text generation, reasoning, chat |
| **QORA-TTS** | Qwen3-TTS | 1.84B | 1.5 GB | Text-to-speech synthesis |
| **QORA-Vision (Image)** | SigLIP 2 Base | 86M | 58 MB | Image embeddings, zero-shot classification |
| **QORA-Vision (Video)** | ViViT Base | 89M | 60 MB | Video action classification |

All engines are pure Rust, CPU-only, single-binary executables with no Python dependencies.

## Building from Source

```bash
cd QORA-TTS
cargo build --release

# Convert from safetensors to .qora-tts binary:
./target/release/qora-tts.exe --model-path ../Qwen3-TTS/ --save model/model.qora-tts
```

---

*Built with QORA - Pure Rust AI Inference*
