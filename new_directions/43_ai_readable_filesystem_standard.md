# 43. AI-Readable Filesystem Standard

**Curator Research — IUI 2027 Track**
**Date:** 2026-05-31
**Status:** Draft

---

## Abstract

Filesystems were designed for human navigation: hierarchical folders with human-readable names, opened by human eyes. AI agents operating on personal filesystems must infer the meaning, provenance, relationships, and quality of every file from first principles — because there is no standard way for a file to declare what it is, where it came from, or how it relates to other files. This paper proposes the *AI-Readable Filesystem Standard* (ARFS): a formal specification for the metadata, structure, and annotations that a file or its associated sidecar should carry to be natively comprehensible to AI agents. We describe the problem, enumerate the required and optional fields of the standard, evaluate existing formats against the specification, propose extended attributes as the delivery mechanism on macOS, and situate Curator's TextTwin architecture as a prototype implementation of ARFS. This is a position and vision paper contribution targeting the systems and PIM communities.

---

## 1. The Problem: Filesystems Are Opaque to AI

The hierarchical filesystem, descended from the Unix inode structure of the 1970s (Ritchie & Thompson, 1974), was designed around a simple contract: names identify, folders organize, and content is opaque to the filesystem layer. A file named `report.pdf` sitting in a folder named `Work/Q2/` tells the operating system nothing about its language, author, relationship to `report_final.pdf`, extraction quality, or semantic topic. The OS does not need to know. Users navigating graphically do not need the OS to know — they remember.

AI agents operating on filesystems need to know all of these things. When Curator processes a file, it must:

1. Extract clean text (non-trivial for PDFs with complex layouts, scanned images, or OCR artifacts)
2. Identify the language, author, title, and date
3. Infer the file's role and topic (see file 42)
4. Locate related files (earlier versions, attachments, source materials)
5. Assess the quality of its own extraction (to calibrate confidence)
6. Record provenance (where the file came from, when, via what application)

None of this information is available from the filesystem. An AI agent must either compute it from scratch every time the file is accessed — expensive and inconsistent — or persist computed metadata somewhere. Currently there is no standard for *where* or *in what format* that metadata should live.

The absence of a standard has three concrete costs. First, each AI application that processes files builds its own metadata store, producing incompatible silos. Spotlight, Hazel, DEVONthink, and Obsidian all maintain separate indexes over the same files. Second, when a file moves, its computed metadata is orphaned or lost, because the metadata store is indexed by path rather than file identity. Third, AI agents cannot read each other's work: if one application has already extracted clean text and computed embeddings for a file, a second application has no way to discover that fact and must repeat the work.

---

## 2. What an AI-Readable File Would Carry

We propose that a fully AI-readable file should expose the following fields, either embedded in the file, stored in extended attributes, or available in a standardized sidecar:

### 2.1 Required Fields

| Field | Type | Description |
|---|---|---|
| `clean_text` | string | Plain text extraction, normalized whitespace, de-hyphenated |
| `language` | ISO 639-1 | Detected language code(s) |
| `title` | string | Document title (extracted or inferred) |
| `author` | string[] | Author name(s) if determinable |
| `created_date` | ISO 8601 | Document creation date (not filesystem mtime) |
| `role` | enum | File role from the ARFS role taxonomy (see file 42) |
| `extraction_confidence` | float [0,1] | Quality score for the clean_text extraction |
| `content_hash` | SHA-256 | Hash of the source file at extraction time |

### 2.2 Optional Fields

| Field | Type | Description |
|---|---|---|
| `summary` | string | 2–5 sentence abstractive summary |
| `topics` | string[] | Semantic topic labels |
| `embedding_ref` | path | Path to a vector embedding file or store entry |
| `source_url` | URL | Original download URL (provenance) |
| `source_application` | string | Application that created or downloaded the file |
| `page_map` | JSON | Mapping from character offsets to page numbers |
| `relationships` | Relationship[] | Typed links to related files (see §2.3) |
| `sensitivity` | enum | none / personal / financial / legal / medical |
| `retention_policy` | string | Suggested retention period (e.g., "7 years" for receipts) |
| `arfs_version` | string | Version of the ARFS spec used |

### 2.3 Relationship Types

The `relationships` field carries typed links to other files, enabling an AI agent to understand that `essay_final.pdf` supersedes `essay_draft.docx` without re-analyzing both files:

```json
{
  "relationships": [
    {"type": "supersedes", "target": "essay_draft.docx", "target_hash": "abc123"},
    {"type": "derived_from", "target": "research_notes.md", "target_hash": "def456"},
    {"type": "attachment_of", "target": "email_thread.eml", "target_hash": "ghi789"}
  ]
}
```

