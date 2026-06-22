# Deliverables & requirements traceability

This document maps the **technical deliverables** in this repository to the
work items they satisfy. It intentionally reproduces **no commercial terms,
figures, party-specific contractual language, or the underlying agreement** —
only the engineering scope and where each item is met.

## In scope for this repository — KV Cache system & benchmarking

The work item covers optimizing the LMCache Valkey connector for KV-cache
workloads. Unlike typical Valkey usage (objects under 1 MB), KV-cache transfers
are frequently large objects (multiple MB), and projects like LMCache benefit
from client-side optimizations to make Valkey competitive with alternative
storage backends.

| # | Requirement (summary) | Status | Where it's satisfied |
|---|---|---|---|
| 1.1.1 | Establish a baseline for the LMCache Valkey connector, then deliver **≥ 2× improvement in large-object (> 1 MB) throughput** vs. that baseline via retrieval optimizations (parallel fetch / prefetch, optimal I/O-thread config), measured under equivalent workloads. | ✅ Met | `docs/RESULTS.md` (2.3–2.6× GET @ 4 MB, 3 reps; worker-count tuning); `benchmarks/valkey_microbench.py`; `docs/SCENARIOS.md` §1–2 |
| 1.1.2 | Publish the optimized connector, recommended configurations, and benchmark results to the LMCache repository or a publicly accessible repository under a BSD / MIT / Apache-2.0 license. | ✅ Met (this repo) | Patches in `patches/`; recommended config (optimal worker count, zero-copy) in `docs/RESULTS.md` and `docs/METHODOLOGY.md`; results in `docs/RESULTS.md`; license: `LICENSE` (MIT). *See note on visibility below.* |
| 1.1.3 | Deliver reproducible benchmarks — scripts, workloads, and representative scenarios (including a corpus of **at least 30 legal / medical documents**) — measuring **throughput, latency, and resource efficiency** of LMCache Valkey connectors. | ✅ Met | `benchmarks/` (3 tools + README); `corpus/` (30 docs: 15 legal + 15 medical, `manifest.csv`); throughput/latency/resource-efficiency in `docs/RESULTS.md`; reproduction in `docs/METHODOLOGY.md`; runnable scenarios in `docs/SCENARIOS.md` |

### Contributions beyond the connector

In addition to the connector-side optimizations, this work contributes a new
upstream capability to **valkey-glide** — `mget(keys, buffers=[...])`, batched
zero-copy multi-key GET — see `patches/valkey_glide_mget_buffers.patch` and
`docs/PATCHES.md`. This closes the small-object/metadata gap where the GLIDE
connector previously trailed a hand-rolled C++ connector.

## Note on 1.1.2 visibility

The license requirement (Apache-2.0 / BSD / MIT) is met (`LICENSE`, MIT). This
repository is currently **private**. If a publicly accessible artifact is
required, the deliverables here (benchmarks, results, patches, corpus) can be
published in a separate **public** repository — kept free of any confidential
material — without changing the technical content.

## Out of scope for this repository

The broader engagement includes additional work items that are **not** part of
this repository:

- **1.1.4** — RDMA-with-Valkey investigation (a separate technical report).
- **1.2** — AI observability / Valkey Search demo, benchmarking harness, and report.
- **1.3** — Content, conference, and community work (blogs, presentations,
  conference talks, podcasts).

These are tracked elsewhere.
