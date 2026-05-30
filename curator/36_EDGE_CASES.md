## ΜΕΡΟΣ 36: EDGE CASES & REAL-WORLD GOTCHAS
*(Έρευνα 2026-05-23)*

---

### ΘΕΜΑ 1: Embedding Quality Edge Cases — Πότε το BGE-M3 αποτυγχάνει

#### Γενικό πρόβλημα: Short text embeddings

Το BGE-M3 είναι εξαιρετικό σε long documents (έως 8192 tokens), αλλά σε πολύ σύντομα κείμενα (filenames, single-line metadata) τα embeddings συχνά **συγκλίνουν σε παρόμοια regions** του latent space ανεξάρτητα από το semantic content. Η έρευνα επιβεβαιώνει ότι το clustering σε short text είναι δυσκολότερο λόγω χαμηλής word co-occurrence. Η cosine similarity μεταξύ `solution.py` και `homework.py` είναι σχεδόν ίδια — και τα δύο είναι απλώς generic filenames.

---

#### Case 1: Screenshots με uninformative filename

**Πρόβλημα:** `Screenshot 2026-01-15 at 14.23.45.png` — το filename δεν έχει semantic content. Το BGE-M3 embed του filename μόνο αποδίδει ένα generic "timestamp string" embedding, που θα cluster με άλλα timestamps, όχι με το actual content.

**Τι κάνει το LLaVA caption:** Η έρευνα δείχνει ότι τα MLLMs (LLaVA) αποτυγχάνουν ~2.6% των φορών με captions όπως "no text", "other objects", ή επαναλαμβανόμενο boilerplate. Για screenshots of PDFs ή terminal output, το LLaVA τείνει να δίνει generic caption: "A screenshot of a document with text" — αχρείαστο για clustering.

**Concrete fix:**
```python
def handle_screenshot(filepath: Path) -> dict:
    """
    Multi-signal approach για screenshots.
    """
    signals = {}

    # Signal 1: LLaVA caption (primary)
    caption = llava_caption(filepath)
    if is_generic_caption(caption):  # π.χ. "screenshot of text"
        # Fallback: OCR the screenshot
        ocr_text = tesseract_ocr(filepath)
        if len(ocr_text.strip()) > 50:
            caption = ocr_text[:512]  # Use OCR text as embedding input

    # Signal 2: EXIF / metadata
    exif = get_exif(filepath)
    # Screenshots macOS δεν έχουν camera EXIF — αυτό είναι διαγνωστικό
    is_screenshot = (
        "Software" in exif and "Screenshot" in exif.get("Software", "")
        or exif.get("Make") is None  # Καμία κάμερα = πιθανό screenshot
    )

    # Signal 3: Pixel dimensions
    # Screenshots macOS: 1440x900, 2560x1600, 3456x2234 (Retina)
    # Φωτογραφίες iPhone: 4032x3024
    img = Image.open(filepath)
    w, h = img.size
    aspect = w / h
    signals["likely_screenshot"] = is_screenshot or (1.5 < aspect < 2.0)

    return {"embedding_text": caption, "signals": signals}

def is_generic_caption(caption: str) -> bool:
    GENERIC_PHRASES = [
        "screenshot of", "image of text", "no text", "other objects",
        "a screenshot", "document with text"
    ]
    return any(p in caption.lower() for p in GENERIC_PHRASES)
```

**Gotcha:** Αν το screenshot είναι από dark mode terminal, το LLaVA συχνά fails εντελώς (hallucination). Fallback: embed filename timestamp + "screenshot" tag.

---

#### Case 2: Blank / scanned PDFs χωρίς extractable text

**Πρόβλημα:** Scanned PDFs (π.χ. σημειώσεις καθηγητή σκαναρισμένες) δεν έχουν text layer. `pdfplumber` / `PyMuPDF` επιστρέφουν empty string. Το BGE-M3 embed του "" ή του filename μόνο βγάζει near-zero vector ή random noise στο embedding space — μπορεί να πάει σε οποιοδήποτε cluster.

