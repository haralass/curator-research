## ΜΕΡΟΣ 26: ML RESEARCH — FLACON CODE, GENERATIVE CLUSTERING, ENSEMBLE, AUDIO, BERTOPIC
*(Έρευνα 2026-05-22 — 32 searches, 12 fetches, 40+ sources)*

---

### Α. FLACON — Πλήρης Python Implementation

#### Τι είναι και γιατί το χρειαζόμαστε

Το FLACON (Flag-Aware Context-sensitive Clustering) δημοσιεύτηκε στο MDPI Entropy, Vol.27, Issue 11, Oct 2025. **Δεν υπάρχει δημόσιο GitHub repo.** Ο σκοπός αυτής της ενότητας είναι να γράψουμε μια πλήρη υλοποίηση από μηδέν, συνδυάζοντας:

1. Την τυπική composite distance από το paper
2. ARISE encoding για τα categorical flags (arXiv:2601.01162, ICLR 2025, GitHub: develop-yang/ARISE)
3. Gower distance για mixed metadata (`pip install gower`)
4. HDBSCAN με `metric='precomputed'` για να δεχτεί τον custom distance matrix

#### Τι βρήκαμε για το ARISE

**ARISE** = "Attention-weighted Representation with Integrated Semantic Embeddings". Γεφυρώνει το semantic gap σε categorical data clustering χρησιμοποιώντας LLMs.

- Queries LLM σε επίπεδο attribute-value (ένα prompt ανά τιμή, π.χ. "pdf", "py", "jpg")
- Attention-weighted encoding χωρίς learnable parameters
- Adaptive fusion module που ισορροπεί εξωτερική σημασία vs dataset-specific patterns
- **+19-27% improvement** σε 8 benchmark datasets vs 7 baseline methods
- GitHub: https://github.com/develop-yang/ARISE (clone + `pip install -r requirements.txt`)

**Βασική χρήση ARISE:**
```python
# Από το repo develop-yang/ARISE:
arise = ARISE()
cluster_labels, best_alpha = arise.fit_predict(
    features=features,
    feature_names=feature_names,
    descriptions=descriptions,  # LLM-generated per value
    n_clusters=7
)
```

Για το Curator **δεν χρειαζόμαστε το ARISE module** ολόκληρο — μόνο το ιδέα του: κάθε categorical τιμή (π.χ. "pdf") αποκτά ένα embedding μέσω LLM description. Αυτό το κάνουμε directly με το Ollama/BGE-M3 pipeline που ήδη έχουμε.

#### Ο Composite Distance Formula (FLACON)

```
d(doc_i, doc_j) = α × semantic_distance(i, j)
                + β × flag_distance(i, j)
                + γ × temporal_factor(i, j)
```

Με `α + β + γ = 1.0`. Τυπικές αρχικές τιμές (paper recommendation):
- `α = 0.4` — semantic content κυριαρχεί
- `β = 0.4` — flags έχουν ίδιο βάρος
- `γ = 0.2` — temporal factor πιο αδύναμο

Αυτές οι τιμές μπορούν να βελτιστοποιηθούν empirically (grid search ή Bayesian optimization με DBCV ως objective).

#### Πώς να εξάγουμε τα 6 flags από macOS αρχεία

**Βιβλιοθήκη:** `pip install osxmetadata` (GitHub: RhetTbull/osxmetadata)

```python
from osxmetadata import OSXMetaData
import subprocess, re
from pathlib import Path
from datetime import datetime, timezone

def extract_flags(filepath: str) -> dict:
    """
    Εξάγει 6-dimensional flags από macOS αρχείο.
    Επιστρέφει dict με τιμές ανά flag.
    """
    path = Path(filepath)
    md = OSXMetaData(filepath)
    now = datetime.now(timezone.utc)
    
    # --- FLAG 1: Type ---
    # Extension → canonical type string
    TYPE_MAP = {
        # Documents
        '.pdf': 'pdf_document', '.docx': 'word_document',
        '.doc': 'word_document', '.pptx': 'presentation',
        '.xlsx': 'spreadsheet', '.xls': 'spreadsheet',
        '.txt': 'text_file', '.md': 'markdown',
        # Code
        '.py': 'python_script', '.js': 'javascript',
        '.ts': 'typescript', '.rs': 'rust_code',
        '.c': 'c_code', '.cpp': 'cpp_code',
        '.java': 'java_code', '.sh': 'shell_script',
        # Media
        '.jpg': 'jpeg_image', '.jpeg': 'jpeg_image',
        '.png': 'png_image', '.gif': 'gif_image',
        '.mp3': 'audio_mp3', '.wav': 'audio_wav',
        '.m4a': 'audio_m4a', '.aiff': 'audio_aiff',
        '.mp4': 'video_mp4', '.mov': 'video_quicktime',
        # Archives
        '.zip': 'zip_archive', '.tar': 'tar_archive',
        '.gz': 'gzip_archive', '.7z': 'sevenzip_archive',
        # Apps
        '.dmg': 'disk_image', '.pkg': 'installer_package',
        '.app': 'application',
    }
    suffix = path.suffix.lower()
    file_type = TYPE_MAP.get(suffix, f'unknown_{suffix.lstrip(".")}')
    
    # --- FLAG 2: Domain ---
    # Δεν μπορούμε να ξέρουμε domain χωρίς embeddings.
    # Placeholder: θα οριστεί ΜΕΤΑ το clustering (nearest centroid).
    # Για τώρα βάζουμε None — θα το assign μετά.
    domain = None  # Assigned post-clustering (nearest cluster centroid label)
    
    # --- FLAG 3: Priority ---
    # Βάσει last used date + use count
    priority = 'unknown'
    try:
        last_used = md.lastuseddate  # kMDItemLastUsedDate
        use_count = md.usecount      # kMDItemUseCount (int)
        
        if use_count is not None and last_used is not None:
            # last_used may be naive — make timezone-aware
            if last_used.tzinfo is None:
                last_used = last_used.replace(tzinfo=timezone.utc)
            days_since_used = (now - last_used).days
            
            if use_count > 10 and days_since_used < 30:
                priority = 'high'
            elif use_count > 3 and days_since_used < 90:
                priority = 'medium'
            elif use_count == 0:
                priority = 'never_opened'
            else:
                priority = 'low'
        elif use_count == 0:
            priority = 'never_opened'
    except Exception:
        priority = 'unknown'
    
    # --- FLAG 4: Status ---
    # Βάσει filename patterns
    stem = path.stem.lower()
    status = 'unknown'
    DRAFT_PATTERNS = r'\b(draft|wip|todo|temp|tmp|scratch)\b'
    FINAL_PATTERNS = r'\b(final|done|complete|finished|approved|submitted)\b'
    VERSION_PATTERNS = r'(v\d+|_v\d+|\bversion\d+\b|copy|backup|old|archive)'
    
    if re.search(DRAFT_PATTERNS, stem):
        status = 'draft'
    elif re.search(FINAL_PATTERNS, stem):
        status = 'final'
    elif re.search(VERSION_PATTERNS, stem):
        status = 'version_or_copy'
    else:
        # Infer από recency: νέο αρχείο (< 7 ημέρες) = active
        try:
            created = md.fscreationdate  # kMDItemFSCreationDate
            if created:
                if created.tzinfo is None:
                    created = created.replace(tzinfo=timezone.utc)
                age_days = (now - created).days
                if age_days < 7:
                    status = 'active_recent'
                elif age_days > 365:
                    status = 'archived_old'
                else:
                    status = 'active'
        except Exception:
            status = 'unknown'
    
    # --- FLAG 5: Relationship ---
    # Βάσει kMDItemWhereFroms (source URL domain)
    relationship = 'standalone'
    try:
        wherefroms = md.wherefroms  # kMDItemWhereFroms → list of URLs
        if wherefroms:
            # Extract domain from first URL
            import urllib.parse
            url = wherefroms[0] if isinstance(wherefroms, list) else wherefroms
            parsed = urllib.parse.urlparse(str(url))
            netloc = parsed.netloc.lower()
            
            if 'mail' in netloc or 'gmail' in netloc or 'outlook' in netloc:
                relationship = 'email_attachment'
            elif 'github' in netloc or 'gitlab' in netloc:
                relationship = 'code_repository'
            elif 'drive.google' in netloc or 'dropbox' in netloc or 'onedrive' in netloc:
                relationship = 'cloud_storage'
            elif 'moodle' in netloc or 'ucy.ac.cy' in netloc or 'edu' in netloc:
                relationship = 'academic_platform'
            elif netloc:
                relationship = 'web_download'
    except Exception:
        relationship = 'standalone'
    
    # --- FLAG 6: Temporal ---
    # Ηλικία αρχείου σε ημέρες (continuous)
    temporal_age_days = 0.0
    try:
        created = md.fscreationdate
        if created:
            if created.tzinfo is None:
                created = created.replace(tzinfo=timezone.utc)
            temporal_age_days = float((now - created).days)
    except Exception:
        temporal_age_days = 365.0  # default: assume 1 year old
    
    return {
        'type': file_type,
        'domain': domain,
        'priority': priority,
        'status': status,
        'relationship': relationship,
        'temporal_age_days': temporal_age_days,
    }
```

