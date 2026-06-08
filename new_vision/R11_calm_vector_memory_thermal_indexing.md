# R11 — Calm Vector Memory & Thermal-Aware Indexing

> **Core principle: "Curator should organize at the speed of the Mac's comfort, not at the maximum possible speed."**
>
> Every file deserves memory. Not every file deserves attention.
>
> Curator must maintain a living, compressed, thermal-aware semantic memory of the user's filesystem — without heating the Mac, consuming RAM unnecessarily, or interrupting the user's work.

---

## Architecture Overview

Semantic memory in Curator is not a single monolithic vector index. It is a **pyramid of layers**, each serving a different resolution of understanding at a different compute cost.

```
                         ┌─────────────────────────────┐
  Layer 5 (Ephemeral)    │  Temporary High-Precision   │ ← hard decisions only
                         │  Memory (loaded on demand)  │
                         └─────────────────────────────┘
                         ┌─────────────────────────────┐
  Layer 4 (Dynamic)      │  Active Context Memory      │ ← currently active courses/projects
                         │  (high budget, warm)        │
                         └─────────────────────────────┘
                         ┌─────────────────────────────┐
  Layer 3 (Compressed)   │  Compressed Vector Memory   │ ← semantic search, similarity
                         │  (TurboVec / usearch / etc) │
                         └─────────────────────────────┘
                         ┌─────────────────────────────┐
  Layer 2 (Sketch)       │  Semantic Sketch Memory     │ ← cheap signals, always available
                         │  (SQLite + FTS5)            │
                         └─────────────────────────────┘
                         ┌─────────────────────────────┐
  Layer 1 (Foundation)   │  File Identity Memory       │ ← inode, hash, path, type
                         │  (always available, trivial)│
                         └─────────────────────────────┘
```

Each layer is independent. A file can exist in Layer 1 without ever reaching Layer 3. The decision of how far a file climbs is governed by the **Semantic Attention Budget** and the **Thermal Governor**.

---

## 1. Semantic Memory Pyramid — Layer Definitions

### Layer 1 — File Identity Memory

**Cost:** trivial  
**Latency:** microseconds  
**Always present for every known file**

Stores:
- `inode` + `device_id` (primary identity, never path)
- `sha256` (partial or full)
- `path` (current + history via `file_path_history`)
- `file_type` (MIME category)
- `size_bytes`, `mtime`
- `state` (13-state machine, R1)

This layer answers: **which file is this, where is it, has it changed.**  
It does not answer what the file means.

### Layer 2 — Semantic Sketch Memory

**Cost:** cheap (regex, headers, first-page snippet)  
**Latency:** milliseconds  
**Populated during Tier 0 / Tier 1 extraction**

Stores:
- Filename tokens (NFC + accent-stripped + Greek transliteration)
- Folder context (parent path tokens)
- Inferred source domain (downloads, desktop, course folder, etc.)
- Course codes extracted via `UCY_COURSE_PATTERN`
- First-page text snippet (≤ 512 chars)
- Excel headers / code import statements
- OCR short snippet (if cheap to produce)
- Short factual summary (if already generated)

This layer answers: **what is this file approximately about, which context does it belong to.**  
Used for FTS5 search, quick group assignment, course detection, folder suggestion.

### Layer 3 — Compressed Vector Memory

**Cost:** moderate (embedding generation is the bottleneck, not search)  
**Latency:** milliseconds for search, seconds for insert  
**Populated during Tier 2 extraction, only when budget allows**

Stores:
- Compressed embedding per file/chunk
- Vector index with stable file IDs
- Filtered search support (by context, by state, by inclusion list)
- Persistent across restarts

This layer answers: **which files are semantically similar, what is nearby in meaning.**  
Used for context boundary detection, soft duplicate detection, group quality scoring.

The backend for this layer is **not yet decided**. See §4 for benchmark plan.

### Layer 4 — Active Context Memory

**Cost:** variable (budget-controlled)  
**Dynamic: grows and shrinks as user activity changes**

Active contexts (current semester courses, active projects, recent downloads) receive:
- Higher embedding depth
- Faster re-indexing on change
- More workers allocated
- Full vector precision (no compression penalty)

Dormant contexts receive only Layer 1 + 2, with Layer 3 in cold storage.

Reactivation trigger: user opens ≥ 3 files from a dormant context → context promoted to active.

### Layer 5 — Temporary High-Precision Memory

**Cost:** expensive, ephemeral, discarded after decision**

Triggered by hard decisions only:
- "Is this new material a new context or should it merge with an existing one?"
- "Is this a true duplicate or a version?"
- "What is the correct staging destination for this ambiguous file?"

Temporarily loads:
- Full-precision embeddings
- Stronger reranking models
- Deeper extraction (full OCR, full PDF parse)
- Context boundary scoring

