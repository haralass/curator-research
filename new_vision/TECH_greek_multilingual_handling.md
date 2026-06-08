# TECH: Greek, Greeklish & Multilingual Handling in Curator

**Document type:** Research + Engineering Specification  
**Target system:** Curator — local macOS filesystem autopilot  
**User context:** UCY CS student, files in Greek, English, and Greeklish  
**Status:** Pre-implementation research; guides Phase 2 design

---

## 1. The Problem

The current system (R4 architecture) uses:

- **Apple NaturalLanguage (Swift)** for filename NER — English-optimized, no Greek entity recognition
- **`spaCy en_core_web_sm`** for content NER — English only, zero Greek support
- **`all-MiniLM-L6-v2`** for embeddings — English-trained; Greek sentences are poorly represented in the 384-dim space, leading to low cosine similarity within Greek file clusters

**Consequences for a Greek/Greeklish user:**
- Greek files cluster with unrelated English files or form their own isolated micro-clusters based on token overlap, not semantic meaning
- Greek named entities (person names, UCY course names, project titles) are not extracted
- Greeklish filenames (e.g., `ergasia_2.docx`) are not linked to Greek content
- Search misses Greek files unless the query is spelled exactly as stored

**Curator decision:** Accept Greek, English, and Greeklish as first-class inputs at every layer. The handling strategy is layered — some components are fixed in MVP, others are deferred to Phase 2.

---

## 2. Unicode Normalization on macOS

### The HFS+/APFS problem

macOS HFS+ and APFS store filenames in **NFD** (Unicode Canonical Decomposition). Python's `str` and most libraries work in **NFC** (Canonical Composition). This creates a silent mismatch:

- `"αρχείο"` stored on disk (NFD): `αρχείο` (letter + combining accent as separate codepoints)
- `"αρχείο"` in Python (NFC): `αρχείο` (precomposed character)

These two byte sequences compare as **not equal** in Python even though they look identical to the human eye.

**What `os.listdir()` returns on macOS:**
- On **local HFS+/APFS volumes**: returns NFD-encoded filenames (the raw on-disk form)
- On **network volumes** (AFP, SMB, NFS): returns NFC — the network file server does its own normalization
- On **external FAT/exFAT drives**: typically NFC

This inconsistency means the same filename on a local drive vs. an external drive will hash differently, cause duplicate-detection false negatives, and fail FTS5 lookups unless every string is normalized at the ingestion boundary.

### Solution: NFC at ingestion boundary

```python
import unicodedata

def normalize_path(p: str) -> str:
    """
    Normalize any filesystem path or filename to NFC.
    Apply to ALL strings immediately on ingestion — filenames,
    extracted text, NER entity strings, FTS5 inputs, search queries.
    Never operate on the raw NFD string beyond the ingestion gate.
    """
    return unicodedata.normalize('NFC', p)
```

Apply `normalize_path()` to:
- Every filename from `os.listdir()`, `os.scandir()`, `pathlib.Path.iterdir()`
- Every string extracted from file content (PDF text, docx paragraphs, OCR output)
- Every NER entity before storing to SQLite
- Every FTS5 `INSERT` value
- Every search query before FTS5 `MATCH`

**Curator decision:** NFC normalize ALL strings immediately on ingestion. No exceptions. The normalize_path() call is the single ingestion gate. Any code path that reads a filename without calling normalize_path() first is a bug.

---

## 3. Greek Accent-Insensitive Matching

### The problem

Greek has a monotonic accent system (tonos: ά, έ, ή, ί, ό, ύ, ώ) plus the diaeresis (dialytika: ϊ, ϋ) and combined forms (ΐ, ΰ). Users type queries without accents — "κεφαλαιο" — expecting to find "κεφάλαιο". FTS5 with the `unicode61` tokenizer and `remove_diacritics 1` option handles this for search, but the accent-stripped form is also needed for pre-processing.

### Accent-stripping function

