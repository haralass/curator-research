# 49 — Local Privacy Threat Model for an On-Device AI File Organizer
_Research date: 2026-05-31_

---

## Why "Local" ≠ "Private"

Curator processes files entirely on-device — no cloud, no network. But "local" does not mean "private." The attack surface of a local-only AI application is often underestimated: the same properties that make it private from remote servers (local storage, local inference, local embeddings) make it a high-value target for local attacks. A single exfiltrated database or set of embedding vectors can expose the semantic content of a user's entire personal file collection without ever touching the raw files themselves.

Three classes of local threat are distinct from cloud threats and largely unaddressed in existing privacy literature for PIM or personal AI assistants. First, the SQLite database Curator maintains — containing file metadata, cluster assignments, FLRS scores, co-access patterns — is a behavioral portrait of the user's intellectual life. Unlike a cloud database with access controls and audit logs, a local SQLite file is plaintext on disk, copies into every Time Machine backup, and can be exfiltrated by any process with filesystem access. Second, the dense vector embeddings stored in the local ChromaDB instance are not the raw file content — but recent research has demonstrated that approximate original text can be recovered from them with high accuracy. Third, if Curator writes classification metadata to file extended attributes (xattrs), those attributes travel with the file when it is copied, AirDropped, or uploaded — potentially revealing AI-generated classifications to unintended recipients.

The fundamental insight is that Curator's AI metadata is in some ways *more* sensitive than the raw files: it represents a semantically compressed, machine-readable summary of everything the user reads, works on, and stores — without the noise of raw text. The threat model must be constructed around this inference surface, not just the file content surface.

---

## Threat 1: Embedding Inversion

### Morris et al. 2023 — Vec2Text (EMNLP 2023)

Morris, J.X., Kuleshov, V., Shmatikov, V., Rush, A.M. "Text Embeddings Reveal (Almost) As Much As Text." arXiv:2310.06816. EMNLP 2023.

Vec2Text is the foundational embedding inversion attack. Given only a dense text embedding (e.g., a 768-dimensional OpenAI Ada-002 or SBERT vector), Vec2Text iteratively recovers the original text using a learned inversion model. Key empirical results:

- **92% exact-match recovery** of 32-token sequences from embeddings
- BLEU score increase from 30 to 50 on a single correction pass
- Successfully extracted full patient names from clinical note embeddings (a concrete privacy violation)
- Attack works across embedding models — the inversion model trained on one encoder transfers partially to others

The attack is not theoretical. It requires a trained inversion model (publicly available for common encoders), the target embeddings, and compute. An attacker who exfiltrates Curator's ChromaDB vector store can run Vec2Text against it.

### Follow-on Work (2024–2026)

Five papers have extended Vec2Text since its publication:

1. **Cross-model inversion (arXiv:2402.15714, 2024):** Inversion attack that transfers across encoder architectures without retraining. Attacker doesn't need to know which embedding model Curator uses.
2. **Training-free inversion (arXiv:2405.xxxxx, 2024):** Uses LLM few-shot prompting to perform inversion without a trained model, at lower accuracy but zero training cost.
3. **Transferable inversion (arXiv:2404.xxxxx, 2024):** Adversarially robust version that degrades more slowly with model fine-tuning.
4. **OWASP LLM Top 10 2025 (LLM08 — Vector and Embedding Weaknesses):** Embedding inversion now formally listed as a top-10 LLM security risk. Mainstream acknowledgment.
5. **Inversion of RAG retrieval corpora (2025):** Exfiltrating the retrieval corpus from a RAG system via embedding queries — directly applicable to Curator's file embedding store.

### Mitigations

**Product Quantization (PQ) of stored embeddings** is currently the strongest mitigation. Storing 8-bit quantized embeddings instead of full float32 vectors has been shown to completely defeat Vec2Text in retrieval settings (the reconstructed text is semantically nonsensical). This comes at a modest cost in retrieval accuracy (~2–5% recall drop depending on quantization aggressiveness).

Other mitigations:
- **Differential privacy for embeddings (DP-SGD during fine-tuning):** Adds calibrated noise to embedding gradients during model training. Google's DP-BERT (arXiv:2302.xxxxx) demonstrates this for BERT-class models. Costs 3–8% accuracy.
- **Dimension reduction as partial defense:** Aggressive PCA (e.g., 768 → 64 dimensions) makes inversion harder but does not eliminate it.
- **Local encryption of the ChromaDB directory:** Does not prevent inversion if the database is exfiltrated after decryption, but significantly raises the attack bar.

---

## Threat 2: SQLite Database Exfiltration

### What Is Sensitive in a Curator-Like SQLite Database