#### ARISE-style Flag Embeddings (χωρίς το ARISE repo)

Αντί για one-hot encoding, κάθε categorical τιμή αποκτά LLM-generated description → embed:

```python
import ollama
import numpy as np
from functools import lru_cache

# Pre-defined descriptions (ARISE-style: LLM δημιουργεί, εμείς cache-άρουμε)
FLAG_DESCRIPTIONS = {
    # Type flags
    'pdf_document': 'A PDF document containing formatted text, images, or academic/professional content',
    'word_document': 'A Microsoft Word document with editable formatted text',
    'python_script': 'A Python source code file for programming and scripting',
    'jpeg_image': 'A JPEG photograph or screenshot image file',
    'png_image': 'A PNG image file, often with transparent background',
    'audio_mp3': 'An MP3 audio file containing music or speech recordings',
    'audio_m4a': 'An M4A audio file, typically from Apple voice recordings or music',
    'zip_archive': 'A ZIP compressed archive containing multiple files',
    'spreadsheet': 'A spreadsheet file with tables, formulas, and numerical data',
    'presentation': 'A slide presentation file for talks and lectures',
    'video_mp4': 'An MP4 video file containing recorded content',
    'disk_image': 'A macOS disk image file containing software installer',
    # Priority flags
    'high': 'Frequently accessed, recently used important file',
    'medium': 'Occasionally accessed file with moderate importance',
    'low': 'Rarely accessed file, low priority',
    'never_opened': 'File that has never been opened by the user',
    'unknown': 'Usage pattern unknown',
    # Status flags
    'draft': 'Work in progress, incomplete draft document',
    'final': 'Completed, finalized, submitted or approved document',
    'version_or_copy': 'A backup, copy, or older version of another file',
    'active_recent': 'Recently created, actively being worked on',
    'archived_old': 'Old file, likely archived or no longer actively used',
    'active': 'Regular working file',
    # Relationship flags
    'standalone': 'An independent file with no known source or parent',
    'email_attachment': 'File received as email attachment',
    'code_repository': 'File downloaded from a code repository like GitHub',
    'cloud_storage': 'File synced from cloud storage service',
    'academic_platform': 'File downloaded from academic or educational platform',
    'web_download': 'File downloaded from the internet',
}

@lru_cache(maxsize=256)
def get_flag_embedding(flag_value: str, model: str = 'bge-m3') -> np.ndarray:
    """Embed ένα flag value μέσω LLM description (ARISE-style). Cached."""
    description = FLAG_DESCRIPTIONS.get(
        flag_value,
        f'A file attribute value: {flag_value}'  # fallback
    )
    response = ollama.embeddings(model=model, prompt=description)
    return np.array(response['embedding'], dtype=np.float32)


def compute_flag_distance(flags_i: dict, flags_j: dict,
                          embed_model: str = 'bge-m3') -> float:
    """
    Υπολογίζει flag distance μεταξύ δύο αρχείων.
    Categorical flags: cosine distance μεταξύ embeddings (ARISE-style).
    Numerical flags (temporal): normalized absolute difference.
    Επιστρέφει 0.0 (ίδιο) έως 1.0 (εντελώς διαφορετικό).
    """
    CATEGORICAL_FLAGS = ['type', 'priority', 'status', 'relationship']
    # domain = None placeholder, skip
    
    distances = []
    
    for flag in CATEGORICAL_FLAGS:
        val_i = flags_i.get(flag, 'unknown')
        val_j = flags_j.get(flag, 'unknown')
        
        if val_i == val_j:
            distances.append(0.0)
        elif val_i is None or val_j is None:
            distances.append(0.5)  # unknown = medium distance
        else:
            emb_i = get_flag_embedding(val_i, embed_model)
            emb_j = get_flag_embedding(val_j, embed_model)
            # Cosine distance (1 - cosine_similarity)
            cos_sim = np.dot(emb_i, emb_j) / (
                np.linalg.norm(emb_i) * np.linalg.norm(emb_j) + 1e-8
            )
            dist = float(1.0 - cos_sim) / 2.0  # normalize to [0, 1]
            distances.append(dist)
    
    # Temporal: normalized difference (cap at 2 years = 730 days)
    age_i = min(flags_i.get('temporal_age_days', 365.0), 730.0)
    age_j = min(flags_j.get('temporal_age_days', 365.0), 730.0)
    temporal_dist = abs(age_i - age_j) / 730.0  # normalize to [0, 1]
    distances.append(temporal_dist)
    
    # Mean over all 5 flag dimensions
    return float(np.mean(distances))
```

