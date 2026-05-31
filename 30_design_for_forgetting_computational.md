# Design for Forgetting: Computational Prior Art and FLRS as the First Computational Instantiation

**Date:** 2026-05-31
**Project:** Curator / FLRS Novelty Analysis
**Purpose:** Establish the intellectual lineage of "Design for Forgetting" in HCI and locate FLRS's precise novelty claim as the first *computational* instantiation of this design philosophy in the context of file management.

---

## 1. The "Design for Forgetting" Intellectual Lineage

### 1.1 Mayer-Schönberger (2009): The Foundational Philosophical Argument

The intellectual foundation for intentional digital forgetting begins with Viktor Mayer-Schönberger's *Delete: The Virtue of Forgetting in the Digital Age* (Princeton University Press, 2009). Mayer-Schönberger observed a structural asymmetry in digital systems: whereas human memory naturally decays, digital storage by default retains everything perfectly and indefinitely. He argued this asymmetry is not neutral — it has real costs to human autonomy, second chances, and dignity, because outdated information remains permanently accessible, stripped of context.

His most concrete proposal was the imposition of **expiration dates on digital information** — metadata attached at creation time that would cause content to automatically degrade or delete after a specified period. This is proto-computational: it imagines a machine-executed forgetting schedule. However, Mayer-Schönberger's framing was primarily policy and philosophical. The expiration date idea was a provocation and a design direction, not an implemented algorithm. No forgetting *curve* was proposed; the model was binary (keep vs. delete at a fixed date), not continuous decay calibrated to human memory dynamics.

This work is the essential precursor that legitimized forgetting as a *design value* rather than a failure.

### 1.2 Odom et al., CHI 2012: "Technology Heirlooms?"

William Odom, Richard Banks, Richard Harper, David Kirk, Siân Lindley, and Abigail Sellen published "Technology Heirlooms? Considerations for Passing Down and Inheriting Digital Materials" at CHI 2012 (Austin, Texas). The paper addressed how physical artifacts carry memory, identity, and relationship across generations, and how digital materials fail to perform this function.

