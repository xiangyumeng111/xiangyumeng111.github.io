---
layout: page
title: Function-Level Verification for LLM-Repaired Decompiled Code
description: Master's thesis project on function-level feedback for LLM-assisted decompiled C/C++ repair.
img: assets/img/decompilation-code-repair-cover.png
importance: 1
category: research
related_publications: false
---

<style>
.decomp-project-hero {
  margin: 1.25rem 0 1.75rem;
}
.decomp-project-hero img,
.decomp-project-figure img {
  width: 100%;
  height: auto;
  border-radius: 10px;
  border: 1px solid var(--global-divider-color);
  background: var(--global-bg-color);
}
.decomp-project-caption {
  margin-top: 0.45rem;
  color: var(--global-text-color-light);
  font-size: 0.9rem;
  line-height: 1.45;
}
.decomp-project-kpis {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 0.75rem;
  margin: 1.25rem 0 1.75rem;
}
.decomp-project-kpi {
  border: 1px solid var(--global-divider-color);
  border-radius: 8px;
  padding: 0.85rem 0.95rem;
  background: rgba(127, 127, 127, 0.035);
}
.decomp-project-kpi .label {
  color: var(--global-text-color-light);
  font-size: 0.82rem;
  margin-bottom: 0.25rem;
}
.decomp-project-kpi .value {
  font-weight: 650;
  line-height: 1.28;
}
@media (max-width: 720px) {
  .decomp-project-kpis {
    grid-template-columns: 1fr;
  }
}
</style>

<div class="decomp-project-hero">
  <img src="{{ '/assets/img/decompilation-code-repair-cover.png' | relative_url }}" alt="Hex-Rays splits a program into functions, ChatGPT repairs each function, and repaired functions are recomposed into a complete program">
  <div class="decomp-project-caption">Project overview: Hex-Rays decompiles a program into functions, ChatGPT repairs each function in isolation, and the repaired functions are recomposed into a complete C/C++ program.</div>
</div>

This is my Master's thesis project at ICT, CAS. It studies how to give LLM-assisted repair of decompiled C/C++ code a more useful correctness signal than compilation alone. The research details and evaluation are being prepared for publication.

<div class="decomp-project-kpis">
  <div class="decomp-project-kpi"><div class="label">Current result</div><div class="value">37 / 49 <code>ls</code> functions repaired</div></div>
  <div class="decomp-project-kpi"><div class="label">Focus</div><div class="value">Function-level feedback for decompiled C/C++ repair</div></div>
  <div class="decomp-project-kpi"><div class="label">Next step</div><div class="value">Broader empirical evaluation</div></div>
</div>

## Problem

Industrial decompilers such as Hex-Rays and Ghidra are excellent reverse-engineering tools, but their output is designed for human understanding rather than direct recompilation. Decompiled pseudocode can contain wrong types, invalid declarations, incomplete function-call parameters, and semantic drift. Prior LLM-based repair systems show that models can fix many of these issues, but **verification overhead** remains the bottleneck.

<div class="decomp-project-figure">
  <img src="{{ '/assets/img/spec2017-cpp-runtime.png' | relative_url }}" alt="Published SPEC CPU2017 C++ workload times totaling 44 minutes and 25 seconds when run sequentially">
</div>

<p class="decomp-project-caption">Source: <a href="https://www.spec.org/cpu2017/results/res2017q4/cpu2017-20171031-00339.html">published SPEC CPU2017 result</a> (56 copies; median elapsed seconds). SPEC lists these four workloads as C++ benchmarks in its <a href="https://www.spec.org/cpu2017/Docs/">benchmark index</a>.</p>

A program-level test answers a coarse question: "does the full rebuilt program still pass the test?" On one published SPEC CPU2017 result, four C++ workloads alone add up to about 44 minutes when run sequentially. The figure uses the reported median times for <code>omnetpp</code>, <code>xalancbmk</code>, <code>deepsjeng</code>, and <code>leela</code>; these are machine- and configuration-specific results, but they illustrate why rerunning a large workload after every repair is **expensive**. A program-level test is also **weak** as a per-function signal: it reveals that the program failed, but not which repair caused the failure.

This motivates a stricter path:

1. make the decompiled function compilable;
2. verify function-level equivalence between repaired function <code>G</code> and original binary function <code>F</code>;
3. only then recompose the repaired functions into a complete C/C++ program.

<div class="decomp-project-figure">
  <img src="{{ '/assets/img/decompilation-code-repair-hero-v3.png' | relative_url }}" alt="Function-level record and replay verification workflow">
</div>

<p class="decomp-project-caption">Function-level record-and-replay verification workflow used to check repaired decompiled functions against original binary behavior.</p>

## Intended impact

The project aims to make verification feedback more practical for decompiled-code repair. Rather than relying only on a late, all-or-nothing program outcome, it is designed to provide more focused feedback during repair and reserve whole-program runs for later validation. The expected effect is a shorter iteration cycle for long-running workloads and failures that are easier to act on.

## Current status

An end-to-end research prototype is operational and has moved beyond a paper-only design. On GNU Coreutils <code>ls</code>, 37 of the 49 selected decompiled functions have been repaired and passed the current function-level validation workflow.

That result is intentionally scoped: it is evidence that the repair-and-check loop works on a non-trivial real utility, not yet a completed benchmark-wide evaluation. The current workflow can run a complete single-function repair experiment, repeat validation across multiple inputs, and separate behavioral mismatches from ordinary build failures.

The remaining work is to finish the unresolved <code>ls</code> functions and expand the evaluation to a larger set of programs. Implementation details, experimental methodology, and broader quantitative results will be released with the accompanying paper.
