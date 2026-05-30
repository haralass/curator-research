## ΜΕΡΟΣ 3: ΟΛΕΣ ΟΙ ΑΚΑΔΗΜΑΪΚΕΣ ΕΡΓΑΣΙΕΣ

### Clustering & Organization

| Paper | Venue | Σχέση με εμάς |
|---|---|---|
| **FLACON: Information-Theoretic Flag-Aware Clustering** | MDPI Entropy 2025 | 7.8× βελτίωση, 6-dimensional flags (Type+Domain+Priority+Status+Relationship+Temporal), κανένα file organizer δεν το έχει εφαρμόσει |
| **k-LLMmeans: Summaries as Centroids** | arXiv:2502.09667, 2025 | Τα centroids είναι LLM-generated κείμενο — αυτόματο folder naming που εξελίσσεται |
| **Information-Theoretic Generative Clustering** | arXiv:2412.13534, AAAI 2025 | KL divergence σε LLM distributions αντί cosine — state-of-the-art, ανεξερεύνητο για files |
| **HERCULES: Hierarchical Recursive Clustering** | arXiv:2506.19992, 2025 | LLM γεννά τίτλους για κάθε επίπεδο ιεραρχίας — sub-folders αυτόματα |
| **LLMs Enable Few-Shot Clustering** | arXiv:2307.00524, TACL 2024 | Cold start solution: LLM pseudo-constraints από τα πρώτα αρχεία |
| **TnT-LLM: Text Mining at Scale** | arXiv:2403.12173, KDD 2024 | Microsoft: zero-shot taxonomy generation + batch prompting 5× φθηνότερο |
| **PRISM: LLM-Guided Semantic Clustering** | arXiv:2604.03180, 2026 | Fine-tune sentence encoder στο δικό σου corpus για καλύτερα clusters |
| **ClusterLLM** | arXiv:2305.14871, EMNLP 2023 | LLM απαντά triplet questions → βελτιώνει clustering χωρίς labels |
| **Online Density-Based Clustering** | arXiv:2601.20680, 2026 | DBSTREAM vs DenStream σε sentence embeddings: DenStream 2× καλύτερο |
| **Concept Drift in Text Stream Mining** | arXiv:2312.02901, ACM TIST 2024 | Πώς fading factor χειρίζεται "παλιά εξάμηνα" που ξεθωριάζουν |
| **Unsupervised Document Clustering** | arXiv:2506.12116, 2025 | 8 encoders × 4 algorithms σε 5 corpora — fused vision+language wins |
| **Embeddings vs Prompting for Classification** | arXiv:2504.04277, 2025 | Embedding-based classification κτυπά prompting στα περισσότερα tasks |

### Version & Duplicate Detection

| Paper | Venue | Σχέση με εμάς |
|---|---|---|
| **RETSim: Resilient and Efficient Text Similarity** | arXiv:2311.17264, ICLR 2024 | Google. 256-dim embeddings. Threshold >0.86 = version, <0.1 = different. `pip install unisim` |
| **Detecting Near-Duplicates for Web Crawling** | Google Research | SimHash 64-bit: Hamming ≤3 bits = near-identical |
| **Revisiting Code Similarity with AST Edit Distance** | arXiv:2404.08817, ACL 2024 | TSED metric: Python >0.23 = functionally equivalent |
| **Transductive Near-Duplicate Image Detection** | arXiv:2410.19437, 2024 | Scanned pages ως images + perceptual hashing |
| **Practical Near-Duplicate Detection** | yorko.github.io, 2023 | MinHash Jaccard 0.8 threshold: 83.3% precision / 92.5% recall |

### Human-Computer Interaction & Trust

| Paper | Venue | Σχέση με εμάς |
|---|---|---|
| **Do People Appropriately Rely on AI?** | CHI 2026 | Over-reliance αυξάνεται με batch size → max 50 items per review session |
| **Trust in Transparency** | arXiv:2510.04968 | Explaining reasoning αυξάνει αποδοχή — δείχνε "γιατί" |
| **Framework for Human-Machine Classification** | arXiv:2601.05974 | Bridge nodes πρέπει να βλέπει ο χρήστης πρώτα |
| **Structural-Clustering Active Learning for GNNs** | arXiv:2312.04307, 2023 | PageRank + betweenness για human review prioritization |
| **Keeping Found Things Found** | W. Jones, book | HCI: люди οργανώνουν κατά project/event, όχι τύπο — αυτό εξηγεί γιατί type-based fails |

### Incremental & Streaming

| Paper | Venue | Σχέση με εμάς |
|---|---|---|
| **BERTopic** | arXiv + GitHub | merge_models() για weekly updates (partial_fit έχει bugs) |
| **River: ML for Streaming Data** | arXiv:2012.04740, JMLR 2021 | Η βιβλιοθήκη που περιέχει DenStream |
| **TextClust** | Springer ACIIDS 2022 | TF-IDF stream clustering (μόνο για baseline σύγκριση) |
| **SCISSOR: Semantic Bias** | arXiv:2506.14587 | Semantic gravity — μεγάλα clusters τραβάνε αδίκαια |

---
