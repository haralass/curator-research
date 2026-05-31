# 60 — PDF Text Extraction Pipeline: PyMuPDF, Greek Encoding, and OCR Fallback
_Research date: 2026-05-31_

---

## Why PDF Extraction Is Non-Trivial

PDF is a presentation format, not a content format. Text is stored as glyph-positioning commands, not as a logical stream of characters. This creates three problem classes:

1. **Digital PDFs with embedded fonts**: Text is present but may be encoded through a font-specific CMAP (character map) that translates glyph IDs to Unicode. If the CMAP is missing, incomplete, or uses a custom encoding (common in older Greek-language typesetters), extracted text is empty, garbled, or produces mojibake.

2. **Scanned PDFs**: These are images — no text layer exists. OCR is mandatory.

3. **Hybrid PDFs**: Scanned PDF with a hidden OCR text layer added by Acrobat or a scanner. The text layer may be low quality, misaligned, or in the wrong encoding.

For Greek specifically, the problem is compounded. Greek academic publishing in Cyprus and Greece heavily used pre-Unicode tools (PageMaker, Ventura, older InDesign) with custom Type1 fonts (GFS fonts, old Monotype Greek faces) whose CMAPs either don't exist or map to Windows-1253 or ISO-8859-7 rather than Unicode. PDFs generated from these workflows in the 1990s–2010s are common in university libraries, ministry documents, and digitised books — exactly the files a Cypriot student organizer will encounter. A single extraction strategy will fail on a significant minority of real-world Greek PDFs. A cascade pipeline is mandatory.

---

## PyMuPDF (fitz): Primary Extractor

### Installation
```bash
pip install pymupdf   # installs as "fitz"
```

### Core API
```python
import fitz  # PyMuPDF

doc = fitz.open("document.pdf")

for page in doc:
    text = page.get_text("text")          # basic extraction
    text = page.get_text("text", sort=True)  # sorted blocks: critical for multi-column
    text = page.get_text("blocks")        # list of (x0,y0,x1,y1,text,block_no,block_type)
    text = page.get_text("rawdict")       # raw glyph IDs — use to debug CMAP issues

print(doc.page_count)
print(doc.metadata)   # title, author, creator, producer, dates
doc.close()
```

### Performance
~10–20ms/page for text extraction on modern hardware. Compares to:
- pdfminer.six: ~1,500–2,000ms/page (**100–135× slower**)
- pdfplumber: ~500–800ms/page (built on pdfminer internally)
- Tesseract (300 DPI): ~2,000–8,000ms/page

For a 200-page thesis: PyMuPDF ~3.6 seconds. pdfminer ~5–7 minutes. The difference is decisive for a file organizer indexing hundreds of documents.

### Greek Unicode Handling — What Works

Modern tools (LaTeX/XeLaTeX, modern Word, modern InDesign) with properly embedded Unicode fonts: PyMuPDF extracts Greek text correctly. The Unicode codepoints in the PDF's content stream are valid and PyMuPDF passes them through faithfully.

### Greek Unicode Handling — What Fails (CMAP Issues)

PyMuPDF relies entirely on the PDF's own CMAP table. It does not attempt to guess encodings. When the CMAP is wrong or missing, three failure modes:

1. **Empty string**: Font has no ToUnicode CMAP entry → `""` for the page
2. **Latin lookalikes**: Greek Type1 fonts where `/Alpha`, `/Beta` were mapped to Latin `/A`, `/B`
3. **Mojibake**: Windows-1253 or ISO-8859-7 content treated as Latin-1 → `Ç½Î¿Î¹ ÏÎ¿Ï`

The `"rawdict"` extraction mode exposes raw glyph IDs to inspect whether Unicode mapping happened:
```python
rawdict = page.get_text("rawdict")
for block in rawdict["blocks"]:
    for line in block.get("lines", []):
        for span in line.get("spans", []):
            chars = [ch["c"] for ch in span.get("chars", [])]
            # If chars are in Private Use Area or pure ASCII when Greek is expected → CMAP broken
```

---

## Greek CMAP Issues: Root Cause and Detection

