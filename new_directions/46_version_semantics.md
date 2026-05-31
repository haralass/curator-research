# 46. Version Semantics

**Curator Research — IUI 2027 Track**
**Date:** 2026-05-31
**Status:** Draft

---

## Abstract

Exact duplicate detection — comparing file hashes to find byte-for-byte copies — is a solved problem. The harder and more practically important problem is detecting *semantic version relationships* between files that share content ancestry but are not identical: the draft and the final, the submitted PDF and the source DOCX, the emailed attachment and the locally saved copy. Personal filesystems routinely contain version chains that users manage manually and imperfectly, resulting in confusion about which file is current, wasted storage from accumulated drafts, and lost context about a document's provenance. This paper introduces *semantic version chain detection* for personal files: the problem of inferring which files constitute versions of the same document, labeling their roles within the chain, and presenting this structure to the user. We propose a version taxonomy, enumerate the multi-signal detection algorithm, survey prior art, describe FLRS implications, and outline the research contributions.

---

## 1. Beyond Exact Duplicates: The Semantic Version Problem

Hash-based duplicate detection (SHA-256 comparison) finds files that are byte-for-byte identical. This handles the case where a user has two copies of the same downloaded PDF in different folders. It does not handle the far more common case where a user has semantically related but textually distinct files that represent evolutionary stages of the same document.

Consider a typical student assignment lifecycle in a personal filesystem:

```
essay.docx                  (initial draft, created Monday)
essay_v2.docx               (revised draft, Tuesday)
essay_final.docx            (self-declared final, Wednesday night)
essay_final_FINAL.docx      (last-minute revision, Wednesday 11:58 PM)
essay_submitted.pdf         (PDF generated for submission, Thursday)
essay_feedback.docx         (file with professor feedback embedded, Friday)
```

These six files share a common content ancestry: each was derived from the previous one. No two are SHA-256 identical. A conventional file organizer sees six independent files and either groups them by location (they happen to be in the same folder) or ignores their relationship entirely. The user, meanwhile, must manually maintain awareness of which is the "real" current version, which is safe to delete, and which constitutes the final submission record.

The practical costs of failing to detect version chains are significant. Bergman et al. (2008) found that version proliferation is one of the most common sources of personal file confusion: users cannot determine which of several similarly-named files is current. Whittaker & Hirschberg (2001) document the accumulation of redundant paper files as a general personal archival behavior pattern; digital file accumulation is the direct analogue. Without automated version chain detection, any AI file organizer that attempts to clean up a filesystem will either leave version clutter untouched or, worse, delete files the user intended to keep.

---

## 2. Version Chain Taxonomy

We define a nine-class taxonomy for the roles a file can play within a version chain:

| Role | Description | Typical signals |
|---|---|---|
| **Draft** | In-progress, incomplete version | `_draft`, `_wip` in filename; high edit frequency; not yet shared |
| **Revision** | Named intermediate version | `_v2`, `_v3`, `_rev` in filename; created within hours/days of prior version |
| **Final** | User-declared final version | `_final` in filename; low post-creation edit activity |
| **Submitted / Delivered** | Version sent to another party | `_submitted`, `_sent`; format change to PDF; timestamp matches submission deadline |
| **Received Feedback** | Version annotated by a third party | Contains comments/track-changes not in prior versions; received via email attachment |
| **Archived** | Moved to cold storage or compressed | In a ZIP/tarball; in an `Archive/` or `Old/` folder |
| **Superseded** | Replaced by a later version but kept | Older timestamp, no edit activity, a later version exists in the same chain |
| **Branch / Variant** | Parallel version exploring different direction | `_v2_altending`, `_revised_approach`; high semantic divergence from mainline |
| **Reference Copy** | Copy kept for comparison or compliance | `_original`, `_received`; unmodified after receipt |

A file may hold multiple roles. A `_final.pdf` may be simultaneously Final and Submitted. An archived draft is both Draft and Archived.

---

## 3. Detection Signals

Version chain detection is a multi-signal problem. No single signal is sufficient; robustness requires fusing at least three of the following signal types.

### 3.1 Filename Pattern Analysis

Filename patterns carry high-precision role signals. A regular expression taxonomy of version-indicative patterns:

```
draft    → r'_(draft|wip|working|temp)\.'
revision → r'_v\d+[\._]|_rev\d+\.'
final    → r'_(final|done|complete|finished)\.'
submitted→ r'_(submitted|submission|sent|delivered)\.'
old/backup → r'_(old|backup|bak|copy|orig)\.'
```

Beyond individual file patterns, *co-occurrence* in filename sets is informative: if `essay.docx`, `essay_v2.docx`, and `essay_final.docx` all exist, the shared stem `essay` with progressive suffixes strongly implies a version chain, even before content analysis.

