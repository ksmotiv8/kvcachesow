# KV-Cache over Valkey — LMCache Connector Optimization

Reproducible benchmarks, verified results, and two contributed patches for
optimizing the **LMCache Valkey connector** (built on
[valkey-glide](https://github.com/valkey-io/valkey-glide)) for large-object
KV-cache transfer in LLM serving.

The work establishes a baseline, delivers a **≥2× large-object throughput**
improvement, contributes optimizations back upstream, and ships a reproducible
benchmark suite with a legal/medical document corpus.

---

## Highlights

- **≥2× large-object throughput (verified).** Parallel-fetch with optimal worker
  tuning yields **2.3–2.6× GET** at 4 MB chunks, measured across repeated runs.
- **Two patches, both reviewed and verified to apply cleanly upstream:**
  - **LMCache connector — pipelined `EXISTS`:** **8.6–12.6×** faster metadata
    lookups (the L2 prefix-scan that gates time-to-first-token).
  - **valkey-glide — batched zero-copy `mget`:** a new `mget(keys, buffers=[...])`
    API giving **2.51×** over the connector's current path, beating a hand-rolled
    C++ connector — batching *and* zero-copy in one round-trip.
- **Reproducible suite:** three benchmark tools, a 30-document legal/medical
  corpus, and full methodology.

See [`docs/RESULTS.md`](docs/RESULTS.md) for the complete tables and caveats.

---

## Results at a glance

Client: AWS G6 (NVIDIA L4). Server: Valkey 9.1.0, 10 I/O threads, single node.

| Metric | Result |
|---|---|
| GET, 4 MB — baseline → optimized | ~1.03 → ~2.4 GiB/s (**2.3–2.6×**, 3 reps) |
| Optimal worker count (single node) | **~8** (GET peaks ~2.76 GiB/s, near NIC ceiling) |
| `EXISTS` prefix-scan — before → after (Patch 1) | 17k → **220k ops/s** (12.6× @ 1024 keys) |
| Small-object batched GET — before → after (Patch 2) | 0.84 → **2.11 GiB/s** (2.51×, beats RESP 1.77) |

---

## Repository layout

```
kvcachesow/
├── README.md                  This file
├── LICENSE                    MIT
├── docs/
│   ├── RESULTS.md             Verified before/after results + caveats
│   ├── METHODOLOGY.md         Environment, how each benchmark works, reproduction
│   ├── SCENARIOS.md           Specific runnable scenarios to try (goal/command/expected)
│   └── PATCHES.md             Deep-dive on both patches (design, before/after, apply)
├── patches/
│   ├── valkey_connector_batched_exists.patch    Patch 1 — LMCache connector
│   └── valkey_glide_mget_buffers.patch          Patch 2 — valkey-glide (mget buffers)
├── benchmarks/
│   ├── README.md              How to run the tools
│   ├── valkey_microbench.py   Baseline vs +parallel vs +zero-copy; worker sweeps
│   ├── connector_compare.py   GLIDE connector vs RESP connector (integrated path)
│   └── bench_exists_patch.py  Before/after for the EXISTS pipelining patch
└── corpus/
    ├── manifest.csv           Index of the 30 legal/medical documents
    ├── legal/                 15 legal documents
    └── medical/               15 medical documents
```

---

## Quick start

Requires `valkey-glide-sync` (≥ 2.3) and a running Valkey 9.x. For the RESP
comparison, an `lmcache` build with the C++ Redis extension is also needed.

```bash
# ≥2x large-object throughput, with the optimization breakdown
python benchmarks/valkey_microbench.py --host <valkey-host> --port 6379 \
    --num-workers 32 --num-keys 128 --chunk-mb 4.0 --loops 10 --compare

# GLIDE connector vs RESP connector on the same server
python benchmarks/connector_compare.py --host <valkey-host> --port 6379 \
    --num-workers 8 --num-keys 128 --chunk-mb 4.0 --loops 10

# Connector EXISTS patch, before vs after
python benchmarks/bench_exists_patch.py --host <valkey-host> --port 6379 \
    --num-workers 8 --num-keys 512 --loops 20
```

See [`benchmarks/README.md`](benchmarks/README.md) for all options,
[`docs/METHODOLOGY.md`](docs/METHODOLOGY.md) for the full setup, and
**[`docs/SCENARIOS.md`](docs/SCENARIOS.md) for ten specific scenarios to try** —
each with its goal, exact command, and expected result (≥2× verification,
worker-count tuning, payload crossover, GLIDE-vs-RESP, both patches' before/after,
NIC/line-rate check, resource efficiency, and the end-to-end corpus run).

---

## The patches

Both are described in detail in [`docs/PATCHES.md`](docs/PATCHES.md), and both
have been verified to apply cleanly to their pristine upstream trees.

```bash
# Patch 1 — LMCache connector (pipelined EXISTS), from an LMCache checkout:
git apply patches/valkey_connector_batched_exists.patch

# Patch 2 — valkey-glide (batched zero-copy mget), from a valkey-glide checkout,
# then rebuild the glide-sync FFI:
git apply patches/valkey_glide_mget_buffers.patch
```

Patch 2 is additive and opt-in: `mget(keys)` is unchanged; the buffer path is
only taken when `buffers=[...]` is passed.

---

## License

MIT — see [`LICENSE`](LICENSE).
