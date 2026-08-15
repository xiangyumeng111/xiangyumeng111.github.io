---
layout: about
title: Home
permalink: /
nav: false
nav_order: 1
subtitle:

profile:
  align: right
  image: about-profile.jpg
  image_circular: false
  more_info: >
    <p>Institute of Computing Technology, CAS</p>
    <p>Beijing, China</p>
    <p><a href="mailto:sekiro_meng@outlook.com">sekiro_meng@outlook.com</a></p>
    <p><a href="https://github.com/apollo600">GitHub</a></p>

selected_papers: false
social: false

announcements:
  enabled: false
  scrollable: true
  limit: 5

latest_posts:
  enabled: true
  scrollable: true
  limit: 3
---

I am a Master's student at the [Institute of Computing Technology, Chinese Academy of Sciences](http://www.ict.ac.cn/) (ICT, CAS), advised in the area of program analysis and binary translation. My work sits at the intersection of **compilers, systems, and AI** — I build things that make code run faster, whether that means rewriting an LLVM register allocator, optimizing GEMM kernels past Intel MKL, or quantizing KV caches for LLM inference.

## Research

My primary research thread is **LLM-driven decompiled code repair with function-level independent verification**. Decompilers (Hex-Rays, Ghidra) produce pseudo-code that cannot be directly recompiled. I built a _Record & Replay_ mechanism that verifies LLM-repaired functions in isolation — without running the entire program — by capturing function context via Intel Pin and replaying with injected return values. The prototype has verified 37/49 Coreutils `ls` functions and targets 100% program-level equivalence (vs. the PCodeTrans baseline of 25.92%).

I also contributed to a **binary translator for custom RISC-V silicon** (x86-64 → RISC-V), a multi-year national project that has served multiple companies and is now being adapted for BOSC (Beijing Open Source Chip Research Institute), with XiangShan Kunming Lake on the roadmap.

## Engineering

On the engineering side, I optimize systems at the hardware-software boundary:

- **nano-vllm: KV Cache FP8 Quantization** — Implemented online KV cache FP8 quantization in Triton to understand LLM inference internals, reducing KV cache memory usage by ~50% with cosine similarity ≥ 0.99 vs. unquantized. Developing fused dequant + FlashAttention kernel to avoid writing dequantized BF16 values to HBM and reading them back for attention.
- **CPU GEMM Optimization** — Achieved **99.29 GFLOPS single-core (FP64)** on Intel Xeon Platinum 9242, outperforming Intel MKL by 21% and OpenBLAS by 28%. Demonstrates hands-on experience with micro-architectural optimization: multi-level tiling, cache blocking, SIMD intrinsics (AVX-512), and register-level tuning.
- **SFI-BPF** — Built a Software Fault Isolation sandbox for eBPF programs, aggregating BPF memory regions through page-table mapping and instrumenting memory accesses via verifier-driven analysis. Reproduced and mitigated 3 high-severity kernel CVEs on Linux 5.10.

## Publications

- [**HIVE**](/assets/pdf/hive-usenix24.pdf) — USENIX Security '24 (third author; second student author)
- **Patent:** "Binary Translation Oriented Register Allocation Method Based on Key Mapping and State Machine" (filed, first inventor), 2026

---

_I am currently seeking a PhD position — my primary focus — in LLM for performance engineering, AI infrastructure (compiler, system software, and container layers). I am also open to new-grad roles in AI chip toolchains and AI compiler engineering. My background in operating systems and compilers enables fast transition across these domains. Feel free to get in touch._
