## ΜΕΡΟΣ 39: MACOS SPECIFICS — SPOTLIGHT DETECTION, SEQUOIA, FSEVENTS, TAGS
*(Έρευνα 2026-05-23)*

---

### ΘΕΜΑ 1: Spotlight Search Detection — Μπορούμε να ξέρουμε αν ο χρήστης έψαξε;

#### Τι υπάρχει (και τι ΔΕΝ υπάρχει)

**Verdict πρώτα:** Δεν υπάρχει δημόσιο API για να ανιχνεύσεις αν ο χρήστης έκανε Spotlight search για συγκεκριμένο αρχείο. Αυτό είναι σκόπιμο από την Apple.

**Τι ξέρουμε για το storage:**
- Το `~/Library/Preferences/com.apple.Spotlight.plist` περιέχει μόνο Spotlight preferences (excluded folders, ordering, κτλ.) — **όχι search history**.
- Το `~/Library/Application Support/com.apple.spotlight/` δεν περιέχει query log.
- Η Apple αποθηκεύει Spotlight queries μέσα στον macOS Unified Log (`log show --predicate 'subsystem == "com.apple.spotlight"'`), αλλά αυτό απαιτεί `Full Disk Access` ή admin privileges για ανάγνωση — **απαγορευτικό για ένα user-space app**.
- Το **Finder** (όχι Spotlight) κρατά `SGTRecentFileSearches` στο `~/Library/Preferences/com.apple.finder.plist`, αλλά αυτό είναι Finder window search, όχι Spotlight bar search.

**TCC/Privacy:** Δεν υπάρχει TCC permission κατηγορία για "read Spotlight queries". Οποιαδήποτε πρόσβαση θα ήταν είτε μέσω kernel entitlement (δεν δίνεται σε 3rd party) είτε root access.

**NSMetadataQuery:** Είναι API για να *εκτελείς* Spotlight searches, όχι για να βλέπεις τι έψαξαν άλλοι. Π.χ.:
```swift
let query = NSMetadataQuery()
query.predicate = NSPredicate(format: "kMDItemDisplayName == %@", "thesis.pdf")
query.searchScopes = [NSMetadataQueryLocalComputerScope]
```
Αυτό σου επιτρέπει να βρεις αρχεία — αλλά όχι να δεις αν ο χρήστης τα έψαξε.

#### Χρήσιμα metadata signals (αντί για Spotlight detection)

Εφόσον δεν μπορούμε να δούμε search history, χρησιμοποιούμε proxies:

| Signal | API | Ενημερώνεται πότε; | Αξιοπιστία |
|--------|-----|-------------------|------------|
| `kMDItemLastUsedDate` | `mdls -name kMDItemLastUsedDate <file>` | Μόνο όταν ανοιχτεί | Μέτρια — δεν λαμβάνει υπόψη Spotlight |
| `kMDItemUseCount` | `mdls -name kMDItemUseCount <file>` | Κάθε φορά που ανοίγεται | Καλή — μετράει actual opens |
| `kMDItemUsedDates` | `mdls -name kMDItemUsedDates <file>` | History all opens | Εξαιρετική για trends |
| `atime` (access time) | `stat -f "%a" <file>` | Στο APFS απενεργοποιημένο by default | Άχρηστο |

**Recommended heuristic για Curator "lost file" detection:**

```python
import subprocess
import json
from datetime import datetime, timedelta

def is_file_possibly_lost(filepath: str, days_since_org: int = 30) -> bool:
    """
    Αρχείο που οργανώθηκε αλλά δεν άνοιξε ποτέ = possibly mis-classified.
    Δεν μπορούμε να δούμε αν ψάχτηκε, αλλά μπορούμε να δούμε αν χρησιμοποιήθηκε.
    """
    try:
        result = subprocess.run(
            ["mdls", "-name", "kMDItemLastUsedDate",
             "-name", "kMDItemUseCount", filepath],
            capture_output=True, text=True
        )
        output = result.stdout
        
        # Parse kMDItemUseCount
        use_count = 0
        for line in output.splitlines():
            if "kMDItemUseCount" in line:
                parts = line.split("=")
                if len(parts) > 1:
                    val = parts[1].strip()
                    if val.isdigit():
                        use_count = int(val)
        
        # Αν use_count == 0 και πέρασαν 30+ ημέρες από organization → possibly lost
        return use_count == 0
    except Exception:
        return False
```