After the decision is made, only the final result is stored. The expensive intermediate data is discarded.

---

## 2. Semantic Attention Budget

Not all files deserve the same indexing depth. Curator must allocate compute budget based on relevance, recency, and uncertainty.

### Budget Levels

| Level | Who gets it | Allowed layers |
|---|---|---|
| **High** | Active course, current project, unresolved Review Hub item | 1–5 |
| **Medium** | Recent downloads, uncertain group, pending decision | 1–4 |
| **Low** | Old dormant context, archived course | 1–2 |
| **Minimal** | Duplicate family (unresolved) | 1 only |
| **Zero** | Locked folder, excluded path, `failed_unreadable` files | Layer 1 identity only |

### Budget Assignment Rules

Budget is assigned per **context group**, not per file. A file inherits the budget of its context.

Budget increases when:
- User opens files from that context
- New files arrive in that context
- A Review Hub decision is pending
- Uncertainty score for the group is high

Budget decreases when:
- Context has had no activity for > N days
- All files in context are in `committed` state
- User explicitly archives the context

### Budget and Thermal Interaction

When the Thermal Governor (§3) reduces available compute, budget tiers are applied strictly:
- Under thermal pressure: only High-budget contexts receive Tier 2 extraction
- Under critical pressure: all extraction pauses, only identity work continues

---

## 3. Thermal Governor

Curator must never make the Mac noticeably hot or slow due to background processing. The Thermal Governor is the system that adapts Curator's compute load to the Mac's current state.

### 3.1 Prior Art

**macOS `NSProcessInfo.thermalState`** — Apple provides four levels: nominal, fair, serious, critical. At serious/critical, the OS may already throttle apps. Curator should throttle itself earlier.

**`IOKit` power source API** — distinguishes battery vs. AC power. On battery, mobile devices should run lighter workloads.

**Activity Monitor / `ps`-level heuristics** — detect whether heavy apps (Xcode, Chrome, Zoom, Final Cut) are in foreground.

**Android Doze mode** — Android's system for deferring background work when the device is idle vs. interactive. Curator needs its own equivalent: **Curator Doze**.

**Linux `ionice` / `nice`** — CPU and I/O priority. Python workers should run at low priority (`nice 19`, `ionice -c 3`) so they yield to user apps.

**Kubernetes resource limits** — hard ceiling on CPU/memory per container. Curator workers should self-impose CPU limits.

### 3.2 Thermal States

| State | Condition | Curator behavior |
|---|---|---|
| **Calm** | `thermalState == nominal`, AC power, no heavy apps, user idle | All tiers active, full concurrency |
| **Attentive** | `thermalState == fair` OR battery OR heavy app in foreground | Reduce OCR/embedding workers by 50%, pause Tier 2 for low-budget contexts |
| **Reduced** | `thermalState == serious` OR CPU load > 70% sustained | Pause all Tier 2, pause embedding, only Tier 0/1 + identity work |
| **Paused** | `thermalState == critical` OR user is actively typing/video calling | All extraction paused, only FSEvents + hash + state machine |
| **Asleep** | Mac screen off, no network, no user activity | Full Tier 2 allowed, maximum concurrency, deferred queue drains |

### 3.3 Thermal Governor Implementation

```python
# Proposed interface — not yet implemented
class ThermalGovernor:
    def current_state(self) -> ThermalState:
        """Poll NSProcessInfo, battery status, CPU load."""
        ...

    def allowed_workers(self, tier: int) -> int:
        """Return max concurrent workers for this tier given current state."""
        ...

    def should_pause_tier(self, tier: int) -> bool:
        """Return True if this tier should stop immediately."""
        ...

    def register_callback(self, fn: Callable[[ThermalState], None]) -> None:
        """Notify when state changes."""
        ...
```

The governor polls every 30 seconds and notifies all registered workers on state change.

### 3.4 Worker Priority

All Python sidecar workers should be launched with:
```python
import os
os.nice(15)  # well below interactive priority
```

And I/O should use `O_RDONLY` with `F_RDAHEAD` disabled for large files (avoid polluting the page cache).

### 3.5 Integration with R10 Failure Intelligence

Thermal pressure is treated as a signal, not an error. When the Thermal Governor enters `Reduced` or `Paused` state, the R10 failure system:
- Does **not** mark pending files as failed
- Marks them as `paused_thermal` in the processing queue
- Resumes automatically when state returns to `Calm` or `Attentive`

The user never sees "thermal throttle." The queue simply drains more slowly.

---

## 4. VectorIndex Abstraction

Curator must not hardcode a specific vector index backend. The correct design is a **VectorIndex interface** with swappable backends, selected based on benchmark results.

