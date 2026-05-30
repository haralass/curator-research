## ΜΕΡΟΣ 38: CONTENT EXTRACTION PIPELINE
*(Έρευνα 2026-05-23)*

---

### ΘΕΜΑ 1: Ανά τύπο αρχείου — ποια library, τι fallback

---

#### PDF: Σύγκριση Libraries

**PyMuPDF (fitz) — η συνιστώμενη επιλογή**

Το PyMuPDF είναι αδιαμφισβήτητα η ταχύτερη Python library για text extraction από PDFs. Benchmarks 2025:

- ~18ms ανά σελίδα για plain text extraction
- 7.031-σέλιδο PDF → εξαγωγή σε ~5 δευτερόλεπτα
- 100–135× ταχύτερο από pdfminer
- 120–165× ταχύτερο από pdfplumber

Greek support: Το PyMuPDF χειρίζεται Greek Unicode μέσω CMAP (Character Map). Αν ένα PDF δεν έχει ενσωματωμένο CMAP για τους Greek χαρακτήρες, επιστρέφει `chr(0xFFFD)` (U+FFFD) αντί για γράμμα. Σε αυτή την περίπτωση, το PyMuPDF υποστηρίζει programmatic Tesseract OCR integration: δημιουργεί προσωρινό image boundary box και καλεί Tesseract για αναγνώριση. Η community έχει confirmed εξαγωγή Greek letters μέσω Discussion #1925 στο GitHub.

**MarkItDown (Microsoft) — για unified pipeline**

Το MarkItDown χρησιμοποιεί internally το `pdfminer` για PDF extraction (όχι PyMuPDF), οπότε είναι πιο αργό. Πλεονέκτημα: ένα API για PDF + DOCX + XLSX + PPTX + HTML + ZIP + Audio. Για PDF επιστρέφει plain text χωρίς heading levels ή layout preservation. Δεν έχει built-in OCR — χρειάζεται Azure Document Intelligence integration για scanned PDFs.

Εσωτερική αρχιτεκτονική (0.1.4, Δεκέμβριος 2025):
- DOCX → mammoth → HTML → BeautifulSoup → Markdown
- XLSX → pandas → Markdown
- PPTX → python-pptx → Markdown
- ZIP → iterative processing των περιεχομένων

**pdfplumber — για tables**

Ειδικεύεται σε table extraction από PDFs. Πολύ πιο αργό (2.99s/page) από PyMuPDF. Χρήσιμο ως fallback για PDFs με πολύπλοκους πίνακες που το PyMuPDF χάνει.

**Συμπέρασμα για PDF text extraction:**
```
Primary:  PyMuPDF (fitz) — ταχύτητα + Unicode/Greek support
Fallback: markitdown — αν fitz αποτύχει με Exception
Tables:   pdfplumber — αν ανιχνευτούν πολλοί πίνακες
```

---

#### Scanned PDFs: OCR

**pytesseract**
- Υποστηρίζει 100+ γλώσσες συμπεριλαμβανομένης της Ελληνικής (tessdata: `ell`)
- Απαιτεί preprocessed, high-resolution images για καλά αποτελέσματα
- Αδύναμο σε noisy images, χαμηλή ανάλυση, έντονα diacritics

**EasyOCR**
- Υποστηρίζει 80+ γλώσσες
- Καλύτερο σε low-quality images και handwritten text
- Πιο robust για noisy/real-world images
- Βαρύτερο model footprint από Tesseract

**Σύσταση για Curator (Greek scanned PDFs):**
Χρησιμοποίησε `easyocr` ως primary για scanned PDFs — καλύτερα αποτελέσματα σε πραγματικές συνθήκες (scanned documents συχνά δεν είναι τέλεια aligned). `pytesseract` ως fallback αν το EasyOCR είναι αργό.

```python
import easyocr
reader = easyocr.Reader(['el', 'en'], gpu=False)  # M2: MPS αντί GPU
result = reader.readtext(image_path, detail=0)
text = ' '.join(result)
```

---

#### DOCX / XLSX / PPTX

**markitdown — η ενιαία λύση**