**Concrete fix — 3-tier fallback:**
```python
def extract_pdf_text(filepath: Path) -> tuple[str, str]:
    """
    Returns (text, extraction_method)
    """
    import fitz  # PyMuPDF
    doc = fitz.open(filepath)
    text = " ".join(page.get_text() for page in doc)

    if len(text.strip()) > 100:
        return text[:4096], "native"

    # Tier 2: OCR με pytesseract
    try:
        from pdf2image import convert_from_path
        images = convert_from_path(filepath, dpi=200, first_page=1, last_page=3)
        ocr_parts = [pytesseract.image_to_string(img, lang="ell+eng") for img in images]
        ocr_text = " ".join(ocr_parts)
        if len(ocr_text.strip()) > 50:
            return ocr_text[:4096], "ocr"
    except Exception:
        pass

    # Tier 3: Filename + metadata only (πολύ weak signal)
    meta = doc.metadata
    fallback = f"{filepath.stem} {meta.get('title', '')} {meta.get('subject', '')}"
    return fallback.strip(), "filename_only"
```

**Αν η method είναι `filename_only`:** Δες το file size. PDF 20MB με blank text = σχεδόν σίγουρα scanned. Tag it ως `needs_manual_review` αντί να το αφήσεις να μπει σε λάθος cluster.

---

#### Case 3: Single-page forms (ελάχιστο κείμενο)

**Πρόβλημα:** Ένα PDF form με πεδία "Όνομα", "Ημερομηνία", "Υπογραφή" έχει ~20 words. Το embedding θα μοιάζει με εκατοντάδες άλλα forms και administrative documents.

**Heuristic rule που δουλεύει:**
```python
def classify_form_pdf(text: str, page_count: int, filepath: Path) -> str:
    FORM_INDICATORS = [
        "υπογραφή", "signature", "ημερομηνία", "date:", "name:", "όνομα:",
        "□", "☐", "_______", "print name", "αρ. μητρώου"
    ]
    form_score = sum(1 for ind in FORM_INDICATORS if ind.lower() in text.lower())
    words = len(text.split())

    if page_count == 1 and words < 100 and form_score >= 2:
        return "administrative_form"
    return None  # Δεν είναι form
```

---

#### Case 4: Code files με minimal comments

**Πρόβλημα:** `solution.py` με 10 γραμμές Python και 0 docstrings. Το BGE-M3 βλέπει: `def solve(n): return n*2` — δεν μπορεί να ξέρει αν είναι EPL236 ή EPL325 assignment.

**Fix — χρήση import statements ως proxy:**
```python
def augment_code_embedding(filepath: Path, code_text: str) -> str:
    """
    Augment με imports και structure signals.
    """
    import ast
    try:
        tree = ast.parse(code_text)
    except SyntaxError:
        return code_text  # Fallback

    imports = [
        node.names[0].name for node in ast.walk(tree)
        if isinstance(node, (ast.Import, ast.ImportFrom))
    ]
    # numpy/scipy = numerical → math/physics course
    # socket/threading = networking → EPL325/EPL426
    # cryptography/hashlib = security → EPL326
    augmented = f"IMPORTS: {', '.join(imports)}\n\n{code_text}"
    return augmented[:2048]
```

**Gotcha:** Αν δεν υπάρχουν imports (π.χ. C file), χρησιμοποίησε `#include` statements ως equivalent signals.

---

#### Case 5: Ελληνικά scanned PDFs — BGE-M3 και OCR errors

**Πρόβλημα:** Το Tesseract OCR σε ελληνικά scanned documents κάνει χαρακτηριστικά λάθη: `"αλγόριθμος"` → `"αλγοριθμος"` (accent loss), `"φ"` → `"ψ"` σε κακή scan quality. Το BGE-M3 είναι multilingual και robust σε minor typos, αλλά αν > 30% των λέξεων είναι wrong, το embedding degraded.

**Fix:**
- Χρησιμοποίησε `pytesseract` με `lang="ell+eng"` και `--oem 3 --psm 6`
- Post-process: Greek spell checker (`pyenchant` με `el_GR` dictionary) για fix obvious errors
- Αν confidence < 60% (από tesseract confidence score), tag ως low-confidence embedding

---

#### Case 6: ZIP files — mixed content

**Πρόβλημα:** `EPL236_assignment2.zip` περιέχει `report.pdf` + `solution.c` + `Makefile` + `test_output.png`. Τι embed κάνουμε για το ZIP;

