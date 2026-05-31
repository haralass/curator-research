# 00 — Research Strategy: What to Pursue Now vs. Later
_Date: 2026-05-31 — Synthesized from full research scan of files 01-46_

---

## The Core Research Framing

**Not:** "I built an AI file organizer."

**Yes:** "Curator: Adaptive and Calibrated Local AI for Personal File Organization with File Biography and Memory-Aware Retrieval"

The five research pillars that hold the IUI 2027 paper:
1. **Level Router** — adaptive tiered analysis
2. **Calibrated Routing** — conformal prediction + review fatigue
3. **File Biography** — Markov-based file lifecycle
4. **FLRS** — memory-aware retrieval with forgetting curves
5. **FileNav Benchmark** — empirical evaluation including foreign user condition

---

## What to Pursue NOW (IUI 2027 scope)

### 🔴 Tier 1 — Core Contributions (must be in paper)

**Level Router**
Formal evaluation: what % of files resolve at each level (0–5)? Accuracy vs. compute tradeoff per level. Optimal stopping rule. This is the novel system contribution that proves Curator is not a naive AI wrapper. Zero prior art in PIM literature.

**Calibrated Routing + Review Fatigue**
Conformal Prediction gives formal coverage guarantees (H(L|I) upper bound via Angelopoulos NeurIPS 2024). Review fatigue: how many decisions before user disengages or approves mechanically? What is the optimal batch size? Both are HCI + ML contributions.

**File Biography + Markov Entropy**
Per-file state sequence + stationary entropy for instability detection. Cross-validated with GhostUMAP2 (file 37). First Markov model of personal file lifecycle.

