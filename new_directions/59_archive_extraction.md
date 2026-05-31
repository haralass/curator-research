# 59 — Archive Content Extraction Pipeline for Curator
_Research date: 2026-05-31_

---

## Why Archives Need Special Treatment

~30% of files in a typical macOS Downloads folder are archives (ZIP, tar.gz, .rar, .7z, .dmg, .cab, etc.). Curator's current approach — filename-only embedding — produces extremely poor semantic representations. A file named `project_backup_2024.zip` and `react-dashboard-v2.zip` would embed similarly despite containing entirely different things.

Archives are containers whose meaning lies entirely inside them. Without content-aware extraction, Curator cannot:
- Distinguish a code project ZIP from a document bundle ZIP
- Surface the actual application name from a `.dmg`
- Identify whether a `.tar.gz` contains a dataset, a library, or a photo backup
- Route archives to correct categories (Dev Tools vs. Documents vs. Media)

The solution: read the archive's central directory and sample text content **in memory, without full extraction to disk**.

---

## Python Standard Library: zipfile and tarfile

### zipfile

The `zipfile` module (stdlib, no install) handles `.zip` files. All reading can be done without writing to disk.

```python
import zipfile

# Validate before opening
if zipfile.is_zipfile(path):  # checks magic bytes, not extension
    ...

# List members — reads ONLY the central directory at end of file
# For a 500 MB ZIP this is nearly instantaneous (no decompression)
with zipfile.ZipFile(path, 'r') as zf:
    names = zf.namelist()    # list of strings
    infos = zf.infolist()    # list of ZipInfo objects with metadata

# Read a specific member in-memory (no extraction to disk)
with zipfile.ZipFile(path, 'r') as zf:
    with zf.open('README.md') as f:
        content = f.read(4096)  # read first 4KB ONLY — do NOT read() the whole thing
```

**ZipInfo attributes useful for Curator:**
```python
info.filename        # path within archive
info.file_size       # uncompressed size in bytes
info.compress_size   # compressed size on disk
info.date_time       # (year, month, day, hour, minute, second)
info.flag_bits       # bitmask — bit 0 indicates encryption
info.is_dir()        # True if directory entry
```

**Encryption detection (no extraction needed):**
```python
def zip_is_encrypted(path: str) -> bool:
    try:
        with zipfile.ZipFile(path, 'r') as zf:
            return any(bool(info.flag_bits & 0x1) for info in zf.infolist())
    except zipfile.BadZipFile:
        return False
```

**Corruption testing:**
```python
with zipfile.ZipFile(path) as zf:
    bad_file = zf.testzip()  # None if all OK, name of first bad member if corrupted
    # namelist() and infolist() still work even if some members are corrupted
```

**Exceptions to handle:** `BadZipFile`, `LargeZipFile`, `RuntimeError` (encrypted member without password), `NotImplementedError` (unsupported compression)

---

### tarfile

Handles `.tar`, `.tar.gz` (`.tgz`), `.tar.bz2`, `.tar.xz`.

```python
import tarfile

tarfile.is_tarfile(path)  # validation

with tarfile.open(path, 'r:*') as tf:  # auto-detects compression
    members = tf.getmembers()   # list of TarInfo objects
    names   = tf.getnames()     # list of strings

# Read a member without extraction
with tarfile.open(path, 'r:*') as tf:
    member = tf.getmember('README.md')
    f = tf.extractfile(member)  # returns file-like object or None (dirs/symlinks)
    if f:
        content = f.read(4096)
        f.close()
```

**TarInfo attributes:** `name`, `size`, `mtime`, `isdir()`, `isfile()`, `issym()`

**⚠️ Critical performance warning:** `.tar.gz` has no random-access index. `getmembers()` must stream the **entire** archive to enumerate all members. A 300 MB `.tar.gz` with 10,000 files can take 3–8 seconds.

