# Next-Generation File Formats: A Survey of the Landscape, Pain Points, and Design Blueprint for the Future of Data Storage

**Date:** 2026-04-20
**Type:** Deep Research Report

---

## Executive Summary

Apache Parquet, introduced in 2013 [3], has become the de facto standard for analytical data storage [1]. But the data landscape has changed dramatically: IDC estimates the global datasphere now exceeds 180 zettabytes annually [59, 60], and a widely cited (though methodologically uncertain) industry estimate holds that roughly 80% of enterprise data is unstructured [60], AI/ML workloads demand random access and vector search over thousands of feature columns [27], and cloud object stores have replaced local HDFS as the dominant storage substrate. Parquet was not designed for any of these conditions.

This report surveys the current file format landscape (Parquet, ORC, Avro, Arrow/Feather, Lance, Nimble, Vortex, and FastLanes), documents the structural pain points that affect cost and performance at scale, evaluates emerging formats and academic research (2022–2026), and synthesizes a design blueprint for a next-generation file format.

**Key findings:**
1. Parquet's biggest problems are structural — immutable files creating the small-file problem [28, 30], chatty I/O on cloud object stores [33, 34, 42], hard-coded encodings [27], and metadata overhead for wide tables [32].
2. Three credible next-gen formats have emerged: **Lance** (AI/ML-optimized) [14, 43], **Vortex** (general-purpose, BtrBlocks-inspired) [18, 20], and **FastLanes** (SIMD/GPU-native from CWI/DuckDB) [55, 56]. **Nimble** (Meta) [16] and **F3** (CMU, SIGMOD 2026) [52] represent additional research directions.
3. The winning design principles converge across all emerging work: pluggable encodings, cloud-native I/O, Arrow-native interop, log-structured appends, and GPU-friendly aligned layouts [14, 18, 52, 55].
4. No single format has yet achieved the combination of Parquet's ecosystem breadth with next-gen performance. The transition will likely be mediated by table formats (Iceberg, Delta Lake) providing pluggable file format APIs [15].

---

## 1. The Current File Format Landscape

### 1.1 Format Timeline

| Year | Format | Creator | Key Innovation |
|------|--------|---------|----------------|
| 2009 | Apache Avro | Doug Cutting [9] | Row-based serialization with schema evolution [8] |
| 2013 | Apache Parquet | Twitter + Cloudera [3] | Dremel-based columnar storage for Hadoop [4] |
| 2013 | Apache ORC | Hortonworks + Facebook [7] | Type-aware columnar with built-in ACID support [5, 6] |
| 2016 | Apache Arrow | Wes McKinney et al. [11, 26] | Language-agnostic in-memory columnar standard [10] |
| 2020 | Feather V2 | Arrow community [12] | Arrow IPC with on-disk compression [12] |
| 2022 | Lance | Chang She, Lei Xu [15] | Columnar for AI/ML with O(1) random access [14] |
| 2023 | Nimble | Meta [16] | Wide-table optimized, extensible encodings [16] |
| 2024 | Vortex | SpiralDB (LF AI&Data from Aug 2025) [18, 19] | BtrBlocks-style cascading compression [20] |
| 2025 | FastLanes | CWI Amsterdam [55] | SIMD/GPU-native file format, 40× faster decode (claimed) [55, 64] |

### 1.2 Format Comparison Matrix

| Dimension | Parquet | ORC | Avro | Arrow/Feather | Lance v2 | Nimble | Vortex |
|-----------|---------|-----|------|---------------|----------|--------|--------|
| **Layout** | Columnar [2] | Columnar [6] | Row [8] | Columnar (memory) [10] | Columnar [14] | Columnar [16] | Columnar [18] |
| **Encoding model** | Hard-coded (dict, RLE, delta, bitpack) [1, 27] | Hard-coded (type-aware) [6] | Block compression [8] | Uncompressed (memory); LZ4/ZSTD (IPC) [12] | Pluggable extensions [14] | Pluggable, cascading [16] | Pluggable, cascading (BtrBlocks) [20] |
| **Row groups** | Yes (fixed) [2] | Yes (stripes) [6] | N/A | N/A | No — per-column pages [14] | Block encoding [16] | Extensible physical layout [18] |
| **Random access** | Poor (page scan) [27] | Poor [27] | Poor | Good (memory) [10] | O(1) via structural encoding [43, 44] | Unknown | 100× faster than Parquet (claimed) [18] |
| **Cloud I/O** | Chatty (multi-GET) [33, 42] | Similar to Parquet | N/A | N/A | Single range read [69] | Unknown | Optimized [18] |
| **Schema evolution** | Limited (add columns only) [37] | Limited | Excellent (forward/backward) [8] | Rich type system [10] | Full (via table format) [13] | Extensible [16] | Extensible [18] |
| **GPU support** | No native [27] | No | No | Aligned but uncompressed [10] | No native | SIMD/GPU targets [16] | GPU pushdown [18] |
| **Ecosystem** | Universal [1] | Hive-centric [5] | Kafka/streaming [8] | Universal (in-memory) [10, 11] | LanceDB, growing [15] | Velox only [16] | Arrow, DuckDB, Spark (growing) [18] |
| **Maturity** | 13 years, production [3] | 13 years, production [5] | 17 years, production [9] | 10 years, production [11] | 4 years, stabilizing [15] | Pre-stable [16] | Format stable (v0.36+), young [18] |

---

## 2. Structural Pain Points with Current Formats

