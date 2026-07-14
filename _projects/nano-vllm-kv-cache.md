---
layout: page
title: nano-vllm — KV Cache FP8 Quantization
description: Online FP8 quantization of KV cache in LLM inference, implemented with Triton kernels and CUDA Graph support.
img: assets/img/nano-vllm-kv-cache-hero.png
importance: 3
category: engineering
---

<style>
.project-detail-hero {
  margin: 1.25rem 0 1.75rem;
}
.project-detail-hero img {
  width: 100%;
  height: auto;
  border-radius: 10px;
  border: 1px solid var(--global-divider-color);
  background: var(--global-bg-color);
}
.project-detail-caption {
  margin-top: 0.45rem;
  color: var(--global-text-color-light);
  font-size: 0.9rem;
  line-height: 1.45;
}
.project-detail-kpis {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 0.75rem;
  margin: 1.25rem 0 1.75rem;
}
.project-detail-kpi {
  border: 1px solid var(--global-divider-color);
  border-radius: 8px;
  padding: 0.85rem 0.95rem;
  background: rgba(127, 127, 127, 0.035);
}
.project-detail-kpi .label {
  color: var(--global-text-color-light);
  font-size: 0.82rem;
  margin-bottom: 0.25rem;
}
.project-detail-kpi .value {
  font-weight: 650;
  line-height: 1.28;
}
@media (max-width: 720px) {
  .project-detail-kpis {
    grid-template-columns: 1fr;
  }
}
</style>

<div class="project-detail-hero">
  <img src="{{ '/assets/img/nano-vllm-kv-cache-hero.png' | relative_url }}" alt="KV cache FP8 quantization pipeline: write path, storage reduction, read path, fused attention next step">
</div>

I started this independent nano-vllm implementation project in 2026.06 to build a deeper, systems-level understanding of LLM inference by modifying the KV-cache path directly. The implementation quantizes keys and values when they are written to cache and dequantizes them when attention reads them back.

<div class="project-detail-kpis">
  <div class="project-detail-kpi"><div class="label">Memory</div><div class="value">KV cache reduced to about 50% of the unquantized size</div></div><div class="project-detail-kpi"><div class="label">Accuracy check</div><div class="value">Cosine similarity >= 0.99; max element error < 0.3</div></div><div class="project-detail-kpi"><div class="label">Latency snapshot</div><div class="value">3.4 ms quantized path vs 3.2 ms unquantized</div></div>
</div>

## Motivation

KV cache memory is a major bottleneck in long-context decoding. Weight quantization helps model storage, but it does not solve the growing memory footprint of cached attention keys and values. Quantizing the KV cache to FP8 targets this runtime memory directly, enabling larger batches or longer contexts on the same hardware.

## Implementation

The current pipeline is deliberately explicit:

1. **Write path:** when an attention layer writes new key/value tensors, a Triton kernel scales and casts the activations into FP8E4M3N before storing them. Quantization is applied per attention head rather than per token, with one scale computed for each head.
2. **Read path:** before attention computation, another Triton kernel loads FP8 cache values and converts them back to BF16.
3. **CUDA Graph integration:** quantization and dequantization kernels are included in CUDA Graph execution to reduce repeated launch overhead during decode.

Ampere does not provide native FP8 tensor-core support, so the project includes manual FP8E4M3N to BF16 type handling in Triton rather than relying on hardware-native FP8 execution.

## Measurement

The current target model is Qwen3-0.6B. In the tested path, quantized and unquantized KV cache values reach cosine similarity of at least 0.99, with maximum per-element error below 0.3. The measured latency is 3.4 ms with quantization versus 3.2 ms without it.

That overhead is expected at this stage: dequantized BF16 values are written back to HBM and then read again by attention. The next optimization is a fused dequantization plus [FlashAttention kernel](https://github.com/Dao-AILab/flash-attention) that avoids the intermediate BF16 HBM round trip. The FP8 to BF16 conversion still exists, but the extra write/read should disappear.

## What I Learned

This project sits below model API usage and inside the inference runtime: cache layout, attention dataflow, CUDA Graph constraints, and Triton kernel behavior all matter. The useful part is not just saving memory; it is building intuition for how small kernel-bound decisions show up as decode latency.

## Technologies

Triton, CUDA Graph, PyTorch, FP8E4M3N, BF16, KV cache optimization, Qwen3-0.6B.