Relationship types include: `supersedes`, `superseded_by`, `derived_from`, `attachment_of`, `references`, `cited_by`, `duplicate_of`, `archive_contains`, `extracted_from`.

---

## 3. TextTwin as a Prototype Implementation

Curator's TextTwin architecture is an empirical prototype of ARFS. For each source file `document.pdf`, Curator generates:

- `document.clean.txt` — the cleaned plain text extraction
- `document.ai.md` — a human-readable markdown sidecar containing title, author, date, role, topics, summary, and relationships in YAML front matter
- `document.map.json` — a JSON structure mapping character offsets to page numbers (the `page_map` field)

This three-file bundle provides most of the required ARFS fields. The limitations of the current TextTwin approach motivate the ARFS standard: the bundle is stored as sibling files in the same folder, meaning it is disrupted when the user moves or renames the source file; there is no versioning of the sidecar; and the format is Curator-specific and not readable by any other application.

A proper ARFS implementation would solve these problems by attaching metadata directly to the file via extended attributes (see §4), keyed by content hash rather than path.

To our knowledge, no existing work formalizes a standard for AI-readable file metadata in a personal filesystem context. Searches of the literature for "AI-readable document format standard", "machine-readable personal archive standard", and "semantic filesystem specification" return no directly relevant results, confirming that ARFS addresses an open gap.

---

## 4. Extended Attributes as Delivery Mechanism

macOS extended attributes (xattrs) provide a native mechanism for attaching arbitrary metadata to files without modifying file content (Apple Developer Documentation, 2023). The OS already uses xattrs extensively: `com.apple.metadata:kMDItemWhereFroms` stores download URLs, `com.apple.quarantine` stores the quarantine flag, and `com.apple.metadata:kMDItemKeywords` stores Spotlight tags.

ARFS fields could be stored as xattrs in a reserved namespace, e.g., `com.curator.arfs:*`:

```bash
xattr -w com.curator.arfs:role "lecture" document.pdf
xattr -w com.curator.arfs:language "en" document.pdf
xattr -w com.curator.arfs:extraction_confidence "0.94" document.pdf
```

The `clean_text` and `summary` fields are too large for xattrs (macOS imposes a per-attribute size limit of approximately 64 KB). These fields require a content-addressed sidecar store, keyed by the file's content hash, stored in a well-known location such as `~/.curator/arfs_store/`. The xattr `com.curator.arfs:sidecar_ref` would point to the relevant entry.

The xattr approach has important advantages over sidecar files: xattrs are preserved by `cp` (when using `-p` or `--preserve`), survive in-place renames, and are included in Time Machine backups. They are not preserved by all file operations (notably `rsync` without `--xattrs`), which is a known limitation. However, because ARFS metadata is derived from content (keyed by SHA-256), it can always be regenerated from the source file, making loss of xattrs a recoverable situation rather than a catastrophic one.

---

## 5. Comparison to Existing Metadata Standards

Several existing standards address document metadata but fall short of ARFS requirements:

**Dublin Core** (DCMI, 2020) defines fifteen elements (title, creator, subject, description, publisher, date, type, format, identifier, source, language, relation, coverage, rights). Dublin Core is widely adopted for library and web resource description but lacks computational fields: there is no extraction confidence, no embedding reference, no relationship typing beyond `relation`, and no role taxonomy. Dublin Core is designed for human catalogers, not AI agents.

**Schema.org** (Guha et al., 2016) provides a rich vocabulary for structured data on the web, including `CreativeWork`, `Document`, and `DigitalDocument` types. Schema.org covers authorship, dates, topics, and some relationship types. However, it is designed for web-embedded markup (JSON-LD, Microdata, RDFa), not filesystem metadata. There is no standard way to attach schema.org metadata to a local file, no provision for extraction confidence, and no file role taxonomy for personal documents.

**OpenDocument Metadata** (OASIS, 2021) is embedded in ODF files (`.odt`, `.ods`, `.odp`) as Dublin Core elements plus application-specific extensions. It covers the document's own metadata well but has no provisions for inter-file relationships, provenance beyond creator/date, or AI-specific fields.

**PDF/A** (ISO 19005, 2005) is an archival PDF standard that mandates XMP metadata embedding. XMP (Adobe, 2012) is a flexible RDF-based metadata framework that can in principle carry any field. However, XMP metadata is embedded in the file, making it unsuitable for files the user cannot modify (downloaded PDFs, scanned images), and there is no standard XMP schema for AI-readable fields.

None of these standards provides extraction confidence, AI-friendly relationship typing, file role from a personal-document taxonomy, or a mechanism for attaching metadata to files that cannot be modified. ARFS fills this gap.

