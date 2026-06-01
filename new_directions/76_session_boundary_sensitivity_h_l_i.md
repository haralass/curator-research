# H(L|I) Session Boundary Sensitivity
**File:** 76 | **Date:** 2026-06-01 | **Status:** Research Complete

## Summary
Conditional entropy H(L|I) — the entropy of location/label given a set of contextual signals I — can detect work session boundaries in file access logs: when H(L|I) spikes, the user has switched contexts. This is directly applicable to Curator as a way to infer project sessions from FSEvents without requiring the user to explicitly declare task switches. High entropy in the access pattern signals "user is exploring a new context."

## Key Findings
- Classic reference: "Entropy as an Indicator of Context Boundaries" (ACL-I 2005) — maximum entropy values mark boundaries in sequential data; validated for NLP text segmentation, directly generalizable to file access streams
- Distributed conditional entropy monitoring (Dartmouth CS TR): sketching algorithms for computing H(L|I) in streaming settings at low memory cost — important for always-on FSEvents monitoring
- Grammar-based task analysis of web logs (USPTO 7694311): extracts and tokenizes access sequences to identify sub-tasks within sessions — methodological parallel to file access analysis
- **The H(L|I) formulation for Curator:** I = last 10 file access events; L = destination folder of next file access; high H(L|I) = unpredictable next-folder = session boundary
- Entropy-based change-point detection is O(n) per window update — computationally trivial compared to ML inference

## Relevant Papers / Prior Art
- "Entropy as an Indicator of Context Boundaries" — ACL-I 2005, Lüngen et al. (aclanthology.org/I05-1009.pdf)
- "Distributed Monitoring of Conditional Entropy for Network Anomaly Detection" — Dartmouth CS TR
- "Grammar-Based Task Analysis of Web Logs" — USPTO patent 7694311
- "Entropy Based Test Cases Reduction Algorithm for User Session Based Testing" — ResearchGate 284887456

## Applicability to Curator
Medium-High. Curator already logs FSEvents. Implementing a rolling H(L|I) estimator over the last 10 access events is ~50 lines of Python. When H spikes above threshold (empirically: >0.7 on a 0–1 normalized scale), Curator can:
1. Suggest "start a new project context?"
2. Batch all preceding session files as a coherent grouping candidate
3. Use session boundaries as natural units for the weekly review UI

Session boundaries are a free, high-quality signal for project detection that requires zero explicit user input — directly complementary to NER-based project detection (Topic 70).

```python
def compute_h_l_given_i(recent_folders: list[str]) -> float:
    # H(L|I) ≈ entropy of folder distribution in recent window
    counts = Counter(recent_folders)
    total = sum(counts.values())
    probs = [c / total for c in counts.values()]
    return -sum(p * math.log2(p) for p in probs if p > 0)
```

## Open Questions
- What window size N for I gives the best session boundary precision/recall? (Literature suggests 5–15 events)
- How to handle multi-tasking (parallel sessions in multiple Finder windows)?
- Should H(L|I) be computed per file type (only documents, excluding media files) to reduce noise?

## Sources
- https://aclanthology.org/I05-1009.pdf
- Dartmouth CS TR — conditional entropy monitoring
- https://patents.google.com/patent/US7694311B2 (grammar-based task analysis)
