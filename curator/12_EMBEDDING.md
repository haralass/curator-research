## ΜΕΡΟΣ 12: EMBEDDING STRATEGY — ΤΙ ΒΑΖΟΥΜΕ ΣΑΝ INPUT
*(Έρευνα 2026-05-22 — 56 sources)*

### 🏆 Model Recommendation: BGE-M3 κερδίζει ξεκάθαρα

| | multilingual-e5-large | **BGE-M3** |
|---|---|---|
| Context | 512 tokens | **8192 tokens** |
| Greek | ✅ | ✅ |
| Prefix required | "query: " ΥΠΟΧΡΕΩΤΙΚΟ | ❌ Κανένα prefix |
| Ollama | community | **official** |
| Chunking needed | ΝΑΙ για >512 | **ΟΧΙ για >90% αρχείων** |

BGE-M3 εξαλείφει την πολυπλοκότητα chunking για τα περισσότερα αρχεία. `ollama pull bge-m3`.

---

### Q1: Τι περιέχει το embedding input;

**Finding (arXiv:2404.05825):** Title prepending = +7-19% recall improvement.
- Chunk only: 0.4476 recall → Title only: 0.5310 → **Title+Content: 0.6244**
- Optimal weights: title×0.5, content×0.1 (weighted sum)

**Κανόνας:** Πρόσθεσε πάντα το filename stem πριν το content.

**Αλλά:** Μόνο αν το filename είναι informative. Αν είναι `IMG_4523.jpg` → embed content only.

#### Filename Informativeness Classifier (3 tiers)

```python
import re, math
from collections import Counter

UNINFORMATIVE_PATTERNS = re.compile(
    r'^(img|dsc|dcim|vid|mov|movi|photo|picture|scan|document|file|'
    r'untitled|copy|new.folder|screenshot|whatsapp.image|'
    r'vlc|snapshot)\d*$', re.IGNORECASE
)
STOPWORDS = {'the','a','an','of','in','for','to','and','or','is','it','my','me'}

def filename_informativeness(stem: str) -> float:
    """Returns 0.0 (uninformative) to 1.0 (highly informative)"""
    clean = stem.lower().strip()
    if UNINFORMATIVE_PATTERNS.match(clean):
        return 0.0
    # Word count heuristic
    words = [w for w in re.split(r'[_\-\s\.]+', clean)
             if w and not w.isdigit() and w not in STOPWORDS and len(w) > 1]
    if len(words) <= 1:
        return 0.0
    if len(words) == 2:
        return 0.3
    return 0.7  # 3+ meaningful words = informative
```

#### Τελικό embedding input ανά τύπο

```python
def build_embedding_input(filename_stem, extension, content) -> str:
    info = filename_informativeness(filename_stem)
    if info > 0.5:
        return f"{filename_stem}\n\n{content}"
    else:
        return content  # uninformative filename adds noise
```

---

### Q2: Truncation για long documents

#### BGE-M3 (8192 tokens) — η απλή περίπτωση

```python
# Αν content < 8192 tokens → embed ολόκληρο (κανένα chunking)
# Αν content > 8192 tokens → Late Chunking (arXiv:2409.04701, Jina AI 2024)
```

**Late Chunking** (state-of-the-art για long-context models):
- Παραδοσιακά: split → embed κάθε chunk → χάνεται το cross-chunk context
- Late: encode ολόκληρο το doc → chunk τα token embeddings ΜΕΤΑ → mean pool
- Δουλεύει χωρίς retraining, directly σε BGE-M3

#### multilingual-e5-large (512 tokens) — αν το χρησιμοποιήσεις

**arXiv:2403.12799 (2024):** Head truncation beats all alternatives:
- Head (first 512 tokens): F1=0.7780 ← καλύτερο
- Head+Tail (256+256): F1=0.7759
- Middle: F1=0.7424
- Tail: F1=0.7370 ← χειρότερο

