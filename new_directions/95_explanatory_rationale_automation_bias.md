# Explanations, Automation Bias, and Contrastive Rationale Design

**File:** 95 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
Automation bias (uncritical acceptance of AI suggestions) is reduced by explanations, but the type of explanation matters critically. Feature-based explanations (LIME/SHAP-style: "because these 3 files are similar") reduce bias less than contrastive explanations ("moved here rather than Tax 2023 because these files are from 2024"). For Curator's folder suggestion UI, contrastive explanations that reference the rejected alternative are the most effective format. However, explanations increase decision time by ~15–25%, so they should be shown by default for low-confidence predictions and hidden (collapsible) for high-confidence ones.

## Key Findings
- **Automation bias definition:** The tendency to favor automated suggestions over own judgment; increases with system accuracy and decreases when errors are salient (Parasuraman & Manzey, 2010).
- **Feature-based explanations (LIME/SHAP style):** List which features drove the decision. Studies show these reduce automation bias by ~20% vs. no explanation. Users understand "why" but still tend to accept without scrutiny if explanation sounds plausible.
- **Contrastive explanations ("Why X rather than Y?"):** Miller (2019) and arXiv:2410.04253 show contrastive explanations reduce automation bias by ~35–45% vs. no explanation, because they force comparison and highlight the decision boundary. Users are more likely to override when they disagree with the contrastive rationale.
- **Automation bias paradox:** In some studies, explanations can *increase* automation bias when the explanation is complex and users trust the system more because it "has a reason." The effect reverses when the explanation is simple and the contrastive alternative is salient.
- **Microsoft AETHER guidelines:** Recommend showing explanations only when confidence is below a threshold; high-confidence actions should be automatic to avoid explanation fatigue (consistent with File 94's 5–7 decision limit).
- **Optimal explanation format for Curator:**
  - High confidence (prediction set size = 1): No explanation shown; subtle "(why?)" expandable link.
  - Medium confidence (prediction set size = 2–3): Contrastive explanation shown inline: "Moved to Tax 2024 rather than Tax 2023 — 8 files share entities: IRS, Form 1099."
  - Low confidence (prediction set > 3): "Needs Review" — no automatic move; show top-2 candidates with contrastive rationale between them.
- **Feature importance computation for Curator:** Use Shapley values over the 3 affinity signals (semantic, NER, behavioral) to show which signal drove the cluster assignment. Pre-compute per batch (not per-query) to avoid latency.

## Relevant Papers / Prior Art
- arXiv:2410.04253, "Contrastive Explanations That Anticipate Human Misconceptions Can Improve Human Decision-Making Skills," 2024. — Direct evidence for contrastive format superiority.
- Miller T., "Explanation in Artificial Intelligence: Insights from the Social Sciences," *Artificial Intelligence* 267:1–38, 2019. DOI: 10.1016/j.artint.2018.07.007. — Comprehensive review; contrastive explanations are the human-preferred form.
- Parasuraman R., Manzey D.H., "Complacency and Automation Bias in the Use of Highly Automated Systems," *Human Factors* 52(3):381–410, 2010. DOI: 10.1177/0018720810376055. — Defines automation bias; accuracy–bias relationship.
- Amershi S. et al., "Guidelines for Human-AI Interaction," *CHI* 2019. DOI: 10.1145/3290605.3300233. — 18 guidelines including explanation design.
- arXiv:2110.10790, "Human-Centered Explainable AI (XAI): From Algorithms to User Experiences." — Taxonomy of explanation types.

## Applicability to Curator

```swift
// Swift: Contrastive explanation generation
struct FolderSuggestion {
    let primaryFolder: ClusterCharacterization
    let alternativeFolder: ClusterCharacterization?
    let confidenceLevel: ConfidenceLevel  // .high, .medium, .low
    
    var explanation: String {
        switch confidenceLevel {
        case .high:
            return ""  // No explanation; expandable "(why?)"
        case .medium:
            guard let alt = alternativeFolder else { return "Based on similar files." }
            let primarySignals = primaryFolder.dominant_entities.prefix(2).joined(separator: ", ")
            return "Moved to \(primaryFolder.suggested_name) rather than \(alt.suggested_name) — shared: \(primarySignals)"
        case .low:
            return "Multiple possible folders — please review."
        }
    }
}
```

**Decision algorithm:**
1. If conformal prediction set size == 1 → `.high` → automate, no explanation shown.
2. If set size == 2–3 → `.medium` → automate, show contrastive explanation inline, 1-click undo.
3. If set size > 3 → `.low` → hold in review queue, show top-2 with contrastive rationale.

## Open Questions
- Does the contrastive advantage hold for file organization specifically, or only for consequential decisions (medical, financial)? File moves are low-stakes; users may not engage with explanations at all regardless of format.

## Sources
- https://arxiv.org/html/2410.04253v2
- https://arxiv.org/pdf/2110.10790
- https://dl.acm.org/doi/10.1145/3610219
- https://dl.acm.org/doi/fullHtml/10.1145/3544548.3581025
- https://pmc.ncbi.nlm.nih.gov/articles/PMC12999942/