**Mitigation — iterate lazily:**
```python
with tarfile.open(path, 'r:*') as tf:
    names = []
    for member in tf:   # lazy iteration
        names.append(member.name)
        if len(names) > 500:   # stop early — seen enough structure
            break
```

**Exceptions to handle:** `tarfile.ReadError`, `tarfile.CompressionError`, `EOFError`, `zlib.error`

---

## libarchive for Non-Standard Formats (.rar, .7z, .cab)

### Format coverage

| Format | zipfile | tarfile | libarchive |
|--------|---------|---------|------------|
| .zip | ✅ | ❌ | ✅ |
| .tar.gz | ❌ | ✅ | ✅ |
| .7z | ❌ | ❌ | ✅ |
| .rar | ❌ | ❌ | ✅ (read only) |
| .cab | ❌ | ❌ | ✅ |
| .iso | ❌ | ❌ | ✅ |
| .dmg | ❌ | ❌ | ❌ (separate tool) |

### macOS installation

```bash
brew install libarchive
pip install libarchive-c

# Point Python bindings at Homebrew version (Apple Silicon)
export LIBARCHIVE=/opt/homebrew/opt/libarchive/lib/libarchive.dylib
```

Or in Python code before importing:
```python
import os
os.environ['LIBARCHIVE'] = '/opt/homebrew/opt/libarchive/lib/libarchive.dylib'
import libarchive
```

### API

```python
import libarchive

with libarchive.file_reader('archive.7z') as archive:
    for entry in archive:
        print(entry.pathname, entry.size, entry.isdir)
        if getattr(entry, 'encrypted', False):
            continue  # encrypted entry, skip
        # Read first 4096 bytes of content
        chunks = []
        for block in entry.get_blocks():
            chunks.append(block)
            if sum(len(c) for c in chunks) >= 4096:
                break
        content_head = b''.join(chunks)[:4096]
```

**Key limitation:** libarchive is strictly streaming — you cannot seek to a specific entry by name without iterating from the start.

**RAR note:** libarchive reads RAR4 and RAR5 in listing mode; full content extraction may require the `unrar` native binary for some RAR5 features.

---

## MarkItDown: Does It Handle Archives?

**Yes, for ZIP only, with significant caveats.**

MarkItDown (Microsoft, v0.1.4) includes a `ZipConverter` that iterates all ZIP members using `zipfile`, recursively applies the appropriate converter to each contained file, and concatenates all Markdown output.

```python
from markitdown import MarkItDown
md = MarkItDown(enable_plugins=False)
result = md.convert("archive.zip")
print(result.text_content)
```

**What it does well:** ZIP + recursive document content (DOCX, XLSX, PDF, HTML, CSV, JSON members)

**What it does NOT do:**
- Does not handle `.7z`, `.rar`, `.tar.gz`, `.cab` — ZIP only
- No size guard — will attempt to fully decompress every member
- No encryption detection — throws on encrypted members
- No partial reads — reads entire members into memory
- `.dmg` not supported

**Verdict for Curator:** Use MarkItDown selectively for small, clean, document-heavy ZIPs (< 20 MB), not as the primary pipeline component.

---

## Semantic Representation Strategy

Given a mixed-content archive, ranked options for producing an embedding-ready text string:

### Option 1 (Best): Structured manifest + README + sampled text files
```
[ARCHIVE:ZIP] react-dashboard-v2.zip (4.2 MB, 87 files)
[EXTENSIONS] .tsx (34), .js (18), .css (12), .md (8), .json (5)
[SAMPLE:README.md] React dashboard component library. Requires Node 18+. Install with npm install.
[SAMPLE:src/index.tsx] import React from 'react'; export const Dashboard...
[SAMPLE:CHANGELOG.md] v2.0.0 - Added dark mode, fixed responsive layout...
```

Captures: archive identity, internal structure, purpose, code flavor. ~500–800 tokens.