Καλύτερη λύση: overlapping chunks → mean pool:
```python
chunks = [text[i:i+400] for i in range(0, len(tokens), 350)]  # 50 tok overlap
embeddings = [model.encode(f"query: {stem}\n\n{chunk}") for chunk in chunks]
final = np.mean(embeddings, axis=0)
```

**⚠️ Ποτέ** μην κάνεις LLM summarization πριν το embed — arXiv:2403.15112 (2024): clustering με original texts > clustering με LLM summaries (small models χάνουν details).

---

### Q3: Code files

**Εύρημα:** Για topic clustering (όχι clone detection), generic text models εξίσου καλά με CodeBERT. AST/GraphCodeBERT εξέλλει σε clone detection, όχι σε topical grouping.

**Τι εξάγεις από Python/code files:**
```python
def extract_code_content(path):
    source = open(path).read()
    # 1. Import statements (domain signal: import pandas = data analysis)
    imports = '\n'.join(re.findall(r'^(?:import|from)\s+\S+', source, re.M))
    # 2. All docstrings + comments
    docstrings = '\n'.join(re.findall(r'"""[\s\S]*?"""|#.*$', source, re.M))
    # 3. First 150 lines
    first150 = '\n'.join(source.split('\n')[:150])
    return f"{imports}\n\n{docstrings}\n\n{first150}"
```

Χρησιμοποίησε το ίδιο model (BGE-M3). ΟΧΙ CodeBERT — mean pool CLS μόνο αν αναγκαστείς.

---

### Q4: Images

**CLIP vs LLM caption:**
- CLIP: καλό για visual similarity clustering (sunsets μαζί) — αλλά modality gap με text
- **LLM caption → text embed:** αρχεία cluster μαζί με σχετικά documents ανεξάρτητα από format

**Για file organizer (topical clustering):** Χρησιμοποίησε LLaVA caption → embed με BGE-M3.

```python
caption = ollama.generate(model='llava:7b', prompt='Describe this image briefly...', images=[path])
embedding_input = f"{filename_stem}\n\n{caption}"
```

Αποτέλεσμα: whiteboard photo με equations cluster με thesis PDFs ✓

---

### Q5: Feature Fusion — Early vs Late

**Early fusion (prepend) > Late fusion (separate embeddings):**
- Concatenate 1024+1024=2048 dims → χειροτερεύει HDBSCAN curse of dimensionality
- Weighted sum (same model) = valid: `0.4*normalize(embed(filename)) + 0.6*normalize(embed(content))`
- **Πιο απλό και εξίσου καλό:** prepend filename στο content, embed once

**ΜΗΝ** συνδυάζεις embeddings από διαφορετικά models (π.χ. CLIP + E5) με weighted sum.

---

### Τελικό Pipeline ανά file type

| Τύπος | Εξαγωγή | Embedding Input |
|---|---|---|
| PDF | PyPDF2, 10 σελίδες | `"{stem}\n\n{text}"` (αν informative stem) |
| DOCX/XLSX | MarkItDown | `"{stem}\n\n{text}"` |
| Python/Code | imports + docstrings + first 150 lines | `"{stem}\n\n{code_content}"` |
| Image | LLaVA caption | `"{stem}\n\n{caption}"` |
| ZIP | Filenames inside + README | `"{stem}\n\nContents: {filelist}"` |
| Markdown/TXT | Full content | `"{stem}\n\n{content}"` |

### Critical "DO NOT" List (validated by research)

| ❌ | Γιατί |
|---|---|
| "query:" prefix με BGE-M3 | Trained without it — degrades performance |
| Omit "query:" με multilingual-e5 | Mandatory per model card |
| LLM summarization πριν embedding | 2024 paper: original > summary για clustering |
| Remove stopwords πριν embedding | BERTopic: hurt clustering quality |
| CLS token μόνο (BERT) | Mean pool all tokens |
| Concatenate από different models | Different manifolds — mathematically invalid |

---
