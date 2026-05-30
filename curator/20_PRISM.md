## ΜΕΡΟΣ 20: PRISM — FINE-TUNING SENTENCE ENCODER
*(Έρευνα 2026-05-22 — 10 searches, 15+ sources)*

### Τι είναι το PRISM (2604.03180)

**Πλήρης τίτλος:** "PRISM: LLM-Guided Semantic Clustering for High-Precision Topics"
**Venue:** ACM Web Conference 2026 (WWW '26), April 13–17, Dubai — published
**arXiv:** https://arxiv.org/abs/2604.03180
**Authors:** Connor Douglas (NYU), Utkucan Balci (Binghamton), Joseph Aylett-Bullock (UN)
**GitHub:** Δεν βρέθηκε public repo

---

### Α. Τι Κάνει Ακριβώς το PRISM

**Ορισμός:** PRISM (Precision-Informed Semantic Modeling) = structured topic modeling framework που συνδυάζει:
- **LLM εκφρασιμότητα** (rich representations) για labeling
- **Sentence encoder efficiency** (local, fast, cheap) για clustering

**Pipeline σε 4 βήματα:**
1. Sample N documents από corpus
2. LLM παράγει binary comparison labels (FR) ή embedding similarity labels (RB) για pairs
3. Fine-tune sentence encoder με αυτά τα labels + **CoSENT loss**
4. Cluster τα fine-tuned embeddings με thresholded clustering (ΟΧΙ HDBSCAN — proximity threshold)

**Βασικό claim:** PRISM κάνει ένα *μικρό, local* sentence encoder (all-mpnet-base-v2, 420MB) να αποδίδει καλύτερα από large frontier embedding models για clustering — γιατί fine-tunes *για το συγκεκριμένο corpus*.

---

### Β. Τι Ακριβώς Fine-Tuneται

**Model:** `all-mpnet-base-v2` (SentenceTransformers) — όχι BGE-M3
**Method:** **Full fine-tuning** (ΟΧΙ LoRA, ΟΧΙ projection head only)
**Loss function:** **CoSENT** (Consistent Sentence Embedding) — από TASLP 2024 paper
- CoSENT εξέλαβε contrastive loss: 50-67% λιγότερο training time, ίδια ή καλύτερη ποιότητα
- Υποστηρίζει binary similarity pairs (αντί για triplets) → πιο εύκολη data collection

**Training Data Formats:**
- **FR (Forced Ranking):** LLM αποφαίνεται "which document is more similar to anchor?" → binary pairs
  - 1,000 labeled samples, κόστος ~$0.20 per dataset
- **RB (Relevance-Based):** LLM δίνει embedding similarity score για pairs
  - 500 embeddings → 249,500 training triples (combinatorial expansion)
  - Κόστος ~$0.05 per dataset

---

### Γ. Αριθμοί — Πόσα Δεδομένα Χρειάζεται

| Μέθοδος | LLM Queries | Training Examples | Κόστος LLM |
|---|---|---|---|
| FR (binary ranking) | 1,000 | 1,000 pairs | ~$0.20 |
| RB (embedding similarity) | 500 | 249,500 triples (combinatorial) | ~$0.05 |
| **Καλύτερο: Emb+RB** | 500 | 249,500 triples | ~$0.05 |

**Κρίσιμη λεπτομέρεια:** Τα 249,500 training examples προέρχονται από **combinatorial expansion** των 500 LLM queries — δεν χρειάζεσαι 249,500 LLM calls! Η ευελιξία αυτή κάνει το σύστημα very cheap.

**Για Curator (personal file organizer):**
- 500 LLM queries × ~200 tokens/query = 100K tokens
- Με Ollama local (llama3.2:3b ή qwen2.5:7b) = $0 κόστος, ~10-15 λεπτά inference time
- Αποτέλεσμα: 249,500 training triples

---

### Δ. Performance Numbers

Από WebFetch του PRISM HTML paper (arxiv.org/html/2604.03180v1):

| Corpus | Metric | PRISM (Emb+RB) | Top2Vec | A∘L_Emb (large model) |
|---|---|---|---|---|
| HumAID (28,802 docs) | Cluster Purity | **0.807** | 0.717 | 0.786 |
| LIAR (10,240 docs) | Cluster Purity | **0.638** | 0.636 | 0.639 |
| IMDB (25,000 docs) | Cluster Purity | **0.966** | 0.913 | 0.957 |

**`A∘L_Emb`** = direct clustering on large frontier embeddings (χωρίς fine-tuning) — αυτό ισοδυναμεί με τη δική μας approach (BGE-M3 → HDBSCAN χωρίς fine-tuning).

**Key insight:** PRISM κερδίζει έναντι large model clustering (+2.1% HumAID, +0.9% IMDB) — αλλά η βελτίωση είναι **μικρή και corpus-dependent**. Στο LIAR dataset το PRISM σχεδόν δεν βελτιώνει (+0.0% ουσιαστικά).

**Ablation study (από paper):**
- "Untuned PRISM yields, at best, marginal gains over Top2Vec" — επιβεβαιώνει ότι το fine-tuning είναι κρίσιμο
- RB-only > FR-only > untuned για όλα τα datasets
- Emb+RB (combined) = καλύτερο

---

### Ε. Training Time — Apple Silicon (MPS)

**Άμεσα benchmarks από paper:** Δεν αναφέρεται training time στο paper.

**Από εξωτερικές πηγές:**
- sentence-transformers υποστηρίζει MPS (Apple Silicon) από v2.2+
- `TrainingArguments(fp16=False)` — **ΥΠΟΧΡΕΩΤΙΚΟ** σε MPS (δεν υποστηρίζει FP16)
- `PYTORCH_ENABLE_MPS_FALLBACK=1` — χρειάζεται αν κάποιες ops δεν υπάρχουν σε MPS

**Εκτίμηση training time για PRISM-style fine-tuning (249,500 pairs, all-mpnet-base-v2, M2 Mac):**
- CPU baseline: ~60-90 λεπτά
- MPS (Metal GPU acceleration): ~15-30 λεπτά (2-4× speedup vs CPU)
- Benchmark reference: SetFit με 8 examples/class σε V100 = 30 seconds → M2 ≈ 2-5 λεπτά για SetFit scale

**Σημείωση:** 249,500 training triples είναι αρκετά μεγάλο dataset για sentence transformer fine-tuning. Για personal file organizer with ~500-5000 files → ίσως χρησιμοποιήσουμε λιγότερα (100 LLM queries → ~9,900 triples) → training ~3-8 λεπτά σε M1/M2.

---

### ΣΤ. LoRA vs Full Fine-Tuning για Embeddings

Από sentence-transformers PEFT documentation + research:

| Method | Quality vs Full FT | Memory | Time | Για Curator |
|---|---|---|---|---|
| Full fine-tuning | 100% (baseline) | High (full model in RAM) | Baseline | PRISM χρησιμοποιεί αυτό |
| LoRA (r=16) | **94-98%** της ποιότητας | **10-20× λιγότερο** | ~50% λιγότερο | Καλή επιλογή |
| Linear probe (frozen + linear layer) | 85-90% | Ελάχιστο | **Πολύ γρήγορο** | Υπερ-απλοποίηση |

**LoRA για sentence-transformers:**
```python
from peft import LoraConfig, get_peft_model, TaskType

# Υποστήριξη μέσω sentence-transformers PEFT integration:
# sbert.net/examples/sentence_transformer/training/peft/README.html
config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["query", "value"],  # attention layers
    lora_dropout=0.1,
    task_type=TaskType.FEATURE_EXTRACTION
)
```

**LoRA στο BGE-M3 specifically:** Δεν υπάρχει published recipe. Αλλά BGE-M3 είναι XLM-RoBERTa βάση → τα standard LoRA configs για RoBERTa εφαρμόζονται.

**Inference latency:** LoRA adapter μπορεί να **merged** με base model (zero inference overhead). `model.merge_and_unload()` → single model file.

---

### Ζ. Lighter-Weight Alternatives

#### SetFit (Hugging Face)
- **Paper:** arXiv:2209.11055 (2022, EMNLP)
- **GitHub:** https://github.com/huggingface/setfit (5,000+ stars)
- **Claim:** 8 examples per class → outperforms GPT-3 few-shot
- **Mechanism:** Siamese contrastive fine-tuning (in-class pairs positive, cross-class negative) → classification head
- **Training time (V100):** 30 seconds με 8 examples/class
- **Limitation:** SetFit είναι classification-focused (fixed classes), ΟΧΙ clustering-focused
- **Για Curator:** Χρήσιμο αν θέλουμε να train classifier από user confirmations (όχι για initial clustering)

#### Projection Layer on Frozen Embeddings
- Από LlamaIndex blog (Jerry Liu): Linear adapter fine-tuning για any embedding model
- Frozen BGE-M3 + trainable linear layer 1024→1024
- Performance: ~85-90% της quality of full fine-tuning
- Training time: <1 λεπτό (πολύ λιγότερες parameters)
- **Πλεονέκτημα:** Βασικό embedding model (BGE-M3) παραμένει αναλλοίωτο

```python
import torch
import torch.nn as nn

class EmbeddingProjection(nn.Module):
    """Lightweight trainable projection on top of frozen BGE-M3."""
    def __init__(self, embed_dim: int = 1024, hidden_dim: int = 512):
        super().__init__()
        self.projection = nn.Sequential(
            nn.Linear(embed_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, embed_dim),
            nn.LayerNorm(embed_dim)
        )
    
    def forward(self, embeddings: torch.Tensor) -> torch.Tensor:
        return self.projection(embeddings)

# Training: freeze BGE-M3, train only projection
# Loss: CoSENT on user confirmation pairs
# Training data: user approves cluster → positive pair
#               user rejects cluster → negative pair
```

---

### Η. Is PRISM Worth It for a Personal File Organizer?

#### Ανάλυση του Use Case μας

| Factor | PRISM Paper | Curator |
|---|---|---|
| Corpus size | 10K-28K documents | 500-5,000 files |
| Domain | Fixed, narrow (news, reviews) | Mixed personal files (CS, personal, downloads) |
| Fine-tuning data | ~500 LLM queries | User confirmations (accumulated over time) |
| Re-training frequency | Once per corpus | After each ~50 user confirmations |
| Base model | all-mpnet-base-v2 (English) | BGE-M3 (multilingual) |
| Performance gain | +2.1% cluster purity (HumAID) | Unknown for personal files |

**Pros για Curator:**
1. Fine-tuning με user confirmations = **χωρίς extra LLM queries** — ο χρήστης ήδη approves clusters
2. Personal files έχουν consistent themes (CS courses, thesis) → domain-specific fine-tuning μπορεί να βοηθήσει
3. CoSENT loss = γρήγορο, simple implementation
4. Ο PRISM βελτιώνει clustering separability — ακριβώς αυτό που χρειαζόμαστε

**Cons για Curator:**
1. **Corpus size:** PRISM tested on 10K-28K docs. Εμείς έχουμε 500-5000 files — πιθανόν **too small για statistically significant fine-tuning**
2. **Frequency:** Re-training μετά από κάθε 50 confirmations = συχνά. Αν training = 15-30 min ανά run → αποδεκτό αλλά επιβαρυντικό
3. **Complexity:** Full PRISM pipeline (LLM labeling + fine-tuning + thresholded clustering) προσθέτει σημαντική complexity
4. **BGE-M3 already strong:** Multilingual, 8192 context — ήδη πολύ καλό baseline
5. **Performance gain uncertain:** +2.1% σε large corpus δεν σημαίνει ίδια βελτίωση σε 500 personal files
6. **Domain shift:** Εμείς έχουμε ελληνικά + αγγλικά mixed → fine-tuning πρέπει να έχει αντιπροσωπευτικά pairs και από τις δύο γλώσσες

---

### Θ. Ablation Study — Πόσο Βοηθά το Fine-Tuning σε Μικρά Corpora

Από STF paper (arXiv:2407.03253, 2024) — sentence transformer fine-tuning για topic categorization με limited data:
- Fine-tuning βοηθά ακόμα και με μικρά datasets
- Outperforms state-of-the-art χωρίς "huge amount of labeled data"
- **Αλλά:** Tested on tweets (very different από personal files)

Από contrastive fine-tuning paper (arXiv:2408.11868, 2024):
- Frozen embedding + linear classifier: **better than fine-tuning** για very small datasets
- Fine-tuning overhead: run time 30-50% αυξάνεται
- **Sweet spot:** >500 training pairs → fine-tuning ξεπερνά frozen approach

**Συμπέρασμα από literature:** Για <500 user confirmations, frozen BGE-M3 + projection layer είναι καλύτερο. Για >500 confirmations, full fine-tuning (ή LoRA) αξίζει.

---

### Ι. Concrete Implementation Paths (σε σειρά πολυπλοκότητας)

**Path A — Zero Fine-Tuning (αυτό που κάνουμε ήδη):**
```
BGE-M3 → UMAP → HDBSCAN → LLM naming
```
Κόστος: 0. Quality: baseline (MMTEB clustering 40.88).

**Path B — Projection Layer (minimal complexity):**
```python
# Φάση 1 (startup): BGE-M3 embed → frozen embeddings
# Φάση 2 (background, μετά 100+ user confirmations):
#   Train EmbeddingProjection on (approved_pair, rejected_pair) tuples
#   Training time: <2 λεπτά σε M1/M2
#   Apply projection BEFORE UMAP → better cluster separation
```

**Path C — LoRA Fine-tuning (moderate complexity):**
```python
from sentence_transformers import SentenceTransformer
from peft import LoraConfig, get_peft_model

model = SentenceTransformer("BAAI/bge-m3")
# Add LoRA adapter (βλ. sbert.net PEFT docs)
# Fine-tune on user confirmations as positive/negative pairs
# Merge adapter after training: model.merge_and_unload()
# Re-embed all files → re-run UMAP+HDBSCAN
```

**Path D — Full PRISM (maximum complexity):**
```python
# 1. Sample 100-200 files from corpus
# 2. Local LLM (qwen2.5:7b via Ollama) generates RB labels for pairs
# 3. CoSENT loss fine-tuning of BGE-M3 (ή all-mpnet-base-v2)
# 4. Re-embed corpus → thresholded clustering
# Εκτιμώμενος χρόνος: 10-15 min LLM labeling + 15-30 min training
```

---

### 🏆 Verdict: SKIP Full PRISM — Implement Path B (Projection Layer)

**Recommendation: Αναβάλλουμε PRISM, υλοποιούμε lightweight projection layer.**

**Αιτιολόγηση:**

1. **Corpus size mismatch:** PRISM δοκιμάστηκε σε 10K-28K documents. Εμείς έχουμε 500-5000 files. Με <500 confirmed pairs → fine-tuning δεν αξίζει (research: frozen > fine-tune για very small datasets).

2. **BGE-M3 είναι ήδη πολύ δυνατό baseline:** Clustering MMTEB 40.88 — αρκετό για personal files. Η βελτίωση PRISM (+2.1% purity σε large corpus) ίσως μηδενίζεται σε 500-file personal corpus.

3. **Complexity cost:** Full PRISM προσθέτει: LLM labeling pipeline + CoSENT training + thresholded clustering → τρεις νέες dependencies, πολλαπλά failure points.

4. **Training data organic accumulation:** Μετά από 50-100 user confirmations, έχουμε φυσικά positive/negative pairs. Η projection layer (Path B) αξιοποιεί αυτά με minimal complexity.

5. **Η διαφορά στη γλώσσα:** Το PRISM χρησιμοποιεί `all-mpnet-base-v2` (English-only). Εμείς χρειαζόμαστε BGE-M3 (Greek+English). Δεν υπάρχει published recipe για PRISM με BGE-M3.

**Πότε να επανεξετάσεις:**
- Αν corpus φτάσει 5000+ files με 500+ user confirmations → δοκίμασε Path C (LoRA)
- Αν clustering quality είναι αποδεκτά κακή μετά από 3 μήνες χρήσης → Path D (full PRISM)
- Αν PRISM βγάλει public GitHub repo με BGE-M3 support → άμεση αξιολόγηση

**Sprint allocation:**
- **Sprint 1-3:** Path A (no fine-tuning) — βάλε στη θέση τους τα basics
- **Sprint 4:** Path B (projection layer) — minimal complexity, collects training data organically
- **Sprint 5+:** Path C/D — μόνο αν εμπειρικά τεκμηριωθεί βελτίωση

---

**Sources Topic 20:**
- PRISM arXiv:2604.03180 — https://arxiv.org/abs/2604.03180
- PRISM HTML full paper — https://arxiv.org/html/2604.03180v1
- PRISM ACM WWW 2026 — https://dl.acm.org/doi/10.1145/3774904.3792944
- SetFit HuggingFace blog — https://huggingface.co/blog/setfit
- SetFit paper arXiv:2209.11055 — https://arxiv.org/pdf/2209.11055
- SetFit GitHub — https://github.com/huggingface/setfit
- sentence-transformers PEFT docs — https://www.sbert.net/examples/sentence_transformer/training/peft/README.html
- CoSENT (IEEE/ACM TASLP 2024) — https://dl.acm.org/doi/10.1109/TASLP.2024.3402087
- STF paper arXiv:2407.03253 — https://arxiv.org/abs/2407.03253
- Contrastive fine-tuning small datasets arXiv:2408.11868 — https://arxiv.org/html/2408.11868v1
- LoRA sentence transformers (PEFT docs) — https://www.sbert.net/examples/sentence_transformer/training/peft/README.html
- sentence-transformers MPS issue #1564 — https://github.com/UKPLab/sentence-transformers/issues/1564
- sentence-transformers MPS HuggingFace docs — https://huggingface.co/docs/transformers/en/perf_train_special
- Linear adapter fine-tuning LlamaIndex — https://medium.com/llamaindex-blog/fine-tuning-a-linear-adapter-for-any-embedding-model-8dd0a142d383
- Improving embeddings contrastive fine-tuning arXiv:2408.00690 — https://arxiv.org/html/2408.00690

---