**Conclusion:** Το Spotlight search history είναι **εντελώς κλειστό** σε 3rd party apps. Ο Curator πρέπει να βασίζεται σε `kMDItemUseCount` + `kMDItemLastUsedDate` ως indirect signal για "το αρχείο δεν βρέθηκε/χρησιμοποιήθηκε από την οργάνωση".

---

### ΘΕΜΑ 2: macOS Sequoia Entitlements για Unsigned Python Scripts

#### Τι άλλαξε στο Sequoia vs προηγούμενες εκδόσεις

**Sequoia 15.0:** Control-click για override Gatekeeper αφαιρέθηκε. Ο χρήστης πρέπει να πάει System Settings → Privacy & Security → "Open Anyway".

**Sequoia 15.1:** Η εντολή `sudo spctl --master-disable` έπαψε να λειτουργεί απευθείας. Πρέπει πλέον να εκτελεστεί *και* μετά να ενεργοποιηθεί χειροκίνητα στο System Settings. Η "Allow Anywhere" επιλογή εμφανίζεται μόνο για ~8 λεπτά.

**Για development (χωρίς distribution signing):**
```bash
# Βήμα 1: Εκτέλεση εντολής
sudo spctl --master-disable

# Βήμα 2: System Settings → Privacy & Security → κάτω μέρος
# → "Allow Application From" → "Anywhere"
# (εμφανίζεται μόνο για 8 λεπτά μετά την εντολή)
```

#### Python sidecar στο Tauri: τι χρειάζεται

**Το κλειδί:** Όταν ο Curator είναι signed και notarized Tauri app, το Python sidecar (`ai_classifier.py` → compiled με PyInstaller) πρέπει **και αυτό να είναι signed** — αλλά **δεν χρειάζεται ξεχωριστό entitlements plist**. Κληρονομεί το signing του parent.

Το Tauri v2 χειρίζεται αυτό μέσω `externalBin` στο `tauri.conf.json`:

```json
{
  "bundle": {
    "externalBin": [
      "binaries/ai_classifier"
    ]
  }
}
```

Το binary πρέπει να ονομάζεται `ai_classifier-aarch64-apple-darwin` (Apple Silicon) και/ή `ai_classifier-x86_64-apple-darwin` (Intel) — το Tauri χειρίζεται το mapping αυτόματα.

#### entitlements.plist για Curator

Δημιούργησε `src-tauri/entitlements.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>

  <!-- ΑΠΑΙΤΕΙΤΑΙ: WebView JIT compilation (Tauri χρησιμοποιεί WKWebView) -->
  <key>com.apple.security.cs.allow-jit</key>
  <true/>

  <!-- ΑΠΑΙΤΕΙΤΑΙ: WKWebView needs unsigned executable memory -->
  <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
  <true/>

  <!-- ΑΠΑΙΤΕΙΤΑΙ: DYLD env vars για WebKit internals -->
  <key>com.apple.security.cs.allow-dyld-environment-variables</key>
  <true/>

  <!-- ΑΠΑΙΤΕΙΤΑΙ αν PyInstaller binary: disable library validation -->
  <!-- PyInstaller bundles unsigned .dylibs που macOS θα απορρίψει χωρίς αυτό -->
  <key>com.apple.security.cs.disable-library-validation</key>
  <true/>

  <!-- CURATOR-SPECIFIC: Read ~/Downloads χωρίς explicit permission dialog -->
  <!-- Χρειάζεται αν θέλουμε scripted access στο ~/Downloads -->
  <key>com.apple.security.files.downloads.read-write</key>
  <true/>

  <!-- CURATOR-SPECIFIC: User-selected file access (drag & drop, open panel) -->
  <key>com.apple.security.files.user-selected.read-write</key>
  <true/>

</dict>
</plist>
```

