# Memory Pressure in Long-Form Whisper Transcription: A Reproducible Engineering Investigation

**Manuscript status:** Technical draft — empirical findings included; final executable source appendix still requires a byte-for-byte review against the tested reference script before public release.  
**Last updated:** 2 November 2026

## Abstract

Whisper is commonly used for speech recognition with word-level timestamps. Problems can appear when an audio input exceeds the model's nominal 30-second encoder window, particularly when token-level timestamp generation is enabled. These problems are often described as out-of-memory (OOM) failures, but memory pressure, sequence-length constraints, timestamp-generation behavior, and boundary-merging errors are different phenomena and should not be conflated.

This investigation examines sequential transcription of Persian audio using `givia/whisper-large-v3-turbo-persian` on CPU with float32 inference. The engineering approach is to split long audio into bounded windows, retain a small overlap between adjacent windows, process windows sequentially, convert local timestamps back to the source timeline, and merge duplicate boundary tokens.

The available measurements demonstrate successful processing of a 96.085-second recording and a 10-second diagnostic sample. They do not establish a universal memory threshold or prove a single root cause for every long-input failure. A 60-second measurement and the final verified source listing remain outstanding. The distinction between observed evidence and hypotheses is central to the conclusions.

## 1. Problem statement

Whisper's encoder is designed around a 30-second audio window. At a 16 kHz sample rate, 30 seconds corresponds to 480,000 waveform samples. The model configuration commonly exposes 1,500 encoder positions for this window.

An audio sequence longer than the nominal window therefore needs an explicit long-form strategy. Depending on the inference path and library version, a caller may encounter:

- input-length or positional-limit errors;
- high memory use during feature extraction, attention, or generation;
- increased runtime from token-level timestamp generation;
- missing or incomplete timestamps near a cut;
- duplicated or omitted words where adjacent windows meet.

These symptoms have different causes. An OOM exception is evidence of an allocation failure; a high resident-memory measurement is evidence of memory pressure, not by itself proof that an OOM occurred. A timestamp overlap is a merge/temporal consistency issue, not an OOM.

## 2. Why word-level timestamps change the workload

In Hugging Face Transformers, Whisper supports segment timestamps and word-level timestamps. The documented `return_timestamps="word"` path requests token timestamps and segment information; word timing is then approximated using Dynamic Time Warping (DTW) over cross-attention information. It is not equivalent to an independent forced-alignment system.

The distinction matters because word-level output requires more than decoding a transcript. It also requires retaining and processing information used to estimate token timing. The pipeline implementation sets `return_token_timestamps=True` and `return_segments=True` for Whisper's word-level mode. That is a documented implementation detail, not proof that timestamp generation alone causes a particular memory spike.

Hugging Face's current pipeline documentation also warns that using `chunk_length_s` with sequence-to-sequence Whisper is experimental and points users toward the model's own long-form generation path. This investigation uses an explicit external windowing strategy around the existing reference inference engine; it should not be interpreted as a claim that this is the only correct Whisper long-form approach.

## 3. Experimental environment

The measurements below were collected in a CPU-only environment:

| Component | Observed configuration |
|---|---|
| Model | `givia/whisper-large-v3-turbo-persian` |
| Runtime | Hugging Face Transformers pipeline |
| PyTorch | `2.15.0+cpu` in the recorded environment |
| Transformers | `5.22.1` in the recorded environment |
| NumPy | `2.4.6` |
| Python | `3.11.9` |
| Device | CPU |
| Inference dtype | float32 |
| Audio sample rate | 16 kHz |
| Language | Persian |
| Timestamp mode | word-level |
| Window size in the long sample | 27 seconds |
| Adjacent overlap | 3 seconds |
| Processing order | sequential |

These are environment observations, not a recommended universal dependency lockfile. Reproduction requires recording the exact package versions, model revision, hardware, operating system, audio preprocessing, and generation parameters.

## 4. Engineering method

The long-form method used in the measured run follows five stages.

### 4.1. Bound the inference window

A long waveform is divided into windows shorter than the nominal 30-second encoder context. The measured long-file configuration used 27-second windows.

