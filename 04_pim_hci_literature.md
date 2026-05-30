# PIM + HCI Literature — "Files as Autobiographical Objects"
**Compiled:** May 2026 | **Relevance:** Theoretical grounding για CHI paper

---

## Το Κεντρικό Εύρημα

**"Files as Autobiographical Objects" δεν υπάρχει ρητά στη βιβλιογραφία — είναι genuine theoretical gap και novel contribution.**

Το substrate υπάρχει (files έχουν autobiographical σημασία), αλλά κανείς δεν έχει ορίσει design theory γύρω από αυτό.

---

## Ιστορικό Πλαίσιο

**Vannevar Bush (1945) — "As We May Think"**
Το Memex: "an enlarged intimate supplement to his memory." Πρώτη articulation ότι personal information system = extension of human autobiographical memory. Κάθε PIM system μετά είναι Memex descendant.

---

## Key Authors & Foundational Papers

| Συγγραφέας | Paper | Key Insight |
|---|---|---|
| **Malone (1983)** | "How Do People Organize Their Desks?" — ACM TOIS | Filer/piler dichotomy. Πρωταρχικός σκοπός οργάνωσης = **reminding** (τι να κάνεις), όχι retrieval |
| **Lansdale (1988)** | "The Psychology of Personal Information Management" — Applied Ergonomics | PIM = memory problem. Retrieval γίνεται μέσω **episodic memory cues** (πότε, πού, context) |
| **Whittaker & Sidner (1996)** | "Email Overload" — CHI 1996 | 3 τύποι: no-filers, frequent-filers, spring-cleaners. Email γίνεται **accretive archive** ανεξάρτητα από πρόθεση |
| **Bergman (2003)** | "The user-subjective approach to PIM" — JASIST | PIM tools βασισμένα σε public IR αποτυγχάνουν — personal info είναι **subjective, contextual, relational** |
| **Boardman (2004)** | "Stuff Goes into the Computer and Doesn't Come Out" — CHI | Cross-tool inconsistency universal. Files "go in and don't come out" |
| **Jones (2007)** | "Personal Information Management" — Annual Review | Canonical field synthesis |
| **Bergman & Whittaker (2016)** | *The Science of Managing Our Digital Stuff* — MIT Press | The definitive book. User-subjective approach. Folder hierarchies = cognitive structures, not to be replaced |

---

## Autobiographical Memory & Digital Objects

