---
layout: page
title: Key-Mapped Vector Register Allocation with State Machines
description: Patent-backed register-allocation work that reduces redundant memory synchronization in x86-to-RISC-V binary translation.
img: assets/img/dbt-vector-register-allocation-cover-speedup.png
importance: 3
category: research
---

<style>
.project-detail-hero { margin: 1.25rem 0 1.75rem; }.project-detail-hero img { width: 100%; height: auto; border-radius: 10px; border: 1px solid var(--global-divider-color); background: var(--global-bg-color); }.project-detail-caption { margin-top: 0.45rem; color: var(--global-text-color-light); font-size: 0.9rem; line-height: 1.45; }.project-detail-kpis { display:grid; grid-template-columns:repeat(3,minmax(0,1fr)); gap:.75rem; margin:1.25rem 0 1.75rem; }.project-detail-kpi { border:1px solid var(--global-divider-color); border-radius:8px; padding:.85rem .95rem; background:rgba(127,127,127,.035); }.project-detail-kpi .label { color:var(--global-text-color-light); font-size:.82rem; margin-bottom:.25rem; }.project-detail-kpi .value { font-weight:650; line-height:1.28; } @media (max-width:720px){.project-detail-kpis{grid-template-columns:1fr;}}
</style>

<div class="project-detail-hero">
  <img src="{{ '/assets/img/dbt-vector-register-allocation-cover-speedup.png' | relative_url }}" alt="SPEC2006 and memory synchronization summary for vector register allocation variants">
  <div class="project-detail-caption">Summary table comparing the baseline, no-spill allocation optimization, and spill/store-aware allocation variant.</div>
</div>

The binary translator represents source-architecture register state in memory so values remain correct across partially known control flow. The cost is repeated loads and stores: one source vector register can be recreated as many separate virtual registers, even though it denotes the same architectural value.

## Key Mapping

Before allocation, I map memory-emulated virtual registers that refer to the same source physical register to one key. It has two benifits:

- The allocator then treats their separate instruction-local ranges as one logical value wherever that is valid.
- The mapping also handles the x86 alias relationship between XMM and the low half of YMM registers, so aliases do not create artificial independent values.

The key order gives memory-emulated registers priority ahead of temporary values, because memory-emulated registers correspond to the actual architectural state (<code>XMM12-XMM15</code>), while temporary values are just intermediates. 

When physical registers are exhausted, the allocator estimates the memory traffic caused by spilling each candidate and chooses the least costly target.

<div class="project-detail-hero">
  <img src="{{ '/assets/img/dbt-vector-register-allocation-architecture.svg' | relative_url }}" alt="Architecture diagram for key-mapped vector register allocation with state-machine synchronization">
  <div class="project-detail-caption">Core idea: map fragmented memory-emulated vector registers to architectural keys, allocate with spill-cost priority, then synchronize only when a physical register's represented key changes.</div>
</div>

## State-Machine Synchronization

Each target physical register has a state machine recording which key it currently represents. Rather than emitting memory synchronization around every local allocation event, the method emits the required load, store, or spill only when that state changes. This targets the redundant synchronization introduced by short, fragmented ranges.

## Evaluation

The evaluation focuses on two questions: whether key mapping can reduce fragmentation in memory-emulated vector-register live ranges, and whether the resulting allocator improves SPEC2006 performance after state-machine synchronization is applied.

**Live-interval consolidation.** The first figure shows that fragmented instruction-local active intervals are merged into larger logical live intervals. This gives the allocator a more stable view of the architectural vector-register state. The merge introduces only a small number of spill events, marked by the diamonds and circles in the figure, rather than turning every local range boundary into memory traffic.

<div class="project-detail-hero">
  <img src="{{ '/assets/img/dbt-vector-register-allocation-hero.png' | relative_url }}" alt="Register allocation usage intervals and state-machine allocation view">
  <div class="project-detail-caption">Fragmented active intervals are merged into larger logical intervals, while only a few spill points are introduced by the allocator.</div>
</div>

**SPEC2006 performance.** The second figure compares normalized SPEC2006 speedup before and after the vector-register allocation optimization. The post-optimization bars show the effect of reducing redundant memory synchronization while preserving correctness through the per-register state machines.

<div class="project-detail-hero">
  <img src="{{ '/assets/img/dbt-vector-register-allocation-spec2006-performance.png' | relative_url }}" alt="SPEC2006 normalized performance comparison before and after vector register allocation optimization">
  <div class="project-detail-caption">Normalized SPEC2006 performance comparison before and after the allocation optimization.</div>
</div>

## Technologies

C/C++, x86-64 and RISC-V vector registers, compiler register allocation, live intervals, spill-cost modeling, state machines, SPEC2006.