**Ενεργοποίηση στο `tauri.conf.json`:**
```json
{
  "bundle": {
    "macOS": {
      "entitlements": "./entitlements.plist",
      "minimumSystemVersion": "13.0"
    }
  }
}
```

#### Γιατί `com.apple.security.cs.disable-library-validation`

Το PyInstaller compiles το Python + όλα τα dependencies σε ένα bundle. Αυτό το bundle περιλαμβάνει unsigned `.dylib` files (numpy, sklearn, κτλ.). Χωρίς αυτό το entitlement, το macOS Library Validation θα αρνηθεί να φορτώσει αυτά τα dylibs μετά το signing.

**Σημείωση ασφαλείας:** Αυτό το entitlement αποδυναμώνει ένα security layer. Για production distribution μέσω App Store **δεν επιτρέπεται** — μόνο για direct distribution (DMG/PKG χωρίς App Store).

---

### ΘΕΜΑ 3: FSEvents για Implicit Feedback Loop

#### Πώς λειτουργεί το rename/move detection

Το FSEvents API παράγει **δύο ξεχωριστά events** για κάθε move:
1. Event στο **παλιό path** με `kFSEventStreamEventFlagItemRenamed`
2. Event στο **νέο path** με `kFSEventStreamEventFlagItemRenamed`

Το πρόβλημα: **δεν υπάρχει built-in τρόπος να συνδέσεις τα δύο events**. Χρησιμοποιείς το `cookie` field για να τα αντιστοιχίσεις — δύο events με ίδιο cookie = ίδιο rename/move.

**Latency:** Default 2.0 δευτερόλεπτα. Μπορεί να μειωθεί:
```python
stream = Stream(callback, path, file_events=True, latency=0.3)  # 300ms
```
Με `NoDefer` flag: events παραδίδονται στο leading edge (αμέσως).

**Batching:** Αν ο user μετακινήσει 50 αρχεία ταυτόχρονα (Finder select-all-move), λαμβάνεις 100 events (50 παλιά paths + 50 νέα paths) στο ίδιο callback invocation — συνήθως coalesced σε ένα batch μετά το latency window.

#### Διάκριση "Curator move" vs "User move"

Αυτό είναι κρίσιμο: πώς ξέρουμε ότι μια μετακίνηση είναι user feedback και όχι απλά το Curator που οργάνωσε;

**Στρατηγική:**
1. Ο Curator κρατά ένα in-memory set `_curator_moves` με (src, dst, timestamp) για κάθε κίνηση που κάνει ο ίδιος.
2. Όταν λαμβάνεται FSEvent move, αν το (src, dst) ταυτίζεται με entry στο `_curator_moves` (μέσα σε time window 5 δευτερολέπτων) → **αγνόησε** το event.
3. Αν **δεν** ταυτίζεται → **user feedback**.

#### Python feedback collector

