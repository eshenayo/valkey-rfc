---
RFC: 37
Status: Proposed
---

# Scalable Vector Search (SVS) for valkey-search

## Abstract

This RFC proposes integrating [Scalable Vector Search (SVS)](https://github.com/intel/ScalableVectorSearch) library into the valkey-search module as a new `ALGORITHM SVS_VAMANA` option alongside the existing HNSW and FLAT algorithms. SVS_VAMANA uses the DynamicVamana graph-based index to provide high-performance approximate nearest neighbor (ANN) search optimized for x86_64 platforms. The integration includes multiple compression backends:

* FP16
* SQ8
* LVQ (4/8-bit locally-adaptive vector quantization)
* LeanVec (dimensionality reduction)

All compression algorithms are delivered via Intel's SVS runtime. A future [C API](https://github.com/intel/ScalableVectorSearch/tree/dev/c-api/bindings/c) migration will enable a swappable library model separating open-source (FP32/FP16/SQ8) from proprietary (LVQ/LeanVec) backends. A key feature is deferred compression, which allows indexes to be searchable immediately from the first vector and transparently transitions to a compressed representation once a configurable vector count threshold is reached.

> This is a Work in Progress and subject to change

## Motivation

The valkey-search module currently provides two vector indexing algorithms: FLAT (brute-force exact search) and HNSW (Hierarchical Navigable Small World graph). While HNSW is effective for many workloads, there are scenarios where alternative graph-based algorithms offer better trade-offs:

1. **Memory efficiency at scale.** Large-scale vector datasets (millions to billions of vectors) benefit from advanced compression techniques. Intel SVS provides Locally-adaptive Vector Quantization (LVQ) and LeanVec dimensionality reduction, which can reduce memory footprint by 4-16x while maintaining high recall, outperforming scalar quantization approaches available in HNSW implementations.

2. **x86_64 hardware optimization.** SVS is purpose-built for Intel platforms, leveraging AVX-512 and AVX2 instruction sets for vectorized distance computations and graph traversal. Deployments running on Intel hardware can achieve higher throughput compared to platform-agnostic implementations.

3. **Cold-start problem.** Compressed indexes traditionally require a minimum dataset size for training (e.g., learning quantization codebooks or projection matrices). SVS's deferred compression model starts the index uncompressed and searchable immediately, then transparently transitions to the compressed backend once sufficient vectors are present. This eliminates the need for a separate "training" phase where the index is unavailable for queries.

4. **Algorithm diversity.** DynamicVamana uses a single-level graph with alpha-pruning and greedy search, which produces different recall/throughput/memory trade-offs compared to HNSW's multi-layer probabilistic navigation. Providing both gives operators the flexibility to select the best fit for their workload characteristics and hardware.

## Design Considerations

### Comparison with Existing Algorithms

| Property | FLAT | HNSW | SVS_VAMANA  |
|----------|------|------|---------------------|
| Search complexity | O(n) exact | O(log n) approximate | O(log n) approximate |
| Graph structure | None | Multi-layer skip-list graph | Single-level Vamana graph with alpha-pruning |
| Compression | None | None | FP16, SQ8, LVQ (4/8-bit), LeanVec (dimensionality reduction) |
| Platform | Any | Any | x86_64 Linux (pre-built runtime); ARM64 macOS (source build, future) |
| Dynamic updates | N/A | Supported | Supported (thread-safe add/remove) |
| Memory overhead | Vectors only | Vectors + multi-layer graph | Vectors + single-layer graph |

### Deployment Architecture

`SVS_VAMANA` is consumed via Intel's pre-built runtime library (`libsvs_runtime.so`), which includes all compression backends (FP32, FP16, SQ8, LVQ, and LeanVec). The runtime is linked into valkey-search at build time.

**Current deployment model:** valkey-search ships with `libsvs_runtime.so` (proprietary binary-only, free license) which provides all compression backends in a single library. All `SVS_VAMANA` features — including LVQ and LeanVec compression — are available out of the box on x86_64 Linux.

**Target deployment model (future):** A stable C API (`svs_c.h`, Apache-2.0) is under development upstream. Once available, this enables a swappable library architecture:

- **Source-built `libsvs.so` (default)** — Compiled from open-source SVS (Apache-2.0). Provides FP32, FP16, and SQ8 backends.
- **Intel binary release `libsvs.so` (optional)** — Drop-in replacement (same soname, same C API symbols). Adds LVQ and LeanVec compression backends.

The target model would keep valkey-search's default dependency chain fully open-source (BSD-3-Clause + Apache-2.0) while allowing users to opt-in to proprietary compression by swapping a single library file at deploy time. This transition is pending upstream C API stabilization and does not block current feature development.

### Deferred Compression

Traditional compressed vector indexes require a training phase: a minimum number of vectors must be inserted before quantization codebooks or projection matrices can be learned. During this phase, the index cannot serve queries.

The SVS library implements deferred compression to eliminate this limitation:

1. The index starts with the `NONE` (FP32) or `FP16` storage backend, regardless of the target compression type.
2. Queries are served immediately using the uncompressed representation.
3. When the number of live vectors reaches `COMPRESSION_TRAINING_THRESHOLD`, the SVS runtime transparently trains the compression model and swaps the data backend to the target compressed representation.
4. The graph structure, ID translator, and entry point are preserved — only the data storage layer changes.

This mechanism is exposed in `FT.INFO` as a `state` field: `"training"` while below threshold, `"ready"` after compression has been applied (or when using non-LeanVec compression types that don't require training).

### Platform Requirements

The upstream SVS library builds and passes CI tests on both x86_64 (Linux) and ARM64 (macOS Apple Silicon). However, Intel's pre-built runtime binaries (`libsvs_runtime.so`) are currently **x86_64 Linux only** — no ARM64 release assets are published.

- **x86_64 Linux** (current): valkey-search consumes the pre-built runtime binary. Optimal performance with AVX-512; functional with AVX2 at reduced throughput. All compression backends (FP32, FP16, SQ8, LVQ, LeanVec) are available.
- **ARM64 / macOS** (future): The SVS library compiles from source on ARM64 macOS (Apple Silicon) and passes tests in upstream CI. Once the C API migration enables source builds, ARM64 support becomes viable. Performance characteristics will differ from x86_64 due to NEON vs AVX SIMD instruction sets.

The `ENABLE_SVS` CMake flag (currently defaults to OFF) controls whether SVS is compiled into valkey-search. Phase 1 will default it to ON on x86_64 Linux where the pre-built runtime is available.

### Comparison with Vector Search in Other Systems

| System | Vamana/DiskANN Support | Compression | Platform-Specific Optimizations |
|--------|----------------------|-------------|-------------------------------|
| RediSearch (≥2.8.10) | Yes (SVS_VAMANA) | LVQ + LeanVec | x86_64 (Intel optimized) |
| Milvus | DiskANN (related to Vamana family) | Scalar/Product quantization | Limited |
| Qdrant | No (HNSW only) | Scalar/Product quantization | No |
| Weaviate | No (HNSW only) | Product quantization | No |
| **valkey-search + SVS** | **Yes SVS_VAMANA** | **LVQ + LeanVec** | **x86_64 AVX-512/AVX2** |

RediSearch added SVS_VAMANA support in Redis 8.2.0 with LVQ and LeanVec compression, using the same underlying Intel SVS library. The valkey-search integration aims for feature parity with this capability while operating within the Valkey ecosystem.

## Specification

### Command API: FT.CREATE with ALGORITHM SVS

The `SVS_VAMANA` algorithm is selected via the `ALGORITHM` parameter in the `VECTOR` field specification of `FT.CREATE`. The following SVS_VAMANA-specific parameters are added:

```
FT.CREATE <index> ... SCHEMA <field> VECTOR SVS <num_params>
    TYPE FLOAT32
    DIM <dimensions>
    DISTANCE_METRIC L2|IP|COSINE
    [INITIAL_CAP <capacity>]
    [GRAPH_MAX_DEGREE <degree>]
    [CONSTRUCTION_WINDOW_SIZE <size>]
    [SEARCH_WINDOW_SIZE <size>]
    [ALPHA <value>]
    [COMPRESSION NONE|FP16|SQ8|LVQ4|LVQ8|LVQ4X4|LVQ4X8|LEANVEC4X4|LEANVEC4X8|LEANVEC8X8]
    [LEANVEC_DIMS <dims>]
    [LEANVEC_TRAINING_THRESHOLD <count>]
    [RAW_VECTOR_STORAGE KEEP|DROP]
```

#### Parameter Reference

| Parameter | Type | Default | Constraints | Description |
|-----------|------|---------|-------------|-------------|
| TYPE | enum | — | FLOAT32 | Vector element type (currently only FLOAT32 supported) |
| DIM | int | — | Required | Vector dimensionality |
| DISTANCE_METRIC | enum | — | L2, IP, COSINE | Distance function for similarity computation |
| INITIAL_CAP | int | 10240 | — | Initial capacity hint for memory pre-allocation |
| GRAPH_MAX_DEGREE | int | 64 | ≥2 | Maximum out-degree of each node in the Vamana graph. Higher values improve recall at the cost of memory and construction time. |
| CONSTRUCTION_WINDOW_SIZE | int | 128 | ≥1 | Candidate window size during graph construction. Larger values produce higher-quality graphs at the cost of construction time. |
| SEARCH_WINDOW_SIZE | int | 10 | ≥1 | Beam width during greedy graph search. Higher values improve recall at the cost of query latency. |
| ALPHA | float | 1.2 | >0.0; ≤1.0 for IP/COSINE | Graph pruning parameter controlling edge diversity. Higher values produce denser, higher-recall graphs. |
| COMPRESSION | enum | NONE | See compression table | Storage backend for vector data. |
| LEANVEC_DIMS | int | — | >0 and <DIM | Target dimensionality after LeanVec projection. Required when COMPRESSION is a LEANVEC variant. |
| LEANVEC_TRAINING_THRESHOLD | int | 10000 | ≥1 | Number of vectors to buffer before training the LeanVec projection matrices and transitioning to compressed storage. |
| RAW_VECTOR_STORAGE | enum | KEEP | KEEP, DROP | Whether to retain the original uncompressed vectors alongside the index. DROP saves memory but disables exact distance reconstruction. |

#### Compression Types

| Compression | Category | Description |
|-------------|----------|-------------|
| NONE | Baseline | Full precision FP32 storage (no compression) |
| FP16 | Baseline | IEEE 754 half-precision float storage |
| SQ8 | Scalar quantization | Scalar 8-bit quantization |
| LVQ4 | LVQ | 4-bit Locally-adaptive Vector Quantization |
| LVQ8 | LVQ | 8-bit Locally-adaptive Vector Quantization |
| LVQ4X4 | LVQ | Two-level LVQ: 4-bit primary + 4-bit residual |
| LVQ4X8 | LVQ | Two-level LVQ: 4-bit primary + 8-bit residual |
| LEANVEC4X4 | LeanVec | LeanVec dimensionality reduction + 4x4 LVQ |
| LEANVEC4X8 | LeanVec | LeanVec dimensionality reduction + 4x8 LVQ |
| LEANVEC8X8 | LeanVec | LeanVec dimensionality reduction + 8x8 LVQ |

All compression types are available when SVS is enabled. In the future target architecture (post C API migration), baseline and SQ8 types will be available in the open-source source-built variant, while LVQ and LeanVec types will require the Intel binary release.

#### Example: Creating an SVS index with LeanVec compression

```
FT.CREATE my_index SCHEMA vec VECTOR SVS 18
    TYPE FLOAT32
    DIM 768
    DISTANCE_METRIC COSINE
    GRAPH_MAX_DEGREE 64
    CONSTRUCTION_WINDOW_SIZE 200
    SEARCH_WINDOW_SIZE 20
    ALPHA 0.95
    COMPRESSION LEANVEC4X8
    LEANVEC_DIMS 128
    LEANVEC_TRAINING_THRESHOLD 50000
```

This creates an `SVS_VAMANA` index that:
1. Immediately accepts vectors and serves queries using FP32 storage.
2. After 50,000 vectors are inserted, transparently trains a LeanVec projection from 768 to 128 dimensions with LVQ4x8 compression on the reduced representation.

### FT.INFO Response

For SVS indexes, `FT.INFO` returns the following fields in the vector field's algorithm section:

**Base fields (all SVS indexes):**
- `algorithm`: `SVS_VAMANA`
- `graph_max_degree`: integer
- `construction_window_size`: integer
- `search_window_size`: integer
- `alpha`: float
- `compression`: string (NONE, FP16, SQ8, LVQ4, LVQ8, LVQ4X4, LVQ4X8, LEANVEC4X4, LEANVEC4X8, LEANVEC8X8)
- `state`: `ready` or `training`
- `raw_vector_storage`: `KEEP` or `DROP`

**Additional fields for LeanVec compression types:**
- `leanvec_dims`: integer
- `leanvec_training_threshold`: integer
- `training_progress`: string in format `"<buffered>/<threshold>"` (e.g., `"7500/10000"`)

### FT.SEARCH Behavior

No new `FT.SEARCH` parameters are introduced for `SVS_VAMANA`. The existing KNN query syntax applies:

```
FT.SEARCH my_index "*=>[KNN 10 @vec $query_vec]" PARAMS 2 query_vec <blob>
```

The recall/latency trade-off is controlled by `SEARCH_WINDOW_SIZE` set at index creation time. Larger values improve recall at the cost of higher query latency.

### RDB

**Status: In Progress**

Approach:

1. **Save**: The SVS runtime v0.4.0 provides a `save()` API that serializes the complete DynamicVamana index (graph, vector data, metadata) to a stream. An `RDBOstreamAdapter` wraps RDB chunk I/O as a `std::streambuf` for this purpose, buffering internally at 4MB boundaries.

2. **Load**: An `RDBIstreamAdapter` provides the input stream for `DynamicVamanaIndex::load()`. The index is reconstructed with all graph edges, vector data, and compression state intact.

3. **Staging state persistence**: For LeanVec indexes below their training threshold (in `kStaging` state), the pending buffer and training data are serialized alongside the index metadata.

4. **Deferred compression simplification**: Once deferred compression (Phase 6) lands, the staging state is eliminated — `save()` always works regardless of whether the index has crossed its compression threshold, and `load()` restores the state transparently.

### Configuration

| Configuration | Scope | Default | Description |
|---------------|-------|---------|-------------|
| `ENABLE_SVS` | Build-time (CMake) | OFF (Phase 1 will default to ON on x86_64 Linux) | Whether to compile SVS support into valkey-search |

### Module API

#### SVS Runtime Integration

valkey-search integrates with SVS via Intel's runtime library (`libsvs_runtime.so`), which provides a C++ interface for the DynamicVamana index. The runtime exposes:
- `DynamicVamanaIndex` — graph-based ANN index with dynamic insert/remove
- All storage backends (FP32, FP16, SQ8, LVQ, LeanVec)
- Thread-safe concurrent `add()` operations
- `save()` / `load()` for persistence
- `reconstruct_at()` for exact vector retrieval
- `get_distance()` for pairwise distance computation

#### Future: C API Migration

A stable C API (`svs_c.h`) is under development upstream. When available, migration to the C API will provide:
- Stable ABI (C linkage, opaque handles) replacing fragile C++ vtable interface
- Custom threadpool callback interface for integration with valkey-search's reader thread pools
- Source-buildable library enabling the swappable deployment model described above
- Runtime capability detection for graceful handling of missing compression backends

### Dependencies

| Dependency | Version | License | Purpose |
|------------|---------|---------|---------|
| Intel SVS Runtime (`libsvs_runtime.so`) | 0.4.0 | Proprietary (binary-only, free license) | DynamicVamana graph, all compression backends (FP32/FP16/SQ8/LVQ/LeanVec) |

The runtime is fetched as a pre-built x86_64 Linux binary at build time and linked into `libsearch.so`. AVX-512 is recommended for optimal performance (AVX2 is the minimum).

**Future (post C API migration):** The dependency will transition to a source-buildable `libsvs.so` (Apache-2.0) for FP32/FP16/SQ8, with Intel's binary release as an optional drop-in for LVQ/LeanVec. This will make the default package fully open-source.

### Observability

`SVS_VAMANA` indexes report the following metrics via the valkey-search metrics framework:

- **Index metrics**: vector count, graph degree statistics (mean/max), memory usage (bytes), compression state
- **Search metrics**: query latency histogram (p50/p95/p99), queries per second
- **Memory accounting**: VmRSS-based tracking with per-index byte attribution via `ShardedAtomic` counters

Dispatch-level latency sampling (SAMPLE_EVERY_N at the search.cc layer) is planned for consistency with HNSW/FLAT metrics.

### Testing

- **Functional tests**: FT.CREATE with ALGORITHM SVS_VAMANA → insert vectors → FT.SEARCH verifies recall ≥ 0.95
- **Platform tests**: Verify SVS functions on x86_64 Linux; verify graceful fallback when ENABLE_SVS=OFF on unsupported platforms
- **Compression backend tests**: Verify all compression types (NONE, FP16, SQ8, LVQ, LeanVec) produce functional indexes with expected recall
- **RDB round-trip tests** (planned): BGSAVE → restart → FT.SEARCH verifies index integrity and recall
- **Deferred compression tests**: Verify threshold triggers training/compression transition; search works throughout
- **Parameter validation tests**: Invalid combinations (LEANVEC_DIMS without LeanVec compression, ALPHA > 1.0 with COSINE, etc.) produce appropriate errors
- **Performance tests**: Search latency and recall benchmarks across compression types and dataset sizes

## Implementation Status

### Completed

| Feature | Description |
|---------|-------------|
| Runtime v0.4.0 integration | save/load, reconstruct_at, get_distance, thread-safe add |
| Memory accounting | VmRSS-based tracking, ShardedAtomic counters, ~586 MiB savings from raw_vectors_ removal |
| Metrics suite | Full SVS-specific metrics in metrics framework |
| Basic index operations | Create, add, search, remove functional |

### Remaining (Feature Parity with HNSW)

| Phase | Feature | Priority | Description |
|-------|---------|----------|-------------|
| 2 | RDB persistence | Critical | Save/load SVS indexes across server restarts |
| 1 | ENABLE_SVS=ON default | High | CMake flag defaults to ON on x86_64 Linux (where pre-built runtime is available) |
| 3 | Dispatch latency sampling | Medium | Per-query latency metrics at the dispatch layer (consistency with HNSW/FLAT) |
| 4 | Partial results on timeout | Medium | Return best results found so far when search times out |
| 6 | Deferred compression | Medium | Transparent FP32 → compressed transition via SVS upstream PR #326 (eliminates staging state) |

### Deferred

| Phase | Feature | Blocker | Description |
|-------|---------|---------|-------------|
| 5 | C API migration + licensing split | Pending additional feature implementation | Migrate to stable C API; enables source-built open-source variant and swappable library model |

#### Known Gaps (C API)

| Gap | Impact | Mitigation |
|-----|--------|------------|
| Filtered search | Hybrid queries (vector + tag/numeric predicates) cannot use native SVS filtering | Over-fetch with inflated K + local post-filter in valkey-search; pursuing upstream contribution |

## Appendix

- [Intel Scalable Vector Search — GitHub](https://github.com/intel/ScalableVectorSearch)
- [Intel SVS Documentation](https://intel.github.io/ScalableVectorSearch/)
- [SVS PR #326 — Deferred Compression](https://github.com/intel/ScalableVectorSearch/pull/326)
- [Dev - C API](https://github.com/intel/ScalableVectorSearch/tree/dev/c-api/bindings/c)

### References

- [ABHT23] Aguerrebere, C.; Bhati, I.; Hildebrand, M.; Tepper, M.; Willke, T.: Similarity search in the blink of an eye with compressed indices. In: Proceedings of the VLDB Endowment, 16(11), 3433–3446. (2023)
- [TBAH24] Tepper, M.; Bhati, I.; Aguerrebere, C.; Hildebrand, M.; Willke, T.: LeanVec: Searching vectors faster by making them fit. In: Transactions on Machine Learning Research (TMLR), ISSN 2835-8856. (2024)

```bibtex
@article{aguerrebere2023similarity,
    title     = {Similarity search in the blink of an eye with compressed indices},
    volume    = {16},
    number    = {11},
    pages     = {3433--3446},
    journal   = {Proceedings of the VLDB Endowment},
    author    = {Cecilia Aguerrebere and Ishwar Bhati and Mark Hildebrand and Mariano Tepper and Ted Willke},
    year      = {2023}
}
```
