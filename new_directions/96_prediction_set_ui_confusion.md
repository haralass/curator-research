# Prediction Set UI: User Confusion, Decision Time, and Design Patterns

**File:** 96 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
Showing users a prediction set (multiple candidate folders, e.g., {Tax 2024, Finance, Documents}) rather than a single recommendation increases decision time by 15–40% and can decrease user confidence in the system, according to HCI studies on uncertainty communication. However, prediction sets reduce over-reliance on wrong AI suggestions. The recommended design for Curator is to show prediction sets only in the "Needs Review" queue (low-confidence cases), and to present them as a ranked short-list with the top candidate pre-selected, not as an unsorted set. Single-candidate suggestions (high confidence) should require no user decision.

## Key Findings
- **Decision time increase:** Showing uncertainty (prediction intervals, sets) forces analytical processing. Studies at CHI and IUI show decision time increases 15–40% when uncertainty is presented vs. a single recommendation. This effect is larger for non-expert users.
- **User confusion patterns:**
  - Users conflate prediction sets with system indecision/failure ("it doesn't know").
  - Users expect a single answer; multiple answers violate the conversational implicature of "give me the right answer."
  - The term "confidence interval" is widely misunderstood by non-statisticians (2024 study, arXiv:2408.12365).
- **The analytical slow-down is beneficial:** Uncertainty communication reduces over-reliance on wrong AI suggestions. Users who saw uncertainty indicators were more likely to override incorrect suggestions (Frontiers in Computer Science, 2025).
- **Apple/Google design patterns:**
  - iOS QuickType keyboard: shows top-3 suggestions as a horizontal strip, first one pre-selected. User taps to select; tapping away dismisses all. This is the correct model for Curator's multi-candidate case.
  - Gmail Smart Reply: shows 3 pre-generated replies; user picks one. No uncertainty indicator. Single-best is highlighted.
  - Apple Maps: single route by default; alternatives accessible via "More Routes" (hidden by default).
  - **Pattern:** Always show a single primary suggestion; reveal alternatives on demand.
- **Prediction set framing for files:** Instead of showing the set as {Tax 2024, Finance, Documents}, frame it as: "Best match: Tax 2024 (or: Finance, Documents)". The "or" structure signals the set is alternatives, not confusion.
- **Optimal set size to show:** 1 primary + at most 2 alternatives. Showing > 3 options causes choice paralysis (Hick's Law: decision time ∝ log₂(n options)).
- **Interaction design:** Pre-select the top-ranked candidate; user can accept (single click) or change selection (click alternative). Never require users to select from a completely unchosen set.

## Relevant Papers / Prior Art
- arXiv:2408.12365, "Enhancing Uncertainty Communication in Time Series Predictions," 2024. — Documents CI vs. credible interval confusion; recommendation for plain-language uncertainty labels.
- Greis M. et al., "Designing for Uncertainty in HCI: When Does Uncertainty Help?", *CHI* 2017, DOI: 10.1145/3027063.3027091. — "Uncertainty doubles/halves usefulness depending on user expertise and task type."
- Hullman J. et al., "Understanding Uncertainty: How Lay Decision-makers Perceive and Interpret Uncertainty in Human-AI Decision Making," *IUI* 2023, DOI: 10.1145/3581641.3584033.
- Frontiers in Computer Science: "Trusting AI: does uncertainty visualization affect decision-making?", 2025, DOI: 10.3389/fcomp.2025.1464348.
- Iyengar S., Lepper M., "When Choice is Demotivating," *J. Personality and Social Psychology* 79(6):995–1006, 2000. DOI: 10.1037/0022-3514.79.6.995. — Choice paralysis; > 6 options reduces selection rate.

## Applicability to Curator

```swift
// SwiftUI: Prediction set display in Review Queue
struct FolderCandidateView: View {
    let candidates: [ClusterCharacterization]  // sorted by confidence, max 3
    @State private var selected: ClusterCharacterization?

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            // Primary suggestion (pre-selected)
            FolderOptionRow(cluster: candidates[0], isSelected: true, isPrimary: true)
                .onTapGesture { selected = candidates[0] }

            // Alternatives (collapsed by default, expandable)
            if candidates.count > 1 {
                DisclosureGroup("Or: \(candidates[1...].map(\.suggested_name).joined(separator: ", "))") {
                    ForEach(candidates.dropFirst(), id: \.cluster_id) { candidate in
                        FolderOptionRow(cluster: candidate, isSelected: selected?.cluster_id == candidate.cluster_id)
                            .onTapGesture { selected = candidate }
                    }
                }
                .font(.caption)
                .foregroundColor(.secondary)
            }
        }
    }
}
```

**Language guidance:**
- DO: "Move to Tax 2024" (single confident action)  
- DO: "Best match: Tax 2024 — or Finance, Documents" (multi-candidate, primary pre-selected)
- DON'T: "Possible folders: [Tax 2024] [Finance] [Documents]" (unsorted set, no clear default)
- DON'T: "Confidence: 67%" (meaningless raw probability to non-statisticians)
- DO: "Tax 2024 matches 8 of your similar files" (count-based social proof)

## Open Questions
- Does the "primary pre-selected" pattern bias users toward accepting the top suggestion even when incorrect? This is the automation bias concern from File 95 — mitigated by contrastive explanation.

## Sources
- https://arxiv.org/pdf/2408.12365
- https://dl.acm.org/doi/10.1145/3027063.3027091
- https://dl.acm.org/doi/10.1145/3581641.3584033
- https://www.frontiersin.org/journals/computer-science/articles/10.3389/fcomp.2025.1464348/full
- https://www.researchgate.net/publication/316613602_Designing_for_Uncertainty_in_HCI_When_Does_Uncertainty_Help