**Στρατηγική — "representative sampling":**
```python
def embed_zip(filepath: Path) -> str:
    """
    Embed ZIP ως summary των περιεχομένων.
    """
    import zipfile
    with zipfile.ZipFile(filepath, 'r') as z:
        names = z.namelist()

    # Classify contents by extension
    exts = Counter(Path(n).suffix.lower() for n in names)
    dominant = exts.most_common(3)

    # Try to read the most informative file (PDF or .txt README)
    text_content = ""
    with zipfile.ZipFile(filepath, 'r') as z:
        for name in names:
            if name.endswith('.pdf'):
                # Extract text from PDF inside ZIP (in-memory)
                with z.open(name) as f:
                    pdf_bytes = f.read()
                text_content = extract_pdf_from_bytes(pdf_bytes)[:1024]
                break
            elif name.lower() in ('readme.txt', 'readme.md', 'report.txt'):
                with z.open(name) as f:
                    text_content = f.read().decode('utf-8', errors='ignore')[:1024]
                break

    summary = f"ZIP archive: {filepath.stem}. Contents: {dict(dominant)}. {text_content}"
    return summary
```

**Gotcha:** Μην κάνεις full extraction σε temp directory για κάθε ZIP — disk I/O overhead. Χρησιμοποίησε in-memory reads.

---

#### Case 7: Junk files — .DS_Store, Thumbs.db, .localized

**Αυτά πρέπει να αγνοούνται εντελώς — ποτέ μην τα embed, ποτέ μην τα μετακινείς:**

```python
ALWAYS_IGNORE = {
    ".DS_Store", "Thumbs.db", ".localized", "desktop.ini",
    ".Spotlight-V100", ".fseventsd", ".Trashes",
    ".DocumentRevisions-V100",
}

IGNORE_EXTENSIONS = {".crdownload", ".part", ".tmp", ".download"}

def should_skip(filepath: Path) -> bool:
    if filepath.name in ALWAYS_IGNORE:
        return True
    if filepath.suffix.lower() in IGNORE_EXTENSIONS:
        return True
    if filepath.name.startswith("._"):  # macOS resource forks
        return True
    if filepath.stat().st_size == 0:   # Empty files
        return True
    return False
```

---

### ΘΕΜΑ 2: Cluster Collisions — Όταν 2 Clusters Μοιάζουν Πολύ

#### Case 1: EPL326 Security vs EPL436 Network Security

**Πρόβλημα:** Και τα δύο courses έχουν PDFs με λέξεις: "encryption", "protocol", "attack", "vulnerability". Το HDBSCAN τα βάζει στο ίδιο cluster με probability > 0.8.

**Disambiguation strategy:**
```python
def disambiguate_security_courses(cluster_docs: list[dict]) -> list[str]:
    """
    Χρησιμοποίησε distinguishing keywords για split.
    """
    EPL326_MARKERS = ["cryptography", "cipher", "PKI", "TLS", "hash function",
                      "symmetric", "asymmetric", "EPL326"]
    EPL436_MARKERS = ["routing", "BGP", "TCP/IP", "socket", "bandwidth",
                      "network layer", "EPL436", "OSI model"]

    for doc in cluster_docs:
        text = doc["text"].lower()
        epl326_score = sum(1 for m in EPL326_MARKERS if m.lower() in text)
        epl436_score = sum(1 for m in EPL436_MARKERS if m.lower() in text)

        if epl326_score > epl436_score + 2:
            doc["suggested_label"] = "EPL326_Security"
        elif epl436_score > epl326_score + 2:
            doc["suggested_label"] = "EPL436_NetworkSecurity"
        else:
            doc["suggested_label"] = "AMBIGUOUS_security"  # Χρειάζεται user review
```

**Γιατί HDBSCAN δεν λύνει αυτό μόνο του:** Το `cluster_selection_epsilon` αν τεθεί χαμηλά, τα κάνει 2 clusters — αλλά κάθε lecture PDF κλπ. θα ανήκει σε διαφορετικό micro-cluster. Χρειάζεσαι post-clustering semantic filtering.

---

#### Case 2: Thesis PDFs vs Research Papers (general)

**Πρόβλημα:** Και τα δύο έχουν Abstract, Introduction, References. Το embedding space τα βάζει μαζί.

