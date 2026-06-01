# SwiftUI Menu Bar Persistent App Architecture
**File:** 64 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary / Recommendation
Use **MenuBarExtra** (macOS 13+) with `LSUIElement = YES` in Info.plist and `.activationPolicy(.accessory)` to create a persistent menu bar app with no Dock icon. This is the modern SwiftUI-native pattern as of 2022. For macOS 12 support, fall back to NSStatusItem + NSHostingController.

## Prior Art & Evidence

### MenuBarExtra (macOS 13+, SwiftUI)
- Introduced WWDC 2022 — `MenuBarExtra("Curator", systemImage: "folder.badge.gear")`
- Two styles: `.menuBarExtraStyle(.menu)` (dropdown menu) or `.menuBarExtraStyle(.window)` (floating panel)
- Works with SwiftUI Scene architecture — no AppKit required

### LSUIElement pattern
- `LSUIElement = YES` in Info.plist: app runs as "agent app" — no Dock icon, no menu bar (app menu)
- Can be combined with MenuBarExtra for status item
- Alternative: `NSApp.setActivationPolicy(.accessory)` at runtime

### Background execution
- macOS does not suspend menu bar apps the way iOS suspends background apps
- `NSBackgroundActivityScheduler` for periodic tasks (file watching, re-indexing)
- `FSEventStreamCreate` for real-time file system events — works indefinitely in background

### Login item (launch at login)
- macOS 13+: `SMAppService.mainApp.register()` — modern API, no LaunchAgent plist
- Pre-13: LaunchAgent plist in ~/Library/LaunchAgents

## Architecture for Curator

```swift
@main
struct CuratorApp: App {
    @NSApplicationDelegateAdaptor(AppDelegate.self) var appDelegate
    
    var body: some Scene {
        MenuBarExtra("Curator", systemImage: "folder.badge.sparkle") {
            ContentView()
                .frame(width: 340, height: 480)
        }
        .menuBarExtraStyle(.window)
    }
}

class AppDelegate: NSObject, NSApplicationDelegate {
    func applicationDidFinishLaunching(_ notification: Notification) {
        NSApp.setActivationPolicy(.accessory) // no Dock icon
    }
}
```

Info.plist:
```xml
<key>LSUIElement</key>
<true/>
<key>LSBackgroundOnly</key>
<false/>
```

## Tradeoffs

| Approach | macOS support | SwiftUI native | Complexity |
|----------|---------------|----------------|------------|
| MenuBarExtra | 13+ | Yes | Low |
| NSStatusItem + NSHostingController | 10.14+ | Partial | Medium |
| NSStatusItem + NSPopover | 10.14+ | Yes | Medium |
| AppKit NSMenu | All | No | High |

## Decision for Curator
**MenuBarExtra with `.window` style** — gives a floating panel (like 1Password mini, Bartender) which is appropriate for Curator's approval/review UI. Require macOS 13 (Ventura) minimum — released Sept 2022, well-adopted by 2026.

## Sources
- https://developer.apple.com/documentation/swiftui/menubarextra
- WWDC 2022: "Complications and collections" session
- https://developer.apple.com/documentation/servicemanagement/smappservice
- https://developer.apple.com/documentation/coreservices/file_system_events
