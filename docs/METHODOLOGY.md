# Methodology

How the benchmarks were run, what each one measures, and how to reproduce them.

## Environment

| Component | Detail |
|---|---|
| Client | AWS G6 instance (NVIDIA L4), us-west-2 |
| Server | Valkey **9.1.0**, started with `--io-threads 10`, CPU-pinned to cores 4–14, single node |
| Network | Client → server over the VPC internal address |
| Client library | `valkey-glide-sync` 2.4.1 (provides the `glide_sync` module) |
| LMCache | `dev` branch; the GLIDE-based `ValkeyConnector` |
| Chunk sizes | 4 MB (LMCache "golden spot" for high-throughput transfer, ≈ 16 tokens for Llama-3.1-8B), with 1 MB and 64 KB sweeps |

Server launched as a container pinned to cores 4–14:

```bash
docker run -d --name valkey --cpuset-cpus=4-14 -p 6379:6379 valkey/valkey:9.1 \
  valkey-server --protected-mode no --save '' --appendonly no --io-threads 10 --port 6379
```

## How each connector is driven

The benchmarks drive each connector **the way LMCache drives it in production**,
so the comparison reflects real behavior rather than a synthetic best case:

- **GLIDE connector**: N worker threads, each with its own `glide_sync` client,
  one in-flight operation per thread. Mirrors `ValkeyConnector._batched_get`,
  which submits one operation per key across a `ThreadPoolExecutor` and gathers.
- **RESP connector** (for the comparison): a single `batch_*_sync` call; the C++
  layer fans out across its own worker threads.

## Timing

- All timings use `time.perf_counter`, measured at the batch level: submit all
  keys, then wait for all to complete.
- Each measured loop is preceded by an **untimed warmup pass** so the reported
  medians reflect steady state, not one-time client/connection setup.
- Reported figures are medians over N loops; where a result is sensitive to
  run-to-run variance (notably 1 MB), multiple back-to-back repetitions were run
  and the spread is reported in `docs/RESULTS.md`.

## Baseline definition (SOW 1.1.1)

"Baseline" is the connector with its retrieval optimizations **disabled** —
1 worker thread (no parallel fetch) on the copy path (no zero-copy buffer GET).
"Optimized" enables them — N workers with zero-copy buffer GET. This isolates
the *retrieval optimizations* the SOW names (parallel fetch, optimal I/O thread
configuration) on identical code and hardware.

A pre-GLIDE connector version is also available in LMCache history (before the
GLIDE-optimization commit) for a connector-version before/after if preferred;
the opts-toggled baseline above is the controlled, reproducible default.

## The three benchmarks

### `valkey_microbench.py`
Drives the connector's internal thread-pool I/O engine directly and reports a
three-config matrix — **baseline (1 worker, copy)**, **+parallel (N workers,
copy)**, **+zero-copy (N workers, buffer GET)** — so each optimization's
contribution is attributable. Supports payload and worker-count sweeps.

### `connector_compare.py`
Runs the GLIDE connector and the RESP connector against the **same** server with
the **same** keys/payloads, each through its native batch path, and reports a
side-by-side table with an explicit "integrated path" caveat (the two batch the
same way each is used in LMCache, not at the raw wire-protocol level).

### `bench_exists_patch.py`
Measures the consecutive-prefix `EXISTS` scan (used by `batched_contains`, the L2
lookup that gates TTFT) before and after Patch 1, through the real thread pool,
with a correctness check that both paths return identical results.

## Document corpus (SOW 1.1.3)

`corpus/` contains **30 documents** — 15 legal and 15 medical — under
`corpus/legal/` and `corpus/medical/`, indexed by `corpus/manifest.csv`
(domain, filename, title, word/char counts). This satisfies the requirement for
a representative corpus of at least 30 legal/medical documents for end-to-end
KV-cache scenarios.

## Reproduction

```bash
# 1.1.1 — ≥2x and the per-optimization breakdown
python benchmarks/valkey_microbench.py --host <host> --port 6379 \
    --num-workers 32 --num-keys 128 --chunk-mb 4.0 --loops 10 --compare

# Worker-count sweep (find the single-node optimum)
for w in 1 4 8 16 32 64; do
  python benchmarks/valkey_microbench.py --host <host> --port 6379 \
    --num-workers $w --num-keys 128 --chunk-mb 4.0 --loops 8
done

# GLIDE vs RESP connector
python benchmarks/connector_compare.py --host <host> --port 6379 \
    --num-workers 8 --num-keys 128 --chunk-mb 4.0 --loops 10

# Patch 1 before/after
python benchmarks/bench_exists_patch.py --host <host> --port 6379 \
    --num-workers 8 --num-keys 512 --loops 20
```
