## ΜΕΡΟΣ 18: ZIP/ARCHIVE HANDLING
*(Έρευνα 2026-05-22 — 12 searches, 6 sources fetched)*

### Γιατί αυτό είναι σημαντικό
Στο ~/Downloads βρίσκονται πολλά .zip, .tar.gz, .7z, .rar αρχεία. Χωρίς έξυπνο handling, τα κάνουμε embed μόνο με το filename (λίγη πληροφορία), ή τα παραλείπουμε εντελώς. Αυτό το μέρος ορίζει πώς να τα επεξεργαστούμε έξυπνα, ασφαλή, και γρήγορα — χωρίς full extraction στο disk.

---

### Α. Python Libraries — Per Format

#### FORMAT: .zip (Python standard library)

```python
import zipfile

# Έλεγχος αν είναι έγκυρο ZIP πριν ανοίξουμε
if not zipfile.is_zipfile(path):
    return None  # δεν είναι ZIP — handle ως unknown

with zipfile.ZipFile(path, 'r') as zf:
    # 1. List contents (zero extraction)
    names = zf.namelist()      # ['file.py', 'README.md', 'data/input.csv']
    info_list = zf.infolist()  # ZipInfo objects με metadata
    
    # Metadata per file (χωρίς extraction):
    for info in info_list:
        print(info.filename)       # path inside zip
        print(info.file_size)      # uncompressed bytes
        print(info.compress_size)  # compressed bytes
        print(info.date_time)      # (year, month, day, hour, min, sec)
    
    # 2. Encrypted check (per member)
    def is_member_encrypted(zinfo):
        return bool(zinfo.flag_bits & 0x1)
    
    # 3. Read specific file in memory (NO disk write)
    # Ψάχνουμε για README/text files
    TEXT_EXTENSIONS = {'.txt', '.md', '.rst', '.readme'}
    readme_content = None
    for name in names:
        ext = os.path.splitext(name.lower())[1]
        if ext in TEXT_EXTENSIONS or 'readme' in name.lower():
            try:
                raw = zf.read(name)  # bytes in memory
                readme_content = raw.decode('utf-8', errors='replace')[:2000]
                break
            except (RuntimeError, Exception):  # RuntimeError = encrypted
                pass
    
    # 4. Corruption check
    bad = zf.testzip()  # returns first bad file or None
```

**Δυνατότητες standard `zipfile`:**
- `namelist()` — list αρχείων χωρίς extraction ✅
- `infolist()` — metadata per file ✅
- `read(name)` — bytes in memory χωρίς disk ✅
- `is_zipfile()` — validation ✅
- `testzip()` — CRC check ✅
- Encrypted detection: `flag_bits & 0x1` ✅
- **ΔΕΣΜΕΥΣΗ:** Δεν μπορεί να δημιουργήσει encrypted files (read-only)
- **ΠΡΟΣΟΧΗ:** Decryption είναι "extremely slow" (native Python, not C)

---

#### FORMAT: .tar.gz / .tar.bz2 / .tar.xz (Python standard library)

```python
import tarfile

if tarfile.is_tarfile(path):
    with tarfile.open(path, 'r:*') as tf:  # 'r:*' = auto-detect compression
        # List contents
        names = tf.getnames()
        members = tf.getmembers()  # TarInfo objects
        
        # Read specific file in memory
        readme_member = None
        for member in members:
            if 'readme' in member.name.lower() or member.name.endswith('.txt'):
                readme_member = member
                break
        
        if readme_member:
            f = tf.extractfile(readme_member)  # file-like object, NO disk write
            if f:
                content = f.read().decode('utf-8', errors='replace')[:2000]
```

**Notes:**
- `r:*` = διαβάζει gz, bz2, xz αυτόματα
- `extractfile()` επιστρέφει file-like object χωρίς extraction στο disk ✅
- `r|*` (pipe mode) = streaming για πολύ μεγάλα αρχεία, αλλά δεν επιτρέπει seek back

---

#### FORMAT: .7z (py7zr — pure Python)

```python
import py7zr

with py7zr.SevenZipFile(path, 'r') as archive:
    # List contents (χωρίς extraction)
    names = archive.getnames()       # list of filenames
    all_files = archive.list()       # ArchiveInfo objects με metadata
    
    # Read specific file to memory
    # py7zr δεν έχει εύκολο "read one file" API —
    # πρέπει να χρησιμοποιήσεις read() με targets
    readme_targets = [n for n in names
                      if 'readme' in n.lower() or n.endswith('.txt')]
    if readme_targets:
        data_dict = archive.read(targets=readme_targets[:1])
        # data_dict = {filename: BytesIO object}
        content = data_dict[readme_targets[0]].read().decode('utf-8', errors='replace')[:2000]
```