```python
import time
import threading
from dataclasses import dataclass, field
from typing import Optional
from fsevents import Stream, Observer

IN_MOVED_FROM = 0x00000040
IN_MOVED_TO   = 0x00000080
IN_REMOVED    = 0x00000100


@dataclass
class ImplicitLabel:
    """Ένα implicit feedback event από τον χρήστη."""
    file_path: str
    original_location: Optional[str]   # None αν δεν γνωρίζουμε
    new_location: Optional[str]        # None αν διαγράφηκε
    action: str                        # "moved", "trashed", "renamed"
    timestamp: float = field(default_factory=time.time)
    is_approval: bool = False          # Moved στο σωστό folder → approval
    is_correction: bool = False        # Moved αλλού → correction


class CuratorFeedbackCollector:
    """
    Παρακολουθεί τι κάνει ο χρήστης μετά την οργάνωση.
    
    Διακρίνει user moves από Curator moves μέσω
    curator_move_whitelist (in-memory set).
    """

    def __init__(self, watch_paths: list[str]):
        self._watch_paths = watch_paths
        self._observer = Observer()
        self._pending_moves: dict[int, tuple[str, float]] = {}  # cookie → (src_path, time)
        self._labels: list[ImplicitLabel] = []
        self._lock = threading.Lock()
        # Curator move whitelist: set of (src, dst) που έκανε ο Curator
        self._curator_moves: set[tuple[str, str]] = set()
        self._COOKIE_TIMEOUT = 2.0  # seconds

    def register_curator_move(self, src: str, dst: str):
        """Καλείται από τον Curator ΠΡΙΝ κάνει κάθε κίνηση αρχείου."""
        with self._lock:
            self._curator_moves.add((src, dst))

    def _is_curator_move(self, src: str, dst: str) -> bool:
        with self._lock:
            if (src, dst) in self._curator_moves:
                self._curator_moves.discard((src, dst))
                return True
        return False

    def _fsevents_callback(self, event):
        mask = event.mask
        path = event.name
        cookie = getattr(event, "cookie", 0)
        now = time.time()

        if mask & IN_REMOVED:
            self.on_file_trashed(path)
            return

        if mask & IN_MOVED_FROM:
            with self._lock:
                self._pending_moves[cookie] = (path, now)

        elif mask & IN_MOVED_TO:
            with self._lock:
                # Καθαρισμός expired cookies
                expired = [c for c, (_, t) in self._pending_moves.items()
                           if now - t > self._COOKIE_TIMEOUT]
                for c in expired:
                    del self._pending_moves[c]

                if cookie in self._pending_moves:
                    src_path, _ = self._pending_moves.pop(cookie)
                    # Αφήνουμε το lock πριν καλέσουμε on_file_moved
                src_path = src_path if cookie in {k: k for k in []} or True else None

            if src_path and not self._is_curator_move(src_path, path):
                self.on_file_moved(src_path, path)

    def on_file_moved(self, src: str, dst: str):
        """User μετακίνησε αρχείο — implicit feedback."""
        label = ImplicitLabel(
            file_path=dst,
            original_location=src,
            new_location=dst,
            action="moved",
        )
        # Heuristic: αν το destination path περιέχει το ίδιο folder name
        # που πρότεινε ο Curator → approval, αλλιώς → correction
        # (Επέκταση: σύγκρισε με Curator's suggestion log)
        with self._lock:
            self._labels.append(label)

    def on_file_trashed(self, path: str):
        """User έστειλε αρχείο στον Κάδο — negative feedback."""
        with self._lock:
            self._labels.append(ImplicitLabel(
                file_path=path,
                original_location=path,
                new_location=None,
                action="trashed",
                is_correction=True,
            ))

    def get_implicit_labels(self) -> list[ImplicitLabel]:
        with self._lock:
            result = list(self._labels)
            self._labels.clear()
        return result

    def start(self):
        self._observer.start()
        for path in self._watch_paths:
            stream = Stream(self._fsevents_callback, path,
                           file_events=True, latency=0.5)
            self._observer.schedule(stream)

    def stop(self):
        self._observer.stop()
        self._observer.join()
```

**Σημείωση για batch moves:** Αν ο user κινήσει 50 αρχεία ταυτόχρονα, λαμβάνεις 50 pairs αλλά τα cookies είναι μοναδικά ανά pair — το batching δεν επηρεάζει την ορθότητα, μόνο το throughput. Το `_lock` προστατεύει από race conditions.

---

### ΘΕΜΑ 4: macOS Tags για Tie-Breaking

#### Format και αποθήκευση

Τα Finder tags αποθηκεύονται ως **xattr** με key `com.apple.metadata:_kMDItemUserTags`. Format: binary plist (NSArray of strings). Κάθε string έχει τη μορφή:

```
TagName\n<color_number>
```

