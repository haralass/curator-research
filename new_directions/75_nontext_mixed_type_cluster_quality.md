# Non-text / Mixed-type Cluster Quality Evaluation
**File:** 75 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
Standard clustering metrics (silhouette, Davies-Bouldin) degrade on mixed-type file corpora (text + images + code + binaries) because they assume a single distance metric over homogeneous features. Curator should use Adjusted Rand Index (ARI) against user-confirmed groupings as the primary quality metric, supplement with cluster-type-conditioned metrics, and treat the proportion of continuous vs. categorical features per cluster as a diagnostic variable.

## Key Findings
- **2025 comparative study (arxiv 2511.19755):** On mixed-type data, KAMILA, LCM, and k-prototypes have best ARI performance; degree of cluster overlap and proportion of continuous variables are the dominant factors in relative performance
- **ARI** is robust to cluster size imbalance and label granularity — important for file corpora where one "cluster" might be 500 Downloads items vs. 3 research papers
- **UMC (arxiv 2405.12775):** Augmentation-view pre-training for multimodal data — 2–6% improvement over SOTA on clustering metrics
- Evaluation practice is underdeveloped: of 63 clinical clustering studies reviewed, only 38% formally evaluated cluster quality — the field norm is weak, so Curator can be a leader here
- For non-text files (images, audio): embedding-based clustering (CLIP for images) followed by standard metrics works; the challenge is that ground truth labels are expensive (user confirmations are the only viable ground truth source for personal files)

## Relevant Papers / Prior Art
- "Clustering Approaches for Mixed-Type Data: A Comparative Study" — arxiv 2511.19755 (2025)
- "Unsupervised Multimodal Clustering for Semantics Discovery" — arxiv 2405.12775
- "Benchmarking Distance-Based Partitioning Methods for Mixed-Type Data" — arxiv 2203.16287
- "Multimodal Image Clustering via Textual Descriptions" — ACM 2024

## Applicability to Curator
Medium. Curator needs an offline evaluation loop to measure whether its clustering is improving over time. Practical implementation: compute ARI separately per file-type stratum (text-only, image-only, mixed) to diagnose which modality is driving errors. This feeds directly into the self-tuning loop (Topic 72).

Evaluation schema:
```
cluster_eval table:
  - cluster_id, file_type_stratum, ari_score, silhouette_score
  - evaluated_at, num_confirmed_labels, num_files
```

Nightly batch: for each cluster that has ≥10 user-confirmed labels, compute ARI against confirmed labels. Surface declining clusters to the user as "this folder is getting less coherent — want to review?"

## Open Questions
- How to collect enough confirmed labels for statistically meaningful ARI estimates? (Need ~50 per cluster type)
- Should cluster quality scores be surfaced to the user as a "confidence in this folder's coherence" metric?
- Is k-prototypes implementable efficiently in Python sidecar for Curator's corpus sizes (1k–100k files)?

## Sources
- https://arxiv.org/pdf/2511.19755
- https://arxiv.org/pdf/2405.12775
- https://arxiv.org/pdf/2203.16287
- https://dl.acm.org/doi/10.1145/3641181.3641191