The research method was **design-as-probe**: the team built three tangible interactive prototypes — Timecard (a timeline device for assembling and hiding family digital content), Digital Slide Viewer (a photo album packaged in a vintage slide viewer form factor), and BackupBox (which locally archives a person's Twitter history in a hand-downable physical form) — and deployed them with eight families via in-home interviews.

Crucially, these prototypes were **design research instruments**, not deployed computational systems. Their purpose was to provoke conversation about digital inheritance practices. The paper's output was a set of design considerations and open questions — for example, how families might want selective visibility into digital materials, or how the patina of time might be conveyed digitally. No algorithm for decay, no score, no automated forgetting schedule was implemented or proposed. The work was entirely in the tradition of reflective and critical design, using physical objects to surface latent human values around memory and legacy.

### 1.3 Sas and Whittaker, CHI 2013: "Design for Forgetting"

Corina Sas (Lancaster University) and Steve Whittaker (UC Santa Cruz) published "Design for Forgetting: Disposing of Digital Possessions after a Breakup" at CHI 2013 (Paris). This paper is the source of the phrase "Design for Forgetting" as a formal HCI research orientation.

The methodology was qualitative: interviews with 24 participants aged 19–34 about their strategies for managing digital possessions after romantic breakups. The study found that digital possessions significantly outnumber physical ones in this context (averaging 5.4 digital to 1.4 physical items), with photographs and social network contacts being the most emotionally evocative. Participants exhibited distinct and often tortured strategies: deletion, archiving to inaccessible locations, blocking, and leaving materials in place while attempting to avoid re-encounter.

The authors proposed that software systems could assist in forgetting by supporting practices like "harvesting" unwanted digital memories using facial recognition, machine learning, or entity extraction. These proposals were speculative design implications — no system was built. The paper was a call to action for HCI researchers, identifying intentional forgetting as an underexplored but important design space.

The contribution is conceptual and taxonomic: it named a design problem, mapped the human strategies that address it, and suggested categories for future systems. It is not a computational implementation. The paper explicitly frames "Design for Forgetting" as a research agenda to be pursued, not a solution delivered.

### 1.4 Odom et al., CHI 2014: "Designing for Slowness, Anticipation and Re-Visitation"

William Odom, Abigail Sellen, Richard Banks, David Kirk, Tim Regan, Mark Selby, Jodi Forlizzi, and John Zimmerman published "Designing for Slowness, Anticipation and Re-Visitation: A Long Term Field Study of the Photobox" at CHI 2014 (Toronto), where it received a Best Paper Award.

The Photobox is a networked oak chest that prints four or five randomly selected photographs from the owner's Flickr collection at random intervals — roughly monthly — and deposits them physically. The user has no control over selection timing or which photos appear. Three homes used the device for fourteen months.

The findings are significant for the FLRS novelty argument in two respects. First, the Photobox operationalized **re-visitation** — the idea that encountering past digital content at a pace determined by the system (not the user) can produce reflection, renewed engagement, and a recalibration of the user's relationship to their own archive. Participants moved from initial frustration at losing control, to acceptance, to genuine appreciation. Second, the paper introduced **slowness** as a first-class design principle: slowing the rate at which digital content surfaces creates psychological space for meaning-making that high-speed, on-demand retrieval forecloses.

However, the Photobox's resurfacing mechanism is **random**, not calibrated to any model of human memory or forgetting. Selection is uniformly distributed across the Flickr collection, not weighted by recency, access frequency, or memory decay rates. There is no forgetting curve, no score, no attempt to model what the user is likely to have forgotten. The "slowness" is achieved by limiting throughput, not by reasoning about cognitive state.

This is the closest prior art in spirit to FLRS-driven resurfacing, yet it remains categorically distinct: the Photobox is a physical object implementing a stochastic process, not a computational system modeling memory retention.

---

## 2. The ForgetIT EU Project: The Closest Computational Prior Art

The ForgetIT project ("Concise Preservation by Combining Managed Forgetting and Contextualized Remembering," EU FP7, grant 600826, 2013–2016) represents the most substantive pre-FLRS attempt to implement forgetting computationally in the context of personal information management.

Coordinated by researchers at DFKI (German Research Center for Artificial Intelligence) and involving institutions including ITI Thessaloniki, the project built a **Preserve-or-Forget Framework** that operated on the Semantic Desktop paradigm. The core concept was the "significance score" — a multi-dimensional measure of how important a digital object is to a user, derived from access frequency, social signals, content analysis, and contextual cues. Objects whose significance score decayed below a threshold were candidates for managed forgetting: condensation, de-duplication, or outright deletion.

Niederée and Kanhabua (2015), in "Forgetful Digital Memory: Towards Brain-Inspired Long-Term Data and Information Management," framed this explicitly as a cognitive-systems analogy: just as the human brain selectively retains high-salience memories and lets low-salience traces fade, a digital information system should do the same. Jilek et al. (2018) followed with "Managed Forgetting to Support Information Management and Knowledge Work" (*KI – Künstliche Intelligenz*, Springer), reporting on five years of prototype deployment in daily knowledge-work contexts.

**The critical distinction from FLRS:** ForgetIT and the Niederée/Jilek managed-forgetting lineage are concerned with **what to keep** — their computational mechanism drives a preservation-vs-deletion decision tree calibrated to content importance. The user's cognitive state is modeled indirectly and only insofar as access patterns signal importance. There is no model of *whether the user remembers where a file is*. The forgetting being modeled is "does this file still matter?" not "does the user remember this file's location?" These are fundamentally different forgetting phenomena.

Furthermore, ForgetIT does not apply the Ebbinghaus forgetting curve directly. Significance scores use access recency and frequency as proxies, but they are not calibrated to the parametric curve R = e^(−t/S) that Ebbinghaus established for memory retention as a function of time elapsed since last encoding. The Ebbinghaus model is referenced in the background literature but is not the algorithmic engine.

---

## 3. Spaced Repetition Systems: Forgetting Curves Operationalized — But for Knowledge, Not Files

The only domain where Ebbinghaus-style forgetting curves have been rigorously implemented as computational algorithms is spaced repetition for knowledge retention. The genealogy runs from Piotr Wozniak's SuperMemo (1987, SM-2 algorithm) through Anki (using SM-2 or the newer FSRS algorithm) to modern adaptive learning platforms.

The SM-2 algorithm computes inter-repetition intervals as a function of a per-card "easiness factor" (E-Factor, ranging 1.1 to 2.5) that adjusts based on rated recall quality. Each review resets the forgetting curve for that item; the next review is scheduled at the point where predicted retention drops to approximately 90%. The FSRS (Free Spaced Repetition Scheduler) system, introduced in Anki 23.10, improves on SM-2 by fitting a more principled model of memory decay to actual user recall data.

These systems operationalize Ebbinghaus precisely and computationally. However, the **domain** is entirely different: spaced repetition schedules review of discrete knowledge items (flashcards, vocabulary, facts) to maintain declarative memory. The question being asked is: "Does the user still know this fact?" The answer to "does the user know where this file is?" — a spatial-navigational memory question about a personal information environment — has never been the target.

No PIM system, no file manager, no desktop operating system environment has applied forgetting-curve modeling to file location memory. This gap is precisely the space FLRS occupies.

---

## 4. Legal Forgetting: GDPR "Right to be Forgotten"

Article 17 of the EU General Data Protection Regulation (GDPR, effective 2018) establishes the "right to erasure" — commonly called the right to be forgotten. Data subjects may request that controllers delete personal data in specified circumstances (data no longer necessary for original purpose, consent withdrawn, unlawful processing, etc.).

This is forgetting implemented at the legal and policy layer. It is:
- **Demand-driven**: initiated by individual request, not automated
- **Binary**: data is either erased or retained, not gradually degraded
- **Legally motivated**: the goal is privacy and data minimization, not cognitive alignment
- **Untethered from cognitive science**: there is no Ebbinghaus curve, no retention score, no model of human memory dynamics

GDPR "right to be forgotten" is relevant as context — it demonstrates society-wide recognition that digital permanence is a problem requiring structural intervention — but it is categorically different from design-for-forgetting in the HCI sense and from FLRS in particular. Legal erasure and cognitive forgetting are different phenomena that happen to share a word.

---

## 5. FLRS: Computational Operationalization of Design for Forgetting in File Management

### 5.1 What FLRS Is

The File Location Retention Score (FLRS), as implemented in Curator (macOS AI file organizer), computes a per-file score that models the probability that a user retains accurate spatial memory of where a given file lives in their file system hierarchy. The score is computed using the Ebbinghaus retention function:

```
R(t) = e^(−t / S)
```

where `t` is the time elapsed since the user last accessed or navigated to a file, and `S` is a stability parameter that reflects how well-consolidated the memory of that file's location is (itself a function of access history, depth in the hierarchy, and organizational regularity). Files for which R drops below a threshold are flagged for proactive resurfacing — Curator surfaces them, contextualizes them, and prompts the user to re-encounter them.

### 5.2 What Makes FLRS Novel

FLRS is the first system to satisfy all of the following conditions simultaneously:

1. **Applies the Ebbinghaus forgetting curve parametrically** — not as a metaphor, not as a proxy accessed through access-frequency heuristics, but as the direct algorithmic engine.

2. **Targets file location memory** — the specific cognitive phenomenon being modeled is the user's spatial-navigational knowledge of their own file system. This is distinct from "does this file matter?" (ForgetIT's question) and "do I still know this fact?" (spaced repetition's question).

3. **Drives re-encounter, not deletion** — unlike ForgetIT and GDPR, FLRS does not use the forgetting model to make a preservation-or-deletion decision. It uses it to trigger resurfacing: bringing a forgotten-location file back to the user's attention before the location memory fully degrades.

4. **Operates on files, not knowledge items** — the objects in the system are files in a personal file system, not flashcard-style knowledge units. The organizational context (folder hierarchy, naming conventions, co-located files) is part of the memory encoding that FLRS models.

5. **Is fully automated** — unlike Sas and Whittaker's design proposals (which remained speculative), unlike the Photobox (which required physical deployment), and unlike GDPR (which requires user-initiated requests), FLRS runs continuously as a background process without user intervention.

### 5.3 Alignment with the Slow Technology Tradition

Odom et al.'s Photobox established that system-mediated re-visitation — encountering your own digital past at a system-controlled pace rather than on demand — produces qualitatively different and often more reflective engagement with personal content. FLRS-triggered resurfacing has the same structural property: the system decides when to surface a file, based on a model of the user's memory state, not on explicit user request.

Where FLRS improves on the Photobox's intellectual model is in the rationale for *when* to surface content. The Photobox uses a random process; FLRS uses a cognitive model. The Photobox surfaces photos at roughly monthly intervals regardless of the user's actual memory of them; FLRS surfaces files at the moment their predicted location memory falls to a threshold, which is different for every file and every user. This makes FLRS a computational instantiation of the re-visitation principle rather than a stochastic approximation of it.

---

## 6. Novelty Verdict

The question posed is whether FLRS is genuinely the first computational operationalization of design-for-forgetting in file management. The evidence supports an affirmative answer, with the following nuances.

**Odom et al. (2012)** conducted design research with physical prototypes to surface values around digital memory and inheritance. No algorithm. No deployed computational system.

**Sas and Whittaker (2013)** named the "Design for Forgetting" agenda, conducted qualitative interviews, and proposed speculative system features. No algorithm. No deployed system.

**Mayer-Schönberger (2009)** proposed expiration dates on digital information. Binary and fixed, not calibrated to Ebbinghaus. Not implemented.

**Odom et al. (2014)** demonstrated re-visitation via a slow-technology physical device using random selection. No forgetting curve. No file system. Not computational.

**ForgetIT / Niederée / Jilek (2013–2018)** implemented computational significance scoring to drive preservation decisions for digital personal information. Uses access patterns as proxies, not Ebbinghaus directly. Targets content importance, not location memory. Does not trigger resurfacing; triggers deletion or condensation.

**Spaced repetition systems (SuperMemo, Anki)** implement Ebbinghaus parametrically and computationally, but for declarative knowledge items (flashcards), not files. No file system integration. No location memory modeling.

**FLRS** applies Ebbinghaus's forgetting curve parametrically to model per-file location memory retention, runs as an automated background process, and uses the score to trigger proactive resurfacing — not deletion. It is the first system to computationally operationalize the HCI design-for-forgetting agenda in the specific domain of personal file management.

The one caveat worth acknowledging: "first" claims are always subject to the completeness of the search. The ForgetIT lineage is the most substantive adjacent prior art, and the distinction from FLRS rests on the difference between modeling content importance (ForgetIT) and modeling location memory (FLRS). This is a real and defensible distinction, not a rhetorical one. It should be articulated precisely in any novelty claim.

---

## 7. Summary Table

| Work | Year | Computational? | Ebbinghaus? | Files? | Location Memory? | Re-visitation? |
|---|---|---|---|---|---|---|
| Mayer-Schönberger *Delete* | 2009 | No (policy) | No | No | No | No |
| Odom et al. CHI 2012 | 2012 | No (design probes) | No | No | No | No |
| Sas & Whittaker CHI 2013 | 2013 | No (qualitative) | No | No | No | No |
| Odom et al. CHI 2014 (Photobox) | 2014 | Partial (physical device, random) | No | No | No | Yes (random) |
| ForgetIT / Niederée / Jilek | 2013–2018 | Yes | Indirectly | Yes | No | No (deletion) |
| SuperMemo / Anki (SM-2) | 1987–present | Yes | Yes | No | No | Yes (knowledge items) |
| **FLRS (Curator)** | 2024–present | **Yes** | **Yes** | **Yes** | **Yes** | **Yes (score-triggered)** |

---

## References

- Viktor Mayer-Schönberger. *Delete: The Virtue of Forgetting in the Digital Age*. Princeton University Press, 2009.
- William Odom, Richard Banks, Richard Harper, David Kirk, Siân Lindley, Abigail Sellen. "Technology Heirlooms? Considerations for Passing Down and Inheriting Digital Materials." *Proceedings of CHI 2012*, pp. 337–346. ACM.
- Corina Sas and Steve Whittaker. "Design for Forgetting: Disposing of Digital Possessions after a Breakup." *Proceedings of CHI 2013*, pp. 1823–1832. ACM.
- William Odom, Abigail Sellen, Richard Banks, David Kirk, Tim Regan, Mark Selby, Jodi Forlizzi, John Zimmerman. "Designing for Slowness, Anticipation and Re-Visitation: A Long Term Field Study of the Photobox." *Proceedings of CHI 2014* (Best Paper Award). ACM, DOI: 10.1145/2556288.2557178.
- Claudia Niederée and Nattiya Kanhabua. "Forgetful Digital Memory: Towards Brain-Inspired Long-Term Data and Information Management." *Proceedings of the Workshop on Enriching Information Retrieval with Life-Long User Modelling*, 2015. Semantic Scholar: 765c404ff910bc1f3fc8f78503f365841036e70e.
- Christian Jilek, Markus Schröder, Sven Schwarz, Heiko Maus, Andreas Dengel. "Managed Forgetting to Support Information Management and Knowledge Work." *KI – Künstliche Intelligenz*, 33(1), pp. 25–35. Springer, 2018. DOI: 10.1007/s13218-018-00568-9. arXiv: 1811.12155.
- ForgetIT Project. "Concise Preservation by Combining Managed Forgetting and Contextualized Remembering." EU FP7, Grant 600826, 2013–2016. https://www.forgetit-project.eu/
- Piotr Wozniak. SuperMemo SM-2 Algorithm, 1987. https://www.supermemo.com/en/supermemo-method
- Hermann Ebbinghaus. *Über das Gedächtnis: Untersuchungen zur experimentellen Psychologie*. Leipzig: Duncker & Humblot, 1885.
