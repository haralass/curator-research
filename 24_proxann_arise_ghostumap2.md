# 24 — ProxAnn + ARISE + GhostUMAP2
_Date: 2026-05-30_

## ProxAnn (ACL 2025)
Paper: arXiv:2507.00828 — "ProxAnn: Use-Oriented Evaluations of Topic Models and Document Clustering"
GitHub: https://github.com/ahoho/proxann

LLM-as-judge for cluster quality, "statistically indistinguishable from human annotators."
**Limitation for Curator:** Text-oriented — needs readable text content per cluster. Not designed for metadata-only or mixed-type clusters. Usable only if cluster members have meaningful textual content (filenames, document text). 

**For us:** Viable for text-heavy clusters (PDFs, code files). Not viable for image/binary/metadata-only clusters without adaptation.

## ARISE (ICLR 2025) ✅ DIRECTLY APPLICABLE
Paper: arXiv:2601.01162 — "Bridging the Semantic Gap for Categorical Data Clustering via LLMs"
GitHub: https://github.com/develop-yang/ARISE

Solves EXACTLY our FLACON flag problem: categorical values (e.g., kMDItemContentType = "public.pdf", "com.apple.application") → LLM generates semantic description per unique value → embed → use as distance basis instead of one-hot encoding.

**For us:** Replace one-hot encoding in FLACON flag distance with ARISE-generated embeddings. One LLM query per unique flag value (done once, cached). This is the implementation of our FLACON β component.

## GhostUMAP2 (arXiv:2507.17174) ✅ INSTALLABLE, USEFUL
`pip install ghostumap2`
GitHub: https://github.com/jjmmwon/ghostumap

Quantifies UMAP point instability by placing M "ghost" duplicates near each point, running UMAP, measuring max displacement. (r,d)-stable if displacement d_i ≤ d.

~1% of points are unstable under typical settings. Unstable points → HDBSCAN boundary members → flag as uncertain before clustering.

**For us:** Run as pre-clustering validation. Unstable file-points → route to human review instead of auto-assignment. ~5x overhead → run only during 6-month refit health checks (as already planned in 23_UMAP_STABILITY).

## 🚪 New Doors
1. **ARISE + FLACON implementation** — now we have the exact paper to implement the flag-semantic embedding. ARISE is from ICLR 2025, well-cited. If we implement ARISE within FLACON for macOS metadata, we can claim "first application of ARISE to filesystem metadata clustering."
2. **GhostUMAP2 + File Biography integration** — unstable UMAP points AND files with high Markov entropy (file biography) should align. If they do, that's an empirical validation that both metrics capture the same underlying instability. Cross-validate.
3. **ProxAnn adaptation for files** — adapting ProxAnn to evaluate file clusters (using filename + extracted text as the judge input) could be a short methods paper. Clean, measurable contribution.
