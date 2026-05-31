# 44. Safe Automation Policy

**Curator Research — IUI 2027 Track**
**Date:** 2026-05-31
**Status:** Draft

---

## Abstract

A local AI file organizer that asks for user approval before every action is safe but unusable. One that acts without restraint is efficient but untrustworthy. Between these extremes lies a design space that has received no formal treatment in the personal information management (PIM) literature: *safe automation policy* for AI file systems. This paper proposes a formal policy framework that partitions file operations into three tiers — AUTO, REVIEW, and NEVER-AUTO — based on action risk, confidence conditions, and reversibility. We define a formal action taxonomy, derive tier-assignment rules, describe a trust calibration mechanism that automatically promotes actions from REVIEW to AUTO as users demonstrate consistent approval patterns, and position undo as the universal safety net that makes the entire framework viable. We survey prior art from robot autonomy, human-computer interaction, and corrigible AI research. The result is the first formal automation policy framework for local AI file organization.

---

## 1. The Problem: Too Passive or Too Aggressive

The design of an AI file organizer confronts a fundamental tension between two failure modes.

**Too passive**: The system presents every candidate action as a suggestion requiring explicit approval. The user must review fifty file moves before leaving for class. Review fatigue sets in; users begin rubber-stamping approvals without reading them, which defeats the purpose of the review mechanism (Cummings, 2004). Alternatively, users stop engaging with the system altogether, reducing it to an ignored background process.

**Too aggressive**: The system acts autonomously, moving files without approval. The user cannot find a file because Curator moved it to a location the user did not anticipate. Trust collapses after the first unexpected relocation. Unlike a file deleted by accident (detectable via absence), a file moved to the wrong location is hard to discover: the user simply assumes the file is missing or was never saved. The asymmetric cost of this error type — high harm, low detectability — argues strongly against unconstrained automation.

The resolution requires a formal policy that specifies exactly which actions may be taken autonomously, under what conditions, and with what safeguards. Such a policy must be grounded in the risk level of each action type, the confidence of the AI's assessment, and the availability of a recovery mechanism. No existing PIM system or research paper, to our knowledge, provides such a framework.

---

## 2. Action Taxonomy

We enumerate the atomic actions a file organizer can take, ordered by risk level:

| ID | Action | Risk | Reversible? |
|---|---|---|---|
| A1 | Read-only analysis (index, embed, extract) | None | N/A |
| A2 | TextTwin / sidecar generation | Negligible | Yes (delete sidecar) |
| A3 | Exact duplicate flagging | Negligible | Yes (unflag) |
| A4 | Rename suggestion (in-place) | Low | Yes (undo rename) |
| A5 | Move suggestion (to different folder) | Medium | Yes (undo move) |
| A6 | Cluster/folder assignment | Medium | Yes (undo move) |
| A7 | Archive suggestion (compress + move) | Medium-High | Yes (extract) |
| A8 | Sensitivity flag | Low | Yes (unflag) |
| A9 | Duplicate deletion suggestion | High | Partial (Trash) |
| A10 | Permanent deletion | Very High | No (bypasses Trash) |
| A11 | Sensitive file processing | Context-dependent | Varies |

Actions A1–A3 have no risk because they are purely additive or read-only. Actions A4–A6 have low-to-medium risk because they can be undone instantaneously. Actions A7–A9 require more careful treatment. Action A10 (permanent deletion) must never be autonomous under any conditions.

---

## 3. Three-Tier Policy Framework

### Tier 1: AUTO (No Approval Required)

Actions assigned to AUTO are executed immediately and silently logged. The undo log is always updated. The criteria for AUTO assignment are:

1. The action is reversible with a single undo command.
2. The action has no potential for data loss.
3. Either the action is read-only, or the AI's confidence exceeds a defined threshold AND the action matches an established pattern (defined below).

Under these criteria, the following actions are unconditionally AUTO:

- A1: Read-only analysis (always)
- A2: TextTwin generation (always; sidecar is always removable)
- A3: Exact duplicate flagging (always; flagging is not deletion)
- A8: Sensitivity flagging (always; flagging is not action)

Actions A4–A6 are conditionally AUTO when:
- `confidence ≥ θ_auto` (default: 0.92)
- The destination folder already exists and contains files of the same role/topic
- The same or analogous action has been approved by the user ≥ 3 times with no undo

### Tier 2: REVIEW (Manifest Required)

REVIEW actions are batched into a manifest — a structured list of proposed changes — presented to the user for bulk approval. The manifest shows: source path, proposed action, destination, confidence score, and the signals that drove the decision. The user can approve all, approve selected, or reject individual entries. The system never proceeds until the user acts on the manifest.