**Εγκατάσταση:** `pip install py7zr`
**Pure Python** — δεν χρειάζεται εξωτερικά binaries
**Encrypted 7z:** Raises `py7zr.exceptions.PasswordRequired` — πρέπει να κάνεις catch

---

#### FORMAT: .rar (rarfile — χρειάζεται εξωτερικό binary)

```python
import rarfile

# Προαπαιτούμενο: unrar ή unar binary στο PATH
# macOS: brew install unrar  OR  brew install unar

try:
    with rarfile.RarFile(path, 'r') as rf:
        names = rf.namelist()
        
        # Check αν είναι encrypted
        if rf.needs_password():
            # Encrypted header — δεν μπορούμε να κάνουμε list
            return None
        
        for name in names:
            if 'readme' in name.lower():
                content = rf.read(name).decode('utf-8', errors='replace')[:2000]
                break

except rarfile.NeedFirstVolume:
    # Multi-part RAR — παράλειψε
    pass
except rarfile.NoCrypto:
    # Encrypted headers, χωρίς crypto module
    pass
```

**Προβλήματα:**
- Απαιτεί `unrar` ή `unar` binary (δεν είναι pre-installed στο macOS)
- Encrypted headers: δεν μπορείς ούτε να κάνεις namelist χωρίς password
- **Εναλλακτικό:** `libarchive-c` (`pip install libarchive-c`) — υποστηρίζει RAR χωρίς χωριστό binary (χρησιμοποιεί τη system libarchive)

---

#### FORMAT: .rar με libarchive-c (καλύτερη επιλογή για .rar)

```python
import libarchive

with libarchive.file_reader(path) as archive:
    for entry in archive:
        name = entry.pathname
        # entry.size = uncompressed size
        # entry.mtime = modification time
        if 'readme' in name.lower():
            # Read chunks
            content_bytes = b''.join(entry.get_blocks())
            content = content_bytes.decode('utf-8', errors='replace')[:2000]
            break
```

**Εγκατάσταση:** `pip install libarchive-c`
**Υποστηρίζει:** TAR, ZIP, 7-ZIP, RAR, XAR, LHA, AR, CAB, ISO
**Streaming:** Διαβάζει σε single pass — δεν γράφει στο disk ✅
**Version:** 5.3 (22 Μαΐου 2025) — ενεργά maintained

---

### Β. Nested Archives και Edge Cases

#### Nested ZIPs (π.χ. 42.zip pattern)

```python
def is_nested_zip(zf: zipfile.ZipFile) -> bool:
    """Ελέγχει αν υπάρχει ZIP μέσα σε ZIP"""
    for name in zf.namelist():
        if name.lower().endswith(('.zip', '.tar.gz', '.7z', '.tar')):
            return True
    return False
```

**Στρατηγική:** Αν βρούμε nested ZIP, **δεν** κάνουμε recursive extraction. Το καταγράφουμε στο embedding ως feature: `"archive containing nested archives"`.

#### Encrypted Archives

| Μορφή | Ανίχνευση | Τι κάνουμε |
|---|---|---|
| .zip encrypted | `info.flag_bits & 0x1` | Embed filename + size only |
| .zip encrypted headers | ZipFile ανοίγει αλλά read() → RuntimeError | Catch → fallback |
| .7z encrypted | `py7zr.exceptions.PasswordRequired` | Catch → fallback |
| .rar encrypted headers | `rf.needs_password()` → True | Catch → fallback |
| .tar.gz | Δεν υποστηρίζει encryption | N/A |

#### Corrupted Archives

```python
# ZIP corruption
try:
    if not zipfile.is_zipfile(path):
        raise ValueError("Not a ZIP")
    with zipfile.ZipFile(path) as zf:
        bad = zf.testzip()
        if bad:
            # Μερική corruption — μπορεί να διαβάσουμε ό,τι γλυτώσει
            names = zf.namelist()  # often still works
except zipfile.BadZipFile:
    # Εντελώς κατεστραμμένο — embed filename only
    pass

# tarfile corruption
try:
    tf = tarfile.open(path, 'r:*')
except tarfile.TarError:
    pass

# py7zr corruption  
try:
    with py7zr.SevenZipFile(path, 'r') as a:
        pass
except Exception:
    pass
```