**FLRS Empirical Validation**
S₀ ≈ 1.0 day has no direct empirical support. A small within-subjects study (N=20, save a test file, measure when user can't recall location) would calibrate S₀ and make FLRS directly empirically grounded. Publishable as CHI/IUI short paper + directly strengthens main paper.

**FileNav Benchmark + Foreign User Condition**
Measure time-to-find: Finder only / Spotlight / Curator semantic search / Curator organized folders. Foreign user condition (can someone else navigate your organized filesystem?) is a novel benchmark element (file 33). N=20–24, within-subjects.

### 🟡 Tier 2 — Strong additions if time permits

**File Role Detection** (file 42)
Topic ≠ role. Lecture vs. assignment vs. receipt vs. draft. First personal file role taxonomy and classifier. Directly improves routing accuracy and smart rename.

**Version Semantics** (file 46)
draft→revision→final→submitted chain detection. Multi-signal (filename patterns + embedding similarity + format change). More powerful than exact duplicate detection.

**H(location|intent) as Quality Metric** (file 29)
Formally define and measure. Connect to CP: `H(L|I) ≤ φ(α, E[|S|])` — formal guarantee language. First information-theoretic PIM quality metric.

**Organizational Burden Index (OBI)** (file 41)
Composite: H̃(L|I) + FDE + OR + NDD + (1-FLRS̄). First operationalization of PIM Burden (JASIST 2022). Dashboard showing organizational health over time.

**Non-text / Mixed-type Cluster Evaluation** (file 24)
Real Downloads have screenshots, zip, code, Office files — not just PDFs. ProxAnn (ACL 2025) only works for text-heavy clusters. Need evaluation metric for mixed-type clusters. No existing method.

**macOS Provenance as Routing Signal** (file 36)
kMDItemWhereFroms (index 1 = referring page URL) gives domain-level routing signal: github.com → developer folder, moodle.ucy.ac.cy → course folder. How much accuracy does this add over semantic embeddings alone?

### ⬜ Tier 3 — Post-IUI 2027

**Save-Point Interception** (file 39) — needs user study, feasibility work
**Temporal Co-Access Graph** (file 40) — 4 weeks of access data needed
**Safe Automation Policy** (file 44) — important but can be incorporated into trust discussion
**Cold Start Personalization** (file 45) — valuable but needs more users
**AI-Readable Filesystem Standard** (file 43) — position paper, separate venue
**IPS-corrected CP / Algorithmic Monoculture** (file 31) — NeurIPS/ICML, separate paper
**Cognitive Health Monitoring** (file 28) — needs ethics board, separate paper

---

## What to DEFER Indefinitely (or reframe)

### ❌ Not in MVP, not in first paper, not in marketing

**Mental Health / Burnout Signals (Theme A, file 28 as product feature)**
Technically the most novel direction (JASIST 2020 confirms zero prior art). But:
- "Curator detects your burnout" = creepy, GDPR red flag
- Needs ethics board approval for any user study
- Alienates users who don't want inference about their mental state
→ **Reframe as:** 1 paragraph of "future work" in paper. Never in marketing.

**Life-Stage Detection (Theme B)**
Inferring parenthood, relationship status, career change from file patterns is too sensitive.
→ **Reframe as:** "Cluster evolution over time" — neutral framing, same technical content.

**Deep Conformal Prediction Theory (C2)**
Coverage-learning tradeoff with formal regret decomposition is NeurIPS/ICML level, not IUI.
→ **Keep basic CP in paper. Save formal theory for separate theory paper.**

**Energy Measurement**
Useful appendix fact: Level Router reduces compute. Not a research contribution by itself.

**Personal Knowledge Graph Schema Drift**
Only relevant after KG is built and deployed (Phase 6+).

---

## The IUI 2027 Paper Outline (Draft)

**Title:** Curator: Adaptive and Calibrated Local AI for Personal File Organization

**Abstract (50 words):** Curator is a privacy-first macOS system for personal file organization. It introduces an adaptive Level Router (tiered analysis from metadata to VLM), conformal prediction routing with formal coverage guarantees, a Markov-based File Biography, and FLRS (File Location Retention Score) — the first memory-aware file organization metric grounded in cognitive neuroscience.

**Sections:**
1. Introduction — the lost context problem, why file organization is hard
2. Related Work — PIM (Bergman, Elsweiler, Teevan), CP routing, spaced repetition in other domains, document organization
3. System Architecture — overview of Curator's pipeline
4. Level Router — tiered analysis, evaluation of level distribution, accuracy/compute tradeoff
5. Conformal Prediction Routing — LAC/RAPS scores, prediction sets, H(location|intent) connection, coverage validation
6. File Biography & Markov Entropy — per-file lifecycle model, instability detection
7. FLRS — formula, empirical grounding (Ebbinghaus, Talamini & Gorree, Antony & Paller), FSRS implementation, S₀ pilot study
8. Evaluation — FileNav benchmark, foreign user condition, review fatigue study, OBI measurement
9. Discussion — trust layer, safe automation, limitations
10. Conclusion + Future Work (mention cognitive health signals here, safely)

**Timeline (10 weeks to August 20, 2026):**
- Weeks 1-2: Implement Level Router + basic pipeline, start automated evaluation
- Weeks 3-4: S₀ pilot study (N=10, quick and dirty for calibration data)
- Weeks 5-7: FileNav user study (N=20, within-subjects, IRB must start NOW)
- Weeks 8-9: Analysis + writing
- Week 10: Revision + submission

**⚠️ IRB/Ethics review must begin immediately given timeline.**

---

## Research Identity in One Sentence

> Curator studies how a local AI system can organize personal files in a way that is useful, safe, explainable, and adapted to how users remember, forget, and re-find their files — with formal guarantees on routing quality and empirical grounding in the neuroscience of spatial memory.

---

## Mapping of All Research Files to Paper Sections

| File(s) | Paper Section |
|---|---|
| 05, 16, 21, 23, 31, 32 | §5 Conformal Prediction Routing |
| 06, 37 | §6 File Biography + Markov Entropy |
| 18, 25, 34 | §7 FLRS |
| 29, 41 | §5 + §8 H(L\|I), OBI |
| 33 | §8 Evaluation — FileNav Benchmark |
| 11, 13 | §8 Evaluation — methodology |
| 19 | §3 Architecture — Toponymy naming |
| 24, 35 | §3 Architecture — ColPali, ARISE |
| 36 | §3 Architecture — provenance routing |
| 14 | §3 Architecture — RAG + embeddings |
| 22 | §3 Architecture — KG (mention) |
| 27 | §3 Architecture — DataMapPlot (mention) |
| 38 | §1 Introduction — IUI scope |
| 26 | §2 Related Work — AMEDIA, autobiographical framing |
| 30 | §2 Related Work — Design for Forgetting |
| 28 | §10 Future Work |
| 39, 40, 43 | §10 Future Work |
| 42, 46 | §3 or §10 depending on implementation |
| 31 | §10 Future Work (IPS correction) |
| 44, 45 | §9 Discussion (safe automation, cold start) |