Το markitdown ήδη εγκατεστημένο στο project. Υποστηρίζει και τους τρεις τύπους:

```python
from markitdown import MarkItDown
md = MarkItDown()
result = md.convert("document.docx")
print(result.text_content)  # Markdown string
```

Για DOCX: χρησιμοποιεί `mammoth` internally — διατηρεί headings, lists, tables.
Για XLSX: χρησιμοποιεί `pandas` internally — επιστρέφει Markdown table format.
Για PPTX: χρησιμοποιεί `python-pptx` internally — εξάγει slide text + speaker notes.

**python-docx vs markitdown για DOCX:**
Το `python-docx` δίνει χαμηλότερου επιπέδου πρόσβαση (paragraph objects, run objects, styles). Χρήσιμο αν χρειαστεί fine-grained control. Το markitdown είναι ταχύτερο για απλή text extraction.

**openpyxl για XLSX (fallback):**
```python
import openpyxl
wb = openpyxl.load_workbook(path, data_only=True)
parts = []
for sheet in wb.sheetnames:
    ws = wb[sheet]
    parts.append(f"Sheet: {sheet}")
    for row in ws.iter_rows(max_row=50, values_only=True):  # πρώτες 50 γραμμές
        parts.append(' | '.join(str(c) for c in row if c is not None))
return '\n'.join(parts)
```
Σημαντικό: `data_only=True` για να πάρεις computed values αντί για formulas.

---

#### Images: PNG / JPEG / HEIC

**Qwen2.5-VL:7b vs LLaVA:7b**

Benchmarks 2025 (Apple Silicon):
- **Qwen2.5-VL 7B**: Ανώτερο σε structured content (charts, tables, text-heavy images, screenshots). Κατά ~3× πιο αργό από LLaMA 3.2 σε complex tasks (~23s vs ~7s), αλλά πολύ καλύτερη ποιότητα.
- **LLaVA 7B**: Decent για general image description, γρηγορότερο.
- **moondream2**: Ο πιο γρήγορος VLM για edge devices. Σε CPU laptop: ~20-30s ανά image. Με Ollama σημαντικά ταχύτερο από raw transformers. Καλύτερο για simple captions, χειρότερο quality από Qwen2.5-VL.

**Σύσταση για Curator:**
```
Primary caption model:  moondream (ollama) — γρήγορο, αρκετό για indexing
Upgrade option:         qwen2.5vl:7b — για images με κείμενο (screenshots, docs)
Fallback (no Ollama):   EXIF metadata μόνο
```

**HEIC (iPhone photos):**
Χρησιμοποίησε `pillow-heif` — το πιο maintained library (2025):

```python
from pillow_heif import register_heif_opener
from PIL import Image
register_heif_opener()  # μία φορά στην αρχή
img = Image.open("photo.heic")  # λειτουργεί ακριβώς σαν JPEG
```

Υποστηρίζει: 8/10/12-bit HEIC, EXIF/XMP/IPTC read-write, thumbnails, depth images. Πλήρης integration με Pillow.

**EXIF extraction:**
Χρησιμοποίησε Pillow built-in για basic extraction, `piexif` για πλήρη read-write-delete:

```python
from PIL import Image, ExifTags
img = Image.open(path)
exif_raw = img.getexif()
exif = {ExifTags.TAGS.get(k, k): v for k, v in exif_raw.items()}
# Παίρνεις: Make, Model, DateTime, GPS coordinates, ImageDescription
```

Metadata που αξίζει για embedding: `DateTime`, `Make`, `Model`, `ImageDescription`, `GPSLatitude/Longitude`, `Artist`.

---

#### Code Files: Python, C, Java, Rust, JS/TS

**tree-sitter Python bindings**

```bash
pip install tree-sitter tree-sitter-python tree-sitter-javascript tree-sitter-c tree-sitter-java tree-sitter-rust
```

API (v0.25.2, September 2025):