### 2.1 The Small-File Problem

Because Parquet files are **immutable** — you cannot append rows to an existing file [28] — every micro-batch write creates a new file. A streaming pipeline checkpointing every 30 seconds to a table with 100 active partitions generates approximately **12,000 files per hour** — or roughly **288,000 files per day** [30].

**Impact chain:**
- Every query must open each file and parse its metadata footer before reading any data [28]
- S3 LIST operations become a bottleneck (limited to 5,500 GET/HEAD requests/second per partition prefix) [42]
- Task planning overhead grows linearly with file count [28]
- Metadata overhead per file is disproportionate to data content [29]

All three major table formats (Delta Lake, Iceberg, Hudi) were partly built to mitigate this, but all require **scheduled compaction** as ongoing operational overhead [30, 38].

### 2.2 Metadata and Footer Overhead

Parquet stores its metadata in a Thrift-encoded footer at the end of each file [2, 32]. This footer must be read before any data access — it's the critical path for every query [32].

**Documented issues:**
- Footer parsing scales **linearly** with columns × row groups [32]. For ML tables with thousands of columns, metadata parsing alone can dominate query latency.
- The Thrift compact protocol lacks random access — all preceding fields must be scanned to locate a specific field [32]. A custom parser in the Rust `parquet` crate achieved **3–9× speedup** over the standard generated parser (arrow-rs v57.0.0, October 2025) [32].
- On cloud object stores, reading the footer requires at minimum **2 network round-trips**: one to read the footer size from the end of file, one to read the footer itself [36, 42].

### 2.3 I/O Amplification on Cloud Object Stores

Parquet was designed for local HDFS reads where seeks are cheap. On S3/GCS/Azure Blob, every seek becomes a separate HTTP GET request with ~50–100ms latency [42, 69].

**Documented evidence:**
- DuckDB's Parquet prefetching creates **separate HTTP GETs per column chunk per row group**. For filtered queries where most row groups are pruned, disabling prefetching is consistently faster (DuckDB issue #21474) [33].
- Arrow's S3FileSystem generated **520,752 requests** for a checkpointing operation where only 74,331 were for actual data — a 7× overhead (Arrow issue #40589) [34].
- S3 throttles at **5,500 GET/HEAD requests/second per partition prefix** (PARQUET-2486 cites approximately this limit) [42], creating a hard ceiling on query parallelism.

### 2.4 Hard-Coded Encodings

Parquet's encoding repertoire (dictionary, RLE, delta, bitpacking) is baked into the format specification [1, 2]. Adding a new encoding requires a format-level change and coordination across all implementations [14].

**Consequences:**
- Parquet's RLE threshold of 8 is hard-coded and non-configurable — suboptimal when common run lengths are 7 [27].
- Dictionary size limit (1MB default) causes fallback to "plain" (no encoding) when the dictionary fills [27].
- No support for modern lightweight encodings: FSST for strings (random-access-capable, LZ4-speed) [48], ALP for floats (orders of magnitude faster than Gorilla/Chimp) [54], or FastLanes for integers (100+ GB/s decode) [55].
- Block compression (ZSTD, Snappy) is often **detrimental** to end-to-end query speed on modern hardware because modern SSDs already deliver GB/s throughput, making the decompression overhead a net negative (Zeng et al., PVLDB 2023) [27].

### 2.5 Predicate Pushdown Limitations

- Parquet's smallest zone-map unit is a page, coarser than ORC's configurable row-level granularity (default 10,000 rows) [27].
- Zone maps (min/max) are useless for randomly distributed data — ranges overlap across all row groups [27, 36].
- Page-level statistics (PageIndex) were only fixed in Parquet 2.9.0; DataFusion's row-level filter pushdown is **not enabled by default** due to performance regressions [36].

### 2.6 Nested Type Complexity

