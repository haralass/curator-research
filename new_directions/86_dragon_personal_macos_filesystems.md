# DRAGON: Applicability to Personal macOS File Collections

**File:** 86 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
DRAGON (arXiv:2602.09071) is a repository classifier achieving 60.8% F1@5 on large-scale software collections using only file/directory names and optional README text. It is not designed for personal macOS filesystem collections — its training and evaluation domain is software repositories on platforms like GitHub. However, its core insight (lightweight structural signals = file paths + directory names are sufficient for classification) directly validates Curator's NER-on-filename approach and suggests that embedding file path tokens is a high-signal feature that Curator may be under-utilizing.

## Key Findings
- **Paper title:** "DRAGON: Robust Classification for Very Large Collections of Software Repositories," arXiv:2602.09071, February 2026.
- **Task:** Multi-class topic classification of source code repositories (e.g., "machine-learning", "web-framework", "data-visualization").
- **Signals used:** (1) file names, (2) directory names, (3) README text (optional). No code content is parsed.
- **Performance:** F1@5 = 60.8%, surpassing prior SotA of 54.8%. When README is absent, F1 drops only 6 pp → file/directory names carry ~90% of signal.
- **Architecture:** Hierarchical bag-of-tokens over directory tree structure; no GNN, no graph component. Token vocabulary built from path segments.
- **Evaluation domain:** Software repositories only. No evaluation on personal documents, photos, PDFs, or mixed-content personal collections.
- **Transferability to Curator:** Direct transfer is not feasible — the label space (software topics) does not match personal folder semantics (project, finance, medical, creative). However, the feature engineering lesson is transferable: **path segment tokens are strong classifiers**, stronger than file content for structural classification.
- **F1 on personal macOS collections:** Not evaluated; cannot be assumed. Personal collections have higher label ambiguity, smaller per-category counts, and more heterogeneous naming conventions. Expected F1 on personal data would be substantially lower (estimate: 30–45% F1@3) without domain-specific fine-tuning.
- **What Curator can borrow:** The path tokenization scheme — split paths by `/`, `_`, `-`, camelCase boundaries; build a TF-IDF or embedding over these tokens as an additional affinity signal alongside semantic embeddings.

## Relevant Papers / Prior Art
- DRAGON: arXiv:2602.09071 (2026) — direct reference; software repo classification.
- Maarek Y.S. et al., "A probabilistic approach to automatic file indexing," *SIGIR* 1991 — early work on filename-based retrieval; shows filename tokens have ~3× the discriminative power of content tokens for retrieval tasks.
- Bergman O. et al., "The effect of folder structure on personal file navigation," *JASIST* 61(12):2426–2441, 2010. DOI: 10.1002/asi.21415. — Empirical study of personal folder semantics; shows personal folders are often project-centric, not topic-centric.

## Applicability to Curator

DRAGON's finding that path tokens dominate suggests adding a fourth affinity signal to Curator's SNF:

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity
import re

def tokenize_path(path: str) -> str:
    """Split path into tokens: directories + filename stem, split on /._- and camelCase."""
    parts = re.split(r'[/\\_\-\.]', path)
    tokens = []
    for part in parts:
        # camelCase split
        tokens.extend(re.sub(r'([A-Z])', r' \1', part).lower().split())
    return ' '.join(tokens)

paths = [tokenize_path(f.path) for f in files]
vec = TfidfVectorizer(ngram_range=(1, 2), min_df=2)
X_paths = vec.fit_transform(paths)
S_path = cosine_similarity(X_paths).astype(np.float32)
# Add S_path as a 4th modality in SNF alongside S_sem, S_ner, S_beh
```

**Recommendation:** Add S_path as a 4th SNF modality. Cost is minimal (TF-IDF is fast); benefit is DRAGON-validated signal that is especially strong for files with descriptive paths (code projects, document hierarchies) and degrades gracefully to noise for files with non-descriptive names (IMG_1234.jpg).

## Open Questions
- Does DRAGON's model (trained on GitHub repos) produce useful embeddings if fine-tuned on user-labeled personal folders? Would require labeled data Curator doesn't have at install time.
- Is path tokenization redundant with NER (File 91)? Partial overlap but complementary — NER extracts entity types (person, org, date), path tokens capture domain vocabulary (invoice, draft, final, v2).

## Sources
- https://arxiv.org/abs/2602.09071
- https://www.arxiv.org/pdf/2602.09071
