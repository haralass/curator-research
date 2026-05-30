# HiPEAC Vision 2024 Rationale (Deliverable)
**Σελίδες:** 226 | **Εποχή:** GPT-4, Llama 2/3, AI tools mainstream

---

## Τι Είναι
Το αναλυτικό companion document του 2024 Vision. Κάθε κεφάλαιο γράφτηκε από ειδικούς. Πιο τεχνικό και βαθύ από το executive summary.

---

## ΘΕΜΑ 1: AI-Assisted Software Engineering (AISE) — σελ. 79-130

### Ευκαιρίες
- **Requirements engineering**: LLMs μετατρέπουν natural language requirements σε formal specs
- **Code generation**: GitHub Copilot, Code Llama — "potential to change software profession more than any other recent technology"
  - Εκτίμηση: $1.5 trillion boost σε global GDP by 2030
- **Quality assurance**: AI για bug prediction, test generation — πάνω από 800 papers, 38% annual growth rate
- **Legacy code**: IBM watsonX μετατρέπει COBOL → Java

### Open Challenges (Research Gaps)
- **Hallucinations in code**: GitHub Copilot παράγει vulnerable code σε ~40% των περιπτώσεων — σελ. 80
- **Test oracle problem**: πώς ξέρεις αν το AI-generated test είναι σωστό; — σελ. 81
- **Reward function design για RL**: πώς ορίζεις σωστά το reward για adaptive systems; — σελ. 82
- **AI confidence quantification**: πώς εκφράζει ένα AI model πόσο σίγουρο είναι; — σελ. 81
- **Synergies across SE lifecycle**: πώς συνδυάζεις πολλά AI tools σε coherent pipeline; — σελ. 82

### Σύνδεση με Curator
Το Curator κάνει ακριβώς "AI-assisted decision making" για files. Η ίδια challenge: πώς εκφράζεις confidence; Το confidence calibration που έχεις (HIGH/MEDIUM/LOW) είναι ακριβώς αυτό που ζητά η έρευνα.

---

## ΘΕΜΑ 2: Autonomous Intelligent Systems (AIS) — σελ. ~700

### Challenges
- *σελ. 698*: "Advanced entities such as autonomous AI introduce entirely new set of challenges"
- **Grounding problem** — σελ. 852: πώς συνδέεις abstract AI representations με real-world meaning;
- AI-to-AI interactions: *σελ. 1003-1007*: "Two AI systems might negotiate contracts" — unprecedented legal/ethical territory
- **HSML/HSTP** (σελ. 865-902): νέα modelling language για human-system interactions

### Open Problems
- Formal verification of autonomous AI decisions
- Legal framework για AI-to-AI contracts
- Trust calibration: πότε να εμπιστεύεσαι AI decision;

---

## ΘΕΜΑ 3: NCP Cybersecurity — σελ. 143+

### Challenges
- *σελ. 127*: "NCP throws up even greater cybersecurity and privacy challenges"
- Decentralized internet paradigm ως solution
- Formal security methods για microarchitecture
- Supply chain security

---

## ΘΕΜΑ 4: DLT/IPFS για NCP — σελ. 163+
- Blockchain + IPFS για decentralized storage στο NCP ecosystem
- Ακόμα ερευνητικό — όχι production ready

---

## Πιο Σημαντικά Research Gaps (από ολόκληρο το document)

| Gap | Σελίδα | Ωριμότητα |
|---|---|---|
| AI code correctness / formal verification | 80, 135 | Low — open problem |
| AI confidence quantification | 81 | Low — partial solutions |
| Test oracle problem για AI-generated code | 81 | Low — open |
| Reward function design για adaptive AI | 82 | Medium |
| AI-to-AI interaction protocols | 1003 | Low — nascent |
| Grounding problem για AIS | 852 | Low — philosophical |
| Privacy-preserving AI at edge | 739 | Medium |

---

## Σημασία για Master's

**AISE** είναι η πιο "employable" κατεύθυνση αν θες industry. Κάθε εταιρεία χρειάζεται engineers που καταλαβαίνουν πώς να χτίσουν reliable AI-assisted systems.

**Formal verification + AI** είναι η πιο "academic" κατεύθυνση — high-impact, low competition, EU-funded.
