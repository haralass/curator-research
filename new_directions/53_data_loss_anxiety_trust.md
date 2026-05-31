# 53 — Data Loss Anxiety & Trust in AI File Organization
_Research date: 2026-05-31_

---

## Why This Matters for Curator

Curator's core value proposition — automatically moving and renaming files — is also its highest-friction action. Users are not afraid of AI in the abstract; they are afraid of irreversibility. A misclassified email or a wrong music recommendation is annoying and correctable. A moved file that cannot be found again may represent years of photos, tax records, or creative work. This asymmetry between the cost of a mistake and the benefit of correct automation shapes everything about how users will approach Curator.

The literature on human-automation interaction makes this concrete. Lee and See (2004) define trust as "the attitude that an agent will help achieve an individual's goals in a situation characterized by uncertainty and vulnerability." The key word is vulnerability — users surrendering file control to an AI are genuinely vulnerable. Hoff and Bashir (2015) add that trust is not a single dial but a layered system: dispositional trust (how much you trust technology in general), situational trust (how risky does this particular action feel right now?), and learned trust (what has this specific system done for you over time). Curator starts with low situational trust and zero learned trust. It must earn its way up this stack.

The stakes are not abstract. Studies on digital attachment show that nearly 70% of users report elevated stress or sadness after significant data loss — digital files are not neutral containers but "active repositories of personal history and emotional significance." Photos, creative projects, and financial documents carry the same psychological weight as physical possessions. Any design for Curator must treat file movement not as a technical operation but as an act that touches emotionally charged territory. The implication: trust must be earned incrementally, every destructive action must be reversible or clearly signposted as permanent, and the system must give users full visibility into everything it has done.

---

## Trust in Automation: Foundational Literature

**Lee & See (2004) — The Seminal Framework**

Lee, J. D., & See, K. A. (2004). Trust in Automation: Designing for Appropriate Reliance. *Human Factors*, 46(1), 50–80. DOI: 10.1518/hfes.46.1.50_30392

Key findings:
- Trust governs reliance when complexity makes complete understanding impractical — precisely the condition when Curator moves files autonomously.
- The design goal is *appropriate reliance*: both over-trust (blindly accepting AI moves) and under-trust (ignoring the system entirely) are failure modes.
- Automation characteristics affect trust through three processes: **analytic** (explicit reasoning about past performance), **analogical** (pattern matching to similar systems), and **affective** (felt sense of comfort or discomfort). Curator must address all three.

**Hoff & Bashir (2015) — Three-Layer Trust Model**

Hoff, K. A., & Bashir, M. (2015). Trust in Automation: Integrating Empirical Evidence on Factors That Influence Trust. *Human Factors*, 57(3), 407–434. DOI: 10.1177/0018720814547570

Meta-analysis of empirical automation trust research (2002–2013). Three layers:
1. **Dispositional trust** — stable personality trait; some users will never trust AI organizers no matter what.
2. **Situational trust** — fluctuates based on task risk, prior experience, anxiety state. Moving a file from Downloads is lower situational risk than reorganizing a decade of photos.
3. **Learned trust** — accumulates through repeated interactions. Early positive experiences with low-stakes actions build learned trust capital for later higher-stakes autonomous operations.

Critical design implication: **Curator should sequence interactions from low situational risk to high.** Start by organizing throwaway files; earn learned trust before touching Photos.

**Muir (1994) — Trust Calibration and Competence Signals**

Muir, B. M. (1994). Trust in Automation: Part I. *Ergonomics*, 37(11), 1905–1922.

Trust is primarily driven by perceived competence. Critically: **trust is reduced by any sign of incompetence, even if the mistake has no net effect on performance**. One bad file move will carry disproportionate weight. Curator must err heavily toward caution and must not make visible errors early in the user relationship.

**Dzindolet et al. (2003) — The Role of Blame and Explanation**

Dzindolet, M. T., et al. (2003). The role of trust in automation reliance. *International Journal of Human-Computer Studies*, 58(6), 697–718. DOI: 10.1016/S1071-5819(03)00038-7

After an automation error, users distrusted even reliable systems — **unless an explanation was provided for why the system might err**. This is directly applicable to Curator: when the AI moves a file to an unexpected location, it must explain its reasoning. Unexplained mistakes are trust catastrophes. Explained mistakes are recoverable.

**Parasuraman, Sheridan & Wickens (2000) — Levels of Automation**

Parasuraman, R., Sheridan, T. B., & Wickens, C. D. (2000). A model for types and levels of human interaction with automation. *IEEE Transactions on Systems, Man, and Cybernetics*, 30(3), 286–297. DOI: 10.1109/3468.844354

