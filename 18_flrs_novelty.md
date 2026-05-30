# 18 — FLRS Novelty: Ebbinghaus + SM-2 for File Location Memory
_Date: 2026-05-30_

## Verdict: 🟢 CONFIRMED NOVEL

No paper applies Ebbinghaus decay or SM-2 to file *location* memory as a formal metric. Zero.

## Critical Prior Art to Cite (NOT invalidating — actually helps)

**Elsweiler, Ruthven & Ma (2007) JASIST** — "Towards Memory Supporting PIM Tools"
The closest prior art. Names the problem: "failure to recall the specific LOCATION of an object" as a distinct lapse type. 25 participants, 1.43 lapses/person/day. Argues PIM tools should support this. **No mathematical model, no decay function, no scoring.** → FLRS solves the open problem they identified.

**Barreau & Nardi (1995) SIGCHI Bulletin** — "Finding and Reminding"
Files placed as spatial memory cues — "reminding function." No formalization.

**Pertzov et al. (2012) PLOS ONE** — "Forgetting What Was Where: Fragility of Object-Location Binding" (UCL)
Peer-reviewed cognitive science: object-location binding is neurologically distinct and MORE fragile than object identity memory. **Best empirical foundation for FLRS's core claim.** Never crossed into PIM/computing literature.

## Near-Misses (cite as related, not prior art)

**Andy Matuschak — OS-level SRS design notes** — Imagines spaced repetition for document content review, not for WHERE files are located. Design vision only, never formalized or published.

**YourMemory (GitHub 2024)** — `strength = importance × e^(−λ × days)` for AI agent conversational memory. Same formula structure, completely different domain.

**Obsidian SRS plugins** — SM-2 for reviewing note content, not for locating notes.

## FLRS Formula
```
FLRS(f, t) = R(t, f) × A(f) × B(f, t)
```
- R(t,f) = e^(-t/S)  — Ebbinghaus decay of location memory
- A(f)   = accessibility penalty (folder depth, naming ambiguity)
- B(f,t) = SM-2 stability factor (multiplies with each access)

**The combination is unique.** Problem documented (Elsweiler 2007), cognitive mechanism confirmed (UCL 2012), but no prior formalization exists.

## Narrative for the Paper
"Elsweiler et al. (2007) documented that location forgetting is a distinct and prevalent PIM failure mode. Pertzov et al. (2012) confirmed the underlying cognitive mechanism neurologically. Yet no prior work has formalized this process mathematically or built a system that actively intervenes. FLRS is the first."

## 🚪 New Doors
1. **Pertzov et al. 2012 (PLOS ONE)** — This cognitive science paper is our theoretical bedrock. Are there follow-up papers from the same UCL group that study HOW location memory decays (rate, individual differences)? The decay parameters in our R(t,f) formula should ideally be grounded in empirical neuroscience data, not assumed.
2. **Elsweiler's diary study (25 participants, 1.43 lapses/day)** — Can we replicate this as Phase 1 of our evaluation, now with a comparison to FLRS-equipped Curator? Direct extension of their 2007 work.
3. **Matuschak collaboration?** — He's actively working on memory tools (Orbit, mnemonic medium). Curator is the file-system instantiation of ideas he's explored conceptually. Could be a high-profile connection.