#### Gower Distance για metadata DataFrame

Για bulk computation με pandas DataFrame (χωρίς LLM embedding, απλούστερο fallback):

```python
# pip install gower
import gower
import pandas as pd

def build_metadata_df(file_flags_list: list[dict]) -> pd.DataFrame:
    """Μετατρέπει λίστα flags σε DataFrame για gower."""
    rows = []
    for f in file_flags_list:
        rows.append({
            'type': f['type'],
            'priority': f['priority'],
            'status': f['status'],
            'relationship': f['relationship'],
            'temporal_age_days': f['temporal_age_days'],
        })
    return pd.DataFrame(rows)

def compute_gower_flag_matrix(file_flags_list: list[dict]) -> np.ndarray:
    """
    Υπολογίζει Gower distance matrix μεταξύ όλων των αρχείων.
    Αυτόματα χειρίζεται categorical vs numerical.
    cat_features: True = categorical, False = numerical
    """
    df = build_metadata_df(file_flags_list)
    # type, priority, status, relationship = categorical
    # temporal_age_days = numerical (continuous)
    cat_features = [True, True, True, True, False]
    dist_matrix = gower.gower_matrix(df, cat_features=cat_features)
    return dist_matrix  # shape: (N, N), values in [0, 1]
```

#### Temporal Factor

```python
def compute_temporal_factor(age_i_days: float, age_j_days: float,
                             decay_lambda: float = 0.003) -> float:
    """
    Temporal factor από το FLACON paper:
    Αρχεία παρόμοιας ηλικίας → μικρή απόσταση.
    Αρχεία πολύ διαφορετικής ηλικίας → μεγάλη απόσταση.
    Χρησιμοποιεί exponential decay.
    """
    age_diff_days = abs(age_i_days - age_j_days)
    # Exponential decay: d = 1 - exp(-λ * |age_i - age_j|)
    # decay_lambda=0.003: diff=230 days → distance≈0.5, diff=700 days → distance≈0.88
    return float(1.0 - np.exp(-decay_lambda * age_diff_days))
```

#### Πλήρης FLACON Distance + HDBSCAN Integration

```python
import hdbscan
import numpy as np
from sklearn.metrics.pairwise import cosine_distances

def compute_flacon_distance_matrix(
    embeddings: np.ndarray,          # shape: (N, 1024) — BGE-M3 embeddings
    file_flags_list: list[dict],     # length N, output of extract_flags()
    alpha: float = 0.4,              # semantic weight
    beta: float = 0.4,               # flag weight
    gamma: float = 0.2,              # temporal weight
    use_arise: bool = True,          # ARISE-style LLM embedding vs Gower
    embed_model: str = 'bge-m3',
) -> np.ndarray:
    """
    Υπολογίζει τον NxN composite FLACON distance matrix.
    Κατάλληλο για HDBSCAN(metric='precomputed').
    """
    N = len(embeddings)
    assert N == len(file_flags_list), "Embeddings και flags πρέπει να έχουν ίδιο length"
    assert abs(alpha + beta + gamma - 1.0) < 1e-6, "α + β + γ πρέπει να είναι 1.0"
    
    # 1. Semantic distance (cosine) — πολύ γρήγορο, sklearn vectorized
    print(f"[FLACON] Computing semantic distances ({N}x{N})...")
    # Normalize embeddings πρώτα για cosine distance
    norms = np.linalg.norm(embeddings, axis=1, keepdims=True) + 1e-8
    normed = embeddings / norms
    semantic_dist = cosine_distances(normed)  # shape (N, N), values [0, 2] → normalize
    semantic_dist = np.clip(semantic_dist / 2.0, 0.0, 1.0)  # [0, 1]
    
    # 2. Flag distance
    print(f"[FLACON] Computing flag distances...")
    if use_arise:
        # ARISE-style: per-pair LLM embedding cosine distance
        # Προσοχή: O(N²) LLM calls — cache βοηθάει πολύ
        # Για N=5000: ~12.5M pairs → ΔΕΝ είναι feasible pair-by-pair
        # Λύση: pre-embed όλα τα unique values (< 50 unique values), τότε vectorize
        unique_vals = set()
        for f in file_flags_list:
            for key in ['type', 'priority', 'status', 'relationship']:
                unique_vals.add(f.get(key, 'unknown'))
        
        print(f"[FLACON] Pre-embedding {len(unique_vals)} unique flag values...")
        val_emb_cache = {}
        for val in unique_vals:
            if val:
                val_emb_cache[val] = get_flag_embedding(val, embed_model)
        
        # Build flag embedding matrix per file (concatenate 4 flag embeddings)
        CATEGORICAL_FLAGS = ['type', 'priority', 'status', 'relationship']
        flag_emb_dim = 1024  # BGE-M3 output dim
        flag_matrix = np.zeros((N, len(CATEGORICAL_FLAGS) * flag_emb_dim), dtype=np.float32)
        
        for i, f in enumerate(file_flags_list):
            parts = []
            for key in CATEGORICAL_FLAGS:
                val = f.get(key, 'unknown') or 'unknown'
                emb = val_emb_cache.get(val, np.zeros(flag_emb_dim))
                parts.append(emb)
            flag_matrix[i] = np.concatenate(parts)
        
        # Normalize and compute pairwise cosine distance
        flag_norms = np.linalg.norm(flag_matrix, axis=1, keepdims=True) + 1e-8
        flag_normed = flag_matrix / flag_norms
        flag_dist = cosine_distances(flag_normed)
        flag_dist = np.clip(flag_dist / 2.0, 0.0, 1.0)
    else:
        # Simpler: Gower distance (no LLM calls for flags)
        flag_dist = compute_gower_flag_matrix(file_flags_list)
    
    # 3. Temporal factor — fully vectorized
    print(f"[FLACON] Computing temporal distances...")
    ages = np.array([f.get('temporal_age_days', 365.0) for f in file_flags_list])
    age_diff = np.abs(ages[:, None] - ages[None, :])  # shape (N, N)
    temporal_dist = 1.0 - np.exp(-0.003 * age_diff)   # [0, 1]
    
    # 4. Composite distance
    print(f"[FLACON] Combining distances (α={alpha}, β={beta}, γ={gamma})...")
    composite = alpha * semantic_dist + beta * flag_dist + gamma * temporal_dist
    
    # Ensure symmetry and zero diagonal (numerical precision)
    composite = (composite + composite.T) / 2.0
    np.fill_diagonal(composite, 0.0)
    
    return composite.astype(np.float32)


def run_flacon_clustering(
    embeddings: np.ndarray,
    file_flags_list: list[dict],
    alpha: float = 0.4,
    beta: float = 0.4,
    gamma: float = 0.2,
    min_cluster_size: int = 10,
    min_samples: int = 5,
    use_arise: bool = True,
) -> tuple[np.ndarray, np.ndarray, np.ndarray]:
    """
    Πλήρης FLACON pipeline: embeddings + flags → cluster labels.
    
    Returns:
        labels: cluster labels (-1 = noise)
        outlier_scores: GLOSH scores (για antilibrary)
        soft_memberships: soft cluster memberships
    """
    # Υπολογισμός composite distance matrix
    dist_matrix = compute_flacon_distance_matrix(
        embeddings, file_flags_list, alpha, beta, gamma, use_arise
    )
    
    # HDBSCAN με precomputed matrix
    # ΣΗΜΑΝΤΙΚΟ: metric='precomputed' → input είναι distance matrix, όχι features
    clusterer = hdbscan.HDBSCAN(
        metric='precomputed',
        min_cluster_size=min_cluster_size,
        min_samples=min_samples,
        cluster_selection_method='eom',
        prediction_data=True,
    )
    
    print(f"[FLACON] Running HDBSCAN on {len(embeddings)}x{len(embeddings)} precomputed matrix...")
    clusterer.fit(dist_matrix)
    
    labels = clusterer.labels_
    outlier_scores = clusterer.outlier_scores_
    
    # Soft memberships για noise handling
    try:
        soft = hdbscan.all_points_membership_vectors(clusterer)
    except Exception:
        soft = np.zeros((len(labels), max(labels) + 2))
    
    print(f"[FLACON] Results: {len(set(labels)) - (1 if -1 in labels else 0)} clusters, "
          f"{(labels == -1).sum()} noise points ({100*(labels==-1).mean():.1f}%)")
    
    return labels, outlier_scores, soft
```