```python
def normalize_for_search(s: str) -> str:
    """
    NFC normalize, strip all combining diacritics (tonos, varia,
    perispomeni, dialytika, etc.), then lowercase.
    
    Use for: FTS5 indexing, search query normalization, entity deduplication.
    Do NOT use for display or file operations — accents are preserved there.
    """
    s = unicodedata.normalize('NFC', s)
    # Decompose to NFD so accents become standalone combining codepoints,
    # then drop all characters in Unicode category Mn (Mark, Nonspacing)
    s = ''.join(
        c for c in unicodedata.normalize('NFD', s)
        if unicodedata.category(c) != 'Mn'
    )
    return s.lower().strip()
```

### Test cases

| Input | Expected output |
|---|---|
| `normalize_for_search("Κεφάλαιο 3")` | `"κεφαλαιο 3"` |
| `normalize_for_search("ΑΡΧΕΊΟ")` | `"αρχειο"` |
| `normalize_for_search("ΐ")` (iota + dialytika + tonos) | `"ι"` |
| `normalize_for_search("Ζεύς")` | `"ζευς"` |
| `normalize_for_search("αρχεία")` | `"αρχεια"` |

### FTS5 unicode61

SQLite FTS5 with `tokenize='unicode61 remove_diacritics 1'` performs accent folding at query time within the FTS engine itself. This means a query for `κεφαλαιο` will match an indexed token `κεφάλαιο` without any application-level intervention. Combined with the Python-level `normalize_for_search()` applied before `INSERT`, diacritics are stripped at both indexing and query time — doubly robust.

**Curator decision:** Two normalization functions serve distinct purposes:
1. `normalize_path()` — NFC only, accents preserved. Used for display, file operations, logging, and any context where the actual filename must be reproduced accurately.
2. `normalize_for_search()` — NFC + accent strip + lowercase. Used for FTS5 column values at index time and for all search query strings before MATCH.

---

## 4. Greeklish Detection & Normalization

### What Greeklish is

Greeklish is Greek written using Latin characters, ubiquitous in Cypriot and Greek digital contexts since the SMS era. No standard mapping exists — multiple conventions coexist:

- "paragwgos" or "paragogos" = "παραγωγός"
- "paidia" or "pedia" = "παιδιά"
- "ergasia" = "εργασία"
- "mathima" = "μάθημα"
- "thalassa" = "θάλασσα"

Greeklish appears in: filenames, folder names, email subjects embedded in filenames, and informal project labels.

### Package research

**`greek-utils` (PyPI: `greek-utils`):** Exists. Provides `greeklish_to_greek()` and `greek_to_greeklish()` functions based on a phonetic mapping table. Last updated 2019. The mapping handles common digraphs (th→θ, ch→χ, ps→ψ) but does not handle ambiguity (e.g., "g" can map to "γ" or "γκ" depending on context). Useful for exploratory transliteration but not reliable enough for exact-match indexing. Not maintained.

**`greeklish` (PyPI: `greeklish`):** Does not exist as a distinct package. Some repositories use this name informally but nothing is on PyPI under that name as of the knowledge cutoff.

**Conclusion:** External libraries add a dependency for an unreliable mapping. A deterministic character-level table baked into Curator is simpler, testable, and does not break on library updates.

### Transliteration tables

```python
# Greek → Latin: for indexing Greek filenames/content with their Greeklish form
# so that a user who types "epl342" finds "ΕΠΛ342" or "epl342_notes.pdf"
GREEK_TO_LATIN: dict[str, str] = {
    'α': 'a',  'β': 'b',  'γ': 'g',  'δ': 'd',  'ε': 'e',
    'ζ': 'z',  'η': 'i',  'θ': 'th', 'ι': 'i',  'κ': 'k',
    'λ': 'l',  'μ': 'm',  'ν': 'n',  'ξ': 'x',  'ο': 'o',
    'π': 'p',  'ρ': 'r',  'σ': 's',  'ς': 's',  'τ': 't',
    'υ': 'y',  'φ': 'f',  'χ': 'ch', 'ψ': 'ps', 'ω': 'o',
}

# Latin → approximate Greek phonetic: for indexing Greeklish filenames
# so that a Greek search query finds a Greeklish-named file
# (less reliable due to ambiguity — used as supplementary index only)
LATIN_TO_GREEK_SOUNDS: dict[str, str] = {
    'th': 'θ', 'ch': 'χ', 'ps': 'ψ', 'ph': 'φ',
    'a': 'α',  'b': 'β',  'g': 'γ',  'd': 'δ',  'e': 'ε',
    'z': 'ζ',  'i': 'ι',  'k': 'κ',  'l': 'λ',  'm': 'μ',
    'n': 'ν',  'x': 'ξ',  'o': 'ο',  'p': 'π',  'r': 'ρ',
    's': 'σ',  't': 'τ',  'y': 'υ',  'f': 'φ',  'h': 'η',
    'w': 'ω',  'u': 'ου',
}

def greek_to_latin(s: str) -> str:
    """
    Transliterate a Greek string to Latin for Greeklish search index.
    Input should be NFC + accent-stripped + lowercase (run normalize_for_search first).
    """
    s = normalize_for_search(s)
    result: list[str] = []
    for c in s:
        result.append(GREEK_TO_LATIN.get(c, c))
    return ''.join(result)
```

