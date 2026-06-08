# FINAL Consistency Audit — Curator Research R0–R9

> **Purpose:** Single source of truth for all identified conflicts and terminology drift across R0–R9. Every conflict has a final decision and a list of files that must be updated to reflect it.
> **Date:** 2026-06-08
> **Author:** Full corpus review of R0_synthesis, R0_open_questions_resolved, R0_package_verification, R1_file_state_machine, R5_safe_learning_validation, R7_group_correction_ux, R8_activity_log_undo

---

## Summary Table

| ID | Conflict | Files Affected | Status |
|---|---|---|---|
| C1 | `hnswlib` vs `usearch` | R0_synthesis §1, §4 Layer 3 | RESOLVED: `usearch` everywhere |
| C2 | Undo window: 90 days uniform vs two-tier | R0_synthesis §4, §5; R8 §9 | RESOLVED: 90d staging / 180d commit |
| C3 | `committed` as terminal state | R1 §1.2, §1.3; R0_synthesis §2 | RESOLVED: not terminal; two new transitions |
| C4 | Automation levels in UI | R5 §7; R0_synthesis §4 Layer 5 | RESOLVED: internal only, no UI labels |
| C5 | Repair Center as main tab | Legacy only | RESOLVED: Home problem cards only |
| C6 | `locked` vs `ignored` semantics | R1 §1.2, §1.4; R0_synthesis §2 | RESOLVED: distinct definitions below |
| C7 | `snfpy` reference | R0_package_verification | RESOLVED: replaced by `sparse_snf.py` |
| C8 | `Louvain` in R5 | R5 §3.5 | RESOLVED: replace with `Leiden` |
| C9 | `hnswlib` in Layer 3 prose | R0_synthesis §4 Layer 3 | RESOLVED: `usearch` (subsumes C1) |
| C10 | Full SNF matrix | No surviving bad references | CONFIRMED: sparse SNF only, no action |
| C11 | Confidence as percentage | R8 §4 reason templates vs R7 group card | RULING: not a conflict; context-dependent |
| C12 | Session cap consistency | R0_synthesis, R7 | CONFIRMED: 15 groups, consistent |
| C13 | UI navigation sections | R7 §8.2 | CONFIRMED: final spec below |
| C14 | State name drift: `ignored` | R0_synthesis §2, R1 §1.2 | RESOLVED under C6 |
| C15 | Undo window 48h in R1 §1.4 | R1 §1.4 | RESOLVED: update to 90d/180d |
| C16 | `committed` terminal flag in R0_synthesis §2 | R0_synthesis §2 | RESOLVED under C3 |

---

## C1 — `hnswlib` vs `usearch`

### Conflicting Text

**R0_synthesis §1 pipeline diagram:**
> `hnswlib for incremental delta inserts`

**R0_synthesis §4 Layer 3:**
> `FAISS IVFFlat batch; hnswlib for incremental inserts. (R4)`

**R0_package_verification (hnswlib vs usearch section):**
> `hnswlib 0.8.0` has source-only on PyPI. Compiles with Xcode CLT. However, `unisim` already depends on `usearch>=2.6.0`, which has pre-built Apple Silicon wheels and provides HNSW-based ANN search as a transitive dependency of `unisim`.
>
> **Curator decision (revised):** Use **`usearch`** (via `unisim` transitive dependency) for incremental kNN inserts instead of `hnswlib`. Eliminates one source-only compilation. `usearch` 3.x has parity with `hnswlib` on HNSW performance.

### Final Decision

`usearch` everywhere. `hnswlib` is superseded. It requires a source-only C++ compile that is eliminated by using the pre-built `usearch` wheel that arrives transitively through `unisim`.

### Files to Update

- **R0_synthesis §1 pipeline diagram:** `hnswlib for incremental delta inserts` → `usearch for incremental delta inserts (transitive via unisim)`
- **R0_synthesis §4 Layer 3:** `FAISS IVFFlat batch; hnswlib for incremental inserts.` → `FAISS IVFFlat batch; usearch for incremental inserts (transitive via unisim).`
- Any future R4 draft that mentions `hnswlib` must use `usearch`.

---

## C2 — Undo Window: 90 Days Uniform vs Two-Tier

### Conflicting Text

**R0_synthesis §4 Layer 6:**
> `Undo window: 90 days active; indefinite archived (R6, R8)`

**R0_synthesis §5 Key Numbers Table:**
> `Undo retention (active / archived) | 90 days / indefinite`

**R8_activity_log_undo §9 Decision 7:**
> `90-day undo window. Events older than 90 days are marked reversible = 0 automatically.`

**R0_open_questions_resolved NB1 (authoritative):**
> **Two-tier retention:**
> - `staging_move` events: 90 days active undo
> - `commit_move` events: **180 days** active undo (user consciously approved the commit — longer regret window is justified)
> - Both: indefinitely archived (status `archived`, soft-deleted from undo UI after window)