```python
from tree_sitter import Language, Parser, Query
import tree_sitter_python as tspython

PY_LANGUAGE = Language(tspython.language())
parser = Parser(PY_LANGUAGE)

tree = parser.parse(source_bytes)
root = tree.root_node

# Query για function definitions + docstrings
query = Query(PY_LANGUAGE, """
    (function_definition name: (identifier) @func.name
        body: (block (expression_statement (string) @func.docstring)?))
    (import_statement) @import
    (import_from_statement) @import
""")
```

Υποστηριζόμενες γλώσσες (confirmed):
- Python (`tree-sitter-python`)
- JavaScript/TypeScript (`tree-sitter-javascript`, `tree-sitter-typescript`)
- C/C++ (`tree-sitter-c`, `tree-sitter-cpp`)
- Java (`tree-sitter-java`)
- Rust (`tree-sitter-rust`)
- 306 γλώσσες συνολικά μέσω `tree-sitter-language-pack`

**Strategy για code files:**
Εξαγωγή: module docstring + imports (πρώτες 30 γραμμές) + function/class names + function docstrings. Δεν χρειάζεται ολόκληρο το source code για embedding — το semantic summary αρκεί.

**Fallback για code:** Αν το tree-sitter δεν έχει grammar για τη γλώσσα → `open().read()[:3000]` (πρώτα 3000 χαρακτήρες).

---

#### ZIP / RAR / 7z

**Libraries:**
- ZIP: `zipfile` (stdlib) — δεν χρειάζεται pip install
- RAR: `rarfile` — απαιτεί `unrar` binary στο PATH
- 7z: `py7zr` — pure Python, δεν χρειάζεται external binary

**Strategy:**
```python
INFORMATIVE_EXTENSIONS = {'.py', '.js', '.ts', '.md', '.txt', '.rst',
                           '.pdf', '.docx', '.json', '.yaml', '.toml',
                           '.csv', '.html', '.readme'}

def extract_archive_content(path: str, size_bytes: int) -> str:
    if size_bytes > 100 * 1024 * 1024:  # > 100MB
        # List only, δεν εξάγουμε
        return list_archive_contents(path)

    # Εξαγωγή μόνο των informative αρχείων
    informative = [f for f in list_contents(path)
                   if Path(f).suffix.lower() in INFORMATIVE_EXTENSIONS]

    texts = []
    for fname in informative[:10]:  # max 10 αρχεία από archive
        content = extract_single_file(path, fname)
        texts.append(f"[{fname}]\n{content[:500]}")  # 500 chars per file

    return '\n\n'.join(texts) or f"Archive: {len(list_contents(path))} files"
```

---

#### Markdown / TXT / RST

Plain `open().read()` αρκεί για embedding purposes. Δεν χρειάζεται parsing με `mistune` ή `markdown-it-py` — το BGE-M3 καταλαβαίνει raw Markdown text. Μόνο αν χρειαστεί structured output (π.χ. headings μόνο) τότε parse.

```python
def extract_text_file(path: str) -> str:
    for encoding in ['utf-8', 'utf-8-sig', 'latin-1', 'cp1253']:  # cp1253 για Greek Windows
        try:
            with open(path, 'r', encoding=encoding) as f:
                return f.read()[:10000]  # cap στους 10k chars
        except UnicodeDecodeError:
            continue
    return ""  # fallback αν όλα αποτύχουν
```

---

#### Audio: MP3 / M4A / WAV

**Επιλογές:**

1. **Skip + filename only** — default για Curator (πιο γρήγορο)
2. **faster-whisper local transcription** — αν ο χρήστης θέλει searchable audio

faster-whisper benchmarks σε M2 Pro (2025):
- 60s audio → ~2.8s με Large-v3-turbo + Metal + Flash Attention
- ~15-30× real-time speed on Apple Silicon
- `pywhispercpp` — Python bindings για whisper.cpp (PyPI: `pywhispercpp`)

**Σύσταση:** Για Curator v1, skip audio content, embed filename + duration. Προσθήκη transcription ως optional feature αργότερα.

```python
import subprocess, json

def get_audio_duration(path: str) -> float:
    result = subprocess.run(
        ['ffprobe', '-v', 'quiet', '-print_format', 'json', '-show_format', path],
        capture_output=True, text=True
    )
    data = json.loads(result.stdout)
    return float(data['format']['duration'])
```