Maps automation across four functional stages at 10 levels from "human does everything" to "computer acts fully autonomously." Curator should operate at **levels 2–4** (AI suggests, humans decide) before earning the right to levels 7–9. Starting at level 10 with file moves is a trust design mistake.

---

## AI-Specific Trust: TAM and Beyond

**Davis (1989) — Technology Acceptance Model**

Davis, F. D. (1989). Perceived usefulness, perceived ease of use, and user acceptance of information technology. *MIS Quarterly*, 13(3), 319–340.

TAM extensions add **perceived risk** and **initial trust** as variables. Research on AI with personal data shows that trust must be treated as a co-equal construct alongside usefulness. Users won't adopt a useful system they don't trust.

**Venkatesh et al. (2003) — UTAUT**

Venkatesh, V., et al. (2003). User acceptance of information technology: Toward a unified view. *MIS Quarterly*, 27(3), 425–478.

"Facilitating conditions" is crucial: users need to know that if things go wrong, they can recover. The absence of facilitating conditions for recovery is a direct brake on adoption.

**Explainability and Trust — Sullivan & Weger (2025)**

Sullivan, V., & Weger, K. (2025). Transparency and Explainability in AI-Assisted Decision Making. *Human Factors*. DOI: 10.1177/10711813251369473

Study with 216 participants: higher transparency levels improved trust, perceived reliability, and ease of understanding. Critical nuance: explanations calibrate trust but can induce **over-reliance** when poorly aligned with actual model accuracy. Curator's explanations must be honest about uncertainty rather than projecting false confidence.

**ACM Systematic Review (2024)**

Scharowski, N., et al. (2024). A Systematic Review on Fostering Appropriate Trust in Human-AI Interaction. *ACM Journal on Responsible Computing.* DOI: 10.1145/3696449

Confirms transparency and explainability are the two most frequently studied trust-building mechanisms, but context matters enormously. Calls for domain-specific trust research.

---

## Data Loss Anxiety & Digital Attachment

**Neave et al. (2019) — Digital Hoarding Scale**

Neave, N., et al. (2019). Digital hoarding behaviours: Measurement and evaluation. *Computers in Human Behavior*, 96, 72–77. DOI: 10.1016/j.chb.2019.01.037

Motivations for digital accumulation: keeping data "just in case," keeping as evidence or proof, laziness/time cost of curation, and **emotional attachment**. Each reason maps to a Curator design requirement:
- "just in case" → undo and recovery must exist
- "evidence" → audit trail of original locations
- "emotional attachment" → never silently delete; always move to discoverable locations
- "laziness" → automation must actually reduce effort

**Sweeten et al. (2018) — Underlying Motivations**

Sweeten, G., et al. (2018). Digital hoarding behaviours: Underlying motivations and potential negative consequences. *Computers in Human Behavior*, 85, 114–121. DOI: 10.1016/j.chb.2018.03.043

Digital hoarding is associated with **anxiety**, **depression**, and feelings of overwhelm from information overload. Users accumulate files precisely because they are afraid of losing them. An AI that moves files without consent triggers the same anxiety that drives hoarding in the first place.

**Data Loss Psychology**

Nearly 70% of users report elevated stress or sadness after significant data loss. Photos and creative work are consistently ranked as the most emotionally irreplaceable digital objects — more than financial documents, which can be reconstructed.

**Barreau & Nardi (2020) — PIM Affect Study**

Barreau, D., & Nardi, B. (2020). Anxious and frustrated but still competent: Affective aspects of interactions with personal information management. *International Journal of Human-Computer Studies*, 142. DOI: 10.1016/j.ijhcs.2020.102470

Identified seven affective dimensions of PIM: **anxiety, efficacy, frustration, desperation, belonging, dependence, and loss of control**. Anxiety and frustration dominate the negative side; efficacy (feeling competent in managing your files) is the key positive. The system must never make users feel they have lost control of their file system.

---

## "Undo" as a Trust-Building Mechanism

**Nielsen's Heuristic #3 — User Control and Freedom**

Nielsen, J. (1994). 10 Usability Heuristics for User Interface Design. Nielsen Norman Group.

"The ability to easily get out of trouble encourages exploration, which facilitates learning and discovery of features." For Curator, **undo is the primary trust-building primitive**. Without it, no rational user should let Curator move their files. With it, users can afford to try the system because the cost of a mistake is bounded.

**Undo Reduces Anxiety and Increases Risk Tolerance**

