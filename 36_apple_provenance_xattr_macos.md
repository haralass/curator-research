# com.apple.provenance and Download Provenance Signals on macOS
## Technical Deep Dive for Curator's Routing Pipeline

**Date:** 2026-05-31  
**Context:** Curator (macOS AI file organizer) — evaluating download provenance as a routing signal

---

## 1. Overview: The Provenance Attribute Family

macOS exposes download provenance through three overlapping mechanisms, each introduced at a different era and serving a different purpose:

| Mechanism | Type | Introduced | Purpose |
|---|---|---|---|
| `com.apple.quarantine` | UTF-8 string xattr | Mac OS X 10.5 Leopard | Gatekeeper quarantine flag |
| `com.apple.metadata:kMDItemWhereFroms` | Binary plist xattr | Early macOS | Spotlight origin metadata |
| `com.apple.provenance` | Binary (11-byte) xattr | macOS 13 Ventura | App execution provenance tracking |

For routing files by download source, the most directly useful signals are `com.apple.quarantine` (which carries a UUID linkable to QuarantineEventsV2.db) and `kMDItemWhereFroms` (which directly embeds the download URL). `com.apple.provenance` tracks *which app last wrote a file*, not *where the file was downloaded from* — an important distinction covered below.

---

## 2. com.apple.provenance: What It Actually Is

### 2.1 Introduction and Scope

`com.apple.provenance` was introduced in **macOS 13 Ventura** (released October 2022). It was initially attached to every file inside downloaded app bundles, but by the time Ventura shipped its first point releases, Apple had narrowed the scope: the xattr is now applied primarily to the `.app` bundle directory itself, not every contained file. Some third-party apps (e.g., the Zed editor) have been observed applying it to files they create or modify, which has caused confusion and occasional tool breakage.

### 2.2 Binary Format

The xattr value is **11 bytes** of binary data, not a plist:

```
[3 bytes: high-order flags/header] [8 bytes: big-endian integer primary key]
```

The 8-byte integer is the **primary key** (`rowid`) of the corresponding row in the `provenance_tracking` table inside `/var/db/SystemPolicyConfiguration/ExecPolicy`. This database, maintained by the System Policy daemon (`syspolicyd`), stores:

- `cdhash` — the code-directory hash of the app that cleared Gatekeeper
- Team ID and signing identity
- Timestamp of first execution clearance
- The provenance ID itself

To read it from the command line:

```bash
xattr -px com.apple.provenance /Applications/SomeApp.app
# → outputs hex: 00 00 01 00 00 00 00 00 00 00 2A  (example)
```

### 2.3 Semantics: App-to-File Attribution

When an app that has been assigned a provenance ID opens a file for writing or creates a new file, **macOS propagates the app's provenance xattr to that file**. This means:

- A file with `com.apple.provenance` does **not** necessarily indicate it was downloaded from the internet.
- It indicates the **last non-Apple, non-App-Store app** that wrote to it.
- Apple's own apps and App Store apps do **not** receive provenance IDs, so their file writes do not propagate the xattr.

### 2.4 What This Means for Curator

`com.apple.provenance` is useful for answering: *"Which third-party tool created or last modified this file?"* — e.g., detecting that a file was produced by a specific version of a download manager or a browser. However, it does **not** carry the download URL directly. For URL-based routing, the correct signals are `kMDItemWhereFroms` and `QuarantineEventsV2.db`.

The ExecPolicy database at `/var/db/SystemPolicyConfiguration/ExecPolicy` requires **root** to read directly, making it unsuitable for a sandboxed app. A non-sandboxed helper or command-line tool could query it.

---

## 3. com.apple.quarantine: The Older Workhorse

### 3.1 Format

`com.apple.quarantine` is stored as a **UTF-8 string** in a well-defined semicolon-delimited format:

```
<flags>;<hex-timestamp>;<agent-name>;<UUID>
```

Example:
```
0083;6138fdb6;Firefox;78B4F60F-4838-431E-8A72-6C666B15E5A6
```