REVIEW is the default tier for A4–A7 when confidence conditions for AUTO are not met. It is also the mandatory tier for any action where the user has previously undone the same action type (a strong signal that the user disagrees with the system's judgment in that domain).

### Tier 3: NEVER-AUTO (Always Explicit)

Certain actions must require explicit, separate user confirmation regardless of confidence, track record, or user preference settings:

- A10: Permanent deletion (bypasses Trash, unrecoverable)
- Any processing of files in user-designated sensitive folders (medical, financial, legal)
- Any action that would modify a file's content (as opposed to its location or name)

The NEVER-AUTO designation is not overridable by user preference settings. It is a hard constraint embedded in Curator's policy engine.

---

## 4. Confidence-Conditional Automation

The boundary between REVIEW and AUTO for medium-risk actions is governed by a confidence threshold with additional guard conditions. Formally, an action `a` on file `f` is promoted to AUTO if and only if:

```
confidence(a, f) ≥ θ_auto
∧ destination_exists(a, f)
∧ pattern_established(a, f)
∧ ¬recently_undone(action_type(a))
```

Where:
- `confidence(a, f)` is the combined confidence of the role, topic, and routing classifiers for this action
- `destination_exists(a, f)` means the target folder is already present in the user's filesystem (not a newly proposed folder)
- `pattern_established(a, f)` means the user has approved ≥ k analogous actions without undo (k = 3 by default)
- `recently_undone` means the user has undone this action type within the last 7 days

The threshold `θ_auto` is initialized conservatively (0.92) and can be adjusted by the user or relaxed automatically as the trust calibration model accumulates evidence (see §7).

---

## 5. Prior Art

### 5.1 Levels of Automation in Human Factors

Sheridan & Verplank (1978) introduced the first formal taxonomy of automation levels in supervisory control systems, later refined by Sheridan (1992) into a ten-level scale ranging from "the computer offers no assistance" (level 1) to "the computer acts fully autonomously and informs the human only if it decides to" (level 10). This framework has been applied extensively in aviation, process control, and military systems. Parasuraman et al. (2000) extended the taxonomy to distinguish automation of information acquisition, information analysis, decision selection, and action implementation — a distinction directly applicable to file organization, where Curator performs all four functions.

The PIM literature has not applied Sheridan's framework to file organization. Our three-tier policy (AUTO / REVIEW / NEVER-AUTO) corresponds approximately to levels 4–6 of Sheridan's scale for REVIEW actions and levels 7–8 for AUTO actions. NEVER-AUTO enforces a ceiling that prevents the system from reaching levels 9–10.

### 5.2 Corrigibility and Safe AI

Soares et al. (2015) introduced *corrigibility* as a property of AI systems that remain amenable to correction, shutdown, or policy modification by their principals. A corrigible file organizer never takes actions that would prevent the user from reversing its decisions. The undo log is the mechanism by which Curator implements corrigibility: every AUTO action is recorded in sufficient detail to be reversed by the user at any time, without requiring AI involvement in the undo operation.

Russell (2019) argues that the uncertainty an AI system maintains about human preferences is a sufficient condition for safe behavior: an uncertain system will defer to humans rather than act unilaterally. Curator's confidence threshold embeds this principle: below-threshold confidence triggers REVIEW rather than autonomous action.

### 5.3 Trust in Automation

Lee & See (2004) provide a comprehensive review of trust in automation, defining trust as "the attitude that an agent will help achieve an individual's goals in a situation characterized by uncertainty and vulnerability." They identify three factors that determine trust: ability (does the system perform well?), benevolence (does the system act in the user's interest?), and integrity (does the system behave consistently with stated principles?). The safe automation policy framework directly supports all three: confidence thresholds support ability, the NEVER-AUTO tier supports benevolence, and formal policy documentation supports integrity.

Cummings (2004) demonstrates that over-automation in safety-critical systems leads to *automation bias* — users over-trust automated recommendations, even when those recommendations are visibly wrong. Our REVIEW tier is designed to mitigate this: the manifest format forces users to read each proposed action rather than click a single approval button.

---

## 6. Undo as Universal Safety Net

The correctness of the AUTO tier depends entirely on the undo log being complete, durable, and trustworthy. We specify the following requirements for Curator's undo log:

1. **Completeness**: Every AUTO action generates a undo record before the action executes. If the undo record cannot be written (disk full, permissions error), the action does not execute.
2. **Atomicity**: For multi-step actions (e.g., rename + move), the undo record captures the entire operation. Undo restores the pre-action state in a single user command.
3. **Durability**: The undo log is persisted to disk and survives process crashes. A SQLite database with WAL mode provides the required durability guarantees.
4. **Accessibility**: The user can access the undo log at any time via Curator's history panel, without needing to know the name or location of the affected file.
5. **Time-bounded retention**: Undo records are retained for 90 days by default. After 90 days, undo may not be possible (e.g., if the file has since been modified or the destination folder deleted), and the record is archived rather than deleted.

The undo guarantee is surfaced to users in plain language during onboarding: "Curator will never make a change that cannot be reversed. Every action is logged and can be undone within 90 days."

---

## 7. Trust Calibration Model

Trust calibration is the mechanism by which Curator automatically adapts the boundary between REVIEW and AUTO as evidence accumulates about a user's preferences. The model operates as follows:

**Initialization**: All A4–A6 actions are assigned to REVIEW. The system collects approval/rejection/undo data.

**Promotion**: When the user approves k consecutive instances of the same action type with no undo (k = 3, configurable), and the average confidence of those actions exceeds θ_review (0.80), the action type is promoted to AUTO for the next 30 days.

**Demotion**: If the user undoes any AUTO action, the action type is immediately demoted to REVIEW for a 7-day cooling-off period. A second undo within 14 days extends the demotion indefinitely until the user manually re-enables AUTO for that action type.

**Per-folder granularity**: Trust is tracked at the (action type × folder domain) level. Approving moves into `EPL342/Lectures/` does not automatically enable AUTO moves into `Work/Contracts/`. Different folder domains may have different trust levels simultaneously.

**Transparency**: The current trust state is always visible in Curator's settings panel. Users can manually override any trust level at any time.

---

## 8. Research Contribution

1. **Action taxonomy**: The first formal enumeration of atomic file organizer actions with associated risk levels and reversibility properties.
2. **Three-tier policy framework**: A principled AUTO / REVIEW / NEVER-AUTO classification with formal promotion conditions.
3. **Trust calibration model**: A formal model of how trust accumulates and decays based on user behavior, with per-domain granularity.
4. **Undo specification**: A formal requirements specification for the undo log that enables the AUTO tier's safety guarantee.
5. **User study**: A between-subjects evaluation comparing user satisfaction, error rate, and trust scores across three policy configurations: always-REVIEW, always-AUTO, and the adaptive policy framework.

---

## References

Cummings, M. L. (2004). Automation bias in intelligent time critical decision support systems. In *AIAA 1st Intelligent Systems Technical Conference* (p. 6313). AIAA. https://doi.org/10.2514/6.2004-6313

Lee, J. D., & See, K. A. (2004). Trust in automation: Designing for appropriate reliance. *Human Factors*, 46(1), 50–80. https://doi.org/10.1518/hfes.46.1.50_30392

Parasuraman, R., Sheridan, T. B., & Wickens, C. D. (2000). A model for types and levels of human interaction with automation. *IEEE Transactions on Systems, Man, and Cybernetics — Part A*, 30(3), 286–297. https://doi.org/10.1109/3468.844354

Russell, S. (2019). *Human compatible: Artificial intelligence and the problem of control*. Viking.

Sheridan, T. B. (1992). *Telerobotics, automation, and human supervisory control*. MIT Press.

Sheridan, T. B., & Verplank, W. L. (1978). *Human and computer control of undersea teleoperators* (Technical Report). MIT Man-Machine Systems Laboratory.

Soares, N., Fallenstein, B., Yudkowsky, E., & Armstrong, S. (2015). Corrigibility. In *Workshops at the Twenty-Ninth AAAI Conference on Artificial Intelligence*. AAAI. https://intelligence.org/files/Corrigibility.pdf

Jones, W., & Teevan, J. (Eds.). (2007). *Personal information management*. University of Washington Press.

Bergman, O., Beyth-Marom, R., & Nachmias, R. (2008). The user-subjective approach to personal information management systems design. *Journal of the American Society for Information Science and Technology*, 59(2), 235–246. https://doi.org/10.1002/asi.20738

Hadfield-Menell, D., Milli, S., Abbeel, P., Russell, S., & Dragan, A. (2016). Inverse reward design. In *Advances in Neural Information Processing Systems* (Vol. 29). Curran Associates. https://proceedings.neurips.cc/paper/2016/hash/f6e07f07e0a8370ceda78b4f7c4be91c-Abstract.html

Amershi, S., Weld, D., Vorvoreanu, M., Fourney, A., Nushi, B., Collisson, P., Suh, J., Iqbal, S., Bennett, P. N., Inkpen, K., Teevan, J., Kikin-Gil, R., & Horvitz, E. (2019). Guidelines for human-AI interaction. In *Proceedings of the 2019 CHI Conference on Human Factors in Computing Systems* (pp. 1–13). ACM. https://doi.org/10.1145/3290605.3300233