**Heuristic fix — structural signals:**
```python
def is_thesis(text: str, filename: str) -> bool:
    THESIS_MARKERS = [
        "university of cyprus", "πανεπιστήμιο κύπρου", "υποβλήθηκε",
        "submitted in partial fulfillment", "supervisor:", "επιβλέπων:",
        "department of", "tmhma", "acknowledgments", "ευχαριστίες",
        "chapter 1", "chapter 2",  # Thesis έχει chapters, papers sections
    ]
    name_markers = ["thesis", "dissertation", "ptychiakih", "πτυχιακή"]

    text_lower = text[:3000].lower()
    filename_lower = filename.lower()

    score = sum(1 for m in THESIS_MARKERS if m in text_lower)
    score += sum(2 for m in name_markers if m in filename_lower)

    return score >= 3
```

---

#### Case 3: Personal photos vs Screenshots

**Κλειδί: EXIF data, όχι embedding!**

```python
from PIL import Image
from PIL.ExifTags import TAGS

def classify_image_type(filepath: Path) -> str:
    try:
        img = Image.open(filepath)
        exif_raw = img._getexif()
    except Exception:
        exif_raw = None

    if exif_raw:
        exif = {TAGS.get(k, k): v for k, v in exif_raw.items()}
        # Φωτογραφία iPhone/camera: έχει Make, Model, GPS
        if exif.get("Make") in ("Apple", "Samsung", "Canon", "Nikon", "Sony"):
            return "personal_photo"
        if "GPSInfo" in exif:
            return "personal_photo"
        if exif.get("Software", "").startswith("Photos"):
            return "personal_photo"

    # Χωρίς EXIF ή generic software: πιθανό screenshot
    # Επιπλέον check: aspect ratio
    w, h = img.size
    # macOS screenshots: πάντα screen aspect (16:9, 16:10)
    # iPhone photos: 4:3
    aspect = w / h
    if 1.3 < aspect < 2.1 and not exif_raw:
        return "likely_screenshot"

    return "unknown_image"
```

**Gotcha:** AirDrop φωτογραφίες από iPhone **διατηρούν** το EXIF. Αλλά screenshots που έγιναν share μέσω AirDrop **χάνουν** κάποια EXIF fields. Έλεγξε για `Software: "Screenshot"` tag.

---

#### Case 4: Multiple versions ίδιου project

**Πρόβλημα:** `report_v1.pdf`, `report_draft.pdf`, `report_final.pdf`, `report_FINAL2.pdf` — θα cluster μαζί (σωστά!), αλλά ο Curator πρέπει να τα βάλει σε versions folder, όχι σε 4 ξεχωριστά folders.

**Version detection pattern:**
```python
import re
from difflib import SequenceMatcher

def detect_version_group(filepaths: list[Path]) -> list[list[Path]]:
    """
    Grouping βάσει base name similarity.
    """
    VERSION_PATTERNS = [
        r"_v\d+", r"_V\d+", r"_draft", r"_final", r"_FINAL",
        r"_copy", r"\(\d+\)", r"_rev\d*", r"_backup"
    ]

    def base_name(p: Path) -> str:
        name = p.stem
        for pattern in VERSION_PATTERNS:
            name = re.sub(pattern, "", name, flags=re.IGNORECASE)
        return name.strip("_- ")

    groups = {}
    for fp in filepaths:
        base = base_name(fp)
        if base not in groups:
            groups[base] = []
        groups[base].append(fp)

    # Μόνο groups με > 1 file είναι version groups
    return [g for g in groups.values() if len(g) > 1]
```

**Policy για version groups:** Δημιούργησε `report_versions/` subfolder, βάλε όλα μέσα. Ποτέ μην διαγράφεις το "παλιό" version.

---

#### HDBSCAN: Πότε 2 clusters merge αυτόματα

Το HDBSCAN χρησιμοποιεί **Excess of Mass (EOM)** cluster selection by default. Δύο clusters merge όταν το combined cluster έχει higher stability score από τα children ξεχωριστά. Πρακτικά:

- Αν τα 2 clusters έχουν **< 15 points** το καθένα και βρίσκονται κοντά, merge είναι πιθανό
- `cluster_selection_epsilon=0.3` (σε cosine space): clusters με distance < 0.3 δεν χωρίζονται
- Για course separation: χρησιμοποίησε `cluster_selection_method='leaf'` για finer granularity

