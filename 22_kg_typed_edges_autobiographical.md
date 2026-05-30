# 22 — Knowledge Graph Typed Edges + Autobiographical Objects
_Date: 2026-05-30_

## Personal Knowledge Graph with 6 Typed Edges
**Verdict: 🟢 STRONG NOVELTY**

Existing systems have at most 2 edge types (content + temporal). No system combines all 6:
semantic_similarity + NER_co-occurrence + temporal_proximity + source_domain + citation_link + workflow_sequence

Prior art:
- Docs2KG (arXiv:2406.02962) — heterogeneous KG from enterprise docs, no workflow/provenance edges
- USPTO 12259911 — personal info graph, 2 edge types
- InfraNodus — semantic + explicit links (2 types), no NER/provenance/workflow
- Obsidian/Logseq — untyped wiki-links (Logseq forum explicitly calls typed edges "missed feature")

**Claim to make:** "No prior personal file organization system constructs a heterogeneous graph combining 6 typed dimensions: semantic, temporal, NER, provenance, citation, and behavioral."

Key citation: arXiv:2304.09572 — PKG Ecosystem Survey (Skjæveland et al.) — cite and differentiate.

## "Files as Autobiographical Objects" Framing
**Verdict: 🟡 MODERATE — framing has prior art, OPERATIONALIZATION is novel**

Strong prior art in HCI digital possessions research:
- **Odom et al. CHI 2012/2013/2014** — digital possessions as emotionally significant, identity-representing. "Technology Heirlooms?", "Digital Artifacts as Legacy" — VERY close to our framing
- **Bell & Gemmell MyLifeBits (CACM 2006)** — "digital archive as autobiography" — explicit in their rhetoric
- **AMEDIA model (2024)** — "Understanding Autobiographical Memory in the Digital Age" — recent, directly named
- **Lifelogging + autobiographical memory theory (PUC 2007)** — bridges autobiographical memory theory and system design

**What's NOVEL:** No prior work builds an ACTIVE FILE ORGANIZER that operationalizes this framing computationally — using forgetting curves, life-stage detection, usage patterns, and social provenance as ORGANIZING SIGNALS. Odom et al. design reflective interfaces; they do not automatically organize files.

**Claim to make:** "We take the autobiographical object framing from digital possessions research [cite Odom, Bell, AMEDIA] and, for the first time, instantiate it as an organizing architecture rather than a reflective design frame."

## 🚪 New Doors
1. **Odom et al. CHI 2012 "Design for Forgetting"** — They study DISPOSAL of digital possessions after breakups. Our antilibrary detection + FLRS-triggered deletion suggestions is a COMPUTATIONAL IMPLEMENTATION of "design for forgetting." Direct connection — cite explicitly.
2. **AMEDIA model (2024, Hutmacher et al.)** — Very recent (2024), directly theorizes digital resources and autobiographical memory. Could be the theoretical framework we anchor our paper to. Read it.
3. **"Slowness" and "re-visitation" (Odom CHI 2014)** — Designing for re-visitation aligns with FLRS-triggered file resurfacing. Our forgetting curve IS their "slowness" design principle, now computational.