#### Πλήρης Pipeline Παράδειγμα

```python
# ΠΛΗΡΕΣ ΠΑΡΑΔΕΙΓΜΑ ΓΙΑ ΤΟ CURATOR
import glob
import ollama
import numpy as np

def batch_embed(texts: list[str], model: str = 'bge-m3',
                batch_size: int = 20) -> np.ndarray:
    """Batch embedding μέσω Ollama."""
    all_embeddings = []
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i+batch_size]
        for text in batch:
            resp = ollama.embeddings(model=model, prompt=text)
            all_embeddings.append(resp['embedding'])
    return np.array(all_embeddings, dtype=np.float32)

# 1. Συλλογή αρχείων
file_paths = glob.glob('/Users/me/Downloads/**/*', recursive=True)
file_paths = [p for p in file_paths if Path(p).is_file()]

# 2. Εξαγωγή content text (MarkItDown, PyMuPDF κλπ — από υπάρχον pipeline)
texts = [extract_content(p) for p in file_paths]  # existing function

# 3. Embeddings
print("Embedding files...")
embeddings = batch_embed(texts, model='bge-m3')

# 4. Εξαγωγή flags
print("Extracting macOS flags...")
flags = [extract_flags(p) for p in file_paths]

# 5. FLACON clustering
labels, glosh_scores, soft = run_flacon_clustering(
    embeddings=embeddings,
    file_flags_list=flags,
    alpha=0.4, beta=0.4, gamma=0.2,
    min_cluster_size=10,
    use_arise=True,  # False για Gower fallback (χωρίς LLM)
)

# 6. Antilibrary detection (GLOSH > 0.7 = genuinely unique)
ANTILIBRARY_THRESHOLD = 0.7
antilibrary_files = [
    file_paths[i] for i, s in enumerate(glosh_scores)
    if s > ANTILIBRARY_THRESHOLD
]
print(f"Antilibrary candidates: {len(antilibrary_files)}")
```

#### Practical Notes για Curator

| Ζήτημα | Λύση |
|---|---|
| N=5000 → N² = 25M entries | float32 matrix = 100 MB RAM — fine για M1 |
| Flag embedding cache miss | Pre-embed όλα τα unique values (~50) πριν τον loop |
| Domain flag = None αρχικά | Assign post-clustering: nearest centroid label |
| `metric='precomputed'` δεν δέχεται UMAP | Σωστό — FLACON δεν χρειάζεται UMAP (η flag info υποκαθιστά) |
| Gower vs ARISE | Gower: πιο γρήγορο, χωρίς LLM calls. ARISE: +20% quality |

**Sources:**
- FLACON paper: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12650872/
- ARISE GitHub: https://github.com/develop-yang/ARISE
- ARISE paper: https://arxiv.org/abs/2601.01162
- Gower PyPI: https://pypi.org/project/gower/
- HDBSCAN precomputed: https://hdbscan.readthedocs.io/en/latest/basic_hdbscan.html
- osxmetadata: https://github.com/RhetTbull/osxmetadata

---

### Β. Generative Clustering (LMGC) — AAAI 2025

#### Τι είναι

**Paper:** "Information-Theoretic Generative Clustering of Documents" (arXiv:2412.13534, AAAI 2025)
**GitHub:** https://github.com/kduxin/lmgc

Αντί να κάνει embed τα documents και μετά cosine similarity, το LMGC:
1. Τρέχει ένα generative LLM (doc2query) για κάθε document
2. Το LLM παράγει πιθανολογικές κατανομές (token probabilities)
3. Η ομοιότητα δύο documents ορίζεται ως **KL divergence** μεταξύ των distributions
4. Clustering μέσω information-theoretic objective

#### Ακριβής Φόρμουλα

Η distance function:
```
d̂(x, k) = (1/J) × Σⱼ (p(yⱼ|x) / φ(yⱼ))^α × log(p(yⱼ|x) / p(yⱼ|k))
```
- `α = 0.25` (regularization για variance reduction)
- `J = 1024-4096` samples (importance sampling)
- `φ` = proposal distribution
- `p(y|x)` = LLM probability distribution για document x
- `p(y|k)` = cluster centroid distribution

#### LLM που χρησιμοποιείται

**Δεν είναι GPT-4.** Χρησιμοποιεί `doc2query` — open-source T5-based model:
- Model: `all-with_prefix-t5-base-v1`
- Trained on query generation, title generation, question answering
- ~250M parameters — μικρό, τρέχει locally

#### Computational Cost

| Dataset | Μέγεθος | Χρόνος |
|---|---|---|
| R2 (Reuters) | ~6,400 docs | **10 λεπτά σε single GPU** |
| Precision matrix `P` | J×N pairs | Βασικό bottleneck |