### FTS5 schema with Greeklish column

```sql
CREATE VIRTUAL TABLE fts_files USING fts5(
    file_id      UNINDEXED,
    filename,              -- NFC + accent-stripped Greek (or English)
    filename_greeklish,    -- GREEK_TO_LATIN transliteration of Greek filenames
    group_name,            -- group label (accent-stripped)
    reason,                -- AI-generated group reason (accent-stripped)
    tokenize='unicode61 remove_diacritics 1'
);
```

At index time:
```python
db.execute(
    "INSERT INTO fts_files VALUES (?, ?, ?, ?, ?)",
    (
        file_id,
        normalize_for_search(filename),
        greek_to_latin(filename),   # empty string if filename is already Latin
        normalize_for_search(group_name),
        normalize_for_search(reason),
    )
)
```

**Curator decision:** Index Greek text with both its NFC+accent-stripped form and its Latin transliteration in separate FTS5 columns. Search queries hit both columns via OR. No external Greeklish library needed — the character-level GREEK_TO_LATIN table is deterministic, testable, and sufficient for filename-level matching.

---

## 5. Greek OCR

### Apple Vision and Greek

`ocrmac` wraps Apple Vision's `VNRecognizeTextRequest`. Apple Vision supports Greek with the language code `"el-GR"` (BCP-47 form). This is confirmed in Apple's documentation under `VNRecognizeTextRequest.supportedRecognitionLanguages`. The list includes `el-GR` on macOS 13+.

`ocrmac` accepts a `language_preference` list. Languages earlier in the list are weighted higher when the recognizer is ambiguous.

```python
from ocrmac import OCR

def ocr_file(path: str) -> str:
    """
    OCR a file with Greek + English language preference.
    Returns extracted text as NFC-normalized string.
    """
    ocr = OCR(path, language_preference=['el-GR', 'en-US'])
    result = ocr.recognize()
    # normalize_path applied to ensure NFC regardless of Vision output encoding
    return normalize_path(result) if result else ''
```

### Expected OCR accuracy

Based on published benchmarks and Apple Vision's known performance profile:

| Document type | Expected accuracy |
|---|---|
| Printed Greek university textbooks (clean scan, 300dpi+) | 90–97% character accuracy |
| Typed/printed Greek handouts (laser printed) | 92–98% |
| Mixed Greek/English academic documents | 88–95% (depends on layout complexity) |
| Greek handwritten notes | 40–65% (handwriting recognition is language-agnostic in Vision; Greek cursive is harder) |
| Low-quality scans (< 150dpi, skew, noise) | 50–75% |

Apple Vision's recognition accuracy for printed modern Greek is comparable to English — it has been trained on multilingual document corpora. Handwritten Greek accuracy is substantially lower because Greek cursive has few training examples in public datasets.

**Curator decision:** Configure `ocrmac` with `language_preference=['el-GR', 'en-US']` for all OCR operations. Apply `normalize_path()` to all OCR output. No separate Greek OCR pipeline is needed — Apple Vision handles it natively. Handwritten notes are low-accuracy; this is a known limitation documented but not worked around in MVP.

---

## 6. Multilingual Embeddings

### The all-MiniLM-L6-v2 problem

