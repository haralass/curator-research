# Siblings Preview in Approval UI
**File:** 73 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
Showing the 3–5 existing files already in a proposed destination folder at the moment of approval decision significantly improves decision quality and reduces regret-driven undos. This is a well-supported UX principle (visible context at decision point) with applied analogues in document approval workflows and context-aware systems. Curator should implement this as a "destination preview strip" in the approval sheet.

## Key Findings
- Context-aware UX research: "showing relevant information right where the choice is being made" measurably reduces cognitive load and error rate
- Salesforce Spring '26 Screen Flow File Preview: in-context file preview in approval flows explicitly reduces download-and-open friction — validates the pattern for production use
- Approvals literature (Spend Matters 2026): contextual information in approval flows "reduces back-and-forth and allows approvers to spend less time reconstructing context"
- Decision fatigue research: contextual interfaces that reduce ambiguous choices (by providing ground truth context) reduce decision fatigue in multi-item approval workflows
- No direct academic study on "sibling file preview in move approval" was found — this is a design pattern application of established principles, not a research gap

## Relevant Papers / Prior Art
- UX Tigers "Think-Time UX: Design to Support Cognitive Latency" — cognitive load framing
- Spend Matters "Approvals and Controls: Scaling Governance Without Slowing Execution" (March 2026)
- Salesforce Screen Flow File Preview Spring '26 — product analogue
- Context-aware MFA (AuthSignal) — decision-quality through contextual information framing

## Applicability to Curator
High and low-effort to implement.

SwiftUI implementation sketch:
```swift
// In approval sheet
HStack {
    ForEach(destinationFolder.recentFiles.prefix(5)) { file in
        FileIconView(file: file)
            .help(file.name)
    }
}
.frame(height: 44)
```

Sort siblings by: (1) semantic similarity to incoming file first, then (2) recency. If destination folder is empty (new folder suggestion): show folder type badge + "New folder" label instead. Semantic-similarity sorting makes it immediately clear WHY Curator chose this destination.

## Open Questions
- Optimal count: 3 is likely optimal (Miller's law); 5 is the upper bound before cognitive overload
- Should siblings be sorted by recency or semantic similarity to the incoming file?
- What if the destination folder is empty (new folder suggestion)? Show folder name + type badge instead.

## Sources
- https://www.spendmatters.com/2026/03/approvals-and-controls-scaling-governance/
- Salesforce Spring '26 release notes (Screen Flow file preview)
- Nielsen Norman Group — contextual information in decision interfaces