### Root Cause

Pre-Unicode Greek publishing used Type1 fonts where the encoding vector mapped character codes to glyph names. A typical example:
```
/Encoding [ ... /Alpha /Beta /Gamma /Delta /Epsilon ... ]
```
Many older tools omitted the ToUnicode CMAP, or generated one that mapped `/Alpha` → `A` (Latin A) instead of `U+0391` (Greek Capital Letter Alpha).

**Common culprits in Greek/Cypriot documents:**
- Documents typeset with **Ventura Publisher** using `Apla`, `Optima Greek`, or custom Type1 faces
- Older **Microsoft Word** with Greek fonts, saved via Acrobat Distiller (pre-2007)
- **Ministry of Education Cyprus** official documents from the 2000s–2010s (PDF/A-1b with Type1 Greek)
- **Greek university repositories** (Pergamos, PSEPHEDA) where theses were scanned and OCR'd with imperfect CMAPs
- **Polytonic Greek**: pre-Unicode polytonic fonts (GreekKeys, SPIonic, Vusillus) almost always have broken/absent CMAPs

### Detection Heuristics

```python
import math
from collections import Counter

def char_entropy(text: str) -> float:
    if len(text) < 2:
        return 0.0
    counts = Counter(text)
    total = len(text)
    return -sum((c/total) * math.log2(c/total) for c in counts.values())

def needs_ocr(page, expected_language: str = "el") -> bool:
    text = page.get_text("text")
    
    # No text at all — definitely scanned
    if len(text.strip()) < 10:
        return True
    
    # Replacement characters — CMAP broken
    if text.count('�') / len(text) > 0.02:
        return True
    
    # Expected Greek but got mostly non-Greek
    if expected_language == "el":
        alpha = sum(1 for c in text if c.isalpha())
        greek = sum(1 for c in text if 'Ͱ' <= c <= 'Ͽ' or 'ἀ' <= c <= '῿')
        if alpha > 50 and (greek / alpha) < 0.15:
            # Verify page actually has glyphs (not blank page)
            rawdict = page.get_text("rawdict")
            glyph_count = sum(
                len(span.get("chars", []))
                for block in rawdict.get("blocks", [])
                for line in block.get("lines", [])
                for span in line.get("spans", [])
            )
            if glyph_count > 50:
                return True
    
    # Entropy anomaly
    entropy = char_entropy(text[:1000])
    if entropy < 1.5 or entropy > 7.5:
        return True
    
    return False
```

**Threshold summary:**

| Metric | Good | Bad → OCR |
|---|---|---|
| Replacement char ratio | < 0.01 | > 0.02 |
| Coverage ratio (chars/glyphs) | > 0.7 | < 0.3 |
| Printable ratio | > 0.95 | < 0.85 |
| Char entropy | 2.5–6.0 | < 1.5 or > 7.5 |
| Greek ratio (when Greek expected) | > 0.4 | < 0.15 |

---

## pdfminer.six: When to Prefer It

```bash
pip install pdfminer.six
```

### When pdfminer.six beats PyMuPDF

1. **Broken CMAP repair**: pdfminer has more aggressive fallback strategies for missing ToUnicode entries, attempting to use the font's Encoding dictionary and glyph name database. For some Greek Type1 fonts, this produces readable text where PyMuPDF returns empty or garbled output.

2. **Complex layout analysis**: LAParams gives fine-grained control over text grouping. For newsletters, multi-column academic papers, or government forms, tuning `boxes_flow` and `char_margin` can produce cleaner output.

3. **Debugging**: pdfminer exposes every intermediate object (LTChar, LTAnon, LTTextBox) — essential for diagnosing CMAP issues.

```python
from pdfminer.high_level import extract_text
from pdfminer.layout import LAParams

# For two-column academic paper (common in Greek journals)
laparams = LAParams(
    line_overlap=0.5,
    char_margin=2.0,
    line_margin=0.5,
    word_margin=0.1,
    boxes_flow=0.3,    # slightly favour horizontal grouping
    detect_vertical=False,
)
text = extract_text("document.pdf", laparams=laparams)

# For newspaper-style layout (many columns, small text)
laparams_newspaper = LAParams(
    boxes_flow=-0.5,   # favour column-first reading order
    char_margin=1.0,
    line_margin=0.2
)
```

