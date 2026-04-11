# Android ASR Benchmark

On-device speech recognition benchmarking for Android e-ink devices.

Compares streaming and offline ASR models on **unseen evaluation data** — [Earnings-22](https://github.com/revdotcom/speech-datasets) financial conference calls and [TED-LIUM 3](https://huggingface.co/datasets/distil-whisper/tedlium-long-form) talks. None of the models were trained on this data.

**Device:** Boox Palma 2 Pro, Qualcomm SM6350 (Snapdragon 690), 6 GB RAM, Android 15.

## Results

### Streaming engines (real-time capable)

| Engine | RTF | Size | Earn-22 | TED | Avg WER |
|--------|-----|------|---------|-----|---------|
| Sherpa zipformer streaming (ORT) | 0.14 | 71 MB | 20.0% | 32.0% | 26.0% |
| Vosk (small-en-us-0.15) | 0.22 | 68 MB | 37.3% | 23.8% | 30.6% |
| whisper.cpp (tiny.en q5_1) | 5.98 | 31 MB | 16.0% | 25.4% | 20.7% |

### Offline models (all ORT format)

| Model | Load | Size | RTF | Earn-22 | TED | Avg WER | Native RAM | Timestamps |
|-------|------|------|------|---------|-----|---------|------------|------------|
| Zipformer streaming (LS) | 2.3s | 71 MB | 0.14 | 20.0% | 32.0% | 26.0% | +226 MB | Yes |
| Zipformer 2023-04-01 (LS) | 5.5s | 181 MB | 0.08 | 13.3% | 14.8% | 14.0% | +200 MB | No |
| Zipformer 2023-06-26 (LS) | 3.4s | 68 MB | 0.06 | 18.7% | 30.3% | 24.5% | +226 MB | No |
| Multidataset (LS+GS+CV) | 3.9s | 123 MB | 0.08 | 8.0% | 16.4% | 12.2% | +200 MB | No |
| **GigaSpeech (YouTube/pod)** | **3.7s** | **68 MB** | **0.06** | **6.7%** | 18.0% | **12.3%** | +216 MB | No |
| Paraformer (Alibaba) | 4.1s | 219 MB | 0.08 | 8.0% | 19.7% | 13.8% | +490 MB | No |
| Parakeet TDT 0.6B (NeMo) | - | 631 MB | - | - | - | OOM | +1645 MB | - |

Training data: LS = LibriSpeech (960h), GS = GigaSpeech (10Kh YouTube/podcasts), CV = CommonVoice. Timestamps = whether `OfflineRecognizerResult` returns per-word tokens and timestamps (sherpa-onnx AAR v6.25.12).

### Key findings

- **GigaSpeech is the best offline model** for real-world audio: lowest WER on conference calls (6.7%), smallest on disk (68 MB), fastest ORT load (3.7s), and moderate memory (+216 MB).
- **Multidataset ties on overall WER** (12.2%) but is better on TED talks (16.4% vs 18.0%). Larger on disk (123 MB).
- **ORT format reduces load time 2-4x** with no impact on WER or RTF.
- **No offline model returns per-word timestamps** in sherpa-onnx AAR v6.25.12 (transducer and Paraformer alike). Only the streaming model provides them.
- **Parakeet TDT 0.6B is not viable on mobile** — 1.6 GB native memory causes OOM on a 6 GB device.
- **LibriSpeech benchmarks are misleading** — models trained on LibriSpeech (960h audiobooks) score 0.5% WER on LibriSpeech test-clean but 14% on unseen real-world audio.

## Evaluation datasets

Both datasets are **unseen by all models** — none were used for training.

- **[Earnings-22](https://github.com/revdotcom/speech-datasets)** — Financial earnings conference calls from Rev.ai. Diverse accents, teleconference audio quality, domain-specific jargon.
- **[TED-LIUM 3](https://huggingface.co/datasets/distil-whisper/tedlium-long-form)** — TED talks. Clear academic speech with domain-specific vocabulary.

## ORT model repos

All offline models converted to ORT format and hosted on HuggingFace:

- [sherpa-onnx-ort-zipformer-en-2023-04-01](https://huggingface.co/imedemi/sherpa-onnx-ort-zipformer-en-2023-04-01)
- [sherpa-onnx-ort-zipformer-en-2023-06-26](https://huggingface.co/imedemi/sherpa-onnx-ort-zipformer-en-2023-06-26)
- [sherpa-onnx-ort-multidataset-transducer-2023-05-04](https://huggingface.co/imedemi/sherpa-onnx-ort-multidataset-transducer-2023-05-04)
- [sherpa-onnx-ort-zipformer-gigaspeech-2023-12-12](https://huggingface.co/imedemi/sherpa-onnx-ort-zipformer-gigaspeech-2023-12-12)
- [sherpa-onnx-ort-paraformer-en-2024-03-09](https://huggingface.co/imedemi/sherpa-onnx-ort-paraformer-en-2024-03-09)

## Files

- `TranscriptionBenchmarkTest.kt` — Main benchmark: all engines on LibriSpeech + unseen eval data, two-pass validation, offline model comparison, memory profiling, latency measurement.
- `WhisperBenchmarkTest.kt` — Whisper-specific benchmarks: model sizes, thread counts, VAD impact.

These are Android instrumented tests (`androidTest`) designed to run on a physical device. They download models and test data on first run.

## Running

```bash
# All transcription benchmarks
./gradlew connectedDebugAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.class=com.writer.perf.TranscriptionBenchmarkTest

# Specific test
./gradlew connectedDebugAndroidTest \
  -Pandroid.testInstrumentationRunnerArguments.class=com.writer.perf.TranscriptionBenchmarkTest \
  -Pandroid.testInstrumentationRunnerArguments.method=benchmark_offline_models
```

Results appear in `adb logcat -s TransBenchmark`.