The NN/g corpus of usability research consistently finds that when users know actions are reversible, they exhibit: lower anxiety, greater willingness to explore, faster learning curves, and higher feature discovery. Reversibility changes the perceived expected cost of any action from "potentially catastrophic" to "recoverable." For file operations this shift is especially large.

**IBM Research — Undo-and-Retry for AI Agents (2025)**

IBM STRATUS system (accepted NeurIPS 2025): every agent action must have a corresponding undo operator. The research formalizes this: **each action must be constrained to be reversible before it is executed, or must require explicit authorization if it cannot be undone**. Curator should adopt this as an engineering principle: moves are reversible; permanent deletions are prohibited unless explicitly authorized.

**Shneiderman (1983) — Rapid Reversible Actions**

Shneiderman, B. (1983). Direct Manipulation: A Step Beyond Programming Languages. *Computer*, 16(8), 57–69.

Argued that direct manipulation interfaces should provide "rapid incremental reversible actions." His warning: "If the computer performs complex inferences, users lose control, predictability can vanish, and the risk of uncertainty increases." The remedy is to keep actions reversible and make system reasoning visible.

---

## Progressive Trust: Gradual Handoff to Automation

**Parasuraman et al.'s Levels as a Progressive Framework**

Applied to Curator:
- **Level 2–3 (Suggest mode):** AI surfaces candidates; human approves all moves. Zero autonomous action. This is the onboarding mode.
- **Level 5–6 (Approve with veto):** AI queues proposed moves, executes in batch after user review window (e.g., 24h). User can veto before execution.
- **Level 7–8 (Act with notification):** AI acts and notifies user. User can undo within 30 days.
- **Level 9 (Act, narrow undo window):** Should require explicit opt-in after extended track record.

**Trust Asymmetry in Automation: Rapid Erosion, Slow Building**

Muir (1994) and subsequent work establish that trust in automation erodes faster than it builds. One bad experience can undo dozens of good ones. Curator should **never escalate autonomy level automatically**. It should require explicit user action to unlock higher automation tiers.

---

## Destructive Action UX Patterns

**Trash/Recycle Bin as Trust Mechanism**

The trash metaphor is itself a soft-delete trust mechanism. Files moved to trash are not deleted — they are staged for deletion. Curator should treat every file move as a "soft move" — the original path and file are preserved in an undo buffer for a defined recovery window.

**Confirmation Dialogs — When They Help and When They Fail**

HCI research establishes key principles:
- Confirmation dialogs are effective for **high-stakes, irreversible actions** but create fatigue if overused for routine operations (the "cry wolf" effect).
- Descriptive button labels ("Move 47 files to Archive/2024/Tax") dramatically outperform vague labels ("Yes/No") in reducing misclicks.
- For maximum-risk actions (permanent deletion, bulk operations), requiring users to **type a confirmation phrase** is the highest-friction but most effective pattern.
- Providing **time-limited undo after confirmation** is superior to pre-confirmation dialogs alone.

**"AgentTrace" Traceability Interface — CHI 2025**

"What Did It Actually Do?": Understanding Risk Awareness and Traceability for Computer-Use Agents. arXiv:2603.28551v2.

Users of agentic computer-use systems struggle to understand what actions were taken, what resources were touched, and what residual effects remain. Core finding: **post-hoc auditability is as important as pre-action transparency**. Users need to understand what happened after the fact, not just before.

---

## Audit Trails & Transparency for Trust

**Explainability and Trust Calibration**

Sullivan & Weger (2025): transparency improved trust across multiple metrics. Explanations help users align their confidence with actual system reliability — trust calibration.

**Audit Trails as Accountability Infrastructure**

AI audit trails serve three trust functions: (1) **post-hoc accountability** — who/what/when/where for each action; (2) **anomaly detection** — users can notice when the system did something unexpected; (3) **governance evidence** — proof that the system operated within boundaries. Studies across enterprise automation show that visible activity logs increase user confidence and reduce anxiety about autonomous operations.

**Transparency Paradox**

The ACM systematic review (Scharowski et al., 2024) notes a paradox: transparency can backfire if it reveals that the AI is uncertain or wrong. The resolution: Curator should surface high-confidence explanations prominently and flag low-confidence moves differently (hold for review rather than auto-execute).

---

## Specific Design Requirements for Curator

