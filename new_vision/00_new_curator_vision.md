# Curator — New Vision (2026-06-08)

> This document is the canonical reference for all research in this folder.
> Every research worker must read this before starting their topic.

---

## What Curator Is (New Definition)

**Curator = local, neurodivergent-friendly filesystem autopilot for macOS.**

Curator reads, understands, groups, stages, and learns from the user's filesystem — with the goal of reducing chaos without overwhelming the user with endless decisions.

**Core positioning:**
> Curator does not organize files. It organizes decisions.

**Target users:** Anyone with a chaotic filesystem — especially people with ADHD, executive dysfunction, cognitive overload, or simply too many files accumulated over years.

---

## What Changed from the Old System

The old Curator pipeline:
```
SNF → Louvain → individual proposals → user approves one by one
```
This is **file/proposal-first**. It fails at scale because 30,000 files → thousands of individual review cards.

The new Curator pipeline:
```
adaptive scan → duplicate/version gate → context graph → group-first understanding
→ physical _Curator Review Hub → group-level decisions → commit/restore
```
This is **context/group-first**. The user guides groups, not individual files.

**Old pipeline status:** SNF + Louvain are kept as prior work / baseline / possible internal clustering tool. They do NOT define the new product flow.

---

## Core Philosophy

**Many files → few groups → few decisions.**
Not: 1000 files → 1000 review cards.
But: 1000 files → 20 organized decision groups.

**Key principles:**
1. The existing folder structure is a **signal, not truth**. A file in `Downloads/University PDFs` may belong elsewhere. A file in `Desktop/New Folder/New Folder 2` may be important.
2. **Lock = ignore completely.** Not read-only. Not "move but don't delete." Total ignore: no scan, no embedding, no graph signal, nothing.
3. **No permanent final organization.** Files have a "current stable organization" that evolves as new context appears.
4. **Locks live on the item, not in Settings.**
5. **If Curator touches a file, the user can see exactly what happened.**

---

## Two Modes

### Initial Cleanup Mode
First run on a large folder (Downloads, Documents). Curator sees everything, builds context graph, identifies groups, duplicates, versions, and stages uncertain material to `_Curator Review`. The user guides groups, not individual files.

### Daily Autopilot Mode
After initial cleanup. Curator works only on new, changed, moved, or recently accessed files. Delta scanning. Fast.

---

## The Pipeline (New Architecture)

```
File enters system
    │
    ▼
[1] Identity Check
    inode + device_id → SHA-256 → filename+size+mtime fallback
    │
    ▼
[2] Duplicate / Version Gate  ← MANDATORY, runs before anything else
    Exact hash → near-dup → version family detection
    │
    ├── exact duplicate → Duplicate Family group
    ├── version → Version Family group
    └── unique → continue
    │
    ▼
[3] Adaptive Reading
    Router decides: cheap signals → medium signals → expensive signals
    "Enough-to-Act" threshold → stop when sufficient for next safe action
    │
    ▼
[4] Context Graph Update
    Add node, compute edges (semantic + NER + behavioral + temporal)
    Context Birth Detection, Context Boundary Detection
    Folder Coherence evaluation
    │
    ▼
[5] Group Assignment
    File assigned to existing context, new context candidate, or uncertain
    │
    ├── high confidence → existing group → stage for commit
    ├── new context candidate → new group → Review Hub
    ├── uncertain → Review Hub / appropriate subfolder
    └── failed to read → Review Hub / Failed to Read
    │
    ▼
[6] _Curator Review Hub (physical staging)
    Real files moved (not aliases) to organized subfolders
    Original path recorded, restore always possible
    │
    ▼
[7] User Guides Groups (not files)
    approve / rename / split / merge / commit / restore / lock / new context
    │
    ▼
[8] Safe Learning
    Corrections accumulate → pattern validation → suggested rule → confirmed rule
    Never auto-rule from single correction
```

---

## Physical `_Curator Review` Hub Structure

```
_Curator Review/
    University Material/
        Unknown Course/
        EPL342 - Candidate/
    New Context Candidates/
        Cyber Security - New?/
    Duplicates/
        Thesis Draft Versions/
        Duplicate Group 001/
    Versions/
        Report Version Family/
    Failed to Read/
        Protected PDFs/
        Corrupted Files/
    Code Projects/
        Possible Node Apps/
    Spreadsheets & Data/
        Finance-like/
    Mixed Folder Contents/
        Downloads - Incoherent/
```

Organization is **context-first**, not file-type-first.

---

## UI Structure (New)

| Section | Content |
|---|---|
| **Home** | Scan status, what needs attention, health (only when problem exists) |
| **Review Hub** | All staged groups, duplicates, failed reads, new context candidates |
| **Organized** | Contexts, courses, projects that Curator has understood and committed |
| **Search** | Semantic search across all indexed files |
| **Activity** | What Curator did, staging moves, commits, undo/restore |
| **Settings** | Only: allowed folders, Review Hub location |

Locks → on the item itself (file/folder/group), not in Settings.
Rules/Learning → in Review Hub or Organized, not in Settings.
Repair → Home status card, not a main tab.

---

## File States

Every file must have a clear state at all times:

| State | Meaning |
|---|---|
| `new` | Just discovered, not yet processed |
| `metadata_only` | Fast pass complete, not yet deeply read |
| `partially_understood` | Some signals extracted, not fully embedded |
| `fully_understood` | Embedded, NER done, graph-ready |
| `duplicate` | Exact or near-duplicate confirmed |
| `version` | Part of a version family |
| `grouped` | Assigned to a context group |
| `staged_review` | In `_Curator Review`, awaiting user guidance |
| `committed` | User approved, in stable/final location |
| `failed_to_read` | Extraction failed after N attempts, in Review Hub |
| `ignored` | Locked — never process again |
| `user_corrected` | User manually overrode Curator's assignment |

---

## Research Layers

**Layer 1 — Foundation** (must complete first, others depend on these)
- R1: File State Machine & Pipeline Gates
- R2: Duplicate & Version Detection
- R3: Adaptive Reading Engine & Enough-to-Act

**Layer 2 — Intelligence** (after Layer 1)
- R4: Context Graph & Boundary Detection
- R5: Safe Learning & Pattern Validation

**Layer 3 — UX & Presentation** (after Layer 1 + R4)
- R6: Review Hub Physical Staging Design
- R7: Group-level Correction UX & Neurodivergent-Friendly Design
- R8: Activity Log & Undo Model

**Parallel** (independent)
- R9: Evaluation Metrics

---

## What Each Research File Must Deliver

Each Rn file must contain:
1. **Prior art** — what exists in literature, GitHub, commercial products
2. **Key papers** with citations and DOIs
3. **GitHub repos** with stars/status
4. **Technical findings** — what approaches work, what fails, why
5. **Design decisions** — specific recommendations for Curator
6. **Open questions** — what is genuinely novel / unsolved

Format: comprehensive markdown. Not a summary. Not just bibliography. Must end with actionable decisions.
