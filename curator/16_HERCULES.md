## ΜΕΡΟΣ 16: HERCULES — ΙΕΡΑΡΧΙΚΗ LLM ΟΝΟΜΑΤΟΔΟΣΙΑ SUB-FOLDERS
*(Έρευνα 2026-05-22 — 35+ searches, 93 tool uses)*

### HERCULES (arXiv:2506.19992)

**Full title:** "HERCULES: Hierarchical Embedding-based Recursive Clustering Using LLMs for Efficient Summarization"
**GitHub/Package:** https://github.com/bandeerun/pyhercules (`pip install pyhercules`)
**Results:** 7× ARI improvement over LSA baseline

**Τι κάνει:**
- Recursive k-means (ή agglomerative) → multi-level cluster tree
- Σε κάθε επίπεδο, LLM γεννά: `title` (max 5-7 words) + `description` (1-2 sentences) per cluster
- **Key mechanism:** Child summaries εισάγονται στο parent-level prompt → context-aware naming
- Configurable depth: `level_cluster_counts=[4, 5, 3]` → 3 επίπεδα

**Prompt construction (Appendix A):**
- Cluster metadata (level, descendant count)
- Representative samples (centroid_closest)
- Child titles+descriptions (από το προηγούμενο επίπεδο)
- `topic_seed` για domain bias (π.χ. "academic files")

---

### Τα 3 Επίπεδα του Curator — Prompting Pattern

**Level 1 (Domain):** University / Work / Personal
```
Prompt: "Here are N files. Name this broad domain/area of life. Max 5 words."
Context: None (top level)
```

**Level 2 (Subject/Project):** EPL326 / EPL232 / Thesis
```
Prompt: "These files belong to '[L1_NAME]'. Name this specific sub-domain or project. Max 5 words."
Context: L1 folder name
Files: Only files WITHIN the L1 cluster (TopicGPT pattern)
```

**Level 3 (Type/Phase):** Assignments / Notes / Exams
```
Prompt: "These files belong to '[L1]/[L2]'. Name this document type or phase.
         Avoid names already used in siblings: {existing_L3_siblings}. Max 4 words."
Context: Full breadcrumb + sibling anti-constraints (TaxoAdapt pattern)
Files: Only files WITHIN the L2 cluster
```

---

### Depth Decision Rules (research-backed)

| Κανόνας | Πηγή | Τιμή |
|---|---|---|
| ≥ N files → create sub-folder | TaxoAdapt, ACL 2025 | **≥40 files** |
| Mean cosine < threshold → split | Research synthesis | **< 0.65** |
| Max depth | PIM research | **3 levels** |
| Silhouette improvement stops | Haschka 2026 | **< 0.05 improvement L(n)→L(n+1)** |

```python
def needs_subfolder(cluster_embeddings, cluster_labels, max_depth=3, current_depth=0):
    if current_depth >= max_depth: return False
    if len(cluster_labels) < 40: return False  # TaxoAdapt threshold
    mean_cosine = compute_mean_intra_cosine(cluster_embeddings)
    if mean_cosine >= 0.65: return False  # tight enough, stay flat
    return True
```

---

### Name Collision — Δεν είναι πρόβλημα

- File system χειρίζεται via full paths: `University/EPL326/Notes/` ≠ `University/EPL232/Notes/`
- Το ίδιο sub-folder name σε διαφορετικά parents = **εντελώς fine**
- Μόνο UX issue: δείξε always parent breadcrumb, ποτέ μόνο leaf name
- Η anti-constraint prompt (sibling names) αφορά **μόνο siblings at the same parent** — δεν χρειάζεσαι global uniqueness

---

### Libraries & Projects

| Project | `pip install` | Levels | Context-aware? |
|---|---|---|---|
| **pyhercules** | `pip install pyhercules` | Configurable | ✅ child summaries |
| **toponymy** (Tutte Institute — creators of UMAP/HDBSCAN!) | `pip install toponymy` | Multi-resolution auto | ✅ layered |
| **topicGPT** | GitHub only | 2 | ✅ parent seed |
| **haschka/semantic-trees** | GitHub only | Continuous tree | ✅ iterative DBSCAN |

**Toponymy** από τους δημιουργούς του UMAP και HDBSCAN — multi-resolution hierarchy, supports HuggingFace backends, exact output: `[["University","Work"], ["EPL326","EPL232",...], ["Assignments","Notes",...]]`. Άξιζει να δοκιμαστεί πρώτα.

---

### 🆕 Novel Finding

**Κανένα project δεν κάνει fully dynamic 3-level recursive folder generation με zero predefined categories.** Curator θα είναι genuinely novel σε αυτό.

---

**ΑΠΟΦΑΣΕΙΣ (2026-05-21):**
1. ✅ Re-runnable με diff — επιλέγεις φάκελο, ξανατρέχεις οποτεδήποτε, βλέπεις μόνο τι άλλαξε
2. ✅ 3 επίπεδα — LLM δημιουργεί υποκατηγορίες εντός cluster (π.χ. EPL326/Assignments/)
3. ✅ Πρόταση διαγραφής για useCount==0 αρχεία
4. ✅ Auto _versions/ για RETSim >0.86, Review για 0.72–0.86
5. ✅ Unmatched νέο αρχείο → Review

---
