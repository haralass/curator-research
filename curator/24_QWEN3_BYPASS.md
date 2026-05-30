## ΜΕΡΟΣ 24: QWEN3 BYPASS + TOPONYMY vs PYHERCULES
*(Έρευνα 2026-05-22 — 20 searches, 18 sources)*

---

### Α. Qwen3-Embedding — Χρήση χωρίς Ollama

#### Α1. Υποστήριξη από sentence-transformers — Επιβεβαιωμένη

Ναι, το Qwen3-Embedding-0.6B υποστηρίζεται πλήρως από τη βιβλιοθήκη `sentence-transformers`. Η ενσωμάτωση ολοκληρώθηκε επίσημα στις **6 Ιουνίου 2025** μέσω του Pull Request #2 στο HuggingFace model repo, που πρόσθεσε:
- Αυτόματη προσθήκη EOS token μέσω tokenizer
- Αρχεία sentence-transformers configuration
- Built-in prompts (π.χ. `prompt_name="query"`)

**Απαιτήσεις:**
```
pip install sentence-transformers>=2.7.0
pip install transformers>=4.51.0
# torch εγκαθίσταται αυτόματα ως dependency
```

Δεν χρειάζεται Ollama. Τρέχει ως καθαρό Python, κατεβάζει το model (~639MB) από HuggingFace Hub αυτόματα.

#### Α2. Ο Κρίσιμος Κίνδυνος: NaN Embeddings στο macOS — Ανοιχτό Bug

