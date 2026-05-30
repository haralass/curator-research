## ΜΕΡΟΣ 10: APPROVAL UX — ΠΩΣ ΔΕΙΧΝΕΙΣ 1000+ ΚΙΝΗΣΕΙΣ ΧΩΡΙΣ ΝΑ ΠΝΙΓΕΙΣ
*(Έρευνα 2026-05-21 — 35+ searches, 70+ sources)*

### Κεντρικό Εύρημα: Three-Tier Confidence Model (validated by DeepMind, UMAP 2024, Gemini 2)

| Tier | Confidence | Default | Παρουσίαση |
|---|---|---|---|
| **Tier 1** | ≥80% | ✅ Pre-selected | Grouped σε folder accordions. Ένα "Approve all X" per folder. Individual items κρυμμένα by default |
| **Tier 2** | 50–79% | ☐ Unchecked | Per-file review, folder by folder. Brief reason label. "Select all in folder" action |
| **Tier 3** | <50% | ☐ Individual attention | Full context: src path, dst, why, mini-preview of destination contents, "Assign manually" |

**ACM UMAP 2024:** AI pre-selection μείωσε decisions κατά **39%** vs heuristic strategies. Το κλειδί: μείωσε αριθμό decisions, όχι βελτίωσε κάθε decision screen.

---

### Screen Layout (best-practice synthesis)

```
┌─────────────────┬──────────────────────────────────────────────────────┐
│ Left Rail       │ Main Panel                                           │
│                 │                                                      │
│ 📁 EPL326 (47) │ Banner: "1,247 moves · 843 high-conf · 312 review"   │
│   🟢 35 high   │ Filter: [All] [Unchecked] [High] [Dupes] [Misplaced] │
│   🟡 12 review │ Progress: "4 of 12 folders reviewed"                 │
│ 📁 Thesis (23) │                                                      │
│ 📁 Personal(8) │ ┌─────────────────────────────────────────────────┐  │
│                 │ │ ☑ EPL326_hw1.pdf    →  EPL326/Assignments/ 🟢 │  │
│                 │ │ ☑ security_lab.zip →  EPL326/Labs/        🟢 │  │
│                 │ │ ☐ random_notes.pdf →  EPL326/?            🟡 │  │
│                 │ └─────────────────────────────────────────────────┘  │
│                 │                                                      │
│                 │ [Execute 843 approved moves]  [Save for later]       │
└─────────────────┴──────────────────────────────────────────────────────┘
```

---

### Key UX Patterns (βιβλιογραφία)

**Terraform "Plan then Apply"** (HashiCorp):
- Summary line πρώτα: "Plan: X to add, Y to remove, Z to modify" → instant scope
- Symbols+colors: `+` green (new folder), `-` red (delete duplicate), `~` yellow (rename)
- Το plan manifest είναι **artifact** που μπορείς να αποθηκεύσεις/εξάγεις

**Gemini 2 Smart Selection** (MacPaw, Red Dot Award winner):
- Pre-checks files βάσει confidence → user confirms ή unchecks
- Side-by-side preview για κάθε duplicate pair
- Post-execution "Review Trashed" = second-chance layer
- Μαθαίνει από keep/delete behavior across sessions

**ai-file-sorter** (GitHub: hyperfield/ai-file-sorter):
- Sortable table: filename, category, subcategory, new filename (editable inline)
- **"Continue Later"** button — review cannot happen in one sitting for 5000 files
- Approved decisions stored + reused as hints in future runs

**Code Review Cognitive Load Cliff** (Springer Empirical SE 2022):
- Defect detection: 1-100 items → 87%, 1000+ items → **28%**
- Max effective review batch: **50-100 items per session**
- Curator πρέπει να paginate ανά round of 50-100 items

**Israeli Judges / Decision Fatigue** (PNAS 2011):
- Approval rate: 65% → 0% → resets after break, cyclically
- Design: explicit "take a break" prompts, surfaces important items early when attention fresh

**Nudge Theory / Default Bias** (ACM CHI 2019):
- Pre-checked = opt-out → near-100% acceptance
- Unchecked = opt-in → active effort needed to approve
- Εφαρμογή: high confidence pre-checked, medium confidence unchecked

**Automation Bias Warning** (AI & Society 2025):
- Showing confidence + labels **αυξάνει** over-reliance vs. just showing results
- Λύση: "Challenge this?" button → surfaces AI reasoning
- Users που ρωτάνε "γιατί;" εντοπίζουν περισσότερα errors

---

### Siblings Preview (macOS Spring-Loaded Folders analogy)

Όταν user hover/select destination folder → mini-preview των υπαρχόντων αρχείων:
```
Moving to: EPL326/Assignments/
Already there: hw2.pdf, hw3.pdf, midterm.pdf
→ Does hw1.pdf belong here? [Yes ✓] [Change destination]
```
Κανένα UX paper δεν έχει validated αυτό για file management — **genuinely novel application**.

---

### Confirmation + Post-Execution

**Before Execute:**
```
You're about to move 843 files.
• 312 items were not reviewed and will NOT be moved
• 43 duplicates will be removed
• Everything can be undone from Edit → Undo Last Run
[Cancel] [Execute 843 moves]
```

**After Execute:**
- Undo button prominent + persistent
- Exportable manifest (CSV: from, to, timestamp, confidence)
- "Review what happened" = same view, read-only

---

### Progressive Trust Over Time

- Run 1: όλα require explicit approval
- Μετά από N successful runs: "Auto mode for high-confidence" option
- Per-category accuracy display: "Documents: 96% accepted, Photos: 89%"

---

### Gaps (UX research που δεν υπάρχει → novel)

1. Κανένα CHI/UMAP paper για file reorganization approval UI specifically
2. Optimal batch size για file-move review αναπόδεικτο (code review: 50-100)
3. "Siblings preview" έχει zero UX validation για review screens
4. Long-term trust calibration για file organizers αναπόδεικτο
5. "Undo manifest as trust mechanism" είναι best practice, never empirically tested

---