---

#### Video: MP4 / MOV / MKV

Εντελώς skip content extraction — transcription και frame analysis είναι πολύ αργά για batch processing 5000 αρχείων.

Embed: filename + duration + file size + codec info (από ffprobe).

```python
def extract_video_metadata(path: str) -> str:
    result = subprocess.run(
        ['ffprobe', '-v', 'quiet', '-print_format', 'json',
         '-show_format', '-show_streams', path],
        capture_output=True, text=True, timeout=10
    )
    data = json.loads(result.stdout)
    fmt = data.get('format', {})
    duration = float(fmt.get('duration', 0))
    size_mb = int(fmt.get('size', 0)) / 1024 / 1024
    name = Path(path).stem.replace('_', ' ').replace('-', ' ')
    return f"Video: {name} | Duration: {duration:.0f}s | Size: {size_mb:.1f}MB"
```

---

### Extraction Decision Table

| Extension | Library | Strategy | Fallback | Skip Condition | Εκτ. χρόνος/αρχείο |
|-----------|---------|----------|----------|----------------|---------------------|
| `.pdf` (text) | PyMuPDF | full text extract | markitdown | >500MB | ~0.2-0.5s |
| `.pdf` (scanned) | EasyOCR | OCR per page | pytesseract | >100 pages | ~3-8s |
| `.docx` | markitdown | full text | python-docx | >50MB | ~0.3s |
| `.xlsx` | markitdown | cells as table | openpyxl (50 rows) | >50MB | ~0.4s |
| `.pptx` | markitdown | slide text + notes | python-pptx | >50MB | ~0.4s |
| `.png`/`.jpg`/`.jpeg` | pillow + moondream | EXIF + caption | EXIF only | >20MB (raw) | ~1-3s |
| `.heic` | pillow-heif + moondream | convert + caption | EXIF only | - | ~2-4s |
| `.py`/`.js`/`.ts` | tree-sitter | imports + docstrings | read[:3000] | >1MB | ~0.05s |
| `.c`/`.cpp`/`.java`/`.rs` | tree-sitter | function names + comments | read[:3000] | >1MB | ~0.05s |
| `.zip` | zipfile | list + extract informative | list only | >100MB | ~0.5-2s |
| `.rar` | rarfile | list + extract informative | list only | >100MB | ~0.5-2s |
| `.7z` | py7zr | list + extract informative | list only | >100MB | ~0.5-2s |
| `.md`/`.txt`/`.rst` | open() | read[:10000] | - | - | ~0.01s |
| `.html` | markitdown | convert to text | bs4 | - | ~0.1s |
| `.csv`/`.json` | open() / json.load() | read/parse[:5000] | - | >10MB | ~0.05s |
| `.mp3`/`.m4a`/`.wav` | ffprobe | filename + duration | skip | - | ~0.1s |
| `.mp4`/`.mov`/`.mkv` | ffprobe | filename + duration + codec | skip | - | ~0.1s |
| `.DS_Store`/`.localized` | — | SKIP ENTIRELY | — | always | — |
| `.crdownload`/`.part` | — | SKIP ENTIRELY | — | always | — |

---

### ΘΕΜΑ 2: Performance — Χρόνος για 5000 αρχεία

#### Benchmarks (2025 research)

- **PyMuPDF**: ~18ms/page, ~0.2-0.5s per standard PDF
- **markitdown (DOCX/XLSX/PPTX)**: ~0.3-0.5s per file
- **EasyOCR scanned PDF**: ~3-8s per page (βαρύ)
- **moondream (Ollama)**: ~2-3s per image σε CPU, γρηγορότερο με Ollama Metal
- **qwen2.5vl (Ollama)**: ~5-10s per image σε M2 Pro
- **tree-sitter code**: <0.05s per file
- **BGE-M3 embedding**: ~0.1-0.3s per document (fastembed)
- **faster-whisper audio**: ~0.05× real-time (60s audio → ~3s)

#### Εκτίμηση Χρόνου για 5000 Αρχεία

