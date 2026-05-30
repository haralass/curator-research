# FileGram Analysis + Publication Venues
**Compiled:** May 2026

---

## FileGram — Ανάλυση (arXiv:2604.04901)

**Τι είναι:** Agent personalization framework — **όχι** file organizer. Στόχος: να κάνει AI agents να συμπεριφέρονται σαν τον χρήστη, inferred από file-system activity. Δεν οργανώνει αρχεία — μαθαίνει *πώς ήδη δουλεύεις* και replicates αυτό.

### 3 Components

**FileGramEngine** — Synthetic data engine. Simulates user personas και generates file-system action sequences (click, open, rename, move) για να λύσει το privacy barrier της real data collection.

**FileGramBench** — Benchmark σε 4 axes: profile reconstruction, trace disentanglement, persona drift detection, multimodal grounding.

**FileGramOS** — Bottom-up memory architecture. 3 parallel channels:
| Channel | Captures | Form |
|---|---|---|
| Procedural | Behavioral fingerprint (browsing ratios, edit frequency) | 17-dim feature vector |
| Semantic | Content meaning (naming conventions, content embeddings) | 1024-dim embeddings |
| Episodic | Temporal sequences, recurrent patterns | LLM-segmented episodes |

Tracks **22 event types** (12 after filtering): file_read, file_write, file_move, file_rename, dir_create, cross_file_ref, κ.α.

### User Profile Schema
19 attributes across 6 behavioral dimensions: Consumption Pattern, Production Style, Organization Preference, Iteration Strategy, Curation behavior, Cross-Modal behavior.

---

## FileGram vs Curator — Positioning

| Dimension | FileGram | Curator |
|---|---|---|
| **Core goal** | Learn & mirror user behavior | Actively organize & surface files |
| **User model** | Inferred από behavioral traces | Dynamic content + semantic understanding |
| **Output** | Personalized agent actions matching user style | Concrete file organization + suggestions |
| **Organizing logic** | Replicates EXISTING patterns | Discovers NEW semantic categories bottom-up |
| **Privacy model** | Requires persistent behavioral surveillance | Local-first, content-based |
| **User interaction** | Passive (agent acts like you) | Active (user reviews & approves) |
| **New user / cold start** | Fails — needs behavioral history | Works from file content alone |
| **KG** | Δεν χρησιμοποιεί — defers to query time | First-class Typed Knowledge Graph |

**Κεντρική διαφορά:**
- FileGram = **behavioral mirror** ("act like this user")
- Curator = **semantic assistant** ("understand what these files mean")

**Χρήσιμο για paper:** Αυτό το distinction πρέπει να είναι explicit στο Related Work section.

---

## Publication Venues

| Venue | Deadline | Conference | Fit |
|---|---|---|---|
| **IUI 2027** | **Aug 20, 2026** (abstract Aug 13) | Feb 8–11, 2027, Helsinki | ⭐ **Excellent** — explicitly AI+HCI intersection. Ideal για Curator vs FileGram comparison paper |
| **CHI 2027** | **Sep 10, 2026** | ~Apr/May 2027 | ⭐ **Excellent** — prestige play. PIM + local AI papers well-accepted. Higher bar, bigger impact |
| **CSCW** | Rolling (anytime) | 2027 TBD | Good — PIM historically welcome. No deadline pressure |
| **UIST 2026** | ~Mar 24, 2026 (PASSED) | Nov 2–5, 2026, Detroit | Too late για 2026 |
| **IMWUT/UbiComp** | Feb 1 / May 1 / Nov 1 (rolling) | Oct 2026, Shanghai | Moderate — local AI + personal data relevant |
| **MobileHCI 2026** | Jan 29, 2026 (PASSED) | Aug 2026, Swansea | Too late |

### Σύσταση

**Near-term target: IUI 2027** — Aug 20, 2026 deadline. Explicitly focused on AI+HCI. Curator vs FileGram comparison = strong submission.

**Prestige target: CHI 2027** — Sep 10, 2026 deadline. Larger scope, higher bar. "Local AI personal file organizer with Files as Autobiographical Objects" + user study = competitive.

**Fallback: CSCW** — Rolling deadline, no time pressure.

---

## Sources
- [FileGram — arXiv 2604.04901](https://arxiv.org/abs/2604.04901)
- [CHI 2027](https://chi2027.acm.org/authors/papers/)
- [IUI 2027](https://iui.acm.org/2027/)
- [CSCW Rolling](https://cscw.acm.org/rolling.html)
- [HCI Deadlines](https://hci-deadlines.github.io/)