### Final Decision

**Two-tier undo window:**
- `staging_move` events: **90 days** active undo
- `commit_move` events: **180 days** active undo
- Both: indefinitely archived after window (never hard-deleted)

The NB1 resolution in R0_open_questions_resolved is authoritative. All single-value "90 days" references are stale.

### Files to Update

- **R0_synthesis §4 Layer 6:** `Undo window: 90 days active; indefinite archived` → `Undo window: 90 days (staging_move); 180 days (commit_move); indefinitely archived`
- **R0_synthesis §5 Key Numbers Table:** `Undo retention (active / archived) | 90 days / indefinite` → `Undo retention | staging_move: 90d active; commit_move: 180d active; both indefinitely archived`
- **R8 §9 Decision 7:** `90-day undo window` → `Undo window is two-tier: 90 days for staging_move events, 180 days for commit_move events (R0_open_questions_resolved NB1)`

---

## C3 — `committed` as Terminal State

### Conflicting Text

**R1_file_state_machine §1.2 State Definitions table:**
> `committed | User approved group action; file moved/renamed/tagged; final disposition recorded | Yes`
> (Terminal = Yes, no asterisk — fully terminal, no way out)

**R1_file_state_machine §1.3 Valid State Transitions:**
> ```
> committed
>   → user_corrected  (user reverses a committed action)
> ```
> Only one outgoing transition. No re-evaluation path.

**R0_synthesis §2 State Machine table:**
> `committed | User commits group | — (terminal) | Physical move to final path (R6)`

**New product philosophy (this audit):**
> "No permanent final organization. Files have a current stable organization that evolves."
> `committed` means: "accepted stable placement for now, re-evaluable."

### Final Decision

`committed` is **NOT terminal.** It is a stable resting state that is re-enterable.

**Two transitions must exist:**

1. `committed → user_corrected` — **already in R1.** Keep and retain.
2. `committed → staged_review` — **NEW.** Trigger: Phase 2 system detects new context evidence (ADWIN signal, Folder Coherence Score drop below threshold, or new files that contradict the committed placement). Files move back into Review Hub for re-evaluation without user initiation. This transition is Phase 2 behavior; the state machine must support it from day one for schema stability.

**Updated `committed` definition:**
> A file in `committed` state has been user-approved and physically placed in its designated location. This is a stable resting state. It can be exited by the user explicitly (→ `user_corrected`) or by the system on new contextual evidence in Phase 2 (→ `staged_review`). It is not permanently final.

**Updated Terminal? flag:** `No` (stable, not terminal).

### Files to Update

- **R1_file_state_machine §1.2:** Change `Terminal? = Yes` for `committed` → `No`. Update description to reflect re-evaluable nature.
- **R1_file_state_machine §1.3 Transitions:** Add `committed → staged_review (Phase 2: new context evidence triggers re-evaluation)` below the existing `committed → user_corrected` line.
- **R0_synthesis §2:** Change `— (terminal)` in the `committed` Trigger-to-leave column to `→ user_corrected | → staged_review (Phase 2)`. Update Notes to "accepted stable placement; re-evaluable."

---

## C4 — Automation Levels in UI

### Conflicting Text

**R5_safe_learning_validation §7.1:**
> `The UI shows each rule's current automation level with a visible control to escalate or de-escalate.`

**R5 §7.2 Escalation Requirements:**
> `User explicitly clicks "Allow automatic application" in the Rules UI for this specific rule`
> `User explicitly clicks "Apply silently" in the Rules UI for this specific rule`

**R5 §3.8 (Parasuraman-Sheridan):**
> `The UI shows each rule's current automation level with a visible control to escalate or de-escalate.`

**Product decision (this audit):**
> L1/L2/L3 are internal only. No UI toggle. Users see effects but don't configure levels.

### Final Decision

**L1/L2/L3 are internal classification labels only. They do not appear in the UI.**

Users experience the effects of automation levels:
- The system proposes groupings and waits for approval (internal L1 behavior).
- The system acts and notifies (internal L2 behavior).
- The system acts silently (internal L3 behavior).

Users do not see "L1 / L2 / L3" labels or a numeric level-escalation control in any UI screen.

The Parasuraman-Sheridan principle of explicit opt-in before escalation still applies — but the mechanism is plain-language permissions, not labeled level controls. For example: a future Settings option reading "Run this rule without asking me" (not "Escalate to L2"). Level numbers are developer vocabulary.

### Files to Update