| Τύπος | Αρχεία (εκτ.) | Χρόνος/αρχείο | Σύνολο |
|-------|--------------|---------------|--------|
| PDF (digital text, 1-10 pages) | 700 | ~0.3s | ~3.5 min |
| PDF (scanned, per page ~5s) | 100 | ~10s avg | ~17 min |
| DOCX/XLSX/PPTX | 400 | ~0.4s | ~2.7 min |
| Images PNG/JPEG (moondream) | 1200 | ~2.5s | ~50 min |
| Images HEIC (iPhone) | 400 | ~3s | ~20 min |
| Code files (.py, .js, .ts) | 500 | ~0.05s | ~0.4 min |
| Text/Markdown/CSV/JSON | 800 | ~0.02s | ~0.3 min |
| ZIP/RAR/7z | 100 | ~1s | ~1.7 min |
| Audio/Video (metadata only) | 200 | ~0.1s | ~0.3 min |
| Misc / skip | 600 | ~0s | ~0 min |
| **BGE-M3 embedding (όλα)** | **5000** | **~0.2s** | **~17 min** |
| **ΣΥΝΟΛΟ (sequential)** | | | **~112 min** |
| **ΣΥΝΟΛΟ (parallel x4)** | | | **~28-35 min** |

#### Bottleneck Identification

1. **Images με VLM (Ollama)** → ~70 min sequential. Αυτός είναι ο κυρίαρχος bottleneck.
2. **Scanned PDFs (EasyOCR)** → ~17 min για 100 files — αναλογικά το πιο αργό per-file.
3. **BGE-M3 embedding** → ~17 min — σταθερό κόστος για όλα τα αρχεία.

**Στρατηγική parallelization:**
- Image captioning: `OLLAMA_NUM_PARALLEL=2` σε M2 Pro (16GB) — 2 concurrent Ollama requests
- Text extraction (PDF/DOCX/code): `ProcessPoolExecutor(max_workers=4)` — CPU-bound
- Embedding: batch mode με fastembed — στέλνει πολλά texts μαζί

---

### ΘΕΜΑ 3: Error Handling & Skip Logic

#### Complete Extraction Function