### Option 2 (Good): Extension summary + README
```
[ARCHIVE:TAR.GZ] dataset-mnist-subset.tar.gz (280 MB, 12000 files)
[EXTENSIONS] .png (11998), .csv (1), .txt (1)
[SAMPLE:README.txt] MNIST handwritten digit dataset, subset of 2000 samples per class.
```

Extension summary alone is highly informative: 200 `.py` files = code project; 200 `.jpg` files = photo backup; `.csv` + `.json` = dataset.

### Option 3 (Acceptable): File listing only
```
[ARCHIVE:ZIP] project.zip (8.3 MB, 34 files)
[FILES] README.md, setup.py, requirements.txt, src/main.py, src/utils.py, tests/test_main.py...
```

No I/O beyond central directory. For large/encrypted/failing archives.

### Option 4 (Last resort): Filename only
Current Curator behavior. Avoid if at all possible.

### Priority order for content sampling

When reading individual members, prioritize:
1. `README.md`, `README.txt`, `README.rst` — purpose description
2. `CHANGELOG.md`, `CHANGES.txt` — version/project context
3. `requirements.txt`, `package.json`, `pyproject.toml`, `Cargo.toml` — tech stack
4. `*.md` files in root — documentation
5. First `.py`, `.js`, `.ts`, `.swift`, `.go` file — code flavor
6. `LICENSE` — project type signal

**Limit each sampled file to 500 characters × 5 files maximum = 2500 chars per archive.**

---

## Encrypted Archive Detection

**ZIP** — via `flag_bits & 0x1` (see above, reads central directory only, no decompression):
```python
def zip_is_encrypted(path: str) -> bool:
    with zipfile.ZipFile(path, 'r') as zf:
        return any(bool(info.flag_bits & 0x1) for info in zf.infolist())
```

**TAR** — tar archives don't natively support encryption; if encrypted, it was done post-creation with GPG (`.tar.gz.gpg`). Detect via GPG magic bytes.

**7z / RAR** — via `libarchive`:
```python
with libarchive.file_reader(path) as archive:
    for entry in archive:
        if getattr(entry, 'encrypted', False):
            return True  # encrypted
```

---

## Corrupted Archive Handling

Never let a single corrupted archive crash the pipeline:

```python
def safe_read_member(zf: zipfile.ZipFile, name: str, max_bytes: int = 500) -> str:
    try:
        with zf.open(name) as f:
            return f.read(max_bytes).decode('utf-8', errors='replace')
    except Exception:
        return ''  # silently skip unreadable members
```

For tar: `extractfile()` returns `None` for non-regular members; catch `KeyError` if member name doesn't exist; catch `zlib.error` for corrupted content.

---

## macOS .dmg Files

### The challenge

`.dmg` files contain an HFS+/APFS filesystem image. No pure-Python library reads them without mounting.

### Option 1: `hdiutil imageinfo` via subprocess — **no mount needed**

```python
import subprocess, plistlib

result = subprocess.run(
    ['hdiutil', 'imageinfo', '-plist', path],
    capture_output=True, timeout=10
)
info = plistlib.loads(result.stdout)

fmt = info.get('Format', 'unknown')       # e.g., 'UDZO'
encrypted = info.get('Encrypted', False)  # bool

# Volume name (= app name) from partition table
volume_name = ''
for p in info.get('partitions', {}).get('partitions', []):
    name = p.get('partition-name', '')
    if name and name not in ('', 'Apple_Free', 'Apple_partition_map'):
        volume_name = name
        break
```

Output: `[ARCHIVE:DMG] MyApp-2.1.dmg (145 MB, format: UDZO, volume: MyApp 2.1)` — the volume name alone is often sufficient for routing.

### Option 2: `dmglib` — mounts temporarily

```bash
pip install dmglib
```

```python
from dmglib import attachedDiskImage

with attachedDiskImage('/path/to/app.dmg') as mount_points:
    for mp in mount_points:
        # mp is a path like /Volumes/MyApp
        import os
        for root, dirs, files in os.walk(mp):
            for fname in files:
                print(os.path.join(root, fname))
```

