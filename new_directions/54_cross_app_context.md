# 54 — Cross-Application Context Capture for File Intent Inference
_Research date: 2026-05-31_

---

## Why This Matters for Curator

A file does not carry its meaning in isolation. A PDF called `2024-notes.pdf` could be a research paper, a tax document, a travel itinerary, or a novel draft — its bytes are the same regardless. What changes is the *context of use*: the other tools open at the same time, the broader task the user is mid-stream on. When Preview and Zotero are both running and the user opens a PDF, the co-presence of Zotero is a strong prior that the document is academic literature. When Xcode, Terminal, and a Markdown viewer are co-active and the user touches a `.txt` file, the most likely category is a coding scratchpad or technical note — not a personal journal.

This insight draws directly from the foundational principle of context-aware computing: context is "any information that can be used to characterize the situation of an entity" (Dey 2001). The running application set is cheap, always-available, and encodes the user's task-type at a coarse but useful granularity. It is a sensor that is already on, producing a continuous stream of task-type signals, requiring no new hardware and no explicit user input.

For Curator's level router — the component that decides whether a file belongs in a project-specific location, a topic-based folder, a time-stamped archive, or a fast-access tray — co-occurring application context is a low-cost, high-signal feature that can meaningfully shift classification confidence before any content-level analysis is done. It is the equivalent of knowing which desk a paper was found on before reading what it says.

---

## Context-Aware Computing: Foundational Literature

### Dey & Abowd (1999/2000/2001) — The Canonical Definition

The most-cited definition of context in computing is Anind K. Dey's: *"Context is any information that can be used to characterize the situation of an entity, where an entity can be a person, place, or physical or computational object."* The key corollary: a system is context-aware if it "uses context to provide relevant information and/or services to the user, where relevancy depends on the user's task." This definition explicitly elevates running applications — as "computational objects" — to first-class context signals.

Three key papers form the canonical arc:

**[1] Abowd, G. D., Dey, A. K., Brown, P. J., Davies, N., Smith, M., & Steggles, P. (1999). Towards a Better Understanding of Context and Context-Awareness.** In *Proceedings of the 1st International Symposium on Handheld and Ubiquitous Computing (HUC '99)*. ACM. DOI: `10.5555/647985.743843`. This workshop paper introduced the "5W" framework for context: *who, what, where, when, why* — establishing that task (the "what") and identity (the "who") are primary context elements, with location and time as secondary.

**[2] Dey, A. K. (2001). Understanding and Using Context.** *Personal and Ubiquitous Computing, 5*(1), 4–7. DOI: `10.1007/s007790170019`. This journal paper is the canonical reference with 5,500+ citations. It operationalises the context definition and introduces categories of context use: proximate selection, automatic contextualisation, context-triggered actions, and attached information/metadata. Curator's file routing maps cleanly onto "proximate selection" and "context-triggered actions."

**[3] Dey, A. K., Salber, D., & Abowd, G. D. (2001). A Conceptual Framework and a Toolkit for Supporting the Rapid Prototyping of Context-Aware Applications.** *Human-Computer Interaction (HCIJ), 16*(2–4), 97–166. This paper introduced the **Context Toolkit**, an architecture with three abstractions: *widgets* (mediators between environment sensors and applications), *aggregators* (combining multiple widget outputs), and *interpreters* (inferring higher-level context from raw signals). The mapping to Curator is direct: NSWorkspace notification listeners are the "widgets," a session aggregator collects co-present apps, and a classifier is the "interpreter" inferring task type.

### Task Context Models

Research in context-aware computing distinguishes between *primary context* (the user's immediate task), *secondary context* (resources being used), and *environmental context* (physical surroundings). For Curator, running applications are secondary context signals from which primary context (task type) can be inferred.

---

## Application Co-Occurrence as Task Signal

### Mobile Research: The Closest Analogue

**[4] Tian, Y., Zhou, K., Lalmas, M., & Pelleg, D. (2020). Identifying Tasks from Mobile App Usage Patterns.** In *Proceedings of SIGIR '20*, 1805–1808. ACM. DOI: `10.1145/3397271.3401441`. This Yahoo Research paper directly addresses task identification from app logs. Key findings: users accomplish tasks (e.g., "book a restaurant") by chaining interactions across multiple apps; temporal proximity, sequence, and app category similarity are the strongest features; a session-based model achieves meaningful task-boundary detection without requiring explicit user labels. This is the closest published analogue to Curator's use case, adapted to desktop.

**AppTrends** (Natarajan et al., RecSys 2013) builds app co-occurrence graphs from session logs, weighting edges by inverse co-occurrence frequency — apps appearing together rarely but consistently form stronger task signals than those that merely both appear often.

### Desktop/Activity Monitoring Research

**ActivityWatch** (open source, GitHub: `ActivityWatch/activitywatch`) is the most complete deployed implementation of desktop app co-occurrence logging. Its architecture: `aw-watcher-window` tracks the active window and app name every ~1 second; events are stored in a local bucket store with a REST API. The data model captures `(timestamp, app_name, window_title, duration)` tuples.

**RescueTime** follows a similar model but cloud-syncs data and categorises applications by productivity tier. It explicitly uses application category co-occurrence for session-level productivity scoring.

---

## Multi-Application Task Models

### The Fragmentation Baseline

**[5] Mark, G., González, V. M., & Harris, J. (2005). No Task Left Behind? Examining the Nature of Fragmented Work.** In *Proceedings of CHI '05*, 321–330. ACM. DOI: `10.1145/1054972.1055017`. The landmark study of information worker task fragmentation. Key empirical findings from observation of 24 knowledge workers:
- Workers averaged **only 3 minutes** on any single task before switching.
- 57% of working spheres were interrupted before completion.
- Workers used an average of **3.9 applications simultaneously** for a given task.
- Most internal interruptions were self-initiated.

The implication for Curator: a file touched while Zoom, Notion, and a browser are co-active is most likely part of a meeting/collaboration task. A file touched while Xcode, Terminal, and a diff viewer are co-active is part of a coding task. The co-application set *is* the task fingerprint.

**[6] Czerwinski, M., Horvitz, E., & Wilhite, S. (2004). A Diary Study of Task Switching and Interruptions.** In *Proceedings of CHI '04*, 175–182. ACM. DOI: `10.1145/985692.985715`. Week-long diary study of information workers. Workers had an average of 8 active task threads at any given time and switched between them with minimal re-orientation cost. Establishes that the co-present app set reflects real concurrent task structure.

**[7] Voida, S., Mynatt, E. D., MacIntyre, B., & Corso, G. M. (2002). Integrating Virtual and Physical Context to Support Knowledge Workers.** *IEEE Pervasive Computing, 1*(3), 73–79. Argues that the desktop computing environment should be modelled as an *activity space* — a named, persistent grouping of applications and documents that encode a task. This is essentially what macOS Focus Modes and Spaces do at a user-visible level, and what Curator would infer automatically.

**[8] Mark, G., Iqbal, S. T., Czerwinski, M., Johns, P., & Sano, A. (2016). Neurotics Can't Focus: An in Situ Study of Online Multitasking in the Workplace.** In *Proceedings of CHI '16*. ACM. Confirmed that simultaneous application use — rather than sequential use — characterises high-cognitive-load tasks. Files accessed during high-app-count sessions are more likely to be task-critical, active-work documents rather than reference or archive material.

---

## macOS Implementation: How to Capture App Co-Occurrence

### NSWorkspace Notification APIs

The core macOS API for monitoring application lifecycle is `NSWorkspace`, part of AppKit. Key notifications:

```swift
// Post when an application launches
NSWorkspace.didLaunchApplicationNotification

// Post when an application becomes frontmost
NSWorkspace.didActivateApplicationNotification

// Post when an application loses frontmost status
NSWorkspace.didDeactivateApplicationNotification

// Post when an application quits
NSWorkspace.didTerminateApplicationNotification
```

Apple Developer Documentation: https://developer.apple.com/documentation/appkit/nsworkspace

A snapshot of currently running applications is available at any moment via:

```swift
let runningApps = NSWorkspace.shared.runningApplications
// Returns [NSRunningApplication] with .localizedName, .bundleIdentifier, .processIdentifier
```

### Subscribing to Notifications

```swift
let nc = NSWorkspace.shared.notificationCenter
nc.addObserver(forName: NSWorkspace.didActivateApplicationNotification,
               object: nil, queue: .main) { notification in
    let app = notification.userInfo?[NSWorkspace.applicationUserInfoKey] 
              as? NSRunningApplication
    let bundleID = app?.bundleIdentifier ?? "unknown"
    // Record to session state
}
```

### Privacy and Sandboxing Constraints

**Sandboxed apps**: Calling `NSWorkspace.shared.runningApplications` does **not** require any special entitlement. It is considered non-sensitive metadata about the user's own session. No TCC (Transparency, Consent, and Control) permission dialog is triggered.

**TCC boundaries that do NOT apply**: Microphone, camera, location, contacts, photos, full disk access — none are triggered by reading running application names. Observing *which apps are running* is not gated by TCC in macOS 12–15.

**What DOES require entitlements**: Reading window titles (requires Accessibility / `com.apple.security.automation.apple-events`), reading document names from other apps' open files. Curator needs only bundle IDs, not window titles — keeping it in the safe zone.

### Focus Mode as Context Signal

macOS Focus Modes (introduced in macOS Monterey 12) encode user-defined task contexts. They are a ground-truth, user-labelled context signal.

Reading current Focus status requires the Intents framework:

```swift
import Intents

INFocusStatusCenter.default.requestAuthorization { status in
    if status == .authorized {
        let isFocused = INFocusStatusCenter.default.focusStatus.isFocused
        // isFocused is true if *any* Focus is active
    }
}
```

Apple Developer Documentation: https://developer.apple.com/documentation/intents/infocusstatuscenter

**Limitation**: `INFocusStatusCenter` only exposes a binary `isFocused: Bool?` — it does not expose the *name* of the active Focus mode. The name is accessible via undocumented APIs (reading `~/Library/DoNotDisturb/DB/Assertions.json`), as demonstrated by the open-source tool `getfocus` (GitHub: `davidolrik/getfocus`). This approach is reliable but could break with OS updates.

WWDC22 session "Meet Focus filters": https://developer.apple.com/videos/play/wwdc2022/10121/

---

## Application Transition Graphs

### The Graph Model

An application transition graph models the desktop session as a directed graph where:
- **Nodes** = application bundle identifiers
- **Edges** = observed transitions (app A was active, then app B became active)
- **Edge weight** = transition frequency or temporal proximity

This structure captures habitual multi-application workflows. A recurring pattern like `Xcode → Terminal → Simulator` is a recognisable task type. A recurring `Zotero → Preview → Word` pattern signals academic writing.

### Research Precedents

**MISApp** (arXiv:2603.21653, 2026) applies multi-hop session graph learning to next-app prediction on mobile devices, constructing graphs of session-level transitions and learning node embeddings that capture task structure. The graph architecture is directly portable to desktop.

**HMM-based task inference**: Hidden Markov Models applied to application-switching sequences have achieved ~85% accuracy in inferring user tasks from mouse movement and app transition data (Frontiers in Psychology, 2015, DOI: `10.3389/fpsyg.2015.00919`). The latent states in an HMM map naturally onto task types (coding, writing, research, communication), with observed application activations as emissions.

### Practical Implementation for Curator

A lightweight directed graph maintained in-memory using a dictionary of `(from_bundle, to_bundle) → count` pairs. Over a session of 50–100 application transitions, this graph becomes informative for identifying the current task cluster. A rolling 30-minute window is sufficient for real-time classification.

---

## Privacy & Consent for App Monitoring

### What macOS Requires

Reading running application names via `NSWorkspace.shared.runningApplications` does **not** require any TCC permission. It is session-level metadata, not personal data under Apple's privacy framework.

### How Deployed Tools Handle Consent

**ActivityWatch**: Explicit local-first, zero-upload model. All data stays on-device; architecture is open-source and auditable. Users are informed on first launch that application names will be logged locally. Privacy policy explicitly distinguishes between data collected (app names, window titles) and data never collected (keystrokes, passwords, file contents). This is the appropriate model for Curator.

**RescueTime**: Provides granular opt-out controls (whitelist specific apps, stop tracking at any time, delete data on demand). Key design pattern: user-visible dashboard showing exactly what is being tracked, with per-app and per-category opt-out.

### Recommended Consent Pattern for Curator

1. **Disclose at onboarding**: "Curator observes which applications are running while you open files. This data never leaves your device and is used only to improve file categorisation."
2. **Use only bundle IDs, not window titles**: Bundle identifiers (`com.apple.Preview`, `org.zotero.zotero`) are less sensitive than window titles.
3. **No persistence beyond session**: Co-occurrence data is ephemeral — used to inform a routing decision at file-open time and then discarded.
4. **User-visible exclusions**: Allow users to mark specific apps as "private" (e.g., a mental health app, a financial app) that Curator will never log as context signals.

---

## Feature Design: Using App Co-Occurrence in Curator's Level Router

### Feature Extraction

From the running application set at file-open time, extract:

1. **Category vector**: Map each bundle ID to a category (IDE, browser, writing, communication, media, finance, etc.) using a lookup table. Represent the session as a bag-of-categories.
2. **Anchor app**: The single most task-specific app running. Zotero anchors "research"; Xcode anchors "development"; Final Cut Pro anchors "video production." Anchor app alone is often sufficient for coarse routing.
3. **Transition recency**: Was the file-receiving app (e.g., Preview) activated within the last 60 seconds from a task-specific app (e.g., Zotero)? Recency of transition is a stronger signal than mere co-presence.
4. **Session diversity score**: Count of distinct app categories active in the current 30-minute window. High diversity → exploratory/research session. Low diversity (2–3 apps, same category) → focused execution session.
5. **Focus mode binary**: Is any Focus mode active? Active Focus strongly predicts intentional, task-directed work.

### Integration into the Level Router

App co-occurrence adds a *session context prior* that re-weights the probability distribution over destination categories before content analysis runs:

```
P(category | file) ∝ P(category | file_metadata) × P(category | app_context)
```

If the anchor app is Zotero and the file is a PDF, the router can skip content analysis and route directly to the research papers folder with high confidence. If no anchor app is detected, the router falls back to content analysis.

### Suggested App-to-Category Mapping Seed

| Bundle ID | Category |
|---|---|
| `org.zotero.zotero` | academic-research |
| `com.apple.dt.Xcode` | software-development |
| `com.microsoft.VSCode` | software-development |
| `com.apple.Terminal` | software-development |
| `com.adobe.Premiere` | video-production |
| `com.adobe.Photoshop` | creative-design |
| `com.figma.Desktop` | creative-design |
| `com.apple.iWork.Pages` | writing |
| `com.microsoft.Word` | writing |
| `com.apple.finder` | (no signal) |
| `com.zoom.xos` | communication |
| `com.apple.mail` | communication |
| `com.intuit.turbotax` | finance |

This table should be user-extensible and crowdsource-able.

---

## Key References

1. Dey, A. K. (2001). Understanding and Using Context. *Personal and Ubiquitous Computing, 5*(1), 4–7. DOI: 10.1007/s007790170019.
2. Abowd, G. D., Dey, A. K., et al. (1999). Towards a Better Understanding of Context and Context-Awareness. *HUC '99*. ACM DOI: 10.5555/647985.743843.
3. Dey, A. K., Salber, D., & Abowd, G. D. (2001). A Conceptual Framework and a Toolkit for Supporting the Rapid Prototyping of Context-Aware Applications. *HCIJ, 16*(2–4).
4. Mark, G., González, V. M., & Harris, J. (2005). No Task Left Behind? *CHI '05*. ACM DOI: 10.1145/1054972.1055017.
5. Czerwinski, M., Horvitz, E., & Wilhite, S. (2004). A Diary Study of Task Switching and Interruptions. *CHI '04*. ACM DOI: 10.1145/985692.985715.
6. Tian, Y., Zhou, K., Lalmas, M., & Pelleg, D. (2020). Identifying Tasks from Mobile App Usage Patterns. *SIGIR '20*. ACM DOI: 10.1145/3397271.3401441.
7. Mark, G., Iqbal, S. T., Czerwinski, M., Johns, P., & Sano, A. (2016). Neurotics Can't Focus. *CHI '16*. ACM.
8. Voida, S., Mynatt, E. D., MacIntyre, B., & Corso, G. M. (2002). Integrating Virtual and Physical Context to Support Knowledge Workers. *IEEE Pervasive Computing, 1*(3), 73–79.
9. Apple Developer Documentation — NSWorkspace. https://developer.apple.com/documentation/appkit/nsworkspace
10. Apple Developer Documentation — INFocusStatusCenter. https://developer.apple.com/documentation/intents/infocusstatuscenter
11. Apple WWDC22 — Meet Focus Filters. https://developer.apple.com/videos/play/wwdc2022/10121/
12. ActivityWatch — Open-source time tracker. GitHub: https://github.com/ActivityWatch/activitywatch
13. ActivityWatch Privacy Policy. https://docs.activitywatch.net/en/latest/privacy.html
14. RescueTime Privacy and Monitoring Options. https://help.rescuetime.com/article/45-monitoring-options
15. MISApp (2026). Multi-Hop Intent-Aware Session Graph Learning for Next App Prediction. arXiv:2603.21653.
16. HMM task inference from interaction sequences. *Frontiers in Psychology* (2015). DOI: 10.3389/fpsyg.2015.00919.
17. davidolrik/getfocus (GitHub). https://github.com/davidolrik/getfocus — reading active Focus mode name from undocumented APIs.