```python
from dataclasses import dataclass
from pathlib import Path
from typing import Optional
import time
import logging

logger = logging.getLogger(__name__)

SKIP_EXTENSIONS = {
    '.ds_store', '.localized', '.crdownload', '.part',
    '.tmp', '.swp', '.lock', '.pid', '.swo',
    '.gitkeep', '.gitignore'  # tiny config files
}

SIZE_LIMIT_BYTES = 500 * 1024 * 1024   # 500MB — skip content
ARCHIVE_SIZE_LIMIT = 100 * 1024 * 1024  # 100MB — list only

@dataclass
class ExtractedContent:
    text: str                    # extracted text (μπορεί κενό)
    method: str                  # ποια library χρησιμοποιήθηκε
    quality: float               # 0.0 - 1.0 εκτίμηση ποιότητας
    fallback_used: bool          # αν χρησιμοποιήθηκε fallback
    skip_reason: Optional[str]   # λόγος skip αν εφαρμόζεται
    extraction_time_ms: float    # milliseconds που πήρε


def extract_content(file_path: str) -> ExtractedContent:
    """
    Central extraction dispatcher για Curator.
    Προσπαθεί έως 3 φορές πριν αποτύχει gracefully.
    """
    path = Path(file_path)
    start = time.perf_counter()

    # 1. Skip list check
    if path.suffix.lower() in SKIP_EXTENSIONS:
        return ExtractedContent(
            text="", method="skip", quality=0.0,
            fallback_used=False,
            skip_reason=f"extension in SKIP_LIST: {path.suffix}",
            extraction_time_ms=0.0
        )

    # 2. Existence check
    if not path.exists():
        return ExtractedContent(
            text="", method="skip", quality=0.0,
            fallback_used=False, skip_reason="file_not_found",
            extraction_time_ms=0.0
        )

    file_size = path.stat().st_size

    # 3. Size limit — content skip, filename only
    if file_size > SIZE_LIMIT_BYTES:
        return ExtractedContent(
            text=f"[Large file: {path.name}, {file_size // 1024 // 1024}MB]",
            method="filename_only", quality=0.1,
            fallback_used=False, skip_reason="file_too_large",
            extraction_time_ms=elapsed_ms(start)
        )

    ext = path.suffix.lower()
    last_error = None
    fallback_used = False

    # 4. Retry loop (max 3 attempts)
    for attempt in range(3):
        try:
            result = _dispatch_extraction(path, ext, file_size, fallback_used)
            result.extraction_time_ms = elapsed_ms(start)
            return result
        except Exception as e:
            last_error = e
            logger.warning(f"Extraction attempt {attempt+1}/3 failed for {path.name}: {e}")
            if attempt == 0:
                fallback_used = True  # δεύτερη προσπάθεια = fallback

    # 5. Αποτυχία μετά από 3 φορές
    logger.error(f"All extraction attempts failed for {path.name}: {last_error}")
    return ExtractedContent(
        text=f"[Extraction failed: {path.name}]",
        method="extraction_failed", quality=0.0,
        fallback_used=True,
        skip_reason=f"extraction_failed: {str(last_error)[:100]}",
        extraction_time_ms=elapsed_ms(start)
    )


def _dispatch_extraction(path: Path, ext: str, size: int,
                          use_fallback: bool) -> ExtractedContent:
    """Routes to the correct extractor based on extension."""

    # PDF
    if ext == '.pdf':
        return _extract_pdf(path, use_fallback)

    # Office formats
    if ext in {'.docx', '.xlsx', '.xls', '.pptx', '.ppt'}:
        return _extract_office(path, ext, use_fallback)

    # Images
    if ext in {'.png', '.jpg', '.jpeg', '.gif', '.bmp', '.tiff', '.webp'}:
        return _extract_image(path, use_fallback)

    if ext == '.heic':
        return _extract_heic(path, use_fallback)

    # Code
    if ext in {'.py', '.js', '.ts', '.jsx', '.tsx', '.c', '.cpp',
               '.h', '.java', '.rs', '.go', '.rb', '.swift', '.kt'}:
        return _extract_code(path, ext, use_fallback)

    # Archives
    if ext in {'.zip', '.rar', '.7z', '.tar', '.gz', '.bz2'}:
        return _extract_archive(path, ext, size)

    # Text-based
    if ext in {'.txt', '.md', '.rst', '.log', '.csv', '.json',
               '.yaml', '.yml', '.toml', '.xml', '.html', '.htm'}:
        return _extract_text(path, ext)

    # Audio
    if ext in {'.mp3', '.m4a', '.wav', '.flac', '.aac', '.ogg'}:
        return _extract_audio_metadata(path)

    # Video
    if ext in {'.mp4', '.mov', '.mkv', '.avi', '.wmv', '.m4v'}:
        return _extract_video_metadata(path)

    # Unknown binary — filename only
    return ExtractedContent(
        text=f"[Unknown format: {path.name}]",
        method="filename_only", quality=0.05,
        fallback_used=False, skip_reason=None,
        extraction_time_ms=0.0
    )


def elapsed_ms(start: float) -> float:
    return (time.perf_counter() - start) * 1000
```

---

#### PDF Extractor (με PyMuPDF + fallbacks)

