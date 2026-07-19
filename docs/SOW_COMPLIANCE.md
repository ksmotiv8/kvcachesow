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
| 1.1.1 | Establish a baseline for the LMCache Valkey connector, then deliver **≥ 2× improvement in large-object (> 1 MB) throughput** vs. that baseline via retrieval optimizations (parallel fetch / prefetch, optimal I/O-thread config), measured under equivalent workloads. | ✅ Met | `benchmarks/bench_baseline.py` (paired, counterbalanced ABBA: **GET ≥2× in every one of 20 reps**, median ~3.0–3.3×, worst rep 2.1× @ 4 MiB; explicit pre-optimization baseline = 1 worker, copy path); `benchmarks/valkey_microbench.py` (per-optimization ablation, 2.3–2.6×; worker-count tuning); `docs/RESULTS.md`; `docs/SCENARIOS.md` §1–2 |
| 1.1.2 | Publish the optimized connector, recommended configurations, and benchmark results to the LMCache repository or a publicly accessible repository under a BSD / MIT / Apache-2.0 license. | ✅ Met | **Public repository published:** [github.com/ksmotiv8/valkey-kvcache-bench](https://github.com/ksmotiv8/valkey-kvcache-bench) (MIT) — the full benchmark suite, results, methodology, corpus + generator, and sample outputs, scrubbed of engagement-specific material. Optimized-connector changes proposed upstream as [LMCache #3955](https://github.com/LMCache/LMCache/pull/3955) and [valkey-glide #6367](https://github.com/valkey-io/valkey-glide/pull/6367). Recommended config (optimal worker count, zero-copy) in `docs/RESULTS.md` / `docs/METHODOLOGY.md`. |
| 1.1.3 | Deliver reproducible benchmarks — scripts, workloads, and representative scenarios (including a corpus of **at least 30 legal / medical documents**) — measuring **throughput, latency, and resource efficiency** of LMCache Valkey connectors. | ✅ Met | `benchmarks/` (7 tools + README): **throughput** — `bench_baseline.py`, `valkey_microbench.py`, `connector_compare.py`, `bench_exists_patch.py`; **latency** — `bench_latency.py` (p50/p99 per op); **resource efficiency** — `bench_resource_efficiency.py` (client CPU/GiB + RSS); **end-to-end scenario** — `bench_corpus_e2e.py` (vLLM+LMCache+Valkey, cold-vs-cached TTFT over the corpus, ~10× across all 30 docs, L2 verified via `keyspace_hits`; sample output in `benchmarks/results/`). `corpus/` (30 docs: 15 legal + 15 medical, `manifest.csv`); results in `docs/RESULTS.md`; reproduction in `docs/METHODOLOGY.md`; runnable scenarios in `docs/SCENARIOS.md` |

### Contributions beyond the connector

In addition to the connector-side optimizations, this work contributes a new
upstream capability to **valkey-glide** — `mget(keys, buffers=[...])`, batched
zero-copy multi-key GET — proposed upstream as
[valkey-glide #6367](https://github.com/valkey-io/valkey-glide/pull/6367)
(snapshot in `patches/valkey_glide_mget_buffers.patch`; see `docs/PATCHES.md`).
This closes the small-object/metadata gap where the GLIDE connector previously
trailed a hand-rolled C++ connector.

## Note on 1.1.2 visibility

Resolved. The public artifact is
[github.com/ksmotiv8/valkey-kvcache-bench](https://github.com/ksmotiv8/valkey-kvcache-bench)
(MIT, published 2026-07-09): the seven benchmark tools, verified results,
methodology, runnable scenarios, the 30-document corpus plus its generator, and
a sample end-to-end run — free of any engagement-specific material. This
private repository remains the internal deliverable of record (adds this
traceability doc and the original patch snapshots).

## Upstream contribution status (as of 2026-07-19)

- **[LMCache #3955](https://github.com/LMCache/LMCache/pull/3955)** — pipelined
  batched EXISTS (8.6–12.6× on the TTFT-gating prefix scan). Open. A
  maintainer-reported test failure was reproduced, fixed (mock pool updated),
  and verified (52/52 locally); DCO and k3-unit-tests green. Awaiting review;
  the second CI job has been stuck in the project's buildkite queue.
- **[valkey-glide #6367](https://github.com/valkey-io/valkey-glide/pull/6367)** —
  zero-copy `buffers` for the sync client's `mget`. Three review rounds fully
  addressed: zero-allocation single-buffer path restored (enum), FFI dedup,
  return-value docs, CHANGELOG, and eight tests (including `itemsize > 1` and a
  cross-slot cluster case verified on a live 3-node cluster). One approval
  (Aryex); commit is signed/**Verified** per repo policy; awaiting final
  approval from the second reviewer.

## Out of scope for this repository

The broader engagement includes additional work items that are **not** part of
this repository:

- **1.1.4** — RDMA-with-Valkey investigation (a separate technical report).
  *Status (2026-07-19): started — two Graviton4 c8gn.16xlarge hosts provisioned
  for the investigation (EFA interfaces still to be attached; current NICs are
  plain ENA). Evaluation of a Valkey DMA module is queued pending repository
  access.*
- **1.2** — AI observability / Valkey Search demo, benchmarking harness, and report.
- **1.3** — Content, conference, and community work (blogs, presentations,
  conference talks, podcasts).

These are tracked elsewhere.
