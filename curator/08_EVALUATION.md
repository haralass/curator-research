## ΜΕΡΟΣ 8: EVALUATION — ΠΩΣ ΞΕΡΟΥΜΕ ΑΝ Η ΟΡΓΑΝΩΣΗ ΕΙΝΑΙ ΚΑΛΗ
*(Έρευνα 2026-05-21 — 52 searches, 80+ papers/projects)*

### 🔴 ΚΡΙΣΙΜΟ: Embedding Model για Multilingual

Το `nomic-embed-text` είναι **αγγλικό μόνο**. Για ελληνικά+αγγλικά αρχεία (π.χ. `EPL326_σημειώσεις.pdf` + `EPL326_notes.pdf`) χρειαζόμαστε:
- **`multilingual-e5`** (sentence-transformers): `pip install sentence-transformers` + `intfloat/multilingual-e5-large`
- **`LaBSE`**: `pip install sentence-transformers` + `sentence-transformers/LaBSE`
- mBERT/nomic: cosine <0.50 cross-lingual, multilingual-e5/LaBSE: >0.70
- Benchmark: arXiv:2402.13512 + PMC 2022

**Ενέργεια πριν υλοποίηση:** Αντικατάσταση nomic-embed-text → multilingual-e5-large (ή LaBSE) για να κάνουν cluster ελληνικά και αγγλικά αρχεία του ίδιου μαθήματος μαζί.

---

### Tier 1 — Ground-Truth-Free Metrics (τρέχουν αυτόματα, πάντα)

| Metric | Paper | Γιατί |
|---|---|---|
| **DBCV** | Moulavi et al., SIAM SDM 2014 | Σωστό intrinsic metric για HDBSCAN. Silhouette ΛΑΘΟΣ γιατί υποθέτει convex clusters. `pip install dbcv` / GitHub: christopherjenness/DBCV |
| **Name-embedding cosine** | k-LLMmeans arXiv:2502.09667 | `cosine(embed(folder_name), mean(embeds(files)))` — μηδενικό κόστος, ground-truth-free |
| **Cohesion Ratio** | arXiv:2511.19350, Nov 2025 | Information-theoretic, interpretable, σχεδιάστηκε για short text embeddings |
| **NPMI** | Aletras & Stevenson, EACL 2013 | Semantic coherence από file content. rs=0.82–0.89 vs human judgment |
| **Temporal churn rate** | arXiv:2512.15210, Dec 2024 | % αρχείων που αλλάζουν cluster όταν φτάνουν νέα (DenStream stability) |

**Σημείωση:** "Ground truth clustering is not the optimum" (Nature Sci. Reports 2025, arXiv:2305.13218) — ακόμα και human-labeled ground truth χειροτερεύει geometric metrics. Χρειαζόμαστε ξεχωριστή human evaluation.

---

### Tier 2 — LLM-as-Judge (automated, at evaluation time)

| Method | Paper | Πρακτική Χρήση |
|---|---|---|
| **ProxAnn** | arXiv:2507.00828, ACL 2025 | LLM infers folder purpose → classifies held-out files. Statistically indistinguishable από human annotators. GitHub: ahoho/proxann |
| **Coherence Verification** | Islam, arXiv:2604.07562, 2026 | LLM boolean: "are these files coherent as a group?" per cluster. Stage 1 of 3-stage pipeline |
| **WALM** | arXiv:2406.09008, TACL 2025 | Joint LLM evaluation labels + member representation. GitHub: Xiaohao-Yang/Topic_Model_Evaluation |
| **Dial-In proxy** | arXiv:2412.09049, EMNLP 2025 | Fine-tuned LLMs: >95% accuracy on cluster coherence, aligned με human judges |

---

### Tier 3 — Implicit Behavioral (deployed app telemetry)

| Signal | Source | Τιμή-Στόχος |
|---|---|---|
| **Search-after-organization rate** | Bergman 2008, ACM TOIS | αν user κάνει Spotlight για αρχείο που οργάνωσες = failed placement. Target <5% |
| **Manual re-organization rate** | Dinneen & Julien 2020, JASIST | `user_moves / organized_files` ανά εβδομάδα. Υψηλό = απόρριψη |
| **Navigation success rate** | Bergman 2010, JASIST | Gold standard: 94% success, ~15 sec, depth ~2.86, ~11.82 files/folder |

---

### Tier 4 — Human Evaluation (occasional benchmarking)

| Framework | Paper | Τι Μετράει |
|---|---|---|
| **CIPHE** | EMNLP 2024 + JDMDH 2025 | Inclusion task: "ανήκει αυτό το αρχείο εδώ?" per file. GitHub: antoneklund/CIPHE |
| **BCubed / ELM** | Amigó 2009, van Heusden 2024 | Precision/recall vs user-labeled ground truth. `pip install bcubed-metrics` |
| **ABCDE** | Google, arXiv:2407.21430, Jul 2024 | A/B eval μεταξύ δύο versions του organizer χωρίς full re-labeling |
| **Task-based retrieval** | Elsweiler & Ruthven, SIGIR 2007 | Standardized re-finding tasks: success rate + time. Blueprint για user study |

---

### Gaps (τι δεν υπάρχει → novel contribution ευκαιρίες)

1. **Multilingual personal file evaluation** — κανένα paper για ελληνικά+αγγλικά personal files
2. **"Reorganization rate" ως formal quality proxy** — χρησιμοποιείται ανεπίσημα, ποτέ δεν validated
3. **Longitudinal evaluation** — κανένα controlled study 6–12 μήνες μετά AI organization
4. **HDBSCAN + LLM-naming end-to-end benchmark** — papers αξιολογούν ξεχωριστά
5. **Personal file benchmark dataset** — δεν υπάρχει public dataset με human-labeled "correct" groupings

---
