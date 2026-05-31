# 61 — IRB & Ethics Approval for the Curator User Study
_Research date: 2026-05-31_

---

## Why This Is Urgent

IUI 2027 submission deadline: approximately **October 2026** (based on IUI 2026 cycle: abstract deadline September, full papers October).

Working backwards:
- Paper writing: August–September 2026
- Data collection + analysis: June–August 2026
- **Ethics approval must be in hand by: late May / early June 2026** → application must be submitted **right now**

If ethics approval is not obtained before data collection begins, the study is unpublishable at ACM venues. ACM may require documentary evidence at any point before or after publication. Papers have been desk-rejected or retracted post-acceptance for missing ethics documentation. This is a hard gate, not a formality.

---

## University of Cyprus Ethics Process

### The Governing Body

UCY does **not** operate a fully independent IRB in the US sense. It operates under a two-layer system:

1. **UCY Code of Conduct for Research** (published 2021) — sets the institutional ethical framework and refers research involving human subjects to the national body.
2. **Cyprus National Bioethics Committee (CNBC)** — the statutory national body appointed by the Council of Ministers. Its three Review Ethics Committees (RECs) are chartered specifically for **biomedical** research.

**Critical finding:** A CS/HCI user study involving task timing and file navigation does **not** fall under CNBC jurisdiction.

### What This Means for a CS User Study

For non-biomedical HCI research at UCY, the applicable process is:

- **Department-level or Faculty-level ethics clearance** — most European CS departments handle this internally for low-risk studies
- **Research and Innovation Support Service (RISS)** at UCY may have a lightweight process or self-declaration form for minimal-risk CS studies
- **Supervisor sign-off + documented study protocol** is common practice for CS Master's/PhD students at UCY

### Immediate Actions Required

Contact the following in parallel **this week**:

1. **Your thesis supervisor** — ask whether the CS department has a specific ethics form or whether supervisor approval + documented protocol is sufficient
2. **UCY RISS** — email: riss@ucy.ac.cy. Ask specifically: "Does a minimal-risk CS user study involving task-completion timing with adult volunteers require CNBC review or is departmental/faculty-level clearance sufficient?"
3. **UCY Graduate School** — for PhD students, check the Guidelines for Doctoral Thesis (November 2025) for updated ethics guidance

### Estimated Timeline (UCY, CS department track)

- Departmental/supervisor review of minimal-risk protocol: **1–3 weeks**
- If escalated to faculty committee: **4–8 weeks**
- If CNBC involvement required (unlikely for this study type): **3–6 months** ← would kill the timeline

Goal: confirm the departmental track applies and secure approval within **2–3 weeks of application**.

---

## CHI/IUI Ethics Requirements

### What ACM/SIGCHI Actually Requires

The ACM Publications Policy on Research Involving Human Participants and Subjects (2021):

1. **Local compliance, not a universal certificate.** ACM requires that "research goes through appropriate ethics review requirements that apply to the authors' research environment." It does not require a specific ethics board's stamp.
2. **Contextual note to reviewers.** Both CHI and IUI require a short note explaining the ethics review context for any study involving human subjects. This is a mandatory submission field.
3. **Documentary evidence on demand.** ACM may require documentary evidence before or after publication. You must be able to produce a signed letter, email approval, or formal certificate.
4. **Self-declaration is acceptable** when the local institution does not require formal committee review — but you must document what your institution's policy says. This requires getting that statement **in writing** from UCY/RISS.

### What You Need by Submission Time

One of the following:
- A formal ethics approval letter from UCY's competent body, **OR**
- A written statement from UCY/RISS confirming the study is minimal-risk and does not require formal CNBC review, combined with supervisor sign-off and a complete documented protocol

---

## Minimal-Risk Classification

### Does the Curator Study Qualify?

| Factor | Curator Study | Minimal Risk? |
|---|---|---|
| Physical harm | None — keyboard/mouse tasks | ✅ |
| Psychological harm | No sensitive topics; routine file navigation | ✅ |
| Privacy | Participants use their OWN files on their OWN machine; researcher takes no copies | ✅ |
| Coercion | Volunteer adults; no power relationship required | ✅ |
| Deception | None planned | ✅ |
| Vulnerable populations | Adult volunteers, no minors | ✅ |
| Data sensitivity | Task times, navigation patterns — not medical/financial/legal content | ✅ |
| Duration | 30-minute session | ✅ |

**One nuance:** Think-aloud and screen recordings can incidentally capture sensitive information (file names, private content visible on screen). This must be addressed in the protocol and consent form, but does not disqualify minimal-risk status — it requires stated safeguards.

### Benefits of Minimal-Risk Status

- Eligible for **expedited review** (~2 weeks) rather than full committee review (months)
- May qualify for **exempt** status at some institutions (no ongoing review required)
- Reviewers and program committees treat minimal-risk studies as routine

---

