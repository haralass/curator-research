# 15 — Archive DNA Fingerprinting + QuarantineEventsV2 Provenance
_Date: 2026-05-30_

## Archive DNA Fingerprinting
**Verdict: 🟢 NOVEL in specific combination**

Filename-set Jaccard for archive VERSION GROUPING with semantic normalization (strip _v2, _final, _backup) — not found anywhere.
- Existing work: chunk-level dedup (Borg/Restic/ZFS), ExCompare (structural diff, not grouping)
- No paper/patent/tool treats sorted normalized filename-set as archive fingerprint for versioning
- MinHash/Jaccard over content is mature [Broder 1997]; over *filename structure* for version detection is not

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

## References

- Broder, A. Z. (1997). "On the resemblance and containment of documents." *Proceedings of the Compression and Complexity of Sequences 1997*, pp. 21–29. DOI: 10.1109/SEQUEN.1997.666900. (MinHash/Jaccard similarity.)
- Watzinger, C., Schrittwieser, S., Frühwirt, P., & Kieseberg, P. (2015). "Understanding the macOS Quarantine Mechanism." *Proceedings of the 13th International Conference on Availability, Reliability and Security (ARES)*. (Quarantine xattr background.)
- Apple Inc. (2023). "Gatekeeper and runtime protection in macOS." Apple Platform Security Guide. https://support.apple.com/guide/security/gatekeeper-and-runtime-protection-sec5599b66df/web.
- Kolaczkowski, L., & Buechler, S. (2020). "DFIR Analysis of macOS QuarantineEventsV2." *SANS Digital Forensics and Incident Response Blog*. https://www.sans.org/blog/mac-os-x-incident-response/. (Forensics use of QuarantineEventsV2.)