#### Συνηθισμένα περιεχόμενα .tar.gz μέσα σε .zip
Αν το namelist() εμφανίζει `something.tar.gz` μέσα στο ZIP, **μην** ξεπακετάρεις — καταγράφης ως "archive-in-archive" στο embedding text.

---

### Γ. Embedding Strategy για Archives

#### Tier 1: Αρχείο που μπορούμε να επιθεωρήσουμε

```python
def build_archive_embedding_text(path: str, max_files_list: int = 50) -> str:
    """
    Δημιουργεί embedding text για archive χωρίς disk extraction.
    Returns: text string για embedding
    """
    stem = Path(path).stem
    suffix = Path(path).suffix.lower()
    
    filelist = []
    readme_content = ""
    is_encrypted = False
    is_nested = False
    file_count = 0
    total_size = 0
    
    try:
        if suffix == '.zip' and zipfile.is_zipfile(path):
            with zipfile.ZipFile(path, 'r') as zf:
                all_info = zf.infolist()
                file_count = len(all_info)
                total_size = sum(i.file_size for i in all_info)
                
                # Check encrypted
                encrypted_count = sum(1 for i in all_info if i.flag_bits & 0x1)
                is_encrypted = encrypted_count > 0
                
                # Check nested
                is_nested = any(
                    n.lower().endswith(('.zip','.tar.gz','.7z'))
                    for n in zf.namelist()
                )
                
                # Filelist (top N)
                filelist = zf.namelist()[:max_files_list]
                
                # README
                if not is_encrypted:
                    for name in zf.namelist():
                        if ('readme' in name.lower() or
                                name.lower().endswith(('.txt', '.md'))):
                            try:
                                raw = zf.read(name)
                                readme_content = raw.decode('utf-8', errors='replace')[:1500]
                                break
                            except RuntimeError:
                                is_encrypted = True
                                break
    
        elif suffix in ('.gz', '.bz2', '.xz') or path.endswith('.tar.gz'):
            if tarfile.is_tarfile(path):
                with tarfile.open(path, 'r:*') as tf:
                    members = tf.getmembers()
                    file_count = len(members)
                    total_size = sum(m.size for m in members)
                    filelist = [m.name for m in members[:max_files_list]]
                    
                    for m in members:
                        if 'readme' in m.name.lower() or m.name.endswith('.txt'):
                            f = tf.extractfile(m)
                            if f:
                                readme_content = f.read().decode('utf-8', errors='replace')[:1500]
                            break
    
        elif suffix == '.7z':
            with py7zr.SevenZipFile(path, 'r') as archive:
                names = archive.getnames()
                file_count = len(names)
                filelist = names[:max_files_list]
                readme_targets = [n for n in names
                                  if 'readme' in n.lower() or n.endswith('.txt')]
                if readme_targets:
                    data = archive.read(targets=readme_targets[:1])
                    readme_content = data[readme_targets[0]].read().decode('utf-8', errors='replace')[:1500]
    
    except Exception:
        # Corrupted or unsupported — fallback
        return f"{stem}\n\n[Archive: could not inspect — corrupted or unsupported format]"
    
    # Build embedding text
    parts = [stem]
    
    if is_encrypted:
        parts.append(f"[Encrypted archive: {file_count} files, {total_size // 1024}KB total]")
    else:
        if is_nested:
            parts.append(f"[Archive containing nested archives]")
        
        parts.append(f"Archive with {file_count} files ({total_size // 1024}KB):")
        parts.append("Contents: " + ", ".join(filelist[:30]))
        
        if file_count > max_files_list:
            parts.append(f"... and {file_count - max_files_list} more files")
        
        if readme_content:
            parts.append(f"\nREADME:\n{readme_content}")
    
    return "\n\n".join(parts)
```

#### Tier 2: Encrypted / Unreadable

```python
def build_encrypted_archive_embedding(path: str) -> str:
    """Fallback για encrypted/unreadable archives"""
    stem = Path(path).stem
    size_kb = Path(path).stat().st_size // 1024
    suffix = Path(path).suffix
    mtime = datetime.fromtimestamp(Path(path).stat().st_mtime)
    
    return (f"{stem}\n\n"
            f"[{suffix.upper()} archive: encrypted or unreadable, "
            f"{size_kb}KB, modified {mtime.strftime('%Y-%m')}]")
```