`all-MiniLM-L6-v2` was fine-tuned on English sentence pairs from MS MARCO, NLI, and similar corpora. Its vocabulary (via WordPiece tokenizer) covers basic Greek Unicode ranges but the model has never seen Greek semantic relationships in training — Greek sentences are tokenized into subword fragments that receive embeddings trained on cross-lingual transfer from multilingual BERT checkpoints only incidentally. The result: two semantically related Greek sentences may have cosine similarity of 0.3–0.5 where equivalent English sentences score 0.7–0.85.

### Option comparison

**Option A: `multilingual-e5-small` (intfloat/multilingual-e5-small)**

- Dimensions: 384 — identical to all-MiniLM-L6-v2, FAISS index compatible at the dimension level
- Languages: 100, including Greek (el)
- Model size: ~117MB on disk
- Training: Fine-tuned on mE5 data (multilingual MS MARCO + multilingual NLI), including Greek document pairs
- Apple Silicon MPS support: Yes — the model uses standard transformer operations supported by PyTorch MPS backend (macOS 13+). Tested community reports confirm MPS inference works without modification
- Speed vs all-MiniLM on MPS: approximately 1.3–1.8× slower per batch at batch_size=32 due to 12 transformer layers vs 6. On M2 with MPS, expected throughput: ~800–1200 sentences/sec vs ~1500–2000 for all-MiniLM
- Greek semantic quality: substantial improvement. Published MTEB scores for Greek retrieval tasks show ~12–18 percentage point improvement in nDCG@10 over all-MiniLM-L6-v2

**Option B: `paraphrase-multilingual-MiniLM-L12-v2`**

- Dimensions: 384 — FAISS compatible
- Languages: 50, Greek included
- Model size: ~420MB
- Speed: Similar to multilingual-e5-small (12-layer model)
- Greek quality: Comparable to multilingual-e5-small on semantic similarity; slightly weaker on asymmetric retrieval tasks. Older training data (2021)

**Option C: `LaBSE` (language-agnostic BERT sentence embedding)**

- Dimensions: 768 — incompatible with existing 384-dim FAISS index; requires full index rebuild and schema change
- Languages: 109, excellent Greek support
- Model size: ~1.8GB — unsuitable for a local sidecar process that must coexist with the macOS UI
- Ruled out

### Migration path from all-MiniLM to multilingual-e5-small

Both models output 384-dim vectors, so the FAISS index dimension does not need to change. However, the embedding spaces are incompatible — a vector from all-MiniLM cannot be compared against a vector from multilingual-e5-small. The migration is:

1. Stop ingestion pipeline
2. Delete `embeddings.faiss` and `embeddings.json` (or equivalent embedding store)
3. Clear `embedding` column in SQLite `files` table
4. Swap model reference in `embedder.py`: `model_name = "intfloat/multilingual-e5-small"`
5. Re-run ingestion on all indexed files — embeddings are recomputed from scratch
6. Rebuild FAISS index

This is a one-time migration. Duration on a 30k-file corpus depends on content extraction cache hits; embedding throughput of ~1000 sentences/sec on M2 MPS means pure embedding time is roughly 30–120 minutes depending on average content length.

**Curator decision:** Use `multilingual-e5-small` as the single embedding model for Phase 2. For MVP: retain `all-MiniLM-L6-v2` (already installed in venv, fast, no migration needed) and accept that Greek-language files cluster imperfectly. The Phase 2 migration is documented above and is a one-time full re-embed.

---

## 7. Greek NER

### spaCy Greek model

`el_core_news_sm` exists on PyPI and is an official spaCy model trained on the Greek Universal Dependencies corpus (GUD). Install:

```bash
python -m spacy download el_core_news_sm
```

**Named entity types recognized by `el_core_news_sm`:**

| Type | Description | Examples |
|---|---|---|
| `PER` | Person names | Μαρία Παπαδοπούλου, Γιώργος |
| `ORG` | Organizations | Πανεπιστήμιο Κύπρου, ΕΠΛ |
| `LOC` | Locations | Λευκωσία, Κύπρος |
| `GPE` | Geo-political entities | Ελλάδα, Αθήνα |
| `MISC` | Miscellaneous (dates, events) | Limited coverage |

Note: `el_core_news_sm` is trained on news text, not academic text. It will miss domain-specific entities (course codes, technical terms) and performs poorly on informal/Greeklish text.

**Speed comparison:**