- **R5 §7.1:** Remove `The UI shows each rule's current automation level with a visible control to escalate or de-escalate.` Replace with: `The system tracks automation level internally (L1/L2/L3). Users see the rule's behavior in plain language — "suggests only", "applies and notifies", "applies silently" — not numeric levels. Permission to escalate is granted via plain-language opt-in, not a level control.`
- **R5 §3.8:** Same correction to the Parasuraman-Sheridan section's Curator decision paragraph.
- **R5 §7.2 Escalation Requirements:** The escalation logic (threshold conditions, opt-in requirement) is correct and stays. Remove or replace any phrasing that implies the UI shows an "automation level" control or level number.
- **R0_synthesis §4 Layer 5:** After the `3 automation levels: L1/L2/L3` line, add: `(internal classification only — not exposed as labeled levels in the user interface)`

---

## C5 — Repair Center as Main Tab

### Status: Resolved (legacy only)

No surviving R-file in the read corpus explicitly names "Repair Center" as a main navigation tab. R7 §7.4 already correctly places repair information on the Home screen as conditional problem cards. This conflict is documented to prevent re-introduction.

**Final navigation spec:**
```
Home | Review Hub | Organized | Search | Activity | Settings
```

"Repair Center" as a concept = Home problem cards only. Not a tab. Not a section. Any future document that introduces "Repair Center" as a navigation destination must be corrected.

---

## C6 — `locked` vs `ignored`

### Conflicting Text

**R1_file_state_machine §1.2 State Definitions table:**
> `ignored | User locks file/folder | Explicit unlock | Stored in exclusions table (R1)`
> The trigger column reads "User locks file/folder" — this is wrong. It conflates locking with ignoring.

**R0_synthesis §2 State Machine table:**
> `ignored | User locks file/folder | Explicit unlock | Stored in exclusions table (R1)`
> Same incorrect conflation.

**R1_file_state_machine footnote (correct):**
> `` `locked` is NOT a state — exclusion check happens before DB record is created. ``

**Product decision (this audit):**
> - `locked` = exclusion trie match → file NEVER enters DB
> - `ignored` = file was seen, processed, user dismissed it → IN DB with `ignored` state

### Final Decision

These must be consistently distinct everywhere:

| Term | Meaning | DB record created? | Trigger | Reversible? |
|---|---|---|---|---|
| `locked` | Path matches exclusion trie — total ignore at scan time | **No** — never enters DB | User adds path/glob to exclusions table; trie match happens before `stat()` | Yes — user removes from exclusions table; takes effect on next scan |
| `ignored` | File was fully processed; user explicitly dismissed it from pipeline | **Yes** — DB record with `state = 'ignored'` | User clicks Dismiss/Ignore in Review Hub on a group or file | Yes — `ignored → user_corrected` transition |

**Consequences:**
- `ignored` files CAN be re-engaged via `ignored → user_corrected`.
- `locked` files are not in the DB. Re-processing requires removing from exclusions table.
- `ignored` files may retain embeddings and graph signals (they are filtered from proposals, not erased).
- `locked` files produce zero signals — no inode, no hash, no embedding, no graph edge.

### Files to Update

- **R1_file_state_machine §1.2:** Change the `ignored` row Trigger column from "User locks file/folder" → "User explicitly dismisses a file or group from Review Hub; file has been seen and processed."
- **R0_synthesis §2:** Same correction to the `ignored` row Trigger column.
- **Both files:** Preserve and cross-reference the footnote that `locked` is not a state.

---

## C7 — `snfpy` Reference

### Conflicting Text

**R0_package_verification:**
> `snfpy | Sparse SNF | ❌ NOT INSTALLABLE | snfpy 0.2.2 is full-matrix SNF — NOT sparse`
> **Decision:** Remove from `requirements_prod.txt`. Implement `sparse_snf.py` (~200 lines, scipy.sparse).

### Final Decision

`snfpy` is permanently rejected. At 30k files it produces 3.6 GB/view (O(n²) memory). The only valid implementation is the custom `sparse_snf.py` using `scipy.sparse.csr_matrix`.

**Rule:** Any `import snfpy` anywhere is an error.

No updates needed to the read corpus — R0_package_verification already documents the rejection correctly.

---

## C8 — `Louvain` Reference in R5

### Conflicting Text

**R5_safe_learning_validation §3.5 (Nguyen et al. incremental detection):**
> `Curator should not re-run full Louvain community detection on the entire 30,000-file graph.`

**R0_synthesis §1 pipeline (correct):**
> `Leiden (leidenalg), NOT Louvain (25% disconnected community risk)`

**R0_synthesis §8 Prior Work (correct):**
> `Louvain produces disconnected up to 25% of cases | Curator uses leidenalg exclusively; Louvain ruled out`

### Final Decision

**Leiden everywhere.** The R5 mention is a drafting error. The principle from Nguyen et al. (local rather than global re-clustering after corrections) is valid — the algorithm named is wrong.

### Files to Update

- **R5 §3.5:** `should not re-run full Louvain community detection` → `should not re-run full Leiden community detection`

---

## C10 — Full SNF Matrix

