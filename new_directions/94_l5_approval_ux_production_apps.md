# L5 Automation Approval UX in Production Apps

**File:** 94 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
"L5 automation" (high-autonomy, low-friction action execution) in file organization apps is best exemplified by Hazel (Noodlesoft) and DEVONthink's AI classification. Both use rule-based or ML-based automation with optional review queues. The key finding from HCI research on mixed-initiative systems is that review fatigue scales with action volume, not action type — users disengage from approval dialogs when shown > 5–7 decisions per interaction session. Curator's L5 design must implement a batch approval queue with a "review later" option, not a per-file modal dialog.

## Key Findings
- **Hazel:** Applies user-defined rules to watched folders; executes actions (move, rename, tag, delete) without per-file approval if the rule confidence is high. Users define approval via rule creation (pre-approval), not action-time confirmation. Actions that can't be undone (delete) optionally prompt. This is the L4 model (pre-approved rules, not AI-inferred).
- **DEVONthink "Classify" AI:** Suggests destination folder for each new document; user can accept or override. DEVONthink shows the top-3 suggestions with a confidence bar. Single-click approval. No batch review queue — each document requires individual decision. User research from DEVONthink forums indicates review fatigue for collections > 20 new items/session.
- **Spark Mail Smart Inbox:** AI-sorts emails into categories; user can move items out. Implicitly approved (email appears in category without confirmation); user corrects errors via drag. This is the L5 model: act first, correct later.
- **HCI research on mixed-initiative systems:** 
  - Cranshaw et al., "Calista," CHI 2017: mixed-initiative scheduling assistant. Key finding: users accepted 78% of suggestions without review when explanation was shown; 45% without explanation. Explanation doubles acceptance.
  - Amershi et al., "Power to the People," *IEEE Intelligent Systems* 2014: guidelines for human-in-the-loop ML. Rule: "The cost of incorrect automation must be less than the cost of correct manual action." For file moves (easily reversible), L5 is appropriate.
  - Lubars & Tan, "Ask Me, But Don't Ask Me Often," CHI 2019: when ML systems ask for confirmation too frequently (> 5× per session), users switch to blanket-approve behavior, defeating the purpose.
- **Review fatigue threshold:** 5–7 confirmations per interaction session before users disengage or blanket-approve (Lubars & Tan 2019). Curator must batch decisions into a single review pane showing N pending moves, not N sequential dialogs.
- **Undo as the primary safety mechanism:** For file moves (reversible), the correct pattern is L5 (move automatically) + Undo (1-click revert) + notification ("Moved 12 files to Tax 2024 — Undo"). This is the Spark Mail / Gmail model. Confirmation dialogs are for irreversible actions only.

## Relevant Papers / Prior Art
- Lubars B., Tan C., "Ask Me, But Don't Ask Me Often: Appropriate Interruptions for Mixed-Initiative Systems," *CHI* 2019. DOI: 10.1145/3290605.3300511. — Quantifies review fatigue; 5–7 confirmation limit.
- Amershi S. et al., "Power to the People: The Role of Humans in Interactive Machine Learning," *IEEE Intelligent Systems* 29(4):33–40, 2014. — Mixed-initiative design guidelines.
- Cranshaw J. et al., "Calista: Toward a Conversational Agent for Life-Task Management," *IUI* 2017. — Explanation effect on acceptance rate.
- Mackay W.E., "Triggers and Barriers to Customizing Software," *CHI* 1991. — Early study; users avoid setting up automation even when they would benefit from it.

## Applicability to Curator

**L5 approval UX design for Curator:**

```
MenuBarExtra notification: "Organized 12 files → 3 folders  [Review] [Undo]"
```

- **Default mode (L5):** Files are moved automatically based on cluster assignment when confidence (conformal prediction coverage) ≥ threshold. 
- **Review queue:** Clicking "Review" opens a panel showing all automated moves since last review, grouped by destination folder. User can un-check individual moves before confirming.
- **Undo:** Single "Undo last batch" button reverts all moves in the last automation run. Implemented as a transaction in GRDB (store previous_path for each moved file).
- **Low-confidence fallback:** When conformal prediction set size > 1 (multiple candidate folders), the file is held in a "Needs Review" queue and not moved automatically. This prevents moving ambiguous files while still automating high-confidence ones.

```swift
// Swift: MenuBarExtra notification with undo
struct AutomationResult {
    let movedFiles: [(fileID: Int, fromPath: String, toPath: String)]
    let heldFiles: [(fileID: Int, reason: String)]  // low confidence
    
    var canUndo: Bool { !movedFiles.isEmpty }
}

func showAutomationNotification(_ result: AutomationResult) {
    let msg = "Organized \(result.movedFiles.count) files"
    // Show in MenuBarExtra popover, not NSAlert (no modality)
    // "Review" → opens review panel
    // "Undo" → calls undoLastBatch()
}
```

**Batch size limit:** Run automation at most once per session (session boundary detection via FSEvents entropy — see File 101). Never interrupt active work sessions with automation.

## Open Questions
- Should Curator offer a "supervised mode" (require approval for all moves) vs. "autonomous mode" (L5) as a user setting? Research suggests supervised mode leads to faster disengagement; default should be autonomous with prominent Undo.

## Sources
- https://podcasts.apple.com/mm/podcast/file-automation-with-hazel-and-devonthink/id458066753?i=1000680465360
- https://rmschneider.wordpress.com/2020/12/04/devonthink-and-hazel/
- https://dl.acm.org/doi/10.1145/3290605.3300511