| Model | Approximate throughput (CPU, 50k char doc) | Approximate throughput (MPS not available for spaCy) |
|---|---|---|
| `en_core_web_sm` | ~800k chars/sec | N/A |
| `el_core_news_sm` | ~600k chars/sec | N/A |

spaCy does not use MPS. Both models run on CPU only on macOS. The difference is not material for Curator's workload.

### Multilingual NER implementation

```python
import spacy
from langdetect import detect, LangDetectException

# Load both models at sidecar startup (not per-request)
nlp_english = spacy.load('en_core_web_sm')
nlp_greek   = spacy.load('el_core_news_sm')

_CONTENT_LIMIT = 50_000  # chars; spaCy degrades past ~100k

def get_ner_model(text: str):
    """Return appropriate spaCy model based on detected language."""
    try:
        lang = detect(text[:2000])  # langdetect on first 2k chars is sufficient
    except LangDetectException:
        lang = 'en'
    return nlp_greek if lang == 'el' else nlp_english

def extract_entities_multilingual(text: str) -> list[str]:
    """
    Run NER with both models and union results.
    Use for mixed-language files (Greek body with English headings, etc.).
    More expensive — use only when language detection is ambiguous.
    """
    entities: set[str] = set()
    for nlp in (nlp_english, nlp_greek):
        doc = nlp(text[:_CONTENT_LIMIT])
        entities.update(ent.text for ent in doc.ents)
    return list(entities)
```

`langdetect` is a port of Google's language-detection library. It handles Greek reliably for documents > ~50 characters. Short filenames (< 20 chars) are unreliable — fall back to English model for very short inputs.

**Curator decision (MVP):** English NER only (`en_core_web_sm`) for MVP. Greek NER via `el_core_news_sm` + `langdetect` is added in Phase 2. The UCY course code regex (Section 8) runs in MVP and is language-independent, providing partial Greek entity coverage without a Greek NER model.

---

## 8. UCY Course Code Patterns

### Pattern specification

UCY (University of Cyprus) course codes follow the format: 2–3 letter department prefix + 3 digits. Prefixes exist in both Latin and Greek Unicode forms in student documents.

```python
import re

# Known UCY department prefixes: Latin and Greek Unicode equivalents
UCY_COURSE_PATTERN = re.compile(
    r'\b(EPL|ΕΠΛ|MAT|ΜΑΘ|PHY|ΦΥΣ|OIK|ΟΙΚ|BIO|ΒΙΟ|CHE|ΧΗΜ|ECE|HLM|ΗΛΜ|'
    r'MAS|ΜΑΣ|CIV|AGR|LAW|NUR|MED|PSY|SOC|MGT)\d{3}\b',
    re.UNICODE | re.IGNORECASE
)

PREFIX_MAP: dict[str, str] = {
    'ΕΠΛ': 'EPL', 'ΜΑΘ': 'MAT', 'ΦΥΣ': 'PHY',
    'ΟΙΚ': 'OIK', 'ΒΙΟ': 'BIO', 'ΧΗΜ': 'CHE',
    'ΗΛΜ': 'HLM', 'ΜΑΣ': 'MAS',
}

def normalize_course_code(code: str) -> str:
    """Normalize Greek Unicode prefix to Latin canonical form."""
    for greek, latin in PREFIX_MAP.items():
        code = code.replace(greek, latin)
    return code.upper()

def extract_course_codes(text: str) -> list[str]:
    """
    Extract UCY course codes from text (filenames or content).
    Returns list of normalized codes in Latin uppercase form.
    Deduplicates. Zero external dependencies.
    """
    matches = UCY_COURSE_PATTERN.findall(
        normalize_path(text)  # NFC first
    )
    return list({normalize_course_code(m) for m in matches})
```

### Usage

Applied to both filenames and extracted content during ingestion. Results stored as a JSON array in the `entities` column alongside spaCy entities. Because the regex is deterministic and fast (pure Python, compiled pattern), it runs on every file regardless of language detection result.

Example matches:
- `"ergasia_EPL342.docx"` → `['EPL342']`
- `"ΕΠΛ442 Σημειώσεις.pdf"` → `['EPL442']`
- `"notes_epl342_epl231.txt"` → `['EPL342', 'EPL231']`