| Paper | Key Insight |
|---|---|
| **Kirk & Sellen (2010)** — ACM TOCHI | 7 archiving values: remembering, **defining the self**, forgetting, fulfilling duty, framing family, connecting with past, honoring others. Files = **identity construction**, not just data |
| **Cushing (2011, 2013)** — JASIST | Digital possessions = extended self (Belk's theory). "It's stuff that speaks to me" — files are active communicative objects |
| **Hutmacher et al. (2023/2024)** | AMEDIA model. Autobiographical remembering in digital age shaped by technology. Risk: "digital amnesia" |

---

## Lifelogging Research

| Project | Key Insight |
|---|---|
| **Bell & Gemmell — MyLifeBits (2006)** | Total capture = possible. Βottleneck = **attention and curation**, not storage |
| **SenseCam / Microsoft Research** | Passive wearable camera. Review of images **improves episodic memory** (CHI 2007, Memory 2011). Images "work differently over time" — early: vivid recall, later: only "knowing" |

**Θεωρητικό συμπέρασμα:** Η *πράξη κuration* (reviewing, organizing) — όχι απλώς η αποθήκευση — ενεργοποιεί autobiographical memory. Ένας file organizer πρέπει να διευκολύνει **reflective engagement**, όχι απλώς efficient retrieval.

---

## File Organization Behavior Studies

**Filer / Piler Taxonomy** (Malone 1983 → Whittaker 2001):
- **Filers**: Premature filing, hierarchical structures, μεγάλα archives με λίγη χρήση. "Highly-structured trash cans."
- **Pilers**: Μικρά, actively-used archives. Rely on spatial/temporal cues. Faster re-retrieval.
- **Spring cleaners**: Periodic bulk sorting, χάνουν αρχεία κατά reorganization.

**Bergman's key finding:** Folder navigation προτιμάται από search για personal files — people trust spatial memory ("third subfolder of Projects") over keyword recall. Undermines "just use search" design.

**Bergman & Whittaker (2016) synthesis:** People keep too much, organize inconsistently, and lose things anyway. Το επιθυμητό σύστημα μειώνει cognitive overhead χωρίς να καταστρέφει spatial memory structures.

---

## Digital Hoarding

| Paper | Key Insight |
|---|---|
| Van Bennekom et al. (2015) | Ορισμός "digital hoarding". Digital Hoarding Questionnaire |
| Fullwood et al. (2018) — CHiB | Emotional attachment = primary driver |
| 2024-2025 research | **Fear of missing out + emotional attachment** = photo hoarding predictors |

DHQ sample item: *"Deleting certain files would be like losing part of myself"* — autobiographical language σε psychometric instrument.

**Key asymmetry:** Declining storage cost → rational to keep. Emotional cost of deletion → remains high. Curator's AI πρέπει να βοηθά **curate** όχι **purge**.

---

## Digital Forgetting

| Paper | Key Insight |
|---|---|
| Brewer et al. — "Time's Sublimest Target" | Deletion = "crude binary". Designed forgetting mechanisms needed |
| Odom et al. CHI 2013 — "Design for Forgetting" | Post-breakup files become "evocative and upsetting". Tools don't support nuanced disposal |

**Design insight για Curator:** **Graceful deprecation** — "faded memories" zone — αντί για binary keep/delete.

---

## Most Adjacent AI System: FileGram (2026) ⚠️

**FileGram: Grounding Agent Personalization in File-System Behavioral Traces** — arXiv:2604.04901, April 2026

Χτίζει user profiles από file-system actions → procedural, semantic, episodic memory channels.

**Η κρίσιμη διαφορά:**
- FileGram: file behavior = **agent personalization signal** (AI adapts to user)
- Curator: file behavior = **self-representation signal** (user understands their own digital history)

Φιλοσοφικά διαφορετικό claim. **Πρέπει να γνωρίζεις αυτό το paper και να κάνεις positioning ρητά στο CHI paper.**

---

## Πώς "Files as Autobiographical Objects" Fits & Differs

### Builds On (cite these):
| Concept | Πώς το επεκτείνεις |
|---|---|
| Lansdale episodic cues | Autobiographical cues = **primary organizational signal**, όχι bug |
| Kirk & Sellen 7 values | Systematize σε design framework για AI curator |
| Bergman's user-subjective | Προσθέτεις temporal/narrative dimension |
| Bell's MyLifeBits | Curation layer (όχι capture) = where meaning is made |
| SenseCam research | Temporal review activates autobiographical memory → Curator UI |

### What's Genuinely Novel:
1. **Autobiographical framing = design-generative**: Existing work describes the phenomenon. Η thesis σου λέει: αν τα files είναι autobiographical, ο organizer πρέπει να λογίζεται ως **biographer/archivist**, όχι search engine.
2. **Behavioral signals = life-narrative signals**: Τα signals χρησιμοποιούνται για relevance prediction. Εσύ τα χρησιμοποιείς ως **biographical significance** — διαφορετική χρήση των ίδιων δεδομένων.
3. **Forgetting as narrative**: Choosing what belongs in your archive = **act of self-authorship**.
4. **Κανένα CHI paper δεν κάνει αυτό το argument για files ρητά** — confirmed gap.

---

## Papers να Cite στο CHI Paper

**Foundational PIM:**
- Malone (1983) — ACM TOIS
- Lansdale (1988) — Applied Ergonomics
- Whittaker & Sidner (1996) — CHI
- Jones (2007) — Annual Review
- Bergman & Whittaker (2016) — MIT Press

**Autobiographical:**
- Kirk & Sellen (2010) — ACM TOCHI
- Cushing (2013) — JASIST
- Hutmacher et al. (2023) — Applied Cognitive Psychology

**Lifelogging:**
- Bush (1945) — Atlantic Monthly
- Bell & Gemmell (2006) — CACM
- Hodges et al. (2011) — Memory (SenseCam)

**File behavior:**
- Boardman (2004) — CHI
- Bergman (2003) — JASIST

**Digital hoarding:**
- Fullwood et al. (2018) — CHiB

**Forgetting:**
- Odom et al. (2013) — CHI

**Most adjacent system (positioning):**
- FileGram — arXiv:2604.04901 (2026)

**Theory:**
- Bowker & Star (1999) — *Sorting Things Out* — MIT Press

---

## Sources
- [FileGram — arXiv 2604.04901](https://arxiv.org/abs/2604.04901)
- [MyLifeBits — CACM](https://dl.acm.org/doi/10.1145/1107458.1107460)
- [Kirk & Sellen — ACM TOCHI](https://dl.acm.org/doi/10.1145/1806923.1806924)
- [Bergman & Whittaker — MIT Press](https://mitpress.mit.edu/9780262035170/the-science-of-managing-our-digital-stuff/)
- [SenseCam — Memory journal](https://www.researchgate.net/publication/51717483_SenseCam_A_wearable_camera_that_stimulates_and_rehabilitates_autobiographical_memory)
- [Cushing — JASIST](https://onlinelibrary.wiley.com/doi/10.1002/asi.22864)
- [Odom et al. — CHI 2013](https://www.researchgate.net/publication/262204635_Design_for_forgetting_Disposing_of_digital_possessions_after_a_breakup)
- [Fullwood et al. — CHiB](https://www.sciencedirect.com/science/article/abs/pii/S0747563218301365)
- [AMEDIA Model](https://www.tandfonline.com/doi/full/10.1080/1047840X.2024.2384125)
- [Malone — ACM TOIS](https://dl.acm.org/doi/10.1145/357423.357430)