### 3.2 Temporal Proximity

Files in a version chain are typically created or modified within a bounded time window. A draft and its corresponding final are unlikely to be separated by more than two weeks in the absence of a sabbatical. We define a temporal proximity window of 21 days as the default: pairs of files with shared content (by embedding similarity) and temporal distance < 21 days are version chain candidates.

Temporal proximity is particularly strong for submission events: a PDF file created within minutes of a `.docx` file on the same day, sharing the same filename stem, is almost certainly the submission version of that document.

### 3.3 Semantic Similarity via Embeddings

Cosine similarity between sentence-transformer embeddings (Reimers & Gurevych, 2019) captures semantic overlap between file versions. A draft and its final version will typically have cosine similarity > 0.85; independently created documents on the same topic will have similarity in the range 0.5–0.75. The threshold θ_version = 0.82 is proposed as the default for version chain candidacy, subject to tuning against a labeled evaluation set.

Unlike exact-match (SHA-256) or near-duplicate detection (MinHash/LSH), embedding similarity captures semantic relatedness across format changes: the `.docx` source and the `.pdf` export of the same document will have high embedding similarity despite being byte-level unrelated.

### 3.4 Size Progression

Drafts tend to grow toward finals. A monotonically increasing file size across a temporal sequence of similarly-named files is a confirmatory signal for a linear draft→final chain. Branches and variants typically show non-monotonic size patterns. Note that size progression is a weak signal in isolation and should only be used to break ties between otherwise equally plausible chain interpretations.

### 3.5 Format Change as Submission Signal

The transition from an editable format (`.docx`, `.odt`, `.pages`) to a read-only format (`.pdf`) within a version chain is a strong signal for the Submitted role. This is because submission to an academic institution, client, or colleague typically requires a fixed format, and generating a PDF is the standard mechanism for creating that fixed format on macOS. The format-change signal has high precision: false positives (PDF generation for non-submission purposes) can be partially filtered by checking whether the PDF was opened or sent shortly after creation.

### 3.6 Archive DNA (Jaccard Similarity on Filename Sets)

When a ZIP archive is uncompressed, the set of filenames it contains is comparable to the set of files in the current working directory using Jaccard similarity. High Jaccard similarity between the archive's filename set and a directory's current filename set identifies the archive as a checkpoint of that directory — an archived version of the project. This signal is already described in file 15 of the Curator research corpus and is referenced here for completeness.

---

## 4. Prior Art

### 4.1 Software Version Control

Git (Torvalds, 2005) provides the most rigorous model of file versioning in existence: every change is a content-addressed commit, parent-child relationships are explicit, and the entire version graph is stored with perfect fidelity. However, Git operates on files that users explicitly add to a repository, with explicit commit actions. Personal file versioning is implicit: users save files with new names without explicitly recording a parent-child relationship. Git's model is the gold standard but requires user discipline that most personal computer users do not apply outside of software development.

### 4.2 Cloud Document Versioning

Google Docs and Microsoft SharePoint maintain automatic version histories for documents edited in their respective web applications. However, these systems only version documents edited within their applications; locally saved files, downloaded documents, and files in other formats are not versioned. More importantly, version history is application-siloed: Google Docs does not know that a DOCX uploaded to Google Drive is a version of a document that was previously edited in Microsoft Word locally.

### 4.3 Personal File Version Detection

We find no published work specifically addressing *automatic detection of semantic version chains in personal filesystems*. Whittaker & Hirschberg (2001) document version accumulation as a problem but propose no solution. Boardman & Sasse (2004) observe that users keep multiple versions of files "just in case" but do not address automatic detection. Bergman et al. (2008) identify version proliferation as a source of retrieval difficulty but focus on retrieval rather than detection.

Searches for "personal file version detection", "document version chain personal information management", and "file evolution tracking filesystem" return no directly relevant results, confirming that this is an open research problem.

---

## 5. Integration with FLRS

Curator's Forgetting-Like Relevance Score (FLRS) models the temporal decay of file relevance using parameters A(f) (accessibility / recency weight) and a role-dependent forgetting function. Version chain membership directly affects FLRS in two ways:

**5.1 Superseded versions**: Files labeled Superseded within a chain should have their A(f) weight reduced relative to the Final or Submitted version. A superseded draft that has been inactive for 30+ days since the final was created should receive a low FLRS score — it is unlikely to be needed for retrieval, though it should not be deleted.

**5.2 Final / Submitted version priority**: The Final and Submitted versions of a chain should receive elevated FLRS scores, because these are the versions most likely to be needed for retrieval, reference, or compliance purposes. A submitted assignment PDF should remain accessible throughout the grading period and beyond.

