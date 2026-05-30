## ΜΕΡΟΣ 5: NOVEL IDEAS — ΚΑΝΕΝΑΣ ΔΕΝ ΤΑ ΕΧΕΙ ΚΑΝΕΙ

### 1. Antilibrary (υλοποιήσιμο σήμερα, 10 γραμμές κώδικα)
```python
import subprocess
result = subprocess.run([
    'mdfind', '-onlyin', '/Users/me/Downloads',
    'kMDItemUseCount == 0 && kMDItemDateAdded < $time.ago(-15552000)'
], capture_output=True, text=True)
```
Αρχεία που δεν άνοιξες ποτέ σε 6+ μήνες. Κανένα file organizer δεν το κάνει.

### 2. File Biography
`kMDItemUseCount`, `kMDItemUsedDates`, `kMDItemLastUsedDate` — το macOS ξέρει ΠΟΤΕ και ΠΟΣΕΣ ΦΟΡΕΣ άνοιξε κάθε αρχείο. Αξία αρχείου = συνάρτηση frequency + recency. Zero published papers.

### 3. NER-based Project Detection
Spacy: αρχεία που αναφέρουν τα ίδια entities ("EPL326", "Prof. Smith") → ίδιο project. Cross-format (PDF + Python + Excel μαζί). Genuinely novel.

### 4. Temporal Co-occurrence
Αρχεία που τροποποιήθηκαν την ίδια ώρα σε πολλές sessions → ίδιο project. CodeScene το κάνει για git commits. Κανένας για personal files.

### 5. Semantic Gravity Correction
Μεγάλα clusters "τραβάνε" αδίκαια. Λύση: `similarity - 0.1 * log(cluster_size)`. Formalized εδώ για πρώτη φορά.

### 6. Graph Betweenness για Approval UI
Αρχεία που "γεφυρώνουν" δύο clusters → δείχνε τα πρώτα στο review. Αρχεία βαθιά μέσα σε cluster → auto-approve. Novel application.

### 7. Folder Biography
Σύγκριση folders: "το EPL326 folder σου φαίνεται ελλιπές vs EPL325 του περσινού εξαμήνου". Zero papers.

### 8. Version Detection χωρίς filename hints
RETSim (Google): cosine >0.86 = version. Δουλεύει σε PDFs, Word, code. `pip install unisim`.

---