**Απόφαση:**
- Αν μπορούμε να επιθεωρήσουμε → Tier 1 (filelist + README)
- Αν encrypted/corrupted → Tier 2 (filename + size + date)
- Δεν στέλνουμε τίποτα στο `_Review` αυτόματα — το embedding με Tier 2 είναι αρκετό για clustering βάσει filename

---

### Δ. Security: Zip Bombs & Zip Slip

#### Zip Bomb Detection (χωρίς extraction)

```python
SAFE_RATIO_THRESHOLD = 50      # 50:1 = ύποπτο
SAFE_MAX_UNCOMPRESSED = 500 * 1024 * 1024  # 500 MB total limit
SAFE_MAX_FILES = 1000          # αρχεία μέσα

def check_zip_bomb(zf: zipfile.ZipFile) -> tuple[bool, str]:
    """
    Ελέγχει για zip bomb ΧΩΡΙΣ extraction.
    Επιστρέφει (is_safe, reason).
    ΣΗΜΑΝΤΙΚΟ: τα header values μπορεί να είναι spoofed —
    αυτή η check μπορεί να ξεγελαστεί από μοντέρνα zip bombs
    με overlapping entries. Χρήση ΜΟΝΟ ως πρώτο φίλτρο.
    """
    all_info = zf.infolist()
    
    if len(all_info) > SAFE_MAX_FILES:
        return False, f"Too many files: {len(all_info)}"
    
    total_uncompressed = sum(i.file_size for i in all_info)
    total_compressed = sum(i.compress_size for i in all_info)
    
    if total_uncompressed > SAFE_MAX_UNCOMPRESSED:
        return False, f"Total uncompressed size too large: {total_uncompressed // 1024 // 1024}MB"
    
    if total_compressed > 0:
        ratio = total_uncompressed / total_compressed
        if ratio > SAFE_RATIO_THRESHOLD:
            return False, f"Suspicious compression ratio: {ratio:.0f}:1"
    
    return True, "OK"

# Χρήση:
with zipfile.ZipFile(path) as zf:
    safe, reason = check_zip_bomb(zf)
    if not safe:
        # Μην κάνεις extraction — embed μόνο filename
        logger.warning(f"Potential zip bomb: {path} — {reason}")
        return build_encrypted_archive_embedding(path)
    # Safe to proceed...
```

**ΚΡΙΣΙΜΗ ΠΡΟΣΟΧΗ:** Μοντέρνα zip bombs (bamsoftware.com/hacks/zipbomb) χρησιμοποιούν overlapping file entries — τα compressed/uncompressed header values δεν δίνουν σωστό ratio! Αξιόπιστη προστασία μόνο με streaming byte counter κατά την εξαγωγή. Για τη χρήση μας (embedding μόνο, όχι full extraction), το header check είναι αρκετό ως πρώτο φίλτρο.

**Εναλλακτικά:** `pip install safezip` — drop-in wrapper για zipfile με:
- `max_per_member_ratio=50.0` (default)
- `max_total_ratio=50.0` (default)
- `max_file_size=100MB` per member
- `max_total_size=500MB` total
- `max_files=1000`

#### Zip Slip (Path Traversal) — ΜΟΝΟ αν κάνουμε extraction

```python
import os

def is_safe_path(base_dir: str, member_path: str) -> bool:
    """
    Ελέγχει ότι το extracted path δεν φεύγει εκτός base_dir.
    Αποτρέπει Zip Slip (../../../etc/passwd κλπ)
    """
    target = os.path.realpath(os.path.join(base_dir, member_path))
    return target.startswith(os.path.realpath(base_dir))

# Αν ποτέ χρειαστεί extraction (δεν συνιστάται για embeddings):
with zipfile.ZipFile(path) as zf:
    for member in zf.infolist():
        if not is_safe_path('/tmp/safe_extract', member.filename):
            logger.warning(f"Zip Slip attempt: {member.filename}")
            continue
        zf.extract(member, '/tmp/safe_extract')
```

**Για το Curator:** Επειδή κάνουμε ΜΟΝΟ in-memory inspection (δεν εξάγουμε στο disk), το Zip Slip δεν αποτελεί κίνδυνο. Αν ποτέ υλοποιηθεί extraction (π.χ. για Preview), χρειάζεται η παραπάνω validation.

---

### Ε. Library Selection Matrix

