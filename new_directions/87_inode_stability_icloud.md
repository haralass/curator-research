# iNode Stability Under iCloud Eviction on APFS

**File:** 87 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
iCloud eviction on macOS Sonoma+ (FileProvider architecture) does **not** change a file's inode number during the evict-then-download cycle, as APFS uses extent swapping to restore file data without reallocating the inode. However, xattrs (extended attributes) are systematically stripped during the iCloud sync round-trip except for Finder Tags and xattrs with the `S` (sync) flag. Curator's node identity (inode + device_id → SHA-256 → confirmed by filename+size+mtime) is therefore robust to eviction for the inode component but must not rely on custom xattrs persisting through iCloud Drive sync.

## Key Findings
- **inode stability on eviction:** Confirmed. The inode number does not change when files are downloaded via APFS extent swapping. A file moved within the same APFS volume retains the same inode, xattrs, and saved versions.
- **iCloud Drive architecture (macOS Sonoma+):** Migrated to the FileProvider framework common to all cloud storage services. FileProvider manages a "snapshot" of the remote state; eviction sets a file to dataless (placeholder) status without deallocating the inode.
- **xattr stripping (critical for Curator):** iCloud Drive now preserves almost no xattrs through the evict-download cycle. Only Finder Tags and xattrs explicitly marked with the `S` (sync) flag survive. **Do not store Curator's graph metadata, cluster assignments, or behavioral edge data in xattrs on iCloud Drive files.**
- **com.apple.cloudkit.recordID:** This xattr is an internal CloudKit identifier, not the same as FileProvider item identifiers. It is not reliably readable by third-party apps and is stripped on download.
- **FileProvider API:** Available via `NSFileProviderManager` (Swift). Provides `itemIdentifier` per item — a stable opaque string that persists across eviction/download cycles. This is more stable than inode for iCloud files, but requires entitlements (`com.apple.developer.fileprovider`).
- **APFS clones:** When iCloud downloads a file, it may use APFS cloning (copy-on-write). The clone shares the same inode as long as it's on the same volume; a cross-volume copy gets a new inode.
- **Practical impact on Curator:** The `inode + device_id` identity scheme works correctly for files that stay local. For iCloud Drive files, add a fallback: if inode lookup fails (file is evicted/placeholder), re-identify by `filename + size + mtime` (the existing fallback in Curator's design). Do NOT add a third fallback using xattrs.
- **iCloud Drive metadata loss (2026):** As of May 2026, reports indicate iCloud Drive strips even more metadata than in Sonoma, including custom icons and some Finder-level metadata. This is an ongoing regression, not a stable API.

## Relevant Papers / Prior Art
- Apple Developer Documentation: `NSFileProviderManager`, `NSFileProviderItem`. — Canonical API reference for stable FileProvider item identifiers.
- Howard Oakley, "iCloud Drive in Sonoma: FileProvider and eviction," *The Eclectic Light Company*, November 2023. — Most detailed public analysis of eviction mechanics and inode behavior.
- Howard Oakley, "Does iCloud Drive now lose almost all metadata?", May 2026. — Documents accelerating xattr stripping regression.
- Apple APFS Reference: "Cloning Files and Directories." — Defines extent-swap semantics that preserve inodes during file content replacement.

## Applicability to Curator

```swift
// Swift: Robust file identity for iCloud Drive files
import Foundation

struct FileIdentity: Hashable {
    let inode: UInt64
    let deviceID: UInt64
    let fallbackHash: String  // SHA-256 of filename+size+mtime

    static func from(url: URL) -> FileIdentity? {
        let resourceValues = try? url.resourceValues(
            forKeys: [.fileResourceIdentifierKey, .fileSizeKey, .contentModificationDateKey]
        )
        guard let attrs = try? FileManager.default.attributesOfItem(atPath: url.path),
              let inode = attrs[.systemFileNumber] as? UInt64,
              let device = attrs[.systemNumber] as? UInt64 else { return nil }

        // Fallback hash for evicted files
        let name = url.lastPathComponent
        let size = resourceValues?.fileSize ?? 0
        let mtime = resourceValues?.contentModificationDate?.timeIntervalSince1970 ?? 0
        let fallback = SHA256.hash(data: Data("\(name):\(size):\(mtime)".utf8))
            .map { String(format: "%02x", $0) }.joined()

        return FileIdentity(inode: inode, deviceID: device, fallbackHash: fallback)
    }
}

// When resolving a stored identity after potential eviction:
// 1. Try inode + device_id lookup first.
// 2. If file not found at expected path (evicted), scan directory for matching fallbackHash.
// 3. Never use xattrs as identity anchor on iCloud Drive paths.
```

**Architecture decision:** Store Curator's GRDB graph entries keyed by the `fallbackHash` (SHA-256 of name+size+mtime) as the primary durable key, with inode+device_id as a fast lookup index. This survives iCloud eviction transparently.

## Open Questions
- Does `NSFileProviderManager.itemIdentifier(for:)` remain stable across macOS upgrades and Apple ID changes? Needs empirical testing.
- Are iCloud Shared Drive files (shared with other users) subject to different inode/xattr behavior than private iCloud Drive files?

## Sources
- https://eclecticlight.co/2023/11/21/icloud-drive-in-sonoma-fileprovider-and-eviction/
- https://eclecticlight.co/2023/10/25/macos-sonoma-has-changed-icloud-drive-radically/
- https://eclecticlight.co/2026/05/11/does-icloud-drive-now-lose-almost-all-metadata/
- https://eclecticlight.co/2024/03/07/icloud-drive-in-sonoma-clones-sparse-files-versions-and-more/
- https://eclecticlight.co/2024/03/18/how-icloud-drive-works-in-macos-sonoma/
