# Engagement status and deliverables traceability

The status page of record for the engagement's work items. It intentionally
reproduces no commercial terms, figures, party-specific contractual language,
or the underlying agreement: only the engineering scope, current status, and
where each deliverable lives.

Artifacts for 1.1.1 through 1.1.3 are hosted in this repository. All other
items link out to their external repositories and channels.

The scope text these items trace to is in [SOW_SCOPE.md](SOW_SCOPE.md)
(commercial figures omitted).

Last status refresh: 2026-08-12 (1.3.3 Unlocked sessions added).

## Status at a glance

| Item | Commitment | Status | Evidence / next step |
|---|---|---|---|
| 1.1.1 | 2x large-object connector improvement vs baseline | Done | benchmarks + `docs/RESULTS.md` (detail below) |
| 1.1.2 | Publish connector, configs, results under a permissive license | Done | [valkey-kvcache-bench](https://github.com/ksmotiv8/valkey-kvcache-bench) (MIT); upstream PRs below |
| 1.1.3 | Reproducible benchmarks + 30-doc corpus | Done | `benchmarks/` + `corpus/` (detail below) |
| 1.1.4 | RDMA-with-Valkey investigation, technical report | In progress | design public ([momentohq/vdma](https://github.com/momentohq/vdma)); written report open |
| 1.2.1 | Demo: Valkey Search in an inference workflow, contributed OSS | In progress | demo built; blog in production; OSS release open |
| 1.2.2 | Valkey Search benchmarking harness | Done | `valkey-lab search` ([cachecannon PR #116](https://github.com/cachecannon/cachecannon/pull/116)) |
| 1.2.3 | Technical report, up to 3 versions, to the community | Done | [published report](https://github.com/ksmotiv8/valkey-kvcache-bench/blob/main/reports/valkey-search-hnsw-bundle-releases.md) (7 configurations) |
| 1.3.1 | 6 blogs (at least one each in May, June, July) | In progress | 1 published, 1 in production, 4 open (tracker below) |
| 1.3.2 | 2 conference presentations (L7+ review before use) | Open | tracker below |
| 1.3.3 | QCon AI Boston talk + Unlocked AI block curation | In progress | Unlocked block curated (4 sessions); QCon status TBD (tracker below) |
| 1.3.4 | 4 podcasts with industry experts on Valkey YouTube | In progress | 3 published, 1 open (tracker below) |

## 1.1 KV Cache system and benchmarking

The work item covers optimizing the LMCache Valkey connector for KV-cache
workloads. Unlike typical Valkey usage (objects under 1 MB), KV-cache transfers
are frequently large objects (multiple MB), and projects like LMCache benefit
from client-side optimizations to make Valkey competitive with alternative
storage backends.

| # | Requirement (summary) | Status | Where it's satisfied |
|---|---|---|---|
| 1.1.1 | Establish a baseline for the LMCache Valkey connector, then deliver at least a 2x improvement in large-object (> 1 MB) throughput vs. that baseline via retrieval optimizations (parallel fetch / prefetch, optimal I/O-thread config), measured under equivalent workloads. | Done | `benchmarks/bench_baseline.py` (paired, counterbalanced ABBA: GET at least 2x in every one of 20 reps, median ~3.0 to 3.3x, worst rep 2.1x at 4 MiB; explicit pre-optimization baseline = 1 worker, copy path); `benchmarks/valkey_microbench.py` (per-optimization ablation, 2.3 to 2.6x; worker-count tuning); `docs/RESULTS.md`; `docs/SCENARIOS.md` sections 1 and 2 |
| 1.1.2 | Publish the optimized connector, recommended configurations, and benchmark results to the LMCache repository or a publicly accessible repository under a BSD / MIT / Apache-2.0 license. | Done | Public repository published: [github.com/ksmotiv8/valkey-kvcache-bench](https://github.com/ksmotiv8/valkey-kvcache-bench) (MIT, 2026-07-09): the full benchmark suite, results, methodology, corpus + generator, and sample outputs, scrubbed of engagement-specific material. Optimized-connector changes proposed upstream as [LMCache #3955](https://github.com/LMCache/LMCache/pull/3955) and [valkey-glide #6367](https://github.com/valkey-io/valkey-glide/pull/6367). Recommended config (optimal worker count, zero-copy) in `docs/RESULTS.md` / `docs/METHODOLOGY.md`. This private repository remains the internal deliverable of record (this traceability doc plus the original patch snapshots). |
| 1.1.3 | Deliver reproducible benchmarks (scripts, workloads, and representative scenarios, including a corpus of at least 30 legal / medical documents) measuring throughput, latency, and resource efficiency of LMCache Valkey connectors. | Done | `benchmarks/` (7 tools + README): throughput via `bench_baseline.py`, `valkey_microbench.py`, `connector_compare.py`, `bench_exists_patch.py`; latency via `bench_latency.py` (p50/p99 per op); resource efficiency via `bench_resource_efficiency.py` (client CPU/GiB + RSS); end-to-end scenario via `bench_corpus_e2e.py` (vLLM+LMCache+Valkey, cold-vs-cached TTFT over the corpus, ~10x across all 30 docs, L2 verified via `keyspace_hits`; sample output in `benchmarks/results/`). `corpus/` (30 docs: 15 legal + 15 medical, `manifest.csv`); results in `docs/RESULTS.md`; reproduction in `docs/METHODOLOGY.md`; runnable scenarios in `docs/SCENARIOS.md` |

### Contributions beyond the connector

In addition to the connector-side optimizations, this work contributes a new
upstream capability to valkey-glide: `mget(keys, buffers=[...])`, batched
zero-copy multi-key GET, proposed upstream as
[valkey-glide #6367](https://github.com/valkey-io/valkey-glide/pull/6367)
(snapshot in `patches/valkey_glide_mget_buffers.patch`; see `docs/PATCHES.md`).
This closes the small-object/metadata gap where the GLIDE connector previously
trailed a hand-rolled C++ connector.

### Upstream contribution status (as of 2026-07-19)

- [LMCache #3955](https://github.com/LMCache/LMCache/pull/3955): pipelined
  batched EXISTS (8.6 to 12.6x on the TTFT-gating prefix scan). Open. A
  maintainer-reported test failure was reproduced, fixed (mock pool updated),
  and verified (52/52 locally); DCO and k3-unit-tests green. Awaiting review;
  the second CI job has been stuck in the project's buildkite queue.
- [valkey-glide #6367](https://github.com/valkey-io/valkey-glide/pull/6367):
  zero-copy `buffers` for the sync client's `mget`. Three review rounds fully
  addressed: zero-allocation single-buffer path restored (enum), FFI dedup,
  return-value docs, CHANGELOG, and eight tests (including `itemsize > 1` and a
  cross-slot cluster case verified on a live 3-node cluster). One approval
  (Aryex); commit is signed/Verified per repo policy; awaiting final approval
  from the second reviewer.

### 1.1.4 RDMA-with-Valkey investigation

Status (2026-08-04): well advanced. The investigation's design is public:
[momentohq/vdma](https://github.com/momentohq/vdma) (one-sided RDMA over
libfabric/efa-direct, server-initiated, RESP control channel), backed by a
working multi-crate transfer substrate (libfabric data path, client, server,
bench). Two Graviton4 c8gn.16xlarge hosts provisioned. The written technical
report to Amazon remains open and is due within the engagement term
(~2026-08-27); the vdma design document plus characterization data are its
planned inputs. A follow-on engineering engagement covering module
implementation is in discussion (tracked separately; not part of this SOW).

## 1.2 AI observability, search, and benchmarking

- **1.2.1 demo** (2026-08-12): built and being productized into a blog;
  details with publication. Remaining to close the item: release to an open
  source repository under a BSD/MIT/Apache-2.0 license.
- **1.2.2 harness**: delivered as `valkey-lab search`
  ([cachecannon PR #116](https://github.com/cachecannon/cachecannon/pull/116),
  open, validated on glove-25-angular at 1.18M vectors), and since extended
  beyond the contracted single-client scope with closed-loop concurrency
  (`--query-clients`, `--query-loops`) and index-reuse modes.
- **1.2.3 report**: delivered, published, and substantially expanded on
  2026-07-23:
  [Valkey Search HNSW: releases, patches, and vs Redis](https://github.com/ksmotiv8/valkey-kvcache-bench/blob/main/reports/valkey-search-hnsw-bundle-releases.md)
  covers a unified cross-host campaign of seven engine configurations (three
  released bundles, the current bundle, two module source builds including
  upstream PR 1163, and Redis 8.8), single-client frontiers plus saturation
  throughput ladders, thread-equalization analysis, and comparison charts.
  This exceeds the "up to 3 versions" requirement and feeds a 1.3.1 blog.

## 1.3 Content, conference, and community

Fill in Date / URL / Description as items land.

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
| Curate AI presentation block for an upcoming Unlocked (3 to 5 AI sessions with Valkey components) | Done | TBD | 4 sessions curated (below); not all presented by Momento, per the scope's intent of producing and curating content across the community |

Unlocked AI block sessions (fill in dates and links):

| # | Session | Speakers | Slides | YouTube |
|---|---|---|---|---|
| 1 | Valkey and Semantic Caching | Dmitry Polyakovsky | not shared by speaker | TBD |
| 2 | Towards Faster Inference: With KV Cache and Beyond | Daniela Miao and Samuel Shen | TBD | TBD |
| 3 | Efficiency at Scale: Our Journey from Redis to Valkey | Vu Pham and Xintian Li | TBD | TBD |
| 4 | Scaling Search with Multithreading and Hybrid Queries | Allen Samuels and Yair Gottdenker | TBD | TBD |

### 1.3.4 Podcasts (4, with industry experts, on Valkey YouTube)

| # | Title | Type | Date | URL | Description |
|---|---|---|---|---|---|
| 1 | Episode #22: Why One Slow Chunk Can Stall an Entire LLM | Podcast | Jun 22, 2026 | https://youtu.be/riIG31ki0dI | Samuel Shen (TensorMesh) and Daniela Miao (Momento) explore how LMCache manages distributed KV cache and optimizes Valkey to improve throughput, large-object performance, and tail latency for LLM inference. |
| 2 | Episode #28: Search Doesn't Belong In A Separate System | Podcast | Jul 13, 2026 | https://youtu.be/WDQGgYxAet8 | Allen Samuels (AWS) and Mike Callahan (Momento) explore when Valkey Search belongs in the operational data layer and how it delivers fast, scalable search for high-throughput workloads. |
| 3 | Episode #29: The Lock Free Architecture Behind Valkey Search | Podcast | Jul 16, 2026 | https://youtu.be/LQqQA7NwDRo | Yair Gottdenker (Google) and Mike Callahan (Momento) explore how Valkey Search uses lock-free concurrency and intelligent query planning to scale low-latency hybrid and vector search across modern CPUs. |
| 4 | TBD | Podcast | TBD | TBD | TBD |
