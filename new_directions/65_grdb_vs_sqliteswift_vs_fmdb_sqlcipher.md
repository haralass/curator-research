# GRDB vs SQLite.swift vs FMDB for SQLCipher Encrypted SQLite in Swift
**File:** 65 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary / Recommendation
**GRDB** is the clear winner. As of v7.10.0 (August 2025), GRDB added native SQLCipher + Swift Package Manager support, eliminating the previous friction point. It has the richest Swift API, active maintenance, and the best query builder. SQLite.swift is a distant second; FMDB is Objective-C and should not be used in new Swift projects.

## Prior Art & Evidence

### GRDB (v7.x, 2025)
- **SQLCipher support:** `pod 'GRDB.swift/SQLCipher'` (CocoaPods) or `GRDB/SQLCipher` SPM target (added v7.10.0, Aug 2025)
- Active development: ~7,500 GitHub stars, commits weekly
- Features: record types, query builder, migrations, reactive (Combine/async), full-text search (FTS5), WAL mode
- Used by: Automattic (WordPress iOS), various production apps
- Swift concurrency: full async/await support added in v6

### SQLite.swift
- SQLCipher: `SQLite.swift/SQLCipher` pod, but SPM SQLCipher integration is more manual
- ~9,000 stars but slower maintenance cadence
- Functional but less ergonomic than GRDB for complex queries
- No built-in migration system

### FMDB
- Objective-C wrapper, Swift usage via bridging header
- ~13,000 stars but legacy — no Swift concurrency, no type safety
- SQLCipher: `FMDB/SQLCipher` pod
- **Not recommended for new Swift projects**

## Tradeoffs

| Dimension | GRDB | SQLite.swift | FMDB |
|-----------|------|-------------|------|
| Language | Pure Swift | Pure Swift | ObjC |
| SQLCipher + SPM | ✅ (v7.10) | Partial | Pod only |
| Query builder | Rich, type-safe | Good | Raw SQL only |
| Migrations | Built-in | Manual | Manual |
| Async/await | ✅ | Partial | ❌ |
| Maintenance | Active | Moderate | Slow |
| FTS5 | ✅ | ❌ | ❌ |

## Decision for Curator
**GRDB with SQLCipher.** 

Curator's metadata store needs: encrypted at rest (user file paths/names are sensitive), FTS5 for filename search, migrations for schema evolution, and async reads on background queue. GRDB provides all four natively.

```swift
// Package.swift dependency
.package(url: "https://github.com/groue/GRDB.swift", from: "7.10.0")
// target dependency
.product(name: "GRDB", package: "GRDB.swift") // or SQLCipher variant
```

Encryption key: derive from user's keychain entry using `SecKeyGenerateSymmetric` or `CryptoKit.SymmetricKey`, pass to SQLCipher via `sqlite3_key`.

## Sources
- https://github.com/groue/GRDB.swift
- https://github.com/groue/GRDB.swift/blob/master/Documentation/SQLCipher.md
- https://github.com/stephencelis/SQLite.swift
- https://github.com/ccgus/fmdb
- SQLCipher docs: https://www.zetetic.net/sqlcipher/sqlcipher-api/