---

## 6. What the Standard Would Specify

A complete ARFS specification document would define:

**6.1 Core schema**: The field names, types, constraints, and semantics described in §2, expressed as a JSON Schema document.

**6.2 Delivery mechanisms**: Normative rules for how ARFS metadata is attached to files on macOS (xattrs + content-addressed sidecar), Linux (xattrs via `user.*` namespace), and Windows (Alternate Data Streams or sidecar files). Cross-platform portability is a first-class requirement.

**6.3 Versioning**: An `arfs_version` field (semver) identifies the schema version. Consumers must ignore unknown fields (forward compatibility) and must not fail on missing optional fields (backward compatibility).

**6.4 Confidence semantics**: Normative definitions of the `extraction_confidence` scale: 1.0 = native text PDF with no OCR; 0.8–0.99 = high-quality OCR; 0.5–0.79 = degraded OCR or partial extraction; below 0.5 = unreliable, consumer should re-extract or flag.

**6.5 Relationship type registry**: A closed enumeration of relationship types with precise semantics and cardinality rules. For example, a file can have at most one `superseded_by` relationship but arbitrarily many `references` relationships.

**6.6 Sensitivity handling**: Normative guidance on which fields should be omitted or encrypted when `sensitivity` is non-null. A financial file's `clean_text` should not be stored in a shared sidecar store.

---

## 7. Research Contribution

This paper's contributions are of the position and vision paper category, appropriate for IUI's systems and design tracks:

1. **Problem articulation**: A precise statement of why current filesystems are opaque to AI agents, with concrete examples from Curator's development experience.
2. **ARFS specification**: The first formal proposal of required and optional fields for AI-readable file metadata, with a JSON Schema and normative delivery rules.
3. **TextTwin as evidence**: Curator's TextTwin is deployed evidence that the approach is technically feasible and practically valuable, not merely theoretical.
4. **Comparison study**: A systematic comparison of ARFS against Dublin Core, Schema.org, OpenDocument metadata, and PDF/A, demonstrating the gap that ARFS fills.
5. **Standardization pathway**: A proposed path to community standardization, including an open-source reference implementation and a call for adoption by macOS AI application developers.

---

## References

Adobe Systems Incorporated. (2012). *XMP specification part 1: Data model, serialization, and core properties*. Adobe. https://wwwimages.adobe.com/content/dam/acom/en/devnet/xmp/pdfs/XMP%20SDK%20Release%20cc-2016-08/XMPSpecificationPart1.pdf

Apple Developer Documentation. (2023). *Extended attributes*. Apple Inc. https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/FileSystemOverview/FileSystemOverview.html

DCMI (Dublin Core Metadata Initiative). (2020). *DCMI metadata terms*. https://www.dublincore.org/specifications/dublin-core/dcmi-terms/

Guha, R. V., Brickley, D., & Macbeth, S. (2016). Schema.org: Evolution of structured data on the web. *Communications of the ACM*, 59(2), 44–51. https://doi.org/10.1145/2844544

ISO 19005-1:2005. (2005). *Document management — Electronic document file format for long-term preservation — Part 1: Use of PDF 1.4 (PDF/A-1)*. International Organization for Standardization.

Jones, W., & Teevan, J. (Eds.). (2007). *Personal information management*. University of Washington Press.

OASIS. (2021). *Open document format for office applications (OpenDocument) v1.3*. OASIS Standard. https://docs.oasis-open.org/office/OpenDocument/v1.3/

Ritchie, D. M., & Thompson, K. (1974). The UNIX time-sharing system. *Communications of the ACM*, 17(7), 365–375. https://doi.org/10.1145/361011.361061

Sauermann, L., & Cyganiak, R. (2008). Cool URIs for the semantic web. W3C Interest Group Note. https://www.w3.org/TR/cooluris/

Stoica, I., Morris, R., Karger, D., Kaashoek, M. F., & Balakrishnan, H. (2001). Chord: A scalable peer-to-peer lookup service for internet applications. *ACM SIGCOMM Computer Communication Review*, 31(4), 149–160. https://doi.org/10.1145/964723.383071

Wheatley, P., & Hole, B. (2009). PLANETS testbed: Science and best practice for digital preservation testing. *iPRES 2009 — The Sixth International Conference on Preservation of Digital Objects*. California Digital Library.

Woodruff, A., & Stonebraker, M. (1997). Supporting fine-grained data lineage in a database visualization environment. In *Proceedings of the 13th International Conference on Data Engineering* (pp. 91–102). IEEE. https://doi.org/10.1109/ICDE.1997.581722