**Limitation:** 100–135× slower than PyMuPDF. Use only as fallback, not primary extractor.

---

## pdfplumber: Table Extraction

```bash
pip install pdfplumber
```

```python
import pdfplumber

with pdfplumber.open("document.pdf") as pdf:
    for page in pdf.pages:
        tables = page.extract_tables()
        for table in tables:
            for row in table:
                print(row)  # list of strings or None
        
        # For tables requiring tuning:
        table = page.extract_table({
            "vertical_strategy": "lines",
            "horizontal_strategy": "lines",
            "snap_tolerance": 3,
        })
```

**Use pdfplumber surgically** — only when `page.find_tables()` returns non-empty. For plain-text extraction, it's slower than PyMuPDF with no benefit for Greek encoding. ~500–800ms/page for table extraction.

---

## Tesseract OCR: Greek Language Fallback

### macOS Installation
```bash
brew install tesseract
brew install tesseract-lang   # installs ALL language packs including Greek (ell)
tesseract --list-langs | grep ell  # verify
```

Language data location:
- Apple Silicon: `/opt/homebrew/share/tessdata/`
- Intel Mac: `/usr/local/share/tessdata/`

```bash
pip install pytesseract Pillow pdf2image
brew install poppler  # required by pdf2image
```

### pytesseract API for Greek PDFs
```python
import pytesseract
from pdf2image import convert_from_path

def ocr_pdf_page(pdf_path: str, page_number: int, lang: str = "ell+eng") -> str:
    images = convert_from_path(
        pdf_path,
        first_page=page_number,
        last_page=page_number,
        dpi=300,   # minimum 300 DPI for acceptable quality
        fmt="PNG"
    )
    if not images:
        return ""
    return pytesseract.image_to_string(
        images[0],
        lang=lang,
        config="--psm 3 --oem 3"  # PSM 3: auto page segmentation; OEM 3: LSTM+legacy
    )
```

**Language configuration:**
```python
lang = "ell+eng"   # mixed Greek/English — most Cypriot academic work
lang = "ell"       # pure Greek (government documents, classical texts)
# Polytonic Greek: Tesseract 4 ell includes polytonic, but quality is lower
```

**Performance:** 2–8 seconds/page at 300 DPI. For Curator, process pages in batches and skip OCR for documents > 50 pages.

**Accuracy expectations:**
- Modern Greek (post-1982 monotonic): 92–97% character accuracy ✅
- Polytonic Greek: 80–90% ⚠️
- Katharevousa (formal archaic): 70–85% ⚠️
- Mixed Greek/English: good with `ell+eng` ✅
- Handwritten Greek: poor — Tesseract is not a handwriting recognizer ❌

---

## MarkItDown: Unified Fallback

```bash
pip install markitdown
```

MarkItDown's PDF support uses **pdfminer.six** under the hood — it adds no extraction capability beyond pdfminer, only wraps output in Markdown. For Greek PDFs it inherits all pdfminer strengths and limitations.

**Where MarkItDown IS useful for Curator:** handling other document types in a unified way — DOCX, PPTX, XLSX, HTML. Use it as the fallback converter for non-PDF formats, not as a PDF specialist.

---

## Complete Extraction Pipeline for Curator

```
PDF input
    │
    ├── is_scanned_pdf()? ──YES──► Tesseract OCR (ell+eng) ──► done
    │
    └── PyMuPDF extract ──► assess_quality()
              │
              ├── "good" ──► has tables? ──YES──► pdfplumber table extraction ──► merge ──► done
              │                           └──NO──► PyMuPDF text ──► done
              │
              ├── "garbled" / "sparse"
              │         ├── try pdfminer.six ──► quality "good"? ──► done
              │         └── still bad ──► Tesseract OCR ──► quality "good"? ──► done
              │                                                        └── last resort: filename only
              │
              └── "empty" ──► Tesseract OCR ──► done
```