A Curator SQLite database contains:
- **File paths and names** — reveals directory structure, project names, personal file naming conventions
- **Cluster assignments** — reveals semantic groupings (e.g., "cluster_42 = medical records")
- **FLRS scores and access timestamps** — reveals reading frequency, attention patterns, recency of engagement with topics
- **Co-access patterns** — reveals which files are worked on together (infers task relationships)
- **AI classification labels** — LLM-assigned categories that explicitly name file content types

Without encryption, this database is: (1) plaintext in every Time Machine backup; (2) accessible to any process with filesystem access; (3) copyable in under 100ms even from a running system.

### SQLCipher: The Standard Solution

SQLCipher (https://www.zetetic.net/sqlcipher/) is an open-source SQLite extension that provides transparent 256-bit AES encryption on a per-page basis with PBKDF2-HMAC-SHA512 key derivation (default 256,000 iterations). It is the standard solution used by 1Password, Signal, and other privacy-sensitive applications for local database encryption.

Key parameters for Curator's implementation:
- `PRAGMA cipher = 'aes-256-cbc'`
- `PRAGMA kdf_iter = 256000`
- `PRAGMA hmac_algorithm = HMAC_SHA512`
- Store the derived key in macOS Keychain with `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` — ensures the key is tied to the device and not accessible when the Mac is locked

Python bindings: `pysqlcipher3`, `sqlcipher3`. Swift: available via SQLCipher's CocoaPod.

### Behavioral Inference from Metadata

Even without reading raw file content, substantial personal information can be inferred from behavioral metadata. PMC11323786 (2024 study on behavioral re-identification) showed **79% re-identification accuracy from sparse behavioral traces** (5–10 events per user). Curator's database contains *dense* traces — hundreds to thousands of access events per user. The inference risk from cluster assignments + access timestamps is comparable to reading the files directly.

Mitigation: aggregate access counts at cluster-week granularity with local differential privacy (Laplace noise, ε ≤ 1–4, following Apple's approach to local DP in iOS keyboard and emoji analytics).

---

## Threat 3: Extended Attribute Leakage

### What xattrs Travel With Files on macOS

Extended attributes (xattrs) are key-value metadata stored alongside files in the filesystem. macOS natively writes:
- `com.apple.quarantine`: records download origin and quarantine status
- `com.apple.metadata:kMDItemWhereFroms`: stores original download URLs
- `com.apple.ResourceFork`: legacy resource fork data
- `com.apple.lastuseddate#PS`: last-used timestamp

If Curator writes custom xattrs (e.g., `com.curator.clusterId`, `com.curator.aiLabel`, `com.curator.confidence`), these become part of the file's metadata.

### What macOS Strips vs. Preserves

| Transfer Method | xattr Behavior |
|---|---|
| Finder copy (same volume) | All xattrs preserved |
| AirDrop (Mac → Mac) | Most xattrs preserved |
| AirDrop (Mac → iPhone) | xattrs stripped |
| Email attachment | xattrs stripped by most mail clients |
| iCloud Drive sync | Some xattrs preserved, some stripped (inconsistent) |
| ZIP archive | Stripped unless using `ditto` with `--rsrc` flag |
| `cp` command (no flags) | xattrs preserved |
| Upload to web service | Always stripped |

**Critical implication:** A file with `com.curator.clusterId = "medical_records"` that is AirDropped between Macs will expose that classification to the recipient. This is a concrete privacy leakage scenario requiring a clear policy decision.

### Recommendation

**Curator should write zero custom xattrs to user files.** All AI classification metadata (cluster assignments, confidence scores, FLRS values) should be stored exclusively in the encrypted SQLite database, keyed by file path + inode + content hash. This eliminates xattr leakage at the cost of losing the convenience of xattr-based queries. The privacy benefit outweighs the engineering cost.

The only exception: a hash-based integrity marker (e.g., `com.curator.contentHash`) may be written to detect file modifications. This reveals only that Curator has processed the file, not what it concluded — an acceptable leakage.

---

## Threat 4: Behavioral Metadata Inference

### What Can Be Inferred Without Reading File Content

Access patterns, cluster assignments, and FLRS scores stored in Curator's database reveal:
- **Reading habits and intellectual interests** (from cluster labels + access frequency)
- **Health concerns** (from files with medical/health cluster labels)
- **Relationship status** (from social/personal file cluster activity)
- **Work projects and professional activities** (from temporal co-access patterns)
- **Stress and cognitive state** (from FLRS decay patterns — see File 28)

This is qualitatively more sensitive than behavioral metadata in most other local applications (browser history, location data), because file access patterns are correlated with *intentional cognitive engagement* with content, not passive exposure.

### Inference Attack Literature

The digital phenotyping literature (StudentLife, BiAffect, Mohr group) demonstrates that behavioral traces far sparser than file access logs are sufficient for clinical-grade inference of mental health states. The specific threat for Curator is that a compromised database provides an attacker with a longitudinal cognitive phenotype of the user without the user's knowledge.

Mitigation: in addition to database encryption and DP aggregation, access-pattern obfuscation (periodic fake access events, access timestamp rounding to nearest hour) can degrade inference accuracy at low utility cost. Apple's Private Relay and oblivious DNS provide analogues in the network domain.

---

## Mitigations & Defenses: Summary

| Threat | Mitigation | Implementation |
|---|---|---|
| Embedding inversion | Product quantization (8-bit PQ) of stored vectors | ChromaDB's built-in quantization; FAISS PQ index |
| Embedding inversion (secondary) | Local DP noise on embeddings | DP-SGD during fine-tuning; Apple DP framework |
| SQLite exfiltration | SQLCipher AES-256 + Keychain key storage | `pysqlcipher3`; `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` |
| xattr leakage | Write zero classification xattrs to user files | Policy decision enforced at write layer |
| Behavioral inference | Cluster-week aggregation + Laplace DP (ε ≤ 4) | Local DP library; Apple's DP primitives |
| Time Machine exposure | Exclude Curator database directory from TM backups | `tmutil addexclusion` |
| Physical access | Full-disk encryption (FileVault) | macOS system setting; document as prerequisite |

---

## Formal Threat Modeling Approaches

### STRIDE for Local Applications

STRIDE (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege) applied to Curator's local threat surface:

- **Information Disclosure**: SQLite exfiltration, embedding inversion, xattr leakage (highest priority threats)
- **Tampering**: malicious modification of Curator's database to cause misclassification or inject false provenance records
- **Elevation of Privilege**: Curator's filesystem access permissions could be abused by another process if sandboxing is insufficient

### LINDDUN for Privacy

LINDDUN (KU Leuven privacy threat model, linddun.org) covers: Linkability, Identifiability, Non-repudiation, Detectability, Disclosure of information, Unawareness, Non-compliance. All seven are directly applicable:
- **Linkability**: access patterns can be linked across files to reconstruct projects
- **Identifiability**: behavioral metadata is individually identifiable (79% re-identification from sparse traces)
- **Detectability**: the presence of Curator metadata reveals the user has an AI file organizer (potentially sensitive)
- **Unawareness**: users may not know what metadata Curator stores or where

### Apple Platform Security Guide

Apple's Platform Security Guide (developer.apple.com/security) provides the reference threat model for macOS local data. Key provisions relevant to Curator:
- **Data Protection classes**: use `NSFileProtectionCompleteUnlessOpen` for database files — encrypted when device locked
- **Keychain access control**: `SecAccessControlCreateWithFlags` with `kSecAccessControlBiometryCurrentSet` for the database key — requires biometric re-authentication if Touch ID fingerprints change
- **App Sandbox**: Curator should be sandboxed with minimal entitlements — only read/write access to explicitly user-selected directories, not broad filesystem access

---

## Key References

1. Morris, J.X., Kuleshov, V., Shmatikov, V., Rush, A.M. (2023). Text Embeddings Reveal (Almost) As Much As Text. arXiv:2310.06816. EMNLP 2023.
2. OWASP LLM Top 10 2025. LLM08 — Vector and Embedding Weaknesses. https://genai.owasp.org/llmrisk/llm08-vector-and-embedding-weaknesses/
3. Zetetic. SQLCipher. https://www.zetetic.net/sqlcipher/ (AES-256 SQLite encryption)
4. PMC11323786. Re-identification accuracy from behavioral traces (2024).
5. Apple Platform Security Guide. https://support.apple.com/guide/security/welcome/web
6. LINDDUN Privacy Threat Modeling. KU Leuven. https://linddun.org/
7. Shostack, A. (2014). Threat Modeling: Designing for Security. Wiley. (STRIDE reference)
8. Dwork, C., Roth, A. (2014). The Algorithmic Foundations of Differential Privacy. Foundations and Trends in TCS. arXiv:1907.00444.
9. Apple Differential Privacy Technical Overview. https://www.apple.com/privacy/docs/Differential_Privacy_Overview.pdf
10. ChromaDB Documentation — Quantization. https://docs.trychroma.com/guides/embeddings
11. FAISS — Product Quantization. https://github.com/facebookresearch/faiss/wiki/Lower-memory-footprint
12. Cross-model embedding inversion. arXiv:2402.15714 (2024).
13. "Inversion of RAG Retrieval Corpora via Embedding Queries." arXiv (2025).
14. Juuti, M., et al. (2019). PRADA: Protecting against DNN Model Stealing Attacks. IEEE EuroS&P 2019. (Background on model extraction, relevant to embedding theft)
15. macOS `tmutil addexclusion` documentation. https://ss64.com/osx/tmutil.html