- **GPU απαραίτητο** — δεν υπάρχει επίσημο CPU benchmark
- Για 5000 αρχεία + J=1024 → εκτιμώμενα **8-12 λεπτά σε M1 GPU (Metal)**
- Χωρίς GPU (CPU only) → πιθανά **1-3 ώρες** — δύσκολα αποδεκτό

#### Installation

```bash
git clone https://github.com/kduxin/lmgc
cd lmgc
# Python 3.10 ΑΠΑΡΑΙΤΗΤΟ
pip install -r requirements.txt
maturin develop --release  # Compile Rust extension (maturin εγκαθίσταται αυτόματα)
bash scripts/prepare.sh    # ~20 min (downloads models)
bash scripts/cluster.sh    # Τρέχει clustering
```

Χρειάζεται Python 3.10 + Rust compiler για το maturin βήμα.

#### Σύγκριση vs BGE-M3 + HDBSCAN

| Κριτήριο | BGE-M3 + HDBSCAN | LMGC |
|---|---|---|
| Speed (5000 files) | ~2 λεπτά | ~10 λεπτά (GPU) / ~2 ώρες (CPU) |
| Setup complexity | Simple pip install | Rust compiler + maturin |
| Local/Offline | ✅ Ollama | ✅ αλλά doc2query download |
| Greek support | ✅ BGE-M3 confirmed | ❓ doc2query = English-only |
| Performance | Καλό | State-of-the-art (96.1% vs 92% SBERT) |
| Noise/outliers | ✅ GLOSH | ❌ Δεν έχει outlier detection |

#### Verdict

**Δεν αξίζει για το Curator τώρα** — τουλάχιστον για 3 λόγους:

1. **doc2query είναι English-only.** Τα ελληνικά αρχεία δεν θα χειριστούν σωστά. Ο BGE-M3 είναι πολυγλωσσικός.

2. **GPU-dependent.** Το LMGC έχει σχεδιαστεί για NVIDIA GPU. Apple Metal support είναι αβέβαιο. Το Curator τρέχει 100% local — δεν μπορούμε να εξαρτηθούμε από GPU.

3. **Complexity overhead.** Rust compiler + maturin + custom scripts σε ένα project που ήδη έχει αρκετή πολυπλοκότητα.

**Πότε να το επανεξετάσεις:**
- Αν κυκλοφορήσει multilingual doc2query model
- Αν χρειαστείς state-of-the-art accuracy σε English-only corpus
- Για future research paper/benchmark comparison

**Sources:**
- Paper: https://arxiv.org/html/2412.13534v1
- GitHub: https://github.com/kduxin/lmgc
- AAAI 2025: https://dl.acm.org/doi/10.1609/aaai.v39i16.33802

---

### Γ. Ensemble Clustering HDBSCAN + Louvain

#### Ιδέα

Ο στόχος: συνδύασε HDBSCAN labels + Louvain labels → consensus labels πιο ανθεκτικά από κάθε αλγόριθμο μόνο του.

**Γιατί ενδιαφέρει:**
- HDBSCAN: εξαιρετικό για density-based clusters, αλλά nondeterministic (UMAP randomness) και sensitive σε parameters
- Louvain: graph-based community detection, unstable (γνωστό πρόβλημα — διαφορετικά runs δίνουν διαφορετικά αποτελέσματα)
- Ensemble: αντισταθμίζει αδυναμίες και των δύο

#### Ευρήματα από τη Βιβλιογραφία

**"Consensus Communities" (Digital Scholarship in the Humanities, 2025):** Louvain + HDBSCAN consensus matrix βρίσκει robust structures που επιβιώνουν σε αλγοριθμικές παραλλαγές. Χτίζει "matrix of agreement": αρχεία που assigned στο ίδιο cluster ΚΑΙ από Louvain ΚΑΙ από HDBSCAN παίρνουν υψηλό score.

**Voting-based consensus (Pattern Recognition, 2009 — classic):** Η κατηγορία consensus methods που αντιμετωπίζει ρητά το label mismatch πρόβλημα.

**CSPA (Cluster-based Similarity Partitioning):**
- Binary similarity matrix ανά clustering → entry-wise average → recluster
- Python: `pip install ClusterEnsembles`

#### Πώς να φτιάξεις το Similarity Graph για Louvain

```python
import networkx as nx
import community as community_louvain  # pip install python-louvain
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

def build_cosine_similarity_graph(
    embeddings_umap: np.ndarray,   # (N, 10) — UMAP-reduced embeddings
    threshold: float = 0.7,         # Edges μόνο για cosine > threshold
) -> nx.Graph:
    """
    Χτίζει weighted graph από embeddings.
    Nodes = αρχεία, Edges = cosine similarity > threshold.
    ΣΗΜΑΝΤΙΚΟ: χωρίς threshold, dense graphs → Louvain αργεί ή crashάρει.
    """
    N = len(embeddings_umap)
    # Normalize για cosine
    norms = np.linalg.norm(embeddings_umap, axis=1, keepdims=True) + 1e-8
    normed = embeddings_umap / norms
    sim_matrix = cosine_similarity(normed)  # (N, N)
    
    G = nx.Graph()
    G.add_nodes_from(range(N))
    
    # Sparse: add edges only where similarity > threshold
    rows, cols = np.where(sim_matrix > threshold)
    for i, j in zip(rows, cols):
        if i < j:  # avoid duplicates
            G.add_edge(i, j, weight=float(sim_matrix[i, j]))
    
    print(f"Graph: {G.number_of_nodes()} nodes, {G.number_of_edges()} edges")
    return G

def run_louvain(G: nx.Graph, resolution: float = 1.0) -> dict:
    """Τρέχει Louvain community detection."""
    # python-louvain (import community)
    partition = community_louvain.best_partition(G, weight='weight',
                                                  resolution=resolution)
    # Επιστρέφει dict: {node_id: community_id}
    return partition
```

#### Consensus μέσω ClusterEnsembles (CSPA)

