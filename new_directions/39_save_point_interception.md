# 39 — Save-Point Interception: AI Folder Suggestion at Save Time
_Date: 2026-05-31_

## Core Idea

Every existing AI file organizer acts **post-hoc**: the file lands somewhere (Downloads, Desktop, an arbitrary folder) and the system later moves or tags it. Save-Point Interception inverts this: Curator intercepts the native macOS Save dialog at the moment the user first names a file, and displays a ranked folder suggestion **before** the file is written to disk. If the suggestion is accepted, the file is written directly to the correct location — it never needs to move.

This is the difference between reactive organization and proactive organization. The reactive model leaves files stranded in the wrong place for hours, days, or permanently. The proactive model eliminates displacement at the moment of creation.

---

## Prior Art

**Default Folder X** (St. Clair Software, v6.x) is the closest commercial product. It injects UI into every Open/Save dialog on macOS using a combination of the Accessibility API and a system extension. However:
- Its suggestions are based purely on **recency and favorites** — no semantic model.
- It has no knowledge of file content, cluster membership, or FLRS scores.
- It does not use any local AI or embedding model.
- It relies on partially private APIs and has historically broken between macOS versions.

No academic paper treats pre-save folder prediction as a structured prediction or classification problem. Searches of ACM DL, IUI proceedings (2020–2025), and CHI proceedings confirm zero publications on this topic.

**Closest academic work:**
- Barreau & Nardi (1995): established that users create folders at save time using "location-based" filing — the moment of saving is cognitively loaded and consequential.
- Bergman et al. (2010): showed that user-created folder hierarchies result in 65% lower retrieval failure than system defaults, confirming that the save-time decision matters.
- Jones & Teevan (2007): PIM survey chapter on filing strategies — identifies "filers" (who place files carefully at save time) vs. "pilers" (who save anywhere and search later). A save-time AI would give pilers filer-quality organization for free.

---

## Technical Implementation

### Signal Extraction at Save Time

When the user opens a Save panel (`NSSavePanel`), the following signals are available without any user interaction:

| Signal | Source | Extraction Method |
|---|---|---|
| Typed filename (partial) | NSSavePanel.nameFieldStringValue | KVO or delegate callback |
| File extension (from type popup) | NSSavePanel.allowedContentTypes | Public API |
| Active application | NSWorkspace.frontmostApplication | Public API |
| Application window title | AXUIElement / Accessibility API | Accessibility permission |
| Time of day | Date() | Trivial |
| Prior session files | Curator's own access log | Internal |

### Injection Mechanism

The standard approach (used by Default Folder X) uses two components:

1. **Finder Sync Extension**: Registered with the OS, guaranteed to be active whenever a Save panel appears. Communicates via XPC with the main Curator agent.
2. **Accessibility overlay**: The main agent reads the frontmost NSSavePanel via `AXUIElement`, identifies the panel's window, and injects either (a) an `accessoryView` if the app supports `NSOpenSavePanelDelegate`, or (b) a companion popover anchored to the panel window.

`NSSavePanel` has a public `accessoryView` property and `NSOpenSavePanelDelegate` protocol (Apple Developer Documentation, AppKit). Apps that implement the delegate (most Cocoa apps) will render an injected view natively inside the panel. For apps that don't, the companion popover approach is a reliable fallback.

This pattern is App Store-distributable (Default Folder X ships on the App Store) and stable through macOS Sequoia 15.

### Classification Model

Given the extracted signals, Curator runs a folder prediction model locally:

```
Input features:
  - filename_embedding: embed(typed_filename) via FLACON/BGE-M3 [128-dim]
  - app_id: one-hot over known bundle IDs (or embed if unseen)
  - window_title_embedding: embed(window_title) [128-dim]
  - hour_of_day: sin/cos encoding [2-dim]
  - top_k_recent_clusters: last 5 cluster IDs accessed [k-dim]

Output: ranked list of folder candidates with confidence scores
```

Latency requirement: **<50ms** (imperceptible to user). A quantized sentence transformer (e.g., MiniLM-L6 quantized to INT8) embeds a filename in ~5ms on Apple Silicon. Folder ranking via cosine similarity over pre-computed cluster centroids takes <1ms. Total pipeline is well within budget.

### Conformal Prediction Integration

