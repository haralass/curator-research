## ΜΕΡΟΣ 11: LLM NAMING QUALITY + MULTILINGUAL EMBEDDINGS
*(Έρευνα 2026-05-21 — 71 searches, 84 sources)*

### 🔴 ΚΡΙΣΙΜΟ: nomic-embed-text v1/v1.5 = ENGLISH ONLY

Confirmed από Nomic maintainer (HuggingFace discussion, Feb 2024): "only English...for now :D"
Αρχεία ελληνικά embed ως noise. `EPL326_σημειώσεις.pdf` + `EPL326_notes.pdf` **δεν θα κάνουν cluster μαζί ποτέ** με nomic v1.

---

### ΤΟΠΙΚΑ Α: LLM Naming Quality

#### Papers

| Paper | Venue | URL | Key Finding |
|---|---|---|---|
| **k-LLMmeans** | ICLR 2026 | arXiv:2502.09667 | Textual summaries ως centroids. Metrics: ACC+NMI+dist. Streaming name drift documented (intentional). Failure: heterogeneous clusters → poor summaries |
| **Evaluation of Text Cluster Naming with LLMs** (Preiss et al.) | JDS 2024 | jds-online.org/journal/JDS/article/1385 | LLM beats humans με best prompt. 4 strategies tested. Quality varies BY DOMAIN — δεν υπάρχει universal best prompt |
| **LLM Topic Labeling** (Khandelwal) | arXiv:2502.18469, Feb 2025 | arxiv.org/abs/2502.18469 | Metric: cosine(embed(label), embed(cluster docs)). BERTopic + LLM pipeline |
| **T5Score** | ACL 2025 Findings | arXiv:2407.17390 | Decompose naming quality σε sub-dimensions για higher inter-annotator agreement |
| **Goldilocks Zone** (Miller & Alexander) | arXiv:2504.04314, Apr 2025 | arxiv.org/abs/2504.04314 | Optimal clusters: 16–22. Name quality = cosine gap (correct cluster vs. nearest wrong). temperature=0 + local models για stability |
| **Scientific Cluster Labels** | arXiv:2511.02601, Nov 2025 | arxiv.org/html/2511.02601v1 | TF-IDF concepts πιο valuable από titles. Iterative refinement με negative feedback βελτιώνει small models |
| **Reasoning-Based Refinement** (Islam) | arXiv:2604.07562, 2025 | arxiv.org/html/2604.07562v2 | 3-stage: coherence check → dedup (>0.85 cosine = merge) → label grounding |

#### Τι κάνει ένα folder name "καλό" (synthesized criteria)

1. **Specificity**: "EPL326 Notes" > "Computer Science" > "Documents"
2. **Discriminability**: cosine distance >0.2 από κάθε άλλο folder name
3. **Conciseness**: 2–5 words (concept-based beats full sentences)
4. **Stability**: temperature=0, few-shot consistency hints (fed back recent names)
5. **Anti-collision**: merge names με cosine similarity >0.85 (arXiv:2604.07562)

#### Self-Evaluation Metric (zero-cost, ground-truth-free)
```python
name_score = cosine(embed(folder_name), mean(embed(files_in_folder)))
# Κόστος: μόνο 1 embed call για το folder name
# Target: >0.6
```

#### Best Prompt Template (synthesized από όλα τα papers)

```
Given these files in a folder:
Filenames: {top_3_filenames}
Key terms: {top_10_tfidf_terms}
Content snippets: {3_diverse_representative_snippets}
Existing folder names to avoid similarity with: {existing_folder_names}

Generate a folder name that is:
- 2-4 words
- Specific (not "Documents" or "Files")
- Distinct from the existing names above

Folder name:
```

Αν το αποτέλεσμα είναι generic → iterative refinement:
```
"This name is too vague. The folder contains files specifically about [X]. Try again."
```

#### Documented Failure Modes

1. **Over-generalization**: cluster Python files → "Programming" αντί "EPL326 Code"
2. **Name drift**: streaming centroids εξελίσσονται, folder names αλλάζουν μεταξύ runs
3. **Name collision**: δύο folders παίρνουν το ίδιο ή πολύ κοντινό όνομα
4. **LLM inconsistency**: ακόμα με temperature=0, 10% variation across runs (use fixed seed)
5. **Hallucination**: LLM εισάγει info που δεν υπάρχει στα αρχεία
6. **Implicit themes fail**: social/emotional content δεν naming well