Fields:
- **flags** — 4 hex digits. Common values:
  - `0081` — newly quarantined, not yet user-approved
  - `0083` — quarantined, user has been warned
  - `00c1` — quarantine cleared (Gatekeeper approved)
- **hex-timestamp** — Unix timestamp in hex (seconds since epoch)
- **agent-name** — the display name of the app that created/downloaded the file (e.g., `Safari`, `Firefox`, `curl`)
- **UUID** — links to a row in `QuarantineEventsV2.db`

### 3.2 Relationship Between Fields

The **UUID** is the critical link. `com.apple.quarantine` is lightweight metadata on the file itself; the full provenance — including the download URL — lives in the SQLite database. The UUID is the `LSQuarantineEventIdentifier` primary key in that database.

### 3.3 LSQuarantineDataURL vs LSQuarantineOriginURL vs LSQuarantineAgentName

These are column names in `QuarantineEventsV2.db` (not fields in the xattr):

| Column | Meaning |
|---|---|
| `LSQuarantineDataURLString` | The **direct download URL** — the actual HTTP(S) URL the bytes were fetched from |
| `LSQuarantineOriginURLString` | The **referring page URL** — the web page the user was on when they clicked download |
| `LSQuarantineAgentName` | Human-readable name of the downloading agent (e.g., `"Safari"`, `"Firefox"`) |
| `LSQuarantineAgentBundleIdentifier` | Bundle ID of the agent (e.g., `com.apple.Safari`) |

For routing by source website, `LSQuarantineOriginURLString` is often more reliable than `LSQuarantineDataURLString` — a file downloaded from GitHub may have a `DataURL` pointing to `objects.githubusercontent.com` while its `OriginURL` correctly shows `github.com/user/repo/releases`.

---

## 4. QuarantineEventsV2.db: Schema and Querying

### 4.1 Location

```
~/Library/Preferences/com.apple.LaunchServices.QuarantineEventsV2
```

This is a per-user SQLite 3 database. Each user has their own copy.

### 4.2 Full Table Schema

The primary table is `LSQuarantineEvent`:

```sql
CREATE TABLE LSQuarantineEvent (
    LSQuarantineEventIdentifier     TEXT PRIMARY KEY,   -- UUID matching xattr
    LSQuarantineTimeStamp           REAL,               -- Cocoa timestamp (seconds since 2001-01-01)
    LSQuarantineAgentBundleIdentifier TEXT,             -- e.g. com.apple.Safari
    LSQuarantineAgentName           TEXT,               -- e.g. "Safari"
    LSQuarantineDataURLString       TEXT,               -- direct download URL
    LSQuarantineSenderName          TEXT,               -- email sender (if from Mail)
    LSQuarantineSenderAddress       TEXT,               -- email address
    LSQuarantineOriginTitle         TEXT,               -- page title of origin
    LSQuarantineOriginURLString     TEXT,               -- referring page URL
    LSQuarantineOriginAlias         BLOB                -- alias record (legacy)
);
```

**Timestamp conversion**: `LSQuarantineTimeStamp` is a **Core Data / Cocoa timestamp** — seconds since 2001-01-01 00:00:00 UTC. To convert to Unix epoch, add `978307200`.

```sql
SELECT datetime(LSQuarantineTimeStamp + 978307200, 'unixepoch') AS download_time
FROM LSQuarantineEvent LIMIT 5;
```

### 4.3 Querying for a Specific File

Given a file path, the workflow is:

1. Read `com.apple.quarantine` xattr from the file
2. Extract the UUID (4th semicolon-delimited field)
3. Query the database with that UUID

```sql
SELECT
    LSQuarantineDataURLString,
    LSQuarantineOriginURLString,
    LSQuarantineAgentName,
    datetime(LSQuarantineTimeStamp + 978307200, 'unixepoch') AS download_time
FROM LSQuarantineEvent
WHERE LSQuarantineEventIdentifier = '<UUID-FROM-XATTR>';
```

