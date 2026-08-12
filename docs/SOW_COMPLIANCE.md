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
  *Status (2026-08-04): well advanced. The investigation's design is public:
  [momentohq/vdma](https://github.com/momentohq/vdma) (one-sided RDMA over
  libfabric/efa-direct, server-initiated, RESP control channel), backed by a
  working multi-crate transfer substrate (libfabric data path, client, server,
  bench). Two Graviton4 c8gn.16xlarge hosts provisioned. The written technical
  report to Amazon remains OPEN and is due within the engagement term
  (~2026-08-27); the vdma design document plus characterization data are its
  planned inputs. A follow-on engineering engagement covering module
  implementation is in discussion (tracked separately; not part of this SOW).*
- **1.2** — AI observability / Valkey Search demo, benchmarking harness, and report.
  *Status (2026-08-04): 1.2.2 harness DELIVERED as `valkey-lab search`
  ([cachecannon PR #116](https://github.com/cachecannon/cachecannon/pull/116), open, validated on
  glove-25-angular at 1.18M vectors), and since extended beyond the contracted
  single-client scope with closed-loop concurrency (`--query-clients`,
  `--query-loops`) and index-reuse modes. 1.2.3 report DELIVERED, published, and
  substantially expanded on 2026-07-23:
  [Valkey Search HNSW: releases, patches, and vs Redis](https://github.com/ksmotiv8/valkey-kvcache-bench/blob/main/reports/valkey-search-hnsw-bundle-releases.md)
  now covers a unified cross-host campaign of seven engine configurations
  (three released bundles, the current bundle, two module source builds
  including upstream PR 1163, and Redis 8.8), single-client frontiers plus
  saturation throughput ladders, thread-equalization analysis, and comparison
  charts. This exceeds the "up to 3 versions" requirement and feeds a 1.3.1
  blog. **1.2.1 demo BUILT** and being productized into a blog; details with
  publication. Remaining to close the item: release to an open source
  repository under a BSD/MIT/Apache-2.0 license.
- **1.3** — Content, conference, and community work (blogs, presentations,
  conference talks, podcasts).

Content and conference commitments are tracked in the commitment tracker below. Last status refresh: 2026-08-12.

## Engagement commitment tracker

One row per commitment across the whole engagement. Fill in Date / URL /
Description as items land. Status: DONE, IN PROGRESS, TBD.

### 1.1 KV Cache system and benchmarking

| Item | Commitment | Status | Evidence |
|---|---|---|---|
| 1.1.1 | 2x large-object connector improvement vs baseline | DONE | benchmarks + docs/RESULTS.md (see table above) |
| 1.1.2 | Publish connector, configs, results (permissive license) | DONE | valkey-kvcache-bench (MIT), LMCache #3955, valkey-glide #6367 |
| 1.1.3 | Reproducible benchmarks + 30-doc corpus | DONE | benchmarks/ + corpus/ |
| 1.1.4 | RDMA-with-Valkey investigation, technical report to Amazon | IN PROGRESS | design public (momentohq/vdma); written report TBD |

### 1.2 AI observability, search, and benchmarking

| Item | Commitment | Status | Evidence |
|---|---|---|---|
| 1.2.1 | 1 demo: Valkey Search in an inference workflow, contributed OSS | IN PROGRESS | demo built; blog in production; OSS release TBD |
| 1.2.2 | Valkey Search benchmarking harness | DONE | valkey-lab search (cachecannon PR #116) |
| 1.2.3 | Technical report, up to 3 versions, to the community | DONE | valkey-search-hnsw-bundle-releases.md (7 configurations) |

### 1.3.1 Blogs (6 total; cadence: at least one each in May, June, and July)

| # | Title | Type | Date | URL | Description |
|---|---|---|---|---|---|
| 1 | Your KV cache benchmark is "hi hi hi" | Blog | Jun 24, 2026 | https://www.gomomento.com/blog/your-kv-cache-benchmark-is-hi/ | Khawaja Shams explains how building a Valkey-backed KV cache connector revealed that repetitive synthetic benchmarks overstate compression and transfer performance compared with realistic production workloads. |
| 2 | Face Recognition with Valkey | Blog | TBD | TBD | In production. |
| 3 | TBD | Blog | TBD | TBD | TBD |
| 4 | TBD | Blog | TBD | TBD | TBD |
| 5 | TBD | Blog | TBD | TBD | TBD |
| 6 | TBD | Blog | TBD | TBD | TBD |

Note: the cadence calls for at least one blog in each of May, June, and July.
Listed entries currently cover June only; if May or July posts exist, add them
above.

### 1.3.2 Conference presentations (2, on KV Cache and Search performance in Valkey; L7+ review before use)

| # | Title | Venue / event | Date | URL / deck | Review status |
|---|---|---|---|---|---|
| 1 | TBD | TBD | TBD | TBD | TBD |
| 2 | TBD | TBD | TBD | TBD | TBD |

### 1.3.3 Conference appearances

| Commitment | Status | Date | Notes |
|---|---|---|---|
| Present Valkey KV Cache and/or Vector Search at QCon AI Boston (June 2026) | TBD | TBD | deck exists (qcon-ai-kvcache) |
| Curate AI presentation block for an upcoming Unlocked (3 to 5 AI sessions with Valkey components) | TBD | TBD | TBD |

### 1.3.4 Podcasts (4, with industry experts, on Valkey YouTube)

| # | Title | Type | Date | URL | Description |
|---|---|---|---|---|---|
| 1 | Episode #22: Why One Slow Chunk Can Stall an Entire LLM | Podcast | Jun 22, 2026 | https://youtu.be/riIG31ki0dI | Samuel Shen (TensorMesh) and Daniela Miao (Momento) explore how LMCache manages distributed KV cache and optimizes Valkey to improve throughput, large-object performance, and tail latency for LLM inference. |
| 2 | Episode #28: Search Doesn't Belong In A Separate System | Podcast | Jul 13, 2026 | https://youtu.be/WDQGgYxAet8 | Allen Samuels (AWS) and Mike Callahan (Momento) explore when Valkey Search belongs in the operational data layer and how it delivers fast, scalable search for high-throughput workloads. |
| 3 | Episode #29: The Lock Free Architecture Behind Valkey Search | Podcast | Jul 16, 2026 | https://youtu.be/LQqQA7NwDRo | Yair Gottdenker (Google) and Mike Callahan (Momento) explore how Valkey Search uses lock-free concurrency and intelligent query planning to scale low-latency hybrid and vector search across modern CPUs. |
| 4 | TBD | Podcast | TBD | TBD | TBD |