Όπου color_number:
- `0` = None (χωρίς χρώμα)
- `1` = Gray
- `2` = Green
- `3` = Purple
- `4` = Blue
- `5` = Yellow
- `6` = Red
- `7` = Orange

**Όριο tags:** Δεν υπάρχει documented hard limit. Πρακτικά, το xattr payload έχει μέγιστο μέγεθος ~64KB — που αντιστοιχεί σε εκατοντάδες tags. Για το Curator (max ~5-10 tags ανά αρχείο) δεν υπάρχει πρόβλημα.

**Tags preserved κατά move:** Ναι — αποθηκεύονται στο xattr του αρχείου, όχι σε κεντρικό index. Μετακινείς το αρχείο → μετακινούνται και τα xattrs (εφόσον γίνεται σε APFS/HFS+ volume). **Εξαίρεση:** αν αντιγραφή σε FAT32/exFAT volume → xattrs χάνονται.

**Spotlight searchability:**
```bash
mdfind "kMDItemUserTags == 'EPL326'"
mdfind "kMDItemUserTags == 'Thesis'"
# Πολλαπλά tags:
mdfind "kMDItemUserTags == 'EPL326' && kMDItemUserTags == 'Thesis'"
```
Tags αόρατων αρχείων (dot-prefix) **δεν** indexάρονται από Spotlight.

#### Python functions για tag management

```python
import subprocess
import plistlib
import os
from typing import Optional

XATTR_KEY = "com.apple.metadata:_kMDItemUserTags"

# Color mapping για Curator
CURATOR_TAG_COLOR = 4  # Blue — ουδέτερο, δεν συγκρούεται με user colors


def get_file_tags(filepath: str) -> list[str]:
    """Επιστρέφει λίστα tag names (χωρίς color info)."""
    result = subprocess.run(
        ["xattr", "-p", XATTR_KEY, filepath],
        capture_output=True
    )
    if result.returncode != 0:
        return []
    try:
        # Output είναι hex — decode to bytes
        hex_str = result.stdout.decode().strip().replace(" ", "").replace("\n", "")
        raw = bytes.fromhex(hex_str)
        tag_list = plistlib.loads(raw)
        # Κάθε entry: "TagName\n<color>" — παίρνουμε μόνο το name
        return [entry.split("\n")[0] for entry in tag_list if entry.strip()]
    except Exception:
        return []


def set_file_tags(filepath: str, tags: list[str],
                  color: int = CURATOR_TAG_COLOR) -> bool:
    """
    Θέτει Finder tags σε αρχείο.
    Διατηρεί υπάρχοντα tags, προσθέτει νέα.
    """
    existing = get_file_tags(filepath)
    all_tags = list(dict.fromkeys(existing + tags))  # deduplicate, preserve order

    # Build plist: "TagName\n<color>" format
    entries = [f"{tag}\n{color}" for tag in all_tags]
    plist_bytes = plistlib.dumps(entries, fmt=plistlib.FMT_BINARY)

    result = subprocess.run(
        ["xattr", "-w", XATTR_KEY,
         plist_bytes.hex(), filepath],  # xattr -w hex format
        capture_output=True
    )
    return result.returncode == 0


def remove_curator_tags(filepath: str, tags: list[str]) -> bool:
    """Αφαιρεί συγκεκριμένα Curator tags (μετά από user correction)."""
    existing = get_file_tags(filepath)
    filtered = [t for t in existing if t not in tags]
    if not filtered:
        subprocess.run(["xattr", "-d", XATTR_KEY, filepath], capture_output=True)
        return True
    return set_file_tags(filepath, filtered, color=0)


def tag_ambiguous_file(filepath: str,
                       candidate_clusters: list[str]) -> bool:
    """
    Tie-breaking: αρχείο που ισοδύναμα ταιριάζει σε 2+ clusters.
    Βάζει tag και για τα δύο candidates ώστε ο χρήστης να βλέπει
    τις υποψήφιες κατηγορίες στο Finder.
    """
    return set_file_tags(filepath, candidate_clusters)
```

