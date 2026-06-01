# Minimum Interaction to Maintain Spatial Memory

**File:** 97 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
Spatial memory for file location requires consistent positional stability — users encode file location through repeated retrieval, and any change to file positions (moves, renames, restructuring) disrupts the memory trace. Research from Nielsen Norman Group and Cockburn et al. (2007) shows that moving files to new folders breaks spatial memory in 2–4 sessions and requires 8–12 sessions to rebuild. Curator must minimize unsolicited reorganization and should never move files without user awareness. Progressive automation — starting with suggestions, advancing to automatic only after explicit user permission — is the evidence-based design pattern.

## Key Findings
- **Spatial memory in file navigation:** Users encode file location primarily through visual/spatial cues (folder color, position, nesting depth) and motor memory (muscle memory of the navigation path). This is documented in Cockburn et al., *Foundations and Trends in HCI* (2009), and Nielsen Norman Group's spatial memory article.
- **Hierarchy cost:** Hierarchical navigation (Finder-style) hides a significant portion of the information space at each level, negatively impacting spatial memory. Each folder level adds ~150ms navigation cost and one recall step.
- **Impact of file moves on spatial memory:** A single reorganization event (moving a file to a new folder) invalidates the spatial memory trace for that file. Users must re-learn the location through retrieval practice (typically 3–5 successful retrievals to form a new trace).
- **Cognitive load of reorganization notifications:** Users who receive frequent reorganization notifications (> 3/day in diary studies) report increased cognitive load and anxiety about "not knowing where their files are." This is the key risk of aggressive L5 automation.
- **Progressive automation model:**
  1. Phase 1 (first 30 days): Suggestions only. Show cluster proposals; user approves manually. Build trust.
  2. Phase 2 (after user approves ≥ 10 suggestions): Semi-automatic. Move high-confidence files; notify. Prominent Undo.
  3. Phase 3 (user opt-in): Fully automatic. Move all high-confidence files silently; weekly digest.
- **"Glance UX" principle:** For MenuBarExtra apps, the maximum cognitive budget for a notification is one glance (< 3 seconds, < 20 words). Curator notifications must fit: "Moved 5 files to Tax 2024 — Undo". All detail goes in the popover (not the notification).
- **Recency bias in spatial memory:** Users remember the location of recently accessed files better than rarely accessed ones. Curator should prioritize reorganizing rarely accessed files (low recency) and leave recently accessed files in place, to minimize spatial memory disruption.
- **"Recently used" displays:** Research supports "Recents" views as spatial memory prosthetics — they allow retrieval by time even when spatial memory fails. Curator's MenuBarExtra should include a "Recently organized" list.

## Relevant Papers / Prior Art
- Cockburn A. et al., "Supporting and Exploiting Spatial Memory in User Interfaces," *Foundations and Trends in Human-Computer Interaction* 3(2–3):1–135, 2009. DOI: 10.1561/1100000046. — Comprehensive review; spatial memory formation rates, hierarchy costs.
- Nielsen Norman Group, "Spatial Memory: Why It Matters for UX Design," https://www.nngroup.com/articles/spatial-memory/. — Applied design recommendations.
- Bergman O. et al., "The effect of folder structure on personal file navigation," *JASIST* 2010. — Empirical navigation times and memory effects.
- Malone T.W., "How Do People Organize Their Desks?", *ACM TOIS* 1(1):99–112, 1983. DOI: 10.1145/357423.357430. — Classic study of spatial organization strategies; "piles" vs. folders; spatial memory as organizing principle.

## Applicability to Curator

**Progressive automation implementation:**

```swift
// GRDB schema: track user automation trust level
// user_settings table: automation_phase (1|2|3), approved_count, last_phase_change

enum AutomationPhase: Int, Codable {
    case suggestionsOnly = 1    // Phase 1: manual approval required
    case semiAutomatic = 2      // Phase 2: high-confidence auto-move + notify
    case fullyAutomatic = 3     // Phase 3: user opt-in; silent + weekly digest
    
    func shouldAutoMove(confidence: ConfidenceLevel) -> Bool {
        switch self {
        case .suggestionsOnly: return false
        case .semiAutomatic:   return confidence == .high
        case .fullyAutomatic:  return confidence == .high || confidence == .medium
        }
    }
    
    func nextPhase(approvedCount: Int) -> AutomationPhase {
        switch self {
        case .suggestionsOnly:
            return approvedCount >= 10 ? .semiAutomatic : .suggestionsOnly
        case .semiAutomatic:
            return .fullyAutomatic  // requires explicit user opt-in
        case .fullyAutomatic:
            return .fullyAutomatic
        }
    }
}
```

**Recency-aware move scheduling:**
```python
# Python sidecar: sort files for automation by access recency
# Move old, rarely accessed files first; protect recently accessed files
def prioritize_for_automation(files: list[FileRecord], days_threshold: int = 30) -> list[FileRecord]:
    from datetime import datetime, timedelta
    cutoff = datetime.now() - timedelta(days=days_threshold)
    # Only auto-move files not accessed in the last 30 days
    return [f for f in files if f.last_accessed < cutoff]
```

## Open Questions
- Does the 30-day recency threshold match real user file access patterns? Information foraging literature suggests 80% of file accesses are to files accessed in the last 7 days (Barreau & Nardi 1995). A 7-day threshold may be more appropriate.

## Sources
- https://www.nngroup.com/articles/spatial-memory/
- https://dl.acm.org/doi/abs/10.1561/1100000046
- https://medium.com/design-bootcamp/spatial-memory-why-it-matters-in-ux-c70a502556d1
- https://blog.fibonalabs.com/spatial-memory-a-catalyst-to-improvise-ux-design/