```python
import hdbscan

clusterer = hdbscan.HDBSCAN(
    min_cluster_size=10,
    min_samples=5,
    cluster_selection_epsilon=0.25,  # Cosine space: τιμή μετά από testing
    cluster_selection_method='leaf',  # Finer clusters
    metric='euclidean',              # UMAP output είναι Euclidean
    prediction_data=True,            # Για soft assignments
)
```

---

### ΘΕΜΑ 3: Cold Start — Τα Πρώτα 30 Λεπτά

#### Προβλήματα κατά το cold start

**HDBSCAN:** Deterministic με ίδια data/params — δεν υπάρχει instability εδώ. Αλλά χωρίς historical context, τα clusters αντικατοπτρίζουν μόνο τη **σημερινή** κατανομή. Αν ο φοιτητής έχει 300 EPL236 files και 5 EPL425 files, το EPL425 θα πάει στο noise (-1).

**UMAP:** Είναι **στοχαστικό**. Χωρίς `random_state`, κάθε run δίνει διαφορετική projection. Multithreading κάνει τα αποτελέσματα ακόμα λιγότερο reproducible. Σε 5000 documents: ο UMAP fitted για πρώτη φορά μπορεί να πάρει 2-5 λεπτά.

```python
# CRITICAL: Πάντα set random_state για reproducibility
from umap import UMAP

umap_model = UMAP(
    n_components=5,         # Όχι 2D — 5D για clustering quality
    n_neighbors=15,
    min_dist=0.0,           # 0.0 για better cluster separation
    metric="cosine",
    random_state=42,        # ΑΠΑΡΑΙΤΗΤΟ για reproducibility
    low_memory=True,        # Για 5000+ documents
    n_jobs=1,               # Single thread = deterministic
)
```

**Gotcha:** Ακόμα και με `random_state=42`, το UMAP σε multi-threaded mode (default) δεν είναι fully reproducible λόγω race conditions. Χρησιμοποίησε `n_jobs=1`.

---

#### Cold Start Checklist

```
COLD START PROTOCOL (5000 αρχεία, πρώτη εκτέλεση):

PRE-FLIGHT:
□ 1. Τρέξε should_skip() για κάθε file — αφαίρεσε .DS_Store, empties, partials
□ 2. Έλεγξε για symlinks: os.path.islink() — skip αν circular
□ 3. Έλεγξε disk space: χρειάζεσαι ~500MB temp για embedding cache
□ 4. Εκτίμηση χρόνου: 5000 files × ~0.1s/file = ~8 min embedding

EMBEDDING PHASE:
□ 5. Batch embed σε groups των 64 (BGE-M3 optimal batch size)
□ 6. Cache embeddings σε SQLite (filepath + mtime hash → vector)
□ 7. Σήμανε files με extraction_method: "native"/"ocr"/"filename_only"
□ 8. Files με "filename_only": separate pool — μην τα βάλεις σε main cluster

CLUSTERING PHASE (conservative settings για cold start):
□ 9.  UMAP με n_jobs=1, random_state=42
□ 10. HDBSCAN με min_cluster_size=15 (αντί 10) — λιγότερα αλλά cleaner clusters
□ 11. Υπολόγισε DBCV score — αν < 0.3, warn χρήστη
□ 12. Κράτα μόνο clusters με >= 5 documents
□ 13. Documents σε noise cluster (-1) → "Unsorted/" folder

USER REVIEW PHASE (ΠΡΙΝ από οποιαδήποτε κίνηση):
□ 14. Παρουσίασε clusters ως "proposed folders" — user βλέπει, δεν γίνεται τίποτα
□ 15. Δείξε top-3 representative files ανά cluster
□ 16. Ζήτα confirmation για κάθε cluster ή "approve all"
□ 17. Version groups: δείξε ξεχωριστά — "Found 4 versions of 'report'"

POST-APPROVAL:
□ 18. Δημιούργησε folders (mkdir) — μόνο μετά από approval
□ 19. Copy αντί move (shutil.copy2) — verify copy, μετά delete original
□ 20. Refit UMAP με confirmed assignments (transform(), όχι fit_transform())
□ 21. Αποθήκευσε cluster centroids για incremental updates
□ 22. Log όλες τις κινήσεις σε JSON (undo support)
```