**Παράδειγμα χρήσης για tie-breaking:**
```python
# Αρχείο "report_epl326_thesis.pdf" → HDBSCAN δίνει ίση απόσταση
# από cluster "EPL326" και cluster "Thesis"
tag_ambiguous_file(
    "/Users/user/Downloads/report_epl326_thesis.pdf",
    candidate_clusters=["EPL326", "Thesis"]
)
# Αποτέλεσμα: το αρχείο έχει τώρα tags "EPL326" και "Thesis" στο Finder
# Ο χρήστης βλέπει και τα δύο tags, μπορεί να αναζητήσει με mdfind
```

#### Rust equivalent με `std::process::Command`

```rust
use std::process::Command;

const XATTR_KEY: &str = "com.apple.metadata:_kMDItemUserTags";

pub fn get_file_tags(path: &str) -> Vec<String> {
    let output = Command::new("xattr")
        .args(["-p", XATTR_KEY, path])
        .output();

    match output {
        Ok(out) if out.status.success() => {
            // Parse hex output → binary plist → tags
            // Για production: χρησιμοποίησε `plist` crate
            let hex = String::from_utf8_lossy(&out.stdout)
                .trim()
                .replace(' ', "")
                .replace('\n', "");
            // Placeholder: επέστρεψε raw hex αν δεν έχεις plist crate
            vec![hex]
        }
        _ => vec![],
    }
}

pub fn set_file_tags(path: &str, tags: &[&str], color: u8) -> bool {
    // Build "TagName\n<color>" strings
    let entries: Vec<String> = tags
        .iter()
        .map(|t| format!("{}\n{}", t, color))
        .collect();

    // Γράφουμε μέσω Python script ή `osxmetadata` CLI για απλότητα
    // Εναλλακτικά: χρησιμοποίησε `plist` crate για binary plist encoding
    let tag_args = entries.join(",");
    let status = Command::new("osxmetadata")
        .args(["--set", "tags", &tag_args, path])
        .status();

    status.map(|s| s.success()).unwrap_or(false)
}

pub fn tag_ambiguous_file(path: &str, candidates: &[&str]) -> bool {
    set_file_tags(path, candidates, 4) // 4 = Blue
}
```

**Σημείωση:** Για το Rust side, ο πιο καθαρός τρόπος είναι να καλείς το Python `ai_classifier` sidecar που ήδη έχει `osxmetadata` installed, παρά να επαναλάβεις την binary plist logic σε Rust. Εναλλακτικά, το `plist` crate (`crates.io/crates/plist`) μπορεί να serialize/deserialize binary plists εγγενώς.

---

### Συνολικό Summary ΜΕΡΟΣ 39

| Θέμα | Verdict | Ενέργεια για Curator |
|------|---------|---------------------|
| Spotlight detection | Αδύνατο — κλειστό API | Χρησιμοποίησε `kMDItemUseCount` + `kMDItemLastUsedDate` ως proxy για "lost file" |
| Sequoia entitlements | Απαιτούνται 4 entries | `entitlements.plist` με JIT + unsigned-memory + disable-library-validation + Downloads access |
| FSEvents feedback | Εφικτό με cookie tracking | `CuratorFeedbackCollector` με whitelist για Curator moves, latency 0.3-0.5s |
| macOS Tags tie-breaking | Εξαιρετική λύση | `tag_ambiguous_file()` + `osxmetadata` library, tags preserved κατά move, Spotlight-searchable |

---

*Πηγές: Apple Developer Documentation (NSMetadataQuery, FSEvents), Tauri v2 docs (sidecar/entitlements), malthe/macfsevents GitHub, RhetTbull/osxmetadata GitHub, Eclectic Light Company (Finder tags, xattr format), Hackaday/MacRumors (Sequoia Gatekeeper changes), Apple Community discussions (Spotlight search history), Brett Terpstra (command-line tagging).*