```python
import fitz  # PyMuPDF

def _extract_pdf(path: Path, use_fallback: bool) -> ExtractedContent:
    if use_fallback:
        # Fallback: markitdown
        try:
            from markitdown import MarkItDown
            md = MarkItDown()
            result = md.convert(str(path))
            return ExtractedContent(
                text=result.text_content[:8000], method="markitdown",
                quality=0.5, fallback_used=True, skip_reason=None,
                extraction_time_ms=0.0
            )
        except Exception:
            return ExtractedContent(
                text=f"[PDF extraction failed: {path.name}]",
                method="extraction_failed", quality=0.0,
                fallback_used=True, skip_reason="both_extractors_failed",
                extraction_time_ms=0.0
            )

    # Primary: PyMuPDF
    doc = fitz.open(str(path))
    texts = []
    is_repaired = doc.is_repaired

    for page in doc:
        page_text = page.get_text("text")
        # Ανίχνευση scanned page (ελάχιστο κείμενο αλλά υπάρχουν images)
        if len(page_text.strip()) < 50 and len(page.get_images()) > 0:
            # Scanned page — OCR needed
            texts.append(_ocr_page(page))
        else:
            texts.append(page_text)

    doc.close()
    full_text = '\n'.join(texts)[:10000]
    quality = 0.9 if not is_repaired else 0.6

    return ExtractedContent(
        text=full_text, method="pymupdf",
        quality=quality, fallback_used=False, skip_reason=None,
        extraction_time_ms=0.0
    )


def _ocr_page(page) -> str:
    """OCR ενός scanned PDF page μέσω EasyOCR."""
    try:
        import easyocr
        import numpy as np
        # Render page σε image
        mat = fitz.Matrix(2, 2)  # 2x zoom για καλύτερο OCR
        pix = page.get_pixmap(matrix=mat)
        img_array = np.frombuffer(pix.samples, dtype=np.uint8).reshape(
            pix.height, pix.width, pix.n
        )
        reader = easyocr.Reader(['el', 'en'], gpu=False)
        return ' '.join(reader.readtext(img_array, detail=0))
    except Exception:
        return ""
```

---

### ΘΕΜΑ 4: Parallelization Strategy

#### CPU-bound vs IO-bound — Διαφορετική Στρατηγική

**CPU-bound tasks** (text extraction από αρχεία):
- PDF parsing, code parsing, DOCX/XLSX
- Χρησιμοποίησε `ProcessPoolExecutor` — παρακάμπτει το Python GIL
- `max_workers=4` σε M2 Pro (6 performance cores, αλλά αφήνουμε 2 για UI/OS)

**IO-bound tasks** (Ollama API calls για image captioning):
- Κάθε request κάθεται και περιμένει HTTP response
- Χρησιμοποίησε `asyncio` + `aiohttp` ή `ThreadPoolExecutor`
- `OLLAMA_NUM_PARALLEL=2` σε M2 Pro 16GB (κάθε slot +15-25% VRAM για 7B model)
- M2 Pro 16GB: base model ~4.5GB (Q4_K_M) + 2 slots → ~5.5GB — ασφαλές

**SQLite cache:** Αν αρχείο έχει ίδιο `mtime` + `size` → skip re-extraction, φόρτωσε cached embedding.

#### Parallel Extraction Pipeline

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor, as_completed
from pathlib import Path
import sqlite3
import hashlib
from typing import List

# Cache check
def is_cached(db_conn: sqlite3.Connection, path: str,
              mtime: float, size: int) -> bool:
    row = db_conn.execute(
        "SELECT 1 FROM extractions WHERE path=? AND mtime=? AND size=?",
        (path, mtime, size)
    ).fetchone()
    return row is not None


def get_file_signature(path: str):
    stat = Path(path).stat()
    return stat.st_mtime, stat.st_size


# Ταξινόμηση αρχείων ανά τύπο για βέλτιστη ουρά
def classify_files(paths: List[str]):
    cpu_tasks = []    # PDF, DOCX, code — ProcessPoolExecutor
    io_tasks = []     # images — Ollama API (async)
    fast_tasks = []   # TXT, JSON, code < 100KB — serial (πολύ γρήγορα)

    for p in paths:
        ext = Path(p).suffix.lower()
        size = Path(p).stat().st_size if Path(p).exists() else 0
        if ext in {'.png', '.jpg', '.jpeg', '.heic', '.gif'}:
            io_tasks.append(p)
        elif ext in {'.txt', '.md', '.json', '.csv'} and size < 50000:
            fast_tasks.append(p)
        else:
            cpu_tasks.append(p)

    return cpu_tasks, io_tasks, fast_tasks


