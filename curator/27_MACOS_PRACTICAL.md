## ΜΕΡΟΣ 27: MACOS PRACTICAL FEATURES
*(Έρευνα 2026-05-22 — 38 searches, 45+ sources)*

---

### Α. Source Domain Clustering (kMDItemWhereFroms)

#### Τι είναι και γιατί μας ενδιαφέρει

Κάθε αρχείο που κατεβαίνει μέσω browser στο macOS λαμβάνει αυτόματα το extended attribute `com.apple.metadata:kMDItemWhereFroms` — ένα binary plist array που περιέχει τα URLs προέλευσης. Αυτό είναι **deterministic, zero-ML signal**: δεν χρειαζόμαστε embedding ή inference για να ξέρουμε ότι ένα αρχείο από `moodle.ucy.ac.cy` ανήκει στο University cluster. Είναι η πιο γρήγορη πληροφορία που μπορούμε να διαβάσουμε.

Κανένα published file organizer δεν χρησιμοποιεί το `kMDItemWhereFroms` ως clustering pre-signal.

#### Τι ακριβώς περιέχει το kMDItemWhereFroms

Το attribute είναι `CFArray of CFStrings`. Συνήθως 2 στοιχεία:
- **Index 0**: Το direct download URL (π.χ. `https://moodle.ucy.ac.cy/mod/resource/view.php?id=12345`)
- **Index 1**: Το HTTP referrer — η σελίδα από την οποία έγινε click (π.χ. `https://moodle.ucy.ac.cy/course/view.php?id=789`)

Μερικές φορές έχει μόνο 1 στοιχείο (αν δεν υπάρχει referrer). Μερικές φορές 0 στοιχεία ή απουσιάζει εντελώς.

