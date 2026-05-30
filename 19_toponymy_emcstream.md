# 19 — Toponymy + EmCStream
_Date: 2026-05-30_

## Toponymy: ✅ USE IT

Real library, maintained by the UMAP/HDBSCAN authors (Leland McInnes, TutteInstitute).
GitHub: https://github.com/TutteInstitute/toponymy
Docs: https://toponymy.readthedocs.io/

**What it does:** Cluster LABELING (not clustering). Sits on top of HDBSCAN, uses LLM to generate human-readable names. Native multi-resolution hierarchy — produces `['University', 'Academic Research', 'ML Papers']` style 3-level paths. No predefined categories.

**BGE-M3 compatible:** Yes — SentenceTransformer API.
**Local LLM support:** Yes — LlamaCpp, HuggingFace backends.
**Replaces k-LLMmeans:** Yes, and it's better — HDBSCAN-native, hierarchy-aware, no fixed k.

`pip install toponymy` (from source) — ready to use.

## EmCStream (METU 2023, Wiley): ❌ DON'T SWITCH FROM DENSTREAM

Full paper: https://onlinelibrary.wiley.com/doi/full/10.1002/sam.11590
Code: https://gitlab.com/alaettinzubaroglu/emcstream

Novel idea: streaming UMAP (2D reduction) + k-means for evolving streams with drift detection.

**Why it fails for Curator:**
1. Requires predefined k — can't know number of file categories in advance
2. 2D UMAP loses too much information from BGE-M3's 1024 dims
3. Designed for high-velocity numeric streams, not 5-20 files/day
4. No text/document benchmark — never tested on document streams
5. Sudden drift detection irrelevant for gradual personal file evolution

DenStream wins on every axis for our use case. Keep it.

## 🚪 New Doors
1. **Toponymy + HERCULES = drop-in replacement** — our 3-level recursive naming can use Toponymy directly instead of custom k-LLMmeans. Reduces implementation complexity significantly.
2. **UMAP/HDBSCAN authors are Toponymy authors** — direct line to Leland McInnes. If we use Toponymy and publish, there's a natural citation and possible outreach opportunity.
3. **DataMapPlot** (also TutteInstitute) — visualizes HDBSCAN clusters as interactive maps. Could become Curator's "visual map of your file system" feature. Search for prior art on interactive file system visualization.