### 4.1 Interface Definition

```python
# Proposed interface — backend is pluggable
class VectorIndex(Protocol):
    def add(self, file_id: int, vector: np.ndarray) -> None: ...
    def delete(self, file_id: int) -> None: ...
    def search(
        self,
        query: np.ndarray,
        k: int,
        allowlist: list[int] | None = None,
        blocklist: list[int] | None = None,
    ) -> list[tuple[int, float]]: ...
    def save(self, path: Path) -> None: ...
    def load(self, path: Path) -> None: ...
    def count(self) -> int: ...
```

Key requirements that any backend must satisfy:
- **Stable file IDs** — Curator uses integer `file_id` (DB primary key), not vector positions
- **Deletions** — files get deleted, renamed, trashed; the index must support removal
- **Filtered search** — search within a context (allowlist) or excluding locked/ignored files (blocklist)
- **Incremental inserts** — new files arrive daily; full rebuild is not acceptable
- **Persistent** — index survives restarts without full re-embedding
- **Local** — no network calls, no cloud API
- **Apple Silicon** — must run well on M-series chips (ARM NEON / Metal preferred)

### 4.2 Candidate Backends

#### TurboVec / TurboQuant (RyanCodrai/turbovec)

**Claimed:** Rust implementation of Google Research's TurboQuant quantization algorithm for vector compression. Python bindings available.

**Claimed compression ratio:** 31GB (10M float32 vectors at 384-dim) → ~4GB compressed.