**Σημαντικές εξαιρέσεις:**
- Firefox (παλαιότερες εκδόσεις): δεν έγραφε `kMDItemWhereFroms` σε downloaded files (Mozilla Bug #337051 — τελικά διορθώθηκε)
- Private Browsing: το Firefox ρητά ΔΕΝ γράφει το attribute σε private mode (Bug #1432915)
- Αρχεία που μετακινήθηκαν από άλλες τοποθεσίες (π.χ. USB): **δεν έχουν** `kMDItemWhereFroms`
- Αρχεία που δημιουργήθηκαν locally (Xcode, TextEdit): **δεν έχουν** `kMDItemWhereFroms`

**Συμπέρασμα:** Το `kMDItemWhereFroms` είναι διαθέσιμο σε ένα σημαντικό αλλά όχι πλήρες υποσύνολο αρχείων. Πρέπει να το χρησιμοποιούμε ως optional enrichment, ποτέ ως hard requirement.

#### Πώς να το διαβάσουμε — 3 τρόποι

**Τρόπος 1: osxmetadata (απλούστερος)**
```python
from osxmetadata import OSXMetaData

md = OSXMetaData("/path/to/file.pdf")
urls = md.kMDItemWhereFroms  # Returns: ['https://...', 'https://...'] or []
```

**Τρόπος 2: xattr + plistlib (χωρίς εξωτερική βιβλιοθήκη)**
```python
import plistlib
import xattr  # pip install xattr

def read_wherefroms_xattr(path: str) -> list[str]:
    """Διαβάζει kMDItemWhereFroms μέσω xattr + plistlib."""
    try:
        raw = xattr.getxattr(path, "com.apple.metadata:kMDItemWhereFroms")
        return plistlib.loads(raw)  # Επιστρέφει list[str]
    except (OSError, KeyError, plistlib.InvalidFileException):
        return []
```

**Τρόπος 3: subprocess mdls (χωρίς εξωτερικές βιβλιοθήκες)**
```python
import subprocess, re

def read_wherefroms_mdls(path: str) -> list[str]:
    """Fallback μέσω mdls command — αργότερο αλλά χωρίς dependencies."""
    result = subprocess.run(
        ["mdls", "-name", "kMDItemWhereFroms", "-raw", path],
        capture_output=True, text=True
    )
    # Output: ("http://...",\n"http://..."\n)
    lines = result.stdout.strip().strip("()").split(",\n")
    return [l.strip().strip('"') for l in lines if l.strip() and l.strip() != '(null)']
```

**Σύσταση:** Χρησιμοποίησε `osxmetadata` — έχει ήδη εγκατασταθεί στο project (Μέρος 2 ΜΕΡΟΣ 2). Fallback σε `xattr + plistlib` αν το osxmetadata αποτύχει.

#### Domain → Cluster Hint Mapping

Για έναν CS φοιτητή στην Κύπρο, αυτό είναι το mapping:

```python
from urllib.parse import urlparse
import re

# Ordered από πιο specific σε πιο general
DOMAIN_CLUSTER_HINTS = [
    # University of Cyprus
    (r".*\.ucy\.ac\.cy",            "University"),
    (r".*\.cut\.ac\.cy",            "University"),         # CUT
    (r".*\.ac\.cy",                  "University"),         # Άλλα κυπριακά πανεπιστήμια
    (r".*\.edu",                     "University"),         # Αμερικανικά πανεπιστήμια
    (r".*\.ac\.uk",                  "University"),         # Βρετανικά
    
    # Research / Papers
    (r"arxiv\.org",                  "Research-Papers"),
    (r".*\.acm\.org",               "Research-Papers"),
    (r"ieeexplore\.ieee\.org",      "Research-Papers"),
    (r"dl\.acm\.org",               "Research-Papers"),
    (r"scholar\.google\.com",       "Research-Papers"),
    (r"semanticscholar\.org",       "Research-Papers"),
    (r"researchgate\.net",          "Research-Papers"),
    (r"springer\.com",              "Research-Papers"),
    (r"sciencedirect\.com",         "Research-Papers"),
    (r"papers\.nips\.cc",           "Research-Papers"),
    (r"openreview\.net",            "Research-Papers"),
    (r"pmc\.ncbi\.nlm\.nih\.gov",   "Research-Papers"),
    
    # Code / Development
    (r"github\.com",                "Code-Projects"),
    (r"gitlab\.com",                "Code-Projects"),
    (r"raw\.githubusercontent\.com","Code-Projects"),
    (r"pypi\.org",                  "Code-Projects"),
    (r"npmjs\.com",                 "Code-Projects"),
    (r"stackoverflow\.com",         "Code-Projects"),
    
    # Media — skip (δεν οργανώνουμε μέσω domain hint)
    (r"youtube\.com",               None),
    (r"youtu\.be",                  None),
    (r"spotify\.com",               None),
    (r"soundcloud\.com",            None),
    (r"netflix\.com",               None),
    (r"twitch\.tv",                 None),
    
    # Piazza (μαθήματα)
    (r"piazza\.com",                "University"),
    
    # Social / General — καμία hint
    (r"facebook\.com",              None),
    (r"instagram\.com",             None),
    (r"twitter\.com",               None),
    (r"reddit\.com",                None),
]

_COMPILED_HINTS = [(re.compile(pat), hint) for pat, hint in DOMAIN_CLUSTER_HINTS]


def get_source_domain(path: str) -> tuple[str | None, str | None]:
    """
    Διαβάζει kMDItemWhereFroms και επιστρέφει (domain, cluster_hint).

    Επιστρέφει:
        (domain, cluster_hint) — π.χ. ("moodle.ucy.ac.cy", "University")
        (domain, None)         — γνωστό domain, καμία hint
        (None, None)           — δεν υπάρχει kMDItemWhereFroms
    """
    urls = _read_wherefroms(path)
    if not urls:
        return (None, None)

    # Χρησιμοποίησε το πρώτο URL (direct download URL)
    primary_url = urls[0]
    try:
        parsed = urlparse(primary_url)
        domain = parsed.netloc.lower().lstrip("www.")
    except Exception:
        return (None, None)

    for pattern, hint in _COMPILED_HINTS:
        if pattern.fullmatch(domain) or pattern.search(domain):
            return (domain, hint)

    return (domain, None)  # Άγνωστο domain — καμία hint


def _read_wherefroms(path: str) -> list[str]:
    """Helper: διαβάζει kMDItemWhereFroms με fallback σε xattr."""
    try:
        from osxmetadata import OSXMetaData
        urls = OSXMetaData(path).kMDItemWhereFroms
        return urls if urls else []
    except Exception:
        pass
    try:
        import xattr
        import plistlib
        raw = xattr.getxattr(path, "com.apple.metadata:kMDItemWhereFroms")
        return plistlib.loads(raw) or []
    except Exception:
        return []
```

#### Ενσωμάτωση στο Pipeline

Το `get_source_domain()` καλείται στη **Φάση 1 (Scan)** — πριν από οποιοδήποτε embedding:

```python
# Στο scan loop:
domain, cluster_hint = get_source_domain(file_path)

file_record = {
    "path": file_path,
    "source_domain": domain,         # αποθηκεύεται στο SQLite
    "domain_cluster_hint": cluster_hint,  # χρησιμοποιείται ως soft constraint
    # ... άλλα metadata
}

# Στο clustering (Φάση 6):
# Αν cluster_hint == "University" → soft constraint: προτίμησε cluster με άλλα University files
# Υλοποίηση: weighted similarity boost (+0.15 cosine) με αρχεία ίδιου hint
```

**Πότε εφαρμόζεται ως hard deterministic sort (αντί για hint):**

Αν `kMDItemCreator` = Safari/Chrome ΚΑΙ `cluster_hint` = "University" ΚΑΙ υπάρχει ήδη University cluster → sort αμέσως, χωρίς embedding. Αυτό είναι η "fast path" για τα νέα αρχεία (Λειτουργία Β).

**Αρχεία χωρίς kMDItemWhereFroms:**
- Χαρακτηρίζονται ως `domain_cluster_hint = None`
- Ακολουθούν κανονικά το BGE-M3 → UMAP → HDBSCAN pipeline
- Δεν χάνονται — απλώς δεν έχουν το extra signal

#### Πηγές

- GitHub RhetTbull/osxmetadata — https://github.com/RhetTbull/osxmetadata
- osxmetadata PyPI — https://pypi.org/project/osxmetadata/
- MoonPoint Support kMDItemWhereFroms examples — https://support.moonpoint.com/os/os-x/downloaded-from/
- Mozilla Bug #337051 (Firefox kMDItemWhereFroms) — https://bugzilla.mozilla.org/show_bug.cgi?id=337051
- Mozilla Bug #1432915 (Private Browsing) — https://bugzilla.mozilla.org/show_bug.cgi?id=1432915
- Elastic Beats PR #5951 (kMDItemWhereFroms implementation) — https://github.com/elastic/beats/pull/5951/files
- gist dunhamsteve/2889617 (Python code) — https://gist.github.com/dunhamsteve/2889617
- Eclectic Light Company "Where did that metadata come from?" — https://eclecticlight.co/2018/01/10/where-did-that-metadata-come-from/
- RudIS blog (xattrs και URLs) — https://rud.is/b/2018/05/30/os-secrets-exposed-extracting-extended-file-attributes-and-exploring-hidden-download-urls-with-the-xattrs-package/

---

### Β. Finder Tags ως Weak Supervision

#### Τι είναι και γιατί μας ενδιαφέρει

Ο χρήστης έχει ήδη βάλει χειροκίνητα Finder tags σε μερικά αρχεία — "EPL326", "Thesis", "Work", ή χρωματικά tags (κόκκινο = urgent, πράσινο = done). Αυτά είναι **human-labeled ground truth** με μηδενικό κόστος συλλογής. Μπορούμε να τα χρησιμοποιήσουμε ως cluster seeds ή must-link constraints: αρχεία με το ίδιο tag → ίδιο cluster.

#### Τεχνική διαφορά: kMDItemUserTags vs _kMDItemUserTags vs NSURLTagNamesKey

| Attribute | Τι είναι | API |
|---|---|---|
| `kMDItemUserTags` | Spotlight MDItem attribute | `mdls -name kMDItemUserTags` |
| `_kMDItemUserTags` | Extended attribute (xattr key) | `com.apple.metadata:_kMDItemUserTags` |
| `NSURLTagNamesKey` | NSURL resource key | Μόνο names, χωρίς colors |

Το `osxmetadata` εκθέτει και τα δύο ως `.tags` — ο απλούστερος τρόπος.

#### Δομή Tag — Binary Plist Encoding

Κάθε tag αποθηκεύεται ως string με format: `"TagName\n7"` (όνομα + newline + color ID 0-7).

```
Color IDs:
0 = Καμία (no color)
1 = Gray
2 = Green
3 = Purple
4 = Blue
5 = Yellow
6 = Red
7 = Orange
```

Το `osxmetadata` αποκωδικοποιεί αυτόματα και επιστρέφει `Tag(name, color)` namedtuples.

#### Python Implementation

```python
from osxmetadata import OSXMetaData, Tag
from pathlib import Path
import xattr
import plistlib


def get_finder_tags(path: str) -> list[str]:
    """
    Επιστρέφει λίστα με τα ονόματα των Finder tags ενός αρχείου.
    Χρησιμοποιεί osxmetadata με fallback σε raw xattr.

    Returns:
        ["EPL326", "Thesis"]  — ονόματα tags (χωρίς color)
        []                    — αρχείο χωρίς tags
    """
    try:
        md = OSXMetaData(path)
        tags = md.tags  # List[Tag(name, color)]
        return [t.name for t in tags] if tags else []
    except Exception:
        pass

    # Fallback: raw xattr
    try:
        raw = xattr.getxattr(path, "com.apple.metadata:_kMDItemUserTags")
        decoded = plistlib.loads(raw)  # List[str]: ["EPL326\n0", "Work\n2"]
        result = []
        for item in decoded:
            # Format: "TagName\nColorID" — κρατάμε μόνο το TagName
            name = item.split("\n")[0].strip()
            if name:
                result.append(name)
        return result
    except Exception:
        return []


def get_finder_tags_with_colors(path: str) -> list[Tag]:
    """
    Επιστρέφει πλήρη Tag objects με name και color.
    Χρήσιμο αν θέλουμε να διαφοροποιήσουμε κόκκινα (urgent) από πράσινα (done).
    """
    try:
        md = OSXMetaData(path)
        return md.tags or []
    except Exception:
        return []
```

**Χρήση μέσω mdls (χωρίς βιβλιοθήκες):**
```bash
# Command line για debugging:
mdls -name kMDItemUserTags /path/to/file.pdf
# Output: kMDItemUserTags = ("EPL326", "Work")
```

#### Χρήση Tags ως Must-Link Constraints στο HDBSCAN

Το βασικό HDBSCAN ΔΕΝ υποστηρίζει native must-link constraints. Υπάρχουν 3 προσεγγίσεις:

**Προσέγγιση 1: Post-processing (απλή, συνιστάται)**

Αφού τρέξει το HDBSCAN, ελέγχουμε αν αρχεία με ίδιο tag βρίσκονται σε διαφορετικά clusters. Αν ναι → reassign στο πλειοψηφικό cluster του tag:

```python
from collections import Counter
import numpy as np


def apply_tag_must_links(
    labels: np.ndarray,
    file_paths: list[str],
    tag_groups: dict[str, list[int]]  # tag_name → [file_indices]
) -> np.ndarray:
    """
    Post-processing: αρχεία με ίδιο Finder tag → ίδιο cluster.
    
    Args:
        labels: HDBSCAN labels array (-1 = noise)
        file_paths: paths αρχείων (ίδια σειρά με labels)
        tag_groups: mapping tag_name → list of file indices με αυτό το tag
    
    Returns:
        Corrected labels array
    """
    new_labels = labels.copy()
    
    for tag_name, indices in tag_groups.items():
        if len(indices) < 2:
            continue  # Μόνο 1 αρχείο με αυτό το tag — τίποτα να κάνουμε
        
        # Βρες τo πλειοψηφικό cluster (εξαιρώντας noise -1)
        cluster_labels = [labels[i] for i in indices if labels[i] != -1]
        if not cluster_labels:
            continue  # Όλα noise — αφησε τα
        
        majority_cluster = Counter(cluster_labels).most_common(1)[0][0]
        
        # Reassign τα υπόλοιπα
        for idx in indices:
            if new_labels[idx] != majority_cluster:
                new_labels[idx] = majority_cluster
    
    return new_labels


def build_tag_groups(file_paths: list[str]) -> dict[str, list[int]]:
    """Δημιουργεί tag_groups dict από list of paths."""
    groups: dict[str, list[int]] = {}
    for i, path in enumerate(file_paths):
        for tag in get_finder_tags(path):
            if tag not in groups:
                groups[tag] = []
            groups[tag].append(i)
    return groups
```

**Προσέγγιση 2: Metric-space warping (embedding-level)**

Πριν το UMAP, αλλάζουμε τα embeddings ώστε αρχεία με ίδιο tag να είναι πιο κοντά:

```python
def warp_embeddings_by_tags(
    embeddings: np.ndarray,
    tag_groups: dict[str, list[int]],
    alpha: float = 0.3  # Πόσο "τραβάμε" τα embeddings μαζί
) -> np.ndarray:
    """
    Προσαρμόζει embeddings ώστε αρχεία με ίδιο tag να cluster μαζί.
    Alpha=0.0 = καμία αλλαγή, Alpha=1.0 = όλα identical.
    """
    warped = embeddings.copy()
    for tag_name, indices in tag_groups.items():
        if len(indices) < 2:
            continue
        centroid = embeddings[indices].mean(axis=0)
        for idx in indices:
            warped[idx] = (1 - alpha) * embeddings[idx] + alpha * centroid
    return warped
```

**Προσέγγιση 3: COP-KMeans (αν θέλεις explicit constraint clustering)**

```python
# pip install (από https://github.com/Behrouz-Babaki/COP-Kmeans)
from copkmeans.cop_kmeans import cop_kmeans

# Δημιουργία must-link pairs από tags
must_links = []
for tag_name, indices in tag_groups.items():
    for i in range(len(indices)):
        for j in range(i + 1, len(indices)):
            must_links.append((indices[i], indices[j]))

# Εκτέλεση
clusters, centers = cop_kmeans(
    dataset=embeddings_umap,
    k=estimated_k,
    ml=must_links,
    cl=[]  # cannot-link: εδώ κενό
)
```

**Σύσταση:** Χρησιμοποίησε **Προσέγγιση 1 (post-processing)** — είναι η πιο απλή και δεν απαιτεί επιπλέον dependencies. Η Προσέγγιση 2 (metric warping) είναι καλύτερη αρχιτεκτονικά αλλά πιο πολύπλοκη. Το COP-KMeans (Προσέγγιση 3) δεν συνδυάζεται εύκολα με HDBSCAN.

#### Tags ως Cluster Seeds (Αρχικοποίηση)

Αν έχουμε tags πριν το clustering, μπορούμε να χρησιμοποιήσουμε τα tag centroids ως initial seeds για να κατευθύνουμε το HDBSCAN:

```python
def compute_tag_centroids(
    embeddings: np.ndarray,
    tag_groups: dict[str, list[int]]
) -> dict[str, np.ndarray]:
    """Υπολογίζει centroid embedding για κάθε tag group."""
    return {
        tag: embeddings[indices].mean(axis=0)
        for tag, indices in tag_groups.items()
        if len(indices) >= 2
    }

# Αυτά τα centroids μπορούν να χρησιμοποιηθούν:
# 1. Ως similarity boost: cosine(file_embedding, tag_centroid) > 0.8 → assign to tag cluster
# 2. Ως folder name hint: αν cluster centroid κοντά σε "EPL326" tag centroid → name it "EPL326"
```

#### Πηγές

- osxmetadata GitHub README — https://github.com/RhetTbull/osxmetadata/blob/master/README.md
- osxmetadata PyPI — https://pypi.org/project/osxmetadata/0.99.38/
- Eclectic Light Company xattr tags — https://eclecticlight.co/2017/12/27/xattr-com-apple-metadata_kmditemusertags-finder-tags/
- Brett Terpstra CLI tagging — https://brettterpstra.com/2017/08/22/tagging-files-from-the-command-line/
- GitHub COP-Kmeans (Behrouz-Babaki) — https://github.com/Behrouz-Babaki/COP-Kmeans
- arXiv:2303.00522 (Semi-Supervised Constrained Clustering overview) — https://arxiv.org/abs/2303.00522
- HDBSCAN Issue #299 (clustering with constraints) — https://github.com/scikit-learn-contrib/hdbscan/issues/299
- MDPI Sensors: Constraint-Based Hierarchical Cluster Selection — https://pmc.ncbi.nlm.nih.gov/articles/PMC8153611/

---

### Γ. Git Repository Awareness

#### Το Πρόβλημα

Ένας φάκελος `~/Downloads/EPL426-compiler-project/` περιέχει `.git/`, `README.md`, `src/`, `tests/`, `Makefile`. Χωρίς ειδικό handling, το HDBSCAN μπορεί να σκορπίσει τα αρχεία σε διαφορετικά clusters: `src/lexer.py` → "Code", `README.md` → "Documentation", `Makefile` → noise. Αυτό καταστρέφει την ακεραιότητα του project.

#### Τρεις Στρατηγικές — Comparison

| Στρατηγική | Τι κάνει | Πλεονεκτήματα | Μειονεκτήματα |
|---|---|---|---|
| **A: Skip** | Παραλείπει ολόκληρο το repo | Απλό | Χάνεις τα αρχεία εντελώς |
| **B: Embed as Unit** | Ένα embedding για ολόκληρο το repo | Ακεραιότητα project | Χάνεις per-file granularity |
| **C: Same Cluster Force** | Embed αρχεία ξεχωριστά, force same cluster post-hoc | Καλύτερη granularity | Πολύπλοκο |

**Σύσταση: Στρατηγική Β (Embed as Unit)**. Ο λόγος: ένα git repo είναι εννοιολογικά μια atomic unit — "EPL426 Compiler Project". Το να έχεις separate embeddings per file δεν προσθέτει αξία γιατί ο χρήστης θέλει να βλέπει το repo ως ένα folder, όχι σκορπισμένα αρχεία.

#### Detection

```python
import os
from pathlib import Path


def is_git_repo(path: str) -> bool:
    """
    Ελέγχει αν ένα directory είναι git repo.
    Ελέγχει για .git/ subdirectory (standard repos) ή .git file (worktrees).
    """
    p = Path(path)
    if not p.is_dir():
        return False
    git_path = p / ".git"
    # Standard repo: .git/ directory
    if git_path.is_dir():
        return True
    # Worktree ή submodule: .git file (περιέχει "gitdir: ...")
    if git_path.is_file():
        return True
    return False


def find_git_repos(root: str) -> list[str]:
    """
    Βρίσκει όλα τα git repos κάτω από root χωρίς να descent μέσα σε repos.
    Αποφεύγει nested repos (submodules) να μετρηθούν δύο φορές.
    
    Returns: list of absolute paths to repo roots
    """
    repos = []
    for dirpath, dirnames, filenames in os.walk(root):
        if ".git" in dirnames or ".git" in filenames:
            repos.append(dirpath)
            # Μην κατεβείς μέσα στο .git/ ή σε nested repos
            dirnames[:] = []  # Σταματάει όλο το descent
        else:
            # Αφαίρεσε hidden dirs για speed
            dirnames[:] = [d for d in dirnames if not d.startswith(".")]
    return repos
```

#### Embedding ενός Git Repo ως Unit

Για να embed ένα git repo ως μια οντότητα, χρειαζόμαστε ένα text representation. Χρησιμοποιούμε: repo name + README + top-level file list + πρόσφατα commit messages.

```python
import os
from pathlib import Path
from collections import Counter


def embed_git_repo(repo_path: str, max_commits: int = 5) -> str:
    """
    Δημιουργεί embedding input text για ένα git repo ως atomic unit.
    
    Χρησιμοποιεί: repo_name, README, top-level files, recent commits.
    Δεν απαιτεί gitpython αν δεν είναι εγκατεστημένο — fallback σε subprocess.
    
    Returns: text string για embed με BGE-M3
    """
    p = Path(repo_path)
    repo_name = p.name  # "EPL426-compiler-project"
    parts = [f"Git Repository: {repo_name}"]
    
    # 1. README content (πιο σημαντικό σήμα)
    readme_content = _read_readme(repo_path)
    if readme_content:
        parts.append(f"README:\n{readme_content[:2000]}")
    
    # 2. Top-level file/folder list (structure signal)
    top_items = _get_top_level_items(repo_path)
    if top_items:
        parts.append(f"Top-level structure: {', '.join(top_items[:30])}")
    
    # 3. Programming language (από extensions)
    main_lang = _detect_main_language(repo_path)
    if main_lang:
        parts.append(f"Primary language: {main_lang}")
    
    # 4. Recent commit messages (context signal)
    commits = _get_recent_commits(repo_path, max_count=max_commits)
    if commits:
        parts.append(f"Recent work:\n" + "\n".join(f"- {c}" for c in commits))
    
    return "\n\n".join(parts)


def _read_readme(repo_path: str) -> str:
    """Διαβάζει README (αν υπάρχει) — δοκιμάζει README.md, README.txt, README."""
    for name in ["README.md", "README.rst", "README.txt", "README", "readme.md"]:
        candidate = os.path.join(repo_path, name)
        if os.path.isfile(candidate):
            try:
                with open(candidate, encoding="utf-8", errors="replace") as f:
                    return f.read(3000)  # Πρώτα 3000 chars
            except OSError:
                pass
    return ""


def _get_top_level_items(repo_path: str) -> list[str]:
    """Επιστρέφει top-level files/dirs εξαιρώντας .git και hidden."""
    try:
        items = os.listdir(repo_path)
        return sorted([
            i for i in items
            if not i.startswith(".") and i != "__pycache__"
        ])
    except OSError:
        return []


def _detect_main_language(repo_path: str, max_files: int = 200) -> str | None:
    """
    Εντοπίζει κύρια γλώσσα από file extensions.
    Απλή heuristic — δεν χρειάζεται Linguist.
    """
    EXT_TO_LANG = {
        ".py": "Python", ".js": "JavaScript", ".ts": "TypeScript",
        ".java": "Java", ".c": "C", ".cpp": "C++", ".h": "C/C++",
        ".rs": "Rust", ".go": "Go", ".rb": "Ruby", ".php": "PHP",
        ".swift": "Swift", ".kt": "Kotlin", ".cs": "C#",
        ".r": "R", ".m": "MATLAB", ".scala": "Scala",
        ".sh": "Shell", ".bash": "Shell",
    }
    counter: Counter = Counter()
    count = 0
    for root, dirs, files in os.walk(repo_path):
        dirs[:] = [d for d in dirs if d not in (".git", "__pycache__", "node_modules")]
        for f in files:
            ext = Path(f).suffix.lower()
            if ext in EXT_TO_LANG:
                counter[EXT_TO_LANG[ext]] += 1
            count += 1
            if count >= max_files:
                break
        if count >= max_files:
            break
    if not counter:
        return None
    return counter.most_common(1)[0][0]


def _get_recent_commits(repo_path: str, max_count: int = 5) -> list[str]:
    """
    Επιστρέφει τα τελευταία commit messages.
    Πρώτα δοκιμάζει gitpython, fallback σε subprocess git.
    """
    # Προσπάθεια με gitpython
    try:
        from git import Repo, InvalidGitRepositoryError
        repo = Repo(repo_path)
        messages = []
        for commit in repo.iter_commits(max_count=max_count):
            # Κρατάμε μόνο την πρώτη γραμμή του message (subject line)
            subject = commit.message.split("\n")[0].strip()
            if subject:
                messages.append(subject)
        return messages
    except Exception:
        pass
    
    # Fallback: subprocess git log
    try:
        import subprocess
        result = subprocess.run(
            ["git", "-C", repo_path, "log",
             f"--max-count={max_count}", "--pretty=format:%s"],
            capture_output=True, text=True, timeout=5
        )
        if result.returncode == 0:
            return [l.strip() for l in result.stdout.splitlines() if l.strip()]
    except Exception:
        pass
    
    return []
```

#### Ενσωμάτωση στο Scan Loop

```python
# Στη Φάση 1 (Scan) του pipeline:

def scan_directory(root: str) -> list[dict]:
    """Scan με git repo awareness."""
    records = []
    git_repos = find_git_repos(root)  # Βρες repos πρώτα
    git_repo_paths = set(git_repos)
    
    # Embed repos ως units
    for repo_path in git_repos:
        embedding_text = embed_git_repo(repo_path)
        records.append({
            "path": repo_path,
            "type": "git_repo",
            "embedding_text": embedding_text,
            "is_atomic_unit": True,
        })
    
    # Scan υπόλοιπα αρχεία (εξαιρώντας repo directories)
    for dirpath, dirnames, filenames in os.walk(root):
        # Αφαίρεσε repo directories από το walk
        dirnames[:] = [
            d for d in dirnames
            if os.path.join(dirpath, d) not in git_repo_paths
        ]
        for fname in filenames:
            full_path = os.path.join(dirpath, fname)
            # Ελέγχει αν το αρχείο είναι ΕΝΤΟΣ κάποιου repo
            if any(full_path.startswith(rp) for rp in git_repo_paths):
                continue  # Skip — το repo ήδη embedded ως unit
            records.append({"path": full_path, "type": "file", "is_atomic_unit": False})
    
    return records
```

#### Τι γίνεται με τα αρχεία ΕΚΤΟΣ .git αλλά ΕΝΤΟΣ repo dir

Π.χ. `~/Downloads/my-project/README.md` — είναι εντός του repo. Η στρατηγική Β τα **παραλείπει** από individual embedding γιατί ήδη included στο `embed_git_repo()`. Ο χρήστης βλέπει `my-project/` ως ένα folder στο approval UI, όχι τα επιμέρους αρχεία.

#### Πηγές

- python.code-maven.com: os.walk + .git skipping — https://python.code-maven.com/python-traversing-directory-tree
- GitHub Gist: detect git repo — https://gist.github.com/brettinternet/0d2225ffb6b224c515643f630a65b463
- GitPython tutorial — https://gitpython.readthedocs.io/en/stable/tutorial.html
- GitPython PyPI — https://pypi.org/project/GitPython/
- gitparse (structured repo parsing) — https://github.com/von-development/gitparse
- Scivision: detect project languages — https://www.scivision.dev/detect-project-primary-languages/
- whats-that-code PyPI — https://pypi.org/project/whats-that-code/

---

### Δ. NER-Based Project Detection

#### Το Πρόβλημα

Ένα assignment για το μάθημα EPL326 μπορεί να αποτελείται από: `EPL326_hw3.pdf` (εκφώνηση), `solution.py` (κώδικας), `report.docx` (αναφορά), `data.csv` (δεδομένα). Τα 4 αυτά αρχεία έχουν πολύ διαφορετικά embeddings (PDF ≠ Python ≠ Word ≠ CSV), οπότε το HDBSCAN μπορεί να τα σκορπίσει σε διαφορετικά clusters. Όμως ΟΛΑ αναφέρουν την entity "EPL326" — αυτό είναι cross-format project signal.

#### Ακαδημαϊκή Υποστήριξη

**arXiv:2412.14867** (ECIR 2025, Keraghel & Nadif): Δημιουργία graph με nodes = documents και edges weighted by named entity similarity → GCN optimization. Αποτέλεσμα: **υπερτερεί** των co-occurrence-based methods, κυρίως για documents rich in named entities. Αυτό ισχύει ακριβώς για academic files (course codes, professor names, assignment numbers).

**arXiv:1807.07777** (2018): "Semantic Document Clustering on Named Entity Features" — πρώιμη εργασία που δείχνει ότι NE features για clustering δουλεύουν καλά σε domain-specific corpora.

#### Ποιο spaCy Model για Greek+English;

**Πρόβλημα:** Τα αρχεία μας είναι mixed Greek+English. Χρειαζόμαστε ένα model που χειρίζεται και τις δύο γλώσσες.

| Model | Greek | English | NER | Σύσταση |
|---|---|---|---|---|
| `en_core_web_sm` | ❌ | ✅ | ✅ | Αγγλικά μόνο |
| `el_core_news_sm` | ✅ | ❌ | ✅ | Ελληνικά μόνο |
| `el_core_news_lg` | ✅ | ❌ | ✅ | Ελληνικά, μεγαλύτερο |
| `xx_ent_wiki_sm` | ✅ (partial) | ✅ | ✅ (NER only) | Πολυγλωσσικό, NER-only |

**Ανακάλυψη:** Το `xx_ent_wiki_sm` έχει διαθεσιμότητα issues στο spaCy 3.6+ (GitHub Discussion #13852). Δεν είναι επίσημα deprecated αλλά η εγκατάσταση μπορεί να αποτύχει.

**Βέλτιστη στρατηγική για Greek+English mixed files:**

Χρησιμοποίησε **dual pipeline**: `en_core_web_sm` για αγγλικά κείμενα και `el_core_news_lg` για ελληνικά, με language detection πριν. Για mixed text, εφάρμοσε και τα δύο και merge τα entities.

Εναλλακτικά, χρησιμοποίησε **custom EntityRuler μόνο** (χωρίς statistical NER) για τα πιο σημαντικά patterns (course codes, assignment numbers) — είναι 100× γρηγορότερο και δεν χρειάζεται model download.

```bash
# Installation
python -m spacy download en_core_web_sm      # 12MB, αγγλικά
python -m spacy download el_core_news_lg     # 82MB, ελληνικά
python -m spacy download xx_ent_wiki_sm      # πολυγλωσσικό (αν δουλέψει)
```

#### Ποιοι Entity Types είναι χρήσιμοι για File Organization

spaCy αναγνωρίζει 18 entity types. Για file organizer:

| Entity Type | Παραδείγματα | Χρήσιμο; |
|---|---|---|
| `ORG` | "University of Cyprus", "Google", "UCY" | ✅ Πολύ χρήσιμο |
| `PERSON` | "Prof. Smith", "Κώστας Παπαδόπουλος" | ✅ Χρήσιμο για courses |
| `PRODUCT` | "Python", "TensorFlow", "LaTeX" | ✅ Χρήσιμο για tech projects |
| `EVENT` | "Midterm 2024", "Assignment 3" | ✅ Χρήσιμο αν αναγνωριστεί |
| `DATE` | "Spring 2025", "Ιανουάριος 2026" | ⚠️ Μερικώς χρήσιμο |
| `GPE` | "Cyprus", "Nicosia" | ❌ Σπάνια χρήσιμο |
| `MONEY`, `CARDINAL` | αριθμοί | ❌ Θόρυβος |

#### Custom EntityRuler για Academic Patterns

Τα πιο πολύτιμα entities για έναν CS φοιτητή δεν αναγνωρίζονται από statistical NER (π.χ. course codes "EPL326"). Χρησιμοποιούμε `EntityRuler` με regex patterns:

```python
import re
import spacy
from spacy.language import Language


# Patterns για academic entities
ACADEMIC_PATTERNS = [
    # Course codes: EPL326, EPL-326, CS101, MATH201, PHYS100
    {"label": "COURSE_CODE",
     "pattern": [{"TEXT": {"REGEX": r"^[A-Z]{2,4}[-_]?\d{3,4}[A-Za-z]?$"}}]},
    
    # Assignment numbers: "Assignment 3", "HW3", "Homework 2", "Lab 4"
    {"label": "ASSIGNMENT",
     "pattern": [
         {"LOWER": {"IN": ["assignment", "hw", "homework", "lab", "exercise", "project"]}},
         {"IS_DIGIT": True}
     ]},
    
    # Semester: "Spring 2025", "Fall 2024", "Winter 2026"
    {"label": "SEMESTER",
     "pattern": [
         {"LOWER": {"IN": ["spring", "fall", "winter", "summer"]}},
         {"TEXT": {"REGEX": r"^\d{4}$"}}
     ]},
    
    # Greek semester: "Εαρινό 2025", "Χειμερινό 2024"
    {"label": "SEMESTER",
     "pattern": [
         {"LOWER": {"IN": ["εαρινό", "χειμερινό", "εαρινο", "χειμερινο"]}},
         {"TEXT": {"REGEX": r"^\d{4}$"}}
     ]},
    
    # Exam types: "Midterm", "Final Exam", "Quiz"
    {"label": "EXAM_TYPE",
     "pattern": [
         {"LOWER": {"IN": ["midterm", "final", "quiz", "exam"]}}
     ]},
    
    # Version markers: "v2", "v1.3", "final", "draft"
    {"label": "VERSION_MARKER",
     "pattern": [{"TEXT": {"REGEX": r"^v\d+(\.\d+)?$"}}]},
    
    # Greek course codes (αν υπάρχουν patterns ΕΠΛ)
    {"label": "COURSE_CODE",
     "pattern": [{"TEXT": {"REGEX": r"^ΕΠΛ\d{3}$"}}]},  # π.χ. ΕΠΛ326
]


def build_nlp_pipeline(lang: str = "en") -> Language:
    """
    Δημιουργεί spaCy pipeline με custom EntityRuler για academic patterns.
    
    Args:
        lang: "en" για αγγλικά, "el" για ελληνικά, "xx" για multilingual
    """
    if lang == "el":
        try:
            nlp = spacy.load("el_core_news_lg")
        except OSError:
            nlp = spacy.load("el_core_news_sm")
    elif lang == "xx":
        try:
            nlp = spacy.load("xx_ent_wiki_sm")
        except OSError:
            nlp = spacy.blank("xx")
    else:
        try:
            nlp = spacy.load("en_core_web_sm")
        except OSError:
            nlp = spacy.blank("en")
    
    # Πρόσθεσε EntityRuler ΠΡΙΝ το ner για να μην overwrite τα custom patterns
    if "entity_ruler" not in nlp.pipe_names:
        ruler = nlp.add_pipe("entity_ruler", before="ner" if "ner" in nlp.pipe_names else "last")
        ruler.add_patterns(ACADEMIC_PATTERNS)
    
    return nlp


# Singleton instances (load once)
_NLP_EN = None
_NLP_EL = None


def get_nlp(lang: str = "en") -> Language:
    global _NLP_EN, _NLP_EL
    if lang == "el":
        if _NLP_EL is None:
            _NLP_EL = build_nlp_pipeline("el")
        return _NLP_EL
    else:
        if _NLP_EN is None:
            _NLP_EN = build_nlp_pipeline("en")
        return _NLP_EN
```

#### Κύρια Συνάρτηση: extract_entities

```python
from collections import defaultdict
import langdetect  # pip install langdetect


USEFUL_ENTITY_TYPES = {
    "ORG", "PERSON", "PRODUCT", "EVENT",
    "COURSE_CODE", "ASSIGNMENT", "SEMESTER", "EXAM_TYPE"
}


def detect_language(text: str) -> str:
    """Ανιχνεύει γλώσσα κειμένου. Επιστρέφει 'el' ή 'en'."""
    try:
        from langdetect import detect
        lang = detect(text[:500])  # Χρησιμοποίησε πρώτα 500 chars
        return "el" if lang == "el" else "en"
    except Exception:
        return "en"  # Default σε αγγλικά


def extract_entities(text: str, max_length: int = 5000) -> dict[str, set[str]]:
    """
    Εξάγει named entities από κείμενο.
    
    Args:
        text: κείμενο (Greek, English, ή mixed)
        max_length: max chars για επεξεργασία (performance bound)
    
    Returns:
        dict[entity_type, set[entity_text]]
        Π.χ.: {"COURSE_CODE": {"EPL326", "EPL232"}, "PERSON": {"Prof. Smith"}}
    """
    if not text or not text.strip():
        return {}
    
    text = text[:max_length]
    
    # Ανίχνευση γλώσσας
    lang = detect_language(text)
    nlp = get_nlp(lang)
    
    # Χρησιμοποίησε nlp.pipe αντί για nlp() για μεγαλύτερα κείμενα
    doc = nlp(text)
    
    entities: dict[str, set[str]] = defaultdict(set)
    
    for ent in doc.ents:
        if ent.label_ in USEFUL_ENTITY_TYPES:
            # Normalize: lowercase για σύγκριση, αλλά φύλαξε original
            normalized = ent.text.strip()
            if len(normalized) >= 2:  # Φίλτρο πολύ μικρών entities
                entities[ent.label_].add(normalized)
    
    return dict(entities)


def extract_entities_batch(
    file_texts: list[tuple[str, str]]  # [(path, text), ...]
) -> dict[str, dict[str, set[str]]]:
    """
    Εξάγει entities για πολλά αρχεία.
    Χρησιμοποιεί nlp.pipe για speed (batch processing).
    
    Returns:
        {file_path: {entity_type: {entity_text, ...}}}
    """
    # Ομαδοποίησε κατά γλώσσα για batch processing
    en_files, el_files = [], []
    for path, text in file_texts:
        if detect_language(text) == "el":
            el_files.append((path, text[:5000]))
        else:
            en_files.append((path, text[:5000]))
    
    results = {}
    
    for lang, files in [("en", en_files), ("el", el_files)]:
        if not files:
            continue
        nlp = get_nlp(lang)
        paths, texts = zip(*files)
        for path, doc in zip(paths, nlp.pipe(texts, batch_size=50)):
            entities = defaultdict(set)
            for ent in doc.ents:
                if ent.label_ in USEFUL_ENTITY_TYPES:
                    normalized = ent.text.strip()
                    if len(normalized) >= 2:
                        entities[ent.label_].add(normalized)
            results[path] = dict(entities)
    
    return results
```

#### Co-occurrence Matrix → Extra Clustering Signal

```python
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity


def build_entity_cooccurrence_signal(
    file_entities: dict[str, dict[str, set[str]]],
    entity_type_weights: dict[str, float] = None
) -> np.ndarray:
    """
    Δημιουργεί similarity matrix βάσει shared entities.
    
    Αρχεία που μοιράζονται entities (π.χ. "EPL326") → high similarity.
    Αυτό χρησιμοποιείται ΩΣ ΕΠΙΠΛΕΟΝ SIGNAL, όχι αντί για embedding similarity.
    
    Args:
        file_entities: {path: {type: {entities}}}
        entity_type_weights: βάρη ανά τύπο entity (default: COURSE_CODE=2.0, άλλα=1.0)
    
    Returns:
        (n_files × n_files) matrix με entity similarity scores [0, 1]
    """
    if entity_type_weights is None:
        entity_type_weights = {
            "COURSE_CODE": 3.0,   # Πιο ισχυρό σήμα — ίδιο μάθημα = ίδιο project
            "ASSIGNMENT": 2.5,    # "Assignment 3" = ισχυρό hint
            "SEMESTER": 1.5,
            "PERSON": 1.5,        # Ίδιος καθηγητής
            "ORG": 1.0,
            "PRODUCT": 1.0,
            "EVENT": 1.0,
            "EXAM_TYPE": 0.5,     # "Midterm" μόνο δεν αρκεί
        }
    
    paths = list(file_entities.keys())
    n = len(paths)
    similarity_matrix = np.zeros((n, n))
    
    for i, path_i in enumerate(paths):
        for j, path_j in enumerate(paths):
            if i >= j:
                continue
            
            sim = _compute_entity_similarity(
                file_entities[path_i],
                file_entities[path_j],
                entity_type_weights
            )
            similarity_matrix[i, j] = sim
            similarity_matrix[j, i] = sim
        
        similarity_matrix[i, i] = 1.0  # Self-similarity = 1
    
    return similarity_matrix


def _compute_entity_similarity(
    entities_a: dict[str, set[str]],
    entities_b: dict[str, set[str]],
    weights: dict[str, float]
) -> float:
    """Υπολογίζει weighted Jaccard similarity μεταξύ entity sets."""
    total_weight = 0.0
    shared_weight = 0.0
    
    all_types = set(entities_a.keys()) | set(entities_b.keys())
    
    for etype in all_types:
        w = weights.get(etype, 1.0)
        set_a = entities_a.get(etype, set())
        set_b = entities_b.get(etype, set())
        
        # Normalize entity text για σύγκριση
        norm_a = {e.lower() for e in set_a}
        norm_b = {e.lower() for e in set_b}
        
        intersection = len(norm_a & norm_b)
        union = len(norm_a | norm_b)
        
        if union > 0:
            shared_weight += w * intersection
            total_weight += w * union
    
    return shared_weight / total_weight if total_weight > 0 else 0.0


def fuse_embedding_and_entity_signals(
    embedding_similarity: np.ndarray,
    entity_similarity: np.ndarray,
    alpha: float = 0.8  # Βάρος για embedding (0.8 = dominant signal)
) -> np.ndarray:
    """
    Συνδυάζει embedding cosine similarity με entity co-occurrence similarity.
    
    Formula: final = alpha * embed_sim + (1 - alpha) * entity_sim
    Προτεινόμενο alpha: 0.8 (embedding dominant, entity ως enrichment)
    """
    return alpha * embedding_similarity + (1 - alpha) * entity_similarity
```

#### Performance — Πόσο Διαρκεί για 5000 Αρχεία

spaCy στο Apple Silicon (M1/M2) τρέχει **8× γρηγορότερα** με το `thinc-apple-ops` package (pip install thinc-apple-ops). Benchmarks από Explosion AI:
- CPU: ~1821 words/sec (transformer models)
- Metal GPU: ~8648 words/sec

Για file organizer (μικρά text snippets, 2000-5000 tokens ανά αρχείο):
- `en_core_web_sm` (CPU, 5000 files × 2000 tokens avg): εκτίμηση **~5-10 λεπτά** χωρίς thinc-apple-ops
- Με `thinc-apple-ops`: **~1-2 λεπτά**
- Χρησιμοποίησε `nlp.pipe(batch_size=50)` για batch processing — 3-5× γρηγορότερο από ένα-ένα

```bash
pip install thinc-apple-ops  # Apple Silicon acceleration για spaCy
pip install langdetect        # Language detection
```

#### Πηγές

- arXiv:2412.14867 (GCN + NER document clustering) — https://arxiv.org/abs/2412.14867
- arXiv:1807.07777 (Semantic clustering on NE features) — https://arxiv.org/abs/1807.07777v1
- spaCy EntityRuler docs — https://spacy.io/api/entityruler/
- spaCy rule-based matching — https://spacy.io/usage/rule-based-matching
- spaCy Greek models — https://spacy.io/models/el (el_core_news_sm/lg)
- spaCy multilingual models — https://spacy.io/models/xx
- spaCy Discussion #13852 (xx_ent_wiki_sm status) — https://github.com/explosion/spaCy/discussions/13852
- thinc-apple-ops PyPI — https://pypi.org/project/thinc-apple-ops/
- Explosion AI Metal Performance Shaders blog — https://explosion.ai/blog/metal-performance-shaders
- HuggingFace spacy/xx_ent_wiki_sm — https://huggingface.co/spacy/xx_ent_wiki_sm

---

### Ε. Sensitive File Audit (Proactive)

#### Η Ιδέα

Αντί να αποκλείουμε σιωπηρά τα sensitive files (αυτό ήδη γίνεται στο ΜΕΡΟΣ 19), προτείνουμε στον χρήστη ενεργητικά: "Βρήκα 3 αρχεία που μοιάζουν με credentials στο Downloads. Θέλεις να τα διαγράψεις;". Αυτό είναι **security hygiene feature** — κανένας file organizer δεν το κάνει.

#### Reuse από ΜΕΡΟΣ 19

Το ΜΕΡΟΣ 19 έχει ήδη:
- `TIER1_FILENAME_GLOBS` — patterns για hard exclusion
- `TIER1_SECRETS_IN_NAME` — λέξεις στο filename
- `_TIER1_GLOB_RE`, `_TIER2_GLOB_RE` — precompiled regex
- `check_path(path)` → `("tier1"|"tier2"|"ok", reason)`

Για το Sensitive Audit, **επαναχρησιμοποιούμε αυτό το σύστημα** και προσθέτουμε:
1. Συλλογή όλων των tier1/tier2 αρχείων (αντί για silent skip)
2. Categorization ανά τύπο κινδύνου
3. UX για display

#### Πότε να τρέχει

| Option | Πότε | Πλεονεκτήματα | Μειονεκτήματα |
|---|---|---|---|
| **Α: At scan time** | Κατά την αρχική σάρωση | Χωρίς extra pass | Μπορεί να μπερδέψει με το main flow |
| **B: Separate "Security" page** | On-demand | Καθαρό UX, focus | Χρήστης πρέπει να το ανακαλύψει |
| **C: On-demand badge** | Notification badge στο UI | Non-intrusive | Μπορεί να αγνοηθεί |

**Σύσταση: Α + C**: Τρέχει at scan time (δωρεάν — ήδη σκανάρουμε), και **badge** στην κύρια σελίδα ("3 sensitive files found"). Κλικ → modal με λεπτομέρειες. Δεν χρησιμοποιούμε macOS notification γιατί χρειάζεται code signing.

#### Χρήση gitleaks για Content-Based Scanning

Το gitleaks (Go binary) μπορεί να σκανάρει directories **χωρίς git repo**:

```bash
# macOS installation
brew install gitleaks

# Scan Downloads folder (χωρίς git)
gitleaks dir /Users/username/Downloads \
    --report-format json \
    --report-path /tmp/gitleaks_report.json \
    --max-archive-depth 0 \
    --verbose

# Output: JSON με findings: [{RuleID, File, StartLine, Secret (redacted), Match}]
```

**Σημαντικό:** Το gitleaks κάνει content inspection (ανοίγει αρχεία). Αυτό είναι **opt-in** feature — ο χρήστης πρέπει να το ενεργοποιήσει ρητά. Η βασική path-only scanning (ΜΕΡΟΣ 19) τρέχει πάντα.

#### UX Design — Τι Δείχνουμε

```
┌─────────────────────────────────────────────────────────────┐
│  Security Audit — 5 Sensitive Files Found                  │
│                                                             │
│  ⚠️  HIGH RISK (2 files)                                   │
│  • ~/.ssh/id_rsa_backup.pem — SSH private key              │
│    Found in: ~/Downloads/old-laptop-backup/                │
│    [Delete] [Move to ~/.ssh/] [Ignore]                     │
│                                                             │
│  • api_keys.txt — Contains "token" in filename             │
│    Found in: ~/Downloads/                                   │
│    [Delete] [Move to Vault] [Ignore]                       │
│                                                             │
│  ⚡ MEDIUM RISK (3 files)                                   │
│  • config.local.json — Local config (may have secrets)     │
│    [Review] [Ignore]                                       │
│  • database.yml — Database config                           │
│    [Review] [Ignore]                                       │
│  • .env.backup — Environment variables backup               │
│    [Delete] [Ignore]                                       │
│                                                             │
│  Privacy: Files are matched by path/name ONLY.             │
│  No content was read.                                      │
│                                                             │
│  [Run Deep Scan with gitleaks] [Dismiss All]               │
└─────────────────────────────────────────────────────────────┘
```

#### Python Implementation

```python
"""
curator_sensitive_audit.py
Proactive sensitive file detection for Curator.
Extends the exclusion system from ΜΕΡΟΣ 19 with user-facing reporting.
"""
import os
from pathlib import Path
from dataclasses import dataclass, field
from typing import Optional
import subprocess
import json


# Import from ΜΕΡΟΣ 19 implementation
# from curator_exclusion import check_path, TIER1_PATH_PREFIXES_EXPANDED


@dataclass
class SensitiveFinding:
    """Ένα εύρημα από το security audit."""
    path: str
    risk_level: str          # "HIGH" ή "MEDIUM"
    reason: str              # Γιατί flagged
    category: str            # SSH_KEY, CREDENTIAL, ENV_FILE, κλπ.
    suggested_action: list[str] = field(default_factory=list)  # ["Delete", "Move", "Ignore"]


# Κατηγορίες και αντίστοιχες προτεινόμενες ενέργειες
REASON_TO_CATEGORY = {
    "id_rsa":          ("SSH_KEY",    "HIGH",   ["Delete", "Move to ~/.ssh/", "Ignore"]),
    "id_ed25519":      ("SSH_KEY",    "HIGH",   ["Delete", "Move to ~/.ssh/", "Ignore"]),
    ".pem":            ("PRIVATE_KEY","HIGH",   ["Delete", "Encrypt", "Ignore"]),
    ".p12":            ("CERT_BUNDLE","HIGH",   ["Delete", "Encrypt", "Ignore"]),
    "credentials":     ("CREDENTIAL", "HIGH",   ["Delete", "Move to Vault", "Ignore"]),
    ".env":            ("ENV_FILE",   "HIGH",   ["Delete", "Review & Delete", "Ignore"]),
    "secret":          ("SECRET_FILE","HIGH",   ["Delete", "Encrypt", "Ignore"]),
    "token":           ("TOKEN_FILE", "HIGH",   ["Delete", "Review", "Ignore"]),
    "apikey":          ("API_KEY",    "HIGH",   ["Delete", "Review", "Ignore"]),
    ".db":             ("DATABASE",   "MEDIUM", ["Review", "Ignore"]),
    ".sqlite":         ("DATABASE",   "MEDIUM", ["Review", "Ignore"]),
    "password":        ("PASSWORD",   "MEDIUM", ["Review & Delete", "Ignore"]),
    "config.local":    ("LOCAL_CONFIG","MEDIUM",["Review", "Ignore"]),
    "database.yml":    ("DB_CONFIG",  "MEDIUM", ["Review", "Ignore"]),
}


def _categorize_finding(reason: str) -> tuple[str, str, list[str]]:
    """Επιστρέφει (category, risk_level, suggested_actions) από reason string."""
    reason_lower = reason.lower()
    for key, (cat, risk, actions) in REASON_TO_CATEGORY.items():
        if key.lower() in reason_lower:
            return (cat, risk, actions)
    return ("SENSITIVE_FILE", "MEDIUM", ["Review", "Ignore"])


def audit_sensitive_files(
    directory: str,
    include_tier2: bool = True,
    max_findings: int = 100
) -> list[SensitiveFinding]:
    """
    Σκανάρει directory για sensitive αρχεία και επιστρέφει λίστα findings.
    
    ΣΗΜΑΝΤΙΚΟ: Ελέγχει ΜΟΝΟ path/filename — ποτέ δεν ανοίγει αρχεία.
    
    Args:
        directory: root directory για scan
        include_tier2: αν συμπεριληφθούν και τα MEDIUM risk (Tier 2)
        max_findings: max αριθμός findings (για performance)
    
    Returns:
        List[SensitiveFinding] ταξινομημένα: HIGH πρώτα
    """
    findings: list[SensitiveFinding] = []
    
    # Import exclusion system
    try:
        from curator_exclusion import check_path
    except ImportError:
        # Inline fallback αν δεν υπάρχει ο module
        check_path = _inline_check_path
    
    for dirpath, dirnames, filenames in os.walk(directory):
        if len(findings) >= max_findings:
            break
        
        # Φιλτράρισε hidden dirs (αλλά ΟΧΙ για audit — θέλουμε να βρούμε τα .env!)
        # Εξαίρεση: .git directory (δεν μας ενδιαφέρει)
        dirnames[:] = [d for d in dirnames if d != ".git"]
        
        for fname in filenames:
            if len(findings) >= max_findings:
                break
            
            full_path = os.path.join(dirpath, fname)
            tier, reason = check_path(full_path)
            
            if tier == "tier1":
                cat, risk, actions = _categorize_finding(reason or fname)
                findings.append(SensitiveFinding(
                    path=full_path,
                    risk_level="HIGH",
                    reason=reason or f"Sensitive filename: {fname}",
                    category=cat,
                    suggested_action=actions
                ))
            elif tier == "tier2" and include_tier2:
                cat, risk, actions = _categorize_finding(reason or fname)
                findings.append(SensitiveFinding(
                    path=full_path,
                    risk_level="MEDIUM",
                    reason=reason or f"Potentially sensitive: {fname}",
                    category=cat,
                    suggested_action=actions
                ))
    
    # Sort: HIGH πρώτα, μετά MEDIUM
    findings.sort(key=lambda f: (0 if f.risk_level == "HIGH" else 1, f.path))
    return findings


def audit_sensitive_files_summary(directory: str) -> dict:
    """
    Γρήγορο summary για badge display στο UI.
    Returns: {"high": N, "medium": M, "total": N+M}
    """
    findings = audit_sensitive_files(directory, max_findings=50)
    high = sum(1 for f in findings if f.risk_level == "HIGH")
    medium = sum(1 for f in findings if f.risk_level == "MEDIUM")
    return {"high": high, "medium": medium, "total": high + medium}


def run_gitleaks_deep_scan(
    directory: str,
    report_path: str = "/tmp/curator_gitleaks.json"
) -> list[dict]:
    """
    Εκτελεί gitleaks dir για content-based secret scanning.
    ΑΠΑΙΤΕΙ: brew install gitleaks
    ΠΡΟΣΟΧΗ: Ανοίγει και διαβάζει αρχεία — χρησιμοποίησε μόνο με user consent.
    
    Returns:
        List of findings από gitleaks JSON output
        [] αν το gitleaks δεν είναι εγκατεστημένο ή αποτύχει
    """
    try:
        result = subprocess.run(
            [
                "gitleaks", "dir",
                directory,
                "--report-format", "json",
                "--report-path", report_path,
                "--max-archive-depth", "0",  # Μην κάνεις zip extraction
                "--no-banner",
            ],
            capture_output=True, text=True, timeout=120
        )
        # gitleaks επιστρέφει exit code 1 αν βρει findings (φυσιολογικό!)
        if os.path.exists(report_path):
            with open(report_path) as f:
                return json.load(f) or []
    except FileNotFoundError:
        # gitleaks δεν είναι εγκατεστημένο
        pass
    except subprocess.TimeoutExpired:
        pass
    except Exception:
        pass
    return []


def format_audit_for_ui(findings: list[SensitiveFinding]) -> dict:
    """
    Formatάρει findings για το Tauri/React UI.
    Returns JSON-serializable dict.
    """
    return {
        "total": len(findings),
        "high_count": sum(1 for f in findings if f.risk_level == "HIGH"),
        "medium_count": sum(1 for f in findings if f.risk_level == "MEDIUM"),
        "findings": [
            {
                "path": f.path,
                "filename": Path(f.path).name,
                "parent_dir": str(Path(f.path).parent),
                "risk_level": f.risk_level,
                "reason": f.reason,
                "category": f.category,
                "actions": f.suggested_action,
            }
            for f in findings
        ],
        "privacy_notice": (
            "Files matched by path/filename only. "
            "No file content was read during this scan."
        )
    }


# Inline fallback αν curator_exclusion δεν είναι import-able
def _inline_check_path(file_path) -> tuple[str, Optional[str]]:
    """Minimal version του check_path από ΜΕΡΟΣ 19."""
    import re, fnmatch
    p = Path(file_path).expanduser().resolve()
    name = p.name
    stem = p.stem.lower()
    
    TIER1_GLOBS = [
        "*.pem", "*.key", "*.p12", "*.pfx", "*.p8",
        "id_rsa", "id_dsa", "id_ed25519", "id_ecdsa",
        ".env", ".env.*", "credentials.json",
        "*secret*", "*token*", "*apikey*", "*credential*",
        ".netrc", ".htpasswd", "*.gpg", ".npmrc",
    ]
    TIER2_GLOBS = ["*.db", "*.sqlite", "*.sqlite3", "*password*", ".gitconfig"]
    
    tier1_re = re.compile(
        '|'.join(f'(?:{fnmatch.translate(g)})' for g in TIER1_GLOBS),
        re.IGNORECASE
    )
    tier2_re = re.compile(
        '|'.join(f'(?:{fnmatch.translate(g)})' for g in TIER2_GLOBS),
        re.IGNORECASE
    )
    SECRET_WORDS = re.compile(
        r'\b(secret|token|apikey|api_key|password|passwd|credentials?|private_?key)\b',
        re.IGNORECASE
    )
    
    if tier1_re.match(name):
        return ("tier1", f"Sensitive filename: {name}")
    if SECRET_WORDS.search(stem):
        return ("tier1", f"Sensitive keyword in filename: {name}")
    if tier2_re.match(name):
        return ("tier2", f"Potentially sensitive: {name}")
    return ("ok", None)
```

#### Ενσωμάτωση στο Curator Pipeline

```python
# Στο Tauri backend (Rust) — καλείται ως Python subprocess:
# python3 ai_classifier.py --mode security_audit --dir ~/Downloads

# Στο Python script:
if __name__ == "__main__" and args.mode == "security_audit":
    findings = audit_sensitive_files(args.dir)
    result = format_audit_for_ui(findings)
    print(json.dumps(result))  # Tauri διαβάζει από stdout

# Στο React UI:
# - Badge στην κύρια σελίδα αν total > 0
# - SecurityAuditModal component για τα findings
# - Κουμπί "Run Deep Scan (gitleaks)" — opt-in, content-based
```

#### Privacy Guarantee

Το `audit_sensitive_files()` **ποτέ δεν ανοίγει αρχεία**. Ελέγχει μόνο:
1. Path prefixes (`~/.ssh`, `~/.aws`, κλπ.) — string `startswith()`
2. Filename patterns — precompiled regex
3. Keywords στο filename stem — regex

Το content-based scanning (gitleaks) είναι **εντελώς opt-in** — εκτελείται μόνο αν ο χρήστης πατήσει "Run Deep Scan". Αυτό τηρεί το privacy guarantee του Curator.

#### Notification Strategy (Χωρίς Code Signing)

Για Tauri app (unsigned Python script), δεν μπορούμε να χρησιμοποιήσουμε UNUserNotificationCenter. Εναλλακτικές:

```python
# Επιλογή 1: osascript (δουλεύει πάντα, χωρίς signing)
import subprocess

def notify_osascript(title: str, message: str) -> None:
    """macOS notification μέσω AppleScript — χωρίς code signing."""
    script = f'display notification "{message}" with title "{title}"'
    subprocess.run(["osascript", "-e", script], check=False)

# Χρήση:
notify_osascript(
    "Curator Security Audit",
    "Found 3 sensitive files in Downloads. Open Curator to review."
)

# Επιλογή 2: In-app badge (React UI) — προτεινόμενο
# Ο Rust backend ενημερώνει React state → badge εμφανίζεται
# Χωρίς macOS notification, χωρίς signing requirement
```

**Σύσταση:** Χρησιμοποίησε in-app badge (Επιλογή 2) ως primary UX. `osascript` notification ως secondary για background scans.

#### Πηγές

- gitleaks GitHub — https://github.com/gitleaks/gitleaks
- gitleaks dir command — https://gitleaks.io/
- gitleaks man page — https://linuxcommandlibrary.com/man/gitleaks
- gitleaks 2026 guide — https://appsecsanta.com/gitleaks
- Python osascript notification — https://gist.github.com/MarcAlx/f7a51a245c7c0450dbd4f4203b9916e0
- desktop-notifier GitHub (code signing requirement) — https://github.com/samschott/desktop-notifier
- desktop-notifier PyPI — https://pypi.org/project/desktop-notifier/
- macos-notifications GitHub — https://github.com/Jorricks/macos-notifications
- pync PyPI — https://pypi.org/project/pync/
- credential_scanner GitHub — https://github.com/vraj002/credential_scanner
- weSecretFinder Codeberg — https://codeberg.org/raginx/weSecretFinder
- GitGuardian Python scan guide — https://blog.gitguardian.com/scan-secrets/
- Medium: Build your own sensitive data scanner — https://medium.com/@bughunterrootkid/stop-leaking-secrets-build-your-own-sensitive-data-scanner-in-python-cb01571220e3

---

## Σύνοψη Αποφάσεων (ΜΕΡΟΣ 27)

| Feature | Απόφαση | Priority |
|---|---|---|
| `kMDItemWhereFroms` | osxmetadata library, soft hint (δεν είναι hard constraint) | Sprint 3 |
| Domain mapping | `*.ucy.ac.cy` → University, `github.com` → Code, `arxiv.org` → Papers | Sprint 3 |
| Finder Tags | post-processing must-links, `apply_tag_must_links()` | Sprint 3 |
| Git repos | embed as unit via `embed_git_repo()`, skip internal files | Sprint 2 |
| NER detection | dual pipeline (en + el), custom EntityRuler για course codes | Sprint 4 |
| NER signal fusion | alpha=0.8 (embedding) + 0.2 (entity similarity) | Sprint 4 |
| Sensitive audit | path-only by default, opt-in gitleaks για deep scan | Sprint 1 |
| Notifications | in-app badge πρώτα, osascript fallback | Sprint 1 |

## pip install Summary

```bash
# Νέα packages για ΜΕΡΟΣ 27 (υπόλοιπα ήδη εγκατεστημένα):
pip install gitpython          # Git repo metadata
pip install langdetect         # Language detection για NER routing
pip install thinc-apple-ops    # Apple Silicon acceleration για spaCy

# spaCy models:
python -m spacy download en_core_web_sm   # English NER
python -m spacy download el_core_news_lg  # Greek NER
python -m spacy download xx_ent_wiki_sm   # Multilingual (αν δουλέψει)

# Optional — ήδη στο system:
# xattr, plistlib (stdlib)
# osxmetadata (ήδη στο project)
# spacy (ήδη στο project)

# Gitleaks (Go binary, optional deep scan):
# brew install gitleaks
```