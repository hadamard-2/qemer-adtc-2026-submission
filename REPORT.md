# Technical Report — Offline Coding Assistant (RAG-Grounded)

**Team ID:** qemer
**Domain:** coding_assistants
**Model:** Qwen3.5-0.8B-Q4_K_M

---

## Problem

This submission targets an offline coding assistant for developers and students in low-connectivity African contexts, where reliable access to online documentation, Stack Overflow, or hosted assistants cannot be assumed. The assistant is designed to teach rather than only answer — code generation, debugging help, and programming tutoring are all grounded in retrieval over technical documentation, so explanations trace back to the actual language and library references a learner is working from. An unreliable answer is worse than none in this setting, which is why retrieval is treated as load-bearing: the model is not relied on to carry accurate, version-specific API detail in its weights, so grounding is what makes tutoring trustworthy rather than an add-on.

**Scope of what this submission measures:** the retrieval-augmented application (embeddings, vector store, retrieval pipeline) is in active development and is not part of the artifact measured by the profiler run below. What is measured here is the base model layer — the component that runs under `llama.cpp` and is subject to the competition's hardware and runtime constraints. The benchmarks and design decisions in this report describe that layer only.

---

## Design Decisions

**Base model:** Qwen3.5-0.8B, quantized to GGUF Q4_K_M (752M actual parameters, verified against the claimed 0.8B via the profiler's GGUF header check — well within the ±15% fraud-check tolerance).

**Why this size and quantization:** four candidates were evaluated end-to-end — Qwen3.5-0.8B, Qwen3.5-2B, Qwen3.5-4B, and a competing base model (LFM2.5-2.6B) — each profiled for throughput, memory, and accuracy. Q4_K_M was chosen over a higher-fidelity quant (UD-Q4_K_XL) after confirming the two produced statistically indistinguishable accuracy (both within noise on a 50-question smoke test) while Q4_K_M gave a smaller, more predictable memory footprint.

**Why 0.8B over larger candidates:** under the organizers' published scoring formula (`S_total = 0.50·S_acc + 0.30·S_perf + 0.20·S_eff − P_thermal`, with `S_perf` normalized against a 15.0 tokens/sec reference and `S_eff` against a 7 GB ceiling), model size trades directly against throughput and memory score. Measured on the organizers' own audit toolchain (see Benchmarks), the 0.8B model scored roughly 13 points higher on the combined `S_perf`/`S_eff` component than the 2B model, purely from being smaller and faster. Against that, the 0.8B trailed the 2B by only ~4.6 accuracy points on a 500-question general-knowledge benchmark (ARC-Easy) — a real but modest gap, and one that general-knowledge multiple-choice does not directly transfer from to free-form code generation. The Qwen3.5-4B and LFM2.5-2.6B candidates were both dominated on the measured axes (2-4x the memory, 2-4x slower) without a countervailing accuracy advantage large enough to justify the cost, and were dropped from consideration.

**Alternatives considered and rejected:**
- **Qwen3.5-4B Q4_K_M** — 4.08 GB peak RSS, 2.2x slower than the 2B on identical hardware; rejected for cost with no measured accuracy gain large enough to offset it.
- **LFM2.5-2.6B Q4_K_M** — largest memory footprint of any candidate (2.9 GB) and the lowest ARC-Easy score observed (0.46 vs 0.72 for the 2B on the same 50-question run); rejected.
- **UD-Q4_K_XL quantization of the 2B** — statistically indistinguishable accuracy from Q4_K_M at a larger file size; rejected in favor of Q4_K_M.

---

## Constraints

- **Target profile:** the ADTC Standard Laptop (4 vCPU, 8 GB RAM, integrated GPU only, CPU-only inference via `llama.cpp`).
- **No GPU acceleration assumed for scoring.** Development and comparison benchmarking used a mix of local hardware (a 12-thread hybrid laptop CPU and an older 4-thread Skylake laptop) and the organizers' own published Docker audit image, to separate what is host-dependent from what will transfer to the evaluation environment.
- **Offline requirement:** per the competition rules, the model must run with zero external network dependencies once profiling begins. No retrieval backend is invoked during the measured run — the RAG layer is deliberately out of scope for this artifact.
- **8 GB hard ceiling:** exceeding it during evaluation is an automatic disqualification, not a score deduction, which weighted quantization and model-size decisions toward measured headroom rather than best-case estimates.

---

## Benchmarks

Two benchmark passes are reported: (1) on local development hardware, using the installed `adtc-profiler`, and (2) on the organizers' published audit Docker image (`Africa-Deep-Tech-Foundation/adtc-profiler`, built from source), which uses a CPU-only `llama.cpp` build with AVX/AVX2/FMA/F16C disabled for parity across evaluation hardware — a materially different (slower) performance profile than a locally optimized build.

| Metric | Local dev hardware | Audit-toolchain container |
|---|---|---|
| Machine | 13th Gen Intel i7-1355U, 31 GB RAM, Ubuntu 24.04 | Same host, Docker (`--memory=7.5g`, audit `llama.cpp` build) |
| Peak RSS | 1.86 GB (2B) / — | 0.69 GB |
| Time to first token | ~14.5 s (2B, cold) | ~30.3 s |
| Generation speed | 20.4 tok/s (accuracy-only host run) | 10.3 tok/s (`-t 4`, representative of a 4 vCPU audit host) |
| ARC-Easy accuracy (n=500) | 0.604 (acc_norm) | not re-run in container; accuracy is host-independent (confirmed identical across three separate machines/builds during development) |
| Thermal throttling | None observed at this model size on the audit build (peak 77°C, well under the 85°C penalty threshold) | None observed |

These are self-reported development benchmarks. Official scores are measured by the ADTC profiler on the standard evaluation machine. Local development hardware consistently overstated throughput relative to the audit toolchain by roughly 2.4x, due to SIMD instruction sets (AVX2/FMA/F16C) present on development hardware but disabled in the organizers' evaluation build — a gap discovered and quantified during this evaluation, and the reason the audit-container column above, not the local-hardware column, should be treated as representative of the scored result.

---

*Model selection in this report reflects measured throughput, memory, and general-knowledge accuracy (ARC-Easy) trade-offs across four candidate models. It does not yet reflect a side-by-side comparison of model outputs on this submission's own two test prompts — that comparison, and any resulting change of model, is expected before final submission.*