## Informed Consent: What Must Be Covered

### Mandatory Elements (all studies)

1. Study purpose and sponsor; institutional affiliation
2. What participation involves — tasks, duration (30 min), setting (participant's own machine)
3. Voluntary participation and right to withdraw at any time without penalty; data destroyed on request
4. Contact information — researcher name, supervisor name, institutional email

### Critical Elements Specific to This Study

5. **File system access scope** — explicitly state: the researcher does NOT copy, transmit, or retain any file contents; only task completion times and interaction patterns (aggregated, anonymized) are recorded
6. **Screen recording** — what will be recorded, how long it is retained, who has access, when it will be deleted. Participants should close sensitive windows before the session.
7. **Think-aloud audio recording** — audio is for research analysis only, not shared; pseudonymized transcripts only in outputs
8. **File metadata** — if any metadata (file names, directory structure, timestamps) is collected, state how it will be anonymized (path truncation, hash substitution)
9. **Data retention** — how long data is kept, where stored, who can access it
10. **GDPR rights** — EU participants have rights to access, rectification, erasure, restriction. Under Article 89 research exemption, some rights may be limited — disclose this.
11. **Publication** — only aggregated, anonymized results will appear in publications
12. **Compensation** — if any, state it; if none, state it
13. **Signature block**

### Best Practice: Two-Document Approach

- **Participant Information Sheet (PIS):** Plain-language 1-page document
- **Consent Form:** Separate 1-page signature document referencing the PIS

This is standard CHI/HCI practice and signals methodological maturity to reviewers.

---

## GDPR Article 89: Research Exemption

### What It Is

GDPR Article 89 provides derogations from certain data subject rights when personal data is processed for **scientific research purposes**, subject to appropriate safeguards. Cyprus implemented GDPR through **Law 125(I)/2018** (in force July 31, 2018).

### What Article 89 Allows

Article 89(2) permits member state law to limit: right of access (Art. 15), right to rectification (Art. 16), right to erasure (Art. 17), right to restriction (Art. 18), right to object (Art. 21) — where necessary for research purposes. In practice, for a short user study, these limitations are rarely invoked — participants can simply withdraw.

### Required Safeguards ("Appropriate Safeguards" per EDPB)

1. **Data minimisation** — collect only what is strictly necessary: task completion times + interaction counts, not raw file listings or file contents
2. **Pseudonymisation at earliest opportunity** — assign participant IDs (P01–P20) on day one; strip names from all datasets immediately
3. **Technical and organisational measures** — encrypted storage (FileVault, encrypted backup drive), access restricted to research team
4. **Anonymisation for publication** — only aggregated statistics appear in the paper
5. **No individual decision-making** — research findings must not be used to make decisions about specific individuals
6. **Retention limits** — set a defined deletion date (e.g., 2 years post-publication per UCY policy)

### Legal Basis for Processing

Use **Article 6(1)(a) — Consent** combined with Article 89 research exemption. Consent must be freely given, specific, informed, and unambiguous. The consent form is your legal instrument.

### DPIA Requirement

A Data Protection Impact Assessment (DPIA) is **not strictly required** for minimal-risk studies with standard safeguards, but a brief "Privacy Risk Assessment" documenting your safeguards is best practice. Check with UCY's DPO (likely contactable via legal@ucy.ac.cy) whether the study must be registered in the institutional Record of Processing Activities (RoPA).

---

## Data Handling Best Practices from HCI Literature

### What to Collect

- Task completion times ✅
- Error rates / navigation paths (aggregated per condition) ✅
- Think-aloud transcripts (pseudonymized) ✅
- Self-reported difficulty ratings (Likert scales) ✅
- System-generated logs: interaction events with timestamps, no file content ✅

### What to Avoid

- **Raw file names** in any dataset, log, or paper → replace with hashes or generic labels ("Document_A", "Folder_B") before storing
- **Directory tree snapshots** → not necessary for task timing metrics
- **Screen recordings of participants' actual desktops** beyond the task window → instruct participants to close all non-study applications before session
- **Audio that captures ambient conversations** → run sessions in private space

### Anonymization Pipeline

```
Raw session data
→ Assign participant ID (P01–P20)
→ Strip all PII (name, email, machine name, username in paths)
→ Hash or rename any file/folder names in logs
→ Store ONLY the anonymized dataset (delete raw logs after verification)
→ Encrypted storage, researcher access only
→ Aggregated statistics only → paper
```

### For the FLRS Study Specifically

Participants save a test file and later try to recall its location — minimal file system interaction on their own machine. The researcher observes only whether the file is found, not the surrounding file system. This is among the most privacy-preserving PIM study designs possible.

For the FileNav benchmark, logs should capture only: event type (click, type, scroll), timestamp, task success/failure — **not** actual file paths accessed.

---

## Recommended Action Plan

### This Week (June 1–7, 2026) 🔴 URGENT

- [ ] Email your supervisor today — ask about the CS department ethics process
- [ ] Email RISS (riss@ucy.ac.cy) — describe the study, ask what approval pathway applies
- [ ] Draft the **study protocol document** (2–3 pages): aims, participant criteria, procedure, data collected, data handling, GDPR safeguards. **Needed for any approval pathway — start now.**
- [ ] Draft the Participant Information Sheet and Consent Form (template below)

### June 8–21, 2026

- [ ] Submit ethics application to whichever UCY body responds
- [ ] If no institutional review required: obtain a **written email** from supervisor + RISS confirming this; keep on file as your ACM ethics documentation
- [ ] Register study's data processing with UCY's DPO if required
- [ ] Finalize recruitment materials (screener, recruitment email)

### June 22 – July 15, 2026

- [ ] Begin **pilot testing** (2–3 participants) once approval in hand
- [ ] Refine protocol based on pilot
- [ ] Begin full data collection

### July 15 – August 31, 2026

- [ ] Complete data collection (N=20–24 FileNav; N=15–20 FLRS)
- [ ] Analyze data
- [ ] Delete or archive raw recordings per consent form commitments

### September–October 2026

- [ ] Write paper
- [ ] Include ethics note in submission: approval body, date, reference number (or written confirmation reference)
- [ ] Submit to IUI 2027

---

## Consent Form Template

### PARTICIPANT INFORMATION SHEET

**Study title:** Evaluating AI-Assisted File Organization for Personal File Navigation and Retrieval

**Researcher:** [Your Name], Department of Computer Science, University of Cyprus
**Supervisor:** [Supervisor Name], [email]
**Ethics approval reference:** [To be completed after approval]

**What is this study about?**
We are studying how people navigate and retrieve files on their computers, and whether an AI-assisted file organizer (Curator) makes this easier. You will complete short file-finding tasks on your own computer.

**What will you be asked to do?**
One 30-minute session. You will find files on your own computer while we measure the time it takes. You may be asked to think aloud (say what you are doing as you do it). The session will be audio-recorded for transcription purposes only.

**What data will be collected?**
- Task completion times
- Your spoken comments during think-aloud (audio recording)
- Screen recording limited to the file browser and Curator interface only

**What will NOT happen:**
- We will not copy, access, or retain any of your files or their contents
- We will not record the rest of your screen beyond the study task window
- We will not collect your file names or directory structure

**Is participation voluntary?**
Yes. You can withdraw at any time. If you withdraw, all data from you will be deleted.

**How will your data be protected?**
- You will be assigned an anonymous participant ID (e.g., P07)
- Your name will not be linked to any data after the session
- Audio recordings deleted within 30 days of analysis completion
- Only aggregated, anonymized results will appear in any publication
- Data stored on encrypted storage, accessible only to the research team
- Data retained for maximum 2 years following publication, then deleted

**Your rights under GDPR:** You have the right to access, correct, restrict, or object to processing of your personal data. Under GDPR Article 89, some rights may be limited where necessary for research integrity. Contact [researcher email] to exercise any of these rights.

---

### CONSENT FORM

Study: Evaluating AI-Assisted File Organization for Personal File Navigation and Retrieval

- [ ] I have read and understood the Participant Information Sheet
- [ ] I understand that my participation is voluntary and I can withdraw at any time
- [ ] I consent to audio recording for research transcription purposes
- [ ] I consent to screen recording limited to the file browser/Curator interface during tasks
- [ ] I understand my data will be anonymized and only aggregated results will be published
- [ ] I understand my GDPR rights and how to contact the researcher to exercise them
- [ ] I agree to participate in this study

Participant name (print): ___________________________

Participant signature: _________________ Date: _________

Researcher signature: _________________ Date: _________

---

## Key References

- ACM Publications Policy on Research Involving Human Participants: https://www.acm.org/publications/policies/research-involving-human-participants-and-subjects
- IUI 2027 Call for Papers (ethics requirement): https://iui.acm.org/2027/call-for-papers/
- UCY Code of Conduct for Research: https://www.ucy.ac.cy/wp-content/uploads/2021/11/Code_of_Conduct_for_Research.pdf
- UCY Research and Innovation Support Service: https://www.ucy.ac.cy/riss/
- Cyprus National Bioethics Committee: https://www.bioethics.gov.cy/
- EDPB Legal Study on GDPR Article 89(1) Safeguards: https://www.edpb.europa.eu/system/files/2022-01/legalstudy_on_the_appropriate_safeguards_89.1.pdf
- Cyprus GDPR Implementation — Law 125(I)/2018: https://www.dlapiperdataprotection.com/index.html?t=law&c=CY
- EUREC Network — Cyprus Ethics Information: http://www.eurecnet.org/information/cyprus.html
- Scoping Review of Informed Consent in HCI Research (ACM TOCHI): https://dl.acm.org/doi/10.1145/3721284
