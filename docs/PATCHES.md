# Patches

Two patches were produced. Both were reviewed (by the author and by an
independent automated review pass) and verified with `git apply --check` to
apply cleanly to their pristine upstream trees.

---

## Patch 1 — LMCache connector: pipelined `EXISTS`

**File:** `patches/valkey_connector_batched_exists.patch`
**Target:** LMCache, `lmcache/v1/storage_backend/connector/valkey_connector.py`
**Upstream change required:** none — uses GLIDE's existing `Batch` API.

### Problem

The connector's consecutive-prefix `EXISTS` scan (`batched_contains` /
`_batched_async_contains`, used for the L2 lookup that gates time-to-first-token)
fanned out **one `EXISTS` per key** across the thread pool. Wall-clock time was
roughly `ceil(N / num_workers)` round-trips, and at small key counts the
per-operation Python overhead dominated.

### Fix

Replace the per-key fan-out with a single pipelined GLIDE `Batch` + `exec`:
build one `EXISTS` command per key into a non-atomic batch, execute it in one
round-trip, then count the consecutive prefix from the ordered result list. The
consecutive-prefix semantics (stop at the first missing key) are preserved
exactly. Edge cases handled: empty input, `exec` returning no results, and the
Redis pipeline ordering guarantee is relied upon (and documented).

### Before / after

Consecutive-prefix `EXISTS` scan, 8 workers, all keys present:

| keys | before (per-key fan-out) | after (Batch + exec) | speedup |
|---:|---:|---:|---:|
| 128 | 17.2k ops/s | 148.1k ops/s | **8.6×** |
| 512 | 17.4k ops/s | 214.4k ops/s | **12.3×** |
| 1024 | 17.5k ops/s | 220.5k ops/s | **12.6×** |

The speedup grows with key count because one pipelined round-trip replaces
`N / num_workers` round-trips. Reproduce with `benchmarks/bench_exists_patch.py`.

### Apply

```bash
# from an LMCache checkout (dev)
git apply patches/valkey_connector_batched_exists.patch
```

---

## Patch 2 — valkey-glide: batched zero-copy `mget`

**File:** `patches/valkey_glide_mget_buffers.patch`
**Target:** valkey-glide — `ffi/src/lib.rs`,
`python/glide-shared/glide_shared/_glide_ffi.py`,
`python/glide-sync/glide_sync/glide_client.py`,
`python/glide-sync/glide_sync/sync_commands/core.py`.
**Additive and opt-in:** `mget(keys)` behavior is unchanged.

### Problem

valkey-glide already had **single-key** zero-copy via `get(key, buffer=)` — the
value is written directly into a caller-owned buffer with no intermediate
`bytes`. But there was no **multi-key** equivalent: `mget(keys)` always
allocated a fresh `bytes` per value. So a workload fetching many values had to
choose between **batching** (`mget`, one round-trip, but a copy per value) and
**zero-copy** (many single `get(buffer=)` calls, but per-key overhead) — never
both. That is exactly what KV-cache retrieval needs: many chunks, landed
directly in pinned host memory / tensors.

### Fix

Add **`mget(keys, buffers=[...])`** — batched **and** zero-copy multi-key GET.
Each value is written into its corresponding caller buffer in a single pipelined
round-trip.

- **Rust FFI (`ffi/src/lib.rs`):** the arena response builder previously copied a
  single bulk string into a single buffer and ignored buffers for `Array`
  responses. It now threads a **slice of buffers**: a scalar uses `bufs[0]`, an
  `Array` hands `bufs[i]` to element `i`. A new FFI entry `command_with_buffers`
  takes parallel arrays of buffer pointers/lengths. Per-element size and
  null-pointer checks guard every `copy_nonoverlapping`; a `Nil` element (missing
  key) leaves its buffer untouched and is reported as a null response.
- **Python (`glide-shared` / `glide-sync`):** the cffi declaration for
  `command_with_buffers`, a `response_buffers` path in `_execute_command` (builds
  the C pointer arrays and keeps them alive across the call), and the public
  `mget(keys, buffers=...)` with a length check.

The command response carries the number of bytes written per element (e.g.
`b'4096'`) when a buffer is used — mirroring the existing single-key
`get(buffer=)` convention.

### Before / after

64 KB × 1024 keys, driven as the connector would (8 workers, each `mget`-ing its
shard into buffers):

| Path | GiB/s |
|---|---:|
| single-client `mget` (bytes) | 0.647 |
| single-client `mget` (zero-copy buffers) | 0.817 (**+26%**) |
| 8 workers, per-key buffer-GET (current connector) | 0.842 |
| **8 workers, `mget`-into-buffers (this patch)** | **2.110 (2.51×, beats RESP 1.77)** |

The 2.51× throughput win is driven by **batching**; **zero-copy** adds the
direct-placement property (values land in caller memory with no intermediate
`bytes`). At line-rate 4 MB the two connectors are throughput-bound, so
zero-copy is neutral there; its benefit is allocation-avoidance, which scales
with the number of values fetched (see `docs/RESULTS.md`).

### Correctness

Verified end-to-end: values land byte-exact in the provided buffers, missing
keys return `None`, and plain `mget` and `get(buffer=)` are unchanged. The
unsafe FFI was reviewed for buffer lifetime, per-element bounds/null checks,
`Nil` handling, and aliasing; two null-pointer issues found in review were
fixed before publishing.

### Apply

```bash
# from a valkey-glide checkout, then rebuild the glide-sync FFI
git apply patches/valkey_glide_mget_buffers.patch
```

### Usage

```python
import glide_sync

bufs = [memoryview(bytearray(CHUNK_BYTES)) for _ in keys]
result = client.mget(keys, buffers=bufs)
# result[i] is the byte count written for keys[i], or None if the key is missing;
# bufs[i][:int(result[i])] holds the value with no intermediate allocation.
```
