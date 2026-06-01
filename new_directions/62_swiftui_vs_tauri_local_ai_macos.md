# SwiftUI vs Tauri for Local AI macOS App
**File:** 62 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary / Recommendation
SwiftUI is the right choice for Curator. Native performance, first-class macOS entitlements (Full Disk Access, file bookmarks, Sandbox), and a clear App Store path make it superior for a file organizer that needs deep OS integration. Tauri offers cross-platform reach but adds unnecessary complexity (Rust core + WebView layer) for a macOS-only app.

## Prior Art & Evidence

### SwiftUI local AI apps shipping on macOS:
- **BoltAI** — native SwiftUI menu bar app with local LLM support (Ollama, LM Studio); App Store + direct distribution
- **Reeder**, **Tot**, **Pockity** — successful SwiftUI-only macOS menu bar apps proving the architecture
- **LM Studio** — Electron (cross-platform), but notably slower and heavier than native equivalents
- **Whisper transcription apps** (Whisper Transcription, MacWhisper) — SwiftUI + Python/CoreML sidecar pattern, shipping on App Store

### Tauri + Python apps:
- Tauri v2 supports iOS/Android but macOS sandboxing with Full Disk Access via Tauri requires custom Rust plugins
- Tauri apps use WKWebView for UI — adds ~30-50ms render overhead vs native SwiftUI
- No App Store SwiftUI-comparable Tauri app exists in the file management category

## Tradeoffs

| Dimension | SwiftUI | Tauri |
|-----------|---------|-------|
| File system access | Native NSOpenPanel, security-scoped bookmarks, FDA entitlement | Custom Rust plugin needed for persistent access |
| Menu bar | MenuBarExtra (macOS 13+), native | Tray icon via JS, less native feel |
| Python sidecar | Simple Process/XPC launch | Same (Tauri can spawn sidecar processes) |
| Performance | Native Metal rendering, <1ms UI | WKWebView adds overhead |
| App Store | Well-understood entitlement path | Possible but less documented |
| Dev experience | Swift + Xcode | Rust + HTML/JS/CSS |
| Cross-platform | macOS only | macOS, Windows, Linux, mobile |

## Decision for Curator
**Confirmed: SwiftUI.** Curator is macOS-only by design (uses macOS-specific APIs for file watching, Spotlight metadata, security-scoped bookmarks). The cross-platform argument for Tauri doesn't apply. Native SwiftUI gives direct access to FileManager, NSMetadataQuery, and the entitlement system needed for Full Disk Access — all critical for a file organizer.

## Sources
- https://tauri.app/v1/guides/building/macos
- https://developer.apple.com/documentation/swiftui/menubarextra
- https://boltai.com (BoltAI native SwiftUI AI app)
- https://github.com/nicklockwood/SwiftFormat (tooling ecosystem evidence)
- WWDC 2022 Session: "Bring multiple windows to your SwiftUI app"
