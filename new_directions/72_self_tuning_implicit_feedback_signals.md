# Self-Tuning Implicit Feedback Signals
**File:** 72 | **Date:** 2026-06-01 | **Status:** Research Complete | **Priority:** HIGH

## Summary
Implicit feedback signals (file open frequency, dwell time, access recency, copy/move actions) are far more abundant than explicit ratings and can drive continuous self-tuning of Curator's organization model without requiring user effort. The key engineering challenge is debiasing: popular/recently created files get inflated signals. Curator should implement dwell-time normalization and positive-unlabeled (PU) learning framing to handle the missing-not-at-random nature of implicit feedback.

## Key Findings
- "Beyond Clicks: Dwell Time for Personalization" (ACM RecSys 2014, Yi & Hong) — item-level dwell time normalized across device/context types is a reliable proxy for relevance; still the standard approach in 2024
- Dual-stage filtering: remove short-dwell noise (<2s) and abnormal click bursts; improves downstream model quality significantly
- Multi-behavior alignment (arxiv 2305.05585): using multiple implicit signal types (open, edit, share, copy) together outperforms single-signal approaches; sparse target behaviors (explicit folder assignments) are enhanced by abundant auxiliary signals
- Positive-Unlabeled (PU) learning framing (PMC 12839574): implicit feedback is missing-not-at-random; files never opened aren't "disliked" — they may be unknown. PU learning handles this correctly.
- Reinforcement learning from user feedback (arxiv 2505.14946): RL framing can adapt organization policies from implicit interaction streams

## Relevant Papers / Prior Art
- "Beyond Clicks: Dwell Time for Personalization" — ACM RecSys 2014, Yi & Hong (dl.acm.org/doi/10.1145/2645710.2645724)
- "Improving Implicit Feedback-Based Recommendation through Multi-Behavior Alignment" — arxiv 2305.05585
- "Positive-Unlabeled Learning in Implicit Feedback from Data Missing-Not-At-Random Perspective" — PMC 12839574
- "Reinforcement Learning from User Feedback" — arxiv 2505.14946
- "Evaluating the Robustness of Learning from Implicit Feedback" — arxiv cs/0605036

## Applicability to Curator
Very High. Curator already intercepts file operations via FSEvents/file monitoring. Every open, move, rename, and delete is a signal. Concrete implementation:

```
Event log schema: (file_id, action_type, duration_ms, timestamp, context_folder)
Nightly batch:
  1. Normalize dwell_time by file_size_bucket and file_type
  2. Remove events < 2s and burst sequences (>20 events/minute)
  3. PU-frame: unobserved files are "unlabeled", not negative
  4. Update folder_affinity_score for each (file, folder) pair
  5. Files with rising folder affinity → promote in next suggestion cycle
```

This is the self-tuning loop that makes Curator improve without explicit user ratings.

## Open Questions
- What is the minimum event volume before implicit signal updates should fire? (Suggested: 50 events or 7-day batch, whichever comes first)
- How to prevent feedback loops where Curator's own suggestions inflate certain folders' apparent popularity?
- Privacy: implicit signals stored locally in SQLite — confirm no telemetry pathway exists

## Sources
- https://dl.acm.org/doi/10.1145/2645710.2645724
- https://arxiv.org/pdf/2305.05585
- https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12839574/
- https://arxiv.org/pdf/2505.14946