### 4.2. Preserve context at the seam

Adjacent windows overlap by 3 seconds. The overlap provides acoustic context around a boundary and gives the merge stage a region in which to identify repeated words.

For a window length (C) and overlap (O), the step between window starts is:

[
S = C - O
]

With (C=27) seconds and (O=3) seconds, the next window starts 24 seconds after the previous one.

### 4.3. Process sequentially

Only one window is sent through inference at a time. This avoids multiplying the model's working memory by batching multiple long windows. Temporary chunk files and inference objects are released after each window where possible.

Sequential processing does not guarantee low memory use: one inference call can still have a high peak. It does, however, avoid deliberately increasing simultaneous inference work.

### 4.4. Restore source-timeline timestamps

Each word timestamp returned for a chunk is local to that chunk. If a chunk starts at offset (T), a local word interval ([a,b]) is mapped to the source timeline as:

[
[a+T, b+T]
]

The resulting intervals can then be merged and inspected in one global timeline.

### 4.5. Merge the overlap

The overlap can cause the same word sequence to appear in both adjacent outputs. A boundary merge should compare token sequences and timestamps, identify repeated material, and retain a single ordered representation.

A merge heuristic is not an acoustic proof. Ambiguous seams should be flagged for review rather than silently altered to make a validation score pass.

## 5. Measurements and evidence

### 5.1. Ten-second diagnostic sample

A 10-second sample was processed as one chunk.

| Metric | Observed result |
|---|---:|
| Audio duration | 10 seconds |
| Chunks | 1 |
| Merged words | 21 |
| ASR call | 48.3268 seconds |
| Job elapsed | 49.36 seconds |
| Timestamp-structure validation | PASS |

A separate model-loading run recorded 14.29 seconds for model initialization. This loading measurement is not part of the 49.36-second job interval.

The result shows that, on the tested CPU, inference dominated the measured job time. It does not predict performance on other hardware or models.

### 5.2. Thirty-second sample

An earlier word-level test on a 30-second sample produced 62 word-level items and passed the timestamp-structure validation used in that test.

The complete per-run memory profile, exact timing breakdown, and full boundary diagnostics are not present in the retained evidence used for this manuscript. The result should therefore be treated as a successful word-timestamp sample, not as a full performance benchmark.

### 5.3. Sixty-second sample

A complete, reproducible 60-second result is not available in the retained test record. No word count, peak-memory value, runtime, or validation outcome is claimed here.

This row is intentionally left open rather than filled with an estimate.

### 5.4. Ninety-six-second sample

A recording of 96.085 seconds was processed using four sequential windows, 27 seconds each with 3 seconds of overlap.

| Window | Source interval | Words | Inference time |
|---|---:|---:|---:|
| 1 | 0–27 s | 57 | 98.21 s |
| 2 | 24–51 s | 60 | 105.34 s |
| 3 | 48–75 s | 57 | 106.00 s |
| 4 | 72–96.085 s | 49 | 94.53 s |
| **Merged** | **0–96.085 s** | **207** | — |

Additional observations from that instrumented run:

- Peak RSS during chunk inference was approximately 10.5–10.8 GB.
- The job's reported elapsed time before report construction was 934.07 seconds.
- No invalid timestamp indices were found.
- No gaps greater than 0.8 seconds were found.
- Two pairs of overlapping word timestamps were detected around the boundary near 72 seconds.
- Timestamp validation therefore returned `pass=false`.

A later end-to-end run through a separate local job-processing path also completed the same 96.085-second input, produced 207 merged words, and reported four of four chunks completed. Its API-recorded interval from start to completion was approximately 6 minutes 39 seconds. These two timings came from different executions and measurement paths; they are not directly comparable and should not be combined into one benchmark claim.

The two timestamp overlaps are a boundary-quality limitation. They are not evidence of an OOM event. The validation rule checks timestamp structure; it does not prove transcription accuracy or completeness.

## 6. Memory observations and interpretation

A model-loading measurement recorded approximately:

| Measurement | RSS |
|---|---:|
| Before loading | 310 MB |
| After loading | 3,469 MB |
| Approximate increase | 3,159 MB |

The long inference run reached sampled peak RSS values around 10.8 GB. Sampling can miss very brief peaks, and RSS includes more than model weights: framework allocations, tensors, audio features, temporary buffers, Python objects, and other process memory all contribute.

These measurements support three cautious conclusions:

1. The tested model has a substantial resident-memory cost even before a long recording is processed.
2. Word-level inference and long-form processing can produce a much higher process peak than model loading alone.
3. Sequential windowing is a practical way to bound input size and avoid simultaneous inference of multiple windows, but it does not by itself guarantee a fixed memory ceiling.

The available record does **not** prove that a specific attention implementation, timestamp option, or single library component is the root cause of all reported memory failures. Such claims require controlled comparisons with one variable changed at a time.

## 7. Limitations

- Only one model family and one Persian model checkpoint are represented.
- Measurements are CPU-only and float32.
- The available audio samples are short; they do not establish behavior for hours-long recordings.
- The 60-second test record is incomplete.
- The 30-second record contains word-count and validation outcome but not a complete memory/time profile.
- Timestamp validation is structural; it does not evaluate semantic transcription accuracy.
- The merge algorithm is heuristic and requires boundary-focused audio review for high-assurance output.
- Peak-memory sampling can miss short-lived peaks.
- Runtime measurements differ between executions and should be repeated under a controlled benchmark protocol.
- The complete tested source file has not yet been appended to this manuscript. It must be reviewed against the exact tested reference script before this draft is labelled fully reproducible.

## 8. Recommended next experiments

A controlled follow-up should use the same model revision, machine, audio preprocessing, package lockfile, and timestamp settings for each duration.

For 10, 30, 60, and 96 seconds, record:

- exact input duration and sample rate;
- number of chunks and their source intervals;
- model load time separately from inference time;
- per-chunk inference time;
- baseline, post-load, peak RSS and USS;
- output word count;
- missing or incomplete timestamp count;
- duplicate/missing words at each seam;
- validation result and a human-reviewed audio sample;
- whether an OOM exception actually occurred.

A useful experiment matrix should also compare timestamp mode (segment versus word) and inference strategy (single long-form generation versus externally windowed sequential generation), changing one factor at a time. Do not infer a cause from a correlation between duration and memory alone.

## 9. Practical conclusions

- Treat the nominal 30-second Whisper window as an architectural constraint to understand, not as a universal guarantee that every longer input will fail in the same way.
- Prefer bounded, sequential processing when the application needs predictable per-window work and explicit progress.
- Use overlap to preserve context, but validate the seam logic with word sequences and audio.
- Keep model loading outside the per-job loop when the service is intended to process multiple jobs sequentially.
- Measure process memory, not only model weight size.
- Separate OOM failures, high memory use, timestamp errors, and merge errors in logs and reports.
- Publish only the measurements actually collected; mark missing experiments as missing.

## References

1. Hugging Face Transformers, *Pipelines — Automatic Speech Recognition*: https://huggingface.co/docs/transformers/main_classes/pipelines
2. Hugging Face Transformers, *Pipeline Tutorial*: https://huggingface.co/docs/transformers/en/pipeline_tutorial
3. Hugging Face Transformers source, *AutomaticSpeechRecognitionPipeline*: https://github.com/huggingface/transformers/blob/main/src/transformers/pipelines/automatic_speech_recognition.py
4. OpenAI, *Robust Speech Recognition via Large-Scale Weak Supervision* (Whisper paper): https://arxiv.org/abs/2212.04356

---

## Publication checklist

- [ ] Insert the exact tested reference script as a complete runnable appendix.
- [ ] Re-run the script from a clean environment and record package/model revisions.
- [ ] Complete the 60-second experiment and recover the full 30-second measurement report if available.
- [ ] Reconcile timing differences between standalone and end-to-end executions using one benchmark protocol.
- [ ] Review the article for any accidental disclosure of unrelated product, application, customer, or service architecture.
- [ ] Only then remove the technical-draft status and publish as a reproducible engineering article.