| Format | Primary Library | Fallback | Notes |
|---|---|---|---|
| **.zip** | `zipfile` (stdlib) | `safezip` για extra security | Πλήρες namelist + in-memory read |
| **.tar.gz / .tar.bz2 / .tar.xz** | `tarfile` (stdlib) | - | `r:*` mode = auto compression |
| **.7z** | `py7zr` | `libarchive-c` | Pure Python, χωρίς binary |
| **.rar** | `libarchive-c` | `rarfile` (needs unrar) | libarchive-c καλύτερο για macOS |
| **.tar.gz inside .zip** | Ανίχνευση μόνο | - | Μην κάνεις recursive extraction |
| **nested .zip** | Ανίχνευση μόνο | - | Flag στο embedding text |

**macOS installation:**
```bash
pip install py7zr rarfile libarchive-c safezip
# Αν θέλεις rarfile με unrar backend:
brew install unrar
# Ή unar (πιο πρόσφατο):
brew install unar
```

---

### ΣΤ. Concrete Implementation — Τελικό Pipeline

```python
import zipfile
import tarfile
import os
import py7zr
from pathlib import Path
from datetime import datetime
import logging

logger = logging.getLogger(__name__)

ARCHIVE_EXTENSIONS = {
    '.zip': 'zip',
    '.tar': 'tar',
    '.gz': 'tar',    # assume .tar.gz
    '.bz2': 'tar',
    '.xz': 'tar',
    '.7z': '7z',
    '.rar': 'rar',
}

SAFE_RATIO = 50
SAFE_MAX_UNCOMP = 500 * 1024 * 1024  # 500MB
SAFE_MAX_FILES = 1000
TEXT_EXTS = {'.txt', '.md', '.rst', '.readme', '.log', '.cfg', '.ini'}


def inspect_archive(path: str, max_list: int = 50) -> dict:
    """
    Επιθεωρεί archive χωρίς disk extraction.
    Returns dict: {filelist, readme, file_count, total_size, encrypted, nested, error}
    """
    result = {
        'filelist': [], 'readme': '', 'file_count': 0,
        'total_size': 0, 'encrypted': False, 'nested': False, 'error': None
    }
    p = Path(path)
    suffix = p.suffix.lower()
    name_lower = p.name.lower()

    try:
        # ZIP
        if suffix == '.zip':
            if not zipfile.is_zipfile(path):
                result['error'] = 'invalid_zip'; return result
            with zipfile.ZipFile(path, 'r') as zf:
                all_info = zf.infolist()
                result['file_count'] = len(all_info)
                result['total_size'] = sum(i.file_size for i in all_info)

                # Zip bomb check (header-based — fast but not foolproof)
                comp_total = sum(i.compress_size for i in all_info)
                if (result['file_count'] > SAFE_MAX_FILES or
                        result['total_size'] > SAFE_MAX_UNCOMP or
                        (comp_total > 0 and result['total_size'] / comp_total > SAFE_RATIO)):
                    result['error'] = 'zip_bomb_suspected'; return result

                names = zf.namelist()
                result['filelist'] = names[:max_list]
                result['nested'] = any(
                    n.lower().endswith(('.zip', '.tar.gz', '.7z'))
                    for n in names
                )

                for info in all_info:
                    if info.flag_bits & 0x1:
                        result['encrypted'] = True; break

                if not result['encrypted']:
                    for name in names:
                        ext = os.path.splitext(name.lower())[1]
                        if ext in TEXT_EXTS or 'readme' in name.lower():
                            try:
                                raw = zf.read(name)
                                result['readme'] = raw.decode('utf-8', errors='replace')[:1500]
                                break
                            except RuntimeError:
                                result['encrypted'] = True; break

        # TAR (gz/bz2/xz)
        elif suffix in ('.gz', '.bz2', '.xz', '.tar') or name_lower.endswith('.tar.gz'):
            if not tarfile.is_tarfile(path):
                result['error'] = 'invalid_tar'; return result
            with tarfile.open(path, 'r:*') as tf:
                members = tf.getmembers()
                result['file_count'] = len(members)
                result['total_size'] = sum(m.size for m in members)
                result['filelist'] = [m.name for m in members[:max_list]]
                for m in members:
                    if ('readme' in m.name.lower() or
                            os.path.splitext(m.name.lower())[1] in TEXT_EXTS):
                        try:
                            f = tf.extractfile(m)
                            if f:
                                result['readme'] = f.read().decode('utf-8', errors='replace')[:1500]
                        except Exception:
                            pass
                        break

        # 7Z
        elif suffix == '.7z':
            with py7zr.SevenZipFile(path, 'r') as a:
                names = a.getnames()
                result['file_count'] = len(names)
                result['filelist'] = names[:max_list]
                readme_targets = [n for n in names
                                  if 'readme' in n.lower() or
                                  os.path.splitext(n.lower())[1] in TEXT_EXTS]
                if readme_targets:
                    data = a.read(targets=readme_targets[:1])
                    result['readme'] = data[readme_targets[0]].read().decode('utf-8', errors='replace')[:1500]

        else:
            result['error'] = 'unsupported_format'

    except py7zr.exceptions.PasswordRequired:
        result['encrypted'] = True
    except Exception as e:
        result['error'] = str(e)

    return result


def build_archive_embedding(path: str) -> str:
    """Builds the final embedding text for an archive file."""
    stem = Path(path).stem
    info = inspect_archive(path)

    if info['error'] in ('invalid_zip', 'invalid_tar'):
        return f"{stem}\n\n[Archive: invalid or corrupted file]"

    if info['error'] == 'zip_bomb_suspected':
        return f"{stem}\n\n[Archive: unusually large decompression ratio — not inspected]"

    if info['error'] == 'unsupported_format':
        sz = Path(path).stat().st_size // 1024
        return f"{stem}\n\n[Archive: unsupported format, {sz}KB]"

    if info['encrypted']:
        sz = Path(path).stat().st_size // 1024
        return f"{stem}\n\n[Encrypted archive: {info['file_count']} files, {sz}KB]"

    parts = [stem]
    notes = []
    if info['nested']:
        notes.append("contains nested archives")
    if notes:
        parts.append(f"[Note: {', '.join(notes)}]")

    parts.append(f"Archive: {info['file_count']} files, {info['total_size'] // 1024}KB")
    parts.append("Contents: " + ", ".join(info['filelist'][:30]))

    if info['file_count'] > 30:
        parts.append(f"... and {info['file_count'] - 30} more files")

    if info['readme']:
        parts.append(f"README:\n{info['readme']}")

    return "\n\n".join(parts)
```

