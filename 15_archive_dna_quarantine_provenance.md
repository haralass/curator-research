# 15 — Archive DNA Fingerprinting + QuarantineEventsV2 Provenance
_Date: 2026-05-30_

## Archive DNA Fingerprinting
**Verdict: 🟢 NOVEL in specific combination**

Filename-set Jaccard for archive VERSION GROUPING with semantic normalization (strip _v2, _final, _backup) — not found anywhere.
- Existing work: chunk-level dedup (Borg/Restic/ZFS), ExCompare (structural diff, not grouping)
- No paper/patent/tool treats sorted normalized filename-set as archive fingerprint for versioning
- MinHash/Jaccard over content is mature; over *filename structure* for version detection is not

Prior art risk: Low-moderate (deep patent search still needed)

## QuarantineEventsV2 as Provenance Signal
**Verdict: 🟢 NOVEL in application**

Using `LSQuarantineDataURLString` domain as routing boost (ucy.ac.cy → university, drive.google.com → work) — gap is clean.
- All existing use is security/forensics (Velociraptor, DFIR tools, Gatekeeper)
- No file organization system reads this database for user-beneficial classification
- Apple's own Provenance Sandbox uses it for access-control, not organization

Implementation: UUID from `com.apple.quarantine` xattr → join on LSQuarantineEventIdentifier → extract domain.
Requires Full Disk Access entitlement.

Notable: macOS Ventura+ adds `com.apple.provenance` xattr (separate, newer mechanism) — worth tracking.

## 🚪 New Doors
1. **com.apple.provenance xattr (Ventura+)** — newer than QuarantineEventsV2, may be more reliable/persistent. Does it carry the same domain info? Howard Oakley's Eclectic Light blog has deep coverage.
2. **Archive DNA as a dataset contribution** — if we release a benchmark of ~500 annotated archives with ground-truth version groups, that's a standalone dataset paper.
3. **Patent opportunity** — both ideas carry low prior art risk. If this ever becomes commercial, these are patentable in their combined form.