Parquet's Dremel model (repetition/definition levels) is powerful but produces persistent interoperability issues [4, 22, 27]:
- PARQUET-1254: Deeply nested records fail via Avro interface — **unresolved open bug** [39].
- HIVE-8909: Parquet's Avro and Thrift object models produce different representations than Hive [40].
- Adding a column to a deeply nested schema in ClickHouse consumed excessive memory, taking "days proportional to the amount of data" (ClickHouse issue #79760) [41].

### 2.7 Schema Evolution Rigidity

- Column renaming is impossible without full file rewrite [37].
- Column insertion between existing columns requires byte-offset recalculation and rewrite [37].
- Type widening (INT32 → INT64) requires data rewrite since encoding is type-dependent [37].
- Table formats (Iceberg's ID-based tracking [50], Delta Lake's Column Mapping [51]) provide workarounds but don't fix the underlying format.

*Note: The Parquet community is actively discussing "Parquet 3.0" / "Parquet Modern" proposals that could address some encoding extensibility and metadata format issues. These efforts are ongoing but have not yet resulted in a ratified spec change as of April 2026.*

---

## 3. Emerging Formats and Academic Research

### 3.1 Lance (LanceDB, 2022–present)

Lance is a columnar container format designed for AI/ML workloads [14]. Its key innovation is **adaptive structural encoding** — encoding column structure (nesting, repetition) independently from data, enabling O(1) random access without sacrificing scan performance [43].

**Design highlights:**
- **No row groups**: Each column manages its own page layout independently, eliminating the row-group tuning problem [14].
- **Pluggable encodings**: The format spec is ~50 lines of protobuf; all encodings are extensions [14].
- **O(1) random access**: Structural encodings enable direct row lookup via single HTTP range requests. Lance's own benchmarks report ~100× faster random access than Parquet [18]; some secondary analyses cite up to 2000× for specific configurations [15, 69]. Independent verification is not yet available.
- **ML-native**: Native blob storage (images, videos), vector indexing (IVF_PQ, HNSW), and support for very wide tables [14, 45].

**Benchmarks** (self-reported, LanceDB v2.2 on EC2 c7i.4xlarge) [35]:
- 52% smaller than Parquet on text data (FineWeb 10M rows)
- 75× faster random blob fetch
- ~40,000× faster schema evolution (add column)
- Parquet remains 40% faster on narrow column projections (Parquet's core strength)

**Academic backing:** Pace et al., "Lance: Efficient Random Access in Columnar Storage through Adaptive Structural Encodings" (arXiv:2504.15247, April 2025) [43].

**Ecosystem:** LanceDB native, Python/PyArrow integration. LanceDB raised $30M Series A (June 2025) [44, 45]. Not yet supported by traditional SQL engines [15].

### 3.2 Vortex (SpiralDB → LF AI&Data, 2024–present)

Vortex aims to be a general-purpose successor to Parquet, achieving Pareto-optimal performance across all dimensions [18, 19].

**Design highlights:**
- **BtrBlocks-style cascading compression**: Automatically selects optimal codec per data block from a menu of lightweight schemes. Incorporates FastLanes (integers), FSST (strings), and ALP (floats) [20].
- **Compute on compressed data**: Uniquely pushes predicate evaluation directly into encoded representations, avoiding decompression entirely when possible ("lazy decompression") [46].
- **Zero-copy Arrow-native**: Designed for zero-copy exchange with Arrow-based engines [18].
- **Format stable** from v0.36.0 with backwards compatibility guarantees [18].

**Benchmarks** (all self-reported by the Vortex team; no independent benchmarks exist as of April 2026) [18, 20]:
- TPC-H SF10: **38% smaller** than Parquet+ZSTD with **10–25× faster decompression** [20]
- 100× faster random access reads vs Parquet (claimed) [18]
- 10–20× faster scans, 5× faster writes (claimed) [18]

**Ecosystem:** Integrations with Arrow, DataFusion, DuckDB, Spark, Pandas, Polars [18]. Iceberg integration planned. Linux Foundation (LF AI&Data) incubation project [19]. 2,828 GitHub stars [21].

### 3.3 FastLanes (CWI Amsterdam, PVLDB 2025)

FastLanes is the most academically rigorous next-gen format proposal, from the CWI database group (Boncz, Afroozeh — also behind FSST and ALP) [55].

**Key innovation:** A "unified transposed layout" that enables SIMD-like parallel decoding using only scalar code, achieving portability across CPU architectures and GPUs without any explicit SIMD instructions [55, 56].

**Performance claims:** 40% better compression and **40× faster decoding** than Parquet [55, 64]. Decoding throughput exceeds 100 billion integers per second [56].

**GPU extension:** G-ALP (DaMoN 2025) extends FastLanes with GPU-optimized floating-point compression [57].

**Adoption:** Partially adopted in DuckDB, Parquet, and Lance Format [56, 64].

### 3.4 Nimble (Meta, 2023–present)

Meta's next-gen columnar format, optimized for wide tables (thousands of columns) and fast decode speed [16].

- **FlatBuffers metadata** (vs Thrift in Parquet) for faster metadata access [16].
- **Cascading encodings**: Encodings can be layered recursively [16].
- **Block encoding** (not stream encoding) for predictable memory usage [16].
- Pre-stable; 707 GitHub stars; tightly coupled with Velox execution engine; no public benchmarks [16, 17].

### 3.5 F3 — The Future-proof File Format (SIGMOD 2026)

From CMU/Tsinghua (Zeng, Pavlo et al.), F3 is a concrete, implemented next-generation file format (with Rust code and FlatBuffer definitions) built around principles of interoperability, extensibility, and efficiency [52]. The companion vision paper, "Towards Functional Decomposition of Storage Formats" (CIDR 2025, Prammer et al.) [53], articulates the broader idea of decomposing monolithic format specs into reusable functional components — an approach F3 puts into practice.

*Note: F3 was published in Proc. ACM Manag. Data Vol. 3, No. 4 (Sep 2025) and will be presented at SIGMOD 2026 (Bengaluru). We use the conference year "SIGMOD 2026" per the authors' own labeling.*

### 3.6 Key Compression Advances

| Technique | Paper | Year | Key Result | Adopted By |
|-----------|-------|------|------------|------------|
| **BtrBlocks** | Kuschewski et al., SIGMOD 2023 [47] | 2023 | 2.2× faster scans than Parquet with competitive compression ratios [24, 47] | Vortex [20] |
| **FSST** | Boncz et al., PVLDB 2020 [48] | 2020 | Random-access string compression at LZ4 speed [48] | DuckDB, Vortex [20] |
| **ALP** | Afroozeh et al., SIGMOD 2024 (Best Artifact) [54] | 2024 | Orders of magnitude faster lossless float compression [54] | DuckDB, Vortex [20] |
| **FastLanes encoding** | Afroozeh & Boncz, PVLDB 2025 [55] | 2025 | 100+ GB/s integer decoding, SIMD-free parallelism [55, 56] | DuckDB, Lance [56] |
| **LeCo** | Liu et al., arXiv 2023 [58] | 2023 | Learned compression; generalizes FOR/Delta/RLE; 5.2× speedup in Arrow [58] | Research |
| **OpenZL** | Meta Engineering, Oct 2025 [63] | 2025 | Format-aware compression: 2× ratio vs zstd at faster speed; universal decoder [63] | Meta internal |

### 3.7 Table Formats as Workaround Layers

Delta Lake [49], Apache Iceberg [50], and Apache Hudi are **table formats** that add ACID transactions, schema evolution, time travel, and compaction management on top of Parquet. They patch Parquet's structural gaps but add their own operational complexity [22, 23, 38].

**Key convergence mechanism:** Delta Lake UniForm automatically generates Iceberg and Hudi metadata [51, 66], and Apache Iceberg's proposed File Format API would let new formats (Lance, Vortex) plug in without modifying core Iceberg logic [15].

---

## 4. Data Growth Pressures

### 4.1 Scale

IDC estimates the global datasphere at ~181 zettabytes for 2025, growing at ~23% CAGR [59, 60]. A widely cited industry estimate (originating from IBM/Gartner circa 2013, with uncertain methodology) holds that approximately 80% of enterprise data is unstructured, growing ~3× faster than structured data [59, 60]. Cloud storage now holds roughly half of all stored data.

### 4.2 Cost Sensitivity

At Q1 2026 cloud prices, storage costs range from $0.018–$0.023/GB/month for standard tiers [61]. At petabyte scale, even a 10% compression improvement saves tens of thousands of dollars monthly. Egress costs ($0.087–$0.12/GB) make selective reads (predicate pushdown) critical for cost management [62].

### 4.3 New Access Patterns

AI/ML workloads introduce access patterns that current formats handle poorly [27, 15]:
- **Random access** for RAG retrieval and secondary index lookups
- **Wide projections** across thousands of feature columns [27]
- **Vector search** over high-dimensional embeddings
- **Multimodal** data (images, video references, embeddings in the same table) [14]
- **Streaming append** for real-time feature stores and CDC

---

## 5. Design Blueprint: A Next-Generation File Format

Based on the convergent evidence from emerging formats, academic research, and industry engineering, we propose the following design principles for a next-generation file format. This is a synthesis, not an original invention — it captures the consensus direction visible across Lance [14], Vortex [18], FastLanes [55], F3 [52], and industry engineering from Meta [63], Snowflake, DuckDB [68], and ClickHouse [67].

### 5.1 Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                    FILE LAYOUT                       │
├─────────────────────────────────────────────────────┤
│  Superblock (fixed offset, first 4KB)               │
│  ├─ Magic bytes + format version                    │
│  ├─ Schema summary (field IDs, types)               │
│  ├─ Global statistics pointer                       │
│  └─ Fragment directory pointer                      │
├─────────────────────────────────────────────────────┤
│  Fragment 0                                         │
│  ├─ Column 0: [encoding header] [pages...]          │
│  ├─ Column 1: [encoding header] [pages...]          │
│  ├─ ...                                             │
│  ├─ Per-column zone maps + bloom filters            │
│  └─ Fragment footer (column offsets, row count)     │
├─────────────────────────────────────────────────────┤
│  Fragment 1                                         │
│  ├─ ...                                             │
├─────────────────────────────────────────────────────┤
│  Optional Index Sections                            │
│  ├─ Secondary indexes                               │
│  ├─ Vector indexes (IVF_PQ, HNSW)                   │
│  └─ Sort-order metadata                             │
├─────────────────────────────────────────────────────┤
│  Global Statistics Block                            │
│  ├─ Per-column global min/max/null_count/NDV        │
│  ├─ Per-fragment row ranges                         │
│  └─ Encoding registry (codec IDs → descriptions)   │
├─────────────────────────────────────────────────────┤
│  File Footer                                        │
│  ├─ Full schema (FlatBuffers, not Thrift)           │
│  ├─ Fragment directory                              │
│  ├─ Encryption envelope                             │
│  └─ Checksum                                        │
└─────────────────────────────────────────────────────┘
```

### 5.2 Core Design Principles

#### Principle 1: Pluggable Encoding Registry

**What:** Encodings are extensions, not part of the format spec. The format defines an encoding ID space and a minimal bootstrap decoder; all specific codecs (dictionary, RLE, FSST, ALP, FastLanes bitpacking) are registered as plugins.

**Why:** Parquet's hard-coded encoding set prevents adoption of modern compression techniques (FSST [48], ALP [54], BtrBlocks-style cascading [47]). Every new encoding requires a format-level spec change and multi-year adoption across implementations [14].

**How:** Each column chunk header contains an encoding ID and a small encoding descriptor (parameters needed to decode). A universal decoder reads the ID and dispatches to the appropriate codec. New codecs can be added by registering an ID without changing the format spec. This follows the approach demonstrated by Lance [14], Vortex [18], Nimble [16], and Meta's OpenZL [63].

#### Principle 2: No Fixed Row Groups — Per-Column Page Management

**What:** Eliminate the row-group concept. Each column independently manages its own page layout and can choose different page sizes based on data characteristics.

**Why:** Row groups force a one-size-fits-all granularity that creates tuning problems [14]. Too large → poor predicate pushdown. Too small → metadata bloat. String columns benefit from different page sizes than integer columns.

**How:** The fragment is the organizational unit (analogous to a row group but flexible). Within a fragment, each column defines its own page boundaries and encoding independently. Column pages are not required to be contiguous in the file. This follows Lance v2's design [14].

#### Principle 3: Cloud-Native I/O with Fixed-Offset Superblock

**What:** Place a compact superblock at a known, fixed offset (first 4KB of file) containing enough metadata to plan all subsequent I/O in a single request.

**Why:** Parquet's footer-at-end design forces at least 2 round-trips before any data access [36, 42]. On S3 with 50–100ms per-GET latency, this is a significant overhead multiplied across millions of files [69].

**How:** The superblock contains the schema summary, global statistics pointer, and fragment directory pointer. A query engine can read the first 4KB, determine which fragments and columns to access, and issue coalesced range reads for all needed data in a single batch. The full footer at the end serves as a redundant copy for integrity checking.

#### Principle 4: Log-Structured Append with Fragment-Based Organization

**What:** New writes create new fragments appended to the file (or as new files in a fragment set). A lightweight manifest tracks the current state. Background compaction merges fragments.

**Why:** Parquet's immutable-file model creates the small-file problem (~12,000 files/hour or ~288,000/day in typical streaming scenarios) [30]. Compaction becomes a mandatory operational burden [30, 38].

**How:** Inspired by ClickHouse's MergeTree [67] and Lance's MVCC [14, 45]: new data is appended as new fragments with a new manifest version. Readers use the latest manifest. Background compaction merges fragments without blocking reads. This eliminates the need for external table-format-level compaction for simple use cases.

#### Principle 5: Adaptive Per-Chunk Codec Selection

**What:** For each data chunk, automatically select the optimal lightweight encoding based on data characteristics (cardinality, sortedness, run lengths, distribution).

**Why:** BtrBlocks (SIGMOD 2023) demonstrated that per-block codec selection achieves 2.2× faster scans than Parquet with competitive compression ratios [47]. Generic block compressors (ZSTD, Snappy) are often net-negative on modern hardware [27].

**How:** At write time, sample each chunk and test candidate codecs (dictionary, RLE, bitpacking, FOR, FSST for strings, ALP for floats). Select the codec that minimizes a cost function balancing compressed size and decode speed. Store the codec choice in the chunk header. Support cascading (e.g., dictionary → bitpacking → optional ZSTD for cold data). This follows the approach used by Vortex [20] and DuckDB's native format [68].

#### Principle 6: GPU-Friendly Aligned Layout

**What:** Use fixed-width, aligned data pages (4KB or 8KB mini-blocks) with data-parallel encodings that can be decoded by GPU threads without synchronization.

**Why:** GPU analytics is growing rapidly, but current formats require full decompression to CPU memory before GPU transfer [27]. FastLanes (PVLDB 2025) showed that data-parallel layouts enable 40× faster decoding and portability across SIMD widths and GPU architectures [55].

**How:** Follow FastLanes' "unified transposed layout" for integer and float columns [55]. Each mini-block is independently decodable. Pages are aligned to DMA boundaries. Support a "GPU-ready" encoding mode where data can be DMA'd directly to GPU memory. G-ALP extends this to floating-point data [57].

#### Principle 7: Arrow-Native In-Memory Representation

**What:** The format's in-memory representation is defined as Arrow-compatible, enabling zero-copy exchange with the Arrow ecosystem [10].

**Why:** Arrow is the de facto in-memory columnar standard with bindings in 10+ languages [10, 11, 26]. Formats that require deserialization to a different in-memory layout pay an unnecessary conversion cost.

**How:** Column pages decode directly to Arrow arrays. Variable-length data uses Arrow's offset-based layout. The schema type system is a superset of Arrow's type system. This follows the approach used by Vortex [18] and Lance [14].

#### Principle 8: Field-ID-Based Schema Evolution

**What:** Identify columns by stable integer field IDs, not names. Support add, drop, rename, reorder, and type-widening operations without file rewrite.

**Why:** Parquet identifies columns by name in the footer, making rename impossible and schema merges fragile [37]. Iceberg solved this with ID-based tracking but at the table format layer [50].

**How:** Each column in the schema has a unique, immutable field ID assigned at creation. The field ID is stored in the column chunk header. Column metadata maps field IDs to names, types, and descriptions. Renaming is a metadata-only operation. Dropping is a metadata-only operation (data is reclaimed on compaction). Type widening (INT32→INT64) is supported if the encoding is compatible.

#### Principle 9: Built-In Column-Level Encryption

**What:** Native support for per-column AES-GCM encryption with envelope encryption and key rotation.

**Why:** Parquet added modular encryption in v1.12 [65], but it's complex and adoption is limited. A next-gen format should make encryption a first-class, easy-to-use feature.

**How:** Each column can be encrypted with a different data encryption key (DEK), wrapped by a key encryption key (KEK) [65]. The superblock and global statistics for non-sensitive columns remain readable. Encrypted columns can still participate in zone-map-based pruning if the zone maps themselves are encrypted with a shared key. Key rotation updates the KEK without rewriting data.

#### Principle 10: Pluggable Index Sections

**What:** Support optional, lazily-computed index sections: zone maps (always on), bloom filters (per-column opt-in), secondary indexes, and vector indexes (IVF_PQ, HNSW).

**Why:** No single indexing strategy covers all workloads. Zone maps are universally useful; bloom filters help equality predicates; vector indexes enable ANN search; secondary indexes support point lookups [27, 14].

**How:** Indexes are stored as optional file sections with their own encoding. They can be computed at write time (eager) or post-hoc (lazy). The fragment directory references available indexes. Vector indexes follow Lance's approach of embedding IVF_PQ or HNSW structures directly in the file [14, 15].

### 5.3 What This Design Addresses

| Problem | Current Format | Proposed Solution |
|---------|---------------|-------------------|
| Small-file problem | Parquet's immutable files [28] | Log-structured append with fragment compaction |
| Cloud I/O amplification | Footer-at-end, multi-GET [33, 42] | Fixed-offset superblock, coalesced range reads |
| Metadata overhead for wide tables | Thrift footer, linear scan [32] | FlatBuffers footer, hierarchical statistics |
| Hard-coded encodings | Parquet spec lock-in [27] | Pluggable encoding registry |
| Poor random access | Page-level granularity only [27] | Per-column page management, structural encodings |
| Compression ceiling | Generic block compressors [27] | Adaptive per-chunk lightweight encoding + cascading [47] |
| GPU unfriendly | Variable-length, unaligned [27] | FastLanes-style aligned mini-blocks [55] |
| Schema evolution rigidity | Name-based columns [37] | Field-ID-based identification |
| No native vector search | External vector DB required | Pluggable vector index sections [14] |
| Encryption complexity | Optional, poorly adopted [65] | First-class column-level encryption |

### 5.4 Migration Path

A critical lesson from format history is that no format succeeds without an adoption path. The proposed format should:

1. **Provide a Parquet transcoder**: Read Parquet files and write them in the new format with a single command. This is how DuckDB and Polars drove Parquet adoption.
2. **Register with Iceberg's File Format API**: The proposed Iceberg File Format API (ReadBuilder/WriteBuilder + Format Registry) enables new formats to be used as Iceberg table backends without modifying core Iceberg logic [15].
3. **Support Arrow as the interchange format**: Any engine that speaks Arrow can read the new format with minimal integration effort [10].
4. **Offer a "compatibility mode"**: Write files that contain a Parquet-compatible subset for engines that can't read the new format yet, alongside extended sections that new-format-aware engines exploit.

---

## 6. Comparison with Existing Next-Gen Approaches

| Design Dimension | This Blueprint | Lance v2 | Vortex | FastLanes | Nimble | F3 |
|-----------------|---------------|----------|--------|-----------|--------|-----|
| Pluggable encodings | ✅ | ✅ [14] | ✅ [20] | ✅ (expression-based) [55] | ✅ [16] | ✅ [52] |
| No row groups | ✅ (fragments) | ✅ [14] | Partial | N/A | Block encoding [16] | Functional decomp [52] |
| Cloud-native superblock | ✅ | ❌ (footer-at-end) | Unknown | Unknown | Unknown | Unknown |
| Log-structured append | ✅ | ✅ (MVCC) [14] | ❌ | ❌ | ❌ | Unknown |
| GPU-friendly layout | ✅ (FastLanes-style) | ❌ | ✅ (GPU pushdown) [18] | ✅ (core design) [55] | ✅ (target) [16] | Unknown |
| Vector indexes | ✅ | ✅ (IVF_PQ, HNSW) [14] | ❌ | ❌ | ❌ | Unknown |
| Field-ID schema evolution | ✅ | ✅ (via table format) [13] | Unknown | Unknown | Unknown | Unknown |
| Column-level encryption | ✅ | ❌ | ❌ | ❌ | ❌ | Unknown |
| Arrow-native | ✅ | ✅ [14] | ✅ [18] | Partial | ❌ (Velox-native) [16] | Unknown |

**Observation:** No existing format combines all these properties. Lance is closest for AI/ML workloads but lacks GPU-native layouts and encryption. Vortex is closest for general analytics but lacks append support and vector indexes. FastLanes provides the best decode performance but is primarily an encoding/layout innovation, not a complete file format with metadata management.

---

## 7. Open Questions and Caveats

1. **Benchmark independence:** Most performance claims for emerging formats (Lance [35], Vortex [18, 20]) are self-reported by their vendors. Independent, reproducible benchmarks (e.g., from the Zeng et al. PVLDB group [27] or a TPC-style standardized test) are needed before making strong comparative claims.

2. **Ecosystem chicken-and-egg:** Parquet's dominance is largely due to universal engine support [1]. A new format must either achieve broad engine adoption (multi-year effort) or integrate via the Iceberg File Format API / Arrow bridge [15].

3. **Write amplification vs. read optimization:** Log-structured append trades write simplicity for read complexity (fragment merging, manifest management). The optimal compaction strategy depends on workload (streaming vs. batch, read-heavy vs. write-heavy).

4. **Complexity budget:** Combining all 10 design principles into a single format risks creating something too complex to implement correctly. F3's functional decomposition approach — building reusable components rather than a monolithic spec — may be the wiser architectural choice [52, 53].

5. **Nimble's trajectory:** Meta's Nimble is pre-stable with no public benchmarks [16, 17]. Given Meta's engineering resources and exabyte-scale internal deployment, Nimble could become a significant force if it reaches maturity and broad ecosystem support.

6. **Learned compression:** LeCo and similar learned-encoding approaches show promise but are still research-stage [58]. The cost of model training at write time and the portability of trained models across deployments are unresolved.

7. **Regulatory pressure:** GDPR's right to erasure and similar regulations require efficient deletion. Log-structured formats handle this better than Parquet's immutable files, but tombstone management and compaction policies need careful design.

---

## 8. Recommendations

### For data engineers choosing a format today:
- **Default to Parquet + Iceberg/Delta Lake** for general analytical workloads. The ecosystem support is unmatched [1, 50, 51, 66].
- **Evaluate Lance** for AI/ML workloads involving vector search, random access, wide feature tables, or multimodal data [14, 35].
- **Watch Vortex** for general-purpose analytics where compression ratio and decode speed are critical [18, 20].
- **Use ZSTD** (level 1-3) as the default Parquet compression codec instead of Snappy [31].

### For format designers building something new:
- **Start with the encoding layer:** BtrBlocks-style adaptive codec selection [47] and FastLanes-style data-parallel layout [55] are the two most impactful innovations available today.
- **Design for cloud first:** Fixed-offset metadata, coalesced range reads, and minimal round-trips [42, 69].
- **Plug into Iceberg:** The File Format API is the most realistic path to ecosystem adoption without building a new table format [15].
- **Don't try to replace Parquet everywhere at once.** Target a specific workload niche (AI/ML, real-time analytics, GPU analytics) and win decisively there first.

---

## Sources

1. Apache Parquet Official Docs — Overview — https://parquet.apache.org/docs/overview/
2. Apache Parquet — File Format Spec — https://parquet.apache.org/docs/file-format/
3. Twitter Engineering Blog — "Announcing Parquet 1.0" (Jul 2013) — https://blog.x.com/engineering/en_us/a/2013/announcing-parquet-10-columnar-storage-for-hadoop
4. Apache Parquet — Wikipedia — https://wikipedia.org/wiki/Apache_Parquet
5. ORC Background — orc.apache.org — https://orc.apache.org/docs/index.html
6. ORC Specification v1 — https://orc.apache.org/specification/ORCv1/
7. Apache ORC — Wikipedia — https://wikipedia.org/wiki/Apache_ORC
8. Apache Avro Specification — https://avro.apache.org/docs/++version++/specification/_print/
9. Cloudera Blog — "Avro: a New Format for Data Interchange" (Nov 2009) — http://clouderatemp.wpengine.com/blog/2009/11/avro-a-new-format-for-data-interchange/
10. Arrow Columnar Format — v23.0.1 — https://arrow.apache.org/docs/format/Columnar.html
11. Apache Arrow 10th Anniversary Blog (Feb 2026) — https://arrow.apache.org/blog/2026/02/12/arrow-anniversary/
12. Ursa Labs — "Feather V2 with Compression Support" (2020) — https://ursalabs.org/blog/2020-feather-v2/
13. Lance Format Specification — https://lance.org/format/
14. Lance v2 Blog Post — LanceDB — https://blog.lancedb.com/lance-v2/
15. Dremio Blog — "Evolving File Format Landscape in AI Era" — https://www.dremio.com/blog/exploring-the-evolving-file-format-landscape-in-ai-era-parquet-lance-nimble-and-vortex-and-what-it-means-for-apache-iceberg/
16. Nimble GitHub README — https://github.com/facebookincubator/nimble/blob/main/README.md
17. Nimble GitHub Repo — https://github.com/facebookincubator/nimble
18. Vortex GitHub README — https://github.com/vortex-data/vortex
19. Vortex Official Site — https://vortex.dev/
20. SpiralDB Blog — "Have your cake and decompress it too" — https://spiraldb.com/post/cascading-compression-with-btrblocks
21. Vortex GitHub Repo (original SpiralDB) — https://github.com/spiraldb/vortex/
22. Liu et al. — "A Deep Dive into Common Open Formats for Analytical DBMSs" (PVLDB 2023) — https://vldb.org/pvldb/vol16/p3044-liu.pdf
23. Springer — "Data formats in analytical DBMSs" (VLDB Journal 2025) — https://link.springer.com/article/10.1007/s00778-025-00911-1
24. BtrBlocks GitHub — https://github.com/maxi-k/btrblocks
25. *(removed)*
26. Wes McKinney — "Apache Arrow and the 10 Things I Hate About pandas" — https://wesmckinney.com/blog/apache-arrow-pandas-internals/
27. Zeng et al. — "An Empirical Evaluation of Columnar Storage Formats" (PVLDB 2023) — https://www.vldb.org/pvldb/vol17/p148-zeng.pdf
28. Körükcü — "Small Files, Big Problems" (Medium, Feb 2026) — https://medium.com/@alper-korukcu/small-files-big-problems-the-silent-killer-of-data-lake-performance-a448db657c94
29. Apache Hudi Blog — "Automated Small File Handling" (Nov 2024) — https://hudi.apache.org/blog/2024/11/19/automated-small-file-handling/
30. IOMETE — "Apache Iceberg Production Anti-Patterns 2026" — https://iomete.com/resources/blog/apache-iceberg-production-antipatterns-2026
31. Mamonas — "Which Compression Saves the Most Storage $?" (2025) — https://mamonas.dev/posts/which-compression-algorithm-saves-most/
32. Apache Arrow Blog — "3x-9x Faster Parquet Footer Metadata" (Oct 2025) — https://arrow.apache.org/blog/2025/10/23/rust-parquet-metadata/
33. DuckDB Issue #21474 — "Parquet prefetching slows filtered queries over S3" — https://github.com/duckdb/duckdb/issues/21474
34. Apache Arrow Issue #40589 — "S3FileSystem generates redundant requests" — https://github.com/apache/arrow/issues/40589
35. LanceDB — "Lance Format v2.2 Benchmarks" (2026) — https://www.lancedb.com/blog/lance-format-v2-2-benchmarks-half-the-storage-none-of-the-slowdown
36. DataFusion Blog — "Parquet Pruning: Read Only What Matters" (Mar 2025) — https://datafusion.apache.org/blog/2025/03/20/parquet-pruning/
37. Uplatz — "Schema Evolution Patterns in Production Data Lakes" (2026) — https://uplatz.com/blog/comprehensive-analysis-of-schema-evolution-patterns-in-production-data-lakes-and-backward-compatibility-strategies/
38. Abstract Algorithms — "Modern Table Formats: Delta Lake vs Iceberg vs Hudi" (Mar 2026) — https://abstractalgorithms.dev/modern-table-formats-delta-lake-vs-apache-iceberg-vs-apache-hudi
39. PARQUET-1254 Jira — "Unable to read deeply nested records with Avro interface" — https://issues.apache.org/jira/browse/PARQUET-1254
40. HIVE-8909 Jira — "Hive doesn't correctly read Parquet nested types" — https://issues.apache.org/jira/browse/HIVE-8909
41. ClickHouse Issue #79760 — "Adding column to deeply nested schema — excessive memory" — https://github.com/ClickHouse/ClickHouse/issues/79760
42. PARQUET-2486 Jira — "Improve Parquet IO Performance within cloud datalakes" — https://issues.apache.org/jira/browse/PARQUET-2486
43. Pace et al. — "Lance: Efficient Random Access in Columnar Storage" (arXiv:2504.15247, Apr 2025) — https://arxiv.org/pdf/2504.15247
44. LanceDB — "Lance File 2.1 is Now Stable" — https://lancedb.com/blog/lance-file-2-1-stable/
45. LanceDB — "Lance File Format 2.2: Taming Complex Data" — https://www.lancedb.com/blog/lance-file-format-2-2-taming-complex-data
46. SpiralDB — "What if we just didn't decompress it?" — https://spiraldb.com/post/what-if-we-just-didnt-decompress-it
47. Kuschewski et al. — "BtrBlocks: Efficient Columnar Compression for Data Lakes" (SIGMOD 2023) — https://www.cs.cit.tum.de/fileadmin/w00cfj/dis/papers/btrblocks.pdf
48. Boncz et al. — "FSST: Fast Random Access String Compression" (PVLDB 2020) — https://www.vldb.org/pvldb/vol13/p2649-boncz.pdf
49. Armbrust et al. — "Delta Lake: High-Performance ACID Table Storage" (VLDB 2020) — https://www.vldb.org/pvldb/vol13/p3411-armbrust.pdf
50. Apache Iceberg — Evolution Documentation — https://iceberg.apache.org/docs/latest/evolution/
51. Delta Lake UniForm Documentation — https://docs.delta.io/delta-uniform/
52. Zeng et al. — "F3: The Open-Source Data File Format for the Future" (Proc. ACM Manag. Data Vol 3 No 4, Sep 2025; SIGMOD 2026) — https://db.cs.cmu.edu/papers/2025/zeng-sigmod2025.pdf
53. Prammer et al. — "Towards Functional Decomposition of Storage Formats" (CIDR 2025) — https://vldb.org/cidrdb/papers/2025/p19-prammer.pdf
54. Afroozeh et al. — "ALP: Adaptive Lossless Floating-Point Compression" (SIGMOD 2024) — https://dl.acm.org/doi/10.1145/3626717
55. Afroozeh & Boncz — "The FastLanes File Format" (PVLDB 2025) — https://ir.cwi.nl/pub/35881
56. cwida/FastLanes — GitHub — https://github.com/cwida/FastLanes
57. Hepkema et al. — "G-ALP: Rethinking Light-weight Encodings for GPUs" (DaMoN 2025) — https://azimafroozeh.org/assets/papers/g-alp.pdf
58. Liu et al. — "LeCo: Lightweight Compression via Learning Serial Correlations" (arXiv 2023) — https://arxiv.org/pdf/2306.15374
59. IDC — Global DataSphere Forecast 2025–2029 — https://www.idc.com/getdoc.jsp?containerId=IDC_P38353
60. Clarifai — "What is Unstructured Data" — https://www.clarifai.com/blog/what-is-unstructured-data
61. CloudToolStack — Cloud Cost Comparison 2026 — https://cloudtoolstack.com/learn/multi-cloud-cost-comparison-guide
62. CloudToolStack — Multi-Cloud Storage Comparison — https://cloudtoolstack.com/learn/multi-cloud-storage-comparison
63. Meta Engineering — "Introducing OpenZL" (Oct 2025) — https://engineering.fb.com/2025/10/06/developer-tools/openzl-open-source-format-aware-compression-framework/
64. DuckDB — The FastLanes File Format — https://duckdb.org/library/fastlanes/
65. Apache Parquet — Modular Encryption — https://parquet.apache.org/docs/file-format/data-pages/encryption/
66. Databricks — "Delta Lake UniForm: Iceberg Compatibility Now GA" — https://databricks.com/blog/delta-lake-universal-format-uniform-iceberg-compatibility-now-ga
67. ClickHouse — Architecture Overview (VLDB 2024) — https://clickhouse.com/docs/academic_overview
68. DuckDB — "Lightweight Compression in DuckDB" (Oct 2022) — https://duckdb.org/2022/10/28/lightweight-compression
69. Databend — "Is Parquet Becoming the Bottleneck?" (Sep 2025) — https://www.databend.com/blog/category-engineering/2025-09-15-storage-format/