### Core Implementation

```python
import fitz
import platform
from pdfminer.high_level import extract_text as pdfminer_extract
from pdfminer.layout import LAParams
import pdfplumber
import pytesseract
from pdf2image import convert_from_path
import math, logging
from collections import Counter
from dataclasses import dataclass
from pathlib import Path

logger = logging.getLogger(__name__)

@dataclass
class ExtractionResult:
    text: str
    method: str      # "pymupdf" | "pdfminer" | "tesseract" | "filename_only"
    quality: str     # "good" | "garbled" | "empty"
    confidence: float
    has_tables: bool
    page_count: int


def extract_pdf_text(
    pdf_path: str | Path,
    expected_language: str = "el",  # "el", "en", "mixed"
    max_ocr_pages: int = 50,
) -> ExtractionResult:
    pdf_path = str(pdf_path)
    try:
        doc = fitz.open(pdf_path)
    except Exception as e:
        logger.error(f"Cannot open {pdf_path}: {e}")
        return ExtractionResult("", "filename_only", "empty", 0.0, False, 0)

    page_count = doc.page_count

    # Step 1: Detect scanned PDF
    if _is_scanned(doc):
        doc.close()
        text = _ocr(pdf_path, expected_language, max_ocr_pages)
        q, c = _assess(text, expected_language)
        return ExtractionResult(text, "tesseract", q, c, False, page_count)

    # Step 2: PyMuPDF
    pymupdf_text, has_tables = _pymupdf_extract(doc)
    doc.close()
    q, c = _assess(pymupdf_text, expected_language)

    if q == "good":
        if has_tables:
            table_text = _pdfplumber_tables(pdf_path)
            if table_text:
                pymupdf_text += "\n\n--- Tables ---\n\n" + table_text
        return ExtractionResult(pymupdf_text, "pymupdf", q, c, has_tables, page_count)

    # Step 3: pdfminer fallback
    pm_text = _pdfminer_extract(pdf_path)
    pm_q, pm_c = _assess(pm_text, expected_language)
    if pm_q == "good" and pm_c > c:
        return ExtractionResult(pm_text, "pdfminer", pm_q, pm_c, False, page_count)

    # Step 4: OCR fallback
    ocr_text = _ocr(pdf_path, expected_language, max_ocr_pages)
    ocr_q, ocr_c = _assess(ocr_text, expected_language)
    if ocr_q == "good":
        return ExtractionResult(ocr_text, "tesseract", ocr_q, ocr_c, False, page_count)

    # Step 5: Return best available
    best = max(
        [(pymupdf_text, "pymupdf", q, c),
         (pm_text, "pdfminer", pm_q, pm_c),
         (ocr_text, "tesseract", ocr_q, ocr_c)],
        key=lambda x: x[3]
    )
    return ExtractionResult(best[0], best[1], best[2], best[3], False, page_count)


def _is_scanned(doc: fitz.Document) -> bool:
    pages = min(3, len(doc))
    empty = 0
    for i in range(pages):
        page = doc[i]
        if len(page.get_text("text").strip()) < 20 and page.get_images():
            empty += 1
    return empty >= min(2, pages)


def _pymupdf_extract(doc: fitz.Document) -> tuple[str, bool]:
    texts, has_tables = [], False
    for page in doc:
        text = page.get_text("text", sort=True)
        texts.append(text)
        if '\t' in text:
            has_tables = True
    return "\n\n".join(texts), has_tables


def _pdfminer_extract(pdf_path: str) -> str:
    try:
        laparams = LAParams(
            char_margin=2.0, line_margin=0.5,
            word_margin=0.1, boxes_flow=0.5
        )
        return pdfminer_extract(pdf_path, laparams=laparams) or ""
    except Exception as e:
        logger.error(f"pdfminer failed: {e}")
        return ""


def _pdfplumber_tables(pdf_path: str) -> str:
    try:
        parts = []
        with pdfplumber.open(pdf_path) as pdf:
            for page in pdf.pages:
                for table in page.extract_tables():
                    if not table:
                        continue
                    rows = []
                    for i, row in enumerate(table):
                        cells = [str(c or "").strip() for c in row]
                        rows.append("| " + " | ".join(cells) + " |")
                        if i == 0:
                            rows.append("|" + "|".join(["---"] * len(row)) + "|")
                    parts.append("\n".join(rows))
        return "\n\n".join(parts)
    except Exception as e:
        logger.error(f"pdfplumber failed: {e}")
        return ""


def _ocr(pdf_path: str, lang: str, max_pages: int) -> str:
    lang_map = {"el": "ell+eng", "en": "eng", "mixed": "ell+eng"}
    tess_lang = lang_map.get(lang, "ell+eng")
    try:
        images = convert_from_path(pdf_path, dpi=300, fmt="PNG", last_page=max_pages)
        return "\n\n".join(
            pytesseract.image_to_string(img, lang=tess_lang, config="--psm 3 --oem 3")
            for img in images
        )
    except Exception as e:
        logger.error(f"OCR failed: {e}")
        return ""


def _assess(text: str, expected_language: str = "el") -> tuple[str, float]:
    if not text or len(text.strip()) < 10:
        return "empty", 1.0
    t = text.strip()
    if t.count('�') / len(t) > 0.02:
        return "garbled", 0.9
    printable = sum(1 for c in t if c.isprintable() or c in '\n\t')
    if printable / len(t) < 0.85:
        return "garbled", 0.85
    if expected_language in ("el", "mixed"):
        alpha = sum(1 for c in t if c.isalpha())
        greek = sum(1 for c in t if 'Ͱ' <= c <= 'Ͽ' or 'ἀ' <= c <= '῿')
        if alpha > 20 and greek / alpha < 0.15:
            return "garbled", 0.8
    counts = Counter(t[:2000])
    total = min(2000, len(t))
    entropy = -sum((c/total) * math.log2(c/total) for c in counts.values())
    if entropy < 1.5 or entropy > 7.5:
        return "garbled", 0.85
    return "good", min(1.0, (entropy - 1.5) / 4.0)
```