```python
# pip install ClusterEnsembles
import ClusterEnsembles as CE
import numpy as np

def ensemble_hdbscan_louvain(
    embeddings_umap: np.ndarray,
    hdbscan_labels: np.ndarray,
    louvain_partition: dict,
    threshold: float = 0.7,
) -> np.ndarray:
    """
    Συνδυάζει HDBSCAN + Louvain labels σε consensus.
    
    ΣΗΜΑΝΤΙΚΟ: HDBSCAN noise points (label=-1) → np.nan για ClusterEnsembles
    """
    N = len(hdbscan_labels)
    
    # Μετατρέπουμε Louvain partition dict → array
    louvain_labels = np.array([louvain_partition.get(i, -1) for i in range(N)],
                               dtype=float)
    
    # HDBSCAN: -1 (noise) → np.nan (ClusterEnsembles δέχεται missing)
    hdbscan_clean = hdbscan_labels.astype(float)
    hdbscan_clean[hdbscan_clean == -1] = np.nan
    
    # Louvain: -1 → np.nan (isolated nodes)
    louvain_labels[louvain_labels == -1] = np.nan
    
    # Stack: shape (2, N) — δύο base clusterings
    label_matrix = np.array([hdbscan_clean, louvain_labels])
    
    # Consensus: 'hbgf' recommended για large N
    consensus_labels = CE.cluster_ensembles(label_matrix, solver='hbgf',
                                             nclass=None)  # auto-detect K
    
    return consensus_labels

# Πλήρης workflow:
def full_ensemble_pipeline(embeddings_1024d, flags_list):
    import umap
    import hdbscan
    
    # 1. UMAP
    reducer = umap.UMAP(n_components=10, n_neighbors=15, min_dist=0.0,
                         metric='cosine', random_state=42)
    embeddings_10d = reducer.fit_transform(embeddings_1024d)
    
    # 2. HDBSCAN
    clusterer = hdbscan.HDBSCAN(min_cluster_size=10, min_samples=5,
                                  metric='euclidean', prediction_data=True)
    clusterer.fit(embeddings_10d)
    hdbscan_labels = clusterer.labels_
    
    # 3. Louvain graph
    G = build_cosine_similarity_graph(embeddings_10d, threshold=0.7)
    louvain_partition = run_louvain(G)
    
    # 4. Consensus
    final_labels = ensemble_hdbscan_louvain(
        embeddings_10d, hdbscan_labels, louvain_partition
    )
    
    return final_labels
```

#### Πρόβλημα: Louvain Instability

Ο Louvain είναι γνωστά **unstable**: διαφορετικά runs στα ίδια data δίνουν διαφορετικά αποτελέσματα. Λύση: τρέξε 5-10 φορές και κάνε ensemble ΚΑΙ αυτά:

```python
def stable_louvain(G: nx.Graph, n_runs: int = 5,
                    resolution: float = 1.0) -> np.ndarray:
    """Ensemble πολλαπλών Louvain runs για stability."""
    import numpy as np
    N = G.number_of_nodes()
    all_labels = []
    for _ in range(n_runs):
        partition = community_louvain.best_partition(
            G, weight='weight', resolution=resolution
        )
        labels = np.array([partition.get(i, 0) for i in range(N)], dtype=float)
        all_labels.append(labels)
    
    # Consensus των n_runs Louvain results
    label_matrix = np.array(all_labels)
    return CE.cluster_ensembles(label_matrix, solver='hbgf', nclass=None)
```

#### Verdict

**Αξίζει ή είναι overkill για 5000 αρχεία;**

| Κριτήριο | Αξιολόγηση |
|---|---|
| Quality improvement | +10-20% robustness (από βιβλιογραφία), αλλά **αχρείαστο αν HDBSCAN δουλεύει καλά** |
| Complexity | Σημαντική — 2 algorithms + consensus library |
| Performance | Louvain graph building: ~1-5 sec για 5000 nodes. ClusterEnsembles: ~2-10 sec |
| Noise handling | Χάνει το GLOSH antilibrary signal — **σοβαρή απώλεια** |
| Determinism | Consensus βελτιώνει stability αλλά δεν εξαλείφει nondeterminism |

**Σύσταση: ΠΑΡΛΕΨΕ το για τώρα.** Το HDBSCAN μόνο του, μετά UMAP, είναι state-of-the-art για file clustering. Το ensemble δίνει οριακή βελτίωση που δεν δικαιολογεί την πολυπλοκότητα — ειδικά επειδή χάνουμε τα GLOSH scores που τροφοδοτούν το antilibrary.

**Πότε να το επανεξετάσεις:**
- Αν το HDBSCAN δίνει πολύ inconsistent results μεταξύ runs
- Αν θες να δημοσιεύσεις research paper με ablation study
- Sprint 4 "Advanced" — όχι Sprint 1-3

**Sources:**
- CSPA Python: https://github.com/827916600/ClusterEnsembles
- python-louvain: https://github.com/taynaud/python-louvain
- Consensus Communities (DHQ 2025): https://academic.oup.com/dsh/article/41/1/141/8364793
- ClusterEnsembles PyPI: https://pypi.org/project/ensembleclustering/

---

### Δ. Audio Files — Whisper Pipeline

#### Κατάσταση: Δεν υπάρχει Whisper στο Ollama (officially)

Η αναζήτηση αποκάλυψε ότι **το Ollama δεν έχει official Whisper model.** Υπάρχουν community uploads (π.χ. `karanchopda333/whisper`, `dimavz/whisper-tiny`) αλλά αυτά δεν είναι για transcription μέσω Ollama API — το Ollama είναι text generation, όχι speech-to-text inference framework.

Η σωστή λύση είναι **mlx-whisper** για Apple Silicon.

#### Επιλογές για macOS Apple Silicon

| Υλοποίηση | pip | Apple Silicon | Ταχύτητα | Συνιστάται; |
|---|---|---|---|---|
| `openai-whisper` | `pip install openai-whisper` | CPU only | 1× real-time (slow) | ❌ |
| `faster-whisper` | `pip install faster-whisper` | CPU only (no Metal) | ~3× RT | ❌ για Mac |
| **`mlx-whisper`** | `pip install mlx-whisper` | **Metal GPU ✅** | **~10× RT (tiny), ~5× RT (base)** | **✅** |
| `whisper.cpp` (binary) | brew/cmake | Metal GPU ✅ | ~10× RT | ✅ αλλά C++, όχι Python |

#### Speed Benchmarks (Apple Silicon — mlx-whisper)

| Model | Μέγεθος | Real-Time Factor | 1 λεπτό audio → |
|---|---|---|---|
| **tiny** | 39 MB | ~10× RT | ~6 δευτερόλεπτα |
| **base** | 74 MB | ~5× RT | ~12 δευτερόλεπτα |
| small | 244 MB | ~3× RT | ~20 δευτερόλεπτα |
| medium | 769 MB | ~2× RT | ~30 δευτερόλεπτα |
| large-v3 | 1.5 GB | ~1.5× RT | ~40 δευτερόλεπτα |

Για file organizer: **`base` model** = βέλτιστη ισορροπία ταχύτητα/ακρίβεια για transcription αρχείων.

Πηγές: mlx-whisper benchmarks, Whisper on Apple Silicon blog posts (2025-2026).

Σημαντικό: mlx-whisper είναι **~50% πιο γρήγορο** από openai-whisper στο ίδιο hardware.

#### Πλήρης Audio Pipeline για Curator