**5.3 Chain-level FLRS**: Beyond per-file scoring, the chain as a whole has a relevance score that decays based on the most recent event in the chain (e.g., feedback received). The chain-level score determines whether Curator proactively surfaces the chain for the user's attention — useful when the user receives feedback on a submitted document and needs to locate the original.

---

## 6. Practical Impact: User-Facing Presentation

The user-facing presentation of version chain detection must be designed carefully to avoid overwhelming the user with version archaeology. We propose the following presentation model:

**Primary file**: The Final or Submitted version is the primary file — the one displayed in search results and folder views by default.

**Chain disclosure**: A small indicator on the primary file signals that Curator has detected a version chain. Expanding the indicator reveals the chain timeline:

```
Economics Essay  (version chain, 6 files)
├── essay.docx              Draft            Apr 14
├── essay_v2.docx           Revision         Apr 15
├── essay_final.docx        Final            Apr 16
├── essay_final_FINAL.docx  Final (later)    Apr 17
├── essay_submitted.pdf     Submitted        Apr 17  ← primary
└── essay_feedback.docx     Received Feedback Apr 20
```

**Cleanup suggestion**: Curator can suggest archiving or deleting the intermediate drafts (with user approval) once the chain is stable (no edits in 14+ days, final version confirmed). This is a REVIEW-tier action under the safe automation policy (file 44).

---

## 7. Research Contribution

1. **Version chain taxonomy**: A nine-class taxonomy for file roles within version chains, grounded in observed personal filesystem behavior and PIM literature.
2. **Multi-signal detection algorithm**: A formal fusion algorithm combining filename patterns, temporal proximity, embedding similarity, size progression, format-change signals, and archive DNA, with defined thresholds and confidence scores.
3. **Evaluation dataset**: A ground-truth dataset of manually annotated version chains from user filesystem samples, enabling precision/recall evaluation of the detection algorithm.
4. **FLRS integration**: A formal model of how version chain membership modifies per-file and chain-level FLRS scores.
5. **User study**: A within-subjects study evaluating whether version chain presentation improves retrieval accuracy and reduces time-to-find for users with version-proliferated filesystems.

---

## References

Barreau, D., & Nardi, B. A. (1995). Finding and reminding: File organization from the desktop. *ACM SIGCHI Bulletin*, 27(3), 39–43. https://doi.org/10.1145/221296.221307

Bergman, O., Beyth-Marom, R., & Nachmias, R. (2008). The user-subjective approach to personal information management systems design. *Journal of the American Society for Information Science and Technology*, 59(2), 235–246. https://doi.org/10.1002/asi.20738

Boardman, R., & Sasse, M. A. (2004). "Stuff goes into the computer and doesn't come out": A cross-tool study of personal information management. In *Proceedings of the SIGCHI Conference on Human Factors in Computing Systems* (pp. 583–590). ACM. https://doi.org/10.1145/985692.985765

Charikar, M. S. (2002). Similarity estimation techniques from rounding algorithms. In *Proceedings of the Thirty-Fourth Annual ACM Symposium on Theory of Computing* (pp. 380–388). ACM. https://doi.org/10.1145/509907.509965

Jones, W., & Teevan, J. (Eds.). (2007). *Personal information management*. University of Washington Press.

Lopresti, D., & Wilfong, G. (2003). A document analysis perspective on information retrieval. In *Proceedings of the Seventh International Conference on Document Analysis and Recognition* (pp. 716–720). IEEE. https://doi.org/10.1109/ICDAR.2003.1227741

Manber, U. (1994). Finding similar files in a large file system. In *Proceedings of the USENIX Winter 1994 Technical Conference* (pp. 1–10). USENIX Association. https://www.usenix.org/conference/usenix-winter-1994-technical-conference/finding-similar-files-large-file-system

Reimers, N., & Gurevych, I. (2019). Sentence-BERT: Sentence embeddings using siamese BERT-networks. In *Proceedings of the 2019 Conference on Empirical Methods in Natural Language Processing* (pp. 3982–3992). ACL. https://doi.org/10.18653/v1/D19-1410

Torvalds, L. (2005). *Git: Fast version control system*. https://git-scm.com/

Whittaker, S., & Hirschberg, J. (2001). The character, value, and management of personal paper archives. *ACM Transactions on Computer-Human Interaction*, 8(2), 150–170. https://doi.org/10.1145/376929.376932

Zelkowitz, M. V. (1978). Perspectives on software engineering. *ACM Computing Surveys*, 10(2), 197–216. https://doi.org/10.1145/356725.356731