**1. Full Reversibility for All Autonomous Moves**
Every file move executed by Curator must be undoable, with original path preserved, for a minimum 30-day recovery window. Permanent deletion must be disabled or require multi-step explicit confirmation with typed acknowledgment. (Basis: Nielsen heuristic #3; Shneiderman 1983; IBM STRATUS 2025)

**2. Progressive Autonomy Unlocked by User Action**
Curator must default to Suggest Mode (level 2–3 in the Parasuraman taxonomy). Higher autonomy modes must be explicitly unlocked by the user. Never auto-escalate. (Basis: Parasuraman et al. 2000; Hoff & Bashir 2015 learned trust model)

**3. Explanation for Every Move**
Every AI-executed move must display a plain-language reason: "Moved 'Q4_invoices.pdf' → Finance/2025/ because filename matches invoice pattern and creation date is Q4 2025." Unexplained errors collapse trust; explained errors are recoverable (Dzindolet et al. 2003).

**4. Persistent Activity Log (Audit Trail)**
A dedicated "Curator History" view showing every action, timestamp, original path, destination path, and reason. Users must be able to undo any individual action from this view. The log must be exportable. (Basis: AgentTrace research; audit trail literature)

**5. Low-Confidence Move Quarantine**
Files the AI is uncertain about should be staged in a "Review Queue" rather than moved automatically. (Basis: Sullivan & Weger 2025; Muir 1994 competence signaling)

**6. Never Move Files Users Explicitly Manage**
Users who maintain personal folder hierarchies should be able to mark directories as "Do Not Touch." (Basis: Barreau & Nardi 2020; digital hoarding motivations)

**7. Emotional Salience Awareness for Photos and Creative Work**
Photos, videos, music, and creative project files should be flagged for manual review rather than auto-moved, given they carry the highest emotional weight and anxiety cost of loss.

**8. Visible Track Record Before Escalation**
Before surfacing the option to unlock higher autonomy, Curator should show the user their accuracy history: "In the last 30 days, Curator suggested 143 file moves. You approved 138, rejected 5, and undid 2. Accuracy: 96%." This provides the analytic trust process Lee & See (2004) describe.

---

## Key References

1. Lee, J. D., & See, K. A. (2004). Trust in Automation: Designing for Appropriate Reliance. *Human Factors*, 46(1), 50–80. DOI: 10.1518/hfes.46.1.50_30392.
2. Hoff, K. A., & Bashir, M. (2015). Trust in Automation. *Human Factors*, 57(3), 407–434. DOI: 10.1177/0018720814547570.
3. Parasuraman, R., Sheridan, T. B., & Wickens, C. D. (2000). A model for types and levels of human interaction with automation. *IEEE Transactions on Systems, Man, and Cybernetics*, 30(3), 286–297. DOI: 10.1109/3468.844354.
4. Muir, B. M. (1994). Trust in Automation: Part I. *Ergonomics*, 37(11), 1905–1922.
5. Dzindolet, M. T., et al. (2003). The role of trust in automation reliance. *International Journal of Human-Computer Studies*, 58(6), 697–718. DOI: 10.1016/S1071-5819(03)00038-7.
6. Davis, F. D. (1989). Perceived usefulness, perceived ease of use, and user acceptance of information technology. *MIS Quarterly*, 13(3), 319–340.
7. Venkatesh, V., et al. (2003). User acceptance of information technology: Toward a unified view. *MIS Quarterly*, 27(3), 425–478.
8. Neave, N., et al. (2019). Digital hoarding behaviours: Measurement and evaluation. *Computers in Human Behavior*, 96, 72–77. DOI: 10.1016/j.chb.2019.01.037.
9. Sweeten, G., et al. (2018). Digital hoarding behaviours: Underlying motivations. *Computers in Human Behavior*, 85, 114–121. DOI: 10.1016/j.chb.2018.03.043.
10. Barreau, D., & Nardi, B. (2020). Anxious and frustrated but still competent. *International Journal of Human-Computer Studies*, 142. DOI: 10.1016/j.ijhcs.2020.102470.
11. Sullivan, V., & Weger, K. (2025). Transparency and Explainability in AI-Assisted Decision Making. *Human Factors*. DOI: 10.1177/10711813251369473.
12. Scharowski, N., et al. (2024). A Systematic Review on Fostering Appropriate Trust in Human-AI Interaction. *ACM Journal on Responsible Computing.* DOI: 10.1145/3696449.
13. Shneiderman, B. (1983). Direct Manipulation. *Computer*, 16(8), 57–69. DOI: 10.1109/MC.1983.1654471.
14. Nielsen, J. (1994). 10 Usability Heuristics for User Interface Design. Nielsen Norman Group.
15. "What Did It Actually Do?" arXiv:2603.28551v2. CHI 2025.
16. IBM Research. (2025). An 'undo-and-retry' mechanism for agents. IBM Research Blog.
17. Scharowski, N., et al. (2021). Measurement of Trust in Automation. *Frontiers in Psychology*, 12, 604977. DOI: 10.3389/fpsyg.2021.604977.