# CPU-bound parallel extraction
def run_cpu_extraction(paths: List[str], db_path: str) -> dict:
    results = {}
    db = sqlite3.connect(db_path)

    # Φιλτράρισμα cached
    uncached = []
    for p in paths:
        mtime, size = get_file_signature(p)
        if not is_cached(db, p, mtime, size):
            uncached.append(p)
        else:
            results[p] = "CACHED"

    db.close()

    with ProcessPoolExecutor(max_workers=4) as executor:
        future_to_path = {executor.submit(extract_content, p): p
                          for p in uncached}
        for future in as_completed(future_to_path):
            path = future_to_path[future]
            try:
                results[path] = future.result()
            except Exception as e:
                results[path] = ExtractedContent(
                    text=f"[Error: {e}]", method="error",
                    quality=0.0, fallback_used=True,
                    skip_reason=str(e), extraction_time_ms=0.0
                )
    return results


# IO-bound async extraction (Ollama image captioning)
async def caption_image_async(session, path: str,
                               semaphore: asyncio.Semaphore) -> tuple:
    async with semaphore:  # max 2 concurrent Ollama requests
        import base64
        from PIL import Image
        import io

        img = Image.open(path).convert("RGB")
        img.thumbnail((512, 512))  # resize για ταχύτητα
        buf = io.BytesIO()
        img.save(buf, format="JPEG")
        img_b64 = base64.b64encode(buf.getvalue()).decode()

        payload = {
            "model": "moondream",
            "prompt": "Describe this image briefly in 1-2 sentences.",
            "images": [img_b64],
            "stream": False
        }
        async with session.post("http://localhost:11434/api/generate",
                                 json=payload) as resp:
            data = await resp.json()
            return path, data.get("response", "")


async def run_image_captioning(paths: List[str], max_parallel: int = 2) -> dict:
    import aiohttp
    semaphore = asyncio.Semaphore(max_parallel)
    results = {}

    async with aiohttp.ClientSession() as session:
        tasks = [caption_image_async(session, p, semaphore) for p in paths]
        for coro in asyncio.as_completed(tasks):
            path, caption = await coro
            results[path] = caption

    return results


# Main pipeline orchestrator
def run_extraction_pipeline(all_paths: List[str],
                             db_path: str = "curator_cache.db") -> dict:
    cpu_tasks, io_tasks, fast_tasks = classify_files(all_paths)

    # Fast tasks — serial (overhead δεν αξίζει)
    fast_results = {p: extract_content(p) for p in fast_tasks}

    # CPU tasks — ProcessPoolExecutor
    cpu_results = run_cpu_extraction(cpu_tasks, db_path)

    # IO tasks — asyncio
    io_results = asyncio.run(run_image_captioning(io_tasks))

    return {**fast_results, **cpu_results, **io_results}
```

---

### Συνοπτικές Αποφάσεις για Curator

| Κατηγορία | Επιλογή | Λόγος |
|-----------|---------|-------|
| PDF primary | PyMuPDF | 100× ταχύτερο, Greek Unicode support |
| PDF OCR | EasyOCR | Καλύτερο σε noisy/real-world scans |
| Office (DOCX/XLSX/PPTX) | markitdown | Ήδη εγκατεστημένο, ένα API για όλα |
| Image caption | moondream (Ollama) | Γρηγορότερο, αρκετό για indexing |
| HEIC | pillow-heif | Maintained, πλήρης Pillow integration |
| Code parsing | tree-sitter | Semantic extraction, 306 γλώσσες |
| Archives | zipfile/rarfile/py7zr | Ανά extension, selective extraction |
| Audio/Video | ffprobe only | Content skip — metadata αρκεί για v1 |
| Parallelization | ProcessPool (CPU) + asyncio (IO) | Βέλτιστο για το mixed workload |
| Cache | SQLite (mtime+size) | Αποφυγή re-extraction |
| Error handling | 3 retries → graceful fail | Δεν σταματά το pipeline |

**Εκτιμώμενος χρόνος πρώτης πλήρους indexing 5000 αρχείων:** ~28-35 λεπτά με parallelization × 4 (bottleneck: image captioning). Κάθε επόμενο run (incremental): ~2-5 λεπτά για νέα/αλλαγμένα αρχεία.

---

*Πηγές: PyMuPDF docs, markitdown GitHub, tree-sitter/py-tree-sitter, pillow-heif PyPI, pdfmux benchmarks 2025, Ollama FAQ, various benchmarks (2025-2026)*