---

### ΘΕΜΑ 4: Catastrophic Cases — Τι ΔΕΝ Πρέπει Ποτέ να Γίνει

#### Never-Do List

**1. Ποτέ μην κινείς αρχεία χωρίς approval — ακόμα και 99% confidence**

Η confidence score δεν είναι ασφάλεια. Ένα `assignment_final.pdf` με 99% confidence EPL326 μπορεί να είναι από διαφορετικό project. Ο χρήστης είναι ο μόνος ground truth.

**2. Ποτέ μην διαγράφεις — μόνο move σε `_antilibrary/`**

```python
# ΛΑΘΟΣ:
os.remove(filepath)

# ΣΩΣΤΟ:
antilibrary = Path.home() / "Downloads" / "_antilibrary"
antilibrary.mkdir(exist_ok=True)
dest = antilibrary / filepath.name
if dest.exists():
    dest = antilibrary / f"{filepath.stem}__{int(time.time())}{filepath.suffix}"
shutil.move(str(filepath), str(dest))
```

**3. Ποτέ μην αγγίζεις system/hidden directories**

```python
UNTOUCHABLE_DIRS = {
    ".git", ".obsidian", ".venv", "__pycache__", "node_modules",
    ".Trash", "Library", ".ssh", ".gnupg", ".config",
    "Contents",  # macOS .app bundles
}

UNTOUCHABLE_EXTENSIONS = {
    ".app", ".framework", ".dylib", ".so", ".kext",
}

def is_atomic_directory(path: Path) -> bool:
    """Directories που δεν πρέπει ποτέ να 'σπάσουν'."""
    return (
        path.name in UNTOUCHABLE_DIRS
        or path.suffix in UNTOUCHABLE_EXTENSIONS
        or path.name.startswith(".")
    )
```

**4. Ποτέ μην κάνεις overwrite — rename με suffix**

```python
def safe_move(src: Path, dest_dir: Path) -> Path:
    dest = dest_dir / src.name
    if dest.exists():
        # Έλεγξε αν είναι identical (SHA256)
        if sha256(src) == sha256(dest):
            src.unlink()  # Duplicate — safe να αφαιρεθεί source
            return dest
        # Διαφορετικό content — rename
        timestamp = int(time.time())
        dest = dest_dir / f"{src.stem}__{timestamp}{src.suffix}"
    shutil.copy2(str(src), str(dest))  # Copy πρώτα
    src.unlink()  # Delete source μόνο μετά επιτυχή copy
    return dest
```

**5. Ποτέ μην τρέχεις σε ~/Library ή system paths**

```python
FORBIDDEN_ROOTS = [
    Path.home() / "Library",
    Path("/System"),
    Path("/usr"),
    Path("/private"),
    Path("/Applications"),
]

def is_safe_to_scan(path: Path) -> bool:
    for forbidden in FORBIDDEN_ROOTS:
        try:
            path.relative_to(forbidden)
            return False  # path είναι inside forbidden
        except ValueError:
            continue
    return True
```

#### Real-world disaster patterns (από community observations)

- **Hazel / file organizer scripts:** Reports ότι regex-based rules έστειλαν `.DS_Store` files σε "Documents" folder επειδή το rule ήταν "move all files to correct folder" χωρίς exclusions.
- **Overwrite bug:** Organizer έστελνε `report.pdf` από EPL236 AND EPL325 στο ίδιο folder → το δεύτερο overwrite το πρώτο silently.
- **Obsidian vault disaster:** Script που έψαχνε για `.md` files σε Downloads και τα έστελνε στο vault — έσπασε internal links (Obsidian χρησιμοποιεί relative paths).

---

### ΘΕΜΑ 5: Performance Edge Cases

#### Case 1: Μεγάλα αρχεία (500MB+ video)

```python
SIZE_LIMITS = {
    ".mp4": 50 * 1024 * 1024,   # Skip videos > 50MB
    ".mov": 50 * 1024 * 1024,
    ".zip": 500 * 1024 * 1024,  # Skip ZIPs > 500MB (πιθανόν installer)
    "default": 100 * 1024 * 1024,
}

def get_embedding_strategy(filepath: Path) -> str:
    size = filepath.stat().st_size
    ext = filepath.suffix.lower()
    limit = SIZE_LIMITS.get(ext, SIZE_LIMITS["default"])

    if size > limit:
        return "filename_only"  # Embed μόνο filename + extension

    if ext in (".mp4", ".mov", ".avi", ".mkv"):
        return "skip"  # Videos: μην embed, βάλε σε "Videos/" by default
```