**ΚΡΙΤΙΚΟ ΕΎΡΗΜΑ:** Υπάρχει confirmed, ακόμα ανοιχτό bug (sentence-transformers Issue #3498, ανοίχτηκε 25 Αυγούστου 2025) που αφορά ακριβώς το Curator:

- **Ποιο πρόβλημα:** Το Qwen3-Embedding-0.6B παράγει `NaN` embeddings στο macOS λόγω SDPA (Scaled Dot-Product Attention)
- **Ποιες συσκευές:** Και CPU **και** MPS — το πρόβλημα είναι στο macOS γενικά, όχι μόνο Metal
- **Άλλα OS:** Windows και Linux δουλεύουν κανονικά
- **Περιβάλλον που αναφέρθηκε:** MacBook Pro M2, macOS Sequoia 15.6.1, transformers 4.55.3, torch 2.7.1, sentence-transformers 5.1.0
- **Κατάσταση:** **Ανοιχτό** — δεν έχει κλείσει, δεν έχει fix

**Workaround που δουλεύει:**
```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer(
    "Qwen/Qwen3-Embedding-0.6B",
    model_kwargs={
        "attn_implementation": "eager",  # ΑΠΑΡΑΙΤΗΤΟ στο macOS
    },
    tokenizer_kwargs={"padding_side": "left"},
)
```

Με `attn_implementation="eager"` το μοντέλο δουλεύει κανονικά. Το "eager" mode απενεργοποιεί τη βελτιστοποιημένη SDPA υλοποίηση και χρησιμοποιεί τον κλασικό attention υπολογισμό. Αποτέλεσμα: λίγο πιο αργό, αλλά σωστά αποτελέσματα.

#### Α3. Πλήρης Κώδικας για τον Curator (macOS-safe)

```python
from sentence_transformers import SentenceTransformer
import numpy as np

# Φόρτωση μοντέλου — κατεβάζει ~639MB από HuggingFace την πρώτη φορά
model = SentenceTransformer(
    "Qwen/Qwen3-Embedding-0.6B",
    model_kwargs={
        "attn_implementation": "eager",  # ΚΡΙΣΙΜΟ για macOS/MPS/M-series
    },
    tokenizer_kwargs={"padding_side": "left"},
)

# Για clustering (η χρήση μας): δεν χρειάζεται instruction prefix στα documents
# Το instruction prefix χρησιμοποιείται μόνο για asymmetric retrieval (query→document)
# Για symmetric clustering: και queries και documents χωρίς prefix

# Batch embedding αρχείων (για 5000 αρχεία)
file_texts = [
    "σημειώσεις.pdf — Σημειώσεις μαθήματος EPL326, Κεφάλαιο 3, Δίκτυα",
    "Εργασία_EPL326.docx — Assignment 1, Ανάλυση πρωτοκόλλου TCP/IP",
    # ... άλλα αρχεία
]

# Χωρίς prompt_name για clustering (symmetric task)
embeddings = model.encode(
    file_texts,
    batch_size=32,
    show_progress_bar=True,
    normalize_embeddings=True,  # L2 normalization — built-in αλλά ρητά για ασφάλεια
    convert_to_numpy=True,
)
# embeddings.shape: (N, 1024) — 1024 dimensions, normalized

# Επαλήθευση: δεν υπάρχουν NaN
assert not np.isnan(embeddings).any(), "NaN detected! Έλεγξε το attn_implementation"
```

**Σημαντικές λεπτομέρειες:**
- Output dimensions: **1024** (default) — ίδιο με BGE-M3
- Τα embeddings είναι αυτόματα normalized (L2) μέσω της αρχιτεκτονικής (Normalize layer)
- Supports MRL (Matryoshka Representation Learning): μπορείς να ζητήσεις 32-1024 dims
- Για clustering: χρησιμοποίησε **χωρίς** `prompt_name` — το prefix `"query:"` είναι μόνο για asymmetric retrieval

#### Α4. Apple Silicon (MPS) — Ταχύτητα και Μνήμη

Η καλύτερη τεκμηριωμένη επίδοση βρέθηκε μέσω MLX (όχι sentence-transformers/PyTorch):

| Υλοποίηση | Hardware | Throughput |
|---|---|---|
| MLX server (jakedahn/qwen3-embeddings-mlx) | MacBook Pro M2 Max 32GB | **44,000 tokens/sec** |
| Ollama (NVIDIA RTX 4060) | GPU | ~99ms/embedding ≈ 10 emb/sec |
| TEI (NVIDIA GPU) | GPU | ~20ms/embedding ≈ 50 emb/sec |

**Μνήμη (Qwen3-Embedding-0.6B):**
- MLX/GGUF: ~900MB RAM
- sentence-transformers/PyTorch FP32: ~1.2-1.5GB RAM
- sentence-transformers/PyTorch FP16 (MPS): ~700MB RAM

**Εκτίμηση για τον Curator (5000 αρχεία, sentence-transformers):**
- Κάθε file text: ~200-500 tokens (filename + extracted content snippet)
- Batch size 32: ~3-5 sec ανά batch με MPS eager mode
- Σύνολο 5000 αρχεία: ~25-45 λεπτά με CPU/MPS eager (χωρίς MLX)
- Σύνολο με MLX server: ~5-10 λεπτά
- BGE-M3 μέσω Ollama: παρόμοιο (Ollama overhead ~99ms/call)

**Σύγκριση ταχύτητας sentence-transformers:**
- BGE-M3 (567M, XLM-RoBERTa encoder): ~30-50 tokens/ms σε MPS
- Qwen3-0.6B (600M, LLM decoder, eager): ~15-25 tokens/ms σε MPS
- Qwen3 είναι **ελαφρά πιο αργό** λόγω decoder architecture + eager mode overhead

#### Α5. Εναλλακτικό Μονοπάτι: fastembed-qwen3 (ONNX)

Υπάρχει community fork του fastembed με υποστήριξη Qwen3:
```bash
pip install fastembed-qwen3   # ή: pip install qwen3-embed
```

- Χρησιμοποιεί ONNX Runtime (γρηγορότερο από PyTorch για inference)
- Ξεφεύγει από το NaN/SDPA πρόβλημα (ONNX δεν χρησιμοποιεί SDPA)
- **Caveat:** Community fork, όχι official qdrant/fastembed — δεν υπάρχει ακόμα επίσημη υποστήριξη (fastembed Issue #528 ανοιχτό)
- Χρήσιμο ως backup αν το sentence-transformers+eager mode παρουσιάσει προβλήματα

#### Α6. Κατάσταση Ollama Bugs — Αναθεώρηση

| Issue | Κατάσταση (Μάιος 2026) | Αλλαγή από ΜΕΡΟΣ 17 |
|---|---|---|
| #12368 (HTTP 500, 0.6B) | Κλειστό **χωρίς fix** | Χωρίς αλλαγή |
| #12633 (0.6B δεν δουλεύει v0.12.5) | Κλειστό χωρίς fix | Χωρίς αλλαγή |
| #12757 (8B "does not support embeddings") | Κλειστό χωρίς fix | Χωρίς αλλαγή |
| #12611 (8B all-zeros) | Δεν αναφέρεται στο #16076 | Άγνωστο |
| #12921 (8B-fp16 NaN) | Δεν αναφέρεται στο #16076 | Άγνωστο |
| #16076 (feature request, consolidation) | **Ανοιχτό**, ανοίχτηκε 10 Μαΐου 2026 | Νέο — ΔΕΝ υπήρχε στο ΜΕΡΟΣ 17 |
| #10989 (original support request, ~70 upvotes) | **Ανοιχτό** | Χωρίς αλλαγή |

**Βασικό εύρημα:** Η root cause είναι αρχιτεκτονική — το Ollama δεν μπορεί να εξάγει `pooling_type` metadata από Safetensors format. Δεν υπάρχει Modelfile escape hatch. Ένα MLX PR (#14884, Μάρτιος 2026) merged για quantized embeddings αλλά **δεν λύνει** το `/api/embed` endpoint. Κανένα maintainer δεν έχει δεσμευτεί για fix.

**Συμπέρασμα:** Τα Ollama bugs για Qwen3-Embedding παραμένουν **αναλλοίωτα** — ούτε ένα δεν έχει επιλυθεί.

---

### Β. toponymy vs pyhercules — Head-to-Head

#### Β1. toponymy — Βαθιά Ανάλυση

**Δημιουργοί:** Tutte Institute for Mathematics and Computing — οι ίδιοι άνθρωποι που έφτιαξαν UMAP και HDBSCAN (Leland McInnes et al.)
**GitHub:** https://github.com/TutteInstitute/toponymy
**Stars:** 97 | **Τελευταία ενημέρωση:** 21 Μαΐου 2026 (v0.5.1) | **Ηλικία:** ~1.5 χρόνια

**Τι κάνει:**
Το toponymy είναι topic naming library. Παίρνει document embeddings + UMAP projection και τους δίνει ονόματα σε πολλαπλά επίπεδα αδρότητας (coarse → fine). Ο τίτλος προέρχεται από τα ελληνικά: *τόπος + ὄνομα* = "ονομασία τόπων στο χώρο πληροφορίας".

**Τι χρειάζεται ως input (fit() method):**
```python
topic_model.fit(
    text,              # List[str] — τα κείμενα των αρχείων
    document_vectors,  # np.array (N, embedding_dim) — π.χ. (5000, 1024) από BGE-M3
    document_map,      # np.array (N, 2 ή 10) — UMAP projection
)
```

Σημαντικό: το toponymy κάνει **δική του clustering** εσωτερικά — δεν δέχεται pre-computed HDBSCAN labels ως primary input. Χρησιμοποιεί `ToponymyClusterer` (δικό του multi-resolution clusterer).

**Τι βγάζει:**
```python
# topic_names_: λίστα από λίστες — ένα επίπεδο ανά layer
topic_names_ = [
    ["University", "Work", "Personal"],           # Layer 0: coarse (3 clusters)
    ["EPL326", "EPL232", "Thesis", "Photos", ...],# Layer 1: medium (N clusters)
    ["Assignments", "Notes", "Exams", ...],       # Layer 2: fine
]

# topics_per_document: για κάθε document, ποιο topic name έχει σε κάθε layer
for cluster_layer in topic_model.cluster_layers_:
    per_doc_names = cluster_layer.topic_name_vector  # np.array (N,) με strings
```

**LLM Backends:**

| Wrapper | Τύπος | Κλειδί |
|---|---|---|
| `OpenAINamer` | Cloud | OpenAI API key |
| `CohereNamer` | Cloud | Cohere API key |
| `AnthropicNamer` | Cloud | Anthropic API key |
| `HuggingFaceNamer` | Local | HuggingFace model |
| `LlamaCppNamer` | Local | GGUF file |
| **`OllamaNamer`** | **Local** | **Ollama server** |

**OllamaNamer — Exact Code:**
```python
from toponymy.llm_wrappers import OllamaNamer

llm = OllamaNamer(
    model="llama3.2",           # οποιοδήποτε model στο Ollama
    host="http://localhost:11434",  # default Ollama host
)
llm.test_llm_connectivity()  # επαληθεύει ότι ο server τρέχει

# Εναλλακτικά: llama3.2:3b, qwen3:4b, gemma3:4b κλπ.
```

**Multi-level hierarchy:**
Το toponymy βγάζει αυτόματα **πολλαπλά επίπεδα** — από coarse (3-5 clusters) μέχρι fine-grained. Ο χρήστης ελέγχει το `min_clusters` στον `ToponymyClusterer`. Δεν χρειάζεται να ορίσεις το βάθος manually — το σύστημα βρίσκει τη φυσική ιεραρχία.

**pip install:**
```bash
pip install toponymy
# Dependencies: sentence-transformers, umap-learn, pandas
# Optional: datamapplot (visualization)
```

---

#### Β2. pyhercules — Βαθιά Ανάλυση

**Δημιουργός:** bandeerun (ανεξάρτητος researcher)
**GitHub:** https://github.com/bandeerun/pyhercules
**Paper:** arXiv:2506.19992 — "HERCULES: Hierarchical Embedding-based Recursive Clustering Using LLMs for Efficient Summarization"
**Stars:** 22 | **Τελευταία ενημέρωση:** 17 Μαΐου 2026 (v1.1.1) | **Open issues:** 0

**Τι κάνει:**
Το pyhercules υλοποιεί τον αλγόριθμο HERCULES: recursive k-means → ιεραρχικό δέντρο clusters, με LLM να γεννά `title` + `description` για κάθε cluster σε κάθε επίπεδο. Τα child summaries ενσωματώνονται στο parent-level prompt για context-aware naming.

**Τι χρειάζεται ως input:**
```python
from pyhercules import Hercules
from pyhercules_functions import local_minilm_l6_v2_embedding, local_gemma_3_4b_it_llm

hercules = Hercules(
    level_cluster_counts=[4, 5, 3],  # 4 top-level → 5 sub → 3 sub-sub
    representation_mode="direct",    # "direct" ή "description"
    text_embedding_client=local_minilm_l6_v2_embedding,  # embedding function
    llm_client=local_gemma_3_4b_it_llm,                  # LLM function
    verbose=1,
)
top_clusters = hercules.cluster(sample_texts, topic_seed="personal files")
```

Δέχεται: raw text, numeric (NumPy/Pandas), ή images. **ΔΕΝ** δέχεται pre-computed HDBSCAN labels — κάνει δικό του clustering (k-means).

**Τι βγάζει:**
```python
# Ιεραρχικό δέντρο από Cluster objects:
hercules.print_hierarchy()
# → University (4 files)
#    ├── EPL326 (12 files): "Computer Networks course materials"
#    │    ├── Assignments (5 files): "Homework and lab submissions"
#    │    └── Notes (7 files): "Lecture notes and summaries"
#    └── ...

# Tabular export:
df = hercules.get_cluster_membership_dataframe()
# JSON save:
hercules.save_model("clusters.json")
```

**LLM Backends:**
Το pyhercules δέχεται οποιαδήποτε Python function ως `llm_client`. Ενσωματωμένες:
- `local_gemma_3_4b_it_llm` (HuggingFace local)
- `local_minilm_l6_v2_embedding` (local embedding)
- Google Gemini (cloud) — μέσω `.env` API key

**Ollama integration — custom function:**
```python
import ollama

def ollama_llm_client(prompt: str) -> str:
    """Custom wrapper για Ollama — περνά ως llm_client στο Hercules"""
    response = ollama.chat(
        model="llama3.2:3b",
        messages=[{"role": "user", "content": prompt}]
    )
    return response["message"]["content"]

def ollama_embedding_client(texts: list[str]) -> list[list[float]]:
    """Custom wrapper για Ollama embeddings — εναλλακτικά BGE-M3"""
    embeddings = []
    for text in texts:
        resp = ollama.embeddings(model="bge-m3", prompt=text)
        embeddings.append(resp["embedding"])
    return embeddings

hercules = Hercules(
    level_cluster_counts=[4, 5, 3],
    representation_mode="direct",
    text_embedding_client=ollama_embedding_client,
    llm_client=ollama_llm_client,
    topic_seed="personal file organization",
)
```

**Multi-level hierarchy:**
Ο χρήστης ορίζει **ρητά** το `level_cluster_counts`: π.χ. `[4, 5, 3]` = 4 top-level clusters, 5 sub-clusters ανά top, 3 sub-sub ανά sub. Ο αριθμός k-means clusters **δεν εξάγεται αυτόματα** — πρέπει να τον ορίσεις. Αυτό είναι μειονέκτημα.

**pip install:**
```bash
pip install pyhercules           # core only
pip install "pyhercules[app]"    # + Dash web UI
```

---

#### Β3. Head-to-Head Comparison Table

| Κριτήριο | toponymy | pyhercules | Νικητής |
|---|---|---|---|
| **Ollama support** | ✅ `OllamaNamer` — built-in | ⚠️ Custom function wrapper | toponymy |
| **Multi-level hierarchy** | ✅ Αυτόματο (data-driven depth) | ⚠️ Manual `level_cluster_counts` | toponymy |
| **Input format** | embeddings + UMAP map + texts | Raw texts (κάνει δικό του embed) | Εξαρτάται |
| **Αποδέχεται pre-computed HDBSCAN** | ❌ Κάνει δικό του clustering | ❌ K-means μόνο | Ισοπαλία (και οι δύο χρειάζονται adaptation) |
| **Output format** | List[List[str]] per layer | Cluster tree (objects) | toponymy (απλούστερο) |
| **Greek language support** | ✅ Εξαρτάται από LLM — αν llama3.2 ή qwen3 → ναι | ✅ Εξαρτάται από LLM | Ισοπαλία |
| **Maintenance (stars)** | 97 ⭐ | 22 ⭐ | toponymy |
| **Last commit** | 21 Μαΐου 2026 (v0.5.1) | 17 Μαΐου 2026 (v1.1.1) | Ισοπαλία |
| **Open issues** | Δεν γνωρίζουμε | 0 | pyhercules (φαινομενικά) |
| **Δημιουργοί** | UMAP+HDBSCAN creators | Ανεξάρτητος researcher | toponymy |
| **Ease of integration (LOC)** | ~10 γραμμές | ~15-20 γραμμές | toponymy |
| **Configuration flexibility** | Λιγότερη (auto depth) | Περισσότερη (manual k) | pyhercules |
| **LLM calls per cluster** | ~1 call per layer per cluster | 1 batched call per cluster | Ισοπαλία |
| **Production readiness** | ✅ Mature (1.5 χρόνια, 97 stars) | ⚠️ Research prototype (22 stars) | toponymy |
| **Paper backing** | Implicit (UMAP/HDBSCAN papers) | ✅ Explicit — arXiv:2506.19992 | pyhercules |

---

#### Β4. Integration με το Pipeline του Curator

**Pipeline μας:** BGE-M3 embeddings (1024-dim) → UMAP 10-dim → HDBSCAN labels → [naming step] → folder names

##### toponymy — Integration Code

```python
import numpy as np
from sentence_transformers import SentenceTransformer
from toponymy import Toponymy, ToponymyClusterer
from toponymy.llm_wrappers import OllamaNamer

# Βήμα 1: Έχουμε ήδη τα embeddings και το UMAP map από το pipeline μας
# document_vectors: np.array (N, 1024) — BGE-M3 embeddings
# document_map: np.array (N, 10) — UMAP 10-dim projection

# Βήμα 2: Embedding model για topicality scoring (toponymy το χρειάζεται)
embedding_model = SentenceTransformer("BAAI/bge-m3")

# Βήμα 3: LLM μέσω Ollama
llm = OllamaNamer(model="llama3.2:3b", host="http://localhost:11434")

# Βήμα 4: Clusterer (min_clusters ελέγχει το coarsest level)
clusterer = ToponymyClusterer(min_clusters=3, verbose=False)

# Βήμα 5: Topic model
topic_model = Toponymy(
    llm_wrapper=llm,
    text_embedding_model=embedding_model,
    clusterer=clusterer,
    object_description="personal files",
    corpus_description="~/Downloads folder",
)

# Βήμα 6: Fit — δίνουμε texts + ήδη-υπολογισμένα vectors
topic_model.fit(file_texts, document_vectors, document_map)

# Βήμα 7: Εξαγωγή ονομάτων
folder_names_per_level = topic_model.topic_names_
# → [["University", "Work", "Personal"], ["EPL326", "Photos", ...], ...]

# Βήμα 8: Per-document assignments
for layer in topic_model.cluster_layers_:
    print(layer.topic_name_vector)  # np.array (N,) με folder name για κάθε αρχείο
```

**Σημείωση για το HDBSCAN:** Το toponymy κάνει δική του clustering εσωτερικά, αγνοώντας τα HDBSCAN labels μας. Αυτό **δεν είναι πρόβλημα** — χρησιμοποιούμε τα HDBSCAN labels για την αρχική ομαδοποίηση (για DenStream, GLOSH scores κλπ.) και δίνουμε στο toponymy το UMAP map για να ονοματίσει τις ομάδες που βλέπει στον χώρο.

##### pyhercules — Integration Code (με Ollama wrapper)

```python
import ollama
from pyhercules import Hercules

# Custom Ollama wrappers
def ollama_embed(texts: list[str]) -> list[list[float]]:
    return [ollama.embeddings(model="bge-m3", prompt=t)["embedding"] for t in texts]

def ollama_llm(prompt: str) -> str:
    return ollama.chat(
        model="llama3.2:3b",
        messages=[{"role": "user", "content": prompt}]
    )["message"]["content"]

# Hercules — αποφασίζεις εσύ πόσα clusters ανά επίπεδο
hercules = Hercules(
    level_cluster_counts=[4, 5, 3],  # 4 top → 5 sub → 3 sub-sub
    representation_mode="direct",
    text_embedding_client=ollama_embed,
    llm_client=ollama_llm,
    topic_seed="personal file organization macOS",
    verbose=1,
)

# Δίνει raw texts — κάνει δικό του embedding και clustering
top_clusters = hercules.cluster(file_texts)
hercules.print_hierarchy()
```

**Σημαντικό πρόβλημα με pyhercules:** Το `level_cluster_counts=[4, 5, 3]` προϋποθέτει ότι **ξέρεις εκ των προτέρων** πόσες κατηγορίες υπάρχουν. Για dynamic discovery (zero predefined categories), αυτό είναι αντίφαση — αναγκάζεσαι να μαντέψεις. Το toponymy αποφεύγει αυτό γιατί βρίσκει αυτόματα τον αριθμό clusters.

---

#### Β5. Greek File Names — Δοκιμή Γλωσσικής Υποστήριξης

**Το κύριο ερώτημα:** Αν έχουμε αρχεία "σημειώσεις.pdf", "Εργασία_EPL326.docx" — θα βγουν ελληνικά folder names;

**Αρχή λειτουργίας:** Και τα δύο εργαλεία περνούν τα **filenames και extracted text** στο LLM. Το LLM αποφασίζει τη γλώσσα των ονομάτων.

| Ερώτημα | toponymy | pyhercules |
|---|---|---|
| Επεξεργάζεται ελληνικό κείμενο; | ✅ (εξαρτάται από embedding model) | ✅ (εξαρτάται από embedding model) |
| Βγάζει ελληνικά folder names; | Εξαρτάται **αποκλειστικά** από το LLM | Εξαρτάται **αποκλειστικά** από το LLM |
| Με llama3.2:3b + Greek input; | Πιθανόν αγγλικά (αδύναμη Greek training) | Πιθανόν αγγλικά |
| Με qwen3:4b + Greek input; | **Πιθανώς ελληνικά** (119 γλώσσες) | **Πιθανώς ελληνικά** |

**Πρακτική σύσταση:** Προσθέστε explicit γλωσσική οδηγία στο LLM:

```python
# Για toponymy:
llm = OllamaNamer(
    model="qwen3:4b",
    host="http://localhost:11434",
    # Η οδηγία γλώσσας περνά μέσω corpus_description:
)
topic_model = Toponymy(
    llm_wrapper=llm,
    object_description="αρχεία του υπολογιστή (personal files)",
    corpus_description="φάκελος ~/Downloads — ονόμασε τα clusters στα ΕΛΛΗΝΙΚΑ",
)

# Για pyhercules:
hercules = Hercules(
    topic_seed="οργάνωση προσωπικών αρχείων macOS — folder names στα ελληνικά",
    ...
)
```

**Verdict γλώσσας:** Χρησιμοποίησε **qwen3:4b ή qwen3:8b** ως LLM backend (όχι llama3.2) για αξιόπιστα ελληνικά folder names. Η επιλογή naming library (toponymy ή pyhercules) δεν επηρεάζει τη γλώσσα — το LLM αποφασίζει.

---

#### Β6. Κρίσιμο Ζήτημα: Pre-computed HDBSCAN Labels

Το pipeline μας τρέχει HDBSCAN και βγάζει labels. Ούτε toponymy ούτε pyhercules τα δέχονται ως primary input. **Λύση:**

**Για toponymy:** Δώσε το UMAP `document_map` (2-dim ή 10-dim). Το toponymy βλέπει την ίδια δομή που είδε το HDBSCAN — θα βγάλει παρόμοια clustering. Η εσωτερική clustering του toponymy (multi-resolution) είναι **καλύτερη** για naming γιατί βλέπει πολλαπλά επίπεδα αδρότητας ταυτόχρονα.

**Για pyhercules:** Δεν γνωρίζει το UMAP projection — κάνει embedding από μηδέν. Αυτό σημαίνει **double embedding** (BGE-M3 για HDBSCAN + ξανά για pyhercules). Περιττό overhead.

---

### Γ. Σύνοψη Αποφάσεων

#### 🏆 Verdict Qwen3: Χρησιμοποίησε ΤΩΡΑ μέσω sentence-transformers (με caveat)

**Σύσταση: Qwen3-Embedding-0.6B μέσω sentence-transformers + `attn_implementation="eager"` — αν και μόνο αν αποδεχτείς το NaN workaround.**

**Ιεραρχία επιλογών:**

| Επιλογή | Ποιότητα | Αξιοπιστία (macOS) | Πολυπλοκότητα |
|---|---|---|---|
| **BGE-M3 μέσω Ollama** | 40.88 MMTEB clustering | ✅ Εξαιρετική | Χαμηλή (status quo) |
| **Qwen3-0.6B μέσω sentence-transformers + eager** | **52.33 MMTEB clustering** | ⚠️ Workaround απαιτείται | Μέτρια |
| **Qwen3-0.6B μέσω Ollama** | 52.33 | ❌ Αναξιόπιστο (bugs) | Άχρηστο |
| **Qwen3-0.6B μέσω MLX server** | 52.33 | ✅ Εξαιρετική | Υψηλή (extra server) |
| **Qwen3-0.6B μέσω fastembed-qwen3** | ~52.33 | ✅ Χωρίς SDPA bug | Μέτρια (community fork) |

**Γιατί αξίζει το +11.45 MMTEB clustering:**
- Το MMTEB clustering gap (+11.45) είναι η πιο σχετική μετρική για τον Curator
- Καλύτερα embeddings → καλύτερα clusters → καλύτερα folder names
- Το workaround (`attn_implementation="eager"`) είναι **μόνο μία γραμμή κώδικα**
- Το NaN bug (sentence-transformers #3498) είναι γνωστό και το fix είναι αποδεδειγμένο

**Γιατί το pipeline σπάει ελαφρά τη uniformity:**
- Αντί για "όλα μέσω Ollama", το embedding τρέχει μέσω sentence-transformers/PyTorch
- Το LLM (για naming) εξακολουθεί να τρέχει μέσω Ollama
- Αυτό είναι αποδεκτό — το embedding είναι batch step (ανά session), όχι real-time

**Απόφαση:**
- Αν ο Curator χρειάζεται **μέγιστη ποιότητα** τώρα: **Qwen3-0.6B via sentence-transformers + eager**
- Αν προτιμάς **μηδενικό κίνδυνο**: **Μείνε στο BGE-M3 via Ollama** (status quo από ΜΕΡΟΣ 17)
- Όταν Issue #16076 κλείσει με fix: switch σε Qwen3 via Ollama αμέσως

---

#### 🏆 Verdict Naming: toponymy — με σαφή λόγο

**Σύσταση: ΧΡΗΣΙΜΟΠΟΙΗΣΕ toponymy.**

**Λόγοι:**

1. **OllamaNamer built-in** — μηδέν custom code για Ollama integration. `OllamaNamer(model="qwen3:4b")` και τελείωσε.

2. **Αυτόματο multi-level depth** — δεν χρειάζεται να ορίσεις `level_cluster_counts`. Για dynamic discovery (zero predefined categories), αυτό είναι κρίσιμο — το σύστημα βρίσκει μόνο του πόσα επίπεδα χρειάζονται.

3. **Δημιουργοί UMAP+HDBSCAN** — οι ίδιοι που έφτιαξαν τα εργαλεία που χρησιμοποιούμε. Σχεδιάστηκε ακριβώς για αυτό το pipeline (embeddings → UMAP → naming).

4. **Δέχεται το UMAP document_map** — ταιριάζει τέλεια με το pipeline: δίνουμε τα ήδη-υπολογισμένα UMAP vectors, δεν κάνει double embedding.

5. **Mature library** — 97 stars, v0.5.1, ενεργό (τελευταίο commit 1 μέρα πριν σήμερα).

6. **Output format απλός** — `List[List[str]]` per layer = απευθείας folder name hierarchy.

**Γιατί ΟΧΙ pyhercules:**

1. **Κανένας built-in Ollama wrapper** — χρειάζεσαι custom function, extra boilerplate.
2. **Manual level_cluster_counts** — αντιφάσκει με το dynamic discovery goal. Πρέπει να μαντέψεις πόσα clusters.
3. **Double embedding** — κάνει δικό του embedding από μηδέν, αγνοώντας τα BGE-M3 embeddings που ήδη έχουμε.
4. **22 stars** — research prototype, όχι production library.
5. **Αρχιτεκτονικό mismatch** — σχεδιάστηκε για raw text input, όχι για pipeline που ήδη έχει UMAP projection.

---

**Sources Μέρος 24:**
- sentence-transformers Issue #3498 (NaN/SDPA macOS) — https://github.com/UKPLab/sentence-transformers/issues/3498
- Qwen3-Embedding-0.6B HuggingFace discussions/2 (ST integration PR) — https://huggingface.co/Qwen/Qwen3-Embedding-0.6B/discussions/2
- Qwen3-Embedding-0.6B HuggingFace model page — https://huggingface.co/Qwen/Qwen3-Embedding-0.6B
- QwenLM/Qwen3-Embedding GitHub — https://github.com/QwenLM/Qwen3-Embedding
- toponymy GitHub — https://github.com/TutteInstitute/toponymy
- toponymy LLM Wrappers docs — https://toponymy.readthedocs.io/en/latest/llm_wrappers.html
- pyhercules GitHub — https://github.com/bandeerun/pyhercules
- HERCULES paper arXiv:2506.19992 — https://arxiv.org/html/2506.19992
- Ollama Issue #16076 (first-class embedding support) — https://github.com/ollama/ollama/issues/16076
- Ollama Issue #12368 (HTTP 500 pooling_type) — https://github.com/ollama/ollama/issues/12368
- fastembed Issue #528 (Qwen3 support request) — https://github.com/qdrant/fastembed/issues/528
- fastembed-qwen3 PyPI fork — https://libraries.io/pypi/fastembed-qwen3
- jakedahn/qwen3-embeddings-mlx (44K tokens/sec on M2 Max) — https://github.com/jakedahn/qwen3-embeddings-mlx
- Medium: Generating embeddings using Qwen3 0.6B — https://medium.com/@smrati.katiyar/generating-embeddings-using-qwen3-0-6b-model-7a4a826a5c6d
- HERCULES paper (themoonlight review) — https://www.themoonlight.io/en/review/hercules-hierarchical-embedding-based-recursive-clustering-using-llms-for-efficient-summarization
- Qwen3-Embedding-0.6B model details (aimodels.fyi) — https://www.aimodels.fyi/models/huggingFace/qwen3-embedding-0.6b-qwen

---