Use for small (< 200 MB) unencrypted DMGs where the volume name isn't sufficient. Does write to disk (mounts image).

---

## Complete Extraction Pipeline for Curator

```python
"""
curator/core/archive_extractor.py
Produces an embedding-ready text string from any archive file.
"""

import os, zipfile, tarfile, zlib, subprocess, plistlib
from pathlib import Path
from typing import Optional

try:
    import libarchive
    LIBARCHIVE_AVAILABLE = True
except ImportError:
    LIBARCHIVE_AVAILABLE = False

MAX_ARCHIVE_SIZE_BYTES = 500 * 1024 * 1024   # 500 MB
MAX_MEMBER_SAMPLE_BYTES = 500
MAX_FILES_TO_SAMPLE = 5
PRIORITY_FILENAMES = {
    'readme.md', 'readme.txt', 'readme.rst', 'readme',
    'changelog.md', 'changes.txt', 'requirements.txt',
    'package.json', 'pyproject.toml', 'cargo.toml', 'pipfile',
    'license', 'license.txt', 'license.md',
}
TEXT_EXTENSIONS = {
    '.py', '.js', '.ts', '.tsx', '.jsx', '.swift', '.go', '.rs',
    '.java', '.kt', '.c', '.cpp', '.h', '.cs', '.rb', '.php',
    '.md', '.txt', '.rst', '.html', '.css', '.json', '.yaml',
    '.yml', '.toml', '.ini', '.cfg', '.sh',
}

def extract_archive_text(path: str) -> str:
    size = os.path.getsize(path)
    suffix = Path(path).suffix.lower()

    # Size guard
    if size > MAX_ARCHIVE_SIZE_BYTES:
        return f"[ARCHIVE] {Path(path).name} ({size/1024/1024:.0f} MB) [TOO LARGE — filename-only]"

    # DMG
    if suffix == '.dmg':
        result = _extract_dmg(path)
        if result: return result

    # ZIP
    if suffix == '.zip':
        result = _extract_zip(path)
        if result: return result

    # TAR family
    if suffix in ('.tar', '.gz', '.bz2', '.xz', '.tgz', '.tbz2', '.txz'):
        result = _extract_tar(path)
        if result: return result

    # libarchive fallback
    result = _extract_libarchive(path)
    if result: return result

    return f"[ARCHIVE] {Path(path).name} ({size/1024/1024:.1f} MB) [FORMAT UNSUPPORTED]"


def _ext_summary(names: list[str]) -> str:
    from collections import Counter
    exts = Counter(Path(n).suffix.lower() for n in names if not n.endswith('/'))
    return ', '.join(f'{e or "[no ext]"} ({c})' for e, c in exts.most_common(8))


def _priority_sort(names: list[str]) -> list[str]:
    def score(n):
        if Path(n).name.lower() in PRIORITY_FILENAMES: return 0
        if Path(n).suffix.lower() in TEXT_EXTENSIONS: return 1
        return 2
    return sorted(names, key=score)


def _decode(data: bytes) -> str:
    return data.decode('utf-8', errors='replace').strip()


def _extract_zip(path: str) -> Optional[str]:
    if not zipfile.is_zipfile(path):
        return None
    try:
        with zipfile.ZipFile(path, 'r') as zf:
            infos = zf.infolist()
            names = [i.filename for i in infos if not i.is_dir()]
            encrypted = [i for i in infos if bool(i.flag_bits & 0x1)]
            total_mb = os.path.getsize(path) / 1024 / 1024
            if len(encrypted) == len(infos):
                return f"[ARCHIVE:ZIP] {Path(path).name} ({total_mb:.1f} MB, {len(names)} files) [ENCRYPTED]"
            lines = [
                f"[ARCHIVE:ZIP] {Path(path).name} ({total_mb:.1f} MB, {len(names)} files)",
                f"[EXTENSIONS] {_ext_summary(names)}",
            ]
            sampled = 0
            for name in _priority_sort(names):
                if sampled >= MAX_FILES_TO_SAMPLE: break
                if Path(name).suffix.lower() not in TEXT_EXTENSIONS and \
                   Path(name).name.lower() not in PRIORITY_FILENAMES: continue
                try:
                    with zf.open(name) as f:
                        text = _decode(f.read(MAX_MEMBER_SAMPLE_BYTES * 3))[:MAX_MEMBER_SAMPLE_BYTES]
                    if text:
                        lines.append(f"[SAMPLE:{name}] {text}")
                        sampled += 1
                except Exception:
                    continue
            return '\n'.join(lines)
    except zipfile.BadZipFile:
        return f"[ARCHIVE:ZIP] {Path(path).name} [CORRUPTED]"
    except Exception as e:
        return f"[ARCHIVE:ZIP] {Path(path).name} [ERROR: {type(e).__name__}]"


def _extract_tar(path: str) -> Optional[str]:
    if not tarfile.is_tarfile(path):
        return None
    try:
        with tarfile.open(path, 'r:*') as tf:
            members, names, lines, sampled = [], [], [], 0
            for m in tf:   # lazy iteration — stop early for large archives
                if m.isfile():
                    members.append(m)
                    names.append(m.name)
                if len(names) > 500: break
            fmt = ('TAR.GZ' if path.endswith(('.tar.gz', '.tgz')) else
                   'TAR.BZ2' if path.endswith(('.tar.bz2',)) else
                   'TAR.XZ' if path.endswith(('.tar.xz',)) else 'TAR')
            total_mb = os.path.getsize(path) / 1024 / 1024
            lines = [
                f"[ARCHIVE:{fmt}] {Path(path).name} ({total_mb:.1f} MB, {len(names)}+ files)",
                f"[EXTENSIONS] {_ext_summary(names)}",
            ]
            for m in sorted(members, key=lambda m: (
                0 if Path(m.name).name.lower() in PRIORITY_FILENAMES else
                1 if Path(m.name).suffix.lower() in TEXT_EXTENSIONS else 2
            )):
                if sampled >= MAX_FILES_TO_SAMPLE: break
                if Path(m.name).suffix.lower() not in TEXT_EXTENSIONS and \
                   Path(m.name).name.lower() not in PRIORITY_FILENAMES: continue
                try:
                    f = tf.extractfile(m)
                    if f is None: continue
                    text = _decode(f.read(MAX_MEMBER_SAMPLE_BYTES * 3))[:MAX_MEMBER_SAMPLE_BYTES]
                    f.close()
                    if text:
                        lines.append(f"[SAMPLE:{m.name}] {text}")
                        sampled += 1
                except Exception:
                    continue
            return '\n'.join(lines)
    except (tarfile.ReadError, tarfile.CompressionError):
        return f"[ARCHIVE:TAR] {Path(path).name} [CORRUPTED]"
    except Exception as e:
        return f"[ARCHIVE:TAR] {Path(path).name} [ERROR: {type(e).__name__}]"


def _extract_libarchive(path: str) -> Optional[str]:
    if not LIBARCHIVE_AVAILABLE: return None
    try:
        names, samples, sampled = [], [], 0
        encrypted_count = 0
        with libarchive.file_reader(path) as archive:
            for entry in archive:
                if entry.isdir: continue
                names.append(entry.pathname)
                if getattr(entry, 'encrypted', False):
                    encrypted_count += 1
                    continue
                if sampled < MAX_FILES_TO_SAMPLE and (
                    Path(entry.pathname).suffix.lower() in TEXT_EXTENSIONS or
                    Path(entry.pathname).name.lower() in PRIORITY_FILENAMES
                ):
                    try:
                        chunks, total = [], 0
                        for block in entry.get_blocks():
                            chunks.append(block)
                            total += len(block)
                            if total >= MAX_MEMBER_SAMPLE_BYTES * 3: break
                        text = _decode(b''.join(chunks))[:MAX_MEMBER_SAMPLE_BYTES]
                        if text:
                            samples.append(f"[SAMPLE:{entry.pathname}] {text}")
                            sampled += 1
                    except Exception:
                        pass
        ext = Path(path).suffix.lower().lstrip('.')
        total_mb = os.path.getsize(path) / 1024 / 1024
        lines = [
            f"[ARCHIVE:{ext.upper()}] {Path(path).name} ({total_mb:.1f} MB, {len(names)} files)",
            f"[EXTENSIONS] {_ext_summary(names)}",
        ]
        if encrypted_count: lines.append(f"[NOTE] {encrypted_count} encrypted entries")
        lines.extend(samples)
        return '\n'.join(lines)
    except Exception as e:
        return f"[ARCHIVE] {Path(path).name} [LIBARCHIVE ERROR: {type(e).__name__}]"


def _extract_dmg(path: str) -> Optional[str]:
    try:
        result = subprocess.run(
            ['hdiutil', 'imageinfo', '-plist', path],
            capture_output=True, timeout=10
        )
        if result.returncode != 0: return None
        info = plistlib.loads(result.stdout)
        fmt = info.get('Format', 'unknown')
        encrypted = info.get('Encrypted', False)
        total_mb = os.path.getsize(path) / 1024 / 1024
        volume_name = ''
        for p in info.get('partitions', {}).get('partitions', []):
            name = p.get('partition-name', '')
            if name and name not in ('', 'Apple_Free', 'Apple_partition_map'):
                volume_name = name
                break
        enc_str = ', ENCRYPTED' if encrypted else ''
        name_str = f', volume: {volume_name}' if volume_name else ''
        return f"[ARCHIVE:DMG] {Path(path).name} ({total_mb:.1f} MB, format: {fmt}{enc_str}{name_str})"
    except Exception as e:
        return f"[ARCHIVE:DMG] {Path(path).name} [ERROR: {type(e).__name__}]"
```

