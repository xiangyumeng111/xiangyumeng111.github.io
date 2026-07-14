---
layout: page
title: Binary Translation Platform Engineering
description: An x86-64-to-RISC-V translation platform, presented as a set of independently scoped compiler and tooling projects.
img: assets/img/binary-translator-hero.svg
importance: 2
category: research
---

<style>
.project-detail-hero { margin: 1.25rem 0 1.75rem; }
.project-detail-hero img { width: 100%; height: auto; border-radius: 10px; border: 1px solid var(--global-divider-color); background: var(--global-bg-color); }
.project-detail-caption { margin-top: 0.45rem; color: var(--global-text-color-light); font-size: 0.9rem; line-height: 1.45; }
.project-detail-toc { margin: 1.6rem 0 1.5rem; padding: 0; border: 0; background: transparent; }
.project-detail-toc strong { display: block; margin-bottom: 0.4rem; }
.project-detail-toc ol { margin: 0; padding-left: 1.25rem; }
.project-detail-toc li { margin: 0.15rem 0; }
</style>

<div class="project-detail-hero">
  <img src="{{ '/assets/img/binary-translator-hero.svg' | relative_url }}" alt="Overview of an x86-64 to RISC-V binary translation workflow">
</div>

The binary translator is a long-running platform project that statically translates x86-64 programs into optimized RISC-V code. Its work spans static translation, LLVM backend integration, register allocation, control-flow recovery, indirect branches, and jump-table recognition. The sections below document the implementation status and validated results for each workstream.

<div class="project-detail-toc">
  <strong>Subprojects</strong>
  <ol>
    <li><a href="{{ '/projects/dbt-vector-register-allocation/' | relative_url }}">Vector Register Allocation Patent</a></li>
    <li><a href="{{ '/projects/dbt-ragreedy-register-allocation/' | relative_url }}">LLVM RAGreedy Vector Allocation</a></li>
    <li><a href="{{ '/projects/dbt-indirect-control-flow-profiling/' | relative_url }}">Indirect-Control-Flow Profiling</a></li>
    <li><a href="{{ '/projects/dbt-static-translation-dataflow/' | relative_url }}">Static Translation Data-Flow Optimization</a></li>
    <li><a href="{{ '/projects/dbt-switch-case-recognition/' | relative_url }}">Switch-Case / Jump-Table Recognition</a></li>
  </ol>
</div>

## Platform Context

The system translates x86-64 binaries to RISC-V code and combines static translation with dynamic fallback. That setting creates several distinct engineering problems: preserving architectural register state across translation boundaries, discovering targets that static analysis cannot see, controlling translation latency, and reconstructing high-level control-flow structures from binary code.

The pages above keep those problems separate. They intentionally do not claim a single aggregate performance result for the whole platform.

## Related Infrastructure

Additional work included a three-level function/TB/instruction binary-search method for register-allocation failures, tooling for mapping emitted code back to functions and translation blocks, testing automation, and MachineIR instrumentation. Those are retained as supporting platform work until their implementation materials can be independently reviewed and presented with the same level of evidence.

## Technologies

C/C++, LLVM MachineIR and backend passes, RISC-V, x86-64, Intel Pin, perf, static and dynamic binary translation, compiler debugging.