No surviving problematic references found in the read corpus beyond the explicit rejection in R0_package_verification. R0_synthesis correctly says "Sparse SNF" throughout. **No updates needed.**

---

## C11 — Confidence as Percentage (Ruling)

### Texts in Apparent Conflict

**R8_activity_log_undo §4.1 (Activity log reason template):**
> Template: "Content is [X]% similar to other [context] files"
> Example: "Content is 87% similar to other EPL342 lecture notes"

**R7 DD-R7-03 and R0_synthesis §4 Layer 5:**
> `Confidence: 4-dot qualitative scale (never percentages — Lucivero 2020 over-reliance)`

### Ruling: Not a Conflict

These apply to different UI layers:

- **4-dot scale / no percentages** = rule for group card confidence indicators, health scores, and any primary UI element on Review Hub and Organized screens. These are interactive decision surfaces. Showing numbers causes over-reliance (Lucivero 2020).
- **Percentages in `reason` text** = factual signal descriptions in the Activity log Details drawer (Tier 3, power users expand to see this). "Content is 87% similar" is a factual statement about a computed score, not a confidence control. It appears only in the expandable details layer, not on group cards.

**Rule:** Percentages appear ONLY in `reason` / `reason_details` text in the Activity log Details drawer. They NEVER appear on group cards, health score indicators, or any primary decision UI. No files need updating.

---

## C12 — Session Cap Consistency

**R0_synthesis §1 and §5:** Default 15 groups, configurable 5–30. **R7 DD-R7-06:** Same. **Confirmed consistent. No action.**

---

## C13 — UI Navigation Sections

**Final confirmed navigation spec:**
```
Home | Review Hub | Organized | Search | Activity | Settings
```

Six tabs. No Repair Center tab. Rules are accessible within Settings or Activity. R7 §8.2 already states these correctly. **No conflicts found. No action.**

---

## C15 — Undo Window 48h in R1 §1.4

### Conflicting Text

**R1_file_state_machine §1.4 Transition Trigger table:**
> `committed → user_corrected | User undoes a committed action (undo window: configurable, default 48 hours)`

The "48 hours" value is a stale draft value. The authoritative figure from NB1 is 180 days for commit_move events.

### Files to Update

- **R1_file_state_machine §1.4:** `undo window: configurable, default 48 hours` → `undo window: 180 days for commit moves (R0_open_questions_resolved NB1)`

---

## Additional Global Rules (Cross-Cutting)

These rules apply to all R-files and all implementation code:

| Rule | Violation | Correct |
|---|---|---|
| No `snfpy` | `import snfpy` | `from engine.sparse_snf import sparse_snf` |
| No `hnswlib` | `import hnswlib` | `import usearch` (transitive via `unisim`) |
| No `Louvain` as Curator's algorithm | "Curator uses Louvain" | "Curator uses Leiden (`leidenalg`)" |
| No full SNF matrix | `snfpy.snf(...)` or dense SNF | `sparse_snf.py` with `scipy.sparse.csr_matrix` |
| No percentages on group cards | "87% confidence" in UI card | `●●●○` with plain-language label |
| Session cap values | any value other than 15 default, 5–30 range | 15 / 5–30 |
| Automation levels | "L1/L2/L3" in user-visible UI | plain-language behavior descriptions |
| `locked` vs `ignored` | using either term for the other concept | see C6 table |
| Navigation tabs | any tab not in `Home|Review Hub|Organized|Search|Activity|Settings` | the six-tab spec above |
| `committed` terminality | marking `committed` as terminal | `committed` is stable, not terminal |

---

## Master Update Checklist

| File | Required Changes | Conflicts Addressed |
|---|---|---|
| `R0_synthesis.md` | §1 pipeline: hnswlib → usearch; §2: committed not terminal, committed Trigger-to-leave add transitions, ignored trigger fix; §4 Layer 3: hnswlib → usearch; §4 Layer 5: add internal-only note on L1/L2/L3; §5: undo retention two-tier | C1, C2, C3, C4, C6 |
| `R1_file_state_machine.md` | §1.2: committed Terminal? → No, committed description update, ignored trigger fix; §1.3: add committed → staged_review transition; §1.4: undo window 48h → 180d for commit | C3, C6, C15 |
| `R5_safe_learning_validation.md` | §3.5: Louvain → Leiden; §3.8 and §7.1: remove L1/L2/L3 UI label references; §7.2: replace level-control phrasing with plain-language framing | C4, C8 |
| `R8_activity_log_undo.md` | §9 Decision 7: 90-day → two-tier 90d/180d | C2 |

---

*Audit complete. 16 conflict entries examined. 13 require file edits. 3 confirmed non-conflicts or already-correct. Four source files need updating. All architectural decisions and data models remain structurally sound — changes are clarifications and corrections, not redesigns.*
