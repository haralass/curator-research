# Folder Biography / Sibling Completeness
**File:** 68 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
A "folder biography" is a lightweight schema describing what a well-formed version of that folder type should contain — inferred from observed patterns across similar folders. Sibling completeness scoring compares a folder's actual contents against this schema and surfaces a completeness score (0–1) the user can act on. This is directly buildable in Curator using existing file-type and NER signals.

## Key Findings
- File organization literature (genealogy, legal, project management) consistently uses manifest-based approaches: define expected subfolders/file types, then flag deviations
- Folder integrity manifests (commercial tools like FolderManifest) combine CRC/SHA-256 hashing with expected-structure definitions — audit-ready completeness reports
- The underlying pattern: `expected_types = infer_from_siblings(folder_class)`, `score = |actual ∩ expected| / |expected|`
- "Sibling" here means other folders of the same inferred type (e.g., other project folders) — Curator can cluster folders by type and learn type-specific expected file distributions
- No academic paper with exactly this framing was found; it's an applied engineering concept with commercial implementations

## Relevant Papers / Prior Art
- FolderManifest.com "Folder Integrity Manifest Guide" — practitioner reference
- ManageEngine ADAudit Plus file integrity monitoring — enterprise reference
- Genealogy folder organization literature (FamilySearch, Family Tree Magazine) — informal but well-developed conventions for completeness-by-folder-type

## Applicability to Curator
High and immediately actionable. Curator already classifies folders; adding a completeness score per folder class is a small step. Surfacing "this project folder is missing a README and a /exports subfolder" is a high-value, low-surprise feature. Implementation: for each folder class (project, client, archive, media), maintain a learned distribution of expected file type proportions; score each folder instance against its class distribution.

## Open Questions
- How many sibling examples are needed before the expected-type schema is reliable? (Minimum 5–10 siblings of same class?)
- Should completeness scores be shown proactively or only on-demand?
- How to handle intentionally sparse folders (scratch pads, temp dumps) without false positives?

## Sources
- https://www.foldermanifest.com/blog/folder-integrity-manifest-guide
- https://familysearch.org (genealogy organization standards)
- ManageEngine ADAudit Plus documentation