---

## Performance Comparison Table

| Library | Avg Time/Page | Memory | Greek CMAP | Tables | Notes |
|---|---|---|---|---|---|
| PyMuPDF (fitz) | 10–20 ms | Low (~50 MB) | Pass-through (fails on broken CMAP) | Basic | Primary |
| pdfminer.six | 1,500–2,000 ms | Medium (~150 MB) | Partial Type1 recovery | No | CMAP fallback |
| pdfplumber | 500–800 ms | Medium (~180 MB) | Same as pdfminer | Excellent | Tables only |
| Tesseract (300 DPI) | 2,000–8,000 ms | High (~300 MB peak) | N/A (image-based) | No | Scanned docs |
| MarkItDown (PDF) | ~2,000 ms | Medium | Same as pdfminer | No | No PDF advantage |

**Practical estimate for Curator:** PyMuPDF handles ~85% of PDFs in under 20ms/page. pdfminer recovers another ~5–8% with broken Greek CMAPs. Tesseract covers scanned documents (~10% of a typical collection). For a 200-document indexing run of Cypriot university materials: expect **1–3 minutes total** with the full pipeline.

---

## Key References

- PyMuPDF docs: https://pymupdf.readthedocs.io/
- PyMuPDF performance: https://pymupdf.readthedocs.io/en/latest/about.html#performance
- pdfminer.six: https://github.com/pdfminer/pdfminer.six
- pdfplumber: https://github.com/jsvine/pdfplumber
- Tesseract: https://github.com/tesseract-ocr/tesseract
- pytesseract: https://github.com/madmaze/pytesseract
- Tesseract Greek (ell): https://github.com/tesseract-ocr/tessdata/blob/main/ell.traineddata
- MarkItDown: https://github.com/microsoft/markitdown
- pdf2image: https://github.com/Belval/pdf2image
- poppler (macOS): `brew install poppler`
- PDF Spec ISO 32000-2:2020, Section 9.10 — ToUnicode CMAP requirements
- Greek Font Society: https://www.greekfontsociety-gfs.gr/
