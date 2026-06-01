# Behavioral Edge Privacy: Threat Model, k-Anonymity, Differential Privacy

**File:** 92 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
Curator's behavioral co-access edges encode which files a user accesses together in a session — a sensitive pattern that reveals work habits, personal relationships, health concerns, and financial activity. The primary threat is backup exposure: Time Machine backups and iCloud backups of the GRDB database expose the full co-access graph to anyone with backup access. The recommended mitigation is: (1) store co-access counts only in the GRDB/SQLCipher database (already encrypted), (2) apply k-anonymity style grouping (don't store individual file-pair counts below threshold k=5), and (3) optionally add Laplace noise (ε-DP) to edge weights before writing to disk.

## Key Findings
- **Threat model:** The behavioral graph is stored in GRDB + SQLCipher. Threats: (a) backup exposure (Time Machine, iCloud backup — attacker with backup access can read plaintext if they have the device key); (b) process injection/memory dump; (c) Curator sidecar process compromise. Curator's SQLCipher encryption addresses (a) for off-device storage but not for on-device memory.
- **k-anonymity for co-access:** A co-access edge between file A and file B is only written to the database if it has been observed at least k=5 times across sessions. Single co-accesses (could be accidental) are discarded. This prevents re-identification of one-off sensitive file pairings.
  - Practical implementation: maintain an in-memory counter; flush to GRDB only when count ≥ k.
- **Differential privacy for edge weights (optional):** Add Laplace noise with scale b = sensitivity/ε to the co-access count before writing. Sensitivity = 1 (one session can contribute at most 1 to any edge count). For ε=1.0: Laplace(0, 1) noise. For ε=0.1: Laplace(0, 10) noise — but this makes low-count edges useless.
  - Recommendation: ε-DP on edge weights is not worth the utility loss for a single-user local app. k-anonymity threshold (k≥5) is sufficient.
- **Backup exposure mitigation:** SQLCipher encryption is the primary defense. Ensure the SQLCipher key is stored in the macOS Keychain (not hardcoded). Time Machine respects file permissions but not encryption — the encrypted GRDB file is backed up as ciphertext, which is safe as long as the Keychain key is not backed up in plaintext.
- **iCloud backup:** Curator's database should NOT be in `~/Library/Application Support/` if it might sync to iCloud. Store in `~/Library/Application Support/com.yourapp.Curator/` with `NSURLIsExcludedFromBackupResourceKey = true` set on the database file.
- **k-anonymity limits:** k-anonymity does not protect against background knowledge attacks or attribute disclosure. For a personal local app, this is an acceptable residual risk given that the attacker model is primarily backup/physical access, not a sophisticated adversary with background knowledge.
- **ε,k combination:** The `(k,ε)-anonymity` framework (arXiv:1710.01615) combines both: k-anonymity prevents linkage, ε-DP limits inference from aggregate queries. For Curator: k=5 threshold only; skip ε-DP to preserve edge weight utility.

## Relevant Papers / Prior Art
- arXiv:1710.01615, "(k,ε)-Anonymity: k-Anonymity with ε-Differential Privacy." — Theoretical framework for combining both.
- Dwork C., "Differential Privacy," *ICALP* 2006. — Foundational DP reference; Laplace mechanism.
- Sweeney L., "k-Anonymity: A Model for Protecting Privacy," *IJUFKS* 10(5):557–570, 2002. DOI: 10.1142/S0218488502001648. — Original k-anonymity.
- Apple Platform Security Guide (2024): SQLCipher key management in macOS Keychain.

## Applicability to Curator

```python
# Python sidecar: behavioral edge accumulator with k-anonymity

from collections import defaultdict
import grdb  # hypothetical Python GRDB binding; use sqlite3 + SQLCipher in practice

K_THRESHOLD = 5  # minimum co-access count before persisting edge

class BehavioralEdgeAccumulator:
    def __init__(self):
        self._pending: dict[tuple[int,int], int] = defaultdict(int)

    def record_coacccess(self, file_id_a: int, file_id_b: int):
        """Record a co-access event (call during session boundary processing)."""
        key = (min(file_id_a, file_id_b), max(file_id_a, file_id_b))
        self._pending[key] += 1

    def flush_to_db(self, db_conn):
        """Write edges that meet k-anonymity threshold to GRDB."""
        to_persist = {k: v for k, v in self._pending.items() if v >= K_THRESHOLD}
        for (fid_a, fid_b), count in to_persist.items():
            db_conn.execute(
                "INSERT OR REPLACE INTO behavioral_edges (file_a, file_b, count) "
                "VALUES (?, ?, ?) ON CONFLICT(file_a, file_b) DO UPDATE SET count = count + ?",
                (fid_a, fid_b, count, count)
            )
        # Remove persisted entries from pending
        for k in to_persist:
            del self._pending[k]
```

```swift
// Swift: Exclude database from iCloud/Time Machine backup
let dbURL = FileManager.default.urls(for: .applicationSupportDirectory, in: .userDomainMask)
    .first!.appendingPathComponent("Curator/curator.db")
var resourceValues = URLResourceValues()
resourceValues.isExcludedFromBackup = true
try? dbURL.setResourceValues(resourceValues)
// SQLCipher key: retrieve from Keychain, never hardcode
```

## Open Questions
- Should co-access edges be time-decayed (older co-accesses count less)? This would require storing timestamps alongside counts, increasing schema complexity but improving freshness.

## Sources
- https://arxiv.org/pdf/1710.01615
- https://jisem-journal.com/index.php/journal/article/download/1198/456
- https://www.cerias.purdue.edu/assets/pdf/bibtex_archive/2010-24-report.pdf