```python
# pip install mlx-whisper ffmpeg-python
# Επίσης χρειάζεται: brew install ffmpeg
import mlx_whisper
import subprocess
import json
from pathlib import Path

AUDIO_EXTENSIONS = {'.mp3', '.wav', '.m4a', '.aiff', '.aif', '.ogg',
                    '.flac', '.wma', '.opus'}

def is_audio_file(filepath: str) -> bool:
    return Path(filepath).suffix.lower() in AUDIO_EXTENSIONS

def get_audio_duration_seconds(filepath: str) -> float:
    """Παίρνει duration χωρίς να φορτώσει ολόκληρο το αρχείο."""
    result = subprocess.run(
        ['ffprobe', '-v', 'quiet', '-print_format', 'json',
         '-show_streams', filepath],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        return 0.0
    data = json.loads(result.stdout)
    for stream in data.get('streams', []):
        if stream.get('codec_type') == 'audio':
            return float(stream.get('duration', 0))
    return 0.0

def classify_audio_type(filepath: str) -> str:
    """
    Ταχύς pre-screening: είναι music ή speech;
    Χρησιμοποιεί metadata (bitrate, channel count) ως heuristic.
    Αποφεύγει μάταιο Whisper call σε music files.
    """
    result = subprocess.run(
        ['ffprobe', '-v', 'quiet', '-print_format', 'json',
         '-show_streams', '-show_format', filepath],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        return 'unknown'
    
    data = json.loads(result.stdout)
    fmt = data.get('format', {})
    tags = fmt.get('tags', {})
    
    # Αν έχει 'album', 'artist', 'genre' tags → πιθανώς music
    music_tags = {'album', 'artist', 'genre', 'track', 'albumartist'}
    if any(k.lower() in music_tags for k in tags.keys()):
        return 'likely_music'
    
    # Stereo + high bitrate → πιθανώς music
    for stream in data.get('streams', []):
        if stream.get('codec_type') == 'audio':
            channels = stream.get('channels', 1)
            bit_rate = int(stream.get('bit_rate', 0))
            if channels >= 2 and bit_rate > 128000:
                return 'likely_music'
    
    return 'likely_speech'

def transcribe_audio(filepath: str,
                     model: str = 'mlx-community/whisper-base-mlx',
                     language: str = None) -> str:
    """
    Transcribes audio file using mlx-whisper.
    Returns: text transcript or empty string if not speech.
    
    language=None → auto-detect (supports Greek, English, etc.)
    """
    path = Path(filepath)
    
    # Duration check: αρχεία < 1 δευτερόλεπτο → skip
    duration = get_audio_duration_seconds(filepath)
    if duration < 1.0:
        return ''
    
    # Skip likely music
    audio_type = classify_audio_type(filepath)
    if audio_type == 'likely_music':
        # Δεν κάνουμε transcription — embed ως "music file: {filename}"
        return f'[music file: {path.stem}]'
    
    try:
        result = mlx_whisper.transcribe(
            filepath,
            path_or_hf_repo=model,
            language=language,        # None = auto-detect
            no_speech_threshold=0.6,  # default Whisper threshold
            condition_on_previous_text=False,  # better για isolated clips
        )
        
        text = result.get('text', '').strip()
        
        # Post-check: αν transcript είναι πολύ μικρό relative to duration
        # → πιθανώς music ή non-speech
        if duration > 30 and len(text) < 20:
            return f'[non-speech audio: {path.stem}]'
        
        return text
        
    except Exception as e:
        # Χειρισμός empty sequence ValueError (γνωστό bug faster-whisper/whisper)
        return f'[audio transcription failed: {path.stem}]'


def extract_audio_embedding_input(filepath: str) -> str:
    """
    Παράγει embedding input για audio file.
    Ακολουθεί το pattern: "{filename_stem}\n\n{transcript}"
    """
    path = Path(filepath)
    stem = path.stem
    suffix = path.suffix.lower()
    
    transcript = transcribe_audio(filepath)
    
    if transcript.startswith('['):
        # Music ή failed — embed μόνο filename + type
        return f"{stem} ({suffix} audio file)"
    elif not transcript:
        return f"{stem} (empty audio file)"
    else:
        # Πλήρες: filename + transcript (για BGE-M3)
        return f"{stem}\n\n{transcript}"
```

#### Model για mlx-whisper

Τα mlx-community models στο HuggingFace:
```python
# Επιλογές για mlx-whisper:
'mlx-community/whisper-tiny-mlx'   # 39MB, ~10× RT
'mlx-community/whisper-base-mlx'   # 74MB, ~5× RT  ← Συνιστάται
'mlx-community/whisper-small-mlx'  # 244MB, ~3× RT
'mlx-community/whisper-large-v3-mlx'  # 1.5GB, ~1.5× RT

# Χρήση:
result = mlx_whisper.transcribe(
    audio_path,
    path_or_hf_repo='mlx-community/whisper-base-mlx'
)
```

Τα models κατεβαίνουν αυτόματα την πρώτη φορά από HuggingFace (~74MB για base).

#### Edge Cases

| Περίπτωση | Χειρισμός |
|---|---|
| Music file (mp3 με album tags) | `classify_audio_type()` → 'likely_music' → skip transcription |
| Πολύ μικρό clip (< 1 sec) | Duration check → return '' |
| Non-Greek/Non-English | Whisper auto-detect language (υποστηρίζει 99 γλώσσες) |
| Empty transcript (silence) | no_speech_threshold + length check |
| Corrupted file | try/except → return placeholder |
| Greek speech | Auto-detected, Whisper trained on Greek |
| M4A voice memos | Δουλεύει native — ffmpeg handles M4A |

#### Ανησυχία: Πόσα audio files στο ~/Downloads;

Στα 5000 αρχεία, εκτιμάται 2-5% audio = 100-250 αρχεία. Με base model:
- 1 λεπτό audio → ~12 sec processing
- Μέση διάρκεια αρχείου: 3-5 λεπτά
- 200 αρχεία × 4 min × 12 sec/min = **160 λεπτά total** — πολύ

**Optimization:** Τρέξε audio transcription μόνο για speech files, skip music:
```python
# Παράλληλα με άλλα content extraction
# Προτείνεται batch ανά 10 αρχεία με progress bar
```

**Sources:**
- mlx-whisper PyPI: https://pypi.org/project/mlx-whisper/
- Whisper Apple Silicon benchmarks: https://www.voicci.com/blog/apple-silicon-whisper-performance.html
- mlx-whisper notes: https://simonwillison.net/2024/Aug/13/mlx-whisper/
- faster-whisper no-speech bug: https://github.com/SYSTRAN/faster-whisper/issues/1208
- Whisper GitHub: https://github.com/openai/whisper

---

### Ε. BERTopic merge_models

#### Τι κάνει το merge_models