---

### Ζ. Integration στο Curator Pipeline

Στο Μέρος 12 (Embedding Strategy Pipeline Table), η γραμμή ZIP αλλάζει:

| Τύπος | Εξαγωγή | Embedding Input |
|---|---|---|
| ZIP / .tar.gz / .7z / .rar | `inspect_archive()` — filelist + README in memory | `build_archive_embedding(path)` |

**Σειρά εφαρμογής στο scan:**
1. `Path(f).suffix.lower() in ARCHIVE_EXTENSIONS` → ναι = archive
2. `inspect_archive(path)` → dict
3. `build_archive_embedding(path)` → text
4. Embed με BGE-M3 → clustering

**Performance εκτίμηση για 5000 αρχεία με ~500 archives:**
- `inspect_archive()` per ZIP: ~50-200ms (network I/O bound αν σε USB, αλλιώς <50ms)
- Δεν γράφει τίποτα στο disk ✅
- Memory peak: ~2MB ανά archive (README content + filelist)
- Total για 500 archives: ~25-100 seconds — αποδεκτό

---

### Η. Gaps & Novel Contribution

1. **Κανένα file organizer** (QiuYannnn, messy-folder-reorganizer, different-ai) δεν κάνει intelligent archive inspection για embeddings — όλοι τα skip ή τα categorize ως "Compressed Files"
2. **filelist-based embedding** για archives είναι novel approach — δεν υπάρχει paper γι' αυτό
3. **Nested archive detection** ως embedding feature → cluster με "developer tools", "backup bundles" κλπ

---

**Sources Topic 2:**
- Python zipfile docs — https://docs.python.org/3/library/zipfile.html
- Python tarfile docs — https://docs.python.org/3/library/tarfile.html
- py7zr PyPI — https://pypi.org/project/py7zr/
- rarfile FAQ — https://rarfile.readthedocs.io/faq.html
- libarchive-c PyPI — https://pypi.org/project/libarchive-c/
- safezip GitHub — https://github.com/barseghyanartur/safezip
- safezip docs — https://safezip.readthedocs.io/en/latest/
- zipguard — https://hivesecurity.gitlab.io/blog/zipguard-safe-zip-extraction-python/
- sunzip — https://github.com/twbgc/sunzip
- Zip bomb deep dive — https://www.bamsoftware.com/hacks/zipbomb/
- CVE-2024-55587 (libarchive path traversal) — https://security.snyk.io/vuln/SNYK-PYTHON-PYTHONLIBARCHIVE-8499646
- Python zipfile security pitfalls — https://runebook.dev/en/docs/python/library/zipfile/decompression-pitfalls

---
