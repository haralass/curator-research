# PRISM Embedding Fine-Tuning
**File:** 71 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
PRISM is an active family of retrieval/personalization frameworks. The most relevant variant for Curator is PRISM-X (arxiv 2605.13307), which shows that preference fine-tuning (P-DPO) on pooled diverse preferences outperforms both generic models and individually-personalized prompting — meaning Curator should fine-tune a shared embedding model on pooled user acceptance/rejection signals rather than building per-user models. The VLM bias-reduction PRISM (ICCV 2025) is separately useful for reducing spurious file type associations in embeddings.

## Key Findings
- **PRISM-X (2605.13307):** 530-participant study across 52 countries — P-DPO (preference-DPO fine-tuning) significantly beats generic models; individual personalization yields only marginal gains over pooled training. Key implication: a single fine-tuned embedding model beats N user-specific models.
- **PRISM paper-retrieval (2507.10057):** multi-aspect query optimization — relevance is multi-faceted (motivation, methodology, findings) — applicable to file retrieval where a file is relevant for multiple reasons simultaneously
- **PRISM VLM (ICCV 2025, arxiv 2507.08979):** LLM-guided embedding projection removes spurious co-occurrence biases — could correct for file-type stereotypes (e.g., `.xlsx` always classified as "finance" even when it's a project tracker)
- **PRISM sequential recommendation (2601.16556):** purified representations with integrated semantic modeling — the "purification" idea (strip spurious features before embedding) is valuable for noisy file metadata

## Relevant Papers / Prior Art
- "PRISM-X: Experiments on Personalised Fine-Tuning with Human and Simulated Users" — arxiv 2605.13307
- "PRISM: Fine-Grained Paper-to-Paper Retrieval with Multi-Aspect-Aware Query Optimization" — arxiv 2507.10057
- "PRISM: Reducing Spurious Implicit Biases in Vision-Language Models with LLM-Guided Embedding Projection" — arxiv 2507.08979 / GitHub MahdiyarMM/PRISM
- PRISM sequential recommendation — arxiv 2601.16556

## Applicability to Curator
Medium. The pooled-preference finding (P-DPO beats per-user) is directly actionable policy: collect accept/reject signals across users, fine-tune once, ship updated model. The bias-reduction framing is useful for correcting file-type stereotypes in the embedding space — an `.xlsx` project tracker should embed closer to other project files, not other spreadsheets. Full PRISM fine-tuning requires training infrastructure beyond a local app, but shapes how Curator should collect and use feedback for future model updates.

## Open Questions
- Does the pooled-preference advantage hold at small N (a single user's 200 approve/reject events)?
- How to implement P-DPO fine-tuning within Curator's privacy-first, local-only model?
- Can the VLM-style bias correction be approximated via metadata re-weighting without full fine-tuning?

## Sources
- https://arxiv.org/abs/2605.13307
- https://arxiv.org/abs/2507.10057
- https://arxiv.org/html/2507.08979v1
- https://github.com/MahdiyarMM/PRISM