### 4.4 Listing Recent Downloads (No UUID Needed)

```sql
SELECT
    LSQuarantineDataURLString,
    LSQuarantineOriginURLString,
    LSQuarantineAgentName,
    datetime(LSQuarantineTimeStamp + 978307200, 'unixepoch') AS download_time
FROM LSQuarantineEvent
ORDER BY LSQuarantineTimeStamp DESC
LIMIT 20;
```

---

## 5. com.apple.metadata:kMDItemWhereFroms

This xattr stores a **binary plist** containing an array of URLs from which the file was obtained. It is set by the downloading application and is the most direct URL signal — no database lookup required.

### 5.1 Structure

The binary plist decodes to an `NSArray` of strings, typically:
- Index 0: The direct download URL (same as `LSQuarantineDataURLString`)
- Index 1: The referring page URL (same as `LSQuarantineOriginURLString`)

### 5.2 Command-line Extraction

```bash
# Pretty-print the binary plist:
xattr -px com.apple.metadata:kMDItemWhereFroms ~/Downloads/somefile.dmg \
  | xxd -r -p \
  | plutil -p -

# Example output:
# [
#   0 => "https://objects.githubusercontent.com/github-production-release-asset-..."
#   1 => "https://github.com/neovim/neovim/releases/tag/v0.10.0"
# ]
```

### 5.3 Spotlight Alternative

Spotlight's `mdls` command also exposes this:

```bash
mdls -name kMDItemWhereFroms ~/Downloads/somefile.dmg
```

---

## 6. Reading Provenance in Swift and Rust

### 6.1 Swift — Using getxattr

Swift can call the POSIX `getxattr` C function directly, or use a URL extension wrapper. The underlying C signature:

```c
ssize_t getxattr(const char *path, const char *name,
                 void *value, size_t size,
                 u_int32_t position, int options);
```

**Swift wrapper pattern:**

```swift
import Foundation

extension URL {
    func extendedAttribute(forName name: String) throws -> Data {
        let data = try self.withUnsafeFileSystemRepresentation { fileSystemPath throws -> Data in
            // First call: get size
            let length = getxattr(fileSystemPath, name, nil, 0, 0, 0)
            guard length >= 0 else { throw POSIXError(POSIXErrorCode(rawValue: errno) ?? .EINVAL) }
            var data = Data(count: length)
            let result = data.withUnsafeMutableBytes {
                getxattr(fileSystemPath, name, $0.baseAddress, length, 0, 0)
            }
            guard result >= 0 else { throw POSIXError(POSIXErrorCode(rawValue: errno) ?? .EINVAL) }
            return data
        }
        return data
    }
}
```

**Reading com.apple.quarantine (UTF-8 string):**

```swift
let fileURL = URL(fileURLWithPath: "/path/to/downloaded/file.dmg")
if let data = try? fileURL.extendedAttribute(forName: "com.apple.quarantine"),
   let quarantineString = String(data: data, encoding: .utf8) {
    let parts = quarantineString.split(separator: ";")
    // parts[0] = flags, parts[1] = hex timestamp, parts[2] = agent, parts[3] = UUID
    let uuid = parts.count >= 4 ? String(parts[3]) : nil
}
```

**Reading kMDItemWhereFroms (binary plist):**

```swift
let attrName = "com.apple.metadata:kMDItemWhereFroms"
if let data = try? fileURL.extendedAttribute(forName: attrName),
   let plist = try? PropertyListSerialization.propertyList(from: data, format: nil),
   let urls = plist as? [String] {
    let downloadURL = urls.first
    let referringURL = urls.count > 1 ? urls[1] : nil
}
```

**Reading com.apple.provenance (binary, 11 bytes):**

```swift
if let data = try? fileURL.extendedAttribute(forName: "com.apple.provenance") {
    // data is 11 bytes: 3 header bytes + 8-byte big-endian integer
    let provenanceID = data.suffix(8).withUnsafeBytes { ptr in
        ptr.load(as: UInt64.self).bigEndian
    }
    // provenanceID is the rowid in ExecPolicy's provenance_tracking table
}
```