---

### ΤΟΠΙΚΑ Β: Multilingual Embedding Models

#### Model Comparison Table

| Model | Greek (el)? | Dims | Size | Ollama | FastEmbed | Context | Notes |
|---|---|---|---|---|---|---|---|
| nomic-embed-text v1/v1.5 | ❌ NO | 768 | 70.9M | Official | Yes | 8192 | **Τωρινό** — πρέπει να αλλαχθεί |
| nomic-embed-text-v2-moe | Πιθανώς (~100 langs) | 256–768 | 475M | Official | No | 512 | MoE, uncertain Greek |
| **multilingual-e5-large** | ✅ YES (XLM-RoBERTa) | 1024 | 560M | Community | **YES** | 512 | Best FastEmbed drop-in |
| multilingual-e5-large-instruct | ✅ YES | 1024 | 560M | Community | No | 512 | MTEB clustering #1 sub-1B (51.5) |
| **BGE-M3** | ✅ YES (100+ langs) | 1024 | 567M | **Official** | No | **8192** | Best για long PDFs |
| paraphrase-multilingual-mpnet | ✅ YES (confirmed list) | 768 | 278M | Official | YES | **128** | Μικρότερο, αλλά 128 token limit! |
| **snowflake-arctic-embed2** | ✅ YES (el confirmed) | varies | varies | **Official** | No | 8192 | Greek explicitly listed |
| **Qwen3-Embedding-0.6B** | ✅ YES (100+ langs) | 1024 | 0.6B | **Official** | No | 32K | **MTEB #1 Multilingual 2025** (70.58) |
| Qwen3-Embedding-8B | ✅ YES | 4096 | 8B | Official | No | 40K | Best quality, heavy |
| LaBSE | ✅ YES (109 langs) | 768 | 471M | Community | No | 512 | No official Ollama |

#### Recommendation Matrix

| Scenario | Model | Command |
|---|---|---|
| **Best overall (Apple Silicon, MTEB #1)** | Qwen3-Embedding-0.6B | `ollama pull qwen3-embedding:0.6b` |
| **Best FastEmbed drop-in** | multilingual-e5-large | `TextEmbedding("intfloat/multilingual-e5-large")` |
| **Best για long PDFs (>512 tokens)** | BGE-M3 | `ollama pull bge-m3` |
| **Best confirmed Greek + Official Ollama** | snowflake-arctic-embed2 | `ollama pull snowflake-arctic-embed2` |
| **Smallest confirmed Greek** | paraphrase-multilingual | `ollama pull paraphrase-multilingual` *(128 token limit!)* |

#### FastEmbed Drop-in Replacement (single line change)

```python
# Πριν:
TextEmbedding("nomic-ai/nomic-embed-text-v1.5")  # 768 dims, English only

# Μετά (Option A — FastEmbed official):
TextEmbedding("intfloat/multilingual-e5-large")  # 1024 dims, Greek ✓

# Μετά (Option B — via Ollama):
# ollama pull qwen3-embedding:0.6b
# Χρήση μέσω http://localhost:11434/api/embeddings
```

⚠️ **Αλλαγή dimensions 768→1024**: πρέπει να re-index το ChromaDB collection.

#### Cross-Lingual Alignment Insight

- Η training objective (not model size) καθορίζει cross-lingual alignment
- Models trained on translation pairs (LaBSE, mE5): SA ~0.68–0.70
- LLM-based embeddings: SA 0.55–0.61 (χειρότερο παρά το μεγαλύτερο size)
- Πρακτικά: `EPL326_σημειώσεις.pdf` θα cluster με `EPL326_notes.pdf` μόνο με mE5/BGE-M3/Arctic

#### Greek NLP Notes (arXiv:2510.20002)

- Modern Greek (modern, monotonic) handles well σε XLM-RoBERTa-based models
- Morphological richness (υψηλά flexed language) → τα SentencePiece tokenizers το handle σωστά
- Greek explicitly in training data XLM-RoBERTa (Wikipedia GR + Common Crawl GR)
- Polytonic ancient Greek → irrelevant για τα αρχεία σου

---