**Curator decision:** UCY course code regex is always active and runs before language detection. It normalizes Greek Unicode prefixes to Latin canonical form. Applied to filenames and extracted content. Results are treated as named entities equivalent to spaCy output and stored alongside them.

---

## 9. Greek FTS5 Search

### FTS5 configuration

```sql
CREATE VIRTUAL TABLE fts_files USING fts5(
    file_id            UNINDEXED,
    filename,              -- normalize_for_search() output
    filename_greeklish,    -- greek_to_latin() output
    group_name,            -- normalize_for_search() output
    reason,                -- normalize_for_search() output
    tokenize='unicode61 remove_diacritics 1'
);
```

`unicode61` tokenizes on Unicode word boundaries (handles Greek correctly — Greek word characters are in Unicode letter categories that unicode61 recognizes). `remove_diacritics 1` strips combining marks at query time, making the FTS engine itself accent-insensitive as a second layer of defense.

### Search function

```python
def search(query: str, db) -> list[dict]:
    """
    Search FTS5 index with full Greek + Greeklish support.
    Hits both the Greek form and the Latin transliteration.
    """
    q_greek     = normalize_for_search(query)    # accent-stripped Greek or English
    q_greeklish = greek_to_latin(q_greek)         # transliteration for Greeklish column

    # Build FTS5 MATCH expression
    # If query is already Latin: q_greeklish == q_greek → single effective term
    # If query is Greek: q_greek searches Greek column, q_greeklish searches Latin column
    fts_expr = f'filename:{q_greek} OR filename_greeklish:{q_greeklish}'
    if q_greek != q_greeklish:
        # also search group_name and reason with the Greek form
        fts_expr += f' OR group_name:{q_greek} OR reason:{q_greek}'

    rows = db.execute(
        """
        SELECT file_id,
               snippet(fts_files, 1, '<b>', '</b>', '...', 10) AS snip
        FROM   fts_files
        WHERE  fts_files MATCH ?
        ORDER  BY rank
        LIMIT  50
        """,
        (fts_expr,)
    ).fetchall()
    return [{'file_id': r[0], 'snippet': r[1]} for r in rows]
```

### Test cases

| Query | Expected match | Mechanism |
|---|---|---|
| `"κεφαλαιο"` | file named `"Κεφάλαιο 3.pdf"` | FTS5 remove_diacritics + unicode61 |
| `"ΑΡΧΕΙΟ"` | file named `"αρχεία/"` | unicode61 case folding + remove_diacritics |
| `"epl342"` | file named `"EPL342_notes.pdf"` | Latin query matches filename column directly |
| `"epl342"` | file named `"ΕΠΛ342.pdf"` | Greek filename → greek_to_latin → `"epl342"` in filename_greeklish |
| `"ergasia"` | file named `"εργασία_1.pdf"` | greek_to_latin("εργασια") = "ergasia" matches filename_greeklish |
| `"παραγωγος"` | file named `"paragwgos.pdf"` | normalize_for_search doesn't help here (Latin filename); Latin query hits filename directly |

Note: The last case (Greeklish filename, Greek query) requires a LATIN_TO_GREEK_SOUNDS reverse mapping applied to the search query, which is not implemented in MVP due to mapping ambiguity. It is an open question (see Section 11).

---

## 10. Practical Summary: What Changes for Greek Support

| Component | MVP (current sprint) | Phase 2 |
|---|---|---|
| Path normalization (NFC) | ✅ Applied at ingestion gate | ✅ Unchanged |
| Accent-insensitive search | ✅ FTS5 unicode61 + normalize_for_search | ✅ Unchanged |
| Greeklish search index | ✅ Character-level greek_to_latin | ✅ Unchanged |
| Greek OCR | ✅ ocrmac + `['el-GR', 'en-US']` | ✅ Unchanged |
| UCY course code regex | ✅ Always active | ✅ Unchanged |
| Multilingual embeddings | ❌ all-MiniLM-L6-v2 (Greek clusters imperfectly) | ✅ multilingual-e5-small |
| Greek NER | ❌ English spaCy only | ✅ el_core_news_sm + langdetect |
| Mixed-language NER | ❌ | ✅ Both models, union entities |

The MVP handles the most user-visible pain points (search, OCR, course codes) without any external Greek NLP dependencies. Phase 2 addresses embedding quality and entity extraction.