#### Case 2: Burst downloads (100 screenshots σε 1 λεπτό)

**Πρόβλημα:** DenStream flooding — 100 νέα points έρχονται γρήγορα, το online clustering τα βάζει όλα σε outlier micro-clusters αντί να ενσωματωθούν σε existing clusters.

**Fix — debounce mechanism:**
```python
import asyncio
from collections import deque

class BurstDebouncer:
    def __init__(self, window_seconds=60, batch_threshold=20):
        self.window = window_seconds
        self.threshold = batch_threshold
        self.pending = deque()

    async def add_file(self, filepath: Path):
        self.pending.append((filepath, time.time()))
        self._cleanup_old()

        if len(self.pending) >= self.threshold:
            # Burst detected: batch process αντί online
            await self._batch_process()
        else:
            # Normal: schedule online update σε 5 sec
            await asyncio.sleep(5)
            if filepath in [p for p, _ in self.pending]:
                await self._online_update(filepath)

    def _cleanup_old(self):
        now = time.time()
        while self.pending and now - self.pending[0][1] > self.window:
            self.pending.popleft()
```

#### Case 3: Symlinks — circular loop protection

```python
def safe_walk(root: Path):
    """
    os.walk με symlink protection.
    """
    visited_inodes = set()

    for dirpath, dirnames, filenames in os.walk(root, followlinks=False):
        # followlinks=False: δεν ακολουθεί symlinks σε dirs
        # Αλλά τα symlinked FILES εξακολουθούν να εμφανίζονται
        for filename in filenames:
            filepath = Path(dirpath) / filename
            if filepath.is_symlink():
                real = filepath.resolve()
                inode = real.stat().st_ino if real.exists() else None
                if inode in visited_inodes:
                    continue  # Circular symlink — skip
                if inode:
                    visited_inodes.add(inode)
            yield Path(dirpath) / filename
```

#### Case 4: Files που αλλάζουν κατά τη scan

```python
def stable_file_hash(filepath: Path, wait_ms=500) -> str | None:
    """
    Διάβασε hash δύο φορές — αν διαφέρουν, το file αλλάζει.
    """
    try:
        h1 = sha256_fast(filepath)
        time.sleep(wait_ms / 1000)
        h2 = sha256_fast(filepath)
        if h1 != h2:
            return None  # File σε χρήση / αλλάζει — skip
        return h1
    except (PermissionError, FileNotFoundError):
        return None
```

#### Case 5: Locked files (PermissionError)

```python
def try_read_file(filepath: Path) -> tuple[bytes | None, str]:
    try:
        with open(filepath, 'rb') as f:
            return f.read(65536), "ok"  # Πρώτα 64KB
    except PermissionError:
        return None, "locked"
    except OSError as e:
        return None, f"error:{e.errno}"
```

**Locked files policy:** Log them, retry after 60 seconds once. Αν εξακολουθεί να είναι locked, skip και mark ως `pending_retry`.

#### Case 6: Very long paths (macOS PATH_MAX = 1024)

```python
def check_path_safety(src: Path, dest_dir: Path) -> Path | None:
    """
    Έλεγξε αν το destination path θα υπερβεί το macOS limit.
    macOS PATH_MAX = 1024 characters (enforced by OS, not just APFS)
    """
    proposed = dest_dir / src.name
    if len(str(proposed)) > 900:  # Conservative limit (900 < 1024)
        # Truncate filename (όχι extension)
        max_stem = 900 - len(str(dest_dir)) - len(src.suffix) - 1
        if max_stem < 10:
            return None  # Path too deep — cannot move
        truncated = src.stem[:max_stem] + src.suffix
        proposed = dest_dir / truncated

    return proposed
```

---

### ΘΕΜΑ 6: macOS Version Compatibility

#### Υποστηριζόμενες εκδόσεις