**Potential fit for Curator:**
- Reduced RAM for large semantic memory
- Fast ARM-optimized kernels (potential benefit on Apple Silicon)
- Filtered search with allowlists (matches Curator's context-scoped search requirement)
- Online ingest support
- Pure local

**Open questions — must be answered by benchmark:**
- Actual recall@10 at Curator's working set sizes (30k, 100k, 1M vectors)?
- Does quantization hurt context boundary detection quality?
- Insert/delete latency at sustained 100 files/day ingestion rate?
- Actual Apple Silicon performance vs. usearch?
- Memory mapped or RAM-only?
- Python GIL behavior?

**Current status:** Research-only until benchmark completes.

#### usearch (unum-cloud/usearch)

**Production ready.** Used in production by multiple organizations.

**Strengths for Curator:**
- HNSW graph with memory-mapped storage (index can exceed RAM)
- Custom quantization (int8, fp16, fp32)
- Stable external ID support
- Filtered search
- Apple Silicon optimized (ARM NEON)
- Python + Swift bindings
- Active development

**Current status:** Default candidate for Layer 3. Already transitively available via `unisim`. Benchmark should verify actual memory/latency profile for Curator's workload.

#### FAISS (facebookresearch/faiss)

**Battle-tested.** Used in Curator's current pipeline.

**Weaknesses for Curator:**
- No stable external IDs (uses internal integer positions — requires separate mapping)
- No built-in filtered search
- Rebuild required for index type changes
- Heavier dependency

**Current status:** Already in use. Keep as fallback. Likely replaced by usearch for Layer 3.

#### sqlite-vec

**Emerging.** SQLite extension for vector search, maintained by Alex Garcia.

**Potential fit:**
- Single-file storage (no separate index file)
- SQL-native filtering (WHERE context_id = X)
- Transactional (ACID)
- Very small dependency

**Weaknesses:**
- Slower than dedicated index for large-scale ANN search
- No HNSW (currently brute-force or simple quantized search)

**Current status:** Interesting for small-scale search (< 50k vectors) or as complement to FTS5. Benchmark at Curator's scale.

#### Raw fallback

For small corpora (< 5k vectors), cosine similarity via numpy is fast enough and requires no index at all. This is the zero-dependency path for edge cases.

### 4.3 Adoption Decision Rule

**No backend is adopted just because it looks impressive.**

Adoption requires passing all of the following Curator-specific checks:

| Check | Threshold |
|---|---|
| Recall@10 at 100k vectors | ≥ 0.90 |
| Insert latency (single file) | ≤ 100ms |
| Delete latency (single file) | ≤ 50ms |
| RAM usage at 100k 384-dim vectors | ≤ 500MB |
| Query latency (filtered, within context of 5k files) | ≤ 20ms |
| Apple Silicon CPU utilization during query | ≤ 5% sustained |
| Thermal impact during batch insert (100 files) | `thermalState` must not exceed `fair` |
| Index save/load time | ≤ 5 seconds |
| Greek filename recall (accent-stripped queries) | ≥ 0.85 |

---

## 5. Semantic Cold Storage

Active and dormant contexts must be handled differently. Storing everything at full resolution forever is not acceptable for a local tool running on a personal Mac.

### 5.1 Cold Storage Model

When a context becomes dormant (no user activity for > N days, all files in `committed` state):

**Kept (cold storage):**
- Layer 1 (file identity — always free)
- Layer 2 sketch (FTS5 — minimal disk, no RAM)
- Compressed vectors in Layer 3 (disk, not RAM-loaded)
- Group membership and summaries

**Released:**
- Active graph structures
- Full-precision vector cache
- High-budget extraction queue entries

**Transition: dormant → active:**
- Triggered when user opens ≥ 3 files from the context within a session
- Curator gradually re-warms the context's Layer 4 budget
- No user action required

### 5.2 Cold Storage and Benchmark

The benchmark (§6) must include a cold storage scenario:
- Index with 1M vectors, 950k in cold storage, 50k active
- Measure: query latency restricted to active only
- Measure: cold-to-warm reactivation time for a 5k-file context

---

## 6. Benchmark Plan

### 6.1 Test Corpus

| Scale | Description |
|---|---|
| 30k vectors | Current Curator working set |
| 100k vectors | Medium growth (2–3 years) |
| 1M vectors | Large scale (chunks, screenshots, versions) |
| Embedding dim | 384 (default from `unisim` / `paraphrase-multilingual-MiniLM-L12-v2`) |
| Language mix | Greek filenames 40%, English 50%, mixed 10% |
| Greeklish queries | 20% of search queries |

### 6.2 Operations Under Test

- **Insert:** single file, batch (100 files), daily incremental (50 files/day for 30 days)
- **Delete:** single file, batch (10 files)
- **Search:** unfiltered ANN, filtered by context (allowlist 5k), filtered excluding locked (blocklist 200)
- **Save/load:** full index persistence
- **Cold start:** load index after OS restart
- **Concurrent access:** 2 workers inserting, 1 worker searching simultaneously

### 6.3 Metrics

For each backend × scale × operation:
- **Latency** (p50, p95, p99)
- **RAM usage** (RSS before, during, after)
- **CPU utilization** (sustained)
- **Thermal pressure** (NSProcessInfo.thermalState readings)
- **Battery drain** (mWh per 1k operations, on battery)
- **Recall@k** (k=5, k=10, k=20)
- **Group quality delta** (does compression degrade Leiden clustering result?)
- **Context boundary accuracy** (does recall loss cause context merges/splits errors?)

### 6.4 Decision Output

After benchmark, each backend is classified:

| Decision | Meaning |
|---|---|
| **MVP** | Adopt immediately for Layer 3 |
| **Phase 2** | Adopt after MVP launches, pending more profiling |
| **Research-only** | Promising but not production-ready for Curator |
| **Reject** | Does not pass Curator-specific requirements |

---

## 7. Relationship to Other Research Documents

| Document | Relationship |
|---|---|
| R0_synthesis.md | Layer 3 vector search referenced; R11 defines the concrete backend choice process |
| R1_file_state_machine.md | State machine gates which files are eligible for Layer 3 indexing |
| R2_duplicate_version_detection.md | Uses Layer 3 similarity scores; compression must not degrade duplicate detection |
| R4_context_graph_boundary.md | Context boundary detection uses Layer 3; compression recall loss must be acceptable |
| R10_failure_intelligence_self_healing.md | Thermal Governor failure states connect to R10 failure categories |
| TECH_engineering_foundation.md | VectorIndex persisted separately from SQLite; path defined in engineering foundation |

---

## 8. Open Questions

| # | Question | Status |
|---|---|---|
| Q1 | Which backend passes the Curator benchmark? | Open — requires benchmark |
| Q2 | What is the correct quantization dimension for 384-dim Greek/English mixed corpus? | Open |
| Q3 | Does TurboQuant compression cause measurable context boundary detection regression? | Open |
| Q4 | What is the correct cold-to-warm reactivation latency target? | Open — needs UX input |
| Q5 | Should Layer 3 be a single shared index or per-context sub-indexes? | Open — depends on filtered search performance |
| Q6 | What is the correct thermal polling interval? 30s? 60s? | Open — needs profiling |
| Q7 | Should the Thermal Governor expose controls to the user? | Likely no — fully automatic |

---

## Summary

- Curator's semantic memory is a **five-layer pyramid**, not a flat index.
- The **Semantic Attention Budget** controls which files climb which layers.
- The **Thermal Governor** adapts compute load to the Mac's thermal/power state.
- The **VectorIndex abstraction** allows pluggable backends without locking in TurboVec.
- **No backend is adopted without passing Curator's benchmark.** TurboVec is a promising candidate, not a committed choice.
- Dormant contexts go into **Semantic Cold Storage** and reactivate automatically.
- The user never sees any of this machinery. They see fast search, calm thermals, and a Mac that stays cool.
