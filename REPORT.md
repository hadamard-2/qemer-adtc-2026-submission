# Technical Report — Qemer Offline Coding Assistant Model Layer

**Team ID:** qemer
**Domain:** coding_assistants
**Model:** Qwen3.5-0.8B-Q4_K_M

---

## Problem

Qemer is an offline coding-assistant project for developers and students in low-connectivity African contexts, where reliable access to online documentation, Stack Overflow, or hosted assistants cannot be assumed. The intended application helps with code generation, debugging, and programming tutoring while keeping inference local.

The broader Qemer project will ground answers in locally retrieved technical documentation so that version-specific explanations can cite the relevant reference material instead of relying only on model memory. That retrieval application is a separate, early-stage companion project. This submission packages and measures the base-model layer only: a local GGUF model executed through `llama.cpp` under the competition's runtime and hardware constraints.

---

## Design Decisions

**Base model:** Qwen3.5-0.8B, quantized to GGUF Q4_K_M. The profiler's GGUF header check reports 752,393,024 parameters, which is within the ±15% tolerance of the declared 0.8B parameter estimate.

**Why this size and quantization:** the selected model prioritizes CPU-only responsiveness and memory headroom on an 8 GB laptop. Q4_K_M provides a compact local artifact while preserving a standard GGUF format compatible with `llama.cpp`. The final Docker audit run measured 0.69 GB peak RSS, leaving substantial room below the 7 GB profiling ceiling.

**Alternatives considered:** larger Qwen3.5 variants and alternative 4-bit quantizations were explored during development. They increase memory use and CPU generation cost, so the 0.8B Q4_K_M variant was selected for this submission's target hardware profile. Model quality will ultimately be assessed on the submitted coding prompts and the organizers' hidden prompts.

---

## Constraints

- **Target profile:** ADTC Standard Laptop — four CPU cores, 8 GB RAM, integrated graphics, and CPU-only inference through `llama.cpp`.
- **Memory ceiling:** the model must remain below the 7 GB peak-RSS profiling budget inside the 8 GB laptop profile; an out-of-memory failure disqualifies a submission.
- **Offline inference:** the model is downloaded before profiling, then the profiler runs the local GGUF file without external network dependencies. The benchmark uses `llama.cpp` with GPU offload disabled.
- **Reproducibility:** the final audit-style run used the organizers' Docker image built from source, a 7.5 GB Docker memory limit, and four pinned CPU cores.

---

## Benchmarks

The final reported telemetry pass used Qwen3.5-0.8B-Q4_K_M in an audit-style Docker run and skipped its optional accuracy stage. A separate development-only ARC-Easy run is reported below; official accuracy is assessed from model responses by the organizers.

| Metric | Audit-toolchain container |
|---|---|
| Runtime | Docker, Debian 13, 7.5 GB memory limit, 4 pinned CPU cores |
| Peak RSS | 0.69 GB |
| Time to first token | 53.88 s |
| Generation speed | 7.41 tok/s |
| Development accuracy | ARC-Easy (n=500): 0.604 acc_norm; reproduced on two development machines |
| Thermal throttling | None observed; 66°C peak |

This is a self-reported development measurement. The container is a conservative reproducibility check because it limits memory and CPU availability, but it is still not the organizers' exact reference laptop. Official scores will be measured on the standard evaluation hardware.

The ARC-Easy result is a development-only general-reasoning check using lm-eval-harness; it is not an official ADTC score and does not replace evaluation of coding-assistant responses on the submitted and hidden prompts.