Curator's existing CP routing layer (files 16, 23) can be applied directly at save time. The prediction set `C(x)` contains all folders with coverage guarantee at level `1-α`. Curator displays the top-3 from `C(x)` in the Save panel UI, ordered by nonconformity score. The user can accept the top suggestion (one click) or expand to see the full prediction set.

This gives the save-time suggestion a **formal coverage guarantee** — a property no existing save dialog tool has.

---

## "Zero-Displacement Saves" Metric

**Definition:** A file save is *zero-displacement* if the file's first-written location equals its long-term stable location (i.e., the file is never moved or re-filed after initial save).

Formally: let `loc_0(f)` be the folder where file `f` is first saved, and `loc_∞(f)` be the folder where it resides after 30 days. A save is zero-displacement iff `loc_0(f) = loc_∞(f)`.

**Zero-Displacement Rate (ZDR):**

```
ZDR = |{f : loc_0(f) = loc_∞(f)}| / |total files saved in period|
```

This is a novel PIM metric with no prior definition in the literature. It is computable from filesystem event logs (Endpoint Security Framework captures both creation and move events). It directly measures organizational efficiency: a ZDR of 1.0 means every file landed in the right place immediately; a ZDR of 0.3 means 70% of files were displaced after saving.

**Hypothesis:** Curator's save-point interception will significantly increase ZDR compared to unassisted saving. This is the primary outcome variable for the proposed user study.

---

## Research Contribution

### Primary Contribution
First empirical study framing **pre-save folder prediction as a structured prediction task**. Features, model architecture, and evaluation methodology are defined for the first time. The ZDR metric is introduced as a standard measure for organizational efficiency at save time.

### Secondary Contribution
First integration of conformal prediction into a save-time UI: the prediction set shown to the user carries a formal coverage guarantee. This is a novel combination of CP theory and interactive system design.

### Study Design
- **Within-subjects**, N=24 participants
- **Phase 1** (baseline): 2 weeks of normal saving behavior, ZDR computed from logs
- **Phase 2** (treatment): 2 weeks with Curator save-point suggestions active
- **Primary metric**: ZDR (pre vs. post)
- **Secondary metrics**: time-in-dialog (does the suggestion slow the user down?), suggestion acceptance rate, post-study recall of file locations (does saving correctly improve memory?)
- **Control**: Default Folder X recency suggestions vs. Curator AI suggestions vs. no assistance

---

## Connection to FLRS

Save-point interception improves FLRS in two ways:

1. **Better initial S₀**: A file saved in the correct location is encountered in the expected location on first retrieval — this strengthens the initial binding (Pertzov et al. 2012). A file saved in a wrong location and later moved creates a false memory trace.

2. **Elimination of "phantom locations"**: When a file is moved after saving, the user may remember the original (wrong) location. This creates a spurious location memory that competes with the correct one, artificially elevating FLRS decay. Zero-displacement saves eliminate this phenomenon.

---

## References

- Apple Inc. (2024). *NSSavePanel*. Apple Developer Documentation. https://developer.apple.com/documentation/appkit/nssavepanel
- Apple Inc. (2024). *NSOpenSavePanelDelegate*. Apple Developer Documentation. https://developer.apple.com/documentation/appkit/nsopensavepaneldelegate
- Apple Inc. (2023). *Finder Sync Extension Programming Guide*. Apple Developer Documentation. https://developer.apple.com/library/archive/documentation/General/Conceptual/ExtensibilityPG/Finder.html
- Barreau, D., & Nardi, B. A. (1995). Finding and reminding: File organization from the desktop. *ACM SIGCHI Bulletin, 27*(3), 39–43. https://doi.org/10.1145/221296.221307
- Bergman, O., Whittaker, S., Sanderson, M., Nachmias, R., & Ramamoorthy, A. (2010). The effect of folder structure on personal file navigation. *Journal of the American Society for Information Science and Technology, 61*(12), 2426–2441. https://doi.org/10.1002/asi.21415
- Jones, W., & Teevan, J. (Eds.). (2007). *Personal Information Management*. University of Washington Press.
- Pertzov, Y., Bays, P. M., Joseph, S., & Husain, M. (2012). Rapid forgetting prevented by brief offline reactivation of newly encoded visuospatial information. *Journal of Experimental Psychology: Human Perception and Performance, 38*(5), 1224–1235. https://doi.org/10.1037/a0028269
- St. Clair Software. (2024). *Default Folder X 6*. https://www.stclairsoft.com/DefaultFolderX/
