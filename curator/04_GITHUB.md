## ΜΕΡΟΣ 4: GITHUB PROJECTS

| Repo | Stars | Τι κάνει | Τι μαθαίνουμε |
|---|---|---|---|
| **PerminovEugene/messy-folder-reorganizer-ai** | ~20 | Rust+Ollama, agglomerative+LLM naming, zero predefined categories | Πιο κοντά στο δικό μας. Batch only, δεν watch live |
| **iyaja/llama-fs** | χιλιάδες | Self-organizing file system, μαθαίνει από manual moves | watch-mode implicit learning pattern |
| **QiuYannnn/Local-File-Organizer** | 3200+ | llama3.2:3b + LlaVA, Nexa SDK, dry-run mode | Multiprocessing pattern |
| **agiresearch/AIOS-LSFS** | - | ICLR 2025: semantic file system με vector DB, "group by" = ad-hoc cluster | Η πιο ambitious academic implementation |
| **different-ai/file-organizer-2000** | 4000+ | Obsidian plugin, LLM folder suggestions, dry-run mode, parallel analysis | Dry-run + approval workflow pattern |
| **obra/knowledge-graph** | - | Obsidian: Louvain community detection, betweenness centrality, PageRank | Graph-based folder discovery implementation |
| **MatthiasCarnein/textClust** | - | TF-IDF stream clustering | Baseline για σύγκριση |
| **AndreasKarasenko/scikit-ollama** | - | ZeroShotOllamaClassifier σε sklearn API | sklearn pipeline με ollama |
| **hyperfield/ai-file-sorter** | - | Taxonomy injection, session consistency hints, learned behavior DB | Session-level RAG για consistency |
| **online-ml/river** | μεγάλο | DenStream, DBSTREAM, TextClust | Η βιβλιοθήκη |
| **idealo/imagededup** | μεγάλο | DHash + CNN για image dedup | imagededup benchmarks |
| **AnubisLMS/Mayat** | - | AST-based code similarity | Code version detection |
| **RhetTbull/osxmetadata** | - | macOS extended attributes | kMDItemWhereFroms, kMDItemCreator |

### Commercial Products (τι μαθαίνουμε)

| Product | Τεχνολογία | Τι μαθαίνουμε |
|---|---|---|
| **Dropbox Smart Move** | Neural net <20 layers, filename+siblings, top 20% confidence | "Potential siblings" πείθει περισσότερο από score |
| **VaultSort** | On-device Apple Neural Engine, ~470MB model, dry-run | Πιο κοντά εμπορικά στο δικό μας |
| **DEVONthink** | TF-IDF cosine similarity (εκτίμηση), implicit training | Cold start limitation — χρειάζεται πολλά αρχεία |
| **Sparkle (Every.to)** | GPT-4o-mini, filename-only, 3-tier (Recents/Manual/AI) | Χρήστες αποδέχονται AI-generated folder names |
| **Google Drive Gemini** | Content embeddings + "Fewer/More folders" slider | User control over granularity |

---
