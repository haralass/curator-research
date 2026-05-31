# 18 — FLRS Novelty: Ebbinghaus + SM-2 for File Location Memory
_Date: 2026-05-30_

## Verdict: 🟢 CONFIRMED NOVEL

No paper applies Ebbinghaus decay or SM-2 to file *location* memory as a formal metric. Zero.

## Critical Prior Art to Cite (NOT invalidating — actually helps)

**Elsweiler, Ruthven & Jones (2007) JASIST** — "Towards Memory Supporting PIM Tools"
The closest prior art. Names the problem: "failure to recall the specific LOCATION of an object" as a distinct lapse type. 25 participants, 1.43 lapses/person/day. Argues PIM tools should support this. **No mathematical model, no decay function, no scoring.** → FLRS solves the open problem they identified. [doi:10.1002/asi.20570]

**Barreau & Nardi (1995) SIGCHI Bulletin** — "Finding and Reminding"
Files placed as spatial memory cues — "reminding function." No formalization. [doi:10.1145/221296.221307]

**Pertzov et al. (2012) PLOS ONE** — "Forgetting What Was Where: Fragility of Object-Location Binding" (UCL)
Peer-reviewed cognitive science: object-location binding is neurologically distinct and MORE fragile than object identity memory. **Best empirical foundation for FLRS's core claim.** Never crossed into PIM/computing literature. [doi:10.1371/journal.pone.0048214]

## Near-Misses (cite as related, not prior art)

**Andy Matuschak — OS-level SRS design notes** — Imagines spaced repetition for document content review, not for WHERE files are located. Design vision only, never formalized or published.

**YourMemory (GitHub 2024)** — `strength = importance × e^(−λ × days)` for AI agent conversational memory. Same formula structure, completely different domain.

**Obsidian SRS plugins** — SM-2 for reviewing note content, not for locating notes.

## FLRS Formula
```
FLRS(f, t) = R(t, f) × A(f) × B(f, t)
```
- R(t,f) = e^(-t/S)  — Ebbinghaus decay of location memory [Ebbinghaus 1885; Murre & Dros 2015]
- A(f)   = accessibility penalty (folder depth, naming ambiguity)
- B(f,t) = SM-2 stability factor (multiplies with each access) [Wozniak 1990]

**The combination is unique.** Problem documented [Elsweiler et al. 2007], cognitive mechanism confirmed [Pertzov et al. 2012], but no prior formalization exists.

## Narrative for the Paper
"Elsweiler et al. (2007) documented that location forgetting is a distinct and prevalent PIM failure mode. Pertzov et al. (2012) confirmed the underlying cognitive mechanism neurologically. Yet no prior work has formalized this process mathematically or built a system that actively intervenes. FLRS is the first."

## References

- Elsweiler, D., Ruthven, I., & Jones, C. (2007). "Towards Memory Supporting Personal Information Management Tools." *Journal of the American Society for Information Science and Technology*, 58(7), 924–946. DOI: 10.1002/asi.20570.
- Barreau, D. K., & Nardi, B. (1995). "Finding and reminding: File organization from the desktop." *ACM SIGCHI Bulletin*, 27(3), 39–43. DOI: 10.1145/221296.221307.
- Pertzov, Y., Dong, M. Y., Peich, M.-C., & Husain, M. (2012). "Forgetting What Was Where: The Fragility of Object-Location Binding." *PLOS ONE*, 7(10), e48214. DOI: 10.1371/journal.pone.0048214.
- Ebbinghaus, H. (1885). *Über das Gedächtnis: Untersuchungen zur experimentellen Psychologie*. Duncker & Humblot, Leipzig. (Translated as: Memory: A Contribution to Experimental Psychology, 1913.)
- Murre, J. M. J., & Dros, J. (2015). "Replication and Analysis of Ebbinghaus' Forgetting Curve." *PLOS ONE*, 10(7), e0120644. DOI: 10.1371/journal.pone.0120644.
- Wozniak, P. A. (1990). "Optimization of Learning." Master's thesis, University of Technology, Poznan, Poland. (SM-2 algorithm underlying SuperMemo spaced repetition.)
- Matuschak, A. (2019). "Spaced repetition systems as a medium, not just a tool." https://notes.andymatuschak.org/Spaced_repetition_systems_as_a_medium,_not_just_a_tool.