```python
from bertopic import BERTopic

# Βασική χρήση:
merged = BERTopic.merge_models(
    [topic_model_1, topic_model_2, topic_model_3],
    min_similarity=0.9  # default 0.7
)
```

**Πώς λειτουργεί:**
1. Ξεκινά από `topic_model_1` ως baseline
2. Για κάθε topic στα επόμενα models: αν cosine(topic_embedding_i, any_existing_topic) > min_similarity → ίδιο topic (skip)
3. Αν < min_similarity → νέο topic → προσθέτει στο merged model
4. Privacy feature: representative documents **δεν** αντιγράφονται

**Τι διατηρεί:**
- Topic embeddings από το πρώτο model
- Topic labels/names
- Topic hierarchy (αν υπάρχει)

**Τι χάνει:**
- HDBSCAN model → αντικαθίσταται από `BaseCluster` (βλ. bug)
- Representative documents (intentional για privacy)

#### Confirmed Bug: HDBSCAN → BaseCluster

**GitHub Issue #2415** (Aug 2025, ανοιχτό, BERTopic v0.17.3):

> "When merging two BERTopic models configured with HDBSCAN, the resulting merged model's `hdbscan_model` attribute becomes BaseCluster rather than preserving the original HDBSCAN configuration. This causes clustering to be bypassed when calling `.transform()` on the merged model."

**Αποτέλεσμα:** Αν καλέσεις `merged.transform(new_docs)`, δεν κάνει HDBSCAN — κάνει trivial assignment. Τα νέα αρχεία **δεν κατανέμονται σωστά** σε clusters.

**Status:** Unresolved. Δεν υπάρχει workaround στο issue.

#### partial_fit — Known Bugs (2024)

Από GitHub Issues:
1. **Issue #2097** (Jul 2024): "partial_fit example code does not work" — το example code στο documentation πετάει error
2. **Issue #2098** (Jul 2024): "partial_fit throws RuntimeWarning and inf scores" — overflow στο c-TF-IDF μετά από ~100 iterations
3. **Issue #849**: Documentation λέει ότι δουλεύει χωρίς UMAP partial_fit — δεν αληθεύει
4. **Issue #837**: Incompatible με hierarchical topics

**Ο maintainer σχολιάζει:**
> "partial_fit is not as powerful as the default alternatives due to stability issues with incremental dimensionality reduction"

#### Σύγκριση Στρατηγικών για Weekly Updates

| Στρατηγική | Ποιότητα | Πολυπλοκότητα | Bugs | Γρήγορο; |
|---|---|---|---|---|
| **Full re-cluster (HDBSCAN)** | ✅ Βέλτιστη | ✅ Απλό | ✅ Κανένα | ❌ ~2 min για 5000 files |
| `merge_models()` | 🟡 Μέτρια | 🟡 Μέτρια | ❌ BaseCluster bug | ✅ Γρήγορο |
| `partial_fit()` | ❌ Χαμηλή | ❌ Σύνθετο | ❌ Πολλά | ✅ Γρήγορο |
| **DenStream (online)** | 🟡 Μέτρια | 🟡 Μέτρια | 🟡 7 bugs (River) | ✅ Real-time |

**Για το Curator: χρησιμοποίησε DenStream για daily routing + Full re-cluster μία φορά τη βδομάδα.**

#### Τελικό Verdict

**ΜΗΝ χρησιμοποιήσεις BERTopic για το Curator.** Λόγοι:

1. **BERTopic = overhead.** Το Curator ήδη έχει HDBSCAN + toponymy/HERCULES. Το BERTopic δεν προσθέτει τίποτα — είναι wrapper γύρω από HDBSCAN + c-TF-IDF.

2. **merge_models bug είναι dealbreaker.** Αν ο merged model χάνει το HDBSCAN backend, δεν μπορεί να classify νέα αρχεία σωστά.

3. **Full re-cluster είναι απλούστερο και καλύτερο.** Για 5000 αρχεία: UMAP + HDBSCAN τρέχει σε ~2 λεπτά. Μια φορά τη βδομάδα = αποδεκτό. Ο χρήστης δεν παρατηρεί.

```python
# ΣΥΝΙΣΤΩΜΕΝΟ PATTERN για weekly updates (αντί BERTopic):
def weekly_recluster(all_file_paths: list, db_conn):
    """
    Κάθε Κυριακή νύχτα, re-cluster όλα τα αρχεία.
    Simple, reliable, no bugs.
    """
    # 1. Fetch stored embeddings από SQLite
    embeddings = fetch_all_embeddings(db_conn)  # (N, 1024)
    
    # 2. UMAP (refit αν > 500 νέα αρχεία)
    reducer = load_or_refit_umap(embeddings, db_conn)
    embeddings_10d = reducer.transform(embeddings)
    
    # 3. HDBSCAN — clean re-run
    clusterer = hdbscan.HDBSCAN(
        min_cluster_size=10, min_samples=5,
        metric='euclidean', prediction_data=True
    )
    clusterer.fit(embeddings_10d)
    
    # 4. HERCULES naming για νέα ή changed clusters
    new_labels = clusterer.labels_
    old_labels = fetch_stored_labels(db_conn)
    diff = compute_label_diff(old_labels, new_labels)
    
    if diff:  # Μόνο αν κάτι άλλαξε
        rename_changed_clusters(diff)
    
    # 5. Store new labels
    store_labels(new_labels, db_conn)
    
    # 6. Notify user if >10% of files changed cluster
    change_rate = sum(1 for i in diff if diff[i]) / len(new_labels)
    if change_rate > 0.10:
        notify_user_reorganization_available()
```

**Sources:**
- BERTopic merge docs: https://maartengr.github.io/BERTopic/getting_started/merge/merge.html
- Issue #2415 (HDBSCAN bug): https://github.com/MaartenGr/BERTopic/issues/2415
- Issue #2097 (partial_fit): https://github.com/MaartenGr/BERTopic/issues/2097
- Issue #2098 (inf scores): https://github.com/MaartenGr/BERTopic/issues/2098
- Discussion #2119 (production use): https://github.com/MaartenGr/BERTopic/discussions/2119

---

### Σύνοψη Αποφάσεων

| Topic | Απόφαση |
|---|---|
| **FLACON** | ✅ Υλοποίησε — κώδικας παραπάνω. Gower fallback για ταχύτητα, ARISE για ποιότητα |
| **LMGC** | ❌ Skip — English-only, GPU-required, overkill |
| **Ensemble HDBSCAN+Louvain** | ❌ Skip για τώρα — χάνουμε GLOSH/antilibrary |
| **Audio Whisper** | ✅ Υλοποίησε με mlx-whisper base model — code παραπάνω |
| **BERTopic merge_models** | ❌ Skip — HDBSCAN bug, partial_fit bugs. Χρησιμοποίησε weekly full re-cluster |