---

## Performance Guidelines

| Operation | Cost | Notes |
|---|---|---|
| `zipfile.is_zipfile()` | ~0 ms | Reads 22-byte end-of-central-directory |
| `ZipFile.namelist()` (1000-file ZIP) | < 5 ms | Reads central directory only |
| `ZipFile.open(name).read(500)` | 1–50 ms | Decompresses minimum needed |
| `tarfile.open()` + `getmembers()` (300 MB .tar.gz) | 3–8 s | Must stream entire archive ⚠️ |
| `libarchive` iteration | 10–500 ms | Streaming, low memory |
| `hdiutil imageinfo` | ~200 ms | subprocess, no mount |

### Size thresholds (recommended)

| Archive size | Strategy |
|---|---|
| < 50 MB | Full pipeline: list all + sample up to 5 text files |
| 50–200 MB | List up to 500 members (lazy stop), sample up to 3 text files |
| 200–500 MB | Manifest only: list members, no content sampling |
| > 500 MB | Filename + size only — skip archive parsing entirely |

**Use `libarchive` over `tarfile` for `.tar.gz` > 50 MB** to avoid the full decompression-to-enumerate problem.

---

## Key References

- Python zipfile module: https://docs.python.org/3/library/zipfile.html
- Python tarfile module: https://docs.python.org/3/library/tarfile.html
- python-libarchive-c GitHub: https://github.com/Changaco/python-libarchive-c
- libarchive-c on PyPI: https://pypi.org/project/libarchive-c/
- libarchive supported formats: https://github.com/libarchive/libarchive/wiki/LibarchiveFormats
- microsoft/markitdown: https://github.com/microsoft/markitdown
- dmglib (macOS DMG Python wrapper): https://github.com/0xbf00/dmglib
- hdiutil man page: https://ss64.com/osx/hdiutil.html
- pyzipper (encrypted ZIP handling): https://pypi.org/project/pyzipper/