| macOS | Version | Status Curator |
|-------|---------|----------------|
| Sequoia | 15.x | Primary target — tested |
| Sonoma | 14.x | Supported — minor API diffs |
| Ventura | 13.x | Supported — older FSEvents API |
| Monterey | 12.x | Not recommended — Python 3.11 issues |

#### kMDItemWhereFroms format

Το `kMDItemWhereFroms` είναι ένα **binary plist (BPLIST)** array που αποθηκεύεται ως extended attribute. Το format είναι **σταθερό από macOS 10.15+** — δεν άλλαξε σε Ventura/Sonoma/Sequoia. Τυπικά περιέχει:
- `[0]`: URL από όπου κατέβηκε το file
- `[1]`: Referrer URL (σελίδα που ξεκίνησε το download)

```python
import xattr
import plistlib

def get_download_source(filepath: Path) -> str | None:
    try:
        raw = xattr.getxattr(str(filepath), "com.apple.metadata:kMDItemWhereFroms")
        urls = plistlib.loads(raw)
        return urls[0] if urls else None
    except (OSError, KeyError, plistlib.InvalidFileException):
        return None

# Χρήση για classification:
# URL contains "mail.google.com" / "ucy.ac.cy" → email attachment
# URL contains "youtube.com" / "googlevideo.com" → YouTube download
# URL contains "github.com" → code/project
# URL contains "learn.ucy.ac.cy" / "blackboard" → course material
```

**Gotcha:** AirDrop files **δεν έχουν** `kMDItemWhereFroms`. Screenshots επίσης όχι. Μόνο browser/email downloads το έχουν.

#### TCC Permissions — Τι χρειάζεται ο Curator

**Minimum required:**
```
Privacy & Security → Files and Folders → Downloads Folder: ✓
```

Αυτό αρκεί για read/write στο `~/Downloads`. ΔΕΝ χρειάζεσαι Full Disk Access.

**Αν χρησιμοποιείς FSEvents για monitoring:**
```
Privacy & Security → Automation: ✓ (μόνο αν χρησιμοποιείς AppleScript/osascript)
```

FSEvents API δεν χρειάζεται ξεχωριστό permission — καλύπτεται από Files and Folders access.

**Gotcha — macOS Sequoia 15:**
Η Apple πρόσθεσε stricter entitlement checks σε Sequoia. Αν το Curator τρέχει ως unsigned script (python3 direct), μπορεί να εμφανιστεί permission dialog **ανά directory** την πρώτη φορά. Λύση: codesign το script ή χρησιμοποίησε `.app bundle` με proper entitlements.

```xml
<!-- entitlements.plist για codesign -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" ...>
<plist version="1.0">
<dict>
    <key>com.apple.security.files.downloads.read-write</key>
    <true/>
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
</dict>
</plist>
```

**Unsandboxed apps (π.χ. Terminal.app) inheriting permissions:**
Αν ο Curator τρέχει μέσα από Terminal και το Terminal έχει Full Disk Access, τότε ο Curator κληρονομεί αυτά τα permissions. Αυτό είναι βολικό σε development αλλά **δεν πρέπει να είναι η production approach**.

---

### Σύνοψη: Priority Matrix για Implementation

| Edge Case | Πιθανότητα | Impact | Priority |
|-----------|-----------|--------|----------|
| Scanned PDF (blank text) | Υψηλή (UCY παλιά notes) | Λάθος cluster | P0 |
| Screenshots clustering | Πολύ υψηλή | Poor UX | P0 |
| Overwrite χωρίς check | Σπάνιο | Catastrophic | P0 |
| Symlink infinite loop | Σπάνιο | Crash/hang | P0 |
| Circular version files | Υψηλή | Duplicate work | P1 |
| Course collision (EPL326/436) | Μέτρια | Wrong folder | P1 |
| Cold start noise cluster | Βέβαιο | Incomplete org | P1 |
| Burst screenshots | Μέτρια | Flooding | P2 |
| Locked files | Μέτρια | Silent skip | P2 |
| TCC permission dialog | Υψηλή σε Sequoia | UX friction | P2 |
| PATH_MAX overflow | Σπάνιο | OSError crash | P3 |

---

*Πηγές: HDBSCAN docs, UMAP reproducibility docs, Eclectic Light Company (macOS TCC), arXiv short-text clustering surveys, PyPI osxmetadata, scikit-learn HDBSCAN contrib, Apple Security docs.*

---