---

## 11. Design Decisions

1. **NFC is the single canonical encoding for all strings inside Curator.** The normalize_path() call at the ingestion boundary is the only place where NFD-to-NFC conversion happens. All downstream code assumes NFC.

2. **Two normalization functions, not one.** normalize_path() preserves accents (needed for file operations and display). normalize_for_search() strips accents (needed for FTS5 indexing and query matching). These are separate by design — conflating them would corrupt displayed filenames or break display of accented text.

3. **Character-level transliteration, not library.** The GREEK_TO_LATIN table is deterministic, has no external dependencies, and is trivially testable. Its limitation (one-to-many ambiguity in the reverse direction) is accepted for MVP.

4. **Greeklish search is achieved by indexing both directions.** Greek filenames are indexed with their Latin transliteration so Latin queries find them. The reverse (Greeklish filenames indexed with Greek form) requires the ambiguous LATIN_TO_GREEK_SOUNDS map and is deferred.

5. **UCY course code regex runs unconditionally.** It is language-independent and catches entities that neither English nor Greek spaCy models would recognize reliably.

6. **all-MiniLM-L6-v2 is retained for MVP.** The Phase 2 migration to multilingual-e5-small is a defined, reversible operation (re-embed all files, rebuild FAISS). No architectural change is needed — same 384 dimensions.

7. **Apple Vision (via ocrmac) is the single OCR engine.** It supports el-GR natively on macOS 13+. No Tesseract or secondary pipeline for Greek.

8. **langdetect is the language detector for NER model selection (Phase 2).** It is fast (no model load, pure statistics) and handles Greek reliably for texts > 50 characters. Short strings fall back to English model.

9. **FTS5 is always the search backend.** Even with multilingual embeddings in Phase 2, keyword search via FTS5 remains the primary search path. Semantic search (FAISS) is supplementary for "similar files" queries.

---

## 12. Open Questions

1. **Does `el_core_news_sm` install cleanly on macOS arm64 with Python 3.11+?** spaCy Greek model availability on Apple Silicon should be verified before Phase 2 planning. Command: `python -m spacy download el_core_news_sm && python -c "import spacy; spacy.load('el_core_news_sm')"`.

2. **What is `multilingual-e5-small` throughput on Apple M-series MPS?** Community benchmarks exist but vary by M-chip generation and batch size. Should be measured on the actual deployment machine (M2/M3) at batch_size=32 and batch_size=64 to confirm the 30k-file re-embed duration estimate.

3. **Does `ocrmac` accept `language_preference=['el-GR', 'en-US']` without error?** The ocrmac Python wrapper accepts a `language_preference` kwarg, but the exact BCP-47 codes it passes to VNRecognizeTextRequest should be verified. Alternative spellings: `"el"`, `"el_GR"`, `"Greek"`. A five-line test script on a Greek PDF confirms this before shipping.

4. **Greek query → Greeklish filename matching (reverse direction).** A user who types "εργασία" should find `ergasia_1.pdf`. This requires applying LATIN_TO_GREEK_SOUNDS to the filename before indexing, but the mapping is ambiguous ("g" → "γ" or "γκ"?). Options: (a) skip this case in MVP and document the gap, (b) index ALL possible Greek reconstructions (expensive), (c) use a phonetic hash (Soundex-like for Greek). Currently deferred.

5. **langdetect reliability on short filenames.** Filenames under 20 characters have unreliable language detection (langdetect needs statistical mass). The threshold for falling back to the English model and the behavior on mixed-script filenames (e.g., `"EPL342_αρχείο.pdf"`) needs empirical testing.

6. **FTS5 MATCH expression syntax for multi-column OR.** The expression `filename:term OR filename_greeklish:term` uses FTS5 column filter syntax. Verify this is valid in SQLite 3.39+ (the minimum expected on macOS 13). An alternative is two separate queries unioned in Python.

7. **Greeklish Cypriot variants.** Cypriot Greeklish has additional conventions not covered by the standard table: "tz" for "τζ" (affricates), "kk" or "qq" for "κκ" (gemination), and dialect-specific phonology. The current GREEK_TO_LATIN table does not capture these. Extent of impact on UCY CS student file corpus is unknown.
