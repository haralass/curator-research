# Folder Biography: Classification, Completeness Scoring & Actionable Suggestions
**File:** 81 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
A folder biography system infers what a folder *is* (its type/class), what it *should* contain (expected file distribution), how complete it currently is (completeness score), and what is actionably missing. The closest existing research comes from repository classification (DRAGON, 2026), patent-based content-similarity folder classification, and document completeness scoring in fintech. No off-the-shelf system does all four steps for general-purpose macOS folders — this is a genuine gap Curator can own.

## Key Findings

### Signal categories beyond file types
| Signal | Weight |
|---|---|
| File-type distribution | High |
| Naming patterns (date-prefix, numbered sequences) | High |
| README / manifest presence | High |
| Subfolder structure depth | Medium |
| Date clustering (creation date spread) | Medium |
| Access recency | Medium |
| Parent folder name ("Clients/", "Archive/") | Low-Med |

### Six folder classes supported by prior art
- **Project** — mixed types, README likely, date-spread, subfolders by phase
- **Client** — named by person/org, invoices + contracts + correspondence
- **Archive** — homogeneous types, old dates, rarely accessed, flat
- **Media** — high ratio of image/video/audio, numbered filenames, no docs
- **Reference** — PDFs/notes, random dates, accessed often, flat
- **Temp** — short lifespan, mixed types, no structure, recent timestamps only

### DRAGON (arXiv 2602.09071, 2026)
The most relevant prior art — classifies software repositories using only filenames + directory names + optional README, achieving F1@5 of 60.8%. Demonstrates that content-free, structure-only signals are sufficient for a first-pass classifier. README presence adds ~6% F1. Directly applicable to Curator's cold-start phase.

### Expected file type distribution — Dirichlet-Multinomial model
- Each class has a prior distribution (e.g., Project: 30% docs, 20% code, 20% images, 10% spreadsheets, 20% misc)
- Priors updated from user's own confirmed folder types (Bayesian update)
- Posterior becomes the "expected" distribution for completeness scoring

### Completeness Score formula (Heron Data pattern)
```
completeness(F) = Σ_i  w_i · present(file_type_i, F)  /  Σ_i  w_i
```
Richer version uses **soft presence**: cosine similarity of the best-matching actual file to the slot's centroid embedding — handles "agreement_v3.pdf" matching the "contract" slot.

### Missing file suggestions — three levels
1. **Type-level (cold):** "This Project folder has no README — 87% of similar folders include one."
2. **Name-level (warm):** "Folders like this typically contain a file matching `*_brief.*` or `*_spec.*`."
3. **Content-level (hot):** "Based on the contract found, a countersigned copy appears to be missing." (requires NER, feeds from File 70)

### Cold start minimum
- Class priors bootstrapped from 50–100 labeled folders from user's own system (5–10 per class)
- Reliable posterior: ~20+ folders per class for Dirichlet update to dominate prior
- Cold start fallback: if <5 similar folders observed, use global prior with explicit uncertainty ("Based on limited data…")

### Graph-based approach
Folders as nodes with: sibling edges (same parent level), contains edges (folder→file bipartite), co-use edges (files co-opened). Label propagation over folder graph using class posteriors as seeds. Feeds into File 82's heterogeneous graph as Folder node features.

## Relevant Papers / Prior Art
- DRAGON (arXiv 2602.09071, 2026) — repo classification from filenames + README
- PathSim / HIN (Sun et al., VLDB 2011) — meta-path similarity for folder-type graph
- HAN (arXiv 1903.07293, 2019) — hierarchical attention over meta-paths
- Document Coherence (arXiv 1507.08234) — entropy-based coherence → classification confidence
- Heron Data Document Completeness Score — checklist-based completeness formula
- USPTO 7630946 / 7937352 — folder classification by content similarity
- USPTO 12517866 — auto-generating destination folder classes from content analysis
- DEVONthink Auto Classify — kNN over TF-IDF; matures with database size

## Applicability to Curator
- **Phase 1 (cold):** DRAGON-style filename + subfolder-name signals + README presence. No content parsing. Single-pass over directory tree.
- **Phase 2 (warm):** Update per-class Dirichlet priors from user's confirmed folder types. Completeness score becomes personalized.
- **Phase 3 (hot):** NER over file content (File 70) feeds entity-level missing-file suggestions.
- The folder biography produces a **node feature vector** for the heterogeneous graph (File 82): folder class + completeness score become edge-type weights.

## Open Questions
- How to handle hybrid folder types (both Client AND Archive)? Multi-label vs. confidence-ranked single label?
- What granularity for file-type slots — MIME type, or higher-level "document role" (contract, brief, asset, backup)?
- Can completeness score serve as a *priority signal* for Curator's suggestions UI — surfacing the most incomplete active folders first?
- How to avoid over-suggesting when a folder legitimately has no README — learn per-user suppression rules?
- Is label propagation over the folder graph meaningfully better than a flat per-folder classifier at Curator's scale?

## Sources
- https://arxiv.org/abs/2602.09071
- https://www.herondata.io/glossary/document-completeness-score
- https://arxiv.org/pdf/1507.08234
- https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/7630946
- https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/12517866
- https://data.research.cornell.edu/data-management/sharing/readme/
- https://arxiv.org/pdf/1903.07293