### 6.2 Rust — Using the xattr Crate

The [`xattr`](https://crates.io/crates/xattr) crate (maintained by Steven Allen) provides a cross-platform POSIX xattr interface. On macOS it wraps `getxattr(2)` directly.

**Cargo.toml:**
```toml
[dependencies]
xattr = "1"
plist = "1"        # for decoding kMDItemWhereFroms binary plist
```

**Reading com.apple.quarantine:**

```rust
use std::path::Path;
use std::str;

fn read_quarantine_uuid(path: &Path) -> Option<String> {
    let data = xattr::get(path, "com.apple.quarantine").ok()??;
    let s = str::from_utf8(&data).ok()?;
    let parts: Vec<&str> = s.split(';').collect();
    if parts.len() >= 4 {
        Some(parts[3].to_string())
    } else {
        None
    }
}
```

**Reading kMDItemWhereFroms:**

```rust
use plist::Value;
use std::io::Cursor;

fn read_where_froms(path: &Path) -> Option<Vec<String>> {
    let data = xattr::get(path, "com.apple.metadata:kMDItemWhereFroms").ok()??;
    let cursor = Cursor::new(data);
    let plist = Value::from_reader(cursor).ok()?;
    if let Value::Array(arr) = plist {
        Some(arr.iter().filter_map(|v| v.as_string().map(String::from)).collect())
    } else {
        None
    }
}
```

**Querying QuarantineEventsV2.db in Rust:**

```rust
use rusqlite::{Connection, Result};

fn lookup_download_url(uuid: &str) -> Result<(Option<String>, Option<String>)> {
    let db_path = format!(
        "{}/Library/Preferences/com.apple.LaunchServices.QuarantineEventsV2",
        std::env::var("HOME").unwrap()
    );
    let conn = Connection::open(&db_path)?;
    let mut stmt = conn.prepare(
        "SELECT LSQuarantineDataURLString, LSQuarantineOriginURLString
         FROM LSQuarantineEvent
         WHERE LSQuarantineEventIdentifier = ?1"
    )?;
    let row = stmt.query_row([uuid], |row| {
        Ok((row.get::<_, Option<String>>(0)?, row.get::<_, Option<String>>(1)?))
    }).ok();
    Ok(row.unwrap_or((None, None)))
}
```

---

## 7. Sandbox Restrictions and Entitlements

### 7.1 Reading xattrs in a Sandboxed App

Reading **xattrs** on files the user has granted access to (via Open panel, drag-and-drop, or file-provider entitlements) does **not** require special entitlements beyond normal file read access. The `getxattr(2)` call inherits the same access control as `open(2)`.

### 7.2 Reading QuarantineEventsV2.db

This is the more complex case:

- The database resides in `~/Library/Preferences/`, which is **outside the sandbox container**.
- A sandboxed app **cannot** access `~/Library/Preferences/` directly unless it has the `com.apple.security.temporary-exception.files.home-relative-path.read-only` entitlement for that path, or uses a user-initiated file open.
- Apple's App Store review team scrutinizes such exceptions. Reading another app's quarantine history could be considered a privacy concern (see Section 8).
- **Practical approach for Curator**: Read `com.apple.metadata:kMDItemWhereFroms` xattr directly on the file — this requires no special entitlements and gives the download URL without a DB lookup. Fall back to the quarantine UUID + DB query only when the xattr is absent (e.g., for files downloaded by curl or non-quarantine-aware tools).

### 7.3 ExecPolicy Database

`/var/db/SystemPolicyConfiguration/ExecPolicy` is **root-owned** and **SIP-protected**. Reading the `provenance_tracking` table requires running as root or via an XPC service with elevated privileges. This is not feasible in an App Store distributed app.

### 7.4 Entitlement Checklist for Curator's Provenance Feature

| Operation | Entitlement Required | App Store Compatible |
|---|---|---|
| Read `kMDItemWhereFroms` xattr on user-selected files | None (file read access) | Yes |
| Read `com.apple.quarantine` xattr | None | Yes |
| Query `QuarantineEventsV2.db` | `files.home-relative-path.read-only` exception or user grant | Possibly (exception review required) |
| Query `ExecPolicy` provenance_tracking | Root / SIP bypass | No |

---

## 8. Privacy Considerations

### 8.1 Information Sensitivity

`QuarantineEventsV2.db` contains a history of every file downloaded from the internet by any quarantine-aware app on the system, including URLs, agent names, and timestamps. This is sensitive because:

- It reveals browsing/download habits
- It can expose internal tool usage (e.g., which GitHub repos a developer downloads releases from)
- The database is not encrypted or access-controlled beyond standard Unix DAC

### 8.2 App Store and Privacy Manifest

As of macOS 14 Sonoma and updated App Store guidelines, apps that access certain categories of "required reason API" must declare a `PrivacyInfo.xcprivacy` manifest explaining why. SQLite databases in `~/Library/Preferences/` that belong to system services fall in a grey area. Apple has not explicitly listed `QuarantineEventsV2` as a required-reason API, but accessing another subsystem's preference database could trigger App Review concerns.

### 8.3 Recommended Approach for Curator

1. **Prefer `kMDItemWhereFroms`** — it is set by the downloading app on the file itself, read-only access is user-transparent, and it requires no database access.
2. **Treat QuarantineEventsV2 as an optional enrichment** — only query it in a non-sandboxed (direct distribution) build, or via a user-approved file picker pointing at the database.
3. **Do not store raw URLs in any cloud-synced or analytics pipeline** without anonymization.

---

## 9. Practical Extraction: Full Pseudocode

Given a file path, here is the complete provenance extraction algorithm for Curator:

```
function extract_download_provenance(file_path):

    # --- Step 1: Try kMDItemWhereFroms (fastest, no DB needed) ---
    xattr_data = getxattr(file_path, "com.apple.metadata:kMDItemWhereFroms")
    if xattr_data is not None:
        urls = decode_binary_plist(xattr_data)  # → [String]
        return ProvenanceResult(
            download_url  = urls[0] if len(urls) > 0 else None,
            referring_url = urls[1] if len(urls) > 1 else None,
            source        = "kMDItemWhereFroms"
        )

    # --- Step 2: Try com.apple.quarantine → QuarantineEventsV2 ---
    quarantine_str = getxattr(file_path, "com.apple.quarantine")
    if quarantine_str is not None:
        parts = quarantine_str.split(";")
        if len(parts) >= 4:
            uuid = parts[3]
            agent_name = parts[2]
            row = sqlite_query(
                db    = "~/Library/Preferences/com.apple.LaunchServices.QuarantineEventsV2",
                sql   = """
                    SELECT LSQuarantineDataURLString,
                           LSQuarantineOriginURLString,
                           LSQuarantineAgentName
                    FROM LSQuarantineEvent
                    WHERE LSQuarantineEventIdentifier = ?
                """,
                params = [uuid]
            )
            if row:
                return ProvenanceResult(
                    download_url  = row.LSQuarantineDataURLString,
                    referring_url = row.LSQuarantineOriginURLString,
                    agent         = row.LSQuarantineAgentName or agent_name,
                    source        = "QuarantineEventsV2"
                )
            else:
                # UUID not in DB (entry may have been cleaned) — use xattr fields only
                return ProvenanceResult(agent=agent_name, source="quarantine-xattr-only")

    # --- Step 3: Check com.apple.provenance (app attribution, not download URL) ---
    provenance_bytes = getxattr(file_path, "com.apple.provenance")
    if provenance_bytes is not None and len(provenance_bytes) == 11:
        provenance_id = big_endian_uint64(provenance_bytes[3:11])
        # provenance_id → look up in ExecPolicy (requires root; skip in sandbox)
        return ProvenanceResult(provenance_app_id=provenance_id, source="com.apple.provenance")

    # --- Step 4: No provenance signal found ---
    return ProvenanceResult(source="none")


function extract_host(url):
    if url is None: return None
    parsed = urlparse(url)
    # Strip www. prefix and return eTLD+1
    return parsed.hostname.removeprefix("www.")
```

**Routing logic in Curator:**

```
provenance = extract_download_provenance(file_path)
host = extract_host(provenance.referring_url or provenance.download_url)

routing_table = {
    "github.com":                  "~/Developer/Downloads",
    "pubmed.ncbi.nlm.nih.gov":    "~/Research/Papers",
    "arxiv.org":                   "~/Research/Papers",
    "drive.google.com":            "~/Documents/Cloud",
    "dropbox.com":                 "~/Documents/Cloud",
    "fonts.google.com":            "~/Library/Fonts/Downloads",
}

destination = routing_table.get(host) or ai_classify(file_path)
```

---

## 10. DFIR Usage and Documented Data Formats

### 10.1 Key Tools Used by Investigators

- **Velociraptor** — has a built-in `MacOS.System.QuarantineEvents` artifact that queries `QuarantineEventsV2.db` across enrolled endpoints and returns structured quarantine event records.
- **Plaso (log2timeline)** — the `ls_quarantine` SQLite plugin parses `QuarantineEventsV2.db` and emits timeline events with timestamps, download URLs, and agent names.
- **osxmetadata (Python)** — a library by Rhett Bull that reads `kMDItemWhereFroms` and other xattrs, useful for bulk forensic processing.
- **xattred** — Howard Oakley's GUI tool for macOS that can inspect and manipulate all xattrs on files, including displaying provenance xattr binary content.

### 10.2 Documented Findings from DFIR Research

From the DFIR community (dfir.ch, eclecticlight.co, jmussman.net):

1. **Quarantine entries persist** long after the downloaded file is deleted or moved. The database is a historical log, not a live index of current files.
2. **QuarantineEventsV2 survives migrations** — when a user migrates to a new Mac using Migration Assistant, the database is copied, giving investigators historical records from previous machines.
3. **The agent name field is not verified** — any application can write any string as the agent name when it sets the quarantine xattr. For forensic-grade attribution, the bundle ID is more reliable.
4. **kMDItemWhereFroms is stripped by some tools** — when files are copied with `cp`, the xattr is preserved; when transferred via AirDrop or iCloud Drive, it may be stripped. `rsync -X` or `ditto` preserve xattrs.
5. **com.apple.provenance is SIP-protected on `.app` bundles** — once set by macOS on an app bundle, the provenance xattr cannot be removed by a non-root process even by the app owner, making it tamper-evident.
6. **Quarantine entries have no TTL** — unlike browser history, Apple does not purge `QuarantineEventsV2` automatically. Users can clear it manually (`sqlite3 ~/Library/Preferences/com.apple.LaunchServices.QuarantineEventsV2 "DELETE FROM LSQuarantineEvent;"`), and some privacy-focused utilities do this on schedule.

### 10.3 Signal Reliability Matrix

| Signal | URL Available | Requires DB | Survives Copy | Works Without Internet Download |
|---|---|---|---|---|
| `kMDItemWhereFroms` | Direct | No | Usually | No |
| `com.apple.quarantine` + DB | Via DB | Yes | Yes (xattr) / Maybe (DB entry) | No |
| `com.apple.provenance` | No (app ID only) | Root required | Usually | Yes (app writes) |

---

## 11. Implementation Recommendation for Curator

Given Curator's routing use case, the optimal implementation strategy is:

1. **Primary signal**: Read `com.apple.metadata:kMDItemWhereFroms` using `getxattr`. Parse the binary plist with `plist` crate in Rust or `PropertyListSerialization` in Swift. Extract `referring_url[1]` as the canonical host — it is the human-visible page URL, not a CDN artifact URL.

2. **Fallback signal**: Parse `com.apple.quarantine` xattr string. For non-sandboxed builds, query `QuarantineEventsV2.db` with the UUID. For sandboxed builds, use only the agent name from the xattr string (fields 2 and 3) to determine which app downloaded the file.

3. **Host normalization**: Strip `www.`, normalize to eTLD+1 (use the `publicsuffix` crate), then match against Curator's routing table.

4. **`com.apple.provenance` for app-origin routing**: Use the provenance xattr to identify files last written by specific developer tools (e.g., Xcode build artifacts, downloaded by a specific downloader app). This is a secondary enrichment signal, not the primary routing key.

5. **Graceful degradation**: If no provenance signal is found, fall back to Curator's AI content-type classifier. Many files (created locally, synced from cloud without quarantine) will have no provenance xattr.

---

## Sources

- [Ventura has changed app quarantine with a new xattr – The Eclectic Light Company](https://eclecticlight.co/2023/03/13/ventura-has-changed-app-quarantine-with-a-new-xattr/)
- [What is macOS Ventura doing tracking provenance? – The Eclectic Light Company](https://eclecticlight.co/2023/03/16/what-is-macos-ventura-doing-tracking-provenance/)
- [How macOS now tracks the provenance of apps – The Eclectic Light Company](https://eclecticlight.co/2023/05/10/how-macos-now-tracks-the-provenance-of-apps/)
- [From quarantine to provenance: extended attributes – The Eclectic Light Company](https://eclecticlight.co/2024/09/12/from-quarantine-to-provenance-extended-attributes/)
- [Quarantine, MACL and provenance: what are they up to? – The Eclectic Light Company](https://eclecticlight.co/2025/12/05/quarantine-macl-and-provenance-what-are-they-up-to/)
- [What is the com.apple.provenance xattr? – Apple Developer Forums](https://developer.apple.com/forums/thread/723397)
- [Michael Tsai - Blog - Ventura Adds com.apple.provenance](https://mjtsai.com/blog/2023/03/16/ventura-adds-com-apple-provenance/)
- [macOS Extended Attributes: Case Study – dfir.ch](https://dfir.ch/posts/macos_extended_attributes/)
- [Extended Attributes macOS Forensics – HackMD](https://hackmd.io/@M4shl3/xattr)
- [MacOS.System.QuarantineEvents – Velociraptor Documentation](https://docs.velociraptor.app/artifact_references/pages/macos.system.quarantineevents/)
- [Quarantine Introduction – 0xmachos](https://0xmachos.com/2019-02-01-Quarantine-Intro/)
- [xattr: com.apple.quarantine, the quarantine flag – The Eclectic Light Company](https://eclecticlight.co/2017/12/11/xattr-com-apple-quarantine-the-quarantine-flag/)
- [xattr: com.apple.metadata:kMDItemWhereFroms – The Eclectic Light Company](https://eclecticlight.co/2017/12/21/xattr-com-apple-metadatakmditemwherefroms-origin-of-downloaded-file/)
- [Beyond Scripting in Swift: Direct access to xattrs – The Eclectic Light Company](https://eclecticlight.co/2017/09/06/beyond-scripting-in-swift-direct-access-to-xattrs-calling-c-and-converting-data-to-strings/)
- [Sandboxing makes quarantine flags almost meaningless – The Eclectic Light Company](https://eclecticlight.co/2019/04/15/sandboxing-makes-quarantine-flags-almost-meaningless/)
- [xattr crate – crates.io](https://crates.io/crates/xattr)
- [xattr crate – docs.rs](https://docs.rs/xattr/latest/xattr/)
- [plaso ls_quarantine plugin – Plaso documentation](https://plaso.readthedocs.io/en/latest/_modules/plaso/parsers/sqlite_plugins/ls_quarantine.html)
- [All files touched by Zed have xattr: com.apple.provenance – GitHub Issue](https://github.com/zed-industries/zed/issues/26908)
- [Mac Forensics in 2026 – Jordan Mussman](https://jmussman.net/posts/mac_dfir/)
