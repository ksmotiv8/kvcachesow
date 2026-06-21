# Benchmark tools

Three self-contained tools for measuring the LMCache Valkey connector. All
report median throughput (GiB/s) or ops/s over N loops, with an untimed warmup
pass first. See `../docs/METHODOLOGY.md` for what each measures.

## Requirements

- `valkey-glide-sync` ≥ 2.3 (provides `glide_sync`)
- An importable `lmcache` package (the microbench and EXISTS bench drive the
  connector's internal `_ThreadWorkerPool`; `connector_compare.py` also needs an
  `lmcache` built with the C++ Redis extension for the RESP arm)
- A running Valkey 9.x reachable from the client

## `valkey_microbench.py`

Three-config matrix (baseline / +parallel / +zero-copy) and worker/payload
sweeps.

```bash
python valkey_microbench.py --host 10.0.0.1 --port 6379 \
    --num-workers 32 --num-keys 128 --chunk-mb 4.0 --loops 10 --compare
```

| Flag | Default | Meaning |
|---|---|---|
| `--num-workers` | 32 | Worker threads (connections) |
| `--num-keys` | 128 | Keys per batch |
| `--chunk-mb` | 4.0 | Payload size per key, MB |
| `--loops` | 10 | Measured loops (median reported) |
| `--compare` | off | Run baseline / +parallel / +zero-copy and report speedup |
| `--no-verify` | off | Skip first-pass data verification |

Without `--compare` it runs a single optimized config — useful for worker sweeps.

## `connector_compare.py`

GLIDE connector vs RESP connector, same server, each via its native batch path.

```bash
python connector_compare.py --host 10.0.0.1 --port 6379 \
    --num-workers 8 --num-keys 128 --chunk-mb 4.0 --loops 10
```

`--backends glide,resp` (default) selects which to run. Output includes an
explicit note that the comparison is the integrated path (GLIDE per-key
thread-pool fan-out vs RESP single native batch), not a raw wire comparison.

## `bench_exists_patch.py`

Before/after for the connector `EXISTS`-pipelining patch (Patch 1), driving the
real thread pool. Applies the patch's pool methods by monkeypatch so it runs
without a rebuilt connector, and verifies both paths return identical results.

```bash
python bench_exists_patch.py --host 10.0.0.1 --port 6379 \
    --num-workers 8 --num-keys 512 --loops 20
```
