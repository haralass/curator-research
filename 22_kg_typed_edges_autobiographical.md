# 22 — Knowledge Graph Typed Edges + Autobiographical Objects
_Date: 2026-05-30_

## Personal Knowledge Graph with 6 Typed Edges
**Verdict: 🟢 STRONG NOVELTY**

Existing systems have at most 2 edge types (content + temporal). No system combines all 6:
semantic_similarity + NER_co-occurrence + temporal_proximity + source_domain + citation_link + workflow_sequence

Prior art:
- Docs2KG (arXiv:2406.02962) — heterogeneous KG from enterprise docs, no workflow/provenance edges [Docs2KG 2025]
- USPTO 12259911 — personal info graph, 2 edge types
- InfraNodus — semantic + explicit links (2 types), no NER/provenance/workflow
- Obsidian/Logseq — untyped wiki-links (Logseq forum explicitly calls typed edges "missed feature")

**Claim to make:** "No prior personal file organization system constructs a heterogeneous graph combining 6 typed dimensions: semantic, temporal, NER, provenance, citation, and behavioral."

Key citation: arXiv:2304.09572 — PKG Ecosystem Survey (Skjæveland et al. 2023) — cite and differentiate [Skjæveland et al. 2023].

## "Files as Autobiographical Objects" Framing
**Verdict: 🟡 MODERATE — framing has prior art, OPERATIONALIZATION is novel**

Strong prior art in HCI digital possessions research:
- **Odom et al. CHI 2012** — "Technology Heirlooms? Considerations for Passing Down and Inheriting Digital Materials" — digital possessions as emotionally significant, identity-representing [Odom et al. 2012]
- **Odom et al. CHI 2013/2014** — "Digital Artifacts as Legacy" — very close to our framing
- **Bell & Gemmell MyLifeBits (CACM 2006)** — "digital archive as autobiography" — explicit in their rhetoric [Gemmell et al. 2006]
- **AMEDIA model (2024)** — "Understanding Autobiographical Memory in the Digital Age" — recent, directly named [Hutmacher et al. 2024]
- **Lifelogging + autobiographical memory theory (PUC 2007)** — bridges autobiographical memory theory and system design

**What's NOVEL:** No prior work builds an ACTIVE FILE ORGANIZER that operationalizes this framing computationally — using forgetting curves, life-stage detection, usage patterns, and social provenance as ORGANIZING SIGNALS. Odom et al. design reflective interfaces; they do not automatically organize files.

**Claim to make:** "We take the autobiographical object framing from digital possessions research [Odom et al. 2012; Gemmell et al. 2006; Hutmacher et al. 2024] and, for the first time, instantiate it as an organizing architecture rather than a reflective design frame."

## References

- Docs2KG Authors. (2025). "Docs2KG: Unified Knowledge Graph Construction from Heterogeneous Documents Assisted by Large Language Models." *Proceedings of the 2025 ACM Web Conference*. arXiv:2406.02962.
- Skjæveland, M. G., Balog, K., Bernard, N., Łajewska, W., & Linjordet, T. (2023). "An Ecosystem for Personal Knowledge Graphs: A Survey and Research Roadmap." arXiv:2304.09572. Published in *AI Open* journal. DOI: 10.1016/j.aiopen.2024.01.001.
- Odom, W., Banks, R., Kirk, D., Harper, R., Lindley, S., & Sellen, A. (2012). "Technology Heirlooms? Considerations for Passing Down and Inheriting Digital Materials." *Proceedings of the SIGCHI Conference on Human Factors in Computing Systems (CHI '12)*, pp. 337–346. DOI: 10.1145/2207676.2207723.
- Gemmell, J., Bell, G., & Lueder, R. (2006). "MyLifeBits: A Personal Database for Everything." *Communications of the ACM*, 49(1), 88–95. DOI: 10.1145/1107458.1107460.
- Hutmacher, F., Kuhbandner, C., & Sedlmeier, P. (2024). "Understanding Autobiographical Memory in the Digital Age: The AMEDIA Model." *Perspectives on Psychological Science*. DOI: 10.1177/17456916231186920.
- Gurrin, C., Smeaton, A. F., & Doherty, A. R. (2014). "Lifelogging: Personal Big Data." *Foundations and Trends in Information Retrieval*, 8(1), 1–125. DOI: 10.1561/1500000033.
